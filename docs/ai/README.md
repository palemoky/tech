# AI

## AI Agent 层级

=== "第一层：会不会用LLM"

    - Prompt
    - Structed Output
    - Tool Calling
    - 基本模型能力

=== "第二层：能不能给 LLM 正确的信息"

    - RAG
    - Chunk
    - Embedding
    - Retrieval
    - Rerank
    - Context Management

=== "第三层：能否控制 LLM 的行为"

    - Workflow
    - LangGraph
    - State Machine
    - Multi-Agent
    - Routing
    - Planning
    - Tool Selection

=== "第四层：是否可以把 Agent 接入到真实的世界"

    - MCP
    - API
    - Database
    - Kafka
    - Python
    - Browser
    - Internal Service

=== "第五层：Agent 犯错如何处理"

    - Guardrail
    - Permission
    - Sandbox
    - Human-in-the-loop
    - Retry
    - Fallback
    - Validation
    - Rollback

=== "第六层：证明 Agent 的价值"

    - Evaluation
    - Trace
    - Bad Case
    - A/B Test
    - Latency
    - Cost
    - Success Rate
    - Business Metric

## Tool Call / Function Calling

LLM 擅长语义理解与意图识别，但存在知识时效截止、缺乏确定性计算能力、无法直接操作外部系统等局限。

核心机制：LLM 本身并不直接执行外部代码，而是根据工具定义（Schema）判断调用时机并生成结构化参数（JSON），由宿主程序实际执行工具，再将执行结果（Observation）返回给 LLM 组织最终回答。常见工具如：实时搜索、计算器、数据库查询、业务 API 等。

## RAG

RAG 的完整流程可分为**离线索引**与**在线推理**两个阶段：

**离线索引**：解析、清洗文档 → 分块（Chunking） → 通过 Embedding 模型向量化 → 存入向量数据库。

**在线推理**：
1. **检索 (Retrieval)**：根据用户问题，从向量数据库中检索语义最相关的 top-K 候选文档片段。
2. **增强 (Augmentation)**：将检索到的文档片段与用户问题组合，构建扩展上下文（Context），作为 Prompt 一并送入 LLM。
3. **生成 (Generation)**：LLM 基于扩展上下文进行回答，确保答案的准确性和可追溯性。

### 向量数据库

| 产品 | 特点 | 适用场景 |
| --- | --- | --- |
| FAISS (Meta) | 单机库，非完整数据库；极致性能研究向 | 原型、离线实验、嵌入现有服务 |
| Milvus | 云原生、分布式、可扩展至百亿级 | 大规模生产、需要分区与多副本 |
| Pinecone | 全托管 SaaS，零运维 | 快速上线、无运维团队 |
| Chroma | 轻量、易上手 | 原型、小项目 |
| Weaviate | GraphQL API、内置模块 | 需要图式查询与生态集成 |
| Qdrant | Rust 实现、过滤丰富、性能强 | 自托管生产、复杂过滤 |

ANN（近似最邻近）算法：
- HNSW（分层导航小世界图）：图索引，查询延迟低，内存占用偏高；高召回常用
- IVF（倒排文件）：聚类划分桶，查询只搜索部分桶；需训练，适合大规模
- PQ（乘积量化）：向量压缩，节省内存、加速距离计算；有损，精度略降

### 图数据库与知识图谱

在 RAG 中，纯向量检索仅关注局部语义相似度，难以捕捉实体间的结构化关联——例如"某人创立了某公司"或"某项目依赖某技术"这类显式关系。图数据库通过构建知识图谱（Knowledge Graph），将离散的文档信息组织为结构化的实体-关系网络，从而帮助 Agent 实现**跨文档关联与多跳关系推理**。

#### Neo4j

Neo4j 是目前最主流的原生图数据库，采用 LPG（Labeled Property Graph，标签属性图）模型，其核心由四个基本元素构成：

