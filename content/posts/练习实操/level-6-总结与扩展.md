+++
title = 'Level 6 总结与扩展：逐函数拆解 GPIO 按键驱动'
date = '2026-06-09T15:04:00+08:00'
draft = false
+++

# Level 6 总结与扩展：逐函数拆解 GPIO 按键驱动

本文是 [level-6-按键.md](level-6-按键.md) 的延伸，以 Amlogic `gpio_keypad.c`（557 行）为样本，**按照代码的物理顺序，逐函数解释它干了什么**。

读完本文你会：能自己读懂一个陌生驱动，知道每个函数在做什么。

驱动源文件：`common/common_drivers/drivers/input/keyboard/gpio_keypad.c`

---

## 代码总览（完整源码）

> 以下是 `gpio_keypad.c` 的完整代码。每个函数后面跟着逐行解释。

```c
// SPDX-License-Identifier: (GPL-2.0+ OR MIT)
/*
 * Copyright (c) 2019 Amlogic, Inc. All rights reserved.
 */

#include <linux/module.h>
#include <linux/init.h>
#include <linux/types.h>
#include <linux/string.h>
#include <linux/kstrtox.h>
#include <linux/input.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>
#include <linux/slab.h>
#include <linux/amlogic/pm.h>
#include <linux/irq.h>
#include <linux/amlogic/power_domain.h>
#include <linux/amlogic/gpiolib.h>

#define DEFAULT_SCAN_PERION	20
#define	DEFAULT_POLL_MODE	0
#define KEY_JITTER_COUNT	1

struct pin_desc {
	int current_status;
	struct gpio_desc *desc;
	int irq_num;
	u32 code;
	u32 key_type;
	const char *name;
	int count;
	bool ignore;
};

struct gpio_keypad {
	int key_size;
	int use_irq;/* 1:irq mode ; 0:polling mode */
	int scan_period;
	struct pin_desc *key;
	struct pin_desc *current_key;
	struct timer_list polling_timer;
	struct input_dev *input_dev;
	struct class kp_class;
	struct class resetkey_class;
	unsigned int reset_count;
};

static irqreturn_t gpio_irq_handler(int irq, void *data)
{
	struct gpio_keypad *keypad;

	keypad = (struct gpio_keypad *)data;
	mod_timer(&keypad->polling_timer,
		  jiffies + msecs_to_jiffies(20));
	return IRQ_HANDLED;
}

static void report_key_code(struct gpio_keypad *keypad, int gpio_val)
{
	struct pin_desc *key = keypad->current_key;

	if (key->count >= KEY_JITTER_COUNT) {
		key->current_status = gpio_val;
		if (key->current_status) {
			input_event(keypad->input_dev, key->key_type,
				    key->code, 0);

			dev_dbg(&keypad->input_dev->dev,
				 "key %d up.\n", key->code);
		} else {
			input_event(keypad->input_dev, key->key_type,
				    key->code, 1);
			// Count key press
			if (strcmp(key->name, "bluetooth") == 0) {
				keypad->reset_count++;
			}

			dev_dbg(&keypad->input_dev->dev,
				 "key %d down.\n", key->code);
		}
		input_sync(keypad->input_dev);

		key->count = 0;
	}
}

static void polling_timer_handler(struct timer_list *t)
{
	int i;
	int gpio_val;
	int is_pressing = 0;
	struct gpio_keypad *keypad = from_timer(keypad, t, polling_timer);

	if (keypad->use_irq) {/* irq mode */
		for (i = 0; i < keypad->key_size; i++) {
			gpio_val = gpiod_get_value(keypad->key[i].desc);
			gpio_val |= keypad->key[i].ignore;

			if (keypad->key[i].current_status != gpio_val) {
				keypad->key[i].count++;
				keypad->current_key = &keypad->key[i];
				report_key_code(keypad, gpio_val);
			} else {
				keypad->key[i].count = 0;
			}

			if (gpio_val == 0 || keypad->key[i].current_status == 0)
				is_pressing = 1;
		}
		if (is_pressing)
			mod_timer(&keypad->polling_timer,
				  jiffies +
				  msecs_to_jiffies(keypad->scan_period));
	} else {/* polling mode */
		for (i = 0; i < keypad->key_size; i++) {
			gpio_val = gpiod_get_value(keypad->key[i].desc);
			gpio_val |= keypad->key[i].ignore;
			if (keypad->key[i].current_status != gpio_val) {
				keypad->key[i].count++;
				keypad->current_key = &keypad->key[i];
				report_key_code(keypad, gpio_val);
			} else {
				keypad->key[i].count = 0;
			}
		}
		mod_timer(&keypad->polling_timer,
			  jiffies + msecs_to_jiffies(keypad->scan_period));
	}
}

static ssize_t table_show(struct class *cls, struct class_attribute *attr,
			  char *buf)
{
	struct gpio_keypad *keypad = container_of(cls,
					struct gpio_keypad, kp_class);
	int i;
	int len = 0;

	for (i = 0; i < keypad->key_size; i++) {
		len += sprintf(buf + len,
			"[%d]: name = %-21s status = %-5d\n", i,
			keypad->key[i].name,
			keypad->key[i].current_status);
	}

	return len;
}

static ssize_t ignore_show(struct class *cls, struct class_attribute *attr, char *buf)
{
	struct gpio_keypad *keypad = container_of(cls, struct gpio_keypad, kp_class);
	int index;
	int len = 0;

	for (index = 0; index < keypad->key_size; index++) {
		len += sysfs_emit_at(buf, len, "%s,%d: %c\n",
				     keypad->key[index].name,
				     keypad->key[index].code,
				     keypad->key[index].ignore ? 'Y' : 'N');
	}

	return len;
}

static ssize_t ignore_store(struct class *cls, struct class_attribute *attr,
			    const char *buf, size_t count)
{
	struct gpio_keypad *keypad = container_of(cls, struct gpio_keypad, kp_class);
	struct device *dev = keypad->input_dev->dev.parent;
	char nbuf[128];
	char tmp[32];
	int index;
	char *value = nbuf;
	char *name;
	bool ignore;
	bool found;

	if (count > sizeof(nbuf)) {
		dev_err(dev, "data is too long\n");
		return -EINVAL;
	}
	memcpy(nbuf, buf, count);
	strreplace(nbuf, '\n', '\0');

	name = strsep(&value, " ");
	if (!name)
		goto err;

	if (!strcmp("all", name)) {
		for (index = 0; index < keypad->key_size; index++)
			keypad->key[index].ignore = true;
		dev_dbg(dev, "ignore all keys\n");
	} else if (!strcmp("none", name)) {
		for (index = 0; index < keypad->key_size; index++)
			keypad->key[index].ignore = false;
		dev_dbg(dev, "enable all keys\n");
	} else {
		if (!value)
			goto err;

		if (kstrtobool(value, &ignore)) {
			dev_err(dev, "invalid '%s', please use Y/N instead\n", value);
			return -EINVAL;
		}

		/* search key */
		found = false;
		for (index = 0; index < keypad->key_size; index++) {
			snprintf(tmp, sizeof(tmp), "%s,%d",
				 keypad->key[index].name,
				 keypad->key[index].code);
			if (!strcmp(tmp, name)) {
				found = true;
				break;
			}
		}

		if (!found) {
			dev_err(dev, "invalid key '%s'\n", name);
			return -EINVAL;
		}
		keypad->key[index].ignore = ignore;
		dev_dbg(dev, "ignore %s: %s\n", name, value);
	}

	return count;
err:
	dev_err(dev, "the correct format is: [name],[code] [ignore]\n");
	return -EINVAL;
}

static CLASS_ATTR_RO(table);
static CLASS_ATTR_RW(ignore);

static struct attribute *meson_gpiokey_attrs[] = {
	&class_attr_table.attr,
	&class_attr_ignore.attr,
	NULL
};

ATTRIBUTE_GROUPS(meson_gpiokey);

// Add show function for reset count
static ssize_t count_show(struct class *cls, struct class_attribute *attr,
                         char *buf)
{
    struct gpio_keypad *keypad = container_of(cls,
                    struct gpio_keypad, resetkey_class);
    return sprintf(buf, "%u\n", keypad->reset_count);
}

static CLASS_ATTR_RO(count);

static struct attribute *resetkey_attrs[] = {
    &class_attr_count.attr,
    NULL
};

ATTRIBUTE_GROUPS(resetkey);


static int meson_gpio_kp_probe(struct platform_device *pdev)
{
	struct gpio_desc *desc;
	int ret, i;
	struct input_dev *input_dev;
	struct gpio_keypad *keypad;
	struct irq_desc *d;
	unsigned int number;
	int wakeup_source = 0;
	int register_flag = 0;

	if (!(pdev->dev.of_node)) {
		dev_err(&pdev->dev,
			"pdev->dev.of_node == NULL!\n");
		return -EINVAL;
	}
	keypad = devm_kzalloc(&pdev->dev,
			      sizeof(struct gpio_keypad), GFP_KERNEL);
	if (!keypad)
		return -EINVAL;
	ret = of_property_read_u32(pdev->dev.of_node,
				   "detect_mode", &keypad->use_irq);
	if (ret)
		/* The default mode is polling. */
		keypad->use_irq = DEFAULT_POLL_MODE;
	ret = of_property_read_u32(pdev->dev.of_node,
				   "scan_period", &keypad->scan_period);
	if (ret)
		/* he default scan period is 20. */
		keypad->scan_period = DEFAULT_SCAN_PERION;
	if (of_property_read_bool(pdev->dev.of_node, "wakeup-source"))
		wakeup_source = 1;
	ret = of_property_read_u32(pdev->dev.of_node,
				   "key_num", &keypad->key_size);
	if (ret) {
		dev_err(&pdev->dev,
			"failed to get key_num!\n");
		return -EINVAL;
	}
	keypad->key = devm_kzalloc(&pdev->dev,
				   (keypad->key_size) * sizeof(*keypad->key),
				   GFP_KERNEL);
	if (!(keypad->key))
		return -EINVAL;
	for (i = 0; i < keypad->key_size; i++) {
		/* get all gpio desc. */
		desc = devm_gpiod_get_index(&pdev->dev, "key", i, GPIOD_IN);
		if (IS_ERR_OR_NULL(desc))
			return -EINVAL;
		keypad->key[i].desc = desc;
		/* The gpio default is high level. */
		keypad->key[i].current_status = 1;
		ret = of_property_read_u32_index(pdev->dev.of_node,
						 "key_code", i,
						 &keypad->key[i].code);
		if (ret < 0) {
			dev_err(&pdev->dev,
				"find key_code=%d finished\n", i);
			return -EINVAL;
		}

		ret = of_property_read_u32_index(pdev->dev.of_node,
						 "key_type", i,
						 &keypad->key[i].key_type);
		if (ret)
			keypad->key[i].key_type = EV_KEY;

		ret = of_property_read_string_index(pdev->dev.of_node,
						    "key_name", i,
						    &keypad->key[i].name);
		if (ret < 0) {
			dev_err(&pdev->dev,
				"find key_name=%d finished\n", i);
			return -EINVAL;
		}
		gpiod_direction_input(keypad->key[i].desc);
		gpiod_set_pull(keypad->key[i].desc, GPIOD_PULL_UP);
	}

	keypad->kp_class.name = "gpio_keypad";
	keypad->kp_class.owner = THIS_MODULE;
	keypad->kp_class.class_groups = meson_gpiokey_groups;
	ret = class_register(&keypad->kp_class);
	if (ret) {
		dev_err(&pdev->dev, "fail to create gpio keypad class.\n");
		return -EINVAL;
	}

	// After registering kp_class, add resetkey class registration
	keypad->resetkey_class.name = "resetkey";
	keypad->resetkey_class.owner = THIS_MODULE;
	keypad->resetkey_class.class_groups = resetkey_groups;
	keypad->reset_count = 0;
	ret = class_register(&keypad->resetkey_class);
	if (ret) {
		dev_err(&pdev->dev, "fail to create resetkey class.\n");
		class_unregister(&keypad->kp_class);
		return -EINVAL;
	}

	/* input */
	input_dev = input_allocate_device();
	if (!input_dev)
		return -EINVAL;
	for (i = 0; i < keypad->key_size; i++) {
		input_set_capability(input_dev, keypad->key[i].key_type,
				     keypad->key[i].code);

		dev_dbg(&pdev->dev, "%s key(%d) type(0x%x) registered.\n",
			 keypad->key[i].name, keypad->key[i].code,
			 keypad->key[i].key_type);
	}
	input_dev->name = "gpio_keypad";
	input_dev->phys = "gpio_keypad/input0";
	input_dev->dev.parent = &pdev->dev;
	input_dev->id.bustype = BUS_ISA;
	input_dev->id.vendor = 0x0001;
	input_dev->id.product = 0x0001;
	input_dev->id.version = 0x0100;
	input_dev->rep[REP_DELAY] = 0xffffffff;
	input_dev->rep[REP_PERIOD] = 0xffffffff;
	input_dev->keycodesize = sizeof(unsigned short);
	input_dev->keycodemax = 0x1ff;
	keypad->input_dev = input_dev;
	ret = input_register_device(keypad->input_dev);
	if (ret < 0) {
		input_free_device(keypad->input_dev);
		return -EINVAL;
	}
	platform_set_drvdata(pdev, keypad);
	timer_setup(&keypad->polling_timer,
		    polling_timer_handler, 0);
	if (keypad->use_irq) {
		for (i = 0; i < keypad->key_size; i++) {
			keypad->key[i].count = 0;
			keypad->key[i].irq_num =
				gpiod_to_irq(keypad->key[i].desc);
			ret = devm_request_irq(&pdev->dev,
					       keypad->key[i].irq_num,
					       gpio_irq_handler,
					       IRQF_TRIGGER_FALLING,
					       "gpio_keypad", keypad);
			if (ret) {
				dev_err(&pdev->dev,
					"Requesting irq failed!\n");
				input_free_device(keypad->input_dev);
				del_timer(&keypad->polling_timer);
				return -EINVAL;
			}
			if (keypad->key[i].code == KEY_POWER) {
				d = irq_to_desc(keypad->key[i].irq_num);
				if (d) {
					number =
					 d->irq_data.parent_data->hwirq - 32;
					pwr_ctrl_irq_set(number, 1, 0);
					register_flag = 1;
				}
			}
		}
	} else {
		mod_timer(&keypad->polling_timer,
			  jiffies + msecs_to_jiffies(keypad->scan_period));
	}
	if (wakeup_source) {
		if (register_flag)
			dev_dbg(&pdev->dev,
				 "succeed to register wakeup source!\n");
		else
			dev_err(&pdev->dev,
				 "failed to register wakeup source!\n");
	}

	return 0;
}

static int meson_gpio_kp_remove(struct platform_device *pdev)
{
	struct gpio_keypad *keypad;

	keypad = platform_get_drvdata(pdev);
	class_unregister(&keypad->resetkey_class);
	class_unregister(&keypad->kp_class);
	input_unregister_device(keypad->input_dev);
	input_free_device(keypad->input_dev);
	del_timer(&keypad->polling_timer);
	return 0;
}

static const struct of_device_id key_dt_match[] = {
	{ .compatible = "amlogic, gpio_keypad", },
	{}
};

static int meson_gpio_kp_suspend(struct device *dev)
{
	struct gpio_keypad *pdata;

	pdata = (struct gpio_keypad *)dev_get_drvdata(dev);
	if (!pdata->use_irq)
		del_timer(&pdata->polling_timer);
	return 0;
}

static int meson_gpio_kp_resume(struct device *dev)
{
	int i;
	struct gpio_keypad *pdata;

	pdata = (struct gpio_keypad *)dev_get_drvdata(dev);
	if (!pdata->use_irq)
		mod_timer(&pdata->polling_timer,
			  jiffies + msecs_to_jiffies(5));
	if (get_resume_method() == POWER_KEY_WAKEUP) {
		for (i = 0; i < pdata->key_size; i++) {
			if (pdata->key[i].code == KEY_POWER) {
				dev_dbg(dev, "gpio keypad wakeup\n");
				input_report_key(pdata->input_dev,
						 KEY_POWER,  1);
				input_sync(pdata->input_dev);
				input_report_key(pdata->input_dev,
						 KEY_POWER,  0);
				input_sync(pdata->input_dev);
				break;
			}
		}
	}
	return 0;
}

static int meson_gpio_kp_restore(struct device *dev)
{
	struct gpio_keypad *keypad = dev_get_drvdata(dev);
	struct gpio_desc *desc;
	int index;

	for (index = 0; index < keypad->key_size; index++) {
		desc = devm_gpiod_get_index(dev, "key", index, GPIOD_IN);
		if (IS_ERR_OR_NULL(desc)) {
			dev_err(dev, "failed to request to gpio while restore\n");
			return -EINVAL;
		}
		gpiod_direction_input(desc);
		gpiod_set_pull(desc, GPIOD_PULL_UP);
		keypad->key[index].desc = desc;
	}

	return meson_gpio_kp_resume(dev);
}

static int meson_gpio_kp_freeze(struct device *dev)
{
	struct gpio_keypad *keypad = dev_get_drvdata(dev);
	int index;
	int ret;

	ret = meson_gpio_kp_suspend(dev);
	if (ret)
		return ret;

	for (index = 0; index < keypad->key_size; index++)
		devm_gpiod_put(dev, keypad->key[index].desc);

	return 0;
}

static const struct dev_pm_ops meson_gpio_kp_pm_ops = {
	.suspend = meson_gpio_kp_suspend,
	.resume = meson_gpio_kp_resume,
	.freeze = meson_gpio_kp_freeze,
	.thaw = meson_gpio_kp_resume,
	.poweroff = meson_gpio_kp_suspend,
	.restore = meson_gpio_kp_restore,
};

static struct platform_driver meson_gpio_kp_driver = {
	.probe = meson_gpio_kp_probe,
	.remove = meson_gpio_kp_remove,
	.driver = {
		.name = "gpio-keypad",
		.pm = &meson_gpio_kp_pm_ops,
		.of_match_table = key_dt_match,
	},
};

int __init meson_gpio_kp_init(void)
{
	return platform_driver_register(&meson_gpio_kp_driver);
}

void __exit meson_gpio_kp_exit(void)
{
	platform_driver_unregister(&meson_gpio_kp_driver);
}
```

