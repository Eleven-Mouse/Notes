> 面试定位：Infrastructure 不是“大杂烩工具层”，它的核心是承载技术实现并保持可替换性。

---

## 一、职责边界

1. 实现 Domain/Application 定义的仓储与网关接口。
2. 集成数据库、缓存、消息队列、搜索、第三方 API。
3. 提供技术型组件：ID 生成器、分布式锁、序列化等。

---

## 二、常见内容

1. ORM 实体与 Mapper（MyBatis/JPA）。
2. Repository 实现类。
3. MQ Producer/Consumer 支撑实现。
4. Redis、ES、对象存储、RPC Client 封装。

---

## 三、设计原则

1. 技术依赖只能向外泄漏到 Adapter/Start，不入侵 Domain。
2. 用接口隔离可变技术，方便替换中间件。
3. 统一可观测能力：日志、监控、重试、熔断。

---

## 四、面试追问速答

### Q1：Infrastructure 为什么容易失控？

- 因为最接近技术细节，若无边界约束会演变为“全能工具层”。

### Q2：Repository 接口放哪层？

- 通常定义在 Domain，实现在 Infrastructure。

### Q3：如何体现可替换性？

- 上层依赖抽象接口，替换 MySQL/Redis/MQ 时仅改实现层。

