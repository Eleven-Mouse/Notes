# Stage 3：LangChain 全链路（详细版）

> 目标：面试时能讲"我用 LangChain 做过什么"，讲清每个模块的原理、版本演进和工程坑。

---

## 1. LangChain 是什么

### 1.1 一句话定义

> **LangChain 是一个 LLM 应用编排框架，本质是"把 LLM 调用 + 工具 + 记忆 + 数据源 串起来的胶水层"。**

### 1.2 为什么要用 LangChain（先理解痛点）

```
没有 LangChain 时，你做一个 Agent 需要自己写：

1. LLM 调用封装（处理不同模型的API差异）
2. Prompt 模板管理（字符串拼接 + 变量注入）
3. 对话历史管理（截断、摘要、Token计数）
4. 工具调用的 ReAct 循环（while + JSON解析 + 错误处理）
5. RAG 检索流程（文档分片 + Embedding + 向量库 + 拼接）
6. 链式调用编排（上一个组件的输出 = 下一个组件的输入）

→ 这些都是重复的"样板代码"，LangChain 帮你统一封装了
```

### 1.3 Java 类比

> **LangChain 之于 LLM 应用，就像 Spring Boot 之于 Java 后端。**
> Spring Boot 把 IOC + AOP + Web + 数据源编排在一起；
> LangChain 把 LLM + Tool + Memory + Retriever 编排在一起。

---

## 2. LangChain 核心模块全景

```
┌────────────────────────────────────────────────────────────┐
│                  LangChain 模块全景图                        │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               核心层（langchain-core）                  │  │
│  │  Model │ Prompt │ Output Parser │ LCEL（管道语法）     │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               Agent 层（langchain）                     │  │
│  │  Agents │ Chains │ Memory │ Tools                     │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               数据层（langchain-community）             │  │
│  │  Retriever │ Document Loaders │ Vector Stores         │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               编排层（langgraph）                       │  │
│  │  StateGraph（有状态图编排，支持分支/循环/并行）          │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               可观测性（langsmith）                     │  │
│  │  Tracing │ Evaluation │ Dataset Management            │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### 2.1 Model（模型层）

**是什么：** 对不同 LLM 的统一封装，一套接口可以切换底层模型。

**Java 类比：** JDBC——统一接口，底层可以接 MySQL / PostgreSQL / Oracle。

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# 换模型只改这一行
llm = ChatOpenAI(model="gpt-4o", temperature=0)
# llm = ChatAnthropic(model="claude-sonnet-4-20250514")

# 调用方式完全一致
response = llm.invoke("什么是JVM？")
```

**Model 层的两个核心抽象：**

| 抽象 | 说明 | Java 类比 |
|------|------|----------|
| `BaseLLM` | 纯文本输入输出 | `HttpClient` |
| `BaseChatModel` | 消息列表输入（支持 system/user/assistant/tool 角色） | `HttpClient` + 结构化请求体 |

### 2.2 Prompt Template（提示词模板）

**是什么：** 管理 Prompt 的模板引擎，支持变量注入和消息组合。

**Java 类比：** Thymeleaf / `String.format()`

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# 基础模板
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}，擅长{skill}。回答要简洁。"),
    ("human", "{question}")
])

# 带历史对话的模板（Memory 集成）
prompt_with_history = ChatPromptTemplate.from_messages([
    ("system", "你是一个客服助手"),
    MessagesPlaceholder("history"),  # 对话历史占位符
    ("human", "{question}")
])
```

### 2.3 Output Parser（输出解析器）

**是什么：** 把 LLM 的文本输出转换成结构化数据。

```python
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser

# 简单文本解析
str_parser = StrOutputParser()

# JSON 解析（自动在 Prompt 中加入 JSON 格式要求）
class OrderInfo(BaseModel):
    order_id: str = Field(description="订单号")
    status: str = Field(description="订单状态")

json_parser = JsonOutputParser(pydantic_object=OrderInfo)

# 使用
chain = prompt | llm | json_parser
result = chain.invoke({"question": "查一下订单123456"})
# result = {"order_id": "123456", "status": "已发货"}
```

### 2.4 LCEL（LangChain Expression Language）—— 核心！

**是什么：** LangChain v0.2 引入的管道语法，用 `|` 操作符串联组件。

```python
# LCEL 管道语法
chain = prompt | llm | output_parser

