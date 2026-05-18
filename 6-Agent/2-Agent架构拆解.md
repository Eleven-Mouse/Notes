# Stage 2：Agent 架构拆解（核心面试区 · 详细版）

> 目标：能在白板上画出 Agent 架构图，讲清 ReAct / Plan-Execute / Reflexion 全部执行模式，
> 讲透 Tool Calling 的底层原理，并给出完整的 Java 接口设计和生产级实现。

---

## 1. ReAct 模式（最重要，面试必考）

### 1.1 什么是 ReAct

ReAct = **Re**asoning + **Act**ing，即"推理 + 行动"交替进行。

这是目前 Agent 最主流的执行范式，由 Yao et al. 2022 年论文《ReAct: Synergizing Reasoning and Acting in Language Models》提出。

**核心思想：LLM 先"想"再"做"，根据"做"的结果再"想"，循环往复直到任务完成。**

### 1.2 ReAct 与 Chain-of-Thought 的关系（面试加分点）

```
Chain-of-Thought（CoT，2022）：
  LLM 通过"逐步推理"提升复杂问题的准确率
  → 但只是"想"，没有"做"的能力
  → "我想了想，答案应该是42"

ReAct（2022）：
  在 CoT 的基础上加入"行动"能力
  → "我想了想，但我需要先查一下数据"
  → 调用工具获取真实信息
  → 基于真实信息继续推理
  → CoT + Tool Use = ReAct
```

**面试怎么讲：**

> ReAct 可以理解为 Chain-of-Thought 的"行动增强版"。CoT 让 LLM 学会"逐步思考"，ReAct 在此基础上加了"逐步行动"——每一步思考后可以调用工具获取真实信息，然后再继续思考。这样就弥合了"推理"和"执行"之间的鸿沟。

### 1.3 ReAct 执行循环（文字架构图）

```
用户目标："帮我查北京明天天气，如果下雨推荐室内景点"
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│                   ReAct Agent Loop                    │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │  第 1 轮                                   │        │
│  │                                            │        │
│  │  Thought（思考）                            │        │
│  │  "我需要先查北京明天的天气"                   │        │
│  │      ↓                                     │        │
│  │  Action（行动）                              │        │
│  │  调用工具: weather_api("北京", "明天")        │        │
│  │      ↓                                     │        │
│  │  Observation（观察）                         │        │
│  │  结果: 小雨，15°C，湿度80%                   │        │
│  └──────────────────────────────────────────┘        │
│                    │                                  │
│                    ▼                                  │
│  ┌──────────────────────────────────────────┐        │
│  │  第 2 轮                                   │        │
│  │                                            │        │
│  │  Thought（再思考）                           │        │
│  │  "明天下雨，需要推荐室内景点"                  │        │
│  │      ↓                                     │        │
│  │  Action（再行动）                            │        │
│  │  调用工具: search("北京室内景点推荐")          │        │
│  │      ↓                                     │        │
│  │  Observation（再观察）                        │        │
│  │  结果: 故宫、国家博物馆、798艺术区...          │        │
│  └──────────────────────────────────────────┘        │
│                    │                                  │
│                    ▼                                  │
│  ┌──────────────────────────────────────────┐        │
│  │  第 3 轮（LLM 判断信息足够）                  │        │
│  │                                            │        │
│  │  Final Answer（最终回答）                    │        │
│  │  "北京明天小雨15°C，推荐室内景点：            │        │
│  │   故宫、国家博物馆、798艺术区"                │        │
│  └──────────────────────────────────────────┘        │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### 1.4 ReAct 的 Prompt 模板（面试深入理解必备）

ReAct 的核心是用 Prompt 模板约束 LLM 的输出格式：

```
System: 你是一个智能助手，可以通过调用工具来回答问题。

你可以使用以下工具：
{tools_description}

请严格按照以下格式回复：

Question: 用户的问题
Thought: 你的思考过程
Action: 要调用的工具名称
Action Input: 工具的参数（JSON格式）
Observation: （工具执行结果会自动填入）

...（可以多轮 Thought/Action/Observation）...

Thought: 我现在知道最终答案了
Final Answer: 最终回答

开始！

