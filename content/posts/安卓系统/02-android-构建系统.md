+++
title = '02-Android构建系统入门'
date = '2026-06-09T18:02:00+08:00'
draft = false
+++

# Android 构建系统入门

> 学习日期: 2026-04-28
> 参考项目: Amlogic S905X5M (ross) Android 14 TV
> 前置知识: 了解 Makefile 基本语法

---

## 1. Android 构建系统的演进

Android 构建系统经历了三个阶段：

```
Before 2014: Android.mk (纯 Makefile)
     ↓
2014-2019:   Android.mk + Android.bp (Soong) 混用
     ↓
2019+:       Android.bp (Soong) 为主，逐步淘汰 Android.mk
     ↓
Future:      Bazel 构建 (Google 正在推进)
```

当前项目的状态：**两者混用**。参考代码库中 vendor/giec 既有 24 个 Android.mk，也有 8 个 Android.bp。Google 推荐新代码用 Android.bp。

---

## 2. Android.bp (Soong) 语法详解

### 2.1 基本结构

Android.bp 使用 **Blueprint** 格式（类似 JSON 的 DSL），每个代码模块定义为 **一个模块类型 + 属性块**：

```blueprint
// 最简单的例子: 把一组文件打包成一个文件组
filegroup {
    name: "giec_android_certificate_directory",
    srcs: [
        "*.pk8",
        "*.pem",
    ],
}
```

> 来源: `vendor/giec/android-certs/Android.bp`

---

### 2.2 常用模块类型

| 模块类型 | 说明 | 对应什么 |
|---------|------|---------|
| `cc_library_shared` | C/C++ 动态库 (.so) | libstbcmdservicehal.so |
| `cc_library_static` | C/C++ 静态库 (.a) | |
| `cc_binary` | C/C++ 可执行文件 | HIDL 服务进程 |
| `java_library` | Java 库 (.jar) | |
| `android_app` | Android 应用 APK | |
| `filegroup` | 文件分组 | "把私钥文件打包成一个模块名" |
| `hidl_package_root` | HIDL 包根路径声明 | |
| `cc_defaults` | 公共编译参数模板 | 被其他模块引用 |
| `vintf_compatibility_matrix` | VINTF 兼容性矩阵 | |

---

### 2.3 实战例 1: C 动态库

```blueprint
// vendor/giec/common/libraries/hwstbcmdapi/hidl_wrapper/Android.bp
cc_library_shared {
    name: "libstbcmdservicehal",    // 模块名, 其他模块通过这个名字引用
    proprietary: true,               // 专有, 不公开源码
    srcs: ["StbCmdShellhal.c"],      // 源文件
    cflags: [                        // 编译 flags
        "-Wlong-long",
        "-Wformat",
        "-Wpointer-arith",
    ],
    shared_libs: [                   // 链接的动态库
        "liblog",
        "libutils",
        "libcutils",
    ],
}
```

编译产物: `libstbcmdservicehal.so`

---

### 2.4 实战例 2: C++ 动态库 + 可执行文件

```blueprint
// vendor/giec/hardware/interfaces/hwstbcmdservice/1.0/default/Android.bp

// 第一部分: 生成 .so 动态库
cc_library_shared {
    name: "giec.hardware.hwstbcmdservice@1.0-impl",
    defaults: ["hidl_giec"],          // 引用公共编译参数
    vendor: true,                       // 放在 vendor 分区
    proprietary: true,
    srcs: [ "Hwstbcmdservice.cpp" ],   // 源文件
    include_dirs: ["..."],             // 头文件搜索路径
    cflags: ["-Wall", "-Werror"],
    shared_libs: [                     // 链接库 (非常重要!)
        "liblog",
        "libhardware",
        "libhidlbase",
        "libutils",
        "libstbcmdservicehal",          // 链接上一个 .bp 编译的库
        "giec.hardware.hwstbcmdservice@1.0",
    ],
}

// 第二部分: 生成可执行文件 (服务进程)
cc_binary {
    name: "giec.hardware.hwstbcmdservice@1.0-service",
    defaults: ["hidl_giec"],
    relative_install_path: "hw",         // 安装在 vendor/bin/hw/ 下
    vendor: true,
    proprietary: true,
    srcs: ["service.cpp"],
    init_rc: ["giec.hardware.hwstbcmdservice@1.0-service.rc"],
    cflags: ["-Wall", "-Werror"],
    shared_libs: [
        "liblog",
        "libbinder",
        "libutils",
        "libhidlbase",
        "giec.hardware.hwstbcmdservice@1.0",
        "giec.hardware.hwstbcmdservice@1.0-impl",
    ],
}
```

**关键理解**:
- `name` 是模块的唯一标识，其他模块通过 name 引用
- `defaults` 可以引用 `cc_defaults` 定义的公共参数，避免重复
- `vendor: true` = 编译产物安装到 `vendor` 分区，`vendor: false` = 安装到 `system` 分区
- `shared_libs` 告诉构建系统链接哪些动态库，构建系统会自动处理依赖顺序

---

### 2.5 公共参数模板

```blueprint
// vendor/giec/hardware/interfaces/Android.bp
cc_defaults {
    name: "hidl_giec",
    cflags: [
        "-Wall",
        "-Werror",
    ],
}
```

其他模块通过 `defaults: ["hidl_giec"]` 引用，所有模块共享这些编译参数。

---

### 2.6 HIDL 包根路径声明

```blueprint
// vendor/giec/hardware/interfaces/Android.bp
hidl_package_root {
    name: "giec.hardware",
    path: "vendor/giec/hardware/interfaces",
}
```

