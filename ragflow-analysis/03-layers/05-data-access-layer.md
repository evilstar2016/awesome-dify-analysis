# 05 - 数据访问层 (Data Access Layer)

## 1. 层职责与定位

### 1.1 职责
数据访问层负责封装所有数据存储和访问操作：
- **ORM 模型**: 双ORM架构
  - **Peewee ORM**: 应用层业务数据模型定义（用户、知识库、文档等）
  - **SQLAlchemy**: 向量数据库操作（OceanBase向量数据库）
- **服务层**: 封装CRUD操作和业务查询
- **搜索客户端**: Elasticsearch/Infinity 向量搜索
- **缓存客户端**: Redis 缓存和消息队列
- **对象存储**: MinIO文件存储操作
- **数据库迁移**: 数据库版本管理

### 1.2 定位
- **架构位置**: 业务逻辑层和基础设施层之间
- **技术选型**: 双ORM架构
  - **Peewee ORM**: 应用层CRUD操作（轻量级、简洁）
  - **SQLAlchemy**: 向量数据库操作（强大、灵活）
  - **各类数据库客户端**: ES、Redis、MinIO等
- **设计模式**: Repository 模式、DAO 模式
- **核心模块**: `api/db/` (Peewee), `rag/utils/ob_conn.py` (SQLAlchemy)

---

## 2. 主要组件/模块

### 2.1 目录结构

```
api/db/                       # Peewee ORM - 应用层业务数据
├── __init__.py
├── db_models.py              # Peewee ORM 模型定义
├── init_data.py              # 初始数据
├── runtime_config.py         # 运行时配置
├── db_utils.py               # 数据库工具
├── services/                 # 服务层
│   ├── __init__.py
│   ├── user_service.py       # 用户服务
│   ├── knowledgebase_service.py  # 知识库服务
│   ├── document_service.py   # 文档服务
│   ├── dialog_service.py     # 对话服务
│   ├── conversation_service.py   # 会话服务
│   ├── task_service.py       # 任务服务
│   └── ...
└── joint_services/           # 联合服务

rag/utils/                    # SQLAlchemy - 向量数据库操作
└── ob_conn.py                # OceanBase向量数据库连接（使用SQLAlchemy）
```

### 2.2 Peewee ORM 模型 (`api/db/db_models.py`)

> **说明**: Peewee用于应用层业务数据（用户、租户、知识库、文档、对话等）

#### 2.2.1 核心模型

**用户模型**:
```python
from peewee import CharField, DateTimeField, BooleanField, TextField
from api.db.db_models import DataBaseModel

class User(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    nickname = CharField(max_length=100, null=False, help_text="nickname", index=True)
    email = CharField(max_length=255, null=False, help_text="email", index=True)
    password = CharField(max_length=255, null=True, help_text="password", index=True)
    avatar = TextField(null=True, help_text="avatar base64 string")
    is_superuser = BooleanField(null=True, help_text="is root", default=False, index=True)
    login_channel = CharField(null=True, help_text="from which user login", index=True)
    last_login_time = DateTimeField(null=True, index=True)
    status = CharField(max_length=1, null=True, default="1", index=True)
    
    def to_dict(self):
        return {
            "id": self.id,
            "nickname": self.nickname,
            "email": self.email,
            "avatar": self.avatar,
            "is_superuser": self.is_superuser,
            # 不包含密码
        }
    
    class Meta:
        db_table = "user"
```

**知识库模型**:
```python
from peewee import CharField, TextField, IntegerField, DateTimeField
from api.db.db_models import JSONField

class Knowledgebase(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    tenant_id = CharField(max_length=32, null=False, index=True)
    name = CharField(max_length=128, null=False)
    description = TextField(null=True)
    language = CharField(max_length=32, default='English')
    embd_id = CharField(max_length=32, null=True)  # 嵌入模型ID
    chunk_num = IntegerField(default=0)  # 分块数量
    doc_num = IntegerField(default=0)  # 文档数量
    parser_config = JSONField(null=True)  # 解析配置
    avatar = CharField(max_length=512, null=True)
    
    class Meta:
        db_table = "knowledgebase"
```