---

# 逐函数解释

以下按代码物理顺序，逐个函数解释它干了什么。

---

## 1. `#include` 头文件 — 引入外部依赖

```c
#include <linux/module.h>       // 提供 MODULE_AUTHOR、module_init() 等宏
#include <linux/init.h>         // 提供 __init、__exit 宏（标记初始化/退出函数）
#include <linux/types.h>        // 提供 u32 类型（无符号 32 位整数）
#include <linux/string.h>       // 提供 strcmp()、strsep()、snprintf()、memcpy()
#include <linux/kstrtox.h>      // 提供 kstrtobool() —— 把字符串 "Y"/"N" 转成 bool
#include <linux/input.h>        // 提供 input_dev、input_event()、EV_KEY、KEY_POWER 等
#include <linux/platform_device.h>  // 提供 platform_device、platform_driver 结构体
#include <linux/of.h>           // 提供 of_property_read_u32() 等 DTS 解析函数
#include <linux/gpio/consumer.h>// 提供 gpiod_get_index()、gpiod_get_value()、gpiod_set_pull()
#include <linux/interrupt.h>    // 提供 request_irq()、irqreturn_t
#include <linux/slab.h>         // 提供 devm_kzalloc() —— 自动释放的内存分配
#include <linux/amlogic/pm.h>   // Amlogic 私有：提供 get_resume_method()
#include <linux/irq.h>          // 提供 irq_to_desc() —— 通过 IRQ 号拿到 IRQ 描述符
#include <linux/amlogic/power_domain.h>  // Amlogic 私有：提供 pwr_ctrl_irq_set()
#include <linux/amlogic/gpiolib.h>       // Amlogic 私有：提供 gpiod_set_pull()
```

