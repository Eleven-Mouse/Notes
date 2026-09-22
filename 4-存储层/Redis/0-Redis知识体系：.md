

## 一， 基础认知
1. Redis 定位
2. 典型场景
3. 单线程 + I/O 多路复用
4. Key / DB / TTL``
5. 常用客户端


------ 

## 二， 数据类型

**A，基础类型**
   1. String
   2. Hash
   3. List
   4. Set
   5. ZSet


**B，扩展类型**
   6. Bitmap
   7. HyperLogLog
   8. Geo
   9. Stream
   10. 选型原则
   11. 复杂度
   12. 内存占用
   13. 访问模式
   14. 可维护性
-----------
## 三，内部原理
   1. 过期与淘汰
   2. 惰性删除
   3. 定期删除
   4. 淘汰策略
   5. 执行模型
   6. 单线程执行
   7. I/O 多路复用
   8. I/O 线程
   9. 内存治理
   10. 内存碎片
   11. active defrag

----------------
## 四，持久化
   1. RDB
   2. AOF
   3. 混合持久化
   4. Rewrite
   5. Fsync 策略
   6. 故障恢复


----------------
## 五，复制与高可用
   1. 主从复制
   2. PSYNC
   3. 复制积压缓冲区
   4. Sentinel 哨兵
   5. 故障转移
   6. Replica 只读


--------------------
## 六，集群与扩容
   1. Cluster 分片集群
   2. 16384 槽
   3. Hash Tag
   4. MOVED / ASK
   5. 迁移与重分片
   6. 读写分离








----------------
## 七，编程与扩展
   1. Redis Functions
   2. Lua 脚本
   3. Modules
   4. Redis Stack
   5. Search
   6. JSON
   7. Vector Search
   8. TimeSeries
   9. Bloom




-----------
## 八，消息与事件
   1. Pub/Sub 订阅
   2. Sharded Pub/Sub
   3. Streams
   4. Consumer Group
   5. PEL
   6. XPENDING
   7. 消息恢复







------------
## 九，可观测性与运维
1. INFO
2. SLOWLOG
3. LATENCY
4. MONITOR
5. MEMORY USAGE
6. Benchmark
7. 监控告警
8. 容量规划
9. 备份恢复







-------------
## 十，性能与治理
1. BigKey
2. HotKey
3. 缓存穿透
4. 缓存击穿
5. 缓存雪崩
6. 连接池
7. 超时与重试
8. 内存优化
9. 数据分片
10. TTL 抖动
