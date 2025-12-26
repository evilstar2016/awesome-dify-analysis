# 04 - 业务逻辑层 (Business Logic Layer)

## 1. 层职责与定位

### 1.1 职责
业务逻辑层是 RAGFlow 的核心引擎，负责：
- **RAG 检索**: 文档检索、向量搜索、重排序
- **LLM 生成**: 调用大语言模型生成答案
- **Agent 编排**: Agent 工作流执行和工具调用
- **文档解析**: 多格式文档解析和 OCR
- **知识图谱**: GraphRAG、实体识别、关系抽取
- **分块策略**: 文档智能分块和嵌入
- **评估基准**: RAG 性能评估和优化

### 1.2 定位
- **架构位置**: API 层和数据访问层之间
- **技术选型**: Python 纯异步实现
- **设计模式**: 策略模式、工厂模式、管道模式
- **核心模块**:
  - `rag/`: RAG 引擎核心逻辑
  - `agent/`: Agent 系统
  - `deepdoc/`: 文档解析引擎
  - `graphrag/`: 知识图谱构建
  - `agentic_reasoning/`: 深度推理

---

## 2. 主要组件/模块

### 2.1 RAG 引擎 (`rag/`)

#### 目录结构
```
rag/
├── __init__.py
├── settings.py           # RAG 配置
├── benchmark.py          # 性能基准测试
├── raptor.py             # RAPTOR 分层检索
├── app/                  # 文档解析器 (各种格式的解析策略)
│   ├── naive.py         # 朴素解析器
│   ├── qa.py            # 问答对解析
│   ├── resume.py        # 简历解析
│   ├── paper.py         # 论文解析
│   ├── book.py          # 书籍解析
│   ├── laws.py          # 法律文档解析
│   ├── manual.py        # 手册解析
│   ├── presentation.py  # 演示文稿解析
│   ├── email.py         # 邮件解析
│   ├── picture.py       # 图片解析
│   └── table.py         # 表格解析
├── flow/                 # Agent 工作流
│   ├── canvas.py        # 画布执行器
│   └── node.py          # 节点定义
├── llm/                  # LLM 抽象层
│   ├── chat_model.py    # 聊天模型 (Base类及各种LLM实现)
│   ├── cv_model.py      # 视觉模型
│   ├── embedding_model.py # 嵌入模型
│   ├── rerank_model.py  # 重排模型
│   ├── sequence2txt_model.py # 序列转文本模型
│   ├── tts_model.py     # 文本转语音模型
│   └── ocr_model.py     # OCR模型
├── nlp/                  # NLP 工具
│   ├── tokenizer.py     # 分词器
│   ├── rag_tokenizer.py # RAG 专用分词
│   └── search.py        # 搜索算法
├── prompts/              # 提示词模板
│   ├── knowledge.py     # 知识库提示词
│   └── chat.py          # 对话提示词
├── svr/                  # 服务层
│   ├── chunk_service.py # 分块服务
│   └── task_broker.py   # 任务调度
└── utils/                # 工具库
    ├── es_conn.py       # ES 连接
    ├── redis_conn.py    # Redis 连接
    └── infinity_conn.py # Infinity 连接
```

#### 2.1.1 LLM 模型抽象层 (`rag/llm/chat_model.py`)

