+++
title = '3.2 Kernel 构建与设备树'
date = '2026-06-09T17:03:00+08:00'
draft = false
+++

# 3.2 Kernel 构建与设备树

> 学习日期: 2026-04-30
> 参考项目: Amlogic S905X5M (ross) Android 14 TV
> 前置知识: Linux 内核基础概念、C 语言、ARM64 架构

---

## 1. 引言：Kernel 在 Android 中的角色

**概念解释**：Linux Kernel（内核）是操作系统的最底层核心，负责管理硬件资源（CPU、内存、外设）、提供进程调度、内存管理、网络协议栈、设备驱动等基础服务。在 Android 系统中，内核是**唯一直接与硬件交互**的软件层，Android 框架和应用程序的所有硬件的操作最终都要通过内核完成。

**代码体现**：参考代码库使用内核版本 **common14-5.15**，即基于 Linux 5.15 主线内核的 Android Common Kernel。构建命令为：

```bash
./mk ross -v common14-5.15
```

**实际价值**：没有内核，操作系统无法管理硬件资源。在嵌入式 Android 中，内核裁剪和驱动适配是 BSP（Board Support Package）的核心工作，直接影响系统的稳定性、性能和功能完整性。

---

## 2. GKI（Generic Kernel Image）

### 2.1 概念解释

**概念解释**：GKI（通用内核映像）是 Google 在 Android 11/12 引入的架构，将内核分为两部分：

- **GKI 内核（通用内核）**：由 Google 维护，包含核心内核代码和与 SoC 无关的驱动。所有设备使用相同的内核二进制。
- **GKI 模块（厂商模块）**：由 SoC 厂商（如 Amlogic）提供，以可加载内核模块（.ko）的形式存在，包含 SoC 特定的驱动。

GKI 的核心目标是实现**内核与厂商驱动的解耦**——SoC 厂商不需要 fork 内核源码，只需要提供与 GKI 接口兼容的模块。

```
传统方式:
  Google 发布内核 → 厂商 fork 并修改 → 每次内核更新都要重新适配

GKI 方式:
  Google 发布 GKI 内核二进制 → 厂商提供兼容的 .ko 模块 → 内核更新不影响厂商驱动
```

### 2.2 代码体现

在参考代码库中，`common/common14-5.15/` 目录是完整的 GKI 内核工程：

```
common/common14-5.15/
├── common/                # GKI 通用内核源码（主线 Linux + Android 补丁）
│   ├── build.config.gki.aarch64     # GKI ARM64 构建配置
│   ├── build.config.amlogic         # Amlogic 厂商构建配置
│   └── common_drivers/              # Amlogic 厂商驱动与设备树
└── driver_modules/                  # 其他外设驱动模块（GPU、WiFi、Camera）
```

**Amlogic 构建配置**（`build.config.amlogic`）：

```bash
# 基于 GKI 通用配置
. ${ROOT_DIR}/${KERNEL_DIR}/build.config.gki.aarch64

# 使用 Amlogic 的 defconfig（最终 = gki_defconfig + amlogic_gki.fragment）
DEFCONFIG=amlogic_gki_defconfig
FRAGMENT_CONFIG=${KERNEL_DIR}/arch/arm64/configs/amlogic_gki.fragment

# 合并配置
PRE_DEFCONFIG_CMDS=".../scripts/kconfig/merge_config.sh -m -r \
    gki_defconfig amlogic_gki.fragment"

# 编译目标
MAKE_GOALS="Image modules Image.lz4 Image.gz dtbs"
```

**构建产物**：
```bash
# ross-kernel/5.15/ 下为预编译产物
device/amlogic/ross-kernel/5.15/
├── ross.dtb              # ross 设备的 DTB
├── ross_soundbar.dtb     # Soundbar 变体 DTB
├── ross_mxl258c.dtb      # MXL258C 变体 DTB
├── dtbo.img              # DTBO（设备树 overlay）
├── gki/boot-lz4.img      # GKI 启动镜像
├── gki/Image.gz          # GKI 内核二进制
├── vendor_boot.modules.load   # vendor_boot 需要加载的模块列表
├── vendor_dlkm.modules.load   # vendor_dlkm 需要加载的模块列表
└── symbols/              # 调试符号
    ├── vmlinux           # 未压缩的内核映像（调试用）
    └── *.ko              # 各个内核模块的调试符号
```

### 2.3 实际价值

GKI 解决了 Android 内核碎片化问题。在 GKI 之前，每个设备厂商 fork 内核后各自修改，导致：
- 安全补丁需要厂商逐个移植，耗时数月
- 内核版本长期不更新，漏洞无法修复
- Google 难以统一测试

GKI 让内核安全更新可以独立于厂商驱动进行。用户可以通过 Play Store 获取内核更新，而不需要等待厂商发布完整 OTA。

---

## 3. Kernel 构建系统

### 3.1 defconfig 与 fragment

**概念解释**：

**defconfig（默认配置）**：内核的默认配置。Linux 内核有数千个可配置选项（驱动启用/禁用、功能开关、调试选项等）。`defconfig` 文件定义了特定平台/设备的基准配置集。内核编译时，`make defconfig` 根据这个文件生成最终的 `.config`。

