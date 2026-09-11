# AI

!!! "AI Agent 层级"

    第一层：会不会用LLM
    - Prompt
    - Structed Output
    - Tool Calling
    - 基本模型能力

    第二层：能不能给 LLM 正确的信息
    - RAG
    - Chunk
    - Embedding
    - Retrieval
    - Rerank
    - Context Management

    第三层：能否控制 LLM 的行为
    - Workflow
    - LangGraph
    - State Machine
    - Multi-Agent
    - Routing
    - Planning
    - Tool Selection

    第四层：是否可以把 Agent 接入到真实的世界
    - MCP
    - API
    - Database
    - Kafka
    - Python
    - Browser
    - Internal Service

    第五层：Agent 犯错如何处理
    - Guardrail
    - Permission
    - Sandbox
    - Human-in-the-loop
    - Retry
    - Fallback
    - Validation
    - Rollback

    第六层：证明 Agent 的价值
    - Evaluation
    - Trace
    - Bad Case
    - A/B Test
    - Latency
    - Cost
    - Success Rate
    - Business Metric

## Function Call

LLM 擅长语义理解与意图识别，但存在知识时效截止、缺乏确定性计算能力、无法直接操作外部系统等局限。

核心机制：LLM 本身并不直接执行外部代码，而是根据工具定义（Schema）判断调用时机并生成结构化参数（JSON），由宿主程序实际执行工具，再将执行结果（Observation）返回给 LLM 组织最终回答。常见工具如：实时搜索、计算器、数据库查询、业务 API 等。

## Agent 规划模式 

规划模式决定了 Agent 如何把复杂目标拆解为具体行动。常见的主流模式如下：

### 1. ReAct (Reasoning + Acting) - 动态循环规划
- **核心机制**：`Thought → Action → Observation` 闭环循环。LLM 走一步看一步，根据上一步的执行结果（Observation）动态决定下一步是继续调用工具还是给出最终答复。
- **适用场景**：探索性、结果不确定、依赖外部反馈的任务（如查阅多层关联资料、排查报错日志）。
- **工程痛点与解法**：
  - **容易死循环或早退**：仅靠 LLM 自身判断容易陷入重复调用或幻觉提前终止。
  - **工程实践**：通常在外部宿主程序中增加 **最大步数限制（Max Steps）**，或引入**状态机/校验器（Validator）**强制进行状态流转判断。

### 2. Plan-and-Execute - 显式拆解规划
- **核心机制**：两阶段分离。由 Planner 先完整拆解出子任务列表（Step 1...N），再交由 Executor 逐个执行，最后由 Summarizer 汇总。
- **适用场景**：步骤明确的长链路任务（如“编写某主题的行业调研报告并导出为 PDF”）。
- **优缺点**：
  - **优点**：执行成本可控，全局结构清晰，便于展示步骤进度条。
  - **缺点**：前期规划容易跟不上现实变化。若前置步骤失败，容易引起连锁雪崩（通常需配合 **Plan-and-Solve / 动态重规划 Replanner**）。

### 3. Reflection (Self-Refine) - 反馈式自我演进
- **核心机制**：`Action → Evaluation → Reflection → Retry`。Agent 产出初步结果后，由专门的 Evaluator/Critic 检查打分，若不合格则输出修改意见，LLM 据此反思并迭代。
- **适用场景**：有明确评价标准、对准确率要求高的任务（如代码生成、SQL 编写、长文本润色）。
- **优缺点**：大幅提升单次输出质量，但会带来多倍的延迟（Latency）与 Token 成本消耗。

### 4. Tree of Thoughts (ToT) - 搜索式树状规划
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


## RAG

## 向量数据库

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

### 向量化

### 检索策略

| 检索方式 | 优势 | 劣势 |
| --- | --- | --- |
| 向量检索 | 语义相似，处理同义词/改写 | 精确词元匹配差，数值模糊 |
| BM25 | 精确词元匹配，无需训练 | 无语义理解，无法跨文档推理 |
| GraphRAG | 图遍历实现多跳推理 | 构建成本高，需要图数据库 |
| 混合检索 | 兼顾语义与精确匹配 | 仍无法解决多跳推理 |


