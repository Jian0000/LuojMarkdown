+++
title = '07-3.1Bootloader：U-Boot+安全启动'
date = '2026-06-09T17:02:00+08:00'
draft = false
+++

# 3.1 Bootloader：U-Boot + 安全启动

> 学习日期: 2026-04-30
> 参考项目: Amlogic S905X5M (ross) Android 14 TV
> 前置知识: 嵌入式系统启动流程、ARM TrustZone 基本概念

---

## 1. 引言：Bootloader 是什么？

**概念解释**：Bootloader（启动加载器）是设备上电后第一个执行的软件。它的职责是初始化硬件（时钟、DDR 内存、存储控制器）、从存储介质（eMMC/NAND）加载操作系统映像到内存，然后跳转执行。在 Android 嵌入式系统中，bootloader 不仅要"启动系统"，还要**验证启动链的完整性**，防止被篡改的系统运行。

**代码体现**：参考代码库中，bootloader 源码位于 `bootloader/uboot-repo/`，构建命令为：
```bash
./mk s7d_bm201 --avb2 --vab
# 或针对 ross 设备：
cd bootloader/uboot-repo && ./mk s7d_bm201 --update-bl2 --update-bl2e --update-bl2x --update-bl31 --update-bl32 --avb2 --vab
```

**实际价值**：没有 bootloader，芯片上电后 CPU 不知道从哪里取指令、内存没有初始化、没有任何代码可以执行。在 Android 安全模型中，bootloader 是第一道防线——如果 bootloader 不可信，后续整个系统都不可信。

---

## 2. ARM 处理器启动流程基础

在深入 Amlogic U-Boot 之前，需要理解 ARM 处理器的启动模式：

### 2.1 ARM TrustZone

**概念解释**：TrustZone 是 ARM 架构的硬件安全扩展，将 SoC 内部资源划分为两个世界：
- **Normal World（普通世界）**：运行 Linux/Android 等富操作系统
- **Secure World（安全世界）**：运行可信执行环境（TEE），处理密钥、认证、DRM 等敏感操作

两个世界通过**监控模式（Monitor Mode）**切换，由硬件强制隔离。

### 2.2 ARM 启动时安全等级

ARM 处理器启动时默认处于 **Secure World**。bootloader 的前几个阶段（BL1/BL2）在安全世界中执行，确保安全启动链从头建立。只有当安全启动链路验证通过后，才会跳转到 Normal World 启动 Android。

### 2.3 Amlogic S905X5M 的处理器架构

S905X5M 基于 ARM 大小核架构：
- **Cortex-A76**：大核（高性能）
- **Cortex-A55**：小核（低功耗）

使用 ARMv8.2-A 架构，支持 64 位运算。多核心间的调度管理由内核的 HMP（Heterogeneous Multi-Processing）或 EAS（Energy-Aware Scheduling）处理。

---

## 3. U-Boot 是什么？

**概念解释**：U-Boot（Universal Bootloader）是嵌入式 Linux 系统中最主流的开源 bootloader，支持 ARM、RISC-V、x86 等多种架构。它提供：
- 硬件初始化（DDR、时钟、存储、网络）
- 命令行交互界面（类似小型 OS 的 shell）
- 从多种介质引导内核（eMMC、NAND、USB、TFTP）
- 支持脚本化和环境变量配置

**代码体现**：Amlogic 参考代码库使用 **U-Boot v2019**（位于 `bootloader/uboot-repo/bl33/v2019/`），这是一个基于 2019 年主线 U-Boot 的 Amlogic 定制版本。同时仓库也保留了 v2015、v2023、v2025 版本用于不同芯片。

**实际价值**：在 Android 嵌入式系统中，U-Boot 负责最终加载 Android 内核（boot.img）并启动系统。Amlogic 在主线 U-Boot 基础上增加了大量定制功能：AVB2 验证、VAB A/B 槽控制、fastboot 下载、分区管理等。

---

## 4. Amlogic 多级启动链（Boot Flow）

这是 Amlogic 芯片独有的启动架构，不同于一般 ARM 平台的两级启动（Boot ROM → U-Boot），Amlogic 将启动过程拆分为 5 个阶段：

