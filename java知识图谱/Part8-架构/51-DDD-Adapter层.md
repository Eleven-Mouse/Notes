> 面试定位：Adapter 层回答重点是“解耦协议差异”，不是“再写一层转发”。

---

## 一、职责边界

1. 入站适配：HTTP、RPC、MQ 消费等请求转为应用命令。
2. 出站适配：把领域端口调用适配到具体外部系统协议。
3. 统一处理参数转换、鉴权上下文透传、异常映射。

---

## 二、常见结构

1. `inbound`：Controller、RPC Provider、MQ Listener。
2. `outbound`：仓储实现适配器、第三方服务客户端适配器。
3. `assembler`：DTO 与领域对象转换器。

---

## 三、设计原则

1. Adapter 只做“翻译”，不写核心业务决策。
2. 对外协议变化不应冲击 Domain。
3. 错误码和异常语义在适配层做统一映射。

---

## 四、面试追问速答

### Q1：Adapter 和 Infrastructure 区别？

- Adapter 处理“接口/协议转换”；Infrastructure 提供“技术能力实现”。

### Q2：为什么需要 DTO 转换？

- 防止领域模型直接暴露给外部，减少耦合与误用。

### Q3：Controller 直接调 Repository 可以吗？

- 不建议，会绕开应用编排与领域规则，架构会快速失控。

