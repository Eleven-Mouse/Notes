# Stage 0：AI / LLM 基础知识详解

> 本文档是 Stage 1~7 的前置知识补充。如果你对 LLM 底层原理、Token 机制、Embedding、Prompt Engineering 等基础概念还不够扎实，先看这份。
> 原则不变：**面向面试，不讲科普，只讲"你必须知道什么、怎么说"。**

---

## 一、LLM（大语言模型）底层原理

### 1.1 LLM 到底在做什么（面试极简版）

> **LLM 本质上是一个"文字接龙机器"——给它一段文本，它预测下一个最可能出现的 Token。**

```
输入: "今天天气"
LLM 预测下一个 Token: "很"（概率 0.35）
                     "不错"（概率 0.25）
                     "真好"（概率 0.15）
                     ...

选择概率最高的 → "今天天气很"
继续预测下一个 → "今天天气很好"
再继续 → "今天天气很好，适合出门。"
```

这就是 **自回归生成（Autoregressive Generation）**：逐个 Token 生成，每次只看之前的内容预测下一个。

**面试类比：** 类似 Java 中的 `StringBuilder.append()`——每次追加一个字符/词，根据已有内容决定追加什么。

### 1.2 Transformer 架构（面试怎么说）

LLM 的底层架构是 Transformer（2017 年 Google 提出）。

**面试只需讲两个核心概念：**

#### ① Self-Attention（自注意力机制）

> 模型在处理每个词时，会"关注"上下文中所有其他词，自动判断哪些词和当前词关系最大。

```
句子："苹果公司今天发布了新品"

处理"苹果"这个词时：
  关注"公司"（权重 0.8）→ 理解"苹果"是公司名，不是水果
  关注"发布"（权重 0.5）
  关注"新品"（权重 0.4）
```

**面试一句话：** "Self-Attention 让模型理解上下文关系，比如区分'苹果'是水果还是公司。"

#### ② 位置编码（Positional Encoding）

> Transformer 本身没有顺序概念（不像 RNN 按顺序读），所以需要额外告诉模型每个词的位置。

**面试一句话：** "Transformer 是并行处理所有词的，位置编码告诉模型每个词在第几个位置。"

### 1.3 主流 LLM 模型对比（2026 面试版）

| 模型 | 厂商 | 上下文窗口 | 特点 | 适用场景 |
|------|------|-----------|------|---------|
| **GPT-4o** | OpenAI | 128K | 综合能力最强、工具调用好 | 复杂推理、多模态 |
| **GPT-4o-mini** | OpenAI | 128K | 便宜、快 | 简单分类/提取 |
| **Claude Opus 4.7** | Anthropic | 200K | 长文本理解强、安全对齐好 | 复杂分析、代码生成 |
| **Claude Sonnet 4.6** | Anthropic | 200K | 性价比高 | 通用 Agent 任务 |
| **Claude Haiku 4.5** | Anthropic | 200K | 极快、极便宜 | 意图分类、简单抽取 |
| **通义千问-Max** | 阿里 | 128K | 国产最强、中文好 | 国内部署首选 |
| **通义千问-Plus** | 阿里 | 128K | 性价比高 | 通用任务 |
| **通义千问-Turbo** | 阿里 | 128K | 极快、便宜 | 意图分类 |
| **DeepSeek-V3** | DeepSeek | 128K | 开源、推理强 | 私有化部署 |
| **GLM-4** | 智谱 | 128K | 开源、中文好 | 私有化部署 |

**面试怎么说选型：**

> 模型选型看三个维度：
> 1. **合规性**：国内业务优先国产模型（通义/DeepSeek/GLM），海外业务可用 OpenAI/Claude
> 2. **任务复杂度**：简单分类用小模型（Haiku/Turbo），复杂推理用大模型（Opus/GPT-4o）
> 3. **成本**：生产环境分级调用——80% 请求走小模型，20% 复杂请求走大模型

### 1.4 训练过程的三个阶段（面试加分）

