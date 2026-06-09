+++
title = 'ADB相关'
date = '2026-06-08T16:00:00+08:00'
draft = false
+++

```
# 设备端启用网络 ADB
su
setprop service.adb.tcp.port 5555
stop adbd
start adbd

ip addr show

# PC 端连接
adb connect 192.168.141.241:5555
    192.168.110.240
    192.168.140.29

#root权限连接
adb root

# 注意: 连接前确保网络可达
ping <device_ip>
```

# ADB 常用命令大全（Markdown 速查）

> 说明：以下命令默认已将 `adb` 加入环境变量；如有多台设备，请优先使用 `-s <serial>` 指定设备（见“多设备/选择目标设备”）。

---

## 0. 基础与排错

```bash
adb version                  # 查看 adb 版本
adb help                     # 查看帮助
adb start-server             # 启动 adb server
adb kill-server              # 关闭 adb server
adb nodaemon server          # 前台启动 server（便于看日志）
adb wait-for-device          # 阻塞直到设备连接可用
adb reconnect                # 重新连接设备（USB/TCP）
adb reconnect device         # 重新连接设备端
adb reconnect offline        # 尝试重连离线设备
```

常见排错：

```bash
adb devices -l               # 查看设备列表（带更多信息）
adb usb                      # 切回 USB 模式
adb tcpip 5555               # 切到 TCP 模式（需 USB 已连接一次）
adb connect <ip>:5555        # 通过网络连接
adb disconnect <ip>:5555     # 断开指定网络设备
adb disconnect               # 断开全部网络设备
```

---

## 1. 多设备/选择目标设备

```bash
adb devices                  # 列出已连接设备
adb devices -l               # 更详细信息
adb -s <serial> shell        # 对指定设备执行命令
adb -d shell                 # 仅对 USB 设备（单一）执行
adb -e shell                 # 仅对模拟器执行
adb -t <transport_id> shell  # Android 11+ 可用：按 transport_id 选择
```

获取序列号：

```bash
adb get-serialno
adb -s <serial> get-serialno
```

---

## 2. 设备信息

```bash
adb shell getprop                    # 查看全部系统属性
adb shell getprop ro.product.model   # 机型
adb shell getprop ro.build.version.release  # Android 版本
adb shell getprop ro.build.version.sdk      # SDK 版本
adb shell settings get system screen_brightness  # 读取系统设置（示例）
adb shell wm size                    # 屏幕分辨率
adb shell wm density                 # 屏幕 DPI
adb shell df -h                      # 存储情况
adb shell uptime                     # 运行时长
```

---

## 3. 安装/卸载/应用管理（pm）

安装：

```bash
adb install app.apk                  # 安装
adb install -r app.apk               # 覆盖安装（保留数据）
adb install -d app.apk               # 允许版本降级
adb install -g app.apk               # 安装时授予所有运行时权限
adb install-multiple base.apk split_*.apk  # 安装 split APK
adb install --user 0 app.apk         # 为 user 0 安装（部分 ROM/场景）
```

卸载：

```bash
adb uninstall <package>              # 卸载（删除数据）
adb uninstall -k <package>           # 卸载但保留数据与缓存（若支持）
```

列出/查询：

```bash
adb shell pm list packages                 # 列出所有包名
adb shell pm list packages -3              # 仅第三方应用
adb shell pm list packages -s              # 仅系统应用
adb shell pm list packages <keyword>       # 按关键字过滤
adb shell pm path <package>                # 查看 APK 路径
adb shell pm dump <package>                # 输出包信息（较多）
adb shell cmd package resolve-activity --brief <intent>  # 解析 Activity（新命令体系）
```

清理数据/授权：

```bash
adb shell pm clear <package>               # 清除应用数据
adb shell pm grant <package> <permission>  # 授予权限
adb shell pm revoke <package> <permission> # 撤销权限
```

启用/停用：

```bash
adb shell pm disable-user --user 0 <package>  # 对 user 0 停用
adb shell pm enable <package>                 # 启用
adb shell pm uninstall --user 0 <package>     # 仅对 user 0 卸载（系统应用常用）
```

---

## 4. 启动/停止应用与 Activity（am）

启动应用主界面：

```bash
adb shell monkey -p <package> -c android.intent.category.LAUNCHER 1
```

启动 Activity：

```bash
adb shell am start -n <package>/<activity>    # 指定组件启动
adb shell am start -a android.intent.action.VIEW -d "https://example.com"
```

启动 Service / 发送广播：

```bash
adb shell am startservice -n <package>/<service>
adb shell am broadcast -a <action> --es key value
```

强制停止/查看当前前台：

```bash
adb shell am force-stop <package>             # 强制停止
adb shell dumpsys activity activities | head  # 查看 Activity 栈（输出很多）
adb shell dumpsys window | grep mCurrentFocus # 当前焦点窗口（部分系统可用）
```

---

## 5. 进入 Shell 与常用 Linux/Android 命令

```bash
adb shell                       # 进入交互式 shell
adb shell <cmd>                 # 直接执行单条命令
adb shell "cd /sdcard && ls"    # 执行复合命令
```

常见：

```bash
adb shell ls /sdcard
adb shell pwd
adb shell cat /proc/cpuinfo
adb shell ps -A                 # 进程列表（不同版本参数略有差异）
adb shell top -n 1
adb shell id                    # 当前用户信息
adb shell su                    # root（仅已 root/工程机）
```

---

## 6. 文件传输（push/pull）与目录管理

```bash
adb push <local> <remote>       # 推送到设备
adb pull <remote> <local>       # 从设备拉取
adb pull <remote>               # 拉取到当前目录
adb shell mkdir -p /sdcard/test
adb shell rm -rf /sdcard/test
```