**文档模型**:
```python
from peewee import CharField, IntegerField, FloatField

class Document(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    kb_id = CharField(max_length=32, null=False, index=True)
    name = CharField(max_length=256, null=False)
    type = CharField(max_length=32, null=True)  # pdf, docx, ppt, etc.
    size = IntegerField(default=0)  # 文件大小（字节）
    location = CharField(max_length=512, null=True)  # MinIO 路径
    parser_id = CharField(max_length=32, null=True)  # 使用的解析器
    parser_config = JSONField(null=True)
    progress = FloatField(default=0)  # 解析进度 0-1
    progress_msg = CharField(max_length=256, null=True)
    chunk_num = IntegerField(default=0)
    token_num = IntegerField(default=0)
    status = CharField(max_length=1, default='1', index=True)  # 状态
    
    class Meta:
        db_table = "document"
```

**对话模型**:
```python
from peewee import CharField, TextField, FloatField, IntegerField, JSONField, DateTimeField

class Dialog(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    tenant_id = CharField(max_length=32, null=False, index=True)
    name = CharField(max_length=128, null=False)
    description = TextField(null=True)
    icon = CharField(max_length=512, null=True)
    kb_ids = JSONField(null=True)  # 关联的知识库ID列表
    llm_id = CharField(max_length=32, null=True)
    llm_setting = JSONField(null=True)  # temperature, top_p, etc.
    prompt_config = JSONField(null=True)  # 提示词配置
    rerank_id = CharField(max_length=32, null=True)
    similarity_threshold = FloatField(default=0.2)
    vector_similarity_weight = FloatField(default=0.3)
    top_k = IntegerField(default=6)
    top_n = IntegerField(default=8)
    status = CharField(max_length=1, default='1')  # 1: 启用, 0: 禁用
    create_time = DateTimeField(null=True)
    update_time = DateTimeField(null=True)
    
    class Meta:
        db_table = "dialog"
```

**会话模型**:
```python
class Conversation(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    dialog_id = CharField(max_length=32, null=False, index=True)
    user_id = CharField(max_length=32, null=True, index=True)
    reference = JSONField(null=True)  # 引用的文档块
    message = JSONField(null=True)  # 消息历史 [{"role": "user", "content": "..."}, ...]
    create_time = DateTimeField(null=True)
    update_time = DateTimeField(null=True)
    
    class Meta:
        db_table = "conversation"
```

**任务模型**:
```python
class Task(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    task_type = CharField(max_length=32, null=True)  # parse_document, build_graph, raptor
    doc_id = CharField(max_length=32, null=True, index=True)
    from_page = IntegerField(default=0)
    to_page = IntegerField(default=-1)
    progress = FloatField(default=0)
    progress_msg = TextField(null=True)
    retry_count = IntegerField(default=0)
    begin_time = DateTimeField(null=True)
    end_time = DateTimeField(null=True)
    
    class Meta:
        db_table = "task"
```

**分块模型（存储在Elasticsearch/Infinity/OceanBase，不在关系数据库）**:
```python
# 分块存储在向量数据库，通过 ES/Infinity/OceanBase 客户端访问
class Chunk:
    """分块数据结构（非 Peewee ORM 模型，存储在向量数据库）"""
    id: str
    kb_id: str
    doc_id: str
    content: str
    content_with_weight: str
    embedding: List[float]
    important_keywords: List[str]
    create_time: datetime
    img_id: str  # 关联图片
    page_num: int
    position: dict  # 页面位置
```

### 2.2.2 SQLAlchemy 向量数据库表结构 (`rag/utils/ob_conn.py`)

> **说明**: SQLAlchemy用于OceanBase向量数据库操作（通过pyobvector客户端）

