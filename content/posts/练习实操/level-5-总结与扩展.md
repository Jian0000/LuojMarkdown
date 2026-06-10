+++
title = 'Level 5 扩展与总结：GPIO 子系统深度分析'
date = '2026-06-09T15:03:00+08:00'
draft = false
+++

# Level 5 扩展与总结：GPIO 子系统深度分析

本文是 [level-5-GPIO.md](level-5-GPIO.md) 的延伸，记录从编写 `fun_led.c` 到发现 `leds-gpio` 现成方案过程中，对 GPIO 子系统的深度分析。

---

## 1. 驱动分析：硬编码 GPIO 的问题

当前 `fun_led.c` 采用的是**硬编码 GPIO 编号**方式：

```c
#define LED_GPIO  28   /* 写死在代码里的 GPIO 编号 */
...
gpio_request(28, "fun_led");
gpio_set_value(28, val);
```

### 硬编码带来的三个风险

**风险 1：内核升级，编号体系变化**

`LED_GPIO 28` 对应 S7D 的 GPIOD_4。这个 28 来自 `meson-s7d-gpio.h` 中 pins 数组的排列顺序。如果 Amlogic 在新内核 BSP 中调整了 pins 数组顺序（例如把 E 组排到 D 组之后），GPIOD_4 的编号就不再是 28 了。驱动里写死的 28 会指向错误的 pin。

**风险 2：换板子不可用**

同一份 `fun_led.c` 源码，想在另一个 S905 型号（比如 S6、S4D）上使用。S6 的 GPIOD_4 = 4，S4D 的 GPIOD_4 可能又是另一个编号。驱动里写着 28，换板子必炸。

**风险 3：驱动和硬件信息耦合**

驱动负责"控制 LED 亮灭"，但它不负责"知道 LED 接在哪个 pin 上"——这是硬件设计的决定，应该由硬件描述文件（DTS）来声明。当前代码把两件事揉在一起了。

### 类比：餐厅点菜

```
硬编码方式：                DTS 方式：
厨师口袋里揣着一张纸条      菜单写在墙上（DTS）
"3 号桌客人要吃宫保鸡丁"    厨师看墙上菜单做菜
纸条上写着菜和桌号           换客人？改墙上的菜单即可
换客人？厨师换纸条          厨师不用换
```

当前 `fun_led.c` 就是那个"把桌号写死在口袋里"的厨师。

---

## 2. DTS 驱动方案分析

目标：驱动不再硬编码 28，而是从 DTS 中读取"我控制的是哪个 GPIO"。

### DTS 侧：声明硬件连接

```dts
/ {
    fun_led {
        compatible = "myvendor,fun-led";
        led-gpios = <&gpio GPIOD_4 GPIO_ACTIVE_HIGH>;
    };
};
```

| 属性 | 含义 | 谁在用 |
|------|------|--------|
| `compatible = "myvendor,fun-led"` | 设备型号标识 | 内核用它匹配对应的驱动 |
| `led-gpios` | 属性名，`-gpios` 后缀是内核约定 | `gpiod_get(dev, "led", ...)` 自动查找此属性 |
| `<&gpio ...>` | 引用 DTS 中的 gpio 控制器节点（`bank@4000`） | of_parse_phandle 解析出 gpiochip |
| `GPIOD_4` | pin 名称（不是数字 28！） | 通过 `#gpio-cells = <2>` 解析，查 `meson-s7d-gpio.h` |
| `GPIO_ACTIVE_HIGH` | 高电平有效 | gpiod 自动处理逻辑电平翻转 |

**关键区别**：硬编码版写的是 `28`（数字），DTS 版写的是 `GPIOD_4`（名称）。名称到数字的转换由内核在解析 DTS 时完成，驱动不感知。

### 驱动侧：gpiod consumer API

```c
// ===== DTS 驱动版（应该写成这样）=====
#include <linux/gpio/consumer.h>   // 注意：consumer.h，不是 gpio.h
#include <linux/of.h>              // 读 DTS 属性

static int fun_led_probe(struct platform_device *pdev)
{
    struct gpio_desc *led;

    // gpiod_get 自动查 DTS 中 "led-gpios" 属性
    led = gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
    if (IS_ERR(led))
        return PTR_ERR(led);

    // 此后只用 gpiod_set_value(led, 1/0)，完全不知道 pin 编号
    ...
}
```

`gpiod_get(&pdev->dev, "led", ...)` 做的工作：

