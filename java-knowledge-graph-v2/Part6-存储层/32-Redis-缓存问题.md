
> 优先级：⭐⭐⭐⭐⭐ | 面试频率：🔥🔥🔥🔥🔥
> 缓存穿透/击穿/雪崩、数据一致性是面试必考，实战中频率极高。

---

## 知识树

```
Redis 缓存问题
├── 三大经典问题
│   ├── 缓存穿透（查不存在的数据）
│   ├── 缓存击穿（热点 key 过期）
│   └── 缓存雪崩（大量 key 同时过期）
├── 缓存与数据库一致性
│   ├── Cache Aside 模式
│   ├── 延迟双删
│   ├── MQ 最终一致性
│   └── Canal 监听 Binlog
├── 过期与淘汰策略
│   ├── 过期删除（惰性+定期）
│   └── 内存淘汰（8种）
├── 热点 Key
│   ├── 检测
│   └── 处理
└── 大 Key
    ├── 发现
    └── 处理
```

---

## 一、缓存穿透

**问题**：查询一个数据库中一定不存在的数据，请求穿过缓存直接打到数据库。

```
用户 → Redis（没有）→ MySQL（没有）→ 返回 null
         ↑ 不缓存 null
用户 → Redis（没有）→ MySQL（没有）→ ...（反复击穿）
```

### 解决方案

**方案1：缓存空值**
```java
public User getUser(Long id) {
    User user = redis.get("user:" + id);
    if (user != null) {
        return "NULL".equals(user) ? null : user;
    }
    user = db.queryById(id);
    if (user == null) {
        redis.set("user:" + id, "NULL", 60);  // 缓存空值，短过期
    } else {
        redis.set("user:" + id, user, 3600);
    }
    return user;
}
```

**方案2：布隆过滤器（推荐）**
```
                    ┌──────────────┐
请求 → 布隆过滤器 → │ 可能存在：放行  │
                    │ 一定不存在：拒绝│
                    └──────────────┘
```
- 位图 + 多次哈希，判断"一定不存在"或"可能存在"
- 有误判率（可配置），但不漏判

---

## 二、缓存击穿

**问题**：一个热点 key 在某个时刻过期，大量并发请求瞬间打到数据库。

```
热点 key 过期
    ↓
1000 个并发请求同时发现缓存 miss
    ↓
1000 个请求同时查询数据库（db 压力暴增）
```

### 解决方案

**方案1：互斥锁（推荐）**
```java
public User getUser(Long id) {
    User user = redis.get("user:" + id);
    if (user != null) return user;

    String lockKey = "lock:user:" + id;
    // 尝试获取互斥锁（SET NX EX）
    if (redis.set(lockKey, "1", "NX", "EX", 10)) {
        try {
            user = db.queryById(id);       // 只有一个线程查DB
            redis.set("user:" + id, user, 3600);
        } finally {
            redis.del(lockKey);
        }
    } else {
        Thread.sleep(50);                   // 其他线程等待后重试
        return getUser(id);
    }
    return user;
}
```

**方案2：逻辑过期（不设真实 TTL）**
```java
// 缓存中存逻辑过期时间，不设 Redis TTL
@Data
class CacheData {
    User data;
    LocalDateTime expireTime;
}

// 发现逻辑过期 → 返回旧数据 + 异步线程更新
if (cacheData.getExpireTime().isBefore(LocalDateTime.now())) {
    executor.submit(() -> {
        // 加锁 + 查DB + 更新缓存
    });
    return cacheData.getData();  // 先返回旧数据
}
```

| 方案 | 优点 | 缺点 |
|------|------|------|
| 互斥锁 | 保证一致性 | 有等待时间 |
| 逻辑过期 | 无等待 | 可能返回旧数据 |

---

## 三、缓存雪崩

**问题**：大量 key 同时过期，或 Redis 宕机，请求全部打到数据库。

### 解决方案

1. **随机过期时间**：`TTL = base + random(0, 300s)`
2. **多级缓存**：本地缓存（Caffeine）→ Redis → 数据库
3. **熔断降级**：Hystrix/Sentinel 保护数据库
4. **Redis 高可用**：Sentinel / Cluster
5. **永不过期 + 异步更新**：热点数据不设过期

---

