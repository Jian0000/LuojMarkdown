+++
title = 'Level 7：追踪完整启动流程'
date = '2026-06-09T14:01:00+08:00'
draft = false
+++

# Level 7：追踪完整启动流程

## 我想做

从上电到 Launcher，完整追踪一次 S905X5M 的启动过程，每个阶段都能在代码中找到对应位置。

## 先回答三个问题

1. **Amlogic 的启动链和通用 Linux 有什么不同？**
   - 通用嵌入式：ROM Code → Bootloader → Kernel → init
   - Amlogic：ROM Code → BL2（DDR 初始化） → BL30/SPL → U-Boot → Kernel → init
   - 多出来的 BL2 和 BL30 是 Amlogic 私有的二进制固件

2. **ROM Code 怎么找到下一阶段的代码？**
   - 芯片上电后，内部 ROM 中的固化代码执行
   - 根据 boot mode 引脚的电平（eMMC/SD 卡/USB burning），去对应介质读取 bootloader
   - 这是芯片设计时固定的，不能修改

3. **内核解压后，第一个用户态进程是什么？**
   - PID 1 → init 进程
   - init 解析 `init.rc` → 启动核心服务（servicemanager、surfaceflinger 等）
   - zygote（Android Java 虚拟机） → 启动 system_server → 启动 Launcher

## 需要懂的知识

**启动级联图**

```
时间轴 →
┌──────────┐   ┌─────────┐   ┌──────────┐   ┌─────────┐   ┌────────┐   ┌──────────┐
│ ROM Code │ → │ BL2     │ → │ BL30/SPL │ → │ U-Boot  │ → │ Kernel │ → │ init     │
│ (芯片内)  │   │(DDR初始化)│   │(安全启动) │   │(加载内核)│   │(驱动初始化)│   │(用户态入口)│
└──────────┘   └─────────┘   └──────────┘   └──────────┘   └────────┘   └──────────┘
    串口:          GX2A:BL:      BL2/BL30       U-Boot        Booting     init:
   "GX2A:BL:"     "DDR init"     加载完成      版本信息      Linux...    starting...
```

完整链路持续到 Launcher：
```
init → zygote → system_server → ActivityManager → Launcher
```

**串口日志中每一阶段的标志**

| 阶段 | 串口中找这个 | 说明 |
|------|-------------|------|
| ROM Code | `GX2A:BL:` 或 `G12A:BL:` | 芯片型号前缀 |
| DDR 初始化 | `DDR init complete` | BL2 完成内存训练 |
| U-Boot | `U-Boot 20xx.xx` | U-Boot 版本和编译时间 |
| Kernel | `Booting Linux on physical CPU` | 内核开始启动 |
| init | `init: init first stage started` | init 进程启动 |
| Zygote | `Zygote: Zygote main` | Java 虚拟机启动 |
| System Server | `SystemServer: Entered the Android system server` | 系统服务就绪 |
| Launcher | `Displayed org.xxx/.Launcher` | 桌面显示完成 |

## 动手方案

```bash
# 第 1 步：接串口，抓一份完整启动日志
# 重新上电，从开机到桌面完全显示
# 保存日志：screen 中用 Ctrl+A → H 记录日志

# 第 2 步：标注日志中的关键事件
grep -n "GX2A:BL\|DDR init\|U-Boot\|Booting Linux\|init first stage\|Zygote:\|SystemServer:\|Displayed" boot.log

# 第 3 步：在代码中找到对应的源文件
# U-Boot 源码
find ~/android/aml/s905x5/aml-s905x5-androidu-v2/ -name "u-boot" -type d 2>/dev/null

# 设备树
ls ~/android/aml/s905x5/aml-s905x5-androidu-v2/common/arch/arm64/boot/dts/amlogic/*ross*

# init.rc 核心文件
adb shell cat /init.rc | head -30

# Amlogic 特有的服务配置
find ~/android/aml/s905x5/aml-s905x5-androidu-v2/device/amlogic/ross/ -name "*.rc"

# 第 4 步：找 Amlogic 特有进程
adb shell ps -A | grep -E "ge2d|mediahal|droidvold|aml"

# 第 5 步：绘制自己的启动时序图
# +0.000s  ROM Code: GX2A:BL:...
# +0.050s  BL2: DDR init complete
# +0.200s  U-Boot: U-Boot 2023.01
# +1.500s  Kernel: Booting Linux on physical CPU
# +3.000s  init: init first stage started
# +10.00s  Zygote: Zygote main
# +18.00s  SystemServer: Entered the Android system server
# +25.00s  Launcher: Displayed ...
```

## 验收清单

| 项目 | 验证方式 |
|------|----------|
| 完整启动日志 | 记录从 ROM Code 到 Launcher 的串口输出 |
| 找到 u-boot 源码路径 | 笔记中记录了实际路径 |
| 标注关键事件 | 能在日志中圈出至少 5 个阶段的事件点 |
| 手绘时序图 | 画出一张 A4 纸的启动级联图，每个阶段注明对应代码位置 |
