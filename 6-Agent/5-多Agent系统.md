# 第五阶段：多 Agent 系统

> **目标：** 理解多 Agent 系统的架构模式、通信机制，能设计一个简单的多 Agent 协作系统。
> **学完本阶段，你应该能判断什么时候需要多 Agent，以及怎么设计多 Agent 架构。**

---

## 1. 为什么需要多 Agent

### 1.1 单 Agent 的局限

```
单 Agent 能做的事很多，但遇到以下场景就力不从心：

1. 任务太复杂，一个 Agent 搞不定
   → "帮我调研竞品、分析市场、写报告、做 PPT"
   → 需要：研究员 + 分析师 + 写手 + 设计师

2. 需要不同专业能力
   → "帮我审查这段代码的安全性、性能和可维护性"
   → 需要：安全专家 + 性能专家 + 架构专家

3. 需要互相校验
   → "帮我写一篇论文"
   → 需要：写手 + 审稿人

类比：
  单 Agent = 一个人干所有事（什么都会一点，但什么都不精）
  多 Agent = 一个团队协作（每人各司其职）
```

### 1.2 多 Agent 的核心思想

> **把复杂任务拆分给多个专业 Agent，每个 Agent 只做自己擅长的事，通过协作完成整体目标。**

```
┌──────────────────────────────────────────────────────────┐
│              单 Agent vs 多 Agent                          │
│                                                          │
│  单 Agent：                                               │
│  用户 ──→ [万能 Agent] ──→ 结果                           │
│           (一个 Agent 试图做所有事)                        │
│           问题：Prompt 太长、工具太多、容易出错             │
│                                                          │
│  多 Agent：                                               │
│  用户 ──→ [协调者] ──→ [研究员] ──→ [分析师] ──→ 结果      │
│           (每个 Agent 专注一个领域)                        │
│           优势：每个 Agent 的 Prompt 短、工具少、更精准     │
└──────────────────────────────────────────────────────────┘
```

---

## 2. 多 Agent 架构模式

### 2.1 模式一：协作式（Peer-to-Peer）

```
┌──────────────────────────────────────────────────────────┐
│              协作式架构                                    │
│                                                          │
│              ┌──────────┐                                │
│              │  路由器    │                                │
│              │ (Router)  │                                │
│              └────┬─────┘                                │
│          ┌────────┼────────┐                             │
│          ▼        ▼        ▼                             │
│     ┌────────┐ ┌────────┐ ┌────────┐                    │
│     │研究员  │ │分析师  │ │写手    │                    │
│     │(搜索)  │ │(计算)  │ │(写作)  │                    │
│     └────┬───┘ └────┬───┘ └────┬───┘                    │
│          │          │          │                          │
│          └──────────┼──────────┘                         │
│                     ▼                                    │
│              最终结果汇总                                  │
│                                                          │
│  特点：                                                   │
│  - 每个 Agent 地位平等                                    │
│  - 路由器决定把任务分给谁                                  │
│  - Agent 之间不直接通信，通过路由器中转                     │
│  - 适合：简单分工场景                                     │
└──────────────────────────────────────────────────────────┘
```

### 2.2 模式二：层级式（Hierarchical / Supervisor）

```
┌──────────────────────────────────────────────────────────┐
│              层级式架构（最常用）                           │
│                                                          │
│              ┌──────────────┐                             │
│              │   Supervisor  │ ← 主控 Agent               │
│              │  (任务分配+    │    负责理解目标、            │
│              │   结果汇总)   │    分配子任务、汇总结果      │
│              └──────┬───────┘                             │
│                     │                                    │
│          ┌──────────┼──────────┐                         │
│          ▼          ▼          ▼                         │
│     ┌────────┐ ┌────────┐ ┌────────┐                    │
│     │研究员  │ │分析师  │ │写手    │ ← 子 Agent          │
│     │        │ │        │ │        │   只负责自己的任务   │
│     │工具:   │ │工具:   │ │工具:   │                     │
│     │- 搜索  │ │- 计算  │ │- 模板  │                     │
│     │- 网页  │ │- 数据库│ │- 格式  │                     │
│     └────────┘ └────────┘ └────────┘                    │
│          │          │          │                          │
│          └──────────┼──────────┘                         │
│                     ▼                                    │
│              结果返回给 Supervisor                         │
│              Supervisor 决定是否需要更多步骤               │
│                                                          │
│  特点：                                                   │
│  - Supervisor 是"大脑"，其他 Agent 是"专家"               │
│  - Supervisor 控制任务流程                                 │
│  - 每个 Agent 有自己的工具和 Prompt                        │
│  - 适合：复杂任务、需要流程控制的场景                       │
└──────────────────────────────────────────────────────────┘
```

