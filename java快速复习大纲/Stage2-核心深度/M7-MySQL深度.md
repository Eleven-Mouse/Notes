# Stage 2-4: MySQL + Redis 深度增强

> 优先级 ⭐⭐⭐⭐⭐ | 面试频率 🔥🔥🔥🔥🔥

---

## Part A: MySQL 深度

---

## 知识点 1：索引原理（B+树）⭐🔥

### 【是什么】
索引是数据库加速查询的数据结构，MySQL InnoDB 默认使用 B+ 树索引。

### 【原理】

#### 小林图解：为什么用 B+ 树而不是 B 树？
```
B树（每个节点都存数据）：
  [10|20|30]           ← 每个节点都存key+data
   /   |    \
 [5] [15] [25]         ← 非叶子也存数据，一页能放的节点少
  ↓    ↓    ↓
[d5] [d15] [d25]

B+树（数据只在叶子节点）：
       [20]                     ← 非叶子只存key（路由）
      /    \
  [10|15]  [25|30]              ← 中间节点不存data，一页能放更多key
   /  |      |   \
 [5|10|15] [20|25|30]           ← 叶子存所有data
   ←→  ←→                       ← 叶子之间双向链表（范围查询快）

B+树三大优势：
1. 非叶子不存data → 每个磁盘页能放更多key → 树更矮 → IO更少
2. 叶子链表连接 → 范围查询只需顺序遍历 → 不用回溯
3. 查询性能稳定 → 每次都走到叶子 → 都是O(logN)

B+树高度与数据量：
  假设一页16KB，每个key+指针约12字节
  一页可放约16KB/12B ≈ 1360个key
  2层B+树：1360 * 1360 ≈ 185万条
  3层B+树：1360^3 ≈ 25亿条（InnoDB默认一个页16KB）
  → 3层B+树即可支撑千万级数据，只需3次磁盘IO
```

#### 聚簇索引 vs 非聚簇索引（二级索引）
```
聚簇索引（主键索引）：
  叶子节点存完整行数据
  一张表只有一个（通常是主键）
  InnoDB：如果没有主键，选第一个唯一非空索引，都没有则生成隐藏rowid

非聚簇索引（二级索引）：
  叶子节点存 主键值 + 索引列值
  查非索引列需要"回表"：先查二级索引拿到主键 → 再查聚簇索引拿完整数据

回表：select * from article where title = 'Java'
  1. 查title索引 → 拿到主键id=5
  2. 拿id=5查聚簇索引 → 拿到完整行
  两次B+树查找 = 回表

覆盖索引：select id, title from article where title = 'Java'
  title索引叶子已经有id和title → 不需要回表
  explain中Extra显示 Using index
```

#### 最左前缀原则
```
联合索引 (a, b, c) 等效于三个索引：
  (a), (a, b), (a, b, c)
  不生效：(b), (c), (b, c)

失效场景：
  where a = 1 and b > 2 and c = 3
  → a走索引，b走索引，c不走（b是范围查询，后面的断了）

索引下推（ICP，Index Condition Pushdown，MySQL 5.6+）：
  联合索引(a, b)，查询 where a like '张%' and b = 10
  无ICP：存储引擎通过a找到所有记录 → Server层逐条过滤b
  有ICP：存储引擎直接在索引中过滤b → 减少回表次数
```

### 【怎么用 — 项目落地】

**场景：博客系统文章表索引设计**
```sql
CREATE TABLE article (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    title VARCHAR(200),
    content TEXT,
    status TINYINT,       -- 0草稿 1发布
    category_id INT,
    created_at DATETIME,
    updated_at DATETIME,
    view_count INT,

    -- 索引设计
    INDEX idx_user_status (user_id, status),        -- 查某用户的已发布文章
    INDEX idx_category_created (category_id, created_at DESC),  -- 按分类查最新
    INDEX idx_title (title)                          -- 按标题搜索（覆盖索引优化）
) ENGINE=InnoDB;

-- 高效查询示例（走覆盖索引）
SELECT id, title, created_at FROM article
WHERE category_id = 1 ORDER BY created_at DESC LIMIT 20;
-- Extra: Using index → 不回表

-- 慢查询示例（不走索引）
SELECT * FROM article WHERE content LIKE '%Java%';  -- 全表扫描！
-- 优化：用全文索引或ES
```

