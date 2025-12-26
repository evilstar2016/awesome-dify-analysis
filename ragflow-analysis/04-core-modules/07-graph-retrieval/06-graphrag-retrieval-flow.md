# GraphRAG 知识图谱启用与检索流程

## 1. 概述

RAGFlow 支持基于知识图谱的增强检索（GraphRAG），通过构建实体-关系图谱，提供结构化的知识检索能力。本文档详细说明如何启用 GraphRAG 功能，以及启用后的知识检索流程。

### 1.1 核心组件

- **存储层**：NetworkX 3.6.1（内存图数据库）+ MinIO/MySQL（持久化）
- **索引层**：Elasticsearch/OpenSearch（实体、关系、社区报告）
- **算法层**：PageRank、Leiden 社区检测、k-hop 遍历
- **检索层**：混合检索（向量 + 全文 + 图谱）

### 1.2 GraphRAG 方法

RAGFlow 提供两种 GraphRAG 实现：

1. **Light GraphRAG**：来自 HKUDS，轻量级实现
2. **General GraphRAG**：来自 Microsoft，功能完整

## 2. 启用 GraphRAG

### 2.1 通过 API 启用

#### 2.1.1 构建知识图谱

调用 API 为知识库构建图谱：

```http
POST /api/v1/kb/run_graphrag
Content-Type: application/json
Authorization: Bearer <your_token>

{
  "kb_id": "your_knowledgebase_id"
}
```

**响应示例**：

```json
{
  "retcode": 0,
  "retmsg": "success",
  "data": {
    "graphrag_task_id": "task_abc123xyz"
  }
}
```

#### 2.1.2 追踪构建进度

```http
GET /api/v1/kb/trace_graphrag?kb_id=your_knowledgebase_id
Authorization: Bearer <your_token>
```

**响应示例**：

```json
{
  "retcode": 0,
  "retmsg": "success",
  "data": {
    "id": "task_abc123xyz",
    "progress": 0.75,
    "progress_msg": "Building knowledge graph...",
    "create_time": "2024-01-15 10:30:00",
    "update_time": "2024-01-15 10:35:00"
  }
}
```

**Progress 状态说明**：
- `-1`：任务失败
- `0.0 - 0.99`：进行中
- `1.0`：完成

#### 2.1.3 启用图谱检索

在对话配置中设置 `use_kg` 标志：

```http
POST /api/v1/dialog/set
Content-Type: application/json
Authorization: Bearer <your_token>

{
  "dialog_id": "your_dialog_id",
  "prompt_config": {
    "use_kg": true,
    "system": "You are a helpful assistant...",
    "quote": true
  }
}
```

### 2.2 通过 Web UI 启用

#### 2.2.1 构建知识图谱

1. 进入知识库管理页面
2. 选择目标知识库
3. 点击 "Knowledge Graph" 标签
4. 点击 "Run GraphRAG" 按钮
5. 等待构建完成（可在任务列表中查看进度）

#### 2.2.2 启用图谱检索

在对话设置中：

1. 进入 "Chat Configuration" 页面
2. 找到 "Search Settings" 部分
3. 勾选 "Use Knowledge Graph" 选项
4. 保存设置

**配置路径**：
- 对话设置：`Dialog Settings -> Prompt Configuration -> use_kg`
- 搜索设置：`Search Settings -> Advanced -> Use Knowledge Graph`

### 2.3 配置参数说明

#### 2.3.1 GraphRAG 构建参数

在 `api/db/db_models.py` 中的 `Knowledgebase` 模型：

```python
class Knowledgebase(BaseModel):
    # GraphRAG 任务追踪
    graphrag_task_id = CharField(max_length=32, null=True)
    graphrag_task_finish_at = DateTimeField(null=True)
```

#### 2.3.2 检索配置参数

在 `graphrag/search.py` 的 `KGSearch.retrieval()` 方法：