提示：Android 10+ 分区存储影响 `/sdcard/Android/data` 等目录访问，可能需要 `run-as`、root 或通过应用自身导出。

---

## 7. 截图/录屏

截图：

```bash
adb shell screencap -p /sdcard/screen.png
adb pull /sdcard/screen.png .
adb shell rm /sdcard/screen.png
```

直接输出到本机文件（避免占用手机存储）：

```bash
adb exec-out screencap -p > screen.png   //这样输出的文件似乎会损坏
```

录屏：

```bash
adb shell screenrecord /sdcard/demo.mp4
adb pull /sdcard/demo.mp4 .
adb shell rm /sdcard/demo.mp4
```

常用参数：

```bash
adb shell screenrecord --time-limit 30 /sdcard/demo.mp4
adb shell screenrecord --size 1280x720 --bit-rate 6000000 /sdcard/demo.mp4
```

---

## 8. 日志（logcat）与抓取问题现场

logcat：

```bash
adb logcat                             # 实时日志
adb logcat -c                          # 清空缓冲区
adb logcat -d > logcat.txt             # 导出当前日志并退出
adb logcat -v time                     # 带时间格式
adb logcat *:W                         # 仅 Warning 及以上
adb logcat <TAG>:V *:S                 # 仅某 TAG
adb logcat --pid=$(adb shell pidof <package>)  # 仅某进程（新版本支持）
logcat -s Activity 
```

抓取更完整的诊断：

```bash
adb bugreport > bugreport.zip          # Android 7+ 通常输出 zip
adb shell dumpsys                      # 输出大量系统服务状态
adb shell dumpsys meminfo <package>    # 内存信息
adb shell dumpsys cpuinfo              # CPU 信息（老设备常用）
```

---

## 9. 模拟输入（input）

点击/滑动/文本：

```bash
adb shell input tap 500 1200
adb shell input swipe 200 1200 200 300         # 向上滑动
adb shell input swipe 200 300 200 1200 500     # 带时长（ms）
adb shell input text "hello_world"             # 空格需转义或用 %s
adb shell input keyevent 3                     # HOME
adb shell input keyevent 4                     # BACK
adb shell input keyevent 26                    # POWER
adb shell input keyevent 24                    # VOLUME_UP
adb shell input keyevent 25                    # VOLUME_DOWN
```

常用 KeyEvent（部分）：

```text
3 HOME, 4 BACK, 19/20/21/22 方向键, 23 DPAD_CENTER/ENTER,
66 ENTER, 67 DEL, 82 MENU, 187 APP_SWITCH
```

---

## 10. 端口转发/反向转发（调试必备）

```bash
adb forward tcp:6100 tcp:7100         # 本机 6100 -> 设备 7100
adb reverse tcp:8081 tcp:8081         # 设备 8081 -> 本机 8081（React Native 常用）
adb forward --list                    # 查看转发列表
adb forward --remove tcp:6100
adb forward --remove-all
adb reverse --list
adb reverse --remove tcp:8081
adb reverse --remove-all
```

---

## 11. 重启与模式切换

```bash
adb reboot                     # 重启系统
adb reboot recovery            # 重启到 recovery
adb reboot bootloader          # 重启到 bootloader/fastboot
adb reboot sideload            # 重启到 sideload（部分机型/Recovery 支持）
adb reboot-bootloader          # 等价写法（部分环境）
```

---

## 12. Root / 分区挂载（需要设备支持）

> 仅工程机/userdebug/已 root 设备可用，商用机通常不可用。

```bash
adb root                      # 以 root 重启 adbd
adb unroot                    # 取消 root
adb remount                   # 重新挂载系统分区为可写（需要 root）
adb disable-verity            # 禁用 verity（危险，谨慎）
adb enable-verity             # 启用 verity
```

---

## 13. run-as（调试应用私有目录）

> 仅对 **debuggable** 应用有效（通常 debug 包可用）。

```bash
adb shell run-as <package> ls
adb shell run-as <package> cat files/config.json
adb shell run-as <package> cp files/a.db /sdcard/a.db
adb pull /sdcard/a.db .
```

---

## 14. AppOps / 权限与设置（进阶）

```bash
adb shell appops get <package>
adb shell appops set <package> <OP> allow|deny|ignore

adb shell settings list system
adb shell settings list secure
adb shell settings list global
adb shell settings get secure android_id
adb shell settings put global animator_duration_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global window_animation_scale 0
```

---

## 15. 复制/粘贴与剪贴板（部分版本/ROM 支持）

```bash
adb shell cmd clipboard get            # Android 10+ 常见
adb shell cmd clipboard set "hello"    # 可能因 ROM/权限限制不可用
```

---

## 16. 常见组合用法示例

查看并过滤应用日志：

```bash
adb shell pidof <package>
adb logcat --pid=<pid> -v time
```

从 URL 拉起浏览器：

```bash
adb shell am start -a android.intent.action.VIEW -d "https://www.android.com"
```

获取当前前台包名（不同版本可能不一致）：

```bash
adb shell dumpsys window | grep -E "mCurrentFocus|mFocusedApp"
```

---

## 17. 备忘：常用参数与注意点

- `-s <serial>`：多设备时指定目标设备（强烈建议养成习惯）
- `adb exec-out ...`：直接把设备输出流重定向到本机文件（截图/录屏/导出更方便）
- Android 10+：分区存储导致部分目录不可读；优先用应用内导出、`run-as` 或调试接口
- 某些命令在不同 Android 版本/ROM 上存在差异（如 `ps` 参数、剪贴板命令等）

