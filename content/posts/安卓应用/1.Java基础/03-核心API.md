+++
title = '1. Java 核心 API'
date = '2026-06-08T16:05:00+08:00'
draft = false
+++

# Java 核心 API

> 参考指导书 2.3 Java 核心 API

## 学习清单

- [x] String 与 StringBuilder
- [x] 包装类（Integer, Long, Double 等）
- [x] Math 类
- [x] 日期与时间（Date, Calendar, LocalDateTime）
- [x] 集合框架
  - [x] List（ArrayList, LinkedList）
  - [x] Set（HashSet, TreeSet）
  - [x] Map（HashMap, TreeMap）
- [x] 泛型（Generics）

---

## 1. String 与 StringBuilder

### 1.1 String 的不可变性

String 是 Java 中最特殊的类之一：**一旦创建，内容永远不能改**。

```java
String s = "Hello";
s = s + " World";    // 看起来像是改了 s，实际是创建了一个新对象
```

`"Hello"` 那个对象本身没变——`s + " World"` 在内存中创建了一个全新的 `"Hello World"` 对象，然后把 `s` 指向了它。原来的 `"Hello"` 如果没被引用，就等着被垃圾回收。

**为什么要把 String 设计成不可变？**

- **安全**：String 无处不在——文件名、URL、数据库密码、类名。如果 String 可变，你在传参的过程中别人偷偷改了内容，后果不可控
- **字符串常量池**：因为不可变，相同内容的字符串才能共享同一份内存（见下面）
- **HashMap 的 key**：String 是最常用的 key。如果 String 可变，hashCode 就会变，存进去的 value 就找不回来了
- **线程安全**：不可变意味着天然线程安全，不需要加锁

### 1.2 字符串常量池

```java
String s1 = "Hello";           // 字面量 → 放入常量池
String s2 = "Hello";           // 池中已有，直接复用
String s3 = new String("Hello"); // new → 强制在堆上创建新对象

System.out.println(s1 == s2);   // true  —— 同一个对象
System.out.println(s1 == s3);   // false —— 不同对象
System.out.println(s1.equals(s3)); // true —— 内容相同
```

`==` 比较的是**内存地址**。`equals()` 比较的是**内容**。String 重写了 `equals()`，所以判断字符串内容是否相同用 `equals()`，不要用 `==`。

### 1.3 常用方法

```java
String s = "  Hello World  ";

s.length()                // 15
s.trim()                  // "Hello World"（去掉首尾空格）
s.toUpperCase()           // "  HELLO WORLD  "
s.substring(2, 7)         // "Hello"（左闭右开：[2,7)）
s.charAt(4)               // 'l'
s.contains("World")       // true
s.indexOf("o")            // 5（第一个出现的位置）
s.replace("World", "Java") // "  Hello Java  "
s.split(" ")              // ["", "", "Hello", "World", "", ""]
```

### 1.4 StringBuilder：为拼接而生

**痛点**：

```java
// 这个循环每一次 += 都会创建一个新的 String 对象
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;          // 创建了 10000 个临时对象！浪费内存 + 耗时
}
```

每次 `+=` 都要：分配新内存 → 拷贝旧内容 → 拼接新内容 → 丢弃旧对象。循环 10000 次就产生 10000 个垃圾对象。

**StringBuilder 解决**：

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);         // 在同一个对象上追加，不创建新的
}
String result = sb.toString();  // 最后一次性转成 String
```

StringBuilder 内部是一个**可扩容的字符数组**，`append()` 直接在数组尾部写入，不需要反复 new 对象。

| | String | StringBuilder |
|------|--------|---------------|
| 可变性 | 不可变 | 可变 |
| 线程安全 | 天然安全 | 不安全（用 StringBuffer 替代） |
| 拼接性能 | 每次创建新对象 | 原地追加，快得多 |
| 适用场景 | 内容不变的字符串 | 频繁拼接/修改的字符串 |

Java 编译时，简单的 `"a" + "b" + "c"` 会被自动优化为 StringBuilder，但循环中的 `+=` 不会——编译器没那么聪明。

---

## 2. 包装类

### 2.1 为什么需要包装类

Java 分两种类型：基本类型（`int`、`double` 等）和引用类型（类、接口）。但有些场景**只能用引用类型**：

```java
// 泛型不能接受基本类型
List<int> list;              // 编译错误！
List<Integer> list;          // 正确

