+++
title = 'Java for Android 速查表'
date = '2026-06-09T18:03:00+08:00'
draft = false
+++

# Java for Android 速查表

> 目标读者：有 C/C++ 嵌入式背景，需要读懂 Android Framework 和 APK 源码
> 原则：**遇到再查**，不用从头啃 Java 教程。本文档只覆盖读代码必须理解的部分。

---

## 1. Java 与 C/C++ 的核心差异

| C/C++ | Java | 说明 |
|-------|------|------|
| `.h` + `.c` / `.cpp` | `.java` | 一个文件一个 public 类，文件名必须和类名一致 |
| `#include` | `import` | import 只是告诉编译器去哪找类，不复制代码 |
| 指针 `*` | **无指针** | 所有对象都通过引用访问，没有 `*` 和 `->` |
| 手动 `malloc/free` | **GC 自动回收** | `new` 分配，JVM 自动回收，不用 free |
| `struct` | 用 `class` 代替 | Java 没有 struct，纯数据用 POJO |
| 全局函数 | 所有函数必须在类里 | 没有全局函数，static 方法近似 |
| `#define` / `const` | `static final` | `public static final int MAX = 100;` |
| `bool` | `boolean` | 注意拼写不一样 |
| `NULL` | `null` | 全小写 |

---

## 2. 类、继承、接口 —— 重点

### 2.1 类 (Class)

定义方式和 C++ 类似，但区别：**所有方法默认是虚函数**（可被重写）。

```java
public class MyClass {
    // 成员变量
    private int count;
    public String name;

    // 构造方法（名和类名相同，没有返回值）
    public MyClass(String name) {
        this.name = name;   // this 和 C++ 一样
        count = 0;
    }

    // 方法
    public void doSomething() {
        count++;
    }
}
```

### 2.2 继承 (Inheritance) —— Framework 随处可见

```java
// Framework 中最常见的写法
public class ResolutionPropSetterService extends Service {
    // ...
}
```

要点：
- **单继承**：一个类只能 extend 一个父类（对比 C++ 可以多继承）
- 子类自动获得父类的 `public` / `protected` 方法和变量
- 调用父类方法用 `super.methodName()`（对比 C++ 的 `Parent::methodName()`）
- 方法默认是虚函数，子类可以 **@Override**

### 2.3 接口 (Interface) —— 理解回调的关键

接口是 **纯抽象** 的，只定义方法签名，不提供实现：

```java
// 定义接口（类似 C++ 的纯虚类）
public interface OnClickListener {
    void onClick(View v);
}

// 实现接口
public class MyActivity extends Activity implements OnClickListener {
    @Override
    public void onClick(View v) {
        // 实现接口方法
    }
}
```

Framework 中接口的常见场景：**回调**、**Listener**、**Callback**

```java
// 匿名内部类实现接口（Java 里非常常见的写法）
button.setOnClickListener(new OnClickListener() {
    @Override
    public void onClick(View v) {
        // 点击后的处理
    }
});
```

### 2.4 抽象类 (Abstract Class)

介于接口和普通类之间，可以有已实现的方法，也可以有抽象方法。

```java
public abstract class BaseTvService {
    // 抽象方法：子类必须实现
    public abstract void onTune(int channel);

    // 已实现方法：子类可以直接用或 override
    public void onStop() {
        // default implementation
    }
}
```

> **什么时候是接口，什么时候是抽象类？** 接口定义"能做什么"，抽象类提供"部分默认实现"。Framework 中大量使用接口进行 IPC 回调。

---

## 3. package / import 机制

### 3.1 package

```java
package cn.giec.shellcmd;
```

- 对应文件系统路径 `cn/giec/shellcmd/`
- 全限定名：`cn.giec.shellcmd.ShellCmd`
- 使用 `package` 避免类名冲突（类似 C++ 的 namespace）

### 3.2 import

```java
// 导入单个类
import android.app.Service;
import android.content.Intent;

// 导入整个包（不推荐在 Android 中使用，降低可读性）
import android.content.*;

// 静态导入（导入静态方法/常量）
import static android.Manifest.permission.INTERACT_ACROSS_USERS;
```

> **不用记 import**：IDE 会自动补全。每次读代码时看到 import 能理解即可。

---

## 4. 异常处理

C 中错误通常靠返回值（-1 / NULL），Java 用 `try-catch`。

### 4.1 基本写法

```java
try {
    // 可能抛出异常的代码
    SystemProperties.set(key, value);
} catch (RuntimeException e) {
    // 处理异常
    Slog.e(TAG, "Failed: " + key, e);
}
```