### 2.3 模式三：辩论式（Debate）

```
┌──────────────────────────────────────────────────────────┐
│              辩论式架构                                    │
│                                                          │
│     ┌─────────────┐          ┌─────────────┐             │
│     │ Agent A     │ ← 辩论 → │ Agent B     │             │
│     │ (正方观点)   │          │ (反方观点)   │             │
│     └──────┬──────┘          └──────┬──────┘             │
│            │                        │                    │
│            └───────────┬────────────┘                    │
│                        ▼                                 │
│                 ┌─────────────┐                          │
│                 │  Judge      │ ← 评判 Agent              │
│                 │  (综合评判)  │   综合双方观点，给出结论   │
│                 └─────────────┘                          │
│                                                          │
│  特点：                                                   │
│  - 多个 Agent 从不同角度分析同一问题                       │
│  - 通过辩论发现盲点和不足                                   │
│  - 适合：决策分析、方案评审、代码审查                       │
└──────────────────────────────────────────────────────────┘
```

### 2.4 三种模式怎么选

| 模式 | 适合场景 | 复杂度 | 成本 |
|------|----------|--------|------|
| 协作式 | 简单分工、任务独立 | 低 | 中 |
| 层级式 | 复杂任务、需要流程控制 | 中 | 高 |
| 辩论式 | 需要多角度分析、决策评审 | 中 | 高 |

**2026 年主流是层级式（Supervisor 模式）。**

---

## 3. AutoGPT / BabyAGI 原理

### 3.1 AutoGPT 的核心思想

```
AutoGPT 是 2023 年最火的 Agent 项目，核心思想：

1. 给定一个目标
2. Agent 自主把目标拆成任务列表
3. 逐个执行任务
4. 根据执行结果动态添加新任务
5. 循环直到所有任务完成

流程：
  目标："创建一个网站"
    │
    ▼
  任务列表：["设计页面结构", "写 HTML", "添加 CSS", "测试"]
    │
    ▼
  执行 "设计页面结构" → 完成 → 可能新增 "设计导航栏"
    │
    ▼
  执行 "写 HTML" → 完成
    │
    ▼
  ... 直到所有任务完成
```

### 3.2 BabyAGI 的核心循环

```
BabyAGI 更简单，就三步循环：

┌──────────────────────────────────────────────┐
│            BabyAGI 循环                       │
│                                              │
│  1. Task Creation（创建任务）                  │
│     根据目标和已有结果，生成新的子任务            │
│                                              │
│  2. Task Prioritization（任务排序）            │
│     对任务列表按优先级排序                      │
│                                              │
│  3. Task Execution（执行任务）                 │
│     取优先级最高的任务，执行，得到结果           │
│     结果反馈给第 1 步                          │
│                                              │
│  循环 1→2→3→1→2→3 直到任务列表为空              │
└──────────────────────────────────────────────┘
```

### 3.3 用 Python 实现一个简化版 BabyAGI