**fragment（配置片段）**：在 GKI 架构中引入的概念。为了保持 GKI 内核的通用性，Google 提供 `gki_defconfig` 作为通用的基准配置，SoC 厂商通过 fragment 文件添加自己的配置需求，两者通过 `merge_config.sh` 合并。

**代码体现**：

```bash
# merge_config.sh 将 gki_defconfig 和 amlogic_gki.fragment 合并
PRE_DEFCONFIG_CMDS="scripts/kconfig/merge_config.sh -m -r \
    arch/arm64/configs/gki_defconfig \
    arch/arm64/configs/amlogic_gki.fragment"
```

**实际价值**：如果没有 defconfig/fragment 机制，SoC 厂商需要维护一份完整的配置文件副本，当上游 gki_defconfig 更新时，需要人工逐个比较差异并合并。fragment 机制让厂商只维护自己的增量部分，大大降低了内核配置的维护成本。

### 3.2 构建流程

```
./mk ross -v common14-5.15
  │
  ▼
mk.sh (common14-5.15/mk.sh)
  │
  ├── 设置参数: KERNEL_DIR=common, BUILD_CONFIG=build.config.amlogic
  │
  ├── Bazel 构建 (GKI 2.0):
  │     tools/bazel run //common:amlogic_dist
  │
  │     步骤:
  │     1. merge_config.sh → 合并 gki_defconfig + amlogic_gki.fragment
  │     2. make Image      → 编译内核
  │     3. make modules    → 编译内核模块
  │     4. make dtbs       → 编译设备树
  │     5. lz4_compress    → 压缩内核映像
  │     6. 复制产物到 dist 目录
  │
  └── Android 构建系统:
        ├── build_kernel_modules.mk → 组织内核模块到各分区
        ├── boot.img 打包           → kernel + ramdisk + dtb
        ├── vendor_boot.img 打包    → vendor 内核模块 + dtbo
        └── dtb-avb.img 生成        → 带 AVB2 签名的 DTB 映像
```

**Bazel**：Google 的构建系统。`common14-5.15` 使用 Bazel 作为主构建引擎（`tools/bazel run`），替代了传统的 `build/build.sh`。Bazel 提供分布式构建、缓存和增量编译能力。

### 3.3 内核压缩格式：lz4

**概念解释**：lz4 是一种高速压缩算法，压缩速度远快于 gzip（约 5-10 倍），解压缩速度极快（约 2-4 GB/s）。在嵌入式系统中，内核映像通常压缩存储以节省空间，启动时由 U-Boot 或内核自解压代码解压。

**代码体现**：
```bash
# build.config.amlogic 中的自定义压缩命令
function lz4_compress() {
    lz4 -f -12 --favor-decSpeed ${OUT_DIR}/arch/arm64/boot/Image \
        ${OUT_DIR}/arch/arm64/boot/Image.lz4
}
```

`-12` 是最高压缩级别，`--favor-decSpeed` 优化解压速度。

**实际价值**：Android 设备启动速度至关重要。使用 lz4 可以减少内核解压时间几百毫秒，对终端用户的"开机速度"感知有明显提升。相比 gzip，lz4 以稍大的压缩体积换取更快的解压速度。

---

## 4. 设备树（Device Tree）

### 4.1 概念解释

**概念解释**：Device Tree（设备树）是一种描述硬件配置的数据结构，以树状层级描述 CPU 数量、内存地址范围、外设（I2C、SPI、UART、GPIO、显示、音频等）的寄存器地址、中断号、时钟频率等硬件信息。

在 Linux 内核引入设备树之前，ARM Linux 内核需要为每一块开发板编写大量的 `board-*.c` 文件，在内核源码中硬编码硬件信息。这种做法导致内核源码急剧膨胀，且每块板子都需要重新编译内核。设备树将硬件描述从内核代码中分离出来，**同一份内核二进制可以在不同硬件上运行**，只需要更换设备树文件。

**关键格式**：

| 扩展名 | 格式 | 说明 |
|--------|------|------|
| **.dts** | Device Tree Source | 设备树源码，人类可读的文本格式 |
| **.dtsi** | Device Tree Source Include | 设备树包含文件（类似 C 语言的 .h 头文件），定义 SoC 级别的共用硬件 |
| **.dtb** | Device Tree Blob | 设备树二进制，编译后的设备树，U-Boot/内核实际使用的格式 |
| **.dtbo** | Device Tree Blob Overlay | 设备树 overlay，动态覆盖/扩展基础 DTB |

**编译关系**：

```
.dts + .dtsi → DTC (Device Tree Compiler) → .dtb
```

### 4.2 DTS 基本语法（零基础入门，以 ross DTS 为例）

设备树语法基于**节点（node）**和**属性（property）**，每个节点描述一个硬件，属性描述该硬件的参数。下面用 ross 产品实际使用的 `s7d_s905x5m_bm201_4g.dts` 逐步拆解。

#### 4.2.1 文件头与引用

```dts
// SPDX-License-Identifier: (GPL-2.0+ OR MIT)
/dts-v1/;

#include "meson-s7d.dtsi"
#include "hdr10plus.dtsi"
```

