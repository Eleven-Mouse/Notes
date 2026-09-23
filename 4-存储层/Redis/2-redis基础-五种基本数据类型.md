---
publishTime: 2026-05-19T14:41:48+08:00
---


>这是redis最基础的知识，但是不代表它们就好学


## String 字符串

Redis string字符串存储**字节序列**，包括文本，序列化对象和二进制数组，它常用于缓存但也支持其他功能，例如实现计数器和执行位运算
Redis 数据库中，字符串是二进制安全的
由于redis 的 key 是字符串类型，当我们使用字符串类型作为 value 时，实际上是将一个字符串 映射到另一个字符串上

---

- 命令使用

| 命令     | 简述                    | 使用                |
| ------ | --------------------- | ----------------- |
| GET    | 获取存储在给定 key 中的 value | GET name          |
| SET    | 设置存储在给定 key 中的 value  | SET name value    |
| DEL    | 删除存储在给定 key 中的 value  | DEL name          |
| INCR   | 将 key 存储的值加1(原子性)     | INCR key          |
| DECR   | 将 key 存储的值减1(原子性)     | DECR key          |
| INCRBY | 将 key 存储的值加上amount    | INCRBY key amount |
| DECRBY | 将 key 存储的值减去amount    | DECRBY key amount |

-----
- 命令执行
```plaintext
> SET bike:1 Deimos
OK
> GET bike:1
"Deimos"

> MSET bike:1 "Deimos" bike:2 "Ares" bike:3 "Vanth"
OK
> MGET bike:1 bike:2 bike:3
1) "Deimos"
2) "Ares"
3) "Vanth"
```
> Tips: [`SET`]如果键已存在，即使该键当前关联的是非字符串值，`SET` 也会覆盖旧值并完成重新赋值。

-----
- 字符串应用
	字符串作为计数器
```
> SET total_crashes 0
OK
> INCR total_crashes
(integer) 1
> INCRBY total_crashes 10
(integer) 11
```
>该[`INCR`]命令将字符串值解析为整数，将其加一，最后将结果写回。类似命令还有 [`INCRBY`]、[`DECR`]、[`DECRBY`]，核心逻辑一致，差别只是增减幅度不同。

- 补充
 1，Redis 的单个 string 最大可到 512MB。  
 2，大多数字符串命令如 `GET`、`SET` 为 O(1)。  
 3，`GETRANGE`、`SETRANGE` 这类按偏移读取/写入子串的命令复杂度与操作长度相关，处理超大字符串时要谨慎。  
 4，`SUBSTR` 是历史命令，官方标记为 deprecated，新代码建议使用 `GETRANGE`。

-----

## List 列表

Redis 列表是按插入顺序排序的字符串序列。可以将元素添加到头部或尾部。
Redis 列表常用于：
- 实现栈和队列。
- 为后台工作系统构建队列管理。
----

- 命令使用

| 命令                          | 介绍                          |     |
| --------------------------- | --------------------------- | --- |
| RPUSH key value1 value2 ... | 在指定列表的尾部（右边）添加一个或多个元素       |     |
| LPUSH key value1 value2 ... | 在指定列表的头部（左边）添加一个或多个元素       |     |
| LSET key index value        | 将指定列表索引 index 位置的值设置为 value |     |
| LPOP key                    | 移除并获取指定列表的第一个元素(最左边)        |     |
| RPOP key                    | 移除并获取指定列表的最后一个元素(最右边)       |     |
| LLEN key                    | 获取列表元素数量                    |     |
| LRANGE key start end        | 获取列表 start 和 end 之间 的元素     |     |

----

- 命令执行
```
lpush rediscomcn java  
(integer) 1  
lpush rediscomcn sql 
(integer) 1  
lpush rediscomcn mongodb 
(integer) 1  
lpush rediscomcn cassandra 
(integer) 1  
lrange rediscomcn 0 10  
"cassandra"  
"mongodb"  
"sql"  
"java"
```

----
- 应用场景
	1，记住社交网络上用户发布的最新动态。
	2，进程间的通信采用生产者-消费者模式，其中生产者将数据项添加到列表中，消费者（通常是_工作进程_）消费这些数据项并执行相应的操作。Redis 提供了特殊的列表命令，使这种使用场景更加可靠高效。
	3，`List` 可以实现简单消息队列，但能力有限（例如消费组、重放、复杂路由等能力不足）。

相对来说，Redis 5.0 引入的 `Stream` 更适合消息队列场景（支持消费组、`XACK` 等机制）；但在大规模堆积治理、跨机房容灾、复杂投递语义等方面，通常仍不如专业 MQ 产品完善。

----
- 补充
>列表的最大长度为 2的32次方 – 1 个元素（超过 40 亿个元素）
>常见复杂度：`LPUSH/RPUSH/LPOP/RPOP/LLEN` 通常为 O(1)；`LRANGE` 为 O(S+N)（S 为起始偏移），`LSET` 为 O(N)。
>实现细节上，现代 Redis 的 list 底层采用 quicklist（由多个 listpack 组成），不再是早期文档常见的“纯双向链表”描述。



