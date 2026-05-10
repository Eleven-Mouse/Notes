
> 锁是并发控制的核心，SQL 调优是实战必备，面试高频组合考点。

---

## 知识树

```
MySQL 锁与调优
├── 锁分类
│   ├── 全局锁
│   ├── 表级锁（表锁/MDL/意向锁）
│   └── 行级锁（Record/Gap/Next-Key）
├── 行锁算法
│   ├── Record Lock
│   ├── Gap Lock
│   └── Next-Key Lock
├── 死锁
│   ├── 检测
│   └── 处理
├── SQL 调优
│   ├── EXPLAIN 执行计划
│   ├── 慢查询日志
│   └── 常见优化技巧
├── 日志系统
│   ├── Redo Log
│   ├── Undo Log
│   ├── Binlog
│   └── 两阶段提交
└── 深分页优化
```

---

## 一、MySQL 锁分类

### 全局锁

```sql
FLUSH TABLES WITH READ LOCK;  -- 加全局读锁
UNLOCK TABLES;                 -- 释放
```

- 整库只读，所有 DML/DDL 阻塞
- 用途：全库逻辑备份（mysqldump --single-transaction 更优）

### 表级锁

#### 表锁
```sql
LOCK TABLES user READ;    -- 读锁（共享）
LOCK TABLES user WRITE;   -- 写锁（排他）
UNLOCK TABLES;
```

#### 元数据锁（MDL，Metadata Lock）
- MySQL 5.5+ 自动加，不需要显式声明
- **读操作**加 MDL 读锁，**写操作（DDL）**加 MDL 写锁
- 作用：防止 DML 执行期间 DDL 修改表结构

```sql
-- 事务A：长查询
SELECT * FROM user;  -- 加 MDL 读锁
-- 事务B：DDL 阻塞等待
ALTER TABLE user ADD COLUMN email VARCHAR(100);  -- 等 MDL 写锁
-- 事务C：DML 也阻塞！等待事务B
SELECT * FROM user;  -- 阻塞（排队等事务B释放写锁后才能拿读锁）
```

#### 意向锁（IS / IX）

| 意向锁 | 含义 |
|--------|------|
| IS（意向共享锁） | 事务打算加行级共享锁 |
| IX（意向排他锁） | 事务打算加行级排他锁 |

**作用**：快速判断表中是否有行锁，避免逐行检查。

```
事务A：SELECT * FROM user WHERE id=1 LOCK IN SHARE MODE;  → 加 IS 表锁 + S 行锁
事务B：LOCK TABLES user WRITE;  → 需要检查是否有行锁
  → 检查 IX/IS 即可（意向锁存在 = 有行锁），不用遍历每一行
```

### 行级锁（InnoDB 独有）

| 锁类型 | 锁定范围 | 说明 |
|--------|---------|------|
| Record Lock | 锁定索引记录 | 锁单行 |
| Gap Lock | 锁定索引间隙（不含记录） | 锁间隙，防插入 |
| Next-Key Lock | Record + Gap（左开右闭） | 默认算法 |

---

## 二、Next-Key Lock 加锁规则

InnoDB 的行锁是**加在索引上**的。

### 等值查询

```sql
-- 表 user，id 主键，age 有索引，假设 age 值有：10, 20, 30, 40
SELECT * FROM user WHERE age = 20 FOR UPDATE;
```

- 命中记录：加 Record Lock 锁住 age=20 这行
- 未命中记录：加 Gap Lock 锁住间隙

```sql
-- 假设 age=15 不存在
SELECT * FROM user WHERE age = 15 FOR UPDATE;
-- 加 Gap Lock：(10, 20)，阻止插入 10 < age < 20 的记录
```

### 范围查询

```sql
SELECT * FROM user WHERE age >= 20 AND age < 30 FOR UPDATE;
-- Next-Key Lock：(20, 30] 和 [20, 20]
-- 即锁住 age=20 到 age=30 的范围
```

### 无索引时行锁退化为表锁

```sql
-- 如果 name 列没有索引
SELECT * FROM user WHERE name = '张三' FOR UPDATE;
-- InnoDB 不得不扫描全表，锁住所有行 → 等同于表锁
```

**这就是为什么更新操作一定要走索引。**

---

## 三、死锁检测和处理

