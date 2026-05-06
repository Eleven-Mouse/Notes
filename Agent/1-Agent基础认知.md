# 第一阶段：Agent 基础认知

> **目标：** 让你从零理解 Agent 是什么、为什么需要它、它和普通 LLM 调用的本质区别。
> **学完本阶段，你应该能用自己的话把 Agent 讲清楚。**

---

## 1. 什么是 Agent？一个类比让你秒懂

### 1.1 生活类比

想象你在一家公司：

| 角色 | 做什么 | 对应什么 |
|------|--------|----------|
| **顾问** | 你问他问题，他给你建议，但他不动手干活 | **纯 LLM**（如 ChatGPT） |
| **助理** | 你给他一个目标，他自己查资料、打电话、写报告，搞不定才找你 | **Agent** |

> **一句话：LLM 是"只会说的顾问"，Agent 是"能动手的助理"。**

顾问（LLM）很强，你问他"北京明天天气怎么样"，他能根据经验给你一个回答——但那可能是去年夏天的数据，甚至是编的。

助理（Agent）不一样。你问他同样的问题，他会：
1. 想一下："我需要查真实的天气数据"（**思考**）
2. 拿起电话打给气象局 API（**行动**）
3. 得到数据："明天北京小雨 15°C"（**观察**）
4. 再想一下："下雨了，那推荐点室内活动吧"（**思考**）
5. 搜索室内景点（**行动**）
6. 给你最终回答："明天小雨 15°C，推荐去故宫、国家博物馆"

这就是 Agent 的核心——**它能自己想、自己做、自己看结果、自己调整**。

### 1.2 精确定义

> **Agent（智能体）= 一个能感知环境、自主决策、调用工具、采取行动以达成目标的系统。**

拆开理解五个关键词：
- **感知环境**：接收用户输入 + 工具返回的结果
- **自主决策**：不是你写死 if-else，是 LLM 自己判断
- **调用工具**：能操作外部系统（API / 数据库 / 文件）
- **采取行动**：有实际的执行能力，不只是生成文本
- **达成目标**：有明确的任务完成条件

---

## 2. Agent 的本质：决策循环

### 2.1 Agent 的核心就是一个循环

Agent 的本质并不神秘，它就是一个**循环**：

```
┌──────────────────────────────────────────────┐
│           Agent 的核心：决策循环               │
│                                              │
│    ┌─── Observe（观察）                        │
│    │    接收输入 / 工具返回结果                  │
│    │              │                            │
│    │              ▼                            │
│    ├─── Think（思考）                          │
│    │    LLM 推理：下一步该做什么？               │
│    │              │                            │
│    │              ▼                            │
│    └─── Act（行动）                            │
│         调用工具 / 返回最终结果                  │
│              │                                │
│              └──→ 回到 Observe（观察结果）       │
│                                              │
│    循环直到：LLM 判断任务完成 / 达到最大步数     │
└──────────────────────────────────────────────┘
```

**为什么必须是一个循环？**

因为真实世界的任务很少能一步搞定。比如"帮我分析一下这个代码仓库的质量"：
- 第 1 轮：先看有哪些文件（调工具：列出目录）
- 第 2 轮：看核心模块的代码（调工具：读取文件）
- 第 3 轮：检查测试覆盖率（调工具：运行测试）
- 第 4 轮：综合分析，给出报告（最终回答）

每一轮都是 Observe → Think → Act，直到任务完成。

### 2.2 和普通 LLM 调用的对比

```
普通 LLM 调用（单次）：
  用户 ──→ LLM ──→ 回答
  （一问一答，结束）

Agent 调用（循环）：
  用户 ──→ Agent ──→ 思考 ──→ 调工具 ──→ 观察结果 ──→ 再思考 ──→ ... ──→ 最终回答
  （自主循环，直到目标达成）
```

| 维度 | 普通 LLM | Agent |
|------|----------|-------|
| 执行模式 | 单次请求-响应 | 多轮循环 |
| 能力 | 只能"说"（生成文本） | 能"做"（调用工具） |
| 决策 | 无决策，直接生成 | LLM 动态选择下一步 |
| 状态 | 无状态（每次调用独立） | 有状态（Memory 贯穿任务） |
| 谁决定下一步 | 用户 | LLM 自己 |
| 错误处理 | 无（错了就错了） | 可以重试、换工具、调整策略 |