// null 只能赋给引用类型
int x = null;                // 编译错误！
Integer x = null;            // 正确（表示"没有值"）
```

包装类就是给每个基本类型套了一个"壳"，让它们能以对象的形式存在。

| 基本类型 | 包装类 |
|----------|--------|
| `byte` | `Byte` |
| `short` | `Short` |
| `int` | `Integer` |
| `long` | `Long` |
| `float` | `Float` |
| `double` | `Double` |
| `char` | `Character` |
| `boolean` | `Boolean` |

### 2.2 自动装箱与拆箱

JDK 5 之后，基本类型和包装类之间可以自动转换：

```java
Integer i = 100;        // 自动装箱：int → Integer（编译器插入 Integer.valueOf(100)）
int j = i;              // 自动拆箱：Integer → int（编译器插入 i.intValue()）

// 运算时也会自动拆箱
Integer a = 10;
Integer b = 20;
Integer c = a + b;      // a、b 先拆箱成 int，计算后结果再装箱成 Integer
```

### 2.3 常用方法

```java
// 字符串 → 数值
int x = Integer.parseInt("123");       // 123
double d = Double.parseDouble("3.14"); // 3.14

// 数值 → 字符串
String s = Integer.toString(100);      // "100"
String s2 = String.valueOf(100);       // "100"（更通用）

// 常量
Integer.MAX_VALUE   // 2147483647
Integer.MIN_VALUE   // -2147483648
```

**注意**：`Integer.parseInt("abc")` 会抛出 `NumberFormatException`。

---

## 3. Math 类

纯工具类，全部是静态方法，不需要创建对象。

```java
Math.abs(-5)       // 5        绝对值
Math.max(3, 8)     // 8        最大值
Math.min(3, 8)     // 3        最小值
Math.sqrt(16)      // 4.0      平方根
Math.pow(2, 3)     // 8.0      次方
Math.random()      // 0.0~1.0  随机数（不含 1）
Math.round(3.6)    // 4        四舍五入
Math.floor(3.6)    // 3.0      向下取整
Math.ceil(3.2)     // 4.0      向上取整
Math.PI            // 3.14159... 圆周率常量
```

---

## 4. 日期与时间

### 4.1 旧 API 的问题

Java 最早的日期类是 `java.util.Date`，但它设计得很糟糕：

```java
Date now = new Date();
System.out.println(now);       // Tue May 26 10:30:00 CST 2026 —— 格式不可控
System.out.println(now.getYear());  // 返回 126！（从 1900 年算起，1900+126=2026）
System.out.println(now.getMonth()); // 返回 4！（0 代表 1 月，4 代表 5 月，反人类）
```

**三大痛点**：

| 问题 | 说明 |
|------|------|
| 月从 0 开始 | 1 月是 0，12 月是 11——无数次 bug 的根源 |
| 可变对象 | `date.setMonth(6)` 会直接修改对象——线程不安全 |
| API 混乱 | Date 只负责时间戳，格式化要靠 `SimpleDateFormat`，计算要靠 `Calendar`——三个类各管各的 |

后来 Java 1.1 推出了 `Calendar`：

```java
Calendar cal = Calendar.getInstance();
cal.set(2026, Calendar.MAY, 26);   // 还是 0-based 的月份！
int year = cal.get(Calendar.YEAR);  // get/set 方法冗长

// 计算 10 天后的日期
cal.add(Calendar.DAY_OF_MONTH, 10);  // 直接修改了原对象！不是返回新对象
```

Calendar 仍然是**可变的**、月份仍从 0 开始、API 依旧啰嗦。

### 4.2 Java 8 新 API：LocalDateTime

Java 8（2014 年）引入了全新的 `java.time` 包，参考了业界优秀的 Joda-Time 库：

```java
// 获取当前时间
LocalDate today = LocalDate.now();           // 2026-05-26
LocalTime now = LocalTime.now();             // 10:30:00.123
LocalDateTime dt = LocalDateTime.now();      // 2026-05-26T10:30:00.123

// 创建指定时间
LocalDate date = LocalDate.of(2026, 5, 26);  // 月份是正常的 1~12！
LocalDateTime dt2 = LocalDateTime.of(2026, 5, 26, 10, 30);

// 操作时间——返回新对象，原对象不变（不可变！）
LocalDate nextWeek = today.plusDays(7);
LocalDate lastMonth = today.minusMonths(1);
int year = today.getYear();                   // 2026
int month = today.getMonthValue();            // 5（不是 4！）

