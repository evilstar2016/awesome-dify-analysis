
## 关于知识库查询的索引方式

根据代码分析，Dify 支持以下几种索引方式：

### 1. 索引类型 (IndexType)
在 index_type.py 中定义了三种主要的索引类型：

- **PARAGRAPH_INDEX** (`"text_model"`) - 段落索引
- **QA_INDEX** (`"qa_model"`) - 问答索引  
- **PARENT_CHILD_INDEX** (`"hierarchical_model"`) - 父子层级索引

### 2. 检索方法 (RetrievalMethod)
在 retrieval_methods.py 中定义了四种检索方法：

- **SEMANTIC_SEARCH** - 语义搜索（基于向量）
- **FULL_TEXT_SEARCH** - 全文搜索（基于关键词）
- **HYBRID_SEARCH** - 混合搜索（语义+全文）
- **KEYWORD_SEARCH** - 关键词搜索

### 3. 索引技术
- **high_quality** - 高质量索引（使用向量数据库）
- **economy** - 经济型索引（仅使用关键词）

## 如何构建分段的索引

### 1. 段落索引 (ParagraphIndexProcessor)

这是最基础的分段方式：

```python
# 分段配置
class Segmentation(BaseModel):
    separator: str = "\n"      # 分隔符，默认换行符
    max_tokens: int           # 最大token数
    chunk_overlap: int = 0    # 重叠token数
```

**构建流程：**
1. **提取** (extract) - 从文档中提取文本
2. **转换** (transform) - 根据分段规则切分文档
3. **加载** (load) - 将分段存入向量数据库或关键词索引

### 2. 父子索引 (ParentChildIndexProcessor)

这是层级结构的索引方式，支持两种父文档模式：

```python
class ParentMode(StrEnum):
    FULL_DOC = "full-doc"     # 整个文档作为父节点
    PARAGRAPH = "paragraph"   # 段落作为父节点
```

**构建特点：**
- 父文档保持较大的上下文
- 子文档用于精确匹配
- 支持双层分段：`segmentation` 和 `subchunk_segmentation`

### 3. 问答索引 (QAIndexProcessor)

将文档转换为问答对的索引方式：
- 使用LLM生成问答对
- 问题用于检索匹配
- 答案作为返回内容

## 实际构建分段索引的步骤

### 1. 配置分段规则

```python
# 定义分段规则
segmentation_config = {
    "separator": "\n\n",      # 使用双换行分割
    "max_tokens": 500,        # 每段最大500个token
    "chunk_overlap": 50       # 段落间重叠50个token
}

# 对于父子索引，还需要配置子分段
subchunk_config = {
    "separator": "\n",
    "max_tokens": 100,
    "chunk_overlap": 10
}
```

### 2. 初始化索引处理器

```python
from core.rag.index_processor.index_processor_factory import IndexProcessorFactory
from core.rag.index_processor.constant.index_type import IndexType

# 选择索引类型
processor_factory = IndexProcessorFactory(IndexType.PARAGRAPH_INDEX)
processor = processor_factory.init_index_processor()
```

### 3. 处理文档

```python
# 1. 提取文档内容
documents = processor.extract(extract_setting, **kwargs)

# 2. 应用分段规则
segmented_docs = processor.transform(documents, process_rule=process_rule)

# 3. 构建索引
processor.load(dataset, segmented_docs, with_keywords=True)
```

### 4. 向量数据库支持

Dify 支持多种向量数据库：
- PostgreSQL (pgvector)
- Chroma
- Weaviate  
- Milvus
- Qdrant
- Elasticsearch
- 等20多种向量数据库

### 5. 文本分割器

核心使用 `RecursiveCharacterTextSplitter`：
- 递归字符级分割
- 保持语义完整性
- 支持重叠设置
- 可配置分隔符

## 最佳实践建议

1. **选择合适的索引类型**：
   - 简单文档：使用段落索引
   - 长文档需要上下文：使用父子索引
   - FAQ场景：使用问答索引

2. **调优分段参数**：
   - `max_tokens`: 根据模型上下文窗口调整
   - `chunk_overlap`: 一般设置为max_tokens的10-20%
   - `separator`: 根据文档结构选择合适分隔符

3. **混合检索策略**：
   - 结合语义搜索和关键词搜索
   - 使用重排序模型提升精度