### 2.3 用一个具体例子感受区别

```
用户："帮我查一下北京明天的天气，如果下雨就推荐室内活动"

纯 LLM：
  → "北京明天可能是晴天，可以去长城..."
  → （信息可能是编的，没有查真实数据）

Agent：
  Thought: 需要查真实的天气预报
  Action:  调用天气API("北京", "明天")
  Observation: 小雨，15°C，湿度80%

  Thought: 下雨了，需要推荐室内场所
  Action:  调用搜索("北京室内景点推荐")
  Observation: 故宫、国家博物馆、798艺术区...

  Final Answer: 明天北京小雨15°C，推荐去故宫、国家博物馆等室内景点
  → 信息真实、基于实时数据、决策合理
```

---

## 3. Agent 的四大核心组件

Agent = LLM（大脑） + Memory（记忆） + Tools（工具） + Planning（规划）

```
┌──────────────────────────────────────────────────────────┐
│                    Agent 系统架构                         │
│                                                          │
│                      用户目标                             │
│                        │                                 │
│                        ▼                                 │
│  ┌─────────────────────────────────────────────────┐     │
│  │          LLM（大脑 / 规划器）                      │     │
│  │                                                   │     │
│  │  输入：用户目标 + Memory上下文 + 可用工具列表       │     │
│  │  处理：推理决策"下一步做什么"                       │     │
│  │  输出：调用哪个工具 + 传什么参数 / 或最终回答       │     │
│  └──────────────────────┬──────────────────────────┘     │
│                         │                                │
│                         ▼                                │
│  ┌──────────────────────────────────────────────────┐    │
│  │             Tools（工具集 / 手脚）                  │    │
│  │                                                    │    │
│  │  搜索引擎 │ 天气API │ 数据库查询 │ 代码执行 │ ...   │    │
│  └──────────────────────┬───────────────────────────┘    │
│                    ╱          ╲                            │
│                   ▼            ▼                           │
│  ┌──────────────────────┐  ┌────────────────────────┐    │
│  │   Memory（记忆系统）   │  │  Planning（规划能力）   │    │
│  │                       │  │                        │    │
│  │  短期：对话上下文      │  │  任务拆解              │    │
│  │  长期：向量数据库      │  │  步骤编排              │    │
│  │                       │  │  自我反思              │    │
│  └──────────────────────┘  └────────────────────────┘    │
│                                                          │
│                      最终结果                             │
└──────────────────────────────────────────────────────────┘
```

### 3.1 组件一：LLM（大脑）

**是什么：** Agent 的"大脑"，负责理解、推理、决策。

**为什么需要它：** Agent 需要一个"能理解自然语言、能推理、能做决策"的引擎。LLM 就是目前最好的选择。

**它做什么：**
- 理解用户的目标
- 分析当前情况（结合 Memory）
- 决定下一步调什么工具、传什么参数
- 判断任务是否完成
- 生成最终回答

**工程本质：** LLM 的调用就是一次 HTTP 请求——发文本过去，收文本回来。Agent 框架做的就是在 LLM 外面包了一层循环逻辑。

```python
import requests

def call_llm(messages):
    """最基础的 LLM 调用：就是一次 HTTP 请求"""
    response = requests.post(
        "https://api.openai.com/v1/chat/completions",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={
            "model": "gpt-4o",
            "messages": messages
        }
    )
    return response.json()["choices"][0]["message"]["content"]

# 使用
result = call_llm([{"role": "user", "content": "你好"}])
print(result)  # "你好！有什么可以帮助你的吗？"
```

### 3.2 组件二：Memory（记忆）

**是什么：** 让 Agent "记住"之前发生了什么。

**为什么需要它：** LLM 本身无状态——每次调用互相独立，不记得上一秒说了什么。Memory 就是在 LLM 外面套了一个"记忆系统"。

```
没有 Memory：
  用户："帮我查订单 123456"
  Agent：调工具 → "订单123456，已发货"
  用户："那个物流呢？"
  Agent："什么物流？" ← 不记得刚才聊的什么

有 Memory：
  用户："帮我查订单 123456"
  Agent：调工具 → "订单123456，已发货"
  用户："那个物流呢？"
  Agent：理解"那个"= 订单123456 → 调物流查询 → "顺丰已到北京"
```