### 【面试怎么问】

**Q：为什么MySQL用B+树而不是B树/红黑树/哈希？**

> "B+树相比B树：1.非叶子不存数据，一页放更多key，树更矮，IO更少 2.叶子链表连接，范围查询高效 3.查询性能稳定。相比红黑树：红黑树太高（层数多），磁盘IO次数多。相比哈希：哈希不支持范围查询和排序。总结：B+树在磁盘IO次数、范围查询、排序场景都是最优选择。3层B+树可存约2000万条数据，只需3次IO。"

**追问：什么是回表？怎么减少？**

> "二级索引查到主键后再查聚簇索引拿完整行数据就是回表。减少方法：1.覆盖索引（查询字段都在索引中）2.尽量用主键查询（直接走聚簇索引）3.联合索引包含常用查询字段。explain中Extra显示Using index就是覆盖索引，没有Using index就是需要回表。"

---

## 知识点 2：事务与 MVCC ⭐🔥

### 【是什么】
MVCC（多版本并发控制）是 InnoDB 实现事务隔离级别的一种机制，通过保存数据的历史版本实现读写不冲突。

### 【原理】

#### MVCC 三要素
```
1. 隐藏字段（每行记录）
   ├── DB_TRX_ID      → 最后修改该行的事务ID
   ├── DB_ROLL_PTR    → 指向Undo Log中上一版本的指针
   └── DB_ROW_ID      → 隐藏自增ID（无主键时使用）

2. Undo Log 版本链
   当前数据 ←ROLL_PTR→ 旧版本1 ←ROLL_PTR→ 旧版本2 ←ROLL_PTR→ ...
   每次UPDATE产生一个旧版本，通过ROLL_PTR串联

3. ReadView（读视图）
   ├── m_ids          → 创建ReadView时，当前活跃（未提交）的事务ID列表
   ├── min_trx_id     → 活跃事务中的最小ID
   ├── max_trx_id     → 下一个将要分配的事务ID
   └── creator_trx_id → 创建该ReadView的事务ID
```

#### 可见性判断规则
```
对于版本链中的某个版本，其 DB_TRX_ID：
  1. DB_TRX_ID == creator_trx_id → 是自己修改的，可见
  2. DB_TRX_ID < min_trx_id      → 在ReadView创建前已提交，可见
  3. DB_TRX_ID >= max_trx_id     → 在ReadView创建后才开始的事务，不可见
  4. min_trx_id <= DB_TRX_ID < max_trx_id
     → 检查是否在m_ids中：
       在m_ids中 → 事务未提交，不可见
       不在m_ids中 → 事务已提交，可见
  不可见就沿ROLL_PTR找上一个版本，重复判断
```

#### RC vs RR 的区别
```
RC（读已提交）：每次SELECT都创建新的ReadView
  → 能看到其他事务已提交的最新数据
  → 可能出现不可重复读

RR（可重复读，InnoDB默认）：整个事务只在第一次SELECT时创建ReadView
  → 后续SELECT复用同一个ReadView
  → 保证了可重复读

这是RC和RR在MVCC层面唯一的区别！
```

### 【怎么用 — 项目落地】

**场景：博客文章编辑的并发问题**
```sql
-- 场景：用户A编辑文章，同时用户B也在编辑
-- InnoDB默认RR隔离级别，MVCC保证了：
-- 1. 用户A读到的是他开始事务时的快照，不会被用户B的修改干扰
-- 2. 但如果两人都提交，后提交的会覆盖先提交的（丢失更新）

-- 解决丢失更新：乐观锁
UPDATE article
SET title = '新标题', version = version + 1
WHERE id = 1 AND version = 5;  -- version不匹配则更新失败

-- 或者用SELECT ... FOR UPDATE（悲观锁）
SELECT * FROM article WHERE id = 1 FOR UPDATE;
-- 加X锁，其他事务必须等待
```

### 【面试怎么问】

**Q：MVCC的原理？RC和RR的区别？**