三个带 `amlogic` 的头文件是 Amlogic 自己的 API，主线内核里没有。`gpiod_set_pull` 用于设置 GPIO 内部上拉/下拉电阻。

---

## 2. 宏定义 — 硬编码的默认值

```c
#define DEFAULT_SCAN_PERION	20    // 默认轮询间隔 = 20 毫秒
#define	DEFAULT_POLL_MODE	0     // 默认检测模式 = polling（0=polling, 1=irq）
#define KEY_JITTER_COUNT	1     // 消抖阈值：连续 2 次（1+1）读到变化才确认
```

这三个值在 probe 中用作 fallback：如果 DTS 没写 `scan_period` 就用 20ms，没写 `detect_mode` 就用 polling。

---

## 3. `struct pin_desc` — 描述一个按键

```c
struct pin_desc {
    int current_status;          // 当前电平：1=高电平(释放), 0=低电平(按下)
    struct gpio_desc *desc;      // GPIO 描述符指针，gpiolib 用来操作这个 pin
    int irq_num;                 // 此 GPIO 对应的中断号（irq 模式下使用）
    u32 code;                    // Linux keycode，比如 KEY_POWER=116，或厂商自定义 600
    u32 key_type;                // 事件类型：EV_KEY（按键）或 EV_SW（开关）
    const char *name;            // 按键名字，比如 "bluetooth"、"mute"
    int count;                   // 消抖计数器：连续检测到变化的次数
    bool ignore;                 // 是否屏蔽此按键（true=按下也不上报事件）
};
```

这个结构体描述了一个按键的**全部信息**：
- 硬件信息 → `desc`（接在哪个 GPIO）
- 输入信息 → `code`、`key_type`（上报什么事件）
- 运行状态 → `current_status`、`count`（消抖用）

---

## 4. `struct gpio_keypad` — 描述整个按键设备

