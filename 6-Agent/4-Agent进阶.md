# 第四阶段：进阶 Agent 能力

> **目标：** 掌握 Agent 的进阶能力——多工具协作、RAG + Agent、多轮推理、自我反思。
> **学完本阶段，你应该能设计和实现中等复杂度的 Agent 应用。**

---

## 1. 多工具协作

### 1.1 什么是多工具协作

> Agent 不只是调一个工具，而是根据任务需要，**在多个工具之间灵活切换和组合**。

```
用户："分析一下苹果公司的股票，看看值不值得买"

Agent 的执行过程：
  Step 1: search("苹果公司 2026 最新新闻")    → 获取最新信息
  Step 2: get_stock_price("AAPL")              → 获取实时股价
  Step 3: get_financial_data("AAPL")           → 获取财务数据
  Step 4: calculate("PE_ratio / growth_rate")  → 计算估值指标
  Step 5: 综合分析，给出建议

→ 5 步调了 4 个不同的工具，每一步的输出是下一步的输入依据
```

### 1.2 多工具协作的关键设计

```
多工具协作的核心挑战：

1. 工具选择：LLM 需要从多个工具中选对
   解决：精确的工具描述 + 合理的工具数量（建议不超过 10 个）

2. 参数传递：上一步的结果可能需要作为下一步的参数
   解决：LLM 通过 Memory（上下文）自动获取之前的工具结果

3. 执行顺序：有些工具有依赖关系
   解决：LLM 通过推理决定先后顺序

4. 结果聚合：多个工具的结果需要综合分析
   解决：LLM 天然擅长综合分析，最后一轮汇总
```

### 1.3 多工具协作实战：智能研究助手

```python
"""
实战案例：智能研究助手
能力：搜索 + 获取数据 + 计算 + 生成报告
"""

from langchain.tools import tool
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field


# ---- 工具定义 ----

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")

@tool("search", args_schema=SearchInput)
def search(query: str) -> str:
    """搜索互联网上的最新信息。当需要查找新闻、事实、数据时使用。"""
    fake_results = {
        "Python": "Python 在 2026 年仍是 TIOBE 排行榜第一，广泛应用于 AI/ML、Web 开发、数据科学",
        "Rust": "Rust 在 2026 年增长迅速，在系统编程领域被越来越多公司采用",
        "LangChain": "LangChain 在 2026 年发布了 v1.0 正式版，Agent 生态更成熟",
        "AI Agent": "AI Agent 在 2026 年成为企业级应用的热门方向",
    }
    for key, val in fake_results.items():
        if key.lower() in query.lower():
            return val
    return f"搜索 '{query}'：暂无详细结果"


class StockInput(BaseModel):
    ticker: str = Field(description="股票代码，如 AAPL")

@tool("get_stock_price", args_schema=StockInput)
def get_stock_price(ticker: str) -> str:
    """获取股票的实时价格和基本数据。当需要股票价格信息时使用。"""
    prices = {
        "AAPL": "价格: $198.50, 涨跌: +1.2%, 市值: 3.1万亿",
        "GOOGL": "价格: $175.20, 涨跌: -0.5%, 市值: 2.2万亿",
        "MSFT": "价格: $420.80, 涨跌: +0.8%, 市值: 3.1万亿",
    }
    return prices.get(ticker.upper(), f"{ticker}: 暂无数据")


class CalculatorInput(BaseModel):
    expression: str = Field(description="数学表达式")

@tool("calculator", args_schema=CalculatorInput)
def calculator(expression: str) -> str:
    """执行数学计算。当需要进行数值计算时使用。"""
    try:
        allowed = set("0123456789+-*/.() ")
        if not all(c in allowed for c in expression):
            return "错误：包含不允许的字符"
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"计算错误：{e}"


class WriteInput(BaseModel):
    topic: str = Field(description="报告主题")
    key_points: str = Field(description="关键要点列表")

@tool("write_report", args_schema=WriteInput)
def write_report(topic: str, key_points: str) -> str:
    """根据关键要点生成一份结构化的分析报告。"""
    return f"""# {topic} 分析报告

## 关键发现
{key_points}

## 结论
基于以上分析，建议持续关注该领域的发展动态。
"""


# ---- 构建 Agent ----

def create_research_agent():
    llm = ChatOpenAI(model="gpt-4o", temperature=0)

    tools = [search, get_stock_price, calculator, write_report]

    prompt = ChatPromptTemplate.from_messages([
        ("system", """你是一个专业的研究分析师。你的任务是：
1. 使用搜索工具获取最新信息
2. 使用数据工具获取具体数据
3. 使用计算工具进行数据分析
4. 最后生成结构化的分析报告

注意：
- 先搜集信息，再分析，最后出报告
- 数据要准确，观点要有依据
- 报告结构要清晰"""),
        ("human", "{input}"),
        MessagesPlaceholder("agent_scratchpad"),
    ])

    agent = create_openai_tools_agent(llm=llm, tools=tools, prompt=prompt)

    return AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,
        max_iterations=8,
    )


# 使用
# agent = create_research_agent()
# result = agent.invoke({"input": "分析一下苹果公司和微软的股票表现，哪个更值得投资？"})
```

