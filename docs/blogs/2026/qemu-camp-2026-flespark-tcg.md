# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"

    - 作者：[@flespark](https://github.com/flespark)

---

## 背景介绍

作为一名工作了7年，工作方向横跨单片机，Android BSP，Linux后台，最近又转到CPU芯片验证/机密计算开发的行业老兵。近2年工作中高强度沉迷于vibe coding，深感在复杂知识库背景、测试相对困难的系统底层开发领域，缺乏一种在无需人工审核时仍能有效验证AI生成代码的通用闭环方案。使用软件模拟硬件不失为该方案的最优路径，早就了解到qemu作为支持平台最多和功能最完整的硬件模拟器，但浅尝之后就被qemu复杂的命令行和代码库阻拦。幸运的是在b站刷到泽文老师的qemu教学视频，也刚好赶上新年度qemu训练营开课。
希望通过训练营的学习，对系统虚拟化建立更深的理解，能够使用qemu解决工作中的问题，找到一个深入学习的切入点。同时希望为开源社区贡献绵薄之力，推动落实AI能力提效系统底层开发。
非常感谢泽文老师在百忙之中搭建的这样一个极具极客精神的开放社区，为众多的ICer探索前沿技术铺平了道路。我作为半路出家的ICer，汲取前辈们的宝贵经验的同时，也会努力尝试分享一些基于个人工作经历的技术理解和思考。

## 专业方向

因为自己对qemu的理解比较表面，这里选择了对于前置知识要求最低的tcg方向作为专业阶段选题。专业方向要求基于qemu tcg提供的接口新增10条riscv扩展指令，10条指令全部是处理数据量比较大的向量指令，涉及矩阵转置、排序、矩阵展开和压缩、向量加法和矩阵乘法等操作，全部是跑AI应用所依赖的。初步尝试使用tcg vec ops实现发现太复杂，遂全部基于tcg helper实现。整体做下来感觉都较简单，重点训练对helper接口的熟悉程度。

## 开发环境

使用CNB云服务器作为远程开发环境编译和跑测试，本地使用的cursor IDE编写代码。

## 学习概述

写代码时关闭了IDE的自动补全，通过bear和clangd在编译时生成qemu的代码符号索引，以熟悉tcg接口的使用（其实开了补全很多也是错的，貌似tcg接口在一些qemu版本间变更比较大）。先自己尝试写一遍，然后编译排除接口的错误使用。再跑测试用例，跑不过就用gdb来定位。最后如果gdb也看不出来才呼叫AI来帮忙，挽救退化严重的编码能力。

整体按照TDD解题步骤，除了第1个习题涉及了一些训练营课程没有介绍到的修改点，剩下的新增指令步骤都是：
1. 在`target/riscv/xg233.decode`中新增decodetree指令格式
2. 在`target/riscv/insn_trans/trans_xg233.c.inc`中新增用于提取context的tcg翻译函数
3. 在`target/riscv/helper.h`中添加用于编译时语义检查helper宏定义声明
4. 在`target/riscv/op_helper.c`中基于helper接口实现具体helper函数

## 学习内容

### 新增CPU和指令集的定义

在添加新的xg233指令之前，要先在相应文件中添加g233 cpu和xg233指令的定义，包含如下4方面改动：
1. 在`target/riscv/cpu-qom.h`中新增g233 cpu type;
2. 在`target/riscv/cpu_cfg_fields.h.inc`中新增xg233扩展定义，在`target/riscv/cpu_cfg.h`中新增统一的通过扩展查询cpu类型接口，同时将扩展归类到`target/riscv/cpu.c`的riscv厂商扩展列表中, 也在`target/riscv/translate.c`中将`has_xg233_p`扩展与`decode_xg233`指令集关联；
3. 在`target/riscv/cpu.c`通过`DEFINE_RISCV_CPU`将完整的cpu属性注册到qemu中，包括寻址位宽XLEN，基础扩展支持，增强扩展支持及版本，私有扩展支持，mmu、iopmp及其他非指令集特性的支持情况;
4. 在预先定义好的qom init接口文件`hw/riscv/g233.c`中将`mc->default_cpu_type`由`TYPE_RISCV_CPU_GEVICO_CV1`改为`TYPE_RISCV_CPU_G233`.
应用这些修改重新编译qemu后，通过`./build/qemu-system-riscv64 -cpu help`确认g233 cpu添加成功，通过跟踪qemu的启动过程可以看到g233的cpu配置注册到qom中的过程:
```sh
./build/qemu-system-riscv64 -M g233 -m 2G -nographic -trace "object_*"
```

### 实现内存到内存搬移矩阵转置DMA指令

添加新的指令集支持第一步是新增decodetree文件描述具体指令格式，参照`target/riscv`其他私有指令实现添加固定的`Fields`，`Argument Sets`和`General Formats`描述：
```
# Fields:
%rs2       20:5
%rs1       15:5
%rd        7:5

# Argument sets:
&r      rd rs1 rs2              !extern

# Formats:
@r    .......   ..... ..... ... ..... .......  &r  %rs2 %rs1 %rd
```
然后添加dma指令对应的具体编码说明：
```
# *** XG233 Extend Instructions ***
dma     0000110 ..... ..... 110 ..... 1111011 @r
```
decodetree文件在编译C代码前会由meson构建脚本生成相应的解码函数，所以需要在meson中添加对新增decodetree文件的检查。

每条指令需提供相应的`trans_*`翻译函数将guest指令的语义翻译TCG中间码，其定义在`target/riscv/insn_trans/trans_xg233.c.inc`中，无论是基于tcg ops还是helper实现的tcg翻译函数，这里只是通过`get_gpr`提取guest的寄存器值等指令所依赖的context，再传递到具体的ops或helper实现。这种分层的核心思想是编译时（翻译时）和运行时的职责分离：`trans_*`是编译时入口，处理指令解码结果的映射和前置条件检查；ops/helper 函数是可复用的运行时逻辑单元。

这种分层复用，优化第一的思想又落实到了helper的声明。
具体是通过`DEF_HELPER_FLAGS_XXX`这类宏在`target/riscv/helper.h`中声明，`helper.h`会被三个 .inc 文件分别 #include，每次 include 之前`DEF_HELPER_FLAGS_XXX`宏的定义不同，因此展开结果也不同：
| 展开文件 | 生成物 | 用途 |
|---|---|---|
| `helper-proto.h.inc` | C 函数原型 `helper_*()` | 给实际 C 实现文件提供函数声明 |
| `helper-gen.h.inc` | `gen_helper_*()` inline 函数 | 在翻译阶段生成 TCG IR 中的 helper call |
| `helper-info.c.inc` | `TCGHelperInfo helper_info_*` | 运行时元数据，供 TCG 调度/优化 |
这种设计一方面避免手写重复且易错的声明，又可以给到tcg额外的编码信息以进行调用优化。

QEMU提供以下几类helper接口，用于模拟guest指令的语义：

A. 访存接口
这是最常见的一类。
cpu_ldb_data_ra / cpu_stb_data_ra：8 位
cpu_lduw_data_ra / cpu_stw_data_ra：16 位
cpu_ldl_data_ra / cpu_stl_data_ra：32 位
cpu_ldq_data_ra / cpu_stq_data_ra：64 位
这些接口都是“按 guest 内存语义访问”，不是 host 内存操作。

还有对应的非 ra 版本，通常在你不需要异常定位信息时用：
cpu_ldl_data
cpu_stl_data
以及更底层的 mmuidx 版本：
cpu_ldl_mmuidx_ra
cpu_stl_mmuidx_ra
当你需要自己指定 MMU 上下文时会用到它们，比如不同特权态、虚拟化态或者特殊访问模式。

B. 指令取值接口
如果是在解码器或取指相关逻辑里，通常用的是 code 系列：
cpu_ldub_code
cpu_lduw_code
cpu_ldl_code
cpu_ldq_code
它们和 data 系列的区别是：
code 代表取指语义，data 代表普通数据访存语义。这在权限检查、I-cache/TLB 路径上可能不同。

C. 异常和退出
helper 里常会见到：
GETPC()
cpu_loop_exit()
cpu_loop_exit_restore()
riscv_raise_exception()
如果指令需要显式抛异常，通常会走这些路径。
比如一个 helper 中做非法操作时，常见写法是：
riscv_raise_exception(env, RISCV_EXCP_ILLEGAL_INST, GETPC());

D. TCG 值和翻译器接口
helper 不是直接被 CPU 执行，而是由 translator 生成调用。
常见链路是：
helper.h 里写 DEF_HELPER_*
op_helper.c 里实现 HELPER(name)
trans_*.c.inc 里用 gen_helper_name(...)

E. 通用辅助工具
helper 开发里也经常用这些：
qemu_log_mask()：写调试日志
assert() / g_assert()：开发阶段快速抓 bug
memcpy() / memset()：只在你已经拿到 host 可直接访问的 RAM 指针时才适合
probe_read() / probe_write()：需要检查访问合法性时用
address_space_* 系列：更底层的内存访问 API

dma指令要求能够对不同大小的FP32矩阵执行转置操作，具体是将源地址的矩阵的行列数据互换后写入目标地址。因为实验手册规定源区域和目标区域不得重叠，这里简单通过2级遍历按索引复制对应位置的数据：使用`cpu_ldl_data_ra(env, src + src_off, ra)`读出源数据，再使用`cpu_stl_data_ra(env, dst + dst_off, val, ra)`写入目标位置。

然后重新执行编译，执行`make docker-image-debian-all-test-cross`下载对应的用于运行测试的docker镜像，之后执行`make -C build/tests/gevico/tcg/riscv64-softmmu/  run-insn-dma`执行测试用例。


### 剩余指令的实现

其他指令的模拟也都基于dma实现囊括的接口函数，部分指令是将结果写回通过gen_set_gpr写回guest寄存器而不是内存。在实现矩阵乘法gemm指令时，因为指令测试程序编译配置和qemu的新增x233 cpu配置不一致，从而触发了一个比较绕的隐藏bug。首先使用gdb定位，运行到如下测试用例代码验证的gemm计算结果时：
```
static void test_gemm_identity(void)
{
    int identity[DIM * DIM] = {
        1, 0, 0, 0,
        0, 1, 0, 0,
        0, 0, 1, 0,
        0, 0, 0, 1
    };

    /* A = {1..16} */
    for (int i = 0; i < DIM * DIM; i++)
        mat_a[i] = i + 1;

    memset(mat_hw, 0, sizeof(mat_hw));

    custom_gemm(mat_hw, mat_a, identity);
    compare(mat_hw, mat_a, DIM * DIM);
}
```
程序会在走到for的时候崩溃，并且显式初始化的identity包含异常的负数。借助AI深入排查到根本原因是测试用例的编译选项打开了RVV优化支持，但前面定义g233 cpu时添加的基本扩展配置为：
```
    DEFINE_RISCV_CPU(TYPE_RISCV_CPU_G233, TYPE_RISCV_VENDOR_CPU,
        .misa_mxl_max = MXL_RV64,
        .misa_ext = RVI | RVM | RVA | RVC | RVU | RVF | RVD,
        .priv_spec = PRIV_VERSION_1_12_0,
        .vext_spec = VEXT_VERSION_1_00_0,
        ...
    )
```
其中的
`.vext_spec = VEXT_VERSION_1_00_0,`
看似启用了RVV，但正确启用需要配置
`.misa_ext = RVI | RVM | RVA | RVC | RVU | RVF | RVD | RVV,`
而且在遇到非法指令异常时，测试用例代码`crt.S`中的异常指令处理汇编`trap`只是简单地把pc往前跳4个字节，导致问题埋得比较深。

## 总结

之前觉得学习qemu到能够去解决工作中实际的问题的程度会很难，但是通过老师精心编排的课程，同时借助AI的知识库，能够步步为营理解qemu中各种概念和设计范式，也解决真实的问题，收获了很多的自信也培养了兴趣。但是当前阶段只是面对模块的复制编程，没有涉及架构设计思想和性能优化路径，但凭自己的基础去硬啃这些前沿方向还是会吃力。希望下面可以补全剩下的几套进阶课程，拓展对qemu应用方式的了解，才能结合工作找一个可以在qemu上深挖的方向，为riscv及开源社区的发展做出实在的贡献。