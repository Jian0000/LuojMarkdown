+++
title = 'Android STB 长按 Power 键弹出关机对话框 — 定位过程总结'
date = '2026-06-08T16:33:00+08:00'
draft = false
+++

# Android STB 长按 Power 键弹出关机对话框 — 定位过程总结

## 背景

在 Android STB（Amlogic S905X5 平台）上，长按遥控器 Power 键会弹出一个系统级关机确认对话框，提示文字为 "Do you want to shut down?"。本文记录如何从现象出发，逐步定位到该对话框的源码位置和触发逻辑。

## 第一步：通过 dumpsys 获取窗口信息

在 `adb shell` 中执行：

```bash
# 1. 查看当前焦点窗口
dumpsys window | grep mCurrentFocus
# 输出: mCurrentFocus=Window{8fd2b64 u0 android}

# 2. 根据窗口 ID 查看详细信息
dumpsys window windows | grep -A 40 -B 10 "8fd2b64"
```

### 关键输出分析

| 属性                                 | 值                      | 含义                                        |
| ------------------------------------ | ----------------------- | ------------------------------------------- |
| `package=android`                    | 属于 `android` 包       | **这是 Framework 系统级窗口，非第三方 App** |
| `mOwnerUid=1000`                     | system 进程             | 进一步确认是系统级                          |
| `type=KEYGUARD_DIALOG`               | 2009                    | 锁屏级对话框，在所有用户上可见              |
| `fl=DIM_BEHIND ALT_FOCUSABLE_IM ...` | 半透明背景 + 可获取焦点 | 典型的模态对话框特征                        |
| `Window{8fd2b64 u0 android}`         | u0 = user 0             | 系统用户                                    |

**结论**：对话框是 Android Framework 中的系统级代码弹出的，不是任何 App 的行为。

---

## 第二步：搜索对话框提示文字

既然确认是 Framework 代码，第一步搜索提示文字定位字符串资源：

```bash
grep -rn "Do you want to shut down" frameworks/base/core/res/res/values/
```

找到：

- **文件**：`frameworks/base/core/res/res/values/strings.xml`

- **第 630 行**：

  ```xml
  <string name="shutdown_confirm_question">Do you want to shut down?</string>
  ```

得到字符串资源 ID：`com.android.internal.R.string.shutdown_confirm_question`。

---

## 第三步：搜索谁引用了这个字符串

```bash
grep -rn "shutdown_confirm_question" frameworks/base/services/ --include="*.java"
```

找到唯一引用：

- **文件**：`frameworks/base/services/core/java/com/android/server/power/ShutdownThread.java`

- **第 182-184 行**：

  ```java
  final int longPressBehavior = context.getResources().getInteger(
          com.android.internal.R.integer.config_longPressOnPowerBehavior);
  final int resourceId = mRebootSafeMode
          ? com.android.internal.R.string.reboot_safemode_confirm
          : (longPressBehavior == 2
                  ? com.android.internal.R.string.shutdown_confirm_question
                  : com.android.internal.R.string.shutdown_confirm);
  ```

### 关键发现

- 只有当 `config_longPressOnPowerBehavior == 2`（即 `LONG_PRESS_POWER_SHUT_OFF`）时，才会使用 "Do you want to shut down?" 这个字符串
- 如果是其他 behavior 值，则使用其他文字（如 "Your phone will shut down."）
- 第 209 行设置了窗口类型：`sConfirmDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_KEYGUARD_DIALOG);`
  - 这正好吻合了第一步 dumpsys 中看到的 `type=KEYGUARD_DIALOG`

---

## 第四步：追踪调用 ShutdownThread 的上游

搜索谁调用了 `ShutdownThread.shutdown()`：

```bash
grep -rn "ShutdownThread.shutdown\|mWindowManagerFuncs.shutdown" frameworks/base/services/
```

找到关键调用在 `PhoneWindowManager.java` 的 `powerLongPress()` 方法中：

- **文件**：`frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java`
- **第 1320-1365 行** — `powerLongPress()` 方法：