这样的设计使得Dify能够灵活处理各种类型的文档，并根据不同的查询需求提供最佳的检索效果。



## 关键词索引的设置时机和位置

### 1. **设置时机 (When)**

#### 1.1 数据集创建时
- **位置**: datasets.py
- **时机**: 用户创建新数据集时指定 `indexing_technique`
- **代码位置**:
```python
# 在 DatasetListApi.post() 方法中
dataset = DatasetService.create_empty_dataset(
    indexing_technique=args["indexing_technique"],  # 可选: "economy" 或 "high_quality"
    # ...其他参数
)
```

#### 1.2 文档索引处理时  
- **位置**: paragraph_index_processor.py
- **时机**: 文档分段后加载到索引时
- **代码逻辑**:
```python
def load(self, dataset: Dataset, documents: list[Document], with_keywords: bool = True, **kwargs):
    if dataset.indexing_technique == "high_quality":
        vector = Vector(dataset)
        vector.create(documents)
        with_keywords = False  # 高质量索引不使用关键词
    if with_keywords:
        keyword = Keyword(dataset)
        keyword.add_texts(documents)  # 使用关键词索引
```

#### 1.3 批量向量索引重建时
- **位置**: deal_dataset_vector_index_task.py
- **时机**: 当数据集的索引技术发生变更时
- **代码逻辑**:
```python
# 重建索引时，根据数据集设置决定是否使用关键词
index_processor.load(dataset, documents, with_keywords=False)  # 向量索引重建时关闭关键词
```

### 2. **设置位置 (Where)**

#### 2.1 配置层面
- **文件**: dataset.py
- **定义**: 
```python
INDEXING_TECHNIQUE_LIST = ["high_quality", "economy", None]
```

#### 2.2 关键词实现层面
- **工厂类**: keyword_factory.py
- **具体实现**: jieba.py
- **配置**:
```python
# 配置文件中的设置
KEYWORD_STORE = "jieba"  # 关键词存储类型

# 关键词表配置
class KeywordTableConfig(BaseModel):
    max_keywords_per_chunk: int = 10  # 每个分段最大关键词数
```

#### 2.3 数据库存储
- **表**: `dataset_keyword_tables` - 存储关键词表
- **表**: `document_segments` - 存储分段及其关键词
- **字段**: `dataset.indexing_technique` - 索引技术类型

### 3. **关键决策逻辑**

#### 3.1 索引技术选择逻辑
```python
# 在各个索引处理器中的通用逻辑
if dataset.indexing_technique == "high_quality":
    # 使用向量索引 (embeddings)
    vector = Vector(dataset)
    vector.create(documents)
    with_keywords = False  # 不使用关键词索引
elif dataset.indexing_technique == "economy":
    # 仅使用关键词索引
    with_keywords = True
    
if with_keywords:
    keyword = Keyword(dataset)
    keyword.add_texts(documents)
```

#### 3.2 检索时的路由逻辑
- **位置**: retrieval_service.py
- **逻辑**: 根据检索方法和数据集配置选择使用向量搜索还是关键词搜索

### 4. **具体的设置流程**

#### 4.1 用户界面设置
1. 用户在前端选择索引技术："高质量" 或 "经济型"
2. 前端发送 POST 请求到 `/datasets` 接口
3. 后端根据 `indexing_technique` 参数创建数据集

#### 4.2 文档处理流程
1. 文档上传后进入处理队列
2. 根据数据集的 `indexing_technique` 决定索引方式:
   - `"economy"` → 仅关键词索引
   - `"high_quality"` → 向量索引 + 可选关键词索引
   - `None` → 默认行为

#### 4.3 关键词提取配置
- **提取器**: 使用 Jieba 分词器
- **存储**: Redis 锁保护的关键词表
- **数量**: 每个分段默认提取10个关键词
- **存储位置**: 
  - 关键词表: `dataset_keyword_tables`
  - 分段关键词: `document_segments.keywords`

### 5. **配置参数**

```python
# 关键词索引相关配置
KEYWORD_STORE = "jieba"                    # 关键词存储类型
max_keywords_per_chunk = 10               # 每段最大关键词数
dataset.keyword_number                    # 数据集级别的关键词数量配置
```

**总结**: 关键词索引主要在数据集创建时通过 `indexing_technique="economy"` 设置，在文档处理时通过索引处理器自动应用，具体实现在 `Keyword` 工厂类和 `Jieba` 关键词提取器中完成。