### 4.1 总体架构

```
Boot ROM (芯片固话)
   ↓ [加载到 SRAM]
BL2 (bootloader 第一阶段)     ← 安全世界
   ↓
BL30 (SCP 协处理器固件)       ← 电源管理
   ↓
BL31 (ARM Trusted Firmware)  ← 安全监控器
   ↓
BL32 (TEE 可信执行环境)       ← 可选，安全服务
   ↓
BL33 (U-Boot)                ← 普通世界
   ↓
boot.img (Linux Kernel + ramdisk)
```

### 4.2 各阶段详解

#### BL2 — 第一阶段 Bootloader

**概念解释**：BL2（Boot Loader stage 2）是芯片 Boot ROM 之后执行的第一段用户代码。它运行在 SRAM（片上静态内存）中，负责：
- 初始化 DDR 内存控制器
- 加载后续各阶段（BL30/BL31/BL32/BL33）到 DDR 内存
- 验证各阶段固件的签名（安全启动启用时）

**代码体现**：BL2 源代码在 `bootloader/uboot-repo/bl2/` 目录，构建后生成 `bl2.bin.sto`（用于 eMMC 启动）和 `bl2.bin.usb`（用于 USB 烧录启动）。

构建流程在 `fip/s7d/build.sh` 中通过 `acpu-imagetool` 工具对 BL2/BL2E/BL2X 进行签名和打包：
```bash
acpu-imagetool create-boot-blobs \
    --chipset-authen-algorithm=rsa,none \
    --device-authen-algorithm=rsa,none \
    --infile-bl2-payload=bl2.bin.sto \
    --scs-family=s7d \
    --outfile-bb1st=${output}/bb1st.sto.bin
```

**实际价值**：BL2 是整个安全启动链的"信任根"（Root of Trust）之后的第一个用户代码。如果 BL2 本身被篡改，整个设备就不安全了。因此 BL2 通常由 Boot ROM 使用芯片熔丝（efuse）中的公钥哈希验证。

#### BL30 — SCP 协处理器固件

**概念解释**：BL30 是运行在 SCP（System Control Processor，系统控制协处理器）上的固件。SCP 是一个独立的微控制器内核（Cortex-M 系列），专门负责：
- 电源管理（DVFS、休眠唤醒）
- 时钟门控和复位控制
- 系统热管理

**代码体现**：BL30 构建脚本在 `fip/build_bl30.sh`，生成的固件由 BL2 加载到 SCP 专用的内存空间。

**实际价值**：将电源管理独立到 BL30/SCP 上，主 CPU（Cortex-A76/A55）可以在空闲时彻底休眠，由 SCP 处理低功耗任务，这对 Android TV 的遥控器唤醒、待机功耗至关重要。

#### BL31 — ARM Trusted Firmware (ATF)

**概念解释**：BL31 运行在 Secure World 的监控模式，是 ARM Trusted Firmware 的核心组件。它实现：
- **SMC（Secure Monitor Call）处理**：普通世界和安全世界之间的网关
- **PSCI（Power State Coordination Interface）**：CPU 电源管理接口（CPU 开启/关闭/休眠）
- **安全中断路由**：确保安全中断直达 Secure World

**代码体现**：参考代码库支持两个 BL31 版本——`v1.3` 和 `v2.7`，位于 `bootloader/uboot-repo/bl31/`。构建脚本 `fip/build_bl31.sh` 根据不同 SoC 选择版本。

构建命令中 `--update-bl31` 参数指定需要重新构建 BL31。

**实际价值**：BL31 是 Android 安全模型在固件层的支柱。Linux 内核通过 PSCI 接口（如 `cpu_on`/`cpu_off`）管理 CPU 核心，而这些接口的实际实现由 BL31 提供。没有 BL31，Android 就无法管理多核 CPU 的电源状态。

#### BL32 — TEE 可信执行环境

**概念解释**：BL32 是运行在 Secure World 中的 TEE（Trusted Execution Environment）操作系统，常见的有：
- **OP-TEE**（Open Portable Trusted Execution Environment，开源）
- **Trusty**（Google 的 TEE 实现）

