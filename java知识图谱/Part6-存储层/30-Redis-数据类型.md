

> 优先级：⭐⭐⭐⭐⭐ | 面试频率：🔥🔥🔥🔥🔥
> Redis 数据类型和底层结构是基础中的基础，面试几乎必问底层数据结构。

---

## 知识树

```
Redis 数据类型
├── 五大基本类型
│   ├── String（字符串）
│   ├── List（列表）
│   ├── Hash（哈希）
│   ├── Set（集合）
│   └── ZSet（有序集合）
├── 补充类型
│   ├── Bitmap（位图）
│   ├── HyperLogLog（基数估算）
│   ├── GEO（地理位置）
│   └── Stream（流）
└── 底层数据结构
    ├── SDS（简单动态字符串）
    ├── ziplist（压缩列表）
    ├── quicklist（快速列表）
    ├── skiplist（跳表）
    ├── hashtable（哈希表）
    └── intset（整数集合）
```

---

## 一、五大数据类型及应用场景

| 类型 | 存储 | 常用命令 | 典型场景 |
|------|------|---------|---------|
| String | key-value | SET/GET/INCR/SETEX | 缓存、计数器、分布式锁、Session |
| List | 有序列表 | LPUSH/RPUSH/LPOP/RPOP/LRANGE | 消息队列、最新列表、栈/队列 |
| Hash | field-value | HSET/HGET/HMGET/HGETALL | 对象存储、购物车 |
| Set | 无序去重集合 | SADD/SMEMBERS/SINTER/SUNION | 标签、共同好友、抽奖 |
| ZSet | 有序去重集合+分数 | ZADD/ZRANGE/ZRANGEBYSCORE | 排行榜、延迟队列、滑动窗口 |

### 补充类型

| 类型 | 场景 |
|------|------|
| Bitmap | 签到打卡、用户在线状态、布隆过滤器 |
| HyperLogLog | UV 统计（允许误差） |
| GEO | 附近的人、距离计算 |
| Stream | 消息队列（支持消费组，5.0+） |

### 常用命令速查

```bash
# String
SET key value [EX seconds] [NX]    # 设置+过期+不存在才设置
GET key                             # 获取
INCR key                            # 自增（原子操作）
SETEX key 60 value                  # 设置+60秒过期

# List
LPUSH list a b c                    # 左边插入（c b a）
RPUSH list a b c                    # 右边插入（a b c）
LRANGE list 0 -1                    # 获取全部
LPOP list                           # 左边弹出

# Hash
HSET user:1 name "张三" age 25      # 设置多个field
HGET user:1 name                    # 获取单个field
HGETALL user:1                      # 获取所有field

# Set
SADD tags "Java" "Redis" "MySQL"    # 添加元素
SINTER set1 set2                    # 交集
SISMEMBER tags "Java"               # 判断是否存在

# ZSet
ZADD rank 100 "张三" 90 "李四"      # 添加+分数
ZRANGE rank 0 -1 WITHSCORES         # 升序排列
ZREVRANGE rank 0 9 WITHSCORES       # 降序TOP10
ZRANGEBYSCORE rank 80 100           # 分数范围查询
```

---

## 二、底层数据结构详解

### 2.1 SDS（Simple Dynamic String）

Redis 自己实现的字符串，替代 C 语言的 char[]。

```
SDS 结构：
┌────────┬────────┬──────────┬──────────────┐
│  len   │  free  │  flags   │    buf[]     │
│ 已用长度│ 婴余空间│ 类型标识  │ 字节数组      │
└────────┴────────┴──────────┴──────────────┘
```

| 特性 | C 字符串 | SDS |
|------|---------|-----|
| 获取长度 | O(n) 遍历 | O(1) 直接读 len |
| 缓冲区溢出 | 不安全（strcpy 可能溢出） | 安全（自动扩容） |
| 修改内存重分配 | 每次都要 | 空间预分配+惰性释放 |
| 二进制安全 | ❌ 靠 \0 判断结束 | ✅ 靠 len 判断长度 |
| 适用场景 | 简单字符串 | 任意二进制数据 |

