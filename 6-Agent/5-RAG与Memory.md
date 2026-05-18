# Stage 5：RAG + Memory（高频面试点 · 详细版）

> 目标：能讲清 RAG 全链路原理、向量库选型、Embedding 模型、Memory 分层设计和 Token 管理。
> 面试定位：RAG 是 Agent 项目中最常见的落地场景，几乎必问。

---

## 1. RAG（Retrieval-Augmented Generation）

### 1.1 一句话理解

> **RAG = 先从知识库检索相关内容，再把检索结果塞给 LLM 生成回答。**

解决的问题：LLM 的知识有截止日期，且不了解你的私有数据（公司文档、业务知识库等）。

### 1.2 为什么需要 RAG（对比 Fine-tuning）

| 维度 | Fine-tuning（微调） | RAG（检索增强） |
|------|-------------------|----------------|
| 成本 | 高（训练数据 + GPU 时间） | 低（只需搭建检索流程） |
| 时效性 | 差（每次更新要重新微调） | 好（知识库更新后立即可用） |
| 可控性 | 差（模型内部黑盒） | 好（可追溯引用了哪些文档） |
| 私有数据 | 需上传到训练平台 | 数据保留在本地，不泄露 |
| 幻觉控制 | 一般 | 更好（基于真实文档生成） |
| 适用场景 | 改变模型行为/风格 | 知识更新快/私有数据查询 |

**面试怎么说：**

> Fine-tuning 是"改变模型的能力"，RAG 是"给模型加外挂知识库"。
> 90% 的企业场景用 RAG 就够了——更快、更便宜、更可控。
> Fine-tuning 适合需要改变模型行为风格的场景（比如让模型学会特定语气说话）。
> 最佳实践是**两者结合**：Fine-tuning 调能力 + RAG 补知识。

### 1.3 RAG 全链路流程（面试必须能画）

```
┌──────────────────────────────────────────────────────────────┐
│                       RAG 全链路                               │
│                                                               │
│  ═══════ 阶段一：离线索引（提前做，不在用户请求路径上） ═══════ │
│                                                               │
│  company_policy.pdf                                           │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐   │
│  │ 文档加载  │───▶│   文档分片     │───▶│    向量化         │   │
│  │ PDF/Word │    │ 按段落切分     │    │ Embedding 模型    │   │
│  │ Markdown │    │ 每片500字     │    │ [0.12,-0.34,...] │   │
│  └──────────┘    └──────────────┘    └────────┬─────────┘   │
│                                               │               │
│                                               ▼               │
│                                      ┌──────────────────┐    │
│                                      │   向量数据库       │    │
│                                      │  Milvus / Pinecone │    │
│                                      └──────────────────┘    │
│                                                               │
│  ═══════ 阶段二：在线检索（用户请求时执行） ═══════            │
│                                                               │
│  用户问题："年假政策是什么？"                                    │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────────┐    ┌──────────────────┐                    │
│  │ 问题向量化     │───▶│  向量相似度检索    │                    │
│  │ Embedding模型  │    │  Top-K 最近邻     │                    │
│  └──────────────┘    └────────┬─────────┘                    │
│                               │                               │
│                               ▼                               │
│                      ┌──────────────────┐                    │
│                      │  Reranker 重排    │ （可选，提精度）     │
│                      │  精选 Top-3~5     │                    │
│                      └────────┬─────────┘                    │
│                               │                               │
│  ═══════ 阶段三：增强生成 ═══════                              │
│                               │                               │
│                               ▼                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Prompt = System指令                                   │   │
│  │        + "请基于以下参考资料回答："                       │   │
│  │        + 检索到的文档片段 1                               │   │
│  │        + 检索到的文档片段 2                               │   │
│  │        + 检索到的文档片段 3                               │   │
│  │        + "用户问题：年假政策是什么？"                      │   │
│  └──────────────────────┬───────────────────────────────┘   │
│                         │                                     │
│                         ▼                                     │
│                    LLM 生成回答                                 │
│                    "根据公司政策，员工年假为..."                   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 1.4 RAG 关键技术点详解

#### ① 文档分片（Chunking）—— 决定 RAG 效果的上限

| 策略 | 说明 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| 固定大小 | 每 500 字切一片 | 简单 | 可能切断语义 | 通用 |
| 按段落/章节 | 按文档结构切 | 保持语义完整 | 文档结构不统一时难处理 | 结构化文档 |
| 语义分片 | 用 LLM 判断语义断点 | 精度高 | 成本高 | 精度要求高 |
| 递归分片 | 按分隔符层级切（段落→句子→字符） | 兼顾语义和大小 | 参数需调优 | **最常用（推荐）** |
| 重叠分片 | 相邻片段有 10%~20% 重叠 | 避免关键信息被切断 | 存储冗余 | **通用加分项** |

```python
# LangChain 中的推荐分片方式
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,       # 每片 500 字符
    chunk_overlap=50,     # 重叠 50 字符
    separators=["\n\n", "\n", "。", "！", "？", "，", " ", ""],
    # 按优先级尝试分隔符
)
chunks = splitter.split_documents(docs)
```

```java
// Java 实现思路
public class DocumentSplitter {
    public List<TextChunk> split(String text, int chunkSize, int overlap) {
        List<TextChunk> chunks = new ArrayList<>();
        int start = 0;
        while (start < text.length()) {
            int end = Math.min(start + chunkSize, text.length());
            chunks.add(new TextChunk(text.substring(start, end), start, end));
            start = end - overlap;  // 重叠部分
        }
        return chunks;
    }
}
```

#### ② Embedding（向量化）

**将文本转换为高维向量，语义相近的文本向量距离更近。**

```
"年假政策" → Embedding → [0.12, -0.34, 0.56, ..., 0.78]  (768维)
"休假制度" → Embedding → [0.11, -0.32, 0.55, ..., 0.77]  (非常接近！)