**OceanBase向量表列定义**:
```python
from sqlalchemy import Column, String, Integer, JSON, Double, ARRAY, text
from sqlalchemy.types import TypeDecorator

# OBConnection 类使用 SQLAlchemy Column 定义向量表结构
class OBConnection:
    def create(self, kb_id: str, vector_size: int):
        """创建向量表"""
        # 使用 SQLAlchemy Column 定义
        columns = {
            "id": Column(String(128), primary_key=True),
            "kb_id": Column(String(128)),
            "doc_id": Column(String(128)),
            "content": Column(String(65535)),  # LONGTEXT
            "embedding": Column(ARRAY(Double), dimensions=vector_size),  # 向量列
            "important_keywords": Column(JSON),
            "img_id": Column(String(128)),
            "create_time": Column(String(32)),
            "page_num": Column(Integer)
        }
        # 通过 pyobvector.ObVecClient 创建表
        self.client.perform_raw_text_sql(
            f"CREATE TABLE {kb_id} (...)"
        )
    
    def search(self, kb_id: str, embd_vec: List[float], topn: int):
        """向量检索"""
        # 使用 SQLAlchemy text() 构建 SQL
        sql = text(f"""
            SELECT id, content, kb_id, 
                   l2_distance(embedding, :vector) as distance
            FROM {kb_id}
            ORDER BY distance
            LIMIT :topn
        """)
        return self.client.execute(sql, {"vector": embd_vec, "topn": topn})
```

### 2.3 服务层 (`services/`)

#### 2.3.1 知识库服务 (`knowledgebase_service.py`)

```python
from api.db.db_models import Knowledgebase
from api.db.services.common_service import CommonService

class KnowledgebaseService(CommonService):
    """知识库服务"""
    model = Knowledgebase
    
    @classmethod
    def get_by_id(cls, kb_id: str) -> Knowledgebase:
        """根据ID获取"""
        try:
            return cls.model.get(cls.model.id == kb_id)
        except Exception:
            return None
    
    @classmethod
    def get_list(cls, tenant_id: str, page: int = 1, page_size: int = 10):
        """分页获取列表"""
        query = cls.model.select().where(cls.model.tenant_id == tenant_id)
        total = query.count()
        kbs = list(query.paginate(page, page_size))
        return kbs, total
    
    @classmethod
    def delete(cls, kb_id: str) -> bool:
        """删除知识库"""
        kb = cls.get_by_id(kb_id)
        if not kb:
            return False
        
        # 删除关联的文档和分块
        from api.db.services.document_service import DocumentService
        DocumentService.delete_by_kb(kb_id)
        
        kb.delete_instance()
        return True
    
    @classmethod
    def update(cls, kb_id: str, **kwargs) -> Knowledgebase:
        """更新知识库"""
        kb = cls.get_by_id(kb_id)
        if not kb:
            return None
        
        for key, value in kwargs.items():
            if hasattr(kb, key):
                setattr(kb, key, value)
        
        kb.save()
        return kb
    
    @classmethod
    def increment_chunk_num(cls, kb_id: str, delta: int = 1):
        """增加分块数量"""
        cls.model.update(chunk_num=cls.model.chunk_num + delta).where(
            cls.model.id == kb_id
        ).execute()
```

#### 2.3.2 文档服务 (`document_service.py`)

```python
from api.db.db_models import Document
from api.db.services.common_service import CommonService

class DocumentService(CommonService):
    """文档服务"""
    model = Document
    
    @classmethod
    def get_by_id(cls, doc_id: str) -> Document:
        """获取文档"""
        try:
            return cls.model.get(cls.model.id == doc_id)
        except Exception:
            return None
    
    @classmethod
    def get_by_kb(cls, kb_id: str) -> list:
        """获取知识库下的所有文档"""
        return list(cls.model.select().where(cls.model.kb_id == kb_id))
    
    @classmethod
    def update_progress(cls, doc_id: str, progress: float, msg: str = ""):
        """更新解析进度"""
        cls.model.update(
            progress=progress,
            progress_msg=msg
        ).where(cls.model.id == doc_id).execute()
    
    @classmethod
    def update_status(cls, doc_id: str, status: str):
        """更新文档状态"""
        cls.model.update(status=status).where(cls.model.id == doc_id).execute()
    
    @classmethod
    def delete(cls, doc_id: str) -> bool:
        """删除文档"""
        doc = cls.get_by_id(doc_id)
        if not doc:
            return False
        
        # 删除MinIO中的文件
        from rag.utils.minio_conn import RAGFlowMinio
        minio = RAGFlowMinio()
        minio.rm(doc.location)
        
        # 删除ES中的分块
        from rag.utils.es_conn import ESConnection
        es = ESConnection()
        es.delete({"doc_id": doc_id}, "doc_id")
        
        doc.delete_instance()
        return True
```