**核心基类**:
```python
class Base(ABC):
    """LLM 聊天模型基类"""
    
    def __init__(self, key, model_name, base_url, **kwargs):
        timeout = int(os.environ.get("LLM_TIMEOUT_SECONDS", 600))
        self.client = OpenAI(api_key=key, base_url=base_url, timeout=timeout)
        self.async_client = AsyncOpenAI(api_key=key, base_url=base_url, timeout=timeout)
        self.model_name = model_name
        # 配置重试参数
        self.max_retries = kwargs.get("max_retries", int(os.environ.get("LLM_MAX_RETRIES", 5)))
        self.base_delay = kwargs.get("retry_interval", float(os.environ.get("LLM_BASE_DELAY", 2.0)))
        self.max_rounds = kwargs.get("max_rounds", 5)
        self.is_tools = False
        self.tools = []
    
    async def async_chat(self, system, history, gen_conf={}, **kwargs):
        """异步对话"""
        if system and history and history[0].get("role") != "system":
            history.insert(0, {"role": "system", "content": system})
        gen_conf = self._clean_conf(gen_conf)

        for attempt in range(self.max_retries + 1):
            try:
                return await self._async_chat(history, gen_conf, **kwargs)
            except Exception as e:
                e = await self._exceptions_async(e, attempt)
                if e:
                    return e, 0
    
    async def async_chat_streamly(self, system, history, gen_conf: dict = {}, **kwargs):
        """异步流式对话"""
        if system and history and history[0].get("role") != "system":
            history.insert(0, {"role": "system", "content": system})
        gen_conf = self._clean_conf(gen_conf)
        ans = ""
        total_tokens = 0

        for attempt in range(self.max_retries + 1):
            try:
                async for delta_ans, tol in self._async_chat_streamly(history, gen_conf, **kwargs):
                    ans = delta_ans
                    total_tokens += tol
                    yield ans

                yield total_tokens
                return
            except Exception as e:
                e = await self._exceptions_async(e, attempt)
                if e:
                    yield e
                    yield total_tokens
                    return
    
    async def async_chat_with_tools(self, system: str, history: list, gen_conf: dict = {}):
        """带工具调用的异步对话"""
        # 支持Function Calling和工具调用
        pass
```

**实现类示例**:
- `XinferenceChat`: Xinference本地模型
- `BaiChuanChat`: 百川模型
- `VolcEngineChat`: 火山引擎
- `BaiduYiyanChat`: 百度文心一言
- `SparkChat`: 讯飞星火
- `HunyuanChat`: 腾讯混元
- `MistralChat`: Mistral AI
- `GoogleChat`: Google Cloud (Gemini/Claude)
- `LiteLLMBase`: 统一接口支持40+种LLM提供商

#### 2.1.2 文档解析器 (`rag/app/`)

**文档解析策略**:
RAG引擎的`app/`目录包含多种文档解析策略,用于处理不同格式和类型的文档:

- **naive.py**: 通用文档解析器,支持基本的文本提取
- **qa.py**: 问答对格式解析,用于FAQ类文档
- **resume.py**: 简历解析,提取结构化信息
- **paper.py**: 学术论文解析,识别章节结构
- **book.py**: 书籍解析,处理长文档的章节划分
- **laws.py**: 法律文档解析,识别条款结构
- **manual.py**: 技术手册解析,提取操作步骤
- **presentation.py**: 演示文稿解析,处理幻灯片内容
- **email.py**: 邮件格式解析
- **picture.py**: 图片内容提取
- **table.py**: 表格数据解析

这些解析器在文档上传和分块时被使用,将不同格式的文档转换为统一的文本块结构。

#### 2.1.3 检索和对话流程

**实际的检索和对话流程由API层和LLM层协作完成**:
        """倒数排名融合 (RRF)"""
        scores = {}
        for results in results_list:
            for rank, chunk in enumerate(results, start=1):
                chunk_id = chunk.id
                scores[chunk_id] = scores.get(chunk_id, 0) + 1 / (60 + rank)
        
        # 按融合分数排序
        ranked_chunks = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return [chunk_map[cid] for cid, score in ranked_chunks]
```

#### 2.1.3 重排序器 (`rag/app/rerank.py`)

**重排序模型**:
```python
class Reranker:
    """重排序器"""
    
    def __init__(self, model_id: str):
        self.model = self._load_model(model_id)
    
    async def rerank(self, query: str, chunks: List[Chunk]) -> List[Chunk]:
        """重排序"""
        # 1. 计算相关性分数
        scores = []