"数据库优化" → Embedding → [0.91, 0.23, -0.45, ..., -0.12]  (完全不同)

→ 向量距离近 = 语义相似 → 检索时能匹配到语义相关的内容
```

**常用 Embedding 模型：**

| 模型 | 特点 | 适用场景 |
|------|------|---------|
| OpenAI `text-embedding-3-small` | 性价比高 | 通用英文 |
| OpenAI `text-embedding-3-large` | 效果最好 | 高精度需求 |
| BGE-large-zh | 中文效果最好 | 中文场景（推荐） |
| M3E | 中文多任务 | 中文通用 |
| 通义文本向量 | 阿里云部署合规 | 国内部署 |

#### ③ 向量检索

**相似度算法：**

| 算法 | 说明 | 特点 |
|------|------|------|
| 余弦相似度 | 向量夹角的余弦值 | **最常用**，归一化后的内积 |
| 欧氏距离 | 向量间的直线距离 | 对绝对大小敏感 |
| 内积（Dot Product） | 向量点乘 | 速度快 |

**检索类型：**

| 类型 | 说明 |
|------|------|
| 精确检索（KNN） | 遍历所有向量，结果精确但慢 |
| 近似检索（ANN） | 用索引加速，牺牲少量精度换速度 |
| MMR（最大边际相关性） | 兼顾相关性和多样性，避免结果重复 |

#### ④ Reranker（重排序—— RAG 效果提升的关键）

> **为什么需要 Reranker：** 向量检索是"粗筛"，Top-10 里可能有低质量结果。Reranker 用交叉编码器做"精排"，只取最相关的 3~5 条。

```
向量检索（粗筛）→ Top-10 个文档片段
        │
        ▼
Reranker（精排）
  - 用更精确的模型计算问题和每个片段的相关性
  - 重新排序
        │
        ▼
精选 Top-3 个最相关的片段 → 喂给 LLM
```

**常用 Reranker：**

| Reranker | 特点 |
|----------|------|
| Cohere Rerank | API 调用，效果好 |
| BGE-Reranker | 开源，中文效果好 |
| Jina Reranker | 开源，轻量级 |

### 1.5 向量数据库选型（面试必问）

| 向量库 | 特点 | 适用场景 | Java 生态 |
|--------|------|---------|----------|
| **Milvus** | 开源、高性能、分布式 | 大规模生产（千万级） | Java SDK 完善 |
| **Pinecone** | 全托管 SaaS | 快速上线、不想运维 | REST API |
| **Weaviate** | 开源、混合检索 | 向量+关键词混合 | Java SDK |
| **Qdrant** | Rust 实现、高性能 | 性能敏感场景 | REST/gRPC |
| **ES 8.x** | 传统搜索加向量能力 | 已有 ES 基础设施 | Java 原生支持 |
| **Redis** | RediSearch 向量模块 | 已有 Redis、量级不大 | Java 原生支持 |

**面试怎么选型：**

> - 已有 ES 且数据量不大 → 先用 ES 向量功能做 MVP
> - 千万级以上 → Milvus 分布式向量存储
> - 快速验证 → Pinecone 全托管
> - 需要混合检索（向量+关键词） → Weaviate 或 ES 8.x

---

## 2. RAG 效果优化（面试深入追问）

### 2.1 RAG 效果不好的排查清单

```
RAG 效果不好？按顺序排查：

