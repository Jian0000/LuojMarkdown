+++
title = 'Level 4：写一个字符设备驱动'
date = '2026-06-09T15:00:00+08:00'
draft = false
+++

# Level 4：写一个字符设备驱动

## 我想做

在内核里加一个最小内核模块，用户态 `echo "hi" > /dev/simple_char` 能写进去，`cat /dev/simple_char` 能读出来。

## 先回答三个问题

1. **内核驱动和用户态程序的根本区别是什么？**
   - 驱动运行在内核态，可以访问所有硬件资源和内存
   - 用户态程序通过系统调用（open/read/write/ioctl）请求内核代为操作
   - 内核态出错 → 整个系统崩溃（kernel panic）；用户态出错 → 只崩那个进程

2. **Linux 设备驱动分哪几类？**
   - **字符设备**──像文件一样 open/read/write/close，顺序访问。例子：串口（`/dev/ttyAML0`）、GPIO、I2C、SPI、按键。你的板子上 `ls -l /dev/` 看到前面是 `c` 的都是字符设备
   - **块设备**──按块随机读写，有缓存层。例子：eMMC（`/dev/block/mmcblk0`）、U 盘、SD 卡。前面是 `b`
   - **网络设备**──通过 socket 接口访问，不通过文件系统。例子：以太网（eth0）、WiFi（wlan0）。`ip link show` 看到的就是

   你的板子上跑 `adb shell ls -l /dev/ | head -20`，看看哪些是字符设备（c）、哪些是块设备（b）

3. **什么是 file_operations？**
   - 它是一个结构体，里面全是函数指针
   - 每个字符驱动需要告诉内核："当用户调用 open() 时，执行我这个函数；调用 read() 时，执行我那个函数"
   - 这是一个多态机制——不同的驱动实现相同的接口，用户态不用关心底层实现

## 需要懂的知识

**驱动模块的骨架**

每个内核模块都有两个必经的函数：
```c
module_init(mod_init);   // insmod 时调用
module_exit(mod_exit);   // rmmod 时调用
```

`__init` 和 `__exit` 标记：这些函数在运行完后可以释放内存。

**register_chrdev 做了什么**

```c
major = register_chrdev(0, "simple_char", &fops);
```
- 向内核注册一个字符设备
- 第一个参数 0 表示让内核自动分配主设备号
- 返回的 major 就是你设备的编号，`cat /proc/devices` 能看到它
- `mknod /dev/simple_char c <major> 0` 就是创建一个指向这个编号的设备节点

**为什么用 copy_to_user 而不是 memcpy**

内核不能直接访问用户态传过来的指针，原因：
1. **安全**——用户态指针可能是伪造的，指向内核敏感数据
2. **分页**——用户态页面可能不在内存中（被换出了），直接访问会触发缺页
3. **权限**——内核必须显式地"拷入/拷出"用户空间

```c
copy_from_user(kernel_buffer, user_buffer, len);
copy_to_user(user_buffer, kernel_buffer, len);
```
这两个函数会检查指针合法性、处理缺页、返回未拷贝成功的字节数（0 表示全部成功）。

**Kconfig 和 Makefile 的角色**

- Kconfig——定义编译选项，`make menuconfig` 的界面就从这里生成
- Makefile——告诉 build 系统这个文件要编成模块（`obj-m`）还是编进内核（`obj-y`）

