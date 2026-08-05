# 第二阶段：Agent 核心机制

> **目标：** 深入理解 Agent 的四个核心机制——ReAct 模式、Tool Calling、Memory、Planning。
> **学完本阶段，你应该能从原理层面解释"Agent 是怎么工作的"。**

---

## 1. ReAct 模式（核心中的核心）

### 1.1 ReAct 是什么

ReAct = **Re**asoning + **Act**ing（推理 + 行动）

这是目前最主流的 Agent 执行模式。核心思想很简单：

> **不要让 LLM 只"想"或只"做"，而是让它在"想"和"做"之间交替进行。**

```
为什么叫 ReAct？

纯推理（Chain of Thought）：想很久，不动手
  → 可能想错方向，因为缺乏真实信息

纯行动（Act without thinking）：直接做，不思考
  → 可能做错事，因为没想清楚

ReAct：想一步 → 做一步 → 看结果 → 再想 → 再做
  → 每一步都有真实信息反馈，方向更准确
```

### 1.2 ReAct 的标准流程

```
┌──────────────────────────────────────────────────────────────┐
│                    ReAct 执行流程                              │
│                                                              │
│  用户输入："北京明天天气怎么样？适合去哪玩？"                     │
│                                                              │
│  ┌─── 第 1 轮 ──────────────────────────────────────────┐    │
│  │ Thought: 用户想知道北京明天的天气和活动推荐。           │    │
│  │          我需要先查询天气数据。                          │    │
│  │ Action: get_weather(city="北京")                       │    │
│  └───────────────────────────────────────────────────────┘    │
│                      │                                       │
│                      ▼                                       │
│  Observation: 北京明天小雨，15°C，湿度80%                       │
│                      │                                       │
│                      ▼                                       │
│  ┌─── 第 2 轮 ──────────────────────────────────────────┐    │
│  │ Thought: 天气是小雨，户外活动不合适。                    │    │
│  │          我应该搜索室内活动推荐。                        │    │
│  │ Action: search(query="北京室内活动推荐")                │    │
│  └───────────────────────────────────────────────────────┘    │
│                      │                                       │
│                      ▼                                       │
│  Observation: 故宫博物院、国家博物馆、798艺术区、三里屯          │
│                      │                                       │
│                      ▼                                       │
│  ┌─── 第 3 轮 ──────────────────────────────────────────┐    │
│  │ Thought: 我已经有了天气信息和活动推荐，                   │    │
│  │          信息足够，可以给出最终回答了。                    │    │
│  │ Final Answer: 明天北京小雨15°C，推荐去故宫、              │    │
│  │              国家博物馆等室内景点。记得带伞！              │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 1.3 ReAct 的 Prompt 模板

ReAct 的核心是用 Prompt 引导 LLM 按 Thought → Action → Observation 的格式输出：

```python
REACT_PROMPT = """你是一个智能助手，通过"思考-行动-观察"循环来解决问题。

你可以使用以下工具：
{tool_descriptions}

请严格按照以下格式回复：

Question: 用户的问题
Thought: 你的思考过程（分析当前情况，决定下一步）
Action: 要调用的工具名
Action Input: 工具参数的 JSON

（然后你会收到 Observation：工具返回的结果）

Observation: 工具执行的结果

... (Thought/Action/Action Input/Observation 可以重复多次)

Thought: 我现在知道最终答案了
Final Answer: 对用户问题的最终回答

开始！

Question: {input}
Thought:{agent_scratchpad}"""
```

**关键理解：** `{agent_scratchpad}` 是之前所有 Thought/Action/Observation 的累积。每一轮循环，这个 scratchpad 会越来越长，LLM 能看到完整的推理历史。

### 1.4 ReAct 的工程实现（Python）

```python
"""
从零实现一个 ReAct Agent
展示 ReAct 模式的核心：Thought → Action → Observation 循环
"""

import json
import re

