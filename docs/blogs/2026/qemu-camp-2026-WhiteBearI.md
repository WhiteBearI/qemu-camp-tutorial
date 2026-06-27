# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"

    - 作者：[@WhiteBearI](https://github.com/WhiteBearI)

---

## 背景介绍

之前学习计算机体系结构的时候仅停留在书本的理论层面，了解的也很浅薄。后来对 Agent 开发比较感兴趣，想看看底层支撑模型推理的 GPU 是怎么工作的，因此选择了 GPU 方向想进一步学习 GPU 的工作原理。刚好看到 QEMU 训练营的招募，提供了一个从源码层面动手实践的机会，就报名参加了。

---

## 专业阶段

选择的是 GPU 方向实验，任务是在 QEMU 中实现一个教育用途的 GPGPU 设备模型，通过 QTest 框架的 17 个测试用例验证正确性。实验覆盖了设备识别、全局控制、VRAM 访问、DMA 传输、中断模拟、SIMT 上下文调度、kernel 执行、低精度浮点格式转换等内容。

### 设备注册与 PCI 设备建模

GPGPU 作为一个 PCIe 设备，首先需要通过 QOM（QEMU Object Model）将设备类型注册到 QEMU 的类型系统中。QOM 是 QEMU 用 C 实现的面向对象框架，设备类型遵循 `Object → Device → PCI_DEVICE → GPGPU` 的继承链：

```c
static const TypeInfo gpgpu_info = {
    .name          = TYPE_GPGPU,
    .parent        = TYPE_PCI_DEVICE,
    .instance_size = sizeof(GPGPUState),
    .class_init    = gpgpu_class_init,
};

static void gpgpu_register_types(void)
{
    type_register_static(&gpgpu_info);
}

type_init(gpgpu_register_types)
```

命令行指定 `-device gpgpu` 时，QEMU 会调用 `gpgpu_realize` 完成硬件资源分配——注册 BAR 空间、分配 VRAM、初始化 MSI-X。GPGPU 设备使用了三个 BAR：

- **BAR0**：控制寄存器，1MB MMIO 空间，包含设备 ID、全局控制、中断、内核分发、DMA、SIMT 上下文等寄存器
- **BAR2**：显存（VRAM），默认 64MB，用于存放内核代码和数据
- **BAR4**：doorbell，64KB MMIO 空间

Guest CPU 读写 BAR 地址时，QEMU 通过 `MemoryRegionOps` 自动调用对应的回调函数。这种方式将地址空间的读写转化为对设备状态的修改，是理解 QEMU 设备模型的核心：**地址空间本质上是一张路由表，MemoryRegion 定义的是访存往哪走，而不是存储怎么分**。

### 寄存器设计与 MMIO 回调

控制寄存器的设计是分层的，从功能上可以划分为：

| 寄存器组 | 偏移范围 | 功能 |
|---------|---------|------|
| 设备信息 | 0x0000~0x0010 | 设备 ID、版本、能力位、VRAM 大小 |
| 全局控制 | 0x0100~0x0108 | 使能、复位、状态、错误 |
| 内核分发 | 0x0300~0x0330 | 内核地址、Grid/Block 维度、调度触发 |
| DMA | 0x0400~0x0418 | 源/目标地址、传输大小、控制 |
| 中断 | 0x0200~0x0208 | 使能、状态、确认 |
| SIMT 上下文 | 0x1000~0x1024 | 线程 ID、Block ID、Warp/Lane ID |
| 同步 | 0x2000~0x2004 | 屏障、线程掩码 |

测试用例通过 QTest 的 `qpci_io_readl/writel` 直接读写这些寄存器，验证设备的行为。例如测试 1 读取设备 ID 寄存器，期望返回 `0x47505055`（ASCII "GPPU"）；测试 3 写入使能位后读回控制寄存器，验证设备正确响应。

### VRAM 访问与 DMA 传输

VRAM 通过 BAR2 暴露给 Host 侧。测试 5 直接往 VRAM 写入测试模式（如 `0xDEADBEEF`）并读回验证。在内核执行场景中，VRAM 同时承担两个角色：

- **代码存储**：内核指令从地址 0x0000 开始存放
- **数据存储**：输出数组从地址 0x1000 开始存放

DMA 寄存器允许配置源地址、目标地址和传输大小，实现 Host 内存和 VRAM 之间的数据搬运。测试 6 验证了 DMA 寄存器的读写正确性。

### SIMT 线程模型

GPGPU 采用 SIMT（Single Instruction, Multiple Threads）执行模型。线程按 Grid → Block → Warp → Lane 四级层次组织：

- **Grid**：整个内核的线程网格，三维
- **Block**：Grid 内的线程块，三维
- **Warp**：Block 内的线程束，32 个 Lane 锁步执行
- **Lane**：最小的执行单元，对应一个线程

每个线程通过 `mhartid` CSR 获取自己的身份编号，编码格式为 `[31:13] block_id | [12:5] warp_id | [4:0] lane_id`，相当于 CUDA 的 `threadIdx + blockIdx` 被编码进一个硬件寄存器。

测试 8~12 验证了 SIMT 上下文寄存器的读写和软复位行为。特别是测试 12 验证了写入 `GPGPU_CTRL_RESET` 后，所有 SIMT 上下文（thread_id、block_id、warp_id、thread_mask）都被清零。

### 内核执行与 RISC-V 指令解释器

这是整个实验最核心的部分。当 Host 写入 `GPGPU_REG_DISPATCH` 寄存器时，设备开始执行内核。

执行入口 `gpgpu_core_exec_kernel` 遍历所有 Block 和 Thread，对每个线程构造一个 `GPGPULane` 结构体，设置好 `mhartid`，然后调用 `gpgpu_exec_lane` 让这个线程跑一遍内核代码。

`gpgpu_exec_lane` 是一个迷你的 RISC-V 指令解释器，工作流程如下：

1. **初始化上下文**：通用寄存器清零（x0 天生为 0，不可写）、浮点寄存器清零、PC 指向内核起始地址、浮点舍入模式设为默认
2. **取指循环**：从 VRAM 当前 PC 位置读取 32 位指令，最多执行 100 万条指令（防止死循环）
3. **解码执行**：通过宏提取 opcode、rd、rs1、rs2、funct3、funct7 等字段，按 opcode 分派到不同的处理逻辑

```c
#define OPCODE(insn)    ((insn) & 0x7f)
#define RD(insn)        (((insn) >> 7) & 0x1f)
#define RS1(insn)       (((insn) >> 15) & 0x1f)
#define RS2(insn)       (((insn) >> 20) & 0x1f)
#define FUNCT3(insn)    (((insn) >> 12) & 7)
#define FUNCT7(insn)    (((insn) >> 25) & 0x7f)
#define IMM_I(insn)     ((int32_t)(insn) >> 20)
```

这些宏对应的是 RISC-V 指令编码的固定字段位置，是硬件规范决定的，不可更改。解释器支持的指令包括：

- **R 型**（opcode 0x33）：add、sub、and、or、xor、sll、srl、sra
- **I 型**（opcode 0x13）：addi、slli、andi、ori、xori、srli、srai、slti、sltiu
- **Load**（opcode 0x03）：lb、lh、lw
- **Store**（opcode 0x23）：sb、sh、sw
- **LUI**（opcode 0x37）：加载高位立即数
- **分支**（opcode 0x63）：beq、bne、blt、bge、bltu、bgeu
- **跳转**（opcode 0x6f/0x67）：jal、jalr
- **系统**（opcode 0x73）：csrrs（读 mhartid）、ebreak（停止）
- **浮点**（opcode 0x53/0x07/0x27）：浮点运算、加载、存储

其中浮点指令由子函数 `gpgpu_exec_fp_insn` 处理，支持 RV32F 基本运算（fadd.s、fsub.s、fmul.s、fdiv.s）以及自定义的 BF16、E4M3、E5M2、E2M1 低精度格式转换指令。

以测试 14 的浮点内核为例，内核功能是 `output[tid] = (int)(tid * 2.0 + 1.0)`。每个线程执行以下步骤：

```c
csrrs  x6, mhartid, x0    // 读取线程编号
andi   x6, x6, 0x1F       // 提取 lane_id
fcvt.s.w f1, x6            // 整数转浮点
fmul.s f4, f1, f2          // tid * 2.0
fadd.s f5, f4, f3          // tid * 2.0 + 1.0
fcvt.w.s x7, f5, RTZ       // 浮点转整数（向零取整）
slli   x8, x6, 2           // 计算字节偏移
lui    x28, 1              // 输出基地址 0x1000
add    x28, x28, x8        // 最终地址
sw     x7, 0(x28)           // 写入结果
ebreak                     // 停止
```

8 个线程并行执行，输出 `[1, 3, 5, 7, 9, 11, 13, 15]`。

### 低精度浮点格式转换

测试 15~17 验证了 BF16、E4M3、E5M2、E2M1 四种低精度浮点格式的往返转换。这些格式在 AI 推理场景中非常常见，用更少的比特数表示浮点数，牺牲精度换取吞吐量。

以 E2M1 为例，仅 4 bit（1 位符号 + 2 位指数 + 1 位尾数），能表示 8 个正数值：0、0.5、1.0、1.5、2.0、3.0、4.0、6.0。转换超出最大值的输入会被饱和到 6.0，NaN/Inf 同样饱和处理。

测试 17 特别验证了边界情况：零值转换、溢出饱和（100 → E2M1 max 6）、Inf 饱和（+Inf → E4M3 max 448）。

### 中断模拟

设备执行完成后通过 MSI-X 通知 Host。MSI-X 是 PCIe 的消息信号中断机制，设备通过写特定的内存地址来触发中断，不需要额外的中断线。测试 7 验证了中断使能和状态寄存器的读写。

---

## 总结

通过专业阶段的学习让我更好的懂得了 QEMU 模拟设备的流程，也进一步了解了 GPU 的一系列寄存器和工作方式。通过阅读 kernel 部分的代码也让我进一步理解了 RISC-V 指令集。

具体来说有以下几方面的收获：

**QEMU 设备建模全链路**：从 QOM 类型注册、BAR 空间映射、MMIO 读写回调到 MSI-X 中断通知，走通了一个 PCI 设备从被发现到完成计算的完整路径。以前看 QEMU 源码总觉得庞杂无从下手，现在有了一条清晰的理解线索。

**GPU 架构与 SIMT 执行模型**：从寄存器级别理解了 GPU 的工作方式——线程如何组织（Grid/Block/Warp/Lane）、如何通过 mhartid 获取身份、如何锁步执行同一条指令但产生不同结果。这和上层框架（CUDA、OpenCL）的抽象形成了对照。

**RISC-V 指令集实践**：通过阅读和调试内核代码，对 RISC-V 的指令编码、字段布局、寻址模式有了具体认识。特别是 x0 零寄存器的设计、I 型立即数的符号扩展、LUI 构造大常数等细节，光看书很容易忽略，实际读代码就理解了。

**低精度浮点格式**：了解了 BF16、E4M3、E5M2、E2M1 等格式在 AI 推理中的意义，以及饱和处理的基本策略。这些知识在后续学习模型量化时会很有用。

对后续学员的建议：如果之前没有接触过 QEMU 源码，建议先花时间理解 QOM 和 MemoryRegion 这两个核心概念，后面的设备建模都是在这个框架上展开的。另外，RISC-V 指令集的手册（尤其是 RV32I 和 RV32F 部分）值得对照着内核代码一起读，理解会深很多。
