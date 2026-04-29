# Stage 1：Agent 基础认知（详细版）

> 本文档面向 Java 后端候选人，目标岗位：AI 平台 / Agent 开发 / 智能后端。
> 核心原则：**不讲科普，只讲面试能说什么、怎么讲、讲多深。**

---

## 1. 先搞懂 LLM（Agent 的地基）

> Agent 的"大脑"是 LLM，如果对 LLM 本身理解不深，面试一追问就会露馅。

### 1.1 LLM 本质是什么

> **LLM（Large Language Model）本质上是一个"超大规模的概率预测引擎"——给定前文，预测下一个最可能出现的 Token（词/子词）。**

```
输入："今天天气"
LLM 预测下一个 Token → "很好"（概率 0.32） / "不错"（概率 0.28） / "糟糕"（概率 0.05）
```

它不"理解"含义，不"思考"，不"记忆"——它只是在做**统计预测**。

### 1.2 LLM 的工程特性（面试必知）

| 特性 | 说明 | 面试影响 |
|------|------|---------|
| **无状态** | 每次调用互相独立，不记得上一次对话 | 必须通过外部机制维护上下文 |
| **上下文窗口有限** | GPT-4o 128K Token，Claude 200K Token | 超出部分直接截断，必须做上下文管理 |
| **输出不确定** | 即使 temperature=0，结果也不完全一致 | Agent 的决策有随机性 |
| **只有文本** | 输入输出都是文本，不能执行代码/调API | 必须通过 Tool Calling 桥接外部能力 |
| **知识有时效性** | 训练数据有截止日期 | 必须通过 RAG 补充最新知识 |
| **会产生幻觉** | 会自信地编造不存在的信息 | 工程上必须加校验和护栏 |
| **Token 计费** | 按输入+输出的 Token 数计费 | Agent 多轮循环成本很高 |

### 1.3 LLM 调用的工程结构（Java 视角）

```java
// 调用 LLM 本质就是一次 HTTP 请求
// 和调一个 REST API 没有本质区别
public class LLMClient {

    // 最基础的调用：发送消息，接收回复
    public String chat(String userMessage) {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.openai.com/v1/chat/completions"))
            .header("Authorization", "Bearer " + apiKey)
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString("""
                {
                  "model": "gpt-4o",
                  "messages": [
                    {"role": "user", "content": "%s"}
                  ]
                }
                """.formatted(userMessage)))
            .build();

        // 发请求、拿响应——就这么简单
        HttpResponse<String> response = httpClient.send(request, BodyHandlers.ofString());
        return parseResponse(response.body());
    }
}
```

**关键理解：** LLM 就是一个 HTTP 服务——你发文本过去，它回文本回来。它本身不能"做"任何事。

### 1.4 为什么单靠 LLM 不够（引出 Agent 的必要性）

```
场景：用户说"帮我查一下订单 123456 的物流状态"

只用 LLM：
  → LLM 回答："很抱歉，我无法查询实际的物流信息，因为..."
  → 它只能"说"，不能"做"

需要 Agent：
  → Agent 思考："需要调用物流查询工具"
  → Agent 执行：调用物流 API，拿到真实数据
  → Agent 回答："您的订单已到北京分拣中心，预计明天送达"
```

> **LLM 是"嘴"，Agent 是"嘴 + 手 + 脑 + 记忆"。**

---

## 2. Agent 的定义与演进（面试加分——展示知识面）

### 2.1 Agent 的精确定义

> **Agent（智能体）= 一个能感知环境、自主决策、调用工具、采取行动以达成目标的系统。**

拆开理解：
- **感知环境**：接收用户输入 + 工具返回结果
- **自主决策**：不是你告诉它怎么做，它自己判断
- **调用工具**：能操作外部系统（数据库/API/文件等）
- **采取行动**：有实际的执行能力，不只是生成文本
- **达成目标**：有明确的任务完成条件

### 2.2 Agent 演进时间线（面试展示知识面）

```
2022.01  GPT-3.5 发布
         ↓ LLM 能力初现，但只是"说话"
2022.06  ReAct 论文发表（Yao et al.）
         ↓ 提出"推理 + 行动"的范式，Agent 的理论基础
2022.10  AutoGPT / BabyAGI 爆火
         ↓ 首批 Agent 产品，但极度不稳定（"玩具"阶段）
2023.03  OpenAI 发布 Function Calling
         ↓ LLM 首次能输出结构化的工具调用指令
2023.10  LangChain v0.1 发布
         ↓ Agent 编排框架成熟，工程化起步
2024.02  OpenAI Assistants API
         ↓ 工具调用 + 代码执行 + 文件管理一体化
2024.06  Anthropic 发布 Tool Use
         ↓ Claude 系列支持结构化工具调用
2024.11  Anthropic 发布 MCP 协议
         ↓ 工具调用走向标准化
2025     LangGraph 成熟 / Agent 生产化
         ↓ 从"玩具"走向"生产级"
2026     Multi-Agent / Agent 工作流 成为主流
         ↓ 企业级 Agent 大规模落地
```