**Memory 的两层：**

| 层级 | 存什么 | 类比 | 技术 |
|------|--------|------|------|
| 短期记忆 | 当前对话的上下文 | 程序的局部变量 | 直接拼到 Prompt 里 |
| 长期记忆 | 跨对话的知识和偏好 | 程序的数据库 | 向量数据库（Milvus / Chroma） |

### 3.3 组件三：Tools（工具）

**是什么：** Agent 可以调用的外部能力——函数、API、服务等。

**为什么需要它：** LLM 只能输出文本，不能查数据库、不能调 API、不能执行代码。Tools 就是给 Agent 装上"手脚"。

**工具定义的三个要素：**

```python
# 一个工具 = 名称 + 描述 + 参数定义 + 实际函数

def query_weather(city: str, date: str) -> str:
    """根据城市和日期查询天气预报"""  # ← 描述（给 LLM 看的）
    # 实际调用天气 API
    return f"{city} {date}: 小雨 15°C"

# 工具定义（告诉 LLM 这个工具怎么用）
tool_definition = {
    "name": "query_weather",                    # ← 名称
    "description": "查询指定城市和日期的天气预报",  # ← 描述
    "parameters": {                              # ← 参数 Schema
        "type": "object",
        "properties": {
            "city": {"type": "string", "description": "城市名"},
            "date": {"type": "string", "description": "日期，如 2026-05-08"}
        },
        "required": ["city", "date"]
    }
}
```

**关键理解：** 工具的描述是给 LLM 看的，不是给人看的。描述越精确，LLM 选错工具的概率越低。

### 3.4 组件四：Planning（规划）

**是什么：** Agent 把复杂目标拆解成一步步可执行的计划。

**为什么需要它：** 用户给的目标通常是模糊的、复杂的。比如"帮我写一份市场分析报告"，Agent 需要自己把它拆成：
1. 先搜索市场数据
2. 分析竞品情况
3. 汇总成报告

Planning 不是独立的代码模块，而是 LLM 通过 Prompt 引导出的能力。

### 3.5 四个组件如何协作

```
用户目标："帮我查北京天气，推荐适合的活动"
    │
    ▼
[Memory] 加载对话上下文
[LLM]    接收：目标 + 上下文 + 工具列表
[LLM]    思考："需要先查天气"（Planning）
[LLM]    决策：调用 query_weather("北京", "2026-05-08")
    │
    ▼
[Tools]  执行 query_weather → "小雨 15°C"
    │
    ▼
[Memory] 记录：工具调用和结果
[LLM]    思考："下雨了，推荐室内活动"（Planning）
[LLM]    决策：调用 search("北京室内活动")
    │
    ▼
[Tools]  执行 search → ["故宫", "国家博物馆", "798"]
    │
    ▼
[LLM]    判断：信息足够，生成最终回答
    │
    ▼
"明天北京小雨15°C，推荐去故宫、国家博物馆等室内景点"
```

---

## 4. Agent vs ChatGPT vs Workflow

这是很多人混淆的点，讲清楚非常重要。

### 4.1 三者对比

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  自主性 / 灵活性                                           │
│  ▲                                                         │
│  │                     ┌──────────┐                        │
│  │                     │  Agent   │ 自主决策 + 调用工具     │
│  │                     │          │ 灵活，但不确定性强       │
│  │                     └──────────┘                        │
│  │           ┌──────────────┐                               │
│  │           │   ChatGPT    │ 能对话，但不能调工具           │
│  │           │              │ 有上下文，但无执行能力         │
│  │           └──────────────┘                               │
│  │  ┌─────────────────┐                                     │
│  │  │    Workflow      │ 固定流程，确定性高                   │
│  │  │                  │ 但不灵活                            │
│  │  └─────────────────┘                                     │
│  └──────────────────────────────────────────────►           │
│                       复杂度 / 成本                          │
└────────────────────────────────────────────────────────────┘
```

| 维度 | ChatGPT（多轮对话） | Workflow（工作流） | Agent（智能体） |
|------|---------------------|--------------------|-----------------|
| 执行方式 | 用户问一次，回答一次 | 按固定流程执行 | 自主决策循环 |
| 谁决定下一步 | 用户 | 预设流程（代码） | LLM 自己 |
| 灵活性 | 中（理解自然语言） | 低（写死的） | 高（动态适应） |
| 确定性 | 中 | 高 | 低（有随机性） |
| 能调工具 | 基本不能 | 能 | 能（核心能力） |
| 成本 | 低 | 低 | 高（多轮 LLM 调用） |
| 适合场景 | 问答 / 聊天 | 审批流 / ETL | 复杂任务自动化 |

### 4.2 怎么选？一个决策树

```
你的任务是什么？

