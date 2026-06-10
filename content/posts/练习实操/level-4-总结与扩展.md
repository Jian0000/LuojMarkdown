+++
title = 'Level 4 总结与扩展'
date = '2026-06-09T15:01:00+08:00'
draft = false
+++

# Level 4 总结与扩展

## 总结：一个字符驱动的完整生命周期

Level 4 的核心命题——"写一个内核模块，用户态能读能写"——拆解为三层：**内核侧（驱动代码）→ 构建侧（编译系统）→ 运行侧（板子验证）**。下面逐层复盘，标注优先级。

---

### 第一层：内核侧 —— 驱动代码本身

这层是**跨平台通用**的 Linux 驱动知识，与 Android 构建系统无关。

**🔴 核心必懂**

| 知识点 | 一句话 | 关联的坑 |
|--------|--------|----------|
| `file_operations` | 驱动的"函数表"——把用户态的系统调用（open/read/write）映射成驱动函数 | 多态机制，所有字符驱动公用 |
| `register_chrdev(0, name, &fops)` | 向内核注册字符设备，参数 0 让内核自动分配主设备号 | 返回值是主设备号，mknod 需要它 |
| `copy_from_user` / `copy_to_user` | 内核与用户空间之间的安全数据拷贝 | 不能直接用 `memcpy`——见下方说明 |
| `dev_read` 必须返回 0 表示 EOF | VFS 规范：当数据读完时必须返回 `0`，否则 `cat` 等工具无限循环 | **L4-05 bug**：忘记返回 0，cat 死循环 |
| `module_init` / `module_exit` | 模块加载/卸载的入口和出口 | 卸载时未 `unregister_chrdev` 会导致空悬设备节点 |

**为什么不能用 `memcpy` 替代 `copy_to_user`？** 三个原因：
1. **安全校验**：`copy_to_user` 会检查用户态指针是否在当前进程的地址空间内，防止内核被伪造指针攻击
2. **缺页处理**：用户态内存可能被换出（swap），`copy_to_user` 能触发缺页中断把页面换回来，`memcpy` 做不到
3. **SMAP/PAN 硬件保护**：ARMv8.1+ 有 PAN（Privileged Access Never）特性，内核态默认禁止直接访问用户态内存，`copy_to_user` 会临时关闭此保护

**VFS 调用链路**（用户态 `open("/dev/simple_char")` → 驱动 `dev_open()`）：

```
用户态 open() → sys_open() → VFS 层 → 从 inode 提取设备号
    → chrdevs[major].fops->open(inode, file)  ← 这就是你的 dev_open()
```

**🟡 了解即可**

- `__init` / `__exit` 标记：让内核在模块初始化/卸载后回收函数占用的内存，对常驻内存的内建驱动有意义，对可加载模块（`.ko`）影响不大
- `MODULE_LICENSE("GPL")`：不声明会导致内核标记模块为"污染"（tainted），但不会阻止加载

---

### 第二层：构建侧 —— 从源码到 .ko

这层有大量 **Android/Amlogic 构建系统特有的知识**，换一个项目可能需要重新理解。

**🔴 核心必懂**

| 知识点 | 一句话 | 关联的坑 |
|--------|--------|----------|
| Kconfig 条目 | 让 `make menuconfig` 或 fragment 能找到并启用 `CONFIG_SIMPLE_CHAR` | `drivers/misc/Kconfig` 添加 `config SIMPLE_CHAR` + `tristate` |
| Makefile 条目 | `obj-$(CONFIG_SIMPLE_CHAR) += simple_char.o`，把编译条件连接到配置项 | **L4-01/L4-02**：忘了加 → 编译时找不到目标 |
| fragment 配置 | 在 `amlogic_gki.fragment` 中添加 `CONFIG_SIMPLE_CHAR=m` | **L4-03**：改了错的 fragment 文件 |
| **两个 fragment 文件** | Bazel 用的在 `common_drivers/arch/arm64/configs/`，传统 make 用的在 `common/arch/arm64/configs/` | 修改配置前先确认 `build.config.amlogic.bazel` 的引用路径 |
| `modules.bzl` 注册 | Bazel 编译后，必须在 `AMLOGIC_COMMON_MODULES` 列表中声明 `.ko` 才会被拷贝到 dist | **L4-04**：编出来了但没拷贝 |
| `LLVM=1` 交叉编译 | 项目无 GCC 工具链，内核用 Clang 编译，目标 triple 为 `aarch64-linux-android31` | vermagic 不匹配就是因为 make 和 Bazel 工具链不同 |

