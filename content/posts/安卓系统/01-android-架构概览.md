+++
title = '01-Android系统架构总览'
date = '2026-06-09T18:01:00+08:00'
draft = false
+++

# Android 系统架构总览

> 学习日期: 2026-04-28
> 参考项目: Amlogic S905X5M (ross) Android 14 TV
> 前置知识: Linux embedded C/C++ 基础

---

## 1. Android 分层架构

Android 系统从上到下分为 5 层，每一层都依赖下一层提供的服务：

```
 ┌─────────────────────────────────────────────────┐
 │                  Applications                   │  ← 系统应用 + 第三方应用
 │   Launcher, Settings, Browser, OTA Client...    │
 ├─────────────────────────────────────────────────┤
 │              Application Framework              │  ← 应用框架（Java）
 │   ActivityManager, WindowManager, PackageManager│
 ├─────────────────────────────────────────────────┤
 │              Android System Services            │  ← 系统服务（Native + Java）
 │   System Server, SurfaceFlinger, MediaServer... │
 ├─────────────────────────────────────────────────┤
 │         Hardware Abstraction Layer (HAL)        │  ← 硬件抽象层（C/C++）
 │   Camera HAL, Audio HAL, Graphics HAL...        │
 ├─────────────────────────────────────────────────┤
 │              Linux Kernel                       │  ← Linux 内核 + 驱动
 │   Display Driver, WiFi, BT, Audio, DMA-BUF...   │
 └─────────────────────────────────────────────────┘
```

### 与传统 Linux embedded 的对比

| 概念 | 传统 Linux | Android |
|------|-----------|---------|
| 内核 | Linux kernel | **同一内核** + Android 特有驱动 (ashmem/binder/ion) |
| init 系统 | systemd/busybox init | **init**（专属，解析 .rc 文件） |
| IPC | D-Bus/socket | **Binder**（Android 核心 IPC 机制） |
| 设备访问 | 直接 open /dev/xxx | 通过 **HAL** + HIDL 接口 |
| 应用格式 | ELF 可执行文件 | **APK**（Java + Native libs） |
| 权限控制 | DAC (user/group) | **DAC + SELinux**（强制） |
| 构建系统 | Makefile/CMake | **Soong (Android.bp)** + Makefile |

---

## 2. 各层详解（结合参考代码库）

### 2.1 Layer 1: Linux Kernel

**位置**: kernel 源码 + 设备树 DTS

**职责**: 硬件驱动、内存管理、进程调度、网络栈

**Android 特有的内核特性**:

- **Binder 驱动**: 进程间通信（IPC）的核心
- **ashmem**: 匿名共享内存
- **ION/DMA-BUF**: 内存分配器（图形/多媒体用）
- **wakelocks**: 电源管理休眠锁

**参考代码库中的体现**:

```bash
# BoardConfig.mk 中定义了内核版本和构建方式
# device/amlogic/ross/BoardConfig.mk:
TARGET_BOARD_PLATFORM := s6
# 内核通过 ./mk ross -v common14-5.15 构建（ross 的 uboot target: s7d_bm201）
# 内核 cmdline 中配置 console、SELinux、分区信息:
BOARD_KERNEL_CMDLINE += console=ttyS0,921600
BOARD_KERNEL_CMDLINE += androidboot.selinux=permissive  # 条件开启
BOARD_BOOTCONFIG += "androidboot.boot_devices=soc/fe08c000.mmc"
```

---

### 2.2 Layer 2: Hardware Abstraction Layer (HAL)

**位置**: `hardware/`、`vendor/`、`device/` 下的 HAL 实现

**职责**: 封装内核驱动接口，向上层提供标准化的硬件访问 API

**关键概念 — Treble 架构**:
Android 8.0 引入 Project Treble，将 HAL 从 Framework 中解耦出来。厂商实现 HAL，Framework 通过 **HIDL**（HAL Interface Definition Language）调用 HAL 服务。

**通信流程**:

```c++
App/Service
    ↓ Binder IPC
Framework (Java)
    ↓ JNI
Native Service (C++)
    ↓ HIDL/hwBinder
HAL Implementation (C/C++)    ← 厂商实现
    ↓ system calls
Kernel Driver
    ↓
Hardware
```

**参考代码库中的体现**:

```makefile
# device/giec/device-giec.mk 中的 HIDL 声明:
DEVICE_MANIFEST_FILE += vendor/giec/common/manifest.xml
DEVICE_MATRIX_FILE += vendor/giec/common/compatibility_matrix.xml

# vendor/giec 的自定义 HAL 服务:
PRODUCT_PACKAGES += \
  giec.hardware.hwstbcmdservice@1.0 \      # HIDL 接口定义
  giec.hardware.hwstbcmdservice@1.0-impl \  # HAL 实现
  giec.hardware.hwstbcmdservice@1.0-service # HAL 服务进程
```

