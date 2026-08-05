# 第三阶段：LangChain 架构拆解

> **目标：** 理解 LangChain 的设计思想，不只是会用，更要懂为什么这样设计。
> **学完本阶段，你应该能看懂 LangChain 源码，并知道它背后每个设计决策的原因。**

---

## 1. LangChain 是什么

### 1.1 一句话概括

> **LangChain 是一个"Agent 编排框架"——它帮你把 LLM + Tools + Memory 串起来，让开发 Agent 变得像搭积木一样。**

### 1.2 为什么需要框架

在第二阶段我们已经手写了 Agent，你会发现核心逻辑其实就几十行。那为什么还需要 LangChain？

```
手写 Agent 的痛点：
1. 每次都要写 Tool 解析逻辑（正则 / JSON 解析）
2. Memory 管理要自己实现（滑动窗口、摘要压缩）
3. 不同 LLM 的 API 格式不一样（OpenAI vs Claude vs 本地模型）
4. 错误处理很繁琐（工具不存在、参数错误、LLM 格式不对）
5. 调试困难（不知道 LLM 中间在想什么）

LangChain 做的事：
1. 统一的工具调用接口（不管底层是 OpenAI 还是 Claude）
2. 内置 Memory 管理（多种策略）
3. 内置 Agent 类型（ReAct、Plan-Execute 等）
4. 内置调试和日志（LangSmith）
5. 丰富的工具生态（搜索、数据库、代码执行...）

本质：LangChain 把你在第二阶段手写的那些"胶水代码"标准化了。
```

### 1.3 LangChain 的设计哲学

```
LangChain 的核心设计思想：

1. 模块化（Modularity）
   → 每个组件（LLM / Tool / Memory / Chain）都是独立的，可以单独替换

2. 可组合性（Composability）
   → 组件之间通过统一接口连接，像乐高一样拼装

3. 抽象但不隐藏（Transparent Abstraction）
   → 封装了复杂性，但你仍然能看到和控制底层细节

类比：
  LangChain 之于 Agent 开发 ≈ Spring 之于后端开发
  → Spring 封装了依赖注入、AOP、MVC，但你不理解原理也能用
  → 但理解原理后，你才能用好、排查问题、做架构设计
```

---

## 2. LangChain 核心模块

```
┌──────────────────────────────────────────────────────────┐
│                LangChain 核心架构                         │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │                 Chains（链）                       │    │
│  │  把多个组件串成一条流水线                            │    │
│  │  LLMChain = PromptTemplate + LLM + OutputParser  │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │                 Agents（智能体）                    │    │
│  │  能自主决策的执行单元                               │    │
│  │  AgentExecutor = Agent + Tools + 循环逻辑          │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │                 Tools（工具）                       │    │
│  │  标准化的外部能力接口                               │    │
│  │  name + description + func + args_schema          │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │                 Memory（记忆）                      │    │
│  │  多种记忆策略                                      │    │
│  │  Buffer / Summary / VectorStore                   │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │              PromptTemplate（模板）                 │    │
│  │  参数化的 Prompt 管理                               │    │
│  │  变量替换 + 格式控制                                │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │            OutputParsers（输出解析）                │    │
│  │  把 LLM 的文本输出转成结构化数据                     │    │
│  │  JSON 解析 / 列表提取 / Pydantic 模型              │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 2.1 PromptTemplate——Prompt 的模板引擎

**本质：** PromptTemplate 就是一个字符串模板引擎，类似 Python 的 `str.format()`，但专门为 LLM Prompt 设计。

```python
"""
PromptTemplate：把 Prompt 从硬编码变成可复用的模板
"""

from langchain.prompts import PromptTemplate, ChatPromptTemplate

# ---- 基础用法 ----

# 硬编码的 Prompt（不好维护）
prompt = f"请用{style}风格，翻译以下文本：{text}"

