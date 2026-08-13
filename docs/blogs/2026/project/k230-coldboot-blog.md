# K230 冷启动全链路：启动链、镜像格式与排障

!!! note "主要贡献者"

    - 作者：[@flamboyante](https://github.com/flamboyante)
    - 技术细节基于实验记录整理，或有疏漏，欢迎指正。

K230 的冷启动链路已经在 QEMU 上跑通:SDK 一条 `make CONF=...` 命令产出 32MiB SPI NOR 镜像，镜像可以直接启动到 Linux shell。本文记录这条链路的完整过程:QEMU 侧补了什么、SDK 改了哪些文件、镜像是什么格式，以及踩过的坑。当前阶段的目标是先把 SPI NOR 这条冷启动链跑通;SD 卡启动也在计划内，这次的链路设计和改动为此留了位置。

## 1. QEMU 验证分支：来源与组成

QEMU 主线的 K230 启动路径只有 `k230_direct_boot()` 和 `k230_firmware_boot()` 两条，都会直接将固件加载到内存，不经过 Flash。`QSPI0/QSPI1/SPI/FLASH` 在主线 `k230.c` 中仍是 `unimplemented` 占位，因此主线 K230 无法从 SPI NOR 镜像启动。

为验证 SPI NOR 冷启动，我在官方主线 merge commit `b428fe0362`（2026-07-31）之上建立了本地组合分支 `k230-coldboot-validation`。它不是一套新的上游 patch 系列，也不是所有 K230 模块的最新版本集合；它只是为冷启动验证收集并组合的一组提交。

这些外设提交来自我在 2026-08-04 对 `lore.kernel.org/qemu-devel` 的公开检索和整理。这个日期只是来源快照：当时收集到的版本和评审状态不代表当前最新状态，也不应以后的 patch 版本反推本验证分支的组成。每组提交的公开来源直接列在下表中。

| 归档模块 | 本分支选取的模块                    | 作者            | 分支提交数 | 当时 lore 来源                                                                                                    |
| ---: | --------------------------- | ------------- | ----: | ------------------------------------------------------------------------------------------------------------- |
|    2 | DesignWare SSI Standard PIO | Kangjie Huang |     5 | [v2 0/5](https://lore.kernel.org/qemu-devel/20260801192848.30606-1-flamboyant.h.01@gmail.com/T/)              |
|    3 | DW APB Timer                | raoyi         |     3 | [v2 0/3](https://lore.kernel.org/qemu-devel/20260730133713.3253-1-rao232328@gmail.com/T/)                     |
|    5 | GSDMA + Decomp Gzip         | Tao Ding      |     7 | [v2 0/7](https://lore.kernel.org/qemu-devel/20260727155702.36484-1-dingtao0430@163.com/T/)                    |
|    7 | DW 8250 UART                | WX Chen       |     3 | [RESEND v2 0/3](https://lore.kernel.org/qemu-devel/20260725-feat-k230-uart-v2-v2-0-d5fe82c47c28@gmail.com/T/) |
|    8 | SRAM                        | Jian Cai      |     1 | [v2 单补丁](https://lore.kernel.org/qemu-devel/20260721060409.9688-2-lingqian_gi@163.com/T/)                     |
|    9 | RMU                         | Jack Wang     |     2 | [v2 0/2](https://lore.kernel.org/qemu-devel/20260719180247.8660-1-163wangjack@gmail.com/T/)                   |
|   10 | DDR Controller + PHY        | Junze Cao     |     3 | [v1 0/3](https://lore.kernel.org/qemu-devel/20260716132423.931427-1-caojunze424@gmail.com/T/)                 |
|   11 | IOMUX                       | Kangjie Huang |     2 | [v2 0/3](https://lore.kernel.org/qemu-devel/cover.1784022244.git.flamboyant.h.01@gmail.com/T/)                |

表中的 26 个提交来自这些当时收集的公开系列。IOMUX 只取了设备模型和板级接线，未将归档系列中的 qtest 提交单独纳入分支。其余两个增量提交分别是整合 qtest 列表的修正，以及新增冷启动 loader；

这个分支的用途是固定一次冷启动验证的源码组合，而不是维护“最新 K230 外设合集”。复现应以 `k230-coldboot-validation` 的具体 HEAD 和对应 bundle 为准。

## 2. BootROM 没有公开规范

真机的启动链路：

```
CPU reset → BootROM → SPL(在 SRAM)→ U-Boot(在 DDR)→ OpenSBI → Linux
```

BootROM 是芯片出厂烧在 ROM 里的一段代码，负责从启动介质读第一级镜像。K230 的 BootROM 支持多种启动介质 (SPI NOR、SD 卡、NAND 等),但具体指令序列、读介质用的协议细节、字节交换算法都没有公开文档，QEMU 无法模拟一段没有规范的代码。本次先做 SPI NOR 介质，SD 卡是后续目标。

主线的两个 boot 函数都是把现成固件喂进内存，冷启动需要的是从 Flash 读镜像再启动——中间缺的正是 BootROM 这一环。所以我在分支上补了一个本地 commit(`7730876ca5 feat: full-boot and ctrl0 reg`),加了 `k230_coldboot_boot()`:host 侧直接读 Flash 偏移 0 的内容，按 BootROM 的方式还原字节交换，检查 K230 magic，拆 528 字节的 firmware header，把 SPL 写入 `0x80300000`,再设 reset vector 让 CPU 跳过去：

```c
static void k230_coldboot_boot(K230MachineState *s, MachineState *machine)
{
    DriveInfo *dinfo = drive_get(IF_MTD, 0, 0);
    BlockBackend *blk;
    hwaddr spl_base = 0x80300000;         /* CONFIG_SPL_TEXT_BASE */
    const size_t spl_part_size = 0x80000; /* CONFIG_SPL_MAX_SIZE */
    const size_t fw_head_size = 528;      /* magic+length+type+verify */
    ...
    /* BootROM reads 32-bit words byte-swapped from flash; undo it. */
    for (i = 0; i + 4 <= len; i += 4) { ... }

    if (raw[0] != 0x4b || raw[1] != 0x32 || raw[2] != 0x33 || raw[3] != 0x30) {
        error_report("k230: bad K230 magic at SPI flash offset 0");
        exit(EXIT_FAILURE);
    }
    ...
    address_space_write(&address_space_memory, spl_base, ...,
                        raw + fw_head_size + version_size, spl_len);
    riscv_setup_rom_reset_vec(machine, &s->soc.c908_cpu, spl_base, ...);
}
```

这个 loader 只是验证启动介质时的 host 侧替代实现，不等同于真实 BootROM。BootROM 没有公开规范，这段代码也不作为上游实现提交，只留在本地验证分支。

分支没有推到公开远端，交付靠增量 bundle(94KB)。没有共享远端时，bundle 是标准方式：体积小、不暴露私有历史，别人 clone 官方仓库、checkout 基线、pull bundle 就能拿到完整分支。

## 3. SDK 只改了 7 个文件

QEMU 侧就绪后看 SDK。官方 SDK 要在 QEMU 上冷启动，只改了 7 个文件，共 +585/-12 行:3 处对官方文件的原地修改，3 个复制官方文件改出的板级派生，1 个顶层 defconfig 派生。公共文件没动 (`k230.dtsi`、公共 dts 原样保留)。

三处修改，每一处对应一个 QEMU 与真机的差异：

**第一处:canmv 板级写死从 SD 卡启动。** 官方 `board.c` 里：

```c
sysctl_boot_mode_e sysctl_boot_get_boot_mode(void)
{
    return SYSCTL_BOOT_SDIO1;
}
```

canmv 真机从 SD 卡启动，所以板级代码覆盖了 SDK 的 weak 实现。QEMU 里当前没有 SD 卡模型，只有 SPI NOR。删掉这个覆盖，回落到底层 weak 实现：它读 boot 区寄存器，而 QEMU 里 boot 区是 unimplemented，读出来全 0,0 对应 NOR 启动。这个改动两边都成立:QEMU 上读 0 = NOR，真机上读 strap 引脚。以后 QEMU 接 SD 卡启动时，同样走这条 weak 路径——让 boot 区返回 SDIO 对应的 strap 值即可，不需要再动板级代码。

**第二处:QEMU 没有 PUFS 硬件。** 官方公共头文件里默认开着 `CONFIG_K230_PUFS`。PUFS 是安全启动用的物理防克隆单元，做镜像哈希校验。QEMU 没有这个硬件，这次验证分支开着就过不去。关掉后，U-Boot 校验镜像哈希走软件 `sha256_csum_wd()` 分支。这个改动只针对当前 QEMU 验证，不代表真机板型都要这样配置。

**第三处:T-HEAD 的私有页表位。** 详见 §7.2。

顶层 defconfig 复制了一份，叫 `k230_canmv_qemu_spinor_defconfig`。改的开关只有 5 个：

```diff
- CONFIG_UBOOT_DEFCONFIG="k230_canmv"
+ CONFIG_UBOOT_DEFCONFIG="k230_canmv_qemu"      # U-Boot 用 QEMU 板配置
+ CONFIG_BUILDROOT_DEFCONFIG="k230d_canmv"      # buildroot 用精简 31 包
- CONFIG_LINUX_DTB="k230_canmv"
+ CONFIG_LINUX_DTB="k230_canmv_qemu"            # Linux DTB 用 QEMU 适配版
+ CONFIG_SPI_NOR=y                               # 官方默认不产 SPI NOR 镜像!
+ CONFIG_SPI_NOR_SUPPORT_CFG_PARAM=y
```

其中 `CONFIG_SPI_NOR=y` 是必须的——官方默认不产 SPI NOR 镜像。

### 三层配置各管一段，不能只改顶层 defconfig

一开始我也容易把这几个名字都当成“QEMU 配置”。实际不是一回事。顶层
`k230_canmv_qemu_spinor_defconfig` 只负责给 SDK 的总 Makefile 选路：U-Boot
用哪个 defconfig，Linux 编哪个 DTB，buildroot 用哪套根文件系统配置。它本身
不会替 U-Boot 或 Linux 描述一颗 SPI Flash。

U-Boot 这一侧有两份派生文件。`k230_canmv_qemu_defconfig` 相对原 canmv 配置
几乎只改了一项：`CONFIG_DEFAULT_DEVICE_TREE` 指到
`k230_canmv_qemu.dts`。后者才是给 SPL 和 U-Boot 的硬件描述：明确启用 SPI0，
挂一个 `jedec,spi-nor` 子节点，并把收发 bus width 限成 1。这不是为了模拟
真实板子全部的 QSPI 能力；当前 QEMU 的这条验证路径只走 Standard 1-1-1。
`u-boot,dm-pre-reloc` 也在这个节点上，SPL 阶段就能拿到 SPI NOR。

Linux 也不能继续沿用原来的 `k230_canmv.dts`。它的 QEMU 派生 DTS 做了四件事：

1. 同样把 SPI NOR 改成 1-1-1；
2. 把 SPI0 的时钟换成 50MHz fixed-clock，并补上九个 `interrupt-names`；
3. 关闭 QEMU 尚未建模的 USB、GPIO、SDHCI、显示和依赖它们的按键节点；
4. 保留 SDK 需要的 Flash 分区骨架，让构建前的脚本按实际镜像配置生成分区。

第二项是后来才补上的。原 DTS 中 SPI0 挂在 K230 的复合时钟上，但 QEMU 没有
完整建模那条 PLL 时钟树，Linux 驱动取得的频率是 0。驱动随后把 SSI 分频值写成
0，控制器按规则不产生 SCLK，SPI NOR probe 就卡住。换成固定 50MHz 不是在给
硬件“造时钟”，而是把这条 QEMU 验证路径所需的可用输入时钟写明确。

所以这三层的关系可以简单记成：顶层 defconfig 选构建组合；U-Boot DTS 让
SPL/U-Boot 能读 NOR；Linux DTS 让内核只去驱动 QEMU 实际提供的那部分硬件。

## 4. 冷启动的链路

| 阶段        | 运行位置                            | 主要职责                           |
| --------- | ------------------------------- | ------------------------------ |
| BootROM   | 片内 ROM(QEMU 里是 `0x91200000` 窗口) | 采样启动模式、选介质、加载 SPL              |
| SPL       | SRAM，入口 `0x80300000`            | pinmux/时钟/复位、DDR 初始化、加载 U-Boot |
| 完整 U-Boot | DDR(`0x08000000` 起)             | 读环境变量、加载 OpenSBI+ 内核            |
| OpenSBI   | DDR                             | SBI 服务，M-mode → S-mode 交接      |
| Linux     | DDR                             | 建页表、初始化驱动、挂 rootfs、起 init      |

需要多级的原因：芯片刚通电时 DDR 还没初始化，只有片内一小块 SRAM 能放代码;BootROM 只有几百行，装不下完整的引导加载器。所以先让最小程序 (SPL) 把 DDR 拉起来，再加载完整的 U-Boot。

## 5. SPI NOR 镜像格式

SDK 产出的 `sysimage-spinor32m.img` 是 32MiB，布局 (目前先做 SPI NOR 变体;SD 卡镜像的格式和装配链类似，后续按需接入):

|        偏移 |         大小 | 分区         | 内容                                                    |
| --------: | ---------: | ---------- | ----------------------------------------------------- |
|  0x000000 |   0x080000 | spl\_boot  | `swap_fn_u-boot-spl.bin`(SPL，字节交换格式)                  |
|  0x080000 |   0x160000 | uboot      | `fn_ug_u-boot.bin`(完整 U-Boot)                         |
|  0x1e0000 |   0x020000 | uboot\_env | environment(含 bootcmd/bootargs)                       |
|  0x200000 | \~0xdc0000 | 产品配置包      | quick\_boot/face\_db/sensor/ai\_mode/speckle/rtt\_app |
|  0xfc0000 |   0x700000 | linux      | `linux_system.bin`(OpenSBI+ 内核+DTB)                    |
| 0x16c0000 |   0x900000 | rootfs     | JFFS2 或 UBI                                           |

两个文件名需要说明：

**`swap_`** **前缀。** SPL 文件名里的 "swap" 指字节交换。SDK 用 `endian-swap.py` 处理过的 SPL 才带这个前缀；本地 loader 按这个产物格式还原字节序。装配时要用 swap 过的版本，不能用原始的。

**`linux_system.bin`** **的打包链：**

```text
OpenSBI fw_payload.bin → k230_gzip → fw_payload.bin.gz
+ 1 字节 rd 占位 + k230.dtb
→ mkimage -T multi -C gzip
→ K230 firmware header(firmware_gen.py)
→ linux_system.bin
```

其中"1 字节 rd 占位"是个坑：`mkimage -T multi` 拒绝真正空的 rd 文件 (报 `Input file rd is empty`)。官方脚本用 1 字节占位。自己复现时 `printf 'a\n' > rd` 即可，Linux 不会把这个占位当 initramfs。

528 字节的 firmware header 不是 PUFS 专属格式。PUFS 是安全启动的额外校验，firmware header 是所有镜像都有的，内容为 magic + length + type + verify。

## 6. 两条装配路线

**路线 A:自制装配。** 官方 `build-image` 无条件调用 RT-Smart 的构建产物 (mkromfs.py、rtthread.\*、big-core opensbi),而 RT-Smart 编译很慢。不想编 RT-Smart，就得绕开官方装配，自己按 genimage-spinor.cfg 的布局逐段拼：

```text
按布局表逐段写入：
  SPL @ 0x000000 → U-Boot @ 0x080000 → env @ 0x1e0000
  → 配置包 (可选)→ linux_system.bin @ 0xfc0000 → rootfs @ 0x16c0000
每段检查:K230 firmware header / JFFS2 magic / 大小不越界
最后补 0xff 到 32MiB
```

rootfs 有两种格式:JFFS2(`mkfs.jffs2`,链路短) 或 UBI/UBIFS(`mkfs.ubifs` + `ubinize`,官方产品常用)。都能放进 9MiB 分区，装配时按 env 里的 `rootfstype` 配套。

自制路线的代价:layout 要自己维护，`sdk_autoconf.h` 里的分区偏移、DTS 分区、genimage cfg 三处要手动对齐——§7.1 的坑就出在这里。

**路线 B:官方一键编译**一条命令走完：

```text
make CONF=k230_canmv_qemu_spinor_defconfig -j1
  → linux → rt-smart → buildroot → uboot → opensbi → build-image(genimage 装配)
```

产物是官方 `sysimage-spinor32m_jffs2.img`(JFFS2 变体) 和 `sysimage-spinor32m.img`(UBI 变体)。镜像布局、分区偏移、DTS 分区全部由官方脚本自动对齐，不需要手动维护。

如果 rootfs 超出 9MiB 分区 (SDK 默认 buildroot 配置装 73 个包),不需要改任何 SDK 脚本，顶层配置一行切换：

```text
CONFIG_BUILDROOT_DEFCONFIG="k230d_canmv"
```

从 73 包的 `k230_evb` 切到 31 包的 `k230d_canmv` 后，rootfs.jffs2 从 12.4MB 降到 5.7MB。

## 7. 排障：四个卡点

### 7.1 mtd9 之谜：官方脚本动态生成分区

症状：早期自制路线手写 12 个 MTD 分区 (rootfs 排第 11,bootargs 写 `mtdblock11`),切到官方流程后镜像挂不上 rootfs，还出现 DTS 分区重复两套。

原因：官方 `menuconfig_to_code.sh` 在 `prepare_memory` 阶段动态重写 Linux DTS 的分区——把 DTS 里 `partition@0` 到 `all_flash` 之间的内容删掉，插入脚本生成的分区：

```bash
# tools/menuconfig_to_code.sh
part_s=$(grep -n partition@0 ${LINUX_DTS_PATH} | head -1 | cut -d: -f1 )
part_e=$(grep -n all_flash ${LINUX_DTS_PATH} | head -1 | cut -d: -f1 )
sed -i -e "${part_s},${part_e}d"  ${LINUX_DTS_PATH}
sed -i -e "$((part_s-1)) r t.sh" ${LINUX_DTS_PATH}
```

生成的分区顺序:spl→uboot→cfg 包→rtt→linux→rootfs→all\_flash,rootfs 排第 9 位，即 mtd9。所以官方 env 里写 `mtdblock9` 是对的，手写的 `mtdblock11` 反而错了。

看着像 bug 的原因：不知道这个脚本的话，会看到"官方 env 写 9，但官方 DTS 只有 spl\_boot+all\_flash 两个分区"。实际上 DTS 在编译前已被脚本改写。

额外的坑：脚本用 `grep` 匹配 `partition@0` 和 `all_flash` 的行号。如果 DTS 注释里出现 `all_flash` 字样，行号会错乱。遇到过：注释写了"分区保持官方骨架 (spl\_boot+all\_flash)",脚本把注释行当成了分区边界。删掉注释后恢复。

这次的问题不是分区表本身写错，而是没有先看 `menuconfig_to_code.sh` 的改写步骤。搞清楚脚本什么时候重写 DTS 后，mtd9 和 env 里的编号就能对上了。

### 7.2 PTE/MAEE:T-HEAD 高位页表位

症状:SDK 官方编译的 Linux 内核在 QEMU 上起不来，日志没有明确报错，死得很早。

原因:T-HEAD C9xx 内核的页表用了私有 MAEE 高位属性位 (bit 59-63):

```c
/* T-HEAD C9xx extend */
#define _PAGE_SEC	(1UL << 59)   /* Security */
#define _PAGE_SHARE	(1UL << 60)   /* Shareable */
#define _PAGE_BUF	(1UL << 61)   /* Bufferable */
#define _PAGE_CACHE	(1UL << 62)   /* Cacheable */
#define _PAGE_SO	(1UL << 63)   /* Strong Order */
```

QEMU 只认标准 RISC-V PTE(bit 0-9),高位置位后页表解析错乱。修复是置 0:

```c
#define _PAGE_SEC	0   /* Security (T-HEAD MAEE, QEMU 置 0) */
#define _PAGE_SHARE	0   /* Shareable (T-HEAD MAEE, QEMU 置 0) */
#define _PAGE_BUF	0   /* Bufferable (T-HEAD MAEE, QEMU 置 0) */
#define _PAGE_CACHE	0   /* Cacheable (T-HEAD MAEE, QEMU 置 0) */
#define _PAGE_SO	0   /* Strong Order (T-HEAD MAEE, QEMU 置 0) */
```

不能直接删宏：`_PAGE_CHG_MASK` 等位置引用着它们，删了编译不过。置 0 语义等价，是最小改动。

内核能编出来、能加载，并不说明 QEMU 能按这套页表属性启动；这里卡的是 T-HEAD 私有位和标准 RISC-V PTE 之间的差异。

### 7.3 buildroot 不自动卸载旧包

症状:buildroot 从 `k230_evb`(73 包) 切到 `k230d_canmv`(31 包) 后，rootfs.cpio 还是 38MB，蓝牙工具还在。

原因:buildroot 靠 `.config` 判断要装什么，但不会卸载 target/ 里已装的旧文件。配置变了，包的 stamp 还在，它认为已安装。

修复：删掉 `target/` 目录重编，只重装 target 包，host 工具缓存保留。

结论：配置切换不等于干净重编。换配置后要么删 target，要么 `make clean`。

### 7.4 genimage 的 RUNPATH 指向构建机

症状：`tools/genimage` 报 `cannot open shared object file: libconfuse.so.2`。

原因:SDK 编译出的 genimage 二进制，RUNPATH 里写死了编译它那台机器的 SDK checkout 路径。换机器或换目录后找不到动态库。

修复：建软链，把 RUNPATH 里的路径桥接到本地 buildroot host/lib。零改动 SDK。这个坑在"二进制产物自带 RUNPATH"的场合普遍存在，换工作目录就可能复现。

另外，老 buildroot(2020.02) 在新宿主系统上编译可能遇到 m4/fakeroot/cmake/scons 的版本兼容问题 (glibc 接口变更、cmake 4 移除旧版本兼容等)。SDK 的 `scripts/fix-env.sh` 里有处理，补丁打在 build 产物目录，不影响 SDK 源码和镜像内容。

## 8. 总结：为什么改

| 改动类型                       | 数量   | 作用                             |
| -------------------------- | ---- | ------------------------------ |
| 原地修改 (board.c / PUFS / PTE) | 3 处  | 修掉 QEMU 与真机环境的差异 (启动介质、安全硬件、页表) |
| 顶层 defconfig 派生 (5 个开关)     | 1 文件 | 把官方构建流程"指向"QEMU 板级             |
| 板级派生 (dts/defconfig)        | 3 文件 | QEMU 板级的静态描述                   |

其余都是 SDK 官方流程自己完成的:Linux 编译、buildroot、U-Boot 编译、genimage 装配。开关给对，官方流程就能干活。

## 9. 直接复现:k230-boot-test 仓库

全部改动和产物在 [k230-boot-test](https://github.com/flamboyante/k230-boot-test) 仓库：

```text
artifacts/   预构建镜像 (JFFS2 + UBI 两个变体)+ rootfs + SPL/U-Boot/env + SHA256SUMS
docs/        实验文档 (从零、含完整复现步骤)
scripts/     宿主环境探测/修复脚本 (check-env.sh / fix-env.sh)
locks/       SDK 与 QEMU 的增量 bundle + 基线锁定 (sources.lock)
```

clone 即跑：

```bash
git clone https://github.com/flamboyante/k230-boot-test.git
cd k230-boot-test
cd artifacts && sha256sum -c SHA256SUMS && cd ..

# 需要能模拟 K230 的 QEMU 构建
timeout 90s qemu-system-riscv64 \
  -M k230,spi-flash=w25q256 \
  -drive if=mtd,format=raw,file=artifacts/sysimage-spinor32m_jffs2.img \
  -nographic -monitor none -display none -no-reboot -snapshot </dev/null
```

从源码完整复现：公开仓库 README 的“版本矩阵”给出 SDK/QEMU 基线 commit，`locks/` 下的增量 bundle 交付本文讲的所有改动（SDK 7 个文件、QEMU 28 个 commit），`docs/coldboot-experiment.md` 提供逐步复现步骤。环境准备使用 `scripts/check-env.sh` 和 `scripts/fix-env.sh`。

<br />

## 10. 后续:SD 卡启动

SD 卡启动尚未验证，后续会单独处理。它会复用启动链的整体结构，但 QEMU 还需要补 SDIO 控制器和对应的介质读取路径。

## 参考资料

\[1] Kendryte. K230 Technical Reference Manual V0.3.1 (2024-11-18).
<https://github.com/revyos/external-docs/blob/master/K230/en-us/K230_Technical_Reference_Manual_V0.3.1_20241118.pdf>

\[2] K230 SDK(U-Boot / Linux / RT-Smart 源码). <https://github.com/kendryte/k230_sdk>

\[3] QEMU. <https://gitlab.com/qemu-project/qemu>

\[4] k230-boot-test 仓库 (冷启动复现 harness，含预构建产物与增量 bundle).
<https://github.com/flamboyante/k230-boot-test>

\[5] QEMU Camp 训练营仓库。<https://github.com/gevico/qemu-camp-tutorial>