## 关键词表的作用时机对比

### 1. **`dataset_keyword_tables` - 数据集关键词表**

#### **作用时机:**

**1.1 索引构建时 (写入)**
- **时机**: 文档分段后，提取关键词并构建索引时
- **位置**: jieba.py
- **具体流程**:
```python
def create(self, texts: list[Document], **kwargs):
    # 获取现有关键词表
    keyword_table = self._get_dataset_keyword_table()
    
    for text in texts:
        # 提取关键词
        keywords = keyword_table_handler.extract_keywords(text.page_content, keyword_number)
        
        # 同时更新两个表:
        # 1. 更新分段关键词
        self._update_segment_keywords(self.dataset.id, text.metadata["doc_id"], list(keywords))
        
        # 2. 添加到关键词表 (倒排索引)
        keyword_table = self._add_text_to_keyword_table(keyword_table or {}, text.metadata["doc_id"], list(keywords))
    
    # 保存关键词表
    self._save_dataset_keyword_table(keyword_table)
```

**1.2 关键词搜索时 (读取)**
- **时机**: 用户进行关键词检索时
- **作用**: 作为倒排索引，快速找到包含特定关键词的文档分段
- **流程**:
```python
def search(self, query: str, **kwargs):
    # 获取关键词表 (倒排索引)
    keyword_table = self._get_dataset_keyword_table()
    
    # 通过关键词表快速检索
    sorted_chunk_indices = self._retrieve_ids_by_query(keyword_table or {}, query, k)
    
    # 根据检索到的节点ID获取具体内容
    for chunk_index in sorted_chunk_indices:
        segment = db.session.query(DocumentSegment).where(
            DocumentSegment.dataset_id == self.dataset.id, 
            DocumentSegment.index_node_id == chunk_index
        ).first()
```

**存储结构**:
```json
{
  "__type__": "keyword_table",
  "__data__": {
    "table": {
      "关键词1": ["segment_id_1", "segment_id_3"],
      "关键词2": ["segment_id_2", "segment_id_4"],
      "关键词3": ["segment_id_1", "segment_id_2"]
    }
  }
}
```

### 2. **`document_segments.keywords` - 分段关键词字段**

#### **作用时机:**

**2.1 索引构建时 (写入)**
- **时机**: 与关键词表同时更新
- **位置**: 同样在关键词提取过程中
```python
def _update_segment_keywords(self, dataset_id: str, node_id: str, keywords: list[str]):
    document_segment = db.session.query(DocumentSegment).where(
        DocumentSegment.dataset_id == dataset_id, 
        DocumentSegment.index_node_id == node_id
    ).first()
    
    if document_segment:
        document_segment.keywords = keywords  # 直接存储该分段的关键词列表
        db.session.commit()
```

**2.2 分段管理和显示时 (读取)**
- **时机**: 查看具体分段信息时
- **作用**: 显示该分段的关键词，用于调试和管理
- **场景**: 
  - 管理后台显示分段详情
  - 调试关键词提取效果
  - 分段级别的关键词分析

**存储结构**:
```python
# document_segments 表中的 keywords 字段
["关键词1", "关键词2", "关键词3"]  # JSON数组格式
```

### 3. **两表的协同关系**

#### **3.1 写入时的协同**
```python
# 同时更新两个存储位置
def add_texts(self, texts: list[Document], **kwargs):
    for text in texts:
        keywords = extract_keywords(text.page_content)
        
        # 1. 更新分段表 - 正向存储 (分段 -> 关键词)
        self._update_segment_keywords(dataset_id, doc_id, keywords)
        
        # 2. 更新关键词表 - 倒排索引 (关键词 -> 分段列表)
        keyword_table = self._add_text_to_keyword_table(keyword_table, doc_id, keywords)
```

#### **3.2 性能优化的设计**
- **关键词表**: 用于快速检索，倒排索引结构
- **分段关键词**: 用于数据完整性和管理展示

#### **3.3 数据一致性保证**
- 两个表在同一事务中更新
- 使用 Redis 分布式锁确保并发安全
```python
with redis_client.lock(f"keyword_indexing_lock_{self.dataset.id}", timeout=600):
    # 同时更新两个存储位置
```

### 4. **具体使用场景总结**