- `//` 是注释，和 C 语言一样
- `/dts-v1/;` 声明文件格式版本，固定写法
- `#include` 像 C 语言一样引用其他文件。`.dtsi` 是**可被包含的设备树文件**（类似 `.h` 头文件）。SoC 级通用定义放在 `.dtsi`，板级特有内容放 `.dts`，这样同一颗芯片的不同板子不用重复写 SoC 部分

#### 4.2.2 根节点（root node）

```dts
/ {
    amlogic-dt-id = "s7d_s905x5m_bm201-4g";
    compatible = "s7d_s905x5m_bm201-4g";
    interrupt-parent = <&gic>;
    #address-cells = <2>;
    #size-cells = <2>;
    // ... 子节点
};
```

- `/` 代表根节点，整棵树的起点
- `属性 = 值;` 描述硬件信息，常见值类型：

| 写法 | 含义 | 例子 |
|------|------|------|
| `"字符串"` | 字符串 | `compatible = "s7d_s905x5m_bm201-4g";` |
| `<数字>` | 32 位整数 | `#address-cells = <2>;` |
| `<0x0 0x80000000>` | 整数数组 | `reg = <0x0 0x07400000 0x0 0x00100000>;` |
| `&gic` | 引用另一个节点 | `interrupt-parent = <&gic>;` |

**三个最核心的属性**：

- **`compatible`**——告诉内核"我是什么设备"，驱动靠它匹配。内核驱动注册时说"我能驱动 `amlogic, meson-ir`"，设备树写 `compatible = "amlogic, meson-ir"`，两者对上号驱动就开始工作
- **`reg`**——硬件在内存中的地址。`reg = <0x0 0x07400000 0x0 0x00100000>` 因为 `#address-cells = <2>` 和 `#size-cells = <2>`（地址和大小都用 2 个 32 位整数表示），所以含义是：物理地址 `0x07400000`，大小 `0x100000`（1MB）
- **`status`**——启用/禁用：`"okay"` 或 `"disabled"`

#### 4.2.3 节点嵌套——树状结构

```dts
/ {                              ← 根节点
    memory@00000000 {             ← 子节点
        device_type = "memory";
        linux,usable-memory = <0x0 0x0 0x0 0xf0000000>;
    };

    gpio_leds {                   ← 子节点
        compatible = "gpio-leds";
        pwr_led {                  ← 子节点的子节点
            label = "pwr_led";
            default-state = "on";
        };
        bt_led {
            label = "bt_led";
            gpios = <&gpio GPIODV_6 GPIO_ACTIVE_HIGH>;
        };
    };
};
```

节点名后的 `@00000000` 表示该设备的起始地址。`memory@00000000` = "内存从地址 0 开始"，`ramoops@0x07400000` = "ramoops 在 0x07400000"。

#### 4.2.4 `&` 引用——在别处补充节点

```dts
&ir {
    status = "okay";
    pinctrl-0 = <&remote_pins>;
    pinctrl-names = "default";
};
```

`&ir` 的意思是"找到名叫 `ir` 的节点，往里面加属性"。`ir` 节点定义在 SoC 级 `.dtsi` 中（`meson-s7d.dtsi`）：

```dts
// meson-s7d.dtsi 中定义：
ir: ir@8000 {
    compatible = "amlogic, meson-ir";
    reg = <0x0 0xfe084080 0x0 0xA4>,
          <0x0 0xfe084000 0x0 0x58>;
    status = "disable";     ← 默认关闭
    protocol = <REMOTE_TYPE_NEC>;
    interrupts = <GIC_SPI 148 IRQ_TYPE_EDGE_RISING>;
    map = <&custom_maps>;
};
```

板级 DTS 用 `&ir` 把它**打开并指定引脚**：

```dts
&ir {
    status = "okay";         ← 改为启用
    pinctrl-0 = <&remote_pins>;
};
```

这种"基定义 + 板级补充"的模式是设备树的核心设计思想。

#### 4.2.5 标签（label）和引用 `&`

```dts
ir: ir@8000 { ... }
```

- `ir:` 是**标签**，类似别名，方便其他地方引用
- `ir@8000` 是**节点名**，`@8000` 是寄存器基地址

其他地方用 `&ir` 引用：

```dts
&ir { status = "okay"; }     // 往 ir 节点加属性
serial0 = &uart_B;            // 引用 uart_B 作为串口0
interrupt-parent = <&gic>;   // 引用 gic 中断控制器
```

#### 4.2.6 aliases——给节点起短名

```dts
aliases {
    serial0 = &uart_B;
    serial1 = &uart_A;
    i2c0 = &i2c0;
    spi0 = &spi_nfc;
};
```

这样内核就知道"串口0"对应 `uart_B`，"I2C0"对应 `i2c0`。

#### 4.2.7 每个按键的完整 DTS 路径

以红外遥控器按键为例，看 DTS 怎么逐层描述硬件：