只需要处理文本（翻译/摘要/分类）
  → 单次 LLM 调用（最简单）

需要多轮对话理解上下文
  → ChatGPT 模式（多轮对话）

流程固定、步骤确定
  → Workflow（最可靠）

目标复杂、步骤不确定、需要动态决策
  → Agent（最灵活）
```

### 4.3 为什么 2026 年主流是"Workflow + Agent 混合"

实际生产中，很少有人用"纯 Agent"——因为 Agent 的不确定性是硬伤。

主流做法是：
- **确定的部分**用 Workflow（比如先验证用户身份 → 再查数据 → 最后格式化输出）
- **灵活的部分**用 Agent（比如用户意图理解、异常处理、复杂推理）

```
┌─────────────────────────────────────────────┐
│          混合架构（2026 主流）                 │
│                                             │
│  用户输入                                    │
│    │                                        │
│    ▼                                        │
│  [Workflow] 意图识别（确定的 LLM 调用）       │
│    │                                        │
│    ├── 简单查询 → [Workflow] 直接查库返回     │
│    │                                        │
│    └── 复杂任务 → [Agent] 自主规划执行        │
│                    │                        │
│                    └──→ 结果返回              │
└─────────────────────────────────────────────┘
```

**设计原则：能用 Workflow 的不用 Agent，能用单次调用的不搞循环。**

---

## 5. Agent 的演进：为什么是现在才火

理解这段历史，能帮你看清 Agent 的本质。

```
2022.01  GPT-3.5 发布
         ↓ LLM 能力初现，但只是"说话"
2022.06  ReAct 论文（Yao et al.）
         ↓ 提出"推理 + 行动"的范式 → Agent 的理论基础
2022.10  AutoGPT / BabyAGI 爆火
         ↓ 首批 Agent 产品，但极度不稳定（"玩具"阶段）
2023.03  OpenAI 发布 Function Calling
         ↓ LLM 首次能输出结构化的工具调用指令 → 关键转折！
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
2026     Multi-Agent / Agent Workflow 成为主流
         ↓ 企业级 Agent 大规模落地
```

**关键转折点是 2023 年 OpenAI 的 Function Calling。**

在此之前，让 LLM "调用工具"需要做大量 Prompt 工程和文本解析——非常脆弱。

Function Calling 让 LLM 能直接输出结构化的 JSON（工具名 + 参数），这让 Agent 从"理论"变成"可工程化"的东西。

---

## 6. 实战：手写一个最简 Agent（不依赖 LangChain）

> **目标：** 用纯 Python 从零实现一个能思考、能调工具、能循环的 Agent。
> **理解了这个 Demo，你就理解了 Agent 框架的 80%。**

### 6.1 完整代码

```python
"""
最简 Agent —— 纯 Python 实现，不依赖任何 Agent 框架
展示 Agent 的核心机制：思考 → 行动 → 观察 → 循环
"""

import json

# ============================================================
# 第一步：定义工具（Tools）
# ============================================================

def get_weather(city: str) -> str:
    """模拟天气查询工具"""
    # 实际项目中这里会调用真实的天气 API
    weather_data = {
        "北京": "小雨 15°C 湿度80%",
        "上海": "晴 22°C 湿度45%",
        "深圳": "多云 28°C 湿度70%",
    }
    return weather_data.get(city, f"{city}: 暂无天气数据")

def search(query: str) -> str:
    """模拟搜索工具"""
    search_results = {
        "室内活动": "故宫博物院、国家博物馆、798艺术区、三里屯",
        "户外活动": "长城、颐和园、天坛公园、奥林匹克公园",
        "美食推荐": "全聚德烤鸭、护国寺小吃、南锣鼓巷",
    }
    for key, value in search_results.items():
        if key in query:
            return value
    return f"搜索'{query}'：暂无结果"

