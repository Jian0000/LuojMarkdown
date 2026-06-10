+++
title = '预置 APK 分析'
date = '2026-06-09T17:00:00+08:00'
draft = false
+++

# 预置 APK 分析

> 分析 `vendor/giec/apps/` 下所有预置 APK 的构建方式、权限模型和功能架构
> 参考代码库：`vendor/giec/apps/`，Android 14 + Amlogic S905X5M ATV

---

## 1. 总览

`device-giec.mk` 中声明的预置 APK：

```makefile
PRODUCT_PACKAGES += \
    LeanKeyboard \       # ATV 定制键盘
    OTAClient \          # OTA 升级客户端
    BazeportLauncher \   # 定制 Launcher（桌面）
    BazeportSystem \     # 系统设置/工具
    luojHello            # 测试程序（C 程序，非 APK）
```

加上条件编译的：
```makefile
# 当 NEED_GLAUNCHER=true 时
PRODUCT_PACKAGES += Glauncher

# 当 NEED_FACTORY_TEST=true 时
PRODUCT_PACKAGES += STB_TEST
```

**所有 APK 的公共特征：**
- 都是**预编译的（prebuilt）** `.apk` 文件，不带 Java 源码
- 全部使用 `LOCAL_CERTIFICATE := platform` —— 平台密钥签名，获得 system 级 UID
- 通过 `Android.mk` + `$(BUILD_PREBUILT)` 集成到构建系统
- 安装路径取决于 `LOCAL_PRIVILEGED_MODULE` 和 `LOCAL_SYSTEM_EXT_MODULE`

---

## 2. Android 预置 APK 的构建模式

> **APK 预置**：Android 构建系统中，厂商可以将预编译的 `.apk` 文件直接打包进系统镜像，而不需要从源码编译。这是厂商定制的标准做法，尤其当 APK 是闭源或由第三方提供时。

构建系统中有两种 APK 集成方式：
- **`$(BUILD_PREBUILT)`** — 直接使用预编译的 APK，重新签名
- **`$(BUILD_PACKAGE)`** — 从 Java/Kotlin 源码编译成 APK

本参考代码库中全是 prebuilt 方式。

### 2.1 简单预置模式

用于 OTAClient、LeanKeyboard、luojHello 等：

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