**面试怎么说：**

> Agent 的概念其实很早就有了（强化学习领域的经典概念），但直到 2022 年 ReAct 论文和 GPT-3.5 的结合，才让基于 LLM 的 Agent 变成现实。
>
> 关键转折点是 2023 年 OpenAI 发布 Function Calling——让 LLM 能输出结构化的工具调用指令，这才让 Agent 从"理论"变成"可工程化"的东西。
>
> 2024 年 MCP 协议的发布，标志着工具调用走向标准化，Agent 生态开始成熟。

### 2.3 Agent 的自主性等级（面试高级讲法）

不是所有 Agent 都一样的"智能"，自主性分等级：

| 等级 | 名称 | 特征 | 类比 |
|------|------|------|------|
| L0 | 无自主性 | 纯 LLM 调用，一问一答 | `HTTP API` |
| L1 | 工具辅助 | LLM + 预定义工具，但流程固定 | `Shell 脚本` |
| L2 | 半自主 | LLM 自主选择工具，但需人工确认关键步骤 | `CI/CD Pipeline`（需审批） |
| L3 | 全自主 | LLM 完全自主规划和执行 | `自动化运维系统` |
| L4 | 多Agent协作 | 多个 Agent 分工协作完成复杂任务 | `微服务集群` |

**面试怎么讲：**

> 面试官问"你对 Agent 怎么理解"，不要笼统回答，而是说：
>
> "Agent 不是二元的'是或不是'，而是一个自主性光谱。从 L0（纯 LLM 调用）到 L4（多 Agent 协作），自主性逐级提升。在工程实践中，我不追求最高自主性，而是根据场景选择合适的等级。比如客服场景用 L2（半自主，关键操作需确认），数据分析场景用 L3（全自主）。"

---

## 3. LLM vs Agent：本质区别（必须讲透）

### 3.1 对比表

| 维度 | LLM（大语言模型） | Agent（智能体） |
|------|-------------------|-----------------|
| 本质 | 概率预测引擎 | 决策执行系统 |
| 能力 | 只能"说话"——接收文本，输出文本 | 能"做事"——规划 + 调用外部工具 + 迭代 |
| 执行模式 | 单次请求-响应 | 多轮循环（思考→行动→观察→再思考） |
| 是否有状态 | 无状态（每次调用互相独立） | 有状态（Memory 贯穿整个任务） |
| 是否能调用工具 | 不能（纯文本生成） | 能（HTTP 调用 / 数据库查询 / 代码执行等） |
| 是否能自主决策 | 不能（你问什么它答什么） | 能（自主决定下一步做什么、调什么工具） |
| 决策方式 | 无决策，直接生成 | LLM 充当"路由器"，动态选择下一步 |
| 错误处理 | 无（输出错误就错误了） | 可以重试、换工具、回退、求助人工 |
| 成本 | 单次调用成本 | 多轮循环，成本 = 步骤数 × 单次成本 |

### 3.2 面试关键区分点

面试官问："Agent 和普通 LLM 调用有什么区别？"

**核心差异就一个词：自主性（Autonomy）。**

- LLM：你让它做什么它做什么，被动执行。像是一个只会说话的顾问。
- Agent：你给它一个目标，它自己拆解、自己决定用什么工具、自己判断是否完成。像是一个能自己动手的项目经理。

### 3.3 一个例子讲透区别

```
用户："北京明天天气怎么样？适合去哪里玩？"

纯 LLM 调用：
  → 基于训练数据回答（可能是去年的天气信息）
  → "北京明天可能是晴天，可以去长城..."（可能完全是编的）

Agent 调用：
  Thought: "需要查真实的天气预报"
  Action: 调用天气API("北京", "明天")
  Observation: 小雨，15°C，湿度80%
  Thought: "下雨了，需要推荐室内场所"
  Action: 调用搜索("北京室内景点推荐")
  Observation: 故宫、国家博物馆、798艺术区...
  Final Answer: "明天北京小雨15°C，推荐去故宫、国家博物馆等室内景点"
  → 信息真实、基于实时数据、决策合理
```

---

## 4. Agent 的最小组成（四个核心组件 + 详解）

### 4.0 架构全景图

