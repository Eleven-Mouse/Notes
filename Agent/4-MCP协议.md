# Stage 4：MCP（Model Context Protocol）—— 2026 重点（详细版）

> 目标：MCP 是 2026 年拉开面试差距的关键点。掌握它能直接加分。
> 一句话定位：**MCP 是 Agent 世界的"USB 接口"——标准化工具调用协议。**

---

## 1. MCP 是什么

### 1.1 一句话理解

> **MCP（Model Context Protocol）是 Anthropic 于 2024 年底发布的开放协议，用于标准化 LLM 与外部工具/数据源之间的连接方式。**

### 1.2 为什么需要 MCP（先理解痛点）

```
没有 MCP 之前的世界：

你想让 Agent 调用 3 个工具：
  工具 A（天气 API）→ 写一套对接代码
  工具 B（数据库）  → 写另一套对接代码
  工具 C（邮件）    → 又要写一套

每个工具的：接口格式、认证方式、错误处理、数据格式 都不一样
换个 LLM（OpenAI → Claude），对接代码又要重写

类比：
  没有 MCP → 每个外设（鼠标/键盘/U盘）都要装专用驱动
  有了 MCP → 所有外设统一用 USB 接口，即插即用
```

### 1.3 MCP 要解决的问题

| 痛点 | 说明 |
|------|------|
| 工具对接碎片化 | 每个 LLM 平台有自己的工具调用格式，互不兼容 |
| 切换成本高 | 从 OpenAI 换到 Claude，工具层代码要重写 |
| 生态割裂 | A 平台的工具无法在 B 平台使用 |
| 重复开发 | 同一个"查天气"功能，每个平台都要实现一遍 |
| 能力发现困难 | Agent 不知道有哪些工具可用 |

---

## 2. MCP 架构详解

### 2.1 核心角色

```
┌─────────────────────────────────────────────────────────┐
│                    MCP 架构                               │
│                                                          │
│  ┌──────────────────┐                   ┌─────────────┐ │
│  │    MCP Host       │  MCP 标准协议      │  MCP Server │ │
│  │   （宿主应用）      │ ◀══════════════▶  │ （工具提供者）│ │
│  │                   │                   │             │ │
│  │  Claude 桌面端     │                   │  天气服务    │ │
│  │  Cursor IDE       │                   │  数据库连接  │ │
│  │  自定义 Agent 应用  │                   │  GitHub API │ │
│  │  Spring Boot 后端  │                   │  文件系统    │ │
│  └────────┬─────────┘                   └─────────────┘ │
│           │            可同时连接多个 MCP Server           │
│           ▼                                              │
│  ┌──────────────────┐                                    │
│  │  MCP Client       │  Host 内部的通信组件                │
│  │ （每个Server一个）  │  负责与 Server 建立连接和通信       │
│  └──────────────────┘                                    │
│                                                          │
│  ┌──────────────────┐                                    │
│  │      LLM          │  做决策的大脑                      │
│  │  (Claude/GPT等)   │                                    │
│  └──────────────────┘                                    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**三个核心角色详解：**

| 角色 | 职责 | Java 类比 | 示例 |
|------|------|----------|------|
| **Host（宿主）** | 运行 Agent 的应用，发起 MCP 连接 | Spring Boot 应用 | Claude Desktop / Cursor |
| **Client（客户端）** | Host 内与 Server 通信的组件 | HTTP Client / Feign Client | 每个 Server 连接一个 Client |
| **Server（服务端）** | 提供具体工具能力的服务 | `@RestController` 暴露的微服务 | 天气/数据库/GitHub MCP Server |

### 2.2 MCP Server 提供的三种能力

```
MCP Server 暴露的能力：
├── Tools（工具）
│   ├── 可被调用的函数
│   ├── 类比：REST API 的 POST 接口
│   └── 示例：query_order(orderId) → 订单详情
│
├── Resources（资源）
│   ├── 可被读取的数据
│   ├── 类比：REST API 的 GET 接口
│   └── 示例：读取配置文件内容、获取数据库 Schema
│
└── Prompts（提示词模板）
    ├── 预定义的提示词
    ├── 类比：邮件模板
    └── 示例："代码审查模板"、"日报生成模板"
