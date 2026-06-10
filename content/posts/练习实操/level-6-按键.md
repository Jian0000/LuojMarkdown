+++
title = 'Level 6：GPIO 按键 — 从设备树到 input 子系统'
date = '2026-06-09T14:00:00+08:00'
draft = false
+++

# Level 6：GPIO 按键 — 从设备树到 input 子系统

## 我想做

板子上有一个丝印 **USER_KEY** 的物理按键（接在 **GPIOD_2**），按下它能被 Linux input 子系统识别到事件，并理解从 DTS → 驱动 → sysfs → /dev/input/eventX 的完整链路。

## 先回答三个问题

1. **Linux 怎么知道一个 GPIO 是按键而不是 LED？**
   - DTS 中的 `compatible` 决定匹配哪个驱动。LED 用 `"gpio-leds"`，按键用 `"amlogic, gpio_keypad"`（或主线 `"gpio-keys"`）
   - 驱动根据 compatible 字符串被内核匹配并 probe，在 probe 中注册 input 设备

2. **按键事件是怎么从硬件传到 App 的？**
   ```
   手指按下 → GPIO 电平变化 → 驱动检测（polling/irq）
     → input_report_key() → input_sync()
       → /dev/input/eventX → App 通过 getevent / InputReader 读取
   ```
   每一层都是标准接口，App 不关心 GPIO 编号，只读 keycode。

3. **Amlogic 的 gpio_keypad 和主线 gpio-keys 有什么区别？**
   - 主线 `gpio-keys`：通用驱动，每个按键独立描述为子节点
   - Amlogic `gpio_keypad`：私有驱动，用一个节点描述多个按键（`key_num` + 数组式属性），额外提供 `/sys/class/gpio_keypad/` 接口和蓝牙键计数功能

## 需要懂的知识

**input 子系统**

Linux 内核用 input 子系统统一管理所有输入设备（键盘、鼠标、触摸屏、遥控器等）。核心概念：

- `input_dev`：一个输入设备，驱动通过 `input_register_device()` 注册
- `input_event()`：报告一个输入事件（类型 + 代码 + 值）
- `input_sync()`：标记一帧事件结束，用户态据此判断读取可以停止
- `/dev/input/eventX`：用户态读取事件的字符设备

关键类型：
```c
EV_KEY    // 按键（KEY_VOLUMEUP, KEY_POWER, KEY_HOME ...）
EV_SW     // 开关（SW_MUTE_DEVICE ...）
EV_ABS    // 绝对坐标（触摸屏）
EV_REL    // 相对坐标（鼠标）
```

**DTS 中如何描述一个 GPIO 按键**

Amlogic `gpio_keypad` 节点的每一行含义（以 S7D bm201 为例）：

```dts
gpio_keypad {
    compatible = "amlogic, gpio_keypad";  // 匹配 Amlogic 私有驱动
    status = "okay";
    scan_period = <20>;                   // 轮询周期 20ms
    key_num = <2>;                        // 共 2 个按键
    key_name = "bluetooth", "mute";       // 按键名称（数组）
    key_code = <600 SW_MUTE_DEVICE>;      // Linux keycode（数组）
    key_type = <EV_KEY EV_SW>;            // 事件类型（数组）
    key-gpios = <&gpio GPIOD_2 GPIO_ACTIVE_HIGH   // GPIO 引脚（数组）
                 &gpio GPIOD_3 GPIO_ACTIVE_HIGH>;
    /* 0:polling mode, 1:irq mode */
    detect_mode = <0>;                    // 检测模式
};
```

注意：DTS 中 `key_name = "bluetooth"` 就是板子丝印上的 **USER_KEY**。`key_code = 600` 是厂商自定义 code（非标准 Linux keycode）。

**GPIO 编号规则（S7D）**

来自 `meson-s7d-gpio.h`——不同 SoC 编号完全不同，必须查对应当前芯片的头文件：

```
GPIOD_0 = 24, GPIOD_1 = 25, GPIOD_2 = 26, GPIOD_3 = 27, GPIOD_4 = 28
```

S7D pin 排列顺序：E → B → C → D → DV → H → X → Z → TEST_N → CC（共 83 pins）。

**polling 和 interrupt 两种检测模式**

| 模式 | detect_mode | 原理 | 优缺点 |
|------|-------------|------|--------|
| polling | 0 | 定时器每 20ms 读一次 GPIO 电平 | CPU 持续消耗，但实现简单，不会丢事件 |
| irq | 1 | 注册 GPIO 中断，边沿触发后启动 timer 消抖 | CPU 开销低，但中断风暴时可能丢事件 |

板子默认使用 polling 模式。

**驱动中的消抖（debounce）机制**

机械按键按下/释放时电平会抖动：

```
理想波形：  ──┐          ┌──
实际波形：  ──┐┌┐┌┐┌┐┌┐┌┐┌┐┌──
```

驱动通过 `KEY_JITTER_COUNT = 1` 消抖：连续 2 次读到相同值才确认状态变化（`gpio_keypad.c:64`）。

**蓝牙键的特殊计数逻辑**

驱动中硬编码了对 `"bluetooth"` 按键的计数（`gpio_keypad.c:76-78`）：

```c
if (strcmp(key->name, "bluetooth") == 0) {
    keypad->reset_count++;
}
```

这个计数暴露在 `/sys/class/resetkey/count`，通常用于实现"长按蓝牙键 N 次恢复出厂设置"。

## 动手方案

### 第 1 步：在板子上确认按键对应的 input 设备

```bash
# 列出所有 input 设备
adb shell getevent -l

# 在输出中找 gpio_keypad：
#   /dev/input/event3: gpio_keypad
#   记下这个 event 编号（假设是 event3）
```

### 第 2 步：监听按键事件