```python
"""
简化版 BabyAGI：任务驱动的多步 Agent
展示 AutoGPT / BabyAGI 的核心循环逻辑
"""

import json


class BabyAGI:
    """
    简化版 BabyAGI

    核心循环：
    1. 从任务队列取最高优先级任务
    2. 执行任务
    3. 根据结果创建新任务
    4. 重新排序任务队列
    5. 重复直到任务队列为空
    """

    def __init__(self, llm_func, max_tasks=10):
        self.llm_func = llm_func
        self.max_tasks = max_tasks
        self.task_queue = []      # 任务队列
        self.completed = []       # 已完成的任务和结果

    def add_task(self, task: str, priority: int = 1):
        """添加任务到队列"""
        self.task_queue.append({
            "task": task,
            "priority": priority,
        })
        # 按优先级排序（数字越小优先级越高）
        self.task_queue.sort(key=lambda x: x["priority"])

    def _execute_task(self, task: str) -> str:
        """执行单个任务"""
        context = "\n".join(
            f"- {c['task']}: {c['result']}"
            for c in self.completed[-3:]  # 取最近 3 条作为上下文
        )

        prompt = f"""请执行以下任务，给出简洁的结果。

已完成的任务：
{context if context else '（这是第一个任务）'}

当前任务：{task}

请直接给出执行结果："""

        return self.llm_func(prompt)

    def _create_new_tasks(self, objective: str, last_task: str, last_result: str):
        """根据执行结果，决定是否需要创建新任务"""
        prompt = f"""根据目标 "{objective}"，我刚刚完成了任务 "{last_task}"，
得到结果：{last_result}

已完成的所有任务：{json.dumps([c['task'] for c in self.completed], ensure_ascii=False)}

请判断：
1. 是否需要创建新的子任务来完成目标？
2. 如果需要，列出新的子任务（JSON 数组格式）

如果没有新任务，输出：[]
如果需要新任务，输出：[{{"task": "任务描述", "priority": 1-5}}]
只输出 JSON 数组。"""

        response = self.llm_func(prompt)
        try:
            import re
            json_match = re.search(r'\[.*\]', response, re.DOTALL)
            if json_match:
                new_tasks = json.loads(json_match.group())
                for t in new_tasks:
                    self.add_task(t["task"], t.get("priority", 3))
        except json.JSONDecodeError:
            pass

    def run(self, objective: str):
        """运行 BabyAGI 主循环"""
        print(f"目标：{objective}\n")

        # 初始任务：分析目标
        self.add_task(f"分析目标 '{objective}'，列出需要完成的子任务", priority=1)

        task_count = 0

        while self.task_queue and task_count < self.max_tasks:
            # 1. 取最高优先级任务
            current = self.task_queue.pop(0)
            task = current["task"]
            task_count += 1

            print(f"--- 任务 {task_count}（优先级 {current['priority']}）---")
            print(f"任务：{task}")

            # 2. 执行任务
            result = self._execute_task(task)
            print(f"结果：{result[:150]}...\n")

            # 记录已完成
            self.completed.append({"task": task, "result": result})

            # 3. 创建新任务
            self._create_new_tasks(objective, task, result)

        # 汇总结果
        print(f"\n{'='*60}")
        print(f"所有任务完成！共完成 {len(self.completed)} 个任务。")
        print(f"{'='*60}")

        # 生成最终报告
        summary = self._generate_final_report(objective)
        print(f"\n最终报告：\n{summary}")
        return summary

    def _generate_final_report(self, objective: str):
        """生成最终报告"""
        results = "\n".join(
            f"- {c['task']}: {c['result']}"
            for c in self.completed
        )
        prompt = f"""根据以下任务执行结果，生成一份总结报告。

目标：{objective}

执行结果：
{results}

请给出简洁的总结报告。"""
        return self.llm_func(prompt)


# ---- 模拟 LLM ----

def mock_llm_baby(prompt: str) -> str:
    """模拟 BabyAGI 的 LLM"""
    if "分析目标" in prompt and "子任务" in prompt:
        return json.dumps([
            {"task": "调研主题背景", "priority": 1},
            {"task": "收集关键数据", "priority": 2},
            {"task": "撰写分析报告", "priority": 3},
        ], ensure_ascii=False)
    elif "调研" in prompt:
        return "调研完成：Python 在 AI 领域占主导地位，Go 在云原生领域增长迅速"
    elif "收集" in prompt:
        return "数据收集完成：Python 使用率 35%，Go 使用率 12%，增长率 Go > Python"
    elif "撰写" in prompt:
        return "报告撰写完成：Python 仍是主流，但 Go 在特定领域有优势"
    elif "总结" in prompt:
        return "研究结论：建议 Python 作为主力语言，Go 作为云原生方向的第二语言"
    elif "JSON" in prompt or "新任务" in prompt or "子任务" in prompt:
        return "[]"
    return "任务完成"


# ---- 运行 ----

if __name__ == "__main__":
    baby = BabyAGI(llm_func=mock_llm_baby, max_tasks=10)
    baby.run("分析 Python 和 Go 语言在 2026 年的发展趋势")
```

