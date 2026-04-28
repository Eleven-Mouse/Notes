# MySQL — 深度拆解

> 优先级：⭐⭐⭐ | 面试频率：🔥🔥🔥 | 掌握程度：深入

---

## 一、知识树

```
MySQL
│
├── 1. 存储引擎 ⭐⭐ 🔥🔥
│   ├── InnoDB（默认，支持事务/行锁/MVCC）
│   ├── MyISAM（不支持事务/表锁/全文索引）
│   └── 对比选型
│
├── 2. 索引 ⭐⭐⭐ 🔥🔥🔥
│   ├── B+ 树结构（为什么用 B+ 树）
│   ├── 聚簇索引 vs 非聚簇索引（二级索引）
│   ├── 回表查询 / 索引覆盖 / 最左匹配
│   ├── 索引下推（ICP）
│   ├── 常见索引失效场景
│   └── 索引设计原则
│
├── 3. 事务 ⭐⭐⭐ 🔥🔥🔥
│   ├── ACID 特性
│   ├── 事务隔离级别（4 种）
│   ├── MVCC（多版本并发控制）
│   │   ├── 隐藏字段（trx_id / roll_pointer）
│   │   ├── Undo Log 版本链
│   │   └── ReadView（读视图）
│   └── 事务日志（Redo Log / Undo Log）
│
├── 4. 锁机制 ⭐⭐⭐ 🔥🔥🔥
│   ├── 全局锁 / 表级锁 / 行级锁
│   ├── 共享锁（S） / 排他锁（X）
│   ├── 意向锁（IS / IX）
│   ├── Gap Lock / Next-Key Lock
│   ├── 行锁的三种算法
│   └── 死锁检测与处理
│
├── 5. SQL 调优 ⭐⭐⭐ 🔥🔥🔥
│   ├── EXPLAIN 执行计划
│   ├── 慢查询分析
│   ├── SQL 优化技巧
│   └── 分页优化
│
└── 6. 架构 ⭐⭐ 🔥
    ├── 主从复制（Binlog）
    ├── 读写分离
    └── 分库分表思路
```

---

## 二、核心知识点精讲

### 2.1 索引 — B+ 树

**为什么 MySQL 用 B+ 树而不是 B 树 / 红黑树 / Hash？**

```
vs B 树：
1. B+ 树数据只在叶子节点 → 非叶子节点只存索引 → 每个节点能存更多索引 → 树更矮 → IO 更少
2. 叶子节点用双向链表连接 → 范围查询高效（不需要中序遍历）
3. 查询性能稳定（每次都要走到叶子节点，路径长度相同）

vs 红黑树：
  树高度不可控（数据量大时高度很大）→ IO 次数多

vs Hash：
  不支持范围查询 / 排序 / 最左匹配
  Hash 适合等值查询（MEMORY 引擎使用）
```

**聚簇索引 vs 非聚簇索引：**

```
聚簇索引（Clustered Index）：
  - 数据和索引存在一起
  - 一张表只有一个（通常是主键）
  - InnoDB: 如果没有主键，选第一个唯一索引；都没有，生成隐藏 rowid

非聚簇索引（Secondary Index / 二级索引）：
  - 叶子节点存主键值（不是数据行地址）
  - 查询流程：二级索引 → 找到主键 → 回聚簇索引查数据（回表）

索引覆盖（Covering Index）：
  - 查询的字段都在二级索引中 → 不需要回表
  - EXPLAIN 中 Extra 显示 Using index
```

**最左匹配原则：**

```sql
-- 联合索引 idx_abc(a, b, c)

-- 命中索引
WHERE a = 1                    -- 命中 a
WHERE a = 1 AND b = 2          -- 命中 a, b
WHERE a = 1 AND b = 2 AND c = 3 -- 命中 a, b, c
WHERE a = 1 AND c = 3          -- 命中 a（c 走不到索引）

-- 不命中索引
WHERE b = 2                    -- 缺少最左列 a
WHERE b = 2 AND c = 3          -- 缺少最左列 a
WHERE c = 3                    -- 缺少最左列 a

-- 范围查询会中断后续列的索引使用
WHERE a > 1 AND b = 2          -- 只有 a 走索引
```