**Bazel 构建流程图**（你的 `./build.sh -k` 实际做了什么）：

```
./build.sh -k
  → BUILD.bazel 加载 common_drivers/amlogic.bzl
  → 读取 build.config.amlogic.bazel
    → defconfig: gki_defconfig（GKI 基础配置）
    → fragment: common_drivers/arch/arm64/configs/amlogic_gki.fragment（厂商叠加）
  → Bazel 在沙箱中执行 make LLVM=1 ...（不污染源码树）
  → 输出 .ko 到 out/bazel/
  → modules.bzl 列出哪些 .ko 需要拷贝
  → 最终产物：out/android14-5.15/dist/simple_char.ko
```

**🟡 了解即可**

- **Bazel 缓存机制**：基于 content hash，输入不变则复用上次结果。改 `.c` 会自动检测，但改 fragment 有时需要清缓存（`rm -rf out/bazel/`）
- **`make mrproper`**：清理所有编译中间文件（`.config`、`.o` 等），Bazel 要求源码树绝对干净
- **vermagic 机制**：内核模块加载时内核会检查模块的 vermagic 字符串（包含内核版本、编译器版本等），不一致则拒绝加载。这就是为什么用 `make M=drivers/misc` 编的 `.ko` 在板子上 `insmod` 失败

---

### 第三层：运行侧 —— 从 .ko 到用户态可用

这层连接"编译产物"和"板子上能跑"。

**🔴 核心必懂**

| 知识点 | 一句话 | 关联的坑 |
|--------|--------|----------|
| `vendor_dlkm` 分区 | Android 厂商内核模块的专用分区，路径 `/vendor/lib/modules/` | 模块随镜像烧录自动部署 |
| `modules.load` 自动加载 | 第一阶段的模块自动加载列表，每次开机内核自动 `insmod` | 不在列表里的模块不会自动加载 |
| `mknod` 创建设备节点 | `mknod /dev/simple_char c <major> 0`，让用户态能通过路径访问设备 | 没有设备节点 → `/dev/simple_char` 不存在 → open 报 ENOENT |
| `chmod 666` | 允许非 root 用户读写设备，不加则 `echo` 可能报 Permission denied | 644 只允许 root 写 |
| `cat /proc/devices` | 查看已加载设备的名称和主设备号 | `mknod` 需要知道 major 号 |

**模块自动加载的两级机制：**

```
第一级（开机时）：modules.load → vendor_dlkm 中的模块
    			为什么需要 modules.load?——————因为有些驱动没有自动探测能力/系统启动必须依赖
第二级（按需加载）：modules.alias → 设备树匹配时 udev 自动加载
    			举例：USB鼠标
                你插入 USB 鼠标。
                系统：
                发现一个USB设备
                设备信息：
                        VID=046D
                        PID=C077
                内核：
                查找谁支持它
                然后：
                自动加载 usbhid.ko
```

`simple_char.ko` 在第一级 `modules.load` 中，因此刷机后自动加载，不需要手动 `insmod`。

**验证流程：**

```
1. ls /vendor/lib/modules/simple_char.ko  → 确认模块已部署
2. cat /proc/devices | grep simple_char   → 确认已加载 + 获取 major
3. mknod /dev/simple_char c <major> 0     → 创建设备节点（只需一次）
4. chmod 666 /dev/simple_char              → 放开权限
5. echo "hi" > /dev/simple_char && cat /dev/simple_char  → 验证读写
6. dmesg | grep simple_char                → 确认驱动函数被调用
```