1. **节点 (Node)**：表示实体或概念（如 `Person`、`Company`、`Project`）。
2. **标签 (Label)**：对节点进行分组分类（类似关系型数据库中"表"的概念，且一个节点可拥有多个标签，如 `(:Person:Founder)`）。
3. **关系 (Relationship)**：连接两个节点的**有向边**，具有明确的方向和类型（如 `-[:FOUNDED]->`、`-[:DEPENDS_ON]->`）。
4. **属性 (Property)**：附加在节点或关系上的键值对（Key-Value），用于存储结构化数据（如节点属性 `{name: "乔布斯", birth: 1955}`，关系属性 `{since: 1976}`）。

```mermaid
graph LR
    A["(:Person {name: 'Steve Jobs'})"] -->|"[:FOUNDED {since: 1976}]"| B["(:Company {name: 'Apple'})"]
    B -->|"[:PRODUCED]"| C["(:Product {name: 'iPhone'})"]
```

##### Cypher 查询语言

Cypher 是 Neo4j 原生的声明式图查询语言，采用直观的 ASCII 艺术箭头语法 `(节点)-[关系]->(节点)` 来描述图模式，使查询逻辑贴近人类对关系的自然表达：

```cypher
// 查询创立了 Apple 公司的所有人姓名
MATCH (p:Person)-[:FOUNDED]->(c:Company {name: 'Apple'})
RETURN p.name;

// 查找两度关系以内的所有上下游（多跳推理）
MATCH (c:Company {name: 'Apple'})-[*1..2]-(related)
RETURN related;
```

### 检索策略

| 检索方式 | 最优场景 | 弱项 |
|----------|----------|------|
| 向量检索 (Vector) | 语义相似，同义词/改写 | 精确词元、多跳推理 |
| BM25 | 精确词元、日期/代码 | 语义理解 |
| 图谱检索 (Graph) | 多跳关系推理 | 模糊语义查询 |
| **RRF 融合** | **覆盖所有场景** | 依赖各路质量 |

```
                   ┌─────────────┐
                   │    Query    │
                   └──────┬──────┘
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   向量检索(Vector)   BM25 检索      图谱检索(Graph)
   语义相似度          精确词元匹配    多跳关系遍历
          └───────────────┼───────────────┘
                          ▼
                   RRF 融合排序
                   score = Σ 1/(k + rank)
                          ▼
                   最终排序结果
```

#### RRF（Reciprocal Rank Fusion）

RRF 是一种无参数的排名融合算法，将多路检索器的排名列表合并为一份最终排序，**只依赖排名、不依赖原始分数**，因此无需对不同检索器的分数做归一化。

$$
\text{RRF}(d) = \sum_{r \in R} \frac{1}{k + \text{rank}_r(d)}
$$

- $R$：所有检索器的结果列表集合
- $k$：平滑常数（默认 60），用于抑制头部排名过度主导，使排名靠后的文档也能贡献有效分数
- $\text{rank}_r(d)$：文档 $d$ 在检索器 $r$ 中的排名（从 1 开始）

**核心优势**：
- **无需分数归一化**——向量/BM25/Graph 三路分数尺度不同也能直接合并
- **只关心排名**——对各检索器的打分分布无假设
- **多路投票效应**——被多个检索器同时召回的文档会获得更高的综合分

#### MMR（Maximal Marginal Relevance）

MMR 在保证相关性的同时引入**多样性约束**，防止检索结果中出现大量语义重复的文档片段：

$$
\text{MMR}(d_i) = \lambda \cdot \text{Sim}(q, d_i) - (1-\lambda) \cdot \max_{d_j \in S} \text{Sim}(d_i, d_j)
$$

- $\text{Sim}(q, d_i)$：文档 $d_i$ 与查询 $q$ 的相关性（如余弦相似度）
- $\max_{d_j \in S} \text{Sim}(d_i, d_j)$：文档 $d_i$ 与已选集合 $S$ 中最相似文档的相似度
- $\lambda$：权衡参数（0~1），$\lambda$ 越大越偏向相关性，越小越偏向多样性