```

### 2.3 MCP 通信方式

| 方式 | 适用场景 | 原理 | Java 类比 |
|------|---------|------|----------|
| **stdio** | 本地开发、同进程 | 标准输入输出 | `ProcessBuilder` + stdin/stdout |
| **SSE（HTTP）** | 远程服务、生产环境 | Server-Sent Events | HTTP 长连接 |

```
stdio 模式：
  Host ← stdin/stdout → MCP Server（同一台机器上的子进程）

SSE 模式：
  Host ← HTTP/SSE → MCP Server（远程服务器，通过网络）
```

### 2.4 MCP 协议交互流程

```
┌────────────────────────────────────────────────────────────┐
│                  MCP 一次完整的工具调用流程                     │
│                                                             │
│  1. 初始化（Initialization）                                  │
│     Host → Server: "你好，我要连接你"                          │
│     Server → Host: "我是天气服务，提供这些能力：[...]"           │
│                                                             │
│  2. 能力发现（Capability Discovery）                           │
│     Host 询问 Server 有哪些 Tools / Resources / Prompts       │
│     Server 返回完整的工具描述列表                               │
│                                                             │
│  3. 工具调用（Tool Invocation）                                │
│     Host → Server: "调用 get_weather({city: '北京'})"         │
│     Server 执行工具 → 返回结果                                 │
│     Server → Host: "小雨，15°C"                               │
│                                                             │
│  4. LLM 生成回答                                              │
│     Host 将工具结果交给 LLM → 生成最终回答                      │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

---

## 3. MCP vs API vs Plugin（面试必问对比）

### 3.1 详细对比表

| 维度 | 直接调 API | OpenAI Plugin | MCP |
|------|-----------|---------------|-----|
| 标准化 | 无（每个API不同） | OpenAI 专有 | **开放标准** |
| 兼容性 | 需适配每个LLM | 仅限 OpenAI | **跨平台通用** |
| 开发成本 | 高（逐个对接） | 中 | **低（一次开发到处用）** |
| 生态 | 无统一生态 | OpenAI 商店 | **开放社区生态** |
| 能力发现 | 手动配置 | 商店浏览 | **Server 自动声明** |
| 认证 | 各自实现 | OpenAI 统一 | **OAuth 2.1（推进中）** |
| 前景 | 无演进方向 | 停滞 | **快速成长** |

### 3.2 面试怎么说

> 直接调 API 的问题是"每对接一个工具就要写一套胶水代码"，OpenAI Plugin 解决了标准化但绑死在 OpenAI 生态里。
>
> MCP 的核心价值是**开放 + 标准化**——不绑定任何 LLM 厂商，一个 MCP Server 开发一次，Claude/GPT/开源模型都能用。
>
> 类比到 Java：早年各家搞自己的 RPC 协议（Dubbo / gRPC / Thrift），最后 REST + OpenAPI Spec 成了通用标准。MCP 就是在做 Agent 工具调用的"REST 化"。

---

## 4. MCP 在 Agent 体系中的位置

```
┌─────────────────────────────────────────────────────┐
│              Agent 系统分层架构                        │
│                                                      │
│  ┌─────────────────────────────────────┐             │
│  │  应用层：业务 Agent                   │             │
│  │  （客服Agent / 代码Agent / 数据Agent）│             │
│  └─────────────────┬───────────────────┘             │
│                    │                                 │
│  ┌─────────────────▼───────────────────┐             │
│  │  编排层：LangChain / LangGraph       │             │
│  │  （管理 Agent 执行流程）              │             │
│  └─────────────────┬───────────────────┘             │
│                    │                                 │
│  ┌─────────────────▼───────────────────┐             │
│  │  协议层：MCP ★ 你在这里 ★            │             │
│  │  （标准化工具调用协议）                │             │
│  └─────────────────┬───────────────────┘             │
│                    │                                 │
│  ┌─────────────────▼───────────────────┐             │
│  │  工具层：各种 MCP Server              │             │
│  │  （天气/数据库/GitHub/文件系统...）    │             │
│  └─────────────────────────────────────┘             │
│                                                      │
└─────────────────────────────────────────────────────┘
```