---

## 扩展

### 驱动的三种存在形式

你写的 `simple_char.c` 最终编成了 `simple_char.ko`（内核模块），但这只是其中一种形式。Linux 驱动有三种存在方式：

| 形式 | 配置符号 | 产物 | 加载时机 | 如何确认 |
|------|---------|------|---------|---------|
| **内建（built-in）** | `CONFIG_XXX=y` | 编入 `vmlinux`（内核本体） | 内核启动时即存在 | 无法 `lsmod` 看到；`/sys/module/xxx/` 可能存在 |
| **模块（module）** | `CONFIG_XXX=m` | `xxx.ko` 文件 | 开机自动加载 或 手动 `insmod` | `lsmod` 能看到；`/vendor/lib/modules/xxx.ko` 存在 |
| **不编译** | `# CONFIG_XXX is not set` | 无 | — | — |

**内建 vs 模块的实际区别：**

```
内建 (y)：      vmlinux 二进制里就包含驱动代码
                优点：不需要任何文件系统，内核一启动就能用
                缺点：vmlinux 变大，占用内存，无法卸载
                典型场景：串口控制台驱动（console= 需要尽早输出）

模块 (m)：      独立的 .ko 文件，存放在文件系统上
                优点：按需加载/卸载，节省内存，方便调试替换
                缺点：依赖文件系统挂载，需要 modules.load 或 udev
                典型场景：WiFi 驱动、USB 设备驱动、你写的 simple_char
```

**你的项目中如何区分：**

```bash
# 查看哪些驱动是内建的（编进了 vmlinux）
grep "=y" common/arch/arm64/configs/amlogic_gki.fragment

# 查看哪些是模块
grep "=m" common/arch/arm64/configs/amlogic_gki.fragment
```

**为什么 simple_char 选 m 而不是 y？** 学习阶段用模块：改代码 → 重编 `.ko` → 推送到板子 → `rmmod && insmod`，秒级迭代。如果选 `y`，每次改驱动都要重编整个内核并重烧镜像，至少 10 分钟。

---

### 驱动和设备的对应关系

你写的 `simple_char` 很特殊——它是**纯软件驱动**，不控制任何硬件。但真正的驱动（GPIO、I2C、SPI、USB 等）需要解决一个问题：**内核如何知道"这个驱动"负责"这个设备"？**

这就是 **驱动-设备匹配** 机制。

#### 总线模型（Bus-Driver-Device）

Linux 把驱动和设备组织在**总线**上：

```
总线（bus）—— 平台总线 / I2C 总线 / SPI 总线 / USB 总线 / PCI 总线
  ├── 驱动侧（driver）—— 声明"我能驱动哪些设备"
  └── 设备侧（device） —— 声明"我是谁"（设备树节点 / ACPI 表 / 动态枚举）
```

总线负责**对号入座**——当设备出现时，遍历已注册的驱动，找到匹配的就调用驱动的 `probe()` 函数。

**对比你的 simple_char：**

| | simple_char | 典型硬件驱动（如 GPIO） |
|--|------------|----------------------|
| 注册方式 | `register_chrdev()` 直接注册 | 先向总线注册 `driver`，匹配成功才 `probe()` |
| 设备来源 | 纯软件，没有对应硬件 | 设备树（`/devicetree`）中的节点 |
| 匹配方式 | 无匹配——创建设备节点就可用 | `compatible` 字符串匹配（见下文） |
| `probe()` | 无 | 有，匹配后初始化硬件 |

#### 三种常见的匹配方式

**方式一：设备树 compatible 匹配（你板子上的主流方式）**

你的板子（S905X5M）的设备树中有这样的节点：

```dts
// 示例：arch/arm64/boot/dts/amlogic/g12a_s905x2.dts 中的串口
serial@24000 {
    compatible = "amlogic,meson-gx-uart";  // ← 这是"我是谁"的声明
    reg = <0x24000 0x18>;
    interrupts = <...>;
};
```

驱动代码中：