**空间预分配策略**：
- len < 1MB：分配与 len 等大的 free 空间
- len >= 1MB：分配 1MB 的 free 空间

### 2.2 ziplist（压缩列表）

紧凑的连续内存结构，省内存但操作效率低。

```
ziplist 内存布局：
┌───────┬───────┬──────┬──────┬──────┬───────┐
│ zlbytes│ zltail│ zllen│entry1│entry2│ zlend │
│ 总字节 │ 尾偏移 │ 节点数│      │      │ 0xFF  │
└───────┴───────┴──────┴──────┴──────┴───────┘
```

**适用条件**（Redis 7.0 前是 listpack 替代）：
- Hash：field 数量 <= 512 且所有 value <= 64 字节
- ZSet：元素数量 <= 128 且所有元素 <= 64 字节

**缺点**：连锁更新（一个节点扩容导致后续所有节点都要扩容）。

### 2.3 quicklist（快速列表）

List 的底层结构，是 ziplist 和 linkedlist 的结合。

```
quicklist 结构：
┌──────────┐    ┌──────────┐    ┌──────────┐
│  node 1  │ ⇄  │  node 2  │ ⇄  │  node 3  │
│ ziplist  │    │ ziplist  │    │ ziplist  │
│[a,b,c,d] │    │[e,f,g,h] │    │[i,j,k,l] │
└──────────┘    └──────────┘    └──────────┘
```

- 每个 node 是一个 ziplist，node 之间用双向指针连接
- 兼顾内存紧凑（ziplist）和插入效率（linkedlist）
- 可配置每个 ziplist 的大小（`list-max-ziplist-size`）

### 2.4 skiplist（跳表）

ZSet 的底层结构之一（元素多时使用）。

```
Level 3:  1 ───────────────────────────── 9
Level 2:  1 ──────────── 5 ────────────── 9
Level 1:  1 ──── 3 ──── 5 ──── 7 ─────── 9
Level 0:  1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9
```

- 平均 O(logN) 查找，最坏 O(N)
- 每个节点随机决定层数（最高 32 层）
- 查找时从最高层往下找，每层比目标大就往下走

**为什么用跳表不用红黑树？**

| 对比项 | 跳表 | 红黑树 |
|--------|------|--------|
| 实现复杂度 | 简单 | 复杂（旋转+变色） |
| 范围查询 | 极方便（底层链表遍历） | 需要中序遍历 |
| 内存占用 | 随机层数，平均 1.33 指针/节点 | 固定 2-3 指针/节点 |
| 并发友好 | 容易加锁（局部） | 难（全局调整） |

### 2.5 hashtable（哈希表）

```c
struct dictht {
    dictEntry **table;    // 哈希表数组
    unsigned long size;   // 哈希表大小
    unsigned long sizemask;
    unsigned long used;
};
```

**渐进式 rehash**：
- Redis 的字典有两个哈希表 ht[0] 和 ht[1]
- 扩容时：分配 ht[1]，逐步将 ht[0] 的元素迁移到 ht[1]
- 每次 CRUD 操作时迁移一小部分，不会阻塞
- 迁移完成后 ht[0] 指向 ht[1]，ht[1] 置空

**触发 rehash 条件**：
- 扩容：负载因子 >= 1 且没有进行 BGSAVE/BGREWRITEAOF；或负载因子 >= 5
- 缩容：负载因子 < 0.1

### 2.6 intset（整数集合）

```c
struct intset {
    uint32_t encoding;   // 编码方式：int16/int32/int64
    uint32_t length;     // 元素个数
    int8_t contents[];   // 整数数组（实际类型由 encoding 决定）
};
```

- Set 的底层结构之一（元素都是整数且数量 <= 512 时）
- 有序、连续内存
- 升级不降级：添加大整数时升级编码（int16→int32→int64）

---

## 三、各数据类型对应的底层结构映射表