1. **对话请求** → API层(`dialog_app.py`)接收用户问题
2. **知识库检索** → 通过向量搜索(Elasticsearch/Infinity)和全文检索获取相关文档块
3. **重排序** → 使用Rerank模型对检索结果重新排序
4. **提示词构建** → 将问题和检索到的文档块组合成提示词
5. **LLM生成** → 调用`rag/llm/chat_model.py`中的LLM客户端生成答案
6. **流式返回** → 通过SSE将答案流式返回给前端

### 2.2 重排序模型 (`rag/llm/rerank_model.py`)

**Rerank模型用于对检索结果进行重新排序**:
```python
class Base(ABC):
    """重排序模型基类"""
    
    @abstractmethod
    def similarity(self, query: str, texts: list[str]):
        """计算查询和文本列表的相似度分数"""
        pass
```

**支持的Rerank模型**:
- **BAAI/bge-reranker-v2-m3**: BGE重排序模型
- **Jina Reranker**: Jina AI重排序服务
- **Cohere Reranker**: Cohere重排序API  
- **Xinference**: 本地部署的重排序模型
- **OpenAI Compatible**: 兼容OpenAI接口的重排序服务

### 2.3 嵌入模型 (`rag/llm/embedding_model.py`)

**嵌入模型用于将文本转换为向量**:
            stream=True
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content
```

**支持的模型**:
- OpenAI (GPT-4, GPT-3.5)
- Anthropic (Claude)
- Google (Gemini)
- Azure OpenAI
- 本地模型 (Ollama, LocalAI)
- 智谱 AI、通义千问、文心一言等

#### 2.2.2 嵌入模型 (`embedding_model.py`)

```python
class BaseEmbeddingModel(ABC):
    @abstractmethod
    async def embed(self, texts: List[str]) -> List[List[float]]:
        """批量嵌入"""
        pass

class OpenAIEmbedding(BaseEmbeddingModel):
    def __init__(self, api_key: str, model_name: str = "text-embedding-3-large"):
        self.client = AsyncOpenAI(api_key=api_key)
        self.model_name = model_name
    
    async def embed(self, texts: List[str]) -> List[List[float]]:
        response = await self.client.embeddings.create(
            model=self.model_name,
            input=texts
        )
        return [item.embedding for item in response.data]
```

### 2.3 Agent 系统 (`agent/`)

#### 目录结构
```
agent/
├── __init__.py
├── settings.py
├── canvas.py            # Agent 画布执行
├── component/           # Agent 组件
│   ├── begin.py        # 开始节点
│   ├── generate.py     # 生成节点
│   ├── retrieval.py    # 检索节点
│   ├── rewrite.py      # 改写节点
│   ├── categorize.py   # 分类节点
│   ├── switch.py       # 条件分支
│   ├── message.py      # 消息节点
│   └── answer.py       # 回答节点
├── templates/           # Agent 模板
│   ├── qa_template.json
│   └── research_template.json
└── tools/               # Agent 工具
    ├── web_search.py   # 网络搜索
    ├── calculator.py   # 计算器
    └── code_runner.py  # 代码执行
```

#### 2.3.1 Agent 画布 (`canvas.py`)

**核心类**:
```python
class AgentCanvas:
    """Agent 工作流画布"""
    
    def __init__(self, canvas_id: str):
        self.canvas_id = canvas_id
        self.nodes = {}  # node_id -> Node
        self.edges = []  # (from_node, to_node, condition)
    
    async def run(self, inputs: dict) -> dict:
        """执行 Agent 工作流"""
        # 1. 找到开始节点
        start_node = self._find_start_node()
        
        # 2. 初始化上下文
        context = {"inputs": inputs, "variables": {}}
        
        # 3. 执行节点图
        result = await self._execute_node(start_node, context)
        
        return result
    
    async def _execute_node(self, node: Node, context: dict) -> dict:
        """递归执行节点"""
        # 1. 执行当前节点
        output = await node.execute(context)
        
        # 2. 更新上下文
        context["variables"][node.id] = output
        
        # 3. 找到下一个节点
        next_nodes = self._get_next_nodes(node, output)
        
        # 4. 递归执行
        if not next_nodes:
            return output  # 终止节点
        
        # 并行执行多个分支
        tasks = [self._execute_node(n, context) for n in next_nodes]
        results = await asyncio.gather(*tasks)
        
        return results
