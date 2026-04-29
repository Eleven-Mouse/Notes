
> 优先级：⭐⭐⭐⭐⭐ | 面试频率：🔥🔥🔥🔥🔥 | 内存泄漏是必考题

---

## 一、知识树

```
ThreadLocal
├── 作用与原理
│   ├── 线程隔离
│   └── Thread → ThreadLocalMap → Entry[]
├── get/set/remove 流程
├── 内存泄漏分析
│   ├── 为什么 key 用弱引用
│   ├── 为什么还是会泄漏
│   └── 解决：必须 remove()
├── 哈希冲突处理（开放定址法）
├── 使用场景
├── InheritableThreadLocal
└── TransmittableThreadLocal（TTL）
```

---

## 二、核心知识点

### 2.1 ThreadLocal 的作用

**核心**：每个线程维护自己的独立变量副本，线程间互不影响，实现线程隔离。

```java
// 每个线程有自己的 userId，互不干扰
private static ThreadLocal<String> currentUser = ThreadLocal.withInitial(() -> "guest");

// 线程A
currentUser.set("user_001");
String user = currentUser.get(); // "user_001"

// 线程B
currentUser.set("user_002");
String user = currentUser.get(); // "user_002"
// 线程A不受影响，仍然拿到 "user_001"
```

**面试官视角**：ThreadLocal 解决的不是共享变量的同步问题，而是**避免共享**——每个线程有自己的副本，天然线程安全。

---

### 2.2 底层原理：存储结构

```
Thread 对象
    │
    ├── ThreadLocal.ThreadLocalMap threadLocals
    │       │
    │       └── Entry[] table
    │               │
    │               ├── Entry extends WeakReference<ThreadLocal<?>>
    │               │       │
    │               │       ├── key   = ThreadLocal 对象（弱引用）←┐
    │               │       └── value = 线程私有数据（强引用）      │
    │               │                                            │
    │               └── ... 更多 Entry                            │
    │                                                             │
    └── ...                                                       │
                                                                  │
ThreadLocal 对象 ───────────────弱引用──────────────────────────────┘
（外部强引用断开后，key 可以被 GC 回收）
```

**关键理解**：
- ThreadLocalMap 是 Thread 的成员变量，不是 ThreadLocal 的
- 每个 Thread 有自己的 ThreadLocalMap
- ThreadLocalMap 的 key 是 ThreadLocal 对象（弱引用），value 是实际数据（强引用）

---

### 2.3 ThreadLocal  get/set/remove 流程

#### set 流程
```
set(value)
    │
    ▼
  获取当前线程 Thread.currentThread()
    │
    ▼
  获取线程的 ThreadLocalMap（threadLocals）
    │
    ├── Map 不为空 → 以 this（ThreadLocal对象）为 key，value 为值存入
    │                  如果 hash 冲突 → 线性探测法找下一个空位
    │
    └── Map 为空 → createMap()，创建新 ThreadLocalMap 并赋值
```

#### get 流程
```
get()
    │
    ▼
  获取当前线程的 ThreadLocalMap
    │
    ├── Map 不为空 → 以 this 为 key 查找 Entry
    │                  ├── 找到 → 返回 value
    │                  └── 找不到 → setInitialValue()（返回初始值并存储）
    │
    └── Map 为空 → setInitialValue()
```

#### remove 流程
```
remove()
    │
    ▼
  获取当前线程的 ThreadLocalMap
    │
    ├── Map 不为空 → 以 this 为 key 找到 Entry
    │                  → Entry.clear()（将 key 置为 null）
    │                  → 清理该 Entry（value 也置为 null）
    │
    └── Map 为空 → 无操作
```

---

### 2.4 内存泄漏原因完整分析（高频考点）

#### 第一步：为什么 key 用弱引用？

```java
// 如果 key 是强引用：
ThreadLocal tl = new ThreadLocal();
tl = null; // 外部强引用断开
// 但 ThreadLocalMap 中的 key 仍然强引用 ThreadLocal 对象
// → ThreadLocal 对象无法被 GC 回收 → 内存泄漏
// 所以 key 设计为弱引用（WeakReference）
```