TEE 提供在安全世界中运行可信应用（TA，Trusted Application）的能力，处理：
- DRM 解密（Widevine L1）
- 指纹/人脸识别数据比对
- 支付密钥管理
- AVB 回滚索引的存储

**代码体现**：OP-TEE 的集成在 `bootloader/uboot-repo/bl32/`。构建时通过 `CONFIG_NEED_BL32` 控制是否包含 BL32。U-Boot 端有 OP-TEE 驱动支持 AVB TA（Trusted Application）。

**实际价值**：没有 TEE，敏感操作（如 DRM 解密、支付认证）必须在普通世界进行，攻击者可以通过 root 提权窃取密钥。TEE 通过硬件隔离确保即使 Linux 内核被攻破，密钥也不会泄露。

#### BL33 — U-Boot

BL33 就是前面介绍的 U-Boot，运行在普通世界。它是整个 bootloader 的最后阶段，负责加载 Android 系统。

### 4.3 FIP 打包

**概念解释**：FIP（Firmware Image Package）是 ARM Trusted Firmware 定义的一种固件打包格式，将 BL2、BL30、BL31、BL32、BL33 等多个固件镜像打包成一个文件，方便烧录和管理。

**代码体现**：构建产物输出到 `fip/_tmp/` 目录，通过 `package` 函数打包成最终的 `u-boot.bin.signed`。最终生成的 bootloader 烧录到 `bootloader` 分区的头部。

```bash
# build_uboot 函数调用链：
pre_build_uboot → init_variable_early → build_uboot → build_blx → package → copy_bootloader
```

**实际价值**：FIP 提供标准化的多固件打包方式，Boot ROM 只需要加载 BL2，BL2 从 FIP 包中解析并加载其余阶段。

---

## 5. AVB2 (Android Verified Boot 2.0)

### 5.1 概念解释

**概念解释**：AVB2（Android Verified Boot 2.0）是 Google 在 Android 中引入的安全启动验证框架。它的核心目标是确保**设备上运行的系统镜像与出厂时签名的一致**，任何篡改都会被检测到。

AVB2 的工作方式是在每个分区（boot、system、vendor 等）的末尾添加**加密认证信息**（hash footer 或 hashtree footer）。启动时逐层验证：

```
vbmeta (最顶层签名)
  ├── boot.img hash  ← 验证内核是否篡改
  ├── system.img hashtree  ← 验证系统文件是否篡改
  ├── vendor.img hashtree
  └── product.img hashtree
```

### 5.2 核心概念

**vbmeta**：AVB2 的核心是一个名为 `vbmeta` 的小分区，它包含：
- 所有已验证分区的 hash/哈希树根
- 公钥（或公钥的 hash）
- 验证模式：**ENFORCE**（强制验证，无法启动被篡改的系统）、**WARNING**（警告但允许启动）、**LOCKED**（设备锁定状态，禁止刷写）

**链式信任**：
```
出厂熔断的公钥 hash → vbmeta 签名 → boot.img hash → Kernel
                                     → system.img hashtree
                                     → vendor.img hashtree
```

**回滚保护（Rollback Protection）**：
- 每个分区都有一个回滚索引（rollback index）
- 当系统 OTA 升级后，回滚索引递增
- 无法启动回滚索引比当前值低的旧系统——防止攻击者刷回旧版本利用已知漏洞

### 5.3 代码体现

**BoardConfig.mk 中的 AVB 配置**：

```makefile
# device/amlogic/ross/BoardConfig.mk
BOARD_AVB_ENABLE := true                          # 启用 AVB2
BOARD_AVB_ALGORITHM := SHA256_RSA4096             # 签名算法
BOARD_AVB_KEY_PATH := external/avb/test/data/testkey_rsa4096.pem  # 签名密钥

# 各分区的 hashtree 配置
BOARD_AVB_SYSTEM_ADD_HASHTREE_FOOTER_ARGS += --hash_algorithm sha256
BOARD_AVB_VENDOR_ADD_HASHTREE_FOOTER_ARGS += --hash_algorithm sha256
BOARD_AVB_PRODUCT_ADD_HASHTREE_FOOTER_ARGS += --hash_algorithm sha256

# vbmeta_system 包含 system 和 system_ext 的验证信息
BOARD_AVB_VBMETA_SYSTEM := system system_ext system_dlkm
BOARD_AVB_VBMETA_SYSTEM_KEY_PATH := external/avb/test/data/testkey_rsa2048.pem
BOARD_AVB_VBMETA_SYSTEM_ALGORITHM := SHA256_RSA2048
```