# ---- 工具定义 ----

def search_web(query: str) -> str:
    """模拟网页搜索"""
    db = {
        "Python": "Python 是一种高级编程语言，由 Guido van Rossum 于 1991 年发布",
        "LangChain": "LangChain 是一个用于构建 LLM 应用的框架，2022 年发布",
        "Agent": "AI Agent 是能自主决策和调用工具的智能系统",
    }
    for key, val in db.items():
        if key.lower() in query.lower():
            return val
    return f"未找到 '{query}' 的相关信息"

def calculator(expression: str) -> str:
    """计算器"""
    try:
        result = eval(expression, {"__builtins__": {}}, {})  # 受限 eval
        return str(result)
    except Exception as e:
        return f"计算错误：{e}"

def get_current_time() -> str:
    """获取当前时间"""
    from datetime import datetime
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

# 工具注册
TOOLS = {
    "search_web": {
        "func": search_web,
        "description": "搜索网络信息。参数：query (str) 搜索关键词",
    },
    "calculator": {
        "func": calculator,
        "description": "数学计算。参数：expression (str) 数学表达式",
    },
    "get_current_time": {
        "func": get_current_time,
        "description": "获取当前日期时间。无参数",
    },
}


class ReActAgent:
    """ReAct Agent —— 推理 + 行动交替循环"""

    def __init__(self, llm_func, tools, max_steps=6):
        self.llm_func = llm_func
        self.tools = tools
        self.max_steps = max_steps

    def _build_tool_descriptions(self):
        return "\n".join(
            f"- {name}: {info['description']}"
            for name, info in self.tools.items()
        )

    def _parse_action(self, text: str):
        """从 LLM 回复中解析 Action 和 Action Input"""
        action_match = re.search(r"Action:\s*(\w+)", text)
        input_match = re.search(r"Action Input:\s*(.+)", text, re.DOTALL)

        if action_match and input_match:
            action = action_match.group(1).strip()
            try:
                action_input = json.loads(input_match.group(1).strip())
            except json.JSONDecodeError:
                action_input = {"query": input_match.group(1).strip()}
            return action, action_input
        return None, None

    def _parse_final_answer(self, text: str):
        """检查是否包含 Final Answer"""
        match = re.search(r"Final Answer:\s*(.+)", text, re.DOTALL)
        if match:
            return match.group(1).strip()
        return None

    def run(self, question: str) -> str:
        """ReAct 主循环"""
        # 构建 Prompt
        tool_desc = self._build_tool_descriptions()

        prompt = f"""你是一个智能助手，使用 Thought-Action-Observation 循环解决问题。

可用工具：
{tool_desc}

格式要求：
Thought: 你的思考过程
Action: 工具名
Action Input: 参数（JSON 格式）

或者当你有足够信息时：
Thought: 我知道答案了
Final Answer: 最终回答

Question: {question}

"""

        scratchpad = ""  # 累积的推理记录

        for step in range(1, self.max_steps + 1):
            print(f"\n--- Step {step} ---")

            # 调 LLM，带上历史推理记录
            full_prompt = prompt + scratchpad
            response = self.llm_func(full_prompt)

            # 打印 LLM 的思考
            thought_match = re.search(r"Thought:\s*(.+?)(?=\n(?:Action|Final)|$)", response, re.DOTALL)
            if thought_match:
                print(f"Thought: {thought_match.group(1).strip()}")

            # 检查是否是最终回答
            final = self._parse_final_answer(response)
            if final:
                print(f"Final Answer: {final}")
                return final

            # 解析工具调用
            action, action_input = self._parse_action(response)
            if action:
                print(f"Action: {action}({action_input})")

                # 执行工具
                tool_info = self.tools.get(action)
                if tool_info:
                    result = tool_info["func"](**action_input)
                else:
                    result = f"工具 '{action}' 不存在"

                print(f"Observation: {result}")

                # 更新 scratchpad
                scratchpad += response + f"\nObservation: {result}\n\n"
            else:
                # 没解析到 action 也没 final answer，可能是格式问题
                scratchpad += response + "\n"
                print(f"(无法解析，继续...)")

        return "达到最大步数限制。"