Question: {user_question}
```

> **关键理解：ReAct 的"智能"不是来自特殊的模型能力，而是来自 Prompt 模板对输出格式的约束。**
> 任何足够强大的 LLM + 合适的 Prompt 模板 = ReAct Agent。

### 1.5 ReAct 的 Java 完整实现（生产级）

```java
/**
 * ReAct 引擎 —— 完整的生产级实现
 *
 * 核心逻辑：while 循环 + LLM 决策 + 工具执行 + 上下文累积
 */
public class ReActEngine implements AgentEngine {

    private final LLMClient llmClient;
    private final ToolRegistry toolRegistry;
    private final MemoryManager memoryManager;
    private final AgentConfig config;

    // ReAct Prompt 模板
    private static final String REACT_PROMPT = """
        你是一个智能助手。请按以下格式回复：

        如果需要调用工具：
        Thought: [你的思考]
        Action: [工具名]
        Action Input: [JSON参数]

        如果可以回答了：
        Thought: [你的思考]
        Final Answer: [最终答案]

        可用工具：
        %s

        对话历史：
        %s
        """;

    @Override
    public AgentResponse execute(AgentRequest request) {
        List<StepRecord> steps = new ArrayList<>();
        int totalTokens = 0;
        long startTime = System.currentTimeMillis();

        // 构建初始上下文
        String toolDescriptions = formatToolDescriptions(request.getTools());
        String memoryContext = memoryManager.buildContext(request.getSessionId());
        String context = String.format(REACT_PROMPT, toolDescriptions, memoryContext)
                       + "\n\nQuestion: " + request.getGoal();

        for (int i = 0; i < config.getMaxIterations(); i++) {
            long stepStart = System.currentTimeMillis();

            // === Thought + Action: LLM 决策 ===
            LLMResponse llmResponse = llmClient.chat(context);
            totalTokens += llmResponse.getTokensUsed();

            // 检测死循环：连续 3 次调用相同工具相同参数
            if (isStuckInLoop(steps, llmResponse)) {
                break;
            }

            // 判断是否是最终回答
            if (llmResponse.isFinalAnswer()) {
                // 保存到 Memory
                memoryManager.save(request.getSessionId(),
                    Message.assistant(llmResponse.getFinalAnswer()));

                return AgentResponse.builder()
                    .answer(llmResponse.getFinalAnswer())
                    .steps(steps)
                    .totalTokens(totalTokens)
                    .latencyMs(System.currentTimeMillis() - startTime)
                    .build();
            }

            // === Action: 执行工具 ===
            StepRecord step = StepRecord.builder()
                .step(i + 1)
                .thought(llmResponse.getThought())
                .action(llmResponse.getToolName())
                .args(llmResponse.getToolArgs())
                .build();

            try {
                ToolResult result = toolRegistry.execute(
                    llmResponse.getToolName(),
                    llmResponse.getToolArgs()
                );
                step.setObservation(result.getOutput());
                step.setSuccess(true);
            } catch (Exception e) {
                step.setObservation("工具执行失败: " + e.getMessage());
                step.setSuccess(false);
            }

            step.setDurationMs(System.currentTimeMillis() - stepStart);
            steps.add(step);

            // === Observation: 更新上下文 ===
            context += String.format(
                "\n\nThought: %s\nAction: %s\nAction Input: %s\nObservation: %s",
                step.getThought(), step.getAction(),
                toJson(step.getArgs()), step.getObservation()
            );

            // Token 预算检查
            if (estimateTokens(context) > config.getMaxContextTokens()) {
                context = compressContext(context, config.getMaxContextTokens());
            }
        }

        // 超过最大迭代次数
        return AgentResponse.builder()
            .answer("抱歉，我无法在限定步骤内完成这个任务")
            .steps(steps)
            .totalTokens(totalTokens)
            .latencyMs(System.currentTimeMillis() - startTime)
            .build();
    }