1. 分片质量
   → 片太大（检索到无关内容）或太小（上下文不完整）
   → 解决：递归分片 + 10% 重叠

2. Embedding 模型
   → 通用模型在垂直领域效果差
   → 解决：用领域数据微调 Embedding，或用 BGE/M3E 中文优化模型

3. 检索方式
   → 纯向量检索可能漏掉关键词精确匹配
   → 解决：混合检索（向量 + BM25） + RRF 融合排序

4. 结果排序
   → Top-K 里可能有低质量结果
   → 解决：加 Reranker 做二次精排

5. Prompt 设计
   → "请回答" vs "请基于以下资料回答，如果资料中没有请说不知道"
   → 解决：加明确的约束指令

6. 文档质量
   → 源文档本身就是乱的
   → 解决：入库前做数据清洗
```

### 2.2 混合检索（Hybrid Search）—— 面试加分

```
用户问题："年假政策"

向量检索：找到语义相关的片段
  → "公司休假制度规定..."  (0.89)
  → "节假日安排如下..."     (0.82)
  → "病假申请流程..."       (0.65)

BM25 关键词检索：找到关键词匹配的片段
  → "年假政策：工作满1年5天..."  (8.2)
  → "年假可以分段使用..."         (6.5)
  → "病假需要提供证明..."         (2.1)

RRF 融合排序：
  → "年假政策：工作满1年5天..."  ← 关键词和语义都匹配
  → "公司休假制度规定..."         ← 语义匹配
  → "年假可以分段使用..."         ← 关键词匹配
```

```java
// 混合检索的 Java 实现
public class HybridSearchService {

    public List<TextChunk> search(String query) {
        // 1. 向量检索
        float[] queryVector = embeddingClient.embed(query);
        List<ScoredChunk> vectorResults = milvus.search(queryVector, 10);

        // 2. BM25 关键词检索
        List<ScoredChunk> bm25Results = elasticsearch.bm25Search(query, 10);

        // 3. RRF 融合排序
        return reciprocalRankFusion(vectorResults, bm25Results, 5);
    }

    // Reciprocal Rank Fusion
    private List<TextChunk> reciprocalRankFusion(
        List<ScoredChunk> listA, List<ScoredChunk> listB, int topK
    ) {
        Map<String, Double> scores = new HashMap<>();
        int k = 60; // RRF 常数

        for (int i = 0; i < listA.size(); i++) {
            scores.merge(listA.get(i).getId(), 1.0 / (k + i + 1), Double::sum);
        }
        for (int i = 0; i < listB.size(); i++) {
            scores.merge(listB.get(i).getId(), 1.0 / (k + i + 1), Double::sum);
        }

        return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(topK)
            .map(e -> getChunkById(e.getKey()))
            .toList();
    }
}
```

---

## 3. Memory（记忆系统）

### 3.1 为什么 Agent 必须有 Memory

```
没有 Memory 的 Agent：
  用户："帮我查一下订单 123456"
  Agent：调用工具 → "订单123456，已发货"
  用户："那个物流呢？"
  Agent："什么物流？" ← 不记得刚才在聊什么

有 Memory 的 Agent：
  用户："帮我查一下订单 123456"
  Agent：调用工具 → "订单123456，已发货"
  用户："那个物流呢？"
  Agent：理解"那个"= 订单123456 → 调用物流查询 → "顺丰已到北京"