```dts
// ① 根节点声明中断控制器
interrupt-parent = <&gic>;

// ② SoC 级定义 IR 控制器（meson-s7d.dtsi）
ir: ir@8000 {
    compatible = "amlogic, meson-ir";
    reg = <0x0 0xfe084080 0x0 0xA4>;    // 寄存器地址
    interrupts = <GIC_SPI 148 IRQ_TYPE_EDGE_RISING>;  // 中断148
    protocol = <REMOTE_TYPE_NEC>;       // NEC 协议
    map = <&custom_maps>;               // 指向按键映射表
};

// ③ 按键映射表（meson-ir-map.dtsi）
custom_maps: custom_maps {
    map0 = <&map_0>;
    map_0: map_0 {
        customcode = <0xfb04>;          // 遥控器地址码
        keymap = <
            REMOTE_KEY(0x47, KEY_0)     // 按键0x47 → KEY_0
            REMOTE_KEY(0x1A, KEY_POWER) // 按键0x1A → KEY_POWER
        >;
    };
};

// ④ 板级使能 IR（s7d_s905x5m_bm201_4g.dts）
&ir {
    status = "okay";
    pinctrl-0 = <&remote_pins>;         // GPIO 引脚
};
```

内核启动后，驱动根据这些信息初始化 IR 硬件，用户按遥控器时驱动查 keymap 表，上报按键事件。

#### 4.2.8 看懂 `#address-cells` 和 `#size-cells`

这两个属性决定了 `reg` 中地址和大小各占几个 32 位整数：

```dts
#address-cells = <2>;   // 地址用 2 个 cell（64位）
#size-cells = <2>;      // 大小用 2 个 cell（64位）

// 所以 reg = <0x0 0x07400000  0x0 0x00100000>
//            [地址高32位]      [大小高32位]
//            [地址低32位]      [大小低32位]
//            = 地址 0x07400000, 大小 0x100000
```

如果 `#address-cells = <1>; #size-cells = <1>;`（32位地址空间），则：

```dts
reg = <0x07400000 0x00100000>;  // 地址0x07400000, 大小0x100000
```

#### 4.2.9 DTS 语法极简速查表

| 语法 | 含义 | 例子 |
|------|------|------|
| `/dts-v1/;` | 文件头 | 每个 DTS 第一行 |
| `#include "x.dtsi"` | 包含文件 | `#include "meson-s7d.dtsi"` |
| `节点名 { };` | 定义硬件节点 | `cpu { }` |
| `标签:节点名@地址` | 节点加引用标签 | `ir: ir@8000` |
| `属性 = "字符串";` | 字符串属性 | `status = "okay";` |
| `属性 = <数字>;` | 整数属性 | `#address-cells = <2>;` |
| `属性 = <1 2 3>;` | 整数数组 | `reg = <0x0 0x80000000 0x0 0x100000>;` |
| `属性 = &标签;` | 引用节点 | `interrupt-parent = <&gic>;` |
| `&标签 { };` | 补充节点属性 | `&ir { status = "okay"; };` |
| `/* */` 或 `//` | 注释 | `// 这是注释` |

#### 4.2.10 一句话总结

**设备树就是一份"硬件连接说明书"**。SoC 厂家写 `.dtsi` 定义芯片内部有什么外设，板卡厂家写 `.dts` 定义板子上接了什么硬件，内核启动时读这份说明书来初始化驱动。

### 4.3 代码体现 — S7D SoC 设备树层级

**概念解释**：Amlogic S905X5M（S7D 平台）的设备树采用三层结构：

```
1. SoC 级 (.dtsi):    meson-s7d.dtsi          ← 定义 S7D 芯片的全部外设
2. SoC 子模块 (.dtsi): mesons7d_drm.dtsi       ← DRM 显示子系统
                       mesons7d_audio.dtsi     ← 音频子系统
                       mesons7d_cpufreq.dtsi   ← CPU 频率调节
3. 板级 (.dts):       s7d_s905x5m_bm201.dts   ← 具体电路板配置
```

**meson-s7d.dtsi（SoC 级）**：定义 S7D 芯片所有 CPU 核心、外设控制器、中断控制器、时钟、引脚复用等，位于：

```
common/common14-5.15/common/common_drivers/arch/arm64/boot/dts/amlogic/meson-s7d.dtsi
```

内容包含：
- **4 核 Cortex-A55 CPU**（通过 PSCI 启停）
- **GIC（Generic Interrupt Controller）**：中断控制器
- **timer**：ARM 架构计时器
- **uart_A/B/C/D/E**：5 个 UART 串口
- **i2c0-4**：5 个 I2C 控制器
- **spicc0/spi_nfc**：SPI 控制器
- **sd_emmc_a/b/c**：SD/eMMC 控制器
- **ethmac**：以太网 MAC
- **clkc**：时钟控制器
- **pinctrl**：引脚功能复用

**概念解释——GIC（Generic Interrupt Controller）**：ARM 架构的标准中断控制器，负责将外设的中断信号路由到 CPU 核心。支持 SPI（共享外设中断）、PPI（私有外设中断，每个 CPU 核心独立）、LPI（本地特定中断）三种类型。

**概念解释——PSCI（Power State Coordination Interface）**：ARM 定义的电源管理接口标准，由 BL31（ATF）实现。内核通过 SMC（Secure Monitor Call）指令调用 PSCI 来管理 CPU 的启停和休眠，如 CPU 热插拔、suspend/resume。

**s7d_s905x5m_bm201.dts（板级）**：ross 设备的具体配置，位于：

```
common/common14-5.15/common/common_drivers/arch/arm64/boot/dts/amlogic/s7d_s905x5m_bm201.dts
```