```
1. 从 dev->of_node 读 DTS 节点的 "led-gpios" 属性
      ↓
2. of_parse_phandle_with_args() 解析 <&gpio GPIOD_4 GPIO_ACTIVE_HIGH>
   → args[0] = GPIOD_4 的编号（查 meson-s7d-gpio.h → 28）
   → args[1] = GPIO_ACTIVE_HIGH
      ↓
3. of_get_named_gpiod_flags() 把解析结果转为 gpio_desc
   → gpio_to_desc(28) → 找到对应的 gpio_desc
      ↓
4. gpiod_direction_output(desc, 0) → 配置为输出、初始低电平
      ↓
5. 返回 gpio_desc 指针给驱动
```

### consumer.h vs gpio.h

`gpio.h` 是 legacy 整数 API，操作的是"全局 GPIO 编号"：

```c
gpio_request(28, "fun_led");     // 28 是什么？查头文件才知道
gpio_direction_output(28, 0);
gpio_set_value(28, 1);           // 控制的是哪根 pin？看代码看不出来
```

`gpio/consumer.h` 是 gpiod API，操作的是"描述符"：

```c
struct gpio_desc *led = gpiod_get(dev, "led", GPIOD_OUT_LOW);
// 描述符里有：pin 编号、所属 chip、方向、初始值……
gpiod_set_value(led, 1);         // LED 亮——代码自解释
```

`struct gpio_desc` 里实际包含了：

```c
struct gpio_desc {
    struct gpio_device *gdev;    // → chip（指向 gpiochip0）
    unsigned long flags;         // 状态标记（是否已申请、方向、开漏...）
    const char *label;           // "fun_led"（gpio_request 时设置的标签）
    // ...
};
// gdev→descs[hw_offset] = 指向自己的数组槽
// gdev→chip→set(chip, hw_offset, value) = 硬件写回调
```

### 驱动如何被 DTS 触发 probe？

```c
// 匹配表：DTS 的 compatible 字符串 → 驱动
static const struct of_device_id fun_led_dt_ids[] = {
    { .compatible = "myvendor,fun-led" },
    { }
};
MODULE_DEVICE_TABLE(of, fun_led_dt_ids);

// platform 驱动：不再用 module_init() 手动注册
static struct platform_driver fun_led_driver = {
    .probe  = fun_led_probe,
    .remove = fun_led_remove,
    .driver = {
        .name = "fun_led",
        .of_match_table = fun_led_dt_ids,
    },
};
module_platform_driver(fun_led_driver);
```

触发时机：

```
内核启动
  → of_platform_populate() 遍历 DTS 根节点下的子节点
  → 找到 fun_led { compatible = "myvendor,fun-led" }
  → 遍历 platform_driver 注册表
  → 匹配：fun_led_dt_ids 中有 "myvendor,fun-led"
  → 调用 fun_led_probe(&pdev)
  → gpiod_get(&pdev->dev, "led", ...) 拿到 GPIO
  → 注册字符设备
  → /dev/fun_led 出现（自动创建，无需 insmod + mknod）
```

---

## 3. 实战：leds-gpio 完整流程追踪

### 发现过程

`insmod fun_led.ko` 报 `-517`（`-EPROBE_DEFER`），追查原因时发现：GPIOD_4 早已被内核的 **`leds-gpio` 驱动**通过 DTS 接管了。这意味着不必写任何新驱动，板子已经有一套**完整的 DTS → 驱动 → sysfs → 硬件**链路在运行。

### 完整链路（6 步）

```
/sys/class/leds/sys_led_red/brightness    ← 用户写 brightness 控制 LED
        ↓
led-class.c:brightness_store()            ← 接到用户写入的值
        ↓
leds-gpio.c:gpio_led_set()                ← 调用 gpiod_set_value()
        ↓
gpiolib.c:gpiod_set_value()               ← 根据 GPIO_ACTIVE_LOW 翻转逻辑值
        ↓
pinctrl-meson.c:meson_gpio_set()          ← Amlogic 平台回调
        ↓
regmap_update_bits(0xFE0040C4, BIT(4), v) ← 硬件寄存器写入
```

#### 第 1 步：SoC 级 DTSI — 声明 GPIO 控制器硬件

文件：`common/common_drivers/arch/arm64/boot/dts/amlogic/meson-s7d.dtsi:456-469`

```dts
periphs_pinctrl: pinctrl@4000 {
    compatible = "amlogic,meson-s7d-periphs-pinctrl";
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    gpio: bank@4000 {
        reg = <0x0 0x4000 0x0 0x004c>,       // mux 寄存器（pinmux 配置）
              <0x0 0x40c0 0x0 0x02cc>;       // gpio 寄存器（输入/输出/中断）
        reg-names = "mux", "gpio";
        gpio-controller;                      // ← 标记
        #gpio-cells = <2>;                    // ← 引用格式 <&gpio PIN FLAGS>
        gpio-ranges = <&periphs_pinctrl 0 0 84>;
    };
};
```