```
阶段一：预训练（Pre-training）
  海量文本 → 学习语言规律 → 得到"基座模型"
  类比：读完了整个图书馆，学会了语言能力
  产出：GPT-4 base、Llama base

阶段二：指令微调（SFT - Supervised Fine-Tuning）
  高质量问答对 → 学会"按指令回答" → 得到"对话模型"
  类比：学会了"有人问你问题，你要回答"
  产出：GPT-4、Claude（对话版）

阶段三：人类反馈强化学习（RLHF）
  人类标注偏好 → 学会"怎么回答更好" → 得到"对齐模型"
  类比：学会了"什么该说、什么不该说"
  产出：ChatGPT、Claude（安全对齐版）
```

**面试一句话：** "LLM 的训练分三步——预训练学语言、SFT 学对话、RLHF 学对齐，所以它既有知识能力，也有对话能力，还知道什么该说什么不该说。"

---

## 二、Token 机制（必须彻底搞懂）

### 2.1 什么是 Token

> **Token 是 LLM 处理文本的最小单位。不等于字符，也不等于单词。**

```
英文: "Hello world"
  Token: ["Hello", " world"]  → 2 个 Token

中文: "你好世界"
  Token: ["你好", "世界"]  → 2 个 Token（通常 1 个中文字 ≈ 1~2 个 Token）

代码: "function hello()"
  Token: ["function", " hello", "(" , ")"]  → 4 个 Token
```

**关键规则：**
- 1 个英文单词 ≈ 1~2 个 Token
- 1 个中文字 ≈ 1~2 个 Token
- 1 个 Token 平均 ≈ 4 个英文字符 或 0.75 个英文单词

### 2.2 Token 为什么重要（三个维度）

| 维度 | 说明 |
|------|------|
| **计费** | API 按 Token 计费（输入 $X/百万Token，输出 $Y/百万Token） |
| **上下文窗口** | 模型一次能处理的最大 Token 数（如 GPT-4o 是 128K） |
| **性能** | Token 越多，推理越慢，成本越高 |

### 2.3 上下文窗口（Context Window）

> **上下文窗口 = 模型一次能"看到"的最大 Token 数。**

```
模型上下文窗口示例：

Claude Opus 4.7:   200K Token ≈ 150,000 英文单词 ≈ 一本中等长度的书
GPT-4o:            128K Token ≈ 100,000 英文单词
通义千问-Max:       128K Token ≈ 100,000 英文单词

200K Token 的分配（以 Agent 场景为例）：
┌────────────────────────────────────────────┐
│ System Prompt     │  1K ~ 5K Token        │
│ 对话历史（Memory） │  10K ~ 50K Token      │
│ 工具定义           │  2K ~ 10K Token       │
│ RAG 检索结果       │  5K ~ 20K Token       │
│ 用户输入           │  0.5K ~ 2K Token      │
│ LLM 输出预留       │  2K ~ 4K Token        │
│ ──────────────────────────────────────     │
│ 总计               │  20K ~ 91K Token      │
└────────────────────────────────────────────┘
```

**面试关键理解：**

> 上下文窗口是稀缺资源。在 Agent 场景中，System Prompt + 工具定义 + 对话历史 + RAG 结果 + 用户输入 + 模型输出，全部共享这个窗口。所以 Memory 管理、RAG 结果裁剪、Prompt 精简，本质上都是"在有限的 Token 预算内做最优分配"。

### 2.4 Token 计费（面试要会算）

```
2026 年主流模型价格（近似）：

模型              输入价格            输出价格
GPT-4o           $2.5/百万Token      $10/百万Token
GPT-4o-mini      $0.15/百万Token     $0.6/百万Token
Claude Sonnet 4.6 $3/百万Token       $15/百万Token
Claude Haiku 4.5  $0.8/百万Token      $4/百万Token
通义千问-Turbo    ¥0.3/百万Token      ¥0.6/百万Token

面试计算题：
  一次 Agent 对话，平均 5 轮 ReAct 循环
  每轮输入约 2000 Token，输出约 500 Token
  使用 Claude Sonnet 4.6

  输入成本: 5 × 2000 × $3/百万 = $0.03
  输出成本: 5 × 500 × $15/百万 = $0.0375
  单次对话成本 ≈ $0.07（约 ¥0.5）

  日均 5 万次对话 → ¥25,000/天 → ¥75 万/月

  所以 Agent 的成本控制是工程上的核心挑战！
```