通过 `#include` 包含 SoC 级 DTSI，然后覆盖或添加板级特有内容：
- **内存布局**：2GB DDR（`reg = <0x0 0x0 0x0 0x80000000>`）
- **保留内存**：为各种子系统预留内存区域
- **外设引脚配置**：GPIO 引脚功能选择（如 I2C 用哪组引脚）
- **电源调节器（regulator）**：板级电压定义
- **音频路由**：codec、DAI link 配置
- **WiFi/BT**：BCM 模块的 GPIO 和电源配置
- **热管理**：thermal zones、降温策略

### 4.4 DTB 的加载流程

```
编译时:
  .dts + .dtsi → DTC → .dtb （内核编译时生成）
  .dtb + overlay → dtc → .dtbo

启动时:
  U-Boot 从 dtb 分区读取 dtb → 传递给内核
  内核解析 DTB → 枚举设备 → 加载对应驱动

  或（vendor_boot 方式）:
  boot.img 包含内核 → vendor_boot.img 包含 DTB + DTBO
  U-Boot 加载两者到内存 → 内核合并 DTB 和 DTBO → 枚举设备
```

**DTBO（Device Tree Overlay）**：允许在不修改主 DTB 的情况下动态添加或覆盖设备节点。在 Android 中用于分离 SoC DTB 和板级差异，当同一 SoC 用于不同产品时，只需更换 DTBO 而不需要重新编译内核。

**代码体现**：
```makefile
# BoardConfig.mk
BOARD_PREBUILT_DTBOIMAGE := $(DEVICE_PRODUCT_PATH)-kernel/$(TARGET_KERNEL_DIR)/dtbo.img
```

`device/amlogic/ross-kernel/5.15/dtbo.img` 是 ross 设备的 DTBO 预编译映像。

### 4.5 实际价值

设备树是嵌入式 Linux 开发的基石。没有设备树：
- 每块新电路板都需要修改内核源码，增加大量 `board-*.c` 文件
- 编译一次内核只能支持一种硬件
- 硬件配置变更（如调整内存大小、更换 WiFi 模块）都需要重新编译内核
- Android 生态碎片化将更加严重

---

## 5. Reserved Memory（保留内存）

**概念解释**：reserved-memory（保留内存）是指在系统总内存中划出一部分，预留给特定硬件或软件功能使用。内核和用户空间不能随意使用这些内存区域。在嵌入式 Android 中，保留内存用于：

- **安全监控**（secmon）：TEE 安全世界使用的内存
- **显存**（ion/CMA）：GPU 和显示系统使用的连续物理内存
- **编解码器**（codec_mm）：视频编解码使用的内存
- **调试**（ramoops）：保存内核 panic 日志
- **安全视频解码**（secure_vdec）：DRM 内容解密后的帧缓存

**代码体现**（从 `s7d_s905x5m_bm201.dts`）：

```dts
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    /* 内核崩溃日志保留区域 — ramoops */
    ramoops@0x07400000 {
        compatible = "ramoops";
        reg = <0x0 0x07400000 0x0 0x00100000>;  // 1MB
        record-size = <0x20000>;    // 128KB 主记录
        console-size = <0x40000>;   // 256KB console 日志
        ftrace-size = <0x80000>;    // 512KB ftrace 跟踪
        pmsg-size = <0x10000>;      // 64KB 用户空间消息
    };

    /* TEE 安全监控内存 */
    secmon_reserved:linux,secmon {
        no-map;                     // 不建立页表映射（安全世界独占）
        reg = <0x0 0x05000000 0x0 0x2400000>;  // 36MB
    };

    /* CMA：用于连续内存分配 */
    dmaheap_cma_reserved:heap-gfx {
        compatible = "shared-dma-pool";
        reusable;                   // 未使用时可被内核使用
        size = <0x0 0x5800000>;     // 88MB（给 GPU 图形用）
    };

    /* 视频编解码 CMA */
    codec_mm_cma:linux,codec_mm_cma {
        reusable;
        size = <0x0 0x16C00000>;    // 364MB（CTS 测试最大场景）
    };

    /* 安全视频解码 */
    secure_vdec_reserved:linux,secure_vdec_reserved {
        no-map;
        size = <0x0 0x1000000>;     // 16MB
    };
};
```

**概念解释**：

- **CMA（Contiguous Memory Allocator）**：内核的一种内存分配机制，专门用于分配大块连续物理内存。嵌入式设备中的 GPU、视频编解码、相机等硬件 DMA 引擎需要连续的物理内存，而标准内核内存分配器（buddy allocator）容易导致碎片化。CMA 预留一个大区域，在未被硬件占用时可被内核普通页面使用，硬件需要时再回收——兼顾了内存利用率和分配需求。

- **ion**：Android 的内存分配器框架，提供多种堆（heap）类型用于不同硬件模块（显存、多媒体、系统）。`ion_cma_reserved` 是 CMA 类型的堆。

- **ramoops**：内核崩溃时，在保留内存中保存最后一次的日志（console log、ftrace 等）。重启后可以通过 `dmesg` 或 `pstore` 文件系统读取上次崩溃的信息。对嵌入式设备调试**极度重要**——因为设备通常没有串口连接。

- **no-map**：DTS 中的标志，指示内核不要为这片内存建立页表映射。用于安全世界独占的内存（如 TEE），普通世界（包括 Linux 内核）无权访问。