告诉 hidl-gen 工具：`giec.hardware.*` 开头的 HIDL 接口定义在这个目录下。

---

### 2.7 子目录包含

```blueprint
// vendor/giec/common/Android.bp
optional_subdirs = ["*"]      // 包含所有子目录

// vendor/giec/hardware/interfaces/Android.bp
subdirs = ["*"]                // 包含所有子目录(非可选)
```

这两行的区别:
- `optional_subdirs = ["*"]`: 子目录可选, 子目录不存在不会报错
- `subdirs = ["*"]`: 子目录必须存在, 否则报错

---

## 3. Android.mk 语法详解

### 3.1 基本结构

Android.mk 是传统的 Makefile 格式。每个模块的写法是：

```makefile
LOCAL_PATH := $(call my-dir)           # 获取当前路径
include $(CLEAR_VARS)                   # 清除上一次的 LOCAL_ 变量

# module settings
LOCAL_MODULE := my_module               # 模块名
LOCAL_SRC_FILES := src/main.c           # 源文件
LOCAL_CFLAGS := -Wall -O2              # C flags
LOCAL_SHARED_LIBRARIES := liblog        # 链接的动态库

include $(BUILD_SHARED_LIBRARY)         # 指定构建类型
```

最后一句的 `BUILD_*` 变量决定了模块类型：

| BUILD_* 变量 | 模块类型 |
|-------------|---------|
| `BUILD_SHARED_LIBRARY` | 动态库 (.so) |
| `BUILD_STATIC_LIBRARY` | 静态库 (.a) |
| `BUILD_EXECUTABLE` | 可执行文件 |
| `BUILD_PACKAGE` / `BUILD_PREBUILT` | APK 包 |
| `BUILD_JAVA_LIBRARY` | Java 库 |

---

### 3.2 实战例 3: 预置 APK (带平台签名)

```makefile
# vendor/giec/apps/OTAClient/Android.mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

# 自动找到目录下的所有 .apk 文件
APPS := $(notdir $(wildcard $(LOCAL_PATH)/*.apk))
APP_NAME := $(basename $(APPS))

LOCAL_MODULE_TAGS := optional
LOCAL_MODULE := $(APP_NAME)          # 模块名 = APK 文件名
LOCAL_SRC_FILES := $(APPS)           # 源文件是预编译的 APK
LOCAL_MODULE_CLASS := APPS           # 声明这是一个 APK 模块
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := platform        # ★ 使用平台 key 签名 → 获得系统权限
LOCAL_PRIVILEGED_MODULE := false     # 是否特权应用 (放在 priv-app)
LOCAL_OPTIONAL_USES_LIBRARIES := androidx.window.extensions androidx.window.sidecar

include $(BUILD_PREBUILT)            # 预编译规则
```

**关键理解**:

- `LOCAL_CERTIFICATE := platform` — 这是厂商定制的精髓。用 platform key 签名的 APK 运行在 system 进程中，可以获得系统级权限。普通用户安装的 APK 没有这个权限。

常见取值:
  - `platform`: 平台签名, 系统权限
  - `media`: 媒体签名, 可访问媒体资源
  - `shared`: 共享签名
  - `PRESIGNED`: 保留原始签名 (不重新签名)

---

### 3.3 实战例 4: JNI 动态库

```makefile
# vendor/giec/common/libraries/hwstbcmdapi/jni/Android.mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE := libhistbcmdservice_jni    # 生成 .so 供 Java 调用
LOCAL_MODULE_TAGS := optional
LOCAL_CFLAGS := -DANDROID_NDK
LOCAL_MULTILIB := both                     # 同时编译 32/64 位
LOCAL_SRC_FILES := cn_giec_adp_cmd.cpp     # JNI 桥接源文件

LOCAL_LDLIBS := -ldl -llog
LOCAL_C_INCLUDES += $(JNI_H_INCLUDE) ...   # 头文件路径

LOCAL_SHARED_LIBRARIES := \               # 链接的系统库
    libandroid_runtime \
    libbinder \
    libutils \
    libcutils \
    libhistbcmdservicemanageclient \
    libgui \
    libhwbinder \
    libnativehelper

include $(BUILD_SHARED_LIBRARY)
```

这个 .so 是 Java → C++ 的桥梁。Java 代码通过 JNI 调用到这个库，再由这个库调用底层 C++ HIDL 服务。

---

### 3.4 目录包含机制

```makefile
# vendor/giec/Android.mk
include $(call all-subdir-makefiles)

# vendor/giec/executable/Android.mk
include $(all-subdir-makefiles)
```

- `$(call all-subdir-makefiles)`: 递归包含所有子目录中的 Android.mk
- `include $(all-subdir-makefiles)`: 包含同一层所有子目录的 Android.mk

这就是为什么只要在根目录加一个 Android.mk，整个目录树的模块都会被编译。

---

## 4. 构建系统的核心流程

### 4.1 从 build.sh 开始

参考项目的构建入口 `build.sh`:

```bash
# 简化后的调用链
./build.sh -acukoj200
  ↓
build.cfg 读取配置 (PRODUCT_NAME, TYPE, ...)
  ↓
source build/envsetup.sh    # 设置环境变量
  ↓
lunch ross-userdebug       # 选择 target 产品
  ↓
make -j200                  # 调用 make, 读取 build/make/core/main.mk
  ↓
递归扫描所有 Android.mk / Android.bp
  ↓
按依赖关系编译所有模块
  ↓
打包成各分区镜像 (system.img, vendor.img, boot.img, ...)
  ↓
生成 OTA 包 (如果 -o 指定)
```

### 4.2 PRODUCT_PACKAGES 机制