# 等价于 Java：
# String result = outputParser.parse(llm.invoke(prompt.format(params)))
```

**LCEL 的三大能力：**

| 能力 | 说明 | Java 类比 |
|------|------|----------|
| **Streaming** | 流式输出，边生成边返回 | `Flux<String>` (WebFlux) |
| **Batch** | 批量处理多个输入 | `parallelStream()` |
| **Async** | 异步执行 | `CompletableFuture` |

```python
# 流式输出
for chunk in chain.stream({"question": "什么是JVM？"}):
    print(chunk, end="", flush=True)

# 批量处理
results = chain.batch([
    {"question": "什么是JVM？"},
    {"question": "什么是GC？"},
])

# 异步
result = await chain.ainvoke({"question": "什么是JVM？"})
```

**LCEL 的 Runnable 接口（类比 Java 的 Function）：**

```java
// Java 等价抽象
@FunctionalInterface
public interface Runnable<I, O> {
    O invoke(I input);
    default Flux<O> stream(I input) { ... }
    default List<O> batch(List<I> inputs) { ... }
}
```

### 2.5 Chains（链）

**是什么：** 将多个组件串起来，形成处理流水线。

**版本演进（面试展示知识面）：**

| 版本 | 方式 | 状态 |
|------|------|------|
| v0.1 | `LLMChain(llm=llm, prompt=prompt)` | **已废弃** |
| v0.2+ | LCEL: `prompt \| llm \| parser` | **推荐** |
| v0.3+ | LangGraph `StateGraph` 做复杂编排 | **最新** |

**常见 Chain 类型：**

| Chain | 用途 | Java 类比 |
|-------|------|----------|
| LLM Chain | 基础 LLM 调用 | 单次 Service 调用 |
| Sequential Chain | 多个 Chain 串行 | 责任链 |
| Router Chain | 根据输入选择子链 | `@RequestMapping` 路由 |

### 2.6 Agents（智能体）

**是什么：** LangChain 中的 Agent 实现，内置了 ReAct 循环。

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor

# 定义工具
@tool
def query_order(order_id: str) -> str:
    """根据订单号查询订单详情"""
    return orderService.query(order_id)

@tool
def query_logistics(order_id: str) -> str:
    """查询订单的物流信息"""
    return logisticsService.track(order_id)

tools = [query_order, query_logistics]

# 创建 Agent
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是客服助手，可以查询订单和物流"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),  # Agent 中间步骤
])

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,               # 打印每步日志
    max_iterations=8,           # 最大迭代次数
    handle_parsing_errors=True  # 解析错误自动处理
)

result = agent_executor.invoke({"input": "查一下订单123456到哪了"})
```

**AgentExecutor 关键参数：**

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `max_iterations` | 最大循环次数 | 5~10 |
| `max_execution_time` | 最大执行时间 | 60s |
| `handle_parsing_errors` | 解析错误处理 | True |
| `verbose` | 详细日志 | 开发True/生产False |
| `return_intermediate_steps` | 返回中间步骤 | 调试时True |

### 2.7 Memory（记忆）

**Memory 类型详解：**

| 类型 | Token 消耗 | 适合场景 | Java 类比 |
|------|----------|---------|----------|
| `ConversationBufferMemory` | 高（全量） | 短对话 | `ArrayList<Message>` |
| `ConversationBufferWindowMemory` | 低 | 长对话 | `RingBuffer` |
| `ConversationSummaryMemory` | 中 | 需要完整上下文 | Redis存摘要 |
| `VectorStoreRetrieverMemory` | 低 | 跨会话长期记忆 | ES 语义检索 |

```python
from langchain.memory import ConversationBufferWindowMemory

memory = ConversationBufferWindowMemory(k=5, return_messages=True)
# 只保留最近 5 轮对话，类比 Java 的 RingBuffer
```

### 2.8 Retriever（检索器—— RAG 核心）

```python
from langchain_community.vectorstores import Milvus
from langchain_openai import OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter

# RAG 完整流程
# 1. 分片
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# 2. 向量化并存储
vectorstore = Milvus.from_documents(chunks, OpenAIEmbeddings())

# 3. 创建检索器
retriever = vectorstore.as_retriever(search_type="mmr", search_kwargs={"k": 5})
```

---

## 3. LangChain 执行流程（面试必讲）

### 3.1 Agent 完整调用链路

