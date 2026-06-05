# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"

    - 作者：[2zuqisong](https://github.com/2zuqisong)

---

## 背景介绍

计算机科学与技术专业，参加过操作系统训练营，常使用qemu，希望这次训练营能对qemu的使用和结构有更深入的了解和掌握。

## 专业阶段

选择的方向是CPU建模（TCG）

### QEMU 怎么跑一条指令？

QEMU 的核心是 **动态二进制翻译**：将 Guest 指令（RISC-V）实时翻译成 Host 指令（x86/ARM）并执行。这个翻译引擎叫 **TCG（Tiny Code Generator）**。

使用llm去对qemu翻译的流程进行追踪，整个调用链如下：

```
cpu_exec_loop()              ← 主循环，不断取指执行
  └→ tb_lookup()             ← 先查 TranslationBlock 缓存
       ├→ 命中 → tcg_qemu_tb_exec()  → 直接跳到 Host 代码执行
       └→ 未命中
            └→ tb_gen_code()
                 └→ translator_loop()
                      ├→ decode_opc()     ← 解码器调度
                      │    └→ decode_Xg233ai()  ← 我们的 decodetree
                      │         └→ trans_xg233ai_vrelu()
                      │              └→ gen_helper_xg233ai_vrelu()
                      └→ tcg_gen_code()   ← 生成 Host 机器码
                           └→ tcg_qemu_tb_exec()  ← 执行
```

一些相关概念：

- **Translation Block (TB)**：一段连续的 Guest 指令序列，以分支指令结束。翻译和缓存的基本单位。
- **TCG IR**：QEMU 内部的中间表示，介于 Guest 指令和 Host 指令之间。`gen_helper_xxx()` 本质上是在构造 TCG IR。
- **decodetree**：QEMU 的指令解码代码生成器。你写一个 `.decode` 文件描述位模式，它自动生成 C 代码。
- **helper 函数**：用纯 C 写的模拟函数。因为有些操作太复杂，不便直接用 TCG IR 表达（比如内存循环访问），就调用一个 C 函数来完成，这个 C 函数就是 helper。

### 翻译路径

```
RISC-V 指令字 (32-bit)
        │
        ▼
┌──────────────────────┐
│  decodetree          │  ← .decode 文件定义
│  (位匹配 & 解码)      │
└──────┬───────────────┘
       │ 提取 rd, rs1, rs2
       ▼
┌──────────────────────┐
│  trans_xxx()         │  ← .c.inc 文件
│  (生成 TCG IR)        │    调用 gen_helper_xxx()
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  gen_helper_xxx()    │  ← 从 helper.h 自动生成
│  (设置参数, 发起调用)  │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  HELPER(xg233ai_xxx) │  ← helper.c 文件
│  (C 语言实现核心逻辑)  │    读写 guest 内存
└──────────────────────┘
```

---

在对整个翻译流程的疏通后，对指令的扩展就涉及到以下几个步骤：

1、在Xg233ai.decode中编写指令的格式

```
xg233ai_vrelu     1010110  ..... ..... 110 ..... 1111011 @r
```

`decodetree` 工具在编译时会处理这个文件，生成 `decode-Xg233ai.c.inc`（解码器）和 `trans_xg233ai_vrelu()` 的函数声明。

2、在insn_trans/trans_xg233ai.c.inc中编写翻译函数

```c
#define GEN_XG233AI_HELPER3(name) \
static bool trans_xg233ai_##name(DisasContext *ctx, arg_r *a) \
{ \
    TCGv dest = get_gpr(ctx, a->rd, EXT_NONE); \
    TCGv src1 = get_gpr(ctx, a->rs1, EXT_NONE); \
    TCGv src2 = get_gpr(ctx, a->rs2, EXT_NONE); \
    gen_helper_xg233ai_##name(tcg_env, dest, src1, src2); \
    return true; \
}

GEN_XG233AI_HELPER3(vrelu)   
```

**这条宏做的事情**：
1. `get_gpr(ctx, a->rd, EXT_NONE)` — 从 RISC-V 寄存器文件读出 rd 寄存器的值（放到 TCG 临时变量）
2. 同理读出 rs1、rs2
3. `gen_helper_xg233ai_vrelu(tcg_env, dest, src1, src2)` — 生成一行 TCG IR，表示"调用 helper，传入这 4 个参数"

3、在helper.h中声明helper函数签名

```c
DEF_HELPER_4(xg233ai_vrelu, void, env, tl, tl, tl)
```

4、在xg233ai_helper.c中实现helper函数

```c

void HELPER(xg233ai_vrelu)(CPURISCVState *env,
                            target_ulong dst,    // a->rd 的值：目标数组基址
                            target_ulong src,    // a->rs1 的值：源数组基址
                            target_ulong n)      // a->rs2 的值：元素个数
{
    long count = (long)n;

    for (long i = 0; i < count; i++) {
        // 从 Guest 内存读取一个 32 位有符号整数
        uint32_t val = cpu_ldl_data(env, src + i * 4);

        // ReLU: max(0, val)
        if ((int32_t)val < 0) {
            val = 0;
        }

        // 写回 Guest 内存
        cpu_stl_data(env, dst + i * 4, val);
    }
}
```
在后续实现指令时，发现有两种指令模式：

1、一种是直接写入内存，无需返回。内存->内存

2、一种是需要返回结果的，寄存器要需要接收Helper的返回值，写回 gpr 。比如vadot、vmax两条指令

注：常见的内存访问函数：

| 函数 | 含义 |
|------|------|
| `cpu_ldl_data(env, addr)` | 读 32-bit little-endian |
| `cpu_stl_data(env, addr, val)` | 写 32-bit little-endian |
| `cpu_ldub_data(env, addr)` | 读 8-bit unsigned |
| `cpu_stb_data(env, addr, val)` | 写 8-bit |

5、在cpu_cfg_fields.h.inc和cpu_cfg.h中添加扩展字段

```c
BOOL_FIELD(ext_Xg233ai)   // 给 CPUState 结构体加一个 bool 成员
```

```c
MATERIALISE_EXT_PREDICATE(Xg233ai)  // 展开出 has_Xg233ai_p() 函数
```

6、在cpu.c中注册ISA扩展和CPU profile

```c
MATERIALISE_EXT_PREDICATE(Xg233ai)  // 展开出 has_Xg233ai_p() 函数
```

7、在translate.c中引入decode文件和加入解码器表

```c
#include "insn_trans/trans_xg233ai.c.inc"

#include "decode-Xg233ai.c.inc"

const RISCVDecoder decoder_table[] = {
    { has_Xg233ai_p, decode_Xg233ai }, 
};
```
8、执行测试指令

```bash
make -C build/tests/gevico/tcg/riscv64-softmmu/  run-insn-vrelu
```

## 进阶实验

### QEMU 指令执行流程

对于一条 Guest 指令：

```text
Guest 指令
    ↓
Decodetree 解码
    ↓
trans_xxx()
    ↓
TCG IR
    ↓
Host 机器码
    ↓
执行
```

---

### Helper 实现方案

### 翻译函数

```c
static bool trans_xg233ai_vrelu(DisasContext *ctx, arg_r *a)
{
    TCGv dst = get_gpr(ctx, a->rd, EXT_NONE);
    TCGv src = get_gpr(ctx, a->rs1, EXT_NONE);
    TCGv n   = get_gpr(ctx, a->rs2, EXT_NONE);

    gen_helper_xg233ai_vrelu(
        tcg_env,
        dst,
        src,
        n);

    return true;
}
```

## Helper 函数

```c
void HELPER(xg233ai_vrelu)(
    CPURISCVState *env,
    target_ulong dst,
    target_ulong src,
    target_ulong n)
{
    for (long i = 0; i < n; i++) {
        int32_t v =
            cpu_ldl_data(env, src + i * 4);

        if (v < 0) {
            v = 0;
        }

        cpu_stl_data(
            env,
            dst + i * 4,
            v);
    }
}
```

---

### 翻译后的 TB

实际上只会生成：

```text
call helper_xg233ai_vrelu
```

一条 Helper 调用。

执行时：

```text
TB
 ↓
call helper
 ↓
执行 C 函数
 ↓
return
 ↓
继续执行 TB
```

---

### 内联 TCG 实现方案

对于固定长度向量（16个），可以直接展开。

## vadd 示例

```c
for (int i = 0; i < 16; i++) {
    tcg_gen_qemu_ld_i32(...);
    tcg_gen_qemu_ld_i32(...);
    tcg_gen_add_i32(...);
    tcg_gen_qemu_st_i32(...);
}
```

生成的 TCG IR 类似：

```text
ld
ld
add
st

ld
ld
add
st

...
```

随后被翻译成 Host 机器码。

---

### 性能对比

### Helper

翻译后的 TB：

```text
call helper
```

执行路径：

```text
Host Code
 ↓
call helper
 ↓
保存寄存器
 ↓
切换 ABI
 ↓
执行 C 函数
 ↓
恢复寄存器
 ↓
return
```

存在明显的调用开销。

---

### 内联 TCG

翻译后的 TB：

```text
ld
ld
add
st

ld
ld
add
st
...
```

执行路径：

```text
Host Code
 ↓
直接执行
```

没有函数调用。

---

对于固定长度、简单数据通路的 AI 指令：使用内联

因为能够消除 Helper 调用开销，并允许 TCG 后端进一步优化。

对于复杂算法和动态循环：

Helper 更优

因为实现简单、可维护性高，同时避免生成过于庞大的 TB。

## 总结

在CPU实验中，系统地了解了 QEMU TCG 的整体架构和指令执行流程，从最初只会使用 QEMU 运行操作系统，到能够追踪一条 Guest 指令从解码、翻译到执行的完整路径。在大模型的辅助下，原本因该花费大量时间看源码，了解系统结构的过程得到了很大的简化。可以迅速的通过llm来追踪指令在qemu中的完成整条路径，为自己的理解提供了很大的帮助。理清了整条链路，实验测题就成了一个填空题，轻松了很多。整体来说收获满满。
