# Stage 6：Agent 进阶（拉开差距 · 详细版）

> 目标：冲击中大厂，能讲 Multi-Agent 架构、Workflow vs Agent 的选型、Agent 的工程问题和全套优化方案。
> 面试定位：这些内容能把"只会调 API"和"真正有工程经验"区分开。

---

## 1. Multi-Agent（多 Agent 协作）

### 1.1 一句话理解

> **Multi-Agent = 多个专职 Agent 各司其职，通过协作完成复杂任务。**
> 类比到 Java：**微服务架构**——单个 Agent 像单体应用，Multi-Agent 像微服务集群。

### 1.2 为什么需要 Multi-Agent

单个 Agent 的问题：

| 问题 | 说明 |
|------|------|
| **工具太多** | 装 20 个工具，LLM 选择准确率骤降到 50% 以下 |
| **职责混乱** | 既查数据库又写代码还做客服，Prompt 写不清楚 |
| **上下文爆炸** | 所有信息塞进一个 Agent 的 Memory，Token 不够用 |
| **难以调试** | 出问题不知道是哪个环节的锅 |
| **不可扩展** | 加一个新功能就要改 Prompt，牵一发动全身 |

Multi-Agent 的思路：**拆职责，每个 Agent 只做一件事。**

### 1.3 Multi-Agent 协作模式

```
模式一：主从模式（Supervisor）—— 最常用
┌──────────────┐
│  Supervisor   │ ← 接收用户请求，分发任务，汇总结果
│  Agent（主管） │
└──────┬───────┘
       │ 分发任务
  ┌────┼────┬────────┐
  ▼    ▼    ▼        ▼
┌───┐┌───┐┌───┐  ┌────┐
│搜索││代码││数据│  │审核│
│Agent│Agent│Agent│Agent│
└───┘└───┘└───┘  └────┘
  │    │    │        │
  └────┴────┴────────┘
       │ 结果汇总
       ▼
  返回给用户

类比：API网关 + 微服务集群

模式二：流水线模式（Pipeline）
┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
│ 需求  │───▶│ 编码  │───▶│ 测试  │───▶│ 部署  │
│ Agent │    │ Agent │    │ Agent │    │ Agent │
└──────┘    └──────┘    └──────┘    └──────┘

类比：CI/CD Pipeline

模式三：辩论模式（Debate）
┌──────┐  反驳   ┌──────┐
│ 正方  │◀══════▶│ 反方  │
│ Agent │        │ Agent │
└──┬───┘        └───┬──┘
   │                │
   └───────┬────────┘
           ▼
      ┌────────┐
      │  Judge  │ ← 裁判 Agent 综合判断
      │  Agent  │
      └────────┘

类比：代码 Review 中 Author vs Reviewer 的讨论

模式四：层级模式（Hierarchical）
┌──────────────┐
│  CEO Agent   │ ← 最高决策
└──────┬───────┘
  ┌────┴────┐
  ▼         ▼
┌─────┐  ┌─────┐
│ CTO  │  │ COO  │ ← 中层管理
│Agent │  │Agent │
└──┬──┘  └──┬──┘
   │         │
  ┌┴┐      ┌┴┐
  ▼ ▼      ▼ ▼
Worker    Worker     ← 执行层
Agent     Agent

类比：公司组织架构
```

### 1.4 Multi-Agent 通信方式（面试必问）

| 方式 | 说明 | Java 类比 | 适用场景 |
|------|------|----------|---------|
| **共享状态** | 所有 Agent 读写同一个 State 对象 | Redis 共享 Session | LangGraph |
| **消息传递** | Agent 之间通过消息传递结果 | MQ（Kafka/RabbitMQ） | 解耦场景 |
| **上下文拼接** | 上游输出直接拼到下游 Prompt | Request Attribute | 简单串行 |

```
通信的关键问题：上下文大小控制

错误做法：
  Agent A 的完整输出（5000 Token）→ 全部传给 Agent B
  → Agent B 的 Prompt 直接超限

正确做法：
  Agent A 的输出 → 提取关键信息（500 Token）→ 传给 Agent B
  → 类比：方法返回值应该是精简的 DTO，不是整个 Entity
```

### 1.5 Multi-Agent 框架对比

| 框架 | 语言 | 特点 | Java 类比 |
|------|------|------|----------|
| **LangGraph** | Python | 底层灵活，StateGraph 编排 | Activiti 工作流引擎 |
| **CrewAI** | Python | 上层易用，角色定义式 | 低代码平台 |
| **AutoGen** | Python | 微软出品，对话式协作 | 消息队列 + 消费者 |
| **MetaGPT** | Python | 模拟软件公司组织架构 | 项目管理系统 |