    // 死循环检测
    private boolean isStuckInLoop(List<StepRecord> steps, LLMResponse current) {
        if (steps.size() < 2) return false;
        int last3 = steps.size() - 3;
        for (int i = Math.max(0, last3); i < steps.size(); i++) {
            StepRecord s = steps.get(i);
            if (s.getAction().equals(current.getToolName())
                && s.getArgs().equals(current.getToolArgs())) {
                return true;
            }
        }
        return false;
    }
}
```

### 1.6 ReAct 的工程问题与解决（面试深入追问）

| 问题 | 具体表现 | 根因 | 解决方案 |
|------|---------|------|---------|
| **死循环** | 反复调同一个工具 | LLM 没有从 Observation 中学到教训 | 循环检测 + 最大迭代次数 |
| **推理发散** | 忘了原始目标 | 上下文太长，注意力分散 | 每轮重申目标 + 摘要压缩 |
| **工具选错** | 调了不相关的工具 | 工具描述不够精确 | 优化描述 + 限制工具数量 ≤ 10 |
| **参数错误** | 传了错误的参数格式 | LLM 对 Schema 理解偏差 | 参数校验 + 重试 |
| **过早结束** | 信息不完整就给答案 | LLM 判断力不足 | 要求标注信息来源 |
| **上下文溢出** | Token 超出窗口 | 步骤太多 | 动态压缩 + 截断旧步骤 |

---

## 2. Plan-Execute 模式

### 2.1 什么是 Plan-Execute

Plan-Execute 是对 ReAct 的改进：**先做完整规划，再逐步执行，执行中可以调整计划。**

**ReAct 的问题：** 每一步都即时决策，没有全局视角，容易"走偏"或重复。

**Plan-Execute 的思路：**
1. **Plan 阶段**：LLM 一次性生成完整计划（步骤列表）
2. **Execute 阶段**：按计划逐步执行
3. **Re-Plan 阶段**（可选）：根据执行结果调整计划

### 2.2 文字架构图

```
用户目标："分析这段 Java 代码的性能问题并给出优化方案"
        │
        ▼
┌──────────────────────────────────────────────────┐
│  Phase 1: Planner（规划阶段）                       │
│                                                    │
│  LLM 生成执行计划：                                  │
│  ┌────────────────────────────────────────────┐   │
│  │ Step 1: 调用 code_analyzer 做静态分析        │   │
│  │ Step 2: 调用 search 查询常见性能反模式        │   │
│  │ Step 3: 调用 profiler 分析热点方法            │   │
│  │ Step 4: 综合分析，生成优化建议                │   │
│  │ Step 5: 给出优化后的代码示例                  │   │
│  └────────────────────────────────────────────┘   │
│                                                    │
│  输出：Plan = [Step1, Step2, Step3, Step4, Step5]   │
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────┐
│  Phase 2: Executor（执行阶段）                      │
│                                                    │
│  执行 Step 1 → 结果: 发现3个热点方法                 │
│  执行 Step 2 → 结果: 常见反模式列表                  │
│  执行 Step 3 → 结果: 数据库查询占70%耗时             │
│      ↓                                             │
│      发现新信息：数据库是主要瓶颈                      │
│      ↓                                             │
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────┐
│  Phase 3: Re-Plan（重新规划，可选）                  │
│                                                    │
│  "发现数据库查询是瓶颈，需要额外查 SQL 优化方案"       │
│  新增 Step 3.5: 调用 search("SQL优化最佳实践")       │
│                                                    │
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
              继续执行剩余步骤...
```

### 2.3 Plan-Execute 的 Java 实现

```java
public class PlanExecuteEngine implements AgentEngine {

    private final LLMClient llmClient;
    private final ToolRegistry toolRegistry;