### 4.2 throws —— 声明方法可能抛异常

```java
// 方法声明时不处理，交给调用者处理
public void readFile(String path) throws IOException {
    // ...
}
```

### 4.3 finally —— 无论如何都执行

```java
try {
    // ...
} catch (Exception e) {
    // ...
} finally {
    // 不管有没有异常，都会执行（类似 RAII 的析构）
    close();
}
```

### 4.4 Framework 中的异常处理风格

参考代码中常见模式：**catch 宽泛的异常**：

```java
// 像 ResolutionPropSetterService 中这样
try {
    SystemProperties.set(key, value);
} catch (Throwable t) {         // Throwable 是最大的异常范畴
    Slog.e(TAG, "Failed", t);
}
```

> `Throwable` > `Exception` > `RuntimeException` — 越宽越省事，但会掩盖具体问题。

---

## 5. 常用修饰符 —— 读代码必懂

| 修饰符 | 含义 | 对比 C++ |
|--------|------|----------|
| `public` | 任何地方可访问 | `public` |
| `private` | 仅类内部可访问 | `private` |
| `protected` | 子类 + 同包可访问 | `protected` |
| `static` | 属于类本身，不属于实例 | `static` |
| `final` | 类：不可继承 / 方法：不可重写 / 变量：常量 | `final + 变量` = `const` |
| `abstract` | 类：不能直接实例化 / 方法：没有实现 | 纯虚函数 |
| `@Override` | 注解（不是修饰符），表示重写父类方法 | C++ 没有此概念 |

---

## 6. static 关键字 —— 理解它的多种用途

```java
public class ShellCmd {
    // 6.1 静态常量
    public static final String OK = "success";

    // 6.2 静态变量
    private static final String TAG = "ShellCmd";

    // 6.3 静态方法（可直接通过类名调用）
    public static String exec(String cmd) {
        // ...
    }

    // 6.4 静态初始化块（类加载时执行一次）
    static {
        System.loadLibrary("histbcmdservice_jni");
    }
}

// 调用静态方法（不需要创建对象）
String result = ShellCmd.exec("ls");
```

> C/C++ 背景理解：`static` 方法 ≈ 普通函数，`static` 变量 ≈ 全局变量，但都放在了类的命名空间里。

---

## 7. 泛型 (Generics) —— Framework 中大量使用

```java
// 写法：类名后面跟 <T>，T 是类型占位符
List<String> nameList = new ArrayList<>();       // 元素必须是 String
Map<String, Integer> scoreMap = new HashMap<>(); // key=String, value=Integer
```

Framework 常见泛型：

```java
// Binder 回调（AIDL 定义的服务）
public interface DeathRecipient {
    void binderDied();
}

// 各种 List / Map 泛型
List<Intent> intentList;
Map<String, IBinder> serviceMap;
```

> 理解泛型就够了：`List<String>` 就是"只能放 String 的列表"，`Map<K,V>` 就是"K→V 的映射表"。

---

## 8. 注解 (Annotation) —— 理解 Framework 写法的关键

注解以 `@` 开头，给编译器/工具提供额外信息。

```java
// 最常见的：声明重写父类方法
@Override
public IBinder onBind(Intent intent) {
    return null;
}

// Framework 中的常见注解
@NonNull         // 此参数/返回值不能为 null
@Nullable        // 此参数/返回值可以为 null
@CallSuper       // 重写此方法时必须调用 super 实现
@IntDef          // 限制 int 参数的可选值
@RequiresPermission  // 需要申明权限
```

> **遇到不认识的就跳过**，不影响理解逻辑流程。注解是元数据，不是业务逻辑。

---

## 9. Framework 核心类速查

### 9.1 Context（上下文）

一句话：Context 是"访问系统资源的入口"。
- 获取资源：`context.getString(R.string.hello)`
- 启动 Activity：`context.startActivity(intent)`
- 获取系统服务：`context.getSystemService(Context.WIFI_SERVICE)`

### 9.2 Intent（意图）

- 显式 Intent：指明要启动哪个类
  ```java
  Intent intent = new Intent(this, MyActivity.class);
  startActivity(intent);
  ```
- 隐式 Intent：描述要做的事，让系统找对应的组件
  ```java
  Intent intent = new Intent("com.example.ACTION_DO_SOMETHING");
  startActivity(intent);
  ```
- 携带数据
  ```java
  Intent intent = new Intent(this, MyService.class);
  intent.putExtra("key", "value");     // 参考代码中这种写法很常见
  String value = intent.getStringExtra("key");
  ```

