+++
title = '实操问题记录'
date = '2026-06-09T16:01:00+08:00'
draft = false
+++

# 实操问题记录

按 `target.md` 的 Level 对应，只记录实操中遇到的**编译/运行错误**和**异常现象**。知识点类内容归属各 Level 文档本身。

---

## Level 3：加一个 native 服务开机自启动

对应学习文档：[level-3-native服务.md](level-3-native服务.md)

### [L3-01] checkpolicy unrecognized character

**现象：** `ERROR 'unrecognized character' at token '`，指向新建的 `.te` 文件

**根因：** `.te` 文件是 Windows CRLF（`\r\n`）换行符，Linux 的 checkpolicy 不识别 `\r`

**修复：** `sed -i 's/\r$//' vendor/giec/common/sepolicy/luojService.te`

**验证：** `cat -A file.te` → 看到 `$`（LF）而非 `^M$`（CRLF）

### [L3-02] neverallow 违规：dac_override

**现象：** `neverallow check failed ... allow luojService self:capability { dac_override }`

**根因：** `dac_override`（绕过文件权限）被系统核心策略（`domain.te:470`）全局禁止，任何 domain 不可申请

**修复：** 从 `.te` 中删除 `allow luojService self:capability { dac_override };`

**启示：** 非必要不声明 capability，纯打日志的服务不需要任何 capability

### [L3-03] init domain transition 失败

**现象：** `start luojService` 返回 `Unable to start service`；dmesg 报 `File /vendor/bin/luojService(labeled "u:object_r:vendor_file:s0") has incorrect label or no domain transition`

**根因：** 只配了 `.te` 文件（定义 domain/exec_type），缺了 `file_contexts` 条目（路径→标签映射）。二进制被标记为默认 `vendor_file`，init 执行它时无法完成 domain transition

**修复：** 在 `vendor/giec/common/sepolicy/file_contexts` 中添加：`/vendor/bin/luojService u:object_r:luojService_exec:s0`

**验证：** `adb shell ls -Z /vendor/bin/luojService` → 输出应为 `u:object_r:luojService_exec:s0`

---

## Level 4：字符设备驱动

对应学习文档：[level-4-字符驱动.md](level-4-字符驱动.md)

### [L4-01] vmlinux 依赖导致 .ko 无法链接

**现象：** `make[2]: *** No rule to make target 'vmlinux', needed by 'drivers/misc/simple_char.ko'. Stop.`，同时有大量 `modpost: ... undefined!` 警告

**根因：** `.ko` 文件需要内核的导出符号表（`vmlinux.symvers`）来解析 `register_chrdev`、`copy_from_user` 等符号。必须先有 `vmlinux` 才能链接模块

**修复：** 走 Bazel 构建系统（`./build.sh -k`），不要直接用 `make` 编模块

### [L4-02] Bazel 编译报错：源码树不干净

**现象：** `./build.sh -kj200` 报 `The source tree is not clean, please run 'make mrproper'`，Bazel 构建失败

**根因：** 在 `common/common14-5.15/common/` 下直接执行了 `make` 或 `make menuconfig`，生成了 `.config`、`include/config/`、`drivers/misc/*.o` 等中间文件。Bazel 构建系统要求源码树绝对干净，检测到这些残留文件后拒绝编译

**修复：** `cd common/common14-5.15/common && make mrproper` 清除所有编译中间文件，再重新 `./build.sh -kj200`

**教训：** 用 Bazel 构建时**不要**在 `common/` 下执行 `make` 或 `make menuconfig`。内核配置通过修改 `arch/arm64/configs/amlogic_gki.fragment` 实现

**验证：** 执行 `./build.sh -k` 前确认源码树无 `.config`、`.o` 等未跟踪文件

### [L4-03] 改错了 fragment 路径，Bazel 未采用配置

**现象：** 在 `amlogic_gki.fragment` 中加了 `CONFIG_SIMPLE_CHAR=m`，但编译后 `.config` 仍是 `# CONFIG_SIMPLE_CHAR is not set`，`simple_char.ko` 未生成

**根因：** 该项目存在**两个**同名的 `amlogic_gki.fragment`，传统 `make` 和 Bazel 各用一个：

| 路径 | 被谁用 |
|------|--------|
| `common/arch/arm64/configs/amlogic_gki.fragment` | 传统 `build.sh` / `make` |
| `common/common_drivers/arch/arm64/configs/amlogic_gki.fragment` | **Bazel**（`build.config.amlogic.bazel`） |