#### 第二步：为什么弱引用还是会泄漏？

```
场景：线程池中的线程（线程不会销毁，长期存活）

1. ThreadLocal 外部引用断开：tl = null
2. GC 发生：key（ThreadLocal 弱引用）被回收 → key = null
3. 但 value 仍然是强引用！且线程不销毁 → ThreadLocalMap 不释放
4. Entry 中：key=null, value=大对象 → value 无法被 GC 回收
5. 如果有大量这样的 Entry → 内存泄漏！

  Entry[]
  ┌─────────────────────────────────────┐
  │ Entry(key=null, value=大对象A)       │ ← key 被 GC 回收了
  │ Entry(key=null, value=大对象B)       │ ← value 仍在强引用
  │ Entry(key=TL3, value=数据C)         │
  └─────────────────────────────────────┘
```

#### 第三步：ThreadLocal 的自清理机制（不够可靠）

ThreadLocal 在 get/set/remove 时会顺便清理 key=null 的 Entry（称为「启发式清理」）。但如果不调用这些方法，Entry 就一直存在。

#### 第四步：解决方案

```java
// 必须在 finally 中 remove
ThreadLocal<User> currentUser = new ThreadLocal<>();
try {
    currentUser.set(user);
    // 业务逻辑
} finally {
    currentUser.remove(); // 关键！清除 Entry，防止泄漏
}
```

---

### 2.5 哈希冲突处理：开放定址法

```java
// ThreadLocalMap 使用开放定址法（线性探测），不是 HashMap 的链表法

// hash 冲突时：
int i = key.threadLocalHashCode & (len - 1); // 计算初始位置
// 如果 table[i] 已被占用 → i = nextIndex(i, len) → 继续往后找空位
// 直到找到空位或 key 相同的 Entry
```

| 对比 | ThreadLocalMap | HashMap |
|------|---------------|---------|
| 冲突处理 | 开放定址法（线性探测） | 链表法（拉链法） |
| 负载因子 | 默认 2/3（len * 2/3） | 默认 0.75 |
| 扩容 | 容量翻倍 + rehash | 容量翻倍 + 重新分布 |

**为什么用开放定址法？** ThreadLocalMap 中 Entry 数量通常较少（一个线程用不了多少 ThreadLocal），开放定址法在小数据量下缓存命中率更高。

---

### 2.6 ThreadLocal 使用场景

| 场景 | 说明 |
|------|------|
| 数据库连接 | 每个线程独立的 Connection，保证事务一致性 |
| 用户上下文 | 存储当前登录用户信息，全链路传递 |
| 日期格式化 | SimpleDateFormat 非线程安全，每个线程独立一份 |
| 事务管理 | Spring 事务用 ThreadLocal 存储当前事务状态 |
| 链路追踪 | 存储_traceId，日志中串联同一请求 |

```java
// 经典场景：Spring 的事务管理
// TransactionSynchronizationManager 用 ThreadLocal 存储当前事务资源
private static final ThreadLocal<Map<Object, Object>> resources =
    new NamedThreadLocal<>("Transactional resources");
```

---

### 2.7 InheritableThreadLocal（父子线程传递）

```java
// 父线程的数据自动传递给子线程
InheritableThreadLocal<String> parentTL = new InheritableThreadLocal<>();
parentTL.set("父线程数据");

new Thread(() -> {
    String value = parentTL.get(); // "父线程数据" —— 自动继承
}).start();
```

**原理**：Thread 创建时，如果父线程的 inheritableThreadLocals 不为空，会将其复制到子线程。

**问题**：线程池中线程复用时，InheritableThreadLocal 的值是创建线程时复制的，后续父线程修改不会同步到子线程。

---

### 2.8 TransmittableThreadLocal（阿里 TTL）

```java
// 解决线程池场景下 ThreadLocal 值传递问题
TransmittableThreadLocal<String> ttl = new TransmittableThreadLocal<>();
ttl.set("主线程数据");

// 用 TTL 包装的线程池
ExecutorService executor = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(4));

executor.submit(() -> {
    String value = ttl.get(); // "主线程数据" —— 正确传递
    // 即使线程被复用，也能拿到最新的值
});
```

