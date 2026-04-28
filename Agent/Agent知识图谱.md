# Agent & AI 工程化 · 知识图谱

> 面向 Java 后端候选人，目标：AI 平台 / Agent 开发 / 智能后端
> 格式：全景树 + 模块依赖链 + 面试链路 + 项目映射 + 一句话速查

---

## 一、Agent 技术全景树

```
Agent & AI 工程化技术体系
│
├── M1: LLM 基础 [地基]
│   ├── 1.1 LLM 本质（概率预测引擎/Token生成/无状态/上下文窗口）
│   ├── 1.2 工程特性（Token计费/输出不确定/幻觉/知识截止/只有文本）
│   ├── 1.3 调用方式（HTTP API/Streaming/Structured Output）
│   └── 1.4 模型选型（GPT-4o/Claude/通义千问/本地部署/分级模型策略）
│
├── M2: Agent 核心 [系统层]
│   ├── 2.1 Agent 定义（感知+决策+执行+达成目标/自主性等级L0~L4）
│   ├── 2.2 四大组件
│   │   ├── Planner（规划器/LLM充当/类比DispatcherServlet）
│   │   ├── Tools（工具集/信息查询+操作执行+计算处理/类比@Service）
│   │   ├── Memory（记忆/三层L1L2L3/类比ThreadLocal+Redis+MySQL）
│   │   └── Executor（执行器/参数校验+权限检查+审计/类比反射调用）
│   ├── 2.3 LLM vs Agent（自主性/工具调用/状态管理/单次vs循环）
│   └── 2.4 演进时间线（2022 ReAct → 2023 Function Calling → 2024 MCP → 2026 生产化）
│
├── M3: 执行模式 [引擎层]
│   ├── 3.1 ReAct（Thought→Action→Observation循环/CoT行动增强版）
│   │   ├── Prompt 模板约束（格式决定行为）
│   │   ├── 死循环检测（最近3步比较/最大迭代次数）
│   │   └── Token 预算管理（动态压缩+截断）
│   ├── 3.2 Plan-Execute（先规划后执行/Re-Plan/稳定性优先）
│   ├── 3.3 Reflexion（执行+自我反思+经验积累/成本高适合离线）
│   └── 3.4 混合模式（外层Plan-Execute + 内层ReAct/工程最优解）
│
├── M4: Tool Calling [通信层]
│   ├── 4.1 核心原理（LLM只输出JSON指令/框架代为执行/间接调用）
│   ├── 4.2 完整流程（注册工具→发请求→LLM决策→返回指令→框架执行→结果回传）
│   ├── 4.3 两次API调用（第1次拿tool_calls + 第2次带结果生成回答）
│   ├── 4.4 工具描述工程（名字动词开头/描述三要素/参数有约束/总数≤10）
│   ├── 4.5 并行工具调用（一次返回多个tool_calls/CompletableFuture并行执行）
│   ├── 4.6 结构化输出（JSON Mode/Structured Output/Schema校验/重试兜底）
│   └── 4.7 Function Calling vs Tool Calling（子集关系/走向标准化）
│
├── M5: 编排框架 [框架层]
│   ├── 5.1 LangChain
│   │   ├── 核心模块（Model/PromptTemplate/OutputParser/Chains/Agents/Memory/Retriever）
│   │   ├── LCEL 管道语法（prompt | llm | parser/Streaming+Batch+Async）
│   │   ├── AgentExecutor（create_tool_calling_agent/参数调优/错误处理）
│   │   ├── 版本演进（v0.1 LLMChain → v0.2 LCEL → v0.3 LangGraph）
│   │   └── 工程坑（版本兼容/Agent不稳定/Memory超Token/调试困难）
│   ├── 5.2 LangGraph
│   │   ├── StateGraph（有状态图/节点+边+条件边）
│   │   ├── vs LangChain（线性管道 vs 图编排/类比责任链vs工作流引擎）
│   │   └── 适用场景（条件分支/循环/并行/复杂Multi-Agent）
│   ├── 5.3 Java替代方案
│   │   ├── LangChain4j（声明式AiService/Java原生/注解驱动）
│   │   └── Spring AI（ChatClient/Function Calling/Spring生态无缝）
│   └── 5.4 选型原则（纯Java→LangChain4j/Spring AI · 复杂编排→LangGraph独立微服务）
│
├── M6: MCP 协议 [协议层]
│   ├── 6.1 核心定义（开放标准/标准化工具调用/类比USB接口）
│   ├── 6.2 架构角色（Host宿主/Client客户端/Server服务端）
│   ├── 6.3 三种能力（Tools可调用函数/Resources可读数据/Prompts模板）
│   ├── 6.4 通信方式（stdio本地/SSE远程/类比ProcessBuilder vs HTTP）
│   ├── 6.5 协议流程（初始化→能力发现→工具调用→LLM生成）
│   ├── 6.6 安全设计（最小权限/纵深防御/审计追踪/Human-in-the-loop）
│   ├── 6.7 Spring Boot 接入（@McpTool注解/薄适配层/零改造现有微服务）
│   └── 6.8 vs API vs Plugin（开放+标准化+跨平台/类比REST标准化RPC）
│
├── M7: RAG [知识层]
│   ├── 7.1 全链路流程（离线：加载→分片→向量化→存储 / 在线：检索→重排→注入Prompt→生成）
│   ├── 7.2 文档分片（固定大小/按段落/语义分片/递归分片/重叠分片）
│   ├── 7.3 Embedding（BGE-large-zh/M3E/OpenAI/向量化=语义编码）
│   ├── 7.4 向量检索（余弦相似度/ANN近似检索/MMR多样性）
│   ├── 7.5 向量数据库选型（Milvus/Pinecone/Weaviate/Qdrant/ES 8.x）
│   ├── 7.6 混合检索（向量+BM25/RRF融合排序/准确率+17%）
│   ├── 7.7 Reranker（粗筛→精排/BGE-Reranker/Cohere/准确率再+4%）
│   └── 7.8 RAG vs Fine-tuning（成本/时效/可控/私有数据/90%场景用RAG）
│
├── M8: Memory [状态层]
│   ├── 8.1 L1 工作记忆（Prompt中/最近5轮/受Token窗口限制/类比局部变量）
│   ├── 8.2 L2 短期记忆（Redis/完整会话/24h过期/类比Session）
│   ├── 8.3 L3 长期记忆（Milvus+MySQL/用户偏好+历史精华/永久/类比ES+DB）
│   ├── 8.4 Token 管理（滑动窗口+摘要压缩/类比LRU缓存淘汰）
│   └── 8.5 Memory 类型对比（Buffer/Window/Summary/Vector/适用场景）
│
├── M9: Multi-Agent [协作层]
│   ├── 9.1 协作模式（主从Supervisor/流水线Pipeline/辩论Debate/层级Hierarchical）
│   ├── 9.2 通信方式（共享状态/消息传递/上下文拼接/控制上下文大小）
│   ├── 9.3 框架选型（LangGraph底层灵活/CrewAI上层易用/AutoGen微软/MetaGPT模拟）
│   └── 9.4 类比微服务（Supervisor=网关/Worker=微服务/通信=Mq/状态=Redis）
│
├── M10: Workflow vs Agent [选型层]
│   ├── 10.1 核心区别（预定义流程 vs LLM动态决策/稳定性 vs 灵活性）
│   ├── 10.2 决策树（流程确定→Workflow/需外部信息→Chain+RAG/需工具→Agent）
│   └── 10.3 混合模式（外层Workflow管大流程/内层Agent做灵活决策）
│
├── M11: 工程优化 [生产层]
│   ├── 11.1 稳定性（死循环检测/重试+退避/输出校验/Fallback熔断/全链路Tracing）
│   ├── 11.2 成本控制（分级模型-60%/工具缓存-30%/精简Prompt-20%/语义缓存-40%）
│   ├── 11.3 幻觉防护（RAG强制引用/事实校验/置信度评估/交叉验证/人工兜底）
│   ├── 11.4 延迟优化（流式输出/并行工具调用/减少循环/预计算缓存）
│   └── 11.5 可观测性（Tracing-LangSmith/Metrics-Prometheus/Logging-ELK）
│
└── M12: 项目实战 [落地层]
    ├── 12.1 智能客服系统（Workflow+Agent混合/分级模型/MCP/RAG+Memory）
    ├── 12.6 技术亮点（6个可讲点/每个30秒讲法）
    └── 12.7 面试防崩（11项追问准备/简历写法/上线流程）
```

