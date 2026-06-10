+++
title = 'Level 3：加一个 native 服务开机自启动'
date = '2026-06-09T16:04:00+08:00'
draft = false
+++

# Level 3：加一个 native 服务开机自启动

## 我想做

在系统里加一个自己的可执行文件，开机自动跑起来，`logcat` 能看到它的输出。

> 为什么要先做这个再做驱动？因为它最快让你体验完整的"加代码→配构建→配 init.rc→写 SELinux policy→编译→烧录→验证"闭环。不到你手写驱动的时间的 1/10，但建立的操作习惯后面每个 Level 都在用。

## 先回答三个问题

1. **Android 系统启动后，用户态的入口是什么？**
   - init 进程是所有用户态进程的祖先（PID 1）
   - init 解析 `init.rc` 文件，按规则启动服务
   - 厂商自己的服务可以通过 `init_rc` 属性添加到 vendor 分区

2. **vendor 分区和 system 分区是什么关系？**
   - system 分区存 Android 通用组件
   - vendor 分区存厂商（Amlogic/你的公司）的专有部分
   - Treble 架构强制隔离二者——你的服务应该放 vendor 分区

3. **为什么加了服务还要写 SELinux policy？**
   - Android 的 SELinux 默认不让任何未定义的进程运行
   - 不加 .te 文件，你的服务一启动就被 kernel 杀掉
   - 不是 bug，是机制

## 需要懂的知识

**Android.bp（Soong 构建）**

AOSP 从 Android 7 开始引入 Soong 替代 Makefile，配置文件是 `Android.bp`。常用模块类型：
```bp
cc_binary {           // 编译为可执行文件
    name: "luojService",
    srcs: ["luojService.cpp"],
    shared_libs: [
        "liblog",
    ],
    vendor: true,      // 安装到 vendor 分区
}
```

**`vendor: true` 决定了产物的安装路径。**
- 源码在 `vendor/giec/apps/luojTest/luojHello.c`
- 编译产物在 `out/target/product/ross/vendor/bin/luojHello`
- 烧录后出现在板子的 `/vendor/bin/luojHello`

流程：`Android.bp` 中的 `vendor: true` → Soong 将产物输出到 `out/.../vendor/bin/` → 打包 vendor.img 时这个目录成为 `/vendor/` 分区 → 板子上就是 `/vendor/bin/luojHello`。

如果去掉 `vendor: true`，默认会安装到 `system/bin/`，但 Treble 架构下厂商模块应该放 vendor 分区。

**Android.mk 写法（项目中同样常见）**
```mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE := luojService
LOCAL_SRC_FILES := luojService.cpp
LOCAL_SHARED_LIBRARIES := liblog
LOCAL_INIT_RC := luojService.rc
LOCAL_VENDOR_MODULE := true

include $(BUILD_EXECUTABLE)
```
这里 `LOCAL_VENDOR_MODULE := true` 等同于 Android.bp 的 `vendor: true`。

**init.rc 语法**

init.rc 是 init 进程的配置文件：
```rc
service luojService /vendor/bin/luojService
    class late_start
    user root
    group root
    disabled

on property:sys.boot_completed=1
    start luojService
```

核心概念：
- `service` ——定义如何启动一个进程
- `class late_start` ——late_start 类服务在系统基本启动完成后拉起
- `on property:X=Y` ——当系统属性 X 等于 Y 时触发操作

**SELinux policy 最小模板**

每个新增的服务都需要三个声明：
```te
type luojService, domain;
type luojService_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(luojService)
```

`allow` 语句的格式：`allow source_type target_type:class { permission };`

> **什么时候需要 .te 文件？**
> 只有当进程以独立 domain 身份后台自启动（通过 init.rc）时才需要配 SELinux policy。
> 如果是 `adb shell` 手动执行，进程继承 shell domain —— `userdebug`/`eng` 版本中 shell 域权限宽松。

> **file_contexts —— 第三步不可少**
> .te 文件只定义了 domain 和 exec_type 的标签名，但没告诉 SELinux "哪个文件路径对应哪个标签"。
>
> | 环节 | 在哪配置 | 作用 |
> |------|---------|------|
> | 定义 domain 和 exec_type | `.te` 文件 | 声明这两个标签存在 |
> | 声明 init 允许执行 | `init_daemon_domain(domain)` | 允许 init 执行对应 exec_type 并转到新 domain |
> | 路径→标签映射 | `file_contexts` | `/vendor/bin/xxx → xxx_exec` |
>
> 检查方法：`adb shell ls -Z /vendor/bin/xxx`，看标签是否为自己定义的 exec_type。

## 动手方案

```bash
# 第 1 步：创建模块目录
mkdir -p vendor/giec/apps/luojService/

# 第 2 步：写源码
cat > vendor/giec/apps/luojService/luojService.cpp << 'EOF'
#include <android/log.h>
#include <unistd.h>

#define LOG_TAG "luojService"

int main() {
    __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "luojService started");
    int count = 0;
    while (1) {
        __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "heartbeat #%d", count++);
        sleep(5);
    }
    return 0;
}
EOF

# 第 3 步：写 Android.bp
cat > vendor/giec/apps/luojService/Android.bp << 'EOF'
cc_binary {
    name: "luojService",
    srcs: ["luojService.cpp"],
    shared_libs: [
        "liblog",
    ],
    init_rc: ["luojService.rc"],
    vendor: true,
}
EOF

# 第 4 步：写 init.rc
cat > vendor/giec/apps/luojService/luojService.rc << 'EOF'
service luojService /vendor/bin/luojService
    class late_start
    user root
    group root
    disabled

on property:sys.boot_completed=1
    start luojService
EOF

# 第 5 步：写 SELinux policy
cat > vendor/giec/common/sepolicy/luojService.te << 'EOF'
type luojService, domain;
type luojService_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(luojService)
EOF

# 第 6 步：加 file_contexts
# 编辑 vendor/giec/common/sepolicy/file_contexts，添加：
#   /vendor/bin/luojService    u:object_r:luojService_exec:s0

# 第 7 步：在 PRODUCT_PACKAGES 中注册
# 编辑 vendor/giec/device-giec.mk，在 PRODUCT_PACKAGES 中追加 luojService

# 第 8 步：编译烧录（参考 level-2）

# 第 10 步：验证
adb shell ps -A | grep luojService
adb shell logcat -s luojService
adb shell dmesg | grep avc | grep luojService
```

## 验收清单

| 项目 | 验证方式 |
|------|----------|
| 服务开机自启动 | `adb shell ps -A \| grep luojService` 能看到进程 |
| logcat 有输出 | `adb shell logcat -s luojService` 看到 heartbeat 日志 |
| 没有 SELinux 拒绝 | `adb shell dmesg \| grep avc \| grep luojService` 无输出 |
| 理解每层的作用 | 你能说清楚：Android.bp 做什么、init.rc 做什么、.te 做什么 |
| SELinux 标签正确 | `adb shell ls -Z /vendor/bin/luojService` 输出 `u:object_r:luojService_exec:s0` |

> 实操中遇到的问题记录在 [debug.md](debug.md#level-3加一个-native-服务开机自启动)