# 用 PromptTemplate（可复用、可管理）
template = PromptTemplate.from_template(
    "请用{style}风格，翻译以下文本：{text}"
)

# 使用
prompt = template.format(style="幽默", text="Hello World")
print(prompt)
# → "请用幽默风格，翻译以下文本：Hello World"

# ---- ChatPromptTemplate（对话场景更常用）----

chat_template = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}，擅长{skill}"),
    ("human", "{input}")
])

messages = chat_template.format_messages(
    role="Python 专家",
    skill="代码审查",
    input="帮我看看这段代码有没有 bug"
)
```

**为什么需要 PromptTemplate？**

```
1. 可复用：同一个模板，不同参数 → 不同 Prompt
2. 可管理：Prompt 集中管理，不用散落在代码各处
3. 可测试：可以单独测试 Prompt 模板，不用调 LLM
4. 可版本化：Prompt 可以独立版本控制

类比：
  PromptTemplate 之于 Prompt ≈ SQL 模板引擎 之于 SQL
  把硬编码的字符串变成参数化的模板
```

### 2.2 OutputParser——把 LLM 输出变成结构化数据

**本质：** LLM 返回的是字符串，OutputParser 把它解析成 Python 对象。

```python
"""
OutputParser：把 LLM 的文本输出转成结构化数据
"""

from langchain.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

# 定义期望的输出结构
class MovieRecommendation(BaseModel):
    title: str = Field(description="电影名称")
    year: int = Field(description="上映年份")
    reason: str = Field(description="推荐理由")
    rating: float = Field(description="评分 1-10")

# 创建解析器
parser = PydanticOutputParser(pydantic_object=MovieRecommendation)

# parser 会自动生成格式说明，拼入 Prompt
format_instructions = parser.get_format_instructions()
print(format_instructions)
# → "请按以下 JSON 格式输出：{"title": ..., "year": ..., ...}"

# 使用
llm_output = '{"title": "盗梦空间", "year": 2010, "reason": "烧脑", "rating": 9.3}'
result = parser.parse(llm_output)
print(result.title)    # "盗梦空间"
print(result.year)     # 2010
print(result.rating)   # 9.3
```

**为什么需要 OutputParser？**

```
因为 LLM 输出的是文本，但程序需要结构化数据。

流程：
  程序定义结构 → OutputParser 生成格式说明 → 拼入 Prompt → LLM 按格式输出 → Parser 解析回 Python 对象

类比：
  OutputParser ≈ JSON.parse()，但多了"告诉 LLM 怎么输出"的部分
```

### 2.3 Chains——组件的流水线

**本质：** Chain 就是把多个组件串成一条流水线——上一个的输出是下一个的输入。

```
┌──────────────────────────────────────────────────────────┐
│                  Chain 的本质                              │
│                                                          │
│  最基础的 Chain：LLMChain                                  │
│                                                          │
│  PromptTemplate → LLM → OutputParser                     │
│       ↓             ↓          ↓                         │
│  组装 Prompt    调LLM拿结果  解析成结构化数据               │
│                                                          │
│  更复杂的 Chain：SequentialChain                          │
│                                                          │
│  Chain1 → Chain2 → Chain3 → ... → 最终结果               │
│  (翻译)   (摘要)   (分析)                                 │
│                                                          │
│  本质就是 Pipeline 模式 / 责任链模式                       │
└──────────────────────────────────────────────────────────┘
```

```python
"""
Chain：把组件串成流水线
"""

from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

# 创建 LLM
llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 创建 Prompt 模板
prompt = PromptTemplate.from_template(
    "请用{style}风格，总结以下文本的要点：\n\n{text}"
)

# 创建 Chain（把 Prompt + LLM 串起来）
chain = LLMChain(llm=llm, prompt=prompt)