## 四、三者对比

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| **穿透** | 查不存在的数据 | 布隆过滤器 / 缓存空值 |
| **击穿** | 热点 key 过期 | 互斥锁 / 逻辑过期 |
| **雪崩** | 大量 key 同时过期或宕机 | 随机过期 / 多级缓存 / 高可用 |

---

## 五、缓存与数据库一致性

### Cache Aside 模式（标准方案）

```
读：先读缓存 → miss → 读数据库 → 写入缓存
写：先更新数据库 → 再删除缓存
```

### 为什么删除缓存而不是更新缓存？

| 操作 | 并发问题 | 复杂度 |
|------|---------|--------|
| 更新缓存 | 并发写可能导致缓存与DB不一致 | 高（每次写都要更新缓存） |
| **删除缓存** | 懒加载，下次读时重建 | 低 |

### 延迟双删策略

解决"先更新DB再删缓存"的极端并发问题：

```
1. 删除缓存
2. 更新数据库
3. 睡眠 500ms（读操作的耗时）
4. 再次删除缓存（防止睡眠期间旧数据被读请求回填）
```

```java
redis.del("user:" + id);              // 第一次删除
db.update(user);                       // 更新DB
Thread.sleep(500);                     // 延迟
redis.del("user:" + id);              // 第二次删除
```

### 基于 MQ 的最终一致性

```
更新数据库 → 发送 MQ 消息 → 消费者删除缓存
                           ↓ 消费失败
                           → 重试 → 死信队列 → 人工处理
```

### Canal 监听 Binlog 方案

```
MySQL Binlog → Canal（伪装从节点）→ 解析变更 → 删除/更新 Redis
```

- **优点**：应用代码无需关心缓存，完全解耦
- **缺点**：引入 Canal 组件，增加复杂度

---

## 六、过期删除策略

### 惰性删除

不主动删除，访问 key 时检查是否过期，过期则删除。

- 优点：CPU 友好
- 缺点：内存不友好（过期 key 可能长期占内存）

### 定期删除

每隔一段时间随机检查一批 key，删除过期的。如果过期比例超过 25%，继续检查。

- 优点：折中方案
- 缺点：难以确定执行频率

**Redis 采用：惰性删除 + 定期删除（配合使用）。**

---

## 七、内存淘汰策略（8 种）

当 Redis 内存使用达到 `maxmemory` 限制时：

| 策略 | 说明 |
|------|------|
| noeviction | 不淘汰，写入返回错误（默认） |
| allkeys-lru | 所有 key 中淘汰最久未使用的 |
| allkeys-lfu | 所有 key 中淘汰使用频率最低的（Redis 4.0+） |
| allkeys-random | 所有 key 中随机淘汰 |
| volatile-lru | 设置了过期时间的 key 中淘汰最久未使用的 |
| volatile-lfu | 设置了过期时间的 key 中淘汰频率最低的 |
| volatile-random | 设置了过期时间的 key 中随机淘汰 |
| volatile-ttl | 设置了过期时间的 key 中淘汰 TTL 最短的 |

**生产建议**：
- 缓存场景：`allkeys-lru` 或 `allkeys-lfu`
- 混合使用（缓存+持久数据）：`volatile-lru`

---

## 八、热点 Key 与大 Key

### 热点 Key 检测与处理

**检测方法**：
- `redis-cli --hotkeys`（Redis 4.0+）
- `MONITOR` 命令统计（生产慎用，有性能开销）
- 代理层/客户端统计

**处理方法**：
1. 本地缓存（Caffeine）挡一层
2. 打散热点 key：`key_1`、`key_2`...`key_N`，随机读一个
3. 永不过期 + 异步刷新

### 大 Key 问题

**什么是大 Key？**
- String > 10KB
- Hash/List/Set/ZSet 元素超过 5000 个

**危害**：
- 阻塞其他请求（操作大 key 耗时长）
- 网络传输慢
- 内存不均匀（Cluster 模式下）

**发现方法**：
```bash
redis-cli --bigkeys          # 扫描大 key
MEMORY USAGE key             # 查看单个 key 内存占用
RDB 分析工具：rdb-tools
```

**处理方法**：
- 拆分大 key：一个 Hash 拆成多个小 Hash
- 定期清理：`HSCAN + HDEL`（不要直接 DEL 大 key）
- 压缩：存储前压缩 value

