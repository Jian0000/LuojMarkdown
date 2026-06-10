+++
title = 'Android 嵌入式全栈学习计划'
date = '2026-06-09T18:00:00+08:00'
draft = false
+++

# Android 嵌入式全栈学习计划

> **背景**: 有 Linux embedded C/C++ 基础（浅层），接触过 BSP API 适配
> **目标**: 掌握 Android 嵌入式全栈开发，能独立完成从底层到应用层的系统定制
> **参考代码库**: `~/android/aml/s905x5/aml-s905x5-androidu-v2/`（Android 14 + Amlogic S905X5M ATV）

---

## 总览

```
Phase 1 ─ 筑基 ──────────────────────────────── 2-3 周
  构建系统入门 + AOSP 架构 + 基本 Java

Phase 2 ─ 厂商定制层 ──────────────────────────── 3-4 周
  从 vendor/giec/ 读懂"厂商如何改系统"

Phase 3 ─ 深入底层 ───────────────────────────── 4-6 周
  Bootloader → Kernel → HAL → SELinux

Phase 4 ─ Framework 与应用 ────────────────────── 4-6 周
  系统服务 → 应用开发 → OTA
```

---

## Phase 1：筑基（预计 2-3 周）

### 1.1 理解 Android 系统架构

目标：搞懂 Android 系统的分层结构，建立整体地图。