```java
private void powerLongPress(long eventTime) {
    final int behavior = getResolvedLongPressOnPowerBehavior();
    switch (behavior) {
        case LONG_PRESS_POWER_NOTHING:
            break;
        case LONG_PRESS_POWER_GLOBAL_ACTIONS:
            mPowerKeyHandled = true;
            showGlobalActions();  // 显示全局操作菜单
            break;
        case LONG_PRESS_POWER_SHUT_OFF:
        case LONG_PRESS_POWER_SHUT_OFF_NO_CONFIRM:
            mPowerKeyHandled = true;
            mWindowManagerFuncs.shutdown(behavior == LONG_PRESS_POWER_SHUT_OFF);
            // confirm=true → 弹出确认对话框
            // confirm=false → 直接关机
            break;
        case LONG_PRESS_POWER_GO_TO_VOICE_ASSIST:
            // 启动语音助手
            break;
        case LONG_PRESS_POWER_ASSISTANT:
            // 启动助理
            break;
    }
}
```

---

## 第五步：追踪电源键手势识别

长按手势是如何被检测到的？

在 `PhoneWindowManager.java` 中有一个内部类 `PowerKeyRule`（第 2516-2573 行），它继承自 `SingleKeyGestureDetector.SingleKeyRule`：

```java
private final class PowerKeyRule extends SingleKeyGestureDetector.SingleKeyRule {
    PowerKeyRule() { super(KEYCODE_POWER); }

    boolean supportLongPress() { return hasLongPressOnPowerBehavior(); }

    void onLongPress(long eventTime) {
        if (beganFromNonInteractive && !mSupportLongPressPowerWhenNonInteractive)
            return;
        powerLongPress(eventTime);  // → 触发上述 switch
    }
}
```

手势检测的触发链：

```
interceptKeyBeforeQueueing()                           // 第 4309 行
  → KEYCODE_POWER case (第 4584 行)
    → interceptPowerKeyDown()                          // 处理 wake lock 等
    → handleKeyGesture()                               // 第 4805 行
      → SingleKeyGestureDetector.interceptKey()
        → PowerKeyRule.onLongPress()
          → powerLongPress()
```

---

## 第六步：确认实际生效的配置值

AOSP 框架默认值在：

- `frameworks/base/core/res/res/values/config.xml` 第 1100 行：`config_longPressOnPowerBehavior = 5`（助理）

但实际设备上的值被 Amlogic 的 DroidOverlay 覆盖了：

- **`vendor/amlogic/common/apps/VendorOverlay/DroidOverlay/res/values/config.xml`** 第 188 行：

  ```xml
  <integer name="config_longPressOnPowerBehavior">2</integer>
  ```

这确认了当前设备上 `behavior = 2`（`LONG_PRESS_POWER_SHUT_OFF`），即长按 Power 键 = 弹出确认对话框后关机。

### `config_longPressOnPowerBehavior` 可选值

| 值   | 常量                                   | 行为                               |
| ---- | -------------------------------------- | ---------------------------------- |
| `0`  | `LONG_PRESS_POWER_NOTHING`             | 无操作                             |
| `1`  | `LONG_PRESS_POWER_GLOBAL_ACTIONS`      | 显示全局操作菜单                   |
| `2`  | `LONG_PRESS_POWER_SHUT_OFF`            | 关机（带确认对话框）← **当前配置** |
| `3`  | `LONG_PRESS_POWER_SHUT_OFF_NO_CONFIRM` | 关机（无确认，直接关）             |
| `4`  | `LONG_PRESS_POWER_GO_TO_VOICE_ASSIST`  | 语音助手                           |
| `5`  | `LONG_PRESS_POWER_ASSISTANT`           | 助理 ← AOSP 默认                   |

---

## 完整调用链路图