// 格式化
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss");
String str = dt.format(fmt);                  // "2026/05/26 10:30:00"
LocalDateTime parsed = LocalDateTime.parse("2026/05/26 10:30:00", fmt);
```

### 4.3 新旧对比总结

| | 旧 API (Date/Calendar) | 新 API (java.time) |
|------|------------------------|---------------------|
| 可变性 | 可变，线程不安全 | **不可变**，线程安全 |
| 月份 | 0~11，反直觉 | 1~12，正常 |
| API 设计 | 分散在多类中 | 统一在 `java.time` 包下 |
| 格式化 | `SimpleDateFormat`（线程不安全！） | `DateTimeFormatter`（线程安全） |
| 计算 | `cal.add()` 修改原对象 | `plusXxx()` 返回新对象 |

> 原则：写新代码**永远用 `java.time` 包**。学旧 API 只是为了能读懂老项目的代码。

---

## 5. 集合框架

### 5.1 为什么需要集合

数组有两个硬伤：

```java
// 问题 1：长度固定
String[] arr = new String[3];
arr[0] = "A"; arr[1] = "B"; arr[2] = "C";
// arr[3] = "D";  越界！数组不能自动扩容

// 问题 2：缺少常用操作
// 数组没有：contains()、indexOf()、remove()、sort()……全要手写
```

集合就是"增强版数组"——自动扩容、内置增删改查、选择不同的数据结构（List/Set/Map）来匹配不同使用场景。

### 5.2 集合家族总览

```
Collection（接口）
├── List（接口）        有序、可重复、有索引
│   ├── ArrayList       底层数组，查快改慢
│   └── LinkedList      底层双向链表，增删快查慢
│
└── Set（接口）         无序、不可重复
    ├── HashSet         基于哈希表，最快
    └── TreeSet         基于红黑树，自动排序

Map（接口）             键值对，独立于 Collection
├── HashMap             基于哈希表，key 不可重复
└── TreeMap             基于红黑树，key 自动排序
```

### 5.3 ArrayList — 最常用的列表

```java
List<String> list = new ArrayList<>();

// 增
list.add("Apple");
list.add("Banana");
list.add(1, "Cherry");       // 在索引 1 插入

// 删
list.remove("Banana");       // 按内容删
list.remove(0);              // 按索引删

// 改
list.set(0, "Updated");

// 查
String s = list.get(0);      // 按索引取
boolean has = list.contains("Apple");
int idx = list.indexOf("Cherry");
int size = list.size();

// 遍历
for (String item : list) { ... }
```

### 5.4 ArrayList vs LinkedList

| | ArrayList | LinkedList |
|------|-----------|------------|
| 底层 | **数组** | **双向链表** |
| 按索引访问 `get(i)` | **快** O(1) | 慢 O(n)（要从头遍历） |
| 头部插入/删除 | 慢 O(n)（元素需搬家） | **快** O(1) |
| 尾部追加 | 快（均摊 O(1)） | 快 O(1) |
| 内存 | 紧凑 | 每个节点多存两个指针 |

**选择原则**：90% 的场景用 ArrayList。除非你的代码频繁在列表头部插入/删除（比如实现一个队列），才考虑 LinkedList。

### 5.5 HashSet — 自动去重的集合

```java
Set<String> set = new HashSet<>();
set.add("A"); set.add("B"); set.add("A");  // 重复的 "A" 不会存进去
System.out.println(set);     // [A, B]（顺序不保证）
System.out.println(set.size()); // 2

// 判断是否存在（最快：O(1)）
if (set.contains("A")) { ... }

// 遍历（顺序不确定）
for (String s : set) { ... }
```

**HashSet 的去重原理**：
1. 先比较 `hashCode()`——不同直接判定不同
2. hashCode 相同再比较 `equals()`——确认是否真的相同
3. 所以放入 HashSet 的对象，**必须正确重写 `hashCode()` 和 `equals()`**。String、Integer 等 JDK 自带类已经重写好了，直接用。

### 5.6 TreeSet — 自动排序的去重集合

```java
Set<Integer> set = new TreeSet<>();
set.add(5); set.add(1); set.add(3); set.add(2);
System.out.println(set);     // [1, 2, 3, 5] —— 自动升序

Set<String> words = new TreeSet<>();
words.add("banana"); words.add("apple"); words.add("cherry");
System.out.println(words);   // [apple, banana, cherry] —— 字典序
```

TreeSet 基于红黑树，插入删除 O(log n)，每次操作后自动保持有序。

### 5.7 HashMap — 最常用的键值对

```java
Map<String, Integer> map = new HashMap<>();

