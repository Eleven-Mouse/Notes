
>  锁机制的核心，源码级理解

---

## 一、知识树

```
synchronized 与 CAS
├── synchronized
│   ├── 三种用法（实例方法/静态方法/代码块）
│   ├── 底层原理（monitorenter/monitorexit）
│   ├── Monitor 对象结构
│   └── 锁升级（偏向锁→轻量级锁→重量级锁）
├── synchronized vs ReentrantLock 对比
├── CAS 原理
│   ├── CPU 指令 cmpxchg
│   ├── Unsafe 类
│   └── 三大问题（ABA/自旋/单变量）
├── 原子类
│   ├── AtomicInteger / AtomicLong / AtomicReference
│   ├── AtomicStampedReference（解决 ABA）
│   └── LongAdder（分段 CAS）
└── 锁升级（JDK6+）
```

---

## 二、核心知识点

### 2.1 synchronized 三种用法

```java
// 1. 实例方法 —— 锁的是当前实例对象 this
public synchronized void method() {
    // 同一时间只有一个线程能访问同一个实例的此方法
}

// 2. 静态方法 —— 锁的是当前类的 Class 对象
public static synchronized void staticMethod() {
    // 同一时间只有一个线程能访问此静态方法（跨实例）
}

// 3. 代码块 —— 锁的是指定对象
synchronized (lockObj) {
    // 锁的是 lockObj 对象
}
```

| 用法 | 锁对象 | 作用范围 |
|------|--------|---------|
| 实例方法 | this（当前实例） | 同一实例的所有 synchronized 方法互斥 |
| 静态方法 | Class 对象 | 所有实例的该静态方法互斥 |
| 代码块(synchronized(obj)) | obj 指定对象 | 只在代码块内互斥 |

---

### 2.2 底层原理：字节码层面

**同步代码块**：
```java
synchronized (obj) {
    // 临界区
}
```

**对应字节码：**
```
monitorenter      // 获取 obj 的 Monitor 锁
// 临界区代码
monitorexit       // 正常退出释放锁
...
monitorexit       // 异常退出释放锁（编译器自动生成）
```

**同步方法**：
```
// 方法访问标志中设置 ACC_SYNCHRONIZED
// JVM 调用时自动获取/释放 Monitor，不需要字节码指令
```

---

### 2.3 Monitor 对象结构

```
┌───────────────────────────┐
│       Monitor 对象         │
│                           │
│  ┌─────────┐              │
│  │  Owner  │ 当前持有锁的线程 │
│  └─────────┘              │
│                           │
│  ┌──────────────────┐     │
│  │   EntryList      │ 阻塞等待获取锁的线程队列 │
│  │  (阻塞队列)       │     │
│  └──────────────────┘     │
│                           │
│  ┌──────────────────┐     │
│  │   WaitSet        │ 调用 wait() 等待的线程集合 │
│  │  (等待队列)       │     │
│  └──────────────────┘     │
│                           │
│  count: 重入计数            │
└───────────────────────────┘
```

**流程**：
1. 线程进入 → CAS 修改 Owner 为自己 → count+1
2. 其他线程 CAS 失败 → 进入 EntryList 阻塞
3. 持有锁线程调用 wait() → 释放锁，进入 WaitSet
4. 持有锁线程执行完 → count-1 → 归零时释放锁 → 唤醒 EntryList 或 WaitSet 中的线程

---

### 2.4 synchronized vs ReentrantLock

| 对比维度 | synchronized | ReentrantLock |
|---------|-------------|---------------|
| 实现层面 | JVM 内置（关键字） | JDK API（AQS 实现） |
| 锁获取 | 自动获取释放 | 手动 lock/unlock |
| 可中断 | 不可中断 | lockInterruptibly() 可中断 |
| 公平性 | 非公平锁 | 可选公平/非公平 |
| 条件变量 | 一个（wait/notify） | 多个（Condition） |
| 超时获取 | 不支持 | tryLock(timeout) 支持 |
| 可重入 | 是 | 是 |
| 异常处理 | 自动释放锁 | 必须在 finally 中 unlock |

---

### 2.5 CAS 原理（Compare And Swap）