**为什么放在 drivers/misc/**

内核源代码树的 `drivers/` 下按功能划分子目录：

| 子系统 | 目录 | 举例 |
|--------|------|------|
| GPIO | `drivers/gpio/` | GPIO 控制器 |
| I2C | `drivers/i2c/` | I2C 总线及从设备 |
| 输入设备 | `drivers/input/` | 按键、触摸屏 |
| 串口 | `drivers/tty/` | UART 终端 |
| 杂项 | `drivers/misc/` | 不属于以上子类的驱动 |

`simple_char` 不涉及任何具体硬件外设，没有对应的子系统归属，所以放在 `drivers/misc/`——内核的"其他"目录。如果写的是 I2C 温度传感器驱动，则应放到 `drivers/i2c/` 以复用框架代码。

**VFS 层：用户 open → 驱动 dev_open 的完整链路**

Linux 内核 VFS（Virtual File System）是一个抽象层，让用户态可以**像操作普通文件一样操作设备**。从 `open("/dev/simple_char", O_RDWR)` 到 `dev_open()` 的完整流程分三步：

```
用户态：int fd = open("/dev/simple_char", O_RDWR);
  │
  ▼
第一步：系统调用入口（内核接手）
  ┌────────────────────────────────────────────────────┐
  │ sys_open() → 将用户态路径解析为 dentry + inode    │
  │ inode 中存有设备号：major = N, minor = 0          │
  │ 发现这是一个"字符设备"（inode 有 S_ISCHR 标记）    │
  └────────────────────────────────────────────────────┘
  │
  ▼
第二步：字符设备查找
  ┌────────────────────────────────────────────────────┐
  │ 内核从 inode 取出 major 号，到字符设备表          │
  │ (cdev_map / chrdevs 数组) 中查找对应的             │
  │ file_operations 结构体指针                        │
  │                                                    │
  │ 这是关键——你在 mod_init 里调用的                  │
  │ register_chrdev(0, "simple_char", &fops)           │
  │ 就是往这张表里写了一条：major → &fops              │
  └────────────────────────────────────────────────────┘
  │
  ▼
第三步：调用具体驱动函数
  ┌────────────────────────────────────────────────────┐
  │ VFS 把找到的 fops 指针填入当前进程的 fd 表中             │
  │ 然后调用：fops->open(inode, filep)                   │
  │ 即你写的：dev_open()                                 │
  │                                                    │
  │ 之后 read/write 走同样的路径                          │
  │ sys_read() → 从 fd 找到 fops → fops->read(...)      │
  └────────────────────────────────────────────────────┘
  │
  ▼
返回用户态：得到 fd = 3
```

**核心要点：**

- **VFS 是中间人**——用户态只跟 VFS 打交道（系统调用），VFS 负责把请求路由到正确的驱动
- **注册就是建立映射**——`register_chrdev` 的本质是在内核维护的**字符设备表**中登记 "major N → `fops`"
- **inode 是桥梁**——`mknod` 创建设备节点时，把 major/minor 写进了磁盘上的 inode；每次 open 时 VFS 从 inode 读设备号，再用设备号查 `file_operations`
- **read/write 不再查路径**——open 完成后，返回的 fd 已经直接关联到 `fops`，后续 read/write 通过 fd 直达驱动函数，不再走路径解析

正是因为这个设计，你才可以在用户态用标准的 `open/read/write` 操作任何字符设备，**驱动代码写的 file_operations 和用户态之间完全不需要约定任何额外接口**。

**ARCH 和 CROSS_COMPILE：交叉编译的必要性**

你的开发机是 x86_64 架构（PC），但板子是 **arm64**（AArch64）架构。x86 的机器编译不出 arm64 的可执行文件——这是**交叉编译**的场景。

**但在你的环境中，内核模块编译不通过命令行传 CROSS_COMPILE**

实际上你的 AOSP 项目 **没有 GCC 交叉编译工具链**（`prebuilts/gcc/` 下无 aarch64 目录），内核使用 **LLVM/Clang** 编译（`build.config.common` 中设了 `LLVM=1`）：

```bash
# 查看内核构建配置中的关键设置
$ grep -rn "LLVM\|CLANG_VERSION\|NDK_TRIPLE" build.config.common build.config.constants
build.config.common:  LLVM=1
build.config.constants: CLANG_VERSION=r487747c
build.config.constants: AARCH64_NDK_TRIPLE=aarch64-linux-android31
```

| 配置项 | 值 | 含义 |
|--------|-----|------|
| `LLVM=1` | 启用 | 使用 clang 而非 GCC 编译内核 |
| `CLANG_VERSION` | `r487747c` | 使用的 clang 版本 |
| `AARCH64_NDK_TRIPLE` | `aarch64-linux-android31` | NDK 目标三元组（带 API level） |

所以正确的编译方式有两种：

**方式一（推荐）：通过 AOSP 构建系统**
```bash
source build/envsetup.sh
lunch ross-userdebug
# 进入内核目录自动有环境变量
cd common/common14-5.15/common
make menuconfig
make modules
```
AOSP 的构建脚本会处理 clang 路径和 flags，不需要手动指定任何交叉编译参数。

**方式二（理解原理）：直接传给 make**
```bash
make ARCH=arm64 LLVM=1 \
    CLANG_TRIPLE=aarch64-linux-android31- \
    CROSS_COMPILE=aarch64-linux-android31- \
    modules
```
此处 `LLVM=1` 让内核使用 clang 系列工具，`CLANG_TRIPLE` 和 `CROSS_COMPILE` 告诉内核目标平台是 aarch64-android（API 31）。注意这里的后缀是 `aarch64-linux-android31-`（带 API level），而非常见的 `aarch64-linux-android-`。

**GCC 方式 vs Clang/LLVM 方式：两种交叉编译哲学**

如果你之前在嵌入式项目（如 DVB Stack、机顶盒 SDK）中见过这样的配置：

```bash
export ALI_TOOLCHAIN=/path/to/arm-none-linux-gnueabihf
```

那是 **GCC 式交叉编译**。和 Android 使用的 **Clang/LLVM 式** 有本质区别：

| | GCC 方式 | Clang/LLVM 方式 |
|--|---------|----------------|
| 编译器 | `arm-none-linux-gnueabihf-gcc` | `clang --target=aarch64-linux-android31` |
| 架构范围 | 一个编译器只编一种架构 | 一个 clang 编所有架构 |
| 换架构 | 换编译器前缀（`aarch64-linux-gnu-`） | 换 `--target` 参数 |
| libc | 捆绑在工具链目录里 | `--sysroot` 在命令行指定，与编译器解耦 |
| 环境配置 | shell 脚本 `export` 变量 | build.config / Bazel 管理 |
| 多架构 | 每个架构一套完整的 binutils+gcc+libc | 一套 Clang + 多份 sysroot |
| sysroot 位置 | 藏在工具链目录内部 | `prebuilts/build-tools/sysroots/`，显式独立 |

**为什么 Android 选 Clang？** Android 一个源码树要同时编译 ARM32、ARM64、x86_64 三种架构（甚至更多）。如果用 GCC，需要维护三套完整的 `binutils-gcc-glibc` 工具链。Clang 则是一个编译器 + 三个 `--target` + 对应的 sysroot 即可——这是 LLVM 的"交叉编译即原生编译"设计哲学。

**为什么 DVB Stack 用 GCC？** 芯片厂商（如 Ali、MStar）提供的 SDK 通常锁定一个固定架构（`arm-none-linux-gnueabihf`），GCC 工具链和 glibc 随 SDK 一起发布，不涉及多架构切换。此时 GCC 的工具链 "开箱即用" 反而是优势。

你的 `simple_char.ko` 和用户态测试程序编译，本质都是 **Clang + --target + sysroot** 这条路径。

**为什么直接 make 不行——Bazel 构建系统**

你的 AOSP 项目已经切换到 **Bazel** 来编译内核（`./build.sh -k` 底层调用 `bazel build`），和传统 `make` 有本质区别：

| | `make` 直接在源码树编译 | `bazel` 沙箱编译 |
|--|------------------------|-----------------|
| 源码树 | 留下 `.o`、`.config` 等中间文件 | **源码目录只读**，不产生任何垃圾 |
| 缓存 | 靠文件时间戳判断 | 基于 content hash，输入不变就复用 |
| 配置 | 读 `.config` | 从 `build.config.*` + defconfig/fragment 重新生成 |
| .ko 产物 | `drivers/misc/simple_char.ko` | 输出到 `out/bazel/` 下，不污染源码树 |

这解释了你实操中遇到的两个问题：

1. **源码树不干净**——在 `common/` 下敲 `make` 后留了中间文件，Bazel 检测到后拒绝编译（`The source tree is not clean, please run 'make mrproper'`）。修复：`make mrproper` 清理。

2. **改了 fragment 但 .config 没变**——Bazel 缓存了旧的 config step，没检测到 fragment 变化。修复：清 Bazel 缓存或 `rm -rf out/bazel/`。

**内核配置的正确修改路径：**

```
修改 amlogic_gki.fragment  →  Bazel 合并 gki_defconfig + fragment  →  生成 .config  →  编译
  你改这里                              Bazel 自动完成                          
```

**不要**在 `common/` 下直接执行 `make` 或 `make menuconfig`，它们会污染源码树且配置不会被 Bazel 采纳。

## 动手方案

```bash
# 第 1 步：找到内核目录
ls ~/android/aml/s905x5/aml-s905x5-androidu-v2/common/
# 这个是内核源码目录（kernel 5.15）

# 第 2 步：把驱动源码放进去
cat > ~/android/aml/s905x5/aml-s905x5-androidu-v2/common/common14-5.15/common/drivers/misc/simple_char.c << 'EOF'
/*
 * simple_char — 最小字符设备驱动
 *
 * 功能：用户态写入的数据暂存在内核缓冲区，读取时返回该数据。
 *       相当于一个"内核版记事本"，用于演示 Linux 字符驱动骨架。
 *
 * 测试方式：
 *   echo "hello" > /dev/simple_char
 *   cat /dev/simple_char
 */