```c
// drivers/tty/serial/meson_uart.c
static const struct of_device_id meson_uart_dt_match[] = {
    { .compatible = "amlogic,meson-gx-uart" },  // ← "我能驱动这个"
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, meson_uart_dt_match);

static struct platform_driver meson_uart_driver = {
    .probe  = meson_uart_probe,   // 匹配成功 → 调用此函数
    .driver = {
        .name = "meson_uart",
        .of_match_table = meson_uart_dt_match,
    },
};
```

**匹配流程：** 内核解析设备树 → 创建 `platform_device` → 扫描已注册的 `platform_driver` → `compatible` 字符串一致 → 调用 `.probe()`

**方式二：设备 ID 表（USB/PCI 设备）**

```
USB 鼠标插入
  → 内核读取设备描述符：VID=046D, PID=C077（罗技）
  → 扫描 USB 驱动中 usb_device_id 表
  → usbhid 驱动声明了 USB_DEVICE(0x046D, 0xC077)
  → 匹配！→ 调用 usbhid_driver.probe()
```

这对应扩展前总结中提到的 `modules.alias` 第二级加载机制。

**方式三：设备名称匹配（platform_driver 的 `.name` 回退）**

```c
// 驱动侧
static struct platform_driver my_driver = {
    .driver = {
        .name = "my_device",  // ← 名称
    },
};

// 设备侧（在板级文件中静态声明，老式方法）
static struct platform_device my_device = {
    .name = "my_device",      // ← 名称一致 → 匹配
};
```

有设备树后这种方式逐渐被 compatible 方式取代，但 `.name` 字段在 `platform_driver_register()` 时的 sysfs 目录命名中仍然重要。

#### 回到你的 simple_char：为什么不需要匹配？

`simple_char` 用 `register_chrdev()` 注册，走的是**最古老的字符设备接口**：

```
register_chrdev() → 在 chrdevs[] 数组中占据一个 major → 返回 major 号
用户态 mknod /dev/simple_char c <major> 0 → 在文件系统中创建入口
用户态 open("/dev/simple_char") → VFS 查 chrdevs[major] → 找到 fops
```

这条路径**不经过总线、不经过 probe、不经过设备树**。它是一个完全由用户态驱动的"虚拟设备"——设备节点存在即可使用，没有对应的硬件。

**你的 Level 5 中会涉及真正硬件驱动的写法**，届时会看到 `platform_driver`、`compatible` 匹配、`probe()` 函数、设备树节点这些概念的实际使用。

### 三类 dlkm (动态独立内核模块)分区：GKI 架构下的内核模块隔离

Android 的 GKI（Generic Kernel Image）架构把内核模块拆到了三个独立分区。你的编译产物中存在三个 `_dlkm` 目录：

```
out/target/product/ross/
├── system_dlkm/    ← GKI 通用模块（Google 维护）
├── vendor_dlkm/    ← Amlogic 厂商模块
└── odm_dlkm/       ← ODM 设备品牌定制模块（本项目中为空）
```

#### 各自的内容和来源

| 分区 | 模块来源 | 板子路径 | 数量 | 典型内容 |
|------|---------|---------|------|---------|
| **system_dlkm** | `common/`（GKI 上游内核） | `/system/lib/modules/<kernel-version>/` | 16 个（你项目） | USB 网卡、PPP、Bluetooth、CAN、zram |
| **vendor_dlkm** | `common_drivers/`（Amlogic 厂商驱动） | `/vendor/lib/modules/` | 128 个（你项目） | Amlogic GPIO/I2C/SPI/CLK/DRM/thermal/`simple_char` |
| **odm_dlkm** | ODM 自己的驱动源码 | `/odm/lib/modules/` | 0 个（你的项目不需要） | 屏幕触摸校准、特殊按键映射等 |

**你的 `simple_char.ko` 在 vendor_dlkm 中**，因为它在 `modules.bzl` 的 `AMLOGIC_COMMON_MODULES` 列表里，这个列表对应 `vendor_dlkm` 分区的模块。