**实际价值**：保留内存机制是嵌入式系统内存管理的关键。如果没有合理的保留内存分配：
- 视频解码找不到足够的连续内存，导致播放卡顿
- 内核崩溃后日志丢失，无法定位问题
- TEE 安全内存被普通世界访问，安全防线被突破
- GPU 运行出现内存碎片导致的停滞

---

## 6. 内核模块（Kernel Modules）

### 6.1 概念解释

**概念解释**：内核模块（.ko = kernel object）是可以在不重新编译内核的情况下动态加载到内核中的代码片段。它实现了设备驱动、文件系统、协议栈等功能，但以独立文件形式存在。需要时用 `insmod` 或 `modprobe` 加载，不需要时用 `rmmod` 卸载。

在 GKI 架构下，SoC 厂商的驱动都以模块形式提供，不直接编译进内核映像。

### 6.2 代码体现

**模块加载列表**（`device/amlogic/ross-kernel/5.15/vendor_boot.modules.load`）：

```text
amlogic-gkitool.ko
amlogic-debug-iotrace.ko
amlogic-uart.ko
amlogic-hwspinlock.ko
amlogic-debug.ko
amlogic-secmon.ko
amlogic-cpuinfo.ko
page_trace.ko
aml_cma.ko
clk-scmi.ko
amlogic-clk.ko
amlogic-clk-soc-s7d.ko
amlogic-aoclk-soc-t5w.ko
amlogic-gpio.ko
amlogic-pinctrl-soc-s7d.ko
amlogic-mailbox.ko
amlogic-pwm.ko
...
```

**模块分类**：

| 类别 | 模块 | 功能 |
|------|------|------|
| 时钟 | `amlogic-clk.ko`, `clk-scmi.ko` | SoC 时钟控制 |
| GPIO | `amlogic-gpio.ko` | GPIO 控制 |
| 引脚复用 | `amlogic-pinctrl-soc-s7d.ko` | SoC 引脚功能配置 |
| 安全监控 | `amlogic-secmon.ko` | 安全世界通信接口 |
| UART | `amlogic-uart.ko` | 串口驱动 |
| 调试 | `amlogic-debug.ko` | 内核调试支持 |
| DDR | `amlogic-memory-debug.ko` | 内存调试 |
| CMA | `aml_cma.ko` | CMA 内存分配器管理 |

**GKI 模块加载顺序**（`vendor_dlkm.modules.load`）：

模块按依赖顺序排列，`modprobe` 按列表顺序加载。

### 6.3 构建集成

```makefile
# build_kernel_modules.mk
# 读取 modules.load 列表
RAMDISK_KERNEL_MODULES_LOAD := $(shell cat \
    $(DEVICE_PRODUCT_PATH)-kernel/$(TARGET_KERNEL_DIR)/vendor_boot.modules.load)

# 将模块打包进 vendor_boot 的 ramdisk
BOARD_VENDOR_RAMDISK_KERNEL_MODULES ?= $(RAMDISK_KERNEL_MODULES)
BOARD_VENDOR_RAMDISK_KERNEL_MODULES_LOAD ?= $(RAMDISK_KERNEL_MODULES_LOAD)
```

### 6.4 实际价值

内核模块机制让嵌入式系统更加灵活：
- 不需要的驱动可以不加载，节省内存和启动时间
- 问题驱动可以单独替换，不需要重新编译整个内核
- GKI 架构下，模块是厂商定制内核的唯一方式——没有模块机制，GKI 就不可能实现

---

## 7. Boot Image 打包架构

### 7.1 boot.img 结构

**概念解释**：boot.img 是 Android 的启动映像，包含内核和初始根文件系统（ramdisk）。U-Boot 将 boot.img 加载到内存后，内核从 ramdisk 启动 init 进程。

```
boot.img
├── kernel           ← 内核映像 (Image.lz4)
├── ramdisk          ← 初始根文件系统（init 和启动脚本）
├── dtb              ← 设备树（部分配置中在 boot.img 末尾）
└── boot header      ← 启动头（内核加载地址、ramdisk 地址等）
```

**代码体现**：
```makefile
# kernel_config_build.mk
KERNEL_ROOTDIR := common
BOARD_MKBOOTIMG_ARGS := --kernel_offset $(BOARD_KERNEL_OFFSET) \
                        --header_version $(BOARD_BOOT_HEADER_VERSION)
```

### 7.2 vendor_boot.img

**概念解释**：vendor_boot.img 是 Android 13+ 引入的启动映像，用于将厂商特定的内核模块和 DTB 与通用 boot.img 分离。这使得 GKI 内核映像（boot.img）可以保持通用，而厂商定制内容放在 vendor_boot.img 中。

```
vendor_boot.img
├── vendor ramdisk       ← 厂商内核模块 (.ko)
├── dtb                  ← 设备树二进制（厂商板级 DTB）
└── dtbo                 ← DTBO overlay
```

### 7.3 DTB 分区

**概念解释**：在 Amlogic 平台中，DTB 可以存储在独立分区（dtb 分区）或内嵌到 boot.img/vendor_boot.img 中。独立 dtb 分区的好处是：更换设备树映像不需要重新烧录整个 boot。

