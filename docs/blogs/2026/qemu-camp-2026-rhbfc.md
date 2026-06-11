# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"

    - 作者：[rhbfc](https://github.com/rhbfc)

---

## 背景介绍

我是一名工作 4 年的软件开发人员，主要负责实时操作系统和 Type1 虚拟机的开发，之前通过阅读 qemu 源码解决了几个 rtos 和 hypervisor 的 bug，萌生了学习 qemu 的念头，正好趁此训练营深入理解 QEMU。

## 专业阶段

选择的方向是 SOC 建模

### 1. 完成实验的流程

先根据测试逐项完成外设的建模，以最小改动方式在 g233.c 的 virt_machine_init 中添加设备，最后根据硬件手册的寄存器定义修改整个 g233.c 文件。

### 2. GPIO

#### 1. 先从 GPIO 开始分析每个外设建模通用的部分

##### 1：TypeInfo

QEMU 维护一个**全局设备类型注册表**。设备必须注册进去，在 g233.c 才能用名字创建它。

注册流程在 g233_gpio.c 文件末尾写一个 `TypeInfo` 结构体，然后 `type_init()` 让它在 QEMU 启动时自动执行：

```c
// TypeInfo
static const TypeInfo g233_gpio_info = {
    .name           = TYPE_G233_GPIO,             // 字符串名，qdev_new 的参数
    .parent         = TYPE_SYS_BUS_DEVICE, 
    .instance_size  = sizeof(G233GPIOState),       // 给每个实例分配的内存
    .instance_init  = g233_gpio_init,              // 对象创建初始化
    .class_init     = g233_gpio_class_init,        // 类初始化
};

// 注册函数：把 TypeInfo 塞进全局注册表
static void g233_gpio_register_types(void)
{
    type_register_static(&g233_gpio_info);
}

// QEMU 启动时自动调用注册函数
type_init(g233_gpio_register_types)
```

`class_init` 设备类型的初始化，只执行一次。在里面挂回调函数：

```c
static void g233_gpio_class_init(ObjectClass *klass, const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->realize = g233_gpio_realize;                // 实例化回调
    device_class_set_legacy_reset(dc, g233_gpio_reset); // 硬件复位回调
    dc->vmsd    = &vmstate_g233_gpio;               // 迁移/快照支持
}
```

##### 2：realize / instance_init / reset / SysBusDevice 这些具体的初始化

`instance_init` 设备声明自己有什么资源，对于 gpio 来说就是有一块 mmio 空间和一条中断线

```c
static void g233_gpio_init(Object *obj)
{
    G233GPIOState *s = G233_GPIO(obj);

    memory_region_init_io(&s->mmio, obj, &g233_gpio_ops, s,
                          TYPE_G233_GPIO, 0x100); //创建 MMIO 区域
    sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->mmio);//将 MMIO 区域注册到系统总线
    sysbus_init_irq(SYS_BUS_DEVICE(obj), &s->irq); //将一条中断线注册到系统总线
}
```

`realize` 设备正式激活时调用，应该放里面的内容有：
    需要 qdev_get_prop_* 获取用户传入属性的逻辑
    连接子设备或后端（chardev、blockdev 等）
    分配依赖构造参数的内存
GPIO 没有这个需求，留空即可，后面其他外设建模会用到

`reset` 每次硬件复位时调用，把所有寄存器和状态设置成复位值，根据《硬件手册》，GPIO 复位值都是 0：

```c
static void g233_gpio_reset(DeviceState *dev)
{
    G233GPIOState *s = G233_GPIO(dev);
    memset(s->regs, 0, sizeof(s->regs));
}
```

`SysBusDevice` 是 QEMU 中所有挂在 SoC 地址空间上的外设基类。

```c
struct G233GPIOState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;         // MMIO 区域（CPU 访问寄存器通过这个空间）
    qemu_irq irq;              // 中断输出线（连接 PLIC）
    uint32_t regs[7];          // 7 个 32-bit 寄存器
};
```

SysBusDevice 提供四个关键 API：

| API                      | 调用位置      | 作用                              |
| ------------------------ | ------------- | --------------------------------- |
| `sysbus_init_mmio()`   | 外设的 instance_init | 「通知 qemu 有一个 MMIO 区域」    |
| `sysbus_init_irq()`    | 外设的 instance_init | 「通知 qemu 有一条中断线」        |
| `sysbus_mmio_map()`    | g233.c        | 「把 MMIO 区域映射到某个区域」 |
| `sysbus_connect_irq()` | g233.c        | 「把 IRQ 连到 PLIC 的某个中断引脚」  |

##### 3：MemoryRegion + read/write 回调 —— "CPU 访问地址时调用的函数"

真实硬件中，CPU 通过地址总线读写外设。QEMU 用 MemoryRegion 模拟：

```
CPU 执行 store 0x10012004, 0xFF
    ↓
QEMU 查找谁注册了 0x10012000 这个地址范围
    ↓
找到 G233GPIOState.mmio（来自 instance_init 中的 memory_region_init_io）
    ↓
调用 MemoryRegionOps 中的 .write = g233_gpio_write
    ↓
g233_gpio_write(opaque, offset, value, size)
    ↓
在 write 函数里更新对应的寄存器
```

```c
// 步骤 1：定义 ops —— "读调这个函数，写调那个函数"
static const MemoryRegionOps g233_gpio_ops = {
    .read = g233_gpio_read,
    .write = g233_gpio_write,
    .endianness = DEVICE_NATIVE_ENDIAN,
    .impl.min_access_size = 4,   // 最小访问宽度 4 字节
    .impl.max_access_size = 4,   // 最大访问宽度 4 字节
};

// 步骤 2：在 instance_init 创建 MemoryRegion 并绑定 ops
memory_region_init_io(&s->mmio, obj, &g233_gpio_ops, s,
                      TYPE_G233_GPIO, 0x100);

// 步骤 3：在 instance_init 中注册到 SysBus
sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->mmio);
```

**read/write**：

```c
static uint64_t g233_gpio_read(void *opaque, hwaddr offset, unsigned int size)
{
}

static void g233_gpio_write(void *opaque, hwaddr offset,
                             uint64_t value, unsigned int size)
{
}
```

#### 2. 根据硬件手册和测试用例实现 GPIO 建模

g233_gpio.c 中其他代码都是外设建模的通用操作，read、write 需要根据硬件手册和测试用例实现

先在 g233.c 的 virt_machine_init 函数中添加 gpio 这个设备，地址和中断号写死，未来再修改

```c
    DeviceState *gpio_dev = qdev_new("g233-gpio");
    sysbus_realize_and_unref(SYS_BUS_DEVICE(gpio_dev), &error_fatal);
    sysbus_mmio_map(SYS_BUS_DEVICE(gpio_dev), 0, 0x10012000);// gpio 基地址
    sysbus_connect_irq(SYS_BUS_DEVICE(gpio_dev), 0,
                       qdev_get_gpio_in(mmio_irqchip, 2));// gpio 中断号
```

`read` 根据手册直接返回对应寄存器

```c
static uint64_t g233_gpio_read(void *opaque, hwaddr offset, unsigned int size)
{
    G233GPIOState *s = G233_GPIO(opaque);
    uint32_t idx = offset / 4;   // 寄存器编号（4 字节对齐）
    return s->regs[idx];
}
```

`write`
先看 test-gpio-basic：需要注意的是 GPIO_IN，该寄存器会受到 GPIO_DIR 和 GPIO_OUT 值的影响。

```c
static void update_state(G233GPIOState *s)
{
    uint32_t dir = s->regs[G233_GPIO_DIR];
    uint32_t out = s->regs[G233_GPIO_OUT];

    out &= dir;
    s->regs[G233_GPIO_OUT] = out;
    // 根据 GPIO_DIR 和 GPIO_OUT 更新 GPIO_IN
    uint32_t in = s->regs[G233_GPIO_IN];
    in &= ~dir;
    in |= (out & dir);
    s->regs[G233_GPIO_IN] = in;
}

static void g233_gpio_write(void *opaque, hwaddr offset,
                             uint64_t value, unsigned int size)
{
    G233GPIOState *s = G233_GPIO(opaque);
    uint32_t idx = offset / 4;

    switch (idx) {
    case G233_GPIO_DIR:
        s->regs[G233_GPIO_DIR] = value;
        break;

    case G233_GPIO_OUT:
        s->regs[G233_GPIO_OUT] = value;
        break;

    case G233_GPIO_IN:
        /* 只读寄存器 */
        break;
    }
    update_state(s);
}

```

然后是 test-gpio-int：修改 GPIO_OUT 时，需要根据 GPIO_TRIG 和 GPIO_POL 触发中断并更新 GPIO_IS 寄存器。需要注意的是硬件手册似乎写反了，和测试用例对不上

```c
// 可以在 write 函数最后面调用
static void g233_gpio_update_irq(G233GPIOState *s)
{
    uint32_t is = s->regs[G233_GPIO_IS];
    uint32_t ie = s->regs[G233_GPIO_IE];
    qemu_set_irq(s->irq, (is & ie) ? 1 : 0);// is 和 ie 都为 1，拉高中断线发出中断
}
```

边缘中断，需要根据 GPIO_OUT 当前值和上次值确定是否触发

```c
    uint32_t old_out = s->regs[G233_GPIO_OUT];
    s->regs[G233_GPIO_OUT] = value;

    uint32_t changed = old_out ^ value;
    // 处理每个 GPIO 的边缘中断
    for (int i = 0; i < 32; i++) {
        if (!(ie & (1u << i))) {
            continue;
        }
        if (trig & (1u << i)) {
            continue;
        }
        if (!(changed & (1u << i))) {
            continue;
        }
        if (((value >> i) & 1) == ((pol >> i) & 1)) {
            s->regs[G233_GPIO_IS] |= (1u << i);
        }
    }

    // 处理每个 GPIO 的电平中断
    for (int i = 0; i < 32; i++) {
        if (!(ie & (1u << i))) {
            s->regs[G233_GPIO_IS] &= ~(1u << i);
            continue;
        }
        if (!(trig & (1u << i))) {
            continue;
        }
        if (((in >> i) & 1) == ((pol >> i) & 1)) {
            s->regs[G233_GPIO_IS] |= (1u << i);
        } else {
            s->regs[G233_GPIO_IS] &= ~(1u << i);
        }
    }

```

### 3. PWM 与 WDT

这两个外设比较相似，相比 gpio 多了 timer 相关 API

timer_new_ns：设备实例化时创建定时器，并且设置回调函数
timer_mod：启动定时器，这个定时器是单次模式，每次到时间都要重新设置
timer_del：停止定时器，清除 timer_mod 所设置的未来时间，并不会真的删除定时器
timer_pending：防重入，确保只有未 pending 时才 timer_mod

`timer_new_ns` 应当在 `realize` 中调用，这个是 gpio 建模时留空的函数

```c
static void g233_pwm_realize(DeviceState *dev, Error **errp)
{
    G233PWMState *s = G233_PWM(dev);
    s->timer = timer_new_ns(QEMU_CLOCK_VIRTUAL, g233_pwm_timer_cb, s);
}
```

reset 时要停止定时器

```c
static void g233_pwm_reset(DeviceState *dev)
{
    G233PWMState *s = G233_PWM(dev);
    s->glb = 0;
    memset(s->ch, 0, sizeof(s->ch));
    timer_del(s->timer);
}
```

每次回调函数中对计数寄存器做递增 (pwm) 或是递减 (wdt),然后设置下次的时间

```c
static void g233_pwm_update_timer(G233PWMState *s)
{
    if (s->glb & (PWM_GLB_CH_EN(0) | PWM_GLB_CH_EN(1) |
                  PWM_GLB_CH_EN(2) | PWM_GLB_CH_EN(3))) {
        if (!timer_pending(s->timer)) {
            timer_mod(s->timer,
                      qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL) + NANOSECONDS_PER_SECOND / G233_PWM_FREQ_HZ);
        }
    } else {
        timer_del(s->timer);
    }
}

static void g233_pwm_timer_cb(void *opaque)
{
    G233PWMState *s = G233_PWM(opaque);

    for (int i = 0; i < G233_PWM_CHANNELS; i++) {
        if (!(s->glb & PWM_GLB_CH_EN(i))) {
            continue;
        }

        s->ch[i].cnt++;

        if (s->ch[i].period > 0 && s->ch[i].cnt >= s->ch[i].period) {
            s->ch[i].cnt = 0;
            s->glb |= PWM_GLB_CH_DONE(i);
        }
    }

    g233_pwm_update_timer(s);
}

```

### 4. SPI

#### 1. CS 片选

此前每次发送中断都是调用`void qemu_set_irq(qemu_irq irq, int level)`函数。按我的理解，这个函数和 irq 并没有必然的联系，实际是对一根线或是一个引脚的拉高拉低的功能。能发送中断是拉高了中断线，通知到了 plic 中断控制器。spi 测试中，通过`qemu_set_irq` 拉低 cs 线通知到 spi 从设备。

#### 2. SSI 总线

SPI 控制器不直接操作 Flash，而是通过 QEMU 的 SSI 框架：

```
G233SPIState (控制器, master)
    │
    └── SSIBus (spi 总线)
          │
          └── w25x16 / w25x32 (Flash 芯片, slave, 在 g233.c 中创建 CS 线)
```

主要用到以下几个 API：

| 函数 | 作用 | 调用位置 |
|------|------|---------|
| `ssi_create_bus` | 创建 SSI 总线 | `g233_spi_realize` |
| `ssi_transfer` | 通过总线发送一个字节给从设备，返回收到的字节 | `g233_spi_write` (写 DR 时) |
| `ssi_get_cs` | 按 `cs_index` 查找总线上的设备 | `g233_spi_reset` |

#### 3. qdev 设备创建与接线

- 板级代码：`qdev_prop_set_uint8` 设 Flash 的 `cs_index` 属性 → `qdev_get_child_bus` 拿到 SPI 总线 → `qdev_realize_and_unref` 实现并挂到总线
- 控制器侧：`g233_spi_realize` 中 `qdev_init_gpio_out_named` 创建命名 CS 输出 → `reset` 时 `qdev_get_gpio_in_named` + `qdev_connect_gpio_out_named` 自动接线

| 函数 | 作用 | 调用位置 |
|------|------|---------|
| `qdev_prop_set_uint8` | 设置设备的 uint8 属性值 | `g233.c` 板级 |
| `qdev_get_child_bus` | 获取设备的子总线（按名称查找） | `g233.c` 板级 |
| `qdev_realize_and_unref` | 实现设备 + 释放引用 | `g233.c` 板级 |
| `qdev_init_gpio_out_named` | 创建一组命名 GPIO 输出引脚（CS 线） | `g233_spi_realize` |
| `qdev_get_gpio_in_named` | 获取设备的命名 GPIO 输入引脚 | `g233_spi_reset` |
| `qdev_connect_gpio_out_named` | 连接输出引脚 → 输入引脚（接线） | `g233_spi_reset` |

#### 4. 实现时要留意的地方

`realize` 中需要创建 ssi 总线

```c
static void g233_spi_realize(DeviceState *dev, Error **errp)
{
    G233SPIState *s = G233_SPI(dev);

    s->spi = ssi_create_bus(dev, "spi");
    //暴露 4 条 CS 线给 SSI 总线
    qdev_init_gpio_out_named(DEVICE(dev), s->cs_lines, "cs", G233_SPI_NUM_CS);
}
```

`reset` 接线，放这里是保证 flash realize 后才去接线

```c
static void g233_spi_reset(DeviceState *dev)
{
    G233SPIState *s = G233_SPI(dev);

    memset(s->regs, 0, sizeof(s->regs));
    s->tx_data = 0;
    s->rx_data = 0;
    qemu_set_irq(s->irq, 0);

    for (int i = 0; i < G233_SPI_NUM_CS; i++) {
        // 复位 cs 初始值
        qemu_set_irq(s->cs_lines[i], 1);
        // ssi 遍历总线
        DeviceState *kid = ssi_get_cs(s->spi, i); 
        if (kid) {
            // 获取 flash 暴露出的 cs 线，
            // flash 那边会调用 qdev_init_gpio_in_named 暴露出 in 型的 cs 线，具体看 m25p80.c
            qemu_irq cs_line = qdev_get_gpio_in_named(kid, SSI_GPIO_CS, 0);
            // flash 的 cs 线接到 realize 时所暴露给 SSI 总线的 CS 线上
            qdev_connect_gpio_out_named(DEVICE(s), "cs", i, cs_line);
        }
    }
}
```

在 g233.c 的 virt_machine_init 函数中添加 spi 设备外，还要添加测试需要的 Flash

```c
    DeviceState *spi_dev = qdev_new("g233-spi");
    DeviceState *flash_dev;

    /* SPI controller */
    sysbus_realize_and_unref(SYS_BUS_DEVICE(spi_dev), &error_fatal);
    sysbus_mmio_map(SYS_BUS_DEVICE(spi_dev), 0, 0x10018000);
    sysbus_connect_irq(SYS_BUS_DEVICE(spi_dev), 0,
                       qdev_get_gpio_in(mmio_irqchip, 5));

    /* CS0: W25X16 (2MB) */
    flash_dev = qdev_new("w25x16");
    qdev_prop_set_uint8(flash_dev, "cs", 0);
    qdev_realize_and_unref(flash_dev,
                           qdev_get_child_bus(spi_dev, "spi"),
                           &error_fatal);

    /* CS1: W25X32 (4MB) */
    flash_dev = qdev_new("w25x32");
    qdev_prop_set_uint8(flash_dev, "cs", 1);
    qdev_realize_and_unref(flash_dev,
                           qdev_get_child_bus(spi_dev, "spi"),
                           &error_fatal);
```

## 总结

通过本次实验，理解了 QEMU 设备建模的基本思路。我觉得进行设备建模时，应该先研究测试用例，搞清预期行为后再动手，往往能事半功倍。建模过程中要以硬件的角度去思考——就像发中断是拉高一根信号线——而不是把它当成一个软件驱动来写。