- `gpio-controller` — 其他设备可以通过 `gpios` 属性引用此节点
- `#gpio-cells = <2>` — 引用需 2 个 cell：pin 编号 + flags
- `gpio-ranges` — GPIO pin ↔ pinctrl pin 映射表
- 标签 `gpio:` 是别名，DTS 中写 `&gpio` 即引用此节点

#### 第 2 步：板级 DTS — 声明 LED 设备

文件：`common/common_drivers/arch/arm64/boot/dts/amlogic/s7d_s905x5m_bm201.dts:216-229`

```dts
gpio_leds {
    compatible = "gpio-leds";
    status = "okay";
    sys_led_red {
        label = "sys_led_red";
        gpios = <&gpio GPIOD_4 GPIO_ACTIVE_LOW>;
        default-state = "on";
    };
};
```

| 属性 | DTS 中写的 | 内核解析结果 |
|------|-----------|-------------|
| `compatible` | `"gpio-leds"` | 匹配 `of_gpio_leds_match[]` |
| `gpios` | `<&gpio GPIOD_4 GPIO_ACTIVE_LOW>` | pin 28, `FLAG_ACTIVE_LOW` |
| `default-state` | `"on"` | 驱动 init 时点亮 |
| `label` | `"sys_led_red"` | sysfs 目录名 |

### 内核如何读取这些 DTS 属性？—— OF API 概览

DTS 文件编译成 DTB（Device Tree Blob）后由 bootloader 传给内核。内核提供了一套 **OF (Open Firmware) API**，驱动通过这些函数读取 DTS 中声明的属性。每个属性类型有对应的 API：

| DTS 属性 | 类型 | 内核读取 API | 谁调用 |
|----------|------|-------------|--------|
| `compatible` | 字符串 | `of_match_device()` → 遍历 `of_match_table` | platform bus（自动） |
| `gpios` | phandle 引用 | `of_parse_phandle_with_args_map()` | `gpiolib-of.c` |
| `default-state` | 字符串 | `fwnode_property_read_string(fwnode, "default-state", ...)` | `leds-gpio.c` → `led_init_default_state_get()` |
| `label` | 字符串 | `fwnode_property_read_string()`（LED core 内部） | `led-class.c` → `led_compose_name()` |
| `status` | 字符串 | `of_device_is_available()` | platform bus（自动） |

下面逐一追踪每种属性的读取过程。

#### gpios 属性解析：`<&gpio GPIOD_4 GPIO_ACTIVE_LOW>` → `gpio_desc*`

这是最复杂的一个。`leds-gpio.c` 调用的入口是：

```c
led.gpiod = devm_fwnode_get_gpiod_from_child(dev, NULL, child, GPIOD_ASIS, NULL);
```

**完整调用链（每个函数标注文件路径）**：

```
[1] leds-gpio.c:154
    devm_fwnode_get_gpiod_from_child(dev, NULL, child, GPIOD_ASIS, NULL)
      │
      ▼
[2] gpiolib.c:4013
    fwnode_get_named_gpiod(fwnode, NULL, index, dflags, label)
      → is_of_node(fwnode)?  YES → 走 OF 路径
      │
      ▼
[3] gpiolib-of.c:358
    gpiod_get_from_of_node(to_of_node(fwnode), propname, index, dflags, label)
      → propname 为 NULL，自动推导属性名为 "gpios"（-gpios 后缀是内核约定）
      │
      ▼
[4] gpiolib-of.c:291    of_get_named_gpiod_flags(node, "gpios", index, &flags)
      │
      ├─ [4a] drivers/of/base.c
      │      of_parse_phandle_with_args_map(np, "gpios", "gpio", 0, &gpiospec)
      │      解析 DTS 属性 "gpios = <&gpio 28 1>":
      │        gpiospec.np        → gpio 控制器的 device_node (bank@4000)
      │        gpiospec.args[0]   → 28   (GPIOD_4 宏展开后的值)
      │        gpiospec.args[1]   → 1    (GPIO_ACTIVE_LOW)
      │        gpiospec.args_count → 2   (由 #gpio-cells = <2> 决定)
      │
      ├─ [4b] gpiolib-of.c:93    of_find_gpiochip_by_xlate(&gpiospec)
      │      → gpiochip_find(gpiospec, of_gpiochip_match_node_and_xlate)
      │       遍历所有已注册的 gpio_chip，找到 of_node == gpiospec.np 的那个
      │       → 返回 gpiochip0（由 bank@4000 注册的 chip）
      │
      └─ [4c] gpiolib-of.c:99    of_xlate_and_get_gpiod_flags(chip, &gpiospec, &flags)
             → chip->of_gpio_n_cells != args_count?  2 == 2 ✓
             → chip->of_xlate(chip, &gpiospec, &flags)
               → of_gpio_simple_xlate()  [gpiolib-of.c:851]
                 if (gpiospec->args[0] >= gc->ngpio)  → 28 < 83 ✓
                 *flags = gpiospec->args[1]            → flags = 1 (GPIO_ACTIVE_LOW)
                 return gpiospec->args[0]              → 返回 28
             → gpiochip_get_desc(chip, 28)
               → 返回 &chip->gpiodev->descs[28]  ← 这就是 gpio_desc*！
```