### 1.6 面试怎么讲 Multi-Agent

> Multi-Agent 的本质是**分治思想**——把复杂任务拆分给专职 Agent，类比微服务架构。
>
> 最常用的是主从模式：Supervisor Agent 负责理解和分发，Worker Agent 各自专职处理一类任务。类似于 API 网关 + 微服务集群。
>
> 通信方面推荐共享状态（LangGraph 的 StateGraph），简单高效。关键要控制上下文大小——Agent 之间的传递要精简，不能把完整输出全传过去。
>
> 框架方面，LangGraph 偏底层灵活（适合复杂定制），CrewAI 偏上层易用（适合快速验证）。

---

## 2. Workflow vs Agent（重要对比，面试高频）

### 2.1 核心区别

| 维度 | Workflow（工作流） | Agent（智能体） |
|------|-------------------|-----------------|
| 决策方式 | **预定义的固定流程** | **LLM 动态决策** |
| 流程确定性 | 确定性，步骤写死 | 不确定，运行时决定 |
| 稳定性 | 高（可预测） | 低（有随机性） |
| 灵活性 | 低（改流程要改代码） | 高（自动适应） |
| 成本 | 低（只有必要节点调 LLM） | 高（每步都调 LLM 做决策） |
| 调试难度 | 低（流程可审查） | 高（路径不确定） |
| 类比 | BPMN 工作流（Activiti） | 自适应路由系统 |
| 适合 | 流程确定的场景 | 流程不确定的场景 |

### 2.2 面试怎么说

> Workflow 和 Agent 的核心区别是"谁做决策"——Workflow 是开发者提前写好流程，Agent 是 LLM 实时决定。
>
> **工程上的选择原则：能确定流程的用 Workflow，不确定的用 Agent。**
>
> 实际项目中，通常是 Workflow + Agent 混合：
> - 外层用 Workflow 管大流程（输入 → 预处理 → 核心处理 → 后处理 → 输出）
> - 内层某个节点用 Agent 处理灵活决策
>
> 类比到 Java：Workflow 像 Spring Batch 的 Job/Step 定义，Agent 像 Job 里的某个 Step 内部用了 LLM 做动态路由。

### 2.3 决策树

```
任务流程确定吗？
├── 是 → 用 Workflow（Chain / DAG）
│        例：翻译 → 摘要 → 格式化
│        例：数据清洗 → 特征提取 → 模型推理
│
└── 否 → 需要调用外部工具吗？
         ├── 否 → 用 Chain + RAG 就够了
         │        例：知识库问答
         │
         └── 是 → 用 Agent（ReAct）
                  例：分析代码 → 搜索文档 → 调API → 生成报告
                  例：查订单 → 查物流 → 判断是否需要通知用户
```

---

## 3. Agent 的工程问题与全套优化方案

### 3.1 问题一：不稳定（Non-determinism）

**表现：** 同一输入，不同时间走不同路径，结果不一致。

**根因：** LLM 输出有随机性，即使 temperature=0 也不完全确定。

**全套解决方案：**

| 层级 | 方案 | 说明 | Java 类比 |
|------|------|------|----------|
| 输入层 | temperature=0 | 减少随机性 | 关闭随机负载均衡 |
| 工具层 | 精确工具描述 | 减少选错概率 | 缩小路由表 |
| 执行层 | 重试机制 | 失败自动重试（指数退避） | Spring Retry |
| 校验层 | 输出 Schema 校验 | LLM 输出做结构化校验 | `@Valid` 参数校验 |
| 兜底层 | Fallback | Agent 失败走预设流程 | 熔断降级 |
| 监控层 | 全链路 Tracing | 记录每步决策 | APM 链路追踪 |

```java
// Agent 稳定性保障——完整实现
public class RobustAgentExecutor {

    private final AgentEngine agentEngine;
    private final WorkflowEngine fallbackWorkflow;
    private final AgentConfig config;

    public AgentResponse execute(AgentRequest request) {
        // 1. 预检查
        if (isHighRiskOperation(request)) {
            return executeWithHumanApproval(request);
        }

        // 2. Agent 执行（带重试）
        for (int attempt = 0; attempt < config.getMaxRetries(); attempt++) {
            try {
                AgentResponse response = agentEngine.execute(request);

                // 3. 输出质量校验
                if (validateOutput(response)) {
                    // 4. 质量评分
                    double qualityScore = evaluateQuality(response);
                    if (qualityScore >= config.getQualityThreshold()) {
                        return response;
                    }
                }
            } catch (Exception e) {
                log.warn("Agent执行失败，第{}次重试", attempt + 1, e);
                sleepWithBackoff(attempt);
            }
        }

        // 5. 所有重试失败，走 Fallback
        log.warn("Agent执行全部失败，走Fallback流程");
        return fallbackWorkflow.execute(request);
    }
}
```

