> 面试定位：Application 层的价值是“编排用例”，不是“承载所有业务代码”。

---

## 一、职责边界

1. 接收请求命令（Command）并启动业务用例。
2. 协调领域对象完成业务流程。
3. 控制事务边界、权限校验、幂等等应用级能力。
4. 返回 DTO，不暴露领域内部细节。

---

## 二、应该写什么

- 用例服务（如 `CreateOrderAppService`）。
- 命令对象与查询对象。
- 调用领域服务、仓储接口、领域事件发布。

---

## 三、不该写什么

1. 不写复杂领域规则（应下沉 domain）。
2. 不直接写 SQL（应走 repository + infrastructure）。
3. 不耦合 Web/RPC 框架细节（应由 adapter 处理）。

---

## 四、面试表达模板

“Application 层负责把一次业务请求串起来，定义步骤和事务边界；真正的业务规则在 Domain。这样业务变更时主要改 Domain，而不是控制器和数据库代码到处改。”

---

## 五、面试追问速答

### Q1：Application 层和 Controller 区别？

- Controller 负责协议适配与参数接收；Application 负责业务用例编排。

### Q2：Application 层可以调用第三方接口吗？

- 可通过领域定义的端口/接口调用，具体实现放 infrastructure。

### Q3：事务注解一般放哪层？

- 通常放 Application 层用例方法，保障一次业务编排的一致性。