**典型用途**：当 top-K 检索结果存在大量近似重复片段时，使用 MMR 重排序可以显著提升上下文覆盖度，让 LLM 获得更全面的参考信息。

#### Cross-Encoder Rerank

Cross-Encoder 是一种精排模型，将 `(query, document)` **拼接为一个序列**送入 Transformer，直接输出相关性得分。相比检索阶段使用的 Bi-Encoder（分别编码 query 和 document 再算相似度），Cross-Encoder 能捕捉更细粒度的交互语义，但计算成本更高。

| 对比 | Bi-Encoder | Cross-Encoder |
|------|-----------|--------------|
| 输入方式 | query 与 doc 分别编码 | query + doc 拼接编码 |
| 速度 | 快（可预计算向量） | 慢（每对都需推理） |
| 精度 | 较高 | **更高**（捕捉交互特征） |
| 典型角色 | 粗排 / 召回 | **精排 / Rerank** |

**工业实践**：先用向量检索 + BM25 粗排召回 top-K 候选（如 50~100 篇），再用 Cross-Encoder 对候选集精排，取 top-N（如 5~10 篇）送入 LLM，在精度与延迟间取得平衡。

综合以上策略，工业级 RAG 系统的典型架构为：**Vector + BM25 + Graph 三路并行召回 → RRF 融合 → MMR 去重 → Cross-Encoder 精排**，逐层筛选出最终送入 LLM 的高质量上下文。

## 记忆

LLM 本身是无状态的——每次请求都是独立的。Agent 的"记忆"系统负责在多轮对话和跨会话之间**持久化关键信息**，让 Agent 具备上下文连贯性和个性化能力。

### 记忆类型

| 类型 | 机制 | 特点 |
|------|------|------|
| **短期记忆（滑动窗口）** | 按 Token 数或消息条数保留最近 N 条对话 | 实现简单，但窗口外的信息直接丢失 |
| **上下文压缩** | 对话达到阈值时，用 LLM 将历史摘要为精简版本 | 节省 Token，保留关键语义，但有信息损耗 |
| **长期记忆（向量库）** | 将重要信息向量化存储，按语义检索召回 | 跨会话持久化，支持大规模记忆 |

### 记忆检索与打分

当记忆条目积累到一定规模后，需要对召回的记忆进行**综合打分排序**，决定哪些记忆注入当前上下文：

| 因子 | 含义 | 计算方式 |
|------|------|----------|
| **相关性 (Relevance)** | 与当前查询的语义相似度 | 向量检索 raw score |
| **近期性 (Recency)** | 越久未访问得分越低 | `exp(-decay_per_hour × hours)` |
| **重要性 (Importance)** | 写入时标注的长期权重 | 0~1，无需再归一化 |

**检索策略选型**：

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 编程类系统 | BM25 + 正则 | 精确匹配函数名、变量名等结构化信息，可覆盖 90%+ 场景 |
| 对话类系统 | BM25 + ANN（向量检索） | 需兼顾精确匹配与语义理解 |

> BM25/关键词检索的优势在于精确性和高性能，但缺乏语义理解能力；向量检索擅长语义匹配但精确匹配差。两者互补使用效果最佳。

### 记忆生命周期管理

记忆不是只增不减的——需要**冷热分离与分层压缩**：高价值信息沉淀为长期记忆，低价值和过时信息逐步遗忘，避免噪音记忆污染召回质量。

当新记忆与已有记忆冲突时，需要选择合适的**更新策略**：

| 策略 | 做法 | 适用场景 |
|------|------|----------|
| **Last-Write-Wins** | 同 key 直接用新值覆盖旧值 | 偏好、当前目标、当前阻塞点（默认首选） |
| **版本化 + 失效标记** | 不物理删除；旧记录标记 `status=stale` / `superseded_by=新id`；检索只取 active | 需要审计或回放"为什么改过" |
| **Merge 合并** | 同主题互补、非互斥 → 合成一条 | 多条可并存的偏好/禁止项（如约束列表） |
| **冲突升级** | 两边都像真又互斥 → 问用户或交由 Reflector 裁决 | 高风险事实（权限、账号、业务规则） |
| **时间窗 / TTL** | 临时状态设过期，到期自动遗忘 | 短期事实（如"这周在赶 DDL"），避免抢占长期偏好的召回 |

