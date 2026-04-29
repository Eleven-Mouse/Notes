# Stage 2-1: JVM 深度增强

> 优先级 ⭐⭐⭐⭐⭐ | 面试频率 🔥🔥🔥🔥🔥

---

## 知识点 1：JVM 内存结构 ⭐🔥

### 【是什么】
JVM 在运行时将管理的内存划分为线程私有（PC、栈、本地方法栈）和线程共享（堆、方法区）两大阵营，每个区域职责分明。

### 【原理】

```
JVM 内存 = 一家公司

线程私有 = 每个员工的工位（互不干扰）
  ├── PC = 工作日志（记录做到哪一步了）
  ├── 虚拟机栈 = 员工的待办事项栈（方法调用栈）
  └── 本地方法栈 = 调用外部专家的通讯录（Native方法）

线程共享 = 公司公共区域
  ├── 堆 = 仓库（存放所有对象/货物）
  └── 方法区 = 制度文档库（类信息/常量/静态变量）

直接内存 = 公司租的外部仓库（NIO DirectByteBuffer）
```

#### 核心数据：各区域参数
| 区域 | 参数 | 默认值 | OOM类型 |
|------|------|--------|---------|
| 堆 | `-Xms` / `-Xmx` | 1/4物理内存 | `Java heap space` |
| 栈 | `-Xss` | 1MB | `StackOverflowError` |
| 元空间 | `-XX:MetaspaceSize` | 21MB(初始) | `Metaspace` |
| 直接内存 | `-XX:MaxDirectMemorySize` | 约等于-Xmx | `Direct buffer memory` |

#### 对象内存布局（64位JVM，开启指针压缩）
```
Object Header (12 bytes)
  ├── Mark Word (8 bytes) → GC信息 + 锁信息 + hashCode
  └── Klass Pointer (4 bytes) → 指向类元数据

Instance Data → 实例字段（按类型大小排列）

Padding → 按8字节对齐填充

示例：new Object() = 16 bytes (12 header + 4 padding)
     new Integer(1) = 16 bytes (12 header + 4 int)
```

### 【怎么用 — 项目落地】

**场景1：线上OOM排查**

```bash
# 1. 发现OOM
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/dump.hprof -jar app.jar

# 2. 分析dump
jmap -histo:live <pid> | head -20    # 看哪些对象最多
# 或用 MAT / Arthas 分析

# 3. 常见原因
# - Java heap space → 大对象/内存泄漏（集合只增不减）
# - Metaspace → 动态代理生成类过多（CGLIB）
# - GC overhead limit → GC回收太频繁
# - Direct buffer → NIO未释放
```

**场景2：博客系统中的JVM内存分析**

```
博客系统的内存分布：
├── 堆
│   ├── 文章对象（Article）→ 长文本String，可能是大对象
│   ├── 评论列表（List<Comment>）→ 集合类
│   ├── 缓存对象（本地Cache）→ Guava Cache / Caffeine
│   └── AI对话上下文（ChatContext）→ 长对话可能占用大量内存
├── 元空间
│   ├── Spring动态代理类（AOP）
│   ├── MyBatis Mapper代理类
│   └── 反射类信息
└── 栈
    └── 深度递归（如树形评论嵌套 → 改为迭代方式避免StackOverflow）
```

### 【问题场景】

| 问题 | 触发条件 | 排查思路 |
|------|----------|----------|
| `Java heap space` | 大对象/内存泄漏 | jmap + MAT分析支配树 |
| `StackOverflowError` | 递归太深/循环调用 | 看异常栈找递归点 |
| `Metaspace` | 动态类生成过多 | 检查CGLIB/反射使用 |
| `Direct buffer` | NIO ByteBuffer未释放 | 检查Netty/DirectByteBuffer |
| GC频繁 | 堆太小/内存泄漏 | jstat看GC频率 + 调大堆 |

### 【面试怎么问】

**Q：说一下 JVM 内存结构？**

> 标准回答（结构化）：
> 
>1. "JVM 内存分为线程私有和线程共享两大类。私有的是程序计数器、虚拟机栈、本地方法栈；共享的是堆和方法区（JDK8后叫元空间）。
>2. 程序计数器是唯一不会OOM的区域，记录当前执行的字节码行号。
>3. 虚拟机栈每个方法调用创建一个栈帧，包含局部变量表、操作数栈、动态链接、返回地址。
>4.  堆是GC的主要区域，所有对象实例和数组都在堆上分配。
>5.  方法区存放类信息、常量、静态变量，JDK8后用元空间实现，使用本地内存。"