```sql
-- 事务A                -- 事务B
UPDATE user SET age=1 WHERE id=1;
                       UPDATE user SET age=2 WHERE id=2;
UPDATE user SET age=3 WHERE id=2;  -- 等B释放id=2
                       UPDATE user SET age=4 WHERE id=1;  -- 等A释放id=1 → 死锁！
```

**检测机制**：`innodb_deadlock_detect = ON`（默认开启），通过等待图（wait-for graph）检测环。

**处理**：检测到死锁后，InnoDB 自动回滚代价最小的事务，释放锁。

**预防死锁**：
1. 按固定顺序访问表和行
2. 事务尽量短小
3. 合理使用索引避免锁升级

---

## 四、EXPLAIN 执行计划详解

```sql
EXPLAIN SELECT * FROM user WHERE name = '张三';
```

### 核心字段

| 字段 | 含义 | 关注点 |
|------|------|--------|
| type | 访问类型 | **最重要的指标** |
| key | 实际使用的索引 | 是否走了期望的索引 |
| rows | 预估扫描行数 | 越少越好 |
| Extra | 额外信息 | 关注 Using filesort / Using temporary |
| possible_keys | 可能用的索引 | 空说明没有可用索引 |
| key_len | 索引使用长度 | 判断联合索引用了几个字段 |

### type 从好到差排序

```
system > const > eq_ref > ref > range > index > ALL
```

| type | 说明 | 示例 |
|------|------|------|
| system | 表中只有一行 | 系统表 |
| const | 主键/唯一索引等值查询 | WHERE id = 1 |
| eq_ref | JOIN 时主键/唯一索引 | JOIN ON t1.id = t2.id |
| ref | 非唯一索引等值查询 | WHERE name = '张三' |
| range | 索引范围扫描 | WHERE age > 20 |
| index | 全索引扫描 | 索引列无过滤 |
| **ALL** | **全表扫描** | **需要优化** |

### Extra 关键信息

| Extra 值 | 含义 | 建议 |
|----------|------|------|
| Using index | 索引覆盖，不回表 | 好 |
| Using index condition | 索引下推 | 好 |
| Using where | Server 层过滤 | 一般 |
| Using filesort | 额外排序（未用索引） | 需优化 |
| Using temporary | 使用临时表 | 需优化 |

---

## 五、慢查询日志

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未用索引的查询

-- 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log_file';
```

分析工具：`mysqldumpslow -s t -t 10 slow.log`（按时间排序取前 10 条）

---

## 六、常见 SQL 优化技巧

### 避免 SELECT *

```sql
-- ❌
SELECT * FROM user WHERE age > 20;
-- ✅
SELECT id, name, age FROM user WHERE age > 20;
```

### 深分页优化

```sql
-- ❌ 传统 LIMIT 深分页（扫描 100010 行，丢弃前 100000 行）
SELECT * FROM user ORDER BY id LIMIT 100000, 10;

-- ✅ 方案1：游标分页（推荐）
SELECT * FROM user WHERE id > 100000 ORDER BY id LIMIT 10;

-- ✅ 方案2：子查询先查主键
SELECT * FROM user WHERE id IN (
    SELECT id FROM user ORDER BY id LIMIT 100000, 10
);

-- ✅ 方案3：INNER JOIN
SELECT u.* FROM user u
INNER JOIN (SELECT id FROM user ORDER BY id LIMIT 100000, 10) tmp
ON u.id = tmp.id;
```

### 合理使用索引

```sql
-- ORDER BY 使用索引
-- 索引 (age, name)
SELECT * FROM user WHERE age = 20 ORDER BY name;  -- ✅ 索引排序
SELECT * FROM user ORDER BY age, name;             -- ✅ 索引排序
SELECT * FROM user ORDER BY name, age;             -- ❌ 索引顺序不匹配
```

---

## 七、Redo Log / Undo Log / Binlog 对比

| 特性 | Redo Log | Undo Log | Binlog |
|------|----------|----------|--------|
| 作用 | 崩溃恢复，保证持久性 | 事务回滚，MVCC | 主从复制，数据备份 |
| 层级 | InnoDB 引擎层 | InnoDB 引擎层 | MySQL Server 层 |
| 内容 | 物理日志（页的修改） | 逻辑日志（反向操作） | 逻辑日志（SQL/行变更） |
| 写入方式 | 循环写（固定大小） | 随事务写入 | 追加写（文件递增） |
| 空间 | 固定（如4个文件×1GB） | 随事务增长 | 无上限 |

### 两阶段提交

保证 Redo Log 和 Binlog 的一致性：

```
1. 写入 Redo Log（prepare 状态）
2. 写入 Binlog
3. 提交 Redo Log（commit 状态）