---

## 二、模块依赖链（谁依赖谁）

```
                ┌─────────────┐
                │  M1: LLM    │ ← 一切 AI 的地基
                │  基础        │
                └──────┬──────┘
                       │ LLM 能力
                ┌──────▼──────┐
                │  M2: Agent  │ ← 在 LLM 之上包了一层
                │  核心        │
                └──────┬──────┘
                       │
           ┌───────────┼───────────┐
           │           │           │
    ┌──────▼──────┐    │    ┌──────▼──────┐
    │  M3: 执行   │    │    │  M4: Tool   │
    │  模式       │    │    │  Calling    │
    │ (ReAct等)   │    │    │  (通信机制)  │
    └──────┬──────┘    │    └──────┬──────┘
           │           │           │
           └─────┬─────┘           │
                 │                 │
          ┌──────▼──────┐          │
          │  M5: 编排   │◀─────────┘
          │  框架       │ (LangChain 封装 Tool Calling)
          │ (LangChain) │
          └──────┬──────┘
                 │
        ┌────────┼────────┐
        │        │        │
 ┌──────▼──┐ ┌──▼──────┐ ┌▼─────────┐
 │ M6: MCP │ │ M7: RAG │ │ M8:Memory│
 │ (协议)  │ │ (知识)  │ │ (状态)   │
 └────┬────┘ └────┬────┘ └────┬─────┘
      │           │           │
      └─────┬─────┘           │
            │                 │
     ┌──────▼──────┐          │
     │ M9: Multi   │◀─────────┘
     │   Agent     │ (共享 Memory)
     └──────┬──────┘
            │
     ┌──────▼──────┐
     │ M10: 选型   │
     │ Workflow    │
     │ vs Agent    │
     └──────┬──────┘
            │
     ┌──────▼──────┐
     │ M11: 工程   │
     │ 优化        │
     │ (稳定性/    │
     │  成本/幻觉)  │
     └──────┬──────┘
            │
     ┌──────▼──────┐
     │ M12: 项目   │
     │ 实战        │
     └─────────────┘

 底层贯穿始终：
 ┌────────┐ ┌────────┐ ┌────────┐
 │  M1    │ │  M3    │ │  M11   │
 │ LLM基础│ │执行模式│ │工程优化│
 └────────┘ └────────┘ └────────┘
```

