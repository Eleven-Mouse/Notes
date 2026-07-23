# Scala 学习路线：以博客日志分析为主

## 结论

Scala 是一门运行在 JVM 上的编程语言。学习大数据时，Scala 的核心价值是：**很多 Spark 代码、API 和源码都和 Scala 关系很深。**

如果目标是大数据开发，不需要一开始把 Scala 学成语言专家。先学够写 Spark 作业，再逐步深入函数式和类型系统。

在博客项目里，Scala 的主要用途是：**编写 Spark 任务，解析博客访问日志，统计 PV、UV、热门文章、访问来源和慢请求。**

## 博客项目主线

你学 Scala 时，不要从复杂语言特性开始，而是围绕日志处理建模：

```scala
case class BlogAccessLog(
  requestId: String,
  userId: Option[Long],
  articleId: Option[Long],
  path: String,
  eventTime: String,
  status: Int,
  costMs: Long
)
```

学习重点对应博客场景：

| Scala 知识 | 博客里的用途 |
| --- | --- |
| case class | 表示访问日志、统计结果 |
| Option | 处理未登录用户、缺失文章 ID、脏数据 |
| 集合操作 | 本地练习过滤、转换、聚合 |
| 模式匹配 | 处理不同访问事件和异常数据 |
| 高阶函数 | 写清晰的数据转换逻辑 |
| trait | 抽象日志解析器、指标计算器 |

优先级：

```text
先会写清洗逻辑，再学复杂类型系统。
```

## 它是什么

Scala 同时支持：

- 面向对象编程
- 函数式编程
- 静态类型
- JVM 生态

它可以调用 Java 类库，也可以被 Java 项目集成。

简单例子：

```scala
val nums = List(1, 2, 3, 4)
val result = nums.filter(_ > 2).map(_ * 10)
```

## 什么时候用

适合用 Scala：

- 写 Spark 批处理任务。
- 阅读 Spark 源码。
- 写数据处理 DSL。
- 团队已有 Scala 技术栈。
- 需要函数式表达能力。

不一定要用 Scala：

- 普通 Java 后端项目。
- 团队没人维护 Scala。
- 简单脚本任务。
- 公司 Spark 主要使用 PySpark 或 Java。

架构判断：

```text
Scala 不是大数据必学语言，但如果你要深入 Spark，它非常值得学。
```

## 能做什么

Scala 可以：

- 编写 Spark RDD / DataFrame / Dataset 程序。
- 写 ETL 任务。
- 写数据清洗逻辑。
- 写 Spark UDF。
- 阅读 Spark 内部实现。
- 构建高表达力的数据处理代码。

典型位置：

```text
Scala 代码
  |
  v
Spark 应用
  |
  v
提交到集群执行
```

## 架构位置

Scala 本身不是架构组件，它是开发语言。

在大数据链路里：

```text
HDFS：存数据
Spark：算数据
Scala：写 Spark 计算逻辑
Doris：查结果
```

一句话：

```text
Scala 是工具，Spark 才是它在大数据场景里的主要战场。
```

## 入门阶段

目标：能看懂和编写基础 Scala 程序。

重点：

- val 和 var
- 基本类型
- if / match
- 函数
- 类和对象
- companion object
- case class
- 集合 List、Map、Set
- map / flatMap / filter / reduce

必须掌握：

```scala
case class User(id: Long, name: String, age: Int)

val users = List(
  User(1, "A", 18),
  User(2, "B", 22)
)

val adults = users.filter(_.age >= 18)
```

## 进阶阶段

目标：能写出清晰的 Spark Scala 代码。

重点：

- Option
- Either
- Try
- 模式匹配
- 隐式参数基础
- 隐式转换基础
- 泛型
- trait
- 高阶函数
- 偏函数

重点理解：

```text
Scala 代码里经常用 Option 表示可能为空，少用 null。
```

例如：

```scala
def findName(id: Long): Option[String] = {
  if (id == 1) Some("Alice") else None
}
```

## 高级阶段

目标：能理解复杂 Scala 代码和 Spark 源码。

重点：

- 类型系统
- 协变和逆变
- 隐式解析规则
- type class 思想
- Future
- 函数式错误处理
- Akka 基础
- Spark 源码中的 Scala 写法

注意：

```text
高级 Scala 很强，但学习成本高。除非你要深入框架源码，不要一开始钻太深。
```

## 重点难点

### 函数式集合操作

Spark 很多 API 都像 Scala 集合操作。

例如：

```scala
data
  .filter(row => row.isValid)
  .map(row => transform(row))
  .groupBy(row => row.date)
```

理解集合操作后，学 Spark 会顺很多。

### Option

Java 里常见：

```java
if (user != null) {}
```

Scala 更推荐：

```scala
userOption match {
  case Some(user) => println(user.name)
  case None => println("not found")
}
```

架构价值：

```text
把“可能没有值”显式表达出来，减少空指针风险。
```

### 隐式

隐式是 Scala 的强大特性，也是新手容易懵的地方。

先达到这个程度即可：

- 看见 implicit 不害怕。
- 知道它可能自动补参数或增强方法。
- 能在 IDE 里找到隐式来源。

## 博客项目实战

做一个博客访问日志清洗程序：

输入：

```text
req-1,1001,2001,/article/2001,2026-07-23 10:00:00,200,35
req-2,,2002,/article/2002,2026-07-23 10:01:00,200,42
bad_line
```

输出：

```text
BlogAccessLog(requestId=req-1, userId=Some(1001), articleId=Some(2001))
BlogAccessLog(requestId=req-2, userId=None, articleId=Some(2002))
```

要求：

- 使用 case class 表示博客访问日志。
- 使用 Option 处理脏数据。
- 使用集合操作完成过滤和转换。
- 写单元测试验证解析逻辑。

验收标准：

- 正常行能解析成功。
- 脏数据不会导致程序崩溃。
- 解析逻辑清晰可测试。
- 能为 Spark 作业复用这套解析逻辑。

## 学习路线

```text
基础语法
  -> 集合操作
  -> case class
  -> Option / Either
  -> 模式匹配
  -> trait 和泛型
  -> Spark Scala API
  -> Spark 源码阅读
```

## 面试和工作重点

必须能讲清楚：

- val 和 var 区别。
- case class 有什么用。
- Option 为什么比 null 更安全。
- map / flatMap / filter 的区别。
- Scala 和 Java 的关系。
- Scala 在 Spark 里的作用。

## 一句话总结

学习 Scala 的正确姿势是：**先用它写清楚 Spark 数据处理逻辑，再根据需要深入函数式和类型系统。**