# ---- 模拟 LLM ----

def mock_llm_react(prompt: str) -> str:
    """模拟 ReAct 模式的 LLM 回复"""
    if "Step 1" not in prompt and "Observation" not in prompt:
        # 第一轮
        if "天气" in prompt or "时间" in prompt:
            return "Thought: 用户想知道当前时间，我需要调用 get_current_time 工具\nAction: get_current_time\nAction Input: {}"
        elif "LangChain" in prompt or "Python" in prompt:
            return "Thought: 我需要搜索相关信息\nAction: search_web\nAction Input: {\"query\": \"" + ("LangChain" if "LangChain" in prompt else "Python") + "\"}"
        elif "+" in prompt or "*" in prompt or "计算" in prompt:
            import re
            expr = re.findall(r'[\d+\-*/(). ]+', prompt)
            expr_str = expr[0].strip() if expr else "1+1"
            return f"Thought: 需要进行数学计算\nAction: calculator\nAction Input: {{\"expression\": \"{expr_str}\"}}"

    # 有 Observation 之后的轮次
    if "Observation:" in prompt:
        last_obs = prompt.split("Observation:")[-1].strip()
        if "未找到" not in last_obs:
            return f"Thought: 我已经获取到了需要的信息，可以回答了\nFinal Answer: 根据查询结果：{last_obs}"
        else:
            return "Thought: 搜索没有找到结果，我直接根据已有知识回答\nFinal Answer: 抱歉，未能找到相关信息。"

    return "Thought: 我需要更多信息才能回答\nFinal Answer: 请提供更多细节。"


# ---- 运行 ----

if __name__ == "__main__":
    agent = ReActAgent(llm_func=mock_llm_react, tools=TOOLS, max_steps=6)

    print("=" * 60)
    result = agent.run("什么是 LangChain？")
    print(f"\n>>> 最终结果：{result}")
```

### 1.5 ReAct 为什么有效

```
对比三种模式：

1. 纯推理（CoT）：
   Thought → Thought → Thought → Answer
   问题：没有真实信息输入，可能在幻觉上推理

2. 纯行动（Act-only）：
   Action → Action → Action → Answer
   问题：没有规划，可能做很多无用的操作

3. ReAct（推理+行动交替）：
   Thought → Action → Observation → Thought → Action → Observation → Answer
   优势：每一步都有真实反馈，推理方向更准确
```

**为什么这样设计？**

因为 LLM 的知识有截止日期且容易幻觉。通过 Action → Observation 引入真实数据，让推理建立在事实基础上，而不是纯靠 LLM 的"记忆"。

---

## 2. Tool Calling 原理

### 2.1 Tool Calling 是什么

Tool Calling = 让 LLM 输出**结构化的工具调用指令**（工具名 + 参数），然后由框架代为执行。

```
没有 Tool Calling 时（纯文本，非常脆弱）：
  LLM 输出："我觉得应该查一下天气，城市是北京"
  → 你需要用正则/解析来提取"天气"和"北京"
  → LLM 可能换个说法："让我看看北京的天气预报"
  → 正则就挂了

有了 Tool Calling 后（结构化 JSON，可靠）：
  LLM 输出：
  {
    "tool_calls": [{
      "function": {
        "name": "get_weather",
        "arguments": "{\"city\": \"北京\"}"
      }
    }]
  }
  → 直接 JSON 解析，100% 可靠
