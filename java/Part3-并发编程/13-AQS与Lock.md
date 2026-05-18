
> 并发框架核心，源码级考点

---

## 一、知识树

```
AQS 与 Lock
├── AQS 核心设计
│   ├── volatile int state（同步状态）
│   ├── CLH 双向队列（FIFO 等待队列）
│   └── 模板方法设计模式
├── 独占模式 vs 共享模式
├── ReentrantLock
│   ├── 公平锁 vs 非公平锁（源码差异）
│   ├── lock() 流程
│   └── unlock() 流程
├── Condition 条件变量
├── ReentrantReadWriteLock
│   └── 读写锁（读读共享/读写互斥/写写互斥）
└── StampedLock（JDK8 乐观读）
```

---

## 二、核心知识点

### 2.1 AQS 核心设计

```
AbstractQueuedSynchronizer
    │
    ├── volatile int state          // 同步状态（0=未锁，>0=已锁/重入次数）
    │
    └── CLH 双向队列（Node 节点链表）
          ┌─────┐    ┌─────┐    ┌─────┐
   head → │Node1│ ←→ │Node2│ ←→ │Node3│ ← tail
          └─────┘    └─────┘    └─────┘
          (线程A)     (线程B)     (线程C)
          每个Node包含：
          - Thread thread       // 等待的线程
          - int waitStatus      // CANCELLED/SIGNAL/CONDITION/PROPAGATE
          - Node prev / next    // 前驱/后继
          - Node nextWaiter     // Condition 队列后继
```

**AQS 三大核心**：
1. **volatile int state**：表示同步状态，子类通过 CAS 修改它来获取/释放锁
2. **CLH 双向队列**：存储等待获取锁的线程，FIFO 排队
3. **模板方法模式**：AQS 定义骨架，子类实现 tryAcquire/tryRelease 等钩子方法

---

### 2.2 AQS 模板方法设计模式

```java
// AQS 定义的模板方法（子类直接使用）
public final void acquire(int arg)       // 独占获取锁
public final void release(int arg)       // 独占释放锁
public final void acquireShared(int arg) // 共享获取锁
public final void releaseShared(int arg) // 共享释放锁

// 子类需要重写的钩子方法
protected boolean tryAcquire(int arg)          // 尝试独占获取
protected boolean tryRelease(int arg)          // 尝试独占释放
protected int tryAcquireShared(int arg)        // 尝试共享获取
protected boolean tryReleaseShared(int arg)    // 尝试共享释放
protected boolean isHeldExclusively()          // 是否被当前线程独占
```

**基于 AQS 的实现类**：
- ReentrantLock → 独占模式
- ReentrantReadWriteLock → 读共享 + 写独占
- Semaphore → 共享模式
- CountDownLatch → 共享模式

---

### 2.3 独占模式 vs 共享模式

| 模式 | 特点 | 代表 |
|------|------|------|
| 独占模式 | 同一时间只有一个线程能获取 | ReentrantLock |
| 共享模式 | 多个线程可同时获取 | Semaphore、CountDownLatch |
| 组合模式 | 读写分离 | ReentrantReadWriteLock |

---

### 2.4 ReentrantLock：公平锁 vs 非公平锁

```java
// 非公平锁（默认）—— NonfairSync
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 直接 CAS 抢锁，不管队列中有没有等待线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // 可重入：state + 1
        int nextc = c + acquires;
        setState(nextc);
        return true;
    }
    return false;
}

// 公平锁 —— FairSync
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 关键区别：先检查队列中是否有等待线程
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        setState(nextc);
        return true;
    }
    return false;
}
```

**核心差异**：非公平锁直接 CAS 抢锁；公平锁先调用 `hasQueuedPredecessors()` 检查队列中是否有先来的线程在等待。

---

### 2.5 lock() 完整流程

```
lock() 调用
    │
    ▼
tryAcquire() ──成功──→ 获取锁，返回
    │
    失败
    ▼
addWaiter()：将当前线程包装成 Node 加入 CLH 队列尾部（CAS 设置 tail）
    │
    ▼
acquireQueued()：在队列中自旋
    │
    ├── 前驱是 head？ ──是──→ tryAcquire() 再次尝试
    │                          │
    │                     成功 → 出队，成为新 head
    │                     失败 → shouldParkAfterFailedAcquire()
    │
    └── 前驱不是 head → shouldParkAfterFailedAcquire()
                              │
                              ▼
                        LockSupport.park(this) // 挂起线程
                        等待被唤醒（unpark）
```