**avbtool**：Google 提供的 AVB2 工具，位于 `external/avb/avbtool.py`。它在构建时被调用为每个分区添加 hash footer：

```makefile
# device/amlogic/common/build_kernel_modules.mk
ifeq ($(BOARD_AVB_ENABLE),true)
    $(AVBTOOL) add_hash_footer --partition_name boot ...
endif
```

**U-Boot 中的 AVB 验证**：在 `bootloader/uboot-repo/bl33/v2019/lib/libavb/` 目录中有完整 AVB 验证库实现：

```
lib/libavb/
├── avb_slot_verify.c    # 核心验证逻辑
├── avb_chain_partition_descriptor.c  # 链式分区验证
├── avb_hashtree_descriptor.c         # hashtree 验证
├── avb_hash_descriptor.c             # hash 验证
└── avb_footer.c                      # footer 解析
```

`cmd/amlogic/cmd_bootctl_avb.c` 实现了 Amlogic 定制的启动控制命令，支持 AVB 模式下的 A/B 槽选择。

**内核 cmdline 传递验证状态**：
```makefile
# BoardConfig.mk
BOARD_KERNEL_CMDLINE += androidboot.selinux=permissive
# AVB 启用时内核会收到 androidboot.verifiedbootstate=green 等参数
```

### 5.4 实际价值

AVB2 解决了 Android 设备最根本的安全问题——**系统完整性**。没有 AVB2：
- 攻击者可以刷入篡改过的 system.img，注入后门
- OTA 升级包被替换后无法被检测
- 设备锁定状态无实际意义

在 Android TV 设备上，AVB2 还满足 Google CTS（兼容性测试套件）的 Verified Boot 要求，未经 AVB2 认证的设备无法预装 Google 服务。

---

## 6. VAB (Verified Android Boot)

### 6.1 概念解释

**概念解释**：VAB（Verified Android Boot）是 Amlogic 在 AVB2 基础上增加的增强层，全称是 **Verified Android Boot with A/B slot control**。它不是替代 AVB2，而是在 AVB2 之上增加 A/B 系统更新的启动控制逻辑。

> VAB = AVB2 + A/B 系统更新 + 启动控制

**A/B 系统更新（无缝更新）**：设备维护两个系统槽位（slot A 和 slot B），当前在用的槽位称为 active slot。OTA 更新在**后台槽位**进行，更新完成后重启切换到新槽位。如果新系统启动失败，自动回滚到旧槽位——用户几乎感觉不到更新过程。

### 6.2 VAB 的核心职责

从 `cmd_bootctl_avb.c` 中的枚举可以看出启动模式：

```c
enum BOOT_MODE {
    BOOT_MODE_NORMAL = 1,  // 普通启动，无验证
    BOOT_MODE_AVB = 2,     // AVB2 验证启动
    BOOT_MODE_VAB = 3,     // AVB2 + A/B 启动控制
};
```

VAB 模式在 AVB 验证的基础上增加了：
1. **读取 misc 分区的 A/B 元数据**：确定当前应启动哪个槽位
2. **槽位有效性检查**：`slot_is_bootable_VAB()` 检查指定槽位的 metadata 是否有效
3. **自动回滚**：如果当前槽位启动失败（如 kernel panic），标记该槽位为 unbootable，切换到另一个槽位
4. **AVB 验证 + 槽位选择的组合**：先验证 vbmeta，再用 VAB 选择槽位

### 6.3 代码体现

**构建时启用 VAB**：
```bash
./mk s7d_bm201 --avb2 --vab
```