```
┌─────────────────────────────────────────────────────────┐
│                    Agent 系统完整架构                      │
│                                                          │
│                    用户目标（Goal）                        │
│                       │                                  │
│                       ▼                                  │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Planner（规划器 / 大脑）               │   │
│  │                                                    │   │
│  │   输入：用户目标 + Memory上下文 + 可用工具列表       │   │
│  │   处理：LLM 推理决策                                │   │
│  │   输出：下一步行动（调什么工具、传什么参数）          │   │
│  └──────────────────────┬───────────────────────────┘   │
│                         │                                │
│                         ▼                                │
│  ┌──────────────────────────────────────────────────┐   │
│  │             Executor（执行器 / 手脚）               │   │
│  │                                                    │   │
│  │   接收 Planner 的决策                               │   │
│  │   → 解析工具名和参数                                │   │
│  │   → 找到对应 Tool 并调用                            │   │
│  │   → 收集执行结果                                    │   │
│  │   → 结果返回给 Planner 做下一轮决策                  │   │
│  └──────────────────────┬───────────────────────────┘   │
│                    ╱          ╲                           │
│                   ▼            ▼                          │
│  ┌──────────────────┐  ┌────────────────────────────┐   │
│  │  Memory（记忆）    │  │     Tools（工具集）          │   │
│  │                   │  │                              │   │
│  │ ┌───────────────┐│  │  ┌───────┐ ┌───────┐        │   │
│  │ │ 短期记忆       ││  │  │ 搜索  │ │ 数据库 │        │   │
│  │ │ (对话上下文)   ││  │  └───────┘ └───────┘        │   │
│  │ └───────────────┘│  │  ┌───────┐ ┌───────┐        │   │
│  │ ┌───────────────┐│  │  │ API   │ │ 代码   │        │   │
│  │ │ 长期记忆       ││  │  └───────┘ └───────┘        │   │
│  │ │ (向量库/DB)   ││  │  ┌───────┐ ┌───────┐        │   │
│  │ └───────────────┘│  │  │ 文件   │ │ 通知   │        │   │
│  └──────────────────┘  │  └───────┘ └───────┘        │   │
│                         └────────────────────────────┘   │
│                                                          │
│                    最终结果（Result）                      │
└─────────────────────────────────────────────────────────┘
```

### 4.1 Planner（规划器 / 大脑）

**是什么：** 决定"下一步做什么"的组件。由 LLM 充当。

**做什么：**
- 分析用户的目标
- 拆解成子步骤
- 决定每一步用什么工具
- 根据执行结果调整后续计划
- 判断任务是否完成

**工作原理：**
```
Planner 的输入（拼成一个超长 Prompt）：
┌─────────────────────────────────────────┐
│  System Prompt（角色设定 + 能力说明）      │
│  + 可用工具列表（名称 + 描述 + 参数Schema）│
│  + Memory 上下文（对话历史 + 知识检索）    │
│  + 用户当前输入                           │
│  + 之前步骤的 Thought/Action/Observation  │
└─────────────────────────────────────────┘
        ↓ 全部发给 LLM
        ↓ LLM 输出：
        Thought: 我需要先查天气
        Action: weather_api("北京")
```

**Java 类比：** 类似 Spring 中的 `DispatcherServlet`——接收请求，决定分发给哪个 Handler。但 DispatcherServlet 是硬编码的路由规则，Planner 是 LLM 动态决策。

```java
// Planner 的本质就是：组装 Prompt → 调 LLM → 解析决策
public class Planner {

    public PlanDecision decide(AgentContext context) {
        // 1. 组装 Prompt
        String prompt = buildPrompt(
            context.getGoal(),          // 用户目标
            context.getMemory(),        // 记忆上下文
            context.getToolList(),      // 可用工具
            context.getHistory()        // 之前的步骤记录
        );

        // 2. 调 LLM
        LLMResponse response = llmClient.chat(prompt);

        // 3. 解析决策
        if (response.containsToolCall()) {
            return PlanDecision.action(
                response.getToolName(),
                response.getToolArgs()
            );
        } else {
            return PlanDecision.finalAnswer(response.getText());
        }
    }
}
```

**Planner 的工程挑战：**

| 挑战 | 说明 | 解决方案 |
|------|------|---------|
| Prompt 太长 | 工具描述+记忆+历史超出窗口 | 滑动窗口+摘要压缩 |
| 选错工具 | 工具描述不精确导致误选 | 精简工具数量、优化描述 |
| 决策不一致 | 同样输入不同结果 | temperature=0 + 限制选择范围 |
| 陷入死循环 | 反复调同一个工具 | 最大迭代次数限制 |

### 4.2 Tools（工具集 / 手脚）

**是什么：** Agent 可以调用的外部能力（函数 / API / 服务）。

**工具分类（面试按这个框架讲）：**

| 类型 | 说明 | 示例 |
|------|------|------|
| **信息查询类** | 获取外部数据 | 搜索引擎、天气API、数据库查询 |
| **操作执行类** | 执行实际操作 | 发邮件、调API、执行代码 |
| **计算处理类** | 数据处理和计算 | 计算器、数据分析、格式转换 |
| **生成创建类** | 生成新内容 | 写文档、画图、生成代码 |

**工具定义的三个要素（面试必讲）：**

```
一个完整的工具定义包含：

1. 名称（name）：给 LLM 看的标识符
   → "query_order"

2. 描述（description）：告诉 LLM 这个工具能做什么、什么时候该用
   → "根据订单号查询订单详情，包含订单状态、金额、商品列表、物流信息"

3. 参数 Schema（parameters）：定义参数类型和约束
   → { orderId: string (必填), detailLevel: enum ["simple", "full"] (选填) }
```