---

## 4. Agent 通信机制

### 4.1 Agent 之间怎么交流

```
多 Agent 系统中，Agent 之间需要传递信息。常见方式：

1. 共享内存（Shared Memory）
   → 所有 Agent 读写同一个数据存储
   → 简单，但有并发问题

2. 消息传递（Message Passing）
   → Agent 之间直接发送消息
   → 更灵活，但实现更复杂

3. 黑板模式（Blackboard）
   → 所有 Agent 读写一个"黑板"（共享状态）
   → 结合了前两者的优点

2026 年主流实现：基于 LLM 的消息传递
  → Agent A 的输出作为 Agent B 的输入
  → 本质就是：LLM 调用 → 结果 → 作为下一个 LLM 调用的 Prompt
```

### 4.2 消息传递的 Python 实现

```python
"""
Agent 通信：简单的消息传递机制
"""

from dataclasses import dataclass, field
from typing import Optional


@dataclass
class Message:
    """Agent 之间的消息"""
    sender: str           # 发送者
    receiver: str         # 接收者（"all" 表示广播）
    content: str          # 消息内容
    msg_type: str = "info"  # 消息类型：info / task / result / error


class MessageBus:
    """消息总线：Agent 之间的通信中心"""

    def __init__(self):
        self.messages = []     # 消息历史
        self.queues = {}       # 每个 Agent 的消息队列

    def register(self, agent_name: str):
        """注册一个 Agent"""
        self.queues[agent_name] = []

    def send(self, message: Message):
        """发送消息"""
        self.messages.append(message)
        if message.receiver == "all":
            # 广播给所有 Agent
            for name, queue in self.queues.items():
                if name != message.sender:
                    queue.append(message)
        else:
            # 发给特定 Agent
            if message.receiver in self.queues:
                self.queues[message.receiver].append(message)

    def receive(self, agent_name: str) -> list:
        """接收消息"""
        messages = self.queues.get(agent_name, []).copy()
        self.queues[agent_name] = []
        return messages


class Agent:
    """基础 Agent 类"""

    def __init__(self, name: str, role: str, bus: MessageBus):
        self.name = name
        self.role = role
        self.bus = bus
        self.bus.register(name)

    def process(self, task: str) -> str:
        """处理任务（子类重写）"""
        raise NotImplementedError


class SupervisorAgent(Agent):
    """协调者 Agent：分配任务、汇总结果"""

    def __init__(self, name: str, bus: MessageBus, workers: list):
        role = "你是任务协调者，负责分配任务和汇总结果"
        super().__init__(name, role, bus)
        self.workers = workers  # 可用的工作 Agent 列表

    def process(self, task: str) -> str:
        """分配任务给工作 Agent"""
        # 这里简化为按顺序分配
        # 实际项目中，这里会让 LLM 决定分给谁
        results = []

        for worker in self.workers:
            self.bus.send(Message(
                sender=self.name,
                receiver=worker.name,
                content=task,
                msg_type="task"
            ))

            # Worker 处理
            result = worker.process(task)
            results.append(f"[{worker.name}] {result}")

        # 汇总
        summary = f"任务完成。\n" + "\n".join(results)
        return summary


class WorkerAgent(Agent):
    """工作 Agent：执行具体任务"""

    def __init__(self, name: str, role: str, bus: MessageBus, llm_func):
        super().__init__(name, role, bus)
        self.llm_func = llm_func

    def process(self, task: str) -> str:
        """用自己的角色处理任务"""
        prompt = f"你是{self.role}。请处理以下任务：{task}"
        return self.llm_func(prompt)


# ---- 使用 ----

def mock_llm_worker(prompt: str) -> str:
    if "安全" in prompt:
        return "安全分析：未发现 SQL 注入和 XSS 风险，但建议增加 CSRF 防护"
    elif "性能" in prompt:
        return "性能分析：数据库查询可以优化，建议添加索引和缓存"
    elif "可维护性" in prompt:
        return "可维护性分析：代码结构清晰，但建议增加单元测试覆盖率"
    return "分析完成"


if __name__ == "__main__":
    bus = MessageBus()

    # 创建工作 Agent
    security_agent = WorkerAgent("安全专家", "代码安全审查专家", bus, mock_llm_worker)
    perf_agent = WorkerAgent("性能专家", "代码性能分析专家", bus, mock_llm_worker)
    quality_agent = WorkerAgent("质量专家", "代码可维护性分析专家", bus, mock_llm_worker)

    # 创建协调者
    supervisor = SupervisorAgent(
        "协调者", bus,
        workers=[security_agent, perf_agent, quality_agent]
    )

    # 运行
    result = supervisor.process("审查这个用户登录模块的代码")
    print(result)
```

