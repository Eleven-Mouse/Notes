---
publishTime: 2026-07-23T22:52:14+08:00
---

# HDFS 学习路线：以博客访问日志为主

## 结论

HDFS 是 Hadoop 的分布式文件系统，适合存储**海量大文件**。它的核心价值是：**用多台普通机器组成一个大容量、高容错的文件系统。**

它不适合大量小文件，也不适合频繁随机修改数据。

在博客项目里，HDFS 最自然的用途是：**保存访问日志、行为日志、历史归档数据，作为 Spark 分析的输入。**

不要用 HDFS 存博客文章正文、用户信息、评论这些在线业务数据，这些仍然更适合 MySQL 这类数据库。

## 博客项目主线

推荐链路：

```text
博客服务产生日志
  |
  v
按天切分日志文件
  |
  v
上传或采集到 HDFS
  |
  v
Spark 读取 HDFS 做统计
```

推荐目录：

```text
/data/blog/access_log/dt=2026-07-23/access.log
/data/blog/access_log/dt=2026-07-24/access.log
```

HDFS 在博客项目里的边界：

| 数据 | 是否适合 HDFS | 原因 |
| --- | --- | --- |
| 访问日志 | 适合 | 追加写、大文件、离线分析 |
| 用户行为日志 | 适合 | 适合长期归档和批处理 |
| 文章正文 | 不适合 | 需要在线查询和修改 |
| 评论数据 | 不适合 | 属于事务型业务数据 |
| 统计中间结果 | 可选 | 数据量大时可落 HDFS |

## 它是什么

HDFS 全称是 Hadoop Distributed File System。

它把一个大文件切成多个 Block，然后分散存到不同机器上：

```text
大文件 log.csv
  |
  |-- Block 1 -> DataNode A / B / C
  |-- Block 2 -> DataNode B / C / D
  |-- Block 3 -> DataNode A / C / D
```

核心角色：

- **NameNode**：管理文件目录、Block 元数据、DataNode 状态。
- **DataNode**：真正存储数据块。
- **Client**：负责上传、下载、读取文件。

## 什么时候用

适合用 HDFS：

- 存储 TB、PB 级别数据。
- 文件很大，比如日志、离线数据、历史归档。
- 读多写少。
- 批处理场景。
- 和 Spark、Hive、MapReduce 配合。

不适合用 HDFS：

- 大量小文件。
- 高频随机写。
- 毫秒级低延迟访问。
- 事务型业务数据。
- 需要像数据库一样频繁 update 单行数据。

架构判断：

```text
HDFS 是大数据湖里的“底层仓库”，不是业务系统的在线数据库。
```

## 能做什么

HDFS 可以：

- 存原始日志。
- 存离线数仓明细数据。
- 存 Spark 输入和输出。
- 存 Hive 表文件。
- 存模型训练数据。
- 存历史归档数据。

典型链路：

```text
业务日志 -> 采集工具 -> HDFS -> Spark / Hive -> Doris / ClickHouse / 报表
```

## 架构位置

在大数据架构中，HDFS 通常位于存储层：

```text
数据源
  |
  v
采集层
  |
  v
HDFS / 对象存储
  |
  v
Spark / Hive / Flink
  |
  v
Doris / 数据集市 / BI
```

它负责“存得下”和“坏了不丢”，不负责复杂查询加速。

## 入门阶段

目标：能操作 HDFS，理解基本角色。

重点：

- NameNode
- DataNode
- Block
- 副本
- 上传文件
- 下载文件
- 查看目录

常用命令：

```bash
hdfs dfs -ls /
hdfs dfs -mkdir /data
hdfs dfs -put access.log /data/
hdfs dfs -cat /data/access.log
hdfs dfs -get /data/access.log ./
hdfs dfs -rm /data/access.log
```

必须理解：

```text
NameNode 不存文件内容，它存文件和 Block 的元数据。
DataNode 才真正存文件块。
```

## 进阶阶段

目标：能解释 HDFS 的读写流程和容错机制。

重点：

- 文件写入流程
- 文件读取流程
- 副本放置策略
- DataNode 心跳
- Block Report
- NameNode 元数据
- Secondary NameNode
- 安全模式
- 小文件问题

写入流程简化理解：

```text
Client 请求 NameNode
  -> NameNode 返回可写 DataNode
  -> Client 把 Block 写入 DataNode
  -> DataNode 之间复制副本
  -> 写入完成后更新元数据
```

读取流程简化理解：

```text
Client 请求 NameNode
  -> NameNode 返回 Block 位置
  -> Client 直接从 DataNode 读取数据
```

重点：

```text
真正的数据读写不经过 NameNode，否则 NameNode 会成为巨大瓶颈。
```

## 高级阶段

目标：能做容量规划、故障排查和架构取舍。

重点：

- NameNode HA
- Federation
- Rack Awareness
- 副本数规划
- 磁盘容量规划
- 小文件治理
- 数据冷热分层
- HDFS 权限和安全
- Kerberos
- HDFS 与对象存储的区别

高级问题：

- NameNode 挂了怎么办？
- DataNode 坏了，副本怎么恢复？
- 为什么小文件会压垮 NameNode？
- 副本数设置为 3 的成本是什么？
- HDFS 和 S3 / OSS / MinIO 该怎么选？

## 重点难点

### 小文件问题

HDFS 不怕大文件，怕海量小文件。

原因：

- 每个文件和 Block 都需要 NameNode 维护元数据。
- 小文件太多会吃掉 NameNode 内存。
- 计算任务读取小文件也会产生大量调度开销。

解决思路：

- 合并小文件。
- 使用 SequenceFile、Parquet、ORC。
- 按日期、业务线组织目录。
- 控制采集端输出文件大小。

### 副本机制

默认副本数通常是 3。

好处：

- 容错。
- 提升读吞吐。

代价：

- 存储成本变为 3 倍。
- 副本同步和修复需要网络资源。

架构取舍：

```text
重要数据副本数高一些，临时中间数据可以低一些。
```

### NameNode 单点

NameNode 负责元数据，是 HDFS 的大脑。

生产环境一般要做 HA：

```text
Active NameNode + Standby NameNode
```

否则 NameNode 故障会影响整个集群访问。

## 博客项目实战

做一个博客访问日志归档系统：

```text
博客访问日志 -> 按日期压缩 -> 上传 HDFS -> Spark 读取分析
```

目录设计：

```text
/data/blog/access_log/dt=2026-07-23/access.log
/data/blog/access_log/dt=2026-07-24/access.log
```

验收标准：

- 能按日期上传博客访问日志。
- 能用 HDFS 命令查看数据。
- 能解释文件被切成几个 Block。
- 能用 Spark 读取该目录统计 PV、UV。
- 能说明小文件治理方案。

## 学习路线

```text
HDFS 基本命令
  -> NameNode / DataNode
  -> Block 和副本
  -> 读写流程
  -> 小文件问题
  -> NameNode HA
  -> 容量规划和故障排查
```

## 面试和工作重点

必须能讲清楚：

- HDFS 为什么适合大文件。
- NameNode 和 DataNode 的职责。
- 文件读写流程。
- 副本机制怎么保证容错。
- 小文件问题为什么严重。
- NameNode HA 的意义。
- HDFS 和普通文件系统有什么区别。

## 一句话总结

HDFS 的核心是：**用多台机器可靠存储海量大文件，让 Spark、Hive 等计算引擎有地方读写数据。**