# 运行（自动走完 Prompt → LLM → 输出 的流水线）
result = chain.run(style="简洁", text="LangChain 是一个...")
print(result)
```

**LLMChain 内部执行流程：**

```
1. 接收参数：style="简洁", text="LangChain 是一个..."
2. PromptTemplate 处理：替换变量 → 生成完整 Prompt
3. 调 LLM：把 Prompt 发给 GPT-4o
4. 返回 LLM 的输出

就这几步。本质就是一个封装了 Prompt + LLM 的函数调用。
```

### 2.4 Tools——标准化的工具接口

**本质：** Tool 把一个 Python 函数包装成 LLM 能理解的"工具描述"。

```python
"""
Tools：把 Python 函数包装成 LLM 能调用的工具
"""

from langchain.tools import tool
from pydantic import BaseModel, Field

# ---- 方式 1：用装饰器（最简单）----

@tool
def search_web(query: str) -> str:
    """在互联网上搜索信息。当需要查找最新信息时使用。"""
    return f"搜索结果：{query} 的相关信息..."

@tool
def calculate(expression: str) -> str:
    """执行数学计算。支持加减乘除和括号。"""
    try:
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"计算错误：{e}"

# LangChain 自动从函数签名和 docstring 生成工具描述
print(search_web.name)         # "search_web"
print(search_web.description)  # "在互联网上搜索信息..."
print(search_web.args)         # {"query": {"type": "string"}}


# ---- 方式 2：用 Pydantic 定义参数（更精确）----

class WeatherInput(BaseModel):
    city: str = Field(description="城市名，如'北京'")
    date: str = Field(default="today", description="日期，默认今天")

@tool("get_weather", args_schema=WeatherInput)
def get_weather(city: str, date: str = "today") -> str:
    """查询指定城市和日期的天气。当用户问天气相关问题时使用。"""
    return f"{city} {date}: 晴 25°C"

# 这样 LLM 就知道每个参数的含义和类型


# ---- 工具列表（传给 Agent 用）----
tools = [search_web, calculate, get_weather]
```

**Tool 的设计原理：**

```
Tool 包装了三个信息：

1. name：工具名（给 LLM 看的标识符）
   → "search_web"

2. description：描述（告诉 LLM 这个工具能做什么、什么时候该用）
   → "在互联网上搜索信息。当需要查找最新信息时使用。"

3. args_schema：参数定义（告诉 LLM 需要传什么参数）
   → {"query": {"type": "string", "description": "搜索关键词"}}

为什么需要这三个？
→ 因为它们会被拼入 Prompt，LLM 根据这些信息决定：
   1. 要不要用这个工具（看 description）
   2. 怎么传参数（看 args_schema）
```

### 2.5 Memory——对话记忆管理

```python
"""
LangChain Memory：多种记忆策略
"""

from langchain.memory import (
    ConversationBufferMemory,      # 全量记忆
    ConversationBufferWindowMemory, # 滑动窗口
    ConversationSummaryMemory,      # 摘要压缩
    VectorStoreRetrieverMemory,     # 向量检索
)
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o")

# ---- 1. 全量记忆（最简单，但浪费 Token）----
memory1 = ConversationBufferMemory()
memory1.save_context({"input": "你好"}, {"output": "你好！"})
memory1.save_context({"input": "我叫小明"}, {"output": "你好小明！"})
print(memory1.load_memory_variables({}))
# → 包含所有对话历史

# ---- 2. 滑动窗口（只保留最近 K 轮）----
memory2 = ConversationBufferWindowMemory(k=2)
# 只保留最近 2 轮对话，超出的自动丢弃
# 适合：长对话、Token 预算有限的场景

# ---- 3. 摘要压缩（调 LLM 生成摘要）----
memory3 = ConversationSummaryMemory(llm=llm)
# 把旧对话压缩成摘要，节省 Token
# 适合：超长对话