**关键点总结**：

1. **`of_parse_phandle_with_args_map()`** — 把 DTS 的属性值 `<&gpio 28 1>` 解析成结构化数据（`of_phandle_args`），包括引用的节点指针（`np`）和参数数组（`args[]`）
2. **`of_find_gpiochip_by_xlate()`** — 根据 phandle 找到对应的 gpio_chip（哪个芯片的 GPIO 控制器）
3. **`of_xlate()`** — chip 的"翻译"回调，把 `args[]` 翻译成 chip 内部的 offset（这里 28 → 28，直通）
4. **`gpiochip_get_desc()`** — 用 offset 从 chip 的描述符数组中取出 `gpio_desc*`

`of_parse_phandle_with_args_map()` 的关键参数：
- `"gpios"` — DTS 中的属性名
- `"gpio"` — 属性值中 phandle cell 的名字（用于查找 `#gpio-cells`）

它会找到 `&gpio` 引用的节点（`bank@4000`），读该节点的 `#gpio-cells = <2>`，知道"后面 2 个 cell 是参数"，然后把 `<28 1>` 填入 `args[0]` 和 `args[1]`。

#### default-state 属性解析：`"on"` → `LEDS_DEFSTATE_ON`

文件：`common/drivers/leds/led-core.c:481-493`

```c
enum led_default_state led_init_default_state_get(struct fwnode_handle *fwnode)
{
    const char *state = NULL;

    // ★ 从 fwnode 读 "default-state" 字符串属性
    if (!fwnode_property_read_string(fwnode, "default-state", &state)) {
        if (!strcmp(state, "keep"))
            return LEDS_DEFSTATE_KEEP;
        if (!strcmp(state, "on"))
            return LEDS_DEFSTATE_ON;      // ← "on" → 返回此值
    }

    return LEDS_DEFSTATE_OFF;             // 缺省值：off
}
```

`fwnode_property_read_string()`（`drivers/base/property.c:392`）是固件无关的属性读取 API，对于 OF 设备，内部调用 `of_property_read_string()` → 直接从 DTB 中的字符串表读取。

#### label 属性解析：`"sys_led_red"` → LED 设备名

`label` 不由 `leds-gpio.c` 直接读取，而是通过 LED core 的 `led_compose_name()` 处理：

文件：`common/drivers/leds/led-core.c`

```c
// 简化逻辑
led_compose_name(parent, init_data, fwnode)
  → fwnode_property_read_string(fwnode, "label", &name)  // 读 "label" 属性
  → 若不存在 label，则用 function + color 或 node 名拼接
  → 返回 name → 用于创建 /sys/class/leds/<name>/
```

#### compatible 属性匹配：`"gpio-leds"` → 驱动 probe

这不由驱动代码主动调用，而是由 platform bus 框架自动完成：

```
platform 总线枚举 DTS 节点
  → of_match_device(drv->of_match_table, dev->of_node)
    → __of_match_node(matches, node)
      遍历 matches[]，对每个 entry:
        → of_compat_cmp(node, match->compatible)
          比较 DTS compatible 字符串与 match->compatible
      匹配成功 → 返回匹配的 of_device_id*
  → 调用驱动的 .probe() 回调
```

文件：`drivers/of/device.c`

### 总结：DTS → 驱动 数据通路全景

```
DTS 文件 (源码)                内核 API                         驱动变量
────────────────────────────────────────────────────────────────────────
compatible = "gpio-leds"  →  of_match_device()             → 触发 probe

label = "sys_led_red"     →  fwnode_property_read_string() → cdev.name

gpios = <&gpio            →  of_parse_phandle_with_args_map()  → gpiospec
  GPIOD_4                 →                                    → args[0]=28
  GPIO_ACTIVE_LOW>        →                                    → args[1]=1
                          →  of_find_gpiochip_by_xlate()    → gpiochip0
                          →  chip->of_xlate()               → offset=28
                          →  gpiochip_get_desc()            → gpio_desc*

default-state = "on"      →  fwnode_property_read_string()  → LEDS_DEFSTATE_ON

retain-state-suspended    →  fwnode_property_present()      → bool true
```