```makefile
# build_kernel_modules.mk
# 将 DTB 打包为 dtb-avb.img（带 AVB hashtree footer）
$(INSTALLED_BOARDDTB_TARGET): $(LOCAL_DTB)
    cp $(LOCAL_DTB) $@
    $(AVBTOOL) add_hash_footer \
        --partition_size $(BOARD_DTBIMAGE_PARTITION_SIZE) \
        --partition_name dt
```

### 7.4 各分区对应关系

| 分区 | 内容 | 来源 |
|------|------|------|
| boot | kernel + generic ramdisk | GKI 预编译 |
| vendor_boot | vendor modules + DTB + DTBO | ross-kernel/5.15/ |
| dtb | 设备树（独立分区） | ross.dtb 等 |
| dtbo | DTBO overlay | dtbo.img |
| vendor_dlkm | 动态加载的 vendor 模块 | ko 文件 + modules.load |

---

## 8. Kernel Command Line

**概念解释**：内核命令行（kernel cmdline）是在内核启动时传递给内核的参数字符串，通过 U-Boot 的 `bootargs` 环境变量设置。内核解析这些参数来配置启动行为，如 console 输出、SELinux 模式、init 路径、分区信息等。

**代码体现**（`BoardConfig.mk`）：

```makefile
BOARD_KERNEL_CMDLINE += bootconfig
BOARD_KERNEL_CMDLINE += androidboot.selinux=permissive   # SELinux 宽松模式（开发版）
BOARD_KERNEL_CMDLINE += androidboot.serialno=$(SERIAL)
BOARD_KERNEL_CMDLINE += firmware_class.path=/vendor/etc/firmware

# Boot console 为 ttyS0，波特率 921600
BOARD_KERNEL_CMDLINE += console=ttyS0,921600
```

含义：
- `androidboot.selinux=permissive`：SELinux 在 permissive 模式运行，只记录不阻止违规操作（开发调试用）
- `console=ttyS0,921600`：串口控制台，用于调试日志输出
- `bootconfig`：启用 Boot Configuration（一种增强内核参数传递机制）

---

## 9. 板级变体与 DTB

ross 设备有多种硬件变体，每种使用不同的 DTB：

| 产物的 DTB | 来源（KERNEL_DEVICETREE） | 对应产品 |
|-----------|--------------------------|---------|
| `ross.dtb` | `s7d_s905x5m_bm201.dtb` + `s7d_s905x5m_bm202.dtb` + `s7d_s905x5m_bm209.dtb` + `s7d_s905x5m_bm201_4g.dtb` 四合一 | ross (S905X5M) 全系列 |
| `ross_soundbar.dtb` | `s7d_s905x5m_bm201_soundbar.dtb` | ross Soundbar 版 |
| `ross_mxl258c.dtb` | `s7d_s905x5m_bm201_mxl258c.dtb` | ross MXL258C 版 |

对应板级 DTS 源码路径：

```
common/.../dts/amlogic/s7d_s905x5m_bm201.dts
common/.../dts/amlogic/s7d_s905x5m_bm202.dts
common/.../dts/amlogic/s7d_s905x5m_bm209.dts
common/.../dts/amlogic/s7d_s905x5m_bm201_4g.dts       ← 当前 ross 4G 版
common/.../dts/amlogic/s7d_s905x5m_bm201_soundbar.dts
common/.../dts/amlogic/s7d_s905x5m_bm201_mxl258c.dts
```

### 9.1 ross.dtb 实际是多 DTB 合并镜像

在 `common/project/amlogic/ross/build.config.meson.arm64.trunk.5.15` 中定义了：

```bash
KERNEL_DEVICETREE="s7d_s905x5m_bm201 s7d_s905x5m_bm202 s7d_s905x5m_bm209 s7d_s905x5m_bm201_4g"
KERNEL_DEVICETREE_SOUNDBAR="s7d_s905x5m_bm201_soundbar"
KERNEL_DEVICETREE_FCC_PIP="s7d_s905x5m_bm201_mxl258c"
```

`KERNEL_DEVICETREE` 包含 4 个 DTS 目标，编译出 4 个独立 DTB。构建脚本 `build_kernel_5.15.sh` 检测到 `dtb_files_count > 1`，使用 dtbTool 将 4 个 DTB 合并为单一 `ross.dtb`：

```bash
# build_kernel_5.15.sh L218-L219
${DTBTOOL_DIR}/dtbTool -o ${DEVICE_KERNEL_DIR}/${BOARD_DEVICENAME}.dtb \
    -p ${DTBTOOL_DIR}/ ${OUT_AMLOGIC_DIR}/dtb/
```

U-Boot 启动时通过硬件检测（如内存大小或 GPIO 电平）自动匹配合适的 DTB 加载，一份 `ross.dtb` 覆盖 4 种硬件变种。当前 ross 产品实际对应 `s7d_s905x5m_bm201_4g.dts`（4GB 内存）。

---

## 10. 流程图汇总

### Kernel 构建与集成流程