```c
struct gpio_keypad {
    int key_size;                     // 总共有几个按键（= DTS 中 key_num 的值）
    int use_irq;                      // 检测模式：0=polling, 1=irq
    int scan_period;                  // 轮询间隔（毫秒）
    struct pin_desc *key;             // 按键数组指针，长度 = key_size
    struct pin_desc *current_key;     // 当前正在处理的按键（消抖时指向它）
    struct timer_list polling_timer;  // 内核定时器，polling 模式用它定时扫描
    struct input_dev *input_dev;      // input 设备指针，用来向上层报告事件
    struct class kp_class;            // 在 /sys/class/gpio_keypad/ 创建属性文件
    struct class resetkey_class;      // 在 /sys/class/resetkey/ 创建属性文件
    unsigned int reset_count;         // 蓝牙键累计按下次数（恢复出厂用）
};
```

`struct pin_desc` 描述**一个**按键，`struct gpio_keypad` 描述**整个设备**（包含多个按键 + 运行参数 + sysfs 接口）。

---

## 5. `gpio_irq_handler()` — 中断来了干什么

```c
static irqreturn_t gpio_irq_handler(int irq, void *data)
{
    struct gpio_keypad *keypad;

    keypad = (struct gpio_keypad *)data;       // ① 从私有数据取出设备指针
    mod_timer(&keypad->polling_timer,           // ② 修改定时器到期时间
              jiffies + msecs_to_jiffies(20)); //    = 当前时间 + 20ms
    return IRQ_HANDLED;                         // ③ 告诉内核"中断已处理"
}
```

**它干了什么**：当一个 GPIO 下降沿触发中断后，这个函数被调用。它只做一件事——把轮询定时器的到期时间设为"20ms 后"。定时器触发后会去读 GPIO 真实电平。

**为什么不在中断里直接读 GPIO**：中断处理函数运行在**中断上下文**，不能调用可能睡眠的函数。`gpiod_get_value()` 对 I2C/SPI 扩展 GPIO 来说可能睡眠，所以推到定时器（软中断上下文）里做。此外，推到 20ms 后还顺便实现了消抖——机械按键的抖动通常在 5-20ms 内结束。

参数说明：
- `irq`：触发中断的中断号（内核传入，此函数没用到）
- `data`：注册中断时传入的私有指针（就是 `struct gpio_keypad *keypad`）

---

## 6. `report_key_code()` — 向 input 子系统上报一个按键事件

```c
static void report_key_code(struct gpio_keypad *keypad, int gpio_val)
{
    struct pin_desc *key = keypad->current_key;  // ① 拿到当前要处理的按键

    if (key->count >= KEY_JITTER_COUNT) {        // ② 消抖：连续 2 次变化才确认
        key->current_status = gpio_val;           // ③ 更新记录的当前电平
        if (key->current_status) {                // ④ 电平=1(高)，说明按键释放了
            input_event(keypad->input_dev, key->key_type,
                        key->code, 0);            //    上报事件，value=0（释放）
        } else {                                  // ⑤ 电平=0(低)，说明按键按下了
            input_event(keypad->input_dev, key->key_type,
                        key->code, 1);            //    上报事件，value=1（按下）
            if (strcmp(key->name, "bluetooth") == 0) {  // ⑥ 如果是蓝牙键
                keypad->reset_count++;                   //    累计按下次数+1
            }
        }
        input_sync(keypad->input_dev);            // ⑦ 发送同步事件，标记一帧结束
        key->count = 0;                           // ⑧ 重置消抖计数器
    }
}
```

**它干了什么**：检查消抖条件是否满足（连续 `KEY_JITTER_COUNT + 1 = 2` 次读到变化），满足就向 input 子系统报告按键按下（value=1）或释放（value=0）。如果按键名叫 "bluetooth"，额外累加 `reset_count`。

`input_event()` 的参数：
- `keypad->input_dev`：要上报到哪个 input 设备
- `key->key_type`：事件类型（`EV_KEY` = 按键，`EV_SW` = 开关）
- `key->code`：键码（比如蓝牙键是 600，mute 是 `SW_MUTE_DEVICE`）
- `0` 或 `1`：0 = 释放，1 = 按下

`input_sync()` 发送 `EV_SYN / SYN_REPORT`，告诉用户态"一帧完整事件结束"。用户态程序（如 `getevent`）靠它来分组事件。

---

## 7. `polling_timer_handler()` — 定时器到了干什么

```c
static void polling_timer_handler(struct timer_list *t)
{
    int i;
    int gpio_val;
    int is_pressing = 0;
    struct gpio_keypad *keypad = from_timer(keypad, t, polling_timer);
    //                                           ↑
    //    from_timer 是个宏：已知 timer_list 成员地址 t
    //    反推出包含它的 struct gpio_keypad 的地址
```

`from_timer(keypad, t, polling_timer)` 展开等价于：
```c
keypad = container_of(t, struct gpio_keypad, polling_timer);
//        已知 t 是 &keypad->polling_timer，反推 keypad 的地址
```

### irq 模式分支

```c
    if (keypad->use_irq) {
        for (i = 0; i < keypad->key_size; i++) {
            gpio_val = gpiod_get_value(keypad->key[i].desc);  // 读 GPIO 电平
            gpio_val |= keypad->key[i].ignore;  // 如果被屏蔽，强制当高电平(释放)

            if (keypad->key[i].current_status != gpio_val) {
                // 电平变了 → 消抖计数+1 → 尝试上报
                keypad->key[i].count++;
                keypad->current_key = &keypad->key[i];
                report_key_code(keypad, gpio_val);
            } else {
                // 电平没变 → 消抖计数清零（刚才只是抖动）
                keypad->key[i].count = 0;
            }

            // 如果有任何按键处于按下状态（读为 0 或 记录为 0）
            if (gpio_val == 0 || keypad->key[i].current_status == 0)
                is_pressing = 1;
        }
        if (is_pressing)
            // 有按键正在按 → 重新设定定时器，继续轮询
            mod_timer(&keypad->polling_timer,
                      jiffies + msecs_to_jiffies(keypad->scan_period));
        // 如果 is_pressing == 0（所有按键都释放了），不重新设定时器
        // → 定时器停止，下次按键由 GPIO 中断重新触发
```

### polling 模式分支

```c
    } else {
        for (i = 0; i < keypad->key_size; i++) {
            gpio_val = gpiod_get_value(keypad->key[i].desc);
            gpio_val |= keypad->key[i].ignore;
            if (keypad->key[i].current_status != gpio_val) {
                keypad->key[i].count++;
                keypad->current_key = &keypad->key[i];
                report_key_code(keypad, gpio_val);
            } else {
                keypad->key[i].count = 0;
            }
        }
        // 无条件重新设定定时器 → 永远每 scan_period ms 触发一次
        mod_timer(&keypad->polling_timer,
                  jiffies + msecs_to_jiffies(keypad->scan_period));
    }
}
```

**两种模式的本质区别在最后几行**：

| | irq 模式 | polling 模式 |
|---|---|---|
| 定时器何时停止 | 所有按键都释放后自动停止 | **永不停** |
| 定时器何时重启 | 下次 GPIO 中断触发后重启 | 不需要重启，一直跑 |
| CPU 空闲功耗 | 零 | 恒定（每 20ms 执行一次） |

---

## 8. `table_show()` — `/sys/class/gpio_keypad/table` 被读时干什么

```c
static ssize_t table_show(struct class *cls, struct class_attribute *attr,
                          char *buf)
{
    struct gpio_keypad *keypad = container_of(cls,
                            struct gpio_keypad, kp_class);
    //  已知 cls 是 keypad->kp_class 的地址，反推 keypad

    int i;
    int len = 0;

    for (i = 0; i < keypad->key_size; i++) {
        len += sprintf(buf + len,
                "[%d]: name = %-21s status = %-5d\n", i,
                keypad->key[i].name,             // 按键名字
                keypad->key[i].current_status);  // 当前电平
    }

    return len;  // 返回写入了多少字节
}
```