所有 API 的入口都是 `fwnode_*`（固件节点），对 OF 设备自动转发到 `of_*` 实现。这样驱动代码不必关心底层是 OF 还是 ACPI——同一套 API 通吃。

#### 第 3 步：驱动匹配

文件：`common/drivers/leds/leds-gpio.c:187-308`

```c
static const struct of_device_id of_gpio_leds_match[] = {
    { .compatible = "gpio-leds" },    // ← 匹配 DTS
    {},
};
static struct platform_driver gpio_led_driver = {
    .probe  = gpio_led_probe,
    .driver = {
        .name   = "leds-gpio",
        .of_match_table = of_gpio_leds_match,
    },
};
module_platform_driver(gpio_led_driver);
```

触发：内核启动 → `of_platform_populate()` → 找到 `compatible = "gpio-leds"` → `gpio_led_probe()`。

#### 第 4 步：GPIO 获取

文件：`common/drivers/leds/leds-gpio.c:130-185`

```c
static struct gpio_leds_priv *gpio_leds_create(struct platform_device *pdev)
{
    device_for_each_child_node(dev, child) {
        // ★ 从 DTS 子节点获取 gpio_desc
        led.gpiod = devm_fwnode_get_gpiod_from_child(dev, NULL, child,
                                                     GPIOD_ASIS, NULL);
        //  内部调用链：
        //    → gpiod_get_from_of_node()
        //      → of_get_named_gpiod_flags()  解析 "gpios" 属性
        //        → of_parse_phandle_with_args()  <&gpio 28 GPIO_ACTIVE_LOW>
        //        → gpio_to_desc(28) → gpio_desc*

        led.default_state = led_init_default_state_get(child);
        create_gpio_led(&led, led_dat, dev, child, NULL);
    }
}
```

#### 第 5 步：LED 注册 → sysfs

文件：`common/drivers/leds/leds-gpio.c:75-123`

```c
static int create_gpio_led(...) {
    led_dat->cdev.brightness_set = gpio_led_set;   // 回调函数
    led_dat->cdev.max_brightness = 1;              // GPIO LED 只有 0/1

    state = (template->default_state == LEDS_GPIO_DEFSTATE_ON);  // "on" → 1
    gpiod_direction_output(led_dat->gpiod, state);  // 配置 GPIO 输出

    devm_led_classdev_register_ext(parent, &led_dat->cdev, &init_data);
    // → 创建 /sys/class/leds/sys_led_red/
}
```

#### 第 6 步：用户态 → 硬件寄存器（完整代码调用链）

```
用户态: echo 1 > /sys/class/leds/sys_led_red/brightness
  │
  ▼
[1] common/drivers/leds/led-class.c:38-65  brightness_store()
      kstrtoul("1") → state = 1
      led_set_brightness(led_cdev, 1)
        │
        ▼
[2] common/drivers/leds/leds-gpio.c:35-56  gpio_led_set(cdev, LED_ON)
      level = (value == LED_OFF) ? 0 : 1   →  level = 1
      gpiod_set_value(led_dat->gpiod, 1)
        │
        ▼
[3] common/drivers/gpio/gpiolib.c           gpiod_set_value_nocheck(desc, 1)
      → 检测 FLAG_ACTIVE_LOW:
        value = !value                       →  物理值 = 0（翻转！）
      → gpiod_set_raw_value_commit(desc, 0)
        │
        ▼
[4] common/drivers/gpio/gpiolib.c:2854     gc = desc->gdev->chip
      gc->set(gc, gpio_chip_hwgpio(desc), 0)
        │
        ▼
[5] common/common_drivers/drivers/gpio/
    pinctrl/pinctrl-meson.c:582            meson_gpio_set(chip, 28, 0)
      → meson_get_bank(pc, 28, &bank)       pin 28 ∈ [24,28] → Bank D
      → meson_calc_reg_and_bit(...)          reg = 0x0C4, bit = 4
      → regmap_update_bits(reg_gpio,         BIT(4) = 0 → 引脚拉低
                            0x0C4, BIT(4), 0)
        │
        ▼
[6] 硬件                                  GPIO 输出寄存器 @ 0xFE0040C4
                                            bit4 写 0 → GPIOD_4 物理电平 LOW
                                            因为 active-low 接线：
                                            LOW on 阴极 → LED 亮 ✓
```