```python
def retrieval(
    self,
    question: str,
    tenant_ids: str | list[str],
    kb_ids: list[str],
    emb_mdl,                      # 嵌入模型
    llm,                           # LLM 模型
    max_token: int = 8196,        # 最大 token 数
    ent_topn: int = 6,            # 返回实体数量
    rel_topn: int = 6,            # 返回关系数量
    comm_topn: int = 1,           # 返回社区报告数量
    ent_sim_threshold: float = 0.3,  # 实体相似度阈值
    rel_sim_threshold: float = 0.3,  # 关系相似度阈值
    **kwargs
):
```

## 3. 检索流程对比

### 3.1 传统检索流程（不使用 GraphRAG）

```
用户查询
  ↓
查询理解（分词、改写）
  ↓
混合检索
  ├─ 向量检索（Infinity）：语义相似度
  └─ 全文检索（ES）：关键词匹配
  ↓
重排序（Rerank）
  ↓
上下文构建
  ↓
LLM 生成答案
```

### 3.2 GraphRAG 增强检索流程（use_kg=true）

```
用户查询
  ↓
查询理解与实体提取
  ├─ LLM 提取查询中的实体
  └─ LLM 识别实体类型关键词
  ↓
三路并行检索
  ├─ 向量检索（Infinity）：文档 chunks
  ├─ 全文检索（ES）：关键词匹配
  └─ 图谱检索（ES + NetworkX）
      ├─ 实体检索：通过向量相似度匹配
      ├─ 关系检索：通过文本相似度匹配
      ├─ N-hop 遍历：获取邻居实体
      └─ 社区检索：获取社区报告
  ↓
图谱结果排序
  ├─ 实体排序：sim × pagerank
  ├─ 关系排序：sim × pagerank × type_boost
  └─ 社区排序：weight
  ↓
上下文融合
  ├─ 知识图谱上下文（插入到首位）
  │   ├─ 实体列表（CSV 格式）
  │   ├─ 关系列表（CSV 格式）
  │   └─ 社区报告（Markdown 格式）
  └─ 文档 chunks 上下文
  ↓
重排序（Rerank）
  ↓
Token 预算管理
  ↓
LLM 生成答案（带引用）
```

## 4. 详细检索流程

### 4.1 查询改写与实体提取

**输入**：用户查询 "What are the side effects of aspirin?"

**步骤**：

1. **实体提取**（`KGSearch.query_rewrite()`）
   ```python
   # LLM Prompt
   "Extract entities and their types from: 'What are the side effects of aspirin?'"
   
   # LLM 输出
   type_keywords = ["side effect", "drug"]
   entities_from_query = ["aspirin"]
   ```

2. **实体向量化**
   ```python
   query_vector = embedding_model.encode("aspirin")
   ```

### 4.2 图谱检索

#### 4.2.1 实体检索

**方法 1：通过关键词检索**（`get_relevant_ents_by_keywords()`）

```python
# ES 查询
{
  "query": {
    "bool": {
      "must": [
        {"term": {"knowledge_graph_kwd": "entity"}},
        {"knn": {"vector": query_vector, "k": 56}}
      ],
      "filter": [{"terms": {"kb_id": kb_ids}}]
    }
  }
}

# 返回
entities_from_query = {
  "aspirin": {
    "sim": 0.92,
    "pagerank": 0.0045,
    "description": "A nonsteroidal anti-inflammatory drug (NSAID)...",
    "n_hop_ents": [...]  # 邻居实体
  }
}
```

**方法 2：通过类型检索**（`get_relevant_ents_by_types()`）

```python
# ES 查询
{
  "query": {
    "bool": {
      "must": [
        {"term": {"knowledge_graph_kwd": "entity"}},
        {"terms": {"entity_type_kwd": ["drug", "side effect"]}}
      ]
    }
  },
  "sort": [{"rank_flt": "desc"}]  # PageRank 排序
}

# 返回
entities_from_types = {
  "stomach irritation": {"pagerank": 0.0032, ...},
  "bleeding": {"pagerank": 0.0028, ...}
}
```