/* module.h：模块的 module_init/module_exit/MODULE_LICENSE 等宏定义 */
#include <linux/module.h>
/* kernel.h：printk 等内核打印函数 */
#include <linux/kernel.h>
/* fs.h：register_chrdev、unregister_chrdev、file_operations 结构体 */
#include <linux/fs.h>
/* uaccess.h：copy_to_user / copy_from_user —— 内核<->用户空间数据拷贝 */
#include <linux/uaccess.h>

/* 设备名称——注册时告诉内核，也用于 /proc/devices 显示和 mknod */
#define DEVICE_NAME "simple_char"

/* major：内核分配的主设备号，mknod 时用来建立设备节点 */
static int major;
/*
 * kernel_buffer：内核侧的存储缓冲区。
 * 用户 write 的数据暂存于此，read 时从此读出。
 * 简单驱动用固定大小；真实驱动需考虑并发、缓冲区溢出等问题。
 */
static char kernel_buffer[256] = {0};

/*
 * dev_open — open() 系统调用的内核侧实现
 *
 * inodep：文件在 VFS 层对应的 inode，用来获取设备信息
 * filep：文件打开后在内核中的表示，每个进程的每次 open 对应一个实例
 *
 * 返回 0 表示成功。大部分简单驱动只需要记录日志即可。
 */
