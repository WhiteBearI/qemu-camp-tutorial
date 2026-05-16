# Qemu GPU 方向

!!! note "主要贡献者"

    - 作者：[@caspian](https://github.com/trace1729)

---

> 背景介绍 & 专业阶段 见 CPU 方向

## 引入

本实验中 GPGPU 是一个挂载在 PCIe 总线上的外部设备。GPGPU 内部分为 互联前端 (gpgpu.c) 和 SIMT 执行后端 (gpgpu_core.c) 两部分。

要如何理解 gpgpu 设备工作的机制呢？根据在cpu blog中提到的方法，拆解分析一个复杂系统通常分为

> 理解一个复杂系统的方式：
>
  > - 第一，理解每个子模块各自的功能；
  > - 第二，理解子模块之间如何交互；
  > - 第三，从更大的视角理解系统整体“做什么”

从第一步到第三步是 Bottom-Up 视角，在此我觉得 Top-Down 视角会更合理，也就是先看系统整体上做什么，再看子系统之间是如何交互的。

## Top-Down 视角分析

### 整体框架

> 首先需要对系统建立一个整体框架，之后再根据这个框架进行拓展。

GPGPU 与 CPU 的协同的整体框架可以用三点概括

1. CPU 配置 GPU，并将数据和程序搬入 GPU 中
2. GPU 根据写入的配置执行程序（线程并行）
3. CPU 把结果读回来

### 子系统的功能 & 交互

#### 一、CPU 配置 GPU，并将数据和程序搬入 GPU 中

##### CPU 配置 GPU

要看懂如何配置 GPU，就要理解 GPU 的设备模型

```
PCIe 总线
   │
   └── gpgpu
        │
        ├── BAR0: 控制寄存器 (1MB MMIO)
        │    ├── 设备信息 (DEV_ID, VERSION)
        │    ├── 全局控制/状态 (GLOBAL_CTRL, GLOBAL_STATUS)
        │    ├── 中断 (IRQ_ENABLE, IRQ_STATUS, IRQ_ACK)
        │    ├── 内核配置 (KERNEL_ADDR, GRID_DIM, BLOCK_DIM, DISPATCH)
        │    ├── DMA 引擎 (DMA_SRC, DMA_DST, DMA_SIZE, DMA_CTRL)
        │    ├── SIMT 上下文 (THREAD_ID, BLOCK_ID, WARP_ID, LANE_ID)
        │    ├── 同步 (BARRIER, THREAD_MASK)
        │    ├── MSI-X Table (偏移 0xFE000)
        │    └── MSI-X PBA (偏移 0xFF000)
        │
        ├── BAR2: VRAM (64MB, 可配置)
        │    ├── kernel 代码 (偏移 0x0000)
        │    ├── 输入数据
        │    └── 输出结果 (偏移 0x1000)
        │
        └── BAR4: Doorbell (64KB, stub)
```

所有的内部状态都存储在 `GPGPUState`

```c
struct GPGPUState {
    PCIDevice parent_obj;       // PCI 设备基类（继承）
    MemoryRegion ctrl_mmio;     // BAR0 对应的 MemoryRegion
    MemoryRegion vram;          // BAR2 对应的 MemoryRegion
    MemoryRegion doorbell_mmio; // BAR4 对应的 MemoryRegion

    ...

    GPGPUSIMTContext simt;      // SIMT 执行上下文（当前线程身份）
};
```

这段设备模型的注册在 `gpgpu_realize` 中完成：

```c
static void gpgpu_realize(PCIDevice *pdev, Error **errp)
{
    GPGPUState *s = GPGPU(pdev);

    /* 分配 VRAM 后端存储 */
    s->vram_ptr = g_malloc0(s->vram_size);         // ← DRAM 物理存储

    /* 注册三块 BAR 空间 */
    memory_region_init_io(&s->ctrl_mmio, ...);      // ← BAR0: 1MB MMIO
    pci_register_bar(pdev, 0, ..., &s->ctrl_mmio);

    memory_region_init_io(&s->vram, ...);           // ← BAR2: VRAM
    pci_register_bar(pdev, 2, ..., &s->vram);

    memory_region_init_io(&s->doorbell_mmio, ...);  // ← BAR4: Doorbell
    pci_register_bar(pdev, 4, ..., &s->doorbell_mmio);

    /* 注册 MSI-X 中断能力 */
    msix_init(pdev, 4, &s->ctrl_mmio, 0, 0xFE000,
                         &s->ctrl_mmio, 0, 0xFF000, ...);
    msi_init(pdev, 0, 1, true, false, ...);

    s->global_status = GPGPU_STATUS_READY;          // ← 初始状态
}
```

##### 数据和程序搬入 GPU 中

```
1. 使能设备
   qpci_io_writel(bar0, GPGPU_REG_GLOBAL_CTRL, ENABLE)
     → gpgpu_ctrl_write(addr=0x0100, val=ENABLE)
       → s->global_ctrl = ENABLE

2. 上传 kernel 代码到 VRAM
   qpci_io_writel(bar2, offset=0x0000, inst[0])   // csrrs
   qpci_io_writel(bar2, offset=0x0004, inst[1])   // andi
   ...
     → gpgpu_vram_write(s, addr, val)
       → memcpy(s->vram_ptr + addr, &val, 4)

3. 初始化输出区域
   qpci_io_writel(bar2, offset=0x1000, 0xDEADBEEF)
     → gpgpu_vram_write(...)

4. 配置内核参数
   qpci_io_writel(bar0, KERNEL_ADDR_LO, 0x0000)   // kernel 在 VRAM 偏移
   qpci_io_writel(bar0, GRID_DIM_X, 1)             // 1 block in X
   qpci_io_writel(bar0, BLOCK_DIM_X, 8)            // 8 threads per block
   ...
     → gpgpu_ctrl_write 将值写入 s->kernel.*
```

这些 `qpci_io_writel` 走的是 QEMU qtest 协议，QEMU 端的处理路径：

```
qtest 协议 ("writel addr val")
  → address_space_write(first_cpu->as, addr, ...)  // ← 走 guest CPU 地址空间
    → flatview_write(fv, addr, ...)
      → pci_register_bar → 找到目标设备
        → memory_region_dispatch_write(MemoryRegion, offset, ...)
          → gpgpu_ctrl_write(s, offset, val)       // ← 最终调用设备回调
            → s->xxx = val                          // ← 更新设备状态
```

#### 二、GPU 根据写入的配置执行程序（线程并行）

这一步是 GPU 的核心。CPU 写 `DISPATCH` 寄存器（偏移 0x0330）触发：

```c
case GPGPU_REG_DISPATCH:
    s->global_status = GPGPU_STATUS_BUSY;          // 设 BUSY
    gpgpu_core_exec_kernel(s);                     // 执行 kernel
    s->global_status = GPGPU_STATUS_READY;         // 恢复 READY

    if (s->irq_enable & GPGPU_IRQ_KERNEL_DONE) {
        s->irq_status |= GPGPU_IRQ_KERNEL_DONE;
        msix_notify(PCI_DEVICE(s), VEC_KERNEL);    // 发中断
    }
    break;
```

##### **SIMT 编程模型**

GPU 采用 SIMT（Single Instruction, Multiple Threads）编程模型。与 CPU 的标量执行不同，GPU 把计算组织为**三级层次**：

```
Grid (三维网格)
  │  grid_dim = (X, Y, Z)  ← X×Y×Z 个 Block
  │
  ├── Block[0,0,0]
  │   │  block_dim = (X, Y, Z)  ← X×Y×Z 个线程
  │   │
  │   ├── Warp 0: 线程 0..31  (锁步执行)
  │   │   ├── Lane 0 (线程 0)
  │   │   ├── Lane 1 (线程 1)
  │   │   └── ...
  │   │
  │   ├── Warp 1: 线程 32..63
  │   └── ...
  │
  ├── Block[0,1,0]
  └── ...
```

**每个 Lane 拿到唯一的 `mhartid`——CPU 和 GPU 的核⼼区别就在这：**

```c
// 一个 CPU 程序：同一个代码段在多个 CPU 核上跑同样的数据
for (int i = 0; i < 8; i++) {
    C[i] = i;   // i 由 外 部循环控制
}

// 一个 GPU kernel：同一个代码段在数千个线程上跑，每个线程的 ID 就是自己的索引
// thread 0: mhartid=0 → C[0] = 0
// thread 1: mhartid=1 → C[1] = 1
// ...
// thread 7: mhartid=7 → C[7] = 7
```

mhartid 的编码：

```
mhartid = (block_id_linear << 13) | (warp_id << 5) | lane_id

例子 Grid(1,1,1) × Block(8,1,1):

Block(0,0,0): block_id_linear=0
  Warp 0 (warp_id=0):
    Lane 0: mhartid = 0 → csrrs → t1=0 → C[0]=0
    Lane 1: mhartid = 1 → csrrs → t1=1 → C[1]=1
    ...
    Lane 7: mhartid = 7 → csrrs → t1=7 → C[7]=7
```

##### **Core 执行**

**SIMT 后端**（`gpgpu_core.c`）实现了从 Grid→Block→Warp→Lane 的三级展开和每一条 RV32I/RV32F 指令的解释执行。

入口是 `gpgpu_core_exec_kernel`：

```c
int gpgpu_core_exec_kernel(GPGPUState *s)
{
    for (int bx = 0; bx < s->kernel.grid_dim[0]; bx++)     // ← 第一层: Grid
    for (int by = 0; by < s->kernel.grid_dim[1]; by++)
    for (int bz = 0; bz < s->kernel.grid_dim[2]; bz++) {
        uint32_t block_id[3] = {bx, by, bz};
        for (int w = 0; w < num_warps; w++) {               // ← 第二层: Block→Warp
            int num_threads = MIN(32, threads_per_block - w*32);
            gpgpu_core_init_warp  // ← 初始化 Warp
            gpgpu_core_exec_warp  // ← 执行 Warp
        }
    }
    return 0;
}
```

`gpgpu_core_init_warp` 设置每个 Lane 的初始状态：

```c
void gpgpu_core_init_warp(GPGPUWarp *warp, uint32_t pc,
                          uint32_t thread_id_base, const uint32_t block_id[3],
                          uint32_t num_threads,
                          uint32_t warp_id, uint32_t block_id_linear)
{
    memset(warp, 0, sizeof(*warp));
    warp->warp_id = warp_id;
    warp->block_id[0] = block_id[0]; ...

    for (int i = 0; i < 32; i++) {
        warp->lanes[i].active = (i < num_threads);  // mask 超出的线程
        warp->lanes[i].pc = pc;                      // 所有线程从同一处开始
        warp->lanes[i].mhartid = MHARTID_ENCODE(
            block_id_linear, warp_id, i);             // 唯一 ID
    }
    warp->active_mask = (1u << num_threads) - 1;
}
```

`gpgpu_core_exec_warp` 是核心执行循环——Warp 内 32 个 Lane 锁步执行：

```
Warp 执行流
┌──────────────────────────────────────────────┐
│                                               │
│  for each cycle:                              │
│                                               │
│  取指: inst = vram_readl(s, lanes[0].pc)     │
│       ↑ 从 VRAM 的 pc 偏移取指令               │
│                                               │
│  译码: opcode = inst & 0x7F                   │
│        funct3 = (inst >> 12) & 0x7            │
│        rs1 = (inst >> 15) & 0x1F              │
│        ...                                    │
│                                               │
│  执行 (for i in 0..31 if active):             │
│    ┌─────────────────┐ ┌─────────────────┐   │
│    │ Lane 0          │ │ Lane 1          │   │
│    │ gpr[32]         │ │ gpr[32]         │   │
│    │ fpr[32]         │ │ fpr[32]         │   │
│    │ mhartid = 0     │ │ mhartid = 1     │   │
│    │ pc = 0x0018     │ │ pc = 0x0018     │   │
│    │ active = 1      │ │ active = 1      │   │
│    └────────┬────────┘ └────────┬────────┘   │
│             │                   │            │
│    ┌────────▼───────────────────▼────────┐   │
│    │  同一条指令 (csrrs t0,mhartid,x0)    │   │
│    │                                     │   │
│    │ Lane 0: csrrs → mhartid=0 → t0=0    │   │
│    │ Lane 1: csrrs → mhartid=1 → t0=1    │   │
│    │ Lane 7: csrrs → mhartid=7 → t0=7    │   │
│    └─────────────────────────────────────┘   │
│                                               │
│  访存: for lane in active:                    │
│    sw rs2, offset(rs1)                        │
│      → vram_writel(s, rs1+offset, rs2)        │
│      → 每个 lane 写到 VRAM 的不同位置         │
│                                               │
│  PC += 4                                      │
│  until ebreak                                  │
└──────────────────────────────────────────────┘
```

每条 Lane 的 `gpr[0]` 在每周期结束时被清零——确保 RISC-V 的 `x0 = 0` 规约：

```c
lane->gpr[0] = 0;  // x0 永远是 0
```

##### **浮点格式转换**

在实验 9-10 要进行浮点格式转换，需要理解浮点数的表示方法。

同样可以通过 TD 的方式来拆解。从熟悉的科学计数法开始，再逐步深入

根据科学计数法，任意实数都可以写成：

```
+/- 1.xxxxx × 10^N
```

底数换成 2 同样成立

```
6(十进制) = 110(二进制) = 1.10 × 2²
                           ↑↑↑↑↑
                         有效数字: 1.10
                          底数: 2
                          指数: 2
```

所以科学计数法的核心就是有效数字，底数，指数

在计算机编程中底数默认是 2, 不表示，将有效数字的绝对值和符号位拆开，也就得到了三要素：

符号(Sign)，指数(Exponent), 有效数字(Mantissa)。

在 ieee754 中单精度浮点的 bit 表示为

```
单精度浮点 = 1 sign bit | 8 exponent bits | 23 mantissa bits

   S    EEEEEEEE    MMMMMMMMMMMMMMMMMMMMMMM
 bit31 bits30..23  bits22..0
```

```
值 = (-1)^S × 2^(E - bias) × 1.M
```

<table>
<tr>
<td>部分<br/></td><td>含义<br/></td><td>float32<br/></td></tr>
<tr>
<td>(-1)^S<br/></td><td>符号位<br/></td><td>1 bit<br/></td></tr>
<tr>
<td>E - bias<br/></td><td>无偏指数，bias=127<br/></td><td>8 bit<br/></td></tr>
<tr>
<td>1.M<br/></td><td>隐含的前导 1 + 小数位<br/></td><td>23 bit mantissa<br/></td></tr>
</table>

E 的分类：

一个 8-bit E 有 256 个可能值 (0~255)，被分成三段：

```
E = 0       │    E = 1 ~ 254     │    E = 255
  ────────────┼────────────────────┼────────────
  零 & 次正规  │    正规数 (含隐含1)  │   Inf & NaN
```

**为什么要引入 bias？**

用补码的话，负数指数和正数指数的**二进制大小比较是反直觉的**。加上 bias=127 后：

```
负指数: -1 → -1+127=126  (0b01111110)
零指数:  0 →   0+127=127  (0b01111111)
正指数:  1 →   1+127=128  (0b10000000)
```

指数按 unsigned integer 比较就能得到正确的大小顺序——这是硬件加速比较的关键设计。

**为什么隐含 1？**

因为规范化二进制数的第一位永远是 `1`：

```
1.10 × 2²  → "1.10" 的前导位永远是 1
1.00 × 2⁵  → "1.00" 的前导位永远是 1
```

与其浪费 bit 去存已知的值，不如省下来多存一位尾数。23 位尾数 + 隐含 1 = 24 位有效精度。

**为什么需要次正规数？**

把 E=0 预留出来是为了**填平 0 到最小正规数之间的空缺**：

```
关掉次正规（E 和 E-1 之间有空隙）:

    0 ───────────────────────── 1.0 × 2⁻¹²⁶
                                   ↑
                             什么值都表示不了——从 0 到这里是空的
```

```
打开次正规（空隙被均匀填满）:

    0 ──────────────────────┐
                    ── 0.000...001 × 2⁻¹²⁶ (最小次正规 ≈ 2⁻¹⁴⁹)
                    ── 0.000...01  × 2⁻¹²⁶
                    ── 0.000...1   × 2⁻¹²⁶
                    ── ...
                    ── 0.1        × 2⁻¹²⁶
    1.0 × 2⁻¹²⁶ ───────────┘
```

没有次正规数，任何比 `2⁻¹²⁶` 小的值直接变成 `0`。这在数值计算中会导致：

- 两个非常接近的数相减，结果不是精确的微小差值，而是 **0**
- 累计误差骤增，甚至除零异常

次正规数让下溢到 0 的过程变成了**渐变**而不是**突变**。

e4m3, e5m2 也就是在指数和有效数字的位数上以及一些特殊值的表示上有区别，理解这个心智模式之后，编码就水到渠成。