**原理**：在任务提交时捕获当前线程的 ThreadLocal 快照，在任务执行前恢复快照，执行后还原。

---

## 三、面试题

### Q1：ThreadLocal 是什么？原理？
**答**：ThreadLocal 实现线程数据隔离，每个线程有自己的变量副本。底层是每个 Thread 对象持有 ThreadLocalMap，以 ThreadLocal 对象为 key、线程私有数据为 value 存储。

### Q2：ThreadLocal 为什么会内存泄漏？怎么解决？
**答**：ThreadLocalMap 的 key 是弱引用，GC 时 key 被回收变为 null，但 value 仍是强引用。如果线程长期存活（如线程池），value 无法被回收导致泄漏。解决：使用完后在 finally 中调用 remove()。

### Q3：ThreadLocal 的 key 为什么用弱引用？
**答**：如果 key 是强引用，当 ThreadLocal 外部引用断开后，ThreadLocalMap 中的引用仍然存在，ThreadLocal 对象本身无法被 GC 回收。弱引用使得 ThreadLocal 对象在外部引用断开后可以被 GC 回收。

### Q4：ThreadLocalMap 用什么解决哈希冲突？
**答**：开放定址法（线性探测）。与 HashMap 的链表法不同，冲突时向后找下一个空位。适合 Entry 数量少的场景，缓存友好。

### Q5：ThreadLocal 使用场景有哪些？
**答**：(1) 数据库连接管理；(2) 用户上下文传递（登录信息）；(3) SimpleDateFormat 线程安全；(4) Spring 事务管理；(5) 链路追踪 traceId。

### Q6：InheritableThreadLocal 和 ThreadLocal 的区别？
**答**：InheritableThreadLocal 在创建子线程时会将父线程的值复制给子线程。但在线程池场景下，线程被复用，创建时复制的值不会随父线程更新。阿里 TTL 框架解决了这个问题。

### Q7：ThreadLocal 和 synchronized 都能解决并发问题，有什么区别？
**答**：synchronized 是「共享加锁」，多个线程共享同一变量但互斥访问。ThreadLocal 是「避免共享」，每个线程有自己的副本，根本不需要同步。思路完全不同。

### Q8：线程池中使用 ThreadLocal 有什么注意事项？
**答**：(1) 必须在 finally 中 remove()，否则线程复用时数据错乱且内存泄漏；(2) InheritableThreadLocal 在线程池中不可靠，用 TTL 替代；(3) 注意 ThreadLocal 数量不宜过多（影响 ThreadLocalMap 性能）。

---

## 四、常见误区

| 误区 | 正确理解 |
|------|---------|
| "ThreadLocal 是用来做同步的" | 是做线程隔离，避免共享 |
| "remove() 可有可无" | 不 remove 必然泄漏（线程池场景） |
| "key 用了弱引用就安全了" | key 回收后 value 仍是强引用，照样泄漏 |
| "InheritableThreadLocal 能在线程池中传递" | 只在创建时复制，线程池复用时不更新 |
| "ThreadLocal 没有并发问题" | ThreadLocalMap 本身非线程安全，但每个线程独有一份所以不需要同步 |
| "ThreadLocalMap 用链表法" | 用开放定址法（线性探测） |

---

## 五、实战场景

1. **Spring 事务**：@Transactional 底层用 ThreadLocal 存储 Connection，保证同一线程内多次 DAO 操作使用同一 Connection
2. **用户上下文**：拦截器将用户信息存入 ThreadLocal，Controller/Service/DAO 全链路可用
3. **SimpleDateFormat**：`ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"))`
4. **链路追踪**：请求入口生成 traceId 存入 ThreadLocal，日志 MDC 自动关联

---

## 六、关联知识链接

- [10-线程基础](./10-线程基础.md) — Thread 对象结构
- [11-JMM与volatile](./11-JMM与volatile.md) — 内存可见性问题
- [14-线程池](./14-线程池.md) — 线程池中 ThreadLocal 泄漏问题
- [12-synchronized与CAS](./12-synchronized与CAS.md) — 对比同步与隔离两种思路