---

## 2. RAG + Agent

### 2.1 RAG 是什么

> **RAG（Retrieval-Augmented Generation）= 检索增强生成**
>
> 先从知识库中检索相关内容，再让 LLM 基于检索到的内容回答问题。

```
为什么需要 RAG？

LLM 的问题：
  1. 知识有截止日期（不知道最新信息）
  2. 不知道你公司的内部数据
  3. 容易幻觉（编造信息）

RAG 的解决：
  1. 把你的文档/数据存入向量数据库
  2. 用户提问时，先从数据库检索相关内容
  3. 把检索到的内容作为上下文交给 LLM
  4. LLM 基于真实数据回答，而不是靠"记忆"

类比：
  LLM = 一个博学但没有看过你公司文档的顾问
  RAG = 先把相关文档递给顾问，让他基于文档回答
```

### 2.2 RAG 的工作流程

```
┌──────────────────────────────────────────────────────────────┐
│                    RAG 工作流程                                │
│                                                              │
│  离线阶段（提前做）：                                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐             │
│  │ 你的文档   │ → │ 切分成块   │ → │ 向量化并存入   │             │
│  │ PDF/网页  │   │ (Chunk)  │   │ 向量数据库     │             │
│  └──────────┘   └──────────┘   └──────────────┘             │
│                                                              │
│  在线阶段（用户提问时）：                                       │
│                                                              │
│  用户提问："公司请假制度是什么？"                                │
│       │                                                     │
│       ▼                                                     │
│  1. 把问题转成向量                                             │
│       │                                                     │
│       ▼                                                     │
│  2. 在向量数据库中检索最相似的文档片段                             │
│       │                                                     │
│       ▼                                                     │
│  3. 找到：                                                   │
│     "年假：工作满1年5天，满3年10天..."                          │
│     "病假：凭医院证明，每年不超过10天..."                        │
│       │                                                     │
│       ▼                                                     │
│  4. 组装 Prompt：                                             │
│     "基于以下内容回答用户问题：                                  │
│      [检索到的文档内容]                                        │
│      用户问题：公司请假制度是什么？"                              │
│       │                                                     │
│       ▼                                                     │
│  5. LLM 基于真实文档内容回答                                    │
│     "根据公司制度，请假分为以下几种..."                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2.3 RAG + Agent 实战：企业知识库助手

```python
"""
RAG + Agent 实战：企业知识库助手
能力：基于公司文档回答问题 + 调用外部工具
"""

# 安装：pip install chromadb langchain langchain-openai langchain-community

from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import Chroma
from langchain.tools import tool
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from pydantic import BaseModel, Field


# ============================================================
# 第一步：构建知识库（RAG 的离线阶段）
# ============================================================

def build_knowledge_base():
    """
    把文档加载 → 切分 → 向量化 → 存入 Chroma
    """
    # 1. 加载文档（实际项目中可以加载 PDF、Word、网页等）
    # 这里用内存中的文档模拟
    from langchain_core.documents import Document

    docs = [
        Document(page_content="""
        公司请假制度：
        1. 年假：工作满1年享有5天年假，满3年10天，满5年15天。
        2. 病假：凭医院证明，每年不超过10天带薪病假。
        3. 事假：需提前申请，每年不超过5天，超过部分扣薪。
        4. 婚假：法定婚假3天，晚婚加7天。
        5. 产假：女员工产假98天，男员工陪产假15天。
        """, metadata={"source": "请假制度", "type": "HR"}),
        Document(page_content="""
        报销流程：
        1. 填写报销单（需附发票原件）
        2. 直属经理审批（1个工作日内）
        3. 财务审核（3个工作日内）
        4. 打款到工资卡（每月15日统一打款）
        注意事项：单笔超过5000元需部门总监审批。
        """, metadata={"source": "报销流程", "type": "财务"}),
        Document(page_content="""
        技术栈规范（2026版）：
        后端：Java 21 + Spring Boot 3.x
        前端：React 19 + TypeScript
        数据库：MySQL 8.0 + Redis 7.x
        消息队列：Kafka 3.x
        容器化：Docker + Kubernetes
        监控：Prometheus + Grafana
        """, metadata={"source": "技术栈", "type": "技术"}),
    ]

    # 2. 切分文档
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=200,      # 每块最大 200 字符
        chunk_overlap=50,    # 块之间重叠 50 字符（保证语义连贯）
    )
    splits = text_splitter.split_documents(docs)

    # 3. 向量化并存入 Chroma
    embeddings = OpenAIEmbeddings()
    vectorstore = Chroma.from_documents(
        documents=splits,
        embedding=embeddings,
        collection_name="company_knowledge"
    )

    return vectorstore


