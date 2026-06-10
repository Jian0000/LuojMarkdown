+++
title = 'Level 2：编译自己的固件并烧录'
date = '2026-06-09T16:03:00+08:00'
draft = false
+++

# Level 2：编译自己的固件并烧录

## 我想做

从源码完整编译出一版固件，烧到板子里，确认跑的是我自己编译的系统。

## 先回答三个问题

1. **你的项目用什么方式编译？**
   - 标准 AOSP：`source build/envsetup.sh` → `lunch` → `make`
   - 本项目（ross）：用 `build.cfg` + `./mk` 脚本
   - 两者的区别是什么？

2. **一个固件由哪些东西组成？**
   - boot.img（内核 + ramdisk）
   - super.img（system + vendor + product）
   - dtb/dtbo（设备树）
   - 这些镜像文件在 `out/target/product/ross/` 下

3. **怎么判断编译成功了？**
   - 最后一行：`#### build completed successfully ####`
   - 关键镜像文件生成且大小合理
   - 烧录后 `ro.build.date` 是你刚才的编译时间

## 需要懂的知识

**AOSP 构建系统骨架**

一个 Android 固件的编译最终是一系列 Makefile 规则的执行，但 AOSP 用了一整套包装：
- `build/envsetup.sh` ——设置环境变量，提供 `lunch`、`mmm`、`make` 等命令
- `lunch` ——选择产品+编译类型，本质是设置 `TARGET_PRODUCT` 和 `TARGET_BUILD_VARIANT`
- `make` ——根据产品配置读取所有 `Android.mk`/`Android.bp`，编译出完整系统

在 ross 项目中，`build.cfg` 代替了 `lunch` 的手动选择：
```bash
# build.cfg 的内容：
PRODUCT_NAME=ross
TYPE=userdebug
BOARD_COMPILE_ATV=false
```
`./mk ross` 就是读取这个文件，自动调用对应的 AOSP 命令。

**几种编译类型**

| 类型 | adb root | SELinux | 用途 |
|------|----------|---------|------|
| eng | 有 | 宽松 | 开发调试，最慢 |
| userdebug | 有 | 强制 | 调试与测试，常用 |
| user | 无 | 强制 | 发布版 |

**烧录方式**

Amlogic 平台两种主流烧录方式：
- **USB Burning Tool（Windows）**：需要 `update.img` 整包，整个 eMMC 全部擦除重写。适用于第一次烧录或板子变砖时
- **fastboot（Android 自带）**：可以只烧单个分区（如只烧 boot.img），迭代开发时快很多。需要板子进入 fastboot 模式

## 动手方案

```bash
# 第 1 步：确认编译环境
cd ~/android/aml/s905x5/aml-s905x5-androidu-v2/
cat build.cfg       # 确认 PRODUCT_NAME=ross
ls device/amlogic/ross/   # 板级配置存在

# 第 2 步：开始编译
./mk ross 2>&1 | tee build.log
# 第一次编译 8-16 小时，之后增量编译快很多

# 第 3 步：确认编译产物
ls -lh out/target/product/ross/*.img
# 应该能看到 boot.img、super.img、dtb*.img 等

# 第 4 步：烧录
# 方式 A：USB Burning Tool（如果有 update.img）
# ./mk ross update   # 生成烧录包
# 在 Windows 上打开 USB_Burning_Tool，导入 update.img，按住板子上的
# 升级按键上电，点击"开始"

# 方式 B：fastboot（推荐，迭代时快得多）
adb reboot bootloader
# 如果 adb reboot bootloader 不行，按住板子升级按键上电直接进
fastboot flash boot out/target/product/ross/boot.img
fastboot flash super out/target/product/ross/super.img
fastboot flash dtb out/target/product/ross/dtb*.img
fastboot reboot

# 第 5 步：验证
adb shell getprop ro.build.date
# 输出应该是你的编译时间
```

## 验收清单

| 项目 | 验证方式 |
|------|----------|
| 编译成功 | `build.log` 最后一行 `#### build completed successfully ####` |
| 烧录后正常开机 | `adb shell` 能进入 |
| 是你编译的固件 | `ro.build.date` 是刚才的编译时间，`ro.build.fingerprint` 包含你的编译信息 |
| dmesg 无核心错误 | `adb shell dmesg \| grep -i "error\|fail"` 没有关键驱动报错 |