如果步骤2之前崩溃 → Redo Log 是 prepare 但没有对应 Binlog → 回滚
如果步骤2之后崩溃 → Redo Log 是 prepare 且 Binlog 完整 → 提交
```

**为什么需要两阶段？** 如果先写 Binlog 再写 Redo Log，Binlog 写完后崩溃，从库执行了 Binlog 但主库没有 Redo Log → 主从数据不一致。

---

## 面试题精选

### 1. MySQL 有哪些锁？
> 全局锁（FTWRL）、表级锁（表锁/MDL/意向锁）、行级锁（Record/Gap/Next-Key）。InnoDB 的行锁是加在索引上的。

### 2. 什么是 Next-Key Lock？
> InnoDB 默认的行锁算法，锁定一个范围并包含记录本身（左开右闭），等于 Gap Lock + Record Lock，用于防止幻读。

### 3. 没有索引时加锁会怎样？
> 行锁退化为表锁。因为没有索引，InnoDB 无法定位行，必须扫描全表，锁住所有行。所以更新操作必须走索引。

### 4. EXPLAIN 的 type 字段有哪些值？
> 从好到差：system > const > eq_ref > ref > range > index > ALL。ALL 表示全表扫描，需要优化。

### 5. 深分页怎么优化？
> 三种方案：游标分页（WHERE id > last_id）、子查询（先查主键再回表）、INNER JOIN（子查询查主键再 JOIN）。

### 6. Redo Log 和 Binlog 有什么区别？
> Redo Log 是 InnoDB 引擎层的物理日志，循环写，用于崩溃恢复；Binlog 是 Server 层的逻辑日志，追加写，用于主从复制。两阶段提交保证二者一致。

### 7. 什么是意向锁？有什么用？
> 意向锁是表级锁（IS/IX），表示事务打算加行锁。作用是快速判断表中是否有行锁，避免逐行检查，提升加锁效率。

### 8. 怎么预防死锁？
> 按固定顺序访问表和行；事务尽量短；合理使用索引；设置锁等待超时 `innodb_lock_wait_timeout`。

### 9. 慢查询怎么排查？
> 开启慢查询日志 → 用 mysqldumpslow 分析 → EXPLAIN 查看执行计划 → 针对性优化（加索引/改写 SQL/分页优化）。

### 10. 为什么 Redo Log 和 Binlog 要两阶段提交？
> 保证两个日志的一致性。如果只写一个就崩溃，主从数据会不一致。两阶段提交通过 prepare 和 commit 两个状态确保可恢复。

---

## 常见误区

| 误区 | 正解 |
|------|------|
| 行锁锁的是数据行 | 行锁锁的是**索引记录** |
| 有索引就不会全表扫描 | 索引可能失效（函数/隐式转换/左模糊） |
| 慢查询一定是索引问题 | 可能是锁等待、网络、IO 等 |
| Redo Log 越大越好 | 过大导致崩溃恢复变慢 |
| MDL 只影响 DDL | MDL 写锁排队会阻塞后续 DML |

---

## 实战场景

**场景 1：线上 ALTER TABLE 阻塞全库**
```sql
-- 问题：DDL 加 MDL 写锁，长事务持有 MDL 读锁 → DDL 排队 → 后续所有查询阻塞
-- 解决：设置 DDL 超时
SET SESSION lock_wait_timeout = 5;
ALTER TABLE user ADD COLUMN email VARCHAR(100);
-- 超时自动放弃，不阻塞其他查询
```

**场景 2：大批量更新避免长锁**
```sql
-- ❌ 一次性更新百万行
UPDATE user SET status = 1 WHERE create_time < '2024-01-01';
-- ✅ 分批更新
UPDATE user SET status = 1 WHERE id BETWEEN 1 AND 10000 AND create_time < '2024-01-01';
-- 循环执行，每次小批量
```

---

## 关联知识

- [27-MySQL-存储引擎与索引](./27-MySQL-存储引擎与索引.md) — 索引是行锁的基础，无索引则行锁退化为表锁
- [28-MySQL-事务与MVCC](./28-MySQL-事务与MVCC.md) — Undo Log 同时服务于事务回滚和 MVCC
- [32-Redis-缓存问题](./32-Redis-缓存问题.md) — 数据库调优与缓存策略的配合