manifest.xml 声明"我这个设备提供了哪些 HAL 服务"，compatibility_matrix.xml 声明"我需要 Framework 给我提供什么"。

**ross 设备的外设配置**:
```makefile
# device/amlogic/ross/BoardConfig.mk
USE_OPENGL_RENDERER := true
USE_HWC2 := true                     # 使用 HWC2 (Hardware Composer 2)
BOARD_HAVE_FRONT_CAM := false        # 无前置摄像头
BOARD_HAVE_BACK_CAM := false         # 无后置摄像头
BOARD_USE_USB_CAMERA := true         # 使用 USB 摄像头
BOARD_USES_ALSA_AUDIO := true        # ALSA 音频架构
```

---

### 2.3 Layer 3: System Services (Native 层)

**位置**: `frameworks/native/`、`system/core/`

**关键进程**:

| 进程 | 作用 | 类比 Linux |
|------|------|-----------|
| **init** | PID 1，解析 init.rc 启动所有服务 | 类似 systemd |
| **surfaceflinger** | 图形合成显示 | 类似 Weston/Wayland compositor |
| **mediaserver** | 多媒体编解码、播放 | 类似 GStreamer 管道 |
| **servicemanager** | Binder 服务注册与发现 | 类似 D-Bus daemon |
| **logd** | 系统日志 | 类似 syslogd |

**init 启动流程**:
```
Boot ROM → Bootloader → Kernel → init → init.rc → Zygote → System Server
                                                           ↓
                                                   所有 Java 系统服务
```

**参考代码库**:

```makefile
# device/amlogic/ross/device.mk
# init 启动脚本:
PRODUCT_COPY_FILES += \
    device/amlogic/ross/init.amlogic.board.rc:$(TARGET_COPY_OUT_VENDOR)/etc/init/hw/init.amlogic.board.rc
```

---

### 2.4 Layer 4: Application Framework (Java 层)

**位置**: `frameworks/base/`

**核心服务（运行在 System Server 进程中）**:

| 服务 | 作用 |
|------|------|
| **ActivityManagerService (AMS)** | 管理应用生命周期、任务栈 |
| **WindowManagerService (WMS)** | 窗口管理、触摸事件分发 |
| **PackageManagerService (PMS)** | APK 安装、权限管理 |
| **PowerManagerService** | 电源管理、休眠策略 |
| **NotificationManagerService** | 通知管理 |

App 通过 Binder IPC 调用这些服务，不需要知道底层实现。

---

### 2.5 Layer 5: Applications

**位置**: `packages/`、`vendor/giec/apps/`

分为两类：
- **系统应用**: 预置在 system 分区，有系统权限
- **用户应用**: 用户安装，权限受限

**参考代码库中的系统应用**:
```makefile
# vendor/giec/device-giec.mk
PRODUCT_PACKAGES += \
    LeanKeyboard \        # 定制输入法
    OTAClient \           # OTA 升级客户端
    BazeportLauncher \    # 定制桌面 Launcher
    BazeportSystem        # 系统工具应用
```

系统应用通过平台 key 签名可以获得 `system` 级别的权限，这是厂商定制的常用手段。

---

## 3. 分区架构

这是 embedded Android 特有的概念。固件烧录到设备后，存储被划分为多个分区：

### Ross (S905X5M) 的分区布局

从 `BoardConfig.mk` 中提取的关键分区信息：

| 分区 | 大小 | 文件系统 | 内容 |
|------|------|---------|------|
| **bootloader** | - | raw | U-Boot（AVB2 + VAB 签名） |
| **boot** | 64MB (67108864) | raw | kernel + ramdisk |
| **dtb** | 252KB (258048) | raw | 设备树 blob |
| **dtbo** | 2MB (2097152) | raw | 设备树 overlay |
| **vendor_boot** | 64MB (67108864) | raw | vendor 内核模块 + DTB |
| **super** | 3.2GB (3355443200) | 逻辑卷 | 包含以下动态分区 |
| ├ system | - | erofs | Android 系统镜像 |
| ├ vendor | - | erofs | 厂商专有文件 |
| ├ product | - | erofs | 产品特定配置 |
| ├ system_ext | - | erofs | 系统扩展 |
| ├ odm | 16MB (16777216) | ext4 | 设备 OEM 定制 |
| └ vendor_dlkm | - | erofs | 可动态加载的内核模块 |
| **userdata** | 550MB (576716800) | f2fs | 用户数据（应用、设置） |
| **metadata** | - | - | 加密 metadata |

> 注: Android 10+ 使用**动态分区**（super 分区），system/vendor/product 不再是物理分区，而是 super 分区内的逻辑卷。这可以灵活调整各分区大小。

### 分区挂载流程

