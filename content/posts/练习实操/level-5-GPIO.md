+++
title = 'Level 5：操作 GPIO'
date = '2026-06-09T15:02:00+08:00'
draft = false
+++

# Level 5：操作 GPIO

## 我想做

控制板子上的一个物理引脚，让它输出高/低电平，用万用表或 LED 确认电平变化。

## 先回答三个问题

1. **GPIO 在系统里是以什么方式呈现的？**
   - 物理上：芯片的一个管脚，可以设置输出高/低电平或读取输入电平
   - 代码上：GPIO controller 是一个硬件模块，通过寄存器操作
   - 给用户的接口：sysfs 或 libgpiod，或者自己写驱动操作寄存器

2. **设备树（DTS）和驱动是什么关系？**
   - DTS 描述硬件有什么、接在哪个总线、地址/中断号是什么
   - 驱动根据 DTS 的信息来初始化硬件
   - 同一个驱动可以支持多个 DTS 描述的实例

3. **你的板子的 GPIO 信息从哪儿来？**
   - 原理图（PDF）——告诉你在板子上哪个排针/接口引出了哪个 GPIO
   - DTS——告诉内核哪个 GPIO controller 对应哪个基地址
   - datasheet——详细到每个寄存器的位定义

## 需要懂的知识

**GPIO 子系统**

内核用 gpiolib 统一管理 GPIO。用户态访问 GPIO 有两种方式：

旧方式（sysfs，kernel < 5.10）：
```bash
echo 100 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio100/direction
echo 1 > /sys/class/gpio/gpio100/value
```

新方式（libgpiod，kernel >= 5.10）：
```bash
gpioinfo
gpioset gpiochip0 100=1
```

你的内核是 5.15，优先用 libgpiod。

**ioremap 的作用**

驱动不能直接访问物理地址（因为 MMU 开启了虚拟地址映射）。`ioremap` 将物理地址映射到内核虚拟地址空间：
```c
void __iomem *base = ioremap(0xff800000, 0x1000);
writel(1, base + 0x14);
readl(base + 0x14);
```

没有 ioremap 会怎样？直接访问物理地址会触发 kernel panic。

**DTS 中的 GPIO 描述**

```dts
gpio@ff800000 {
    compatible = "amlogic,meson-g12a-gpio";
    reg = <0x0 0xff800000 0x0 0x1000>;
    gpio-controller;
    #gpio-cells = <2>;
};
```

- `reg` —— 寄存器基地址和范围
- `gpio-controller` —— 标记这是一个 GPIO 控制器
- `#gpio-cells` —— 每个 GPIO 描述需要几个 cell（通常 2：引脚号 + 标志位）

**如何查询某个 GPIO 名称的全局编号？**

问题：原理图上标的是 `GPIOD_4`，但 `gpio_set_value()` 需要的是一个整数。我怎么知道 `GPIOD_4` = 28？

答案在 **dt-bindings 头文件**。每个 SoC 有一个 `xxx-gpio.h` 文件，为所有 GPIO pin 分配全局唯一编号：

文件：`common/common_drivers/include/dt-bindings/gpio/meson-s7d-gpio.h`

```c
/* GPIOE */              ← S7D 以 E 组起始
#define GPIOE_0   0
...
/* GPIOB */
#define GPIOB_0   2
...
/* GPIOC */
#define GPIOC_0  16
...
/* GPIOD */
#define GPIOD_0  24
#define GPIOD_4  28      ← 这就是你要找的！S7D 上 GPIOD_4 = 28

/* GPIOX */
#define GPIOX_0  48
...
```

**编号规则**：全局编号 = 该 pin 在 `meson_s7d_periphs_pins[]` 数组中的索引位置。S7D 顺序为 E→B→C→D→DV→H→X→Z→TEST_N→CC（共 83 pins），与 S6 不同（S6 是 D→F→E→B→C→X→H→Z→A→TEST_N→CC，101 pins）。

**查找方法**：
```bash
# 方法 1：直接看头文件
grep GPIOD_4 common/common_drivers/include/dt-bindings/gpio/meson-s7d-gpio.h

# 方法 2：在 pinctrl 驱动中看 pins 数组的排列顺序（索引从 0 起）
grep "MESON_PIN" common/common_drivers/drivers/gpio/pinctrl/pinctrl-meson-s7d.c | head -35
```

**不同 SoC 的编号完全不同，必须查对应当前芯片的头文件！**

| 芯片 | GPIOD_4 编号 | GPIOA_4 编号 | 总 pins |
|------|-------------|-------------|---------|
| S6   | 4           | 86          | 101     |
| S7D  | 28          | 不存在      | 83      |





## 动手方案

### 前提

- 板子：S905X5M (S7D)，内核 5.15
- 目标 GPIO：**GPIOD_4（pin 28）**，原理图标号 **FUN_LED**
- GPIO 控制器：`/dev/gpiochip0`（`bank@4000`），83 pins，基址 `0xfe004000`
- **重要发现**：GPIOD_4 已被 `leds-gpio` 驱动接管（DTS 中声明为 `sys_led_red`），可直接通过 sysfs 控制，无需写驱动