```

#### 2.3.2 Agent 节点类型

**检索节点**:
```python
class RetrievalNode(Node):
    """检索节点"""
    
    async def execute(self, context: dict) -> dict:
        query = self.config["query_template"].format(**context["variables"])
        kb_ids = self.config["kb_ids"]
        
        retriever = Retrieval(kb_ids)
        chunks = await retriever.retrieve(query, top_k=10)
        
        return {"chunks": chunks}
```

**生成节点**:
```python
class GenerateNode(Node):
    """LLM 生成节点"""
    
    async def execute(self, context: dict) -> dict:
        prompt = self.config["prompt_template"].format(**context["variables"])
        llm = self._get_llm(self.config["llm_id"])
        
        answer = await llm.chat([{"role": "user", "content": prompt}])
        
        return {"answer": answer}
```

**条件分支节点**:
```python
class SwitchNode(Node):
    """条件分支节点"""
    
    async def execute(self, context: dict) -> dict:
        condition = self.config["condition"]
        
        # 评估条件
        if self._eval_condition(condition, context):
            return {"branch": "true"}
        else:
            return {"branch": "false"}
```

### 2.4 文档解析引擎 (`deepdoc/`)

#### 目录结构
```
deepdoc/
├── __init__.py
├── parser/
│   ├── pdf_parser.py      # PDF 解析
│   ├── docx_parser.py     # Word 解析
│   ├── excel_parser.py    # Excel 解析
│   ├── ppt_parser.py      # PPT 解析
│   ├── html_parser.py     # HTML 解析
│   ├── markdown_parser.py # Markdown 解析
│   └── image_parser.py    # 图片 OCR
└── vision/
    ├── layout_analyzer.py # 版面分析
    ├── table_detector.py  # 表格检测
    └── ocr_engine.py      # OCR 引擎
```

#### 2.4.1 PDF 解析器 (`parser/pdf_parser.py`)

**解析流程**:
```python
class PDFParser:
    """PDF 解析器"""
    
    async def parse(self, pdf_path: str) -> Document:
        """解析 PDF 文档"""
        # 1. 提取文本
        text_blocks = await self._extract_text(pdf_path)
        
        # 2. 版面分析
        layout = await self._analyze_layout(pdf_path)
        
        # 3. 表格检测和提取
        tables = await self._extract_tables(pdf_path)
        
        # 4. 图片提取和 OCR
        images = await self._extract_images(pdf_path)
        ocr_results = await self._ocr_images(images)
        
        # 5. 合并结果
        document = self._merge_results(text_blocks, layout, tables, ocr_results)
        
        return document
    
    async def _extract_text(self, pdf_path: str) -> List[TextBlock]:
        """提取文本块"""
        import pdfplumber
        
        blocks = []
        with pdfplumber.open(pdf_path) as pdf:
            for page_num, page in enumerate(pdf.pages):
                text = page.extract_text()
                blocks.append(TextBlock(
                    text=text,
                    page_num=page_num,
                    bbox=page.bbox
                ))
        
        return blocks
    
    async def _extract_tables(self, pdf_path: str) -> List[Table]:
        """提取表格"""
        import camelot
        
        tables = camelot.read_pdf(pdf_path, pages='all')
        return [Table.from_camelot(table) for table in tables]