// 增/改
map.put("Zhang San", 90);
map.put("Li Si", 85);

// 查
int score = map.get("Li Si");          // 85
boolean has = map.containsKey("Zhang San");

// 删
map.remove("Li Si");

// 遍历（三种方式）
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " -> " + entry.getValue());
}

for (String key : map.keySet()) { ... }      // 只遍历 key
for (Integer value : map.values()) { ... }   // 只遍历 value
```

HashMap 的 key 也是靠 `hashCode()` + `equals()` 来判断是否重复，跟 HashSet 一样的规则。

---

## 6. 泛型

### 6.1 没有泛型时的噩梦

Java 1.4 及以前，集合是这样用的：

```java
// 没有泛型：集合里存什么全靠自觉
List list = new ArrayList();
list.add("Hello");
list.add(123);           // 什么都往里塞——编译器不拦

// 取出时必须强制转型
String s = (String) list.get(0);  // 每次都要转型

// 直到运行时才发现问题
String s2 = (String) list.get(1);  // Integer 不能转 String → ClassCastException 崩了
```

**问题**：类型错误被推迟到运行时才发现。编译期完全不管，写错了也能编译通过，到了线上才爆炸。

### 6.2 泛型解决：编译期把关

```java
List<String> list = new ArrayList<>();  // 声明：这个 list 只能放 String
list.add("Hello");
// list.add(123);           // 编译直接报错！

String s = list.get(0);     // 取出来就是 String，不需要转型
```

**效果**：把类型检查从运行时提前到编译期——bug 发现越早，修复成本越低。

### 6.3 泛型类

```java
// 定义一个可以装任意类型数据的箱子
class Box<T> {                          // T = Type，占位符，调用时确定具体类型
    private T data;

    public void set(T data) { this.data = data; }
    public T get() { return data; }
}

Box<String> strBox = new Box<>();
strBox.set("Hello");
String s = strBox.get();                // 取出来是 String，不用转型

Box<Integer> intBox = new Box<>();
intBox.set(100);
int i = intBox.get();                   // 取出来是 Integer，自动拆箱
```

### 6.4 泛型方法

一个方法需要处理多种类型的数组，不写泛型就得为每种类型各写一个：

```java
// 泛型方法：一个方法通吃所有类型
public static <T> void printArray(T[] arr) {
    for (T item : arr) {
        System.out.print(item + " ");
    }
    System.out.println();
}

String[] words = {"Hello", "World"};
Integer[] nums = {1, 2, 3};
printArray(words);     // 自动推断 T = String
printArray(nums);      // 自动推断 T = Integer
```

### 6.5 菱形语法

```java
// JDK 7 之前：右边也要写全泛型类型
List<String> list = new ArrayList<String>();  // 啰嗦

// JDK 7+：右边可以省略，编译器自动推断
List<String> list = new ArrayList<>();        // 简洁
```

### 6.6 泛型的核心价值

1. **类型安全**：错误从运行时提前到编译期
2. **消除强制转型**：取出时直接就是正确的类型
3. **代码复用**：一个 Box\<T\> 替代 N 个 BoxString、BoxInteger...

---

## 关键记忆点

### String
1. String **不可变**——所有操作返回新对象，原对象不变
2. `==` 比地址，`equals()` 比内容——字符串判等用 `equals()`
3. 循环中频繁拼接用 **StringBuilder**，不要用 `+=`

### 包装类
4. 基本类型不能用于泛型，不能赋 `null`——包装类解决这两个问题
5. 自动装箱 `Integer i = 100` 本质是 `Integer.valueOf(100)`

### 日期时间
6. **写新代码永远用 `java.time` 包**（LocalDate / LocalDateTime），旧 API 只是为了读老项目
7. 旧 API 的月份从 0 开始（0 = 1 月），新 API 从 1 开始

### 集合框架
8. ArrayList 查快改慢（数组），LinkedList 增删快查慢（链表）——默认选 ArrayList
9. HashSet 去重靠 `hashCode()` + `equals()`——放入的对象要正确重写这两个方法
10. HashMap 的 key 同样靠 `hashCode()` + `equals()` 判重

### 泛型
11. 泛型把类型检查从运行时提前到**编译期**，消除 `ClassCastException`
12. `Box<T>` 是"类型参数化"——一个类服务所有类型，不用每种类型写一个版本