```

### 2.2 Tool Calling 的完整流程

```
┌──────────────────────────────────────────────────────────┐
│              Tool Calling 完整流程                         │
│                                                          │
│  1. 你告诉 LLM：你有这些工具可以用                         │
│     ┌────────────────────────────────────────┐           │
│     │  请求体中包含 "tools" 字段：             │           │
│     │  "tools": [{                            │           │
│     │    "type": "function",                  │           │
│     │    "function": {                        │           │
│     │      "name": "get_weather",             │           │
│     │      "description": "查询天气",         │           │
│     │      "parameters": { JSON Schema }      │           │
│     │    }                                    │           │
│     │  }]                                     │           │
│     └────────────────────────────────────────┘           │
│                    │                                     │
│                    ▼                                     │
│  2. LLM 返回两种情况之一：                                │
│     ┌─────────────────┐  ┌────────────────────────┐     │
│     │ 情况 A：调工具    │  │ 情况 B：直接回答        │     │
│     │ tool_calls: [...]│  │ content: "答案是..."   │     │
│     └─────────────────┘  └────────────────────────┘     │
│            │                                             │
│            ▼（如果是情况 A）                               │
│  3. 框架执行工具，拿到结果                                 │
│     tool_result = get_weather(city="北京")                │
│     → "小雨 15°C"                                        │
│            │                                             │
│            ▼                                             │
│  4. 把工具结果发给 LLM，让它继续决策                        │
│     messages.append({                                    │
│       "role": "tool",                                    │
│       "content": "小雨 15°C"                              │
│     })                                                   │
│            │                                             │
│            ▼                                             │
│  5. LLM 看到工具结果，生成最终回答                          │
│     → "北京明天小雨，建议带伞"                              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 2.3 Tool Calling 的 OpenAI API 调用示例

```python
"""
展示真实的 OpenAI Tool Calling API 用法
（需要有 OpenAI API Key 才能运行）
"""

import openai
import json

# 工具定义（JSON Schema 格式，直接发给 OpenAI）
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询指定城市的当前天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名，如'北京'、'上海'"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "温度单位，默认摄氏度"
                    }
                },
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search",
            "description": "在互联网上搜索信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "搜索关键词"
                    }
                },
                "required": ["query"]
            }
        }
    }
]

# 工具实现
def get_weather(city: str, unit: str = "celsius") -> str:
    """实际调用天气 API"""
    # 这里模拟，实际项目调真实 API
    weather = {"北京": "小雨 15°C", "上海": "晴 22°C", "深圳": "多云 28°C"}
    return weather.get(city, f"{city}: 暂无数据")

def search(query: str) -> str:
    """实际调用搜索 API"""
    return f"搜索 '{query}' 的结果：..."

tool_map = {
    "get_weather": get_weather,
    "search": search,
}

def run_with_tool_calling(user_input: str):
    """
    使用 OpenAI Tool Calling 的完整流程
    """
    messages = [{"role": "user", "content": user_input}]

    while True:
        # 第 1 步：调用 LLM，带上工具定义
        response = openai.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
            tool_choice="auto",  # auto = LLM 自己决定是否调工具
        )

        msg = response.choices[0].message
        messages.append(msg)

        # 第 2 步：检查 LLM 是否要调工具
        if not msg.tool_calls:
            # 没有工具调用 → LLM 直接给了回答
            print(f"最终回答：{msg.content}")
            return msg.content

        # 第 3 步：执行工具调用
        for tool_call in msg.tool_calls:
            func_name = tool_call.function.name
            func_args = json.loads(tool_call.function.arguments)

            print(f"调用工具：{func_name}({func_args})")

            # 执行
            result = tool_map[func_name](**func_args)
            print(f"工具结果：{result}")

            # 第 4 步：把结果返回给 LLM
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": str(result)
            })

        # 继续循环，让 LLM 看到工具结果后决定下一步

# 使用
# run_with_tool_calling("北京天气怎么样？适合户外运动吗？")
```

### 2.4 Tool Calling 的关键设计

**为什么工具描述这么重要？**

