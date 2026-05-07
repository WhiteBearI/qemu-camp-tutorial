

# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"

    - 作者：[@Kcang-gna](https://github.com/Kcang-gna)

以下是专业阶段实验的学习过程总结。

---

## 背景介绍

计算机科学大三学生，希望通过 Qemu 训练营提升一下技术水平

## 专业阶段

Rust(FFI) 方向

## 学习概述

第一个实验比较简单就不说了，后两个实验的呢，可以用一个心智模型来理解：就是 mmio -> 控制器 -> 设备

客户端通过读写寄存器控制控制器从而控制设备

## 学习内容
### 配置修改

在 G233 开发板的配置项中，添加反向依赖，确保启用 G233 时，自动启用 Rust I2C GPIO 设备：

```kconfig
config GEVICO_G233
    bool
    default y
    depends on RISCV
    # 新增:启用 Rust I2C GPIO 设备（依赖 HAVE_RUST 环境）
    select X_I2C_GPIO_RUST if HAVE_RUST
```



定义 Rust I2C GPIO 设备的配置项，用于被 C 侧 board 选中：

```kconfig
config X_I2C_GPIO_RUST
    bool
```

配置完成后，QEMU 编译时会根据 `HAVE_RUST` ,自动包含 Rust I2C GPIO 相关代码。

### C 侧开发

C 侧的核心作用是：提供 Rust 函数的调用接口、将 I2C GPIO 设备挂载到 G233 开发板的内存映射中、注册设备树节点，让 QEMU 与 Guest OS 能够识别该设备。

在 `include/hw/gpio/i2c_gpio.h` 中，声明设备类型、结构体及 Rust 暴露的函数接口，确保 C 侧能够调用 Rust 实现的设备创建函数：

```c
#ifndef HW_I2C_GPIO_H
#define HW_I2C_GPIO_H

#include "hw/core/sysbus.h"
#include "qom/object.h"

// 设备类型定义
#define TYPE_I2C_GPIO "i2c_gpio"

// QOM 类型声明
OBJECT_DECLARE_SIMPLE_TYPE(I2CGPIOState, TYPE_I2C_GPIO);

// Rust 实现的设备创建函数，C 侧调用该函数挂载设备
DeviceState *i2c_gpio_create(hwaddr addr, qemu_irq irq);

#endif
```

#### 挂载设备到 G233 开发板

G233 开发板的设备挂载逻辑位于 `hw/riscv/g233.c`,核心是将 I2C GPIO 设备添加到内存映射数组，再调用 Rust 函数创建设备。

#### 添加内存映射

G233 板的 `virt_memmap` 数组定义了所有挂载设备的内存地址与大小，新增 I2C GPIO 设备的映射：

```c
// g233.c 中的内存映射数组
static const MemMapEntry virt_memmap[] = {
    // 其他设备映射...
    [VIRT_I2C_GPIO] =     { 0x10013000,          0x14 },  // 新增:I2C GPIO 设备，地址 0x10013000，大小 0x14
};

// g233.h 中添加对应的枚举
typedef enum {
    // ...
    VIRT_I2C_GPIO,  // 新增:I2C GPIO 设备枚举
} VirtG233Memmap;
```

#### 调用 Rust FFI 创建设备

在 G233 板的初始化函数（如 `virt_machine_init`）中，调用 Rust 暴露的 `i2c_gpio_create` 函数，将设备挂载到指定内存地址：

```c
// 在 virt_machine_init 或对应初始化函数中添加
i2c_gpio_create(s->memmap[VIRT_I2C_GPIO].base, NULL);
```

#### 设备树注册

Guest OS（如 Linux）需要通过设备树识别设备，因此需在 C 侧编写设备树节点创建函数，将 I2C GPIO 设备的信息写入设备树：

```c
static void create_fdt_i2c_gpio(RISCVG233State *s) {
    g_autofree char *name = NULL;
    MachineState *ms = MACHINE(s);
    name = g_strdup_printf("/soc/i2c_gpio@%"HWADDR_PRIx, s->memmap[VIRT_I2C_GPIO].base);
    qemu_fdt_add_subnode(ms->fdt, name);  // 添加子节点
    qemu_fdt_setprop_string(ms->fdt, name, "compatible", "kcang,i2c_gpio"); 
    qemu_fdt_setprop_sized_cells(ms->fdt, name, "reg", 2, s->memmap[VIRT_I2C_GPIO].base, 1, s->memmap[VIRT_I2C_GPIO].size);
    qemu_fdt_setprop_cell(ms->fdt, name, "#address-cells", 1);  // 地址单元格数量
    qemu_fdt_setprop_cell(ms->fdt, name, "#size-cells", 0);     // 大小单元格数量
    qemu_fdt_setprop(ms->fdt, name, "i2c-controller", NULL, 0); // 标识为 I2C 控制器
}
```

最后，在 G233 板的设备树初始化函数`finalize_fdt`中调用上述函数，确保设备树节点被正确添加。

### Rust 侧开发

Rust 侧负责实现 I2C GPIO 设备的核心逻辑：寄存器定义、QOM 类型注册、内存区域初始化、读写回调函数，以及设备初始化与暴露给 C 侧的接口。

#### 寄存器实现

```rust
use bilge::prelude::*;

/// 寄存器偏移量
#[repr(u64)]
#[derive(Debug, Eq, PartialEq, common::TryInto)]
pub enum RegisterOffset {
    /// 控制寄存器（读操作，控制 I2C 行为）
    CTRL = 0x00,
    /// 状态寄存器（写操作，展示 I2C 状态）
    STATUS = 0x04,
    /// 地址寄存器（读操作，展示 I2C 从设备地址）
    ADDR = 0x08,
    /// 数据寄存器（读写操作，传输 I2C 数据）
    DATA = 0x0C,
    /// 预分频寄存器（读写操作，设置 I2C 时钟预分频）
    PRESCALE = 0x10,
}

#[derive(Clone, Copy, Default)]
pub struct Registers {
    ctrl: Control,      // 控制寄存器（0x00）
    status: Status,     // 状态寄存器（0x04）
    addr: u8,           // 地址寄存器（0x08）
    data: u8,           // 数据寄存器（0x0C）
    prescale: u8,       // 预分频寄存器（0x10）
}

impl Registers {
    /// 读寄存器：根据偏移量返回对应寄存器的值（u8）
    pub fn read(&self, offset: RegisterOffset) -> u8 {
        use RegisterOffset::*;
        match offset {
            CTRL => u8::from(self.ctrl),
            STATUS => u8::from(self.status),
            ADDR => self.addr,
            DATA => self.data,
            PRESCALE => self.prescale,
        }
    }

    /// 写寄存器：根据偏移量设置对应寄存器的值
    pub fn write(&mut self, offset: RegisterOffset, value: u8) {
        use RegisterOffset::*;
        match offset {
            CTRL => self.ctrl = Control::from(value),
            STATUS => self.status = Status::from(value),
            ADDR => self.addr = value,
            DATA => self.data = value,
            PRESCALE => self.prescale = value,
        };
    }
}

/// 控制寄存器（8 位）
#[bitsize(8)]
#[derive(Clone, Copy, Default, DebugBits, FromBits)]
struct Control {
    pub en: bool,        // 使能位（1:启用 I2C,0:禁用）
    pub start: bool,     // 起始位（1:触发 I2C 起始条件）
    pub stop: bool,      // 停止位（1:触发 I2C 停止条件）
    pub wr: bool,        // 读写控制位（0:写，1:读）
    _reserved: u4,       // 保留位（4 位，未使用）
}

/// 状态寄存器（8 位）
#[bitsize(8)]
#[derive(Clone, Copy, Default, DebugBits, FromBits)]
struct Status {
    pub busy: bool,      // 忙状态位（1:I2C 正在传输，0:空闲）
    pub ack: bool,       // 应答位（1:收到应答，0:未收到应答）
    pub done: bool,      // 完成位（1:传输完成，0:传输中）
    _reserved: u5,       // 保留位（5 位，未使用）
}
```

#### 定义 QOM 对象

导入 QEMU Rust 绑定相关的 crate，定义 I2C GPIO 设备的 QOM 结构体：

```rust
mod registers;
...
pub const TYPE_I2C_GPIO: &CStr = c"i2c_gpio";

/// I2C GPIO 设备 QOM 结构体
#[repr(C)]
#[derive(qom::Object)] // 这个宏的功能与 C 中的类型注册相同
struct I2CGPIOState {
    parent: ParentField<SysBusDevice>,  // 父类
    pub mmio: MemoryRegion,          
    pub regs: BqlRefCell<Registers>,   // 寄存器集合
}

qom_isa!(I2CGPIOState: SysBusDevice, DeviceState, Object);
```

#### 实现 QOM 相关 Trait

QEMU 的 Rust 绑定通过 Trait 实现 QOM 类型的注册与初始化：

```rust
// 实现 ObjectType:定义 QOM 类相关信息
unsafe impl ObjectType for I2CGPIOState {
    type Class = SysBusDeviceClass;  // 父类 Class
    const TYPE_NAME: &'static CStr = crate::TYPE_I2C_GPIO;  // 设备类型名
}

// 实现 ObjectImpl:设备初始化相关逻辑
impl ObjectImpl for I2CGPIOState {
    type ParentType = SysBusDevice;  // 父类类型
    const CLASS_INIT: fn(&mut Self::Class) = Self::Class::class_init::<Self>;  // 类初始化函数

    // 实例初始化函数：初始化设备自身成员
    const INSTANCE_INIT: Option<unsafe fn(ParentInit<Self>)> = Some(Self::init);

    // 实例后初始化函数：完成内存区域暴露等后续操作
    const INSTANCE_POST_INIT: Option<fn(&Self)> = Some(Self::post_init);
}

// 实现必要 Trait
unsafe impl DevicePropertiesImpl for I2CGPIOState {}
impl ResettablePhasesImpl for I2CGPIOState {}
impl DeviceImpl for I2CGPIOState {}
impl SysBusDeviceImpl for I2CGPIOState {}
```

#### 设备初始化

设备可以初始化分为两步：`init`初始化内存区域与寄存器，`post_init`暴露内存区域给 board，让 board 能够将其映射到 Guest 内存。

```rust
impl I2CGPIOState {
    /// 实例初始化：初始化 mmio 内存区域与寄存器
    unsafe fn init(mut this: ParentInit<Self>) {
        // 1. 定义内存区域操作
        static I2CGPIO_OPS: MemoryRegionOps<I2CGPIOState> =
            MemoryRegionOpsBuilder::<I2CGPIOState>::new()
                .read(&I2CGPIOState::read)  // 读回调
                .write(&I2CGPIOState::write)  // 写回调
                .little_endian()  // 小端模式
                .impl_sizes(4, 4)  // 读写字节范围 4~4 字节
                .build();

        // 2. 初始化 mmio 内存区域
        MemoryRegion::init_io(
            &mut uninit_field_mut!(*this, mmio),
            &I2CGPIO_OPS,
            "i2c_gpio",
            0x14,
        );

        // 3. 初始化寄存器（默认值）
        uninit_field_mut!(*this, regs).write(Default::default());
    }

    /// 后初始化：将 mmio 内存区域暴露给 board
    fn post_init(&self) {
        self.init_mmio(&self.mmio);
    }

    // 暴露内存区域给 board
    //fn init_mmio(&self, iomem: &MemoryRegion) {
    //    assert!(bql::is_locked());
    //    unsafe {
    //        system_sys::sysbus_init_mmio(self.upcast().as_mut_ptr(), iomem.as_mut_ptr());
    //    }
    }
}
```

#### 内存区域读写回调

QEMU 通过内存区域的读写回调函数，处理 Guest 对设备寄存器的读写操作。

```rust
impl I2CGPIOState {
    /// 读回调：处理 Guest 对寄存器的读操作
    fn read(&self, addr: hwaddr, _size: u32) -> u64 {
        // 转换地址为寄存器偏移量
        let offset = RegisterOffset::try_from(addr).unwrap();
        // 读取对应寄存器的值，转换为 u64 返回
        self.regs.borrow().read(offset).into()
    }

    /// 写回调：处理 Guest 对寄存器的写操作
    fn write(&self, addr: hwaddr, data: u64, _size: u32) {
        // 转换地址为寄存器偏移量
        let offset = RegisterOffset::try_from(addr).unwrap();
        self.regs
            .borrow_mut()
            .write(offset, u8::try_from(data).unwrap());
    }
}
```

#### 暴露 C 接口

通过`no_mangle`修饰函数，确保函数名不被 Rust 编译器混淆，让 C 侧能够正确调用该函数创建设备：

```rust
/// 暴露给 C 侧的设备创建函数
#[no_mangle]
pub unsafe extern "C" fn i2c_gpio_create(addr: hwaddr, _irq: *mut IRQState) -> *mut DeviceState {
    // 创建 I2C GPIO 设备实例
    let dev = I2CGPIOState::new();
    // 初始化系统总线设备
    dev.sysbus_realize().unwrap_fatal();
    // 将 mmio 内存区域映射到指定地址
    dev.mmio_map(0, addr);
    // 返回设备指针
    dev.as_mut_ptr()
}
```

剩下的就是 TDD 了

#### 调试方法

对于复杂问题，可通过 GDB 调试 QTest 测试用例，查看寄存器读写过程与设备状态：

```bash
# 运行 QTest 测试并启用 GDB 
meson test -C build "path/to/test/file"  --print-errorlogs --verbose --gdb
```

```bash
sudo gdb -p pidOfQemu
```

调试技巧：

- 在 qemu 侧时可以在相应函数上添加断点后，然后执行命令`continue`.
- 在 qtest 侧，如果执行的内容会触发 qemu 中的断点，则可以到 qemu 侧进行 gdb

# 总结

学习 Rust 有一段时间了，但一直都是做的都是玩具级项目，这一次借助 QEMU 训练营，将 Rust 与 QEMU 设备开发相结合，既巩固了 Rust 编程能力，也深入理解了 QEMU 设备虚拟化的核心逻辑，收获满满。