```makefile
# 在 device.mk 或 device-giec.mk 中:
PRODUCT_PACKAGES += \
    LeanKeyboard \           # → 找 name = "LeanKeyboard" 的模块
    OTAClient \              # → 找 name = "OTAClient" 的模块
    BazeportLauncher \       # → 找 name = "BazeportLauncher" 的模块
    libstbcmdservicehal      # → 找 name = "libstbcmdservicehal" 的模块
```

`PRODUCT_PACKAGES` 是**产品清单**。构建系统会在所有 Android.mk/Android.bp 中搜索 `name` 匹配的模块，编译并打包进镜像。

**不写在 PRODUCT_PACKAGES 中的模块不会被编译**，即使 Android.mk 文件存在。这是理解 Android 构建的关键。

### 4.3 文件拷贝机制

另一种常用的方式是把文件直接拷贝到镜像中：

```makefile
# device/amlogic/ross/device.mk
PRODUCT_COPY_FILES += \
    device/amlogic/ross/init.amlogic.board.rc:$(TARGET_COPY_OUT_VENDOR)/etc/init/hw/init.amlogic.board.rc
```

格式: `源文件:目标路径(相对分区根目录)`

`$(TARGET_COPY_OUT_VENDOR)` = `vendor`，所以目标路径是 `vendor/etc/init/hw/init.amlogic.board.rc`

---

## 5. 分区与模块安装位置

Android.bp 中的属性决定模块安装到哪个分区：

| Android.bp 属性 | 安装分区 | 对应的 make 变量 |
|----------------|---------|----------------|
| `vendor: true` | `/vendor/` | `$(TARGET_COPY_OUT_VENDOR)` |
| `product_specific: true` | `/product/` | `$(TARGET_COPY_OUT_PRODUCT)` |
| `system_ext_specific: true` | `/system_ext/` | `$(TARGET_COPY_OUT_SYSTEM_EXT)` |
| 默认 | `/system/` | |

```blueprint
// 示例: 不同分区的模块
cc_binary {
    name: "vendor_bin",
    vendor: true,                              # → /vendor/bin/
}

cc_binary {
    name: "system_bin",
    # 没有 vendor/product_specific/system_ext_specific
    # → /system/bin/ (默认)
}

cc_binary {
    name: "product_bin",
    product_specific: true,                    # → /product/bin/
}
```

---

## 6. Android.mk vs Android.bp 对比

| 对比项 | Android.mk | Android.bp |
|-------|-----------|-----------|
| 语法 | Makefile | Blueprint (JSON-like) |
| 条件语句 | 支持 (ifeq/ifneq) | ❌ **不支持** |
| 变量 | 随意定义 | ❌ 不支持自定义变量 |
| 通配符 | 支持 (wildcard) | ❌ 必须显式列出 |
| 推荐度 | 旧项目, 正在淘汰 | 新项目首选 |
| 何时用 | 需要条件判断 | 模块配置固定 |

**经验法则**: 如果不需要 `ifeq` 做条件判断，用 Android.bp。需要根据变量决定编译内容，用 Android.mk。

当前项目中两种混用的原因：
- `vendor/giec/apps/OTAClient/Android.mk` — 因为它要自动通配 APK 文件
- `vendor/giec/hardware/interfaces/Android.bp` — 因为 HIDL 是新特性，用新语法

---

## 7. 核心概念总结

```
┌─────────────────────────────────────────────────────┐
│                    build.sh                          │
│  读取 build.cfg → 设置环境 → 调用 make                │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│                  build/make/core/main.mk             │
│   Android 构建系统入口, 加载所有 mk/bp 文件           │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│   Android.mk / Android.bp 扫描                        │
│   每个模块定义: name, srcs, 依赖, 安装位置             │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│   PRODUCT_PACKAGES 筛选                               │
│   只有列在 PRODUCT_PACKAGES 中的模块才被编译           │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│   依赖分析 → 编译 → 链接 → 打包 → 生成镜像            │
└─────────────────────────────────────────────────────┘
```

---

## 8. 从构建到镜像：模块、文件与分区组装

构建系统的最终产物是**各个分区镜像**。理解镜像如何组装，需要区分三类来源：

### 8.1 镜像内容的三大来源

```
                              ┌──────────────────────┐
        Android.bp/Android.mk │  Modules (模块)       │  ← 编译出来的 .so/.apk/可执行文件
        PRODUCT_PACKAGES +=   │  cc_binary, android_app│
                              └──────────┬───────────┘
                                         ↓
                              ┌──────────────────────┐
        PRODUCT_COPY_FILES    │  RAW Files (文件拷贝)   │  ← 直接拷贝的配置文件/脚本
        / PRODUCCT_COPY_FILES  │  .rc, .xml, .cfg, .sh │
                              └──────────┬───────────┘
                                         ↓
                              ┌──────────────────────┐
        BoardConfig.mk        │  Image Config (镜像配置)│  ← 文件系统类型、大小、签名
        device.mk             │  erofs, ext4, f2fs     │
                              └──────────────────────┘
                                         ↓
                              ┌──────────────────────┐
                              │   Final Image         │
                              │  system.img / vendor.img / boot.img ... │
                              └──────────────────────┘
```

**来源 1 — 编译出来的模块 (Modules)**:

Android.bp/Android.mk 定义模块 → PRODUCT_PACKAGES 选中 → 编译 → 安装到对应分区

```makefile
# device-giec.mk 中: 这些模块会被编译并装入 system 或 vendor 分区
PRODUCT_PACKAGES += \
    LeanKeyboard \              # APK → system/system_ext/app/
    libstbcmdservicehal \       # .so → vendor/lib64/
    giec.hardware.hwstbcmdservice@1.0-service  # 可执行文件 → vendor/bin/hw/
```