```python
# 差的工具描述（模糊）：
{
    "name": "query",
    "description": "查询信息"
}
# LLM 不知道什么时候该用这个工具，也不知道能查什么

# 好的工具描述（精确）：
{
    "name": "query_order",
    "description": "根据订单号查询订单详情，返回订单状态、金额、商品列表、物流信息。当用户询问订单相关问题时使用此工具。",
    "parameters": {
        "type": "object",
        "properties": {
            "order_id": {
                "type": "string",
                "description": "订单号，如 'ORD20260508001'"
            }
        },
        "required": ["order_id"]
    }
}
```

**tool_choice 参数的三种模式：**

| 值 | 含义 | 使用场景 |
|----|------|----------|
| `"auto"` | LLM 自己决定是否调工具 | 默认值，最灵活 |
| `"none"` | 禁止调工具，强制 LLM 直接回答 | 简单问答场景 |
| `{"name": "xxx"}` | 强制调用指定工具 | 你明确知道需要调哪个工具时 |

---

## 3. Memory 系统

### 3.1 为什么需要 Memory

```
LLM 的致命缺陷：无状态

调用 1：
  你："我叫小明"
  LLM："你好小明！"

调用 2：（LLM 完全不记得调用 1）
  你："我叫什么？"
  LLM："抱歉，我不知道你叫什么"

解决：每次调用时，把之前的对话历史全部拼入 Prompt
  → 这就是 Memory 的基本原理
```

### 3.2 Memory 的三层架构

```
┌─────────────────────────────────────────────────────────┐
│               Memory 三层架构                             │
│                                                         │
│  Layer 1：工作记忆（Working Memory）                      │
│  ┌───────────────────────────────────────────────┐      │
│  │  存在哪：LLM 的 Prompt（直接拼接在消息列表中）    │      │
│  │  存什么：当前任务的最近几轮对话 + 工具调用记录     │      │
│  │  大小限制：受 LLM 上下文窗口限制（通常 4K~8K）    │      │
│  │  生命周期：单次 Agent 执行                         │      │
│  │  类比：程序的局部变量                               │      │
│  └───────────────────────────────────────────────┘      │
│                                                         │
│  Layer 2：短期记忆（Short-term Memory）                   │
│  ┌───────────────────────────────────────────────┐      │
│  │  存在哪：内存 / Redis                            │      │
│  │  存什么：当前会话的完整对话历史 + 摘要              │      │
│  │  大小限制：无硬限制，但要做 Token 预算管理          │      │
│  │  生命周期：一次会话（关掉就没了）                  │      │
│  │  类比：Redis Session                              │      │
│  └───────────────────────────────────────────────┘      │
│                                                         │
│  Layer 3：长期记忆（Long-term Memory）                    │
│  ┌───────────────────────────────────────────────┐      │
│  │  存在哪：向量数据库 + 关系数据库                   │      │
│  │  存什么：用户偏好、历史精华、知识库                │      │
│  │  大小限制：无限制                                 │      │
│  │  生命周期：永久，跨会话                            │      │
│  │  类比：MySQL + Elasticsearch                      │      │
│  │  技术：Chroma / Milvus / Pinecone + MySQL        │      │
│  └───────────────────────────────────────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 3.3 Memory 的核心难题：Token 预算管理

```
LLM 上下文窗口 = 128K Token（GPT-4o）
但要分给：
  - System Prompt：     ~1K Token
  - 工具描述：          ~2-5K Token
  - RAG 检索结果：       ~3-5K Token
  - 历史对话：           ？？？
  - 用户当前输入：       ~0.5K Token
  - LLM 输出预留：      ~2K Token

留给历史对话的预算 ≈ 128K - 10K ≈ 118K
但实际生产中一般控制在 4K~8K Token，因为：
  1. Token 越多成本越高
  2. 太长的上下文会降低 LLM 的注意力质量
  3. 延迟也会增加

→ 所以必须做 Memory 的裁剪和压缩！
```

### 3.4 Memory 管理的 Python 实现

```python
"""
Memory 管理：滑动窗口 + 摘要压缩
"""