```
内核源码构建（common14-5.15/）
┌─────────────────────────────────────────────────┐
│  gki_defconfig + amlogic_gki.fragment           │
│          ↓ merge_config.sh                      │
│     .config (完整配置)                            │
│          ↓ make                                 │
│  ┌──────┼──────────┐                            │
│  ↓      ↓          ↓                            │
│ Image  modules    dtbs                          │
│  ↓ lz4   ↓          ↓                           │
│ Image.lz4 .ko  s7d_s905x5m_bm201.dtb            │
└──────┬──────┬──────────┬────────────────────────┘
       │      │          │
       ▼      ▼          ▼
    Android 构建系统集成
┌─────────────────────────────────────────────────┐
│  boot.img  ← Image.lz4 + generic ramdisk         │
│  vendor_boot.img ← .ko + dtb + dtbo              │
│  dtb.img   ← dtb (with AVB footer)               │
│                                                    │
│  最终产物:                                          │
│  out/target/product/ross/                          │
│  ├── boot.img                                      │
│  ├── vendor_boot.img                               │
│  ├── dtb-avb.img                                   │
│  └── vendor/lib/modules/*.ko                       │
└─────────────────────────────────────────────────┘
```

### 设备树层级关系

```
meson-s7d.dtsi             ← SoC 级（S7D 芯片所有外设）
  ├── mesons7d_drm.dtsi    ← DRM 显示子系统
  ├── mesons7d_audio.dtsi  ← 音频子系统
  ├── mesons7d_cpufreq.dtsi← CPU 频率调节
  └── firmware_ab.dtsi     ← A/B 系统固件区域

s7d_s905x5m_bm201.dts      ← 板级（ross 设备）
  └── #include meson-s7d.dtsi
       ├── memory: 2GB DDR
       ├── reserved-memory: secmon/ramoops/CMA
       ├── regulator: 电压调节器定义
       ├── audio: I2S/SPDIF/PCM 路由
       ├── WiFi/BT: BCM 模块 GPIO
       └── thermal: 散热策略
```

---

## 11. 关键代码文件索引

| 功能 | 路径 |
|------|------|
| GKI 内核工程根目录 | `common/common14-5.15/` |
| Amlogic 构建配置 | `common/common14-5.15/common/build.config.amlogic` |
| Amlogic defconfig fragment | `common/common14-5.15/common/common_drivers/arch/arm64/configs/amlogic_gki.fragment` |
| S7D SoC 级设备树 | `common/common14-5.15/common/common_drivers/arch/arm64/boot/dts/amlogic/meson-s7d.dtsi` |
| ross 板级设备树 | `common/common14-5.15/common/common_drivers/arch/arm64/boot/dts/amlogic/s7d_s905x5m_bm201.dts` |
| 内核构建脚本 | `common/common14-5.15/mk.sh` |
| 内核模块集成 | `device/amlogic/common/build_kernel_modules.mk` |
| ross 内核预编译产物 | `device/amlogic/ross-kernel/5.15/` |
| ross 内核配置 | `device/amlogic/ross/kernel_config_build.mk` |
| BoardConfig (kernel 部分) | `device/amlogic/ross/BoardConfig.mk` |

---

## 12. 自问自答

> **Q**: GKI 和传统内核构建的主要区别？
> **A**: 传统方式厂商 fork 整个内核源码修改，GKI 方式厂商只提供内核模块和设备树，通用内核直接用 Google 预编译的 GKI 映像。区别类似于"从头写一个类 vs 继承并重写几个方法"——GKI 大幅降低了厂商维护成本。

> **Q**: DTS 和 DTB 是什么关系？
> **A**: DTS 是源码（文本），DTB 是编译后的二进制文件（机器可读）。类比：.c → 编译器 → .elf。内核启动时解析的是 DTB，不是 DTS。所以烧录到设备上的是 .dtb 文件。

> **Q**: 为什么需要保留内存？直接 malloc 不行吗？
> **A**: 不行。普通应用程序通过 malloc 分配的内存是虚拟地址，物理内存可能不连续。但 DMA 硬件（GPU、视频编解码）需要物理连续的内存，这就是 CMA 保留内存的作用。此外，安全世界（TEE）用的内存不能给 Linux 使用，也需要预先保留。

> **Q**: ramoops 怎么用？
> **A**: 设备内核崩溃后重启，进入系统后执行 `cat /sys/fs/pstore/console-ramoops-0` 查看上次崩溃前的内核日志。这对没有串口的量产设备调试非常有用。

> **Q**: 一个 SoC 多个产品怎么管理 DTB？
> **A**: 通用部分放在 SoC 级 .dtsi，差异部分放在板级 .dts。更灵活的方式是使用 DTBO——SoC DTB 通用，不同产品的差异通过 DTBO overlay 叠加。这样只需要维护一份 SoC DTB 和多个小 DTBO。

> **Q**: vendor_boot 和 boot 分开了，好处是什么？
> **A**: 厂商更新驱动只需要替换 vendor_boot 分区，boot 分区（GKI 内核）保持不变。这对安全更新意义重大——Google 可以通过 Play Store 推送 GKI 内核更新而不影响厂商的驱动。

---

## 13. 下一步

理解内核构建设备树后，建议继续深入：
- [ ] **HAL 与 Treble**（`notes/09-hal-treble-hidl接口.md`）：理解内核之上 Android 如何抽象硬件
- [ ] **SELinux**（`notes/10-selinux-安全策略.md`）：理解内核安全子系统
- [ ] **深入驱动**：选择一个 Amlogic 驱动模块（如 `amlogic-pinctrl` 或 `amlogic-secmon`），阅读源码理解内核驱动开发模式