---

### 2.6 unlock() 完整流程

```
unlock() 调用
    │
    ▼
tryRelease()：state - 1
    │
    ├── state != 0 → 还没完全释放（重入锁），返回 false
    │
    └── state == 0 → 完全释放
            │
            ▼
        setExclusiveOwnerThread(null)
        setState(0)
            │
            ▼
        unparkSuccessor()：唤醒后继节点
            │
            ├── 后继节点有效 → LockSupport.unpark(node.thread)
            │
            └── 后继节点无效（cancelled）→ 从尾部向前找最前面的有效节点唤醒
```

---

### 2.7 Condition 条件变量

```java
ReentrantLock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();

// 生产者
lock.lock();
try {
    while (queue.isFull()) {
        notFull.await();   // 释放锁，进入 notFull 等待队列
    }
    queue.add(item);
    notEmpty.signal();     // 唤醒 notEmpty 等待队列中的一个线程
} finally {
    lock.unlock();
}

// 消费者
lock.lock();
try {
    while (queue.isEmpty()) {
        notEmpty.await();  // 释放锁，进入 notEmpty 等待队列
    }
    Item item = queue.take();
    notFull.signal();      // 唤醒 notFull 等待队列中的一个线程
} finally {
    lock.unlock();
}
```

| 对比 | Object wait/notify | Condition await/signal |
|------|-------------------|----------------------|
| 条件数量 | 1 个（一个 Monitor） | 多个（多个 Condition） |
| 使用前提 | synchronized 块内 | lock.lock() 之后 |
| 精确唤醒 | notify 随机唤醒 | signal 唤醒对应 Condition |
| 底层 | Monitor 的 WaitSet | AQS 的 ConditionQueue |

---

### 2.8 ReentrantReadWriteLock

```
                读写锁规则
  ┌──────────┬──────────┬──────────┐
  │          │  读锁获取  │  写锁获取  │
  ├──────────┼──────────┼──────────┤
  │ 读锁持有  │   允许    │   阻塞    │
  │ 写锁持有  │   阻塞    │   阻塞    │
  └──────────┴──────────┴──────────┘
  简记：读读不互斥，读写互斥，写写互斥
```

**state 的拆分**：
```
state 是 32 位 int：
  高 16 位：读锁持有计数（共享锁）
  低 16 位：写锁持有计数（独占锁）
```

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
ReentrantReadWriteLock.ReadLock readLock = rwLock.readLock();
ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();

// 读操作
readLock.lock();
try { /* 读取数据 */ } finally { readLock.unlock(); }

// 写操作
writeLock.lock();
try { /* 修改数据 */ } finally { writeLock.unlock(); }
```

**注意**：写锁可以降级为读锁（获取写锁→获取读锁→释放写锁），但读锁不能升级为写锁（会死锁）。

---

### 2.9 StampedLock（JDK8 乐观读）

```java
StampedLock sl = new StampedLock();

// 乐观读（不加锁，性能最高）
long stamp = sl.tryOptimisticRead();    // 获取乐观读戳
int value = sharedData;                  // 读取数据
if (!sl.validate(stamp)) {              // 验证读期间是否有写操作
    stamp = sl.readLock();              // 升级为悲观读锁
    try {
        value = sharedData;
    } finally {
        sl.unlockRead(stamp);
    }
}

