+++
title = 'Level 1：确认板子是活的'
date = '2026-06-09T16:02:00+08:00'
draft = false
+++

# Level 1：确认板子是活的

## 我想做

拿到一块 S905X5M 板子，通电，确认它能正常工作，搞清楚"我手里有什么"。

## 先回答三个问题

1. **板子需要哪些物理连接才能工作？**
   - 电源（看板子丝印，12V/5V？DC 口还是接线柱？）
   - 串口（TX/RX/GND 三根线，接 TTL-USB 转接板）
   - 显示（HDMI 接显示器）
   - 网络或 USB（用来 adb 连接）
   - 找出你的板子上每个接口的位置

2. **怎么判断板子"活着"？**
   - 串口有输出（bootloader 启动日志）
   - HDMI 有画面（Android 启动动画或桌面）
   - adb 能连上（`adb devices` 看到设备）

3. **什么是 Android 的板级信息？**
   - `ro.board.platform`——芯片平台代号（S905X5M 是什么？）
   - `ro.build.version.release`——Android 版本号
   - `ro.build.fingerprint`——完整构建指纹（包含厂商/产品/版本/编译号）
   - `ro.build.date`——固件编译时间
   - `/proc/version`——内核版本和编译工具链
   - 这些信息由谁写入的？→ build.prop 文件

## 需要懂的知识

**adb 基础命令**

adb（Android Debug Bridge）是 Android 调试的核心工具，三个组件：adb client（你输入命令的终端）、adb server（后台进程）、adbd（板子上跑的守护进程）。只需要记住：
- `adb shell <cmd>` ——在板子上执行命令
- `adb devices` ——查看连接的设备列表
- `adb push/pull` ——传文件

**Linux 虚拟文件系统**

- `/proc/` ——内核暴露的运行信息（进程、内存、版本），不是真实文件
- `/sys/` ——内核暴露的设备参数，可以读写来配置驱动
- `getprop` ——读取 Android 系统的属性，底层就是读取 `/` 下的属性文件

**分区概念**

板子上的存储（eMMC）被划分为多个分区，每个分区有独立用途：
- `boot` ——内核 + ramdisk
- `super` ——system/vendor/product 的合并分区（Android 10+）
- `vendor` ——厂商专有文件和 HAL
- `dtb/dtbo` ——设备树
这些分区在 `/dev/block/` 下可见。

## 动手方案

```bash
# 第 1 步：接串口线
# 板子上找标注了 TX/RX/GND 的排针，接 TTL-USB 转接板（交叉连接：TX→RX，RX→TX）
# 波特率 115200，用 screen/minicom/putty 打开

# 第 2 步：上电，串口会输出 bootloader 日志
# 注意看有没有类似下面的输出：
#   G12A:BL:...
#   DDR init complete
#   U-Boot ...

# 第 3 步：等板子启动完成，接网络或 USB，adb 连接
adb devices
# 如果显示 "unauthorized"，板子上点允许

# 第 4 步：收集板子信息（记录到笔记里）
adb shell getprop ro.board.platform
adb shell getprop ro.build.version.release
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.date
adb shell cat /proc/version
adb shell cat /proc/cpuinfo | head -20

# 第 5 步：看分区布局
adb shell cat /proc/partitions
adb shell df -h

# 第 6 步：找到你的源码里的板级目录
ls ~/android/aml/s905x5/aml-s905x5-androidu-v2/device/amlogic/ross/

# 第 7 步：找设备树文件
find ~/android/aml/s905x5/aml-s905x5-androidu-v2/ -name "*.dts" -path "*s905*" 2>/dev/null | head -5
```

## 验收清单

| 项目 | 验证方式 |
|------|----------|
| 串口有完整启动日志 | 看到 ROM Code→DDR→U-Boot→kernel 输出 |
| adb 能连接 | `adb shell` 进入板子终端 |
| 记录了板子的 Android 版本和内核版本 | 笔记中有 `getprop` 和 `/proc/version` 输出 |
| 找到板级配置目录 | `device/amlogic/ross/` 确认存在 |
| 找到至少一个 dts 文件 | `find` 返回了有效路径 |