# ============================================================
# 第二步：创建 RAG 检索工具
# ============================================================

class KnowledgeSearchInput(BaseModel):
    query: str = Field(description="要在知识库中搜索的问题")

# 全局变量（实际项目中用依赖注入）
_vectorstore = None

def set_vectorstore(vs):
    global _vectorstore
    _vectorstore = vs

@tool("search_knowledge", args_schema=KnowledgeSearchInput)
def search_knowledge(query: str) -> str:
    """在公司知识库中搜索信息。当用户询问公司制度、流程、规范等内部信息时使用。"""
    if _vectorstore is None:
        return "知识库未初始化"

    # 检索最相关的 3 个文档片段
    results = _vectorstore.similarity_search(query, k=3)
    if not results:
        return "在知识库中未找到相关信息"

    # 拼接检索结果
    context = "\n\n".join(
        f"[来源: {doc.metadata.get('source', '未知')}]\n{doc.page_content}"
        for doc in results
    )
    return context


# ---- 其他工具 ----

@tool("search_web")
def search_web(query: str) -> str:
    """搜索互联网信息。当知识库中没有答案时使用。"""
    return f"网络搜索 '{query}' 的结果：暂无"


# ============================================================
# 第三步：构建 RAG Agent
# ============================================================

def create_rag_agent(vectorstore):
    set_vectorstore(vectorstore)

    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    tools = [search_knowledge, search_web]

    prompt = ChatPromptTemplate.from_messages([
        ("system", """你是公司的智能助手，可以回答关于公司制度、流程、技术规范的问题。

工作方式：
1. 先在公司知识库中搜索相关信息
2. 基于检索到的内容回答用户问题
3. 如果知识库中没有，再尝试网络搜索
4. 回答时要注明信息来源

重要：必须基于检索到的真实内容回答，不要编造！"""),
        ("human", "{input}"),
        MessagesPlaceholder("agent_scratchpad"),
    ])

    agent = create_openai_tools_agent(llm=llm, tools=tools, prompt=prompt)

    return AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,
        max_iterations=5,
    )


# ============================================================
# 使用
# ============================================================

# vs = build_knowledge_base()
# agent = create_rag_agent(vs)
#
# # 测试：查询公司内部信息
# result = agent.invoke({"input": "公司年假怎么算的？"})
# # → Agent 会调 search_knowledge → 检索到请假制度 → 基于真实内容回答
#
# result = agent.invoke({"input": "报销流程是什么？单笔5000以上怎么办？"})
# # → 检索到报销流程文档 → 回答并特别说明5000以上的额外审批
#
# result = agent.invoke({"input": "公司用的什么技术栈？"})
# # → 检索到技术规范文档 → 列出完整技术栈
```

### 2.4 RAG 的常见问题

```
1. 检索不到相关内容
   原因：文档切分太粗/太细、Embedding 模型不够好、查询表述差异
   解决：调整 chunk_size、换更好的 Embedding 模型、加查询改写

2. 检索到但回答不准
   原因：Prompt 没有强调"基于检索内容回答"
   解决：Prompt 加约束："必须基于以下内容回答，如果内容中没有相关信息，请说不知道"

3. 检索慢
   原因：向量数据库数据量太大
   解决：加索引、预过滤、减少返回数量

4. 内容更新后检索到旧内容
   原因：向量数据库没有同步更新
   解决：建立文档更新机制，定期重建索引