static int dev_open(struct inode *inodep, struct file *filep) {
    printk(KERN_INFO "simple_char: open()\n");
    return 0;
}

/*
 * dev_read — read() 系统调用的内核侧实现
 *
 * buffer：用户态传入的缓冲区指针（带 __user 标记，不可直接解引用）
 * len：用户态请求读取的字节数
 * offset：文件读写偏移位置，记录已读到的位置。读操作从这里继续，读完一段后向后移动
 *
 * copy_to_user：将内核数据拷贝到用户空间
 *   - 返回值 0 表示全部拷贝成功
 *   - 返回值非零 = 未能拷贝的字节数
 *
 * 返回规则：
 *   0  = EOF，表示已读到末尾，调用者（如 cat）收到 0 后会停止读循环并退出
 *   >0 = 本次实际读取的字节数
 *   <0 = 错误码
 */
static ssize_t dev_read(struct file *filep, char __user *buffer,
                        size_t len, loff_t *offset) {
    int data_len = strlen(kernel_buffer);

    /* 已读到数据末尾，返回 EOF */
    if (*offset >= data_len)
        return 0;

    /* 限制读取长度不超过剩余数据 */
    if (len > data_len - *offset)
        len = data_len - *offset;

    if (copy_to_user(buffer, kernel_buffer + *offset, len))
        return -EFAULT;

    *offset += len;
    printk(KERN_INFO "simple_char: read() %zu bytes, offset=%lld\n", len, *offset);
    return len;
}

/*
 * dev_write — write() 系统调用的内核侧实现
 *
 * buffer：用户态传入的数据指针（带 __user，需用 copy_from_user 访问）
 * len：用户态请求写入的字节数
 *
 * copy_from_user：将用户空间数据拷贝到内核缓冲区
 *   - 检查用户指针合法性，处理缺页
 *   - 返回值 = 未能拷贝成功的字节数（0 = 全部成功）
 */
static ssize_t dev_write(struct file *filep, const char __user *buffer,
                         size_t len, loff_t *offset) {
    int ret = copy_from_user(kernel_buffer, buffer, len);
    printk(KERN_INFO "simple_char: write() %zu bytes\n", len);
    return ret ? -EFAULT : len;
}

/*
 * file_operations 结构体：
 * 字符驱动的核心——将用户态系统调用（open/read/write/ioctl/release...）
 * 映射到驱动中的具体实现函数。这是 Linux 驱动多态机制的体现。
 *
 * 未被赋值的成员（如 release/ioctl）内核会使用默认行为。
 */
static struct file_operations fops = {
    .open = dev_open,
    .read = dev_read,
    .write = dev_write,
};