#### 4.2.2 关系检索

**通过文本相似度检索**（`get_relevant_relations_by_txt()`）

```python
# ES 查询
{
  "query": {
    "bool": {
      "must": [
        {"term": {"knowledge_graph_kwd": "relation"}},
        {"knn": {"vector": query_vector, "k": 56}}
      ]
    }
  }
}

# 返回
relations_from_txt = {
  ("aspirin", "stomach irritation"): {
    "sim": 0.85,
    "pagerank": 0.0021,
    "description": "Aspirin can cause stomach irritation..."
  },
  ("aspirin", "bleeding"): {
    "sim": 0.78,
    "pagerank": 0.0019,
    "description": "Aspirin inhibits blood clotting..."
  }
}
```

#### 4.2.3 N-hop 遍历

从查询实体出发，遍历图谱获取邻居：

```python
# 从 n_hop_ents 字段获取
for entity in entities_from_query.values():
    for neighbor in entity["n_hop_ents"]:
        path = neighbor["path"]  # ["aspirin", "COX-2", "inflammation"]
        weights = neighbor["weights"]  # PageRank 权重
        # 构建 nhop_pathes
```

**距离衰减**：
```python
sim_score = entity_sim / (2 + hop_distance)
```

#### 4.2.4 社区检索

**检索社区报告**（`_community_retrieval_()`）

```python
# ES 查询
{
  "query": {
    "bool": {
      "must": [
        {"term": {"knowledge_graph_kwd": "community_report"}},
        {"terms": {"entities_kwd": ["aspirin", "stomach irritation"]}}
      ]
    }
  },
  "sort": [{"weight_flt": "desc"}]
}

# 返回
community_reports = [
  {
    "docnm_kwd": "NSAID Drug Interactions Community",
    "content_with_weight": {
      "report": "This community discusses NSAIDs and their side effects...",
      "evidences": ["Evidence 1: Study on aspirin...", ...]
    }
  }
]
```

### 4.3 结果排序与融合

#### 4.3.1 实体排序

```python
# 公式：P(E|Q) = P(E) × P(Q|E) = pagerank × sim
# 类型匹配加权
if entity in entities_from_types:
    entities_from_query[entity]["sim"] *= 2

# 排序
entities_sorted = sorted(
    entities_from_query.items(),
    key=lambda x: x[1]["sim"] * x[1]["pagerank"],
    reverse=True
)[:ent_topn]  # 取 top 6
```

#### 4.3.2 关系排序

```python
# 关系加权
for (from_ent, to_ent), rel in relations_from_txt.items():
    boost = 0
    if from_ent in entities_from_types: boost += 1
    if to_ent in entities_from_types: boost += 1
    if (from_ent, to_ent) in nhop_pathes: boost += nhop_sim
    
    rel["sim"] *= (boost + 1)

# 排序
relations_sorted = sorted(
    relations_from_txt.items(),
    key=lambda x: x[1]["sim"] * x[1]["pagerank"],
    reverse=True
)[:rel_topn]  # 取 top 6
```

#### 4.3.3 上下文构建

**格式化输出**（CSV + Markdown）：

```python
# 实体表格
"""
---- Entities ----
,Entity,Score,Description
0,aspirin,0.41,A nonsteroidal anti-inflammatory drug (NSAID)...
1,stomach irritation,0.15,Inflammation of the stomach lining...
"""

# 关系表格
"""
---- Relations ----
,From Entity,To Entity,Score,Description
0,aspirin,stomach irritation,0.18,Aspirin can cause stomach irritation...
1,aspirin,bleeding,0.14,Aspirin inhibits blood clotting...
"""

# 社区报告
"""
---- Community Report ----
# 1. NSAID Drug Interactions Community
## Content
This community discusses NSAIDs and their side effects...
## Evidences
Evidence 1: Study on aspirin...
"""
```

#### 4.3.4 插入到检索结果