---

## 面试题精选

### 1. 什么是缓存穿透？怎么解决？
> 查询数据库中不存在的数据，请求穿过缓存直接打到DB。解决：布隆过滤器（拦截不存在的 key）、缓存空值（短过期）。

### 2. 缓存击穿和缓存雪崩的区别？
> 击穿是单个热点 key 过期，大量并发请求打到DB；雪崩是大量 key 同时过期或 Redis 宕机，请求全部打到DB。

### 3. 怎么保证缓存和数据库的一致性？
> 标准方案是 Cache Aside：先更新DB再删缓存。强一致性很难保证，可以用延迟双删、MQ 最终一致性、Canal 监听 Binlog 来提升一致性。

### 4. 为什么是删缓存而不是更新缓存？
> 更新缓存在并发写场景可能导致数据不一致，且每次写都要更新缓存开销大。删缓存是懒加载，下次读时重建，简单且一致性好。

### 5. Redis 的过期删除策略是什么？
> 惰性删除 + 定期删除。访问时检查过期则删除（惰性），同时定期随机抽样检查并删除过期 key。二者配合使用。

### 6. Redis 有哪些内存淘汰策略？生产用哪个？
> 8种：noevication/allkeys-lru/allkeys-lfu/allkeys-random/volatile-lru/volatile-lfu/volatile-random/volatile-ttl。纯缓存场景推荐 allkeys-lru 或 allkeys-lfu。

### 7. 什么是大 Key？怎么处理？
> String > 10KB 或集合元素 > 5000 的 key。危害：阻塞、网络慢、内存不均。处理：拆分、HSCAN+HDEL 渐进删除、压缩。

### 8. 布隆过滤器的原理？有什么缺点？
> 位数组 + 多次哈希，判断"一定不存在"或"可能存在"。缺点：有误判率（可配置降低）、不支持删除（Counting Bloom Filter 支持）。

### 9. 延迟双删是什么？
> 更新DB前删一次缓存，更新DB后延迟一段时间再删一次。防止更新DB期间读请求把旧数据写回缓存。延迟时间应大于一次读请求的耗时。

### 10. 热点 Key 怎么处理？
> 检测：`--hotkeys` 或代理层统计。处理：本地缓存、打散热点 key（多个 key 存相同数据）、永不过期 + 异步刷新。

---

## 常见误区

| 误区 | 正解 |
|------|------|
| 先删缓存再更新DB | 并发时可能导致读到旧数据写回缓存（应先更新DB再删缓存） |
| 设置相同过期时间 | 会导致同时过期引发雪崩 |
| 用 DEL 删大 key | 大 key 的 DEL 会阻塞主线程，应渐进删除 |
| 布隆过滤器 100% 准确 | 有误判率，只能说"一定不存在"或"可能存在" |
| Redis 能保证强一致性 | Redis 是缓存，只能做到最终一致性 |

---

## 实战场景

**场景 1：电商商品详情缓存**
```java
// Cache Aside + 随机过期 + 互斥锁
String key = "product:" + productId;
String data = redis.get(key);
if (data == null) {
    if (redis.set("lock:" + productId, "1", "NX", "EX", 5)) {
        Product p = db.queryById(productId);
        int ttl = 3600 + new Random().nextInt(600); // 随机过期
        redis.set(key, JSON.toJSONString(p), ttl);
        redis.del("lock:" + productId);
    }
}
```

**场景 2：大 Hash 渐进删除**
```bash
# 不要直接 DEL 大 Hash
# HSCAN + HDEL 渐进删除
redis-cli --scan --pattern "big_hash" | xargs -L 1 hscan | xargs hdel
```

---

## 关联知识

- [30-Redis-数据类型](./30-Redis-数据类型.md) — 不同数据类型的缓存设计选择
- [31-Redis-持久化与集群](./31-Redis-持久化与集群.md) — 集群高可用防止雪崩
- [27-MySQL-存储引擎与索引](./27-MySQL-存储引擎与索引.md) — 缓存与 MySQL 的配合优化
- [28-MySQL-事务与MVCC](./28-MySQL-事务与MVCC.md) — Canal 监听 Binlog 的前提