### 方式 A（推荐）：通过现有 sysfs LED 接口控制（零代码）

GPIOD_4 在板级 DTS `s7d_s905x5m_bm201.dts:224-228` 中已声明为 `gpio-leds` 设备的子节点 `sys_led_red`。内核 `leds-gpio` 驱动已接管该 pin，提供标准 sysfs 接口：

```bash
# 直接控制 LED 亮灭（无需 insmod / mknod）
adb shell "echo 1 > /sys/class/leds/sys_led_red/brightness"   # LED 亮
adb shell "echo 0 > /sys/class/leds/sys_led_red/brightness"   # LED 灭

# 查看当前状态
adb shell "cat /sys/class/leds/sys_led_red/brightness"

# 设置心跳闪烁
adb shell "echo heartbeat > /sys/class/leds/sys_led_red/trigger"

# 恢复手动控制
adb shell "echo none > /sys/class/leds/sys_led_red/trigger"

# 验证底层 GPIO 确实被占用
adb shell "cat /sys/kernel/debug/gpio"    # 会看到 gpio-28 标为 "sys_led_red"
```

完整调用链路（详细分析见下一节）：
```
echo 1 > /sys/class/leds/sys_led_red/brightness
  → led-class.c:brightness_store()
    → leds-gpio.c:gpio_led_set()
      → gpiolib.c:gpiod_set_value()          // FLAG_ACTIVE_LOW 自动翻转
        → pinctrl-meson.c:meson_gpio_set()
          → regmap_update_bits()             // 写 GPIO 输出寄存器
```

### 方式 B：编写 fun_led.c 字符设备驱动（学习用，需先解除 GPIO 占用）

> **注意**：此方式与现有 `leds-gpio` 冲突。若强行 insmod 会报 `-517`（`-EPROBE_DEFER`），详见 debug.md [L5-02]。如需测试此方式，须先在 DTS 中 disable `sys_led_red` 节点后重新编译烧录。

以下是原方案（仅供理解构建流程）：

### 第 1 步：确认 GPIO 子系统正常

```bash
adb shell ls /dev/gpiochip0        # 确认 gpiochip 存在
adb shell cat /sys/bus/gpio/devices/gpiochip0/uevent
# → OF_NAME=bank
# → OF_FULLNAME=/soc/apb4@fe000000/pinctrl@4000/bank@4000
```

### 第 2 步：从原理图/DTS 确认 GPIO 信息

原理图截图中 GPIOD_4 标为 FUN_LED。交叉验证 DTS 中该 pin 未被占用：

```bash
# 在对应的 S7D DTS 中确认 GPIOD_4 未被占用
grep "GPIOD_4" common/common_drivers/arch/arm64/boot/dts/amlogic/你板子对应的.dts
```

### 第 3 步：编写驱动 `fun_led.c`

基于 Level 4 的字符设备框架，加入 GPIO 控制。驱动代码：

- 文件：`common/drivers/misc/fun_led.c`
- 用 `<linux/gpio.h>` 的 legacy API：`gpio_request(28, "fun_led")` → `gpio_direction_output(28, 0)` → `gpio_set_value(28, val)`
- 注册为字符设备 `/dev/fun_led`
- write("1") = LED 亮，write("0") = LED 灭，read = 读当前状态

### 第 4 步：修改构建配置

| 文件 | 改动 |
|------|------|
| `common/drivers/misc/Kconfig` | 新增 `CONFIG_FUN_LED`（依赖 GPIOLIB） |
| `common/drivers/misc/Makefile` | 新增 `obj-$(CONFIG_FUN_LED) += fun_led.o` |
| `common/common_drivers/arch/arm64/configs/amlogic_gki.fragment` | 新增 `CONFIG_FUN_LED=m` |
| `common/common_drivers/modules.bzl` | 新增 `"drivers/misc/fun_led.ko"` |

### 第 5 步：编译

```bash
cd ~/android/aml/s905x5/aml-s905x5-androidu-v2
./build.sh -kj200
```

确认产物：`out/android14-5.15/dist/fun_led.ko`

### 第 6 步：烧录并测试

```bash
# 烧录后，在板子上
adb shell insmod /vendor/lib/modules/fun_led.ko
dmesg | grep fun_led                    # 看 major 号

# 用获取到的 major 创建设备节点（假设 major=237）
adb shell mknod /dev/fun_led c 237 0

# 控制 LED
echo 1 > /dev/fun_led                   # LED 亮
echo 0 > /dev/fun_led                   # LED 灭
cat /dev/fun_led                        # 读取当前状态

# 验证：万用表测 GPIOD_4 引脚电平变化

# 卸载
adb shell rmmod fun_led
```


## 关键收获

### 1. 你实际完成了什么

最初的目标是"控制 GPIOD_4 (FUN_LED)"。最终发现**不需要写任何驱动**，内核的 `leds-gpio` 驱动已经通过 DTS 接管了该引脚，提供标准 sysfs 接口：

```bash
echo 1 > /sys/class/leds/sys_led_red/brightness   # 亮
echo 0 > /sys/class/leds/sys_led_red/brightness   # 灭
```