### 3.2 问题二：成本高（Cost）

**表现：** Agent 每步都调 LLM，一次任务可能消耗数万 Token。

**全套优化方案：**

| 方案 | 节省幅度 | 实现方式 | Java 类比 |
|------|---------|---------|----------|
| **分级模型** | 60%~80% | 简单决策用小模型，复杂推理用大模型 | 分库分表（冷热分离） |
| **工具缓存** | 30%~50% | 相同参数的工具调用做缓存 | Redis 缓存 |
| **精简 Prompt** | 20%~40% | System Prompt 和工具描述精简 | SQL 优化（减少 SELECT *） |
| **提前终止** | 10%~30% | 检测到"足够好"就停止 | 短路求值 |
| **批量处理** | 20%~40% | 多个工具调用合并一次 LLM 请求 | 批量插入 |
| **语义缓存** | 40%~60% | 语义相似的请求复用历史结果 | CDN 缓存 |

```java
// 分级模型策略
public class TieredModelStrategy {

    public AgentResponse execute(AgentRequest request) {
        // 第一步：用小模型判断复杂度
        ComplexityLevel level = smallModel.classifyComplexity(request);

        return switch (level) {
            case SIMPLE -> {
                // 简单任务：用小模型（成本 $0.01/次）
                yield smallModelAgent.execute(request);
            }
            case MEDIUM -> {
                // 中等任务：用中等模型（成本 $0.05/次）
                yield mediumModelAgent.execute(request);
            }
            case COMPLEX -> {
                // 复杂任务：用大模型（成本 $0.30/次）
                yield largeModelAgent.execute(request);
            }
        };
    }
}

// 工具调用缓存
public class CachedToolExecutor {
    private final Cache<String, ToolResult> cache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build();

    public ToolResult execute(String toolName, Map<String, Object> args) {
        String cacheKey = toolName + ":" + digest(args);

        return cache.get(cacheKey, key -> {
            return actualExecutor.execute(toolName, args);
        });
    }
}
```

### 3.3 问题三：幻觉（Hallucination）

**表现：** LLM 编造不存在的信息，自信地给出错误回答。

**全套优化方案：**

| 方案 | 说明 | Java 类比 |
|------|------|----------|
| **RAG 强制引用** | 要求标注每句话的引用来源 | 论文引用 |
| **事实校验** | 第二个 LLM 对输出做事实核查 | Code Review |
| **置信度评估** | 让 LLM 自评置信度，低于阈值转人工 | 健康检查 |
| **Human-in-the-loop** | 关键场景加人工审核 | 审批流程 |
| **工具结果优先** | 明确要求"只基于工具返回的数据回答" | 单一数据源 |
| **多模型交叉验证** | 两个 LLM 独立回答，结果一致才输出 | 双写校验 |

```
幻觉防护层级：

Level 1 - Prompt 约束
  "请只基于提供的参考资料回答，不要编造信息"

Level 2 - RAG 引用
  "回答时请标注每条信息的来源文档"

Level 3 - 自评置信度
  "请评估你对此回答的置信度（1-10）"

Level 4 - 交叉验证
  用第二个 LLM 验证第一个 LLM 的回答

Level 5 - 人工兜底
  置信度 < 7 或涉及高风险操作 → 转人工
```

### 3.4 问题四：延迟高（Latency）

**表现：** Agent 多轮循环，总耗时 = 步骤数 × 每步 LLM 延迟（1~3s）。

**全套优化方案：**

| 方案 | 优化效果 | 实现方式 |
|------|---------|---------|
| **流式输出** | 用户感知延迟降低 50%+ | SSE / WebSocket |
| **并行工具调用** | 多工具并行节省总时间 | `CompletableFuture` |
| **减少循环次数** | 总延迟直接减少 | 优化 Prompt + 限制步骤 |
| **预计算缓存** | 常见查询提前算好 | Redis 预热 |
| **模型推理加速** | 单步延迟降低 | 量化 / 推理优化 |
| **分级模型** | 简单步骤用快速小模型 | 路由策略 |

```java
// 延迟优化：并行工具调用
public class ParallelToolExecutor {

    private final ExecutorService executor = Executors.newFixedThreadPool(5);

    public List<ToolResult> executeParallel(List<ToolCall> calls) {
        List<CompletableFuture<ToolResult>> futures = calls.stream()
            .map(call -> CompletableFuture.supplyAsync(
                () -> toolRegistry.execute(call.getName(), call.getArgs()),
                executor
            ))
            .toList();

        // 等所有完成，取结果
        return futures.stream()
            .map(f -> f.exceptionally(e -> ToolResult.error(e.getMessage())).join())
            .toList();
    }
}

// 流式输出
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String message) {
    return agentEngine.streamExecute(message)
        .map(chunk -> "data: " + chunk + "\n\n");
}
```