```
┌──────────────────────────────────────────────────────────────────────┐
│                       遥控器 Power 键（长按）                          │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Linux Kernel Input 子系统 → /dev/input/eventX                        │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  InputReader (frameworks/native/services/inputflinger)                │
│  读取原始输入事件，分发给 InputDispatcher                             │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PhoneWindowManager.interceptKeyBeforeQueueing()                     │
│  frameworks/base/services/core/java/com/android/server/policy/       │
│  PhoneWindowManager.java                                             │
│  ├── KEYCODE_POWER → interceptPowerKeyDown()                         │
│  └── handleKeyGesture() → SingleKeyGestureDetector                   │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PowerKeyRule.onLongPress()                                          │
│  → powerLongPress(eventTime)                    第 1320 行            │
│  → getResolvedLongPressOnPowerBehavior()                              │
│  → behavior = 2 (LONG_PRESS_POWER_SHUT_OFF)                          │
│     （由 vendor/amlogic/.../DroidOverlay/config.xml 覆盖）            │
│  → mWindowManagerFuncs.shutdown(true)            第 1345 行            │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ShutdownThread.shutdownInner(context, confirm=true)                  │
│  frameworks/base/services/core/java/com/android/server/power/        │
│  ShutdownThread.java                                    第 158 行     │
│  ├── 读取 config_longPressOnPowerBehavior = 2                         │
│  ├── 因为 behavior == 2，选用 shutdown_confirm_question 字符串        │
│  │   即 "Do you want to shut down?"                  第 182 行         │
│  ├── 构建 AlertDialog                                第 195 行         │
│  │   ├── Title: "Power off"                                            │
│  │   ├── Message: "Do you want to shut down?"                          │
│  │   ├── Positive: "Yes" → beginShutdownSequence()                     │
│  │   └── Negative: "No"                                                │
│  └── 设置窗口类型为 TYPE_KEYGUARD_DIALOG             第 209 行         │
│      → dialog.show()                                                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 涉及的关键文件一览

| 文件路径                                                     | 作用                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `frameworks/base/core/res/res/values/strings.xml:630`        | 定义 "Do you want to shut down?" 字符串                      |
| `frameworks/base/services/core/java/com/android/server/power/ShutdownThread.java:158-213` | 创建并显示关机确认对话框                                     |
| `frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java:1320-1365` | `powerLongPress()` — 根据 behavior 决定长按电源键的行为      |
| `frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java:2516-2573` | `PowerKeyRule` — 电源键手势识别规则                          |
| `frameworks/base/core/java/android/view/WindowManager.java:2098` | `TYPE_KEYGUARD_DIALOG` 常量定义                              |
| `frameworks/base/core/res/res/values/config.xml:1100`        | AOSP 默认 `config_longPressOnPowerBehavior = 5`              |
| **`vendor/amlogic/common/apps/VendorOverlay/DroidOverlay/res/values/config.xml:188`** | **实际生效的覆盖配置：`config_longPressOnPowerBehavior = 2`** |

---

## 调试方法论总结

定位这类系统级 UI 问题的通用方法：

1. **先确认窗口归属**：`dumpsys window` 查看 `package` 和窗口 `type`，判断是 System 还是 App 的窗口
2. **搜索字符串**：根据 UI 上显示的文字在 `strings.xml` 中找到资源 ID
3. **反向追踪引用**：搜索谁引用了这个资源 ID，找到 UI 创建代码
4. **正向追踪调用链**：从 UI 创建处向上游搜索调用者，找到触发入口
5. **确认实际配置**：搜索所有 overlay（包括 `device/`、`vendor/` 目录），找到最终生效的配置值，因为 AOSP 默认值往往会被设备厂商覆盖

---

# 实战案例：将长按 Power 键从关机改为重启

## 需求

将长按遥控器 Power 键的行为从"弹出关机确认对话框并关机"改为"弹出重启确认对话框并重启"。

## 需求分析

回顾当前的调用链路：

```
PhoneWindowManager.powerLongPress()
  → behavior = 2 (LONG_PRESS_POWER_SHUT_OFF)
  → mWindowManagerFuncs.shutdown(true)
  → WindowManagerService.shutdown(true)
  → ShutdownThread.shutdown(context, reason, true)
      → mReboot = false
      → shutdownInner(context, true)
          → 构建 AlertDialog
              ├── Title: "Power off"
              ├── Message: "Do you want to shut down?"
              ├── Positive: "Yes" → beginShutdownSequence() → 关机
              └── Negative: "No"