| 表名 | 主要用途 | 使用时机 | 数据结构 |
|------|---------|---------|----------|
| `dataset_keyword_tables` | 快速检索 (倒排索引) | 搜索时读取<br/>索引时写入 | `{关键词: [分段ID列表]}` |
| `document_segments.keywords` | 分段管理和展示 | 管理界面显示<br/>调试分析 | `[关键词列表]` |

### 5. **删除时的协同处理**
```python
def delete_by_ids(self, ids: list[str]):
    # 1. 从关键词表中移除这些分段的引用
    keyword_table = self._delete_ids_from_keyword_table(keyword_table, ids)
    self._save_dataset_keyword_table(keyword_table)
    
    # 2. 分段删除时，keywords字段也会一起删除 (数据库级联)
```

**总结**: `dataset_keyword_tables` 是为了**检索性能**而设计的倒排索引，在搜索时发挥关键作用；而 `document_segments.keywords` 是为了**数据完整性和管理**而设计的正向存储，主要用于分段的展示和管理功能。两者在索引构建时同时更新，确保数据一致性。



## "high_quality"模式下的可选关键词索引详解

### 1. **当前实际实现情况**

根据代码分析，**当前的Dify实现中，"high_quality"模式实际上并不支持可选关键词索引**。我之前的表述有误，实际情况是：

```python
# 在所有索引处理器中的实现
def load(self, dataset: Dataset, documents: list[Document], with_keywords: bool = True, **kwargs):
    if dataset.indexing_technique == "high_quality":
        vector = Vector(dataset)
        vector.create(documents)
        with_keywords = False  # 强制设置为False
    if with_keywords:
        keyword = Keyword(dataset)
        keyword.add_texts(documents)  # 这段代码不会执行
```

### 2. **实际的索引模式**

**当前Dify的实际索引模式：**

| 索引技术 | 向量索引 | 关键词索引 | 实际效果 |
|---------|---------|----------|----------|
| `"economy"` | ❌ | ✅ | 仅关键词 |
| `"high_quality"` | ✅ | ❌ | 仅向量 |

### 3. **混合搜索的实现方式**

虽然不能同时建两种索引，但可以通过**检索方法**实现混合搜索：

```python
# 检索服务中的实现
class RetrievalMethod(StrEnum):
    SEMANTIC_SEARCH = "semantic_search"     # 仅语义搜索
    FULL_TEXT_SEARCH = "full_text_search"  # 仅全文搜索  
    HYBRID_SEARCH = "hybrid_search"        # 混合搜索
    KEYWORD_SEARCH = "keyword_search"      # 关键词搜索
```

### 4. **混合搜索的工作原理**

在 `RetrievalService.retrieve()` 中：

```python
# 混合搜索时并行执行两种检索
if RetrievalMethod.is_support_fulltext_search(retrieval_method):
    # 关键词/全文搜索
    futures.append(cls.keyword_search(...))
    
if RetrievalMethod.is_support_semantic_search(retrieval_method):
    # 语义搜索
    futures.append(cls.embedding_search(...))

# 混合搜索时去重和重排序
if retrieval_method == RetrievalMethod.HYBRID_SEARCH:
    all_documents = cls._deduplicate_documents(all_documents)
    data_post_processor = DataPostProcessor(...)
    all_documents = data_post_processor.invoke(...)
```

### 5. **"可选关键词"的真实含义**

我之前提到的"可选关键词"实际上指的是：

**在某些特殊任务中，可以通过参数控制是否启用关键词索引：**

```python
# 在向量索引重建任务中
index_processor.load(dataset, documents, with_keywords=False)  # 明确禁用关键词

# 在常规文档处理中  
index_processor.load(dataset, documents, with_keywords=True)   # 默认启用
```

但由于 `high_quality` 模式的硬编码逻辑，`with_keywords` 总是被强制设为 `False`。

### 6. **如何实现真正的混合索引**

如果要实现真正的 `high_quality + 关键词` 混合索引，需要修改代码：

```python
# 修改后的实现（当前代码中不存在）
def load(self, dataset: Dataset, documents: list[Document], with_keywords: bool = True, **kwargs):
    if dataset.indexing_technique == "high_quality":
        vector = Vector(dataset)
        vector.create(documents)
        # 不强制设置 with_keywords = False
        
    if with_keywords:
        keyword = Keyword(dataset)
        keyword.add_texts(documents)
```