**来源 2 — 直接拷贝的文件 (File Copy)**:

不需要编译，直接把源文件按目标路径打包进镜像

```makefile
# device/amlogic/ross/device.mk 中的例子:
PRODUCT_COPY_FILES += \

    # 配置文件 → vendor/etc/
    device/amlogic/ross/files/mixer_paths.xml:$(TARGET_COPY_OUT_VENDOR)/etc/mixer_paths.xml \
    device/amlogic/ross/files/mesondisplay.cfg:$(TARGET_COPY_OUT_VENDOR)/etc/mesondisplay.cfg \

    # init 启动脚本 → vendor/etc/init/hw/
    device/amlogic/ross/init.amlogic.board.rc:$(TARGET_COPY_OUT_VENDOR)/etc/init/hw/init.amlogic.board.rc \

    # shell 脚本 → vendor/bin/
    vendor/giec/executable/get_display_id.sh:vendor/bin/get_display_id.sh \

    # adb 公钥
    vendor/giec/common/keys/adbkey.pub:vendor/etc/adb/preinstalled_keys

    # 隐私政策
    vendor/giec/files/Privacy_Policy.txt:vendor/etc/Privacy_Policy.txt
```

路径中的 `$(TARGET_COPY_OUT_VENDOR)` = `vendor`，所以目标路径是 `vendor/etc/mixer_paths.xml`，最终在镜像中对应 `/vendor/etc/mixer_paths.xml`。

**来源 3 — 镜像配置 (Image Config)**:

```makefile
# BoardConfig.mk 中:
BOARD_SYSTEMIMAGE_FILE_SYSTEM_TYPE := erofs    # system 分区用 erofs (只读)
BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE := erofs    # vendor 分区用 erofs
BOARD_USERDATAIMAGE_FILE_SYSTEM_TYPE := f2fs   # userdata 分区用 f2fs (读写)
BOARD_ODM_EXTIMAGE_FILE_SYSTEM_TYPE := ext4     # odm_ext 分区用 ext4

BOARD_USERDATAIMAGE_PARTITION_SIZE := 576716800 # userdata 分区大小 (550MB)
BOARD_BOOTIMAGE_PARTITION_SIZE := 67108864      # boot 分区大小 (64MB)
BOARD_SUPER_PARTITION_SIZE := 3355443200        # super 分区大小 (3.2GB)
```

---

### 8.2 各分区镜像构成详解

以 ross (S905X5M) 为例，完整的分区表如下（来自 `device/amlogic/ross/part_table_5_15.txt`）:

```
// partitionName, startsddr, size, gap, attr
bootloader_a, -, 8M,  1M, 0x0001    ← 物理分区
boot_a,       -, 64M, 1M, 0x0001    ← 物理分区, 包含 kernel+ramdisk
super,        -, 3200M, 1M, 0x0001  ← 物理分区, 内部再分割为逻辑分区
userdata,     -, -,    1M, 0x1004   ← 剩余所有空间
```

每个镜像的构成拆解：

| 镜像文件 | 分区 | 内容由什么决定 | 主要来源 |
|---------|------|--------------|---------|
| **bootloader.img** | bootloader_a/b | U-Boot 编译产物 | `bootloader/uboot-repo/` 单独构建 |
| **boot.img** | boot_a/b | kernel + ramdisk | `kernel/` 编译 + `BOARD_BOOTIMAGE_PARTITION_SIZE` |
| **vendor_boot.img** | vendor_boot_a/b | vendor 内核模块+DTB | `vendor_dlkm` + DTB |
| **dtb.img** | dtb (分区) | 设备树编译产物 | `./mk ross -v common14-5.15` 构建 |
| **dtbo.img** | dtbo_a/b | 设备树 overlay | `BOARD_DTBOIMG_PARTITION_SIZE` |
| **super.img** | super (动态分区) | 打包所有动态分区 | 见下方拆分 |
| ├ **system.img** | *(逻辑卷)* | system 分区内容 | `TARGET_COPY_OUT_SYSTEM` |
| ├ **vendor.img** | *(逻辑卷)* | vendor 分区内容 | `TARGET_COPY_OUT_VENDOR` |
| ├ **product.img** | *(逻辑卷)* | product 分区内容 | `TARGET_COPY_OUT_PRODUCT` |
| ├ **system_ext.img** | *(逻辑卷)* | system_ext 分区内容 | `TARGET_COPY_OUT_SYSTEM_EXT` |
| ├ **odm_ext.img** | *(逻辑卷)* | odm 分区内容 | `TARGET_COPY_OUT_ODM` |
| ├ **vendor_dlkm.img** | *(逻辑卷)* | 可加载内核模块 | `TARGET_COPY_OUT_VENDOR_DLKM` |
| ├ **system_dlkm.img** | *(逻辑卷)* | 系统内核模块 | `TARGET_COPY_OUT_SYSTEM_DLKM` |
| └ **odm_dlkm.img** | *(逻辑卷)* | OEM 内核模块 | `TARGET_COPY_OUT_ODM_DLKM` |
| **vbmeta.img** | vbmeta_a/b | AVB 验证元数据 | `BOARD_AVB_*_ADD_HASHTREE_FOOTER_ARGS` |
| **userdata.img** | userdata | 空文件系统 | 首次启动后填充 |

---

### 8.3 以 vendor.img 为例：完整的文件来源追踪

vendor 分区是最复杂的，它同时包含模块产物、拷贝文件、专有二进制：

