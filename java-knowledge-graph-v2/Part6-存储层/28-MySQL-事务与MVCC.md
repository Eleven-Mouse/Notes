
> 优先级：⭐⭐⭐⭐⭐ | 面试频率：🔥🔥🔥🔥🔥
> 事务和 MVCC 是 InnoDB 的灵魂，面试必问，尤其是 MVCC 的实现原理。

---

## 知识树

```
MySQL 事务与 MVCC
├── ACID 四大特性
│   ├── 原子性 → Undo Log
│   ├── 一致性 → 约束 + 业务
│   ├── 隔离性 → 锁 + MVCC
│   └── 持久性 → Redo Log
├── 事务隔离级别
│   ├── READ UNCOMMITTED（读未提交）
│   ├── READ COMMITTED（读已提交）
│   ├── REPEATABLE READ（可重复读）← 默认
│   └── SERIALIZABLE（串行化）
├── 并发问题
│   ├── 脏读
│   ├── 不可重复读
│   └── 幻读
└── MVCC 原理
    ├── 隐藏字段（trx_id / roll_pointer）
    ├── Undo Log 版本链
    ├── ReadView（4个核心字段）
    └── 可见性判断规则
```

---

## 一、ACID 四大特性及实现方式

| 特性 | 含义 | 实现方式 |
|------|------|---------|
| **A**tomicity（原子性） | 事务中的操作要么全做，要么全不做 | **Undo Log**：回滚时恢复原始数据 |
| **C**onsistency（一致性） | 事务前后数据满足完整性约束 | 约束（主键/外键/唯一）+ 业务层保证 |
| **I**solation（隔离性） | 并发事务之间互不干扰 | **锁**（写写）+ **MVCC**（读写） |
| **D**urability（持久性） | 事务提交后数据永久保存 | **Redo Log**：WAL 机制保证 |

**一致性是终极目标，其他三个特性是手段。**

---

## 二、事务隔离级别详解

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|---------|------|-----------|------|------|
| READ UNCOMMITTED | ❌ 可能 | ❌ 可能 | ❌ 可能 | 最高 |
| READ COMMITTED（RC） | ✅ 避免 | ❌ 可能 | ❌ 可能 | 高 |
| **REPEATABLE READ（RR）** | ✅ 避免 | ✅ 避免 | ⚠️ 一定程度上避免 | 中 |
| SERIALIZABLE | ✅ 避免 | ✅ 避免 | ✅ 避免 | 最低 |

**MySQL 默认隔离级别：REPEATABLE READ（RR）。**

### 脏读 / 不可重复读 / 幻读

**脏读（Dirty Read）**：读到其他事务未提交的数据。
```sql
-- 事务A                          -- 事务B
UPDATE user SET age=20 WHERE id=1;
                                  SELECT age FROM user WHERE id=1; -- 读到 20
ROLLBACK; -- 回滚了！age 恢复
                                  -- 事务B拿到的 age=20 是脏数据
```

**不可重复读（Non-Repeatable Read）**：同一事务中两次读同一行，结果不同。
```sql
-- 事务A                          -- 事务B
SELECT age FROM user WHERE id=1;  -- age=18
                                  UPDATE user SET age=20 WHERE id=1;
                                  COMMIT;
SELECT age FROM user WHERE id=1;  -- age=20 ← 不一样了！
```

**幻读（Phantom Read）**：同一事务中两次范围查询，行数不同。
```sql
-- 事务A                                -- 事务B
SELECT * FROM user WHERE age BETWEEN 10 AND 20; -- 5行
                                        INSERT INTO user(age) VALUES(15);
                                        COMMIT;
SELECT * FROM user WHERE age BETWEEN 10 AND 20; -- 6行 ← 多了一行！
```

> 不可重复读侧重**数据被修改**，幻读侧重**行数变化**。

---

## 三、MVCC 原理详解

MVCC（Multi-Version Concurrency Control，多版本并发控制）让读操作不加锁，读写不冲突。

### 3.1 隐藏字段

每行数据有两个隐藏字段：

| 字段 | 大小 | 含义 |
|------|------|------|
| `trx_id` | 6B | 最后修改该行的事务 ID |
| `roll_pointer` | 7B | 回滚指针，指向 Undo Log 中的上一个版本 |

### 3.2 Undo Log 版本链

```
当前行（trx_id=300, age=25）
    │
    │ roll_pointer
    ▼
Undo Log 版本（trx_id=200, age=20）
    │
    │ roll_pointer
    ▼
Undo Log 版本（trx_id=100, age=18）
```

每次 UPDATE 都会在 Undo Log 中保留旧版本，通过 roll_pointer 串成链表。

### 3.3 ReadView 的 4 个核心字段

| 字段 | 含义 |
|------|------|
| `m_ids` | 生成 ReadView 时，当前所有**活跃事务**的 ID 列表 |
| `min_trx_id` | 活跃事务中最小的事务 ID（即 m_ids 中的最小值） |
| `max_trx_id` | 下一个将要分配的事务 ID（全局最大事务 ID + 1） |
| `creator_trx_id` | 创建该 ReadView 的事务 ID |

### 3.4 可见性判断规则（完整流程）

对于版本链中的某个版本（trx_id）：

```
1. trx_id == creator_trx_id
   → ✅ 可见（自己的修改）

2. trx_id < min_trx_id
   → ✅ 可见（在 ReadView 创建前已提交）

3. trx_id >= max_trx_id
   → ❌ 不可见（在 ReadView 创建后才开启的事务）

4. min_trx_id <= trx_id < max_trx_id
   → 看 trx_id 是否在 m_ids 中
     - 在 m_ids 中 → ❌ 不可见（事务未提交）
     - 不在 m_ids 中 → ✅ 可见（事务已提交）
```