    @Override
    public AgentResponse execute(AgentRequest request) {
        // === Phase 1: Plan ===
        String planPrompt = buildPlanPrompt(request.getGoal(), request.getTools());
        LLMResponse planResponse = llmClient.chat(planPrompt);
        List<PlanStep> plan = parsePlan(planResponse.getText());

        // === Phase 2: Execute ===
        String executionContext = "";
        List<StepRecord> steps = new ArrayList<>();

        for (int i = 0; i < plan.size(); i++) {
            PlanStep currentStep = plan.get(i);

            // 执行当前步骤（可以用 ReAct 执行单个步骤）
            StepRecord step = executeStep(currentStep, executionContext);
            steps.add(step);
            executionContext += "\nStep " + (i+1) + " 结果: " + step.getObservation();

            // === Phase 3: Re-Plan（每 3 步检查一次）===
            if ((i + 1) % 3 == 0 && i < plan.size() - 1) {
                RePlanResult rePlan = checkRePlan(
                    request.getGoal(), plan, executionContext, i + 1
                );
                if (rePlan.needsRePlan()) {
                    // 用新计划替换剩余步骤
                    plan = mergePlan(plan, rePlan.getNewSteps(), i + 1);
                }
            }
        }

        // === 生成最终回答 ===
        String finalAnswer = generateFinalAnswer(request.getGoal(), executionContext);
        return AgentResponse.builder()
            .answer(finalAnswer)
            .steps(steps)
            .build();
    }
}
```

### 2.4 ReAct vs Plan-Execute 详细对比

| 维度 | ReAct | Plan-Execute |
|------|-------|-------------|
| 决策方式 | 每步即时决策 | 先规划再执行 |
| 全局视角 | 无（走一步看一步） | 有（先看全貌） |
| 灵活性 | 高（随时调整） | 中（需Re-Plan） |
| 稳定性 | 低（容易跑偏） | 高（有全局计划） |
| Token 消耗 | 可能较多（循环不确定） | 相对可控（计划固定步骤数） |
| 适用场景 | 简单/探索性任务 | 复杂多步任务 |
| 调试难度 | 高（路径不确定） | 低（计划可审查） |
| 类比 | 敏捷开发（迭代调整） | 瀑布开发（先规划后执行） |

### 2.5 面试怎么说

> 实际工程中，通常不是二选一，而是**混合使用**：
> - 外层用 Plan-Execute 管理大流程
> - 内层某个步骤用 ReAct 灵活处理
> - 定期 Re-Plan 确保方向正确
>
> 类比到项目管理：Plan-Execute 是项目计划（里程碑+时间线），ReAct 是每日站会（根据实际情况灵活调整当天工作）。

---

## 3. Reflexion 模式（面试加分——展示前沿知识）

### 3.1 什么是 Reflexion

> **Reflexion = ReAct + 自我反思。Agent 执行完任务后，会"回顾"自己的决策过程，找出不足，下次做得更好。**

2023 年论文《Reflexion: Language Agents with Verbal Reinforcement Learning》提出。

### 3.2 Reflexion 执行流程

```
┌───────────────────────────────────────────────────┐
│              Reflexion 循环                          │
│                                                     │
│  第 1 次尝试（正常的 ReAct）                          │
│  Thought → Action → Observation → ... → Answer      │
│      ↓                                              │
│  反思（Reflection）                                   │
│  "我第一次尝试选错了工具，应该先用数据库查询..."        │
│      ↓                                              │
│  第 2 次尝试（带着反思经验）                           │
│  Thought（参考反思结论）→ Action → ... → Better Answer│
│      ↓                                              │
│  再次反思...                                         │
│      ↓                                              │
│  直到结果满意                                        │
│                                                     │
└───────────────────────────────────────────────────┘
```

### 3.3 面试怎么讲

> Reflexion 是在 ReAct 基础上加了一个"自我反思"环节。Agent 完成任务后，会让 LLM 审查整个决策过程，找出可以改进的地方，把反思结论存入 Memory，下次执行类似任务时会参考。
>
> 类比到 Java：就像 CI/CD 中跑了测试发现 bug → 自动修复 → 重新跑测试 → 直到通过。Reflexion 是在 LLM 层面做这种"自动修复"。
>
> 实际工程中，Reflexion 的成本很高（每次反思都要额外调 LLM），所以通常只在离线评估阶段用，不在线上实时链路中使用。

---

## 4. Tool Calling 深度原理（2026 必考）

### 4.1 Tool Calling 的本质

> **LLM 不执行任何工具，它只是输出一个结构化的"调用指令"（JSON），由 Agent 框架代为执行。**

这是一个"间接调用"模式，不是"直接执行"模式。

```
┌────────────────────────────────────────────────────────┐
│             Tool Calling 的本质：间接调用                  │
│                                                         │
│  LLM 的能力：                                           │
│  ✅ 理解自然语言                                        │
│  ✅ 推理和决策                                          │
│  ✅ 输出结构化 JSON                                     │
│  ❌ 不能执行代码                                        │
│  ❌ 不能访问网络                                        │
│  ❌ 不能读写文件                                        │
│  ❌ 不能调数据库                                        │
│                                                         │
│  所以 LLM 的"调用工具"实际上是：                          │
│  1. LLM 理解用户意图                                    │
│  2. LLM 决定需要哪个工具                                 │
│  3. LLM 输出 JSON 格式的调用指令                         │
│  4. Agent 框架接住这个 JSON                              │
│  5. 框架调用真正的函数/接口                               │
│  6. 框架把结果喂回 LLM                                   │
│                                                         │
│  类比：                                                  │
│  LLM = 将军（下命令）                                    │
│  框架 = 传令兵（传递命令）                                │
│  Tool  = 士兵（执行命令）                                │
│                                                         │
└────────────────────────────────────────────────────────┘
```

### 4.2 Tool Calling 的 HTTP 层面发生了什么（面试能讲这个很加分）

```
=== 第一次 HTTP 请求 ===