#### 模块 (Android.bp/Android.mk → 编译)

```blueprint
// .so 动态库 → vendor/lib64/
cc_library_shared {
    name: "libstbcmdservicehal",
    vendor: true,                    # 装到 vendor 分区的 lib64/
    ...
}

// 可执行文件 → vendor/bin/
cc_binary {
    name: "giec.hardware.hwstbcmdservice@1.0-service",
    vendor: true,
    relative_install_path: "hw",     # → vendor/bin/hw/
    ...
}

// APK → vendor/app/ 或 vendor/priv-app/
// 如果没有指定 vendor: true, 默认装到 system 分区
```

#### 文件拷贝 (PRODUCT_COPY_FILES → 直接打包)

```makefile
# 直接拷贝到 vendor 分区的各处:
→ vendor/etc/mixer_paths.xml           # ALSA 音频配置
→ vendor/etc/media_codecs.xml          # 媒体编解码配置
→ vendor/etc/mesondisplay.cfg          # 显示配置
→ vendor/etc/mali_platform.config      # GPU 配置
→ vendor/etc/init/hw/init.amlogic.board.rc   # 开机启动脚本
→ vendor/bin/get_display_id.sh         # shell 脚本
→ vendor/bin/setup_station.sh          # 网络配置脚本
```

#### 专有二进制 (Prebuilt)

```makefile
# vendor/giec/apps/OTAClient/Android.mk
LOCAL_SRC_FILES := $(APPS)             # 预编译的 APK, 不是源码
```

第三方厂商提供的闭源 .so 也属于此类，放在 `vendor/amlogic/restricted_libs/` 或 `vendor/amlogic/proprietary/`。

---

### 8.4 各分区的典型内容速查

| 分区 | 典型内容 | 文件系统 | 读/写 |
|------|---------|---------|------|
| **system** | Android Framework, 系统应用, 核心库 | erofs | 只读 |
| **system_ext** | 厂商扩展的系统能力 | erofs | 只读 |
| **vendor** | BSP 库, HAL 实现, 设备配置, 驱动模块 | erofs | 只读 |
| **product** | 产品特定配置, 壁纸, 铃声, OTA 配置 | erofs | 只读 |
| **odm** | OEM 定制 (品牌标识, 差异化配置) | ext4 | 只读(可挂载为rw) |
| **vendor_dlkm** | 可动态加载的内核模块 (.ko) | erofs | 只读 |
| **system_dlkm** | GKI 兼容的内核模块 | erofs | 只读 |
| **boot** | kernel + ramdisk | raw | - |
| **vendor_boot** | vendor 内核模块 + DTB + vendor ramdisk | raw | - |
| **userdata** | 应用数据, 设置, 用户安装的应用 | f2fs | 读写 |
| **metadata** | 加密 metadata, 锁屏密码相关 | ext4 | 读写 |

---

### 8.5 总结：一次构建中发生了什么

```
source build/envsetup.sh
  → 设置各种 TARGET_COPY_OUT_* 环境变量
  → 设置 BOARD_* 分区配置

lunch ross-userdebug
  → 选择 device/amlogic/ross 的配置
  → 加载 BoardConfig.mk (分区大小, 文件系统)
  → 加载 device.mk (PRODUCT_PACKAGES, PRODUCT_COPY_FILES)

make -j200
  → Soong 解析所有 Android.bp
  → Kati 解析所有 Android.mk (转换成 ninja)
  → Ninja 执行编译
  → 编译产物安装到 out/target/product/ross/
  → 按分区目录组装:
      out/.../system/  ← 所有 TARGET_COPY_OUT_SYSTEM 模块+文件
      out/.../vendor/  ← 所有 vendor:true 模块+文件
      out/.../product/ ← 所有 product_specific 模块+文件
  → 打包成镜像:
      system.img ← out/.../system/ (erofs)
      vendor.img ← out/.../vendor/ (erofs)
      super.img  ← system.img + vendor.img + product.img + ... (动态分区)
      boot.img   ← kernel + ramdisk (raw)
      vbmeta.img ← AVB 签名信息
  → 生成 OTA 包 (如果 -o 指定)
```

---

## 9. 实操练习

要检验理解，可以尝试以下操作：

### 练习 1: 追踪模块（以 `libstbcmdservicehal` 为例）

下面是对模块 `libstbcmdservicehal` 的完整追踪，精确到每个文件位置和配置原因。

#### 文件 1: 模块定义 — Android.bp

**文件位置**: `vendor/giec/common/libraries/hwstbcmdapi/hidl_wrapper/Android.bp`

```blueprint
cc_library_shared {
    name: "libstbcmdservicehal",
    proprietary: true,
    srcs: ["StbCmdShellhal.c"],
    cflags: ["..."],
    shared_libs: ["liblog", "libutils", "libcutils"],
}
```

**为什么这么配置**:
- `cc_library_shared` → 编译产物是 `.so` 动态库，供其他模块在运行时链接
- `proprietary: true` → 这是厂商专有代码，不对外公开源码（APACHE 协议的项目不会包含它）
- `srcs: ["StbCmdShellhal.c"]` → 源文件在**同目录** `vendor/giec/common/libraries/hwstbcmdapi/hidl_wrapper/StbCmdShellhal.c`
- `shared_libs: ["liblog", "libutils", "libcutils"]` → 链接 Android 基础库（日志、工具函数、系统裁剪库）

**这个模块是干什么的**:
查看 `StbCmdShellhal.c` 的代码，核心逻辑只有两个函数:

- `do_system_cmd()` — 调用 `popen()` 执行 shell 命令
- `hsInvokeNative()` — 对外暴露的接口，接收命令字符串，返回执行结果