```

### 3.2 Memory 三层架构

```
┌─────────────────────────────────────────────────────┐
│            Agent Memory 分层架构                      │
│                                                      │
│  L1: 工作记忆（Working Memory）                       │
│  ┌─────────────────────────────────────────────┐     │
│  │  存在哪：LLM 上下文窗口（拼在 Prompt 里）      │     │
│  │  存什么：最近几轮对话 + 当前任务步骤             │     │
│  │  大小限制：受 Token 窗口限制                    │     │
│  │  生命周期：单次请求                              │     │
│  │  Java类比：方法参数 + 局部变量                   │     │
│  │  技术：直接拼接到 Prompt 字符串                  │     │
│  └─────────────────────────────────────────────┘     │
│                    ↓ 超出的部分 ↓                      │
│  L2: 短期记忆（Short-term Memory）                    │
│  ┌─────────────────────────────────────────────┐     │
│  │  存在哪：Redis                                  │     │
│  │  存什么：完整会话历史 + 摘要                     │     │
│  │  大小限制：无硬限制，Token 预算管理              │     │
│  │  生命周期：一次会话（24h 过期）                   │     │
│  │  Java类比：Redis Session                        │     │
│  │  技术：Redis List + 自动过期                     │     │
│  └─────────────────────────────────────────────┘     │
│                    ↓ 需要跨会话 ↓                      │
│  L3: 长期记忆（Long-term Memory）                     │
│  ┌─────────────────────────────────────────────┐     │
│  │  存在哪：向量数据库 + MySQL                      │     │
│  │  存什么：用户偏好、历史精华、知识积累             │     │
│  │  大小限制：无限制                                │     │
│  │  生命周期：永久，跨会话                          │     │
│  │  Java类比：MySQL + Elasticsearch                │     │
│  │  技术：Milvus 向量检索 + MySQL 结构化存储        │     │
│  └─────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘
```

### 3.3 Memory 的 Java 工程实现

```java
@Service
public class AgentMemoryService {

    private final RedisTemplate<String, String> redis;
    private final VectorStore memoryVectorStore;
    private final EmbeddingClient embeddingClient;
    private final LLMClient llmClient;

    private static final int MEMORY_TOKEN_BUDGET = 6000;
    private static final int WINDOW_SIZE = 10;

    // ========== L1 + L2: 保存短期记忆 ==========
    public void saveShortTerm(String sessionId, Message message) {
        String key = "agent:memory:" + sessionId;
        redis.opsForList().rightPush(key, toJson(message));
        redis.expire(key, 24, TimeUnit.HOURS);
    }

    // ========== L1: 构建给 LLM 的上下文 ==========
    public String buildContext(String sessionId) {
        List<Message> allMessages = getAllMessages(sessionId);

        if (estimateTokens(allMessages) <= MEMORY_TOKEN_BUDGET) {
            return formatMessages(allMessages);
        }

        // 滑动窗口：保留最近 N 轮
        List<Message> recent = allMessages.subList(
            Math.max(0, allMessages.size() - WINDOW_SIZE * 2), allMessages.size()
        );

        // 摘要压缩：旧消息生成摘要
        List<Message> old = allMessages.subList(0, allMessages.size() - WINDOW_SIZE * 2);
        String summary = getOrCreateSummary(sessionId, old);

        return "对话摘要：" + summary + "\n\n最近对话：\n" + formatMessages(recent);
    }

    // ========== L3: 保存长期记忆 ==========
    public void saveLongTerm(String userId, String content) {
        // 结构化存 MySQL
        memoryRepository.save(MemoryEntity.builder()
            .userId(userId).content(content).createdAt(LocalDateTime.now()).build());

        // 向量化存 Milvus
        float[] vector = embeddingClient.embed(content);
        memoryVectorStore.upsert(userId, content, vector);
    }

    // ========== L3: 检索相关记忆 ==========
    public String retrieveRelevant(String query, String userId) {
        float[] vector = embeddingClient.embed(query);
        List<TextChunk> results = memoryVectorStore.search(vector, 5, userId);
        return results.stream().map(TextChunk::getText).collect(Collectors.joining("\n"));
    }
}
```

### 3.4 Token 管理（Memory 的核心挑战）

```
Token 预算分配（以 128K 上下文为例）：

  System Prompt：         ~1K Token  （固定）
  工具描述（10个工具）：    ~3K Token  （固定）
  RAG 检索结果：           ~4K Token  （每次检索）
  Memory 历史：           ~6K Token  （可控制）
  用户当前输入：           ~0.5K Token
  LLM 输出预留：          ~2K Token
  ─────────────────────────────────
  实际使用：              ~16.5K Token（远小于 128K）

  为什么不全用满？
  1. Token 越多成本越高（$0.01/1K Token）
  2. 太长上下文 LLM 注意力质量下降
  3. 延迟增加（每 1K Token 约增加 50ms）

  所以 Memory 预算通常控制在 4K~8K Token