```
Boot ROM
  ↓ 读取启动设备
Bootloader (U-Boot)
  ↓ 验证 vbmeta → 验证 boot.img
  ↓ 加载 kernel + ramdisk 到内存
Kernel
  ↓ init 进程启动
  ↓ 解析 fstab 挂载分区
  ├── mount /system      (erofs, 只读)
  ├── mount /vendor      (erofs, 只读)
  ├── mount /product     (erofs, 只读)
  └── mount /data        (f2fs, 读写)
```

**参考代码库中的 fstab**:
```
device/amlogic/ross/fstab.ab_oem.amlogic
```

---

## 4. 启动流程完整时序

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐    ┌─────────┐
│ Boot ROM │───→│ U-Boot   │───→│ Kernel   │───→│ init    │───→| Zygote  │
│ (芯片固件) │    │ (启动加载器) │    │ (Linux)  │    │ (PID 1) │    │ (Java VM)│
└──────────┘    └──────────┘    └──────────┘    └─────────┘    └─────────┘
                                                     │              │
                                                     ↓              ↓
                                                 init.rc       System Server
                                                     │              │
                                                     ↓              ↓
                                                Vendor Init    AMS/PMS/WMS...
                                                (init.amlogic.   │
                                                 board.rc)       ↓
                                                              Launcher
                                                              (桌面启动)
```

**关键说明**:
1. **Boot ROM**: 芯片出厂固化的代码，上电后从 eMMC/NAND 加载 bootloader
2. **U-Boot**: 初始化硬件(ddr、时钟)、验证启动镜像签名(AVB2)、加载 kernel
3. **Kernel**: 启动内核、初始化驱动、挂载根文件系统
4. **init**: PID 1，执行 init.rc 脚本启动核心服务
5. **Zygote**: Android Java 虚拟机进程，所有 Java 应用的父进程
6. **System Server**: 启动所有 Java 系统服务
7. **Launcher**: 桌面应用启动，用户看到 UI

---

## 5. 代码库目录结构对照

| 目录 | 对应架构层 | 说明 |
|------|-----------|------|
| `bootloader/uboot-repo/` | Bootloader | U-Boot 源码 |
| `kernel/` | Kernel | Linux 内核 |
| `hardware/amlogic/` | HAL | Amlogic 硬件抽象层（gralloc, hwcomposer, camera 等） |
| `hardware/interfaces/` | HAL | HIDL 接口定义 |
| `device/amlogic/` | HAL + 设备配置 | 设备树、BoardConfig、init rc |
| `vendor/giec/` | 各层 | 厂商全栈定制（HAL 服务、SELinux、APK） |
| `frameworks/base/` | Framework | Java Framework、系统服务 |
| `frameworks/native/` | Framework + Native | Native 服务、SurfaceFlinger |
| `packages/` | 应用 | 系统应用 |
| `system/core/` | 核心工具 | init、adbd、logd |
| `build/make/soong/` | 构建系统 | Android.bp / Makefile 构建规则 |

---

## 6. 自问自答（检验理解）

> **Q**: Android 和传统 Linux embedded 的主要区别是什么？
> **A**: 主要是框架层。底层还是 Linux 内核，但上层套了 Java Framework、HAL 隔离层、Binder IPC、SELinux、独特的构建系统。对嵌入式开发者来说，从 kernel 一路往上看到 HAL 层是平滑过渡的。

> **Q**: Project Treble 解决了什么问题？
> **A**: 以前厂商改 HAL 必须连 Framework 一起更新，导致 Android 版本升级困难。Treble 用 HIDL 把 HAL 标准化、独立化，Framework 和厂商 HAL 可以分别更新。

> **Q**: super 分区和传统分区有什么不同？
> **A**: 以前 system、vendor 都是独立的物理分区，大小固定不能改。Android 10+ 用 super 动态分区，把这些分区变成 super 内的逻辑卷，空间利用率更高，OTA 更新也更灵活。

> **Q**: boot.img 里有什么？
> **A**: 包含 kernel（Linux 内核）+ ramdisk（init 程序和挂载脚本）。Android 13+ 还引入了 init_boot.img 把 generic ramdisk 独立出来，实现 GKI（通用内核镜像）。

---

## 7. 下一步

理解架构后，下一步建议进入 Phase 1.2: **构建系统入门**，了解 Android 如何用 Android.bp/Android.mk 组织代码编译。

如果对某层特别感兴趣，也可以深入：
- [ ] Launcher/APK 层 → 看 `vendor/giec/apps/BazeportLauncher/`
- [ ] HAL 层 → 看 `hardware/amlogic/hwcomposer/` 或 `vendor/giec/hardware/interfaces/`
- [ ] SELinux → 看 `vendor/giec/common/sepolicy/`
- [ ] init 启动 → 看 `device/amlogic/ross/init.amlogic.board.rc`