class MemoryManager:
    """
    三层 Memory 管理：
    1. 工作记忆：当前 Prompt 中的最近 N 轮
    2. 短期记忆：完整的会话历史（内存中）
    3. 摘要压缩：超出的部分生成摘要
    """

    def __init__(self, max_recent_messages=10, max_tokens=4000):
        self.max_recent = max_recent_messages  # 保留最近 N 条
        self.max_tokens = max_tokens
        self.full_history = []     # 完整历史
        self.summary = ""          # 历史摘要

    def add(self, role: str, content: str):
        """添加一条消息"""
        self.full_history.append({"role": role, "content": content})

    def get_context(self) -> list:
        """
        获取给 LLM 的上下文（工作记忆）

        策略：摘要 + 最近 N 轮
        """
        if len(self.full_history) <= self.max_recent:
            # 没超过窗口大小，直接返回全部
            return self.full_history.copy()

        # 超过了：摘要 + 滑动窗口
        context = []

        # 1. 如果有摘要，加入
        if self.summary:
            context.append({
                "role": "system",
                "content": f"之前的对话摘要：{self.summary}"
            })

        # 2. 保留最近 N 轮
        recent = self.full_history[-self.max_recent:]
        context.extend(recent)

        return context

    def compress(self, llm_func):
        """
        压缩历史：把旧消息生成摘要
        在实际项目中，当 Token 超预算时调用
        """
        if len(self.full_history) <= self.max_recent:
            return  # 不需要压缩

        old_messages = self.full_history[:-self.max_recent]
        old_text = "\n".join(
            f"{m['role']}: {m['content']}" for m in old_messages
        )

        # 调 LLM 生成摘要
        new_summary = llm_func(
            f"请用 2-3 句话总结以下对话的要点：\n\n{old_text}"
        )

        # 更新摘要，截断历史
        self.summary = new_summary
        self.full_history = self.full_history[-self.max_recent:]

    def clear(self):
        """清空记忆"""
        self.full_history = []
        self.summary = ""


# 使用示例
memory = MemoryManager(max_recent_messages=6)

# 模拟多轮对话
memory.add("user", "帮我查一下北京天气")
memory.add("assistant", "北京今天晴 25°C")
memory.add("user", "那上海呢")
memory.add("assistant", "上海多云 22°C")
memory.add("user", "这两个城市哪个更适合旅游")
memory.add("assistant", "今天北京天气更好，适合户外旅游")
memory.add("user", "北京有什么景点推荐")
memory.add("assistant", "推荐故宫、长城、颐和园")

# 获取上下文
context = memory.get_context()
print(f"工作记忆中有 {len(context)} 条消息")
for msg in context:
    print(f"  [{msg['role']}] {msg['content'][:50]}...")
```

### 3.5 长期记忆（向量数据库）

短期记忆解决"当前会话记住什么"，长期记忆解决"跨会话记住什么"。

```python
"""
长期记忆：用向量数据库存储和检索
（以 Chroma 为例，最简单的向量数据库）
"""

# 安装：pip install chromadb

import chromadb

class LongTermMemory:
    """
    长期记忆：用向量数据库实现语义检索

    核心思想：
    1. 把文本转成向量（Embedding）
    2. 存入向量数据库
    3. 查询时用语义相似度检索最相关的记忆
    """

    def __init__(self):
        self.client = chromadb.Client()  # 内存模式，生产用 chromadb.PersistentClient
        self.collection = self.client.get_or_create_collection(
            name="agent_memory",
            metadata={"hnsw:space": "cosine"}  # 用余弦相似度
        )

    def store(self, text: str, metadata: dict = None):
        """存储一条记忆"""
        doc_id = f"mem_{self.collection.count()}"
        self.collection.add(
            documents=[text],
            ids=[doc_id],
            metadatas=[metadata or {}]
        )

    def recall(self, query: str, top_k: int = 3) -> list:
        """检索相关记忆"""
        results = self.collection.query(
            query_texts=[query],
            n_results=top_k
        )
        return results["documents"][0] if results["documents"] else []