#### 2.3.3 对话服务 (`dialog_service.py`)

```python
from api.db.db_models import Dialog
from api.db.services.common_service import CommonService

class DialogService(CommonService):
    """对话服务"""
    model = Dialog
    
    @classmethod
    def get_by_id(cls, dialog_id: str) -> Dialog:
        """获取对话助手"""
        try:
            return cls.model.get(cls.model.id == dialog_id)
        except Exception:
            return None
    
    @classmethod
    def get_list(cls, tenant_id: str):
        """获取租户下的所有对话助手"""
        return list(cls.model.select().where(cls.model.tenant_id == tenant_id))
    
    @classmethod
    def update_llm_setting(cls, dialog_id: str, setting: dict):
        """更新LLM配置"""
        cls.model.update(llm_setting=setting).where(
            cls.model.id == dialog_id
        ).execute()
```

### 2.4 搜索客户端 (`rag/utils/`)

#### 2.4.1 Elasticsearch 客户端 (`es_conn.py`)

```python
from elasticsearch import AsyncElasticsearch

class ESConnection:
    """Elasticsearch 连接管理"""
    
    def __init__(self):
        self.client = None
    
    def connect(self, hosts: List[str], username: str, password: str):
        """建立连接"""
        self.client = AsyncElasticsearch(
            hosts=hosts,
            basic_auth=(username, password),
            verify_certs=False,
            max_retries=3,
            retry_on_timeout=True
        )
    
    async def create_index(self, index_name: str, mappings: dict):
        """创建索引"""
        await self.client.indices.create(
            index=index_name,
            body={
                "settings": {
                    "number_of_shards": 3,
                    "number_of_replicas": 1
                },
                "mappings": mappings
            }
        )
    
    async def insert_chunk(self, chunk: Chunk):
        """插入分块"""
        await self.client.index(
            index="ragflow_chunks",
            id=chunk.id,
            body=chunk.to_dict()
        )
    
    async def vector_search(self, vector: List[float], kb_ids: List[str], top_k: int):
        """向量检索"""
        query = {
            "size": top_k,
            "query": {
                "bool": {
                    "must": [
                        {"terms": {"kb_id": kb_ids}},
                        {
                            "knn": {
                                "embedding": {
                                    "vector": vector,
                                    "k": top_k
                                }
                            }
                        }
                    ]
                }
            }
        }
        
        response = await self.client.search(index="ragflow_chunks", body=query)
        return response["hits"]["hits"]
    
    async def fulltext_search(self, query_text: str, kb_ids: List[str], top_k: int):
        """全文检索"""
        query = {
            "size": top_k,
            "query": {
                "bool": {
                    "must": [
                        {"terms": {"kb_id": kb_ids}},
                        {"match": {"content": query_text}}
                    ]
                }
            }
        }
        
        response = await self.client.search(index="ragflow_chunks", body=query)
        return response["hits"]["hits"]

# 全局单例
ELASTICSEARCH = ESConnection()
```

#### 2.4.2 Infinity 客户端 (`infinity_conn.py`)

```python
import infinity

class InfinityConnection:
    """Infinity 向量数据库连接"""
    
    def __init__(self):
        self.client = None
    
    def connect(self, host: str, port: int):
        """建立连接"""
        self.client = infinity.connect(infinity.common.NetworkAddress(host, port))
    
    async def create_table(self, table_name: str, columns: dict):
        """创建表"""
        db = self.client.get_database("default")
        table = db.create_table(
            table_name,
            {
                "id": {"type": "varchar"},
                "content": {"type": "varchar"},
                "embedding": {"type": "vector,1536,float"},  # 1536维向量
                # ...更多列
            }
        )
    
    async def insert_chunk(self, chunk: Chunk):
        """插入分块"""
        db = self.client.get_database("default")
        table = db.get_table("ragflow_chunks")
        
        table.insert([chunk.to_dict()])
    
    async def vector_search(self, vector: List[float], kb_ids: List[str], top_k: int):
        """向量检索"""
        db = self.client.get_database("default")
        table = db.get_table("ragflow_chunks")
        
        result = table.output(["id", "content", "kb_id"]).knn(
            "embedding", vector, "float", "cosine", top_k
        ).filter(f"kb_id IN ({','.join(kb_ids)})").to_pl()
        
        return result

# 全局单例
INFINITY = InfinityConnection()
```