> **面试关键点：工具描述的质量直接决定 Agent 的表现。** 描述越精确，LLM 选错工具的概率越低。

**Java 类比：** 类似 Spring 中的 `@Service` Bean 集合——Planner 通过接口描述找到对应的服务并调用。

```java
// 工具定义的 Java 实现
public class ToolDefinition {
    private String name;           // "query_order"
    private String description;    // "查询订单详情"
    private JsonSchema parameters; // 参数的 JSON Schema
    private Method handler;        // 实际执行的 Java 方法
    private Object bean;           // Spring Bean 实例

    // 工具执行
    public ToolResult execute(Map<String, Object> args) {
        try {
            // 参数校验
            validate(args);
            // 权限检查
            checkPermission();
            // 反射调用
            Object result = handler.invoke(bean, mapToArgs(args));
            return ToolResult.success(result);
        } catch (Exception e) {
            return ToolResult.error(e.getMessage());
        }
    }
}
```

**工具设计的工程原则：**

| 原则 | 说明 | 反例（不要这样） |
|------|------|-----------------|
| 单一职责 | 一个工具只做一件事 | "万能查询"工具 |
| 描述精确 | 说清楚能做什么、不能做什么 | "查询工具"（太模糊） |
| 参数简洁 | 参数越少越好 | 10 个参数的工具 |
| 错误明确 | 返回清晰的错误信息 | 直接抛异常不返回原因 |
| 幂等安全 | 查询类工具要幂等 | 查询操作有副作用 |

### 4.3 Memory（记忆系统 / 上下文管理）

**是什么：** 存储 Agent 执行过程中的上下文信息，让 Agent "记住"发生过什么。

**为什么必须要有 Memory：**

```
没有 Memory 的 Agent：
  用户："帮我查一下订单 123456"
  Agent：调用工具 → "订单123456，已发货"
  用户："那个物流呢？"
  Agent："什么物流？" ← 不记得刚才在聊什么

有 Memory 的 Agent：
  用户："帮我查一下订单 123456"
  Agent：调用工具 → "订单123456，已发货"
  用户："那个物流呢？"
  Agent：理解"那个"= 订单123456 → 调用物流查询 → "顺丰已到北京"
```

**Memory 的三层架构：**

```
┌─────────────────────────────────────────────────────┐
│               Memory 三层架构                         │
│                                                      │
│  L1: 工作记忆（Working Memory）                       │
│  ┌─────────────────────────────────────────────┐     │
│  │  存在哪：LLM 的上下文窗口（拼在 Prompt 里）    │     │
│  │  存什么：当前对话的最近几轮 + 当前任务的步骤     │     │
│  │  大小限制：受 LLM 上下文窗口限制                │     │
│  │  生命周期：单次请求                              │     │
│  │  Java类比：方法参数 + 局部变量                   │     │
│  │  技术：直接拼接到 Prompt 字符串中                │     │
│  └─────────────────────────────────────────────┘     │
│                    ↓ 超出的部分 ↓                      │
│  L2: 短期记忆（Short-term Memory）                    │
│  ┌─────────────────────────────────────────────┐     │
│  │  存在哪：Redis / 内存                           │     │
│  │  存什么：当前会话的完整对话历史 + 摘要            │     │
│  │  大小限制：无硬限制，但要做 Token 预算管理        │     │
│  │  生命周期：一次会话（关浏览器消失）              │     │
│  │  Java类比：Redis Session                        │     │
│  │  技术：Redis List + 自动过期                     │     │
│  └─────────────────────────────────────────────┘     │
│                    ↓ 需要跨会话 ↓                      │
│  L3: 长期记忆（Long-term Memory）                     │
│  ┌─────────────────────────────────────────────┐     │
│  │  存在哪：向量数据库 + MySQL                      │     │
│  │  存什么：用户偏好、历史行为精华、知识库           │     │
│  │  大小限制：无限制                                │     │
│  │  生命周期：永久，跨会话                          │     │
│  │  Java类比：MySQL + Elasticsearch                │     │
│  │  技术：Milvus 向量检索 + MySQL 结构化存储        │     │
│  └─────────────────────────────────────────────┘     │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**Memory 的核心难题——Token 管理：**

```
LLM 上下文窗口 = 128K Token
但要留给：
  - System Prompt：        ~1K Token
  - 工具描述：             ~2-5K Token
  - RAG 检索结果：          ~3-5K Token
  - 历史对话：             ？？？
  - 用户当前输入：          ~0.5K Token
  - LLM 输出预留：         ~2K Token

  历史对话的 Token 预算 ≈ 128K - 10K = ~118K
  但实际生产中一般控制在 4K~8K，因为：
  1. Token 越多成本越高
  2. 太长的上下文会降低 LLM 的注意力质量
  3. 延迟也会增加

所以必须做 Memory 的裁剪和压缩！
```

```java
// Memory Token 管理的 Java 实现
public class MemoryManager {