---

## 三、面试链路图（一个 Agent 请求的全链路面试考点）

```
用户："帮我查订单123456物流，如果还没到就催一下快递"
│
├── [1] Spring Boot 网关层
│   ├── 会话管理（Session/Token）
│   ├── 限流（令牌桶）
│   └── 面试考点：为什么 Agent 需要 Gateway？
│       → 限流防 Token 爆炸 / 鉴权防未授权工具调用
│
├── [2] 意图识别（Workflow - 小模型）
│   ├── 通义-turbo / Haiku（延迟<200ms，成本$0.01）
│   ├── 输出：order_query + logistics_check
│   └── 面试考点：为什么用 Workflow 不用 Agent？
│       → 分类是确定性任务 / 小模型成本低延迟快
│       → M10 选型原则：能确定流程的用 Workflow
│
├── [3] 路由到「物流 Agent」
│   ├── LangGraph StateGraph 编排
│   ├── 面试考点：LangGraph vs LangChain？
│       → LangChain 线性管道，LangGraph 支持分支/循环/状态
│       → 类比：责任链 vs Activiti 工作流引擎
│
├── [4] ReAct 循环开始
│   │
│   ├── [4.1] Thought（LLM 决策）
│   │   ├── Prompt = System指令 + 工具描述 + Memory + 用户输入
│   │   └── 面试考点：ReAct 的 Prompt 模板怎么写？
│   │       → 约束输出格式（Thought/Action/Observation）
│   │       → M3 ReAct 模式
│   │
│   ├── [4.2] Action（Tool Calling）
│   │   ├── LLM 输出 JSON：{"tool":"query_logistics","args":{"orderId":"123456"}}
│   │   ├── Agent 框架解析 JSON → 找到 MCP Server → 执行
│   │   └── 面试考点：Tool Calling 的本质？
│   │       → LLM 不执行，只输出 JSON 指令（间接调用模式）
│   │       → 一次 Tool Calling = 两次 LLM API + 一次工具执行
│   │       → M4 Tool Calling
│   │
│   ├── [4.3] Observation（工具结果）
│   │   ├── MCP Server 返回："顺丰SF1234，已到北京分拣中心"
│   │   └── 面试考点：MCP 的价值？
│   │       → 标准化工具调用 / 一次开发到处用 / 类比 USB 接口
│   │       → M6 MCP 协议
│   │
│   ├── [4.4] 第二轮 Thought
│   │   ├── LLM 判断："还没到用户手里，需要催一下"
│   │   ├── Action: notify_courier({orderId:"123456"})
│   │   └── 面试考点：如何防止 Agent 死循环？
│   │       → 最大迭代8次 + 死循环检测 + Fallback
│   │       → M11 稳定性优化
│   │
│   └── [4.5] Final Answer
│       ├── "您的订单已到北京分拣中心，已帮您催促快递..."
│       └── 面试考点：如何防止幻觉？
│           → RAG 强制引用 + 置信度评估 + 高风险转人工
│           → M11 幻觉防护
│
├── [5] Memory 更新
│   ├── L1：当前5轮对话拼入 Prompt（~2K Token）
│   ├── L2：Redis 存完整会话（24h过期）
│   ├── L3：Milvus+MySQL 存长期记忆
│   └── 面试考点：Memory 为什么分三层？
│       → Token 窗口有限 → 必须分层管理 → 滑动窗口+摘要压缩
│       → M8 Memory 系统
│
├── [6] RAG 知识检索（按需触发）
│   ├── 用户问商品详情 → 触发 RAG（不经过 Agent）
│   ├── 向量检索(Top-10) + BM25(Top-10) → RRF融合 → Reranker精排 → Top-3
│   └── 面试考点：RAG 效果不好怎么优化？
│       → 混合检索 + Reranker / 分片策略 / Embedding 模型
│       → M7 RAG
│
└── [7] 可观测性记录
    ├── LangSmith：每步 Thought/Action/Observation 全记录
    ├── Prometheus：Token消耗/工具成功率/P99延迟
    └── 面试考点：Agent 怎么调试？
        → LangSmith Tracing + verbose=True + return_intermediate_steps
        → M11 可观测性

底层贯穿：
┌────────┐ ┌────────┐ ┌────────┐
│M1 LLM  │ │M3 执行 │ │M11 优化│
│基础     │ │模式    │ │工程    │
└────────┘ └────────┘ └────────┘
```