**它干了什么**：遍历所有按键，把每个按键的名字和当前电平写入 buf。用户执行 `cat /sys/class/gpio_keypad/table` 时会调用这个函数。

输出示例：
```
[0]: name = bluetooth           status = 1
[1]: name = mute                status = 1
```

`%-21s` 表示左对齐占 21 个字符宽度，`%-5d` 表示左对齐占 5 个数字宽度。

---

## 9. `ignore_show()` — `/sys/class/gpio_keypad/ignore` 被读时干什么

```c
static ssize_t ignore_show(struct class *cls, struct class_attribute *attr,
                           char *buf)
{
    struct gpio_keypad *keypad = container_of(cls, struct gpio_keypad, kp_class);
    int index;
    int len = 0;

    for (index = 0; index < keypad->key_size; index++) {
        len += sysfs_emit_at(buf, len, "%s,%d: %c\n",
                             keypad->key[index].name,    // 按键名
                             keypad->key[index].code,    // 键码
                             keypad->key[index].ignore ? 'Y' : 'N');  // 是否屏蔽
    }

    return len;
}
```

**它干了什么**：遍历所有按键，输出每个按键的"名字,键码: 是否屏蔽"。用户执行 `cat /sys/class/gpio_keypad/ignore` 时调用。

输出示例：
```
bluetooth,600: N
mute,248: N
```

`sprintf` 和 `sysfs_emit_at` 的区别：`sysfs_emit_at` 是 `sprintf` 的安全替代，能防止 buffer overflow，sysfs 函数中应该优先使用。

---

## 10. `ignore_store()` — 向 `/sys/class/gpio_keypad/ignore` 写入时干什么

这是驱动中最长的函数（66 行），因为要处理用户输入的各种情况。

```c
static ssize_t ignore_store(struct class *cls, struct class_attribute *attr,
                            const char *buf, size_t count)
{
    struct gpio_keypad *keypad = container_of(cls, struct gpio_keypad, kp_class);
    struct device *dev = keypad->input_dev->dev.parent;  // 用来打日志
    char nbuf[128];           // 栈上的临时缓冲区
    char tmp[32];             // 用来拼 "name,code" 字符串
    int index;
    char *value = nbuf;       // 指向空格后面的部分（"Y" 或 "N"）
    char *name;               // 指向空格前面的部分（"name,code" 或 "all"/"none"）
    bool ignore;
    bool found;
```

### 第 1 步：防溢出 + 去换行

```c
    if (count > sizeof(nbuf)) {                    // 防止栈溢出
        dev_err(dev, "data is too long\n");
        return -EINVAL;
    }
    memcpy(nbuf, buf, count);                      // 拷到栈缓冲区
    strreplace(nbuf, '\n', '\0');                  // 把换行符替换为 '\0'（字符串结束符）
```

### 第 2 步：用空格分割输入

```c
    name = strsep(&value, " ");                    // 分割：空格前面 → name，空格后面 → value
    if (!name)
        goto err;                                  // 没有空格 → 格式错误
```

`strsep(&value, " ")` 在第一个空格处切断字符串，返回空格前的部分，`value` 指针移动到空格后面。

用户写入 `"bluetooth,600 Y\n"` 时：
```
切割前: nbuf = "bluetooth,600 Y\0"
切割后: name  → "bluetooth,600"
        value → "Y"
```

### 第 3 步：分支处理

```c
    if (!strcmp("all", name)) {                    // 写入 "all 1"
        for (index = 0; index < keypad->key_size; index++)
            keypad->key[index].ignore = true;
    } else if (!strcmp("none", name)) {            // 写入 "none 1"
        for (index = 0; index < keypad->key_size; index++)
            keypad->key[index].ignore = false;
    } else {                                       // 写入 "name,code Y"
        if (!value)
            goto err;                              // 没有 Y/N 部分

        if (kstrtobool(value, &ignore)) {          // "Y"/"1" → true, "N"/"0" → false
            dev_err(dev, "invalid '%s', please use Y/N instead\n", value);
            return -EINVAL;
        }

        // 在按键数组中搜索匹配的 name,code
        found = false;
        for (index = 0; index < keypad->key_size; index++) {
            snprintf(tmp, sizeof(tmp), "%s,%d",
                     keypad->key[index].name,
                     keypad->key[index].code);
            if (!strcmp(tmp, name)) {
                found = true;
                break;                             // 找到，跳出
            }
        }

        if (!found) {
            dev_err(dev, "invalid key '%s'\n", name);
            return -EINVAL;
        }
        keypad->key[index].ignore = ignore;        // 设置屏蔽标志
    }

    return count;                                  // 返回 consumed 字节数
```

**它干了什么**：解析用户写入的字符串，支持三种格式：
- `all 1` 或 `all Y` → 屏蔽所有按键
- `none 1` 或 `none Y` → 取消所有屏蔽
- `bluetooth,600 Y` → 屏蔽名为 bluetooth、键码为 600 的按键

---

## 11. class 属性声明 — 把函数绑定到 sysfs 文件

```c
static CLASS_ATTR_RO(table);       // 声明 table 是只读属性 → table_show
static CLASS_ATTR_RW(ignore);      // 声明 ignore 是读写属性 → ignore_show + ignore_store
```

这组宏展开后创建了 `class_attr_table` 和 `class_attr_ignore` 两个结构体变量，内核 sysfs 框架会自动在 `/sys/class/gpio_keypad/` 下创建 `table` 和 `ignore` 文件。

```c
static struct attribute *meson_gpiokey_attrs[] = {
    &class_attr_table.attr,
    &class_attr_ignore.attr,
    NULL                              // 必须以 NULL 结尾
};

ATTRIBUTE_GROUPS(meson_gpiokey);      // 生成 meson_gpiokey_groups 变量
```

`ATTRIBUTE_GROUPS(meson_gpiokey)` 展开为：
```c
const struct attribute_group meson_gpiokey_groups[] = {
    { .attrs = meson_gpiokey_attrs },
    {}
};
```

然后在 class 注册时赋值给 `kp_class.class_groups`，内核自动创建对应的 sysfs 文件。

---

## 12. `count_show()` — `/sys/class/resetkey/count` 被读时干什么

```c
static ssize_t count_show(struct class *cls, struct class_attribute *attr,
                         char *buf)
{
    struct gpio_keypad *keypad = container_of(cls,
                    struct gpio_keypad, resetkey_class);
    return sprintf(buf, "%u\n", keypad->reset_count);
}

static CLASS_ATTR_RO(count);

static struct attribute *resetkey_attrs[] = {
    &class_attr_count.attr,
    NULL
};

ATTRIBUTE_GROUPS(resetkey);
```

**它干了什么**：输出 `reset_count` 的值，即蓝牙键自驱动加载以来被按下的累计次数。用于实现"长按蓝牙键 N 次恢复出厂"。

和 `table_show` 一样，这是一个 class 属性，但属于另一个 class（`resetkey_class`），所以在 `/sys/class/resetkey/count` 下暴露。

---

## 13. `meson_gpio_kp_probe()` — 驱动入口，设备匹配时调用

这是驱动的**核心函数**（173 行）。当内核发现 DTS 中有 `compatible = "amlogic, gpio_keypad"` 的节点时，这个函数被调用。

### 第 1 步：分配 per-device 数据

```c
    keypad = devm_kzalloc(&pdev->dev,
                          sizeof(struct gpio_keypad), GFP_KERNEL);
```