> **MCP 的定位：编排层和工具层之间的标准化协议层。**
> 往上看，Agent 框架通过 MCP Client 调用工具；
> 往下看，各种工具以 MCP Server 的形式暴露能力。

---

## 5. MCP 实际工程应用

### 5.1 场景 1：Spring Boot 微服务暴露为 MCP Server

```java
// 你公司已有 20 个微服务（订单/用户/商品/支付...）
// 只需要加一层薄薄的 MCP 适配，就能让 Agent 调用

@Component
public class OrderMcpServer {

    @Autowired private OrderService orderService;
    @Autowired private LogisticsService logisticsService;

    @McpTool(name = "query_order", description = "根据订单号查询订单详情，返回状态、金额、商品信息")
    public OrderDetail queryOrder(
        @McpParam(name = "orderId", description = "订单号，纯数字格式") String orderId
    ) {
        return orderService.getById(orderId);
    }

    @McpTool(name = "cancel_order", description = "取消未发货的订单，需要提供取消原因")
    public Result cancelOrder(
        @McpParam(name = "orderId", description = "订单号") String orderId,
        @McpParam(name = "reason", description = "取消原因") String reason
    ) {
        return orderService.cancel(orderId, reason);
    }

    @McpTool(name = "query_logistics", description = "查询订单的物流跟踪信息")
    public LogisticsInfo queryLogistics(
        @McpParam(name = "orderId", description = "订单号") String orderId
    ) {
        return logisticsService.track(orderId);
    }
}
```

### 5.2 场景 2：MCP Server 适配层设计

```
┌─────────────────────────────────────────────────┐
│             MCP 适配层架构                        │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  MCP Server（薄适配层）                    │   │
│  │                                            │   │
│  │  - 扫描 @McpTool 注解                      │   │
│  │  - 生成工具描述（JSON Schema）              │   │
│  │  - 处理 MCP 协议通信（stdio / SSE）         │   │
│  │  - 路由工具调用到对应的 Spring Bean          │   │
│  │                                            │   │
│  └──────────────────┬───────────────────────┘   │
│                     │ 内部调用（Spring 内部）       │
│                     ▼                            │
│  ┌──────────────────────────────────────────┐   │
│  │  现有 Spring Boot 微服务                    │   │
│  │  OrderService / UserService / PayService   │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  改造成本：零改动业务代码，只加一层 MCP 适配       │
│                                                  │
└─────────────────────────────────────────────────┘
```

### 5.3 MCP 的生产价值

| 价值 | 说明 |
|------|------|
| **工具复用** | 一次开发，客服Agent/运维Agent/数据Agent 共享同一套 MCP Server |
| **权限隔离** | MCP Server 独立部署，可以做细粒度鉴权 |
| **热插拔** | 新增工具只需启动新 MCP Server，Agent 无需重启 |
| **多租户** | 不同业务线可以有独立的 MCP Server 集群 |
| **零改造迁移** | 现有 Spring Boot 微服务加一层薄适配即可接入 |

---

## 6. MCP 的安全考虑（面试深入点）

### 6.1 安全风险

| 风险 | 说明 | 防护措施 |
|------|------|---------|
| 提示注入 | 恶意输入误导 LLM 调用危险工具 | 参数校验 + 高危操作人工确认 |
| 越权调用 | LLM 尝试调用超出权限的工具 | MCP Server 端做鉴权 |
| 数据泄露 | 工具返回敏感信息给 LLM | 返回结果脱敏 |
| 恶意 MCP Server | 伪造的 Server 提供恶意工具 | Server 白名单 + 签名验证 |