# ---- 4. 向量检索记忆 ----
# memory4 = VectorStoreRetrieverMemory(retriever=vectorstore.as_retriever())
# 用语义相似度检索相关记忆
# 适合：跨会话的长期记忆
```

---

## 3. AgentExecutor——LangChain 的心脏

### 3.1 AgentExecutor 是什么

AgentExecutor 是 LangChain 中最核心的类——它就是那个 "while 循环"。

```
AgentExecutor = Agent + Tools + Memory + while 循环

它做的事情和我们在第二阶段手写的一样：
1. 把 Prompt + 工具描述 + Memory 拼好，发给 LLM
2. 解析 LLM 的回复（调工具 or 最终回答）
3. 如果调工具：执行 → 结果加入上下文 → 回到第 1 步
4. 如果最终回答：返回

区别是：AgentExecutor 做了大量工程化处理（错误重试、超时控制、回调等）
```

### 3.2 AgentExecutor 的内部执行流程

```
┌──────────────────────────────────────────────────────────────┐
│              AgentExecutor 内部执行流程                        │
│                                                              │
│  1. 初始化                                                    │
│     ├── 加载 Agent（决定用哪种 Agent 类型：ReAct / OpenAI 等） │
│     ├── 加载 Tools（注册工具列表）                             │
│     └── 加载 Memory（初始化记忆）                              │
│                                                              │
│  2. 进入主循环（最多 max_iterations 次）                       │
│     │                                                        │
│     ├── 2.1 构建 Prompt                                      │
│     │   ├── System Prompt（角色设定）                         │
│     │   ├── Tool 描述（告诉 LLM 有哪些工具）                   │
│     │   ├── Memory 上下文（对话历史）                          │
│     │   ├── 用户输入                                          │
│     │   └── Agent Scratchpad（之前步骤的记录）                 │
│     │                                                        │
│     ├── 2.2 调 LLM                                           │
│     │   └── 返回 AgentAction（调工具）或 AgentFinish（完成）   │
│     │                                                        │
│     ├── 2.3 如果是 AgentAction（要调工具）                     │
│     │   ├── 查找 Tool（根据名字）                              │
│     │   ├── 校验参数                                          │
│     │   ├── 执行 Tool                                         │
│     │   ├── 记录结果到 Scratchpad                              │
│     │   └── 回到 2.1（继续循环）                               │
│     │                                                        │
│     ├── 2.4 如果是 AgentFinish（任务完成）                     │
│     │   ├── 保存到 Memory                                     │
│     │   └── 返回最终结果                                       │
│     │                                                        │
│     └── 2.5 如果出错                                          │
│         ├── 记录错误                                          │
│         ├── 如果有 handle_parsing_errors → 继续               │
│         └── 否则 → 抛异常                                     │
│                                                              │
│  3. 后处理                                                    │
│     ├── 更新 Memory                                           │
│     └── 触发回调（callbacks）                                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 3.3 完整的 LangChain Agent 示例