在 `fip/mk_script.sh` 中：
```bash
# 解析 --vab 参数
--vab)
    CONFIG_CMD_BOOTCTOL_VAB=1
    export CONFIG_CMD_BOOTCTOL_VAB=1
    ;;

# 传递给 U-Boot 编译
build_uboot ${CONFIG_SYSTEM_AS_ROOT} ${CONFIG_AVB2} ${CONFIG_CMD_BOOTCTOL_VAB} \
            ${CONFIG_FASTBOOT_WRITING_CMD} ${CONFIG_AVB2_RECOVERY} ${CONFIG_TESTKEY} \
            ${CONFIG_AB_UPDATE} ${CONFIG_AML_GPT}
```

传入 `make` 的是 `BOOTCTRLMODE=$3` 参数，在 U-Boot 的 Makefile 中转换为 CONFIG 宏。

**misc 分区布局**（来自 `cmd_bootctl_avb.c` 的注释）：

```
misc 分区内存布局:
0    - 2K     Bootloader Message（命令、状态、恢复信息）
2K   - 16K    Bootloader 私有空间（含 A/B 元数据: 2K-4K）
16K  - 32K    Wipe package（恢复模式清数据用）
32K  - 64K    系统空间（AOSP 功能）
```

**VAB 回滚逻辑**（`cmd_bootctl_avb.c`）：

```c
// 加载并验证 A/B 槽位信息
boot_info_load_VAB(&boot_ctrl, miscbuf);

if (!boot_info_validate_VAB(&boot_ctrl)) {
    // 元数据损坏，标记为需要修复
    mark_boot_info_as_unrecoverable(&boot_ctrl);
}

// 获取当前活动槽位
slot = get_active_slot_VAB(&boot_ctrl);

// 检查槽位是否可启动
bootable_a = slot_is_bootable_VAB(&boot_ctrl.slot_info[0]);
bootable_b = slot_is_bootable_VAB(&boot_ctrl.slot_info[1]);
```

### 6.4 VAB vs AVB2 对比

| 特性 | AVB2 | VAB (Amlogic) |
|------|------|---------------|
| 分区完整性验证 | ✅ | ✅（继承 AVB2） |
| 设备锁定/解锁 | ✅ | ✅ |
| 回滚保护 | ✅ | ✅ |
| A/B 槽位切换 | ❌ | ✅ |
| 自动回滚 | ❌ | ✅ |
| misc 分区管理 | ❌ | ✅ |

### 6.5 实际价值

VAB 让 Android TV 设备的系统更新更加可靠。如果没有 VAB：
- OTA 更新过程中断电 → 设备变砖
- 新系统启动失败 → 需要用户手动进入 Recovery 模式恢复
- 无法实现"静默更新、重启生效"的无缝体验

A/B 系统（由 VAB 控制）是 Android TV 设备满足质量要求的关键特性。

---

## 7. Fastboot 模式

**概念解释**：Fastboot 是 Android 设备的刷机协议，允许用户通过 USB 连接 PC，执行烧录分区、解锁设备、重启等操作。fastboot 运行在 bootloader 阶段（U-Boot 中），不依赖 Android 系统。

**代码体现**：U-Boot 编译时通过 `--fastboot-write` 参数启用 fastboot 命令支持：

```bash
# build_uboot 参数
build_uboot ${CONFIG_SYSTEM_AS_ROOT} ${CONFIG_AVB2} ${CONFIG_CMD_BOOTCTOL_VAB} \
            ${CONFIG_FASTBOOT_WRITING_CMD} ...
```

**实际流程**：
```
PC: fastboot flash boot boot.img
               ↓ USB
U-Boot fastboot: 接收镜像数据
               ↓
U-Boot: 写入 boot 分区
               ↓
U-Boot: AVB2 验证镜像签名（可选）
```

**实际价值**：Fastboot 是 Android 开发调试的核心工具。没有它，烧录系统需要拆机使用串口/USB 烧录工具，开发效率极低。量产时也可以通过 fastboot OEM 命令锁定设备。

---

## 8. Amlogic 签名和打包体系

### 8.1 签名流程

根据 `fip/s7d/build.sh`（读取完整文件后的理解），S905X5M 的签名流程分为两层：

**芯片级签名（CS，Chipset Signing）**：
- 使用 RSA 算法对 BL2/BL2E/BL2X 签名
- 签名后生成 `bb1st.bin.rsa.rsa` 等文件
- 由 Boot ROM 使用芯片公钥验证