Request:
POST https://api.openai.com/v1/chat/completions
{
  "model": "gpt-4o",
  "messages": [
    {"role": "user", "content": "北京明天天气怎么样？"}
  ],
  "tools": [                                    ← 工具描述随请求一起发送
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "查询指定城市的天气",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {"type": "string", "description": "城市名称"},
            "date": {"type": "string", "description": "日期，如'明天'"}
          },
          "required": ["city"]
        }
      }
    }
  ]
}

Response:                                       ← LLM 没有直接回答，而是返回了工具调用指令
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,                          ← 注意：content 为空
      "tool_calls": [{                          ← 返回的是工具调用指令
        "id": "call_abc123",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\":\"北京\",\"date\":\"明天\"}"
        }
      }]
    }
  }]
}

=== Agent 框架拿到响应后，执行真正的天气API调用 ===
假设结果为：小雨，15°C，湿度80%

=== 第二次 HTTP 请求（带着工具结果再问 LLM）===

Request:
POST https://api.openai.com/v1/chat/completions
{
  "model": "gpt-4o",
  "messages": [
    {"role": "user", "content": "北京明天天气怎么样？"},
    {"role": "assistant", "tool_calls": [{"id": "call_abc123", ...}]},
    {"role": "tool",                             ← 工具执行结果
      "tool_call_id": "call_abc123",
      "content": "小雨，15°C，湿度80%" }
  ],
  "tools": [...]
}

Response:                                       ← LLM 基于工具结果生成最终回答
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "北京明天小雨，气温15°C，湿度80%，建议带伞出门。"
    }
  }]
}
```

> **关键理解：一次完整的 Tool Calling = 两次 LLM API 调用 + 一次工具执行。**
> 这也是为什么 Agent 的成本和延迟更高。

### 4.3 多工具并行调用

2024+ 的 LLM 支持在一次响应中返回多个工具调用：

```json
// LLM 一次返回两个工具调用
{
  "tool_calls": [
    {"id": "call_1", "function": {"name": "get_weather", "arguments": "{\"city\":\"北京\"}"}},
    {"id": "call_2", "function": {"name": "get_weather", "arguments": "{\"city\":\"上海\"}"}}
  ]
}
```

```java
// 并行执行多个工具调用
public List<ToolResult> executeParallel(List<ToolCall> toolCalls) {
    return toolCalls.parallelStream()
        .map(call -> toolRegistry.execute(call.getName(), call.getArgs()))
        .toList();
}
```

### 4.4 为什么 Tool Calling 这样设计（安全性深度理解）

> **为什么不让 LLM 直接执行工具？**

```
假设 LLM 能直接执行代码，看看会发生什么：

用户："帮我删除所有过期的订单"
LLM: 直接执行 DELETE FROM orders WHERE ... → 💥 数据全没了！

或者更危险的情况——提示注入攻击：
用户输入中隐藏了恶意指令
LLM 被误导，执行了危险操作 → 💥

所以必须是"间接调用"模式：
1. LLM 只能"建议"调什么工具
2. 框架可以做参数校验（拒绝危险参数）
3. 框架可以做权限检查（这个用户能调这个工具吗？）
4. 框架可以做审计日志（谁在什么时间调了什么）
5. 高风险操作可以加人工确认（Human-in-the-loop）
```

### 4.5 工具描述的工程最佳实践

```json
// ❌ 差的工具描述
{
  "name": "query",
  "description": "查询数据",
  "parameters": {
    "type": "object",
    "properties": {
      "q": {"type": "string"}
    }
  }
}
// 问题：名字太模糊、描述太笼统、参数没有说明