# 使用
ltm = LongTermMemory()

# 存储
ltm.store("用户喜欢室内的文化活动，不喜欢户外运动", {"type": "preference"})
ltm.store("用户住在北京海淀区", {"type": "profile"})
ltm.store("用户之前问过故宫的门票价格", {"type": "history"})

# 检索
results = ltm.recall("推荐活动")
print(f"相关记忆：{results}")
# → 会找到"用户喜欢室内文化活动"这条记忆
```

**为什么需要向量数据库而不是直接用数据库搜索？**

```
用户说："推荐点好玩的地方"
传统数据库搜索：精确匹配 "好玩的地方" → 找不到任何匹配
向量数据库搜索：语义搜索 → 找到 "用户喜欢室内文化活动" → 推荐故宫

向量数据库理解的是"语义"，不是"关键词"
```

---

## 4. Planning（规划能力）

### 4.1 Planning 是什么

Planning = Agent 把复杂目标拆解成一步步可执行的计划。

这不是独立的代码模块，而是通过 Prompt 引导 LLM 展现出的能力。

### 4.2 Planning 的三种模式

```
┌─────────────────────────────────────────────────────────┐
│            Planning 的三种模式                            │
│                                                         │
│  模式 1：ReAct（边想边做）                                │
│  ┌─────────────────────────────────────────┐            │
│  │ Thought → Action → Observation → ...    │            │
│  │ 每一步都是实时决策，不预设计划             │            │
│  │ 优点：灵活，能根据中间结果调整             │            │
│  │ 缺点：可能走弯路                         │            │
│  └─────────────────────────────────────────┘            │
│                                                         │
│  模式 2：Plan-then-Execute（先规划后执行）                 │
│  ┌─────────────────────────────────────────┐            │
│  │ Step 1: 制定完整计划                      │            │
│  │   1. 查天气  2. 搜索景点  3. 综合推荐     │            │
│  │ Step 2: 按计划逐步执行                    │            │
│  │ Step 3: 如果某步失败，重新规划             │            │
│  │ 优点：方向明确，效率高                     │            │
│  │ 缺点：计划可能不适应新情况                 │            │
│  └─────────────────────────────────────────┘            │
│                                                         │
│  模式 3：Plan-Execute-Replan（规划-执行-重规划）           │
│  ┌─────────────────────────────────────────┐            │
│  │ Step 1: 制定初始计划                      │            │
│  │ Step 2: 执行一步                          │            │
│  │ Step 3: 根据结果重新评估计划               │            │
│  │ Step 4: 调整计划或继续执行                 │            │
│  │ 优点：兼顾方向性和灵活性                   │            │
│  │ 缺点：Token 消耗更高                      │            │
│  └─────────────────────────────────────────┘            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.3 Plan-then-Execute 的 Python 实现