#### system_dlkm vs vendor_dlkm 的细微差异

除了源码来源不同，路径结构也有区别：

```
# system_dlkm —— 带内核版本号，保留 kernel 源码路径层级
/system/lib/modules/5.15.170-android14-11-g3efc8295014d-ab12916019/
    kernel/drivers/net/usb/usbnet.ko

# vendor_dlkm —— 不带版本号，扁平化存放
/vendor/lib/modules/
    simple_char.ko
```

`system_dlkm` 带完整的内核版本号路径，是为了支持**多个内核版本共存**——Google 可以用同一个 system 镜像配合不同版本的内核。`vendor_dlkm` 由 Amlogic 随 BSP 一起发布，内核版本固定，不需要版本化路径。

#### 为什么分成三个分区——独立 OTA 更新

这是 GKI 架构的核心价值所在：

```
Google 更新 Linux 内核 CVE 漏洞：
  只需推送 system_dlkm 分区，不动 vendor_dlkm 和 odm_dlkm

Amlogic 更新 HDMI 驱动 bug：
  只需推送 vendor_dlkm 分区，不动 system_dlkm

设备品牌修改屏幕参数：
  只需推送 odm_dlkm 分区，不动前两个
```

三个分区各自独立签名、独立更新，互不干扰。这解决了旧 Android 的毒瘤——"内核一锅粥"：vendor 改了内核源码，Google 的安全补丁就合不进去；Google 更新了内核，vendor 的驱动就挂。GKI 通过**接口冻结**（KMI，Kernel Module Interface）保证 `system_dlkm` 模块升级后，`vendor_dlkm` 模块不会因为符号表变化而加载失败。

#### 和你学过的 modules.load 的对应关系

每个分区有自己的 `modules.load`，由各自的 `init` 阶段按顺序加载：

```
开机启动顺序：
  1. system_dlkm 模块 → /system/lib/modules/<ver>/modules.load
  2. vendor_dlkm 模块 → /vendor/lib/modules/modules.load   ← simple_char.ko 在这里
  3. odm_dlkm 模块    → /odm/lib/modules/modules.load
```

这解释了为什么 `simple_char.ko` 在 `vendor_dlkm/lib/modules/modules.load` 中——它是 vendor_dlkm 分区的第一级加载列表，加载时机在 system_dlkm 之后。

### 一个驱动对应多个设备

你的 `simple_char` 是 **一对一** 关系——一个驱动管理一个设备。但现实中的驱动通常是 **一对多**：一个 LED 驱动同时控制板子上的 3 个 LED，一个 GPIO 驱动管理 32 路 GPIO。

下面用**酒店比喻**从头到尾解释一对多的完整机制。目标是让你看懂每一行代码在"真实世界"中对应什么。

---

#### 整体类比

```
酒店集团（驱动）开分店
  → 分店1 在 上海（LED 红灯）
  → 分店2 在 北京（LED 绿灯）
  → 分店3 在 深圳（LED 蓝灯）
```

**概念映射表：**

| 概念 | 酒店比喻 | 代码中的东西 |
|------|---------|------------|
| 酒店集团 = 驱动本身 | "simple_led 酒店集团" | `platform_driver` 结构体 |
| 分店地址编号 = 设备号 | 上海=251.00, 北京=251.01, 深圳=251.02 | `dev_t` (major + minor) |
| 分店名 = 设备节点 | 上海分店, 北京分店... | `/dev/simple_led_0` |
| 共享服务标准 = fops | 所有分店共用同一套"客房服务流程" | `file_operations` |
| 分店选址清单 = 设备树 | 工商局的"待建分店规划" | device tree 节点 |

---

#### 第一步：酒店集团注册领执照

```
你要开连锁酒店 → 先去工商局注册公司 → 拿到营业执照号
```

代码中的 `alloc_chrdev_region`：

```c
static dev_t dev_num;   // 营业执照号

/* 向内核（工商局）申请一段连续的设备号 */
alloc_chrdev_region(&dev_num, 0, MAX_LEDS, "simple_led");
/*         ↑          ↑    ↑    ↑        ↑
 *     "帮我分配"   执照号  从01 最多16  公司名
 */
```