```python
"""
完整的 LangChain Agent 示例
展示：自定义 Tool + 构建 Agent + 执行任务
"""

from langchain.agents import (
    AgentExecutor,
    create_openai_tools_agent,
)
from langchain.tools import tool
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationBufferMemory
from pydantic import BaseModel, Field


# ============================================================
# 第一步：定义工具
# ============================================================

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")

@tool("search", args_schema=SearchInput)
def search(query: str) -> str:
    """在互联网上搜索信息。当需要查找最新信息、事实、数据时使用此工具。"""
    search_db = {
        "LangChain": "LangChain 是用于构建 LLM 应用的开源框架，2022 年发布，支持 Python 和 JavaScript",
        "Python": "Python 是一种高级编程语言，广泛用于 AI/ML、Web 开发、数据科学",
        "Agent": "AI Agent 是能自主决策和调用工具的智能系统",
    }
    for key, val in search_db.items():
        if key.lower() in query.lower():
            return val
    return f"搜索 '{query}' 未找到相关结果"


class CalculatorInput(BaseModel):
    expression: str = Field(description="数学表达式，如 '2+3*4'")

@tool("calculator", args_schema=CalculatorInput)
def calculator(expression: str) -> str:
    """执行数学计算。当需要进行加减乘除等运算时使用此工具。"""
    try:
        allowed = set("0123456789+-*/.() ")
        if not all(c in allowed for c in expression):
            return "错误：表达式包含不允许的字符"
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"计算错误：{e}"


class WeatherInput(BaseModel):
    city: str = Field(description="城市名")

@tool("get_weather", args_schema=WeatherInput)
def get_weather(city: str) -> str:
    """查询指定城市的天气。当用户询问天气、温度、是否下雨时使用此工具。"""
    weather_data = {
        "北京": "小雨 15°C 湿度80% 东北风3级",
        "上海": "晴 22°C 湿度45% 东南风2级",
        "深圳": "多云 28°C 湿度70% 南风1级",
    }
    return weather_data.get(city, f"{city}: 暂无天气数据")


# ============================================================
# 第二步：构建 Agent
# ============================================================

def create_agent():
    """创建一个完整的 LangChain Agent"""

    # 1. LLM
    llm = ChatOpenAI(
        model="gpt-4o",
        temperature=0,  # Agent 场景用 0，减少随机性
    )

    # 2. 工具列表
    tools = [search, calculator, get_weather]

    # 3. Prompt（必须包含 agent_scratchpad）
    prompt = ChatPromptTemplate.from_messages([
        ("system", """你是一个智能助手，可以使用工具来帮助用户。

规则：
1. 先思考，再行动
2. 如果需要实时信息，使用搜索工具
3. 如果需要计算，使用计算器
4. 如果需要天气，使用天气查询
5. 给出回答时要有理有据"""),
        MessagesPlaceholder("chat_history"),  # 对话历史（Memory）
        ("human", "{input}"),
        MessagesPlaceholder("agent_scratchpad"),  # Agent 的推理记录
    ])

    # 4. Memory
    memory = ConversationBufferMemory(
        memory_key="chat_history",
        return_messages=True,
    )

    # 5. 创建 Agent
    agent = create_openai_tools_agent(
        llm=llm,
        tools=tools,
        prompt=prompt,
    )

    # 6. 创建 AgentExecutor（核心：把 Agent + Tools + Memory 串起来）
    agent_executor = AgentExecutor(
        agent=agent,
        tools=tools,
        memory=memory,
        verbose=True,           # 打印详细日志
        max_iterations=6,       # 最多循环 6 次
        handle_parsing_errors=True,  # 解析错误时不崩溃
    )

    return agent_executor


# ============================================================
# 第三步：使用 Agent
# ============================================================

# agent = create_agent()
#
# # 测试 1：简单查询（可能直接回答，不调工具）
# result = agent.invoke({"input": "什么是 Python？"})
#
# # 测试 2：需要调工具
# result = agent.invoke({"input": "北京天气怎么样？适合户外运动吗？"})
#
# # 测试 3：需要计算
# result = agent.invoke({"input": "帮我算一下 (123 + 456) * 2"})
#
# # 测试 4：有记忆的对话
# result1 = agent.invoke({"input": "我叫小明"})
# result2 = agent.invoke({"input": "我叫什么？"})  # 应该记住"小明"
```

### 3.4 LangChain Agent 执行时发生了什么（一步步）

```
用户输入："北京天气怎么样？"

第 1 步：AgentExecutor 拼装 Prompt
  System: "你是一个智能助手..."
  Tool 描述: "search: 搜索信息..."
           "calculator: 数学计算..."
           "get_weather: 查询天气..."
  Chat History: （空，第一次对话）
  Human: "北京天气怎么样？"
  Agent Scratchpad: （空，第一步）

第 2 步：发给 LLM
  LLM 看到 Prompt，分析：
  - 用户问天气 → 应该用 get_weather 工具
  - 城市是"北京"

第 3 步：LLM 返回工具调用
  tool_calls: [{
    "function": {
      "name": "get_weather",
      "arguments": "{\"city\": \"北京\"}"
    }
  }]

第 4 步：AgentExecutor 执行工具
  result = get_weather(city="北京")
  → "小雨 15°C 湿度80% 东北风3级"

第 5 步：把工具结果加入 Prompt，再次调 LLM
  Agent Scratchpad: "调用了 get_weather('北京') → 小雨 15°C..."

第 6 步：LLM 看到工具结果，生成最终回答
  "北京今天小雨，气温 15°C，湿度较高。建议带伞出门，不太适合户外活动。"

第 7 步：AgentExecutor 返回结果，更新 Memory
```