### Mem0

Mem0 是一个开源的 Agent 记忆管理框架，提供开箱即用的记忆提取、存储、检索和更新能力，支持自动从对话中识别并持久化关键信息。

**核心流程**：

```
用户对话 → LLM 提取记忆事实 → 去重/冲突检测 → 存储（向量库 + 图谱）→ 下次对话时语义检索召回
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| **自动记忆提取** | 用 LLM 从对话中抽取关键事实（偏好、约束、背景），无需手动标注 |
| **双存储引擎** | 向量库（语义检索）+ 知识图谱（关系推理），互补召回 |
| **记忆去重与更新** | 新记忆写入时自动检测冲突，支持合并/覆盖/版本化 |
| **多层级记忆** | 支持 User / Agent / Session 三个粒度的记忆隔离 |
| **多模型支持** | 兼容 OpenAI、Anthropic、本地模型等 |

**代码示例**：

```python
from mem0 import Memory

m = Memory()

# 添加对话，Mem0 自动提取并存储记忆
m.add("我喜欢用 Python，偏好 FastAPI 做后端", user_id="alice")
m.add("我的项目用的是 PostgreSQL 数据库", user_id="alice")

# 语义检索相关记忆
results = m.search("帮我选一个 ORM 框架", user_id="alice")
# → 召回: "喜欢 Python"、"使用 PostgreSQL"

# 查看用户所有记忆
m.get_all(user_id="alice")