结果：拿到了营业执照号 `major=251`（主设备号）。内核知道"有一个叫 simple_led 的公司，它的执照号范围是 `251.00 ~ 251.15`"。

**但是此时还没开任何店，也没招员工。**

---

#### 第二步：注册连锁品牌

```
拿到了营业执照 → 还要注册品牌名 → 方便客人通过品牌名找到分店
```

代码中的 `class_create`：

```c
led_class = class_create(THIS_MODULE, "simple_led");
```

效果：内核在 `/sys/class/` 下创建了一个 `simple_led` 目录。

```
/sys/class/simple_led/     ← 品牌名注册处
```

这个 `class` 的作用是：后面你每开一家分店（`device_create`），它会自动在 `/sys/class/simple_led/` 下创建一个条目，udev（内核的"物业系统"）看到后自动去 `/dev/` 下创建设备节点。

**没有 class，你得手动 `mknod`。有 class，系统自动生成。**

---

#### 第三步：声明"我能管理什么类型的店"

```
酒店集团发布公告：「我们能管理任何位于 上海/北京/深圳 的地段」
```

代码中的 `of_device_id` 匹配表：

```c
/* 酒店集团的"选址标准"——什么样的地段我会去开店 */
static const struct of_device_id led_dt_match[] = {
    { .compatible = "simple,led-driver" },  /* ← 我只看地块上写没写这句话 */
    { }   /* 表结束标记 */
};
```

然后把它装进 `platform_driver`（酒店管理公司资质）：

```c
static struct platform_driver led_driver = {
    .probe  = led_probe,         /* 选好址后，派我去管理这家店 */
    .driver = {
        .name = "simple_led",
        .of_match_table = led_dt_match,   /* ← 我的选址标准 */
    },
};
```

`platform_driver_register(&led_driver)` —— 向"建设局（内核）"注册。内核（建设局）收到注册后，立刻翻开"在建/已建项目清单"（设备树），**找所有写着 `compatible = "simple,led-driver"` 的地块**。每找到一个，就派 `led_probe()` 去一次。

---

#### 第四步：设备树——建设局的"待建清单"

设备树就是政府的土地规划图，上面画好了每一块地要建什么：

```dts
leds {
    compatible = "simple,led-driver";   /* ← 这块地皮属于 simple_led 集团 */

    led_red {
        label = "red";
        gpios = <&gpio 15 0>;           /* 建在地块 15 上 */
    };
    led_green {
        label = "green";
        gpios = <&gpio 16 0>;           /* 建在地块 16 上 */
    };
    led_blue {
        label = "blue";
        gpios = <&gpio 17 0>;           /* 建在地块 17 上 */
    };
};
```

内核启动时解析设备树，为每个子节点创建一个 `platform_device`（相当于"建设用地批复文件"），然后等着匹配驱动。

**至此，所有准备工作完成。下面发生的事就是一切自动串联起来的。**

---

#### 第五步：probe —— 派店长去管理每一家分店

当 `platform_driver_register` 执行后，建设局（内核）的动作：

```
建设局翻开清单：
  → 找到 led_red:  地块上写着 "simple,led-driver" ✓   → 派 led_probe()
  → 找到 led_green:地块上写着 "simple,led-driver" ✓   → 派 led_probe()
  → 找到 led_blue: 地块上写着 "simple,led-driver" ✓   → 派 led_probe()
```

`led_probe()` 被调用了 **三次**，每次进来时 `pdev` 指向不同的地块：

```c
/* 第一次进来：pdev = led_red
 * 第二次进来：pdev = led_green
 * 第三次进来：pdev = led_blue
 */
static int led_probe(struct platform_device *pdev) {
```

**probe 是"店长"**，它负责：