/*
 * mod_init — 模块入口（insmod 时执行）
 * __init 标记：此函数运行结束后内核可回收其内存
 *
 * register_chrdev(0, DEVICE_NAME, &fops)：
 *   - 参数 0 表示让内核自动分配主设备号
 *   - 返回的 major 保存分配到的设备号
 */
static int __init mod_init(void) {
    major = register_chrdev(0, DEVICE_NAME, &fops);
    printk(KERN_INFO "simple_char: loaded, major=%d\n", major);
    return 0;
}

/*
 * mod_exit — 模块出口（rmmod 时执行）
 * __exit 标记：模块内建时不编译此函数以节省内存
 *
 * unregister_chrdev：注销设备，释放主设备号
 * 必须在 exit 中注销，否则模块卸载后 /dev/simple_char 成为空悬节点。
 */
static void __exit mod_exit(void) {
    unregister_chrdev(major, DEVICE_NAME);
    printk(KERN_INFO "simple_char: unloaded\n");
}

/* 注册模块的初始化和退出函数 */
module_init(mod_init);
module_exit(mod_exit);

/* GPL 声明：使用 GPL 协议，不声明则内核拒绝加载或标记为污染 */
MODULE_LICENSE("GPL");
EOF

# 第 3 步：修改 Kconfig（让它在 menuconfig 中可见）
# 编辑 common/drivers/misc/Kconfig，在合适位置加入：
# tristate代表了三态 y/n/m  m表示模块，选择y那么simple_char.c 被编译进内核，选择m的话就会生成simple_char.ko
 config SIMPLE_CHAR
    tristate "Simple character device driver"
    help
      A minimal character device driver for learning.

# 第 4 步：修改 Makefile 加入编译
# 编辑 common/drivers/misc/Makefile，加入：
# 这里的CONFIG_SIMPLE_CHAR依赖于第3步中配置的SIMPLE_CHAR，系统会自动变成CONFIG_SIMPLE_CHAR
obj-$(CONFIG_SIMPLE_CHAR) += simple_char.o

# 第 5 步：注册到 Bazel 模块输出列表
# 编辑 common/common14-5.15/common/common_drivers/modules.bzl
# 在 AMLOGIC_COMMON_MODULES 列表中（按字母序）加入：
#     "drivers/misc/simple_char.ko",
# 否则 Bazel 虽然编译了模块但不会复制到 dist 输出目录
#  modules.bzl (AMLOGIC_COMMON_MODULES 列表)
#      → Bazel 编译生成 simple_char.ko
#      → depmod 生成 modules.dep（模块依赖关系）
#      → amlogic_utils.sh 把 modules.dep 转成 modules.load（去掉依赖信息，只留模块名）
#      → 放入 out/target/product/ross/vendor_dlkm/lib/modules/modules.load


# 第 6 步：编译内核模块
cd ~/android/aml/s905x5/aml-s905x5-androidu-v2/
cd common/common14-5.15/common
make menuconfig #在
# 在 Device Drivers → Misc devices 中找到 Simple character device driver，设为 M（模块）
make modules

# 第 7 步：测试连接
# 接上板子的 adb 连接
adb devices
# 预期输出：列出已连接的设备，状态为 "device"

# 确认模块已加载（烧录后自动加载，无需 insmod）
adb shell ls /vendor/lib/modules/simple_char.ko
# 预期输出：/vendor/lib/modules/simple_char.ko（文件存在）

adb shell cat /proc/devices | grep simple_char
# 预期输出：类似 "252 simple_char"
# 说明：主设备号（252 是示例，每次可能不同）是内核分配的，mknod 时需用这个数字

# 创建设备节点（让用户态能通过 /dev/simple_char 访问驱动）
adb shell mknod /dev/simple_char c <上一步中的major> 0
adb shell chmod 666 /dev/simple_char
# 说明：mknod 参数 c=字符设备，<major>=主设备号，0=次设备号
#       chmod 666 让所有用户可读写，避免权限拒绝

adb shell ls -l /dev/simple_char
# 预期输出：类似 "crw-rw-rw- 1 root root 252, 0 ... /dev/simple_char"
# 说明：开头的 c 表示字符设备，252,0 是主次设备号

# 第 8 步：验证驱动读写
# 写入数据：用 echo 重定向到设备节点，触发驱动的 dev_write()
adb shell "echo 'hello from userspace' > /dev/simple_char"

