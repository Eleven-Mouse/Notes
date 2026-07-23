# Doris 学习路线：从小白到架构思维

## 结论

Apache Doris 是实时分析型数据库，属于 OLAP 系统。它的核心价值是：**让大量明细或聚合数据可以被快速查询，用来支撑报表、BI、数据分析和实时看板。**

Spark 偏计算，Doris 偏查询。不要把它们混成一个东西。

## 它是什么

Doris 是一个分布式 MPP 分析型数据库。

简单理解：

```text
数据导入 Doris
  |
  v
按分区、分桶、列式存储组织
  |
  v
SQL 查询
  |
  v
快速返回聚合分析结果
```

核心角色：

- **FE**：Frontend，负责元数据、SQL 解析、查询规划、集群管理。
- **BE**：Backend，负责数据存储和查询执行。

## 什么时候用

适合用 Doris：

- 报表查询。
- BI 分析。
- 实时看板。
- 多维聚合。
- 明细查询。
- 用户行为分析。
- 数仓结果服务化。
- 替代部分传统 OLAP 查询场景。

不适合用 Doris：

- 替代 HDFS 存全部原始大文件。
- 替代 Spark 做复杂离线计算。
- 高频事务型更新。
- 强事务核心业务库。
- 小团队没查询性能诉求时过早引入。

架构判断：

```text
Doris 适合放在数据链路末端，承接计算后的结果查询。
```

## 能做什么

Doris 可以：

- 存储用户行为明细。
- 做秒级或亚秒级聚合查询。
- 支撑大屏看板。
- 支撑运营报表。
- 支撑多维分析。
- 接入 BI 工具。
- 查询 Spark / Flink 清洗后的结果。

典型链路：

```text
业务数据 / 日志
  |
  v
HDFS / Kafka
  |
  v
Spark / Flink
  |
  v
Doris
  |
  v
BI / 报表 / 数据服务
```

## 架构位置

Doris 位于分析服务层：

```text
存储层：HDFS / Hive / 对象存储
计算层：Spark / Flink
分析层：Doris
应用层：BI / API / 看板
```

它的重点不是“存所有东西”，而是“让该查的数据查得快”。

## 入门阶段

目标：能建表、导入、查询。

重点：

- FE
- BE
- Database
- Table
- SQL 查询
- 数据导入
- 分区
- 分桶

建表示例思路：

```sql
CREATE TABLE user_event_daily (
  dt DATE,
  event_type VARCHAR(32),
  pv BIGINT,
  uv BIGINT
)
DUPLICATE KEY(dt, event_type)
PARTITION BY RANGE(dt) ()
DISTRIBUTED BY HASH(event_type) BUCKETS 8;
```

必须理解：

```text
分区解决“查哪一段数据”，分桶解决“数据怎么分散到多台机器”。
```

## 进阶阶段

目标：能设计合理表模型并支撑常见分析。

重点：

- Duplicate Key
- Aggregate Key
- Unique Key
- Primary Key
- Routine Load
- Stream Load
- Broker Load
- Spark Connector
- Flink Connector
- 物化视图
- Rollup
- Bitmap
- 查询执行计划

表模型选择：

- 明细日志：Duplicate Key。
- 预聚合指标：Aggregate Key。
- 根据主键更新：Unique Key 或 Primary Key。

架构判断：

```text
表模型选错，后面查询、导入、更新都会难受。
```

## 高级阶段

目标：能做 Doris 集群设计、性能优化和稳定性治理。

重点：

- FE 高可用
- BE 扩容缩容
- Tablet
- Compaction
- 副本
- 数据倾斜
- 查询 Profile
- 慢查询分析
- 分区生命周期
- 冷热数据治理
- 权限管理
- 资源隔离

高级问题：

- 查询慢是扫描数据太多，还是 Join 太重？
- 分桶数应该怎么设？
- 为什么导入积压？
- Compaction 为什么影响性能？
- FE 和 BE 分别怎么扩容？

## 重点难点

### 表模型

Doris 的表模型非常关键。

简单判断：

```text
只追加明细：Duplicate Key
需要预聚合：Aggregate Key
需要按主键更新：Unique Key / Primary Key
```

表模型不是语法问题，而是查询和写入方式的架构选择。

### 分区和分桶

分区常按时间：

```text
dt=2026-07-23
```

好处：

- 查询可以裁剪分区。
- 删除历史数据方便。
- 冷热数据好管理。

分桶常按高频过滤或 Join 字段。

问题：

- 分桶太少，并行度不够。
- 分桶太多，元数据和小 Tablet 压力变大。

### 物化视图

物化视图适合加速固定模式的聚合查询。

适合：

- 报表指标固定。
- 聚合维度稳定。
- 查询频率高。

不适合：

- 查询模式经常变。
- 数据导入压力已经很大。
- 维护成本超过收益。

## 实战项目

做一个 Doris 用户行为分析库：

```text
Spark 统计结果 -> Doris 指标表 -> SQL 查询日报
```

表设计：

- 用户事件明细表
- 用户日活指标表
- 页面访问排行表

查询目标：

- 每日 PV。
- 每日 UV。
- 页面 Top 10。
- 不同事件类型占比。

验收标准：

- 能导入数据。
- 能设计分区和分桶。
- 能写出分析 SQL。
- 能解释为什么选某种表模型。
- 能通过执行计划判断是否扫描过多数据。

## 学习路线

```text
OLAP 概念
  -> FE / BE
  -> 建表和导入
  -> 表模型
  -> 分区分桶
  -> 查询优化
  -> 物化视图
  -> 集群扩容和稳定性
```

## 面试和工作重点

必须能讲清楚：

- Doris 是什么，和 MySQL、Spark 区别是什么。
- FE 和 BE 的职责。
- 分区和分桶的作用。
- Duplicate Key、Aggregate Key、Unique Key 怎么选。
- Doris 适合什么查询场景。
- 查询慢怎么排查。
- Doris 在数仓架构里的位置。

## 一句话总结

Doris 的核心是：**把已经加工好的数据变成能被快速 SQL 查询的分析服务。**

