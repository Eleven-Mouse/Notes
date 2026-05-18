
> 并发理论基础，高频考点

---

## 一、知识树

```
JMM 与 volatile
├── JMM 内存模型
│   ├── 主内存 vs 工作内存
│   ├── 三大特性（可见性/原子性/有序性）
│   └── happens-before 8 条规则
├── 指令重排序
│   ├── 编译器重排
│   ├── 处理器重排
│   └── 内存屏障（LoadLoad/StoreStore/LoadStore/StoreLoad）
├── volatile 关键字
│   ├── 可见性保证（MESI 协议）
│   ├── 禁止指令重排序（内存屏障）
│   ├── volatile 不能保证原子性
│   └── 适用场景
└── DCL 单例与 volatile
    ├── double-check locking
    └── 为什么需要 volatile
```

---

## 二、核心知识点

### 2.1 JMM 内存模型

```
┌──────────────────────────────────────────────┐
│                  主内存（Heap）                │
│          ┌──────┐  ┌──────┐  ┌──────┐        │
│          │ var A│  │ var B│  │ var C│        │
│          └──┬───┘  └──┬───┘  └──┬───┘        │
│             │ read/write │         │          │
└─────────────┼────────────┼─────────┼──────────┘
              │            │         │
     ┌────────┴──┐   ┌────┴────┐   ┌┴──────────┐
     │ 工作内存   │   │ 工作内存  │  │ 工作内存    │
     │ (Thread-1) │  │(Thread-2)│  │(Thread-3) │
     │ ┌───────┐  │  │┌───────┐ │  │┌───────┐  │
     │ │ copy A│  │  ││copy B │ │  ││copy C │  │
     │ └───────┘  │  │└───────┘ │  │└───────┘  │
     │  CPU缓存   │  │  CPU缓存  │  │  CPU缓存   │
     └────────────┘  └─────────┘   └────────────┘
```

**核心规则**：
- 每个线程有自己的**工作内存**（CPU 缓存抽象）
- 线程不能直接读写主内存，必须经过工作内存
- 不同线程间无法访问对方工作内存，必须通过主内存传递

### 2.2 并发三大特性

| 特性 | 含义 | 破坏示例 | 保证手段 |
|------|------|---------|---------|
| 可见性 | 一个线程修改，其他线程立即可见 | 工作内存未刷新到主内存 | volatile、synchronized |
| 原子性 | 操作不可分割 | i++ 实际是读-改-写三步 | synchronized、CAS |
| 有序性 | 程序执行顺序符合预期 | 指令重排序 | volatile、happens-before |

---

### 2.3 happens-before 8 条规则

happens-before 不是「时间上先发生」，而是「前一个操作的结果对后一个操作可见」。

| 规则 | 说明 |
|------|------|
| 1. 程序顺序规则 | 同一线程内，前面的操作 happens-before 后面的操作 |
| 2. volatile 变量规则 | volatile 写 happens-before 后续对该变量的读 |
| 3. 锁规则 | unlock happens-before 后续对同一把锁的 lock |
| 4. 传递性 | A happens-before B，B happens-before C → A happens-before C |
| 5. 线程启动规则 | Thread.start() happens-before 该线程的所有操作 |
| 6. 线程终止规则 | 线程所有操作 happens-before Thread.join() 返回 |
| 7. 线程中断规则 | interrupt() happens-before 被中断线程检测到中断 |
| 8. 对象终结规则 | 构造函数执行完 happens-before finalize() |

---

### 2.4 指令重排序

```java
// 可能被重排的代码
int a = 1;    // 语句1
int b = 2;    // 语句2
int c = a + b;// 语句3
// 编译器/CPU 可能将 1、2 重排，但 3 一定在 1、2 之后（数据依赖性）
```

**重排分类**：
- **编译器重排**：编译器在编译期优化指令顺序
- **处理器重排**：CPU 乱序执行（Out-of-Order Execution）

**内存屏障**（Memory Barrier）：

| 屏障类型 | 作用 |
|---------|------|
| LoadLoad | Load1 必须在 Load2 之前完成 |
| StoreStore | Store1 必须在 Store2 之前对其他处理器可见 |
| LoadStore | Load1 必须在 Store2 之前完成 |
| StoreLoad | Store1 必须在 Load2 之前对其他处理器可见（开销最大） |

---

### 2.5 volatile 原理

#### 可见性 —— MESI 缓存一致性协议

```
volatile 写操作：
  1. 将当前 CPU 缓存行数据写回主内存
  2. 通过 MESI 协议通知其他 CPU，将该缓存行标记为 Invalid
  3. 其他 CPU 下次读取时必须从主内存重新加载

volatile 读操作：
  1. 从主内存中加载最新值
  2. JMM 层面：读操作前插入 LoadLoad 屏障
```

#### 禁止重排序 —— 内存屏障插入策略

```
volatile 写前：插入 StoreStore 屏障
volatile 写后：插入 StoreLoad 屏障
volatile 读后：插入 LoadLoad + LoadStore 屏障
```

---

### 2.6 volatile 不能保证原子性（i++ 完整分析）

```java
volatile int count = 0;

// 100 个线程各执行 1000 次 count++
// 结果大概率不是 100000
for (int i = 0; i < 100; i++) {
    new Thread(() -> {
        for (int j = 0; j < 1000; j++) {
            count++; // volatile 保证读到最新值，但读-改-写不是原子操作
        }
    }).start();
}
```