```
用户输入："帮我查一下订单123456的物流状态"
        │
        ▼
┌──────────────────────────────────────────────────┐
│  1. Prompt Template                                │
│     System指令 + 工具描述 + Memory + 用户输入      │
│     + agent_scratchpad（中间步骤占位符）            │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│  2. LLM 决策（第 1 次 API 调用）                    │
│     返回 tool_calls: query_logistics({orderId})    │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│  3. AgentExecutor 执行工具                         │
│     Observation: "顺丰SF1234，已到北京分拣中心"      │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│  4. LLM 决策（第 2 次 API 调用）                    │
│     判断信息已足够 → Final Answer                   │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│  5. Memory 更新 + 返回给用户                        │
└──────────────────────────────────────────────────┘
```

---

## 4. LangGraph（LangChain 的升级版编排）

### 4.1 为什么需要 LangGraph

```
LangChain 的局限：
  Chain 是线性的：A → B → C → D
  不能：条件分支、循环、并行、状态共享

LangGraph 的能力：
  用图（Graph）抽象编排：
  - 节点（Node）：一个处理步骤
  - 边（Edge）：步骤之间的跳转
  - 条件边：根据结果走不同路径
  - 状态（State）：节点间共享数据
```

| 维度 | LangChain | LangGraph |
|------|-----------|-----------|
| 编排方式 | 线性管道 | 图（有环有向图） |
| 条件分支 | 不支持 | 支持 |
| 循环 | 不支持 | 支持（Agent Loop） |
| 状态管理 | 无 | State 对象 |
| 并行 | 不支持 | 支持 |
| Java 类比 | 责任链模式 | 工作流引擎（Activiti） |

```python
from langgraph.graph import StateGraph, END

class AgentState(TypedDict):
    messages: list
    next_action: str

# 构建图
graph = StateGraph(AgentState)
graph.add_node("intent", intent_recognition)
graph.add_node("order_agent", order_agent)
graph.add_node("logistics_agent", logistics_agent)

graph.add_conditional_edges("intent", route_intent)  # 条件路由
graph.add_edge("order_agent", END)
graph.add_edge("logistics_agent", END)

app = graph.compile()
```

---

## 5. LangChain 版本演进（面试展示技术视野）

| 版本 | 时间 | 核心变化 | 面试怎么讲 |
|------|------|---------|----------|
| v0.1 | 2023.10 | 初始版本，`LLMChain` | "最早用过，API已废弃" |
| v0.1.x | 2024.01 | LCEL 引入 `|` 管道语法 | "开始用LCEL重构" |
| v0.2 | 2024.05 | `langchain-core` 拆分，`Runnable` 统一 | "核心稳定，推荐版本" |
| v0.3 | 2024.10 | 弃用旧API，`langgraph` 成熟 | "迁移到LangGraph编排" |
| v0.3.x | 2025+ | MCP 集成，`Structured Output` | "最新版本，生产级" |

---

## 6. LangChain 的坑（面试加分）

### 坑 1：版本迭代太快

> **问题：** v0.1 的 `LLMChain` 在 v0.2 标记 deprecated，v0.3 移除。
> **解决：** 锁定版本 + 只看官方文档 + 关注 Changelog。

### 坑 2：Agent 执行不稳定

> **问题：** 同样输入有时 2 步完成，有时 5 步或死循环。
> **解决：** `max_iterations=8` + 死循环检测 + Fallback 机制。

### 坑 3：Memory 超 Token

> **问题：** 长对话 Prompt 超出上下文窗口。
> **解决：** 滑动窗口（最近 10 轮）+ 摘要压缩（旧对话用 LLM 总结）。

### 坑 4：调试困难

> **问题：** 多轮 LLM 调用，出错不知道哪一步。
> **解决：** `verbose=True` + LangSmith 全链路 Tracing。

### 坑 5：Python 生态 vs Java 后端

> **问题：** LangChain 是 Python，公司后端是 Java。
> **解决方案：**
> - 方案 A：LangChain 部署为独立微服务，Java 通过 HTTP 调用
> - 方案 B：用 LangChain4j（Java 版 LangChain）
> - 方案 C：Spring AI（Spring 官方 AI 框架）

---

## 7. LangChain4j / Spring AI（Java 候选人必知）

### 7.1 LangChain4j

