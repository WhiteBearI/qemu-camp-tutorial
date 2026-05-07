# QEMU CPU 方向

!!! note "主要贡献者"

    - 作者：[@caspian](https://github.com/trace1729)

---

## 背景

计算机专业，之前参加过中国科学院大学 ["一生一芯" 项目][1]。想通过参加这次训练营来加深自己的理解，同时也想看心智社会中提到的[大型系统的分析方法](#总结)，处理一个中型大小的代码库是否有效。

感谢 @zevorn 老师搭建的实验框架和这一笔记分享与展示交流平台，让我能够分享自己阅读源码时的感悟。

## 核心任务

从 CPU 建模阶段来建立自己对 QEMU 这个大型系统的理解，为之后在 QEMU 做其他方向打下基础。

CPU 方向的具体任务：根据 G233 CPU 指令扩展手册中的指令规格，在 QEMU TCG 前端为 Xg233ai 扩展实现指令翻译。整体流程为 **Decodetree 译码 → TCG 翻译 → Helper 实现 → 测试验证**。

> 本文通过 QEMU 中的翻译流程切入问题，再根据遇到的问题逐步深入。

---

## 翻译流程：从 CPU 执行循环到指令译码

首先需要了解 QEMU 是如何把一条 guest 指令变成 host 指令的。可以从 `cpu_exec_loop` 开始追踪：

```text
cpu_exec_loop()
  └─ tb_gen_code()
       └─ setjmp_gen_code()
            └─ riscv_translate_code()        [translate.c:1438]
                 └─ translator_loop(..., &riscv_tr_ops, ...)
                      │  iterates per instruction:
                      └─ riscv_tr_translate_insn()    [translate.c:1368]
                           └─ decode_opc(env, ctx)    [translate.c:1242]
                                ├─ 16-bit: decode_insn16() directly
                                └─ 32-bit: iterate ctx->decoders[]
                                     └─ decode_insn32()  ← 基础指令译码
                                          ├─ 匹配 → calls trans_ADD(ctx, &u)
                                          └─ 不匹配 → returns false
```

调用链末端是 `decode_opc`，译码逻辑如下：

- 如果指令是 2 字节压缩指令，使用 `decode_insn16` 解码
- 如果指令是 4 字节常规指令，遍历 `ctx->decoders` 数组，按优先级使用各 decoder 尝试译码

```c
// target/riscv/translate.c:decode_opc()
    if (ctx->cur_insn_len == 2) {
        ctx->opcode = (uint16_t)opcode;
        if ((has_ext(ctx, RVC) || ctx->cfg_ptr->ext_zca) &&
            decode_insn16(ctx, opcode)) {
            return;
        }
    } else {
        for (guint i = 0; i < ctx->decoders->len; ++i) {
            riscv_cpu_decode_fn func = g_ptr_array_index(ctx->decoders, i);
            if (func(ctx, opcode)) {
                return;
            }
        }
    }
```

继续追踪 `ctx->decoders`：

- `ctx->decoders` 在 `riscv_tr_init_disas_context` 中由 `cpu->decoder` 赋值
- `cpu->decoder` 在 `riscv_tcg_cpu_finalize_dynamic_decoder` 中用预定义的 `decoder_table` 赋值

```c
const RISCVDecoder decoder_table[] = {
    { always_true_p, decode_insn32 },        // 基础 RISC-V ISA
    { has_xmips_p, decode_xmips },
    { has_xthead_p, decode_xthead },
    { has_XVentanaCondOps_p, decode_XVentanaCodeOps },
    // 自定义扩展在这里添加，如:
    // { has_Xg233ai_p, decode_Xg233ai },
};
```

`decoder_table` 中的每个 decoder（如 `decode_insn32`、`decode_Xg233ai`）是使用 `decodetree.py` 脚本读取对应的 `.decode` 文件自动生成的 C 代码。

追踪到这里，译码逻辑就比较清晰了：为了添加自定义指令集，需要在 `decoder_table` 中添加 Xg233ai 相关的项，然后在 `.decode` 文件中定义指令格式。

---

??? note "decoder_table 什么时候被设置？"

    ## 初始化链：decoder_table 是如何被设置的

    上文提到 `riscv_tcg_cpu_finalize_dynamic_decoder` 设置了 `cpu->decoder`，那又是谁调用的它呢？反向回溯调用链：

    ```text
    riscv_tcg_cpu_finalize_dynamic_decoder ← target/riscv/tcg/tcg-cpu.c
    riscv_cpu_finalize_features             ← target/riscv/cpu.c
    riscv_cpu_realize                       ← target/riscv/cpu.c
    ```

    `riscv_cpu_realize` 的函数指针由 `riscv_cpu_common_class_init` 设置，后者被注册为 `RISCVCPU` 类型的 `class_init`。

    ```typescript
    // DEFINE_TYPES(riscv_cpu_type_infos) 展开为:
    static void do_qemu_init_riscv_cpu_type_infos(void) {
        type_register_static_array(riscv_cpu_type_infos, ...);
    }
    type_init(do_qemu_init_riscv_cpu_type_infos)

    // type_init → module_init → __attribute__((constructor))
    static void __attribute__((constructor)) do_qemu_init_riscv_cpu_type_infos(void) {
        register_module_init(do_qemu_init_riscv_cpu_type_infos, MODULE_INIT_QOM);
    }
    ```

    > TypeInfo 定义了一个 class 的 schema:
    > - 这个 class 在整个系统中的 hierarchy 是什么 (parent)
    > - 如何创建 class (class_init)
    > - 如何创建 class 对应 instance (instance_init)
    > - 该类型生产出来的 instance 占据多大空间 (instance_size)

    **正向流程**：在 QEMU 初始化时，依次调用 `main()` → `qemu_init()` → `module_call_init()`，`module_call_init` 会遍历此前 `register_module_init` 注册的所有 module，调用对应的 init 函数（即 `do_qemu_init_xxx_infos`），将 TypeInfo 注册到全局类型哈希表中。

    使用 GDB 可以验证这个流程：

    ```text
    (gdb) b do_qemu_init_riscv_cpu_type_infos
    (gdb) bt
    #0  do_qemu_init_riscv_cpu_type_infos () at ../target/riscv/cpu.c:3405
    #1  module_call_init (type=MODULE_INIT_QOM) at ../util/module.c:109
    #2  qemu_init_subsystems () at ../system/runstate.c:979
    #3  qemu_init (...) at ../system/vl.c:2895
    #4  main (...) at ../system/main.c:71
    ```

    接着看 `riscv_cpu_common_class_init` 是什么时候被调用的：

    ```text
    #0  riscv_cpu_common_class_init (c=0x...)
    #1  type_initialize (ti=0x...) at ../qom/object.c:417
    #2  type_initialize (ti=0x...) at ../qom/object.c:365         ← 父类初始化
    #3  type_initialize (ti=0x...) at ../qom/object.c:365         ← 逐层向上
    #4  ...
    #5  object_class_foreach_tramp (...) at ../qom/object.c:1110
    #6  g_hash_table_foreach ()
    #7  object_class_foreach (...) at ../qom/object.c:1132
    #8  object_class_get_list (...) at ../qom/object.c:1189
    #9  select_machine (...) at ../system/vl.c:1679
    #10 qemu_create_machine (...) at ../system/vl.c:2192
    #11 qemu_init (...) at ../system/vl.c:3766
    #12 main (...) at ../system/main.c:71
    ```

    这个 backtrace 初看有些令人困惑——riscv_cpu_common_class_init 为什么会在 `qemu_create_machine` 时被调用？原因和 QEMU 中初始化类型的逻辑有关：

    - `qemu_create_machine` 通过 `select_machine` 在类型哈希表中查找命令行指定的 machine type（比如 `-M g233`）
    - 此处的"查找"并非简单的 key-value 查找，而是遍历整个类型哈希表，将符合要求的类型对象收集到链表中
    - 遍历过程中，**路径上遇到的所有未初始化的类型都会被初始化**，包括它们的父类和子类

    `select_machine` 获取 machine_type 之后，会使用该类型 class_init 设置的 realize 回调（即 `virt_machine_init`）启用 machine：

    - 根据 machine 结构体中设置的 `default_cpu_type` 实例化 CPU
    - 调用 `riscv_cpu_realize` 设置 CPU 运行的线程函数 `mttcg_start_vcpu_thread`（内部调用执行核心逻辑 `cpu_exec_loop`）
    - 在这里调用了我们最开始关心的 `riscv_tcg_cpu_finalize_dynamic_decoder` 函数

    ```text
    #0  riscv_tcg_cpu_finalize_dynamic_decoder (cpu=0x...)
    #1  riscv_cpu_finalize_features (cpu=0x...)
    #2  riscv_cpu_realize (dev=0x...)
    ...
    #9  riscv_hart_realize (s=0x..., idx=0, cpu_type="gevico-cpu-v1")
    #10 riscv_harts_realize (dev=0x...)
    #11 device_set_realized (obj=0x...)
    ...
    #18 virt_machine_init (machine=0x...) at hw/riscv/g233.c:1585
    #19 machine_run_board_init (machine=0x...)
    ...
    #22 qemu_init (...) at ../system/vl.c:3847
    #23 main (...) at ../system/main.c:71
    ```

    文件职责总结：

    | 文件 | 职责 |
    |---|---|
    | `target/$ARCH/cpu.c` | 定义 $ARCH 的 CPU 蓝图 |
    | `hw/$ARCH/$MACHINE` | 定义 $ARCH 下 $MACHINE 的虚拟化实现 |

---

## 添加自定义指令集扩展

### 插桩：从命令行到后端

> 从在命令行指定扩展，到在后端翻译时启用扩展支持，这个过程是怎么在 QEMU 中发生的？

关键数据结构：`RISCVCPUConfig`、`isa_edata_arr`、`riscv_cpu_vendor_exts`。

时间线：

1. 在 `riscv_tcg_cpu_instance_init` 创建 instance 时，遍历 `riscv_cpu_vendor_exts`，为每一个扩展在 instance 的 property 哈希表中注册
2. 命令行中指定的 ISA 字符串与 `isa_edata_arr` 做比较，使用对应的 getter/setter 设置 `cfg->ext_xg233ai = true`
3. `isa_edata_arr` 同时负责权限检查（如某些扩展需要更高权限的硬件支持）
4. 在 decoder 中，`decoder_table` 的条件函数（如 `has_Xg233ai_p`）检查 `cfg->ext_xg233ai` 决定是否启用该 decoder

```c
object_property_add(obj, "xtheadba", "bool",
    cpu_get_multi_ext_cfg,       // getter: 读 cfg->ext_xtheadba
    cpu_set_multi_ext_cfg,       // setter: 写 cfg->ext_xtheadba
    NULL, (void*)&ENTRY);        // OPAQUE = 数组条目
```

所以为了添加自定义扩展，需要在 `RISCVCPUConfig`、`isa_edata_arr`、`riscv_cpu_vendor_exts` 三个关键数据结构中各添加一项。

> 为了兼容 QEMU 的构建系统，还需要修改 `meson.build`，并在 build 目录中添加对应的头文件（decodetree 构建产物和用户自定义的 `trans_Xg233ai.c.inc`）

### 指令实现

具体的指令实现可分为 5 步：

1. 在 `target/riscv/Xg233ai.decode` 中插入指令模式
2. 在 `target/riscv/insn_trans/trans_Xg233ai.c.inc` 中编写对应的 `trans_{insn}` 函数，该函数调用 `op_helper.c` 中的 `gen_helper_{insn}` 辅助函数
3. 在 `target/riscv/helper.h` 中声明 helper 函数原型
4. 根据指令语义，在 `target/riscv/op_helper.c` 中实现 `helper_{insn}` 函数
5. 使用 `make -C build/tests/gevico/tcg/riscv64-softmmu run-insn-{insn}` 检查实现是否正确

#### Step 1: .decode 文件

根据[模拟客户机指令](https://qemu.gevico.online/tutorial/2026/ch2/qemu-insn/#_5)可以了解到 .decode 文件的四层抽象：

- **Field** — 字段提取器，定义如何从指令编码中提取特定位段
- **Argument** — 字段容器，将提取出的字段打包为结构体
- **Format** — 编码布局容器，将 Field 组合成指令格式
- **Pattern** — 具体指令，指定指令编码位模式，匹配成功后调用 `trans_{insn}` 函数

自顶向下分析：Pattern 提取指令的 `opcode`、`func3`、`func7` 字段，与模板比较；匹配成功后，使用 Format 提取指令各字段（如 `rd`、`rs1`、`rs2`）；Format 具体提取什么字段由 Argument 指定并打包为结构体；Field 则提供提取字段的具体方法。

#### Step 2–5: trans 函数和 Helper

通过观察[硬件手册](https://qemu.gevico.online/exercise/2026/stage1/cpu/cpu-datasheet/)，指令实现涉及三种数据元素的操作：

1. **获取寄存器的值**：`gen_helper_*` 中通过 `env->gpr[rs1]` 直接访问 RISC-V 通用寄存器
2. **读取 guest 地址的数据**：`cpu_ld{size}_mmuidx_ra(env, addr, mem_idx, ra)`，其中 size 为 b(8位)/w(16位)/l(32位)/q(64位)
3. **写入 guest 地址**：`cpu_st{size}_mmuidx_ra(env, addr, value, mem_idx, ra)`

需要传入正确的 `MemOpIdx`。`MemOpIdx` 包含了两个信息：

- `MemOp`：访问大小（字节/半字/字/双字）和端序
- `mmuidx`：MMU 索引，决定使用哪个页表做地址翻译

详情可参见 [MMU 文档](https://qemu.gevico.online/tutorial/2026/ch2/qemu-softmmu/)。

有了这三类 API 之后，在 helper 函数中实现指令语义和写普通的 C 代码差别不大。

---

## 调试记录

### 问题 1：忘记打开扩展选项

在实现 DMA 指令时，忘记打开对应机器的扩展选项。运行测试程序时 QEMU 没有直接 crash，而是 crash 在测试断言里，令人摸不着头脑。

使用 `-d` 参数查看 QEMU 的翻译日志，发现确实生成了 **illegal instruction** 异常。那为什么 QEMU 在这种情况下还可以继续运行呢？

在[模拟中断和异常](https://qemu.gevico.online/tutorial/2026/ch2/qemu-intr/)这一节中，发现 QEMU 在翻译流程中如果遇到了异常，会设置 PC 为 `stvec`。`stvec` 的值又是在哪里设置的呢？

在编译 `test-xxx` 测试用例时，会和裸机运行时环境 `tests/gevico/tcg/riscv64/crt` 一起构建。在 `crt/crt.S` 中指定了 `exception_handler`，并将其设置为 `stvec` 的值。exception_handler 的处理逻辑为**直接跳过当前指令**，所以即使遇到 illegal instruction，程序也能继续执行，最终在测试断言处失败。

### 问题 2：trans 函数中误用 C 整型

在实现 `trans_vdot` 时，如果直接传递 `a->rd` 给 `gen_helper_vdot`，会导致编译报错：

```c
// 错误：gen_helper_* 期望 TCGv 参数，不是 C 的 int
static bool trans_vdot(DisasContext *ctx, arg_r *a) {
    gen_helper_vdot(tcg_env, a->rd, a->rs1, a->rs2);
    return true;
}
```

原因：在 `trans_` 系列函数内部，所有操作都应生成 TCG 中间表示。TCG IR 的每条指令参数都必须是 `TCGv`（虚拟寄存器），即使是立即数也需要用 `tcg_constant_tl` 将 C 的 int 包装成 TCG 能理解的"常量寄存器"：

```c
// 正确：用 tcg_constant_tl 包裹
static bool trans_vdot(DisasContext *ctx, arg_r *a) {
    gen_helper_vdot(tcg_env,
        tcg_constant_tl(a->rd),
        tcg_constant_tl(a->rs1),
        tcg_constant_tl(a->rs2));
    return true;
}
```

`gen_helper_*` 也是类似的原理——它生成一条 `INDEX_op_call` 的 TCG IR 指令，用于在翻译时插入对一个 C 函数（helper）的调用。

---

## 总结

### 如何分析像 QEMU 这样的大型系统

我认为 Marvin Minsky 在《心智社会》中提到的方法比较实用：

1. 一个大型系统由数百万个各自只做琐碎小事的部件组成。
2. 理解一个复杂系统的方式：
    - 第一，理解每个子模块各自的功能；
    - 第二，理解子模块之间如何交互；
    - 第三，从更大的视角理解系统整体"做什么"。
    > 因为要真正"知道"一个东西，必须知道它的外部效应。

??? note "还原论和整体论"
    实际上是两种观点的综合。

    - [还原论](https://en.wikipedia.org/wiki/Reductionism)：认为复杂系统可以解释为其各部分的总和
    - [整体论](http://en.wikipedia.org/wiki/Holism)：认为系统作为整体具有超越其组成部分属性的特性

具体到 QEMU：

1. **从外部效应建立基本框架**：在本题框架内，QEMU 的核心工作流是：guest 二进制指令 → [解码] → [翻译成 TCG 中间表示] → [编译成 host 机器码] → [在真实 CPU 上执行]
2. **展开到细节**：设备的接入、CPU 的配置、内存的地址翻译，都是围绕这条主线的工作
3. **继续拆解到对象交互**：QOM 类型系统

QEMU 由数百个模块组成，每个模块（CPU 类型、机器板卡、网卡、PCI 总线）都是一个 QOM 类，QOM 定义了一套通用接口：

1. **TypeInfo**：注册一个类——名字、大小、父类、初始化函数
2. **Class 和 Instance**：
    - `callback function`：定义一个对象与外界交互的方式
    - `persistent data structure`：存储自身信息的数据结构

通过 LLM，可以快速地将 QEMU 的功能映射为 QOM 对象之间的交互：[opencode 示例](https://opncd.ai/share/pnjNBbeR)

> 其实 The Society of Mind 这段描述是 Marvin Minsky 对智能的定义，和现在的 Agent 系统比较相似。

### 如何理解函数回调

QEMU 的实现中充斥着各种函数回调，理解起来比较困难。抓住两个关键行为进行分析：

- **函数回调设置点**：回调函数在哪里被注册/赋值
- **函数回调调用点**：回调函数在哪里被调用

[1]: https://ysyx.oscc.cc/