我们改了第一个，但 Bazel 引用的是第二个。Bazel 通过 `common_drivers/build.config.amlogic.bazel` 指定了厂商驱动目录的配置路径。

**修复：** 将 `CONFIG_SIMPLE_CHAR=m` 添加到正确的文件 `common/common_drivers/arch/arm64/configs/amlogic_gki.fragment`

**验证：** 通过以下 4 个文件确认 Bazel 使用的 fragment 路径：

```
BUILD.bazel:811  →  common_drivers/amlogic.bzl
                              ↓ build_config
                   build.config.amlogic.bazel:7 → project/build.config.gki10
                                                           ↓ COMMON_DRIVERS_DIR
                                                  common_drivers
                   build.config.amlogic.bazel:31 → ${KERNEL_DIR}/${COMMON_DRIVERS_DIR}/arch/arm64/configs/amlogic_gki.fragment
                   → 实际路径：common/common_drivers/arch/arm64/configs/amlogic_gki.fragment
```

**教训：** 修改内核配置前，先确认 `build.config.amlogic.bazel` 中引用的 fragment 路径，而非按直觉找 `common/arch/arm64/` 下的同名文件

### [L4-04] 模块编译成功但未被拷贝到 dist 目录

**现象：** `./build.sh -kj200` 编译完成，`simple_char.o` 编译成功，但最终报 `ERROR: modules are built but not copied`，`out/android14-5.15/dist/` 下没有 `simple_char.ko`

**根因：** Bazel 编译模块后，需要通过 `modules.bzl` 中的 `AMLOGIC_COMMON_MODULES` 列表声明哪些 `.ko` 文件需要拷贝到 dist 目录。未声明的模块即使编译成功也不会被拷贝

**修复：** 在 `common/common_drivers/modules.bzl` 的 `AMLOGIC_COMMON_MODULES` 列表中添加 `"drivers/misc/simple_char.ko"`（保持字母序）

**验证：** 重新 `./build.sh -kj200` 后检查 `out/android14-5.15/dist/simple_char.ko` 存在

### [L4-05] dev_read() 永不返回 EOF 导致 cat 死循环

**现象：** `cat /dev/simple_char` 触发 ~40 次 `read() 4096 bytes`，无限循环读取同一段数据

**根因：** `dev_read()` 没有使用 `offset` 参数追踪读取位置，也没有在数据读完后返回 `0`（EOF）。VFS 规范要求：当 `*offset >= data_len` 时必须返回 `0`，否则 `cat` 等用户态工具认为还有数据可读，不断重试

**修复：** 在 `dev_read()` 中加入 offset 逻辑：
- 用 `strlen(kernel_buffer)` 算出实际数据长度
- `*offset >= data_len` 时返回 `0`
- 用 `kernel_buffer + *offset` 作为 `copy_to_user` 的源地址
- 拷贝成功后 `*offset += len`

**验证：** `cat /dev/simple_char` 应打印一次数据后正常退出（而非循环打印）

---

## Level 5：操作 GPIO

对应学习文档：[level-5-GPIO.md](level-5-GPIO.md)

### [L5-01] /sys/kernel/debug/gpio 不存在

**现象：** 板子上 `/sys/kernel/debug/` 目录下没有 `gpio` 文件/目录

**根因：** 内核配置中 `# CONFIG_GPIO_SYSFS is not set`，旧的 sysfs GPIO 接口已被禁用（kernel 5.15 默认行为）。虽然 `CONFIG_DEBUG_FS=y` 且 gpiolib 理论上会创建 `/sys/kernel/debug/gpio`，但实际可能需先挂载 debugfs 或加载 gpio 驱动模块

**排查步骤：**
1. `adb shell mount | grep debugfs` — 确认 debugfs 是否挂载；若未挂载：`adb shell mount -t debugfs none /sys/kernel/debug`
2. `adb shell lsmod | grep gpio` — 确认 amlogic-gpio.ko 是否加载；若未加载：`adb shell modprobe amlogic-gpio`
3. 若 debugfs gpio 文件确实不存在，改用 gpio 字符设备接口