> "MVCC通过隐藏字段+Undo版本链+ReadView实现。每行有事务ID和回滚指针，修改时产生旧版本链。ReadView记录当前活跃事务，通过可见性规则判断哪个版本对当前事务可见。RC每次SELECT创建新ReadView，所以能看到其他事务最新提交的数据。RR只在第一次SELECT创建ReadView，后续复用，所以可重复读。两者MVCC机制相同，区别只在ReadView的创建时机。"

---

## 知识点 3：MySQL 锁 ⭐

### 【原理】

```
InnoDB行锁类型：
┌────────────────┬─────────────────────────────────────────┐
│ Record Lock    │ 锁住索引记录（行锁）                      │
│ Gap Lock       │ 锁住索引记录之间的间隙（不包含记录本身）    │
│ Next-Key Lock  │ Record + Gap（左开右闭区间）              │
│ Insert Intention│ 插入意向锁（Gap Lock的特殊情况）          │
└────────────────┴─────────────────────────────────────────┘

Next-Key Lock 示例（索引中有 5, 10, 15）：
  对10加Next-Key Lock → 锁住 (5, 10] 区间
  作用：防止幻读（RR级别下）

加锁规则（丁奇总结）：
  1. 加锁的基本单位是Next-Key Lock
  2. 查找过程中访问到的对象才会加锁
  3. 等值查询，唯一索引，Next-Key Lock退化为Record Lock
  4. 等值查询，向右遍历到最后不满足条件时，Next-Key Lock退化为Gap Lock
```

### 【面试怎么问】

**Q：MySQL有哪些锁？什么时候用行锁什么时候用表锁？**

> "InnoDB支持行锁（Record/Gap/Next-Key）和表锁。默认用行锁，但如果SQL没有走索引（全表扫描），行锁会升级为表锁。所以一定要确保查询走索引。RR级别下，Next-Key Lock防止幻读。Gap Lock只在RR级别生效，RC级别没有Gap Lock。"

---

## Part B: Redis 深度

---

## 知识点 4：Redis 数据结构 ⭐🔥

### 【原理】

#### 五种基础类型底层实现
```
┌───────────┬──────────────────────────────────────────┐
│ String    │ SDS（Simple Dynamic String）              │
│           │ 二进制安全 / O(1)获取长度 / 预分配减少 realloc │
│           │ 应用：缓存/计数器/分布式锁                   │
├───────────┼──────────────────────────────────────────┤
│ List      │ quickList（ziplist + 双向链表）            │
│           │ 应用：消息队列/最新文章列表                   │
├───────────┼──────────────────────────────────────────┤
│ Hash      │ ziplist（小） / hashtable（大）            │
│           │ 渐进式rehash：分多次迁移，避免阻塞           │
│           │ 应用：存储对象（用户信息/文章详情）           │
├───────────┼──────────────────────────────────────────┤
│ Set       │ intset（纯整数小集合） / hashtable          │
│           │ 应用：点赞集合/标签集合                      │
├───────────┼──────────────────────────────────────────┤
│ ZSet      │ ziplist（小） / skiplist + hashtable（大） │
│           │ skiplist：O(logN)查找/插入，实现简单        │
│           │ 应用：排行榜/热门文章/延迟队列               │
└───────────┴──────────────────────────────────────────┘
```

### 【怎么用 — 项目落地】

**场景：博客系统Redis数据设计**
```redis
# 1. 文章详情缓存（Hash）
HSET article:1001 title "Java并发编程" content "..." author "张三" view_count 1024

# 2. 文章点赞（Set）
SADD article:1001:likes user:1 user:2 user:3
SCARD article:1001:likes  # 点赞数
SISMEMBER article:1001:likes user:1  # 是否已点赞

# 3. 热门文章排行（ZSet，score=浏览量）
ZADD hot_articles 1024 article:1001
ZADD hot_articles 2048 article:1002
ZREVRANGE hot_articles 0 9 WITHSCORES  # Top 10

# 4. 最新文章列表（List）
LPUSH latest_articles article:1001
LRANGE latest_articles 0 19  # 最新20篇

# 5. AI对话计数器（String）
SET ai:quota:user:1 100 EX 86400  # 每日100次，24h过期
DECR ai:quota:user:1
```