`GPIO_ACTIVE_LOW` 的含义：`gpios = <&gpio GPIOD_4 GPIO_ACTIVE_LOW>` 告诉内核"低电平有效"。内核在 `gpiod_set_value_nocheck()` 中自动翻转：用户写 `brightness=1` → `gpiod_set_value(1)` → 检测 `FLAG_ACTIVE_LOW` → `value = 0` → 物理写 LOW → LED 亮。调用者无需关心 active-high/low。

### fun_led.ko 为何失败（-EPROBE_DEFER 根因）

**层面 1：资源冲突（根本原因）**

GPIOD_4 已在 DTS 中声明为 `gpio-leds` 的子节点 `sys_led_red`。内核启动时 `leds-gpio` 驱动通过 `gpiod_get()` 申请了该 pin。当 `fun_led.ko` 再尝试 `gpio_request(28)` 时，gpiolib 检测到 `FLAG_REQUESTED` 已置位 → 应返回 `-EBUSY`。

**层面 2：为什么报 -517 而非 -EBUSY**

`gpiod_request()` 返回值初始值就是 `-EPROBE_DEFER`（`gpiolib.c:1988`）：

```c
int gpiod_request(struct gpio_desc *desc, const char *label)
{
    int ret = -EPROBE_DEFER;           // ← 初始值 = -517
    if (try_module_get(gdev->owner)) { // ← GPIO chip owner 模块可用？
        ret = gpiod_request_commit(desc, label);  // 进入实际申请
    }
    return ret;
}
```

在 GKI 架构下，gpio_chip 的 `owner` 指向 vendor 模块。`try_module_get()` 失败时维持 -517，不会进入 `gpiod_request_commit()` → 不会触发"已占用"的 `-EBUSY` 检查。

`fun_led.ko` 被**两层拒绝**：第一层模块依赖检查（-517），第二层资源冲突（-EBUSY）。最终返回第一层的 -517。

### 两种方案对比

```
                    硬编码版（fun_led.c）       DTS + gpio-leds（现成方案）
                    ────────────────────       ──────────────────────────
GPIO 信息位置       驱动代码 #define LED_GPIO    DTS gpios = <&gpio GPIOD_4 ...>

GPIO 标识方式       数字 28                     名称 GPIOD_4（跨 SoC 不变）

驱动代码            需要自己写                   内核已有，零代码

用户接口            /dev/fun_led                /sys/class/leds/sys_led_red/
                    echo 1 > /dev/fun_led       echo 1 > brightness

接口能力            只有 0/1                    支持 trigger（timer/heartbeat/...）
                                               支持 default-state

申请 GPIO 方式      gpio_request(28, ...)        gpiod_get(dev, "led", ...)
                    legacy 整数 API             gpiod consumer API

驱动入口            module_init()               module_platform_driver()
                    手动 insmod                 compatible 自动触发 probe

可移植性            差                           好
                    换 SoC 改 C 代码             换板子只改 DTS
```

---

## 4. 深入理解：gpio_set_value(28, 0) 如何定位到硬件寄存器

从 **全局 GPIO 编号 28** 到 **硅片上的寄存器 bit**，经过 4 层查表。

### 形象类比：写字楼找工位

```
你        gpio_set_value(28, 0)
        │
        ▼
前台      gpio_to_desc(28)
        │ "28 号员工？我帮你查..."
        │  遍历 gpio_devices 链表
        │  找到负责 #28 的 gpiochip → 返回工牌（gpio_desc）
        │
        ▼
工牌      gpio_desc {
        │   .gdev→chip = 指向管理这个 pin 的 gpio_chip
        │   硬件编号    = 28（chip 内的 offset）
        │ }
        │
        ▼
部门经理  gc->set(chip, 28, 0)
        │ "我是 GPIO 控制器，我来处理 #28"
        │  调用 Amlogic pinctrl 驱动的 meson_gpio_set()
        │
        ▼
楼层表    meson_get_bank(pin=28)
        │  查 bank 表 ←── 这就是你要找的「偏移表」！
        │  ┌─────────────────────────────────────────┐
        │  │ Bank │ first  │ last  │ out寄存器 │ bit  │
        │  │ E    │   0    │   1   │  0x041   │   0  │
        │  │ B    │   2    │  15   │  0x061   │   0  │
        │  │ C    │  16    │  23   │  0x051   │   0  │
        │  │ D    │  24    │  28   │  0x031   │   0  │ ← pin28 命中!
        │  │ DV   │  29    │  35   │  0x071   │   0  │
        │  │ ...  │  ...   │  ...  │   ...    │  ... │
        │  └─────────────────────────────────────────┘
        │
        ▼
计算器    meson_calc_reg_and_bit()
        │  pin = 28, bank->first = 24
        │  bit = (0 + 28 - 24) × 1 = 4
        │  reg = (0x031 + 4/32) × 4 = 0x0C4
        │
        ▼
门禁刷卡  regmap_update_bits(reg_gpio, 0x0C4, BIT(4), 0)
        │  绝对地址 = 0xfe0040c0 + 0x0C4 = 0xfe004184
        │  把 bit4 写 0 → 引脚拉低
```