-----
## Hash 哈希

哈希是键值对的集合。在 Redis 中，哈希是字符串字段和字符串值之间的映射。因此，它们适合表示对象。

----
- 命令使用

| 命令                                        | 介绍                                   |
| ----------------------------------------- | ------------------------------------ |
| HSET key field value                      | 设置指定哈希表中指定字段的值                       |
| HSETNX key field value                    | 只有指定字段不存在时设置指定字段的值                   |
| HSET key field1 value1 field2 value2 ...  | 同时将一个或多个 field-value (域-值)对设置到指定哈希表中 |
| HGET key field                            | 获取指定哈希表中指定字段的值                       |
| HMGET key field1 field2 ...               | 获取指定哈希表中一个或者多个指定字段的值                 |
| HGETALL key                               | 获取指定哈希表中所有的键值对                       |
| HEXISTS key field                         | 查看指定哈希表中指定的字段是否存在                    |
| HDEL key field1 field2 ...                | 删除一个或多个哈希表字段                         |
| HLEN key                                  | 获取指定哈希表中字段的数量                        |
| HINCRBY key field increment               | 对指定哈希中的指定字段做运算操作（正数为加，负数为减）          |

----
- 命令执行
```
HSET userInfoKey name "guide" description "dev" age 24
OK
HEXISTS userInfoKey name # 查看 key 对应的 value中指定的字段是否存在。
(integer) 1
HGET userInfoKey name # 获取存储在哈希表中指定字段的值。
"guide"
HGET userInfoKey age
"24"
HGETALL userInfoKey # 获取在哈希表中指定 key 的所有字段和值
1) "name"
2) "guide"
3) "description"
4) "dev"
5) "age"
6) "24"
HSET userInfoKey name "GuideGeGe"
HGET userInfoKey name
"GuideGeGe"
HINCRBY userInfoKey age 2
(integer) 26
```
-----
- **字段过期**
Redis 7.4 引入了为单个哈希字段指定过期时间或生存时间 (TTL) 值的功能。此功能类似于[键过期]，并包含许多类似的命令。

-  使用以下命令可以为特定字段设置精确的过期时间或 TTL 值：
	- [`HEXPIRE`]：设置剩余 TTL 值（以秒为单位）。
	- [`HPEXPIRE`]：设置剩余的 TTL 值（以毫秒为单位）。
	- [`HEXPIREAT`]: 将过期时间设置为以秒为单位指定的时间戳。
	- [`HPEXPIREAT`]: 将过期时间设置为以毫秒为单位的指定时间戳。
- 使用以下命令可以检索特定字段的确切过期时间或剩余 TTL：
	- [`HEXPIRETIME`]：获取以秒为单位的时间戳作为过期时间。
	- [`HPEXPIRETIME`]：获取过期时间，以毫秒为单位的时间戳。
	- [`HTTL`]：获取剩余的 TTL（以秒为单位）。
	- [`HPTTL`]：获取剩余的 TTL（以毫秒为单位）。
- 使用以下命令移除特定字段的过期时间：
	- [`HPERSIST`]：移除过期时间。

Redis 8.0 引入了以下命令：
- [`HGETEX`]：获取给定哈希键的一个或多个字段的值，并可选择设置其过期时间或生存时间（TTL）。
- [`HSETEX`]：设置给定哈希键的一个或多个字段的值，并可选择设置其过期时间或生存时间（TTL）。

---

- **常见的字段过期使用场景：**
 1. **事件跟踪**：使用哈希键存储过去一小时内的事件。将每个事件的生存时间 (TTL) 设置为一小时。用于`HLEN`统计过去一小时内的事件数量。
    
2. **欺诈检测**：创建一个包含事件每小时计数器的哈希表。将每个字段的 TTL 设置为 48 小时。查询该哈希表以获取过去 48 小时内每小时的事件数。
    
3. **客户会话管理**：将客户数据存储在哈希键中。为每个会话创建一个新的哈希键，并在客户的哈希键中添加一个会话字段。当会话过期时，会话键和客户哈希键中的会话字段都会自动过期。
    
4. **活动会话跟踪**：将所有活动会话存储在哈希键中。设置每个会话的生存时间 (TTL)，使其在不活动后自动过期。用于`HLEN`统计活动会话数。

---
- 补充
>1，每个哈希可以存储多达 2的32次方 - 1 个键-值对。实际上，哈希表的大小仅受限于托管 Redis 部署的虚拟机上的总内存。
>2，大多数 Redis 哈希命令的时间复杂度为 O(1)。
>少数命令如 [`HKEYS`]、[`HVALS`]、[`HGETALL`] 为 O(n)，其中 _n_ 是字段数量。
>`HMSET` 自 Redis 4.0 起标记为 deprecated，建议统一使用可一次写入多个 field-value 的 `HSET`。