`devm_kzalloc` 干了三件事：
1. 分配 `sizeof(struct gpio_keypad)` 字节内存
2. 内存清零（kz = kernel zero）
3. 把内存"挂"到 `pdev->dev` 上（devm = device managed），设备移除时自动释放

### 第 2 步：从 DTS 读取参数

```c
    ret = of_property_read_u32(pdev->dev.of_node,        // DTS 节点
                               "detect_mode",             // 属性名
                               &keypad->use_irq);         // 读到这个变量
    if (ret)   // 返回负值 = 属性不存在
        keypad->use_irq = DEFAULT_POLL_MODE;              // 用默认值 0 (polling)
```

`of_property_read_u32` 从 DTS 节点中读取一个 u32 属性。如果属性不存在（比如 DTS 里没写 `detect_mode`），返回负值，驱动用默认值。

同样方式读取 `scan_period`（默认 20ms）：

```c
    ret = of_property_read_u32(pdev->dev.of_node,
                               "scan_period", &keypad->scan_period);
    if (ret)
        keypad->scan_period = DEFAULT_SCAN_PERION;
```

读取布尔属性（存在即 true）：

```c
    if (of_property_read_bool(pdev->dev.of_node, "wakeup-source"))
        wakeup_source = 1;                            // DTS 中有 wakeup-source 属性
```

读取必需属性（不存在就报错返回）：

```c
    ret = of_property_read_u32(pdev->dev.of_node,
                               "key_num", &keypad->key_size);
    if (ret) {
        dev_err(&pdev->dev, "failed to get key_num!\n");
        return -EINVAL;                                // 返回错误，probe 失败
    }
```

### 第 3 步：分配按键数组

```c
    keypad->key = devm_kzalloc(&pdev->dev,
                               (keypad->key_size) * sizeof(*keypad->key),
                               GFP_KERNEL);
```

根据 `key_num` 的值，分配能容纳所有按键的 `pin_desc` 数组。

### 第 4 步：逐按键获取 GPIO 和解析属性

```c
    for (i = 0; i < keypad->key_size; i++) {

        desc = devm_gpiod_get_index(&pdev->dev, "key", i, GPIOD_IN);
```

`devm_gpiod_get_index(dev, "key", i, GPIOD_IN)` 干了什么：
1. 在 DTS 中查找 `key-gpios` 属性（内核自动在 `"key"` 后追加 `-gpios` 后缀）
2. 取该属性的第 `i` 个条目
3. 返回 GPIO 描述符，并设置为输入模式（`GPIOD_IN`）

对应 DTS：
```dts
key-gpios = <&gpio GPIOD_2 GPIO_ACTIVE_HIGH   ← i=0
             &gpio GPIOD_3 GPIO_ACTIVE_HIGH>; ← i=1
```

然后逐个解析数组属性：

```c
        keypad->key[i].desc = desc;                    // 保存 GPIO 描述符
        keypad->key[i].current_status = 1;             // 初始状态 = 高电平(释放)

        // 读取 key_code 数组的第 i 个值
        ret = of_property_read_u32_index(pdev->dev.of_node,
                                         "key_code", i,
                                         &keypad->key[i].code);

        // 读取 key_type 数组的第 i 个值（可选，默认 EV_KEY）
        ret = of_property_read_u32_index(pdev->dev.of_node,
                                         "key_type", i,
                                         &keypad->key[i].key_type);
        if (ret)
            keypad->key[i].key_type = EV_KEY;

        // 读取 key_name 数组的第 i 个值（字符串数组）
        ret = of_property_read_string_index(pdev->dev.of_node,
                                            "key_name", i,
                                            &keypad->key[i].name);
```

DTS 中的数组属性：
```dts
key_code = <600 SW_MUTE_DEVICE>;   ← index=0 是 600, index=1 是 SW_MUTE_DEVICE
key_type = <EV_KEY EV_SW>;          ← index=0 是 EV_KEY, index=1 是 EV_SW
key_name = "bluetooth", "mute";    ← index=0 是 "bluetooth", index=1 是 "mute"
```

最后配置 GPIO 硬件：

```c
        gpiod_direction_input(keypad->key[i].desc);        // 设为输入模式
        gpiod_set_pull(keypad->key[i].desc, GPIOD_PULL_UP);// 使能内部上拉电阻
    }
```

`gpiod_set_pull(GPIOD_PULL_UP)` 使能芯片内部的上拉电阻，确保按键未按下时 GPIO 读到的是确定的高电平（而不是悬空）。

### 第 5 步：注册 sysfs class 接口

```c
    keypad->kp_class.name = "gpio_keypad";               // class 名称 → /sys/class/gpio_keypad/
    keypad->kp_class.owner = THIS_MODULE;
    keypad->kp_class.class_groups = meson_gpiokey_groups; // 绑定的属性组 (table + ignore)
    ret = class_register(&keypad->kp_class);              // 注册 → sysfs 文件自动创建

    keypad->resetkey_class.name = "resetkey";             // class 名称 → /sys/class/resetkey/
    keypad->resetkey_class.owner = THIS_MODULE;
    keypad->resetkey_class.class_groups = resetkey_groups;// 绑定的属性组 (count)
    keypad->reset_count = 0;
    ret = class_register(&keypad->resetkey_class);
    if (ret) {
        class_unregister(&keypad->kp_class);              // 失败时要注销已注册的 class
        return -EINVAL;
    }
```

`class_register` 会触发内核在 `/sys/class/` 下创建对应目录，并根据 `class_groups` 自动创建属性文件。

### 第 6 步：注册 input 设备

```c
    input_dev = input_allocate_device();                  // 分配 input_dev 结构体
```

`input_allocate_device` 分配一个新的 `input_dev`，后续往里填属性。

```c
    for (i = 0; i < keypad->key_size; i++) {
        input_set_capability(input_dev, keypad->key[i].key_type,
                             keypad->key[i].code);
    }
```

`input_set_capability` 告诉 input 子系统"这个设备能产生这种事件"。上层程序（如 Android InputReader）通过查询 capability 来判断设备用途。比如声明了 `EV_KEY / KEY_VOLUMEUP`，系统就知道用这个设备来控制音量。

```c
    input_dev->name = "gpio_keypad";                     // 设备名，getevent -l 会显示
    input_dev->phys = "gpio_keypad/input0";              // 物理路径标识
    input_dev->dev.parent = &pdev->dev;                  // 父设备（建立 sysfs 层级关系）
    input_dev->id.bustype = BUS_ISA;
    input_dev->id.vendor  = 0x0001;
    input_dev->id.product = 0x0001;
    input_dev->id.version = 0x0100;
    input_dev->rep[REP_DELAY]  = 0xffffffff;             // 禁用自动按键重复
    input_dev->rep[REP_PERIOD] = 0xffffffff;             //   （驱动自己处理）
```

`rep[REP_DELAY] = 0xffffffff`：设为最大值，等于禁用 input 子系统的自动长按重复功能。驱动自己通过消抖逻辑处理按键状态，不需要 input 核来产生额外的重复事件。

```c
    keypad->input_dev = input_dev;                       // 保存指针
    ret = input_register_device(keypad->input_dev);      // 注册 → 创建 /dev/input/eventX
    if (ret < 0) {
        input_free_device(keypad->input_dev);            // 注册失败要释放
        return -EINVAL;
    }
```

`input_register_device` 在内核中注册设备，用户态会出现 `/dev/input/eventX`。

### 第 7 步：绑定数据 + 启动检测