### 2.5 Token 计数（Java 实现）

```java
// Token 计数——Agent 成本控制的基础
@Component
public class TokenCounter {

    // 精确计数需要调用模型的 Tokenizer
    // 简单估算：英文 1 词 ≈ 1.3 Token，中文 1 字 ≈ 1.5 Token
    public int estimateTokens(String text) {
        int chineseCount = 0;
        int englishWordCount = 0;

        for (char c : text.toCharArray()) {
            if (Character.toString(c).matches("[\\u4e00-\\u9fa5]")) {
                chineseCount++;
            }
        }
        englishWordCount = text.split("\\s+").length;

        return (int) (chineseCount * 1.5 + englishWordCount * 1.3);
    }

    // 检查是否超出 Token 预算
    public boolean withinBudget(String prompt, int maxTokens) {
        return estimateTokens(prompt) < maxTokens;
    }
}
```

---

## 三、Embedding（向量化）—— RAG 和 Memory 的底层基础

### 3.1 什么是 Embedding

> **Embedding = 把文本转换成一个高维数字向量（数组），让"语义相似"的文本在向量空间中距离更近。**

```
"年假政策"     → [0.12, -0.34, 0.56, 0.78, ...]  (1536 维)
"休假制度"     → [0.11, -0.32, 0.55, 0.79, ...]  (距离很近！语义相似)
"订机票"       → [0.89, 0.23, -0.45, 0.12, ...]  (距离很远！语义不同)
```

### 3.2 为什么 Embedding 有效

```
向量空间可视化（简化为 2 维）：

        ┌──────────────────────────────────┐
        │                                  │
        │    ●年假政策  ●休假制度           │   ← 聚类在一起（语义相近）
        │       ●请假流程                   │
        │                                  │
        │                                  │
        │              ●订机票  ●买火车票    │   ← 另一个聚类
        │                                  │
        │    ●退款流程  ●退货政策            │   ← 又一个聚类
        │                                  │
        │                                  │
        │              ●写代码              │   ← 孤立（和其他都不相关）
        │                                  │
        └──────────────────────────────────┘

关键能力：语义理解
  "年假政策"和"休假制度"字面不同，但语义接近
  Embedding 能捕捉这种语义关系
  传统关键词搜索（ES BM25）做不到！
```

### 3.3 Embedding 模型选型

| 模型 | 维度 | 特点 | 适用场景 |
|------|------|------|---------|
| OpenAI `text-embedding-3-small` | 1536 | 均衡 | 通用英文场景 |
| OpenAI `text-embedding-3-large` | 3072 | 精度高 | 精度要求高 |
| BGE-large-zh | 1024 | 中文效果好 | 中文场景首选 |
| M3E-large | 1024 | 中文开源、效果好 | 私有化部署 |
| 通义文本向量 | 1536 | 阿里云集成方便 | 阿里云生态 |

**面试怎么说：**

> Embedding 模型的选型原则：中文场景优先 BGE / M3E，英文场景用 OpenAI，需要私有化部署用开源模型。维度越高精度越好，但存储和检索成本也越高。实际项目中 1024 维通常是性价比最优的选择。

### 3.4 相似度计算

```java
// 余弦相似度——最常用的向量相似度算法
public class VectorUtils {

    /**
     * 计算两个向量的余弦相似度
     * 结果范围 [-1, 1]，越接近 1 表示越相似
     */
    public static double cosineSimilarity(float[] a, float[] b) {
        double dotProduct = 0.0;
        double normA = 0.0;
        double normB = 0.0;

        for (int i = 0; i < a.length; i++) {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }

        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}

// 相似度阈值设定：
// > 0.95  → 几乎相同
// > 0.85  → 高度相关（RAG 通常用这个阈值）
// > 0.70  → 有一定相关性
// < 0.50  → 基本不相关
```

---

## 四、Prompt Engineering（提示词工程）