# 记忆更新（自动检测冲突并覆盖）
m.add("我们已经从 PostgreSQL 迁移到 MySQL 了", user_id="alice")
# → 旧记忆 "PostgreSQL" 被更新为 "MySQL"
```

> **适用场景**：需要快速为 Agent 集成记忆能力、且不想从零搭建记忆管线（提取→去重→存储→召回）的项目。

## Agent 规划模式 

规划模式决定了 Agent 如何把复杂目标拆解为具体行动。常见的主流模式如下：

### ReAct (Reasoning + Acting) - 动态循环规划
- **核心机制**：`Thought → Action → Observation` 闭环循环。LLM 走一步看一步，根据上一步的执行结果（Observation）动态决定下一步是继续调用工具还是给出最终答复。
- **适用场景**：探索性、结果不确定、依赖外部反馈的任务（如查阅多层关联资料、排查报错日志）。
- **工程痛点与解法**：
  - **容易死循环或早退**：仅靠 LLM 自身判断容易陷入重复调用或幻觉提前终止。
  - **工程实践**：通常在外部宿主程序中增加 **最大步数限制（Max Steps）**，或引入**状态机/校验器（Validator）**强制进行状态流转判断。

### Plan-and-Execute - 显式拆解规划
- **核心机制**：两阶段分离。由 Planner 先完整拆解出子任务列表（Step 1...N），再交由 Executor 逐个执行，最后由 Summarizer 汇总。
- **适用场景**：步骤明确的长链路任务（如“编写某主题的行业调研报告并导出为 PDF”）。
- **优缺点**：
  - **优点**：执行成本可控，全局结构清晰，便于展示步骤进度条。
  - **缺点**：前期规划容易跟不上现实变化。若前置步骤失败，容易引起连锁雪崩（通常需配合 **Plan-and-Solve / 动态重规划 Replanner**）。

### Reflection (Self-Refine) - 反馈式自我演进
- **核心机制**：`Action → Evaluation → Reflection → Retry`。Agent 产出初步结果后，由专门的 Evaluator/Critic 检查打分，若不合格则输出修改意见，LLM 据此反思并迭代。
- **适用场景**：有明确评价标准、对准确率要求高的任务（如代码生成、SQL 编写、长文本润色）。
- **优缺点**：大幅提升单次输出质量，但会带来多倍的延迟（Latency）与 Token 成本消耗。

### Tree of Thoughts (ToT) - 搜索式树状规划
- **核心机制**：将思考过程建模为树结构，每一步生成多个潜在候选思路（分支），借助 BFS（广度优先）/ DFS（深度优先）或评分机制评估各分支价值，并支持**回溯（Backtracking）**。
- **适用场景**：需要全局试错与多步博弈的复杂决策（如 24点游戏、数独、策略制定、算法设计）。
- **优缺点**：推理能力上限最高，但调用次数指数级膨胀，通常限于离线高价值场景。

---

### 模式对比与选型指南

| 模式 | 思考时机 | 适应环境变化能力 | Token/延迟成本 | 适用典型场景 |
| :--- | :--- | :--- | :--- | :--- |
| **ReAct** | 边走边想 | ⭐⭐⭐⭐⭐（强，随时调整） | 中等 | 动态排错、多跳信息检索 |
| **Plan-and-Execute** | 想好再走 | ⭐⭐（偏弱，静态为主） | 低 | 结构化流程、标准多步骤任务 |
| **Reflection** | 做完复盘 | ⭐⭐⭐（中等） | 较高 | 代码/文案生成质量调优 |
| **Tree of Thoughts** | 穷举推演 | ⭐⭐⭐⭐（支持回溯） | 极高（成倍翻番） | 复杂逻辑推理、算法解题 |

## Multi Agent

### 3种模式

=== "Pipeline"

    ```
    请求
    │
    ▼
    [Retriever] ──▶ [Reranker] ──▶ [Generator] ──▶ [Verifier] ──▶ 响应
    ```

=== "Hub and Spoke"

    ```
                    ┌─── Retriever ──┐
                    │                │
    用户 ──▶ Orchestrator ─── Reranker ───▶ 合成回复
                    │                │
                    └─── Generator ──┘
    ```

=== "Blackboard"

    ```
            写入              读取
    Agent A ──────▶ Blackboard ◀────── Agent C
    Agent B ──────▶     │     ◀────── Agent D
                        │
                变更通知（事件）
    ```

3 种模式的对比：

| 维度 | Pipeline | Hub-and-Spoke | Blackboard |
|------|----------|---------------|------------|
| 复杂度 | 低 | 中 | 高 |
| 调试难度 | 易 | 中 | 难 |
| 并行能力 | 低 | 高 | 高 |
| 适合任务 | 线性 | 多角色协作 | 迭代协商 |
| 典型实现 | Redis Queue | FastAPI + gRPC | DB + Pub-Sub |

### 通信机制

| 机制 | 代表技术 | 延迟 | 耦合度 | 适用场景 |
|------|----------|------|--------|----------|
| **Sync RPC** | HTTP / gRPC | 低 | 高 | 实时决策、简单编排 |
| **Async Queue** | Redis / Kafka / SQS | 中 | 低 | 耗时操作、高并发 |
| **Pub-Sub** | SNS / Kafka Topic | 中 | 很低 | 广播事件、多订阅方 |
| **Shared Storage** | S3 + DB + 信号 | 高 | 很低 | 大文件、artifacts 共享 |

### 优雅降级

**场景**：Reranker 服务不可用时，系统不应整体崩溃，而应**降级**继续提供服务。

```
正常路径：  retrieve → rerank → generate  ✅
降级路径1： retrieve → (skip rerank) → generate  ⚠️ 质量略降
降级路径2： (retrieve timeout) → return cached snippets  ⚠️ 更大降级
兜底路径：  return static fallback message  🆘
```

### 三条铁律

1. **每条消息带 `request_id`**，处理前先做幂等检查
2. **每个 Agent 暴露 latency + status**，不监控等于盲飞
3. **提前设计失败路径**，不要等故障发生后再想降级逻辑

### 常用框架

Microsoft Agent Framework/AutoGen 与 CrewAI

## LangGraph

LangGraph 是 LangChain 团队推出的**底层 Agent 编排框架**，把 Agent 的执行过程建模为一张**有状态的图**。与 LangChain 早期的 Chain（线性 DAG）不同，LangGraph 的核心卖点是**支持循环**——这正是 ReAct 这类「思考 → 调工具 → 观察 → 再思考」模式所需要的。

### 核心概念

| 概念 | 含义 | 代码实现 |
|---|---|---|
| State | 在节点间共享、流转的数据状态，是图的「内存」 | `TypedDict` / Pydantic `BaseModel`，字段可用 `Annotated[..., reducer]` 指定合并方式 |
| Node | 执行一步具体工作（调模型、调工具、业务逻辑） | 普通函数 `(state) -> dict`，返回的是**状态的增量更新**而非完整状态 |
| Edge | 决定下一步执行哪个节点 | `add_edge`（固定边）、`add_conditional_edges`（条件边）、`START` / `END` 虚拟节点 |
| Reducer | 多个节点写同一字段时如何合并 | 默认覆盖；`operator.add` 追加；`add_messages` 按消息 ID 追加/更新 |
| Graph | 由节点和边组成的**有向图（允许有环）** | `StateGraph(State)` → `.compile()` 得到可执行的 `CompiledGraph` |
| Checkpointer | 每步执行后保存状态快照，是记忆、中断、恢复的基础 | `InMemorySaver`、`SqliteSaver`、`PostgresSaver`，按 `thread_id` 隔离会话 |

最小示例：

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import InMemorySaver

class State(TypedDict):
    messages: Annotated[list, add_messages]  # 追加而非覆盖

def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}  # 只返回增量

builder = StateGraph(State)
builder.add_node("chatbot", chatbot)
builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)

graph = builder.compile(checkpointer=InMemorySaver())
graph.invoke({"messages": [("user", "hi")]}, {"configurable": {"thread_id": "1"}})
```