-----
## SET 集合

集合（set）是 Redis 中的无序字符串集合。`SADD`、`SREM`、`SISMEMBER` 这类单元素操作通常是 O(1)，但集合运算和全量读取不是 O(1)。

----
- 命令使用

|命令|介绍|
|---|---|
|SADD key member1 member2 ...|向指定集合添加一个或多个元素|
|SMEMBERS key|获取指定集合中的所有元素|
|SCARD key|获取指定集合的元素数量|
|SISMEMBER key member|判断指定元素是否在指定集合中|
|SINTER key1 key2 ...|获取给定所有集合的交集|
|SINTERSTORE destination key1 key2 ...|将给定所有集合的交集存储在 destination 中|
|SUNION key1 key2 ...|获取给定所有集合的并集|
|SUNIONSTORE destination key1 key2 ...|将给定所有集合的并集存储在 destination 中|
|SDIFF key1 key2 ...|获取给定所有集合的差集|
|SDIFFSTORE destination key1 key2 ...|将给定所有集合的差集存储在 destination 中|
|SPOP key count|随机移除并获取指定集合中一个或多个元素|
|SRANDMEMBER key count|随机获取指定集合中指定数量的元素|
- 命令执行
```
SADD mySet value1 value2
(integer) 2
SADD mySet value1 # 不允许有重复元素，因此添加失败
(integer) 0
SMEMBERS mySet
1) "value1"
2) "value2"
SCARD mySet
(integer) 2
SISMEMBER mySet value1
(integer) 1
SADD mySet2 value2 value3
(integer) 2
```

- **适用场景**
	你可以使用 Redis 集合高效地执行以下操作：
		1，跟踪唯一项目（例如，跟踪访问给定博客文章的所有唯一 IP 地址）。
		2，表示关系（例如，具有给定角色的所有用户的集合）。
		3，执行常见的集合运算，例如交集、并集和差集。

- 补充
>1，集合中的最大成员数为 2的32次方 -1 个元素（超过 40 亿个元素）。
>2，大多数单元素操作（添加、删除、成员判断）时间复杂度为 O(1)。
>3，`SMEMBERS` 为 O(n)，会一次性返回全集合；大 Key 场景建议使用 [`SSCAN`](https://redis.io/docs/latest/commands/sscan/) 分批遍历，避免单次响应过大。


-----
## ZSET 有序集合
Redis 有序集合类似于 Redis 集合，也是一组非重复的字符串集合。但是，排序集的每个成员都与一个分数相关联，该分数用于获取从最小到最高分数的有序排序集。虽然成员是独特的，但可以重复分数。 

---
有序集合（ZSET）在 Redis 中的核心实现是：
	1，哈希表：用于 `member -> score` 的快速定位。  
	2，跳跃表（skiplist）：用于按 score 有序维护，支持范围查询和排名。  
对于较小的 ZSET，Redis 会使用更紧凑的 `listpack` 编码以节省内存（早期版本常见 `ziplist` 说法，现已被 `listpack` 取代）。
	
-------

- 命令使用

|命令|介绍|
|---|---|
|ZADD key score1 member1 score2 member2 ...|向指定有序集合添加一个或多个元素|
|ZCARD KEY|获取指定有序集合的元素数量|
|ZSCORE key member|获取指定有序集合中指定元素的 score 值|
|ZINTERSTORE destination numkeys key1 key2 ...|将给定所有有序集合的交集存储在 destination 中，对相同元素对应的 score 值进行 SUM 聚合操作，numkeys 为集合数量|
|ZUNIONSTORE destination numkeys key1 key2 ...|求并集，其它和 ZINTERSTORE 类似|
|ZDIFFSTORE destination numkeys key1 key2 ...|求差集，其它和 ZINTERSTORE 类似|
|ZRANGE key start end|获取指定有序集合 start 和 end 之间的元素（score 从低到高）|
|ZREVRANGE key start end|获取指定有序集合 start 和 end 之间的元素（score 从高到底）|
|ZREVRANK key member|获取指定有序集合中指定元素的排名(score 从大到小排序)|

----
- 命令执行
	```
ZADD myZset 2.0 value1 1.0 value2
(integer) 2
ZCARD myZset
2
ZSCORE myZset value1
2.0
ZRANGE myZset 0 1
1) "value2"
2) "value1"
ZREVRANGE myZset 0 1
1) "value1"
2) "value2"
ZADD myZset2 4.0 value2 3.0 value3
(integer) 2
	```

----

- 适用场景：
	1，**需要根据权重/分数进行排序并快速查询排名的场景**
	例如：排行榜，相关命令：`ZRANGE` (从小到大排序)、`ZREVRANGE`（从大到小排序）、`ZREVRANK` (指定元素排名)。
	2，常见复杂度：`ZADD`、`ZREM`、`ZRANK` 通常为 O(logN)；`ZRANGE` 为 O(logN + M)（M 为返回元素数量）。