### 7. **当前的混合搜索解决方案**

Dify 通过以下方式实现混合搜索能力：

1. **向量数据库的混合搜索功能**：
   - Milvus: `MILVUS_ENABLE_HYBRID_SEARCH=true`
   - Tencent Vector: `TENCENT_VECTOR_DB_ENABLE_HYBRID_SEARCH=true`
   - OceanBase: `OCEANBASE_ENABLE_HYBRID_SEARCH=true`

2. **检索时的策略组合**：
   - 用户可以选择 `HYBRID_SEARCH` 检索方法
   - 系统会并行执行语义搜索和关键词搜索
   - 最后进行结果融合和重排序

### 8. **总结**

**当前Dify中并不存在真正的"high_quality + 关键词"双索引模式**。我之前的描述是基于理论可能性，但实际代码实现是互斥的：

- `"economy"` = 纯关键词索引
- `"high_quality"` = 纯向量索引
- 混合搜索通过检索时的策略组合实现，而非索引时的双重构建

如果需要真正的混合索引，需要修改 `IndexProcessor` 的核心逻辑。



## 关键词表自定义方案

基于代码分析，关键词表的建设和自定义有以下几种方式：

### 1. 通过API直接更新分段关键词

**最直接的方式：使用PATCH API更新分段**

```bash
# API端点
PATCH /console/api/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}

# 请求体示例
{
  "keywords": ["自定义关键词1", "自定义关键词2", "专业术语", "业务概念"]
}
```

**代码实现路径：**
1. `DatasetDocumentSegmentUpdateApi.patch()` 接收keywords参数
2. 调用 `SegmentService.update_segment()` 处理更新
3. 最终通过 `VectorService.update_segment_vector()` 更新索引

### 2. 关键词更新的核心逻辑

在 vector_service.py 中的 `update_segment_vector` 方法：

```python
def update_segment_vector(cls, keywords: list[str] | None, segment: DocumentSegment, dataset: Dataset):
    if dataset.indexing_technique == "high_quality":
        # 高质量模式：仅更新向量索引
        vector = Vector(dataset=dataset)
        vector.delete_by_ids([segment.index_node_id])
        vector.add_texts([document], duplicate_check=True)
    else:
        # 经济模式：更新关键词索引
        keyword = Keyword(dataset)
        keyword.delete_by_ids([segment.index_node_id])
        
        # 保存自定义关键词到关键词索引
        if keywords and len(keywords) > 0:
            keyword.add_texts([document], keywords_list=[keywords])
        else:
            keyword.add_texts([document])  # 使用默认提取的关键词
```

### 3. 两个关键词表的作用时机

**`dataset_keyword_tables` 表：**
- **作用时机：** 仅在 `indexing_technique = "economy"` 时生效
- **更新时机：** 通过上述API更新分段keywords时
- **查询时机：** 使用 `KEYWORD_SEARCH` 或 `FULL_TEXT_SEARCH` 检索时

**`document_segments.keywords` 字段：**
- **作用时机：** 两种模式都会存储，但仅在economy模式参与检索
- **更新时机：** 
  - 自动提取：文档解析时通过Jieba分词
  - 手动设置：通过API传入keywords参数
- **存储格式：** JSON数组形式存储在数据库中

### 4. 实践建议

**如果要大规模自定义关键词表，建议：**

1. **批量API调用方式：**
```python
# 伪代码示例
for segment_id, custom_keywords in segments_keywords_mapping.items():
    requests.patch(
        f"/console/api/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}",
        json={"keywords": custom_keywords}
    )
```

2. **直接数据库操作方式（高级）：**
```python
# 注意：需要同时更新segment和keyword_table
segment.keywords = custom_keywords
db.session.add(segment)

# 还需要重建关键词索引
keyword_service = Keyword(dataset)
keyword_service.delete_by_ids([segment.index_node_id])
keyword_service.add_texts([document], keywords_list=[custom_keywords])
```

### 5. 重要注意事项

1. **模式限制：** `high_quality` 模式下设置的keywords会被存储但不会用于检索
2. **索引一致性：** 修改keywords后必须重建相应的关键词索引
3. **并发安全：** 系统使用Redis锁防止并发索引冲突
4. **权限检查：** API调用需要相应的数据集编辑权限

