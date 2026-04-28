
> 优先级：⭐⭐⭐ | 面试频率：🔥🔥🔥 | 掌握程度：深入

---


## 一、知识树

  

```

Java 基础

│

├── 1. 数据类型 ⭐⭐⭐

│   ├── 基本类型（8 种）：byte/short/int/long/float/double/char/boolean

│   ├── 引用类型：类 / 接口 / 数组

│   ├── 自动装箱与拆箱（IntegerCache 缓存 -128~127）

│   ├── 类型转换（隐式 / 强制）

│   └── String 的不可变性（final char[] / final byte[]）

│

├── 2. 面向对象 ⭐⭐⭐

│   ├── 封装：访问修饰符（private/default/protected/public）

│   ├── 继承：单继承 + 接口多实现

│   ├── 多态：动态绑定 / 方法重写 / 向上转型

│   ├── 抽象类 vs 接口（JDK8 default 方法后区别缩小）

│   └── Object 类核心方法（equals/hashCode/toString/clone）

│

├── 3. 泛型 ⭐⭐

│   ├── 泛型类 / 泛型方法 / 泛型接口

│   ├── 类型擦除（编译后变 Object，运行时无泛型信息）

│   ├── 通配符：? extends T（上界）/ ? super T（下界）/ PECS 原则

│   └── 泛型与数组的区别

│

├── 4. 反射 ⭐⭐⭐

│   ├── Class 对象获取方式（.class / forName / getClass）

│   ├── 运行时获取字段 / 方法 / 构造器

│   ├── setAccessible(true) 突破 private 限制

│   └── 反射的性能开销（比直接调用慢 10~100 倍）

│

├── 5. 注解 ⭐⭐

│   ├── 内置注解（@Override / @Deprecated / @SuppressWarnings）

│   ├── 元注解（@Retention / @Target / @Documented / @Inherited）

│   ├── 自定义注解

│   └── 注解 + 反射 = 框架的基石（Spring / MyBatis 全靠这套）

│

└── 6. 异常体系 ⭐⭐

    ├── Throwable → Error / Exception

    ├── 受检异常（Checked）vs 非受检异常（RuntimeException）

    ├── try-catch-finally 执行顺序（finally 一定执行）

    └── try-with-resources（AutoCloseable）

```

  

---

  

## 二、核心知识点精讲

  

### 2.1 数据类型 — 关键细节

  

**基本类型与内存占用：**
| 类型      | 大小   | 默认值   | 范围    |
| ------- | ---- | ----- | -------------- |
| byte    | 1 字节 | 0     | -128 ~ 127     |
| short   | 2 字节 | 0     | -32768 ~ 32767 |
| int     | 4 字节 | 0     | -2^31 ~ 2^31-1 |
| long    | 8 字节 | 0L    | -2^63 ~ 2^63-1 |
| float   | 4 字节 | 0.0f  | IEEE 754       |
| double  | 8 字节 | 0.0d  | IEEE 754       |
| char    | 2 字节 | '�'   | 0 ~ 65535      |
| boolean | ~    | false | true/false     |

  

**Integer 缓存陷阱（必考）：**

  

```java

Integer a = 127;

Integer b = 127;

System.out.println(a == b);  // true（缓存范围内）

  

Integer c = 128;

Integer d = 128;

System.out.println(c == d);  // false（超出缓存范围）

  

// 原因：IntegerCache 缓存了 -128 到 127

// 自动装箱调用 Integer.valueOf()，该方法内部有缓存判断

```

  

**String 不可变性：**

  

```java

// String 被 final 修饰，内部 byte[] 也被 final 修饰

// 任何"修改"操作都创建新对象

String s1 = "hello";

String s2 = "hello";

// s1 == s2 → true（字符串常量池）

// s1 = s1 + "world" → 新建对象，s1 指向新对象

```

  

### 2.2 面向对象 — 易混淆点

  

**抽象类 vs 接口（2026 年标准答案）：**
| 维度   | 抽象类         | 接口                          |
| ---- | ----------- | --------------------------- |
| 构造方法 | 可以有         | 不能有                         |
| 成员变量 | 可以有任意类型     | 只能 public static final      |
| 方法实现 | 可以有具体方法     | JDK8+ 可以有 default/static 方法 |
| 多继承  | 单继承         | 多实现                         |
| 设计意图 | "是什么"（is-a） | "能做什么"（can-do）              |

  