### 4.1 为什么 Prompt 重要

> **同样一个 LLM，Prompt 写得好效果 90 分，写得差效果 50 分。Prompt 就是给 LLM 的"需求文档"。**

类比到 Java：Prompt 就像方法签名 + 注释——描述越清晰，实现越准确。

### 4.2 核心 Prompt 技巧（面试必知）

#### ① System Prompt（系统提示词）

```
System Prompt 定义 LLM 的"角色"和"行为边界"。

示例：
┌─────────────────────────────────────────────────────┐
│ System:                                              │
│ 你是一个电商客服助手。你的职责是：                      │
│ 1. 回答用户关于订单、物流、退换货的问题                  │
│ 2. 只基于提供的工具返回数据回答，不要编造信息            │
│ 3. 如果无法确定答案，请回复"我需要进一步确认"           │
│ 4. 涉及退款、取消订单等操作时，必须先征得用户确认       │
│ 5. 回答要简洁专业，不要超过 200 字                     │
└─────────────────────────────────────────────────────┘
```

**面试一句话：** "System Prompt 是 LLM 的'岗位说明书'，定义了角色、能力范围和行为约束。"

#### ② Few-Shot（少样本示例）

```
给 LLM 看几个"输入→输出"的例子，让它理解你期望的格式。

示例：
┌─────────────────────────────────────────────────────┐
│ 请将用户的自然语言转换为 SQL 查询：                     │
│                                                      │
│ 用户：查一下最近7天的订单总数                           │
│ SQL：SELECT COUNT(*) FROM orders WHERE created_at    │
│      > DATE_SUB(NOW(), INTERVAL 7 DAY)               │
│                                                      │
│ 用户：上海地区的用户消费总额                            │
│ SQL：SELECT SUM(amount) FROM orders WHERE region     │
│      = '上海'                                         │
│                                                      │
│ 用户：{用户实际输入}                                   │
│ SQL：                                                │
└─────────────────────────────────────────────────────┘
```

**面试一句话：** "Few-Shot 就是通过给几个示例来教 LLM 输出格式，比纯描述更直观。"

#### ③ Chain-of-Thought（思维链）

```
让 LLM "展示思考过程"，而不是直接给答案。

普通 Prompt：
  "399 × 407 = ?"
  → LLM 可能算错

CoT Prompt：
  "请一步一步计算 399 × 407"
  → LLM 输出思考过程：
     "先把 399 拆成 400 - 1
      400 × 407 = 162800
      1 × 407 = 407
      162800 - 407 = 162393"
  → 正确率大幅提升

面试一句话：**CoT 是 ReAct 的理论基础**
  CoT 让 LLM "想清楚再说"
  ReAct 在此基础上加了"想清楚再行动"
```

#### ④ 结构化输出（Structured Output）

```
要求 LLM 输出 JSON 等结构化格式。

Prompt 示例：
┌─────────────────────────────────────────────────────┐
│ 请分析以下用户的情感倾向，以 JSON 格式输出：             │
│                                                      │
│ 输出格式：                                            │
│ {                                                    │
│   "sentiment": "positive" | "negative" | "neutral",  │
│   "confidence": 0.0-1.0,                             │
│   "keywords": ["关键词1", "关键词2"],                  │
│   "summary": "一句话总结"                             │
│ }                                                    │
└─────────────────────────────────────────────────────┘

工程上的保障措施（三层）：
  1. Prompt 明确要求 JSON 格式 + 给出示例
  2. 开启模型的 JSON Mode（如果支持）
  3. 代码层做 JSON 解析兜底 + 重试
```

### 4.3 Prompt 设计原则（面试总结版）

| 原则 | 说明 | 反面案例 |
|------|------|---------|
| **具体** | 明确告诉 LLM 要做什么 | "帮我处理一下"（太模糊） |
| **有约束** | 明确边界和格式 | "随便回答"（没约束） |
| **有示例** | 给 Few-Shot 示例 | 只描述不给例子 |
| **有角色** | 设定 System Prompt | 不设定角色 |
| **有兜底** | 告诉它"不知道就说不知道" | 不设定兜底行为 |