```python
# 在 dialog_service.py 中
if prompt_config.get("use_kg"):
    kg_chunk = settings.kg_retriever.retrieval(
        question, tenant_ids, kb_ids, embd_mdl, llm
    )
    if kg_chunk["content_with_weight"]:
        kbinfos["chunks"].insert(0, kg_chunk)  # 插入首位
```

### 4.4 LLM 生成

**上下文注入**：

```python
knowledge_context = """
---- Entities ----
aspirin: A nonsteroidal anti-inflammatory drug (NSAID)...
stomach irritation: Inflammation of the stomach lining...

---- Relations ----
aspirin -> stomach irritation: Aspirin can cause stomach irritation...
aspirin -> bleeding: Aspirin inhibits blood clotting...

---- Community Report ----
NSAID Drug Interactions Community: This community discusses...

---- Document Chunks ----
[1] Aspirin is commonly used for pain relief...
[2] Side effects include gastrointestinal problems...
"""

# LLM Prompt
system_prompt = f"""
You are a helpful assistant. Use the following knowledge to answer the question.

{knowledge_context}

Query: What are the side effects of aspirin?
"""
```

## 5. 关键数据结构

### 5.1 知识图谱存储

#### 5.1.1 实体（Entity）

**Elasticsearch 索引**：`ragflow_{tenant_id}` / `knowledge_graph_kwd: "entity"`

```json
{
  "chunk_id": "ent_uuid_123",
  "kb_id": "kb_001",
  "knowledge_graph_kwd": "entity",
  "entity_kwd": "aspirin",
  "entity_type_kwd": "drug",
  "rank_flt": 0.0045,  // PageRank
  "content_with_weight": "{\"description\": \"A nonsteroidal...\", \"n_hop_ents\": [...]}"
}
```

**NetworkX 图**（内存）：

```python
G = nx.DiGraph()
G.add_node("aspirin", type="drug", pagerank=0.0045)
```

#### 5.1.2 关系（Relation）

**Elasticsearch 索引**：`ragflow_{tenant_id}` / `knowledge_graph_kwd: "relation"`

```json
{
  "chunk_id": "rel_uuid_456",
  "kb_id": "kb_001",
  "knowledge_graph_kwd": "relation",
  "from_entity_kwd": "aspirin",
  "to_entity_kwd": "stomach irritation",
  "weight_int": 21,  // PageRank
  "content_with_weight": "{\"description\": \"Aspirin can cause...\"}"
}
```

**NetworkX 图**（内存）：

```python
G.add_edge("aspirin", "stomach irritation", weight=0.0021, description="...")
```

#### 5.1.3 社区报告（Community Report）

**Elasticsearch 索引**：`ragflow_{tenant_id}` / `knowledge_graph_kwd: "community_report"`

```json
{
  "chunk_id": "comm_uuid_789",
  "kb_id": "kb_001",
  "knowledge_graph_kwd": "community_report",
  "docnm_kwd": "NSAID Drug Interactions Community",
  "entities_kwd": ["aspirin", "ibuprofen", "stomach irritation"],
  "weight_flt": 0.85,
  "content_with_weight": "{\"report\": \"...\", \"evidences\": [...]}"
}
```

#### 5.1.4 子图（Subgraph）

用于 UI 可视化：

```json
{
  "knowledge_graph_kwd": "graph",
  "content_with_weight": {
    "nodes": [
      {"id": "aspirin", "pagerank": 0.0045},
      {"id": "stomach irritation", "pagerank": 0.0032}
    ],
    "edges": [
      {"source": "aspirin", "target": "stomach irritation", "weight": 0.0021}
    ]
  }
}
```

### 5.2 检索结果结构

**KGSearch.retrieval() 返回**：