### 9.3 Activity（页面）

相当于 Android 的"主函数入口"，一个屏幕对应一个 Activity。
生命周期：`onCreate()` → `onStart()` → `onResume()` → `onPause()` → `onStop()` → `onDestroy()`

**只需知道**：`onCreate()` 是入口，Framework 代码中常在 `onCreate()` 中做初始化。

### 9.4 Service（后台服务）

- 和 Activity 同级，但没有 UI 界面
- 参考代码中的 `ResolutionPropSetterService` 就是一个 Service
- 关键方法：`onStartCommand()`（收到启动命令时调用）、`onBind()`（绑定服务时调用）

### 9.5 BroadcastReceiver（广播接收者）

接收系统或应用发出的广播：

```java
// 常见的系统广播
Intent.ACTION_BOOT_COMPLETED   // 开机完成
Intent.ACTION_PACKAGE_ADDED    // 安装应用
```

---

## 10. 理解 Framework Java 代码的关键模式

### 模式 1：Activity 继承 + 生命周期回调

```java
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);     // 必须先调 super
        setContentView(R.layout.activity_main); // 设置布局
    }
}
```

### 模式 2：Service + onStartCommand —— 参考代码中有

```java
public class MyService extends Service {
    @Override
    public IBinder onBind(Intent intent) {
        return null;  // 不支持绑定
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 从 intent 取出参数
        String data = intent.getStringExtra("key");
        // 处理...
        stopSelf(startId);   // 处理完就停
        return START_NOT_STICKY;
    }
}
```

### 模式 3：静态方法工具类 —— ShellCmd 模式

```java
// 类似 C 的全局函数集合
public class ShellCmd {
    static { System.loadLibrary("foo"); }

    public static native String hsInvokeJni(String cmd, int type);

    public static String exec(String cmd) {
        return hsInvokeJni(cmd, 0);
    }
}
```

### 模式 4：匿名内部类做回调

```java
view.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        // 处理点击
    }
});
```

> **Java 8+ 可以简写为 lambda**：`view.setOnClickListener(v -> { ... });`，但 AOSP 很多老代码还是匿名内部类写法。

---

## 11. 速查：从 C/C++ 映射到 Java 的常见写法

| C/C++ | Java |
|-------|------|
| `int *arr = malloc(n * sizeof(int));` | `int[] arr = new int[n];` |
| `printf("x=%d\n", x);` | `Slog.d(TAG, "x=" + x);` |
| `#define MAX 100` | `public static final int MAX = 100;` |
| `if (ptr == NULL)` | `if (obj == null)` |
| `// 单行注释` | `// 单行注释`（完全一样） |
| `/* 多行 */` | `/* 多行 */`（完全一样） |
| `for (int i = 0; i < n; i++)` | 完全一样 |
| `if/else/while/do-while` | 完全一样（`bool` → `boolean`） |
| `// 返回值表示错误` | `try-catch` 处理错误 |
| `namespace` | `package` |
| `#include` | `import` |

---

## 12. 读代码时常见的"看着怪"的写法

```java
// 1. 链式调用（每个方法返回 this）
context.getSystemService(Context.WIFI_SERVICE);

// 2. 三元运算符
String value = intent != null ? intent.getStringExtra("key") : null;

// 3. instanceof 判断类型
if (obj instanceof String) { ... }

// 4. == 和 equals() 的区别
//    == 比较引用（是否同一个对象）
//    equals() 比较内容（是否相等）
String a = new String("hello");
String b = new String("hello");
a == b       // false（不同对象）
a.equals(b)  // true（内容相同）

// 5. StringBuilder——大量字符串拼接时用
StringBuilder sb = new StringBuilder();
sb.append("a=").append(a).append(", b=").append(b);
String result = sb.toString();
```

---

## 13. 学习建议

1. **别背语法**：把这份速查表放旁边，读代码时遇到不懂的回来查
2. **先用 Java 读代码**：当前阶段目标是"读懂"不是"会写"
3. **重点关注的 Java 特性优先级**：
   - 高优先级：类/继承/接口、try-catch、package/import、static
   - 中优先级：泛型、注解、匿名内部类
   - 低优先级：反射、Stream API、Lambda（遇到再学）
4. **搭配 IDE**：Android Studio 或 VS Code + Java 插件，代码导航（Ctrl+点击跳转定义）+ 自动补全极大降低读 Java 代码的门槛

---

> 遇到具体的 Java 语法问题时，可以用搜索快速定位，或者在参考代码库中搜类似用法。**速查表的目的不是一次性读完，而是作为参考夹在笔记本里。**