```

关键发现：`WindowManagerFuncs` 接口**已经内置了 `reboot(boolean confirm)` 方法**：

```java
// WindowManagerPolicy.java:254-256
public void shutdown(boolean confirm);
public void reboot(boolean confirm);
public void rebootSafeMode(boolean confirm);
```

而 `ShutdownThread` 也已有 `reboot()` 静态方法（第 247-253 行），它设置 `mReboot = true` 后调用同一个 `shutdownInner()`，底层 `run()` 方法会根据 `mReboot` 标志决定执行 `lowLevelReboot()` 还是 `lowLevelShutdown()`。

**但问题在于**：`shutdownInner()` 构建对话框时**没有检查 `mReboot` 标志**，所以直接调用 `reboot()` 会导致对话框仍然显示 "Power off" / "Do you want to shut down?"，但点击确认后实际执行的是重启——这是不一致的用户体验。

因此需要修改三处：

1. **PhoneWindowManager**：将 `shutdown()` 调用改为 `reboot()`
2. **ShutdownThread**：在 `shutdownInner()` 中根据 `mReboot` 显示不同的对话框文字
3. **strings.xml**：新增重启确认的字符串资源

## 方案对比

在实施前考虑了三种方案：

| 方案                                                    | 改动量     | 用户体验                     | 是否采用     |
| ------------------------------------------------------- | ---------- | ---------------------------- | ------------ |
| 方案一：改 PhoneWindowManager + ShutdownThread + 字符串 | 3 个文件   | 直接弹出重启确认框           | ✅ 采用       |
| 方案二：改 overlay 配置，behavior 从 2 改为 1           | 1 个文件   | 弹出全局操作菜单，需再选重启 | ❌ 体验不佳   |
| 方案三：利用已有 `persist.sys.power.key.action` 属性    | 需额外开发 | 运行时配置                   | ❌ 仅支持短按 |

**选择方案一**的理由：

- `WindowManagerFuncs.reboot()` 已经实现，底层基础设施完备
- 直接弹出 "Do you want to restart?" 确认框，用户体验最佳
- 改动精准，不影响其他调用路径（如 `ShutdownThread.shutdown()` 的调用者）
- 同时完善了 `shutdownInner()` 的逻辑，使所有通过 `reboot()` 的调用都能显示正确的对话框

## 具体修改

### 改动 1：PhoneWindowManager.java（长按行为从 shutdown 改为 reboot）

**文件**：`frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java`
**位置**：`powerLongPress()` 方法，第 1342-1345 行

```diff
 performHapticFeedback(HapticFeedbackConstants.LONG_PRESS_POWER_BUTTON, false,
-        "Power - Long Press - Shut Off");
+        "Power - Long Press - Reboot");
 sendCloseSystemWindows(SYSTEM_DIALOG_REASON_GLOBAL_ACTIONS);
-mWindowManagerFuncs.shutdown(behavior == LONG_PRESS_POWER_SHUT_OFF);
+mWindowManagerFuncs.reboot(behavior == LONG_PRESS_POWER_SHUT_OFF);
```

**说明**：

- `behavior == LONG_PRESS_POWER_SHUT_OFF` 仍然为 `true`（因为 overlay 配置 `config_longPressOnPowerBehavior = 2`），所以 `confirm` 参数为 `true`，会弹出确认对话框
- 调用链变为：`mWindowManagerFuncs.reboot(true)` → `WindowManagerService.reboot(true)` → `ShutdownThread.reboot(context, reason, true)` → `mReboot = true` → `shutdownInner(context, true)`

### 改动 2：ShutdownThread.java（对话框文字根据 mReboot 显示）

**文件**：`frameworks/base/services/core/java/com/android/server/power/ShutdownThread.java`
**位置**：`shutdownInner()` 方法

**改动 2a**：消息文字（第 180-185 行）

```diff
 final int resourceId = mRebootSafeMode
         ? com.android.internal.R.string.reboot_safemode_confirm
-        : (longPressBehavior == 2
-                ? com.android.internal.R.string.shutdown_confirm_question
-                : com.android.internal.R.string.shutdown_confirm);
+        : (mReboot
+                ? com.android.internal.R.string.reboot_confirm_question
+                : (longPressBehavior == 2
+                        ? com.android.internal.R.string.shutdown_confirm_question
+                        : com.android.internal.R.string.shutdown_confirm));
```

**改动 2b**：标题文字（第 196-199 行）

```diff
 .setTitle(mRebootSafeMode
         ? com.android.internal.R.string.reboot_safemode_title
-        : com.android.internal.R.string.power_off)
+        : (mReboot
+                ? com.android.internal.R.string.global_action_restart
+                : com.android.internal.R.string.power_off))
```

**说明**：

- 优先级：`mRebootSafeMode` > `mReboot` > `longPressBehavior`
- `mReboot=true` 时，使用 `global_action_restart`（"Restart"）作为标题，`reboot_confirm_question`（"Do you want to restart?"）作为消息
- 不影响 `mReboot=false` 的调用路径（如直接调用 `ShutdownThread.shutdown()` 的场景）

### 改动 3：strings.xml（新增字符串资源）

**文件**：`frameworks/base/core/res/res/values/strings.xml`
**位置**：`shutdown_confirm_question` 之后

```xml
<!-- Reboot Confirmation Dialog.  When the user chooses to restart the device, it asks
     the user if they'd like to reboot.  This is the message.  This is used instead of
     shutdown_confirm when the system is configured to use long press to go directly to the
     reboot dialog instead of shutdown. -->