**追问1：堆里面怎么分代的？为什么要分代？**

> "堆分为新生代（Young）和老年代（Old），新生代又分Eden + S0 + S1。
> 分代是因为大部分对象朝生夕死（如方法局部变量），只有少数长期存活。
> 新生代用复制算法（效率高，存活少），老年代用标记-整理（存活多，避免碎片）。
> 默认比例 Young:Old = 1:2，Eden:S0:S1 = 8:1:1。"

**追问2：对象什么时候进入老年代？**

> "四种情况：
> 1. 年龄达到阈值（默认15，CMS下6），每次Minor GC年龄+1
> 2. 大对象直接进入（-XX:PretenureSizeThreshold）
> 3. 动态年龄判断：Survivor中相同年龄对象大小总和超过Survivor空间一半
> 4. Minor GC后Survivor放不下，通过担保机制进入老年代"

---

## 知识点 2：垃圾回收 ⭐🔥

### 【是什么】
JVM 自动回收不再被引用的对象所占的内存，程序员无需手动释放。

### 【原理】

#### 用"垃圾分类"类比

```
判断对象是否存活 = 判断垃圾是否还有人用

可达性分析 = 从GC Roots出发，能到达的 = 还在用
GC Roots 包括：
  ├── 虚拟机栈中的引用（局部变量）
  ├── 方法区中静态变量引用
  ├── 方法区中常量引用
  ├── 本地方法栈JNI引用
  ├── JVM内部引用（基本类型对应的Class对象/常驻异常/类加载器）
  └── 所有被同步锁(synchronized)持有的对象

四种引用强度：
  强引用 > 软引用 > 弱引用 > 虚引用
  Object obj = new Object()  → 强引用（永不回收）
  SoftReference<Object>      → 软引用（内存不足时回收，适合缓存）
  WeakReference<Object>      → 弱引用（下次GC回收，ThreadLocal）
  PhantomReference<Object>   → 虚引用（仅用于跟踪GC，配合引用队列）
```

#### GC算法对比

```
┌──────────────┬──────────┬──────────┬──────────────────┐
│    算法       │  优点    │  缺点     │   适用场景        │
├──────────────┼──────────┼──────────┼──────────────────┤
│ 标记-清除     │ 简单     │ 碎片多     │ 老年代(CMS)      │
│ 复制算法      │ 无碎片   │ 空间减半   │ 新生代(S0/S1)    │
│ 标记-整理     │ 无碎片   │ 移动开销   │ 老年代(Serial)   │
│ 分代收集      │ 综合最优 │ 实现复杂   │ 现代JVM默认       │
└──────────────┴──────────┴──────────┴──────────────────┘
```

#### 收集器全景图

```
                    新生代收集器                    老年代收集器
                 ┌──────────────┐              ┌──────────────┐
                 │   Serial     │─────────────→│ Serial Old   │
                 └──────────────┘              └──────────────┘
                 ┌──────────────┐              ┌──────────────┐
                 │ ParNew       │─────────────→│   CMS        │
                 └──────────────┘              └──────────────┘
                 ┌──────────────┐
                 │ Parallel     │
                 │ Scavenge     │
                 └──────────────┘

              全堆收集器
              ┌──────────────────────────────────┐
              │  G1（JDK9默认）                    │
              │  ZGC（JDK15+，亚毫秒停顿）         │
              │  Shenandoah（与ZGC竞争）           │
              └──────────────────────────────────┘
```

#### G1 核心原理（重点）

```
传统收集器：按代划分连续内存
G1：将堆划分为等大Region（1~32MB），每个Region可以是Eden/Survivor/Old/Humongous

                  ┌─────┬─────┬─────┬─────┬─────┬─────┐
                  │ Eden│ Old │S0   │ Eden│ Old │Huge │
                  ├─────┼─────┼─────┼─────┼─────┼─────┤
                  │ Old │ Eden│S1   │ Old │ Eden│ Eden│
                  └─────┴─────┴─────┴─────┴─────┴─────┘
                  每个 Region 独立回收，优先回收垃圾最多的Region（Garbage First）

G1 三种回收模式：
  Young GC → 回收所有Eden + Survivor Region
  Mixed GC → 回收所有Young + 部分垃圾最多的Old Region
  Full GC  → 退化为串行回收整个堆（应极力避免）
```