    private static final int MEMORY_TOKEN_BUDGET = 6000; // 留给记忆的 Token 预算
    private final RedisTemplate<String, String> redis;
    private final LLMClient llmClient; // 用于生成摘要

    // 保存消息
    public void save(String sessionId, Message message) {
        String key = "agent:memory:" + sessionId;
        redis.opsForList().rightPush(key, toJson(message));
        redis.expire(key, 24, TimeUnit.HOURS);
    }

    // 构建给 LLM 的记忆上下文
    public String buildContext(String sessionId) {
        List<Message> allMessages = getAllMessages(sessionId);

        if (estimateTokens(allMessages) <= MEMORY_TOKEN_BUDGET) {
            // 没超预算，直接返回全部
            return formatMessages(allMessages);
        }

        // 超预算了：滑动窗口 + 摘要压缩
        // 1. 保留最近 5 轮（滑动窗口）
        List<Message> recent = allMessages.subList(
            Math.max(0, allMessages.size() - 10), allMessages.size()
        );

        // 2. 之前的消息生成摘要
        List<Message> old = allMessages.subList(0, allMessages.size() - 10);
        String summary = getOrCreateSummary(sessionId, old);

        return "对话摘要：" + summary + "\n\n最近对话：\n" + formatMessages(recent);
    }
}
```

### 4.4 Executor（执行器 / 调度中心）

**是什么：** 实际调用工具、获取结果的组件。是 Planner 和 Tools 之间的桥梁。

**做什么：**
- 接收 Planner 的决策（调用哪个工具、传什么参数）
- 解析和校验参数
- 找到对应的 Tool 实现
- 执行工具调用
- 收集执行结果（包括成功/失败）
- 将结果返回给 Planner 进行下一轮决策

**Executor 为什么是独立组件（面试理解）：**

> Planner（LLM）不做执行，只做决策。分离的好处：
> 1. **安全性**：LLM 不可控，如果直接执行可能造成破坏
> 2. **可观测性**：Executor 是统一的拦截点，可以加日志/审计/限流
> 3. **可扩展性**：工具的实现可以独立变化，不影响决策逻辑
> 4. **可测试性**：可以 Mock Executor 做单元测试

**Java 类比：**
- Executor ≈ `反射调用` 或 `RPC 框架`——根据方法签名找到实现并执行
- Executor ≈ `Spring AOP 代理`——在调用前后加拦截逻辑

```java
// Executor 的完整实现
public class ToolExecutor {

    private final Map<String, ToolDefinition> toolRegistry;

    public ToolResult execute(String toolName, Map<String, Object> args) {
        long startTime = System.currentTimeMillis();

        try {
            // 1. 查找工具
            ToolDefinition tool = toolRegistry.get(toolName);
            if (tool == null) {
                return ToolResult.error("工具不存在: " + toolName);
            }

            // 2. 参数校验（JSON Schema 校验）
            ValidationResult validation = validateParams(tool, args);
            if (!validation.isValid()) {
                return ToolResult.error("参数校验失败: " + validation.getErrors());
            }

            // 3. 权限检查
            if (!checkPermission(toolName)) {
                return ToolResult.error("无权限调用工具: " + toolName);
            }

            // 4. 执行工具
            Object result = tool.execute(args);

            // 5. 记录审计日志
            auditLog(toolName, args, result, System.currentTimeMillis() - startTime);

            return ToolResult.success(result);

        } catch (Exception e) {
            // 6. 异常处理
            auditLog(toolName, args, "ERROR: " + e.getMessage(),
                     System.currentTimeMillis() - startTime);
            return ToolResult.error("工具执行失败: " + e.getMessage());
        }
    }
}
```

---

## 5. 单轮调用 vs 多轮对话 vs Agent（面试必考对比）

### 5.1 单轮调用（Single Turn）

```
用户 → LLM → 回答
```

- 就是一次 API 调用
- 无状态、无记忆
- 典型场景：翻译、摘要、代码补全、文本分类

```java
// 单轮调用的 Java 实现
public String translate(String text) {
    return llmClient.chat("请翻译成英文：" + text);
}
```

### 5.2 多轮对话（Multi-Turn Chat）

```
用户 → LLM → 回答 → 用户追问 → LLM → 回答 → ...
         ↑_____________________________↓
              （对话历史拼接到 Prompt）
```

- 有对话上下文（History）
- 但本质仍是被动响应——用户不问，它不动
- 每次调用把之前的对话历史全部拼入 Prompt
- 典型场景：ChatGPT 对话

```java
// 多轮对话的 Java 实现
public class ChatService {
    private List<Message> history = new ArrayList<>();

    public String chat(String userInput) {
        // 把完整历史 + 新输入一起发给 LLM
        history.add(Message.user(userInput));
        String response = llmClient.chat(history);
        history.add(Message.assistant(response));
        return response;
    }
}
```

### 5.3 Agent（自主执行）

```
用户（给目标）→ Agent 自主循环：
  Thought → Action → Observation → Thought → Action → Observation → ...
  → 直到目标完成 → 返回最终结果