<string name="reboot_confirm_question">Do you want to restart?</string>
```

## 修改后的调用链路

```
PhoneWindowManager.powerLongPress()
  → behavior = 2 (LONG_PRESS_POWER_SHUT_OFF)
  → mWindowManagerFuncs.reboot(true)              ← 改为 reboot
  → WindowManagerService.reboot(true)
  → ShutdownThread.reboot(context, reason, true)
      → mReboot = true                            ← 设置重启标志
      → shutdownInner(context, true)
          → 读取 config_longPressOnPowerBehavior = 2
          → mReboot = true, 选用 reboot_confirm_question
          → 构建 AlertDialog
              ├── Title: "Restart"                ← 改为 Restart
              ├── Message: "Do you want to restart?"  ← 改为重启文字
              ├── Positive: "Yes" → beginShutdownSequence() → 重启
              └── Negative: "No"
```

## 对话框文字对照

| 调用方式                                  | mReboot         | 标题                | 消息                                  | 确认后行为   |
| ----------------------------------------- | --------------- | ------------------- | ------------------------------------- | ------------ |
| `shutdown(..., true)` (直接关机)          | false           | Power off           | Do you want to shut down?             | 关机         |
| `shutdown(..., true)` (behavior=2)        | false           | Power off           | Do you want to shut down?             | 关机         |
| `reboot(..., true)` **(长按 Power 改后)** | **true**        | **Restart**         | **Do you want to restart?**           | **重启**     |
| `rebootSafeMode(..., true)`               | true + safeMode | Reboot to safe mode | Do you want to reboot into safe mode? | 安全模式重启 |

## 扩展思考：如何将长按 Power 改为其他功能？

基于这个案例的分析，如果要将长按 Power 改为启动自定义 Activity、发送广播、或其他功能，有以下几种方式：

### 方式 A：修改 `config_longPressOnPowerBehavior` overlay

```xml
<!-- vendor/amlogic/common/apps/VendorOverlay/DroidOverlay/res/values/config.xml -->
<!-- 改为 1：全局操作菜单；改为 0：无操作 -->
<integer name="config_longPressOnPowerBehavior">1</integer>
```

### 方式 B：在 `powerLongPress()` 中新增 case

在 `PhoneWindowManager.java` 的 `powerLongPress()` 中增加自定义行为分支：

```java
case CUSTOM_ACTION:
    mPowerKeyHandled = true;
    // 启动自定义 Activity
    Intent intent = new Intent("com.example.CUSTOM_ACTION");
    mContext.startActivity(intent);
    break;
```

### 方式 C：利用 Amlogic 的 `persist.sys.power.key.action` 属性

```bash
# 短按关机 (当前 powerPress() 已支持)
setprop persist.sys.power.key.action 1
# 短按重启
setprop persist.sys.power.key.action 2
```

此属性的处理逻辑在 `PhoneWindowManager.powerPress()` 第 1057 行，仅影响短按。如需影响长按，可在 `powerLongPress()` 中增加类似判断逻辑。

---

## 本案例关键经验

1. **Framework 接口往往已经提供了所需能力**——`WindowManagerFuncs.reboot()` 和 `ShutdownThread.reboot()` 都已存在，只需要把它们串起来
2. **修改系统对话框文字时，要检查底层方法的"隐式假设"**——`shutdownInner()` 方法名暗示它只为关机服务，但实际上 `reboot()` 也复用它，只是它内部没有正确处理 `mReboot` 标志
3. **静态变量的状态传递**——`mReboot` 是静态变量，在 `shutdown()`/`reboot()` 中设置，在 `shutdownInner()` 和 `run()` 中使用。这种隐式的状态传递是遗留代码常见模式，修改时需要注意不要破坏它
4. **字符串资源要放在正确的 values 目录**——框架代码引用 `com.android.internal.R.string.*`，因此字符串必须定义在 `frameworks/base/core/res/res/values/strings.xml` 中，不能放在 vendor overlay 里