### 【怎么用 — 项目落地】

**场景：博客系统GC调优**

```bash
# 典型配置（4核8G服务器，博客+AI系统）
java -Xms4g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \        # 目标停顿200ms
     -XX:G1HeapRegionSize=4m \          # Region大小
     -XX:InitiatingHeapOccupancyPercent=45 \  # 老年代45%触发Mixed GC
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/data/logs/dump.hprof \
     -jar blog-ai-system.jar

# AI接口的特殊考虑：
# - AI对话上下文是大对象 → 可能直接进老年代
# - 用SoftReference包装AI上下文缓存，内存不足自动回收
```

### 【问题场景】

| 问题 | 原因 | 解决 |
|------|------|------|
| Full GC频繁 | 老年代空间不足/内存泄漏 | 排查泄漏 + 调大老年代 |
| Young GC时间长 | Eden区太大 | 减小Young区或换G1 |
| CPU飙高 | GC线程占用 | jstat看GC频率 |
| 线程卡死 | STW（Stop The World） | 换低停顿收集器(G1/ZGC) |

### 【面试怎么问】

**Q：说一下常见的垃圾收集器？G1的原理？**

> "Serial是单线程收集器，适合客户端。ParNew是多线程版Serial，常和CMS配合。Parallel Scavenge注重吞吐量。CMS关注低停顿，但产生浮动垃圾和空间碎片。G1是JDK9默认，将堆划分为Region，优先回收垃圾最多的Region，可控停顿时间。ZGC是JDK15+引入的，通过染色指针和读屏障实现亚毫秒停顿，适合超大堆。"

**追问1：CMS的问题？为什么被废弃？**

> "CMS四个阶段：初始标记(STW) → 并发标记 → 重新标记(STW) → 并发清除。
> 问题：1.浮动垃圾（并发标记阶段新产生的垃圾）2.空间碎片（标记-清除不整理）3.并发阶段占CPU。4.Concurrent Mode Failure（老年代预留不够，退化为Serial Old Full GC）。
> JDK9标记废弃，JDK14移除，因为G1已经足够好。"

**追问2：G1什么时候会Full GC？怎么避免？**

> "当Mixed GC回收速度跟不上分配速度，老年代被填满时会退化为Serial Full GC。
> 避免：1.增大堆或调低IHOP阈值提前Mixed GC 2.增加回收的Old Region数量 3.排查内存泄漏 4.最终方案是换ZGC。"

---

## 知识点 3：类加载机制 ⭐🔥

### 【是什么】
JVM 将 `.class` 文件加载到内存，并转换为 `java.lang.Class` 对象的过程。

### 【原理】

#### 五个阶段
```
.java → javac → .class → JVM类加载

加载 → 验证 → 准备 → 解析 → 初始化
 │       │       │       │       │
 │       │       │       │       └─ 执行<clinit>()，静态变量赋值+static块
 │       │       │       └─ 符号引用→直接引用
 │       │       └─ 静态变量赋零值（注意：不是初始值）
 │       └─ 文件格式/元数据/字节码/符号引用验证
 └─ 通过全限定名获取二进制字节流（可从jar/war/网络/动态代理生成）
```

#### 双亲委派机制
```
类加载请求自底向上委托，加载自顶向下尝试：

       Bootstrap ClassLoader（rt.jar，JAVA_HOME/lib）
              │
       Extension ClassLoader（ext目录）
              │
       Application ClassLoader（classpath）
              │
       自定义 ClassLoader（自定义加载路径）

核心规则：先让父加载器加载，父加载器加载不了才自己加载
目的：保证Java核心类（如Object）不会被用户类覆盖
```

#### 打破双亲委派的场景
```
场景1：SPI机制（JDBC）
  DriverManager在rt.jar（Bootstrap加载）→ 需要加载厂商Driver（classpath）
  解决：线程上下文类加载器（Thread Context ClassLoader）

场景2：Tomcat
  一个Tomcat部署多个应用，每个应用需要独立类加载器
  解决：WebAppClassLoader，先自己加载再委托父类

场景3：OSGi / 模块化
  网状类加载，而非树状
  解决：每个Bundle有自己的ClassLoader
```

### 【怎么用 — 项目落地】