```

#### 2.4.2 OCR 引擎 (`vision/ocr_engine.py`)

**支持多种 OCR**:
```python
class OCREngine:
    """OCR 引擎"""
    
    def __init__(self, engine_type: str = "paddle"):
        self.engine_type = engine_type
        self.engine = self._init_engine()
    
    async def ocr(self, image: np.ndarray) -> List[TextRegion]:
        """执行 OCR"""
        if self.engine_type == "paddle":
            return await self._paddle_ocr(image)
        elif self.engine_type == "tesseract":
            return await self._tesseract_ocr(image)
        elif self.engine_type == "cloud":
            return await self._cloud_ocr(image)
    
    async def _paddle_ocr(self, image: np.ndarray) -> List[TextRegion]:
        """PaddleOCR"""
        from paddleocr import PaddleOCR
        
        ocr = PaddleOCR(lang='ch')
        results = ocr.ocr(image)
        
        return [TextRegion(
            text=line[1][0],
            confidence=line[1][1],
            bbox=line[0]
        ) for line in results[0]]
```

### 2.5 知识图谱 (`graphrag/`)

#### 目录结构
```
graphrag/
├── __init__.py
├── entity_resolution.py      # 实体识别和消歧
├── entity_resolution_prompt.py
├── query_analyze_prompt.py
├── search.py                 # 图谱检索
├── utils.py
├── general/                  # 通用图谱
│   ├── build.py             # 图谱构建
│   └── query.py             # 图谱查询
└── light/                    # 轻量图谱
    ├── build.py
    └── query.py
```

#### 2.5.1 实体识别 (`entity_resolution.py`)

**实体抽取**:
```python
class EntityResolver:
    """实体识别和消歧"""
    
    async def extract_entities(self, text: str) -> List[Entity]:
        """抽取实体"""
        # 1. 使用 LLM 提取实体
        prompt = ENTITY_EXTRACTION_PROMPT.format(text=text)
        llm_output = await self.llm.chat(prompt)
        
        # 2. 解析 JSON 输出
        entities = json.loads(llm_output)
        
        # 3. 实体消歧（链接到知识库）
        resolved_entities = await self._resolve_entities(entities)
        
        return resolved_entities
    
    async def extract_relations(self, text: str, entities: List[Entity]) -> List[Relation]:
        """抽取关系"""
        prompt = RELATION_EXTRACTION_PROMPT.format(
            text=text,
            entities=json.dumps([e.to_dict() for e in entities])
        )
        
        llm_output = await self.llm.chat(prompt)
        relations = json.loads(llm_output)
        
        return [Relation(**r) for r in relations]
```

#### 2.5.2 图谱构建 (`general/build.py`)

**构建流程**:
```python
class GraphBuilder:
    """知识图谱构建器"""
    
    async def build(self, kb_id: str):
        """构建知识图谱"""
        # 1. 获取所有文档
        documents = await self._get_documents(kb_id)
        
        # 2. 批量提取实体和关系
        entities = []
        relations = []
        for doc in documents:
            doc_entities = await self.entity_resolver.extract_entities(doc.content)
            doc_relations = await self.entity_resolver.extract_relations(doc.content, doc_entities)
            
            entities.extend(doc_entities)
            relations.extend(doc_relations)
        
        # 3. 实体融合（去重和合并）
        merged_entities = self._merge_entities(entities)
        
        # 4. 存储到图数据库
        await self._store_graph(merged_entities, relations)
        
        # 5. 计算图统计（PageRank, 社区检测）
        await self._compute_graph_stats(kb_id)
