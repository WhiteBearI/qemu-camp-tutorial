# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"

    - 作者：[@dazuo233](https://github.com/dazuo233)

---

## 背景介绍

一枚网络空间安全专业研究生，目前在做固件模糊测试方向，动态测试依赖 QEMU 仿真和外设建模，希望通过参加训练营来更加了解 QEMU

**开发环境**

+ OS：Ubuntu 24.01.1 LTS
+ AI：opencode + glm5.1

## 专业阶段

主要完成 SOC 方向设备建模的实现

### 1 实验分析

10 道 测试用例

测试位置位于 tests/gevico/qtest/，第一个测试test-board-g233已经通过

| #    | 测试名称                  | 子测试数 |
| ---- | ------------------------- | -------- |
| 1    | test-board-g233           | 4        |
| 2    | test-gpio-basic           | 4        |
| 3    | test-gpio-int             | 5        |
| 4    | test-pwm-basic            | 7        |
| 5    | test-wdt-timeout          | 7        |
| 6    | test-spi-jedec            | 3        |
| 7    | test-flash-read           | 4        |
| 8    | test-flash-read-interrupt | 4        |
| 9    | test-spi-cs               | 6        |
| 10   | test-spi-overrun          | 2        |

实验目标：在 QEMU 仿真框架中，为虚拟 RISC-V 开发板 G233 完成 **GPIO、PWM、WDT、SPI 控制器与 SPI Flash** 五类外设的设备建模

+ GPIO 控制器测试（地址 0x10012000，PLIC IRQ 2）
+ PWM 控制器测试（地址 0x10015000，PLIC IRQ 3）
+ WDT 看门狗测试（地址 0x10010000，PLIC IRQ 4）
+ SPI 控制器测试（地址 0x10018000，PLIC IRQ 5）

###  2 学习 QEMU 框架

#### QOM

QEMU Object Model (QOM) 是 QEMU 的设备抽象框架。每个设备模型遵循统一模式：

```
TypeInfo
  ├── .name          → 设备类型名字符串（如 "g233-gpio"）
  ├── .parent        → 父类型（通常为 TYPE_SYS_BUS_DEVICE）
  ├── .instance_size → 状态结构体大小
  ├── .instance_init → 实例初始化（注册 MMIO 区域、IRQ）
  ├── .class_init    → 类初始化（绑定 realize/reset/vmsd）
  └── type_init()    → 注册 TypeInfo

DeviceClass
  ├── dc->realize   → 设备实现（初始化 ptimer 等）
  ├── dc->reset     → 硬件复位
  └── dc->vmsd      → VMState（用于迁移/快照）
```

**核心数据流**：Guest 软件通过 MMIO 读写操作设备 → `MemoryRegionOps.read/write` 回调 → 更新设备状态 → 通过 `qemu_set_irq()` 驱动中断控制器。

#### ptimer

QEMU 的 `ptimer` 是对底层 `QEMUTimer` 的高级封装，用于模拟硬件计数器：

- `ptimer_init(callback, opaque, policy)` — 创建定时器
- `ptimer_set_period(timer, ns)` — 设置每个 tick 的纳秒数
- `ptimer_set_limit(timer, count, reload)` — 设置计数上限并可选重载
- `ptimer_run(timer, oneshot)` — 启动递减计数（0=连续模式）
- `ptimer_stop(timer)` — 停止计数
- `ptimer_get_count(timer)` — 读取当前计数值

**关键原理**：在 qtest 模式下，虚拟时钟由 `qtest_clock_step(ns)` 手动推进。ptimer 内部基于 `QEMU_CLOCK_VIRTUAL`，当时钟推进时，已过期的定时器回调被触发。

本实验使用 **1μs/tick（1MHz）** 的计数频率，确保：

- WDT LOAD=0x10 时，16μs 即超时（测试步进 500ms 远大于此）
- WDT LOAD=0xFFFF 时，10ms 内计数器可观察到递减
- PWM PERIOD=0xFFFF 时，1ms 内 CNT 可观察到递增

### 3 学习 G233 的外设交互

#### **SysBusDevice**

G233 的所有外设均为系统总线设备，挂载在 CPU 的物理地址空间上。每个设备需要：

1. 在 `instance_init` 中调用 `memory_region_init_io()` 创建 MMIO 区域
2. 调用 `sysbus_init_mmio()` 将区域注册到系统总线
3. 调用 `sysbus_init_irq()` 注册中断输出引脚
4. 板级代码通过 `sysbus_create_simple(base, irq)` 实例化并映射

#### **PLIC** 

G233 使用 SiFive PLIC 作为外部中断控制器。关键特性：

- **Level-triggered**：外设通过 `qemu_set_irq(irq, 1/0)` 驱动电平变化
- **Pending 是 sticky 的**：中断源电平降低后，PLIC pending 位不会被自动清除，必须通过 claim/complete 流程清除
- 外设需要维护自己的中断状态，在条件满足时拉高 IRQ，条件消失时拉低 IRQ

各外设 PLIC 中断号分配：

| 外设 | IRQ  | 说明                    |
| ---- | ---- | ----------------------- |
| GPIO | 2    | 所有引脚中断状态的 OR   |
| PWM  | 3    | 所有通道 DONE 标志的 OR |
| WDT  | 4    | 超时中断                |
| SPI  | 5    | TXE/RXNE/OVERRUN 中断   |

#### SSI 总线

QEMU 的 SSI 总线用于连接 SPI 主控和从设备：

- **主控（Master）**：通过 `ssi_create_bus()` 创建总线，调用 `ssi_transfer()` 发送/接收数据
- **从设备（Peripheral）**：继承 `TYPE_SSI_PERIPHERAL`，实现 `transfer()` 回调处理每个字节
- **CS（片选）**：通过名为 `SSI_GPIO_CS` 的 GPIO 线控制，低电平有效（`SSI_CS_LOW`）

数据流：主控写 DR → `ssi_transfer(bus, data)` → 所有从设备的 `transfer_raw` 被调用 → 仅 CS 激活的从设备通过 `transfer()` 返回响应 → 结果 OR 后返回给主控。

### 4 实现

新增建模文件

主要将硬件手册里面的内容实现为代码

~~~
hw/gevico/
├── g233_gpio.c          (215 行)  GPIO 控制器
├── g233_wdt.c           (230 行)  看门狗定时器
├── g233_pwm.c           (248 行)  PWM 控制器
├── g233_spi.c           (212 行)  SPI 控制器
├── meson.build          (6 行)    编译规则
└── Kconfig              (6 行)    设备依赖配置

include/hw/gevico/
└── g233_periph.h        (12 行)   设备类型声明
~~~

修改配置文件

| 文件                      | 修改内容                                        |
| ------------------------- | ----------------------------------------------- |
| `hw/meson.build`          | 添加 `subdir('gevico')`                         |
| `hw/Kconfig`              | 添加 `source gevico/Kconfig`                    |
| `hw/riscv/Kconfig`        | GEVICO_G233 增加 `select GEVICO_G233_PERIPH`    |
| `include/hw/riscv/g233.h` | 添加 4 个内存映射枚举值 + 4 个 IRQ 常量         |
| `hw/riscv/g233.c`         | 添加设备 include + memmap 条目 + 外设实例化代码 |

## 总结

**QOM 建模流程**

~~~
→ type_init(g233_gpio_register_types)
  → type_register_static(&g233_gpio_info)
    → g233_gpio_class_init
         → dc->realize = g233_gpio_realize
         → dc->reset = g233_gpio_reset  
         → dc->vmsd = vmstate_g233_gpio
~~~

**设备实例化流程**

~~~
sysbus_create_simple(TYPE_G233_GPIO, 0x10012000, plic_irq(GPIO_IRQ));
~~~

**感受**

该实验几乎全部基于 VibeCoding 完成，当前我们对 MCU 固件的测试运行主要通过 qemu 仿真指令集，外设的仿真通过其他方式随机的提供，我们认为这样可以减少全系统仿真的手动建模开销。然而，这对于一个读过对应硬件手册的 agent 来说或许并不是难事。