```c
    platform_set_drvdata(pdev, keypad);                  // 把 keypad 存到 pdev 里
```

`platform_set_drvdata` 把 per-device 数据存到 `platform_device` 的私有字段里。后续 `remove`、`suspend`、`resume` 中通过 `platform_get_drvdata(pdev)` / `dev_get_drvdata(dev)` 取出。

```c
    timer_setup(&keypad->polling_timer,                   // 初始化定时器
                polling_timer_handler,                    //   回调函数
                0);                                       //   flags
```

`timer_setup` 初始化定时器，但不启动。定时器到期时调用 `polling_timer_handler`。

### 第 8 步：根据模式启动检测

```c
    if (keypad->use_irq) {
        // ===== 中断模式 =====
        for (i = 0; i < keypad->key_size; i++) {
            keypad->key[i].count = 0;
            keypad->key[i].irq_num = gpiod_to_irq(keypad->key[i].desc);
            //                           ↑ GPIO 描述符 → IRQ 中断号

            ret = devm_request_irq(&pdev->dev,
                                   keypad->key[i].irq_num,      // 中断号
                                   gpio_irq_handler,             // 处理函数
                                   IRQF_TRIGGER_FALLING,         // 下降沿触发
                                   "gpio_keypad",                // 名称（/proc/interrupts 中可见）
                                   keypad);                      // 传给 handler 的 data
            // ...
```

`gpiod_to_irq` 查询 GPIO 对应的中断号。`devm_request_irq` 注册中断处理函数——当该 GPIO 出现下降沿（高→低）时，内核调用 `gpio_irq_handler`。

`IRQF_TRIGGER_FALLING`：因为空闲时上拉到高电平，按下时拉低，下降沿就是按键按下的瞬间。

**对 KEY_POWER 的特殊处理**：

```c
            if (keypad->key[i].code == KEY_POWER) {
                d = irq_to_desc(keypad->key[i].irq_num);     // 拿到 IRQ 描述符
                if (d) {
                    number = d->irq_data.parent_data->hwirq - 32; // 算出硬件 IRQ 编号
                    pwr_ctrl_irq_set(number, 1, 0);          // Amlogic 私有：设为唤醒源
                    register_flag = 1;
                }
            }
```

`pwr_ctrl_irq_set` 是 Amlogic 的 API，把这个 GPIO 中断注册为**唤醒源**——即使系统休眠了，按下电源键仍然能唤醒。

```c
    } else {
        // ===== polling 模式 =====
        mod_timer(&keypad->polling_timer,
                  jiffies + msecs_to_jiffies(keypad->scan_period));
        //         ↑ 当前时间 + scan_period ms 后触发定时器
    }
```

`mod_timer` 修改定时器的到期时间。第一次调用等于"在 `scan_period` ms 后触发"。定时器触发后调用 `polling_timer_handler`，在那个函数里会再次调用 `mod_timer` 实现循环。

**`jiffies`** 是内核维护的全局变量，记录自系统启动以来的时钟节拍数。`msecs_to_jiffies(20)` 把 20ms 转换成对应的节拍数。

### 第 9 步：唤醒源状态日志

```c
    if (wakeup_source) {
        if (register_flag)
            dev_dbg(&pdev->dev, "succeed to register wakeup source!\n");
        else
            dev_err(&pdev->dev, "failed to register wakeup source!\n");
    }

    return 0;   // probe 成功
}
```

---

## 14. `meson_gpio_kp_remove()` — 设备被移除时调用

```c
static int meson_gpio_kp_remove(struct platform_device *pdev)
{
    struct gpio_keypad *keypad;

    keypad = platform_get_drvdata(pdev);         // 拿出 probe 时存的数据

    class_unregister(&keypad->resetkey_class);   // ① 注销 /sys/class/resetkey/
    class_unregister(&keypad->kp_class);         // ② 注销 /sys/class/gpio_keypad/
    input_unregister_device(keypad->input_dev);  // ③ 注销 /dev/input/eventX
    input_free_device(keypad->input_dev);        // ④ 释放 input_dev 内存
    del_timer(&keypad->polling_timer);           // ⑤ 删除定时器（防止之后触发）
    return 0;
}
```

**它干了什么**：按与 probe 相反的注册顺序，依次注销 class、注销 input 设备、释放内存、删除定时器。

**`del_timer` 为什么放最后**：`del_timer` 是同步的——如果定时器正在执行，会等它执行完。这时候 `input_dev` 还有效，定时器 handler 如果正好在跑，访问 `input_dev` 也不会崩。

`devm_kzalloc` 分配的内存**不需要在这里 kfree**——设备移除时内核自动释放。

---

## 15. `of_device_id` — DTS 匹配表

```c
static const struct of_device_id key_dt_match[] = {
    { .compatible = "amlogic, gpio_keypad", },  // 匹配 DTS 中 compatible = "amlogic, gpio_keypad"
    {}                                          // 必须以空条目结尾
};
```

**它干了什么**：告诉内核"我的驱动负责处理 compatible 为 `"amlogic, gpio_keypad"` 的设备"。当内核解析 DTS 遇到匹配节点时，调用 probe。

---

## 16. 电源管理回调 — 休眠/唤醒时干什么

### `suspend` — 系统要休眠了

```c
static int meson_gpio_kp_suspend(struct device *dev)
{
    struct gpio_keypad *pdata;
    pdata = (struct gpio_keypad *)dev_get_drvdata(dev);     // 拿到 per-device 数据

    if (!pdata->use_irq)
        del_timer(&pdata->polling_timer);                    // polling 模式：停定时器省电
    // irq 模式：不动，GPIO 中断本身就是唤醒源

    return 0;
}
```

### `resume` — 系统从休眠恢复了

```c
static int meson_gpio_kp_resume(struct device *dev)
{
    int i;
    struct gpio_keypad *pdata;
    pdata = (struct gpio_keypad *)dev_get_drvdata(dev);

    if (!pdata->use_irq)
        mod_timer(&pdata->polling_timer,                     // polling：重新启动定时器
                  jiffies + msecs_to_jiffies(5));            // 5ms 后开始（比正常更快）

    if (get_resume_method() == POWER_KEY_WAKEUP) {           // 如果是电源键唤醒的
        for (i = 0; i < pdata->key_size; i++) {
            if (pdata->key[i].code == KEY_POWER) {
                // 模拟一次电源键按下+释放，通知上层"用户按了电源键"
                input_report_key(pdata->input_dev, KEY_POWER, 1);
                input_sync(pdata->input_dev);
                input_report_key(pdata->input_dev, KEY_POWER, 0);
                input_sync(pdata->input_dev);
                break;
            }
        }
    }
    return 0;
}
```

**它干了什么**：
1. polling 模式：重新启动定时器（5ms 后第一次触发，比正常 20ms 更快，确保快速感知按键）
2. 如果系统是被电源键唤醒的，向 input 子系统上报一次电源键事件。这样 Android 框架层能识别"用户按了电源键唤醒"，正确处理亮屏逻辑。

### `freeze` — S4 休眠（Suspend-to-Disk）冻结

```c
static int meson_gpio_kp_freeze(struct device *dev)
{
    struct gpio_keypad *keypad = dev_get_drvdata(dev);
    int index;
    int ret;

    ret = meson_gpio_kp_suspend(dev);        // 先执行 suspend：停定时器
    if (ret)
        return ret;

    for (index = 0; index < keypad->key_size; index++)
        devm_gpiod_put(dev, keypad->key[index].desc);  // 释放所有 GPIO 描述符

    return 0;
}
```