### 4.4 Prompt 模板设计（Java 实现）

```java
// Prompt 模板管理——类似 Thymeleaf 模板引擎
public class PromptTemplateManager {

    // Agent System Prompt 模板
    private static final String AGENT_SYSTEM_PROMPT = """
        你是一个{role}。

        你的职责是：{responsibilities}

        可用工具：
        {tools_description}

        行为规则：
        1. 只使用上述工具获取数据，不要编造信息
        2. 每次只调用一个工具
        3. 如果工具返回错误，尝试换一种方式或告知用户
        4. 涉及{high_risk_operations}等操作时，必须先征得用户确认
        5. 回答控制在{max_words}字以内

        输出格式：严格按 JSON 格式输出
        """;

    public String buildAgentPrompt(Map<String, String> variables) {
        String result = AGENT_SYSTEM_PROMPT;
        for (Map.Entry<String, String> entry : variables.entrySet()) {
            result = result.replace("{" + entry.getKey() + "}", entry.getValue());
        }
        return result;
    }
}
```

---

## 五、LLM 调用的工程细节

### 5.1 调用方式：同步 vs 流式

```
同步调用（Sync）：
  用户请求 → 等待 LLM 完整输出 → 一次性返回
  优点：实现简单
  缺点：用户要等很久（可能 10~30 秒）
  适用：后台任务、Agent 内部调用

流式调用（Streaming / SSE）：
  用户请求 → LLM 生成一个 Token 就返回一个 → 逐步展示给用户
  优点：用户感知快（看到"正在输入"效果）
  缺点：实现复杂，需要处理流式响应
  适用：面向用户的对话界面
```

```java
// 流式调用的 Java 实现
@Service
public class LLMStreamService {

    public Flux<String> chatStream(String prompt) {
        return Flux.create(sink -> {
            llmClient.chatStream(prompt, new StreamCallback() {
                @Override
                public void onToken(String token) {
                    sink.next(token);  // 每收到一个 Token 就推给前端
                }

                @Override
                public void onComplete() {
                    sink.complete();
                }

                @Override
                public void onError(Exception e) {
                    sink.error(e);
                }
            });
        });
    }
}
```

### 5.2 温度参数（Temperature）

```
Temperature 控制输出的随机性：

Temperature = 0   → 最确定，每次输出几乎一样（适合：代码生成、数据提取）
Temperature = 0.7 → 适度随机，有创造性但不太离谱（适合：对话、写作）
Temperature = 1.0 → 较随机，更有创造性（适合：头脑风暴、创意生成）
Temperature = 1.5+ → 很随机，可能胡言乱语

面试怎么说：
  "Agent 场景中，工具调用相关的 Prompt 我会设 Temperature=0，
   确保决策尽量稳定。只有需要创造性回答的场景才会提高 Temperature。"
```

### 5.3 超时与重试

```java
// LLM 调用的超时与重试策略
@Service
public class RobustLLMClient {

    private static final int MAX_RETRIES = 3;
    private static final long INITIAL_BACKOFF_MS = 1000;

    public LLMResponse chatWithRetry(List<Message> messages) {
        Exception lastException = null;

        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            try {
                return llmClient.chat(messages); // 带超时的 HTTP 调用
            } catch (TimeoutException e) {
                lastException = e;
                long backoff = INITIAL_BACKOFF_MS * (1L << attempt); // 指数退避
                log.warn("LLM调用超时，{}ms后重试（第{}次）", backoff, attempt + 1);
                Thread.sleep(backoff);
            } catch (RateLimitException e) {
                // 限流：等待更长时间
                Thread.sleep(5000);
            }
        }

        throw new LLMCallException("LLM调用失败", lastException);
    }
}
```

### 5.4 常见错误码与处理

| 错误类型 | HTTP 状态码 | 原因 | 处理方式 |
|---------|-----------|------|---------|
| `rate_limit_exceeded` | 429 | 调用频率过高 | 指数退避重试 |
| `context_length_exceeded` | 400 | Token 超出上下文窗口 | 裁剪 Prompt / 摘要压缩 |
| `invalid_api_key` | 401 | API Key 无效 | 检查配置 |
| `model_overloaded` | 503 | 模型过载 | 重试或降级到备用模型 |
| `content_filter` | 400 | 内容触发安全过滤 | 修改 Prompt 或提示用户换说法 |