**替代方案（推荐）：** 内核 5.15 已启用 `CONFIG_GPIO_CDEV=y`，使用 libgpiod 工具：
```bash
adb shell gpiodetect        # 列出所有 gpiochip
adb shell gpioinfo           # 查看所有 GPIO 的详细信息
adb shell gpioset gpiochip0 <pin>=1    # 设置输出
adb shell gpioget gpiochip0 <pin>      # 读取输入
```

**参考代码路径：**
- GPIO 控制器节点：`common/common_drivers/arch/arm64/boot/dts/amlogic/meson-s7d.dtsi`（S7D）
- GPIO pin 定义：`common/common_drivers/include/dt-bindings/gpio/meson-s7d-gpio.h`（83 pins，GPIOD_0=24 ~ GPIOZ_12=80）
- gpiolib debugfs 创建：`common/drivers/gpio/gpiolib.c:4509-4515`（`debugfs_create_file("gpio", ...)` 在 `#ifdef CONFIG_DEBUG_FS` 下）
- 构建配置链：`common/common_drivers/build.config.amlogic.bazel:33`（合并 `gki_defconfig + amlogic_gki.fragment + amlogic_gki.10 + amlogic_gki.debug`）

**板子 LED 硬件信息：** S905X5M (S7D) 参考板的 RGB LED 通过 **AW9523B I2C GPIO 扩展芯片**控制，不是直接接在 SoC GPIO 上。原理图中 **GPIOD_4 标为 FUN_LED**，对应全局编号 28（`meson-s7d-gpio.h`），可直接用于 GPIO 驱动练习

### [L5-02] insmod fun_led.ko 报 -517（-EPROBE_DEFER），GPIOD_4 已被 leds-gpio 占用

**现象：** `insmod fun_led.ko` 成功加载模块，但 `dmesg` 报 `gpio_request failed, ret=-517`（`-EPROBE_DEFER`）。LED 在重启时闪烁红灯，说明硬件连接正常。

**根因：** **两层拒绝**。

**第一层（DTS 资源冲突，根本原因）：**
GPIOD_4 已在板级 DTS `s7d_s905x5m_bm201.dts:224-228` 中声明为 `gpio-leds` 设备的子节点 `sys_led_red`。内核 `leds-gpio` 驱动（`drivers/leds/leds-gpio.c`）通过 `devm_fwnode_get_gpiod_from_child()` → `gpiod_get()` 申请了该 pin。`fun_led.ko` 的 `gpio_request(28)` 是重复申请同一 pin。

**第二层（GPIO chip owner 模块检查，导致报 -517 而非 -EBUSY）：**
`gpiod_request()` 返回值初始值就是 `-EPROBE_DEFER`（`gpiolib.c:1988`）。在 GKI 架构下，gpio_chip 的 `owner` 指向 vendor 模块，`try_module_get(gdev->owner)` 返回 false 时维持 -517，不会进入 `gpiod_request_commit()` → 不会触发"已占用"的 `-EBUSY` 检查。

**正确的控制方式（现成方案，零代码）：**
```bash
echo 1 > /sys/class/leds/sys_led_red/brightness   # LED 亮
echo 0 > /sys/class/leds/sys_led_red/brightness   # LED 灭
echo heartbeat > /sys/class/leds/sys_led_red/trigger  # 心跳闪烁
```

**完整分析写入：** [level-5-GPIO.md](level-5-GPIO.md) → "实战：板子上已有的 DTS + gpio-leds 方案"

**关键代码路径：**
- DTS SoC 级：`common/common_drivers/arch/arm64/boot/dts/amlogic/meson-s7d.dtsi:456-469`（`bank@4000` 控制器节点）
- DTS 板级：`common/common_drivers/arch/arm64/boot/dts/amlogic/s7d_s905x5m_bm201.dts:216-229`（`gpio_leds { sys_led_red }` 节点）
- 驱动匹配：`common/drivers/leds/leds-gpio.c:187-190`（`of_gpio_leds_match[]`）
- GPIO 获取：`common/drivers/leds/leds-gpio.c:154`（`devm_fwnode_get_gpiod_from_child()`）
- LED 初始化：`common/drivers/leds/leds-gpio.c:75-123`（`create_gpio_led()`）
- sysfs 接口：`common/drivers/leds/led-class.c:38-65`（`brightness_store()`）
- GPIO 调用链：`common/drivers/gpio/gpiolib.c` → `common/common_drivers/drivers/gpio/pinctrl/pinctrl-meson.c` → 硬件寄存器