```
CAS(V, Expected, New)
  if (V == Expected) {
      V = New;    // 更新成功
      return true;
  } else {
      return false; // 更新失败，重试（自旋）
  }
```

**CPU 指令级别**：`cmpxchg` 指令（x86 架构），是一条**原子指令**，在硬件层面保证比较和交换的原子性。

```java
// AtomicInteger 的 incrementAndGet 源码简化
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// Unsafe#getAndAddInt
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset); // 读取最新值
    } while (!compareAndSwapInt(o, offset, v, v + delta)); // CAS 自旋
    return v;
}
```

---

### 2.6 Unsafe 类

```java
// Unsafe 提供 CAS 原子操作（native 方法）
public final native boolean compareAndSwapInt(Object o, long offset,
                                               int expected, int x);
// 作用：
// 1. 直接操作内存（allocateMemory/putInt 等）
// 2. CAS 原子操作
// 3. 线程挂起/唤醒（park/unpark）
// 4. 内存屏障（loadFence/storeFence）
```

---

### 2.7 CAS 的三大问题

| 问题 | 说明 | 解决方案 |
|------|------|---------|
| **ABA 问题** | 值从 A→B→A，CAS 认为没变 | AtomicStampedReference（版本号） |
| **自旋开销** | CAS 失败不断重试，消耗 CPU | LongAdder（分散竞争点） |
| **只能单变量** | 一次只能 CAS 一个变量 | AtomicReference（封装多字段） |

**ABA 问题详细分析**：
```java
// ABA 问题示例
AtomicInteger ai = new AtomicInteger(100);

// 线程1
int old = ai.get();            // 100
// 线程2 在线程1 操作前：100 → 200 → 100
ai.compareAndSet(100, 200);
ai.compareAndSet(200, 100);
// 线程1 继续执行
ai.compareAndSet(100, 999);    // 成功！但值已经被改过

// 解决：AtomicStampedReference（带版本号）
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(100, 1);
int[] stampHolder = new int[1];
Integer old = ref.get(stampHolder);
int oldStamp = stampHolder[0];
ref.compareAndSet(old, 999, oldStamp, oldStamp + 1); // 版本号也要匹配
```

---

### 2.8 原子类概览

| 类别 | 类名 | 说明 |
|------|------|------|
| 基本类型 | AtomicInteger | int 原子操作 |
| 基本类型 | AtomicLong | long 原子操作 |
| 基本类型 | AtomicBoolean | boolean 原子操作 |
| 引用类型 | AtomicReference | 对象引用原子操作 |
| 引用类型 | AtomicStampedReference | 带版本号，解决 ABA |
| 数组类型 | AtomicIntegerArray | int 数组原子操作 |
| 字段更新器 | AtomicIntegerFieldUpdater | 对已有类的 volatile 字段原子更新 |
| 累加器 | LongAdder | 高并发累加，优于 AtomicLong |
| 累加器 | LongAccumulator | 自定义累加函数 |

---

### 2.9 LongAdder 原理（分段 CAS）

```
AtomicLong：所有线程 CAS 竞争同一个 value
  ┌────────┐
  │ value  │ ← 线程1 CAS、线程2 CAS、线程3 CAS ... 全部竞争
  └────────┘

LongAdder：分散到多个 Cell，减少竞争
  ┌──────┐
  │ base │ ← 无竞争时直接 CAS
  └──────┘
  ┌──────┬──────┬──────┬──────┐
  │Cell 0│Cell 1│Cell 2│Cell 3│ ← 有竞争时，各线程 CAS 不同 Cell
  └──────┴──────┴──────┴──────┘
  sum() = base + Cell[0] + Cell[1] + ... + Cell[n]
```

**核心思想**：空间换时间。将一个 value 分散到多个 Cell 中，每个线程 CAS 不同的 Cell，减少竞争。最终求和时累加所有 Cell。

**适用场景**：高并发下的计数统计（如统计接口 QPS），不需要精确的实时值。

---

## 三、面试题

### Q1：synchronized 的底层原理？
>**答**：同步代码块通过 monitorenter/monitorexit 字节码指令实现，同步方法通过 ACC_SYNCHRONIZED 标志位实现。底层都依赖 Monitor 对象，包含 Owner（持有者）、EntryList（阻塞队列）、WaitSet（等待队列）。