---

## 四、12 大模块速查卡（面试前过一遍）

### M1: LLM 基础

| 维度 | 说明 |
|------|------|
| **核心作用** | Agent 的"大脑"，一切智能的来源 |
| **系统位置** | 最底层，Agent 四大组件中 Planner 的实现 |
| **面试必考** | LLM vs Agent 区别（自主性）/ LLM 的工程限制（无状态/幻觉/Token） |
| **Java 类比** | LLM = 纯文本 HTTP API |
| **一句话** | LLM 是概率预测引擎，只能"说话"不能"做事" |

### M2: Agent 核心

| 维度 | 说明 |
|------|------|
| **核心作用** | 把 LLM 升级为能自主决策和执行的完整系统 |
| **系统位置** | LLM 之上的抽象层，封装了感知+决策+执行+记忆 |
| **面试必考** | 四大组件 / 自主性等级 L0~L4 / Java 类比 |
| **Java 类比** | Planner=DispatcherServlet / Tools=@Service / Memory=Redis+MySQL / Executor=反射 |
| **一句话** | Agent = LLM + Tools + Memory + Planner，用 LLM 替代 if-else 路由 |

### M3: 执行模式

| 维度 | 说明 |
|------|------|
| **核心作用** | 定义 Agent "怎么循环执行"的策略 |
| **系统位置** | Agent 核心的执行引擎 |
| **面试必考** | ReAct 流程（Thought→Action→Observation）/ Plan-Execute vs ReAct / 死循环处理 |
| **Java 类比** | ReAct=while循环 / Plan-Execute=瀑布开发 / Reflexion=CI/CD自动修复 |
| **一句话** | ReAct 是核心（推理+行动循环），复杂任务用 Plan-Execute，工程上混合使用 |