简单说：**这个库是系统应用执行 shell 命令的桥梁**。系统应用通过 JNI 调用到这个库，最终用 `popen()` 执行命令。

#### 文件 2: PRODUCT_PACKAGES 引用 — device-giec.mk

**文件位置**: `vendor/giec/device-giec.mk`

```makefile
PRODUCT_PACKAGES += \
    libhistbcmdservice_jni \
    libhistbcmdservicemanageclient \
    libstbcmdservicehal       # ← 在这里
```

**为什么这么配置**:
- `PRODUCT_PACKAGES` 是"产品清单"，只有列在这里的模块才会被编译进固件
- 这里同时列出了相关的 3 个库，构成了完整的调用链：
  ```
  Java App → libhistbcmdservice_jni (JNI 桥) 
           → libhistbcmdservicemanageclient (客户端管理)
           → libstbcmdservicehal (实际执行 shell 命令)
  ```

#### 文件 3: 依赖引用 — Android.bp

**文件位置**: `vendor/giec/hardware/interfaces/hwstbcmdservice/1.0/default/Android.bp`

```blueprint
cc_library_shared {
    name: "giec.hardware.hwstbcmdservice@1.0-impl",
    ...
    shared_libs: [
        ...
        "libstbcmdservicehal",     # ← 链接到这个库
        "giec.hardware.hwstbcmdservice@1.0",
    ],
}
```

**为什么这么配置**:
- HIDL 服务的实现 (`-impl`) 需要实际执行 shell 命令的能力，所以链接了 `libstbcmdservicehal`
- 这构成了一条完整链路: `HIDL Service → libstbcmdservicehal → popen() → shell 命令执行`

#### 文件 4-5-6: 包含链（为什么 device-giec.mk 会生效）

```
ross.mk  line 228                    # 产品主配置
  └→ $(call inherit-product, device/amlogic/common/products/mbox/product_mbox.mk)
       line 1
       └→ $(call inherit-product, device/amlogic/common/core_amlogic.mk)
            line 40
            └→ $(call inherit-product-if-exists, vendor/giec/device-giec.mk)
                 line 30
                 └→ PRODUCT_PACKAGES += libstbcmdservicehal
```

| 文件 | 位置 | 作用 |
|------|------|------|
| `ross.mk` | `device/amlogic/ross/ross.mk:228` | S905X5M 产品入口 |
| `product_mbox.mk` | `device/amlogic/common/products/mbox/product_mbox.mk:1` | mbox 产品类型通用配置 |
| `core_amlogic.mk` | `device/amlogic/common/core_amlogic.mk:40` | Amlogic 平台核心配置，**触发 GIEC 定制** |

**为什么这样设计**:
- Amlogic 把 `vendor/giec/device-giec.mk` 放在 `core_amlogic.mk` 中条件包含，意味着**只要用的是 Amlogic 平台，厂商定制自动生效**
- GIEC 只需要维护 `vendor/giec/` 目录，不需要修改 Amlogic 的核心配置

#### 文件 7: 模块扫描入口 — Android.mk

**文件位置**: `vendor/giec/Android.mk`

```makefile
include $(call all-subdir-makefiles)
```

**为什么这么配置**:
- 这一行让构建系统**递归扫描** `vendor/giec/` 下所有子目录的 `Android.mk` 和 `Android.bp`
- 文件虽然简短，但它构成了**模块发现的起点**。没有它，`libstbcmdservicehal` 即使有 Android.bp 也不会被构建系统看到

#### 完整追踪总结

```
libstbcmdservicehal.so 的完整生命周期
═══════════════════════════════════════

[模块发现]
vendor/giec/Android.mk                      ← 递归扫描所有子目录
    ↓
[模块定义]
vendor/giec/common/libraries/hwstbcmdapi/
  hidl_wrapper/Android.bp                    ← name: "libstbcmdservicehal"
    ↓                                          cc_library_shared → .so
[源文件]
  hidl_wrapper/StbCmdShellhal.c              ← popen() 执行 shell 命令
    ↓
[产品清单]
vendor/giec/device-giec.mk                   ← PRODUCT_PACKAGES += libstbcmdservicehal
    ↓
[包含链]
device/amlogic/ross/ross.mk                ← 产品入口
  → product_mbox.mk                          ← 产品类型
  → core_amlogic.mk                          ← 核心配置, include device-giec.mk
    ↓
[编译]
构建系统扫描所有 Android.bp/Android.mk
→ 匹配 name: "libstbcmdservicehal"
→ 编译 StbCmdShellhal.c → libstbcmdservicehal.so
→ 拷贝到 out/target/product/ross/vendor/lib64/
    ↓
[运行时链接]
giec.hardware.hwstbcmdservice@1.0-service    ← HIDL 服务进程
  → giec.hardware.hwstbcmdservice@1.0-impl   ← 实现库
      → libstbcmdservicehal.so               ← 实际执行 shell 命令
          → popen() → system commands
    ↓
[JNI 调用]
libhistbcmdservice_jni.so                     ← Java → Native 桥接
  → 系统 APK (BazeportSystem 等)             ← Java 层调用
```

### 练习 2: 理解依赖链（以 OTAClient 为例）

追踪 OTAClient，它不是一个简单的模块，而是涉及构建系统、产品配置、运行时属性、系统应用集成的完整案例。

#### 引子: OTAClient 是什么

OTAClient 是 GIEC 厂商的**定制 OTA 升级客户端**。它替代了 AOSP 原生的系统更新功能，与 GIEC 自建的 OTA 服务器通信，完成固件升级包的下载和安装。