**它干了什么**：停定时器 + **释放 GPIO 描述符**。S4 休眠时内核镜像会完全重建，GPIO 描述符（内核地址）不再有效，必须释放。

### `restore` — S4 休眠恢复

```c
static int meson_gpio_kp_restore(struct device *dev)
{
    struct gpio_keypad *keypad = dev_get_drvdata(dev);
    struct gpio_desc *desc;
    int index;

    for (index = 0; index < keypad->key_size; index++) {
        desc = devm_gpiod_get_index(dev, "key", index, GPIOD_IN); // 重新获取 GPIO
        if (IS_ERR_OR_NULL(desc))
            return -EINVAL;
        gpiod_direction_input(desc);                             // 重新设为输入
        gpiod_set_pull(desc, GPIOD_PULL_UP);                     // 重新使能上拉
        keypad->key[index].desc = desc;                          // 更新描述符指针
    }

    return meson_gpio_kp_resume(dev);                            // 复用 resume 逻辑
}
```

**它干了什么**：S4 恢复后，所有 GPIO 描述符都失效了，需要**重新获取**。获取后重新配置为输入+上拉。最后调用 `resume` 启动检测。

### PM ops 汇总

```c
static const struct dev_pm_ops meson_gpio_kp_pm_ops = {
    .suspend  = meson_gpio_kp_suspend,   // S3 休眠 → 停定时器
    .resume   = meson_gpio_kp_resume,    // S3 恢复 → 启定时器 + 模拟电源键
    .freeze   = meson_gpio_kp_freeze,    // S4 冻结 → 停定时器 + 释放 GPIO
    .thaw     = meson_gpio_kp_resume,    // S4 解冻 → 复用 resume
    .poweroff = meson_gpio_kp_suspend,   // 关机 → 复用 suspend
    .restore  = meson_gpio_kp_restore,   // S4 恢复 → 重新获取 GPIO + resume
};
```

`thaw` 是 freeze 的反操作，这里直接复用了 `resume` 的逻辑（重新启动定时器 + 模拟电源键）。`poweroff` 是关机的回调，直接复用 `suspend`（停定时器）。

---

## 17. `platform_driver` 结构体 — 把驱动各部件组装起来

```c
static struct platform_driver meson_gpio_kp_driver = {
    .probe  = meson_gpio_kp_probe,       // 设备匹配时调用
    .remove = meson_gpio_kp_remove,      // 设备移除时调用
    .driver = {
        .name           = "gpio-keypad",           // 驱动名称
        .pm             = &meson_gpio_kp_pm_ops,   // 电源管理回调
        .of_match_table = key_dt_match,            // DTS compatible 匹配表
    },
};
```

**它干了什么**：把前面写的所有函数（probe、remove、PM 回调、匹配表）组装成一个 `platform_driver`。这是一个描述"这个驱动能干吗"的汇总结构体。

---

## 18. `module_init` 和 `module_exit` — 驱动的入口和出口

```c
int __init meson_gpio_kp_init(void)
{
    return platform_driver_register(&meson_gpio_kp_driver);
    //      ↑ 告诉内核："我准备好了，有匹配的设备就叫我"
}

void __exit meson_gpio_kp_exit(void)
{
    platform_driver_unregister(&meson_gpio_kp_driver);
    //      ↑ 告诉内核："我要退出了，之前绑定的设备请解绑"
}
```

**`platform_driver_register` 干了什么**：把 `meson_gpio_kp_driver` 注册到内核的 platform 总线。内核遍历已解析的 DTS 节点，如果 `compatible` 匹配就调用 probe。

`__init` 宏：告诉编译器"这个函数只在初始化阶段使用"，初始化完成后内核可回收该代码占用的内存。

---

## 函数调用关系总图

```
platform_driver_register()
  └─ .probe = meson_gpio_kp_probe()
       ├─ devm_kzalloc()                            ← 分配 per-device 数据
       ├─ of_property_read_u32() × N                ← 从 DTS 读参数
       ├─ devm_kzalloc()                            ← 分配按键数组
       ├─ for each key:
       │    ├─ devm_gpiod_get_index()               ← 获取 GPIO
       │    ├─ of_property_read_u32_index() × 2     ← 读 key_code, key_type
       │    ├─ of_property_read_string_index()      ← 读 key_name
       │    ├─ gpiod_direction_input()              ← 设为输入
       │    └─ gpiod_set_pull(PULL_UP)              ← 使能上拉
       ├─ class_register() × 2                      ← 创建 /sys/class/... 接口
       ├─ input_allocate_device()                   ← 分配 input 设备
       ├─ input_set_capability() × N                ← 声明设备能力
       ├─ input_register_device()                   ← 注册 input → /dev/input/eventX
       ├─ platform_set_drvdata()                    ← 绑定数据
       ├─ timer_setup()                             ← 初始化定时器
       └─ if irq:
            ├─ gpiod_to_irq() × N                   ← GPIO → IRQ 号
            └─ devm_request_irq() × N               ← 注册中断
          else:
            └─ mod_timer()                          ← 启动轮询定时器

运行时:
  中断触发 → gpio_irq_handler()                    ← 中断上下文
    └─ mod_timer(20ms 后)
         └─ polling_timer_handler()                  ← 软中断上下文
              ├─ gpiod_get_value() × N               ← 读 GPIO 电平
              └─ report_key_code()                   ← 消抖 + 上报
                   ├─ input_event()                  ← 向 input 上报
                   └─ input_sync()                   ← 同步帧

sysfs 访问:
  cat /sys/class/gpio_keypad/table → table_show()
  cat /sys/class/gpio_keypad/ignore → ignore_show()
  echo "xxx" > /sys/class/gpio_keypad/ignore → ignore_store()
  cat /sys/class/resetkey/count → count_show()

休眠/恢复:
  suspend → 停轮询定时器
  resume  → 启轮询定时器 + (如果是电源键唤醒)上报电源键事件
  freeze  → 停轮询定时器 + 释放所有 GPIO 描述符
  restore → 重新获取所有 GPIO + resume

驱动卸载:
  platform_driver_unregister()
    └─ .remove = meson_gpio_kp_remove()
         ├─ class_unregister() × 2
         ├─ input_unregister_device()
         ├─ input_free_device()
         └─ del_timer()
```

---

## 驱动编写常用模式速查

读完这份驱动，你遇到了以下模式——在新驱动中会反复出现：

| 模式 | 函数/宏 | 在哪里看到 |
|------|----------|-----------|
| 自动内存管理 | `devm_kzalloc()` | probe 第 1 步 |
| DTS 属性解析 | `of_property_read_u32()` / `_index()` | probe 第 2、4 步 |
| GPIO consumer API | `devm_gpiod_get_index()` | probe 第 4 步 |
| 容器反查 | `container_of()` / `from_timer()` | table_show, polling_timer_handler |
| sysfs 属性暴露 | `CLASS_ATTR_RO` / `CLASS_ATTR_RW` | 代码 §11 |
| input 设备注册 | `input_allocate_device()` + `input_register_device()` | probe 第 6 步 |
| 定时器 | `timer_setup()` + `mod_timer()` | probe 第 7 步 |
| 中断注册 | `devm_request_irq()` | probe 第 8 步 |
| per-device 数据绑定 | `platform_set_drvdata()` / `_get_drvdata()` | probe 第 7 步 |
| 电源管理 | `dev_pm_ops` 结构体 | 代码 §16 |
| platform 驱动模板 | `platform_driver` 结构体 | 代码 §17 |