### M4: Tool Calling

| 维度 | 说明 |
|------|------|
| **核心作用** | 让 LLM 能调用外部工具（间接调用模式） |
| **系统位置** | Agent 和外部系统之间的桥梁 |
| **面试必考** | LLM 只输出 JSON 不执行 / 两次 API 调用 / 安全性设计 / 工具描述 5 条规则 |
| **Java 类比** | Tool Calling = HTTP RPC / ToolRegistry = ApplicationContext |
| **一句话** | LLM 输出 JSON 指令，框架代为执行，间接调用模式是为了安全性 |

### M5: 编排框架

| 维度 | 说明 |
|------|------|
| **核心作用** | 把 LLM+Tool+Memory 串起来的"胶水" |
| **系统位置** | Agent 系统的骨架，管理组件间的数据流 |
| **面试必考** | LangChain 核心模块 / LCEL 管道语法 / LangGraph vs LangChain / Java替代方案 |
| **Java 类比** | LangChain=Spring Boot / LCEL=Stream管道 / LangGraph=Activiti |
| **一句话** | LangChain 是 AI 领域的 Spring Boot，LangGraph 是工作流引擎，Java 用 LangChain4j/Spring AI |

### M6: MCP 协议

| 维度 | 说明 |
|------|------|
| **核心作用** | 标准化 Agent 与工具之间的连接协议 |
| **系统位置** | 编排层和工具层之间的协议层 |
| **面试必考** | MCP 是什么 / vs API vs Plugin / 核心价值 / Spring Boot 接入方式 |
| **Java 类比** | MCP = REST+OpenAPI 标准化 / MCP Server = @RestController |
| **一句话** | MCP 是 Agent 世界的 USB 接口，一次开发到处用 |

### M7: RAG

| 维度 | 说明 |
|------|------|
| **核心作用** | 给 LLM 补充私有知识和最新信息 |
| **系统位置** | Agent 的知识来源层，替代 LLM 的训练数据 |
| **面试必考** | 全链路流程 / 分片策略 / 向量库选型 / 混合检索+Reranker / vs Fine-tuning |
| **Java 类比** | RAG = ES 查询 + 注入上下文 / 向量库 = 语义版 ES |
| **一句话** | 先检索再生成，混合检索+Reranker 是提效关键 |

### M8: Memory

| 维度 | 说明 |
|------|------|
| **核心作用** | 让 Agent 维护状态，理解上下文 |
| **系统位置** | 贯穿 Agent 全生命周期，所有组件共享 |
| **面试必考** | 三层架构（L1/L2/L3）/ Token 管理 / vs ChatGPT 对话历史 |
| **Java 类比** | L1=局部变量 / L2=Redis Session / L3=MySQL+ES |
| **一句话** | 三层记忆（Prompt/Redis/向量库），核心挑战是 Token 管理 |

### M9: Multi-Agent

| 维度 | 说明 |
|------|------|
| **核心作用** | 通过分治处理超复杂任务 |
| **系统位置** | 多个 Agent 组成的协作网络 |
| **面试必考** | 四种协作模式 / 通信方式 / 上下文大小控制 / 框架选型 |
| **Java 类比** | Multi-Agent = 微服务集群 / Supervisor = API 网关 |
| **一句话** | 分治思想在 AI 的应用，Supervisor+Worker 模式最常用 |

### M10: Workflow vs Agent

| 维度 | 说明 |
|------|------|
| **核心作用** | 选型决策——什么时候用什么 |
| **系统位置** | 架构设计的前置决策 |
| **面试必考** | 核心区别 / 决策树 / 混合模式 |
| **Java 类比** | Workflow = Spring Batch / Agent = 自适应路由 |
| **一句话** | 确定流程用 Workflow，灵活决策用 Agent，工程上混合使用 |

### M11: 工程优化

| 维度 | 说明 |
|------|------|
| **核心作用** | 把 Agent 从"玩具"变成"生产级" |
| **系统位置** | 贯穿所有层，生产化的关键 |
| **面试必考** | 稳定性方案 / 成本优化 / 幻觉防护 / 可观测性 |
| **Java 类比** | 熔断+降级+限流+重试+APM，思路完全一样 |
| **一句话** | 和 Java 后端的稳定性保障思路完全一致——加护栏、有兜底、可观测 |