```

- **关键区别：中间步骤不需要人参与**
- Agent 自己决定调什么工具、调几次
- 典型场景："帮我查一下北京明天的天气，如果下雨就推荐室内活动"

```java
// Agent 的 Java 实现（简化版）
public class Agent {
    public String execute(String goal) {
        String context = goal;
        for (int i = 0; i < MAX_ITERATIONS; i++) {
            // LLM 决策
            LLMResponse decision = planner.decide(context);

            if (decision.isFinalAnswer()) {
                return decision.getText();
            }

            // 执行工具
            ToolResult result = executor.execute(
                decision.getToolName(),
                decision.getToolArgs()
            );

            // 结果加入上下文，继续循环
            context += "\nObservation: " + result;
        }
        return "任务超时";
    }
}
```

### 5.4 完整对比表

| 维度 | 单轮调用 | 多轮对话 | Agent |
|------|---------|---------|-------|
| 交互方式 | 一问一答 | 多次问答 | 给目标，自主执行 |
| 是否有记忆 | 无 | 对话历史 | 对话 + 工具结果 + 知识库 |
| 是否调用工具 | 否 | 否（或极少） | 是（核心能力） |
| 是否自主决策 | 否 | 否 | 是 |
| 循环次数 | 1 次 | N 次（用户驱动） | N 次（LLM 驱动） |
| 谁决定下一步 | 用户 | 用户 | LLM 自己决定 |
| 成本 | 低 | 中 | 高 |
| 稳定性 | 高 | 高 | 中（有不确定性） |
| 典型框架 | OpenAI API | ChatGPT | LangChain / LangGraph |
| 典型场景 | 翻译、摘要 | 客服对话 | 复杂任务自动化 |

### 5.5 面试怎么选（决策树）

```
你的任务需要什么？

只处理文本（翻译/摘要/分类）
  → 单轮 LLM 调用就够了

需要多轮对话理解上下文
  → 多轮对话（Chat）

需要调用外部工具/API
  → Agent

需要多个Agent协作
  → Multi-Agent（Stage 6 详讲）
```

---

## 6. Agent 完整生命周期（面试可以讲的执行流程）

```
┌──────────────────────────────────────────────────────────┐
│               Agent 完整执行生命周期                        │
│                                                           │
│  1. 接收目标                                               │
│     用户："帮我分析订单123456退款进度"                       │
│                     │                                     │
│                     ▼                                     │
│  2. 加载上下文                                             │
│     → 从 Memory 加载对话历史                                │
│     → 从 RAG 检索相关知识                                   │
│     → 加载可用工具列表                                      │
│                     │                                     │
│                     ▼                                     │
│  3. Planner 决策（第 1 轮）                                 │
│     Thought: "需要先查订单状态"                              │
│     Action: query_order("123456")                          │
│                     │                                     │
│                     ▼                                     │
│  4. Executor 执行                                          │
│     → 找到 query_order 工具                                 │
│     → 参数校验 + 权限检查                                   │
│     → 调用 OrderService.query("123456")                    │
│     Observation: "订单已申请退款，退款审核中"                  │
│                     │                                     │
│                     ▼                                     │
│  5. Planner 决策（第 2 轮）                                 │
│     Thought: "需要查退款审核进度"                             │
│     Action: query_refund_status("123456")                  │
│                     │                                     │
│                     ▼                                     │
│  6. Executor 执行                                          │
│     Observation: "退款审核通过，预计3个工作日到账"              │
│                     │                                     │
│                     ▼                                     │
│  7. Planner 判断：信息完整，生成最终回答                      │
│     Final Answer: "订单123456的退款已审核通过..."             │
│                     │                                     │
│                     ▼                                     │
│  8. 后处理                                                 │
│     → 保存对话到 Memory                                     │
│     → 记录审计日志                                          │
│     → 更新监控指标                                          │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

---

## 7. Agent 的类型（面试展示知识面）

### 7.1 按对话模式分类

| 类型 | 说明 | 示例 |
|------|------|------|
| **对话式 Agent** | 通过自然语言对话交互 | ChatGPT + 插件 |
| **任务式 Agent** | 给一个目标，自主完成 | AutoGPT / Devin |
| **辅助式 Agent** | 辅助人类完成任务（Copilot） | GitHub Copilot / Cursor |

### 7.2 按自主程度分类

| 类型 | 说明 | 示例 |
|------|------|------|
| **ReAct Agent** | 推理-行动交替循环 | LangChain Agent |
| **Plan-Execute Agent** | 先规划后执行 | LangGraph Workflow |
| **反射式 Agent（Reflexion）** | 执行后自我反思和改进 | 学术实验为主 |
| **层级式 Agent** | 多层 Agent 嵌套 | Supervisor 模式 |

### 7.3 面试怎么说