### Q2：synchronized 和 ReentrantLock 的区别？
>**答**：synchronized 是 JVM 内置关键字，自动获取释放锁；ReentrantLock 是 JDK API，需手动 lock/unlock。ReentrantLock 支持可中断、超时获取、公平锁、多条件变量，功能更强但使用更复杂。

### Q3：CAS 是什么？有什么问题？
>**答**：CAS（Compare And Swap）是比较并交换，通过 CPU 原子指令 cmpxchg 实现。三大问题：ABA 问题（用 AtomicStampedReference 解决）、自旋开销（高并发下 CPU 空转）、只能操作单变量。

### Q4：什么是 ABA 问题？怎么解决？
>**答**：值从 A 变为 B 再变为 A，CAS 认为没变过。解决方法是用 AtomicStampedReference，每次更新带一个版本号，CAS 时同时比较值和版本号。

### Q5：LongAdder 和 AtomicLong 有什么区别？
>**答**：AtomicLong 所有线程 CAS 竞争同一个变量，高并发时自旋开销大。LongAdder 采用分段 CAS，将值分散到多个 Cell，各线程竞争不同 Cell，最终求和。高并发写场景 LongAdder 性能远优于 AtomicLong。

### Q6：synchronized 是可重入锁吗？怎么实现的？
>**答**：是的。Monitor 对象有 count 计数器，同一线程每次获取锁 count+1，每次释放 count-1，归零才真正释放。所以同一个线程可以重复进入自己持有锁的同步代码。

### Q7：Unsafe 类了解吗？为什么叫 Unsafe？
>**答**：Unsafe 提供 CAS、内存操作、线程挂起等底层 native 方法。叫 Unsafe 是因为直接操作内存，使用不当会导致内存泄漏、数据损坏、甚至 JVM 崩溃。JDK9+ 用 VarHandle 逐步替代。

### Q8：乐观锁和悲观锁的区别？
>**答**：悲观锁（synchronized）先加锁再操作，适合写多场景。乐观锁（CAS）先操作再检查冲突，适合读多写少场景。CAS 是乐观锁的实现方式。

### Q9: ynchronized 和 volatile 的区别
>

### Q10：如果一个共享变量被 volatile 修饰，同时用 synchronized 修饰对它的操作方法，这样的组合有意义吗？能解决什么问题？
>

### Q11：synchronized 在 JDK1.6 之后做了哪些优化，让它从 “重量级锁” 变得没那么重吗？
>JDK1.6 给 synchronized 加了锁升级机制，从无锁开始，先升级为偏向锁，只允许第一个获取锁的线程反复进入；如果有线程竞争，就升级为轻量级锁，用 CAS 自旋尝试获取锁，避免直接进入内核态；只有自旋失败，才会升级为重量级锁。这样大部分低竞争场景下，锁都不用走到重量级那一步，性能提升很多。






----

## 四、常见误区

| 误区 | 正确理解 |
|------|---------|
| "synchronized 效率很低" | JDK6 引入锁升级后性能大幅提升 |
| "CAS 没有开销" | 高并发自旋会消耗 CPU |
| "AtomicLong 线程安全就够用" | 高并发场景 LongAdder 性能远超 |
| "ABA 不会造成问题" | 某些场景（如链表操作）ABA 会导致严重错误 |
| "synchronized 和 Lock 功能一样" | Lock 支持可中断、超时、公平锁等高级特性 |

---

## 五、实战场景

1. **计数器**：接口调用次数统计 → LongAdder
2. **库存扣减**：CAS 乐观锁防超卖 → AtomicInteger.compareAndSet
3. **并发控制**：限流/互斥 → synchronized 或 ReentrantLock
4. **ABA 检测**：链表/pop 操作 → AtomicStampedReference

---

## 六、关联知识链接

- [11-JMM与volatile](./11-JMM与volatile.md) — synchronized 保证可见性的原理
- [13-AQS与Lock](./13-AQS与Lock.md) — ReentrantLock 的 AQS 实现原理
- [15-并发容器](./15-并发容器.md) — ConcurrentHashMap 中的 CAS 与 synchronized
- [14-线程池](./14-线程池.md) — 线程池中的 CAS 应用