# 自动取当前目录下唯一的 .apk 文件名作为模块名
APPS := $(notdir $(wildcard $(LOCAL_PATH)/*.apk))
APP_NAME := $(basename $(APPS))

LOCAL_MODULE := $(APP_NAME)
LOCAL_SRC_FILES := $(APPS)
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := platform          # ★ 使用平台密钥签名
LOCAL_PRIVILEGED_MODULE := false

include $(BUILD_PREBUILT)
```

### 2.2 复杂预置模式（带权限配置）

用于 BazeportLauncher：

```makefile
LOCAL_PRIVILEGED_MODULE := true        # ★ 特权应用
LOCAL_SYSTEM_EXT_MODULE := true        # ★ 装在 system_ext 分区
LOCAL_PREBUILT_JNI_LIBS := ...         # ★ 携带 Native 库

LOCAL_REQUIRED_MODULES += \            # ★ 需要额外配置文件
    privapp-permissions-bazeport-next.xml \
    default-permissions-bazeport-next.xml
```

### 2.3 `LOCAL_CERTIFICATE` 字段含义

> **Android 签名机制**：Android 要求每个 APK 都必须用证书签名。系统在安装 APK 时会验证签名。对于系统预置的应用，构建系统可以在打包时重新签名。

| 值 | 含义 | 效果 |
|-----|------|------|
| `platform` | 用平台的 platform 密钥 | APK 获得 `android.uid.system` UID，可申请系统级权限 |
| `media` | 用 media 密钥 | 获得 `android.uid.media` UID，可访问媒体资源 |
| `PRESIGNED` | 保留 APK 原有签名 | 不修改原签名，适用于有独立签名要求的应用 |

---

## 3. OTAClient —— OTA 升级客户端

### 3.1 基本信息

| 条目 | 值 |
|------|-----|
| 包名 | `com.giec.otaclient` |
| 版本 | 1.0.6 |
| 技术栈 | Kotlin + Jetpack Compose |
| 目标 SDK | 34 (Android 14) |
| minSdk | 30 (Android 11) |
| APK 大小 | 43MB |
| DEX 数量 | 3 个 |

### 3.2 权限清单

| 权限 | 用途 |
|------|------|
| `RECEIVE_BOOT_COMPLETED` | 开机启动检查更新 |
| `REBOOT` | 升级完成后重启系统 |
| `RECOVERY` | 进入 Recovery 模式刷机 |
| `MOUNT_UNMOUNT_FILESYSTEMS` | 挂载系统分区写入升级包 |
| `MANAGE_EXTERNAL_STORAGE` | 读写 `/sdcard/` 下载升级包 |
| `FOREGROUND_SERVICE_DATA_SYNC` | 前台下载服务 |
| `READ_PRIVILEGED_PHONE_STATE` | 获取设备标识 |
| `SYSTEM_ALERT_WINDOW` | 升级提醒弹窗 |

### 3.3 应用架构

从 DEX 中提取的完整包结构：

```
com.giec.otaclient/
├── OTAClientApplication          # Application 入口
├── receiver/
│   └── BootCompleteReceiver       # 开机广播 → 触发自动检查更新
├── service/
│   └── AutoCheckWorker            # 后台静默检查更新（WorkManager）
├── ui/
│   ├── activities/
│   │   ├── MainActivity           # 主界面/OTA 入口
│   │   ├── OnlineCheckActivity    # 在线检查更新
│   │   ├── OTADownloadActivity    # 下载升级包
│   │   ├── OTAUpdateActivity      # 执行系统升级
│   │   ├── FileSelectorActivity   # 选择本地升级包文件
│   │   └── RebootDialogActivity   # 重启确认弹窗
│   ├── views/                     # Compose UI 组件
│   └── theme/                     # 主题
├── viewmodel/
│   ├── DialogViewModel            # 弹窗状态管理
│   └── FileSelectorViewModel      # 文件选择状态管理
├── data/
│   └── FileItemData               # 文件条目数据模型
└── util/
    ├── NetworkRequestUtils        # 网络请求封装
    ├── OkHttpUtils                # OkHttp 客户端
    ├── ResponseUtils              # 响应解析
    ├── UpdateParserUtils          # 升级包信息解析
    └── Md5Utils                   # MD5 校验
```

### 3.4 OTA 升级流程

根据 README 和代码结构，完整 OTA 流程如下：

```
开机
  │
  ▼
BootCompleteReceiver          ← 监听 BOOT_COMPLETED 广播
  │
  ▼
AutoCheckWorker               ← WorkManager 后台任务，静默请求 OTA 服务器
  │
  ├─ 无更新 → 静默结束
  │
  ▼ (有更新)
OnlineCheckActivity            ← 展示新版本信息
  │
  ▼ (用户确认)
OTADownloadActivity            ← 从服务器下载升级包
  │                              → /sdcard/Download/ota_gateway_net.zip
  ▼
OTAUpdateActivity              ← 复制到系统更新目录
  │                              → /data/ota_package/update.zip
  ▼
UpdateManagerUtils             ← 调用 Android update_engine / RecoverySystem
  │
  ▼
RebootDialogActivity           ← 确认重启
  │
  ▼
系统重启进入 Recovery → 应用升级
```

### 3.5 OTA 服务器配置

通过系统属性配置（在 `device-giec.mk` 或 `device.mk` 中设置）：

```makefile
# ohm 设备
persist.sys.otaurl.ext=http://192.168.140.41:48080/app-api/ota/device/check
vendor.build.app.id=KhFacd
vendor.build.app.secret=fXP1K3DFfY2HgDoB

# oppen 设备
persist.sys.otaurl.ext=http://192.168.140.41:48080/app-api/ota/device/check
vendor.build.app.id=g0XDey
vendor.build.app.secret=X8tWx1wg5WcXPTdM
```

### 3.6 三种升级模式

服务器下发 `updatePattern` 参数定义：

| 模式 | 值 | 行为 |
|------|-----|------|
| 立即升级 | 1 | 强制弹窗，显示下载进度条，可取消 |
| 确认升级 | 2 | 弹窗确认后下载，不可取消 |
| 静默升级 | 3 | 静默下载，用户无感知（未完成） |

---

## 4. BazeportLauncher —— 定制桌面

### 4.1 基本信息

| 条目 | 值 |
|------|-----|
| 包名 | `com.bazeport.next` |
| 版本 | 1.1.4 (versionCode 114) |
| 技术栈 | **Flutter**（基于 libflutter.so + libapp.so） |
| 目标 SDK | 33 (Android 13) |
| 安装位置 | `/system_ext/priv-app/BazeportLauncher/` |

**Flutter 的证据**：`lib/` 目录下包含 `libflutter.so`（7.6MB）和 `libapp.so`（14MB），这是典型的 Flutter 应用结构——`libflutter.so` 是 Flutter 引擎，`libapp.so` 是 Dart 代码编译产物。

### 4.2 权限分析 —— 86 个权限

BazeportLauncher 声明了 86 个权限，是参考代码库中权限最多的应用。按功能分组：

**系统管理权限：**
| 权限 | 作用 |
|------|------|
| `INSTALL_PACKAGES` | 安装/更新 APK |
| `DELETE_PACKAGES` | 卸载应用 |
| `REBOOT` / `SHUTDOWN` | 重启/关机 |
| `CLEAR_APP_CACHE` / `CLEAR_APP_USER_DATA` | 清理应用数据 |
| `FORCE_STOP_PACKAGES` | 强制停止应用 |
| `SET_TIME` / `SET_TIME_ZONE` | 设置时间 |

**硬件控制权限：**
| 权限 | 作用 |
|------|------|
| `HDMI_CEC` | HDMI-CEC 控制 |
| `INJECT_EVENTS` | 注入输入事件（模拟按键） |
| `CAPTURE_VIDEO_OUTPUT` / `CAPTURE_AUDIO_OUTPUT` | 屏幕/音频录制 |
| `HARDWARE_TEST` | 硬件测试 |
| `MANAGE_USB` | USB 设备管理 |

**网络权限：**
| 权限 | 作用 |
|------|------|
| `NETWORK_STACK` | 网络栈管理 |
| `TETHER_PRIVILEGED` | 网络共享 |
| `OVERRIDE_WIFI_CONFIG` | 覆盖 WiFi 配置 |
| `CHANGE_CONFIGURATION` | 修改系统配置 |

### 4.3 三类权限配置

> Android 10+ 引入了对系统特权应用（PrivApp）的权限管控机制，要求权限声明必须通过 XML 文件在白名单中明确列出，否则系统不会授予这些权限。

**第一类：privapp-permissions（特权白名单）**

`privapp-permissions-bazeport-next.xml` 放在 `/system_ext/etc/permissions/`，包含约 60 个系统级权限。系统在开机时加载此文件，授予 `com.bazeport.next` 对应的权限。

```xml
<privapp-permissions package="com.bazeport.next">
    <permission name="android.permission.INSTALL_PACKAGES"/>
    <permission name="android.permission.REBOOT"/>
    <!-- ... 约 60 个 -->
</privapp-permissions>
```

**第二类：default-permissions（运行时权限预授权）**

`default-permissions-bazeport-next.xml` 放在 `/system_ext/etc/default-permissions/`，用于预授权运行时权限，避免用户使用时弹窗：

```xml
<exception package="com.bazeport.next">
    <permission name="android.permission.ACCESS_FINE_LOCATION" fixed="true"/>
    <!-- fixed="true" → 用户不可撤销此权限 -->
</exception>
```

**第三类：AndroidManifest.xml 中的 uses-permission**

APK 自身的清单文件中声明的权限，与应用请求的权限对应。

**三种配置的关系：**
```
App 在 AndroidManifest.xml 声明需要什么权限
  │
  ▼
如果是普通权限 → 安装时自动授予
如果是危险权限 → 运行时请求（default-permissions 可预授权）
如果是系统级权限 → 必须在 privapp-permissions XML 中白名单
                    同时 APK 必须用 platform 密钥签名
```

### 4.4 Native 库分析

`lib/` 目录下携带了大量原生库：

| 库 | 大小 | 用途 |
|----|------|------|
| `libflutter.so` | 7.6 MB | Flutter 引擎核心 |
| `libapp.so` | 14.0 MB | Flutter 应用代码（Dart 编译产物） |
| `librive_text.so` | 40.9 MB | Rivet 文本渲染引擎（动画文本） |
| `libpdfium.so` | 3.1 MB | PDF 渲染引擎 |
| `libpdfrx.so` | 1.4 MB | PDF 读取扩展 |
| `libffmpegJNI.so` | 828 KB | FFmpeg 多媒体解码 JNI 桥接 |
| `libexosmlext.so` | 166 KB | ExoPlayer 扩展 |
| `libpimdec-android.so` | 29 KB | PIM 解码 |

> Flutter 应用通过 `libapp.so` 将 Dart 代码编译为原生代码，因此 APK 中没有 DEX 文件中的应用逻辑。这种方式的好处是启动速度快、性能接近原生。

### 4.5 启动声明

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.HOME" />          <!-- 桌面 -->
    <category android:name="android.intent.category.LEANBACK_LAUNCHER" />  <!-- ATV -->
</intent-filter>
```

同时声明了 `HOME`（系统桌面）和 `LEANBACK_LAUNCHER`（ATV 启动器），意味着它会替换 Android TV 的原生桌面。

---

## 5. BazeportSystem —— 系统设置工具

### 5.1 基本信息

| 条目 | 值 |
|------|-----|
| 包名 | `com.bazeport.system` |
| 版本 | 1.4.0 (versionCode 140) |
| APK 大小 | 40MB |

从 AndroidManifest 中可以看到它声明了与 BazeportLauncher 类似的大量系统权限，但少了很多（没有 INSTALL_PACKAGES、REBOOT 等），定位是辅助工具。

### 5.2 安装配置

```makefile
LOCAL_CERTIFICATE := platform
# 没有 LOCAL_PRIVILEGED_MODULE ← 不是特权应用
# 没有 LOCAL_REQUIRED_MODULES ← 不需要权限 XML
```

与 BazeportLauncher 不同，BazeportSystem 没有声明为特权应用（`PRIVILEGED_MODULE` 默认为 false），也不需要单独的权限 XML。说明它需要的权限相对温和。

---

## 6. Glauncher —— 可选的定制桌面

### 6.1 基本信息

| 条目 | 值 |
|------|-----|
| 包名 | `cn.giec.glauncher` |
| 版本 | 1.1.5 |
| 目标 SDK | 34 |
| minSdk | 21 |

### 6.2 条件编译

```makefile
# device-giec.mk 中
ifeq ($(NEED_GLAUNCHER), true)
    PRODUCT_PACKAGES += Glauncher
endif
```

通过环境变量控制是否编译，与 BazeportLauncher 是**二选一**的关系。

### 6.3 与 BazeportLauncher 对比

| 维度 | BazeportLauncher | Glauncher |
|------|-----------------|-----------|
| 技术栈 | Flutter | 传统 Android |
| 包名 | `com.bazeport.next` | `cn.giec.glauncher` |
| APK 大小 | 7.6 MB | 29 MB |
| 权限数量 | 86 个 | 13 个 |
| 默认启用 | 是 | 否（需 `NEED_GLAUNCHER=true`） |
| 安装位置 | system_ext/priv-app | system/priv-app |

Glauncher 权限少得多，只有基础的位置、网络、存储和 WiFi 权限，没有系统管理特权。两套 Launcher 可能是针对不同的客户需求。

---

## 7. LeanKeyboard —— ATV 定制键盘

### 7.1 基本信息

| 条目 | 值 |
|------|-----|
| 包名 | `com.liskovsoft.leankeyboard` |
| 版本 | 6.1.13 |
| 来源 | [GitHub: yuliskov/LeanKeyboard](https://github.com/yuliskov/LeanKeyboard) |
| 签名 | 平台密钥（重签名） |

### 7.2 为什么需要它？

Android TV 默认的键盘在遥控器操作下体验很差。LeanKeyboard 专为电视优化：
- 大字体、大按键间距
- 全遥控器支持（红外/蓝牙）
- 无 Google 服务依赖（在国产 Android TV 上至关重要，因为多数 ATV 设备不带 Google 服务）
- 多语言支持

### 7.3 系统配置要求

它声明了 `BIND_INPUT_METHOD` 权限（系统级），这意味着它作为系统输入法被注册。要使其生效，还需要在 Framework 配置中启用它（通常在 `vendor/overlay/` 中配置）：

```xml
<!-- 系统设置中启用 LeanKeyboard 作为默认输入法 -->
<string-array name="config_input_methods">
    <item>com.liskovsoft.leankeyboard/com.liskovsoft.leankeyboard.LeanKeyKeyboard</item>
</string-array>
```

---

## 8. STB-TEST —— 产线测试工具

### 8.1 基本信息

| 条目 | 值 |
|------|-----|
| 包名 | `com.giec.stb.test` |
| 版本 | 1.4.8 (versionCode 20251023) |
| 签名 | 平台密钥 |

### 8.2 功能

产线测试工具，通过 U 盘加载配置文件进行自动测试：
1. U 盘根目录放入 `giec_config.配置说明.txt`
2. 插入待测设备
3. 按提示操作执行各项硬件测试
4. 全部通过后标记为"已测试"

条件编译控制：
```makefile
ifeq ($(NEED_FACTORY_TEST), true)
    PRODUCT_PACKAGES += STB_TEST
endif
```

---

## 9. luojHello —— C 测试程序

### 9.1 这不是 APK

虽然 `device-giec.mk` 的 `PRODUCT_PACKAGES` 列表中包含了 `luojHello`，但它是一个 C 程序，不是 APK：

```c
#include <stdio.h>
int main(void) {
    printf("Hello Android\n");
    return 0;
}
```

```bp
cc_binary {
    name: "luojHello",
    srcs: ["luojHello.c"],
    cflags: ["-Wall"],
    vendor: true,       # 安装在 vendor 分区
}
```

编译产物是 `/vendor/bin/luojHello` 可执行文件，可以在 adb shell 中直接运行。

### 9.2 学习价值

这是一个绝佳的**最小验证工具**——修改后编译进系统，确认构建流程正常工作。适合用作：
- 验证 vendor 分区挂载是否正确
- 测试 `PRODUCT_PACKAGES` 机制
- 确认自定义模块能成功编译和打包

---

## 10. 签名与安全总结

### 10.1 Android 四组签名密钥

> Android 系统定义了四组标准签名密钥，每组对应不同的 UID 和权限等级。系统在安装 APK 时，根据签名密钥决定授予什么级别的权限。

| 密钥文件 | 对应 UID | 典型用途 |
|---------|----------|---------|
| `testkey.pk8/.x509.pem` | 应用自有 UID | 开发者调试 |
| `platform.pk8/.x509.pem` | `android.uid.system` | 系统应用 |
| `shared.pk8/.x509.pem` | `android.uid.shared` | 共享 UID 的应用 |
| `media.pk8/.x509.pem` | `android.uid.media` | 媒体相关应用 |

密钥文件位于 `build/make/target/product/security/`（testkey）或厂商自己的安全目录。

### 10.2 本参考代码库的签名策略

```makefile
# device-giec.mk
ifeq ($(USE_RELEASE_KEY),true)
    PRODUCT_DEFAULT_DEV_CERTIFICATE := $(CERTIFICATE_DIR)/releasekey
else
    # 默认使用 AOSP 测试密钥
    PRODUCT_DEFAULT_DEV_CERTIFICATE := build/make/target/product/security/testkey
endif
```

所有预置 APK 的 `LOCAL_CERTIFICATE := platform` 意味着它们都会使用 `PRODUCT_DEFAULT_DEV_CERTIFICATE` 目录下的 `platform.pk8` 签名。

### 10.3 安全分层

```
system_ext/priv-app/       ← privapp 特权应用（BazeportLauncher）
  ├── BazeportLauncher.apk (platform 签名)
  ├── etc/permissions/privapp-permissions-*.xml    ← 特权白名单
  └── etc/default-permissions/default-permissions-*.xml  ← 权限预授权

system/priv-app/           ← 普通特权应用（Glauncher）
  └── Glauncher.apk (platform 签名)

system/app/                ← 系统应用（OTAClient, LeanKeyboard, ...）
  ├── OTAClient.apk (platform 签名)
  ├── LeanKeyboard.apk (platform 签名)
  └── ...

vendor/bin/                ← 原生可执行文件
  └── luojHello
```

---

## 11. 学习要点

### 11.1 预置 APK 的核心问题

1. **为什么都用 platform 签名？** 因为厂商应用需要访问系统 API（如 `REBOOT`、`INSTALL_PACKAGES`），只有 platform 签名的 APK 才能获得这些权限。

2. **为什么 BazeportLauncher 需要单独的权限 XML（而 OTAClient 不需要）？** 在 Android 10+ 上，只有枚举在 `privapp-permissions-<package>.xml` 中的系统级权限才会被授予。BazeportLauncher 的权限涉及系统核心操作（安装包、重启），而这些权限对系统安全性影响重大，Android 要求显式白名单。

3. **为什么 Flutter 应用需要额外的权限 xml？** 这与技术栈无关，与权限等级有关。只要应用声明了"系统级"（protectionLevel=signature|privileged）的权限，就必须有白名单。

### 11.2 从代码库中学到的模式

| 场景 | 做法 |
|------|------|
| 需要系统级权限的应用 | `LOCAL_CERTIFICATE := platform` + privapp-permissions XML |
| 需要预授权运行时权限 | default-permissions XML |
| 需要 native 库的 APK | `LOCAL_PREBUILT_JNI_LIBS` 自动扫描 |
| 修改系统桌面 | 声明 `HOME` + `LEANBACK_LAUNCHER` 类别 |
| 开机自启服务 | `BootCompleteReceiver` 监听 `BOOT_COMPLETED` |
| 按需编译 | `ifeq($(VAR),true)` 条件包含 |
| 移除预置应用 | `remove_unused_module` + `LOCAL_OVERRIDES_PACKAGES` |

### 11.3 深入阅读

- 分析 OTA 完整流程：`bootable/recovery/` + `frameworks/base/core/java/android/os/RecoverySystem.java`
- 理解权限系统：`frameworks/base/services/core/java/com/android/server/pm/permission/`
- Flutter 应用集成 AOSP：搜索 Flutter engine 的 AOSP 构建方式