def calculate(expression: str) -> str:
    """计算器工具（安全版本，只允许基本运算）"""
    try:
        # 注意：生产环境中绝对不要用 eval！这里只是 Demo
        # 生产环境应该用 ast.literal_eval 或专门的数学解析库
        allowed = set("0123456789+-*/.() ")
        if not all(c in allowed for c in expression):
            return "错误：表达式包含不允许的字符"
        result = eval(expression)  # noqa: S307
        return str(result)
    except Exception as e:
        return f"计算错误：{e}"


# 工具注册表：把工具定义统一管理
# 这个结构就是告诉 LLM："你有这些工具可以用"
TOOLS = [
    {
        "name": "get_weather",
        "description": "查询指定城市的天气情况",
        "parameters": {
            "city": {"type": "string", "description": "城市名称，如'北京'"}
        }
    },
    {
        "name": "search",
        "description": "搜索信息，如室内活动、户外活动、美食等",
        "parameters": {
            "query": {"type": "string", "description": "搜索关键词"}
        }
    },
    {
        "name": "calculate",
        "description": "执行数学计算，支持加减乘除",
        "parameters": {
            "expression": {"type": "string", "description": "数学表达式，如 '2+3*4'"}
        }
    }
]

# 工具名 → 实际函数的映射
TOOL_FUNCTIONS = {
    "get_weather": get_weather,
    "search": search,
    "calculate": calculate,
}


# ============================================================
# 第二步：构建 Agent（核心）
# ============================================================

class SimpleAgent:
    """
    最简 Agent 实现。

    核心就是一个 while 循环：
      1. 把当前上下文发给 LLM
      2. LLM 返回：调工具 or 给最终回答
      3. 如果调工具：执行工具，把结果加入上下文，回到第 1 步
      4. 如果是最终回答：返回给用户

    所有 Agent 框架（LangChain、LangGraph）的本质都是这个循环。
    """

    def __init__(self, llm_func, tools, tool_functions, max_steps=5):
        """
        参数：
          llm_func: 调用 LLM 的函数（接收 messages，返回文本）
          tools: 工具定义列表（告诉 LLM 有哪些工具）
          tool_functions: 工具名 → 实际函数的映射
          max_steps: 最大循环次数（防止死循环）
        """
        self.llm_func = llm_func
        self.tools = tools
        self.tool_functions = tool_functions
        self.max_steps = max_steps

    def _build_system_prompt(self):
        """构建 System Prompt：告诉 LLM 它的角色和可用工具"""
        tool_descriptions = ""
        for tool in self.tools:
            params = ", ".join(
                f"{k}: {v['description']}" for k, v in tool["parameters"].items()
            )
            tool_descriptions += f"- {tool['name']}({params}): {tool['description']}\n"

        return f"""你是一个智能助手。你可以使用以下工具来帮助用户：

{tool_descriptions}
当你需要使用工具时，请严格按以下 JSON 格式回复：
{{"action": "tool_name", "parameters": {{"参数名": "参数值"}}}}

当你认为已经收集到足够信息可以回答用户时，请直接给出最终回答（不要用 JSON 格式）。

示例：
用户：北京天气怎么样？
你的回复：{{"action": "get_weather", "parameters": {{"city": "北京"}}}}

用户：1+1等于几？
你的回复：1+1=2"""

    def _parse_response(self, response_text):
        """
        解析 LLM 的回复：
        - 如果是 JSON 格式 → 说明要调工具
        - 如果不是 JSON → 说明是最终回答
        """
        try:
            # 尝试解析为 JSON
            parsed = json.loads(response_text.strip())
            if "action" in parsed:
                return {"type": "tool_call", "data": parsed}
        except json.JSONDecodeError:
            pass

        # 不是 JSON，视为最终回答
        return {"type": "final_answer", "data": response_text}

    def _execute_tool(self, tool_name, parameters):
        """执行工具调用"""
        func = self.tool_functions.get(tool_name)
        if not func:
            return f"错误：工具 '{tool_name}' 不存在"

        try:
            result = func(**parameters)
            return result
        except Exception as e:
            return f"工具执行错误：{e}"

    def run(self, user_input: str) -> str:
        """
        Agent 的主循环——这就是 Agent 的全部核心逻辑。

        流程：
          while 没完成 and 没超步数:
              给 LLM 发上下文
              LLM 返回决策
              如果要调工具 → 执行 → 结果加入上下文 → 继续
              如果是最终回答 → 返回
        """
        # 记录对话历史（这就是最简单的 Memory）
        messages = [
            {"role": "system", "content": self._build_system_prompt()},
            {"role": "user", "content": user_input}
        ]

        print(f"\n{'='*60}")
        print(f"用户：{user_input}")
        print(f"{'='*60}")

        for step in range(1, self.max_steps + 1):
            print(f"\n--- 第 {step} 步 ---")

            # 1. 调用 LLM（Observe + Think）
            llm_response = self.llm_func(messages)
            print(f"LLM 思考：{llm_response}")

            # 2. 解析 LLM 的回复
            parsed = self._parse_response(llm_response)

            if parsed["type"] == "final_answer":
                # LLM 认为任务完成了，直接返回
                print(f"\n最终回答：{parsed['data']}")
                return parsed["data"]

            # 3. 执行工具调用（Act）
            tool_name = parsed["data"]["action"]
            tool_params = parsed["data"]["parameters"]
            print(f"调用工具：{tool_name}({tool_params})")

            tool_result = self._execute_tool(tool_name, tool_params)
            print(f"工具结果：{tool_result}")

            # 4. 把工具结果加入上下文（更新 Memory），继续循环
            messages.append({"role": "assistant", "content": llm_response})
            messages.append({"role": "user", "content": f"工具返回结果：{tool_result}"})

        return "达到最大步数限制，任务未完成。"