### 代码调用链（与上面一一对应）

```
gpio_set_value(28, 0)                       ← gpio.h:69，代码入口
  gpiod_set_raw_value(gpio_to_desc(28), 0)  ← 前台：查号 → 拿工牌
    gpio_to_desc(28)                        ← gpiolib.c:108，遍历 gpio_devices
      → 找到 gdev，返回 &gdev->descs[28 - gdev->base]
    gpiod_set_raw_value_commit(desc, 0)     ← gpiolib.c:2854
      gc = desc->gdev->chip                 ← 取 gpio_chip
      gc->set(gc, gpio_chip_hwgpio(desc), 0)  ← 调部门经理
        meson_gpio_set(chip, 28, 0)         ← pinctrl-meson.c:582
          meson_pinconf_set_drive(pc, 28, 0)
            meson_pinconf_set_gpio_bit(pc, 28, REG_OUT, 0)
              meson_get_bank(pc, 28, &bank) ← 查楼层表：pin28→Bank D
              meson_calc_reg_and_bit()      ← 算出 reg=0x0C4, bit=4
              regmap_update_bits(..., 0x0C4, BIT(4), 0)  ← 写硬件!
```

### bank 表在代码中的具体位置

**S7D 表定义** — `common/common_drivers/drivers/gpio/pinctrl/pinctrl-meson-s7d.c:1323-1343`：

```c
static struct meson_bank meson_s7d_periphs_banks[] = {
    //   name   first       last        ... dir        out         in
    BANK_DS("E",  GPIOE_0,  GPIOE_1,   ... 0x042,0,  0x041,0,  0x040,0),
    BANK_DS("B",  GPIOB_0, GPIOB_13,   ... 0x062,0,  0x061,0,  0x060,0),
    BANK_DS("C",  GPIOC_0,  GPIOC_7,   ... 0x052,0,  0x051,0,  0x050,0),
    BANK_DS("D",  GPIOD_0,  GPIOD_4,   ... 0x032,0,  0x031,0,  0x030,0),  ← pin 28
    BANK_DS("DV", GPIODV_0, GPIODV_6,  ... 0x072,0,  0x071,0,  0x070,0),
    BANK_DS("H",  GPIOH_0, GPIOH_11,   ... 0x022,0,  0x021,0,  0x020,0),
    BANK_DS("X",  GPIOX_0, GPIOX_19,   ... 0x012,0,  0x011,0,  0x010,0),
    BANK_DS("Z",  GPIOZ_0, GPIOZ_12,   ... 0x002,0,  0x001,0,  0x000,0),
};
```

**查表函数** — `common/drivers/pinctrl/meson/pinctrl-meson.c:72-106`：

```c
static int meson_get_bank(struct meson_pinctrl *pc, unsigned int pin,
                          struct meson_bank **bank) {
    for (i = 0; i < pc->data->num_banks; i++)
        if (pin >= pc->data->banks[i].first &&
            pin <= pc->data->banks[i].last) {
            *bank = &pc->data->banks[i];
            return 0;
        }
}

static void meson_calc_reg_and_bit(struct meson_bank *bank, unsigned int pin,
                                   enum meson_reg_type reg_type,
                                   unsigned int *reg, unsigned int *bit) {
    struct meson_reg_desc *desc = &bank->regs[reg_type];
    *bit = (desc->bit + pin - bank->first) * meson_bit_strides[reg_type];
    *reg = (desc->reg + (*bit / 32)) * 4;
    *bit &= 0x1f;
}
```

### 为什么 GPIOD_4(28) 不会和 GPIOB_4(6) 或 GPIOC_4(20) 冲突？

bank 表用 **first/last 划定了唯一区间**：

```
GPIOD_4 = 28 → 28 ∈ [24, 28] → Bank D
GPIOB_4 = 6  →  6 ∈ [ 2, 15] → Bank B
GPIOC_4 = 20 → 20 ∈ [16, 23] → Bank C
```

不同 bank → 不同寄存器偏移 → 不同 bit。全局编号 28 只属于 Bank D。

---

## 5. /sys/class/leds/sys_led_red/ 下的文件是怎么来的？