```java
// LangChain4j —— 声明式 AI Service
interface CustomerServiceAgent {
    @SystemMessage("你是客服助手，可以查询订单和物流信息")
    String chat(@UserMessage String userMessage);
}

CustomerServiceAgent agent = AiServices.builder(CustomerServiceAgent.class)
    .chatLanguageModel(chatModel)
    .tools(orderQueryTool, logisticsTool)
    .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
    .build();

String answer = agent.chat("查一下订单123456的物流");
```

### 7.2 Spring AI

```java
@RestController
public class AgentController {

    private final ChatClient chatClient;

    public AgentController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("你是客服助手")
            .defaultFunctions("queryOrder", "queryLogistics")
            .build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt().user(message).call().content();
    }
}
```

### 7.3 面试怎么选型

| 框架 | 语言 | 优势 | 适用场景 |
|------|------|------|---------|
| LangChain | Python | 生态最全、社区最大 | AI 编排微服务 |
| LangChain4j | Java | Java 原生 | Java 团队快速上手 |
| Spring AI | Java | Spring 生态无缝 | 已有 Spring Boot 项目 |
| LangGraph | Python | 复杂编排、状态管理 | 复杂 Agent 工作流 |

> **面试说法：** "如果团队纯 Java，推荐 LangChain4j 或 Spring AI。如果 Agent 逻辑复杂（Multi-Agent），建议 LangChain + LangGraph 独立微服务，Java 后端 API 调用。"

---

## 8. 面试实战

### 【面试回答（标准版）】—— 30 秒口语化

> LangChain 是 LLM 应用的编排框架，类似 AI 领域的 Spring Boot。
>
> 核心模块：Model 统一封装 LLM、Prompt Template 管理提示词、Chains 串联处理流程（LCEL 管道语法）、Agents 实现 ReAct 循环、Tools 注册外部工具、Memory 管理对话上下文。
>
> 我用 LangChain 搭建了客服 Agent，流程是 Prompt 拼接 → Agent 决策 → 工具调用 → 结果回传。踩过的坑：版本兼容、Agent 不稳定（限制迭代次数）、Memory 超 Token（滑动窗口+摘要压缩）。
>
> 复杂场景用 LangGraph（StateGraph 图编排，类比 Activiti 工作流引擎）。Java 项目可选 LangChain4j 或 Spring AI。

### 【面试官可能追问】

**追问 1：LangChain 和直接调 OpenAI API 有什么区别？**

> 直接调 API 是单次请求-响应。用 LangChain 的理由：工具调用编排、多模型切换、记忆管理、RAG 检索、可观测性。
> 但如果需求很简单，不需要 LangChain。**不要为了用框架而用框架。**

**追问 2：Chain 和 Agent 有什么区别？**

> Chain = 固定流程（类比 if-else 硬编码），Agent = LLM 动态决策（类比把 if-else 替换成 LLM 路由）。
> 选择原则：能用 Chain 的优先用 Chain（稳定、低成本），Chain 搞不定的才上 Agent。

**追问 3：你对 LangGraph 了解吗？**

> LangGraph 是 LangChain 团队的新一代编排框架，用图抽象：支持条件分支、循环、并行、状态管理。
> 类比到 Java：LangChain 像责任链模式，LangGraph 像 BPMN 工作流引擎（Activiti）。
> 简单场景用 LCEL，复杂编排用 LangGraph StateGraph。

**追问 4：为什么选 Python LangChain 而不是 Java 的 LangChain4j？**

> 三个原因：Python 生态更全（新模型/工具优先支持）、迭代更快、架构解耦（Agent 层独立微服务）。
> 当然纯 Java 团队用 LangChain4j/Spring AI 也合理。

### 【常见错误】

| 错误说法 | 正确说法 |
|---------|---------|
| "LangChain 是 AI 模型" | "LangChain 是编排框架" |
| "用了 LangChain 就不用写代码" | "减少样板代码，但核心工作仍在" |
| "所有项目都该用 LangChain" | "根据复杂度选择，简单场景直接调 API" |
| "LangChain 只能配合 OpenAI" | "统一接口，底层模型可插拔" |

---

## 9. 一句话总结（背诵用）

> **LangChain 是 LLM 应用的编排框架（AI 领域的 Spring Boot），核心是 LCEL 管道语法串联组件，Chain 做固定流程、Agent 做动态决策，LangGraph 做复杂工作流编排（类比 Activiti），Java 项目可选 LangChain4j 或 Spring AI。**

---

> **Stage 3（详细版）结束。**