#### Neo4j

节点
关系
属性

### 评估

#### 四大指标

| 指标 | 含义 | 参考阈值 | 可能原因 | 优化方向 |
| --- | --- | --- | --- | --- |
| Context Recall | 判断检索出的文档是否包含回答问题所需的信息 | 0.8-0.9 | top_k 太小；chunk 太大；embedding 召回差 | 增加 top_k；缩小 chunk_size；更换 embedding 模型 |
| Context Precision | 判断检索出的文档是否与问题相关，去除噪音 | 0.7-0.85 | 检索结果包含大量无关片段 | 增加 Rerank；提高哦相似度阈值；改善文档质量 |
| Answer Relevancy | 判断最终答案是否切题 | 0.8-0.9 | Prompt 没有引导模型直接回答；答案太长绕弯 | Query重写；优化 response prompt |
| Faithfulness | 判断最终答案是否基于检索到的上下文 | 0.85-0.95 | System prompt 约束弱；检索内容不足 | 强化system prompt；先提到recall |

> 优先级建议：先修 Recall → 再修 Faithfulness → 再修 Precision → 最后 Relevancy

#### ragas/DeepEval



## LangGraph

### 核心概念

|概念|含义|代码实现|
|---|---|
|State|工作流中的数据状态|Dict["str", Any]|
|Node|可复用的节点|Runnable（Prompt，Model，Chain，Custom Runnable 等）|
|Edge|节点之间的连接|ConditionalEdge、SimpleEdge、DirectEdge 等|
|Graph|由节点和边组成的有向无环图（DAG）|Graph|

### 控制流

### 中断

human in the loop


### 护栏

#### PII 
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

## Hermes

## 记忆

策略：

- 短期记忆（滑动窗口）：根据token或消息条数，保留最近N条消息
- 上下文压缩：触发阈值压缩上下文以节省token
- 长期记忆（向量库）：向量库存储，语义检索召回

记忆打分

| 因子 | 含义 | 来源 |
|---|---|---|
| 相关性 Relevance | 与当前查询的语义相似度 | 向量检索 raw score |
| 近期性 Recency | 越久未访问得分越低 | `exp(-decay_per_hour × hours)` |
| 重要性 Importance | 写入时标注的长期权重 | 0~1，无需再归一化 |

现在通过keywords+BM25可以实现比RAG更好的记忆搜索效果，优点在于精确性和高性能，但对语义理解较差。所以对于编程类系统，BM25和正则可以解决90%以上的问题，而对话类系统则需要结合BM25+ANN

冷热分离的记忆

观测记忆分层压缩：关键信息沉淀，低价值信息遗忘


| 策略 | 做法 | 适合 |
|------|------|------|
| **Last-Write-Wins** | 同 key 直接用新值覆盖旧值 | 偏好、当前目标、当前阻塞点（默认首选） |
| **版本化 + 失效标记** | 不物理删；旧记录 `status=stale` / `superseded_by=新 id`；检索只取 active | 要审计、要回放“为什么改过” |
| **Merge 合并** | 同主题互补、非互斥 → 合成一条（如约束列表） | 多条可并存的偏好/禁止项 |
| **冲突升级** | 两边都像真又互斥 → 问用户，或交给 Reflector 按更新时间/置信度裁决 | 高风险事实（权限、账号、业务规则） |
| **时间窗 / TTL** | 临时状态设过期，到期自动 Forget | “这周在赶 auth”类短期事实，避免抢长期偏好的召回 |



### mem0

## Harness



## MCP


## Cache 命中
降低token成本：
- 模型路由
- 提高cache命中率：
    - 命中类似于MySQL联合索引的最左原则，
    - 静态内容放在前边，动态内容放在后边
    - 通过心跳保持Cache不会被淘汰，否则会重新计算token

## Gateway

业界常用Portkey AI Gateway，可提供统一路由、虚拟key预算、fallback和跨供应商成本跟踪等。

agent任务是分钟级而非毫秒级，因此应该用Temporal等工具支持状态的持久化、重试、任务队列与定时器等功能