```

---

## 3. 多轮推理（Chain of Thought）

### 3.1 什么是 CoT

> **Chain of Thought（CoT）= 让 LLM 把推理过程一步步展示出来，而不是直接给答案。**

```
没有 CoT：
  问：一个商店打8折后是120元，原价是多少？
  答：150元（LLM 可能算错，而且你不知道它怎么算的）

有 CoT：
  问：一个商店打8折后是120元，原价是多少？请一步步推理。
  答：
    打8折意味着价格是原价的 80%
    所以 原价 × 0.8 = 120
    原价 = 120 / 0.8 = 150
    答案是 150 元

  （每一步都清晰可见，如果中间哪步错了你能发现）
```

**为什么 CoT 有效？**

```
LLM 是逐 Token 生成的，每个 Token 只能看到之前的 Token。

没有 CoT 时：
  "150元" ← LLM 直接生成答案，中间推理在"脑子里"一步完成
  → 如果问题复杂，一步推理容易出错

有 CoT 时：
  "打8折意味着..." → "原价 × 0.8 = 120" → "原价 = 150"
  → LLM 逐步生成推理过程，每一步都帮助它"看清"下一步
  → 相当于把复杂问题拆成了多个简单问题

类比：
  CoT ≈ 草稿纸
  没有草稿纸时，你在脑子里算 123 × 456，容易出错
  有草稿纸时，你一步步列竖式，准确率大幅提升
```

### 3.2 CoT 的几种方式

```python
"""
Chain of Thought 的几种 Prompt 方式
"""

# ---- 方式 1：Zero-shot CoT（最简单）----
# 就是在 Prompt 末尾加一句话

prompt1 = """
一个商店打8折后是120元，原价是多少？

请一步步思考。
"""
# 关键就一句："请一步步思考" / "Let's think step by step"


# ---- 方式 2：Few-shot CoT（给示例）----
# 给 LLM 看几个"带推理过程的问答示例"

prompt2 = """
请按以下格式回答问题：

Q: 15 × 12 = ?
A: 15 × 12 = 15 × 10 + 15 × 2 = 150 + 30 = 180

Q: 一个长方形长5宽3，面积和周长分别是？
A: 面积 = 长 × 宽 = 5 × 3 = 15
   周长 = 2 × (长 + 宽) = 2 × (5 + 3) = 16

Q: 如果一本书每天看30页，12天看完，这本书一共多少页？
A:
"""


# ---- 方式 3：Auto-CoT（自动生成推理链）----
# 让 LLM 自己生成推理示例，然后用这些示例来回答

prompt3 = """
请为以下问题生成详细的推理过程：

问题：小明有100元，买了3本书每本25元，又买了一支笔15元，还剩多少钱？

推理过程：
"""
```

### 3.3 CoT 在 Agent 中的应用

```
在 Agent 中，CoT 体现在两个地方：

1. ReAct 的 Thought 步骤
   → Agent 的每一步 Thought 就是一个 CoT 推理
   → "用户问天气 → 我需要调天气工具 → 城市是北京"

2. 最终回答前的推理
   → Agent 在给出 Final Answer 前，先展示推理过程
   → "根据查询结果，北京小雨 → 不适合户外 → 推荐室内"

CoT 和 ReAct 的关系：
  CoT = 推理方法（让 LLM 一步步思考）
  ReAct = Agent 框架（把推理和行动结合起来）
  ReAct 中的 Thought 就是 CoT
```

---

## 4. 自我反思（Reflection Agent）

### 4.1 什么是 Reflection

> **Reflection = 让 Agent 执行完后，检查自己的结果，发现不足，然后改进。**

```
为什么需要 Reflection？

Agent 的问题：
  - 第一次执行可能不完美
  - 可能遗漏了重要信息
  - 回答质量可能不够好

Reflection 的解决：
  - 执行完后，让 LLM 审查自己的输出
  - 找出不足
  - 重新执行或修改

类比：
  写作文 → 自己检查一遍 → 发现问题 → 修改 → 再检查 → 定稿
```

### 4.2 Reflection 的流程

```
┌──────────────────────────────────────────────────────────┐
│              Reflection Agent 流程                         │
│                                                          │
│  1. 执行（Execute）                                       │
│     Agent 完成任务，生成初始结果                             │
│          │                                               │
│          ▼                                               │
│  2. 反思（Reflect）                                       │
│     LLM 审查初始结果，找出不足：                             │
│     - 信息是否完整？                                       │
│     - 逻辑是否自洽？                                       │
│     - 有没有遗漏？                                        │
│          │                                               │
│          ▼                                               │
│  3. 改进（Revise）                                        │
│     根据反思结果，改进输出                                   │
│          │                                               │
│          ▼                                               │
│  4. 重复 2-3，直到质量达标或达到最大轮数                      │
│          │                                               │
│          ▼                                               │
│  5. 输出最终结果                                           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 4.3 Reflection Agent 实战

