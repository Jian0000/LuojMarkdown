+++
title = 'AndroidFrameworkConsole命令速查'
date = '2026-06-09T17:04:00+08:00'
draft = false
+++

# Android Framework Console 命令速查

> 适用场景：`adb shell` 下执行的 Android framework 层调试命令
> 覆盖工具：am / pm / dumpsys / settings / wm / input / service / cmd / appops / content / ime / monkey / svc / media

---

## 目录

- [am — Activity Manager](#1-am--activity-manager)
- [pm — Package Manager](#2-pm--package-manager)
- [dumpsys — 系统服务诊断](#3-dumpsys--系统服务诊断)
- [settings — 系统设置](#4-settings--系统设置)
- [wm — Window Manager](#5-wm--window-manager)
- [input — 输入事件模拟](#6-input--输入事件模拟)
- [service — Service 调用](#7-service--service-调用)
- [cmd — 服务命令接口](#8-cmd--服务命令接口)
- [appops — 应用权限管理](#9-appops--应用权限管理)
- [content — ContentProvider 操作](#10-content--contentprovider-操作)
- [ime — 输入法管理](#11-ime--输入法管理)
- [monkey — UI 压力测试](#12-monkey--ui-压力测试)
- [svc — 网络/电源服务控制](#13-svc--网络电源服务控制)
- [media — 多媒体控制](#14-media--多媒体控制)
- [其他实用命令](#15-其他实用命令)

---

## 1. am — Activity Manager

> 管理 Activity、Service、Broadcast、Intent 的启动与调试。

### 1.1 启动 Activity

```bash
# 标准启动（指定完整 Action）
am start -a android.intent.action.VIEW -d http://www.google.com

# 启动指定包名/Activity
am start -n com.android.settings/.Settings
am start -n com.android.camera/.Camera

# 带 extra 参数
am start -n com.android.settings/.Settings \
  -e :android:show_fragment wifi

# 启动并传递 String 类型的 extra
am start -n com.example/.MainActivity \
  -e key1 value1 -e key2 value2

# 启动并传递 int/boolean/float 类型的 extra
am start -n com.example/.MainActivity \
  --ei int_key 123 \
  --ez bool_key true \
  --ef float_key 1.5

# 以 FLAG_ACTIVITY_NEW_TASK 方式启动
am start -n com.example/.MainActivity -f 0x10000000

# 指定 action 和 category
am start -a android.intent.action.MAIN \
  -c android.intent.category.HOME

# 启动 Activity 并等待结果（用于测试）
am start -W -n com.android.settings/.Settings
# 输出：WaitTime / CallTime / TotalTime
```

**常用参数：**
| 参数 | 说明 |
|------|------|
| `-a <ACTION>` | Intent Action |
| `-d <DATA_URI>` | Intent Data URI |
| `-n <COMPONENT>` | 完整组件名（package/activity） |
| `-t <MIME_TYPE>` | MIME 类型 |
| `-e <KEY> <VALUE>` | String Extra |
| `--ei <KEY> <VALUE>` | int Extra |
| `--ez <KEY> <VALUE>` | boolean Extra |
| `--ef <KEY> <VALUE>` | float Extra |
| `-f <FLAGS>` | Intent Flags（十六进制） |
| `-c <CATEGORY>` | Intent Category |
| `-W` | 等待启动完成并打印耗时 |

### 1.2 启动 Service

```bash
# 启动 Service
am startservice -n com.example/.MyService

# 带 extra 启动 Service
am startservice -n com.example/.MyService -e cmd play

# 绑定 Service 并发送命令
am startservice -a com.example.action.START \
  --es cmd "start_playback"

# Android 12+ 使用 foreground service 需声明
am start-foreground-service -n com.example/.MyService
```

### 1.3 发送广播

```bash
# 发送显式广播
am broadcast -n com.example/.MyReceiver

# 发送标准广播（如开机完成）
am broadcast -a android.intent.action.BOOT_COMPLETED

# 发送带权限的广播
am broadcast -a com.example.CUSTOM_BROADCAST \
  --es msg "hello" \
  --receiver-permission com.example.PERMISSION

# 发送有序广播
am broadcast -a com.example.ORDERED_BROADCAST --ordered
```

**常用标准 Action：**
| Action | 说明 |
|--------|------|
| `android.intent.action.BOOT_COMPLETED` | 开机完成 |
| `android.intent.action.SCREEN_OFF` | 屏幕关闭 |
| `android.intent.action.SCREEN_ON` | 屏幕点亮 |
| `android.intent.action.BATTERY_LOW` | 电量低 |
| `android.intent.action.BATTERY_OKAY` | 电量恢复 |
| `android.intent.action.TIME_SET` | 时间设置 |
| `android.intent.action.ACTION_POWER_CONNECTED` | 连接电源 |
| `android.intent.action.ACTION_POWER_DISCONNECTED` | 断开电源 |

### 1.4 进程管理

```bash
# 查看当前 Activity 栈
am stack list

# 查看所有可见 Activity
am monitor
# 每次有 Activity 启动时打印信息

# 强制停止应用
am force-stop com.example.app

# 杀死进程（不推荐，用 force-stop）
am kill com.example.app

# 杀死所有后台进程
am kill-all

# 设置应用为不活动（idle）
am idle

# 获取当前焦点 Activity
# 方式一：
dumpsys activity | grep mResumedActivity
# 方式二：
dumpsys window | grep mCurrentFocus

# 设置屏幕冻结/解冻状态
am screen-compat on/off <package>
```

---

## 2. pm — Package Manager

> 管理应用安装、卸载、权限、组件状态。

### 2.1 列出应用

```bash
# 列出所有已安装应用
pm list packages

# 仅列出第三方应用
pm list packages -3

# 仅列出系统应用
pm list packages -s

# 按关键字搜索
pm list packages | grep giec
pm list packages | grep google

# 显示 APK 路径
pm list packages -f
pm list packages -f | grep camera

# 显示安装来源
pm list packages -i

# 显示原生库路径
pm list packages -l

# 显示应用特征（是否为系统/第三方/已禁用等）
pm list packages --show-stored-sizes

# 显示已卸载但仍保留数据的应用
pm list packages -u

# 显示已禁用的应用
pm list packages -d

# 显示已启用的应用
pm list packages -e

# 仅显示有调试模式的应用
pm list packages --debug

# 显示应用的 ABI
pm list packages --abi
```

**常用过滤器：**
```bash
pm list packages [options] [filter]

# 按包名前缀过滤
pm list packages android.
pm list packages com.android.
pm list packages com.google.

# 按 UID 过滤
pm list packages --uid 1000

# 显示包名包含特定字符串
pm list packages -f | grep launcher
```

### 2.2 安装与卸载

```bash
# 安装 APK
pm install /data/local/tmp/app.apk
pm install -r /data/local/tmp/app.apk    # 覆盖安装（保留数据）
pm install -d /data/local/tmp/app.apk    # 降级安装（版本号更低时用）
pm install -g /data/local/tmp/app.apk    # 授予所有运行时权限
pm install -t /data/local/tmp/app.apk    # 允许安装测试 APK
pm install -i com.android.vending /data/local/tmp/app.apk  # 指定安装来源

# 多用户安装
pm install --user 10 /data/local/tmp/app.apk

# 卸载应用
pm uninstall com.example.app
pm uninstall -k com.example.app          # 卸载但保留数据/cache
pm uninstall --user 10 com.example.app   # 卸载指定用户的

# 清除应用数据
pm clear com.example.app
```

**`pm install` 常见失败原因：**
| 错误 | 原因 |
|------|------|
| `INSTALL_FAILED_ALREADY_EXISTS` | 已安装，需加 `-r` |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | 签名不一致，需完全卸载 |
| `INSTALL_FAILED_VERSION_DOWNGRADE` | 降级安装，需加 `-d` |
| `INSTALL_FAILED_INVALID_APK` | APK 损坏或不完整 |
| `INSTALL_FAILED_INSUFFICIENT_STORAGE` | 存储空间不足 |
| `INSTALL_FAILED_USER_RESTRICTED` | 用户限制（如多用户配置） |
| `INSTALL_FAILED_PERMISSION_MODEL_DOWNGRADE` | targetSdk 降级导致权限模型变化 |

### 2.3 查询应用信息

```bash
# 查看应用详细信息
pm dump com.example.app
pm dump com.example.app | grep -A 20 "permissions"
pm dump com.example.app | grep -A 10 "activities"

# 查看应用路径
pm path com.example.app
# 输出：package:/data/app/com.example.app-xxx/base.apk

# 查看应用版本
pm dump com.example.app | grep versionName

# 查看应用 targetSdk
pm dump com.example.app | grep targetSdk

# 查看应用 UID
pm list packages -U | grep com.example.app

# 查看应用的启动 Activity
pm resolve-activity --brief com.example.app
pm dump com.example.app | grep -A 3 "MainActivity"

# 查看应用使用的库
pm dump com.example.app | grep "shared libraries"
```

### 2.4 组件管理（启用/禁用）

```bash
# 禁用应用（用户不可见，无法启动）
pm disable com.example.app

# 启用应用
pm enable com.example.app

# 禁用 Activity
pm disable com.example.app/.SomeActivity

# 启用 Activity
pm enable com.example.app/.SomeActivity

# 隐藏应用（Launcher 中不可见，但仍可启动）
pm hide com.example.app

# 取消隐藏
pm unhide com.example.app

# 挂起应用（冻结应用进程）
pm suspend com.example.app

# 取消挂起
pm unsuspend com.example.app

# 设置应用为默认启动器
pm set-home-activity com.example.app/.MainActivity
```

### 2.5 权限管理

```bash
# 列出应用声明的权限
pm list permissions -d -g com.example.app

# 列出应用已授予的运行时权限
pm dump com.example.app | grep -A 200 "runtime permissions" | head -50

# 授予权限
pm grant com.example.app android.permission.CAMERA
pm grant com.example.app android.permission.ACCESS_FINE_LOCATION

# 撤销权限
pm revoke com.example.app android.permission.CAMERA

# 重置所有已授予的运行权限
pm reset-permissions com.example.app

# 列出所有 dangerous 权限
pm list permissions -d -g

# 列出所有分组
pm list permission-groups

# 根据权限分组列出
pm list permissions -g

# 检查权限是否已授予
pm check-permission android.permission.CAMERA com.example.app
```

### 2.6 多用户

```bash
# 创建新用户
pm create-user <user_name>

# 删除用户
pm remove-user <user_id>

# 列出所有用户
pm list users

# 为指定用户安装/卸载
pm install --user 10 /path/to/app.apk
pm uninstall --user 10 com.example.app

# 获取用户 ID
pm get-primary-storage-user-id
```

---

## 3. dumpsys — 系统服务诊断

> dumpsys 是 Android 最重要的诊断工具，用于查询系统服务的内部状态。

### 3.1 基础用法

```bash
# 列出所有可用服务
dumpsys -l

# 查询指定服务
dumpsys <service_name>
dumpsys activity
dumpsys window
dumpsys package

# 服务帮助信息（查看支持哪些参数）
dumpsys <service_name> -h
dumpsys activity -h
dumpsys package -h

# 输出到文件
dumpsys activity > /data/local/tmp/activity.txt
```

### 3.2 Activity 相关

```bash
# 当前焦点 Activity
dumpsys activity | grep -E "mResumedActivity|mFocusedActivity"

# 当前 Activity 栈
dumpsys activity activities

# 最近 Activity 历史
dumpsys activity recents

# 所有正在运行的进程
dumpsys activity processes

# 查看进程中的 Service
dumpsys activity services

# 查看 BroadcastReceiver 状态
dumpsys activity broadcasts

# 查看 Provider 状态
dumpsys activity providers

# 查看 oom_adj（重要：判断进程是否被杀）
dumpsys activity processes | grep com.example.app
```

**oom_adj 值含义：**
| 值 | 含义 |
|----|------|
| -17 | 系统核心进程（永远不会被杀） |
| 0~100 | 前台进程 |
| 100~200 | 可见进程 |
| 200~300 | 服务进程 |
| 300~900 | 后台进程（越靠后越先被杀） |
| 900+ | 空进程 |

### 3.3 Package / 包管理

```bash
# 查看包基本信息
dumpsys package com.example.app

# 查看包中所有 Activity
dumpsys package com.example.app | grep -A 5 "Activity"

# 查看包中注册的 Service
dumpsys package com.example.app | grep -A 5 "Service"

# 查看包中注册的 Receiver
dumpsys package com.example.app | grep -A 5 "Receiver"

# 查看包权限
dumpsys package com.example.app | grep -A 50 "runtime permissions"

# 查看包安装信息
dumpsys package com.example.app | grep -E "version|install|firstInstall|lastUpdate"

# 查看包的签名信息
dumpsys package com.example.app | grep -A 10 "Signatures"

# 系统中所有包的信息汇总（通常输出很大）
dumpsys package packages
```

### 3.4 Window 显示相关

```bash
# 当前窗口焦点（最常用）
dumpsys window | grep mCurrentFocus

# 当前窗口栈
dumpsys window windows

# 显示信息（分辨率、密度、刷新率）
dumpsys window displays

# 查看输入法窗口
dumpsys window | grep InputMethod

# 查看 Surface 信息
dumpsys window surfaces

# 查看动画状态
dumpsys window animations

# 查看屏幕旋转状态
dumpsys window | grep rotation
```

### 3.5 内存相关

```bash
# 系统内存概览
dumpsys meminfo

# 查看进程内存详情
dumpsys meminfo <pid>
dumpsys meminfo com.example.app

# 查看 GPU 内存使用
dumpsys gfxinfo com.example.app

# 查看 Graphics 内存
dumpsys graphicsstats

# 查看数据库内存
dumpsys dbinfo com.example.app
```

### 3.6 其他常用服务

```bash
# 电池状态
dumpsys battery
dumpsys battery set level 50           # 模拟电量 50%
dumpsys battery set status 1           # 模拟电池状态（1=未知, 2=充电, 3=放电, 4=未充, 5=满）
dumpsys battery unplug                 # 模拟断开充电器
dumpsys battery reset                  # 重置电池模拟

# 电源管理
dumpsys power
dumpsys power | grep "Wake Locks"     # 查看 wakelock

# 网络
dumpsys wifi
dumpsys connectivity
dumpsys netstats
dumpsys ethernet

# 蓝牙
dumpsys bluetooth_manager

# 显示
dumpsys display

# 通知
dumpsys notification

# 设置（自定义设置服务）
dumpsys settings

# 磁盘使用
dumpsys diskstats

# CPU 信息
dumpsys cpuinfo

# 输入法
dumpsys input_method

# 位置
dumpsys location

# 媒体
dumpsys media_session
dumpsys audio

# 传感器
dumpsys sensorservice

# 壁纸
dumpsys wallpaper

# 用户管理
dumpsys user
```

### 3.7 dumpsys 服务速查表

| 服务名 | 关键用途 |
|--------|----------|
| `activity` | Activity 栈、进程状态、oom_adj、Service |
| `window` | 焦点窗口、显示参数、窗口层级 |
| `package` | 包信息、权限、签名、组件注册 |
| `meminfo` | 内存占用（PSS/RSS/USS） |
| `power` | wakelock、屏幕待机状态 |
| `battery` | 电量、温度、充电状态 |
| `batterystats` | 电量历史统计（需先 `dumpsys batterystats --reset`） |
| `wifi` | Wi-Fi 连接信息 |
| `connectivity` | 网络连接状态 |
| `netstats` | 网络流量统计 |
| `bluetooth_manager` | 蓝牙状态 |
| `display` | 显示设备参数 |
| `notification` | 通知栏内容 |
| `media_session` | 媒体播放/暂停状态 |
| `diskstats` | 磁盘分区使用量 |
| `cpuinfo` | CPU 负载 |
| `input_method` | 输入法状态 |
| `gfxinfo` | GPU 渲染性能 |
| `sensorservice` | 传感器数据 |
| `user` | 多用户状态 |

---

## 4. settings — 系统设置

> 读写 Settings 命名空间（system/secure/global），修改系统配置。

### 4.1 读取设置

```bash
# 读取所有设置（三个命名空间）
settings list system
settings list secure
settings list global

# 读取指定 key
settings get system screen_brightness
settings get secure android_id
settings get global airplane_mode_on

# 读取并查看类型
settings get system screen_brightness  # 返回数值
settings get global device_name         # 返回字符串
```

### 4.2 写入设置

```bash
# 写入值
settings put system screen_brightness 200
settings put global airplane_mode_on 1
settings put secure lock_screen_show_notifications 1

# 写入字符串
settings put global device_name "My Android TV"

# 写入 float
settings put global window_animation_scale 0.5

# 删除 key（恢复默认）
settings delete system screen_brightness
```

### 4.3 常用设置项

```bash
# 窗口动画速度（0=关闭动画）
settings put global window_animation_scale 0
settings put global transition_animation_scale 0
settings put global animator_duration_scale 0

# 屏幕超时（毫秒）
settings put system screen_off_timeout 300000

# 屏幕亮度 0-255
settings put system screen_brightness 120

# 屏幕自动亮度
settings put system screen_brightness_mode 1    # 开启
settings put system screen_brightness_mode 0    # 关闭

# 字体大小
settings put system font_scale 1.0

# 时区
settings put global timezone Asia/Shanghai

# 日期格式
settings put system date_format yyyy-MM-dd

# 屏幕旋转
settings put global accelerometer_rotation 1    # 自动旋转
settings put global accelerometer_rotation 0    # 锁定

# 开发者选项
settings put global development_settings_enabled 1

# 显示屏幕刷新率
settings put global show_refresh_rate 1
settings put global show_fps_info 1

# 显示触摸点
settings put global show_touches 1

# 不保留 Activity（退出即销毁）
settings put global always_finish_activities 1

# 应用兼容模式（强制跑在 32 位）
settings put global development_settings_enabled 1  # 需先开启开发者选项
# 然后在开发者选项中设置
```

**三个命名空间的区别：**
| 命名空间 | 权限 | 多用户 | 用途 |
|----------|------|--------|------|
| `system` | 可写 | 每用户独立 | 用户个人偏好（亮度、音量、字体） |
| `secure` | 需 WRITE_SECURE_SETTINGS（系统级签名） | 每用户独立 | 安全相关（锁屏、定位） |
| `global` | 需 WRITE_SECURE_SETTINGS | 全局 | 系统级配置（飞行模式、时区） |

---

## 5. wm — Window Manager

> 查询和修改窗口管理器的参数（分辨率、密度、缩放）。

```bash
# 显示当前参数
wm size              # 显示分辨率（如 1920x1080）
wm density           # 显示密度 dpi（如 320）
wm overscan          # 显示过扫描区域

# 修改分辨率（用于模拟不同屏幕）
wm size 1280x720
wm size 1920x1080
wm size reset        # 恢复原始分辨率

# 修改密度
wm density 160
wm density 240
wm density reset     # 恢复原始密度

# 设置过扫描（移除边缘像素，解决 overscan 问题）
wm overscan 0,0,0,0      # reset
wm overscan 10,10,10,10  # 四边各收缩 10px

# 查看屏幕截图尺寸
wm size -w               # 宽度
wm size -h               # 高度
```

**使用场景：**
- 用 `wm size` 模拟手机/Pad/电视不同分辨率
- 用 `wm density` 测试 UI 在不同 dpi 下的布局
- `wm overscan` 对适配电视的过扫描裁剪很有用

---

## 6. input — 输入事件模拟

> 模拟触摸、按键、滚轮等输入事件。

### 6.1 按键事件

```bash
# 模拟按键（KEYCODE）
input keyevent KEYCODE_HOME
input keyevent KEYCODE_BACK
input keyevent KEYCODE_MENU
input keyevent KEYCODE_DPAD_UP
input keyevent KEYCODE_DPAD_DOWN
input keyevent KEYCODE_DPAD_LEFT
input keyevent KEYCODE_DPAD_RIGHT
input keyevent KEYCODE_DPAD_CENTER
input keyevent KEYCODE_ENTER
input keyevent KEYCODE_DEL
input keyevent KEYCODE_VOLUME_UP
input keyevent KEYCODE_VOLUME_DOWN
input keyevent KEYCODE_MUTE
input keyevent KEYCODE_POWER
input keyevent KEYCODE_CAMERA
input keyevent KEYCODE_APP_SWITCH

# 组合键模拟
input keyevent KEYCODE_WAKEUP           # 唤醒屏幕
input keyevent KEYCODE_SLEEP            # 睡眠

# 游戏/媒体按键
input keyevent KEYCODE_MEDIA_PLAY_PAUSE
input keyevent KEYCODE_MEDIA_STOP
input keyevent KEYCODE_MEDIA_NEXT
input keyevent KEYCODE_MEDIA_PREVIOUS

# 电视专用键
input keyevent KEYCODE_DPAD_UP          # 方向键上
input keyevent KEYCODE_DPAD_DOWN        # 方向键下
input keyevent KEYCODE_DPAD_LEFT        # 方向键左
input keyevent KEYCODE_DPAD_RIGHT       # 方向键右
input keyevent KEYCODE_DPAD_CENTER      # 确认
input keyevent KEYCODE_BACK             # 返回
input keyevent KEYCODE_HOME             # 主页
```

### 6.2 触摸事件

```bash
# 点击指定坐标
input tap x y
input tap 500 500
input tap 100 200

# 模拟滑动
input swipe x1 y1 x2 y2
input swipe 500 500 500 1000      # 从上往下滑
input swipe 500 1000 500 500      # 从下往上滑

# 带持续时间的滑动（毫秒）
input swipe 500 500 500 1000 2000  # 2 秒内从上滑到下

# 拖拽（长按 + 滑动）
input draganddrop x1 y1 x2 y2 duration_ms
input draganddrop 300 300 300 500 1000

# 长按（TextSelection 需要）
input swipe x y x y duration_ms
input swipe 500 500 500 500 2000   # 在 (500,500) 处长按 2 秒
```

### 6.3 文本输入

```bash
# 输入文本（需要输入框有焦点）
input text "Hello World"
input text "adb shell input test"
input text "你好世界"

# 输入特殊字符（需转义）
input text "password123!"
input text "user@example.com"

# 清空输入框再输入（模拟全选+删除+输入）
input keyevent KEYCODE_MOVE_END
input keyevent --longpress KEYCODE_DEL
input text "new text"
```

### 6.4 滚轮事件

```bash
# 模拟滚轮（电视遥控器/鼠标滚轮）
input roll dx dy
input roll 0 -10     # 向上滚动
input roll 0 10      # 向下滚动
input roll 10 0      # 向右滚动
```

### 常用 KEYCODE 速查

| KEYCODE | 值 | 说明 |
|---------|-----|------|
| `HOME` | 3 | Home 键 |
| `BACK` | 4 | 返回键 |
| `MENU` | 82 | 菜单键 |
| `DPAD_UP` | 19 | 方向键上 |
| `DPAD_DOWN` | 20 | 方向键下 |
| `DPAD_LEFT` | 21 | 方向键左 |
| `DPAD_RIGHT` | 22 | 方向键右 |
| `DPAD_CENTER` | 23 | 确认键 |
| `VOLUME_UP` | 24 | 音量+ |
| `VOLUME_DOWN` | 25 | 音量- |
| `POWER` | 26 | 电源键 |
| `ENTER` | 66 | 回车键 |
| `DEL` | 67 | 退格键 |
| `MUTE` | 91 | 静音 |
| `MEDIA_PLAY_PAUSE` | 85 | 播放/暂停 |
| `MEDIA_STOP` | 86 | 停止 |
| `APP_SWITCH` | 187 | 最近任务 |
| `WAKEUP` | 224 | 唤醒屏幕 |
| `SLEEP` | 223 | 睡眠 |

---

## 7. service — Service 调用

> 调用已注册的 Binder 服务的方法。

```bash
# 列出所有已注册的服务
service list

# 查看服务包含的方法
service check <service_name>

# 调用服务方法
service call <service_name> <code> [args...]

# 获取窗口服务信息（code 1 通常是 list）
service call window 1

# 获取 Activity 信息
service call activity 1

# 调用 IPowerManager（设置屏幕亮度）
service call power 12 i32 120

# 调用音量服务
service call audio 3 i32 3 i32 7 i32 0
# audio service code 3 = setStreamVolume
# i32 3 = STREAM_MUSIC, i32 7 = 音量值, i32 0 = flag
```

**参数类型说明：**
| 参数 | 说明 | 示例 |
|------|------|------|
| `i32 N` | 32 位整数 | `i32 123` |
| `s16 "str"` | 16 位字符串 | `s16 "hello"` |
| `null` | 空引用 | `null` |
| `f N` | Float | `f 1.5` |
| `d N` | Double | `d 3.14` |

**注意事项：**
- `service call` 需要知道服务方法的 code 编号，不方便使用
- 推荐用 `cmd` 命令（见下节）代替，更直观
- `service list` 显示的服务名可用于 `dumpsys` 直接查询

---

## 8. cmd — 服务命令接口

> Android 11+ 引入的服务命令接口，比 `service call` 更易用。

```bash
# 查看服务支持哪些命令
cmd <service_name> help
cmd role help
cmd wifi help
cmd notification help

# 角色管理（Android 10+）
cmd role list                               # 列出所有角色
cmd role add-role-holder <role> <package>   # 分配角色
cmd role remove-role-holder <role> <package> # 移除角色

# 角色相关操作示例
cmd role list role:android.role.HOME        # 查看当前默认 Launcher
cmd role add-role-holder role:android.role.HOME com.example.launcher

# 通知管理
cmd notification get_snooze_settings
cmd notification set_navigation_bar_color <color>

# WiFi 管理
cmd wifi set-wifi-enabled enabled
cmd wifi get-wifi-enabled

# overlay 管理
cmd overlay list
cmd overlay enable <package>
cmd overlay disable <package>

# sensor 管理
cmd sensorservice set-uid-state <uid> <state>

# alarm 管理
cmd alarm list
```

---

## 9. appops — 应用权限管理

> AppOps 是 Android 4.4+ 引入的细粒度权限操作管理框架，控制应用对敏感操作的访问。

```bash
# 查看应用的 AppOps 模式
appops get <package>
appops get com.example.app

# 列出所有 Op
appops list

# 设置某个 Op 的模式
# mode: allow / deny / ignore / foreground
appops set <package> <op> <mode>
appops set com.example.app CAMERA deny
appops set com.example.app LOCATION allow
appops set com.example.app POST_NOTIFICATIONS ignore

# 重置所有 AppOps
appops reset <package>
appops reset com.example.app

# 查询特定 op
appops query-op <op> <mode>
appops query-op CAMERA deny
appops query-op LOCATION allow
```

**常用 AppOps：**
| Op | 说明 |
|----|------|
| `CAMERA` | 相机访问 |
| `LOCATION` | 位置访问 |
| `RECORD_AUDIO` | 录音 |
| `READ_CONTACTS` | 读取联系人 |
| `POST_NOTIFICATIONS` | 发送通知（Android 13+） |
| `ACCESS_NOTIFICATIONS` | 访问通知 |
| `SYSTEM_ALERT_WINDOW` | 悬浮窗 |
| `WRITE_SETTINGS` | 修改系统设置 |
| `REQUEST_INSTALL_PACKAGES` | 安装 APK |

---

## 10. content — ContentProvider 操作

> 直接查询、插入、更新、删除 ContentProvider 的数据。

```bash
# 查询（类似 SQL SELECT）
content query --uri content://settings/secure
content query --uri content://settings/secure \
  --projection name:value
content query --uri content://settings/system \
  --where "name='screen_brightness'"

# 插入数据
content insert --uri content://settings/system \
  --bind name:s:screen_brightness \
  --bind value:i:128

# 更新数据
content update --uri content://settings/system \
  --bind value:i:200 \
  --where "name='screen_brightness'"

# 删除数据
content delete --uri content://settings/system \
  --where "name='screen_brightness'"

# 读取联系人
content query --uri content://contacts/people/

# 读取短信
content query --uri content://sms/inbox

# 读取日历
content query --uri content://calendar/calendars
```

**参数说明：**
| 参数 | 说明 |
|------|------|
| `--uri <URI>` | ContentProvider URI |
| `--projection col:col` | 指定返回列 |
| `--where "col='val'"` | 过滤条件 |
| `--bind col:type:val` | 绑定参数 |
| `--sort "col ASC/DESC"` | 排序 |

---

## 11. ime — 输入法管理

> 管理输入法（软键盘）的列表、切换和调试。

```bash
# 列出已安装的输入法
ime list
ime list -s                            # 仅显示系统输入法
ime list -a                            # 显示所有输入法（含第三方）

# 启用/禁用输入法
ime enable com.android.inputmethod.latin/.LatinIME
ime disable com.example.ime/.MyIME

# 设置默认输入法
ime set com.android.inputmethod.latin/.LatinIME

# 重置为默认输入法
ime reset

# 查看当前输入法
settings get secure default_input_method
```

---

## 12. monkey — UI 压力测试

> 生成伪随机用户事件流进行压力测试。

```bash
# 基本用法（向系统发送 500 个随机事件）
monkey 500

# 指定包名（只向指定应用发事件）
monkey -p com.example.app 500

# 指定多个包
monkey -p com.example.app -p com.android.settings 500

# 设置事件间隔（毫秒）
monkey -p com.example.app --throttle 200 500

# 调整事件类型比例（百分比）
monkey -p com.example.app \
  --pct-touch 50 \
  --pct-motion 20 \
  --pct-trackball 0 \
  --pct-nav 5 \
  --pct-majornav 5 \
  --pct-syskeys 0 \
  --pct-appswitch 10 \
  --pct-anyevent 10 \
  1000

# 记录日志
monkey -p com.example.app -v -v -v 1000 2>&1 | tee /data/local/tmp/monkey.log

# 跟踪 ANR 和崩溃
monkey -p com.example.app \
  --kill-after-error false \
  --ignore-crashes \
  --ignore-timeouts \
  --ignore-security-exceptions \
  --monitor-native-crashes \
  10000

# 指定种子值（使事件序列可重现）
monkey -p com.example.app -s 12345 500
```

**事件类型 `--pct-*` 说明：**
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--pct-touch` | 30% | 触摸事件（down/up） |
| `--pct-motion` | 15% | 滑动事件（down/move/up） |
| `--pct-trackball` | 10% | 轨迹球事件 |
| `--pct-nav` | 25% | D-pad 导航 |
| `--pct-majornav` | 15% | 主要导航（back、menu） |
| `--pct-syskeys` | 2% | 系统按键（Home、音量） |
| `--pct-appswitch` | 2% | Activity 切换 |
| `--pct-anyevent` | 1% | 其他事件 |

**常用选项：**
| 选项 | 说明 |
|------|------|
| `-v` | 日志详细级别（-v / -v -v / -v -v -v） |
| `-s <seed>` | 随机数种子（使用相同种子复现相同事件序列） |
| `--throttle <ms>` | 事件间延迟 |
| `--ignore-crashes` | 忽略应用崩溃继续测试 |
| `--ignore-timeouts` | 忽略 ANR 继续测试 |
| `--kill-after-error` | 错误后是否杀死应用 |
| `--monitor-native-crashes` | 监控 native crash |

---

## 13. svc — 网络/电源服务控制

> 控制 WiFi、移动数据、蓝牙和电源状态。

```bash
# WiFi 控制
svc wifi enable              # 打开 WiFi
svc wifi disable             # 关闭 WiFi

# 移动数据控制
svc data enable              # 打开移动数据
svc data disable             # 关闭移动数据

# 存储挂载
svc mount external           # 挂载外部存储
svc unmount external         # 卸载外部存储

# USB 控制
svc usb setState disconnect  # 断开 USB
svc usb setState connect     # 连接 USB

# Stay-on（插入电源时不休眠）
svc power stayon true        # 开启
svc power stayon false       # 关闭
svc power stayon usb         # USB 供电时保持唤醒
svc power stayon ac          # 交流电供电时保持唤醒
svc power stayon wireless    # 无线充电时保持唤醒
```

---

## 14. media — 多媒体控制

> 控制媒体播放服务。

```bash
# 查看媒体会话
media list

# 向指定媒体会话发送控制命令
media play <session-id>
media pause <session-id>
media stop <session-id>
media next <session-id>
media previous <session-id>

# 查看媒体会话详情
media output <session-id>

# 查看所有音量组
media volume --show
media volume --stream 3 --get          # 获取音乐音量
media volume --stream 3 --set 10       # 设置音乐音量为 10
media volume --stream 3 --adj raise    # 提高音量
media volume --stream 3 --adj lower    # 降低音量

# 音频路由
media routing --show
```

**音频流类型（stream）：**
| 编号 | 类型 | 说明 |
|------|------|------|
| 0 | STREAM_VOICE_CALL | 通话 |
| 1 | STREAM_SYSTEM | 系统声音 |
| 2 | STREAM_RING | 铃声 |
| 3 | STREAM_MUSIC | 音乐/媒体 |
| 4 | STREAM_ALARM | 闹钟 |
| 5 | STREAM_NOTIFICATION | 通知 |
| 6 | STREAM_BLUETOOTH_SCO | 蓝牙 SCO |
| 10 | STREAM_ACCESSIBILITY | 无障碍 |

---

## 15. 其他实用命令

### 15.1 系统信息

```bash
# 系统属性（最常用调试命令）
getprop                              # 列出所有属性
getprop ro.build.version.sdk         # API level
getprop ro.build.version.release     # Android 版本
getprop ro.product.name              # 产品名
getprop ro.product.cpu.abi           # CPU ABI
getprop ro.serialno                  # 序列号
getprop ro.boot.vbmeta.device        # vbmeta 设备
getprop persist.sys.timezone         # 时区
getprop ro.build.fingerprint         # 完整 build 指纹

# 设置系统属性
setprop debug.sf.nobootanimation 1   # 跳过开机动画
setprop persist.vendor.xxx.yyy val   # 持久化属性

# 查看 SELinux 状态
getenforce                           # Enforcing / Permissive
setenforce 0                         # 设为宽容模式
setenforce 1                         # 设为强制模式

# 获取设备 ID
settings get secure android_id

# 获取当前 UTC 时间
date
date -u
```

### 15.2 进程管理

```bash
# 查看进程
ps -A                                # 所有进程
ps -A | grep system_server           # 过滤特定进程
ps -A -o PID,NAME,%CPU,%MEM          # 自定义输出格式
ps -T -p <pid>                       # 查看进程中的线程

# 查看线程
ps -T -p <pid_of_system_server>

# top（实时监控）
top -d 1 -n 5                        # 每秒刷新，共 5 次
top -d 1 -m 10                       # 每秒刷新，显示前 10 行

# 杀死进程
kill <pid>
kill -9 <pid>                        # 强制杀死

# 调试跟踪
strace -p <pid>                      # 跟踪系统调用
strace -f -p <pid>                   # 跟踪子进程
strace -e openat -p <pid>            # 只跟踪文件打开

# 查看进程打开的文件
ls -l /proc/<pid>/fd/
```

### 15.3 文件系统操作

```bash
# 查看磁盘空间
df -h
df /data

# 查看目录大小
du -sh /data/app/*
du -sh /data/data/com.example.app

# 统计文件/目录数量
ls -la /data/data/ | wc -l

# 查看文件类型
file /system/bin/app_process
file /data/app/com.example.app/base.apk

# 计算文件校验
md5sum /data/local/tmp/app.apk
sha1sum /data/local/tmp/app.apk

# 查看系统链接库
ls -l /system/lib64/
ldd /system/bin/app_process64        # 查看动态链接
```

### 15.4 锁定与重置

```bash
# 锁屏（模拟锁屏状态）
input keyevent KEYCODE_SLEEP

# 解锁（如果无密码）
input keyevent KEYCODE_WAKEUP
input swipe 300 1000 300 300          # 滑动解锁

# 重置屏幕超时
settings put system screen_off_timeout 60000

# 删除锁屏密码
# 删除 /data/system 下密码相关文件
rm /data/system/gesture.key
rm /data/system/locksettings.db*
reboot
```

### 15.5 DEBUG 相关

```bash
# 生成 bugreport（包含所有诊断信息的压缩包）
bugreport
# 输出路径：/data/user_de/0/com.android.shell/files/bugreports/
# 或直接：
bugreportz > /sdcard/bugreport.zip

# 启用/禁用 adb 调试
setprop persist.service.adb.enable 1
setprop persist.service.adb.enable 0

# 跟踪 Binder 调用
# 需要开启 Binder 跟踪
echo 1 > /sys/kernel/debug/tracing/events/binder/binder_transaction/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

### 15.6 命令组合实战

```bash
# 一键获取应用的完整调试信息
PACKAGE=com.example.app
echo "===== 路径 ====="
pm path $PACKAGE
echo "===== 版本 ====="
pm dump $PACKAGE | grep version
echo "===== 权限 ====="
pm dump $PACKAGE | grep -A 10 "runtime permissions"
echo "===== Activity 栈 ====="
dumpsys activity | grep -E "mResumedActivity|mFocusedActivity"
echo "===== 内存 ====="
dumpsys meminfo $PACKAGE | head -20
echo "===== oom_adj ====="
dumpsys activity processes | grep $PACKAGE
echo "===== SELinux 拒绝 ====="
dmesg | grep "avc: denied" | grep $PACKAGE | tail -10
echo "===== 最近崩溃 ====="
logcat -b crash -d | tail -20

# 模拟遥控器按键测试电视应用
PACKAGE=com.example.tvapp
am start -n $PACKAGE/.MainActivity
sleep 2
# 向下导航几次
input keyevent KEYCODE_DPAD_DOWN
sleep 1
input keyevent KEYCODE_DPAD_DOWN
sleep 1
input keyevent KEYCODE_DPAD_CENTER
sleep 2
input keyevent KEYCODE_BACK
sleep 1

# 一键降低系统动画速度（流畅度测试）
settings put global window_animation_scale 0
settings put global transition_animation_scale 0
settings put global animator_duration_scale 0

# 快速清除应用数据并重启
pm clear com.example.app
am start -n com.example.app/.MainActivity
```

---

## 附录：命令索引

| 命令 | 主要用途 | 包位置 |
|------|----------|--------|
| `am` | Activity/Service/Broadcast 控制 | `/system/bin/am` |
| `pm` | 包管理、权限、安装卸载 | `/system/bin/pm` |
| `dumpsys` | 系统服务状态查询 | `/system/bin/dumpsys` |
| `settings` | 系统设置读写 | `/system/bin/settings` |
| `wm` | 窗口参数管理 | `/system/bin/wm` |
| `input` | 输入事件模拟 | `/system/bin/input` |
| `service` | Binder 服务调用 | `/system/bin/service` |
| `cmd` | 服务命令接口（替代 service call） | `/system/bin/cmd` |
| `appops` | 细粒度权限管理 | `/system/bin/appops` |
| `content` | ContentProvider 操作 | `/system/bin/content` |
| `ime` | 输入法管理 | `/system/bin/ime` |
| `monkey` | UI 压力测试 | `/system/bin/monkey` |
| `svc` | 网络/电源控制 | `/system/bin/svc` |
| `media` | 媒体播放控制 | `/system/bin/media` |
| `getprop/setprop` | 系统属性 | `/system/bin/toolbox` |
| `bugreport` | 诊断报告生成 | `/system/bin/bugreport` |
| `top/ps` | 进程信息 | `/system/bin/toolbox` |
| `dmesg` | 内核日志 | `/system/bin/toolbox` |
| `logcat` | 日志查看 | `/system/bin/logcat` |
| `strace` | 系统调用跟踪 | `/system/bin/strace` |

> 所有命令均在 `adb shell` 下执行。
> 其中 `toolbox` 是 Android 的嵌入式 busybox 替代品，包含 top/ps/dmesg/getprop 等基础命令。