```

---

## 4. RAG + Memory 在 Agent 中的整合

```
┌──────────────────────────────────────────────────────┐
│              Agent + RAG + Memory 整合架构              │
│                                                       │
│  用户："上次讨论的那个方案，现在有新进展吗？"             │
│        │                                              │
│        ▼                                              │
│  ┌──────────────┐                                     │
│  │ L3 长期记忆  │ → 向量检索"上次讨论的方案"              │
│  └──────┬───────┘                                     │
│         ▼                                             │
│  ┌──────────────┐                                     │
│  │ RAG 知识库   │ → 检索"方案的最新状态"                 │
│  └──────┬───────┘                                     │
│         ▼                                             │
│  ┌──────────────┐                                     │
│  │ L1+L2 记忆   │ → 最近对话上下文                      │
│  └──────┬───────┘                                     │
│         ▼                                             │
│  ┌──────────────┐                                     │
│  │ Agent 决策    │ → LLM 综合所有信息生成回答             │
│  │ (ReAct)      │ → 可能调用工具获取实时数据             │
│  └──────┬───────┘                                     │
│         ▼                                             │
│  ┌──────────────┐                                     │
│  │ Memory 更新   │ → 保存到 L1/L2/L3                    │
│  └──────────────┘                                     │
└──────────────────────────────────────────────────────┘
```

---

## 5. 面试实战

### 【面试回答（标准版）】—— 30 秒口语化

> RAG 是检索增强生成，核心思路是"先检索再生成"。离线阶段把文档分片、向量化存入向量数据库；在线阶段把用户问题向量化，在向量库中检索最相关的文档片段，注入 Prompt 让 LLM 基于真实数据回答。
>
> 相比微调，RAG 成本低、时效性强、数据不外泄。我们项目用 Milvus 做向量存储，递归分片 500 字加 10% 重叠，检索用向量+BM25 混合检索加 Reranker 精排。
>
> Memory 分三层：L1 工作记忆在 Prompt 中传最近几轮对话，L2 短期记忆用 Redis 存完整会话，L3 长期记忆用向量库做语义检索历史对话。Token 管理用滑动窗口加摘要压缩。

### 【面试官可能追问】

**追问 1：RAG 检索效果不好怎么办？**

> 逐项排查：分片质量 → Embedding 模型 → 检索方式 → 结果排序 → Prompt 设计。
> 最有效的优化是两个：**混合检索**（向量 + BM25 + RRF 融合）和 **Reranker 重排**。我们的经验是混合检索准确率从 75% 提到 88%，加 Reranker 后到 92%。

**追问 2：向量数据库和 ES 有什么区别？**

> ES 核心是关键词匹配（BM25），向量库是语义相似度。"年假政策" vs "休假制度"——ES 可能搜不到（词不同），但向量检索能匹配（语义相同）。
> ES 8.x 已加向量能力。已有 ES 且量不大先用 ES；千万级以上用 Milvus。最佳实践是混合检索。

**追问 3：长期记忆怎么实现？**

> 存：对话结束后提取关键信息 → 向量化存 Milvus + 结构化存 MySQL。
> 取：新问题来时 → 向量检索相关历史记忆（Top-5）→ 注入 Prompt。
> 类比 Java：就是"写时建索引、读时查索引"，和 ES 使用方式一样。

### 【常见错误】

| 错误说法 | 正确说法 |
|---------|---------|
| "RAG 就是把文档全部传给 LLM" | "RAG 先检索相关片段，只把相关内容传给 LLM" |
| "向量库比 MySQL 好" | "两者互补，向量库语义检索，MySQL 结构化查询" |
| "Memory 就是保存聊天记录" | "Memory 是分层的，工作/短期/长期各司其职" |
| "RAG 能完全解决幻觉" | "RAG 显著降低幻觉概率，但仍需引用追溯和事实校验" |

---

## 6. 一句话总结（背诵用）

> **RAG = 文档分片 → 向量化 → 存向量库 → 在线检索 → Reranker 精排 → 注入 Prompt → LLM 生成，核心优化是混合检索 + Reranker；Memory 分三层（Prompt/Redis/向量库+MySQL），核心挑战是 Token 管理（滑动窗口+摘要压缩）。**

---

> **Stage 5（详细版）结束。**