**设备级签名（DV，Device Signing）**：
- 使用设备特定密钥对 FIP 头部签名
- 生成 `device-fip-header.bin.rsa.rsa`
- 由 BL2 验证

### 8.2 构建产物

```
build/u-boot.bin                ← 原始 U-Boot 镜像
build/u-boot.bin.sd.bin         ← SD 卡启动镜像
build/u-boot.bin.sd.bin.signed  ← SD 卡启动签名镜像
build/u-boot.bin.usb.signed     ← USB 烧录签名镜像
build/boot.img.encrypt          ← 加密 boot 镜像（可选）
```

这些产物被复制到 `device/amlogic/ross/upgrade/` 目录，作为最终固件打包的输入。

---

## 9. 与 Android 构建系统的集成

Bootloader 的构建虽然独立于 AOSP 的 Soong/Makefile 构建系统，但通过以下方式集成：

### 9.1 独立构建命令

```bash
# 在 uboot-repo 目录中单独构建
cd bootloader/uboot-repo
./mk s7d_bm201 --update-bl2 --update-bl2e --update-bl2x --update-bl31 --update-bl32 --avb2 --vab
```

### 9.2 集成到完整构建

```
./mk ross -v common14-5.15
```

当 `TARGET_NO_BOOTLOADER := false`（BoardConfig.mk 第 43 行）时，构建系统会自动调用 uboot-repo 的构建流程。

### 9.3 固件打包

构建完成后，bootloader 被包含在最终烧录包 `aml_upgrade_package.img` 中。打包配置参考 `device/amlogic/common/factory.mk`。

---

## 10. 流程图汇总

### Amlogic S905X5M 完整启动时序

```
上电
  │
  ▼
Boot ROM (芯片固化)
  │ 1. 加载 BL2 到 SRAM
  │ 2. 验证 BL2 签名 (芯片公钥)
  │ 3. 跳转到 BL2
  │
  ▼
BL2 (SRAM 中运行)
  │ 1. 初始化 DDR 内存
  │ 2. 从 FIP 包中加载 BL30/BL31/BL32/BL33 到 DDR
  │ 3. 验证各阶段签名
  │ 4. 跳转到 BL31
  │
  ▼
BL31 (ATF, Secure World)
  │ 1. 初始化 SMC 处理
  │ 2. 启动 BL32 (TEE, 可选)
  │ 3. 跳转到 BL33 (切换到 Normal World)
  │
  ▼
BL33 (U-Boot, Normal World)
  │ 1. 初始化存储、显示、网络等外设
  │ 2. [AVB2 验证] 读取 vbmeta → 验证 boot.img
  │ 3. [VAB 控制] 选择 A/B 槽位
  │ 4. 将 kernel + dtb + ramdisk 加载到内存
  │ 5. 跳转到 Kernel
  │
  ▼
Linux Kernel
  │ 1. 解压并初始化
  │ 2. 挂载 rootfs (ramdisk)
  │ 3. 启动 init 进程
  │
  ▼
init → Android 系统
```

### AVB2 验证链扩展

```
┌────────────────────────────────────────────────────────────────┐
│                        Boot ROM                                │
│  熔断的公钥 hash → 验证 BL2 签名                                  │
└────────────────────┬───────────────────────────────────────────┘
                     │ 信任传递
                     ▼
┌────────────────────────────────────────────────────────────────┐
│                    BL2 → BL31 → BL33 (U-Boot)                  │
│  每级固件都验证下一级的签名，确保启动链未被篡改                       │
└────────────────────┬───────────────────────────────────────────┘
                     │ 信任传递  
                     ▼
┌────────────────────────────────────────────────────────────────┐
│  U-Boot 执行 AVB2 验证                                          │
│                                                                │
│  vbmeta (用 OEM 公钥签名)                                       │
│   ├── boot.img hash    ← 使用 vbmeta 中记录的 hash 值              │
│   ├── system.img hashtree                                        │
│   ├── vendor.img hashtree                                        │
│   └── product.img hashtree                                       │
│                                                                │
│  vbmeta_system (用 system 密钥签名)                               │
│   ├── system_ext.img hashtree                                    │
│   └── system_dlkm.img hashtree                                   │
└────────────────────────────────────────────────────────────────┘
```