---

## 六、模型能力边界（面试必知"不能做什么"）

### 6.1 LLM 的根本局限

| 局限 | 说明 | 工程应对 |
|------|------|---------|
| **知识截止** | 训练数据有时间截止日期 | RAG 补充实时知识 |
| **幻觉** | 会编造不存在的信息 | RAG + 引用追溯 + 事实校验 |
| **不擅长数学** | 复杂计算容易出错 | 让 Agent 调用计算器工具 |
| **不能访问网络** | 本身无法联网 | Agent 调用搜索工具 |
| **不能执行代码** | 只能"写"代码不能"运行" | Agent 调用代码执行沙箱 |
| **不可复现** | 同一输入可能不同输出 | Temperature=0 + 重试 + 校验 |
| **上下文有限** | 超出窗口的内容会丢失 | Memory 管理 + 摘要压缩 |

### 6.2 幻觉（Hallucination）详解

```
幻觉的三种类型：

1. 事实性幻觉：
   问："Java 21 有什么新特性？"
   答："Java 21 引入了模式匹配的 sealed interface 语法 sugar..."
   → 可能部分编造，部分真实

2. 引用性幻觉：
   问："RAG 系统中，Smith et al. 2023 的论文说了什么？"
   答："Smith et al. (2023) 提出了 Adaptive Chunking 方法..."
   → 这篇论文可能根本不存在

3. 一致性幻觉：
   同一个问题的两次回答自相矛盾

工程防御（多层防护）：
  ┌──────────────────────────────────────┐
  │ Layer 1: RAG 检索（提供真实数据）      │
  │ Layer 2: Prompt 约束（要求基于数据回答）│
  │ Layer 3: 输出校验（检查是否有引用来源） │
  │ Layer 4: 置信度评估（自评打分）         │
  │ Layer 5: 人工审核（高风险场景）         │
  └──────────────────────────────────────┘
```

---

## 七、向量数据库深入（Stage 5 的工程补充）

### 7.1 向量数据库 vs 传统数据库

```
传统数据库（MySQL / PostgreSQL）：
  查询方式：精确匹配 + 范围查询
  SQL: SELECT * FROM products WHERE name = 'iPhone 15'
  → 精确匹配，要么找到要么找不到

向量数据库（Milvus / Pinecone）：
  查询方式：语义相似度检索
  查询: 找和"好用的手机"语义最接近的商品
  → 返回 iPhone 15、华为 Mate 60、小米 14...
  → 即使文本中没出现"好用的手机"，也能匹配

类比：
  传统数据库 = 字典查词（精确查找）
  向量数据库 = 图书馆员推荐（"你说的我懂，给你推荐类似的"）
```

### 7.2 向量索引算法（面试了解即可）

| 算法 | 原理 | 特点 |
|------|------|------|
| **HNSW** | 多层跳表图，类似六度空间理论 | 查询快、内存占用大、最常用 |
| **IVF** | 先聚类再查，类似 B+ 树的分区 | 适合大规模数据 |
| **PQ（乘积量化）** | 向量压缩，用更少内存存储 | 适合内存受限场景 |

**面试一句话：** "向量数据库底层用 HNSW 或 IVF 算法做近似最近邻搜索（ANN），不需要精确遍历所有向量，所以能在千万级数据中实现毫秒级检索。"

### 7.3 Milvus 核心概念

```
Milvus 数据模型（类比 MySQL）：

Milvus                    MySQL
─────────────            ─────────────
Collection                Table（表）
Field                     Column（列）
Entity / Vector           Row（行）
Index                     Index（索引）

Collection（集合）设计示例：

collection: product_knowledge
├── field: id (INT64, 主键)
├── field: content (VARCHAR, 文本内容)
├── field: source (VARCHAR, 来源文档)
├── field: embedding (FLOAT_VECTOR, 1024维, 向量)
└── index: embedding (HNSW, M=16, efConstruction=256)
```

