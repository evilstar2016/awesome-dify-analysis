# RAG 检索引擎模块 (RAG Retrieval Module)

## 模块概述

RAG 检索引擎是 RAGFlow 的核心检索模块，实现了混合检索、重排序、上下文组装等功能。采用向量检索 + 全文检索的混合策略，通过 RRF 融合和重排序提升检索精度。

### 核心价值
- **混合检索**：向量检索 + BM25 全文检索
- **智能融合**：RRF (Reciprocal Rank Fusion) 结果融合
- **精准重排**：基于 Cross-Encoder 的重排序
- **灵活配置**：支持多种检索策略和参数调优

---

## 功能清单

### 1. 向量检索 (Vector Search)
- **密集向量检索** - Embedding 相似度搜索
- **ANN 算法** - HNSW、IVF、PQ 等近似最近邻
- **多向量支持** - 支持多个嵌入模型
- **过滤条件** - 知识库、文档、元数据过滤

### 2. 全文检索 (Full-Text Search)
- **BM25 算法** - 经典 TF-IDF 检索
- **Elasticsearch** - 分布式搜索引擎
- **分词器** - 中文分词（jieba）、英文分词
- **高亮显示** - 关键词高亮

### 3. 混合检索 (Hybrid Search)
- **双路召回** - 向量 + 全文并行检索
- **RRF 融合** - 倒数排名融合算法
- **权重配置** - 调整向量/全文权重
- **TopK 合并** - 结果去重和截断

### 4. 重排序 (Reranking)
- **Cross-Encoder** - 交叉编码器重排
- **BGE Reranker** - 高精度重排模型
- **Cohere Rerank** - 商业重排服务
- **LLM Rerank** - 基于 LLM 的重排序

### 5. 上下文组装 (Context Assembly)
- **分块选择** - 选择最相关分块
- **上下文窗口** - 控制上下文长度
- **引用标注** - 标注来源文档和页码
- **格式化输出** - Markdown、JSON 等格式

---

## 目录结构

```
rag/
  ├── app/                       # 各种文档解析应用
  │   ├── naive.py               # 通用解析
  │   ├── qa.py                  # 问答解析
  │   └── ...                    # 其他解析器
  ├── nlp/
  │   ├── search.py              # 检索与重排序 (Dealer 类)
  │   ├── query.py               # 查询处理与相似度计算
  │   └── rag_tokenizer.py       # 分词器
  ├── utils/
  │   ├── es_conn.py             # Elasticsearch 连接
  │   ├── infinity_conn.py       # Infinity 向量数据库连接
  │   └── ob_conn.py             # OceanBase 连接 (混合搜索)
  └── llm/                       # LLM 和嵌入模型
      └── ...                    # 模型抽象层
```

---

## 技术栈

### 向量搜索
- **Elasticsearch + vector plugin** - 向量搜索
- **Infinity** - 高性能向量数据库
- **HNSW** - 分层可导航小世界图
- **FAISS** - Facebook AI 相似度搜索

### 全文搜索
- **Elasticsearch** - 分布式搜索引擎
- **BM25** - Okapi BM25 算法
- **Analyzer** - ik_smart、ik_max_word

### 重排序
- **sentence-transformers** - Cross-Encoder
- **BGE Reranker** - BAAI 重排模型
- **Cohere API** - 商业重排服务

---

## 核心算法

### RRF 融合算法
```python
def rrf_fusion(results_list, k=60):
    scores = defaultdict(float)
    for results in results_list:
        for rank, doc_id in enumerate(results):
            scores[doc_id] += 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

### BM25 评分公式
```
score(D, Q) = Σ IDF(qi) · f(qi, D) · (k1 + 1) 
              / (f(qi, D) + k1 · (1 - b + b · |D| / avgdl))
```

---

## API 接口

### 检索接口
```python
def retrieve(
    query: str,
    kb_ids: List[str],
    top_k: int = 1024,
    top_n: int = 8,
    similarity_threshold: float = 0.2,
    vector_weight: float = 0.3,
    rerank_id: str = None
) -> dict:
    """
    混合检索并重排序
    
    Returns:
        {
            "chunks": [...],
            "doc_aggs": {...}
        }
    """
```

---

## 配置参数

### 检索配置
```json
{
  "top_k": 1024,
  "top_n": 8,
  "similarity_threshold": 0.2,
  "vector_similarity_weight": 0.3,
  "rerank_id": "bge-reranker-large",
  "keyword_similarity_weight": 0.7
}
```

### Elasticsearch 配置
```yaml
es:
  hosts: ["http://es01:9200"]
  timeout: 600
  vector_similarity: "cosine"
  vector_dimensions: 1536
```

---

## 性能优化

### 检索优化
- **批量查询** - 减少网络开销
- **缓存热查询** - Redis 缓存
- **索引优化** - 合理设计 ES 索引
- **并行检索** - 向量和全文并行

### 重排优化
- **批量重排** - 一次重排多个文档
- **截断策略** - 只重排 TopK 结果
- **模型量化** - 降低推理延迟

---

## 监控指标

- **检索 QPS** - 每秒查询数
- **平均延迟** - P50、P90、P99
- **召回率** - 相关文档召回比例
- **MRR** - Mean Reciprocal Rank
- **NDCG** - Normalized Discounted Cumulative Gain

---

## 相关文档

- [混合检索流程时序图](./01-hybrid-search-sequence.puml)
- [重排序流程时序图](./02-reranking-sequence.puml)
- [上下文组装时序图](./03-context-assembly-sequence.puml)