### 【面试怎么问】

**Q：Redis为什么用跳表不用红黑树实现ZSet？**

> "跳表和红黑树都是O(logN)，但跳表优势：1.实现简单，代码易理解和维护 2.范围查询更高效（链表顺序遍历）3.插入删除只需修改相邻节点指针，红黑树需要旋转 4.内存开销可控（通过调节层数概率）。Redis作者antirez的说法：'跳表和平衡树性能相当，但更简单更灵活'。"

---

## 知识点 5：缓存三大问题 ⭐🔥

### 【是什么】
缓存穿透（查不存在的数据）、缓存击穿（热点key过期）、缓存雪崩（大量key同时过期）。

### 【原理】

```
穿透：恶意请求查id=-1的数据，缓存没有→DB没有→缓存也不存→反复打DB
击穿：某个热点key（如爆款文章）过期瞬间，大量请求同时打到DB
雪崩：大量key同一时刻过期（如批量设置相同TTL），或Redis集群宕机
```

#### 解决方案对比
```
┌──────────┬──────────────────────┬────────────────────────┐
│ 问题      │ 方案                  │ 优劣                    │
├──────────┼──────────────────────┼────────────────────────┤
│ 穿透     │ 1. 缓存空值（短TTL）   │ 简单，但占内存           │
│          │ 2. 布隆过滤器          │ 省内存，有误判           │
│          │ 3. 参数校验            │ 根本解决                 │
├──────────┼──────────────────────┼────────────────────────┤
│ 击穿     │ 1. 互斥锁（SETNX）     │ 简单，但有死锁风险       │
│          │ 2. 逻辑过期（不设TTL）  │ 用户体验好，可能短暂脏读  │
│          │ 3. 热点key永不过期      │ 简单粗暴                 │
├──────────┼──────────────────────┼────────────────────────┤
│ 雪崩     │ 1. TTL加随机值         │ 简单有效                 │
│          │ 2. 多级缓存            │ 可用性高，复杂度高       │
│          │ 3. Redis集群+sentinel  │ 高可用保障               │
│          │ 4. 限流降级            │ 兜底方案                 │
└──────────┴──────────────────────┴────────────────────────┘
```

### 【怎么用 — 项目落地】

**场景：博客系统缓存方案**
```java
@Service
public class ArticleCacheService {

    // 击穿解决方案：互斥锁
    public Article getArticle(Long id) {
        String key = "article:" + id;
        // 1. 查缓存
        String json = redis.get(key);
        if (json != null) {
            return "NULL".equals(json) ? null : JSON.parseObject(json, Article.class);
        }

        // 2. 缓存未命中，加锁重建
        String lockKey = "lock:article:" + id;
        try {
            if (redis.setnx(lockKey, "1", 10, TimeUnit.SECONDS)) {
                // 获取锁成功 → 查DB → 写缓存
                Article article = articleMapper.selectById(id);
                if (article == null) {
                    redis.set(key, "NULL", 60);  // 空值防穿透
                } else {
                    // TTL加随机值防雪崩
                    int ttl = 3600 + new Random().nextInt(600);
                    redis.set(key, JSON.toJSONString(article), ttl);
                }
                return article;
            } else {
                // 获取锁失败 → 短暂等待后重试
                Thread.sleep(100);
                return getArticle(id);  // 递归重试（注意深度）
            }
        } finally {
            redis.del(lockKey);
        }
    }
}
```

### 【面试怎么问】

**Q：缓存和数据库一致性怎么保证？**

> "常用Cache Aside模式：读时先缓存后DB，写时先DB后删缓存。为什么删缓存而不是更新？因为并发下更新缓存可能产生数据不一致（A先更新DB但缓存更新慢，B已读了旧值）。延迟双删：先删缓存→更新DB→延迟一段时间再删缓存。但仍有极端情况，终极方案是监听Binlog（Canal）异步删除缓存，实现最终一致性。面试中建议回答：允许短暂不一致用延迟双删，要求强一致用Binlog监听。"