```java
// Milvus Java SDK 操作示例
@Service
public class MilvusService {

    private final MilvusServiceClient milvusClient;

    // 创建 Collection
    public void createCollection(String collectionName, int dimension) {
        FieldType idField = FieldType.newBuilder()
            .withName("id").withDataType(DataType.Int64)
            .withPrimaryKey(true).withAutoID(true).build();

        FieldType contentField = FieldType.newBuilder()
            .withName("content").withDataType(DataType.VarChar)
            .withMaxLength(2000).build();

        FieldType vectorField = FieldType.newBuilder()
            .withName("embedding").withDataType(DataType.FloatVector)
            .withDimension(dimension).build();

        CreateCollectionParam param = CreateCollectionParam.newBuilder()
            .withCollectionName(collectionName)
            .withDescription("知识库向量存储")
            .withShardsNum(2)
            .addFieldType(idField)
            .addFieldType(contentField)
            .addFieldType(vectorField)
            .build();

        milvusClient.createCollection(param);
    }

    // 插入向量
    public void insert(String collectionName, List<String> contents, List<List<Float>> vectors) {
        List<InsertParam.Field> fields = List.of(
            new InsertParam.Field("content", contents),
            new InsertParam.Field("embedding", vectors)
        );
        milvusClient.insert(InsertParam.newBuilder()
            .withCollectionName(collectionName)
            .withFields(fields).build());
    }

    // 向量检索
    public List<String> search(String collectionName, List<Float> queryVector, int topK) {
        SearchParam param = SearchParam.newBuilder()
            .withCollectionName(collectionName)
            .withVectors(List.of(queryVector))
            .withTopK(topK)
            .withExpr("")  // 可加过滤条件
            .build();

        R<SearchResults> result = milvusClient.search(param);
        return result.getData().getResults(0).stream()
            .map(score -> score.getEntity().get("content").toString())
            .toList();
    }
}
```

---

## 八、LLM API 调用实战（面试手写）

### 8.1 OpenAI API 调用（Java）

```java
// 使用官方 SDK 调用 OpenAI
@Service
public class OpenAIService {

    private final OpenAiClient client;

    // 基础对话
    public String chat(String systemPrompt, String userMessage) {
        ChatCompletionRequest request = ChatCompletionRequest.builder()
            .model("gpt-4o")
            .temperature(0.0)               // Agent 场景用 0
            .maxTokens(2000)                 // 限制输出长度
            .messages(List.of(
                new SystemMessage(systemPrompt),
                new UserMessage(userMessage)
            ))
            .build();

        ChatCompletionResponse response = client.chatCompletion(request);
        return response.getChoices().get(0).getMessage().getContent();
    }

    // 带 Tool Calling 的对话
    public ChatCompletionResponse chatWithTools(
            String systemPrompt,
            String userMessage,
            List<ToolDefinition> tools) {

        ChatCompletionRequest request = ChatCompletionRequest.builder()
            .model("gpt-4o")
            .temperature(0.0)
            .messages(List.of(
                new SystemMessage(systemPrompt),
                new UserMessage(userMessage)
            ))
            .tools(tools.stream().map(this::toOpenAITool).toList())
            .build();

        return client.chatCompletion(request);
        // 如果返回中有 tool_calls，需要执行工具再把结果喂回去
    }
}
```

### 8.2 Claude API 调用（Java）

```java
// 使用 Anthropic Java SDK 调用 Claude
@Service
public class ClaudeService {

    private final AnthropicClient client;

    public String chat(String systemPrompt, String userMessage) {
        MessageCreateRequest request = MessageCreateRequest.builder()
            .model("claude-sonnet-4-20250514")
            .maxTokens(2000)
            .system(systemPrompt)
            .messages(List.of(
                Message.builder()
                    .role(Role.USER)
                    .content(userMessage)
                    .build()
            ))
            .build();

        Message response = client.messages().create(request);
        return response.getContent().get(0).getText();
    }
}
```

### 8.3 通义千问 API 调用（Java）