### 2. 核心认知

- **DTS 描述硬件，驱动不再硬编码** — 你的 `fun_led.c` 把 `LED_GPIO 28` 写死在 C 代码里，而 `leds-gpio` 通过 DTS 的 `gpios = <&gpio GPIOD_4 GPIO_ACTIVE_LOW>` 声明硬件连接，换板子只改 DTS
- **gpiod consumer API 替代 legacy 整数 API** — `gpiod_get(dev, "led", ...)` 返回描述符，代替 `gpio_request(28, ...)`
- **内核设备模型统一管理 sysfs** — 驱动只声明业务属性（`dev_groups`），通用文件（device/subsystem/uevent/power）由 `device_add()` 自动创建

### 3. fun_led.ko 为何失败

GPIOD_4 已被 DTS 中的 `leds-gpio` 驱动占用，重复申请导致 -517（-EPROBE_DEFER）。详见 [debug.md L5-02](debug.md)。

### 4. 延伸阅读

所有深度分析已整理至 **[level-5-总结与扩展.md](level-5-总结与扩展.md)**。

## 关键文件路径速查

### 设备树（DTS）

| 用途 | 路径 |
|------|------|
| SoC 级 DTSI（GPIO 控制器 `bank@4000`） | `common/common_drivers/arch/arm64/boot/dts/amlogic/meson-s7d.dtsi` |
| 板级 DTS（`gpio_leds { sys_led_red }`） | `common/common_drivers/arch/arm64/boot/dts/amlogic/s7d_s905x5m_bm201.dts` |
| GPIO pin 编号宏（`GPIOD_4 = 28`） | `common/common_drivers/include/dt-bindings/gpio/meson-s7d-gpio.h` |

### GPIO / pinctrl 驱动

| 用途 | 路径 |
|------|------|
| S7D pinctrl 驱动（bank 表、pinmux） | `common/common_drivers/drivers/gpio/pinctrl/pinctrl-meson-s7d.c` |
| Meson pinctrl 通用驱动（`meson_get_bank`、`meson_calc_reg_and_bit`） | `common/drivers/pinctrl/meson/pinctrl-meson.c` |
| gpiolib 核心（`gpio_to_desc`、`gpiod_request`） | `common/drivers/gpio/gpiolib.c` |
| gpiolib OF 解析（`of_get_named_gpiod_flags`、`of_gpio_simple_xlate`） | `common/drivers/gpio/gpiolib-of.c` |
| pinctrl 核心（`pinctrl_gpio_request`） | `common/drivers/pinctrl/core.c` |

### LED 驱动与 sysfs

| 用途 | 路径 |
|------|------|
| leds-gpio 驱动（`compatible = "gpio-leds"`） | `common/drivers/leds/leds-gpio.c` |
| LED class 核心（sysfs 属性注册） | `common/drivers/leds/led-class.c` |
| LED core 辅助（`led_init_default_state_get`） | `common/drivers/leds/led-core.c` |
| fwnode 属性读取（`fwnode_property_read_string`） | `common/drivers/base/property.c` |
| 设备模型核心（`device_add`、sysfs 创建） | `common/drivers/base/core.c` |
| 电源管理 sysfs（`power/` 目录） | `common/drivers/base/power/sysfs.c` |

### fun_led.c（实验用硬编码驱动）

| 用途 | 路径 |
|------|------|
| 板子上的 ko | `/vendor/lib/modules/fun_led.ko` |
| 内核树源码 | `common/drivers/misc/fun_led.c` |
| 本地实验参考 | `learn/experiments/fun_led.c` |

### 构建配置

| 用途 | 路径 |
|------|------|
| Kconfig | `common/drivers/misc/Kconfig` |
| Makefile | `common/drivers/misc/Makefile` |
| Bazel fragment（实际生效） | `common/common_drivers/arch/arm64/configs/amlogic_gki.fragment` |
| Bazel 模块拷贝列表 | `common/common_drivers/modules.bzl` |
| 构建脚本 | `./build.sh`（工作区根目录） |

### 板子上用户态路径

| 用途 | 命令/路径 |
|------|----------|
| LED 控制 | `echo 1 > /sys/class/leds/sys_led_red/brightness` |
| GPIO 信息 | `gpioinfo` / `gpiodetect` / `gpioset` / `gpioget` |
| debugfs GPIO 列表 | `/sys/kernel/debug/gpio`（需先挂载 debugfs） |
| gpiochip | `/dev/gpiochip0` |
| 字符设备 | `/dev/fun_led`（方式 B 创建） |

## 验收清单

| 项目 | 验证方式 |
|------|----------|
| 找到板子的 GPIO controller | 通过 `gpioinfo` 或 `/sys/kernel/debug/gpio` 确认 |
| 控制电平 | 万用表/逻辑分析仪测量到引脚电平变化 |
| 理解设备树 | 你能在 DTS 中找到 GPIO controller 节点，说出 `reg` 和 `gpio-controller` 的含义 |
| ioremap | 你能回答"为什么不直接访问物理地址" |