// ✅ 好的工具描述
{
  "name": "query_order",
  "description": "根据订单号查询订单详情，返回订单状态、金额、商品列表和物流信息。当用户询问订单相关问题时使用此工具。",
  "parameters": {
    "type": "object",
    "properties": {
      "orderId": {
        "type": "string",
        "description": "订单号，格式为纯数字，如 '123456789'"
      },
      "detailLevel": {
        "type": "string",
        "enum": ["simple", "full"],
        "description": "simple只返回状态，full返回完整详情"
      }
    },
    "required": ["orderId"]
  }
}
```

**工具描述的 5 条规则：**

| 规则 | 说明 |
|------|------|
| 1. 名字动词开头 | `query_order` 而不是 `order` |
| 2. 描述说三件事 | 做什么 + 返回什么 + 什么时候该用 |
| 3. 参数有约束 | type + enum + description 一个不能少 |
| 4. 区分必填选填 | required 只放真正必须的 |
| 5. 总数不超过 10 | 工具太多 LLM 选择准确率骤降 |

---

## 5. Function Calling vs Tool Calling（区分清楚）

### 5.1 演进历史

```
2023.06  OpenAI 发布 Function Calling（GPT-4）
         ↓ LLM 首次能输出结构化的函数调用 JSON

2023.11  OpenAI 发布 Assistants API
         ↓ 集成了 Function Calling + Code Interpreter + File Search

2024.01  OpenAI 将 Function Calling 升级为 Tool Calling
         ↓ 统一了函数调用、代码执行、文件检索等所有"外部能力"的接口

2024.11  Anthropic 发布 MCP
         ↓ 把 Tool Calling 的标准从厂商级提升到行业级