---

## 4. Agent 的可观测性（Observability）—— 生产必备

### 4.1 为什么需要可观测性

```
Agent 是一个"黑盒"：
  输入 → [??? 多轮 LLM 调用 + 工具执行 ???] → 输出

没有可观测性：
  - 输出错了，不知道是哪一步出的错
  - 延迟高，不知道卡在哪个环节
  - 成本高，不知道哪步消耗最多 Token
  - 无法优化，因为不知道瓶颈在哪
```

### 4.2 可观测性三件套

| 维度 | 工具 | 监控什么 |
|------|------|---------|
| **Tracing** | LangSmith / Jaeger | 每一步的输入输出、耗时、Token |
| **Metrics** | Prometheus + Grafana | QPS / 延迟分布 / 成功率 / Token 消耗 |
| **Logging** | ELK / Loki | 每步决策日志、工具调用记录、错误详情 |

### 4.3 关键监控指标

```java
// Agent 监控指标设计
@Component
public class AgentMetrics {

    private final MeterRegistry registry;

    // 1. 执行成功率
    public void recordExecution(boolean success) {
        registry.counter("agent.execution", "status", success ? "success" : "fail").increment();
    }

    // 2. 总延迟
    public void recordLatency(long latencyMs) {
        registry.timer("agent.latency").record(latencyMs, TimeUnit.MILLISECONDS);
    }

    // 3. Token 消耗
    public void recordTokens(int tokens) {
        registry.counter("agent.tokens").increment(tokens);
    }

    // 4. 工具调用统计
    public void recordToolCall(String toolName, boolean success, long latencyMs) {
        registry.counter("agent.tool.call", "tool", toolName,
                         "status", success ? "success" : "fail").increment();
    }

    // 5. 循环次数分布
    public void recordIterations(int iterations) {
        registry.histogram("agent.iterations").record(iterations);
    }
}
```

---

## 5. 面试实战

### 【面试回答（标准版）】—— 30 秒口语化

> Multi-Agent 是微服务思想在 AI 领域的应用——把复杂任务拆分给专职 Agent，常见模式有主从、流水线、辩论三种。
>
> Workflow 和 Agent 的核心区别是"谁做决策"——Workflow 预定义流程、Agent 动态决策。工程上通常混合使用。
>
> Agent 的三大工程问题是不稳定、高成本、幻觉。我的优化策略是：分级模型（简单用小模型节省 60% 成本）、工具调用缓存、强制引用校验、失败走 Fallback、全链路 Tracing。
>
> 类比到 Java 后端，这些优化思路和熔断、降级、限流完全一致——核心就是"加护栏、有兜底、可观测"。

### 【面试官可能追问】

**追问 1：Multi-Agent 的通信开销怎么处理？**

> 三种通信方式：共享状态（LangGraph）、消息传递（MQ）、上下文拼接。
> 关键是控制上下文大小——Agent 之间传递精简摘要而非完整输出，类比 Java 中方法返回 DTO 而非 Entity。

**追问 2：怎么评估场景该不该上 Agent？**

> 三个维度：任务复杂度（3步以内不需要）、是否需要外部信息（不需要就用 Chain）、容错空间（高风险谨慎）。原则：简单确定用 Workflow，复杂灵活用 Agent，高风险加 Human-in-the-loop。

**追问 3：Agent 的未来怎么看？**

> 标准化（MCP）、多 Agent 协作、可靠性提升是三个方向。但短期内不会替代后端核心链路——定位是 Copilot 辅助，不是 Autopilot 替代。Agent 更适合 B 端企业内部落地（客服、运维、数据分析）。

### 【常见错误】

| 错误说法 | 正确说法 |
|---------|---------|
| "Multi-Agent 就是多个 ChatGPT 一起聊天" | "是有分工的协作系统，类比微服务" |
| "Agent 会完全替代传统后端" | "Agent 是后端的补充，适合辅助决策" |
| "Agent 不需要人干预" | "关键环节需要 Human-in-the-loop" |
| "Workflow 过时了，全用 Agent" | "各有适用场景，通常混合使用" |

---

## 6. 一句话总结（背诵用）

> **Multi-Agent 是微服务思想在 AI 领域的应用（分治）；Workflow 做确定流程、Agent 做灵活决策；三大工程问题（不稳定/高成本/幻觉）的优化思路和 Java 后端的熔断/降级/限流完全一致——核心是"加护栏、有兜底、可观测"。**

---

> **Stage 6（详细版）结束。**