```python
"""
Plan-then-Execute 模式
先让 LLM 制定计划，再逐步执行
"""

class PlanExecuteAgent:
    """先规划，后执行"""

    def __init__(self, llm_func, tools, max_steps=8):
        self.llm_func = llm_func
        self.tools = tools
        self.max_steps = max_steps

    def _make_plan(self, goal: str) -> list:
        """让 LLM 制定计划"""
        plan_prompt = f"""你是一个任务规划师。请为以下目标制定一个执行计划。

目标：{goal}

请输出一个 JSON 数组，每个元素是一个步骤：
[
  {{"step": 1, "action": "描述这一步要做什么", "tool": "需要用的工具名", "reason": "为什么需要这一步"}},
  ...
]

只输出 JSON，不要其他内容。"""

        response = self.llm_func(plan_prompt)
        try:
            # 提取 JSON
            import re
            json_match = re.search(r'\[.*\]', response, re.DOTALL)
            if json_match:
                return json.loads(json_match.group())
        except json.JSONDecodeError:
            pass
        return [{"step": 1, "action": "直接回答用户问题", "tool": "none"}]

    def run(self, goal: str) -> str:
        # Step 1: 制定计划
        print(f"目标：{goal}\n")
        plan = self._make_plan(goal)

        print("=== 执行计划 ===")
        for i, step in enumerate(plan):
            print(f"  {step.get('step', i+1)}. {step.get('action', '?')}")
            if step.get('reason'):
                print(f"     原因：{step['reason']}")
        print()

        # Step 2: 按计划执行
        results = []
        for i, step in enumerate(plan):
            if i >= self.max_steps:
                break

            action = step.get("action", "")
            tool_name = step.get("tool", "")

            print(f"--- 执行步骤 {i+1}: {action} ---")

            # 调 LLM 执行这一步
            step_prompt = f"""目标：{goal}
当前步骤：{action}
{'之前的执行结果：' + str(results) if results else '这是第一步'}

请执行当前步骤，给出结果。"""
            result = self.llm_func(step_prompt)
            results.append({"step": action, "result": result})
            print(f"结果：{result[:100]}...\n")

        # Step 3: 综合结果
        summary_prompt = f"""目标：{goal}
执行结果：{json.dumps(results, ensure_ascii=False)}

请根据执行结果，给出最终的完整回答。"""
        final = self.llm_func(summary_prompt)

        print(f"=== 最终回答 ===\n{final}")
        return final


# 使用
# agent = PlanExecuteAgent(llm_func=your_llm, tools=TOOLS)
# agent.run("分析 Python 和 JavaScript 的优劣，给出学习建议")
```

### 4.4 三种模式怎么选

| 模式 | 适合场景 | 不适合场景 |
|------|----------|-----------|
| ReAct | 工具依赖强的任务（需要中间结果决定下一步） | 需要全局规划的复杂任务 |
| Plan-then-Execute | 步骤明确、依赖关系清晰的任务 | 中间结果会大幅改变计划的场景 |
| Plan-Execute-Replan | 复杂任务、环境可能变化 | 简单任务（过度设计） |

**工程实践中的选择：2026 年主流是 ReAct + 简单 Plan 的混合模式。**

---

## 5. 常见误区

| 误区 | 真相 |
|------|------|
| "ReAct 就是 while 循环调 LLM" | while 循环是骨架，ReAct 的关键是 Thought-Action-Observation 的格式约束 |
| "Tool Calling 是 LLM 直接执行代码" | LLM 只输出 JSON 指令，代码是框架/执行器代为调用的 |
| "Memory 就是把对话历史拼到 Prompt 里" | 那只是最简单的短期记忆，还有摘要压缩和向量检索 |
| "Planning 需要单独的模型" | Planning 是通过 Prompt 引导出来的能力，不是独立模型 |
| "向量数据库什么都能搜" | 向量搜索是语义相似度，不是万能的，精确查询还是用传统数据库 |

---

## 👉 这一阶段你应该掌握的能力

1. **能解释 ReAct 模式的原理**：为什么推理和行动要交替进行
2. **能手写 ReAct Agent**：用 Prompt 模板引导 LLM 按 Thought → Action → Observation 格式输出
3. **理解 Tool Calling 的完整流程**：从工具定义 → LLM 返回 JSON → 框架执行 → 结果返回
4. **能设计 Memory 系统**：滑动窗口 + 摘要压缩 + 向量检索
5. **能根据场景选择 Planning 模式**：ReAct / Plan-then-Execute / Plan-Execute-Replan

---

> **下一阶段预告：** 第三阶段将拆解 LangChain 的架构设计——Chains、Agents、Tools、Memory 模块的设计思想，以及 AgentExecutor 的内部工作原理。