1. **给这家店分配编号**（minor）——你是第几家开的分店
2. **租地建店**（获取 GPIO）——这块地（GPIO 引脚）归我管
3. **挂上招牌**（`device_create` → `/dev/simple_led_N`）——客人能找到了

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/gpio/consumer.h>
#include <linux/cdev.h>
#include <linux/fs.h>

#define DEVICE_NAME "simple_led"
#define MAX_LEDS    16

static dev_t dev_num;           /* 营业执照号（第一次分配到的起始设备号） */
static struct class *led_class; /* 品牌名（/sys/class/simple_led/） */
static int led_count;           /* 记录开了几家分店，用作次设备号索引 */

/* ---------- 每家分店的私有数据 ---------- */
struct led_device {
    struct cdev     cdev;       /* 这家店的"营业许可证" */
    struct gpio_desc *gpio;     /* 这家店占用的地块（GPIO） */
    int             minor;      /* 分店编号（次设备号） */
    bool            state;      /* 灯当前亮灭 */
};

/* ---------- probe：每一次调用 = 开一家新分店 ---------- */
static int led_probe(struct platform_device *pdev) {
    struct led_device *led;
    struct device *dev = &pdev->dev;   /* 这家店的规划文件 */
    dev_t devt;

    /* 1. 租场地——从设备树取 GPIO */
    led->gpio = devm_gpiod_get(dev, NULL, GPIOD_OUT_LOW);
    /*    "这块地（GPIO）归我管了，默认输出低电平（灯灭）" */

    /* 2. 分店编号：第一家拿 0，第二家拿 1，第三家拿 2 */
    led->minor = led_count++;

    /* 3. 办营业许可证 + 挂招牌 */
    devt = MKDEV(MAJOR(dev_num), led->minor);   /* 分店完整地址：251.0, 251.1, 251.2 */
    cdev_init(&led->cdev, &led_fops);            /* 制作"服务菜单" */
    cdev_add(&led->cdev, devt, 1);               /* 正式取得营业许可 */
    device_create(led_class, dev, devt, NULL,
                  "simple_led_%d", led->minor);  /* 挂招牌 → 自动生成 /dev/simple_led_N */

    dev_info(dev, "LED %d probed\n", led->minor);
    return 0;
}
```

三次 probe 完成后，局面：

```
/dev/simple_led_0  ← 次设备号=0，控制 GPIO 15（red）
/dev/simple_led_1  ← 次设备号=1，控制 GPIO 16（green）
/dev/simple_led_2  ← 次设备号=2，控制 GPIO 17（blue）
```

**三个 `/dev/` 节点，共用同一套 `led_fops`。**

#### fops 声明和读写实现：

```c
static ssize_t led_read(struct file *filep, char __user *buf,
                        size_t len, loff_t *off) {
    /* filep->private_data — "你的房卡"上的分店标记 */
    struct led_device *led = filep->private_data;
    char val = led->state ? '1' : '0';
    if (*off >= 1) return 0;
    if (copy_to_user(buf, &val, 1)) return -EFAULT;
    *off += 1;
    return 1;
}

static ssize_t led_write(struct file *filep, const char __user *buf,
                         size_t len, loff_t *off) {
    struct led_device *led = filep->private_data;
    char val;
    if (copy_from_user(&val, buf, 1)) return -EFAULT;
    led->state = (val == '1');
    gpiod_set_value(led->gpio, led->state);  /* 只操作这一个 LED 的 GPIO */
    return len;
}

static struct file_operations led_fops = {
    .owner = THIS_MODULE,
    .read  = led_read,
    .write = led_write,
};
```

---

#### 第六步：用户来住店 —— private_data 是怎么连上的

用户态执行：

```bash
echo 1 > /dev/simple_led_0   # 点亮红灯
```

内核内部流程：

```
用户 open("/dev/simple_led_0")
  → VFS 从文件名查到 inode
  → 从 inode 提取设备号 (251, 0)
  → 从字符设备表找到 cdev（= red 那份）
  → 从 cdev 找到 fops（所有店共用 led_fops）
  → 调用 open 实现（把 filep->private_data 指向 red 的 led_device）
  → 返回 fd = 3 给用户