**索引失效的常见场景：**

| 场景 | 示例 | 原因 |
|------|------|------|
| 对索引列使用函数 | WHERE YEAR(create_time) = 2024 | 破坏了 B+ 树有序性 |
| 隐式类型转换 | WHERE varchar_col = 123 | 转换了索引列 |
| LIKE 左模糊 | WHERE name LIKE '%张' | 无法利用 B+ 树有序性 |
| OR 连接非索引列 | WHERE a = 1 OR b = 2（b 无索引） | 需要全表扫 b |
| NOT IN / NOT EXISTS | WHERE id NOT IN (...) | 优化器可能放弃索引 |
| 数据量太小 | 几百行 | 优化器认为全表扫更快 |

### 2.2 事务与隔离级别

**ACID：**

| 特性 | 含义 | 实现方式 |
|------|------|---------|
| 原子性（Atomicity） | 事务要么全成功要么全失败 | Undo Log |
| 一致性（Consistency） | 数据从一个一致状态到另一个 | 业务 + 约束保证 |
| 隔离性（Isolation） | 并发事务互不干扰 | 锁 + MVCC |
| 持久性（Durability） | 提交后永久保存 | Redo Log |

**四种隔离级别：**

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 实现 |
|---------|------|-----------|------|------------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 直接读最新数据 |
| READ COMMITTED | 不会 | 可能 | 可能 | 每次查询生成新 ReadView |
| **REPEATABLE READ**（默认） | 不会 | 不会 | 可能* | 事务开始时生成 ReadView |
| SERIALIZABLE | 不会 | 不会 | 不会 | 所有查询加共享锁 |

*注：InnoDB 在 RR 级别通过 Next-Key Lock 在很大程度上解决了幻读

### 2.3 MVCC 原理

```
每行数据隐藏字段：
  trx_id：最后修改该行的事务 ID
  roll_pointer：指向 Undo Log 中的上一个版本

版本链示例：
  ┌──────────────────┐
  │ 当前版本 (trx=5)  │
  │ name='Alice'      │
  └────────┬─────────┘
           │ roll_pointer
           ▼
  ┌──────────────────┐
  │ 历史版本 (trx=3)  │
  │ name='Bob'        │
  └────────┬─────────┘
           │ roll_pointer
           ▼
  ┌──────────────────┐
  │ 历史版本 (trx=1)  │
  │ name='Charlie'    │
  └──────────────────┘

ReadView（读视图）判断可见性：
  - m_ids：生成 ReadView 时当前活跃的事务 ID 列表
  - min_trx_id：m_ids 中最小的事务 ID
  - max_trx_id：下一个将要分配的事务 ID
  - creator_trx_id：创建该 ReadView 的事务 ID

判断规则：
  trx_id < min_trx_id → 该版本在 ReadView 创建前已提交 → 可见
  trx_id >= max_trx_id → 该版本在 ReadView 创建后才出现 → 不可见
  min_trx_id <= trx_id < max_trx_id → 检查是否在 m_ids 中
    在 → 未提交 → 不可见
    不在 → 已提交 → 可见
```

### 2.4 锁机制

**InnoDB 行锁的三种算法：**

```
Record Lock：锁定单行记录
Gap Lock：锁定索引记录之间的间隙（防止插入）
Next-Key Lock：Record Lock + Gap Lock（左开右闭区间）

示例（索引列有值 10, 20, 30）：
  事务 A: SELECT * FROM t WHERE id = 20 FOR UPDATE
  锁定范围：(10, 20] + (20, 30)  → Next-Key Lock
  即：事务 B 不能插入 id=15 或 id=25 的记录

无索引时：行锁退化为表锁！
```