```python
{
  "chunk_id": "kg_chunk_uuid",
  "content_ltks": "",  # 空
  "content_with_weight": """
    ---- Entities ----
    aspirin,0.41,A nonsteroidal...
    
    ---- Relations ----
    aspirin,stomach irritation,0.18,Aspirin can cause...
    
    ---- Community Report ----
    # 1. NSAID Drug Interactions Community
    ## Content
    This community discusses...
  """,
  "doc_id": "",  # 空
  "docnm_kwd": "Related content in Knowledge Graph",
  "kb_id": ["kb_001"],
  "important_kwd": [],
  "image_id": "",
  "similarity": 1.0,
  "vector_similarity": 1.0,
  "term_similarity": 0,
  "vector": [],
  "positions": []
}
```

## 6. 性能优化

### 6.1 索引优化

**实体索引优化**：

```json
{
  "mappings": {
    "properties": {
      "entity_kwd": {"type": "keyword"},
      "entity_type_kwd": {"type": "keyword"},
      "rank_flt": {"type": "float"},
      "q_768_vec": {"type": "dense_vector", "dims": 768}
    }
  }
}
```

**关系索引优化**：

```json
{
  "mappings": {
    "properties": {
      "from_entity_kwd": {"type": "keyword"},
      "to_entity_kwd": {"type": "keyword"},
      "weight_int": {"type": "integer"}
    }
  }
}
```

### 6.2 查询优化

**分页查询**：
- 实体检索：`topk=56`
- 关系检索：`topk=56`
- 社区检索：`topk=1`

**相似度阈值**：
- 实体相似度：`0.3`（默认）
- 关系相似度：`0.3`（默认）
- 向量相似度：`0.1`（降级重试）

**Token 预算管理**：

```python
max_token = 8196
for entity in entities:
    max_token -= num_tokens_from_string(str(entity))
    if max_token <= 0:
        entities = entities[:-1]
        break
```

### 6.3 缓存策略

**Redis 缓存**：
- 实体向量：TTL 1 小时
- 查询改写结果：TTL 30 分钟
- 社区报告：TTL 24 小时

**MinIO 持久化**：
- 图谱序列化：`{kb_id}/graph.pkl`
- 社区结构：`{kb_id}/communities.json`

## 7. 监控与调试

### 7.1 日志记录

**关键日志点**：

```python
# graphrag/search.py
logging.info(f"Q: {qst}, Types: {ty_kwds}, Entities: {ents}")
logging.info(f"Retrieved entities: {list(ents_from_query.keys())}")
logging.info(f"Retrieved relations: {list(rels_from_txt.keys())}")
logging.info(f"Retrieved entities from types({ty_kwds}): {list(ents_from_types.keys())}")
logging.info(f"Retrieved N-hops: {list(nhop_pathes.keys())}")
```

### 7.2 性能指标

**检索耗时分解**：

```python
# dialog_service.py
retrieval_start = timer()

# 向量检索
vector_search_time = timer() - retrieval_start

# 图谱检索
if use_kg:
    kg_search_start = timer()
    kg_chunk = settings.kg_retriever.retrieval(...)
    kg_search_time = timer() - kg_search_start

retrieval_ts = timer()
total_retrieval_time = retrieval_ts - retrieval_start
```

**关键指标**：
- 查询改写耗时：`~500ms`（LLM 调用）
- 实体检索耗时：`~100ms`（ES 向量检索）
- 关系检索耗时：`~80ms`（ES 向量检索）
- N-hop 遍历耗时：`~50ms`（内存计算）
- 社区检索耗时：`~60ms`（ES 查询）
- 总检索耗时：`~800ms`（含 LLM）

### 7.3 调试工具

**测试脚本**：

```bash
# 测试图谱检索
cd /path/to/ragflow
source .venv/bin/activate
export PYTHONPATH=$(pwd)

python -m graphrag.search \
  -t tenant_id_123 \
  -d kb_id_456 \
  -q "What are the side effects of aspirin?"
```

**查询 ES 索引**：

```bash
# 查看实体
curl -X POST "localhost:9200/ragflow_tenant_123/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"term": {"knowledge_graph_kwd": "entity"}}, "size": 10}'

# 查看关系
curl -X POST "localhost:9200/ragflow_tenant_123/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"term": {"knowledge_graph_kwd": "relation"}}, "size": 10}'
```