**场景：Spring Boot 的类加载**
```java
// Spring Boot 打成 fat jar 后
// 使用 LaunchedURLClassLoader 加载 BOOT-INF/classes 和 BOOT-INF/lib
// 每个 Spring Boot 应用有独立的类加载器
// 支持嵌套 jar 加载（传统URLClassLoader不支持jar in jar）

// 项目中碰到的问题：
// 多模块项目中，同一依赖的不同版本被不同ClassLoader加载
// 导致 ClassCastException（类相同但ClassLoader不同）
// 解决：统一依赖版本 或 排除冲突依赖
```

### 【问题场景】

| 问题 | 原因 | 解决 |
|------|------|------|
| ClassNotFoundException | 类路径下找不到 | 检查classpath/依赖 |
| NoClassDefFoundError | 编译时有运行时无 | 检查运行时依赖 |
| ClassCastException | 不同ClassLoader加载的同类 | 统一ClassLoader |
| Metaspace OOM | 大量动态代理类 | 增大Metaspace/排查泄漏 |

### 【面试怎么问】

**Q：说一下类加载机制？什么是双亲委派？**

> "类加载五个阶段：加载、验证、准备、解析、初始化。双亲委派是类加载的委托机制：收到加载请求后先委托父加载器，父加载器加载不了才自己加载。好处是保证核心类唯一性，比如Object不管哪个ClassLoader加载最终都来自Bootstrap。"

**追问1：什么时候会打破双亲委派？**

> "三种经典场景：SPI机制如JDBC，Bootstrap需要加载classpath的Driver，用线程上下文类加载器。Tomcat部署多应用，每个应用需要隔离，WebAppClassLoader先自己加载再委托父类。OSGi模块化，网状加载而非树状。Spring Boot的LaunchedURLClassLoader也做了类似处理来支持fat jar。"

---

## 知识点 4：JVM 调优实战 ⭐

### 【是什么】
通过调整JVM参数和使用工具，优化应用的内存使用和GC表现。

### 【原理 — 工具链】

```bash
# 命令行工具（必会）
jps -lv                  # 查看Java进程
jstat -gc <pid> 1000 10  # 每秒打印GC统计，打10次
jmap -heap <pid>         # 查看堆内存分布
jmap -histo:live <pid>   # 查看存活对象统计
jstack <pid>             # 查看线程堆栈（排查死锁/CPU高）

# Arthas（阿里开源，线上诊断神器）
dashboard                # 实时面板
thread -n 3              # CPU最高的3个线程
heapdump /tmp/dump.hprof # 导出堆dump
sc -d com.blog.*         # 搜索类信息
watch com.blog.Service method params  # 方法入参观察
```

### 【怎么用 — 项目落地】

**经典排查流程：CPU 100%**

```bash
# 1. 找到Java进程PID
jps -lv

# 2. 找到CPU最高的线程
top -Hp <pid>
# 或
printf "%x\n" <thread_id>  # 转为十六进制

# 3. 查看线程堆栈
jstack <pid> | grep <hex_thread_id> -A 30

# 常见原因：
# - 死循环（代码逻辑bug）
# - 正则回溯（ReDoS）
# - GC线程占用（频繁Full GC）
# - 加密运算（密集计算）
```

**经典排查流程：内存泄漏**

```bash
# 1. 确认内存泄漏
jstat -gcutil <pid> 1000
# 观察老年代是否持续上升，Full GC后不下降

# 2. 导出dump
jmap -dump:format=b,file=dump.hprof <pid>
# 或配置 -XX:+HeapDumpOnOutOfMemoryError

# 3. MAT分析
# - Dominator Tree（支配树）找最大对象
# - Leak Suspects（泄漏嫌疑）
# - GC Roots引用链

# 博客系统中常见泄漏：
# - ThreadLocal未remove → 线程池复用导致Entry不回收
# - 静态集合只put不remove → 类级别的内存泄漏
# - 监听器/回调注册后未注销
```

### 【面试怎么问】

**Q：线上CPU 100%怎么排查？**

> "四步：1.top找到Java进程PID 2.top -Hp找到CPU最高的线程 3.printf转十六进制 4.jstack导出线程栈grep定位。如果是GC线程，jstat看GC频率判断是否内存问题。如果是业务线程，看堆栈定位代码行。"

**追问：线上频繁Full GC怎么办？**

> "jstat -gcutil确认Full GC频率和老年代使用率。如果Full GC后老年代不下降，说明内存泄漏，dump后MAT分析支配树。如果下降后很快又满，说明老年代不够或对象太快晋升，调大堆或调低晋升阈值。配置用G1 + -XX:MaxGCPauseMillis控制停顿。"
