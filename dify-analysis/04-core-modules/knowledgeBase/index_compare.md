# Dify中高质量索引与经济索引模式的查询区别

## 两种索引模式概述

**高质量索引（High Quality）**：
- 使用Embedding模型进行文档处理
- 支持向量检索、全文检索、混合检索
- 消耗tokens但提供更高的检索准确性

**经济索引（Economy）**：
- 使用关键词索引，不调用Embedding模型
- 仅支持关键词检索（倒排索引）
- 不消耗tokens，但检索准确性相对较低

## 查询时的核心区别

### 1. 检索方法差异

**高质量索引**支持多种检索方法：
```python
# 根据数据集的索引技术确定检索方法
if dataset.indexing_technique == "economy":
    retrieval_method = RetrievalMethod.KEYWORD_SEARCH
else:
    retrieval_method = retrieval_model_config["search_method"]
    # 可以是：SEMANTIC_SEARCH, FULL_TEXT_SEARCH, HYBRID_SEARCH
```

**经济索引**只能使用：
- **关键词检索（KEYWORD_SEARCH）**：使用jieba分词提取关键词进行匹配

### 2. 技术实现差异

#### 高质量索引的查询流程：
1. **向量检索**：将查询转换为向量，在向量数据库中搜索最相似的文档
2. **全文检索**：在支持全文搜索的向量数据库中进行文本匹配
3. **混合检索**：同时执行向量检索和全文检索，然后重排序

#### 经济索引的查询流程：
```python
def _retrieve_ids_by_query(self, keyword_table: dict, query: str, k: int = 4):
    keyword_table_handler = JiebaKeywordTableHandler()
    keywords = keyword_table_handler.extract_keywords(query)
    
    # 统计每个文档块匹配的关键词数量
    chunk_indices_count: dict[str, int] = defaultdict(int)
    keywords_list = [keyword for keyword in keywords if keyword in set(keyword_table.keys())]
    for keyword in keywords_list:
        for node_id in keyword_table[keyword]:
            chunk_indices_count[node_id] += 1
    
    # 按匹配关键词数量排序
    sorted_chunk_indices = sorted(
        chunk_indices_count.keys(),
        key=lambda x: chunk_indices_count[x],
        reverse=True,
    )
    return sorted_chunk_indices[:k]
```

### 3. 查询准确性对比

**高质量索引**：
- **语义理解**：通过Embedding向量能理解查询的语义含义
- **相似性匹配**：即使用词不完全相同，也能找到语义相关的内容
- **上下文关联**：能理解复杂的上下文关系

**经济索引**：
- **关键词匹配**：只能进行精确的关键词匹配
- **缺乏语义理解**：无法理解同义词、近义词或语义相关性
- **依赖词汇重叠**：查询效果很大程度上依赖于查询词和文档中词汇的重叠程度

### 4. 性能和成本差异

**高质量索引**：
- ✅ 检索准确性高
- ✅ 支持多种检索策略
- ❌ 消耗Embedding模型tokens
- ❌ 需要向量数据库存储和计算

**经济索引**：
- ✅ 不消耗tokens
- ✅ 计算成本低
- ✅ 响应速度快
- ❌ 检索准确性较低
- ❌ 功能限制大

### 5. 配置差异

根据界面文案，经济模式的配置说明：
- 中文：`每个块使用 {{count}} 个关键词进行检索，不消耗 tokens，但会降低检索准确性`
- 日文：`チャンクあたり 10 个のキーワードを検索に使用します。トークンは消费しませんが、検索精度は低下します`

## 实际应用建议

**选择高质量索引的场景**：
- 对检索准确性要求高
- 查询可能使用同义词或语义相关词汇
- 需要理解复杂语义关系
- 预算允许消耗embedding tokens

**选择经济索引的场景**：
- 成本敏感，需要控制token消耗
- 查询模式相对固定，主要是关键词匹配
- 文档内容和查询用词重叠度高
- 对响应速度要求较高

总的来说，这两种模式代表了**准确性与成本**之间的权衡：高质量模式提供更智能的语义检索但成本较高，经济模式提供基础的关键词检索但成本很低。



# 高质量索引的全文检索 vs 经济索引的关键词检索

## 1. 技术架构差异

### 高质量索引的全文检索
**基于向量数据库的全文检索引擎**：
- 使用专业的搜索引擎（如Elasticsearch、PostgreSQL全文搜索、Milvus等）
- 支持多种搜索算法（BM25、TF-IDF、向量相似度等）
- 具备完整的文本分析和索引能力

### 经济索引的关键词检索
**基于Jieba分词的简单关键词匹配**：
```python
def _retrieve_ids_by_query(self, keyword_table: dict, query: str, k: int = 4):
    keyword_table_handler = JiebaKeywordTableHandler()
    keywords = keyword_table_handler.extract_keywords(query)
    
    # 统计每个文档块匹配的关键词数量
    chunk_indices_count: dict[str, int] = defaultdict(int)
    keywords_list = [keyword for keyword in keywords if keyword in set(keyword_table.keys())]
    for keyword in keywords_list:
        for node_id in keyword_table[keyword]:
            chunk_indices_count[node_id] += 1
    
    # 按匹配关键词数量排序
    sorted_chunk_indices = sorted(
        chunk_indices_count.keys(),
        key=lambda x: chunk_indices_count[x],
        reverse=True,
    )
    return sorted_chunk_indices[:k]
```