### 2.5 Redis 客户端 (`rag/utils/redis_conn.py`)

```python
import redis.asyncio as redis

class RedisConnection:
    """Redis 连接管理"""
    
    def __init__(self):
        self.client = None
    
    def connect(self, host: str, port: int, password: str, db: int = 0):
        """建立连接"""
        self.client = redis.Redis(
            host=host,
            port=port,
            password=password,
            db=db,
            decode_responses=True,
            max_connections=100
        )
    
    async def set(self, key: str, value: str, expire: int = None):
        """设置缓存"""
        await self.client.set(key, value, ex=expire)
    
    async def get(self, key: str) -> str:
        """获取缓存"""
        return await self.client.get(key)
    
    async def delete(self, key: str):
        """删除缓存"""
        await self.client.delete(key)
    
    async def publish(self, channel: str, message: str):
        """发布消息"""
        await self.client.publish(channel, message)
    
    async def subscribe(self, channel: str):
        """订阅消息"""
        pubsub = self.client.pubsub()
        await pubsub.subscribe(channel)
        async for message in pubsub.listen():
            if message["type"] == "message":
                yield message["data"]

# 全局单例
REDIS_CONN = RedisConnection()

class RedisDistributedLock:
    """Redis 分布式锁"""
    
    def __init__(self, lock_key: str, lock_value: str, timeout: int = 60):
        self.lock_key = f"lock:{lock_key}"
        self.lock_value = lock_value
        self.timeout = timeout
    
    def acquire(self) -> bool:
        """获取锁"""
        return REDIS_CONN.client.set(
            self.lock_key,
            self.lock_value,
            nx=True,  # 只在键不存在时设置
            ex=self.timeout
        )
    
    def release(self):
        """释放锁"""
        # Lua脚本保证原子性
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        REDIS_CONN.client.eval(lua_script, 1, self.lock_key, self.lock_value)
```

### 2.6 MinIO 客户端 (`rag/utils/minio_conn.py`)

```python
from rag.utils import BaseConnection

class RAGFlowMinio(BaseConnection):
    """RAGFlow MinIO 对象存储客户端"""
    
    def __init__(self):
        super().__init__()
        # 从配置加载连接信息
        self.conn_cfg = self.get_connection_cfg()
    
    def put(self, bucket: str, fnm: str, binary: bytes):
        """上传文件"""
        # 实际实现使用MinIO SDK
        pass
    
    def get(self, bucket: str, fnm: str) -> bytes:
        """下载文件"""
        # 返回文件二进制内容
        pass
    
    def rm(self, fnm: str):
        """删除文件"""
        # 删除指定文件
        pass
    
    def obj_exist(self, bucket: str, fnm: str) -> bool:
        """检查对象是否存在"""
        pass
```

---

## 3. 对外提供的接口或服务

### 3.1 Peewee ORM 模型服务（应用层业务数据）

| 服务 | 功能 | 方法 |
|------|------|------|
| `UserService` | 用户CRUD | `save()`, `get_by_id()`, `delete()` |
| `KnowledgebaseService` | 知识库CRUD | `save()`, `get_list()`, `update()` |
| `DocumentService` | 文档CRUD | `save()`, `update_progress()`, `delete()` |
| `DialogService` | 对话CRUD | `save()`, `get_by_id()`, `update()` |
| `ConversationService` | 会话CRUD | `save()`, `get_list()`, `append_message()` |

### 3.1.5 SQLAlchemy 向量数据库服务

| 服务 | 功能 | 方法 |
|------|------|------|
| `OBConnection` | OceanBase向量操作 | `create()`, `upsert()`, `search()`, `hybrid_search()`, `delete()` |

### 3.2 搜索服务

| 服务 | 功能 | 方法 |
|------|------|------|
| `ELASTICSEARCH` | ES检索 | `vector_search()`, `fulltext_search()` |
| `INFINITY` | Infinity检索 | `vector_search()` |

