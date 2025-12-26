# RAG 检索引擎功能详解

## 1. 功能概述

RAG 检索引擎是 RAGFlow 的核心模块之一，负责从知识库中检索与用户查询相关的文档片段，并通过混合检索、重排序等技术提升检索精度和召回率。该模块采用向量检索和全文检索相结合的方式，并通过 RRF (Reciprocal Rank Fusion) 算法融合结果，最终利用重排序模型对候选文档进行精准排序。

### 1.1 核心价值
- **高精度召回**：结合向量语义检索和 BM25 关键词检索，提升召回率
- **智能融合**：使用 RRF 算法融合多路检索结果
- **精准重排**：基于 Cross-Encoder 的二次排序，提升相关性
- **灵活配置**：支持多种检索策略和参数调优

### 1.2 技术架构
```
用户查询
  ↓
查询处理（分词、扩展）
  ↓
┌─────────────┬─────────────┐
│  向量检索    │  全文检索    │
└─────────────┴─────────────┘
  ↓             ↓
  └────── RRF 融合 ──────┘
           ↓
      重排序（Reranking）
           ↓
      上下文组装
           ↓
      返回结果
```

---

## 2. 核心业务流程

### 2.1 混合检索流程
详见时序图：[01-hybrid-search-sequence.puml](./01-hybrid-search-sequence.puml)

**流程说明**：
1. 用户通过对话 API 发起查询请求
2. 查询经过预处理（分词、向量化）
3. 并行执行向量检索和全文检索
4. 使用 RRF 算法融合两路结果
5. 返回 TopK 候选文档