**equals 与 hashCode 契约（必考）：**

  

```

规则 1：equals 相等 → hashCode 必须相等

规则 2：hashCode 相等 → equals 不一定相等（哈希冲突）

规则 3：重写 equals 必须同时重写 hashCode

  

违反后果：HashMap/HashSet 中行为异常（同一对象存两份 / 找不到已存在的 key）

```

  

### 2.3 泛型 — 类型擦除

  

```java

// 编译前

List<String> list1 = new ArrayList<>();

List<Integer> list2 = new ArrayList<>();

  

// 编译后（类型擦除）

List list1 = new ArrayList();

List list2 = new ArrayList();

  

// 证明：运行时 list1.getClass() == list2.getClass() → true

```

  

**PECS 原则（Producer Extends, Consumer Super）：**

  

```java

// 如果只读不写（生产者）：用 ? extends T

List<? extends Number> producer = new ArrayList<Integer>();

Number n = producer.get(0);    // OK

// producer.add(1);            // 编译失败

  

// 如果只写不读（消费者）：用 ? super T

List<? super Integer> consumer = new ArrayList<Number>();

consumer.add(1);               // OK

// Integer i = consumer.get(0); // 编译失败（只能当 Object）

```

  

### 2.4 反射 — 框架基石

  

```java

// Spring IOC 创建 Bean 的核心逻辑（简化）

Class<?> clazz = Class.forName("com.example.UserService");

Constructor<?> constructor = clazz.getDeclaredConstructor();

Object instance = constructor.newInstance();

  

Method method = clazz.getMethod("save", String.class);

method.invoke(instance, "data");

```

  

**为什么框架大量使用反射？**

- 运行时动态创建对象，不需要硬编码 new

- 读取注解元数据，驱动框架行为（@Autowired / @RequestMapping）

- 这是 Spring / MyBatis / Hibernate 的底层原理

  

### 2.5 注解 — 从定义到运行时处理

  

```java

// 自定义注解

@Retention(RetentionPolicy.RUNTIME)  // 运行时保留（反射可读）

@Target(ElementType.FIELD)           // 作用于字段

public @interface MyColumn {

    String name();

    String type() default "VARCHAR";

}

  

// 使用

public class User {

    @MyColumn(name = "user_name")

    private String username;

}

  

// 运行时通过反射读取注解 → 这是 MyBatis/ORM 映射的原理

Field field = User.class.getDeclaredField("username");

MyColumn col = field.getAnnotation(MyColumn.class);

System.out.println(col.name()); // "user_name"

```

  

### 2.6 异常体系

  

```

                    Throwable

                   /         \

              Error          Exception

             /    \         /         \

      StackOverflow   IOException   RuntimeException

      OutOfMemoryError  SQLException  NullPointerException

                                      IndexOutOfBoundsException

                                      ClassCastException

```

  
| 分类                | 特点          | 示例                                    |
| ----------------- | ----------- | ------------------------------------- |
| Error             | JVM 级别，不可恢复 | OutOfMemoryError / StackOverflowError |
| Checked Exception | 编译器强制处理     | IOException / SQLException            |
| RuntimeException  | 编译不强制，运行时抛出 | NPE / ArrayIndexOutOfBounds           |

  

---

  

## 三、高频面试题（10 题）

  

### Q1：== 和 equals 的区别？

```

==：比较引用地址（基本类型比较值）

equals：比较内容（Object 默认用 ==，String 重写后逐字符比较）

  

关键点：重写 equals 必须重写 hashCode

```

  

### Q2：String / StringBuilder / StringBuffer 的区别？

```

String：不可变，线程安全（final 修饰）

StringBuilder：可变，非线程安全，性能最高

StringBuffer：可变，线程安全（synchronized），性能略低

  

场景：少量拼接用 String+，循环拼接用 StringBuilder，多线程用 StringBuffer

```

  

### Q3：Java 中的泛型是怎么实现的？什么是类型擦除？