```
/sys/class/leds/sys_led_red/
├── brightness        ← LED class 属性组
├── max_brightness    ← LED class 属性组
├── trigger           ← LED trigger 属性组
├── device            ← 设备模型 symlink（内核通用）
├── subsystem         ← 设备模型 symlink（内核通用）
├── uevent            ← 设备模型标准属性（内核通用）
└── power/            ← 电源管理 sysfs（内核通用）
```

三类来源，由 `device_add()` 一站式组装。

### ① LED class 属性（brightness、max_brightness、trigger）

文件：`common/drivers/leds/led-class.c:77-104`

```c
// 普通属性组
static struct attribute *led_class_attrs[] = {
    &dev_attr_brightness.attr,           // DEVICE_ATTR_RW(brightness)
    &dev_attr_max_brightness.attr,       // DEVICE_ATTR_RO(max_brightness)
    NULL,
};
static const struct attribute_group led_group = {
    .attrs = led_class_attrs,
};

// trigger 是二进制属性（内容可能很长），单独成组
static BIN_ATTR(trigger, 0644, led_trigger_read, led_trigger_write, 0);
static const struct attribute_group led_trigger_group = {
    .bin_attrs = led_trigger_bin_attrs,
};

// 打包注册到 class
static const struct attribute_group *led_groups[] = {
    &led_group,           // → brightness, max_brightness
    &led_trigger_group,   // → trigger
    NULL,
};
```

绑定发生在 `led-class.c:539`：`leds_class->dev_groups = led_groups;`

设备创建时，`device_add()` 自动应用 class 的 `dev_groups`。

### ② 设备模型标准文件（uevent、device、subsystem）

文件：`common/drivers/base/core.c:3361-3368`

```c
int device_add(struct device *dev)
{
    error = device_create_file(dev, &dev_attr_uevent);   // → uevent
    error = device_add_class_symlinks(dev);               // → device, subsystem
    error = device_add_attrs(dev);                        // → 应用 ① 的 led_groups
    error = dpm_sysfs_add(dev);                           // → power/
}
```

`device_add_class_symlinks()`（`core.c:3122-3150`）：
```c
sysfs_create_link(&dev->kobj, &dev->class->p->subsys.kobj, "subsystem");
sysfs_create_link(&dev->kobj, &dev->parent->kobj, "device");
```

### ③ power/ 目录

文件：`common/drivers/base/power/sysfs.c`

`dpm_sysfs_add()` 创建 `power/` 子目录，内含 `control`、`autosuspend_delay_ms`、`wakeup` 等文件。

### 整体注册链

```
devm_led_classdev_register_ext()                 ← leds-gpio.c 调用
  → device_create_with_groups(leds_class, ...)   ← 传入 led_groups
    → device_register()
      → device_add()                              ← ★ 核心函数
        ├─ device_create_file(uevent)             → uevent
        ├─ device_add_class_symlinks()            → device, subsystem
        ├─ device_add_attrs()                     → 遍历 class->dev_groups
        │   ├─ led_group                          → brightness, max_brightness
        │   └─ led_trigger_group                  → trigger
        ├─ dpm_sysfs_add()                        → power/
        └─ kobject_uevent(KOBJ_ADD)               → 通知 udev
```

**设计要点**：驱动只声明"这类设备有什么特有属性"（`dev_groups`），通用文件由内核设备模型统一提供。一套机制，所有 class 共用。

---

## 6. 如何查询 GPIO 名称的全局编号？

问题：原理图标 `GPIOD_4`，但 API 要整数。怎么知道 `GPIOD_4` = 28？

答案在 **dt-bindings 头文件** — `common/common_drivers/include/dt-bindings/gpio/meson-s7d-gpio.h`：

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
#define GPIOD_4  28      ← S7D 上 GPIOD_4 = 28

/* GPIOX */
#define GPIOX_0  48
...
```

**编号规则**：全局编号 = 该 pin 在 `meson_s7d_periphs_pins[]` 数组中的索引位置。S7D 顺序为 E→B→C→D→DV→H→X→Z→TEST_N→CC（83 pins）。

**查找方法**：
```bash
grep GPIOD_4 common/common_drivers/include/dt-bindings/gpio/meson-s7d-gpio.h
grep "MESON_PIN" common/common_drivers/drivers/gpio/pinctrl/pinctrl-meson-s7d.c | head -35
```

**不同 SoC 编号完全不同**：

| 芯片 | GPIOD_4 编号 | GPIOA_4 编号 | 总 pins |
|------|-------------|-------------|---------|
| S6   | 4           | 86          | 101     |
| S7D  | 28          | 不存在      | 83      |