### M12: 项目实战

| 维度 | 说明 |
|------|------|
| **核心作用** | 面试中能讲的完整项目 |
| **系统位置** | 所有模块的整合落地 |
| **面试必考** | 架构图 / 技术亮点 / 挑战和解决 / 简历写法 |
| **项目模板** | 智能客服 Agent（Workflow+Agent+MCP+RAG+三层Memory） |
| **一句话** | Workflow+Agent 混合 + MCP 标准化 + RAG 混合检索 + 分级模型省 60% 成本 |

---

## 五、面试优先级（先学什么、后学什么）

```
优先级 P0 —— 面试 100% 会问（必须掌握）
├── Agent 四大组件 + Java 类比
├── ReAct 执行流程（能画出循环图）
├── Tool Calling 原理（LLM 只输出 JSON）
├── RAG 全链路（分片→向量化→检索→注入）
└── LLM vs Agent 区别（自主性）

优先级 P1 —— 面试 80% 会问（拉开差距）
├── MCP 协议（2026 重点）
├── LangChain 核心模块 + LCEL
├── Memory 三层架构 + Token 管理
├── Workflow vs Agent 选型
└── 项目讲法（30秒开场 + 2分钟深入）

优先级 P2 —— 面试 50% 会问（冲击中大厂）
├── Multi-Agent 协作模式
├── LangGraph vs LangChain
├── Plan-Execute vs ReAct
├── 稳定性/成本/幻觉 优化方案
└── 混合检索 + Reranker

优先级 P3 —— 面试 20% 会问（加分项）
├── Reflexion 模式
├── LangChain4j / Spring AI
├── 可观测性三件套
├── Agent 自主性等级 L0~L4
└── 安全设计（提示注入防护）
```

---

## 六、Java 后端 → Agent 工程师 知识映射

```
你已会的 Java 知识        →    对应 Agent 知识
─────────────────────────────────────────────────
Spring IOC 容器           →    ToolRegistry（工具注册中心）
DispatcherServlet 路由    →    Planner（LLM 做动态路由）
@Service 业务方法         →    Tools（Agent 可调用的能力）
Redis Session             →    L2 短期记忆
MySQL + ES                →    L3 长期记忆
反射调用 / RPC            →    Executor（执行具体方法）
责任链模式                →    LangChain Chain
Activiti 工作流            →    LangGraph StateGraph
REST + OpenAPI 标准化      →    MCP 协议
Feign 远程调用             →    MCP Client
熔断/降级/限流/重试        →    Agent 稳定性保障
Prometheus + Grafana      →    LangSmith + Prometheus
微服务 + 网关              →    Multi-Agent + Supervisor
Spring Batch Job          →    Workflow（固定流程）
ES 搜索 + 索引            →    RAG（向量检索 + 文档索引）
LRU 缓存淘汰              →    Memory Token 管理
```

---

## 七、7 Stage 文件索引

| 文件 | 对应模块 | 面试定位 |
|------|---------|---------|
| [Stage1-Agent基础认知.md](1-Agent基础认知.md) | M1+M2 | 必考：LLM vs Agent / 四组件 / Java类比 |
| [Stage2-Agent架构拆解.md](2-Agent架构拆解.md) | M3+M4 | 必考：ReAct / Tool Calling / 接口设计 |
| [Stage3-LangChain全链路.md](3-LangChain全链路.md) | M5 | 高频：模块 / LCEL / LangGraph / 坑 |
| [Stage4-MCP协议.md](4-MCP协议.md) | M6 | 2026重点：架构 / 安全 / 接入 |
| [Stage5-RAG与Memory.md](5-RAG与Memory.md) | M7+M8 | 必考：RAG全链路 / 向量库 / Memory三层 |
| [Stage6-Agent进阶.md](6-Agent进阶.md) | M9+M10+M11 | 拉差距：Multi-Agent / 优化 / 可观测 |
| [Stage7-项目表达.md](7-项目表达.md) | M12 | 最关键：项目讲法 / 简历写法 / 防崩清单 |

---

> **知识图谱文件结束。配合 7 个 Stage 详细文档使用。**