> 面试官问"你了解哪些 Agent 类型"，回答要有层次：
>
> "从交互模式分，有对话式、任务式、辅助式三种。对话式适合客服场景，任务式适合自动化流程，辅助式适合开发工具。
>
> 从执行模式分，最核心的是 ReAct 模式——推理和行动交替循环。更高级的有 Plan-Execute 模式，先规划后执行，稳定性更好。
>
> 在实际工程中，2026 年最主流的是 Workflow + Agent 混合模式——确定的流程用 Workflow，灵活的决策用 Agent。"

---

## 8. Java 类比总结（面试必背，一张表讲清）

面试时如果面试官是 Java 背景，用这个类比非常加分：

> **Agent 本质上就是一个"自动化决策系统"，类比到 Java 后端：**
>
> | Agent 组件 | Java 类比 | 说明 |
> |-----------|----------|------|
> | Planner | `DispatcherServlet` / 网关路由 | 决定分发到哪，但 Agent 用 LLM 替代了 if-else |
> | Tools | `@Service` 层的业务方法 | 具体干活的能力 |
> | Memory (短期) | `ThreadLocal` / `HttpSession` | 方法级/会话级的状态 |
> | Memory (长期) | `Redis` + `MySQL` | 持久化的状态和知识 |
> | Executor | `反射调用` / `RPC 框架` | 根据签名找到实现并执行 |
> | Tool Calling | `HTTP RPC` / `Feign 调用` | 远程调用外部服务 |
> | RAG | `Elasticsearch 查询` | 检索相关知识 |
> | Prompt | `SQL 查询语句` / `配置文件` | 定义处理逻辑的"代码" |
>
> **LLM 充当的是 Planner 中的"决策大脑"，用自然语言理解来替代硬编码的 if-else 路由逻辑。**
>
> 所以 Agent 不是魔法，它是一个**用 LLM 替代硬编码决策、用工具调用替代纯文本输出**的工程系统。

---

## 9. 面试实战

### 【面试回答（标准版）】—— 30 秒口语化

> Agent 和普通 LLM 调用的核心区别在于**自主性**。
>
> 普通 LLM 是单次的请求-响应，你问什么它答什么，没有工具调用能力，也没有状态管理。
>
> Agent 是一个由 LLM 驱动的**决策执行系统**，它有四个核心组件：Planner 负责规划下一步做什么，Tools 提供外部调用能力，Memory 管理上下文和状态，Executor 执行具体操作。
>
> 简单来说，LLM 只能"说话"，Agent 能"做事"——它接收一个目标后，会自主规划、调用工具、观察结果、迭代决策，直到完成目标。
>
> 类比到 Java 后端，就像是把传统的 if-else 路由逻辑替换成了 LLM 做动态决策，底层调用的还是我们熟悉的 Service 方法。

### 【面试回答（深入版）】—— 1 分钟（面试官让你展开讲时用）

> 我从三个层面来理解 Agent：
>
> **第一，它和 LLM 的本质区别是自主性。** LLM 是无状态的概率预测引擎，给它文本它返回文本，仅此而已。Agent 是在 LLM 之上包了一层"决策-执行"循环——LLM 充当大脑做规划，框架负责执行工具调用和管理记忆。
>
> **第二，Agent 的核心是四个组件。** Planner 用 LLM 替代传统的硬编码路由，Tools 把外部能力标准化为可调用的接口，Memory 分三层管理从短期对话到长期知识的上下文，Executor 是 LLM 和实际工具之间的安全桥梁。
>
> **第三，Agent 的自主性是一个光谱。** 从 L0 的纯 LLM 调用到 L4 的多 Agent 协作，不是所有场景都需要最高自主性。在工程实践中，我会根据场景选择合适的等级——简单查询用 Chain，需要工具调用的用 Agent，高风险操作加人工确认。
>
> 落地到 Java 中，就是把 Agent 看成后端系统的一个新组件——Planner 对应 DispatcherServlet，Tools 对应 Service 层，Memory 对应 Redis + MySQL，整体架构思路和传统后端一脉相承。

### 【面试官可能追问】

**追问 1：Agent 一定能比普通 LLM 调用效果更好吗？**

> 不一定。Agent 的优势在于**需要多步推理和工具调用的场景**。
>
> 如果任务很简单（翻译一句话、总结一段文字），直接调 LLM 就够了，用 Agent 反而增加了复杂度和延迟。
>
> Agent 适合的场景是：目标复杂、需要外部信息、需要多步决策。比如"帮我分析这个代码仓库的安全漏洞并生成报告"。
>
> 工程上的取舍：Agent 的调用成本（Token 消耗 + 延迟）远高于单次 LLM 调用，所以不是所有场景都适合。**能用 Chain 解决的不上 Agent，能用单次调用的不上 Chain。**

**追问 2：你说 Agent 有 Memory，这和 ChatGPT 的对话历史有什么区别？**

