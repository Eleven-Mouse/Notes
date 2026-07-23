# Spark 学习路线：以博客访问日志分析为主

## 结论

Spark 是分布式计算引擎，核心价值是：**把大规模数据处理任务拆成很多小任务，在多台机器上并行执行。**

它适合 ETL、批处理、交互式分析、机器学习和流式处理，但不是数据库，也不是文件系统。

在博客项目里，Spark 最适合做访问日志的离线分析：**每天定时统计 PV、UV、热门文章、访问来源、状态码分布和慢请求。**

不要让用户每次访问博客都触发 Spark。Spark 是后台批处理，不是在线接口。

## 博客项目主线

推荐分析链路：

```text
HDFS 博客访问日志
  |
  v
Spark 读取
  |
  v
解析、清洗、过滤脏数据
  |
  v
按日期、文章、来源聚合
  |
  v
写入 Doris
```

优先做这些指标：

| 指标 | 含义 | 架构价值 |
| --- | --- | --- |
| PV | 页面访问次数 | 判断整体流量 |
| UV | 独立访客数 | 判断真实访问用户规模 |
| 文章 Top 10 | 热门内容 | 支撑内容优化 |
| 状态码分布 | 200、404、500 等 | 发现错误和异常 |
| 平均耗时 | cost_ms 平均值 | 发现慢接口 |
| 来源分布 | referer 统计 | 分析流量来源 |

## 它是什么

Spark 可以理解为一个大数据计算框架：

```text
输入数据
  |
  v
Spark 读取
  |
  v
转换 / 清洗 / 聚合 / Join
  |
  v
输出结果
```

它可以从很多地方读写数据：

- HDFS
- Hive
- Kafka
- MySQL
- Doris
- Parquet / ORC / JSON / CSV

## 什么时候用

适合用 Spark：

- 数据量大，单机处理慢。
- 需要复杂 ETL。
- 需要多表 Join。
- 需要离线批处理。
- 需要从 HDFS 或数据湖读取数据。
- 需要把清洗后的结果写到 Doris、Hive、MySQL 等。

不适合用 Spark：

- 毫秒级在线请求。
- 简单 CRUD。
- 小数据量任务。
- 高频单条写入。
- 替代在线数据库。

架构判断：

```text
Spark 是数据加工厂，不是在线交易系统。
```

## 能做什么

Spark 可以：

- 清洗日志。
- 统计 PV、UV、GMV。
- 做用户画像。
- 做离线数仓 ETL。
- 处理大规模 Join。
- 训练机器学习模型。
- 消费 Kafka 做准实时计算。
- 把结果写入 Doris 供报表查询。

典型链路：

```text
HDFS / Kafka / Hive
  |
  v
Spark
  |
  v
Doris / Hive / MySQL / 文件
```

## 架构位置

在大数据系统中，Spark 通常位于计算层：

```text
数据源
  |
  v
存储层：HDFS / Hive / 对象存储
  |
  v
计算层：Spark
  |
  v
服务层：Doris / API / BI
```

Spark 不负责长期存储，也不负责低延迟查询服务。

## 入门阶段

目标：能写简单 Spark 程序。

重点：

- SparkSession
- RDD
- DataFrame
- Dataset
- Transformation
- Action
- 读取文件
- 写出结果

入门代码思路：

```scala
val spark = SparkSession.builder()
  .appName("AccessLogJob")
  .getOrCreate()

val df = spark.read.option("header", "true").csv("/data/access.csv")

df.groupBy("event_type").count().show()
```

必须理解：

```text
Transformation 是定义计算过程。
Action 才会真正触发执行。
```

## 进阶阶段

目标：能写生产可用的 ETL 和分析任务。

重点：

- Spark SQL
- UDF
- Window Function
- Join
- Broadcast Join
- Shuffle
- Partition
- Cache / Persist
- Parquet / ORC
- 任务提交
- 日志排查

重点理解：

```text
Spark 慢，很多时候不是代码行数问题，而是 Shuffle、Join、数据倾斜和资源配置问题。
```

## 高级阶段

目标：能优化 Spark 任务并做架构取舍。

重点：

- Catalyst 优化器
- Tungsten
- 执行计划
- Adaptive Query Execution
- 数据倾斜治理
- 动态资源分配
- Executor 配置
- Driver 和 Executor
- 内存管理
- Checkpoint
- Structured Streaming

要能回答：

- 为什么任务卡在某个 Stage？
- 为什么一个 Executor 特别慢？
- 为什么 Join 导致 Shuffle 巨大？
- 什么情况下用 Broadcast Join？
- 数据倾斜怎么定位和处理？
- Spark Streaming 和 Flink 怎么选？

## 重点难点

### Shuffle

Shuffle 是 Spark 性能优化的核心。

它通常发生在：

- groupBy
- reduceByKey
- join
- distinct
- orderBy

问题：

- 网络传输大。
- 磁盘 IO 大。
- 容易出现数据倾斜。

优化思路：

- 提前过滤数据。
- 减少宽依赖。
- 合理设置分区数。
- 小表用 Broadcast Join。
- 聚合前先局部预聚合。

### Partition

Partition 是 Spark 并行度的基础。

分区太少：

- 并行度不够。
- 单个任务太重。

分区太多：

- 调度开销大。
- 小文件变多。

架构判断：

```text
分区不是越多越好，要结合数据量、集群资源和输出文件大小。
```

### 数据倾斜

数据倾斜指某些 key 的数据特别多，导致部分任务跑很久。

常见解决：

- 过滤异常热点 key。
- 加盐打散。
- Broadcast Join。
- 两阶段聚合。
- 单独处理热点数据。

## 博客项目实战

做一个博客访问日志 ETL：

```text
HDFS 博客原始访问日志
  |
  v
Spark 清洗
  |
  v
按天统计 PV / UV / 热门文章 / 慢请求
  |
  v
写入 Doris
```

要求：

- 读取 CSV 或 JSON。
- 过滤脏数据。
- 转换时间字段。
- 按日期聚合。
- 按文章 ID 统计 Top 10。
- 按状态码统计错误分布。
- 输出 Parquet。
- 写入 Doris。
- 记录任务日志。

验收标准：

- 能本地运行。
- 能提交到集群运行。
- 结果条数正确。
- 脏数据不会导致任务失败。
- 能解释 Spark UI 中的 Stage 和 Task。

## 学习路线

```text
SparkSession
  -> RDD / DataFrame
  -> Spark SQL
  -> 文件读写
  -> Transformation / Action
  -> Join 和聚合
  -> Shuffle 和 Partition
  -> Spark UI 排查
  -> 性能优化
  -> Structured Streaming
```

## 面试和工作重点

必须能讲清楚：

- Spark 和 Hadoop MapReduce 的区别。
- RDD、DataFrame、Dataset 区别。
- Transformation 和 Action 区别。
- 什么是 Shuffle。
- 什么是数据倾斜。
- Spark 任务为什么会慢。
- Driver 和 Executor 的职责。
- Spark 在数仓链路里的位置。

## 一句话总结

Spark 的核心是：**把大数据计算任务拆开并行跑，真正的能力在于 ETL、聚合、Join 和性能优化。**