用户 write(fd, "1", 1)
  → sys_write() → 通过 fd 找到 file 结构体
  → file->private_data → 指向了前面存好的 led_device（red，GPIO 15）
  → 调用 led_write()：

        struct led_device *led = filep->private_data;  // = red 那个实例
        gpiod_set_value(led->gpio, 1);                  // 只操作 GPIO 15，不动其他灯
```

**`private_data` 的桥梁作用：**

```
probe 时：创建 led_device（red, gpio=15）→ 记在内存里
open 时：根据次设备号找到对应的 led_device → 塞进 filep->private_data
write 时：从 private_data 取出 led_device → 操作它的 GPIO
```

每个进程打开的文件拥有独立的 `file` 结构体，所以多个进程同时操作不同的 LED，各自的 `private_data` 指向不同的 LED 实例，不会搞混。

---

#### 第七步：每个概念的关键代码映射

```c
// 1. 领营业执照（分配设备号范围）
alloc_chrdev_region(&dev_num, 0, MAX_LEDS, "simple_led");

// 2. 注册品牌（创建设备类，使自动生成 /dev/ 节点成为可能）
led_class = class_create(THIS_MODULE, DEVICE_NAME);

// 3. 声明选址标准（告诉内核我的匹配规则）
static const struct of_device_id led_dt_match[] = {
    { .compatible = "simple,led-driver" },
    { }
};
MODULE_DEVICE_TABLE(of, led_dt_match);

// 4. 向建设局注册触发匹配
static struct platform_driver led_driver = {
    .probe  = led_probe,             // 配上一个被调用三次的函数
    .driver = { .of_match_table = led_dt_match, },
};
platform_driver_register(&led_driver);  // → 内核立刻扫设备树，匹配就 probe

// ===== 以上：准备工作 =====
// ===== 以下：probe 里的事，每个匹配的设备分支自动执行一次 =====

// 5. 租场地（申请 GPIO）
led->gpio = devm_gpiod_get(dev, NULL, GPIOD_OUT_LOW);

// 6. 分店编号（分配 minor）
led->minor = led_count++;

// 7. 正式营业（cdev_add + device_create）
devt = MKDEV(MAJOR(dev_num), led->minor);
cdev_init(&led->cdev, &led_fops);
cdev_add(&led->cdev, devt, 1);
device_create(led_class, dev, devt, NULL, "simple_led_%d", led->minor);
```

---

#### 第八步：对比 simple_char 和你以前见过的代码

| 对比项 | simple_char（你写的） | 本页的 LED 驱动（现代标准写法） |
|-------|---------------------|------------------------------|
| 注册函数 | `register_chrdev(0, ...)` 全范围注册（旧 API） | `alloc_chrdev_region()` + `cdev_add()`（精确控制） |
| 设备实例数 | 1 个，hardcode 在源码里 | 每匹配一个设备树节点自动创建一个 |
| 区分实例 | 无（只有一个） | `filep->private_data` |
| 设备节点 | 手动 `mknod` | `class_create` + `device_create` 自动生成 |
| 和硬件关系 | 纯虚拟，无硬件 | `gpiod_set_value()` 操作真实 GPIO |
| 和设备树关系 | 无关 | `of_match_table` + `probe()` 配对 |

`register_chrdev` 是 Linux 2.6 之前的遗留接口，现代驱动都用 `cdev_add` + `device_create`——你 Level 5 中做 GPIO 驱动时会直接使用后者的模式。

> **一篇写给自己的话：** 当初学这一节时最困惑的是——为什么这么复杂？`register_chrdev` 一行能搞定的事情，为什么要拆成 `alloc_chrdev_region` + `class_create` + `cdev_init` + `cdev_add` + `device_create` 五步？答案在于场景：`register_chrdev` 适合"一个驱动只管一个设备"的玩具代码，真正产品级的驱动需要管理多个设备实例（多个 LED、多个 GPIO bank），每一步拆开就是为了精确控制每一个实例的生命周期。