> ChatGPT 的对话历史本质上只是把之前的对话拼接到 prompt 里，是一种**短期记忆**。
>
> Agent 的 Memory 更丰富，分三层：
> - **L1 工作记忆**：当前 Prompt 中最近几轮对话（受 Token 窗口限制）
> - **L2 短期记忆**：Redis 存完整会话历史，可以做摘要压缩
> - **L3 长期记忆**：向量数据库存用户偏好和历史精华，跨会话持久化
>
> 关键区别是 Agent 的 Memory 是**面向工具调用和任务执行**的，不只是对话文本。它需要记住"调了什么工具、返回了什么结果、任务进行到哪一步了"。
>
> 落地到 Java 中，短期记忆用 Redis，长期记忆用向量数据库 + MySQL 配合使用。

**追问 3：Agent 的"自主决策"靠谱吗？不会出错吗？**

> 这是 Agent 最大的工程挑战——**LLM 的决策不可靠**。
>
> 可能出现的问题：
> - 选错工具（工具描述不精确）
> - 传错参数（LLM 理解偏差）
> - 陷入死循环（一直调同一个工具）
> - 过早认为任务完成（信息不完整就给出结论）
> - 产生幻觉（编造不存在的工具调用结果）
>
> 工程上的解决方案：
> - **限制最大迭代次数**（通常 5~10 次，防止死循环）
> - **工具描述要精确**（减少选错概率）
> - **参数 Schema 校验**（LLM 输出的参数做类型和范围校验）
> - **加人工确认环节**（高风险操作前 Human-in-the-loop）
> - **Fallback 机制**（Agent 失败时走预设的安全流程）
> - **全链路 Tracing**（记录每一步决策，方便排查问题）

**追问 4：你说 Agent 用 LLM 替代了 if-else，那 LLM 的"路由"和传统路由有什么优劣？**

> 好问题，这是工程选型的核心。
>
> **传统 if-else 路由的优势：**
> - 确定性高，100% 可预测
> - 延迟低，无额外 API 调用
> - 成本低，无 Token 消耗
>
> **LLM 路由的优势：**
> - 灵活性高，能处理模糊/未预定义的输入
> - 维护成本低，不需要手动维护路由规则
> - 可扩展，新场景不需要改代码
>
> **工程上的选择：**
> - 路由规则明确且稳定 → 传统 if-else（如支付渠道选择）
> - 路由规则复杂/频繁变化 → LLM 路由（如客服意图识别）
> - 混合使用 → 传统路由做粗筛，LLM 路由做精细判断

**追问 5：你对 Agent 的未来发展怎么看？**

> 我觉得有三个方向：
>
> **1）标准化：** MCP 这类协议会继续成熟，工具调用会像 REST API 一样标准化。
>
> **2）多 Agent 协作：** 单个 Agent 的能力有限，Multi-Agent 是解决复杂问题的必然方向。
>
> **3）可靠性提升：** 通过更好的 Prompt 工程、输出校验、人工兜底，Agent 的可靠性会逐步提升到生产可用水平。
>
> 但我认为短期内 Agent 不会替代传统后端核心链路——它的定位是辅助（Copilot），不是替代（Autopilot）。在 Java 后端中，Agent 更像是一个新的中间件层。

### 【常见错误——这些话不要说】

| 错误说法 | 为什么错 | 正确说法 |
|---------|---------|---------|
| "Agent 就是更智能的 ChatGPT" | Agent 不是"更智能的聊天"，而是能自主调用工具的系统 | "Agent 是由 LLM 驱动的决策执行系统" |
| "Agent 就是让 AI 自己写代码" | 这是 Agent 的一种应用，不是本质 | "Agent 能调用各种工具，代码执行只是其中一种" |
| "Agent 不需要人干预" | 过度夸大，实际工程中必须有护栏机制 | "Agent 减少了人工干预，但关键环节仍需要 Human-in-the-loop" |
| "Agent 会取代传统后端" | Agent 和后端是协作关系，不是替代 | "Agent 是后端系统的一个新组件，用 LLM 替代硬编码决策逻辑" |
| "Agent 就是一个 while 循环调 LLM" | 过度简化，忽略了 Planner/Memory/Tool 的复杂交互 | "Agent 是一个包含规划、记忆、工具调用、执行的完整系统" |
| "LLM 有记忆功能" | LLM 本身无状态，"记忆"是通过外部拼接实现的 | "LLM 本身无状态，Agent 通过 Memory 组件维护上下文" |
| "Agent 的工具调用是 LLM 直接执行的" | LLM 只输出 JSON 指令，框架代为执行 | "LLM 输出结构化调用指令，Executor 代为执行" |

---

## 10. 一句话总结（背诵用）

> **Agent = LLM 做大脑 + Tools 做手脚 + Memory 做记忆 + Planner 做决策，本质是把传统的硬编码自动化系统升级为"自然语言驱动的动态决策系统"，类比到 Java 就是把 DispatcherServlet 的 if-else 路由换成 LLM 动态决策。**

---

> **Stage 1（详细版）结束。后续 Stage 2~7 已在同名文件夹中。**