### 控制流

| 方式 | 作用 | 典型场景 |
|---|---|---|
| **固定边** `add_edge(a, b)` | a 执行完必然走 b | 线性流水线 |
| **条件边** `add_conditional_edges(a, router)` | 根据 `router(state)` 的返回值选择下一个节点 | 判断是否需要调工具、意图路由 |
| **循环** | 条件边指回之前的节点 | ReAct：`agent → tools → agent`，直到不再调工具才走 `END` |
| **并行（扇出/扇入）** | 一个节点连多条边，下游节点在同一步并发执行，结果经 reducer 合并 | 多路检索、多角度分析 |
| **`Send`（Map-Reduce）** | 在运行时动态决定分发几个并行任务，每个任务携带独立输入 | 对 N 篇文档各生成摘要后汇总 |
| **`Command`** | 节点内同时完成「更新状态 + 跳转」，无需预先声明条件边 | Multi-Agent 之间的 handoff |
| **子图（Subgraph）** | 把一张编译好的图当作另一张图的节点 | 模块化复用、Multi Agent 分层 |

ReAct 循环的典型写法：

```python
from langgraph.prebuilt import ToolNode, tools_condition

builder.add_node("agent", call_model)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)  # 有 tool_calls → "tools"，否则 → END
builder.add_edge("tools", "agent")                       # 形成循环
```

> 防止死循环：调用时传 `{"recursion_limit": 25}`，超过步数会抛出 `GraphRecursionError`。

### 中断

中断是 LangGraph 实现 **Human in the Loop** 的机制：图执行到某处暂停，把状态持久化到 Checkpointer，等待人工输入后再从断点继续。**必须配置 Checkpointer 和 `thread_id`**，否则无法恢复。

两种方式：

| 方式 | 写法 | 特点 |
|---|---|---|
| **动态中断**（推荐） | 节点内调用 `interrupt(payload)` | 可以按条件中断，并把上下文信息抛给前端 |
| **静态断点** | `compile(interrupt_before=["tools"])` / `interrupt_after` | 在指定节点前/后固定暂停，多用于调试 |