### 6.2 安全设计原则

```
1. 最小权限原则
   MCP Server 只暴露必要的工具，不暴露全部能力
   例：只暴露 query_order，不暴露 delete_all_orders

2. 纵深防御
   LLM 输出 → 参数校验 → 权限检查 → 执行 → 结果脱敏
   每一层都有安全检查

3. 审计追踪
   记录每次工具调用：谁调的、调了什么、传了什么参数、返回了什么

4. Human-in-the-loop
   高危操作（删除、转账、发布）必须人工确认
```

---

## 7. 面试实战

### 【面试回答（标准版）】—— 30 秒口语化

> MCP 是 Anthropic 提出的开放协议，用于标准化 LLM 和外部工具之间的连接。
>
> 它解决的核心问题是工具调用的碎片化——以前每个工具都要写专用对接代码，换 LLM 就要重写。MCP 通过统一协议让工具开发一次就能被任何 LLM 使用，类比 USB 接口标准化了外设连接。
>
> 架构上，MCP 位于编排层和工具层之间。Host 通过 MCP Client 与 MCP Server 通信，Server 可以暴露 Tools、Resources、Prompts 三种能力。
>
> 在我的项目中，我把现有的 Spring Boot 微服务通过 MCP 适配层暴露出去，多个 Agent 共享同一套 MCP Server，新增工具只需启动新的 Server。

### 【面试官可能追问】

**追问 1：MCP 和 Function Calling 有什么关系？**

> 两者不在同一层面：
> - **Function Calling** 是 LLM 的**模型能力**——模型能输出结构化的工具调用 JSON
> - **MCP** 是**传输协议**——定义了 Agent 应用如何发现和调用工具
>
> MCP 传输的工具调用指令，最终通过 Function Calling 能力触发。类比：Function Calling 是"你会说英语"（能力），MCP 是"HTTP 协议"（通信标准）。不是替代，是互补。

**追问 2：MCP 在生产环境有什么挑战？**

> 三个主要挑战：
> 1. **性能**：远程 MCP Server 有网络开销 → 本地用 stdio、远程做连接池
> 2. **安全**：提示注入可能导致未授权操作 → Server 端权限控制 + 参数校验
> 3. **可观测性**：跨 Host-Client-Server 链路追踪困难 → 加 Trace ID + OpenTelemetry

**追问 3：MCP 生态成熟度如何？**

> 截至 2026 年初，MCP 生态在快速成长：
> - **成熟部分**：官方 SDK（Python/TypeScript/Java）稳定，社区有大量开源 MCP Server，Claude Desktop/Cursor 已原生支持
> - **演进部分**：远程 Server 认证标准（OAuth 2.1）在推进，多 Server 编排最佳实践在探索
> - **我的策略**：内部工具集成 → 可用；对外暴露 → 谨慎；核心链路 → 观望

### 【常见错误】

| 错误说法 | 正确说法 |
|---------|---------|
| "MCP 是 Anthropic 闭源协议" | "MCP 是开放标准，Apache 2.0 许可" |
| "MCP 替代了 Function Calling" | "MCP 标准化通信，Function Calling 是模型能力，互补关系" |
| "MCP 只能配合 Claude 使用" | "MCP 是跨平台开放标准" |
| "用了 MCP 就不需要写工具代码" | "MCP 减少对接成本，工具逻辑仍需开发" |

---

## 8. 一句话总结（背诵用）

> **MCP 是 Agent 世界的 USB 接口——标准化 LLM 与外部工具的连接协议，核心价值是"一次开发，到处使用"，在架构中位于编排层和工具层之间，类似 REST + OpenAPI 标准化 RPC 的过程。**

---

> **Stage 4（详细版）结束。**