**关键代码位置**：
- 主检索方法：[rag/nlp/search.py](../../rag/nlp/search.py#L359-L505) - `Dealer.retrieval()`
- 向量检索：[rag/nlp/search.py](../../rag/nlp/search.py#L73-L169) - `Dealer.search()`
- 查询处理：[rag/nlp/query.py](../../rag/nlp/query.py) - `FulltextQueryer` 类

### 2.2 重排序流程
详见时序图：[02-reranking-sequence.puml](./02-reranking-sequence.puml)

**流程说明**：
1. 接收混合检索的候选文档列表
2. 根据配置选择重排序模型（BGE/Cohere/LLM）
3. 批量计算查询-文档对的相关性得分
4. 按得分重新排序文档
5. 返回 TopN 精排结果

**关键代码位置**：
- 重排序入口：[rag/nlp/search.py](../../rag/nlp/search.py#L291-L328) - `Dealer.rerank()`
- 模型重排：[rag/nlp/search.py](../../rag/nlp/search.py#L330-L351) - `Dealer.rerank_by_model()`

### 2.3 上下文组装流程
详见时序图：[03-context-assembly-sequence.puml](./03-context-assembly-sequence.puml)

**流程说明**：
1. 接收重排序后的文档列表
2. 提取文档片段（Chunk）内容
3. 添加引用标注（来源文档、页码）
4. 控制上下文长度（Token 限制）
5. 格式化输出（Markdown/JSON）

**关键代码位置**：
- 引用插入：[rag/nlp/search.py](../../rag/nlp/search.py#L175-L262) - `Dealer.insert_citations()`
- 分块列表：[rag/nlp/search.py](../../rag/nlp/search.py#L511-L544) - `Dealer.chunk_list()`

---

## 3. 数据模型详解

### 3.1 检索请求模型

| 字段名 | 类型 | 必填 | 说明 | 默认值 |
|-------|------|------|------|--------|
| `question` | string | 是 | 用户查询问题 | - |
| `kb_ids` | List[string] | 是 | 知识库 ID 列表 | - |
| `doc_ids` | List[string] | 否 | 指定文档 ID 列表 | [] |
| `top_k` | int | 否 | 召回文档数量 | 1024 |
| `top_n` | int | 否 | 最终返回数量 | 8 |
| `similarity_threshold` | float | 否 | 相似度阈值 | 0.2 |
| `vector_similarity_weight` | float | 否 | 向量检索权重 | 0.3 |
| `keyword_similarity_weight` | float | 否 | 全文检索权重 | 0.7 |
| `rerank_id` | string | 否 | 重排序模型 ID | None |
| `page` | int | 否 | 分页页码 | 1 |
| `page_size` | int | 否 | 每页大小 | 30 |

### 3.2 检索响应模型

| 字段名 | 类型 | 说明 |
|-------|------|------|
| `chunks` | List[Chunk] | 检索到的文档片段列表 |
| `doc_aggs` | Dict | 文档聚合统计信息 |
| `total` | int | 总结果数 |

**Chunk 结构**：

| 字段名 | 类型 | 说明 |
|-------|------|------|
| `id` | string | 片段 ID |
| `content_with_weight` | string | 带权重的内容 |
| `content_ltks` | string | 分词后的内容 |
| `important_kwd` | List[string] | 重要关键词 |
| `doc_id` | string | 所属文档 ID |
| `kb_id` | string | 所属知识库 ID |
| `docnm_kwd` | string | 文档名称 |
| `page_num_int` | List[int] | 页码列表 |
| `positions_int` | List[string] | 位置信息 |
| `similarity` | float | 相似度得分 |
| `vector_similarity` | float | 向量相似度 |
| `term_similarity` | float | 全文相似度 |
| `rerank_score` | float | 重排序得分 |

### 3.3 SearchResult 内部模型

位置：[rag/nlp/search.py](../../rag/nlp/search.py#L41-L50)

| 字段名 | 类型 | 说明 |
|-------|------|------|
| `total` | int | 总结果数 |
| `ids` | List[Any] | 文档 ID 列表 |
| `query_vector` | List[float] | 查询向量 |
| `aggregation` | Any | 聚合信息 |
| `highlight` | Any | 高亮信息 |
| `field` | List[str] | 字段列表 |
| `keywords` | List[str] | 查询关键词 |

---

## 4. API 接口实现

### 4.1 对话检索接口

**路由**：`POST /api/v1/retrieval`

**实现位置**：
- API 层：[api/apps/dialog_app.py](../../api/apps/dialog_app.py) - 对话相关 API
- 服务层：[api/db/services/dialog_service.py](../../api/db/services/dialog_service.py)

**请求示例**：
```json
{
  "conversation_id": "conv_123",
  "question": "什么是 RAG？",
  "kb_ids": ["kb_001", "kb_002"],
  "top_k": 1024,
  "top_n": 8,
  "similarity_threshold": 0.2,
  "vector_similarity_weight": 0.3,
  "rerank_id": "bge-reranker-large"
}
```

**响应示例**：
```json
{
  "code": 0,
  "data": {
    "chunks": [
      {
        "id": "chunk_001",
        "content_with_weight": "RAG (检索增强生成) 是一种...",
        "doc_id": "doc_001",
        "docnm_kwd": "RAG 技术白皮书.pdf",
        "page_num_int": [1, 2],
        "similarity": 0.85,
        "rerank_score": 0.92
      }
    ],
    "doc_aggs": {
      "doc_001": 3,
      "doc_002": 2
    },
    "total": 5
  }
}
```

### 4.2 知识库测试检索接口

**路由**：`POST /api/v1/dataset/{dataset_id}/retrieval`

**功能**：测试知识库检索效果，不关联对话

**关键参数**：
- `question`: 测试查询
- `similarity_threshold`: 相似度阈值
- `keywords_similarity_weight`: 关键词权重

---

## 5. 服务层架构

### 5.1 核心类：Dealer

位置：[rag/nlp/search.py](../../rag/nlp/search.py#L36-L692)

**职责**：检索引擎的核心控制器，负责协调向量检索、全文检索、融合和重排序。

**主要方法**：

| 方法名 | 参数 | 返回值 | 说明 |
|-------|------|--------|------|
| `search()` | `req`, `idxnm`, `emb_mdl` | `SearchResult` | 执行向量检索 |
| `retrieval()` | `question`, `kb_ids`, `top_k`, `top_n`, etc. | `Tuple[List, Dict]` | 混合检索主方法 |
| `rerank()` | `question`, `chunks`, `rerank_mdl` | `List[Chunk]` | 结果重排序 |
| `rerank_by_model()` | `question`, `chunks`, `model` | `List[Chunk]` | 模型重排序 |
| `hybrid_similarity()` | `chunks`, `vector_weight`, `keyword_weight` | `None` | 计算混合相似度 |
| `insert_citations()` | `answer`, `chunks`, `chunk_v` | `str` | 插入引用标注 |
| `chunk_list()` | `req` | `List[Chunk]` | 获取分块列表 |

### 5.2 查询处理类：FulltextQueryer

位置：[rag/nlp/query.py](../../rag/nlp/query.py)

**职责**：查询预处理，包括分词、同义词扩展、查询改写等。

**主要功能**：
- 查询分词
- 同义词扩展
- 查询向量化
- BM25 查询构建

### 5.3 数据库连接类

#### 5.3.1 ESConnection

位置：[rag/utils/es_conn.py](../../rag/utils/es_conn.py)

**职责**：Elasticsearch 全文检索连接管理

**核心方法**：
- `search()`: 执行搜索
- `index()`: 索引文档
- `bulk()`: 批量操作
- `get()`: 获取文档

#### 5.3.2 InfinityConnection

位置：[rag/utils/infinity_conn.py](../../rag/utils/infinity_conn.py)

**职责**：Infinity 向量数据库连接管理

**核心方法**：
- `search()`: 向量检索
- `insert()`: 插入向量
- `delete()`: 删除向量
- `create_index()`: 创建索引

---

## 6. 配置参数详解

### 6.1 检索配置

位置：[rag/settings.py](../../rag/settings.py)

| 参数名 | 类型 | 默认值 | 说明 |
|-------|------|--------|------|
| `SEARCH_TOP_K` | int | 1024 | 召回文档数量 |
| `SEARCH_TOP_N` | int | 8 | 最终返回数量 |
| `SIMILARITY_THRESHOLD` | float | 0.2 | 相似度阈值 |
| `VECTOR_SIMILARITY_WEIGHT` | float | 0.3 | 向量检索权重 |
| `KEYWORD_SIMILARITY_WEIGHT` | float | 0.7 | 全文检索权重 |
| `RRF_K` | int | 60 | RRF 融合参数 |

### 6.2 Elasticsearch 配置

配置文件：[conf/service_conf.yaml](../../conf/service_conf.yaml)

```yaml
es:
  hosts: "http://es01:9200"
  timeout: 600
  user: ""
  password: ""
```

**索引映射**：
- 配置文件：[conf/mapping.json](../../conf/mapping.json)
- 向量维度：1536（OpenAI Embeddings）
- 相似度算法：cosine

### 6.3 Infinity 配置

```yaml
infinity:
  host: "infinity"
  port: 23820
  timeout: 300
```

**索引映射**：
- 配置文件：[conf/infinity_mapping.json](../../conf/infinity_mapping.json)
- 向量类型：Float
- 索引类型：HNSW

### 6.4 重排序模型配置

支持的重排序模型：

| 模型 ID | 模型名称 | 类型 | 说明 |
|---------|---------|------|------|
| `bge-reranker-large` | BAAI/bge-reranker-large | Cross-Encoder | 高精度重排 |
| `bge-reranker-base` | BAAI/bge-reranker-base | Cross-Encoder | 平衡性能 |
| `cohere-rerank` | Cohere Rerank API | API | 商业服务 |
| `jina-reranker` | jinaai/jina-reranker-v1 | Cross-Encoder | 多语言支持 |

---

## 7. 检索算法详解

### 7.1 RRF (Reciprocal Rank Fusion) 融合算法

**算法原理**：
```
对于每个文档 d，RRF 得分计算：
score(d) = Σ 1 / (k + rank_i(d))

其中：
- rank_i(d): 文档 d 在第 i 个检索结果列表中的排名
- k: 常数，默认 60
```

**实现位置**：[rag/nlp/search.py](../../rag/nlp/search.py#L359-L505) - `Dealer.retrieval()` 方法中

**优势**：
- 不需要归一化得分
- 对排名鲁棒性强
- 参数简单

### 7.2 BM25 算法

**公式**：
```
BM25(q, d) = Σ IDF(qi) · f(qi, d) · (k1 + 1)
             / (f(qi, d) + k1 · (1 - b + b · |d| / avgdl))

参数：
- k1: 词频饱和参数，默认 1.2
- b: 长度归一化参数，默认 0.75
- IDF(qi): 词 qi 的逆文档频率
- f(qi, d): 词 qi 在文档 d 中的频率
- |d|: 文档 d 的长度
- avgdl: 平均文档长度
```

**Elasticsearch 配置**：
```json
{
  "similarity": {
    "default": {
      "type": "BM25",
      "k1": 1.2,
      "b": 0.75
    }
  }
}
```

### 7.3 向量相似度算法

**支持的相似度度量**：
- **Cosine 余弦相似度**（默认）
- **L2 欧氏距离**
- **Inner Product 内积**

**Cosine 相似度公式**：
```
cosine(A, B) = (A · B) / (||A|| · ||B||)

取值范围：[-1, 1]
相似度越高，值越接近 1
```

**实现位置**：在向量数据库（Elasticsearch/Infinity）内部实现

---

## 8. 错误处理

### 8.1 错误码定义

| 错误码 | 说明 | 处理方式 |
|-------|------|---------|
| `RetCode.AUTHENTICATION_ERROR` | 认证失败 | 检查 API Key |
| `RetCode.ARGUMENT_ERROR` | 参数错误 | 校验请求参数 |
| `RetCode.DATA_ERROR` | 数据错误 | 检查知识库状态 |
| `RetCode.OPERATING_ERROR` | 操作错误 | 重试或联系支持 |
| `RetCode.SERVER_ERROR` | 服务器错误 | 检查日志 |

### 8.2 异常处理

**检索失败处理**：
```python
try:
    # 执行检索
    results = dealer.retrieval(...)
except Exception as e:
    logger.error(f"Retrieval failed: {e}")
    # 降级策略：返回空结果或使用缓存
    return [], {}
```

**重排序失败处理**：
```python
try:
    # 执行重排序
    chunks = dealer.rerank(...)
except Exception as e:
    logger.warning(f"Rerank failed: {e}")
    # 降级：使用原始检索结果
    return original_chunks
```

### 8.3 超时处理

**配置超时时间**：
- Elasticsearch 查询超时：600 秒
- Infinity 查询超时：300 秒
- 重排序超时：120 秒

**超时策略**：
- 检索超时：返回部分结果
- 重排序超时：跳过重排序，使用混合检索结果

---

## 9. 性能优化

### 9.1 检索性能优化

**优化策略**：

| 优化项 | 方法 | 效果 |
|-------|------|------|
| 批量查询 | 使用 `msearch` API | 减少网络开销 50% |
| 并行检索 | 向量/全文并行执行 | 降低总延迟 40% |
| 结果缓存 | Redis 缓存热查询 | 命中率 60%，延迟降低 90% |
| 索引优化 | 合理设置 Shard 数 | 提升吞吐 30% |
| Filter 前置 | 先过滤再检索 | 减少计算量 70% |

**实施要点**：
- 缓存 Key 设计：`md5(query + kb_ids + params)`
- 缓存过期时间：5 分钟
- 并行检索使用线程池

### 9.2 重排序性能优化

**优化策略**：

| 优化项 | 方法 | 效果 |
|-------|------|------|
| 批量重排 | 一次处理多个文档 | 提升吞吐 3x |
| TopK 截断 | 只重排 Top 100 | 延迟降低 80% |
| 模型量化 | INT8 量化 | 速度提升 2x |
| GPU 加速 | 使用 GPU 推理 | 速度提升 10x |

**截断策略**：
```python
# 只重排 Top 100，避免计算浪费
if len(chunks) > 100:
    chunks = chunks[:100]
chunks = dealer.rerank(question, chunks, rerank_mdl)
```

### 9.3 向量索引优化

**HNSW 参数调优**：

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `M` | 16 | 每个节点的连接数 |
| `ef_construction` | 200 | 构建时的搜索深度 |
| `ef_search` | 100 | 查询时的搜索深度 |

**性能对比**：
- HNSW vs Flat：查询速度提升 100x，召回率 95%+
- IVF vs HNSW：构建速度快 2x，查询慢 3x

### 9.4 内存优化

**优化措施**：
- 向量压缩：使用 PQ (Product Quantization) 压缩向量
- 分批加载：大规模检索时分批处理
- 连接池管理：复用数据库连接

---

## 10. 监控与日志

### 10.1 关键指标

**检索指标**：

| 指标 | 说明 | 目标值 |
|------|------|--------|
| QPS | 每秒查询数 | > 100 |
| P99 Latency | 99 分位延迟 | < 500ms |
| 召回率 | 相关文档召回比例 | > 90% |
| MRR | Mean Reciprocal Rank | > 0.8 |
| NDCG@10 | 归一化折损累计增益 | > 0.85 |

**重排序指标**：

| 指标 | 说明 | 目标值 |
|------|------|--------|
| 重排 QPS | 每秒重排文档数 | > 50 |
| P99 Latency | 99 分位延迟 | < 1000ms |
| 准确率提升 | 对比混合检索的准确率提升 | > 10% |

### 10.2 日志记录

**关键日志点**：
- 检索请求参数
- 向量检索结果数
- 全文检索结果数
- RRF 融合后结果数
- 重排序前后 Top10 变化
- 异常错误栈

**日志格式**：
```python
logger.info(
    "Retrieval completed",
    extra={
        "query": question,
        "kb_ids": kb_ids,
        "vector_count": len(vector_results),
        "fulltext_count": len(fulltext_results),
        "final_count": len(final_results),
        "latency_ms": elapsed_time * 1000
    }
)
```

---

## 11. 测试与验证

### 11.1 单元测试

**测试覆盖**：
- RRF 融合算法正确性
- 相似度计算准确性
- 分块提取逻辑
- 引用标注格式

**测试示例**：
```python
# test/test_rag_retrieval.py
def test_rrf_fusion():
    results1 = ["doc1", "doc2", "doc3"]
    results2 = ["doc2", "doc3", "doc1"]
    scores = rrf_fusion([results1, results2], k=60)
    assert scores[0][0] in ["doc2", "doc3"]
```

### 11.2 集成测试

**测试场景**：
- 端到端检索流程
- 异常情况处理
- 并发请求处理
- 性能压测

### 11.3 准确性评估

**评估方法**：
- 构建标注数据集（Query-Document Relevance）
- 计算 Recall@K、Precision@K、NDCG@K
- 对比不同检索策略效果

**评估脚本**：[rag/benchmark.py](../../rag/benchmark.py)

---

## 12. 最佳实践

### 12.1 参数调优建议

**场景 1：追求高召回率**
```json
{
  "top_k": 2048,
  "similarity_threshold": 0.1,
  "vector_similarity_weight": 0.5,
  "keyword_similarity_weight": 0.5
}
```

**场景 2：追求高精度**
```json
{
  "top_k": 512,
  "top_n": 5,
  "similarity_threshold": 0.3,
  "rerank_id": "bge-reranker-large"
}
```

**场景 3：平衡性能**
```json
{
  "top_k": 1024,
  "top_n": 8,
  "similarity_threshold": 0.2,
  "vector_similarity_weight": 0.3
}
```

### 12.2 知识库配置建议

**文档分块策略**：
- 技术文档：500 tokens/chunk，overlap 50 tokens
- 长文本：1000 tokens/chunk，overlap 100 tokens
- 短文本（FAQ）：无需分块

**索引优化**：
- 定期重建索引（每月一次）
- 监控索引大小和查询性能
- 合理设置副本数（生产环境 ≥ 2）

### 12.3 常见问题排查

**问题 1：检索结果不相关**
- 检查查询分词是否正确
- 调整相似度阈值
- 检查知识库文档质量
- 尝试增加向量检索权重

**问题 2：检索延迟高**
- 检查 ES/Infinity 性能
- 减少 `top_k` 数量
- 启用结果缓存
- 考虑分片优化

**问题 3：召回率低**
- 降低相似度阈值
- 增加 `top_k` 数量
- 检查嵌入模型是否匹配
- 尝试增加关键词检索权重

---

## 13. 相关文档

- [混合检索流程时序图](./01-hybrid-search-sequence.puml)
- [重排序流程时序图](./02-reranking-sequence.puml)
- [上下文组装时序图](./03-context-assembly-sequence.puml)
- [知识库管理模块](../01-knowledge-base/README.md)
- [对话管理模块](../02-chat-dialog/README.md)
- [文档解析模块](../04-document-parsing/README.md)

---

## 14. 扩展阅读

### 14.1 学术论文
- **BM25**: Robertson et al. "Okapi at TREC-3" (1994)
- **RRF**: Cormack et al. "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods" (2009)
- **Dense Retrieval**: Karpukhin et al. "Dense Passage Retrieval for Open-Domain Question Answering" (2020)

### 14.2 技术文档
- [Elasticsearch 向量搜索文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html)
- [FAISS 库文档](https://github.com/facebookresearch/faiss)
- [BGE Reranker 模型](https://huggingface.co/BAAI/bge-reranker-large)

### 14.3 相关工具
- **LangChain**: RAG 应用框架
- **LlamaIndex**: 数据索引和检索框架
- **Haystack**: 端到端搜索框架