```python
"""
Reflection Agent：写代码 → 审查 → 改进
"""

def reflection_agent(llm_func, task: str, max_rounds: int = 3) -> str:
    """
    一个简单的 Reflection Agent
    流程：执行 → 反思 → 改进 → 再反思 → ...
    """
    # ---- 第 1 步：执行 ----
    execute_prompt = f"""请完成以下任务：

{task}

请给出你的最佳答案。"""

    current_output = llm_func(execute_prompt)
    print(f"=== 初始结果 ===\n{current_output}\n")

    # ---- 第 2-4 步：反思 + 改进循环 ----
    for round_num in range(1, max_rounds + 1):
        # 反思
        reflect_prompt = f"""请审查以下输出，找出不足之处：

任务：{task}

输出：
{current_output}

请从以下维度审查：
1. 完整性：是否遗漏了重要内容？
2. 准确性：信息是否正确？
3. 清晰度：表述是否清晰？
4. 可操作性：建议是否可以执行？

如果输出已经很好，请回复"无需改进"。
否则请列出具体的改进建议。"""

        reflection = llm_func(reflect_prompt)
        print(f"--- 第 {round_num} 轮反思 ---")
        print(f"反思结果：{reflection}\n")

        if "无需改进" in reflection:
            print("质量达标，停止反思。")
            break

        # 改进
        revise_prompt = f"""基于反思意见，改进你的输出：

原始任务：{task}

之前的输出：
{current_output}

反思意见：
{reflection}

请给出改进后的输出。"""

        current_output = llm_func(revise_prompt)
        print(f"改进结果：{current_output[:200]}...\n")

    return current_output


# ---- 模拟 LLM ----

def mock_llm_reflection(prompt: str) -> str:
    """模拟带反思的 LLM"""
    if "请完成以下任务" in prompt and "写一个" in prompt:
        return """
def binary_search(arr, target):
    left, right = 0, len(arr)
    while left < right:
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid
    return -1
"""
    elif "审查" in prompt:
        return """
审查结果：
1. 完整性：缺少边界条件处理（空数组）
2. 准确性：right 初始值应该是 len(arr) - 1，否则可能越界
3. 清晰度：缺少函数文档和注释
4. 可操作性：缺少使用示例
"""
    elif "改进" in prompt:
        return """
def binary_search(arr: list, target: int) -> int:
    \"\"\"在有序数组中二分查找目标值，返回索引，未找到返回 -1。\"\"\"
    if not arr:
        return -1

    left, right = 0, len(arr) - 1
    while left <= right:
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1

# 示例
print(binary_search([1, 3, 5, 7, 9], 5))  # 2
print(binary_search([1, 3, 5, 7, 9], 4))  # -1
"""
    return "无需改进"


# ---- 运行 ----

if __name__ == "__main__":
    result = reflection_agent(
        llm_func=mock_llm_reflection,
        task="写一个 Python 二分查找函数",
        max_rounds=3,
    )
    print(f"\n=== 最终结果 ===\n{result}")
```

---

## 5. 完整实战案例：智能旅行规划 Agent

