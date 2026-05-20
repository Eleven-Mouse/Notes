
Redis 知识体系
>    1. 基础认知
        Redis 定位
        典型场景
        单线程 + I/O 多路复用
        Key / DB / TTL
        常用客户端 
	  2. 数据类型
        基础类型
          String
          Hash
          List
          Set
          ZSet
        扩展类型
          Bitmap
          HyperLogLog
          Geo
          Stream
        选型原则
          复杂度
          内存占用
          访问模式
          可维护性
      3. 命令与交互
        常用命令
        Scan 系列
        Pipeline
        事务
          MULTI
          EXEC
          WATCH
        Lua 脚本
        Pub/Sub
        Keyspace Notifications
        Client-side Caching
        RESP3
      4. 内部原理
        对象模型
          redisObject
          SDS
          dict
          intset
          listpack
          quicklist
          skiplist
        过期与淘汰
          惰性删除
          定期删除
          淘汰策略
        执行模型
          单线程执行
          I/O 多路复用
          I/O 线程
        内存治理
          内存碎片
          active defrag
      5. 持久化
        RDB
        AOF
        混合持久化
        Rewrite
        Fsync 策略
        故障恢复
      6. 复制与高可用
        主从复制
        PSYNC
        复制积压缓冲区
        Sentinel 哨兵
        故障转移
        Replica 只读
      7. 集群与扩容
        Cluster 分片集群
        16384 槽
        Hash Tag
        MOVED / ASK
        迁移与重分片
        读写分离
      8. 编程与扩展
        Redis Functions
        Lua 脚本
        Modules
        Redis Stack
          Search
          JSON
          Vector Search
          TimeSeries
          Bloom
      9. 消息与事件
        Pub/Sub 订阅
        Sharded Pub/Sub
        Streams
        Consumer Group
        PEL
        XPENDING
        消息恢复
      10. 安全
        ACL
        AUTH
        TLS
        Protected Mode
        网络隔离
      11. 可观测性与运维
        INFO
        SLOWLOG
        LATENCY
        MONITOR
        MEMORY USAGE
        Benchmark
        监控告警
        容量规划
        备份恢复
      12. 性能与治理
        BigKey
        HotKey
        缓存穿透
        缓存击穿
        缓存雪崩
        连接池
        超时与重试
        内存优化
        数据分片
        TTL 抖动
