# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"

    - 作者：[@First-da](https://github.com/First-da)

---

## 背景介绍

电子信息研一，目前在学习计算机操作系统相关的知识和课程，未来想往嵌入式Linux方向发展，偶然的机会了解到qemu2026训练营，抱着学习的心态参加了2026训练营。希望借此机会，提升自己开发实际项目的能力，掌握一些CPU、SOC以及模拟器的底层原理。

## 专业阶段

由于第一次接触到qemu和如此大型的项目仓库，只做了cpu和soc两个专业阶段的基础实验。

### CPU 实验

CPU 实验的任务是在 QEMU 的 RISC-V 前端中实现 Xg233ai 自定义指令扩展。该扩展包含 10 条 R-type 指令，统一使用 `custom-3` 编码空间，opcode 为 `0x7B`。具体实现流程主要包括三部分：

1. 根据硬件手册中规定的 opcode、funct3、funct7 以及 `rd`、`rs1`、`rs2` 字段，在 `target/riscv/insn32.decode` 中添加对应的指令译码规则，使 QEMU 能够识别这些自定义指令。

2. 在 RISC-V 指令翻译文件中补充对应的 `trans_xxx()` 函数。`trans` 函数负责取得客户机寄存器中的参数，并生成相应的 TCG IR。对于较简单的运算，可以直接调用 TCG 提供的接口；对于矩阵运算、排序、向量处理等较复杂的指令，则通过 Helper 函数实现。

3. 使用 Helper 时，需要先在 `helper.h` 中声明函数，再在对应的 C 文件中实现 `helper_xxx()`，最后在 `trans_xxx()` 中通过 `gen_helper_xxx()` 生成 Helper 调用。需要注意的是，`trans_xxx()` 和 `gen_helper_xxx()` 工作在翻译阶段，真正的 `helper_xxx()` 则在翻译后的代码运行时执行。

### SOC实验

SoC 实验主要围绕 G233 主板及其外设建模展开。实验一已经提供了主板的基础框架和启动流程，后续九个实验则重点实现 GPIO、PWM、WDT 和 SPI 等外设的寄存器行为、中断、定时器以及总线连接。

#### 2.1 QOM 设备模型

QEMU 通过 QOM（QEMU Object Model）在 C 语言中实现对象、继承和类型注册机制。外设通常沿着 `Object → DeviceState → SysBusDevice → 具体设备` 的层次进行扩展。

设备类型通过 `TypeInfo` 描述，并使用 `type_init()` 完成注册。其中，`instance_init` 负责初始化设备实例，例如创建 MMIO 区域、IRQ 输出或外设总线；`class_init` 用于设置设备类级别的回调函数，如 `reset`。设备被主板代码创建并 realize 后，才真正进入可用状态。

#### 2.2 SoC 外设接入流程

在 G233 SoC 实验中，每个外设都需要完成设备模型和主板连接两部分。

设备模型中需要定义状态结构体、寄存器读写回调、复位逻辑以及 `MemoryRegionOps`，并在初始化阶段注册 MMIO 区域和 IRQ 输出。主板代码则负责创建设备、完成 realize、将 MMIO 区域映射到指定物理地址，并把设备中断连接到 PLIC。

整体流程可以概括为：

```text
定义设备类型
→ 初始化 MMIO 和 IRQ
→ 在主板中创建设备
→ 映射寄存器地址
→ 连接中断或 GPIO
→ 通过 qtest 验证寄存器行为
```

SPI 实验还进一步涉及 SSI 总线和片选信号，需要将 Flash 设备挂载到 SSI 总线上，并通过 GPIO 信号连接 SPI 控制器与 Flash 的 CS 引脚。

#### 2.3 MMIO 与地址空间

QEMU 使用 `MemoryRegion` 描述内存、RAM 和外设寄存器区域。外设通过 `memory_region_init_io()` 创建 MMIO 区域，并绑定对应的读写回调函数。

主板初始化时，再使用 `memory_region_add_subregion()` 将该区域加入系统地址空间。客户机访问相应物理地址时，QEMU 会根据地址找到对应的 `MemoryRegion`，最终调用外设实现的 `read` 或 `write` 函数。

外设寄存器访问链路可以表示为：

```text
客户机访问物理地址
→ QEMU 地址空间查找
→ 命中外设 MemoryRegion
→ 调用 MMIO read/write
→ 更新设备内部状态
```

通过这一过程，外设状态结构体中的软件变量被映射为客户机可见的硬件寄存器。

## 总结

对我来说是第一次接触并使用QEMU，通过 CPU 和 SoC 实验，我对 QEMU 的指令翻译、设备建模、MMIO 和中断连接有了一些初步认识，但对 QEMU 整体架构和内部运行机制仍缺乏系统理解。后续还需要继续阅读源码，并结合调试工具跟踪设备创建和运行流程，才能逐步加深认识。