```python
from langgraph.types import interrupt, Command

def human_review(state: State):
    decision = interrupt({"question": "是否执行该操作？", "tool_call": state["tool_call"]})
    if decision == "approve":
        return Command(goto="execute")
    return Command(goto=END)

config = {"configurable": {"thread_id": "order-42"}}
graph.invoke(inputs, config)                 # 执行到 interrupt 处暂停，返回 __interrupt__ 信息
graph.invoke(Command(resume="approve"), config)  # 带着人工决定恢复执行
```

常见 HITL 模式：

- **审批**：高风险工具调用（转账、删库、发邮件）前要求人工确认
- **编辑状态**：人工修改模型生成的草稿/参数后再继续
- **补充信息**：Agent 缺少关键参数时向用户追问

注意事项：

- 恢复时**节点会从头重新执行**（而非从 `interrupt()` 那一行继续），所以 `interrupt()` 之前的代码要保证幂等，副作用操作放到中断之后
- 同一节点内有多个 `interrupt()` 时按调用顺序匹配 resume 值，不要把它放在顺序不固定的逻辑里
- 借助 Checkpointer 还能做**时间旅行**：`get_state_history()` 查看历史快照，从任意 checkpoint 分叉重跑

## 编排与持久化

Agent 任务是**分钟级甚至小时级**的，而非毫秒级的普通 RPC 调用，进程重启、模型超时、工具报错都可能发生在任务执行到一半时。因此长链路 Agent 需要一层持久化编排（如 Temporal），提供状态持久化、失败重试、任务队列与定时器等能力，让任务可以从中断处恢复而不是从头重跑。

> LangGraph 解决的是**单次运行内**的流程编排，Temporal 解决的是**跨进程、跨重启**的任务可靠性，两者互补。

## MCP

## Harness

### Hermes

## 护栏

### PII 
脱敏身份证号、手机号、邮箱、信用卡号等敏感信息

处理策略
| 策略 | 行为 | 是否保留可辨识性 | 典型场景 |
|---|---|---|---|
| redact | 整段替换为 `[REDACTED_类型]` | 否 | 合规、日志脱敏 |
| mask | 部分遮盖，保留少量尾部可读信息 | 否 | 客服界面、人工核对 |
| hash | 替换为确定性哈希 `<类型_hash:摘要>` | 是（假名化，可关联同一值） | 分析、排错 |
| block | 检测到即抛出 `PIIDetectionError` | N/A | 严禁出现该类信息 |

拦截Prompt注入
有害内容过滤 涉及恐怖暴力等
Human in the Loop
Before Agent 输入过滤
After Agent 输出安全

可以有基于规则和LLM语义分析的两种护栏模式，前者更快、更便宜，但容易被绕过，后者更全面，但更贵、更慢

## 评估

RAG 系统的评估需要同时衡量**检索质量**和**生成质量**两个维度。当前主流方案采用 **LLM-as-Judge**——用一个 LLM 对另一个 LLM 的输出进行自动化评分，替代人工标注。

### 核心指标

**检索质量**（评估检索器是否找到了正确且精准的上下文）：

| 指标 | 含义 | 参考阈值 | 低分原因 | 优化方向 |
|------|------|----------|----------|----------|
| Context Recall | 检索结果是否**覆盖**了回答所需的全部信息 | 0.8–0.9 | top_k 太小；chunk 太大；Embedding 召回差 | 增大 top_k；缩小 chunk_size；更换 Embedding 模型 |
| Context Precision | 检索结果中**有多少是真正相关的**（信噪比） | 0.7–0.85 | 召回了大量无关片段 | 增加 Rerank；提高相似度阈值；改善文档质量 |

>  Precision 与 Recall 是此消彼长的关系

**生成质量**（评估 LLM 是否基于上下文给出了准确且切题的回答）：

| 指标 | 含义 | 参考阈值 | 低分原因 | 优化方向 |
|------|------|----------|----------|----------|
| Faithfulness | 答案是否**忠实于**检索到的上下文（无幻觉） | 0.85–0.95 | System Prompt 约束弱；检索内容不足 | 强化 System Prompt 约束；优先提升 Recall |
| Answer Relevancy | 答案是否**切题**回答了用户问题 | 0.8–0.9 | Prompt 未引导直接回答；答案冗长绕弯 | Query 重写；优化 Response Prompt |

