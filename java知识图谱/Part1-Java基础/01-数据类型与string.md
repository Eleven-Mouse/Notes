 >掌握程度：深入
----
## 一、基本数据类型（8 种）

| 类型 | 大小 | 默认值 | 范围 | 面试考点 |
|------|------|--------|------|---------|
| byte | 1 字节 | 0 | -128 ~ 127 | 溢出计算 |
| short | 2 字节 | 0 | -32768 ~ 32767 | 很少考 |
| **int** | 4 字节 | 0 | -2^31 ~ 2^31-1 | 整型默认类型 / 溢出 |
| long | 8 字节 | 0L | -2^63 ~ 2^63-1 | 后缀 L / 毫秒时间戳 |
| float | 4 字节 | 0.0f | IEEE 754 | 精度丢失 |
| double | 8 字节 | 0.0d | IEEE 754 | 浮点默认类型 |
| char | 2 字节 | '' | 0~65535 | Unicode / UTF-16 |
| boolean | ~ | false | true/false | JVM 用 int 实现 |

### 细化考点

**1. 整型溢出**

```java
int max = Integer.MAX_VALUE; // 2147483647
int result = max + 1; // -2147483648（溢出为负数，不报错！）

// 防止溢出：用 Math.addExact（溢出时抛 ArithmeticException）
int safe = Math.addExact(max, 1); // 抛异常
```

**2. 浮点精度丢失**

```java
double a = 0.1 + 0.2; // 0.30000000000000004

// 解决：BigDecimal
BigDecimal b = new BigDecimal("0.1").add(new BigDecimal("0.2")); // 0.3

// 注意：BigDecimal 必须用 String 构造，不用 double
```

**3. 类型转换**

```java
// 自动（隐式）：小 → 大（byte → short → int → long → float → double）
// 强制（显式）：大 → 小（可能丢失精度/溢出）

// 特例：
byte a = 1;
byte b = 2;
byte c = a + b; // 编译报错！a+b 提升为 int
byte c = (byte)(a + b); // OK
```

---

## 二、自动装箱与拆箱

### 原理

```java
// 编译前
Integer a = 100;
int b = a;

// 编译后（反编译）
Integer a = Integer.valueOf(100); // 装箱
int b = a.intValue(); // 拆箱
```

### Integer 缓存池（必考）

```java
Integer a = 127;
Integer b = 127;
a == b; // true（缓存范围内 -128~127）

Integer c = 128;
Integer d = 128;
c == d; // false（超出缓存，new 了不同对象）

// 源码：IntegerCache.cache 数组缓存了 -128 到 127 的 Integer 对象
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

### 拆箱的 NPE 陷阱

```java
Integer num = null;
int n = num; // NullPointerException！自动拆箱调用了 null.intValue()

// 场景：数据库查询返回的 Integer 字段为 null
// DTO 中的 Integer → Service 中的 int 赋值 → NPE

// 规范：DO/DTO 中使用包装类型，局部变量可以使用基本类型
```

### 各包装类缓存范围

| 包装类 | 缓存范围 | 备注 |
|--------|---------|------|
| Integer | -128 ~ 127 | 可通过 -XX:AutoBoxCacheMax 扩大上限 |
| Long | -128 ~ 127 | 同 Integer |
| Short | -128 ~ 127 | 同 Integer |
| Byte | -128 ~ 127 | 全部缓存（byte 范围就是这个） |
| Character | 0 ~ 127 | ASCII 范围 |
| Boolean | TRUE / FALSE | 两个常量 |
| Float / Double | 无缓存 | 每次都 new |

---

## 三、String 深度解析

### 3.1 不可变性

```java
// String 类定义
public final class String implements java.io.Serializable, Comparable, CharSequence {
    private final byte[] value;  // JDK9+: byte[] (之前是 char[])
    private final byte coder;    // 编码标记 LATIN1 / UTF16
    private int hash;            // 懒计算哈希值
}

// 不可变原因：
// 1. 类被 final 修饰 → 不可继承
// 2. value 被 final 修饰 → 引用不可变
// 3. value 被 private 修饰 + 无 setter → 外部无法修改
// 4. String 中所有方法都不会修改原对象 → 返回新对象
```

**为什么设计为不可变？**

1. 字符串常量池：共享引用，节省内存（如果可变，一个修改影响所有引用）
2. HashMap 的 key：不可变保证 hashCode 稳定（hash 缓存）
3. 线程安全：不可变对象天然线程安全
4. 安全性：网络连接/文件路径用 String 传递，防止被篡改

### 3.2 字符串常量池

```java
// 直接赋值 → 从常量池取（或创建）
String s1 = "hello"; // 常量池
String s2 = "hello"; // 复用常量池中的
s1 == s2; // true