---

## 5. 多 Agent 系统实战：代码审查团队

```python
"""
完整案例：多 Agent 代码审查团队
架构：层级式（Supervisor + 3 个专业 Agent）
"""

import json


class CodeReviewTeam:
    """
    多 Agent 代码审查团队

    架构：
    Supervisor（协调者）
      ├── SecurityReviewer（安全审查员）
      ├── PerformanceReviewer（性能审查员）
      └── StyleReviewer（代码风格审查员）

    流程：
    1. Supervisor 接收代码
    2. 分发给三个审查员
    3. 收集审查结果
    4. 综合生成报告
    """

    def __init__(self, llm_func):
        self.llm_func = llm_func
        self.reviewers = {
            "security": {
                "name": "安全审查员",
                "focus": "SQL注入、XSS、CSRF、权限控制、敏感数据处理",
                "prompt": """你是代码安全审查专家。请从以下方面审查代码：
1. SQL 注入风险
2. XSS 跨站脚本风险
3. CSRF 跨站请求伪造
4. 权限控制是否完善
5. 敏感数据是否加密
6. 是否有信息泄露风险

请给出：
- 严重程度（高/中/低）
- 问题描述
- 修复建议"""
            },
            "performance": {
                "name": "性能审查员",
                "focus": "数据库查询、缓存、并发、内存泄漏",
                "prompt": """你是代码性能分析专家。请从以下方面审查代码：
1. 数据库查询是否高效
2. 是否有不必要的 N+1 查询
3. 是否有缓存策略
4. 并发处理是否合理
5. 是否有内存泄漏风险
6. 算法时间复杂度

请给出：
- 严重程度（高/中/低）
- 问题描述
- 优化建议"""
            },
            "style": {
                "name": "代码风格审查员",
                "focus": "命名规范、代码结构、可维护性、测试覆盖",
                "prompt": """你是代码质量审查专家。请从以下方面审查代码：
1. 命名是否清晰规范
2. 函数是否过长（建议不超过 20 行）
3. 是否有重复代码
4. 错误处理是否完善
5. 是否有单元测试
6. 注释是否合理

请给出：
- 严重程度（高/中/低）
- 问题描述
- 改进建议"""
            },
        }

    def review(self, code: str) -> str:
        """执行多 Agent 代码审查"""
        print(f"{'='*60}")
        print(f"开始代码审查...")
        print(f"{'='*60}\n")

        all_results = {}

        # 1. 每个审查员独立审查
        for key, reviewer in self.reviewers.items():
            print(f"--- {reviewer['name']}审查中 ---")

            prompt = f"""{reviewer['prompt']}

代码如下：
```
{code}
```"""

            result = self.llm_func(prompt)
            all_results[key] = {
                "reviewer": reviewer["name"],
                "result": result
            }
            print(f"审查完成\n")

        # 2. Supervisor 综合报告
        print("--- 协调者汇总报告 ---")
        report = self._generate_report(code, all_results)
        return report

    def _generate_report(self, code: str, results: dict) -> str:
        """生成综合审查报告"""
        findings = "\n\n".join(
            f"## {v['reviewer']}\n{v['result']}"
            for v in results.values()
        )

        prompt = f"""你是一个代码审查协调者。请根据以下各专家的审查意见，生成一份综合审查报告。

## 各专家审查结果
{findings}

## 请生成报告，包含：
1. 总体评价（通过/需修改/严重问题）
2. 严重问题汇总（需要立即修复的）
3. 改进建议汇总（建议优化的）
4. 优先级排序（先修什么后修什么）"""

        return self.llm_func(prompt)


# ---- 模拟 LLM ----

def mock_llm_review(prompt: str) -> str:
    if "安全" in prompt and "SQL注入" in prompt:
        return """### 安全审查结果

严重程度：中

发现的问题：
1. [中] 用户输入未做参数化查询，存在 SQL 注入风险
   修复：使用参数化查询替代字符串拼接
2. [低] 密码在日志中可能泄露
   修复：日志脱敏处理

总结：安全性基本合格，建议修复 SQL 注入风险。"""

    elif "性能" in prompt and "数据库" in prompt:
        return """### 性能审查结果

严重程度：中

发现的问题：
1. [高] getUserOrders 方法中存在 N+1 查询问题
   优化：使用 JOIN 查询替代循环查询
2. [中] 列表查询缺少分页，可能导致 OOM
   优化：添加分页参数

总结：性能有优化空间，重点关注 N+1 问题。"""

    elif "风格" in prompt or "质量" in prompt:
        return """### 代码风格审查结果

严重程度：低

发现的问题：
1. [中] processOrder 函数超过 80 行，建议拆分
2. [低] 变量名 a, b 不够清晰
3. [低] 缺少单元测试

总结：代码风格整体可以，建议优化长函数。"""

    elif "综合" in prompt or "协调" in prompt or "报告" in prompt:
        return """# 代码审查综合报告

## 总体评价：需修改

## 严重问题（需立即修复）
1. SQL 注入风险 → 使用参数化查询
2. N+1 查询问题 → 使用 JOIN 查询

## 改进建议
1. processOrder 函数过长 → 拆分为子函数
2. 列表查询添加分页
3. 添加单元测试

## 修复优先级
1. SQL 注入（安全）
2. N+1 查询（性能）
3. 函数拆分（可维护性）
4. 添加测试（质量）"""

    return "审查完成"


# ---- 运行 ----

if __name__ == "__main__":
    sample_code = """
def get_user_orders(user_id):
    query = f"SELECT * FROM orders WHERE user_id = {user_id}"
    orders = db.execute(query)
    result = []
    for order in orders:
        items = db.execute(f"SELECT * FROM items WHERE order_id = {order.id}")
        order.items = items
        result.append(order)
    return result
"""

    team = CodeReviewTeam(llm_func=mock_llm_review)
    report = team.review(sample_code)
    print(report)
```