### 2.5 EXPLAIN 执行计划

```
EXPLAIN SELECT * FROM user WHERE name = 'Alice';

重点字段：
  type：访问类型（从好到差）
    system > const > eq_ref > ref > range > index > ALL

  key：实际使用的索引
  rows：预估扫描行数
  Extra：
    Using index → 索引覆盖（好）
    Using filesort → 额外排序（需优化）
    Using temporary → 使用临时表（需优化）
    Using where → 存储引擎返回数据后 Server 层过滤
```

**type 字段含义（重点掌握前 5 个）：**

| type | 含义 | 示例 |
|------|------|------|
| const | 主键/唯一索引等值查询 | WHERE id = 1 |
| eq_ref | 关联查询时被驱动表用主键匹配 | JOIN 时 |
| ref | 非唯一索引等值查询 | WHERE name = 'Alice' |
| range | 索引范围扫描 | WHERE id > 10 |
| index | 全索引扫描 | 覆盖索引但没过滤条件 |
| ALL | 全表扫描 | 没走索引 |

### 2.6 Redo Log 与 Undo Log

```
Redo Log（重做日志）：
  作用：保证持久性（crash-safe）
  原理：WAL（Write-Ahead Logging），先写日志再写数据
  流程：修改数据 → 写 Redo Log Buffer → fsync 到磁盘 → 后台异步刷数据页

Undo Log（回滚日志）：
  作用：保证原子性（回滚）+ 实现 MVCC
  原理：记录数据的反向操作
  流程：修改数据前先记录旧值到 Undo Log → 修改数据 → 回滚时用 Undo Log 恢复
```

---

## 三、高频面试题（12 题）

### Q1：MySQL 为什么用 B+ 树做索引？
```
1. 磁盘 IO 友好：非叶子节点只存索引，单页存更多索引，树更矮，IO 更少
2. 范围查询高效：叶子节点双向链表，顺序/范围查询只需遍历链表
3. 查询性能稳定：所有查询都要走到叶子节点，路径长度相同
```

### Q2：什么是回表？怎么避免？
```
回表：通过二级索引找到主键 → 再去聚簇索引查完整数据
避免：索引覆盖（查询字段都在索引中，不需要回表）

示例：
  索引 idx_name_age(name, age)
  SELECT name, age FROM user WHERE name = 'Alice'  → 覆盖索引，不回表
  SELECT * FROM user WHERE name = 'Alice'           → 需要回表
```

### Q3：事务的隔离级别？MySQL 默认是哪个？
```
READ UNCOMMITTED > READ COMMITTED > REPEATABLE READ > SERIALIZABLE
（隔离级别越高，性能越差）

MySQL InnoDB 默认：REPEATABLE READ
Oracle / PostgreSQL 默认：READ COMMITTED
```

### Q4：MVCC 是怎么实现的？
```
核心组件：
1. 隐藏字段（trx_id / roll_pointer）→ 记录修改事务 ID
2. Undo Log 版本链 → 保存历史版本
3. ReadView → 判断哪个版本对当前事务可见

RC：每次 SELECT 生成新 ReadView
RR：事务开始时生成一次 ReadView，后续复用
```

### Q5：InnoDB 有哪几种锁？
```
全局锁：FTWRL（备份用）
表级锁：表锁 / 元数据锁（MDL） / 意向锁
行级锁：Record Lock / Gap Lock / Next-Key Lock

注意：行锁是加在索引上的，没有索引则退化为表锁
```

### Q6：什么是幻读？怎么解决？
```
幻读：同一事务中两次查询结果集不同（多了或少了行）

解决：
1. RR 级别 + Next-Key Lock：锁定间隙防止新插入
2. SERIALIZABLE 级别：所有查询加共享锁
3. MVCC：快照读看不到其他事务的新插入（但当前读可以）
```