```

### 2.6 分块服务 (`rag/svr/chunk_service.py`)

**智能分块**:
```python
class ChunkService:
    """文档分块服务"""
    
    async def chunk_document(self, document: Document, strategy: str = "semantic") -> List[Chunk]:
        """智能分块"""
        if strategy == "semantic":
            return await self._semantic_chunk(document)
        elif strategy == "fixed":
            return await self._fixed_chunk(document)
        elif strategy == "recursive":
            return await self._recursive_chunk(document)
    
    async def _semantic_chunk(self, document: Document) -> List[Chunk]:
        """语义分块"""
        # 1. 句子分割
        sentences = self.sentence_splitter.split(document.content)
        
        # 2. 计算句子嵌入
        embeddings = await self.embedding_model.embed(sentences)
        
        # 3. 相似度聚类
        clusters = self._semantic_clustering(embeddings)
        
        # 4. 生成分块
        chunks = []
        for cluster in clusters:
            chunk_text = " ".join([sentences[i] for i in cluster])
            chunks.append(Chunk(
                content=chunk_text,
                doc_id=document.id,
                embedding=np.mean([embeddings[i] for i in cluster], axis=0)
            ))
        
        return chunks
```

---

## 3. 对外提供的服务

### 3.1 RAG 服务

| 服务 | 功能 | 接口 |
|------|------|------|
| 对话服务 | 同步/流式对话 | `RAGChat.chat()` / `RAGChat.stream()` |
| 检索服务 | 向量/全文/图谱检索 | `Retrieval.retrieve()` |
| 重排服务 | 结果重排序 | `Reranker.rerank()` |
| 嵌入服务 | 文本嵌入 | `EmbeddingModel.embed()` |

### 3.2 Agent 服务

| 服务 | 功能 | 接口 |
|------|------|------|
| 工作流执行 | 运行 Agent 画布 | `AgentCanvas.run()` |
| 工具调用 | 执行外部工具 | `ToolExecutor.execute()` |
| 记忆管理 | 短期/长期记忆 | `MemoryManager.store()` |

### 3.3 文档服务

| 服务 | 功能 | 接口 |
|------|------|------|
| 文档解析 | 多格式解析 | `Parser.parse()` |
| 文档分块 | 智能分块 | `ChunkService.chunk_document()` |
| OCR 识别 | 图片文字识别 | `OCREngine.ocr()` |

### 3.4 图谱服务

| 服务 | 功能 | 接口 |
|------|------|------|
| 实体抽取 | 提取实体和关系 | `EntityResolver.extract_entities()` |
| 图谱构建 | 构建知识图谱 | `GraphBuilder.build()` |
| 图谱检索 | 基于图谱的检索 | `GraphRetrieval.search()` |

---

## 4. 与其他层的交互方式

### 4.1 与 API 层交互

```python
# API 层 (dialog_app.py)
from rag.app.chat import RAGChat

@dialog_app.route('/chat', methods=['POST'])
async def chat():
    req = await request.get_json()
    
    # 调用业务逻辑层
    chat_engine = RAGChat(
        dialog_id=req["dialog_id"],
        kb_ids=req["kb_ids"],
        llm_id=req["llm_id"]
    )
    
    result = await chat_engine.chat(req["question"])
    
    return get_data_error_result(data=result)
```

### 4.2 与数据访问层交互

```python
# 业务逻辑层 (rag/app/retrieval.py)
from rag.utils.es_conn import ELASTICSEARCH

class Retrieval:
    async def _vector_search(self, query: str, top_k: int):
        # 调用 ES 客户端
        response = await ELASTICSEARCH.search(
            index="ragflow_chunks",
            body={
                "query": {
                    "knn": {
                        "embedding": {
                            "vector": query_vector,
                            "k": top_k
                        }
                    }
                }
            }
        )
        
        return response["hits"]["hits"]