包名: `com.giec.otaclient`，版本 1.0.5，预编译 APK（无源码）。

#### 文件 1: 模块定义 / 安装清单 — Android.mk

**文件位置**: `vendor/giec/apps/OTAClient/Android.mk`

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

APPS := $(notdir $(wildcard $(LOCAL_PATH)/*.apk))    # 自动找到 OTAClient.apk
APP_NAME := $(basename $(APPS))

LOCAL_MODULE_TAGS := optional
LOCAL_MODULE := $(APP_NAME)                          # → 模块名: "OTAClient"
LOCAL_SRC_FILES := $(APPS)
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := platform                         # ★ 平台签名 → 系统权限
LOCAL_PRIVILEGED_MODULE := false                      # 普通系统应用，非 priv-app

include $(BUILD_PREBUILT)                              # 预编译 APK 规则
```

**为什么这样配置**:
- `LOCAL_CERTIFICATE := platform` — 用平台 key 签名。这个 APK 需要访问系统级 API（比如调用 Recovery 升级），普通签名没有这个权限
- `LOCAL_PRIVILEGED_MODULE := false` — 安装在 `system/app/` 而非 `system/priv-app/`。priv-app 有更高的权限等级，但 OTAClient 当前的设计不需要
- `BUILD_PREBUILT` — 不是从源码编译，直接打包预编译的 `OTAClient.apk`

#### 文件 2: PRODUCT_PACKAGES 引用 — device-giec.mk

**文件位置**: `vendor/giec/device-giec.mk`

```makefile
PRODUCT_PACKAGES += \
    LeanKeyboard \
    OTAClient \                  # ← 列在产品清单中
    BazeportLauncher \
    BazeportSystem
```

**为什么**: `PRODUCT_PACKAGES` 是"产品安装清单"，只有列在这里的模块才会被编译进固件。

#### 文件 3: 包含链（为什么 device-giec.mk 会生效）

```
ross.mk  line 228
  └→ $(call inherit-product, device/amlogic/common/products/mbox/product_mbox.mk)
       line 1
       └→ $(call inherit-product, device/amlogic/common/core_amlogic.mk)
            line 40
            └→ $(call inherit-product-if-exists, vendor/giec/device-giec.mk)
                 line 47 → PRODUCT_PACKAGES += OTAClient
```

| 文件 | 位置 | 作用 |
|------|------|------|
| `ross.mk` | `device/amlogic/ross/ross.mk:228` | S905X5M 产品入口 mk |
| `product_mbox.mk` | `device/amlogic/common/products/mbox/product_mbox.mk:1` | mbox 产品类型配置，包含 core_amlogic.mk |
| `core_amlogic.mk` | `device/amlogic/common/core_amlogic.mk:40` | **触发 GIEC 定制: include vendor/giec/device-giec.mk** |
| `device-giec.mk` | `vendor/giec/device-giec.mk:47` | 列出 OTAClient |

#### 文件 4: OTA 运行时配置 — product_mbox.mk

**文件位置**: `device/amlogic/common/products/mbox/product_mbox.mk`

```makefile
# OTA update config (line 271-276)
PRODUCT_PROPERTY_OVERRIDES += \
    persist.sys.otaurl.ext=$(OTA_UPDATE_REQUEST_URL_EXT) \
    persist.sys.otaurl.int=$(OTA_UPDATE_REQUEST_URL_INT) \
    vendor.build.app.id=$(APP_ID) \
    vendor.build.app.secret=$(APP_SECRET)
```

**为什么这样配置**:
- 这些是**运行时属性**，OTAClient 在设备启动后读取这些属性来连接 OTA 服务器
- `persist.sys.otaurl.ext` — OTA 服务器外网地址
- `persist.sys.otaurl.int` — OTA 服务器内网地址
- `vendor.build.app.id` / `.secret` — 设备身份认证，每个产品型号不同

#### 文件 5: 配置源头 — build.cfg

**文件位置**: 项目根目录 `build.cfg`

```bash
# build.cfg 第 75 行
GIEC_BUILD_NUMBER=1.0.00000                    # 固件版本号

#OTA 服务器地址（第 104-105 行，默认注释掉）
#OTA_UPDATE_REQUEST_URL_EXT=http://14.21.46.164:880/app-api/ota/device/check
#OTA_UPDATE_REQUEST_URL_INT=http://14.21.46.164:880/app-api/ota/device/check

# 各产品的 APP_ID/APP_SECRET（第 86-102 行）
#RAMAN_APP_ID=Kr1Jtb
#RAMAN_APP_SECRET=bykGNPARzlFhoG4E
#ROSS_APP_ID=lFxCQa
#ROSS_APP_SECRET=4FitnLGrsQJkorPg
...
```

**为什么这样配置**: build.sh 读取 build.cfg，根据 `PRODUCT_NAME` 选择对应的 `APP_ID`/`APP_SECRET`，然后 export 为环境变量，传递给 make 系统。

#### 文件 6: 构建时版本注入 — build.sh

**文件位置**: 项目根目录 `build.sh`

```bash
# line 5: 版本号文件路径
VERSION_FILE="/home/workspace/venus/public_repository/.an14_build_version"

# line 218: 自动版本号递增逻辑
export GIEC_BUILD_NUMBER=$new_version

# line 351-370: 从 build.cfg 读取配置
GIEC_BUILD_NUMBER)   ... ;;
OTA_UPDATE_REQUEST_URL_EXT) export OTA_UPDATE_REQUEST_URL_EXT=$value ;;
OHM_WV4_APP_ID)      export APP_ID=$value ;;
OHM_WV4_APP_SECRET)  export APP_SECRET=$value ;;
```

#### 文件 7: 系统设置集成 — AboutFragment.java

**文件位置**: `packages/apps/TvSettings/Settings/src/com/android/tv/settings/about/AboutFragment.java:399-412`

```java
case KEY_SYSTEM_UPDATE_SETTINGS:
    /*
     * Due to an update in Google's GMS, there is a conflict
     * between GMS and our OTAClient over handling the system update.
     * This conflict may cause the system update to launch GMS update
     * instead of our custom OTAClient.
     *
     * To resolve this issue, we explicitly set the Intent's target
     * component to com.giec.otaclient's MainActivity.
     */
    Intent systemUpdateIntent = new Intent();
    systemUpdateIntent.setComponent(
        new ComponentName(
            "com.giec.otaclient",
            "com.giec.otaclient.ui.activities.MainActivity"
        )
    );
    startActivity(systemUpdateIntent);
    break;
```

**为什么这样修改**: GMS（Google Mobile Services）会抢占系统更新的 intent。厂商通过**修改 AOSP 源码**（TvSettings），强制将系统更新按钮指向自己的 OTAClient，而不是 Google 的更新组件。

这表明 GIEC 不仅自己写应用，也**修改了 AOSP 原生代码**来集成自己的功能。

#### 完整依赖链总结

```
OTAClient 的完整生命周期
════════════════════════════

[配置源头]
build.cfg                                          ← GIEC_BUILD_NUMBER, APP_ID/APP_SECRET, OTA URL
    ↓
[构建脚本读取]
build.sh                                           ← 解析 build.cfg → export 环境变量
    ↓
[版本管理]
.an14_build_version                                ← 自动维护版本号递增
    ↓
[模块定义]
vendor/giec/apps/OTAClient/Android.mk              ← 定义模块 OTAClient
  LOCAL_CERTIFICATE := platform                      → 平台签名
  BUILD_PREBUILT                                     → 预编译 APK
    ↓
[产品清单]
vendor/giec/device-giec.mk                         ← PRODUCT_PACKAGES += OTAClient
    ↓
[包含链]
ross.mk → product_mbox.mk → core_amlogic.mk → device-giec.mk
    ↓
[编译]
构建系统匹配 OTAClient 模块
→ 对 OTAClient.apk 用 platform key 重新签名
→ 拷贝到 out/target/product/ross/system/app/OTAClient/
    ↓
[OTA 运行时属性注入]
product_mbox.mk                                    ← 将 OTA URL、APP_ID、APP_SECRET 写入系统属性
    ↓
[设备运行时]
OTAClient APK 启动
→ 读取 persist.sys.otaurl.ext 获取服务器地址
→ 读取 vendor.build.app.id/.secret 进行身份认证
→ 请求 OTA 服务器检查更新
→ 下载升级包到 /sdcard/Download/ota_gateway_net.zip
→ 复制到 /data/ota_package/update.zip
→ 调用 mUpdateEngine 执行升级
    ↓
[系统设置入口]
TvSettings 的 AboutFragment.java                    ← GMS 冲突修复
→ 强制将"系统更新"按钮指向 com.giec.otaclient
```

#### 关键对比: OTAClient vs libstbcmdservicehal

| 维度 | OTAClient | libstbcmdservicehal |
|------|-----------|-------------------|
| 类型 | APK (预编译) | .so (源码编译) |
| 构建方式 | `BUILD_PREBUILT` | `cc_library_shared` |
| 签名 | `platform` key | 无需签名 |
| 安装位置 | `system/app/` | `vendor/lib64/` |
| 开发语言 | Java (预编译) | C |
| 功能 | OTA 升级 UI + 逻辑 | shell 命令执行桥梁 |

### 练习 3: 自己写一个 Android.bp

尝试写一个简单的 cc_binary:

```blueprint
cc_binary {
    name: "hello_android",
    srcs: ["hello.c"],
    cflags: ["-Wall"],
    vendor: true,
}
```

把它放到某个目录下，在对应 device.mk 的 PRODUCT_PACKAGES 中加上 `hello_android`，然后编译验证。

---

## 10. 学习要点速查

- [x] Android.bp 的基本结构 (模块类型 + 属性块)
- [ ] Android.mk 的基本结构 (LOCAL_变量 + BUILD_类型)
- [ ] `name` 是模块的唯一标识，跨文件引用
- [ ] `PRODUCT_PACKAGES` 决定哪些模块被编译进镜像
- [ ] `LOCAL_CERTIFICATE := platform` 给 APK 系统权限
- [ ] `vendor: true` 控制安装分区
- [ ] `cc_defaults` 用于共享公共编译参数
- [ ] `shared_libs` / `LOCAL_SHARED_LIBRARIES` 管理依赖
- [ ] `defaults` 引用公共参数模板
- [ ] `PRODUCT_COPY_FILES` 直接拷贝文件到镜像
- [ ] 镜像内容三来源: 编译模块 + 文件拷贝 + 镜像配置
- [ ] super 动态分区包含多个逻辑卷 (system/vendor/product/...)
- [ ] 物理分区 (boot/bootloader/vbmeta) 和动态分区 (super 内) 的区别

---

## 11. 下一步

构建系统理解后，建议进入 Phase 2: **厂商定制层**，从 vendor/giec/ 的完整目录结构开始，看厂商在一个 AOSP 项目上到底加了哪些东西。

或者也可以先横向对比练习：在参考代码库中找 3 个不同的模块（APK、so 库、可执行文件），分别用 bp 和 mk 的方式追踪它们的完整构建链路。