```python
"""
完整案例：智能旅行规划 Agent
能力：天气查询 + 景点搜索 + 酒店查询 + 费用计算 + 行程生成
展示：多工具协作 + RAG + CoT 的综合运用
"""

from langchain.tools import tool
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field


# ---- 工具定义 ----

class CityInput(BaseModel):
    city: str = Field(description="城市名")

@tool("get_weather", args_schema=CityInput)
def get_weather(city: str) -> str:
    """查询城市天气。用于了解目的地天气状况。"""
    data = {
        "北京": "晴 25°C 空气质量良",
        "上海": "多云 22°C 湿度较高",
        "成都": "阴 20°C 可能有小雨",
        "三亚": "晴 30°C 适合海滩",
    }
    return data.get(city, f"{city}: 暂无数据")


class SearchInput(BaseModel):
    query: str = Field(description="搜索内容")

@tool("search_attractions", args_schema=SearchInput)
def search_attractions(query: str) -> str:
    """搜索景点信息。当需要了解某城市有什么好玩的时使用。"""
    db = {
        "北京": "故宫(¥60)、长城(¥40)、颐和园(¥30)、天坛(¥15)、798艺术区(免费)",
        "上海": "外滩(免费)、迪士尼(¥475)、豫园(¥40)、南京路(免费)、东方明珠(¥220)",
        "成都": "大熊猫基地(¥55)、宽窄巷子(免费)、锦里(免费)、都江堰(¥80)",
        "三亚": "亚龙湾(免费)、天涯海角(¥81)、蜈支洲岛(¥144)、南山寺(¥129)",
    }
    for city, spots in db.items():
        if city in query:
            return spots
    return "暂无景点信息"


class BudgetInput(BaseModel):
    destination: str = Field(description="目的地")
    days: int = Field(description="天数")

@tool("estimate_budget", args_schema=BudgetInput)
def estimate_budget(destination: str, days: int) -> str:
    """估算旅行预算。根据目的地和天数给出大致费用。"""
    daily_costs = {
        "北京": {"住宿": 400, "餐饮": 150, "交通": 80, "门票": 100},
        "上海": {"住宿": 450, "餐饮": 160, "交通": 80, "门票": 120},
        "成都": {"住宿": 300, "餐饮": 120, "交通": 60, "门票": 80},
        "三亚": {"住宿": 500, "餐饮": 180, "交通": 100, "门票": 130},
    }
    costs = daily_costs.get(destination, {"住宿": 350, "餐饮": 130, "交通": 70, "门票": 100})
    total = sum(costs.values()) * days
    breakdown = "\n".join(f"  {k}: ¥{v}/天 × {days}天 = ¥{v*days}" for k, v in costs.items())
    return f"{destination}{days}天预估费用：\n{breakdown}\n  合计: ¥{total}"


class CalculatorInput(BaseModel):
    expression: str = Field(description="数学表达式")

@tool("calculate", args_schema=CalculatorInput)
def calculate(expression: str) -> str:
    """数学计算。"""
    try:
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"错误：{e}"


# ---- 构建 Agent ----

def create_travel_agent():
    llm = ChatOpenAI(model="gpt-4o", temperature=0.7)

    tools = [get_weather, search_attractions, estimate_budget, calculate]

    prompt = ChatPromptTemplate.from_messages([
        ("system", """你是一个专业的旅行规划师。你的任务是帮用户制定旅行计划。

工作步骤：
1. 了解目的地的天气（调用天气工具）
2. 搜索当地景点（调用景点搜索）
3. 估算预算（调用预算工具）
4. 综合信息，制定详细的行程规划

输出要求：
- 每天安排 2-3 个景点，考虑地理位置就近
- 根据天气推荐室内/户外活动
- 给出预算明细
- 提供实用建议（穿衣、交通、注意事项）"""),
        ("human", "{input}"),
        MessagesPlaceholder("agent_scratchpad"),
    ])

    agent = create_openai_tools_agent(llm=llm, tools=tools, prompt=prompt)

    return AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,
        max_iterations=10,
    )


# 使用
# agent = create_travel_agent()
# result = agent.invoke({"input": "帮我规划一个北京3天的旅行计划，预算尽量控制"})
# print(result["output"])
```

---

## 6. 常见误区

| 误区 | 真相 |
|------|------|
| "RAG 就是搜索 + LLM" | RAG 的核心难点在文档切分、检索质量、上下文拼接策略 |
| "CoT 就是让 LLM 多说几句" | CoT 的关键是把复杂推理拆成简单步骤，每一步都让 LLM "看见" |
| "Reflection 一定能提升质量" | Reflection 可能引入过度修改，反而降低质量，需要设定停止条件 |
| "工具越多 Agent 越强" | 工具越多 LLM 选错工具的概率越高，建议不超过 10 个 |
| "RAG 能解决所有幻觉问题" | RAG 减少了幻觉，但如果检索到的文档本身有误，LLM 照样会错 |

---

## 👉 这一阶段你应该掌握的能力

1. **能设计多工具协作的 Agent**：工具选择、参数传递、结果聚合
2. **能实现 RAG + Agent**：文档加载 → 向量化 → 检索 → 基于 LLM 回答
3. **理解 CoT 的原理**：为什么一步步推理比直接给答案更准确
4. **能实现 Reflection Agent**：执行 → 反思 → 改进的循环
5. **能综合运用这些能力**：设计并实现一个完整的中等复杂度 Agent

---

> **下一阶段预告：** 第五阶段将深入多 Agent 系统——协作式架构、层级式架构、Agent 通信机制，以及一个完整的多 Agent Demo。