```

---

## 5. 关键代码文件路径

### 5.1 RAG 引擎

| 文件 | 路径 | 行数 | 说明 |
|------|------|------|------|
| 对话引擎 | `rag/app/chat.py` | ~800 | 核心对话逻辑 |
| 检索器 | `rag/app/retrieval.py` | ~600 | 多路检索 |
| 重排序 | `rag/app/rerank.py` | ~300 | 结果重排 |
| LLM 基类 | `rag/llm/chat_model.py` | ~500 | 模型抽象 |
| 嵌入模型 | `rag/llm/embedding_model.py` | ~400 | 嵌入接口 |

### 5.2 Agent 系统

| 文件 | 路径 | 行数 | 说明 |
|------|------|------|------|
| 画布执行 | `agent/canvas.py` | ~700 | 工作流引擎 |
| 检索节点 | `agent/component/retrieval.py` | ~200 | 检索组件 |
| 生成节点 | `agent/component/generate.py` | ~250 | 生成组件 |
| 工具调用 | `agent/tools/` | ~1000 | 各种工具 |

### 5.3 文档解析

| 文件 | 路径 | 行数 | 说明 |
|------|------|------|------|
| PDF 解析 | `deepdoc/parser/pdf_parser.py` | ~600 | PDF 处理 |
| OCR 引擎 | `deepdoc/vision/ocr_engine.py` | ~400 | OCR 识别 |
| 版面分析 | `deepdoc/vision/layout_analyzer.py` | ~500 | 版面理解 |

### 5.4 知识图谱

| 文件 | 路径 | 行数 | 说明 |
|------|------|------|------|
| 实体识别 | `graphrag/entity_resolution.py` | ~400 | 实体抽取 |
| 图谱构建 | `graphrag/general/build.py` | ~600 | 构建流程 |
| 图谱检索 | `graphrag/search.py` | ~500 | 检索逻辑 |

---

## 6. 技术实现细节

### 6.1 RAG 检索策略

#### 6.1.1 混合检索 (Hybrid Search)
- **向量检索**: 语义相似度（Cosine Similarity）
- **全文检索**: BM25 算法
- **图谱检索**: 子图匹配
- **融合算法**: RRF (Reciprocal Rank Fusion)

#### 6.1.2 重排序算法
- **Cross-Encoder**: BERT-based 模型
- **ColBERT**: 基于 token 的细粒度匹配
- **LLM Rerank**: 使用 LLM 评分

### 6.2 Agent 工作流

#### 6.2.1 DAG 执行
- **拓扑排序**: 确定执行顺序
- **并行执行**: 独立分支并行
- **状态管理**: 上下文传递

#### 6.2.2 工具集成
- **Web Search**: DuckDuckGo, Google
- **Code Execution**: Python, JavaScript
- **API Calls**: REST, GraphQL

### 6.3 文档解析优化

#### 6.3.1 OCR 优化
- **图像预处理**: 去噪、二值化
- **多模型融合**: Paddle + Tesseract
- **后处理**: 拼写纠正、格式化

#### 6.3.2 表格提取
- **表格检测**: YOLOv5
- **单元格识别**: Grid 分析
- **结构化输出**: Markdown/HTML

---

## 7. 性能优化

### 7.1 向量搜索优化
- **索引优化**: HNSW, IVF
- **量化**: INT8, Binary
- **批处理**: 批量嵌入

### 7.2 LLM 推理优化
- **模型量化**: GPTQ, AWQ
- **批处理**: Dynamic Batching
- **缓存**: KV Cache

### 7.3 异步并发
- **AsyncIO**: 全异步实现
- **连接池**: 复用连接
- **并发控制**: Semaphore

---

## 8. 总结

### 8.1 层的特点
- ✅ **高性能**: 异步架构、批处理
- ✅ **可扩展**: 插件化设计
- ✅ **多模态**: 文本、图片、表格
- ✅ **智能化**: LLM、Agent、GraphRAG
- ✅ **可配置**: 灵活的策略选择

### 8.2 最佳实践
1. **异步优先**: 全异步实现
2. **模块化**: 清晰的职责划分
3. **性能优化**: 缓存、批处理、并发
4. **错误处理**: 优雅降级
5. **监控日志**: 详细的执行日志

---

**文档版本**: 1.0  
**最后更新**: 2025-12-18  
**维护者**: RAGFlow 团队