> **调优优先级**：Recall → Faithfulness → Precision → Relevancy
> 检索召回是一切的基础——如果相关文档根本没被检索到，后续的精排和生成都无法挽救。

!!! note "MRR（Mean Reciprocal Rank）"

    MRR 衡量的是**第一个正确结果出现的排名位置**，是检索系统常用的评估指标：

    $$\text{MRR} = \frac{1}{|Q|}\sum_{i=1}^{|Q|}\frac{1}{\text{rank}_i}$$

    其中 $\text{rank}_i$ 是第 $i$ 个查询中**首个正确结果的排名**。例如三次查询的首个正确结果分别排在第 1、3、2 位，则 $\text{MRR} = \frac{1}{3}(\frac{1}{1} + \frac{1}{3} + \frac{1}{2}) \approx 0.611$。

    MRR 只关心**第一个**命中的位置，适用于"用户只看第一条结果"的场景（如问答、实体查找）。若需评估整体排序质量，应结合 Precision/Recall 使用。

### 评估框架：Ragas & DeepEval

| 对比 | Ragas | DeepEval |
|------|-------|---------|
| 定位 | 专注 RAG 评估的轻量框架 | 通用 LLM 评估平台（覆盖 RAG + Agent + 对话） |
| 指标 | Faithfulness、Answer Relevancy、Context Precision/Recall | 同名指标 + Hallucination、Toxicity、Bias 等 |
| LLM-as-Judge | 默认 OpenAI，可切换任意 LLM | 同上，支持自定义评估模型 |
| 集成 | LangChain、LlamaIndex | LangChain、LlamaIndex、Pytest（`deepeval test run`） |
| 可视化 | 需搭配外部工具 | 内置 Confident AI 云端仪表盘 |
| 适用场景 | 快速验证 RAG Pipeline | CI/CD 集成、回归测试、多维度综合评估 |

> DeepEval 额外支持 `HallucinationMetric` 幻觉检测

### 生产可观测性：Langfuse

Langfuse 是开源的 LLM 可观测性平台，专注于**生产环境的追踪与监控**，与 Ragas/DeepEval 的离线评估形成互补：

| 能力 | 说明 |
|------|------|
| **Tracing** | 端到端追踪每次 LLM 调用链路（检索 → 增强 → 生成），记录输入/输出、耗时、Token 消耗 |
| **成本监控** | 按模型、用户、会话维度统计 Token 用量与费用 |
| **在线评估** | 支持接入 LLM-as-Judge 对生产流量实时打分（Faithfulness、Relevancy 等） |
| **Prompt 管理** | 版本化管理 Prompt 模板，支持 A/B 测试与回滚 |
| **数据集管理** | 从生产 Trace 中提取 case 构建评测数据集，反哺离线评估 |

**与 Ragas/DeepEval 的协作模式**：

```
开发阶段                          生产阶段
┌──────────────────┐            ┌──────────────────┐
│  Ragas/DeepEval  │            │     Langfuse     │
│  离线跑评测集      │            │  追踪生产流量      │
│  验证指标达标      │───上线──→   │  在线评估打分      │
│                  │            │  发现 bad case    │
└──────────────────┘            └────────┬─────────┘
        ▲                                │
        └────── 提取 case 反哺评测集 ──────┘
```

> **一句话总结**：Ragas/DeepEval 管"上线前够不够好"，Langfuse 管"上线后跑得怎么样"，两者结合形成评估闭环。

## 模型路由

## Cache 命中
降低token成本：
- 模型路由
- 提高cache命中率：
    - 命中类似于MySQL联合索引的最左原则，
    - 静态内容放在前边，动态内容放在后边
    - 通过心跳保持Cache不会被淘汰，否则会重新计算token

## Gateway

LiteLLM
业界常用Portkey AI Gateway，可提供统一路由、虚拟key预算、fallback和跨供应商成本跟踪等。