```

实现方式：编译时类型检查 + 运行时擦除为 Object

类型擦除：编译后 List<String> 和 List<Integer> 是同一个类

局限：不能 new T()、不能 instanceof 泛型、不能创建泛型数组

```

  

### Q4：什么是反射？有什么应用场景？

```

定义：运行时获取类的信息（字段/方法/构造器）并动态调用

场景：Spring IOC（动态创建 Bean）、MyBatis（动态映射）、JUnit（找@Test方法）

缺点：性能开销大、破坏封装性

```

  

### Q5：接口和抽象类的区别？什么时候用哪个？

```

接口：定义行为契约，多实现，JDK8+ 有 default 方法

抽象类：定义公共实现，单继承，可以有构造方法

  

选择标准：

- 多个不相关的类共享行为 → 接口（Comparable）

- 有共同状态和实现 → 抽象类（AbstractList）

```

  

### Q6：Java 中的异常体系是怎样的？受检异常和非受检异常的区别？

```

受检异常：编译器强制 try-catch 或 throws，如 IOException

非受检异常：编译不强制，如 NullPointerException

  

争议：业界有人认为 Checked Exception 是设计失误（增加了代码耦合）

Spring 体系统一转为 RuntimeException

```

  

### Q7：Integer a = 127 和 Integer b = 127，a == b 结果是什么？128 呢？

```

127 → true（IntegerCache 缓存范围 -128~127）

128 → false（超出缓存，new 了不同对象）

  

注意：Integer.valueOf() 走缓存，new Integer() 永远新建对象

```

  

### Q8：final / finally / finalize 的区别？

```

final：修饰类（不可继承）、方法（不可重写）、变量（不可修改引用）

finally：try-catch-finally 中最后执行的代码块（释放资源）

finalize：Object 的方法，GC 回收前调用（JDK9 已废弃，不推荐使用）

```

  

### Q9：什么是自动装箱和拆箱？有什么需要注意的？

```

装箱：基本类型 → 包装类型（int → Integer，调用 Integer.valueOf()）

拆箱：包装类型 → 庺本类型（Integer → int，调用 intValue()）

  

注意：

1. 频繁装箱/拆箱有性能开销

2. 拆箱时如果对象为 null → NullPointerException

3. 缓存池：Integer(-128~127) / Character(0~127) / Boolean

```

  

### Q10：Java 中的深拷贝和浅拷贝？

```

浅拷贝：复制对象引用，修改副本影响原对象（clone() 默认行为）

深拷贝：递归复制所有层级，完全独立

  

实现方式：

1. 实现 Cloneable + 重写 clone()（手动深拷贝每个字段）

2. 序列化/反序列化（最通用）

3. 拷贝构造函数

```

  

---

  

## 四、常见误区

  
| 误区 | 正确理解 |
|------|---------|
| String 是基本类型 | String 是引用类型，不可变（final） |
| 泛型在运行时也存在 | 类型擦除，运行时只有原始类型 |
| equals 相等 hashCode 就一样 | 对，但反过来不对（哈希冲突） |
| 抽象类不能有构造方法 | 可以有，子类会调用 |
| Java 参数传递是引用传递 | Java 只有值传递（对象传的是引用的副本） |
| finally 一定会执行 | System.exit() 除外，或线程被杀 |

  

---

  

## 五、实战应用场景

  
| 知识点 | 实际使用场景 |
|--------|------------|
| String 不可变 | 字符串常量池优化 / HashMap 的 key 安全 / 多线程共享 |
| 泛型 + 反射 | Spring IOC 容器创建和管理 Bean |
| 注解 + 反射 | @Autowired 注入 / @RequestMapping 路由映射 |
| 异常体系 | Spring 统一异常处理 @RestControllerAdvice |
| equals/hashCode | HashSet 去重 / HashMap 的 key 判等 |
| 泛型 PECS | Collections 工具类的方法签名设计 |

  

---

  

## 六、关联模块

  

- → [02-JVM.md](02-JVM.md)：Java 基础中的类型系统、反射等在 JVM 层面的实现

- → [03-并发编程.md](03-并发编程.md)：面向对象 + 泛型是并发容器的基础

- → [05-Spring全家桶.md](05-Spring全家桶.md)：反射 + 注解是 Spring 的底层机制