### Q7：慢查询怎么排查和优化？
```
排查：
1. 开启慢查询日志（slow_query_log）
2. mysqldumpslow 分析 Top N 慢 SQL
3. EXPLAIN 分析执行计划

优化：
1. 检查是否走了索引（type 是否为 ALL）
2. 是否有 Using filesort / Using temporary
3. 添加合适的索引
4. 避免 SELECT *
5. 分页优化（游标分页代替 OFFSET）
```

### Q8：Redo Log 和 Binlog 的区别？
```
Redo Log：InnoDB 引擎层，物理日志，循环写，crash-safe
Binlog：Server 层，逻辑日志（SQL/行变更），追加写，主从复制

两阶段提交：保证 Redo Log 和 Binlog 一致性
1. 写 Redo Log（prepare 状态）
2. 写 Binlog
3. 写 Redo Log（commit 状态）
```

### Q9：主键索引和唯一索引的区别？
```
主键索引：一张表只有一个，不能为 NULL，聚簇索引
唯一索引：可以有多个，可以为 NULL（MySQL 允许多个 NULL），二级索引

选择：优先用自增主键（避免页分裂，插入性能更好）
```

### Q10：COUNT(1) / COUNT(*) / COUNT(字段) 的区别？
```
COUNT(*) / COUNT(1)：统计总行数（包含 NULL），InnoDB 优化后一样快
COUNT(字段)：统计该字段非 NULL 的行数

性能：COUNT(*) ≈ COUNT(1) > COUNT(字段)
建议：统计总行数用 COUNT(*)
```

### Q11：分库分表的方案？
```
垂直拆分：按业务模块拆（用户库 / 订单库 / 商品库）
水平拆分：按数据量拆（user_0 / user_1 / user_2）

分片键选择：查询频率最高的字段（通常是 user_id / order_id）
扩容方案：一致性哈希 / 双写迁移

中间件：ShardingSphere / MyCat
```

### Q12： varchar(50) 中的 50 是什么意思？
```
MySQL 5.0+：50 表示最多存 50 个字符（不是字节）
  utf8mb4 编码：每个字符最多 4 字节 → 50 个字符最多占 200 字节
  utf8 编码：每个字符最多 3 字节 → 50 个字符最多占 150 字节

过长的 varchar 会影响索引创建（索引长度限制 767 字节 / 3072 字节）
```

---

## 四、常见误区

| 误区 | 正确理解 |
|------|---------|
| 索引越多越好 | 索引占空间、降低写入性能、优化器可能选错 |
| InnoDB 不会幻读 | RR 级别快照读不幻读，当前读仍可能幻读 |
| 事务隔离级别越高越好 | SERIALIZABLE 性能差，通常用 RR 或 RC |
| COUNT(*) 慢 | InnoDB 有优化，走最小的非聚簇索引 |
| 有索引就一定走索引 | 优化器基于成本决策，可能选择全表扫描 |

---

## 五、实战应用场景

| 场景 | 优化方案 |
|------|---------|
| 深分页查询慢 | 游标分页：WHERE id > last_id LIMIT 100 |
| 大表加字段 | pt-online-schema-change / gh-ost |
| 热点数据查询 | Redis 缓存 + 数据库兜底 |
| 批量插入 | batch insert / LOAD DATA INFILE |
| 读写压力大 | 主从复制 + 读写分离 |
| 单表数据过大 | 水平分表（按时间/ID 范围/Hash） |

---

## 六、关联模块

- ← [04-锁机制.md](04-锁机制.md)：MySQL 行锁是锁机制在数据库层面的体现
- → [05-Spring全家桶.md](05-Spring全家桶.md)：Spring 事务与 MySQL 事务隔离配合
- → [07-Redis.md](07-Redis.md)：Redis 缓存 + MySQL 持久化的经典组合
- → [08-MyBatis.md](08-MyBatis.md)：MyBatis 是 Java 与 MySQL 的桥梁
