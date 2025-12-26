# RAGFlow 数据架构文档

## 概述

RAGFlow 采用多层次、多存储的混合数据架构设计，结合关系型数据库、向量数据库、全文检索引擎、缓存系统和对象存储，构建了一个高效、可扩展的 RAG（检索增强生成）系统。

### 设计理念

- **存储分离**：将结构化数据、向量数据、文档内容和文件分别存储，实现存储层的解耦
- **弹性扩展**：支持多种数据库实现（MySQL/PostgreSQL、Elasticsearch/OpenSearch、Infinity/OceanBase），适应不同部署场景
- **高性能**：通过 Redis 缓存和任务队列机制，实现高并发处理和低延迟响应
- **一致性保证**：基于事务机制和级联删除策略，确保数据一致性

### 技术选型

| 存储类型 | 技术选型 | 主要用途 |
|---------|---------|---------|
| **关系型数据库** | MySQL 5.7+ / PostgreSQL 13+ | 存储业务元数据、用户信息、租户配置、知识库信息等 |
| **向量数据库** | Infinity / OceanBase Vector | 存储文档向量，支持向量相似度检索 |
| **全文检索引擎** | Elasticsearch 8.x / OpenSearch 2.x | 存储文档块（chunks），支持全文检索和混合检索 |
| **图数据库** | NetworkX（内存图） | 存储知识图谱（GraphRAG），实体和关系网络 |
| **缓存/队列** | Redis 6.x+ (Valkey) | 缓存热点数据、分布式锁、任务队列 |
| **对象存储** | MinIO / S3 / OSS / Azure Blob | 存储原始文件、图片、解析结果等 |

---

## 数据库架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用层 (API/Service)                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┬───────────────┐
         │               │               │               │
         ▼               ▼               ▼               ▼
┌────────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   ORM/Peewee   │ │ ES/OS Client │ │ Redis Client │ │NetworkX Graph│
└────────┬───────┘ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
         │                │                 │                 │
         ▼                ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         数据持久化层                                      │
├─────────────┬─────────────┬─────────────┬───────────┬────────────┬──────┤
│  MySQL/PG   │   Infinity  │   ES/OS     │  NetworkX │   Redis    │MinIO │
│  (元数据)   │  (向量索引) │  (全文检索) │ (知识图谱)│ (缓存/队列)│(文件)│
└─────────────┴─────────────┴─────────────┴───────────┴────────────┴──────┘
```

---

## 1. MySQL/PostgreSQL 数据库设计

### 1.1 连接管理

**配置文件**: `conf/service_conf.yaml`

```yaml
mysql:
  name: 'rag_flow'
  user: 'root'
  password: 'infini_rag_flow'
  host: 'localhost'
  port: 5455
  max_connections: 900      # 最大连接数
  stale_timeout: 300        # 连接过期时间(秒)
  max_allowed_packet: 1073741824  # 最大包大小(1GB)
```

**连接池设置**:
- 使用 Peewee ORM 的 `PooledMySQLDatabase` 或 `PooledPostgresqlDatabase`
- 连接池大小: 900
- 自动清理过期连接: 30 秒间隔
- 支持连接重试机制

### 1.2 核心数据模型

#### 1.2.1 用户与租户模型

**User (用户表)**
```python
{
  id: CharField(32, primary_key)         # 用户ID
  access_token: CharField(255, indexed)  # 访问令牌
  nickname: CharField(100, indexed)      # 昵称
  password: CharField(255, indexed)      # 密码哈希
  email: CharField(255, indexed)         # 邮箱（唯一）
  avatar: TextField                      # 头像 Base64
  language: CharField(32)                # 语言设置
  color_schema: CharField(32)            # 主题设置
  timezone: CharField(64)                # 时区
  last_login_time: DateTimeField         # 最后登录时间
  is_authenticated: CharField(1)         # 认证状态
  is_active: CharField(1)                # 激活状态
  is_superuser: BooleanField             # 是否超级用户
  login_channel: CharField               # 登录渠道
  status: CharField(1, default='1')      # 状态
  create_time: BigIntegerField           # 创建时间戳
  update_time: BigIntegerField           # 更新时间戳
}

索引:
- PRIMARY KEY: id
- INDEX: access_token, email, nickname
- INDEX: create_time, update_time
```

**Tenant (租户表)**
```python
{
  id: CharField(32, primary_key)         # 租户ID
  name: CharField(100, indexed)          # 租户名称
  public_key: CharField(255)             # 公钥
  llm_id: CharField(128, indexed)        # 默认LLM ID
  embd_id: CharField(128, indexed)       # 默认嵌入模型ID
  asr_id: CharField(128, indexed)        # 默认ASR模型ID
  img2txt_id: CharField(128, indexed)    # 默认图像转文本模型ID
  rerank_id: CharField(128, indexed)     # 默认重排模型ID
  tts_id: CharField(256)                 # 默认TTS模型ID
  parser_ids: CharField(256)             # 文档处理器列表
  credit: IntegerField(default=512)      # 积分
  status: CharField(1, default='1')      # 状态
  create_time: BigIntegerField
  update_time: BigIntegerField
}