---

## 11. 关键代码文件索引

| 功能 | 路径 |
|------|------|
| U-Boot 构建入口 | `bootloader/uboot-repo/fip/mk_script.sh` |
| U-Boot 源码 (v2019) | `bootloader/uboot-repo/bl33/v2019/` |
| BL2 源码 | `bootloader/uboot-repo/bl2/` |
| BL31 (ATF v1.3) | `bootloader/uboot-repo/bl31/bl31_1.3/` |
| BL31 (ATF v2.7) | `bootloader/uboot-repo/bl31/bl31_2.7/` |
| S7D SoC 构建脚本 | `bootloader/uboot-repo/fip/s7d/build.sh` |
| AVB 验证库 | `bootloader/uboot-repo/bl33/v2019/lib/libavb/` |
| AVB U-Boot 命令 | `bootloader/uboot-repo/bl33/v2019/cmd/avb.c` |
| VAB 启动控制 | `bootloader/uboot-repo/bl33/v2019/cmd/amlogic/cmd_bootctl_avb.c` |
| AVB 配置 (BoardConfig) | `device/amlogic/ross/BoardConfig.mk` |
| avbtool | `external/avb/avbtool.py` |
| 签名测试密钥 | `external/avb/test/data/` |
| 设备升级配置 | `device/amlogic/ross/upgrade/platform.conf` |

---

## 12. 自问自答

> **Q**: U-Boot 和 Boot ROM 什么关系？
> **A**: Boot ROM 是芯片出厂固化的代码，不可修改。它负责初始化基本硬件（如 SRAM）并加载 BL2。U-Boot（BL33）在启动链的末端，是最后一个 bootloader 阶段，负责加载操作系统。Boot ROM 只信任 BL2，BL2 信任 BL31/BL33，形成信任链。

> **Q**: AVB2 能防什么攻击？
> **A**: 防"离线篡改"——攻击者拆机读取 eMMC，修改 system.img 加入后门，再写回去。AVB2 的 hashtree 验证会发现根 hash 不匹配，拒绝启动。但 AVB2 不防"在线攻击"（设备运行时通过漏洞注入），那是 SELinux 和沙箱的职责。

> **Q**: VAB 和 AVB2 是不是二选一？
> **A**: 不是。VAB 依赖 AVB2。AVB2 负责"验证镜像是否被篡改"，VAB 负责"在 AVB2 验证通过的基础上，决定启动哪个槽位"。可以只有 AVB2 没有 VAB，但不能只有 VAB 没有 AVB2。

> **Q**: A/B 系统怎么做到无缝更新？
> **A**: 假设设备运行在 slot A。OTA 下载新系统到 slot B，安装完成后重启。U-Boot 检查到 slot B 标记为"更新完成"，切换 active slot 到 B，启动新系统。如果 B 无法启动，VAB 标记 B 为失败，自动回滚到 A。整个过程除了重启一刻，用户不需要等待安装。

> **Q**: Fastboot flash 为什么设备要解锁？
> **A**: 解锁设备后，U-Boot 在验证 vbmeta 时发现设备是 unlocked 状态，会允许写入并启动未签名的镜像。锁定状态下，U-Boot 拒绝写入受保护分区（如 boot、system），也拒绝启动签名校验失败的镜像。解锁动作通常需要用户确认并清除所有数据。

> **Q**: 开发时可以用 testkey 吗？
> **A**: 可以。`external/avb/test/data/testkey_rsa4096.pem` 是 AVB2 的测试密钥。但测试密钥公开在 AOSP 源码中，任何人都可以用它签名镜像。因此使用测试密钥的设备只能用于开发调试，不能出厂。量产时必须使用厂商私有的签名密钥。

---

## 13. 下一步

理解 bootloader 后，建议继续深入：
- [ ] **Kernel 构建与设备树**（`notes/08-kernel-dts.md`）：理解内核如何通过设备树匹配硬件
- [ ] **HAL 与 Treble**（`notes/09-hal-treble-hidl.md`）：理解内核之上 Android 如何抽象硬件
- [ ] **SELinux**（`notes/10-selinux-android.md`）：理解设备启动后的安全策略实施