2025-2026  各厂商跟进 MCP，Tool Calling 逐步标准化
```

### 5.2 对比表

| 维度 | Function Calling | Tool Calling |
|------|-----------------|-------------|
| 来源 | OpenAI 术语（2023） | OpenAI 术语（2024+）/ 通用术语 |
| 范围 | 特指函数级别的调用 | 更广泛，包含函数、API、MCP服务等 |
| 格式 | OpenAI 专有格式 | 逐渐标准化（JSON Schema） |
| 标准化程度 | 厂商私有 | 走向开放标准（MCP） |
| 本质 | 是 Tool Calling 的一种 | 是更通用的概念 |

### 5.3 面试怎么说

> Function Calling 是 OpenAI 2023 年提出的概念，让 LLM 能输出结构化的函数调用请求。2024 年 OpenAI 将其升级为 Tool Calling，统一了所有外部能力调用的接口。
>
> 现在 Tool Calling 是更广义的概念——不仅包含函数调用，还包括 MCP 工具调用、代码执行、API 调用等。
>
> 2026 年的趋势是 MCP 走向标准化，Tool Calling 的格式和协议会统一。

---

## 6. 面试实战

### 【面试回答（标准版）】—— 30 秒口语化

> Agent 的核心执行模式是 ReAct，即 Reasoning + Acting 的循环。
>
> 具体来说，LLM 先思考当前应该做什么，然后选择一个工具调用，观察执行结果，再基于结果继续思考。这个 Thought → Action → Observation 的循环会一直进行，直到 LLM 认为已经可以给出最终答案。
>
> ReAct 本质上是 Chain-of-Thought 的"行动增强版"——CoT 让 LLM 学会逐步推理，ReAct 在此基础上加了工具调用能力。
>
> 对于复杂任务，我们会用 Plan-Execute 模式——先让 LLM 生成一个完整的执行计划，然后逐步执行，每 3 步可以做一次 Re-Plan。
>
> 工具调用的本质是 LLM 输出结构化的 JSON 指令，由 Agent 框架代为执行。一次完整的 Tool Calling 涉及两次 LLM API 调用和一次工具执行。LLM 本身不直接调用任何外部工具——这样设计是为了安全性，可以在框架层加权限校验、参数校验和审计日志。
>
> 落地到 Java 中，我会设计一个 ToolRegistry 来管理工具的注册和调用，类似于 Spring 的 IoC 容器管理 Bean 的方式。

### 【面试回答（深入版）】—— 1 分钟

> Agent 的架构核心是 ReAct 循环，但不是唯一的执行模式。
>
> **ReAct** 适合简单、探索性的任务——每步即时决策，灵活但不稳定。我在实现中会加死循环检测、Token 预算管理和上下文压缩。
>
> **Plan-Execute** 适合复杂多步任务——先规划全局步骤再逐步执行，稳定性更好但灵活性差一些。我会在每 3 步做一次 Re-Plan 检查。
>
> **Reflexion** 是更前沿的模式——执行完任务后自我反思，把经验存入 Memory 下次参考。但成本高，适合离线评估。
>
> 实际工程中，我推荐**混合模式**：外层 Plan-Execute 管大流程，内层每个步骤用 ReAct 灵活执行。
>
> 工具调用方面，关键理解是 LLM 不执行工具，只输出 JSON 指令。这个设计有三层安全价值：参数校验、权限控制、审计追踪。我的 Java 实现是在 Executor 层统一做这三件事。

### 【面试官可能追问】

**追问 1：ReAct 模式有什么问题？你怎么解决？**

> ReAct 有六个主要问题，我逐个说：
>
> **1）死循环**：LLM 可能反复调用同一个工具。
> - 解决：循环检测（比较最近 3 步）+ 最大迭代次数限制。
>
> **2）推理发散**：中间步骤多了，LLM 容易"忘记"原始目标。
> - 解决：每轮 prompt 中重申原始目标 + 已完成步骤的摘要。
>
> **3）工具选择错误**：LLM 选了不合适的工具。
> - 解决：优化工具描述 + 限制可选工具数量 ≤ 10。
>
> **4）参数格式错误**：LLM 输出的参数不符合 Schema。
> - 解决：JSON Schema 校验 + 重试 + Structured Output 强制格式。
>
> **5）过早结束**：信息不完整就给出答案。
> - 解决：要求回答时标注信息来源，缺失信息的不能直接回答。
>
> **6）上下文溢出**：步骤太多超过 Token 窗口。
> - 解决：动态摘要压缩旧步骤，只保留最近 3 轮的完整记录。

**追问 2：你提到 Tool Calling 是 LLM 输出 JSON，如果 LLM 输出的 JSON 格式不对怎么办？**

> 四层防护：
>
> **第一层：Prompt 约束。** System prompt 中明确要求输出 JSON 格式并给出示例。
>
> **第二层：Structured Output。** GPT-4o / Claude 都支持强制按 Schema 输出，在 API 层面约束。
>
> **第三层：解析兜底。** try-catch 包裹 JSON 解析，失败时最多重试 3 次。
>
> **第四层：Fallback。** 多次重试仍失败，返回预设错误提示，不崩系统。

**追问 3：Plan-Execute 和 ReAct 能结合使用吗？**

> 可以，而且实际工程中经常这样用。
>
> 具体做法：
> - 先 Plan：让 LLM 生成一个步骤列表
> - 每一步用 ReAct 执行：灵活处理意外情况
> - 定期 Re-Plan：每完成几个步骤，重新评估计划
>
> 类比到项目管理：Plan-Execute 是项目计划（里程碑），ReAct 是每日站会（灵活调整），Re-Plan 是季度复盘（调整方向）。
>
> 框架方面，LangGraph 的 StateGraph 天然支持这种混合模式。

**追问 4：如果 Agent 需要调用 20 个工具怎么办？**

> 这是一个常见的工程问题。工具太多，LLM 的选择准确率会急剧下降。
>
> 解决方案有三种：
>
> **1）工具分组**：按业务域把 20 个工具分成 4 组，先让小模型做意图分类确定用哪组，然后只把那 5 个工具的描述传给 Agent。
>
> **2）工具检索**：把工具描述也向量化，根据用户问题检索最相关的 5 个工具。
>
> **3）Multi-Agent**：每个 Agent 只装 5 个工具，Supervisor Agent 做路由分发。
>
> 实际项目中，我推荐方案 1——实现简单，效果确定。

### 【Java 工程落地——完整接口设计】

```java
// ========== 1. 核心领域模型 ==========

// Agent 执行引擎（顶层接口）
public interface AgentEngine {
    AgentResponse execute(AgentRequest request);
}

// Agent 请求
@Data @Builder
public class AgentRequest {
    private String goal;                    // 用户目标
    private String sessionId;               // 会话ID
    private List<ToolDefinition> tools;     // 可用工具列表
    private AgentConfig config;             // 配置
}