**TODO:**
- [ ] 阅读 [Android 架构图](https://source.android.com/docs/core/architecture) 官方文档
- [ ] 理清分层：App → Framework → 系统服务 → HAL → Kernel
- [ ] 理解分区：bootloader / boot / system / vendor / product / data / cache

**输出笔记**: `notes/01-android-架构概览.md`

### 1.2 Android 构建系统入门

目标：能读懂 Android.bp 和 Android.mk，知道一个模块如何被编译进系统。

**TODO:**
- [ ] 学习 Soong (Android.bp) 基本语法：`cc_binary`、`cc_library`、`android_app`、`cc_defaults`
- [ ] 学习 Android.mk 基本语法：`include $(CLEAR_VARS)` / `LOCAL_MODULE` / `LOCAL_SRC_FILES` / `include $(BUILD_...)`
- [ ] 理解 `PRODUCT_PACKAGES` 如何把模块打包进镜像
- [ ] 理解 `$(call inherit-product, ...)` 的包含链
- [ ] **结合参考代码库**：打开 `build/make/core/main.mk` 入口，宏观理解构建流程

**输出笔记**: `notes/02-android-构建系统.md`
**实操**: 在参考代码库中找 1 个 cc_binary 和 1 个 android_app 的 Android.bp，读通它。

### 1.3 Java 基础（够用就行）

目标：能读懂 Android Framework 和 APK 源码，不要求能写复杂应用。

**TODO:**
- [ ] Java 基本语法：类、继承、接口、抽象类
- [ ] 理解 `package` / `import` 机制
- [ ] 异常处理：`try-catch`、`throws`
- [ ] Android 特有的 Java 用法：`Context`、`Intent`、`Activity`、`Service`、`BroadcastReceiver`

**不建议**: 啃完整 Java 教程。遇到代码时查，配合 IDE 提示学习最快。

**输出笔记**: `notes/03-java-速查表.md`（自己整理的关键语法速查）

---

## Phase 2：厂商定制层（预计 3-4 周）

这是参考代码库最有价值的部分。绕过 AOSP 大海量的通用代码，直接看厂商写了什么。

### 2.1 从 device-giec.mk 出发

目标：把 `device-giec.mk` 中每一行提到的文件都找到并理解其作用。

**TODO:**
- [ ] 读通 `vendor/giec/device-giec.mk`（已完成）
- [ ] 逐行追踪包含的每个文件和模块：
  - [ ] `vendor/giec/common/sepolicy/` — SELinux 策略
  - [ ] `vendor/giec/common/manifest.xml` — HIDL 服务声明
  - [ ] `vendor/giec/common/compatibility_matrix.xml` — Framework 需求声明
  - [ ] `vendor/giec/common/hwstbcmdservice/` — HIDL 服务实现
  - [ ] `vendor/giec/java-libs/shellCmd.jar` — 系统应用特权库

**输出笔记**: `notes/04-厂商定制层.md`

### 2.2 预置 APK 分析

目标：理解厂商预置应用的代码结构和构建方式。

**TODO:**
- [ ] 阅读 `vendor/giec/apps/OTAClient/` — OTA 升级客户端（涉及网络请求、系统更新流程）
- [ ] 阅读 `vendor/giec/apps/BazeportLauncher/` — 定制 Launcher
- [ ] 了解 APK 签名机制：testkey vs release key

**输出笔记**: `notes/05-预置APK分析.md`

### 2.3 设备配置

目标：理解一个 device 的完整配置。

**TODO:**
- [ ] 选择 `device/amlogic/ross/`（S905X5M）作为分析对象
- [ ] 读通它的 `device.mk` / `BoardConfig.mk` / `AndroidBoard.mk`
- [ ] 理解它如何选择 kernel、uboot、分区大小、硬件特性

**输出笔记**: `notes/06-设备配置.md`

---

## Phase 3：深入底层（预计 4-6 周）

这是你 C/C++ 背景能发挥的阶段。

### 3.1 Bootloader：U-Boot + 安全启动

**TODO:**
- [ ] 阅读 `bootloader/uboot-repo` 构建命令：`./mk s6_bl201 --vab --avb2`
- [ ] 理解 VAB (Verified Android Boot) + AVB2 (Android Verified Boot 2.0)
- [ ] 理解 U-Boot 如何加载 boot.img 并校验签名
- [ ] 了解 fastboot 模式

**输出笔记**: `notes/07-bootloader-uboot-安全启动.md`

### 3.2 Kernel 构建与设备树

**TODO:**
- [ ] 理解 `./mk ross -v common14-5.15` — Kernel 版本选择与构建
- [ ] 找到设备树文件：`device/amlogic/ross/` 下的 DTS/DTSI
- [ ] 对比 Linux 主线 DTS 和 Android 设备树的差异

**输出笔记**: `notes/08-kernel-设备树.md`

### 3.3 HAL 与 Treble

**TODO:**
- [ ] 理解 HIDL：接口定义语言和代码生成
- [ ] 分析 `vendor/giec/hardware/interfaces/` 的 HIDL 定义
- [ ] 理解 hwservicemanager、服务注册与客户端获取服务的完整流程
- [ ] 对比：传统 Linux 驱动 ioctl 方式 vs Android HIDL 方式

**输出笔记**: `notes/09-hal-treble-hidl接口.md`

### 3.4 SELinux

**TODO:**
- [ ] 读通 `vendor/giec/common/sepolicy/` 下的所有 .te 文件
- [ ] 理解 type、allow、domain、context 等核心概念
- [ ] 理解 Android 独有的 sepolicy 语法（`file_type`、`domain`、`typeattribute`）
- [ ] 实操：尝试加一条 SELinux 规则并编译验证

**输出笔记**: `notes/10-selinux-安全策略.md`

---

## Phase 4：Framework 与应用（预计 4-6 周）

### 4.1 Android Framework 核心服务

**TODO:**
- [ ] 理解 System Server 启动流程（`frameworks/base/services/java/com/android/server/SystemServer.java`）
- [ ] 了解核心服务：ActivityManagerService、PackageManagerService、WindowManagerService
- [ ] 理解 Binder IPC 机制（对标 Linux D-Bus）

**输出笔记**: `notes/11-framework-核心服务.md`

### 4.2 Android TV 特性

**TODO:**
- [ ] 了解 TV Input Framework (TIF)
- [ ] 了解 TV 特有的 UI/UX 设计（Leanback UI）
- [ ] 理解参考代码库中如何集成 MindTheGapps

**输出笔记**: `notes/12-android-tv-特性.md`

### 4.3 OTA 升级系统

**TODO:**
- [ ] 分析 OTA 升级包的生成流程（`./build.sh -o`）
- [ ] 分析 `vendor/giec/apps/OTAClient/` 的升级逻辑
- [ ] 理解 Recovery 模式（`bootable/recovery/`）

**输出笔记**: `notes/13-ota-升级系统.md`

### 4.4 系统应用开发入门

**TODO:**
- [ ] 用 Java 写一个简单的系统应用（需签名平台 key）
- [ ] 通过 `shellCmd.jar` 调用系统级命令
- [ ] 编译进系统并刷机验证

**输出笔记**: `notes/14-系统应用开发.md`

---

## 学习方式建议

### 读代码原则

1. **先粗后细**：先看 Makefile / 目录结构，再深入具体源码
2. **追踪调用链**：看到一个函数调用，如 `some_func()`，按住 Ctrl 点进去看实现
3. **修改验证**：光读不强，改一行代码 → 编译 → 刷机验证，印象最深

### 笔记策略

- 每个模块输出一篇自己的理解笔记，用大白话写（这是检验真懂的标志）
- 画图：架构图、调用时序图、分区布局图
- 记录踩坑：编译错误、SELinux denials、分区空间不足

### 遇到问题时的排查顺序

```
编译错误 → 看构建系统的错误输出，搜相同问题
运行时错 → logcat 看日志
权限拒绝 → dmesg | grep avc 看 SELinux 报错
功能异常 → 逐层排查：App → Service → HAL → Kernel
```

---

## 最终成果预期

完成此计划后，你应该能：

1. ✅ 独立完成一个 Android 设备的 BSP 适配（kernel + uboot + device 配置）
2. ✅ 添加新的硬件 HIDL 服务并编写 sepolicy 规则
3. ✅ 预置/开发系统应用，理解签名和权限机制
4. ✅ 执行完整构建并生成 OTA 包
5. ✅ 看懂 AOSP 大部分代码结构，能定位问题所在层

---

> **提示**: 这是一个参考计划，每个阶段的时长可根据自己的节奏调整。关键是**每一步都要有笔记输出**，写下来才是自己的。