# 读出数据：用 cat 读取设备节点，触发驱动的 dev_read()
adb shell cat /dev/simple_char
# 预期输出：hello from userspace
# 说明：读出内容应与写入的一致，证明数据经过内核缓冲区转储正确

# 查看内核日志，确认驱动各函数被成功调用
adb shell dmesg | grep simple_char
# 预期输出：
#   simple_char: loaded, major=252          ← mod_init()，模块加载
#   simple_char: open()                      ← dev_open()，echo 触发 open
#   simple_char: write() 21 bytes            ← dev_write()，写入的数据长度
#   simple_char: open()                      ← dev_open()，cat 触发 open
#   simple_char: read() 21 bytes             ← dev_read()，读取的数据长度
```

## 验收清单

| 项目 | 验证方式 | 预期结果 |
|------|---------|----------|
| 模块已加载 | `cat /proc/devices \| grep simple_char` | 输出 `simple_char` 及主设备号 |
| 设备节点已创建 | `ls -l /dev/simple_char` | 输出 `c` 开头，带主次设备号 |
| 写入再读出 | `echo... > /dev/simple_char && cat /dev/simple_char` | 读出内容与写入一致 |
| 驱动函数被调用 | `dmesg \| grep simple_char` | 能看到 `open()/write()/read()` 日志 |
| 理解 VFS 调用链 | 能回答"用户态 open 如何到达 dev_open" | VFS → inode 设备号 → 字符设备表 → fops.open |
| 理解 copy_to_user | 能回答"为什么不用 memcpy" | 安全（指针伪造）、分页（缺页）、权限（内核显式拷贝） |

## 附：最小用户态测试程序

用 C 语言写一个测试程序，手动触发驱动的 `open/write/read/close`，比 shell 命令更直观地演示系统调用到驱动的完整链路：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    char wbuf[] = "hello driver";
    char rbuf[64] = {0};
    int fd;

    /* open: 用户态 open → VFS → 驱动 dev_open */
    fd = open("/dev/simple_char", O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }
    printf("open() fd=%d\n", fd);

    /* write: 用户态 write → VFS → 驱动 dev_write */
    write(fd, wbuf, strlen(wbuf));
    printf("write() %zu bytes\n", strlen(wbuf));

    /* read: 用户态 read → VFS → 驱动 dev_read */
    read(fd, rbuf, sizeof(rbuf));
    printf("read() got: %s\n", rbuf);

    /* close: 用户态 close → VFS → 驱动 dev_release（本例未实现，内核默认处理） */
    close(fd);
    printf("close()\n");

    return 0;
}
```

编译推送到板子运行：

```bash
# PC 端交叉编译（使用 musl 静态链接，无需依赖目标板的 libc）
CLANG=~/android/aml/s905x5/aml-s905x5-androidu-v2/prebuilts/clang/host/linux-x86/clang-r487747c/bin/clang
SYSROOT=~/android/aml/s905x5/aml-s905x5-androidu-v2/prebuilts/build-tools/sysroots/aarch64-unknown-linux-musl

$CLANG --target=aarch64-unknown-linux-musl \
    --sysroot=$SYSROOT \
    -static \
    -rtlib=compiler-rt \
    -unwindlib=libunwind \
    -o test_simple_char test_simple_char.c

# 验证二进制
file test_simple_char
# 预期输出：ELF 64-bit LSB executable, ARM aarch64, statically linked ...

# 推送到板子运行
adb push test_simple_char /data/local/tmp/
adb shell /data/local/tmp/test_simple_char

# 预期输出：
# open() fd=3
# write() 13 bytes
# read() got: hello driver
# close()
```

> **为什么用 musl 而不用 Android NDK 编译？**
> 项目的 `out/soong/ndk/sysroot` 只生成了 `arm-linux-androideabi`（32 位），没有 aarch64 的 NDK 库。但 `prebuilts/build-tools/` 下有完整的 **aarch64 musl** 工具链（crt、libc.a、头文件齐全）。用 `--target=aarch64-unknown-linux-musl` + `-static` 编译出的二进制不依赖目标板的 libc，在任何 aarch64 Linux（包括 Android Bionic）上都能运行。本质是内核只认 syscall，不管 libc 是谁。

对照驱动源码，每一行用户态程序都对应驱动 `file_operations` 中的一个函数调用。

对照驱动源码，每一行用户态程序都对应驱动 `file_operations` 中的一个函数调用。