**i++ 的三条指令**：
```
1. getfield    // 从主内存读取 count 的值（volatile 保证读到最新）
2. iadd        // 在工作内存中 +1
3. putfield    // 写回主内存（volatile 保证写后可见）
```

**问题**：线程 A 读到 count=0，线程 B 也读到 count=0（最新值），各自 +1，写回 count=1，丢失一次更新。

**解决**：用 AtomicInteger 或 synchronized 保证原子性。

---

### 2.7 volatile 适用场景

**场景一：状态标志位**

```java
volatile boolean running = true;

// 工作线程
while (running) {
    // 执行任务
}

// 停止线程
running = false; // 可见性保证，工作线程立即可见
```

**场景二：DCL 单例模式**

```java
public class Singleton {
    private static volatile Singleton instance; // 必须 volatile

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {             // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (instance == null) {     // 第二次检查（有锁）
                    instance = new Singleton(); // 问题出在这里
                }
            }
        }
        return instance;
    }
}
```

**DCL 为什么需要 volatile？**

`instance = new Singleton()` 不是原子操作，实际分为三步：
```
1. 分配对象内存空间            memory = allocate()
2. 初始化对象                  ctorInstance(memory)
3. 将 instance 指向内存地址     instance = memory
```

如果发生指令重排序（2 和 3 互换）：
```
线程 A：1 → 3 → 2（还没初始化，但 instance 已经不为 null）
线程 B：第一次检查 instance != null，直接返回一个未初始化的对象 → NPE / 数据错误
```

**volatile 的作用**：禁止 2 和 3 的重排序，确保对象完全初始化后才对其他线程可见。

---

## 三、面试题

### Q1：volatile 能保证线程安全吗？
**答**：不能完全保证。volatile 只保证可见性和有序性，不保证原子性。对于 i++ 这种复合操作，volatile 无法保证线程安全，需要用 AtomicInteger 或 synchronized。

### Q2：volatile 和 synchronized 有什么区别？
**答**：volatile 保证可见性和有序性但不保证原子性，synchronized 三者都保证。volatile 不会阻塞线程，synchronized 会。volatile 只能修饰变量，synchronized 可以修饰方法和代码块。

### Q3：happens-before 是什么？和「时间上先发生」一样吗？
**答**：不一样。happens-before 是 JMM 定义的可见性保证规则——A happens-before B 意味着 A 的结果对 B 可见。时间上先发生的操作不一定 happens-before 后续操作。

### Q4：为什么 DCL 单例需要 volatile？
**答**：`new Singleton()` 会被指令重排序为「分配内存→引用赋值→初始化对象」，其他线程可能拿到未初始化的对象。volatile 禁止重排序，保证对象完全初始化后才可见。

### Q5：volatile 的底层实现原理？
**答**：写操作时，JVM 插入 StoreStore + StoreLoad 内存屏障，并将缓存行写回主内存，通过 MESI 协议使其他 CPU 缓存失效。读操作时，插入 LoadLoad + LoadStore 屏障，从主内存读取最新值。

### Q6：什么场景下应该用 volatile？
**答**：(1) 状态标志位（boolean running）；(2) DCL 单例模式；(3) 一次性写入的配置值。核心原则：只有「一个线程写、多个线程读」的简单场景才适合。

### Q7：MESI 协议了解吗？
**答**：MESI 是 CPU 缓存一致性协议，定义了缓存行的四种状态：Modified（已修改）、Exclusive（独占）、Shared（共享）、Invalid（无效）。volatile 写操作通过使其他 CPU 缓存行 Invalid 来保证可见性。

### Q8：JMM 和 JVM 内存结构（堆栈方法区）有什么关系？
**答**：两者是不同概念。JMM 是并发编程的抽象模型（主内存和工作内存），描述线程间如何交互。JVM 内存结构是运行时数据区划分（堆、栈、方法区），描述内存如何分配。没有直接关系。

---

## 四、常见误区

| 误区 | 正确理解 |
|------|---------|
| "volatile 保证线程安全" | 只保证可见性和有序性，不保证原子性 |
| "volatile 变量操作都是原子的" | 只有单次读/写是原子的，i++ 不是 |
| "happens-before 就是时间先后" | 是可见性规则，不是时序关系 |
| "synchronized 也用 MESI" | synchronized 基于 Monitor，volatile 基于 MESI |
| "DCL 不用 volatile 也行" | 指令重排序可能导致获取到未初始化的对象 |

---

## 五、实战场景

1. **停止线程标志**：`volatile boolean running` 配合 while 循环
2. **DCL 单例**：Spring Bean 默认单例，框架层面大量使用
3. **配置热更新**：后台线程修改 volatile 配置变量，工作线程实时感知

---

## 六、关联知识链接

- [10-线程基础](10-线程基础.md) — 线程创建与生命周期
- [12-synchronized与CAS](12-synchronized与CAS.md) — 原子性的保证手段
- [13-AQS与Lock](13-AQS与Lock.md) — AQS 中 volatile state 的应用
- [15-并发容器](15-并发容器.md) — ConcurrentHashMap 中的 volatile 运用