// 写锁
long stamp = sl.writeLock();
try {
    sharedData = newValue;
} finally {
    sl.unlockWrite(stamp);
}
```

**StampedLock vs ReentrantReadWriteLock**：
- StampedLock 支持乐观读，读不阻塞写
- StampledLock 不可重入
- StampledLock 不支持 Condition

---

## 三、面试题

### Q1：AQS 是什么？核心组成部分？
```
**答**：AQS 是 AbstractQueuedSynchronizer，JUC 包下并发工具的基石。核心由 volatile int state（同步状态）+ CLH 双向队列（等待线程队列）组成。通过模板方法模式，子类只需实现 tryAcquire/tryRelease 等钩子方法。
```
### Q2：ReentrantLock 公平锁和非公平锁有什么区别？
```
**答**：非公平锁在 tryAcquire 时直接 CAS 抢锁，不管等待队列；公平锁先调用 hasQueuedPredecessors() 检查队列中是否有先到的线程在等待。非公平锁吞吐量更高但可能饿死线程，公平锁按 FIFO 但吞吐量略低。
```
### Q3：AQS 的 CLH 队列是什么？为什么要用双向队列？
```
**答**：CLH 是一种基于链表的自旋锁队列。AQS 用双向队列因为：每个节点需要知道前驱节点状态来决定是否 park；释放锁时需要从尾部向前查找有效后继节点来唤醒。
```
### Q4：ReentrantLock 的 lock() 过程？
```
**答**：(1) tryAcquire 尝试 CAS 修改 state；(2) 成功则获取锁；(3) 失败则 addWaiter 将线程包装成 Node 入队；(4) acquireQueued 在队列中自旋，如果前驱是 head 则再次 tryAcquire；(5) 否则 LockSupport.park 挂起。
```
### Q5：Condition 和 Object 的 wait/notify 有什么区别？
```
**答**：Condition 支持多个等待队列（多条件变量），可以精确唤醒某个条件的线程；Object 只有一个 WaitSet，notify 唤醒哪个线程不确定。Condition 必须配合 Lock 使用。
```
### Q6：读写锁是什么？什么场景用？
```
**答**：ReentrantReadWriteLock 分读锁和写锁，读读不互斥、读写互斥、写写互斥。适合读多写少场景（如缓存）。写锁可以降级为读锁，但读锁不能升级为写锁。
```
### Q7：StampedLock 是什么？和 ReadWriteLock 有什么区别？
```
答：JDK8 引入，支持乐观读——读操作不阻塞写操作，通过 validate 验证读期间是否被写。性能比 ReadWriteLock 更高，但不可重入、不支持 Condition。
```
### Q8：为什么非公平锁性能更好？
```
答：线程释放锁后，新来的线程直接 CAS 抢锁，避免了「唤醒挂起线程→线程调度→线程运行」的延迟。线程切换开销大，直接让当前运行的线程获取锁效率更高。
```

### Q9：什么是死锁？产生死锁的四个必要条件是什么？
```
死锁就是多个线程相互争夺锁，而进行永久等待阻塞的场景

必要条件：
		1，互斥条件，资源只能被一个线程占有
		2，不剥夺条件，资源不能被强行剥夺
		3，请求与保持条件，线程持有资源的同时又在请求其他资源
		4，循环等待，线程间形成资源请求的循环链
		
```

---

## 四、常见误区

| 误区 | 正确理解 |
|------|---------|
| "AQS 就是 ReentrantLock" | AQS 是框架，ReentrantLock 只是 AQS 的一种实现 |
| "公平锁就是 FIFO 执行" | 公平锁是 FIFO 获取锁，但执行顺序还受 CPU 调度影响 |
| "读写锁读完全不影响写" | 读写互斥，有读锁时写锁必须等 |
| "StampedLock 可以替代 ReadWriteLock" | StampedLock 不可重入、不支持 Condition，各有适用场景 |
| "CLH 队列就是单向链表" | AQS 的 CLH 是双向队列（有 prev 和 next） |

---

## 五、实战场景

1. **生产者-消费者**：ReentrantLock + 多 Condition 实现精确唤醒
2. **缓存系统**：ReentrantReadWriteLock，读多写少场景
3. **限流**：Semaphore 基于 AQS 共享模式
4. **高性能计数**：StampedLock 乐观读，适合读远多于写的场景

---

## 六、关联知识链接

- [12-synchronized与CAS](12-synchronized与CAS.md) — CAS 是 AQS 的底层操作
- [14-线程池](14-线程池.md) — 线程池中 Worker 使用 AQS
- [17-并发工具类](17-并发工具类.md) — CountDownLatch/Semaphore 基于 AQS
- [11-JMM与volatile](11-JMM与volatile.md) — AQS state 用 volatile 修饰