```java
// 使用阿里云 SDK 调用通义千问
@Service
public class QwenService {

    private final OpenAiClient client; // 通义千问兼容 OpenAI 协议

    public String chat(String systemPrompt, String userMessage) {
        // 基础配置和 OpenAI 完全一样，只是 base_url 和 api_key 不同
        ChatCompletionRequest request = ChatCompletionRequest.builder()
            .model("qwen-max")
            .temperature(0.0)
            .messages(List.of(
                new SystemMessage(systemPrompt),
                new UserMessage(userMessage)
            ))
            .build();

        return client.chatCompletion(request)
            .getChoices().get(0).getMessage().getContent();
    }
}
```

### 8.4 统一 LLM 接口设计（面试加分）

```java
// 统一 LLM 接口——支持多模型切换
public interface LLMProvider {
    String chat(LLMRequest request);
    Flux<String> chatStream(LLMRequest request);
    LLMResponse chatWithTools(LLMRequest request, List<ToolDefinition> tools);
    String getName(); // "openai" / "claude" / "qwen"
}

// OpenAI 实现
@Component("openaiProvider")
public class OpenAIProvider implements LLMProvider { ... }

// Claude 实现
@Component("claudeProvider")
public class ClaudeProvider implements LLMProvider { ... }

// 通义千问实现
@Component("qwenProvider")
public class QwenProvider implements LLMProvider { ... }

// 路由层——根据配置选择 Provider
@Service
public class LLMRouter {

    @Autowired private Map<String, LLMProvider> providers;

    public LLMProvider getProvider(String modelName) {
        // 根据模型名路由到对应的 Provider
        if (modelName.startsWith("gpt")) return providers.get("openaiProvider");
        if (modelName.startsWith("claude")) return providers.get("claudeProvider");
        if (modelName.startsWith("qwen")) return providers.get("qwenProvider");
        throw new IllegalArgumentException("Unknown model: " + modelName);
    }

    // 分级调用：根据任务复杂度选模型
    public LLMProvider selectByComplexity(TaskComplexity complexity) {
        return switch (complexity) {
            case SIMPLE -> providers.get("qwenProvider");    // 用便宜的
            case MEDIUM -> providers.get("openaiProvider");  // 用均衡的
            case COMPLEX -> providers.get("claudeProvider"); // 用最强的
        };
    }
}
```

---

## 九、面试验证清单

看完这份文档，你应该能回答以下问题（遮住答案自测）：

| 问题 | 能否回答 |
|------|---------|
| LLM 本质上在做什么？ | 文字接龙，预测下一个 Token |
| 什么是上下文窗口？为什么重要？ | 一次能处理的最大 Token 数，决定了能"记住"多少内容 |
| Token 怎么计费？中文和英文的区别？ | 按 Token 计费，中文约 1 字 = 1~2 Token |
| Embedding 的作用？ | 把文本变向量，让语义相似的文本距离近 |
| 余弦相似度是什么？ | 衡量两个向量方向的接近程度，范围 [-1, 1] |
| Temperature=0 意味着什么？ | 输出最确定，每次几乎一样 |
| RAG 和 Fine-tuning 的区别？ | RAG 加外挂知识库，Fine-tuning 改模型能力 |
| 什么是幻觉？怎么防？ | LLM 编造信息，用 RAG + 引用追溯 + 事实校验 |
| System Prompt 的作用？ | 定义 LLM 的角色和行为约束 |
| Few-Shot 是什么？ | 给几个输入输出示例教 LLM 格式 |
| 向量数据库和 ES 的区别？ | 向量库做语义检索，ES 做关键词匹配 |
| Agent 场景中 Token 预算怎么分配？ | System + Memory + Tools + RAG + 输出 共享窗口 |
| 流式和同步调用区别？ | 流式逐 Token 返回，同步等完整输出 |
| 怎么实现多模型切换？ | 统一 LLMProvider 接口 + 路由层 |

---

> **Stage 0 基础知识详解完毕。这份是 Stage 1~7 所有内容的前置基础，建议在阅读其他 Stage 前先确保这里的每个知识点都理解。**