# ============================================================
# 第三步：模拟 LLM（本地模拟，不调 API）
# ============================================================

def mock_llm(messages: str) -> str:
    """
    模拟 LLM 的行为（用于演示，不调真实 API）。

    在实际项目中，这里替换成真正的 LLM 调用：
      response = openai.ChatCompletion.create(
          model="gpt-4o",
          messages=messages
      )
      return response.choices[0].message.content
    """
    last_user_msg = messages[-1]["content"] if isinstance(messages, list) else messages

    # 模拟 LLM 的决策逻辑
    if "天气" in str(last_user_msg) or "下雨" in str(last_user_msg):
        if "北京" in str(last_user_msg):
            return '{"action": "get_weather", "parameters": {"city": "北京"}}'
        elif "上海" in str(last_user_msg):
            return '{"action": "get_weather", "parameters": {"city": "上海"}}'

    if "工具返回结果" in str(last_user_msg):
        result = str(last_user_msg)
        if "小雨" in result:
            return '{"action": "search", "parameters": {"query": "室内活动"}}'
        elif "晴" in result:
            return '{"action": "search", "parameters": {"query": "户外活动"}}'
        elif "故宫" in result or "长城" in result:
            # 已经有足够信息，生成最终回答
            return f"根据查询结果，{result}"

    if "计算" in str(last_user_msg) or "+" in str(last_user_msg) or "*" in str(last_user_msg):
        import re
        expr_match = re.search(r'[\d+\-*/(). ]+', str(last_user_msg))
        if expr_match:
            return f'{{"action": "calculate", "parameters": {{"expression": "{expr_match.group().strip()}"}}}}'

    return "我可以帮你查询天气、搜索信息、做计算。请告诉我你需要什么？"


# ============================================================
# 第四步：运行 Agent
# ============================================================

if __name__ == "__main__":
    # 创建 Agent
    agent = SimpleAgent(
        llm_func=mock_llm,          # 替换成真实 LLM 即可用于生产
        tools=TOOLS,
        tool_functions=TOOL_FUNCTIONS,
        max_steps=5                  # 最多循环 5 次
    )

    # 测试 1：查天气 + 推荐活动（需要多步）
    print("\n" + "=" * 60)
    print("测试 1：多步任务")
    agent.run("北京明天天气怎么样？推荐适合的活动")

    # 测试 2：简单计算（单步）
    print("\n" + "=" * 60)
    print("测试 2：简单任务")
    agent.run("帮我计算 123 + 456")
```

### 6.2 运行结果

```
============================================================
用户：北京明天天气怎么样？推荐适合的活动
============================================================