## 8. 最佳实践

### 8.1 何时使用 GraphRAG

**适合场景**：
- ✅ 需要多跳推理的复杂问题
- ✅ 实体关系密集的领域（医药、金融、法律）
- ✅ 需要解释性答案（引用实体和关系）
- ✅ 需要社区级别的知识汇总

**不适合场景**：
- ❌ 简单的关键词查找
- ❌ 时间敏感的实时查询（图谱构建需要时间）
- ❌ 文档数量少于 100 篇
- ❌ 文档中实体关系稀疏

### 8.2 配置建议

**小型知识库（< 1000 文档）**：
```python
ent_topn = 4
rel_topn = 4
comm_topn = 1
ent_sim_threshold = 0.4
rel_sim_threshold = 0.4
```

**中型知识库（1000 - 10000 文档）**：
```python
ent_topn = 6
rel_topn = 6
comm_topn = 1
ent_sim_threshold = 0.3
rel_sim_threshold = 0.3
```

**大型知识库（> 10000 文档）**：
```python
ent_topn = 8
rel_topn = 8
comm_topn = 2
ent_sim_threshold = 0.25
rel_sim_threshold = 0.25
```

### 8.3 质量保证

**检查图谱质量**：

```python
# 通过 API 获取图谱可视化
GET /api/v1/kb/{kb_id}/knowledge_graph

# 检查
# 1. 节点数量：应 < 10000（过多会影响性能）
# 2. 平均度数：应在 2-10 之间（过稀疏或过密集都不好）
# 3. 社区数量：应在 5-50 之间
# 4. PageRank 分布：应呈幂律分布
```

**删除低质量图谱**：

```http
DELETE /api/v1/kb/{kb_id}/knowledge_graph
```

然后重新构建。

## 9. 故障排查

### 9.1 常见问题

**问题 1：GraphRAG 任务一直卡在进行中**

```bash
# 检查 Task 表
SELECT * FROM task WHERE id = 'task_abc123xyz';

# 检查 Redis 分布式锁
redis-cli KEYS "GRAPHRAG:*"
redis-cli DEL "GRAPHRAG:kb_001"  # 清除锁
```

**问题 2：图谱检索返回空结果**

```bash
# 检查 ES 索引
curl -X GET "localhost:9200/ragflow_tenant_123/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"term": {"knowledge_graph_kwd": "entity"}}}'

# 如果没有结果，重新构建图谱
POST /api/v1/kb/run_graphrag {"kb_id": "kb_001"}
```

**问题 3：查询改写失败**

```python
# 检查日志
grep "query_rewrite" /var/log/ragflow/api.log

# 常见原因：
# - LLM API 调用失败
# - Token 超限
# - Prompt 格式错误
```

### 9.2 性能问题

**问题：图谱检索耗时过长（> 2s）**

**排查步骤**：

1. 检查实体数量
   ```bash
   curl -X GET "localhost:9200/ragflow_tenant_123/_count" \
     -d '{"query": {"term": {"knowledge_graph_kwd": "entity"}}}'
   ```

2. 优化 PageRank 计算
   ```python
   # 增加迭代次数或调整阻尼系数
   nx.pagerank(G, alpha=0.85, max_iter=200)
   ```

3. 减少 topk 参数
   ```python
   ent_topn = 4  # 从 6 降到 4
   rel_topn = 4  # 从 6 降到 4
   ```

4. 启用缓存
   ```python
   # Redis 缓存实体向量
   redis.setex(f"ent_vec:{entity}", 3600, vector)
   ```

## 10. 未来增强

### 10.1 计划功能

- [ ] 时态知识图谱：支持时间维度
- [ ] 多模态图谱：支持图像、视频实体
- [ ] 动态图谱更新：增量更新而非全量重建
- [ ] 自定义实体类型：用户定义实体本体
- [ ] 图谱推理：基于规则的逻辑推理

### 10.2 性能优化计划