// new → 堆上新建对象（不管常量池有没有）
String s3 = new String("hello"); // 堆上新对象
s1 == s3; // false

// intern() → 将字符串加入常量池并返回常量池引用
String s4 = s3.intern();
s1 == s4; // true
```

### 3.3 String / StringBuilder / StringBuffer

| 维度 | String | StringBuilder | StringBuffer |
|------|--------|--------------|-------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 安全（不可变） | 不安全 | 安全（synchronized） |
| 性能 | 拼接慢（每次新建） | 最快 | 略慢（锁开销） |
| 适用 | 少量操作 | 单线程拼接 | 多线程拼接 |

```java
// 编译器优化：常量拼接在编译期完成
String s = "a" + "b" + "c";
// 编译后等价于 String s = "abc";

// 变量拼接 → 编译器自动用 StringBuilder
String a = "a";
String s = a + "b" + "c";
// 编译后等价于 new StringBuilder().append(a).append("b").append("c").toString()

// 循环中拼接 → 必须手动用 StringBuilder（编译器不会优化循环内的 +）
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i); // 正确
    // result += i; // 错误！每次循环都 new StringBuilder
}
```

### 3.4 JDK9 String 优化（compact strings）

- JDK8：`char[]` 存储，每个字符 2 字节
- JDK9：`byte[]` + `coder`
  - LATIN1（纯 ASCII）：每个字符 1 字节（省一半内存）
  - UTF16（含中文等多字节字符）：每个字符 2 字节
- 影响：大部分纯英文场景内存减半

---

## 四、面试题精选

### Q1：`"hello"` 和 `new String("hello")` 的区别？

- `"hello"`：常量池中（可能复用）
- `new String("hello")`：堆上新建对象（不管常量池有没有）
- 区别：内存位置不同，`==` 比较为 false

延伸：`new String("hello")` 创建了几个对象？
- 常量池没有 `"hello"`：2 个（常量池 1 个 + 堆 1 个）
- 常量池已有 `"hello"`：1 个（堆上的）

### Q2：Integer 缓存什么范围？为什么？

- 范围：`-128 ~ 127`
- 原因：Short/Integer/Long 的 `valueOf()` 使用缓存数组
- Java 认为这个范围的整数使用最频繁
- 可通过 `-XX:AutoBoxCacheMax=<size>` 扩大上限

实战陷阱：

```java
Integer a = 200;
Integer b = 200;
if (a == b) { ... } // false！比较包装类型用 equals
```

### Q3：为什么浮点数计算会丢失精度？怎么解决？

- 原因：IEEE 754 二进制浮点数无法精确表示某些十进制小数
- `0.1` 在二进制中是无限循环小数

解决：
1. `BigDecimal`（金融场景必须用）
2. 整数运算后除法（金额用分存储，long 类型）

BigDecimal 注意：
- 构造用 String 或 valueOf，不要用 double 构造
- 比较用 `compareTo`，不要用 `equals`（equals 认为 2.0 ≠ 2.00）

### Q4：String 的 hashCode 是怎么计算的？

```java
h = 0;
for (int i = 0; i < value.length; i++) {
    h = 31 * h + value[i];
}
```

为什么用 31？
1. 31 是质数，减少哈希冲突
2. `31 * i = (i << 5) - i` → 位运算优化，计算高效
3. 历史原因，经典选择

### Q5：switch 支持 String 的原理？

- JDK7+ 支持 `switch(String)`
- 原理：编译后转换为字符串的 hashCode 比较
- `switch(str)` → 编译为 `switch(hashCode)`
- 如果 hash 冲突 → 再用 equals 确认
- 注意：case 标签必须是编译期常量表达式

---

## 五、关联知识

- → [02-面向对象.md](02-面向对象.md)：String 的 final 设计体现了不可变模式
- → [Part2-JVM/06-内存结构.md](06-内存结构.md)：字符串常量池在 JVM 中的位置
- → [Part3-并发编程/10-线程基础.md](10-线程基础.md)：String 不可变与线程安全的关系