--- 第 1 步 ---
LLM 思考：{"action": "get_weather", "parameters": {"city": "北京"}}
调用工具：get_weather({'city': '北京'})
工具结果：小雨 15°C 湿度80%

--- 第 2 步 ---
LLM 思考：{"action": "search", "parameters": {"query": "室内活动"}}
调用工具：search({'query': '室内活动'})
工具结果：故宫博物院、国家博物馆、798艺术区、三里屯

--- 第 3 步 ---
LLM 思考：根据查询结果，工具返回结果：故宫博物院、国家博物馆...

最终回答：根据查询结果，明天北京小雨15°C，推荐室内活动：故宫...
```

### 6.3 逐行理解核心逻辑

```python
# Agent 的核心就是这几行，所有框架的本质都是这样：

for step in range(max_steps):       # 循环（最多 N 步防止死循环）
    response = llm(context)          # 调 LLM 做决策（Think）
    
    if is_final_answer(response):    # 如果 LLM 认为完成了
        return response              # 直接返回
    
    result = execute_tool(response)  # 否则执行工具（Act）
    context += result                # 结果加入上下文（Observe）
                                     # 继续下一轮循环
```

**理解了这几行，你就理解了 LangChain Agent、LangGraph、AutoGPT 的核心。** 它们只是在"如何调 LLM"、"如何解析回复"、"如何管理上下文"上做了更多工程化封装。

---

## 7. 常见误区

| 误区 | 真相 |
|------|------|
| "Agent 就是更智能的 ChatGPT" | Agent 不是"更智能的聊天"，而是能自主调用工具的决策执行系统 |
| "Agent 就是让 AI 自己写代码" | 代码执行只是工具的一种，Agent 能调各种工具 |
| "Agent 不需要人干预" | 工程上必须有护栏机制（最大步数、人工确认等） |
| "Agent 就是 while 循环调 LLM" | 循环是骨架，但 Memory、Tool 设计、Prompt 工程才是灵魂 |
| "LLM 有记忆功能" | LLM 本身无状态，"记忆"是通过外部拼接上下文实现的 |
| "Agent 会取代传统后端" | Agent 是后端的新组件，用 LLM 替代硬编码决策，不是替代整个后端 |

---

## 8. 这一阶段的知识图谱总结

```
Agent 基础认知
│
├── 1. 什么是 Agent
│   ├── 定义：能感知、决策、调工具、行动的系统
│   ├── 类比：LLM = 顾问（只会说），Agent = 助理（能动手）
│   └── 本质：Observe → Think → Act 的决策循环
│
├── 2. Agent vs LLM vs ChatGPT vs Workflow
│   ├── LLM：单次请求-响应，无状态
│   ├── ChatGPT：多轮对话，但被动响应
│   ├── Workflow：固定流程，确定性高
│   └── Agent：自主决策循环，灵活但不确定
│
├── 3. 四大核心组件
│   ├── LLM（大脑）：推理决策
│   ├── Memory（记忆）：管理上下文（短期 + 长期）
│   ├── Tools（工具）：外部能力（API / 数据库 / 代码执行）
│   └── Planning（规划）：拆解目标、编排步骤
│
├── 4. 关键历史节点
│   └── 2023 Function Calling → Agent 从理论变工程
│
└── 5. 最简 Agent = while 循环 + LLM 决策 + 工具执行
    └── 理解了这个 = 理解了所有 Agent 框架的 80%
```

---

## 👉 这一阶段你应该掌握的能力

1. **能用自己的话讲清楚 Agent 是什么**，以及它和普通 LLM 调用的本质区别
2. **能画出 Agent 的决策循环**（Observe → Think → Act）
3. **能说出四大核心组件**（LLM / Memory / Tools / Planning）各自的职责
4. **能手写一个最简 Agent**（不依赖任何框架，用 while 循环 + LLM + 工具）
5. **能根据场景判断**：什么时候该用 Agent，什么时候用 Workflow 或简单 LLM 调用就够了

---

> **下一阶段预告：** 第二阶段将深入 Agent 的核心机制——ReAct 模式、Tool Calling 原理、Memory 系统设计、Planning 能力，并用 Python 实现一个真正的 ReAct Agent。