**总结：关键词表的建设核心就是通过API或直接操作为分段设置自定义keywords，系统会自动维护 `dataset_keyword_tables` 的倒排索引结构。**



## 查询时的分词和关键词表使用流程

### 1. 查询分词过程

当你在页面输入查询内容时，**确实会被分词**！具体流程是：

```python
# 在 _retrieve_ids_by_query 方法中
keyword_table_handler = JiebaKeywordTableHandler()
keywords = keyword_table_handler.extract_keywords(query)  # 对查询进行分词
```

### 2. 两个关键词表的作用时机

**`dataset_keyword_tables` 表（倒排索引）：**
- **使用时机：** 仅在 `indexing_technique = "economy"` 且检索方法为 `KEYWORD_SEARCH` 或 `FULL_TEXT_SEARCH` 时使用
- **查询流程：**
  1. 用户输入查询 → Jieba分词提取关键词
  2. 在倒排索引中查找包含这些关键词的文档分段
  3. 按关键词匹配数量排序返回结果

**`document_segments.keywords` 字段：**
- **使用时机：** 主要用于存储分段的关键词，但在检索时**不直接参与匹配**
- **作用：** 
  - 存储每个分段的关键词（自动提取+手动设置）
  - 用于关键词索引的重建
  - 在更新分段时保持关键词的一致性

### 3. 具体查询匹配逻辑

```python
def _retrieve_ids_by_query(self, keyword_table: dict, query: str, k: int = 4):
    # 1. 对查询进行分词
    keyword_table_handler = JiebaKeywordTableHandler()
    keywords = keyword_table_handler.extract_keywords(query)
    
    # 2. 统计每个分段匹配的关键词数量
    chunk_indices_count = defaultdict(int)
    keywords_list = [keyword for keyword in keywords if keyword in set(keyword_table.keys())]
    
    # 3. 遍历匹配的关键词，累计分段的匹配分数
    for keyword in keywords_list:
        for node_id in keyword_table[keyword]:  # 使用倒排索引
            chunk_indices_count[node_id] += 1
    
    # 4. 按匹配关键词数量排序
    sorted_chunk_indices = sorted(
        chunk_indices_count.keys(),
        key=lambda x: chunk_indices_count[x],
        reverse=True,
    )
    
    return sorted_chunk_indices[:k]
```

### 4. 关键要点总结

**是的，查询会被分词！** 而且：

1. **分词工具相同：** 查询和文档都使用相同的Jieba分词器，确保一致性
2. **主要使用倒排索引：** `dataset_keyword_tables` 是查询的核心，提供倒排索引功能
3. **分段关键词辅助：** `document_segments.keywords` 主要用于数据管理，不直接参与检索匹配
4. **匹配策略：** 按关键词重叠数量排序，匹配越多的分段排名越高

**示例流程：**
```
用户查询: "人工智能机器学习算法"
   ↓ Jieba分词
关键词: ["人工智能", "机器学习", "算法"]
   ↓ 倒排索引查找
dataset_keyword_tables: {
  "人工智能": {"seg_001", "seg_005"},
  "机器学习": {"seg_001", "seg_003"}, 
  "算法": {"seg_001", "seg_007"}
}
   ↓ 匹配计分
seg_001: 3分 (匹配3个关键词)
seg_003: 1分, seg_005: 1分, seg_007: 1分
   ↓ 排序返回
结果: [seg_001, seg_003, seg_005, seg_007]
```

所以是的，你的查询确实会被分词，并且主要通过 `dataset_keyword_tables` 的倒排索引进行匹配！



## 方案一：通过API为分段添加自定义关键词（推荐）

这是最直接有效的方法，将专有名称作为关键词添加到相关分段中：

```bash
# 1. 首先找到包含专有名称的分段ID
# 2. 通过API为这些分段添加专有名称作为关键词

PATCH /console/api/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}

# 请求体
{
  "keywords": ["专有名称A", "技术术语B", "业务概念C", "原有关键词1", "原有关键词2"]
}
```

**优势：**
- 精确控制哪些分段包含特定专有名称
- 不影响其他分段的索引
- 可以随时更新和调整

## 方案二：扩展Jieba词典（系统级方案）

如果专有名称较多且固定，可以在Jieba分词器中添加自定义词典： 

Ran `read_file` 
Completed with input: {
  "end_line": 30,
  "relative_path": "api/core/rag/datasource/keyword/jieba/jieba_keyword_table_handler.py",
  "start_line": 1
}