// Agent 响应
@Data @Builder
public class AgentResponse {
    private String answer;                  // 最终回答
    private List<StepRecord> steps;         // 执行步骤记录
    private int totalTokens;                // Token 消耗
    private long latencyMs;                 // 总耗时
    private boolean success;                // 是否成功
}

// 单步执行记录（可观测性核心）
@Data @Builder
public class StepRecord {
    private int step;                       // 步骤序号
    private String thought;                 // LLM 的思考
    private String action;                  // 调用的工具名
    private Map<String, Object> args;       // 工具参数
    private String observation;             // 工具返回结果
    private long durationMs;                // 本步耗时
    private boolean success;                // 本步是否成功
    private int tokensUsed;                 // 本步Token消耗
}

// ========== 2. 工具系统 ==========

// 工具定义
public interface ToolDefinition {
    String getName();
    String getDescription();
    JsonSchema getParameterSchema();
}

// 工具执行结果
@Data @Builder
public class ToolResult {
    private String output;
    private boolean success;
    private String error;

    public static ToolResult success(String output) {
        return ToolResult.builder().output(output).success(true).build();
    }
    public static ToolResult error(String error) {
        return ToolResult.builder().error(error).success(false).build();
    }
}

// 工具注册中心（类似 Spring ApplicationContext）
@Component
public class ToolRegistry {
    private final Map<String, ToolDefinition> definitions = new ConcurrentHashMap<>();
    private final Map<String, BiFunction<Map<String, Object>, ToolResult>> executors = new ConcurrentHashMap<>();

    public void register(String name, ToolDefinition def,
                         BiFunction<Map<String, Object>, ToolResult> executor) {
        definitions.put(name, def);
        executors.put(name, executor);
    }

    public List<ToolDefinition> getAllDefinitions() {
        return new ArrayList<>(definitions.values());
    }

    public ToolResult execute(String toolName, Map<String, Object> args) {
        // 1. 工具存在性检查
        if (!executors.containsKey(toolName)) {
            return ToolResult.error("工具不存在: " + toolName);
        }
        // 2. 参数校验
        ValidationResult v = validateParams(toolName, args);
        if (!v.isValid()) return ToolResult.error(v.getErrors());
        // 3. 执行
        return executors.get(toolName).apply(args);
    }
}

// ========== 3. Agent 配置 ==========

@Data @Builder
public class AgentConfig {
    @Builder.Default private int maxIterations = 8;
    @Builder.Default private int maxContextTokens = 100000;
    @Builder.Default private String model = "gpt-4o";
    @Builder.Default private double temperature = 0.0;
    @Builder.Default private boolean enableTracing = true;
    @Builder.Default private boolean enableLoopDetection = true;
}
```

### 【常见错误——这些话不要说】

| 错误说法 | 为什么错 | 正确说法 |
|---------|---------|---------|
| "LLM 会直接调用你的函数" | LLM 不执行任何代码，只输出 JSON 指令 | "LLM 输出结构化调用指令，由框架代为执行" |
| "ReAct 就是循环调 LLM" | 不只是循环，核心是 Thought-Action-Observation 推理链 | "ReAct 是推理和行动交替进行的决策循环" |
| "工具越多 Agent 越强" | 工具太多反而干扰 LLM 选择 | "工具要精简且描述精确，一般不超过 10 个" |
| "Plan-Execute 比 ReAct 好" | 各有适用场景 | "简单任务用 ReAct，复杂任务用 Plan-Execute，实际中常混合使用" |
| "Tool Calling 是 OpenAI 独有的" | Claude / 通义等主流模型都支持 | "Tool Calling 是 2024+ 所有主流 LLM 的标配能力" |
| "一次 Tool Calling 只调一个工具" | 新版模型支持并行多工具调用 | "主流模型已支持单次返回多个工具调用" |

---

## 7. 一句话总结（背诵用）

> **Agent 的核心执行模式是 ReAct（Thought→Action→Observation 循环），它是 Chain-of-Thought 的行动增强版；LLM 只负责决策不执行，Tool Calling 本质是两次 LLM API 调用加一次工具执行的"间接调用"模式；复杂任务用 Plan-Execute（先规划后执行），更前沿的 Reflexion 加了自我反思但成本高。**

---

> **Stage 2（详细版）结束。**