### 3.3 缓存服务

| 服务 | 功能 | 方法 |
|------|------|------|
| `REDIS_CONN` | 缓存操作 | `set()`, `get()`, `delete()` |
| `RedisDistributedLock` | 分布式锁 | `acquire()`, `release()` |

### 3.4 存储服务

| 服务 | 功能 | 方法 |
|------|------|------|
| `RAGFlowMinio` | 对象存储 | `put()`, `get()`, `rm()`, `obj_exist()` |

---

## 4. 与其他层的交互方式

### 4.1 与业务逻辑层交互

```python
# 业务逻辑层 (rag/app/retrieval.py)
from rag.utils.es_conn import ELASTICSEARCH

class Retrieval:
    async def retrieve(self, query: str):
        # 调用数据访问层
        hits = await ELASTICSEARCH.vector_search(
            vector=query_vector,
            kb_ids=self.kb_ids,
            top_k=10
        )
        return hits
```

### 4.2 与基础设施层交互

```python
# 数据访问层连接到基础设施层
ELASTICSEARCH.connect(
    hosts=["elasticsearch:9200"],
    username="elastic",
    password="password"
)

REDIS_CONN.connect(
    host="redis",
    port=6379,
    password="password"
)
```

---

## 5. 关键代码文件路径

| 文件 | 路径 | 行数 | 说明 |
|------|------|------|------|
| Peewee ORM模型 | `api/db/db_models.py` | ~1500 | 应用层业务数据模型 |
| 知识库服务 | `api/db/services/knowledgebase_service.py` | ~300 | 知识库CRUD |
| 文档服务 | `api/db/services/document_service.py` | ~400 | 文档CRUD |
| SQLAlchemy向量DB | `rag/utils/ob_conn.py` | ~1600 | OceanBase向量数据库操作 |
| ES客户端 | `rag/utils/es_conn.py` | ~500 | Elasticsearch |
| Redis客户端 | `rag/utils/redis_conn.py` | ~300 | Redis |
| MinIO客户端 | `rag/utils/minio_conn.py` | ~200 | 对象存储 |

---

## 6. 技术实现细节

### 6.1 Peewee ORM优化

#### 6.1.1 索引优化
```python
class Knowledgebase(DataBaseModel):
    tenant_id = CharField(max_length=32, null=False, index=True)  # 添加索引
    kb_id = CharField(max_length=32, null=False, index=True)
    status = CharField(max_length=1, null=True, index=True)
    
    class Meta:
        db_table = "knowledgebase"
        indexes = (
            (('tenant_id', 'status'), False),  # 复合索引
        )
```

#### 6.1.2 查询优化
```python
# 使用 select() 仅获取需要的字段
kbs = Knowledgebase.select(Knowledgebase.id, Knowledgebase.name)

# 使用 prefetch 优化关联查询
from peewee import prefetch
kbs = Knowledgebase.select()
docs = Document.select()
kbs_with_docs = prefetch(kbs, docs)
```

### 6.2 连接池管理

```python
from peewee import MySQLDatabase, PostgresqlDatabase
from playhouse.pool import PooledMySQLDatabase, PooledPostgresqlDatabase

# 使用连接池数据库
db = PooledMySQLDatabase(
    'ragflow',
    max_connections=20,
    stale_timeout=300,
    timeout=0,
    user='ragflow',
    password='infiniflow',
    host='localhost',
    port=3306
)
```

---

## 7. 总结

- ✅ **双ORM架构**: 
  - **Peewee**: 应用层业务数据CRUD（轻量级、简洁）
  - **SQLAlchemy**: 向量数据库操作（强大、灵活）
- ✅ **服务模式**: Repository模式简化调用
- ✅ **多数据源**: 支持MySQL/PostgreSQL、OceanBase、ES、Redis、MinIO
- ✅ **连接池**: 高效管理数据库连接
- ✅ **事务管理**: 保证数据一致性
- ✅ **职责分离**: 业务数据与向量数据分层管理

---

**文档版本**: 1.0  
**最后更新**: 2025-12-18  
**维护者**: RAGFlow 团队