可以通过修改Jieba初始化来添加自定义词典：

```python
# 在 JiebaKeywordTableHandler.__init__ 中添加
def __init__(self):
    import jieba.analyse
    import jieba
    
    # 添加自定义词典
    jieba.add_word("专有名称A", freq=1000, tag=None)
    jieba.add_word("技术术语B", freq=1000, tag=None)
    jieba.add_word("业务概念C", freq=1000, tag=None)
    
    # 或者从文件加载
    # jieba.load_userdict("custom_terms.txt")
    
    from core.rag.datasource.keyword.jieba.stopwords import STOPWORDS
    jieba.analyse.default_tfidf.stop_words = STOPWORDS
```

## 方案三：数据预处理时批量添加关键词

在文档索引时，通过正则表达式或关键词匹配自动为包含专有名称的分段添加关键词： 

Ran `read_file` 
Completed with input: {
  "end_line": 70,
  "relative_path": "api/core/rag/datasource/keyword/jieba/jieba.py",
  "start_line": 50
}

可以在关键词提取时添加专有名称检测逻辑：

```python
def extract_keywords_with_custom_terms(self, text: str, max_keywords_per_chunk: int = 10) -> set[str]:
    """Extract keywords with custom term detection."""
    import jieba.analyse
    
    # 定义专有名称字典
    custom_terms = {
        "专有名称A": ["别名1", "缩写A"],
        "技术术语B": ["简称B", "英文名B"],
        "业务概念C": ["通俗说法C"]
    }
    
    # 常规关键词提取
    keywords = jieba.analyse.extract_tags(
        sentence=text,
        topK=max_keywords_per_chunk,
    )
    
    # 检测专有名称
    text_lower = text.lower()
    for main_term, aliases in custom_terms.items():
        # 检测主名称
        if main_term in text:
            keywords.append(main_term)
        # 检测别名
        for alias in aliases:
            if alias in text_lower:
                keywords.append(main_term)  # 统一映射到主名称
    
    return set(keywords)
```

## 方案四：使用同义词映射

在查询时将专有名称的各种表达方式映射到统一的标准词：

```python
def normalize_query_terms(query: str) -> str:
    """将查询中的专有名称标准化"""
    synonyms_map = {
        "AI": "人工智能",
        "ML": "机器学习", 
        "深度学习": "机器学习",
        "神经网络": "机器学习"
    }
    
    normalized_query = query
    for alias, standard_term in synonyms_map.items():
        normalized_query = normalized_query.replace(alias, standard_term)
    
    return normalized_query
```

## 具体实施建议

**针对你的场景，我推荐以下组合方案：**

### 1. 立即可行的方案（方案一）
```python
# 批量脚本示例
import requests

# 专有名称和相关分段的映射
term_segments_mapping = {
    "专有技术A": ["segment_id_1", "segment_id_2"],
    "业务概念B": ["segment_id_3", "segment_id_4"],
    "产品名称C": ["segment_id_5"]
}

def add_custom_keywords_to_segments():
    for term, segment_ids in term_segments_mapping.items():
        for segment_id in segment_ids:
            # 获取现有关键词
            response = requests.get(f"/api/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}")
            current_keywords = response.json().get("keywords", [])
            
            # 添加专有名称
            if term not in current_keywords:
                current_keywords.append(term)
                
                # 更新分段
                requests.patch(
                    f"/api/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}",
                    json={"keywords": current_keywords}
                )
```

### 2. 长期优化方案（方案二+三）
1. 扩展Jieba词典，确保专有名称被正确分词
2. 在文档处理流程中自动检测和添加专有名称关键词
3. 建立专有名称的同义词映射表

### 3. 验证效果
```python
# 测试查询匹配效果
def test_custom_term_matching():
    test_queries = [
        "专有技术A的应用场景",
        "如何使用业务概念B",
        "产品名称C的特点"
    ]
    
    for query in test_queries:
        results = search_dataset(query)
        print(f"查询: {query}")
        print(f"匹配结果: {len(results)} 个分段")
        for result in results[:3]:
            print(f"  - {result['content'][:100]}...")
```

**总结：** 对于专有名称匹配问题，最快速有效的方法是通过API为相关分段手动添加专有名称作为关键词。这样可以确保用户使用专有名称查询时能精确匹配到对应内容。