索引:
- PRIMARY KEY: id
- INDEX: llm_id, embd_id, rerank_id
```

**UserTenant (用户-租户关联表)**
```python
{
  id: CharField(32, primary_key)
  user_id: CharField(32, indexed)        # 外键 -> User.id
  tenant_id: CharField(32, indexed)      # 外键 -> Tenant.id
  role: CharField(32, indexed)           # 角色（UserTenantRole）
  invited_by: CharField(32, indexed)     # 邀请人
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: id
- INDEX: user_id, tenant_id, role
```

#### 1.2.2 知识库模型

**Knowledgebase (知识库表)**
```python
{
  id: CharField(32, primary_key)
  avatar: TextField                      # 头像
  tenant_id: CharField(32, indexed)      # 外键 -> Tenant.id
  name: CharField(128, indexed)          # 知识库名称
  language: CharField(32)                # 语言
  description: TextField                 # 描述
  embd_id: CharField(128, indexed)       # 嵌入模型ID
  permission: CharField(16, default='me')# 权限：me|team
  created_by: CharField(32, indexed)     # 创建者
  doc_num: IntegerField(default=0)       # 文档数量
  token_num: IntegerField(default=0)     # Token总数
  chunk_num: IntegerField(default=0)     # 分块数量
  similarity_threshold: FloatField(0.2)  # 相似度阈值
  vector_similarity_weight: FloatField(0.3) # 向量权重
  parser_id: CharField(32, indexed)      # 默认解析器ID
  pipeline_id: CharField(32, indexed)    # Pipeline ID
  parser_config: JSONField               # 解析配置
  pagerank: IntegerField(default=0)      # PageRank值
  graphrag_task_id: CharField(32)        # Graph RAG任务ID
  graphrag_task_finish_at: DateTimeField
  raptor_task_id: CharField(32)          # RAPTOR任务ID
  raptor_task_finish_at: DateTimeField
  mindmap_task_id: CharField(32)         # 思维导图任务ID
  mindmap_task_finish_at: DateTimeField
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: id
- INDEX: tenant_id, name, embd_id, parser_id
- INDEX: created_by, pipeline_id
```

**Document (文档表)**
```python
{
  id: CharField(32, primary_key)
  thumbnail: TextField                   # 缩略图
  kb_id: CharField(256, indexed)         # 外键 -> Knowledgebase.id
  parser_id: CharField(32, indexed)      # 解析器ID
  pipeline_id: CharField(32, indexed)    # Pipeline ID
  parser_config: JSONField               # 解析配置
  source_type: CharField(128, indexed)   # 来源类型
  type: CharField(32, indexed)           # 文件扩展名
  created_by: CharField(32, indexed)     # 创建者
  name: CharField(255, indexed)          # 文件名
  location: CharField(255)               # 存储位置（MinIO路径）
  size: IntegerField(indexed)            # 文件大小（字节）
  token_num: IntegerField                # Token数量
  chunk_num: IntegerField                # 分块数量
  progress: FloatField(indexed)          # 处理进度(0-1)
  progress_msg: TextField                # 进度消息
  process_begin_at: DateTimeField        # 处理开始时间
  process_duration: FloatField           # 处理时长(秒)
  meta_fields: JSONField                 # 元数据字段
  suffix: CharField(32, indexed)         # 真实文件后缀
  run: CharField(1, default='0')         # 运行状态：0停止,1运行,2取消
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: id
- INDEX: kb_id, parser_id, source_type, type
- INDEX: progress, process_begin_at
```

#### 1.2.3 对话模型

**Dialog (对话应用表)**
```python
{
  id: CharField(32, primary_key)
  tenant_id: CharField(32, indexed)      # 外键 -> Tenant.id
  name: CharField(255, indexed)          # 对话应用名称
  description: TextField                 # 描述
  icon: TextField                        # 图标
  language: CharField(32)                # 语言
  llm_id: CharField(128)                 # LLM模型ID
  llm_setting: JSONField                 # LLM配置
  prompt_type: CharField(16, indexed)    # 提示类型：simple|advanced
  prompt_config: JSONField               # 提示配置
  meta_data_filter: JSONField            # 元数据过滤
  similarity_threshold: FloatField(0.2)
  vector_similarity_weight: FloatField(0.3)
  top_n: IntegerField(default=6)         # 检索Top N
  top_k: IntegerField(default=1024)      # 重排Top K
  do_refer: CharField(1, default='1')    # 是否引用
  rerank_id: CharField(128)              # 重排模型ID
  kb_ids: JSONField(default=[])          # 关联知识库ID列表
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: id
- INDEX: tenant_id, name, prompt_type
```

**Conversation (对话记录表)**
```python
{
  id: CharField(32, primary_key)
  dialog_id: CharField(32, indexed)      # 外键 -> Dialog.id
  name: CharField(255, indexed)          # 对话名称
  message: JSONField                     # 消息内容
  reference: JSONField(default=[])       # 引用信息
  user_id: CharField(255, indexed)       # 用户ID
  create_time: BigIntegerField
  update_time: BigIntegerField
}

索引:
- PRIMARY KEY: id
- INDEX: dialog_id, user_id
```

**API4Conversation (API对话记录表)**
```python
{
  id: CharField(32, primary_key)
  dialog_id: CharField(32, indexed)      # 外键 -> Dialog.id
  user_id: CharField(255, indexed)       # 用户ID
  message: JSONField                     # 消息
  reference: JSONField(default=[])       # 引用
  tokens: IntegerField(default=0)        # Token消耗
  source: CharField(16, indexed)         # 来源：none|agent|dialog
  dsl: JSONField(default={})             # DSL配置
  duration: FloatField(indexed)          # 请求时长
  round: IntegerField(indexed)           # 轮次
  thumb_up: IntegerField(indexed)        # 点赞数
  errors: TextField                      # 错误信息
}

索引:
- PRIMARY KEY: id
- INDEX: dialog_id, user_id, source
- INDEX: duration, round, thumb_up
```

#### 1.2.4 LLM模型管理

**LLMFactories (LLM厂商表)**
```python
{
  name: CharField(128, primary_key)      # 厂商名称
  logo: TextField                        # Logo Base64
  tags: CharField(255, indexed)          # 标签：LLM,Embedding,Image2Text,ASR
  rank: IntegerField(default=0)          # 排序
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: name
- INDEX: tags
```

**LLM (LLM模型表)**
```python
{
  llm_name: CharField(128, indexed)      # 模型名称
  model_type: CharField(128, indexed)    # 模型类型
  fid: CharField(128, indexed)           # 厂商ID
  max_tokens: IntegerField(default=0)    # 最大Token数
  tags: CharField(255, indexed)          # 标签
  is_tools: BooleanField(default=False)  # 是否支持工具
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: (fid, llm_name)
- INDEX: model_type, tags
```

**TenantLLM (租户LLM配置表)**
```python
{
  tenant_id: CharField(32, indexed)      # 外键 -> Tenant.id
  llm_factory: CharField(128, indexed)   # 厂商名称
  model_type: CharField(128, indexed)    # 模型类型
  llm_name: CharField(128, indexed)      # 模型名称
  api_key: TextField                     # API密钥
  api_base: CharField(255)               # API Base URL
  max_tokens: IntegerField(default=8192)
  used_tokens: IntegerField(default=0)
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: (tenant_id, llm_factory, llm_name)
- INDEX: model_type, max_tokens, used_tokens
```

#### 1.2.5 任务管理

**Task (任务表)**
```python
{
  id: CharField(32, primary_key)
  doc_id: CharField(32, indexed)         # 外键 -> Document.id
  from_page: IntegerField(default=0)     # 起始页
  to_page: IntegerField(default=100000000) # 结束页
  task_type: CharField(32)               # 任务类型
  priority: IntegerField(default=0)      # 优先级
  begin_at: DateTimeField(indexed)       # 开始时间
  process_duration: FloatField           # 处理时长
  progress: FloatField(indexed)          # 进度
  progress_msg: TextField                # 进度消息
  retry_count: IntegerField(default=0)   # 重试次数
  digest: TextField                      # 任务摘要
  chunk_ids: LongTextField               # 分块ID列表
  create_time: BigIntegerField
  update_time: BigIntegerField
}

索引:
- PRIMARY KEY: id
- INDEX: doc_id, progress, begin_at
```

#### 1.2.6 文件管理

**File (文件表)**
```python
{
  id: CharField(32, primary_key)
  parent_id: CharField(32, indexed)      # 父文件夹ID
  tenant_id: CharField(32, indexed)      # 租户ID
  created_by: CharField(32, indexed)     # 创建者
  name: CharField(255, indexed)          # 文件/文件夹名
  location: CharField(255)               # 存储位置
  size: IntegerField(indexed)            # 大小
  type: CharField(32, indexed)           # 类型
  source_type: CharField(128, indexed)   # 来源类型
}

索引:
- PRIMARY KEY: id
- INDEX: parent_id, tenant_id, created_by
- INDEX: name, type, source_type
```

**File2Document (文件-文档关联表)**
```python
{
  id: CharField(32, primary_key)
  file_id: CharField(32, indexed)        # 外键 -> File.id
  document_id: CharField(32, indexed)    # 外键 -> Document.id
}

索引:
- PRIMARY KEY: id
- INDEX: file_id, document_id
```

#### 1.2.7 Agent画布

**UserCanvas (用户画布表)**
```python
{
  id: CharField(32, primary_key)
  avatar: TextField
  user_id: CharField(255, indexed)       # 用户ID
  title: CharField(255)                  # 标题
  permission: CharField(16, indexed)     # 权限：me|team
  description: TextField                 # 描述
  canvas_type: CharField(32, indexed)    # 画布类型
  canvas_category: CharField(32, indexed)# 分类：agent_canvas|dataflow_canvas
  dsl: JSONField(default={})             # DSL配置
}

索引:
- PRIMARY KEY: id
- INDEX: user_id, canvas_type, canvas_category
```

**UserCanvasVersion (画布版本表)**
```python
{
  id: CharField(32, primary_key)
  user_canvas_id: CharField(255, indexed)# 外键 -> UserCanvas.id
  title: CharField(255)
  description: TextField
  dsl: JSONField(default={})
}

索引:
- PRIMARY KEY: id
- INDEX: user_canvas_id
```

#### 1.2.8 数据源连接器

**Connector (连接器表)**
```python
{
  id: CharField(32, primary_key)
  tenant_id: CharField(32, indexed)
  name: CharField(128)
  source: CharField(128, indexed)        # 数据源：S3,SharePoint等
  input_type: CharField(128, indexed)    # 输入类型：poll|event
  config: JSONField(default={})          # 连接配置
  refresh_freq: IntegerField(default=0)  # 刷新频率(秒)
  prune_freq: IntegerField(default=0)    # 清理频率(秒)
  timeout_secs: IntegerField(default=3600)
  indexing_start: DateTimeField(indexed)
  status: CharField(16, indexed)         # 状态：schedule等
}

索引:
- PRIMARY KEY: id
- INDEX: tenant_id, source, input_type, status
```

**Connector2Kb (连接器-知识库关联)**
```python
{
  id: CharField(32, primary_key)
  connector_id: CharField(32, indexed)   # 外键 -> Connector.id
  kb_id: CharField(32, indexed)          # 外键 -> Knowledgebase.id
  auto_parse: CharField(1, default='1')  # 是否自动解析
}

索引:
- PRIMARY KEY: id
- INDEX: connector_id, kb_id
```

**SyncLogs (同步日志表)**
```python
{
  id: CharField(32, primary_key)
  connector_id: CharField(32, indexed)   # 外键 -> Connector.id
  kb_id: CharField(32, indexed)
  status: CharField(128, indexed)        # 状态
  from_beginning: CharField(1)
  new_docs_indexed: IntegerField
  total_docs_indexed: IntegerField
  docs_removed_from_index: IntegerField
  error_msg: TextField
  error_count: IntegerField
  full_exception_trace: TextField
  time_started: DateTimeField(indexed)
  poll_range_start: DateTimeTzField
  poll_range_end: DateTimeTzField
}

索引:
- PRIMARY KEY: id
- INDEX: connector_id, kb_id, status, time_started
```

#### 1.2.9 评估系统

**EvaluationDataset (评估数据集)**
```python
{
  id: CharField(32, primary_key)
  tenant_id: CharField(32, indexed)
  name: CharField(255, indexed)
  description: TextField
  kb_ids: JSONField                      # 关联知识库ID列表
  created_by: CharField(32, indexed)
  create_time: BigIntegerField(indexed)
  update_time: BigIntegerField
  status: IntegerField(default=1)
}

索引:
- PRIMARY KEY: id
- INDEX: tenant_id, name, created_by, create_time
```

**EvaluationCase (评估用例)**
```python
{
  id: CharField(32, primary_key)
  dataset_id: CharField(32, indexed)     # 外键 -> EvaluationDataset.id
  question: TextField                    # 测试问题
  reference_answer: TextField            # 参考答案
  relevant_doc_ids: JSONField            # 相关文档ID
  relevant_chunk_ids: JSONField          # 相关分块ID
  metadata: JSONField                    # 元数据
  create_time: BigIntegerField
}

索引:
- PRIMARY KEY: id
- INDEX: dataset_id
```

**EvaluationRun (评估运行)**
```python
{
  id: CharField(32, primary_key)
  dataset_id: CharField(32, indexed)
  dialog_id: CharField(32, indexed)
  name: CharField(255)
  config_snapshot: JSONField             # 配置快照
  metrics_summary: JSONField             # 指标汇总
  status: CharField(32)                  # PENDING|RUNNING|COMPLETED|FAILED
  created_by: CharField(32, indexed)
  create_time: BigIntegerField(indexed)
  complete_time: BigIntegerField
}

索引:
- PRIMARY KEY: id
- INDEX: dataset_id, dialog_id, created_by, create_time
```

**EvaluationResult (评估结果)**
```python
{
  id: CharField(32, primary_key)
  run_id: CharField(32, indexed)         # 外键 -> EvaluationRun.id
  case_id: CharField(32, indexed)        # 外键 -> EvaluationCase.id
  generated_answer: TextField            # 生成的答案
  retrieved_chunks: JSONField            # 检索到的分块
  metrics: JSONField                     # 指标
  execution_time: FloatField             # 执行时间
  token_usage: JSONField                 # Token使用量
  create_time: BigIntegerField
}

索引:
- PRIMARY KEY: id
- INDEX: run_id, case_id
```

#### 1.2.10 其他模型

**APIToken (API令牌表)**
```python
{
  tenant_id: CharField(32, indexed)
  token: CharField(255, indexed)
  dialog_id: CharField(32, indexed)
  source: CharField(16, indexed)         # none|agent|dialog
  beta: CharField(255, indexed)
}

索引:
- PRIMARY KEY: (tenant_id, token)
- INDEX: dialog_id, source, beta
```

**Search (搜索配置表)**
```python
{
  id: CharField(32, primary_key)
  avatar: TextField
  tenant_id: CharField(32, indexed)
  name: CharField(128, indexed)
  description: TextField
  created_by: CharField(32, indexed)
  search_config: JSONField               # 搜索配置
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: id
- INDEX: tenant_id, name, created_by
```

**Memory (记忆表)**
```python
{
  id: CharField(32, primary_key)
  name: CharField(128)
  avatar: TextField
  tenant_id: CharField(32, indexed)
  memory_type: IntegerField(indexed)     # 1=raw,2=semantic,4=episodic,8=procedural
  storage_type: CharField(32, indexed)   # table|graph
  embd_id: CharField(128)
  llm_id: CharField(128)
  permissions: CharField(16, indexed)
  description: TextField
  memory_size: IntegerField(default=5242880)
  forgetting_policy: CharField(32)       # lru|fifo
  temperature: FloatField(default=0.5)
  system_prompt: TextField
  user_prompt: TextField
}

索引:
- PRIMARY KEY: id
- INDEX: tenant_id, memory_type, storage_type
```

**MCPServer (MCP服务器表)**
```python
{
  id: CharField(32, primary_key)
  name: CharField(255)
  tenant_id: CharField(32, indexed)
  url: CharField(2048)
  server_type: CharField(32)
  description: TextField
  variables: JSONField(default={})
  headers: JSONField(default={})
}

索引:
- PRIMARY KEY: id
- INDEX: tenant_id
```

**PipelineOperationLog (Pipeline操作日志)**
```python
{
  id: CharField(32, primary_key)
  document_id: CharField(32, indexed)
  tenant_id: CharField(32, indexed)
  kb_id: CharField(32, indexed)
  pipeline_id: CharField(32, indexed)
  pipeline_title: CharField(32, indexed)
  parser_id: CharField(32, indexed)
  document_name: CharField(255)
  document_suffix: CharField(255)
  document_type: CharField(255)
  source_from: CharField(255)
  progress: FloatField(indexed)
  progress_msg: TextField
  process_begin_at: DateTimeField(indexed)
  process_duration: FloatField
  dsl: JSONField(default={})
  task_type: CharField(32)
  operation_status: CharField(32)
  avatar: TextField
  status: CharField(1, default='1')
}

索引:
- PRIMARY KEY: id
- INDEX: document_id, tenant_id, kb_id, pipeline_id
- INDEX: progress, process_begin_at
```

### 1.3 数据关系图

```
User 1──N UserTenant N──1 Tenant
                              │
                              ├──1───N─→ Knowledgebase ──1───N─→ Document
                              │                  │
                              ├──1───N─→ Dialog  │
                              │         │        │
                              │         1        1
                              │         │        │
                              │         N        N
                              │         │        │
                              └──1───N─→ API4Conversation
                                        Conversation

Tenant ──1───N─→ TenantLLM ──N───1─→ LLMFactories
                                      │
                                      └──1───N─→ LLM

Document ──1───N─→ Task
         ──N───1─→ File (via File2Document)

Connector ──N───M─→ Knowledgebase (via Connector2Kb)
          ──1───N─→ SyncLogs

EvaluationDataset ──1───N─→ EvaluationCase
                  ──1───N─→ EvaluationRun ──1───N─→ EvaluationResult

UserCanvas ──1───N─→ UserCanvasVersion
```

---

## 2. 向量数据库设计

RAGFlow 支持多种向量数据库实现，默认使用 **Infinity**，也支持 **OceanBase Vector** 扩展。

### 2.1 Infinity 向量存储方案

**配置**: `conf/service_conf.yaml`

```yaml
infinity:
  uri: 'localhost:23817'     # Infinity 服务地址
  db_name: 'default_db'      # 数据库名称
```

**向量存储特性**:
- **向量维度**: 根据嵌入模型动态配置（常见: 768/1024/1536）
- **索引类型**: HNSW（Hierarchical Navigable Small World）
- **距离度量**: 内积（Inner Product）/ 余弦相似度（Cosine）
- **存储结构**: 
  - 每个知识库对应一个 Collection
  - Collection 名称格式: `{tenant_id}_{kb_id}`

**核心字段**:
```python
{
  "doc_id": "STRING",           # 文档ID
  "chunk_id": "STRING",         # 分块ID
  "content": "VARCHAR",         # 文本内容
  "q_{embd_id}_vec": "VECTOR",  # 向量字段（动态命名）
  "available_int": "INT",       # 可用标记
}
```

**索引定义**:
```python
Index(
  name=f"q_{embd_id}_vec_idx",
  index_type=IndexType.Hnsw,
  params={
    "M": 16,                    # HNSW图连接数
    "ef_construction": 200,     # 构建时的候选数
    "metric": "ip"              # 内积距离
  }
)
```

### 2.2 OceanBase Vector 方案

**配置**: `conf/service_conf.yaml`

```yaml
oceanbase:
  scheme: 'oceanbase'
  config:
    db_name: 'test'
    user: 'root@ragflow'
    password: 'infini_rag_flow'
    host: 'localhost'
    port: 2881
```

**向量存储特性**:
- 基于 OceanBase 4.4+ 的向量扩展
- 支持 IVF_FLAT 索引
- 支持 L2 和余弦距离
- 最大向量维度: 65535

### 2.3 向量检索流程

```
1. 用户查询
   ↓
2. 查询向量化（Embedding Model）
   ↓
3. 向量相似度检索（Top K）
   └→ Infinity/OceanBase: KNN Search
   └→ 使用 HNSW 或 IVF_FLAT 索引
   ↓
4. 返回候选文档块 ID 列表
   ↓
5. 从 Elasticsearch 获取完整文档内容
```

### 2.4 性能优化

1. **索引优化**
   - HNSW 参数调优: M=16, ef_construction=200
   - 定期索引重建（Compact）

2. **批量操作**
   - 批量插入向量: 每批 100-1000 条
   - 使用异步插入提升吞吐量

3. **缓存策略**
   - 热门查询向量缓存（Redis）
   - 结果缓存（TTL: 300s）

---

## 3. Elasticsearch/OpenSearch 数据设计

### 3.1 连接配置

**Elasticsearch 配置**: `conf/service_conf.yaml`

```yaml
es:
  hosts: 'http://localhost:1200'
  username: 'elastic'
  password: 'infini_rag_flow'
```

**OpenSearch 配置**:
```yaml
os:
  hosts: 'http://localhost:1201'
  username: 'admin'
  password: 'infini_rag_flow_OS_01'
```

### 3.2 索引设计

**索引命名规则**: `{tenant_id}`
- 每个租户对应一个索引
- 索引下所有知识库共享

**Mapping 定义** (`conf/mapping.json`):

```json
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "max_result_window": 100000,
    "analysis": {
      "analyzer": {
        "ik_smart": {...},
        "ik_max_word": {...}
      }
    }
  },
  "mappings": {
    "properties": {
      "id": {"type": "keyword"},
      "doc_id": {"type": "keyword"},
      "kb_id": {"type": "keyword"},
      "content_with_weight": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "content_ltks": {"type": "text"},
      "content_sm_ltks": {"type": "text"},
      "important_kwd": {"type": "keyword"},
      "important_tks": {"type": "text"},
      "title_tks": {"type": "text"},
      "pagerank_fea": {"type": "float"},
      "tag_fea": {"type": "keyword"},
      "available_int": {"type": "integer"},
      "create_time": {"type": "date"},
      "create_timestamp_flt": {"type": "float"}
    }
  }
}
```

### 3.3 核心字段说明

| 字段名 | 类型 | 说明 |
|-------|------|------|
| `id` | keyword | 分块唯一ID |
| `doc_id` | keyword | 文档ID（关联MySQL） |
| `kb_id` | keyword | 知识库ID（关联MySQL） |
| `content_with_weight` | text | 带权重的内容（用于检索） |
| `content_ltks` | text | 长文本内容 |
| `important_kwd` | keyword | 重要关键词 |
| `pagerank_fea` | float | PageRank特征 |
| `tag_fea` | keyword | 标签特征 |
| `available_int` | integer | 可用标记（0=已删除,1=可用） |
| `create_time` | date | 创建时间 |
| `create_timestamp_flt` | float | 创建时间戳（浮点） |

### 3.4 检索策略

1. **全文检索**（BM25）
   ```python
   query = {
     "bool": {
       "must": [
         {"match": {"content_with_weight": user_query}},
         {"term": {"kb_id": kb_id}},
         {"term": {"available_int": 1}}
       ]
     }
   }
   ```

2. **关键词检索**
   ```python
   query = {
     "bool": {
       "should": [
         {"match": {"important_kwd": keywords}},
         {"match": {"important_tks": keywords}}
       ]
     }
   }
   ```

3. **混合检索**（向量 + 全文）
   - 向量检索获取 Top K1（如 1024）
   - 全文检索获取 Top K2（如 1024）
   - 取交集或加权融合
   - 重排序（Rerank）获取最终 Top N（如 6）

### 3.5 性能优化

1. **索引优化**
   - 分片数: 1（单租户场景）
   - 副本数: 0（非高可用场景）
   - Refresh Interval: 30s（降低实时性换取性能）

2. **查询优化**
   - 使用 `max_result_window` 扩大结果窗口
   - 避免深度分页（使用 `search_after`）
   - 使用 Filter Context 替代 Query Context

3. **缓存优化**
   - Query Cache: 热门查询结果缓存
   - Field Data Cache: 聚合字段缓存

---

## 4. 图数据库设计（GraphRAG）

### 4.1 NetworkX 内存图存储

RAGFlow 使用 **NetworkX** 作为内存图数据库，支持 GraphRAG（Graph-based RAG）功能，用于构建和查询知识图谱。

**实现方式**：
- 纯 Python 实现的图数据结构
- 内存存储，支持序列化到文件系统
- 支持有向图和无向图
- 提供丰富的图算法（社区发现、PageRank等）

### 4.2 GraphRAG 数据模型

**节点类型（Entities）**：
```python
{
  "entity_id": "uuid",           # 实体ID
  "entity_name": "实体名称",     # 实体名称
  "entity_type": "类型",         # 实体类型（人物、地点、组织等）
  "description": "描述",         # 实体描述
  "source_id": "doc_id",        # 来源文档ID
  "chunk_ids": ["chunk1", ...], # 关联的文档块
  "importance": 0.8,             # 重要性分数
  "embedding": [...],            # 实体嵌入向量（可选）
}
```

**边类型（Relations）**：
```python
{
  "source": "entity_id_1",      # 源实体
  "target": "entity_id_2",      # 目标实体
  "relation_type": "关系类型",  # 关系类型（属于、位于、创建等）
  "description": "关系描述",    # 关系描述
  "weight": 0.9,                 # 关系权重
  "source_chunk": "chunk_id",   # 来源文档块
}
```

### 4.3 GraphRAG 实现方式

RAGFlow 支持两种 GraphRAG 方法：

#### 4.3.1 Light GraphRAG
- **来源**：[HKUDS/LightRAG](https://github.com/HKUDS/LightRAG)
- **特点**：轻量级实现，快速构建
- **存储**：内存图 + Elasticsearch/Infinity

#### 4.3.2 General GraphRAG（Microsoft）
- **来源**：[microsoft/graphrag](https://github.com/microsoft/graphrag)
- **特点**：
  - 层次化社区检测（Hierarchical Communities）
  - 使用 Leiden 算法进行社区发现
  - 生成社区报告（Community Reports）
- **存储**：内存图 + MySQL + Elasticsearch

### 4.4 图构建流程

```
1. 文档解析
   ↓
2. 实体抽取（Entity Extraction）
   → 使用 LLM 识别文档中的实体
   → 实体类型：人物、地点、组织、事件等
   ↓
3. 关系抽取（Relation Extraction）
   → 使用 LLM 识别实体间的关系
   → 关系类型：属于、位于、创建、影响等
   ↓
4. 实体消解（Entity Resolution）
   → 合并同名或相似实体
   → 基于字符串匹配或向量相似度
   ↓
5. 图构建
   → 添加节点（实体）到 NetworkX 图
   → 添加边（关系）到 NetworkX 图
   ↓
6. 社区检测（可选，General 模式）
   → 使用 Leiden 算法
   → 生成层次化社区结构
   ↓
7. 持久化
   → 图序列化到 JSON（存储到 MySQL 或文件系统）
   → 实体/关系存储到 Elasticsearch（用于检索）
```

### 4.5 图查询和检索

**查询类型**：
1. **实体查询**：根据实体名称或类型查找
2. **关系查询**：查找两个实体间的关系路径
3. **子图查询**：提取包含特定实体的子图
4. **社区查询**：查询属于同一社区的实体

**检索流程**：
```
1. 用户查询 → 提取查询中的实体
   ↓
2. 实体匹配 → 在图中查找匹配的实体节点
   ↓
3. 子图提取 → 提取相关实体的 k-hop 邻居
   ↓
4. 上下文构建 → 基于子图构建 Prompt
   ↓
5. LLM 生成 → 结合子图上下文生成答案
```

### 4.6 存储位置

**数据库字段**（Knowledgebase 表）：
```python
graphrag_task_id: CharField(32)         # GraphRAG 任务ID
graphrag_task_finish_at: DateTimeField  # 完成时间
```

**文件存储**：
- 图序列化文件：`{tenant_id}/{kb_id}/graph.json`
- 社区报告：`{tenant_id}/{kb_id}/communities.json`
- 实体嵌入：存储在 Infinity 向量数据库

### 4.7 性能优化

1. **增量更新**：
   - 文档更新时只重建受影响的子图
   - 使用 `GraphChange` 跟踪变更

2. **缓存策略**：
   - 热门实体查询结果缓存（Redis）
   - 社区结构缓存

3. **分布式处理**：
   - 大规模图分片处理
   - 并行社区检测

### 4.8 图算法支持

使用 NetworkX 提供的图算法：
- **PageRank**：计算实体重要性
- **Leiden**：社区检测
- **最短路径**：查找实体间关系路径
- **度中心性**：识别关键实体
- **betweenness_centrality**：识别桥接实体

---

## 5. Redis 数据设计

### 4.1 连接配置

```yaml
redis:
  db: 1
  username: ''
  password: 'infini_rag_flow'
  host: 'localhost:6379'
```

**连接池配置**:
- 使用 Valkey（Redis 兼容）客户端
- 连接池大小: 10
- 连接超时: 5s
- Socket 超时: 30s

### 4.2 缓存 Key 规范

| Key 模式 | 用途 | TTL |
|---------|------|-----|
| `user:{user_id}` | 用户信息缓存 | 3600s |
| `tenant:{tenant_id}` | 租户信息缓存 | 3600s |
| `kb:{kb_id}` | 知识库配置缓存 | 1800s |
| `dialog:{dialog_id}` | 对话配置缓存 | 1800s |
| `doc:{doc_id}` | 文档元数据缓存 | 600s |
| `llm:{tenant_id}:{llm_id}` | LLM配置缓存 | 3600s |
| `search_result:{hash}` | 搜索结果缓存 | 300s |

### 4.3 任务队列

**队列命名规则**: `rag_flow_svr_queue_{priority}`
- Priority 范围: 0-9（0最高优先级）

**队列实现**: Redis Streams
```python
# 消息结构
{
  "message": {
    "task_id": "xxx",
    "doc_id": "xxx",
    "task_type": "parse",
    "from_page": 0,
    "to_page": 100,
    ...
  }
}

# 消费者组
consumer_group = "task_executor_group"
consumer_name = f"{hostname}_{pid}_{timestamp}"
```

**任务流程**:
```
1. 任务生产（API Layer）
   → XADD rag_flow_svr_queue_{priority}
   
2. 任务消费（Task Executor）
   → XREADGROUP GROUP task_executor_group {consumer_name}
   
3. 任务处理
   → 更新进度到 MySQL
   
4. 任务确认
   → XACK rag_flow_svr_queue_{priority}
```

### 4.4 分布式锁

**Lua 脚本实现**:
```lua
-- 删除锁（CAS）
local current_value = redis.call('get', KEYS[1])
if current_value and current_value == ARGV[1] then
    redis.call('del', KEYS[1])
    return 1
end
return 0
```

**锁命名规则**: `lock:{resource_name}`
- 锁超时: 60s
- 自动续期机制

**使用场景**:
- 数据库表初始化: `lock:init_database_tables`
- 任务执行器清理: `lock:clean_task_executor`
- 知识库索引重建: `lock:rebuild_index:{kb_id}`

### 4.5 会话管理

**Session Key**: `session:{session_id}`
- 存储用户会话状态
- TTL: 1800s（30分钟）
- 支持滑动过期

### 4.6 监控数据

| Key | 类型 | 用途 |
|-----|------|------|
| `TASKEXE` | Set | 活跃任务执行器列表 |
| `{consumer_name}` | Sorted Set | 任务执行器心跳（score=timestamp） |
| `task_stats:{date}` | Hash | 任务统计数据 |

---

## 6. 对象存储设计

### 5.1 MinIO 配置

```yaml
minio:
  user: 'rag_flow'
  password: 'infini_rag_flow'
  host: 'localhost:9000'
  bucket: ''                  # 默认bucket名称
  prefix_path: ''             # 对象key前缀
```

### 5.2 Bucket 规划

| Bucket名称 | 用途 | 生命周期策略 |
|-----------|------|-------------|
| `{tenant_id}` | 租户文件存储 | 永久保留 |
| `{tenant_id}-temp` | 临时文件 | 7天后自动删除 |
| `ragflow-assets` | 系统资源（Logo等） | 永久保留 |

### 5.3 对象 Key 规范

**文档原始文件**:
```
{tenant_id}/{kb_id}/{doc_id}/{filename}
```

**文档解析结果**:
```
{tenant_id}/{kb_id}/{doc_id}/parsed/{chunk_id}.json
```

**图片资源**:
```
{tenant_id}/{kb_id}/{doc_id}/images/{image_id}.{ext}
```

**用户头像**:
```
{tenant_id}/avatars/{user_id}.{ext}
```

### 5.4 访问策略

1. **预签名 URL**
   - 上传: 有效期 3600s
   - 下载: 有效期 600s
   - 使用 HMAC-SHA256 签名

2. **访问控制**
   - Bucket 策略: 私有（Private）
   - 通过应用层鉴权控制访问
   - 租户间数据隔离

### 5.5 性能优化

1. **并发控制**
   - 最大并发上传/下载: 10（环境变量 `MAX_CONCURRENT_MINIO`）
   - 使用 Semaphore 限流

2. **分片上传**
   - 大文件（>100MB）使用分片上传
   - 分片大小: 5MB

3. **CDN 加速**（可选）
   - 静态资源通过 CDN 分发
   - 降低 MinIO 带宽压力

---

**GraphRAG 图数据**:
```
{tenant_id}/{kb_id}/graphrag/graph.json
{tenant_id}/{kb_id}/graphrag/communities.json
{tenant_id}/{kb_id}/graphrag/entity_embeddings.npy
```

---

## 7. 数据一致性保证

### 6.1 事务处理

**关系型数据库事务**:
- 使用 Peewee ORM 的 `@DB.connection_context()` 装饰器
- 支持嵌套事务（Savepoint）
- 自动回滚机制

```python
@DB.connection_context()
def create_document_with_task(doc_data, task_data):
    with DB.atomic() as transaction:
        doc = Document.create(**doc_data)
        task = Task.create(doc_id=doc.id, **task_data)
        return doc, task
```

### 6.2 级联删除策略

**知识库删除**:
```
1. 标记 Knowledgebase.status = '0'
2. 标记所有关联 Document.status = '0'
3. 标记所有关联 Task.status = '0'
4. 从 Elasticsearch 删除所有 chunks (available_int = 0)
5. 从 Infinity 删除向量索引
6. 保留 MinIO 文件（软删除）
```

**文档删除**:
```
1. 标记 Document.status = '0'
2. 从 Elasticsearch 删除 chunks (available_int = 0)
3. 从 Infinity 删除向量
4. 保留 MinIO 文件（软删除）
```

### 6.3 数据备份策略

1. **MySQL/PostgreSQL**
   - 每日全量备份（凌晨 2:00）
   - 每小时增量备份（Binlog/WAL）
   - 备份保留: 7天

2. **Elasticsearch**
   - 快照备份（Snapshot）: 每日一次
   - 保留策略: 最近7个快照

3. **MinIO**
   - 版本控制: 启用对象版本
   - 跨区域复制（可选）

4. **Redis**
   - RDB 快照: 每小时一次
   - AOF 日志: 每秒同步

---

## 8. 性能优化策略

### 7.1 主数据库优化

1. **索引策略**
   - 所有外键字段建立索引
   - 高频查询字段（如 `status`, `create_time`）建立索引
   - 联合索引: `(tenant_id, status, create_time)`

2. **查询优化**
   - 避免 N+1 查询（使用 JOIN 或预加载）
   - 使用 `select_related()` 和 `prefetch_related()`
   - 分页查询使用 Cursor 而非 Offset

3. **连接池管理**
   - 最大连接数: 900
   - 空闲连接超时: 300s
   - 定期清理过期连接（30s 间隔）

4. **读写分离**（可选）
   - 主库: 写操作
   - 从库: 读操作
   - 延迟监控: <1s

### 7.2 向量数据库优化

1. **索引参数调优**
   - HNSW M=16: 平衡精度和速度
   - ef_construction=200: 构建时搜索范围
   - ef_search=100: 查询时搜索范围

2. **批量操作**
   - 向量插入: 每批 500-1000 条
   - 使用异步插入（asyncio）

3. **定期维护**
   - Compact 索引: 每周一次
   - 删除标记清理: 每日一次

### 7.3 Elasticsearch 优化

1. **索引优化**
   - Refresh Interval: 30s（降低实时性）
   - 禁用副本: `number_of_replicas: 0`
   - 合并策略: `max_merged_segment: 5gb`

2. **查询优化**
   - 使用 Filter Context 而非 Query Context
   - 避免深度分页（使用 `search_after`）
   - 启用查询缓存

3. **硬件优化**
   - 内存: 堆内存设置为物理内存 50%
   - 磁盘: 使用 SSD
   - CPU: 多核并行

### 8.4 图数据库优化

1. **内存管理**
   - 大图分片加载
   - LRU缓存机制
   - 定期图压缩

2. **查询优化**
   - 子图查询限制深度（max k-hop）
   - 索引关键实体
   - 预计算常用路径

3. **增量更新**
   - 局部图更新
   - 避免全图重建

### 8.5 缓存优化

1. **多级缓存**
   - L1: 应用内存缓存（LRU, 1000条）
   - L2: Redis 缓存（TTL: 300-3600s）
   - L3: 数据库

2. **缓存预热**
   - 系统启动时加载热点数据
   - 定时刷新缓存（每5分钟）

3. **缓存失效策略**
   - 主动失效: 数据更新时删除缓存
   - 被动失效: TTL 过期
   - 缓存雪崩防护: 随机TTL

---

## 9. 监控与维护

### 8.1 监控指标

**数据库监控**:
- 连接数: 当前/最大
- QPS: 每秒查询数
- 慢查询: >1s 的查询
- 锁等待: 等待时间 >5s

**Elasticsearch 监控**:
- 集群状态: Green/Yellow/Red
- 索引大小: GB
- 查询延迟: P95 < 100ms
- 磁盘使用率: <80%

**Redis 监控**:
- 内存使用率: <80%
- 缓存命中率: >90%
- 队列堆积: <1000

**MinIO 监控**:
- 存储使用率: <80%
- 带宽: 上传/下载速率
- 请求延迟: P95 < 500ms

### 8.2 告警规则

| 指标 | 阈值 | 级别 |
|------|------|------|
| 数据库连接数 | >800 | Warning |
| 慢查询数量 | >100/min | Warning |
| ES 集群状态 | Yellow | Warning |
| ES 集群状态 | Red | Critical |
| Redis 内存 | >90% | Critical |
| 任务队列堆积 | >5000 | Warning |
| MinIO 磁盘 | >90% | Critical |

### 8.3 日志系统

**日志级别**:
- DEBUG: 详细调试信息
- INFO: 关键操作日志
- WARNING: 警告信息
- ERROR: 错误日志
- CRITICAL: 严重错误

**日志存储**:
- 应用日志: `/var/log/ragflow/`
- 数据库日志: `/var/log/mysql/` 或 `/var/log/postgresql/`
- Nginx 日志: `/var/log/nginx/`

**日志轮转**:
- 按天切割: `logrotate`
- 保留策略: 30天
- 压缩: gzip

---

**NetworkX 监控**:
- 图节点数: < 1,000,000
- 图边数: < 5,000,000
- 内存使用: < 2GB
- 查询延迟: P95 < 200ms

---

## 10. 数据迁移与版本管理

### 9.1 数据迁移策略

**Schema 迁移**:
使用 Peewee Migrate 或 Playhouse Migrate:

```python
def migrate_db():
    migrator = MySQLMigrator(DB) 或 PostgresqlMigrator(DB)
    
    # 添加字段
    migrate(
        migrator.add_column(
            'knowledgebase', 
            'pipeline_id', 
            CharField(max_length=32, null=True, indexed=True)
        )
    )
    
    # 修改字段类型
    migrate(
        migrator.alter_column_type(
            'tenant_llm', 
            'api_key', 
            TextField(null=True)
        )
    )
```

**数据迁移脚本**: `docker/migration.sh`
- 自动检测版本差异
- 逐步应用迁移
- 回滚机制

### 9.2 Schema 版本控制

**版本号格式**: `YYYYMMDD_description`
- 示例: `20240115_add_pipeline_support`

**迁移文件位置**:
- `api/db/migrations/`

**执行流程**:
```bash
# 1. 备份数据库
mysqldump -u root -p rag_flow > backup_$(date +%Y%m%d).sql

# 2. 执行迁移
bash docker/migration.sh

# 3. 验证迁移
python -c "from api.db import init_database_tables; init_database_tables()"
```

---

## 11. 安全性设计

### 10.1 数据加密

**传输加密**:
- MySQL/PostgreSQL: SSL/TLS 连接（可选）
- Elasticsearch: HTTPS（可选）
- Redis: TLS（可选）
- MinIO: HTTPS

**存储加密**:
- 敏感字段（密码、API Key）: AES-256 加密
- 存储前加密: `serialize_b64()` / `deserialize_b64()`
- MinIO 对象: 服务端加密（SSE）

### 10.2 访问控制

**数据库访问**:
- 用户名/密码认证
- IP 白名单（可选）
- 最小权限原则（Least Privilege）

**API 访问**:
- JWT Token 认证
- API Key 认证
- 租户隔离: 所有查询强制加 `tenant_id` 过滤

**对象存储访问**:
- 预签名 URL（时效性）
- Bucket 策略: 私有
- 应用层鉴权

### 10.3 数据隔离

**租户隔离**:
- MySQL: 通过 `tenant_id` 字段隔离
- Elasticsearch: 每租户一个索引
- Infinity: 每租户一个 Collection
- MinIO: 每租户一个 Bucket

**软删除**:
- 不物理删除数据，只标记 `status='0'`
- 查询时过滤 `status='1'`
- 定期清理（可选）

---

## 12. 扩展性设计

### 11.1 水平扩展

**应用层**:
- 无状态设计
- 负载均衡: Nginx / HAProxy
- 会话共享: Redis

**数据库层**:
- MySQL: 主从复制 + 读写分离
- Elasticsearch: 多节点集群
- Redis: 主从复制 + Sentinel
- MinIO: 分布式部署

### 11.2 垂直扩展

**硬件升级**:
- CPU: 增加核心数
- 内存: 增加容量（建议 32GB+）
- 磁盘: 升级为 NVMe SSD

**配置优化**:
- 数据库连接池: 增加最大连接数
- Elasticsearch 堆内存: 增加至物理内存 50%
- Redis 最大内存: 增加至物理内存 70%

---

## 附录

### A. 环境变量配置

| 环境变量 | 默认值 | 说明 |
|---------|-------|------|
| `DATABASE_TYPE` | mysql | 数据库类型：mysql/postgres |
| `MAX_CONCURRENT_MINIO` | 10 | MinIO 最大并发数 |
| `ENABLE_TIMEOUT_ASSERTION` | false | 是否启用超时断言 |
| `RAGFLOW_LOG_LEVEL` | INFO | 日志级别 |

### B. 数据量级参考

**小型部署** (< 100 用户):
- 知识库: < 100
- 文档: < 10,000
- 文档块: < 1,000,000
- 向量维度: 768
- 存储: < 100GB

**中型部署** (100-1000 用户):
- 知识库: < 1,000
- 文档: < 100,000
- 文档块: < 10,000,000
- 向量维度: 1024
- 存储: < 1TB

**大型部署** (> 1000 用户):
- 知识库: > 1,000
- 文档: > 100,000
- 文档块: > 10,000,000
- 向量维度: 1536
- 存储: > 1TB

### C. 相关文档

- [RAGFlow 架构设计](../02-architecture/architecture-overview.md)
- [API 文档](../04-core-modules/api-layer.md)
- [部署指南](../../docker/README.md)
- [性能调优指南](./performance-tuning.md)（待补充）

---

**文档版本**: v1.0  
**更新日期**: 2024-12-19  
**维护者**: RAGFlow Team