- [ ] GPU 加速 PageRank 计算
- [ ] 图谱索引优化（HNSW for graphs）
- [ ] 异步图谱构建（流式处理）
- [ ] 分布式图谱存储（GraphDB 替代 NetworkX）

## 附录 A：完整检索流程图

参见：[06-graphrag-retrieval-sequence.puml](06-graphrag-retrieval-sequence.puml)

## 附录 B：API 参考

### B.1 GraphRAG API

| 端点 | 方法 | 描述 |
|-----|------|------|
| `/api/v1/kb/run_graphrag` | POST | 启动图谱构建任务 |
| `/api/v1/kb/trace_graphrag` | GET | 追踪任务进度 |
| `/api/v1/kb/{kb_id}/knowledge_graph` | GET | 获取图谱可视化数据 |
| `/api/v1/kb/{kb_id}/knowledge_graph` | DELETE | 删除知识图谱 |
| `/api/v1/datasets/{id}/run_graphrag` | POST | SDK 接口 |

### B.2 检索 API

| 端点 | 方法 | 描述 |
|-----|------|------|
| `/api/v1/dialog/set` | POST | 设置对话配置（含 use_kg） |
| `/api/v1/dialog/chat` | POST | 发起对话（自动使用图谱） |

## 附录 C：配置示例

### C.1 完整对话配置

```json
{
  "dialog_id": "dialog_123",
  "kb_ids": ["kb_001", "kb_002"],
  "top_n": 6,
  "top_k": 1024,
  "similarity_threshold": 0.2,
  "vector_similarity_weight": 0.3,
  "llm_setting": {
    "model_name": "gpt-4",
    "max_tokens": 2048,
    "temperature": 0.7
  },
  "prompt_config": {
    "use_kg": true,
    "toc_enhance": false,
    "quote": true,
    "empty_response": "I don't have enough information to answer that.",
    "system": "You are a helpful medical assistant. Use the knowledge graph to provide accurate answers with citations."
  }
}
```

### C.2 GraphRAG 检索参数

```python
# 在代码中配置
settings.kg_retriever.retrieval(
    question=user_query,
    tenant_ids=tenant_ids,
    kb_ids=kb_ids,
    emb_mdl=embedding_model,
    llm=chat_model,
    max_token=8196,
    ent_topn=6,
    rel_topn=6,
    comm_topn=1,
    ent_sim_threshold=0.3,
    rel_sim_threshold=0.3
)
```

## 附录 D：数据库 Schema

### D.1 Knowledgebase 表

```sql
CREATE TABLE knowledgebase (
  id VARCHAR(32) PRIMARY KEY,
  tenant_id VARCHAR(32) NOT NULL,
  name VARCHAR(128) NOT NULL,
  embd_id VARCHAR(32),
  graphrag_task_id VARCHAR(32),  -- GraphRAG 任务 ID
  graphrag_task_finish_at DATETIME,  -- 完成时间
  create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_graphrag_task (graphrag_task_id)
);
```

### D.2 Task 表

```sql
CREATE TABLE task (
  id VARCHAR(32) PRIMARY KEY,
  doc_id VARCHAR(32),
  from_page INT DEFAULT 0,
  to_page INT DEFAULT -1,
  progress FLOAT DEFAULT 0.0,  -- -1: 失败, 0-1: 进度
  progress_msg TEXT,
  create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
  update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## 附录 E：参考资料

- [NetworkX Documentation](https://networkx.org/documentation/)
- [Elasticsearch Vector Search](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html)
- [Microsoft GraphRAG](https://github.com/microsoft/graphrag)
- [HKUDS Light GraphRAG](https://github.com/HKUDS/LightRAG)
- [PageRank Algorithm](https://en.wikipedia.org/wiki/PageRank)
- [Leiden Community Detection](https://www.nature.com/articles/s41598-019-41695-z)

---

**文档版本**：1.0  
**最后更新**：2024-01-15  
**作者**：RAGFlow Team