---

## 6. 常见误区

| 误区 | 真相 |
|------|------|
| "多 Agent 一定比单 Agent 好" | 多 Agent 成本更高（多个 LLM 调用），简单任务用单 Agent 就够 |
| "Agent 越多越好" | Agent 越多协调越复杂、成本越高，3-5 个通常是最佳范围 |
| "多 Agent 系统更智能" | 多 Agent 的优势是"专业化"，不是"更智能" |
| "AutoGPT/BabyAGI 可以完全自主" | 实际上极度不稳定，2026 年主流是"半自主 + 人工确认" |
| "所有 Agent 都要用自己的 LLM 调用" | 可以共享 LLM，区别在于 Prompt 和工具不同 |

---

## 👉 这一阶段你应该掌握的能力

1. **理解三种多 Agent 架构模式**：协作式、层级式、辩论式，以及各自的适用场景
2. **理解 AutoGPT / BabyAGI 的核心循环**：任务创建 → 排序 → 执行 → 新任务
3. **能实现 Agent 通信机制**：消息传递、共享内存
4. **能设计一个简单的多 Agent 系统**：Supervisor + Workers 模式
5. **能判断什么时候需要多 Agent**：复杂任务、多专业领域、需要互相校验

---

> **下一阶段预告：** 第六阶段将设计并实现一个完整的 Agent 项目——包含系统架构、模块拆分、技术选型、可运行代码。
