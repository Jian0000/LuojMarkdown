+++
title = 'Level 8：App 点一下控制板子上的 LED'
date = '2026-06-09T14:02:00+08:00'
draft = false
+++

# Level 8：App 点一下控制板子上的 LED

## 我想做

在手机上装一个自己写的 App，点一下按钮，板子上的物理 LED 亮或灭。打通从 App 到 GPIO 寄存器的完整链路。

## 先回答三个问题

1. **App 是怎么跟硬件通信的？**
   ```
   App (Java)
     ↓ Binder IPC
   Native Service (C++)
     ↓ hw_get_module()
   HAL (xxx_module_t)
     ↓ open("/dev/xxx")
   Kernel Driver
     ↓ writel()
   GPIO 寄存器
   ```
   每一层都是独立的模块，层与层之间有标准接口。替换其中任何一层不影响其他层。

2. **HAL 解决了什么问题？**
   - 没有 HAL 之前：App 直接 open/ioctl 设备节点
   - 问题：驱动接口变了，所有 App 都得改
   - HAL 的解决：给上层提供稳定的抽象接口，驱动变化只改 HAL 实现，不涉及 Framework 和 App
   - 在 Treble 架构中，HAL 更是 vendor 和 system 的契约边界

3. **Binder 是什么？**
   - Android 特有的 IPC（进程间通信）机制
   - App 和 Native Service 是不同进程，不能直接函数调用
   - Binder 让它们像调用本地函数一样跨进程通信
   - Binder 还负责权限校验——ServiceManager 检查调用方有没有权限访问某个服务

## 需要懂的知识

**ioctl 驱动的写法（在 Level 4 驱动基础上扩展）**

```c
#define GPIO_LED_MAGIC 'L'
#define GPIO_LED_ON   _IO(GPIO_LED_MAGIC, 1)
#define GPIO_LED_OFF  _IO(GPIO_LED_MAGIC, 2)

static long gpio_led_ioctl(struct file *file, unsigned int cmd, unsigned long arg) {
    switch (cmd) {
        case GPIO_LED_ON:
            writel(1, gpio_base + GPIO_OUTPUT_REG);
            break;
        case GPIO_LED_OFF:
            writel(0, gpio_base + GPIO_OUTPUT_REG);
            break;
    }
    return 0;
}
```

在 `.file_operations` 中加入 `.unlocked_ioctl = gpio_led_ioctl`

**HAL 模块模板**

HAL 本质上是一个约定格式的动态库（.so），它必须暴露一个名为 `HAL_MODULE_INFO_SYM` 的符号：

```c
const struct hw_module_t HAL_MODULE_INFO_SYM = {
    .tag = HARDWARE_MODULE_TAG,
    .id = "led",
    .methods = &led_module_methods,
};
```

上层通过 `hw_get_module("led", &module)` 加载这个 so，然后通过 `module->methods->open()` 拿到设备操作接口。

**AIDL 与 Binder Service**

Service 端实现一个接口（如 `ILedService`），注册到 ServiceManager：
```cpp
defaultServiceManager()->addService("led", new LedService());

// App 端获取
sp<IBinder> binder = defaultServiceManager()->getService("led");
sp<ILedService> service = ILedService::asInterface(binder);
service->setLedOn();
```

## 动手方案

（各层代码从 Level 3、4、5 复用和扩展）

```bash
# 第 1 步：确认驱动已有 ioctl（扩展 Level 4 的驱动）
# 在 simple_char.c 中加入 GPIO_LED_ON/OFF 的 ioctl 命令

# 第 2 步：写 HAL 模块
# 位置：hardware/amlogic/led/ 或 vendor/amlogic/hardware/led/

# 第 3 步：写 Native Service
# 位置：device/amlogic/ross/ledservice/

# 第 4 步：写 App
# 用 Android Studio 或 aapt 编译

# 第 5 步：编译烧录（参考 level-2）
```

## 验收清单

| 项目 | 验证方式 |
|------|----------|
| 全链路通 | App 点按钮，板子上 LED 亮/灭 |
| 理解每层作用 | 你能画出去掉每一层后系统会怎样（例如没有 HAL 行不行） |
| SELinux 没有拦 | `dmesg \| grep avc` 没有你的服务的错误 |