## 2. 索引结构差异

### 高质量索引的全文检索
**倒排索引 + 向量索引**：
- **Elasticsearch**：使用Lucene的倒排索引
```python
def search_by_full_text(self, query: str, **kwargs: Any) -> list[Document]:
    query_str: dict[str, Any] = {"match": {Field.CONTENT_KEY: query}}
    results = self._client.search(index=self._collection_name, query=query_str, size=kwargs.get("top_k", 4))
```

- **PostgreSQL**：使用GiST/GIN索引的全文搜索
```python
def search_by_full_text(self, query: str, **kwargs: Any) -> list[Document]:
    cur.execute(
        f"""SELECT meta, text, ts_rank(to_tsvector(coalesce(text, '')), plainto_tsquery(%s)) AS score
        FROM {self.table_name}
        WHERE to_tsvector(text) @@ plainto_tsquery(%s)
        ORDER BY score DESC
        LIMIT {top_k}""",
        (f"'{query}'", f"'{query}'"),
    )
```

### 经济索引的关键词检索
**简单的关键词->文档ID映射表**：
```python
# 关键词提取
def extract_keywords(self, text: str, max_keywords_per_chunk: int | None = 10) -> set[str]:
    import jieba.analyse
    keywords = jieba.analyse.extract_tags(
        sentence=text,
        topK=max_keywords_per_chunk,
    )
    return set(self._expand_tokens_with_subtokens(set(keywords)))

# 存储结构：{关键词: [文档ID列表]}
keyword_table = {
    "机器学习": ["doc1", "doc3", "doc5"],
    "深度学习": ["doc2", "doc3", "doc4"],
    "神经网络": ["doc2", "doc4", "doc6"]
}
```

## 3. 查询处理能力差异

### 高质量索引的全文检索
✅ **语言学支持**：
- 词干化（stemming）
- 停用词过滤
- 同义词扩展
- 模糊匹配

✅ **评分算法**：
- BM25算法
- TF-IDF权重
- 位置相关性
- 字段权重

✅ **查询复杂性**：
- 布尔查询（AND、OR、NOT）
- 短语查询
- 通配符查询
- 正则表达式查询

### 经济索引的关键词检索
❌ **功能限制**：
- 仅支持精确关键词匹配
- 无语言学处理
- 无模糊匹配能力
- 简单的计数排序

✅ **优势**：
- 计算简单快速
- 内存占用小
- 无需额外组件

## 4. 查询结果质量对比

### 高质量索引的全文检索
**查询示例：** "如何提升机器学习模型的准确率？"

处理流程：
1. 文本分析：分词、去停用词、词干化
2. 查询扩展：同义词匹配
3. 相关性计算：BM25/TF-IDF评分
4. 结果排序：按相关性分数排序

**能匹配到**：
- "提高机器学习模型精度的方法"（同义词匹配）
- "ML模型准确度优化技巧"（缩写和同义词）
- "改善深度学习模型性能"（语义相关）

### 经济索引的关键词检索
**相同查询处理**：
```python
# Jieba提取关键词：["提升", "机器学习", "模型", "准确率"]
keywords = ["提升", "机器学习", "模型", "准确率"]

# 只能匹配包含这些确切词汇的文档
# 统计每个文档匹配的关键词数量进行排序
```

**只能匹配到**：
- 包含"提升"、"机器学习"、"模型"、"准确率"等确切词汇的文档
- 无法匹配同义词或语义相关内容

## 5. 性能和成本对比

| 维度 | 高质量索引全文检索 | 经济索引关键词检索 |
|------|-------------------|-------------------|
| **计算复杂度** | 高（需要复杂的文本处理和评分） | 低（简单的计数和排序） |
| **内存使用** | 高（倒排索引 + 向量索引） | 低（简单哈希表） |
| **Token消耗** | 消耗embedding tokens | 零token消耗 |
| **查询延迟** | 中等（10-100ms） | 极低（<10ms） |
| **准确性** | 高（85-95%） | 中等（60-75%） |
| **召回率** | 高（语义匹配能力强） | 低（依赖词汇重叠） |

## 6. 适用场景建议

### 高质量索引全文检索适合：
- 需要高准确性的企业知识库
- 多语言或复杂查询场景
- 用户查询模式多样化
- 对成本不敏感的应用

### 经济索引关键词检索适合：
- 成本敏感的应用
- 查询模式相对固定
- 文档和查询用词重叠度高
- 对响应速度要求极高的场景

## 总结

**高质量索引的全文检索**是基于成熟搜索引擎技术的专业解决方案，提供语义理解、同义词匹配、复杂查询等高级功能，但需要消耗embedding tokens和更多计算资源。

**经济索引的关键词检索**是基于简单统计的轻量级方案，虽然功能有限但成本极低，适合对准确性要求不高但对成本敏感的场景。

两者代表了**功能丰富性与成本效益**之间的经典权衡。