---

## 4. LangChain 的设计哲学总结

```
为什么 LangChain 要这样设计？

1. PromptTemplate
   Why: Prompt 是 LLM 应用的"代码"，需要版本管理、复用、测试
   How: 参数化模板，把变量从 Prompt 中分离

2. OutputParser
   Why: LLM 输出是文本，程序需要结构化数据
   How: 在 Prompt 中告诉格式要求，解析输出

3. Chain
   Why: 单次 LLM 调用不够用，需要多步处理
   How: Pipeline 模式，串联多个组件

4. Tool
   Why: LLM 只能"说"，需要标准化的"做"的接口
   How: 把函数包装成 LLM 能理解的描述

5. Memory
   Why: LLM 无状态，需要外部上下文管理
   How: 多种策略（全量/窗口/摘要/向量）

6. AgentExecutor
   Why: Agent 的循环逻辑是通用的，不应每次重写
   How: 标准化的 while 循环 + 错误处理 + 回调

一句话总结：
  LangChain 的本质就是把"手写 Agent 的胶水代码"标准化成可复用的模块。
  它不是魔法，是你第二阶段手写的那些代码的工程化版本。
```

---

## 5. LangChain vs 手写 Agent：什么时候用什么

| 场景 | 手写 Agent | LangChain |
|------|-----------|-----------|
| 学习/理解原理 | 最好 | 太抽象，不利于理解 |
| 简单原型 | 够用 | 可能过度 |
| 多工具 + 复杂 Prompt | 代码量大 | 内置支持，方便 |
| 需要调试/观测 | 自己写日志 | LangSmith 集成 |
| 需要换 LLM 提供商 | 改代码 | 换一行配置 |
| 生产环境 | 自己造轮子风险高 | 生态成熟 |

**建议：先手写理解原理（已完成），再用框架提高效率。**

---

## 6. 常见误区

| 误区 | 真相 |
|------|------|
| "LangChain 是 Agent 的唯一选择" | 还有 LangGraph、CrewAI、AutoGen 等框架 |
| "用了 LangChain 就不需要理解原理" | 不理解原理出问题时完全无法排查 |
| "Chain 已经过时了，都用 LCEL" | LCEL（LangChain Expression Language）是新的组合方式，但 Chain 的设计思想不变 |
| "LangChain 性能很好" | LangChain 的抽象层有性能开销，极致优化场景可能需要手写 |
| "AgentExecutor 能处理所有情况" | 复杂工作流需要 LangGraph |

---

## 👉 这一阶段你应该掌握的能力

1. **理解 LangChain 的六大核心模块**：PromptTemplate / OutputParser / Chain / Tool / Memory / AgentExecutor
2. **知道每个模块为什么这样设计**：解决什么问题、核心抽象是什么
3. **能用 LangChain 构建一个完整的 Agent**：自定义 Tool + 构建 Agent + 执行任务
4. **能追踪 AgentExecutor 的内部执行流程**：Prompt 拼装 → LLM 调用 → 工具执行 → 循环
5. **能在手写和框架之间做选择**：简单场景手写，复杂场景用框架

---

> **下一阶段预告：** 第四阶段将深入 Agent 的进阶能力——多工具协作、RAG + Agent、Chain of Thought、Reflection Agent，包含完整的实战案例。