如果当前版本不可见，沿 roll_pointer 找上一个版本，重复判断。

### 3.5 RC vs RR 的 ReadView 生成时机

| 隔离级别 | ReadView 生成时机 | 效果 |
|---------|------------------|------|
| **RC** | 每次 SELECT 都生成新的 ReadView | 能看到其他事务已提交的最新数据 → 避免脏读，但不避免不可重复读 |
| **RR** | 事务中第一次 SELECT 生成 ReadView，后续复用 | 事务内多次读取结果一致 → 避免不可重复读 |

**这就是 RC 和 RR 的本质区别：ReadView 的生成频率不同。**

---

## 四、RR 级别为什么在一定程度上解决幻读

### 快照读（普通 SELECT）

RR 下 MVCC 使用第一次 SELECT 时的 ReadView，后续读到的都是同一快照，不会出现幻读。

### 当前读（加锁读）

```sql
SELECT * FROM user WHERE age BETWEEN 10 AND 20 FOR UPDATE;
```

当前读读的是**最新已提交数据**，不是快照。此时通过 **Next-Key Lock**（间隙锁 + 行锁）来防止其他事务在该范围内插入新行，从而避免幻读。

### 快照读 vs 当前读

| 类型 | SQL 示例 | 读取数据 | 机制 |
|------|---------|---------|------|
| 快照读 | 普通 SELECT | 快照版本 | MVCC |
| 当前读 | SELECT ... FOR UPDATE / INSERT / UPDATE / DELETE | 最新数据 | 锁 |

**注意**：RR 级别并未完全解决幻读。如果先快照读、再当前读，中间有新行被其他事务插入，当前读会看到这行，产生幻读。

---

## 面试题精选

### 1. MySQL 的 ACID 分别是怎么实现的？
> 原子性靠 Undo Log 回滚，一致性靠约束和业务层，隔离性靠锁和 MVCC，持久性靠 Redo Log 的 WAL 机制。

### 2. MVCC 的原理是什么？
> 每行数据有隐藏字段 trx_id 和 roll_pointer，UPDATE 产生 Undo Log 版本链。事务读取时创建 ReadView，通过可见性判断规则沿版本链找到第一个可见的版本。RC 每次 SELECT 创建新 ReadView，RR 只在第一次创建。

### 3. RC 和 RR 的本质区别是什么？
> ReadView 的生成时机不同。RC 每次 SELECT 都生成，所以能看到其他事务已提交的最新数据；RR 只在第一次 SELECT 生成并复用，所以同一事务内多次读取结果一致。

### 4. ReadView 的可见性判断规则？
> 先判断是否是自己的事务，再看是否在生成 ReadView 之前已提交（trx_id < min_trx_id），再看是否在之后才开启（trx_id >= max_trx_id），最后看是否在活跃事务列表 m_ids 中。

### 5. RR 级别能不能完全解决幻读？
> 不能完全解决。快照读通过 MVCC 避免幻读，当前读通过 Next-Key Lock 避免幻读。但如果先快照读再当前读，中间有新插入的行，当前读会看到，产生幻读。

### 6. 什么是快照读和当前读？
> 快照读是普通 SELECT，读取 MVCC 快照；当前读是加锁读（FOR UPDATE/LOCK IN SHARE MODE）和写操作，读取最新已提交数据。

### 7. 为什么 MySQL 默认用 RR 而不是 RC？
> RR 在大多数场景下能避免不可重复读和部分幻读，对应用层更友好。RC 下同一事务两次读可能结果不同，增加业务复杂度。

### 8. Undo Log 版本链什么时候清理？
> 当没有任何活跃事务需要访问某个历史版本时，由 purge 线程清理。长事务会阻止清理，导致 Undo Log 膨胀。

### 9. 事务 ID 是怎么分配的？
> 事务开始时向系统申请一个严格递增的事务 ID。只读事务在第一次执行读操作时分配，写事务在第一次修改时分配。

### 10. 什么是长事务？有什么危害？
> 执行时间很长的事务。危害：Undo Log 无法清理导致空间膨胀；占用锁资源导致阻塞；ReadView 无法释放。

---

## 常见误区

| 误区 | 正解 |
|------|------|
| MVCC 可以解决所有并发问题 | 不能解决丢失更新，不能完全解决幻读 |
| RR 完全解决幻读 | RR 通过 MVCC 和 Next-Key Lock 在大多数场景解决，但不完全 |
| ReadView 是每行一个 | 每个事务一个 ReadView（RR），或每次 SELECT 一个（RC） |
| Undo Log 只用于回滚 | 还用于 MVCC 版本链，以及崩溃恢复 |

---

## 实战场景

**场景 1：长事务导致 Undo Log 膨胀**
```sql
-- 监控长事务
SELECT * FROM information_schema.innodb_trx
WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;
```

**场景 2：用 BEGIN 但不及时 COMMIT 导致锁等待**
```sql
-- 建议设置事务超时
SET SESSION innodb_lock_wait_timeout = 10;
-- 或设置只读事务
START TRANSACTION READ ONLY;
```

---

## 关联知识

- [27-MySQL-存储引擎与索引](./27-MySQL-存储引擎与索引.md) — InnoDB 索引是聚簇索引，与 MVCC 的隐藏字段存储在同一页
- [29-MySQL-锁与调优](./29-MySQL-锁与调优.md) — 锁与 MVCC 的配合、Undo Log 与锁的关系
- [32-Redis-缓存问题](./32-Redis-缓存问题.md) — 事务一致性在缓存场景的延伸