```bash
# 实时监听按键事件，按下板子上的 USER_KEY 观察输出
adb shell getevent -l /dev/input/event3

# 按下时输出类似：
#   EV_KEY      00000258        DOWN       ← 0258 = 600 = bluetooth keycode
#   EV_SYN      SYN_REPORT      00000000
# 释放时输出类似：
#   EV_KEY      00000258        UP
#   EV_SYN      SYN_REPORT      00000000
```

### 第 3 步：查看 sysfs 接口

```bash
# 查看所有按键的当前状态
adb shell cat /sys/class/gpio_keypad/table
# 输出：
# [0]: name = bluetooth           status = 1      ← 1=释放, 0=按下
# [1]: name = mute                status = 1

# 查看蓝牙键按下次数
adb shell cat /sys/class/resetkey/count

# 按几下 USER_KEY 再读，数字会递增
```

### 第 4 步：控制是否忽略指定按键

```bash
# 忽略蓝牙键（按下后无事件）
adb shell "echo 'bluetooth,600 Y' > /sys/class/gpio_keypad/ignore"

# 此时按 USER_KEY，getevent 不会再收到事件

# 恢复
adb shell "echo 'bluetooth,600 N' > /sys/class/gpio_keypad/ignore"

# 忽略所有按键
adb shell "echo 'all 1' > /sys/class/gpio_keypad/ignore"

# 恢复所有按键
adb shell "echo 'none 1' > /sys/class/gpio_keypad/ignore"
```

### 第 5 步：从内核日志确认驱动加载

```bash
# 查看 gpio_keypad 驱动的 probe 日志
adb shell dmesg | grep -i "gpio.keypad\|gpio_keypad"

# 查看 GPIOD_2 的 GPIO 占用
adb shell cat /sys/kernel/debug/gpio | grep -i "GPIOD_2\|bluetooth"
```

### 第 6 步：追踪驱动源码调用链

从 DTS probe 到按键事件上报的完整调用链：

```
platform_driver_register(&meson_gpio_kp_driver)
  → meson_gpio_kp_probe()             // gpio_keypad.c:264
    → devm_gpiod_get_index()          // 获取 GPIOD_2、GPIOD_3 的 GPIO 描述符
    → of_property_read_string_index() // 解析 key_name[]
    → gpiod_direction_input()         // 设为输入模式
    → gpiod_set_pull(PULL_UP)         // 使能内部上拉
    → input_register_device()         // 注册 input 设备
    → mod_timer(&polling_timer)       // 启动 20ms 定时轮询

定时器触发:
  → polling_timer_handler()           // gpio_keypad.c:89
    → gpiod_get_value()               // 读 GPIO 电平
    → report_key_code()               // gpio_keypad.c:60
      → input_event(EV_KEY, code, val)// 报告按键事件
      → input_sync()                  // 同步帧
```

## 关键文件路径速查

### 设备树（DTS）

| 用途 | 路径 |
|------|------|
| 板级 DTS（gpio_keypad 节点） | `common/common_drivers/arch/arm64/boot/dts/amlogic/s7d_s905x5m_bm201.dts:781` |
| GPIO pin 编号宏（GPIOD_2 = 26） | `common/common_drivers/include/dt-bindings/gpio/meson-s7d-gpio.h:42` |

### 按键驱动

| 用途 | 路径 |
|------|------|
| Amlogic gpio_keypad 驱动 | `common/common_drivers/drivers/input/keyboard/gpio_keypad.c` |
| 驱动 Makefile | `common/common_drivers/drivers/input/keyboard/Makefile` |
| 主线 gpio-keys 驱动（对照参考） | `common/drivers/input/keyboard/gpio_keys.c` |
| 主线 gpio-keys 驱动（polled 版本） | `common/drivers/input/keyboard/gpio_keys_polled.c` |

### GPIO / pinctrl 驱动

| 用途 | 路径 |
|------|------|
| S7D pinctrl 驱动 | `common/common_drivers/drivers/gpio/pinctrl/pinctrl-meson-s7d.c` |
| Meson pinctrl 通用驱动 | `common/drivers/pinctrl/meson/pinctrl-meson.c` |
| gpiolib 核心 | `common/drivers/gpio/gpiolib.c` |
| gpiolib OF 解析 | `common/drivers/gpio/gpiolib-of.c` |

### 板子上用户态路径

| 用途 | 命令/路径 |
|------|----------|
| 监听按键事件 | `adb shell getevent -l /dev/input/eventX` |
| 查看按键状态表 | `cat /sys/class/gpio_keypad/table` |
| 查看/控制按键忽略 | `cat /sys/class/gpio_keypad/ignore` |
| 查看蓝牙键计数 | `cat /sys/class/resetkey/count` |
| 查看 GPIO 占用 | `cat /sys/kernel/debug/gpio` |

## 延伸阅读

驱动代码的逐段深度分析（数据结构设计、probe 流程、input 子系统集成、sysfs class 接口、电源管理等）已整理至 **[level-6-总结与扩展.md](level-6-总结与扩展.md)**。

## 验收清单

| 项目 | 验证方式 |
|------|----------|
| 找到按键对应的 input 设备 | `getevent -l` 看到 `gpio_keypad` |
| 按下按键看到事件 | `getevent -l /dev/input/eventX` 看到 KEY DOWN/UP |
| 理解 sysfs 接口 | 能查看 `table`，会用 `ignore` 开关按键 |
| 理解 DTS 节点 | 能说清 DTS 中 `key_name`、`key_code`、`key-gpios` 的对应关系 |
| 理解消抖机制 | 能解释为什么需要 `KEY_JITTER_COUNT` |
| 区分 polling vs irq | 能从 DTS `detect_mode` 判断当前模式，并说清两种模式的优缺点 |