| 数据类型 | 编码（底层结构） | 条件 |
|---------|----------------|------|
| String | int | 值为整数且 <= LONG_MAX |
| String | embstr | 字符串长度 <= 44 字节 |
| String | raw | 字符串长度 > 44 字节 |
| List | quicklist（listpack） | 始终 |
| Hash | ziplist（listpack） | field <= 512 且 value <= 64B |
| Hash | hashtable | 不满足上述条件 |
| Set | intset | 元素都是整数且 <= 512 个 |
| Set | hashtable | 不满足上述条件 |
| ZSet | ziplist（listpack） | 元素 <= 128 且每个 <= 64B |
| ZSet | skiplist + hashtable | 不满足上述条件 |

### String 的三种编码

```
int    → 整数值，直接存在 ptr 指针位置（节省内存）
embstr → 短字符串，SDS 和 RedisObject 一次分配（一次内存分配）
raw    → 长字符串，SDS 和 RedisObject 分开分配（两次内存分配）
```

---

## 面试题精选

### 1. Redis 为什么快？
> 纯内存操作、单线程避免上下文切换、IO 多路复用、高效的数据结构（SDS/跳表/ziplist）。

### 2. SDS 和 C 字符串有什么区别？
> SDS 有 len 字段 O(1) 获取长度；二进制安全不靠 \0 判断结尾；自动扩容防溢出；空间预分配和惰性释放减少内存重分配。

### 3. 为什么 ZSet 用跳表不用红黑树？
> 跳表实现简单、范围查询方便（底层链表顺序遍历）、内存可控、并发友好。红黑树实现复杂，范围查询需要中序遍历。

### 4. Redis 的渐进式 rehash 是什么？
> 字典扩容时不一次性迁移所有数据，而是在每次 CRUD 操作时迁移一小部分，分摊到多次操作中完成，避免长时间阻塞。

### 5. String 的三种编码是什么？
> int（整数值直接存）、embstr（短字符串 <= 44B，一次内存分配）、raw（长字符串，两次内存分配）。

### 6. 什么时候 Hash 用 ziplist，什么时候用 hashtable？
> field 数量 <= 512 且所有 value <= 64 字节时用 ziplist 省内存，否则用 hashtable。阈值由 `hash-max-ziplist-entries` 和 `hash-max-ziplist-value` 控制。

### 7. Set 的底层数据结构是什么？
> 元素全是整数且数量 <= 512 时用 intset（有序紧凑），否则用 hashtable。

### 8. Redis List 的底层结构是什么？
> quicklist，由多个 ziplist（Redis 7.0 后是 listpack）通过双向链表连接，兼顾内存紧凑和操作效率。

---

## 常见误区

| 误区 | 正解 |
|------|------|
| Redis 只有 String/List/Hash/Set/ZSet | 还有 Bitmap/HyperLogLog/GEO/Stream |
| ZSet 底层就是跳表 | 小数据量用 ziplist，大数据量用 skiplist + hashtable |
| String 都是 SDS | 整数用 int 编码，不创建 SDS |
| Set 只有无序功能 | 支持交集/并集/差集运算，非常适合社交关系 |
| Redis 数据类型和底层结构一一对应 | 一种类型可能有多种底层编码，根据数据量动态选择 |

---

## 实战场景

**场景 1：用 ZSet 实现排行榜**
```bash
# 添加分数
ZADD game_rank 100 "player1" 95 "player2" 88 "player3"
# TOP 10
ZREVRANGE game_rank 0 9 WITHSCORES
# 查某玩家排名
ZREVRANK game_rank "player1"
# 加分
ZINCRBY game_rank 5 "player2"
```

**场景 2：用 Bitmap 实现用户签到**
```bash
# 用户 1001 在第 100 天签到
SETBIT sign:1001 100 1
# 检查第 100 天是否签到
GETBIT sign:1001 100
# 统计连续签到天数
BITCOUNT sign:1001
```

---

## 关联知识

- [31-Redis-持久化与集群](31-Redis-持久化与集群.md) — 数据类型的持久化方式
- [32-Redis-缓存问题](32-Redis-缓存问题.md) — 数据类型在缓存场景的最佳实践
- [27-MySQL-存储引擎与索引](27-MySQL-存储引擎与索引.md) — B+ 树 vs 跳表的对比思路
