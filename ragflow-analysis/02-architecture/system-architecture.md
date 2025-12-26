# RAGFlow 系统架构文档

## 1. 架构概述

### 1.1 架构模式

RAGFlow 采用**现代化微服务架构**，融合了以下多种架构模式：

- **前后端分离架构**: React SPA (单页应用) + Python RESTful API
- **分层架构**: 展示层、API层、业务逻辑层、数据持久层清晰分离
- **微服务架构**: 多个独立可扩展的服务组件
- **插件架构**: 核心功能 + 可扩展插件系统
- **事件驱动架构**: 基于 Redis 的异步任务处理
- **容器化部署**: Docker + Kubernetes 云原生架构

### 1.2 设计原则

- **模块化**: 各功能模块高内聚、低耦合
- **可扩展性**: 支持水平扩展和垂直扩展
- **高可用性**: 无状态服务设计，支持多实例部署
- **异步处理**: 长耗时任务异步化，提升响应速度
- **服务隔离**: 核心服务与辅助服务分离
- **数据分层**: 热数据缓存，冷数据归档

### 1.3 技术架构图

参考 [system-component-architecture.puml](./system-component-architecture.puml) 查看完整的组件架构图。

---

## 2. 系统分层架构

### 2.1 架构层次

```
┌─────────────────────────────────────────────────────────┐
│                      客户端层                            │
│   Web Browser / Mobile App / Third-party Integration   │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                      展示层                              │
│      React SPA (Ant Design + UmiJS + Tailwind CSS)     │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                      网关层                              │
│         Nginx (反向代理 + 负载均衡 + SSL)               │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                      API 服务层                          │
│   Quart API Server (Blueprint 模块化 + Auth 中间件)     │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   业务逻辑层                             │
│   RAG Engine / Agent System / Document Parser          │
│   GraphRAG / Plugin Manager / MCP Server               │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   数据访问层                             │
│  Peewee(业务) + SQLAlchemy(向量) + Search + Storage   │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   基础设施层                             │
│   MySQL / ES/OpenSearch / Redis / MinIO / Infinity     │
└─────────────────────────────────────────────────────────┘
```

### 2.2 各层职责

#### 客户端层
- **职责**: 用户交互入口
- **支持**: Web浏览器、移动应用、第三方集成（SDK、Chrome扩展等）

#### 展示层 (Presentation Layer)
- **位置**: `web/`
- **技术栈**: React 18 + TypeScript + UmiJS
- **职责**:
  - 用户界面渲染
  - 用户交互处理
  - 状态管理 (React Query + Zustand)
  - 前端路由 (UmiJS Router)
  - 数据可视化 (AntV G2/G6)
- **主要页面**:
  - 知识库管理
  - 对话聊天
  - Agent 工作流编辑器
  - 文档查看器
  - 系统管理

#### 网关层 (Gateway Layer)
- **位置**: `docker/nginx/`
- **技术**: Nginx
- **职责**:
  - 反向代理
  - 负载均衡
  - SSL/TLS 终止
  - 静态资源服务
  - 请求路由
  - 压缩和缓存
- **端口配置**:
  - HTTP: 80
  - HTTPS: 443
  - API: 9380

#### API 服务层 (API Service Layer)
- **位置**: `api/`
- **技术**: Quart (异步 Flask) + Blueprint
- **职责**:
  - RESTful API 提供
  - 请求验证和参数校验
  - 身份认证和授权
  - Session 管理
  - 请求路由分发
  - 错误处理和日志记录
  - API 文档 (Swagger/Flasgger)
- **核心 Blueprints**:
  - `kb_app.py`: 知识库管理 API
  - `document_app.py`: 文档管理 API
  - `conversation_app.py`: 对话管理 API
  - `dialog_app.py`: 对话交互 API
  - `canvas_app.py`: Agent 画布 API
  - `llm_app.py`: LLM 配置 API
  - `user_app.py`: 用户管理 API
  - `system_app.py`: 系统管理 API

#### 业务逻辑层 (Business Logic Layer)
- **位置**: `rag/`, `agent/`, `deepdoc/`, `graphrag/`, `plugin/`
- **职责**: 核心业务逻辑实现
- **主要组件**:
  1. **RAG 引擎** (`rag/`)
     - 文档分块 (Chunking)
     - 向量嵌入 (Embedding)
     - 相似度检索 (Retrieval)
     - 上下文组装 (Context Assembly)
     - LLM 调用和响应生成
  
  2. **Agent 系统** (`agent/`, `agentic_reasoning/`)
     - Agent 工作流编排
     - 工具调用 (Tool Calling)
     - 思维链推理 (Chain-of-Thought)
     - 深度研究 (Deep Research)
  
  3. **文档解析器** (`deepdoc/`)
     - 多格式文档解析
     - OCR 文字识别
     - 版面分析
     - 表格提取
  
  4. **图RAG** (`graphrag/`)
     - 实体抽取
     - 知识图谱构建
     - 图查询分析
  
  5. **插件管理器** (`plugin/`)
     - 插件加载和卸载
     - 插件生命周期管理
     - LLM 工具插件

#### 数据访问层 (Data Access Layer)
- **位置**: `api/db/`, `rag/utils/`, `common/`
- **职责**:
  - 数据库操作抽象
  - 搜索引擎客户端
  - 对象存储客户端
  - 缓存访问
  - 连接池管理
- **主要组件**:
  - **应用层ORM** (Peewee - `api/db/db_models.py`)
    - 用于业务数据模型：User, Tenant, Knowledgebase, Document, Dialog等
    - 轻量级、简单易用，适合CRUD密集操作
  - **向量层ORM** (SQLAlchemy - `rag/utils/ob_conn.py`)
    - 专用于OceanBase向量数据库操作
    - 定义向量表结构（Column定义）
    - 执行复杂SQL（全文搜索、向量相似度计算）
  - **Service Layer** (`api/db/services/`)
  - **Search Client** (Elasticsearch/OpenSearch/Infinity)
  - **Storage Client** (MinIO/S3)
  - **Cache Client** (Redis)

#### 基础设施层 (Infrastructure Layer)
- **职责**: 提供底层存储和计算资源
- **组件**:
  - **关系数据库**: MySQL / PostgreSQL / OceanBase
  - **搜索引擎**: Elasticsearch / OpenSearch / Infinity
  - **向量数据库**: Infinity / Elasticsearch with Vector
  - **缓存**: Redis
  - **对象存储**: MinIO (S3兼容)
  - **消息队列**: Redis (作为任务队列)

---

## 3. 核心组件详解

### 3.1 前端应用 (Web Application)

#### 组件结构
```
web/
├── src/
│   ├── pages/           # 页面组件
│   │   ├── knowledge/   # 知识库管理
│   │   ├── chat/        # 对话界面
│   │   ├── flow/        # Agent 工作流
│   │   └── settings/    # 设置页面
│   ├── components/      # 可复用组件
│   ├── services/        # API 服务封装
│   ├── hooks/          # 自定义 Hooks
│   ├── utils/          # 工具函数
│   ├── locales/        # 国际化
│   └── models/         # 数据模型
```

#### 技术选型
- **UI框架**: React 18 (函数式组件 + Hooks)
- **应用框架**: UmiJS (约定式路由 + 插件体系)
- **组件库**: Ant Design 5.x + Radix UI
- **样式方案**: Tailwind CSS + CSS Modules
- **状态管理**: 
  - React Query (服务端状态)
  - Zustand (客户端状态)
- **图表**: AntV G2 (图表) + AntV G6 (图可视化)
- **流程图**: XYFlow React
- **表格**: Tanstack Table
- **编辑器**: Monaco Editor (代码编辑) + Lexical (富文本)

#### 数据流
1. 用户操作 → UI 组件
2. 触发 Service 方法 → API 请求
3. React Query 管理请求状态
4. 响应数据 → 更新组件状态
5. 组件重新渲染

### 3.2 API 服务器 (API Server)

#### 架构设计
- **框架**: Quart (异步 Flask 替代品)
- **模式**: Blueprint 蓝图模式
- **认证**: Quart-Auth + Flask-Login
- **文档**: Flasgger (Swagger/OpenAPI)

#### 核心 Blueprints
| Blueprint | 文件 | 职责 |
|-----------|------|------|
| 知识库 API | `kb_app.py` | 知识库 CRUD、配置管理 |
| 文档 API | `document_app.py` | 文档上传、解析、删除 |
| 对话 API | `conversation_app.py` | 对话会话管理 |
| 对话交互 API | `dialog_app.py` | 消息发送、流式响应 |
| Agent 画布 API | `canvas_app.py` | 工作流编排、节点管理 |
| 分块 API | `chunk_app.py` | 文档分块查看、编辑 |
| LLM API | `llm_app.py` | LLM 配置、模型列表 |
| 用户 API | `user_app.py` | 用户注册、登录、管理 |
| 系统 API | `system_app.py` | 系统配置、健康检查 |
| 搜索 API | `search_app.py` | 全局搜索 |
| 连接器 API | `connector_app.py` | 第三方数据源集成 |
| MCP 服务器 API | `mcp_server_app.py` | MCP 协议支持 |

#### 中间件链
```
请求
  ↓
CORS 中间件 (quart-cors)
  ↓
认证中间件 (quart-auth)
  ↓
Session 中间件 (flask-session)
  ↓
请求日志中间件
  ↓
Blueprint 路由
  ↓
业务逻辑处理
  ↓
响应
```

### 3.3 RAG 核心引擎

#### 组件结构
```
rag/
├── app/              # RAG 应用逻辑
├── flow/             # 工作流引擎
├── llm/              # LLM 抽象层
│   ├── chat_model.py       # 聊天模型
│   ├── embedding_model.py  # 嵌入模型
│   ├── rerank_model.py     # 重排序模型
│   └── ...
├── nlp/              # NLP 工具
├── prompts/          # 提示词模板
├── svr/              # RAG 服务
└── utils/            # RAG 工具
```

#### RAG 工作流
```
文档上传
  ↓
文档解析 (deepdoc)
  ↓
文档分块 (Chunking)
  ↓
向量嵌入 (Embedding)
  ↓
索引存储 (ES/Infinity)
  ↓
用户查询
  ↓
查询理解和改写
  ↓
多路召回
  ├── 向量检索
  ├── 关键词检索
  └── 混合检索
  ↓
融合重排序 (Reranking)
  ↓
上下文组装
  ↓
提示词构建
  ↓
LLM 推理
  ↓
响应生成（流式/非流式）
  ↓
引用追溯
  ↓
返回用户
```

#### LLM 抽象层
- **设计模式**: 工厂模式 + 策略模式
- **支持的 LLM**:
  - OpenAI (GPT-4, GPT-3.5)
  - Anthropic (Claude 3)
  - Google (Gemini)
  - 阿里云通义千问
  - 百度文心一言
  - 智谱 AI
  - Mistral AI
  - Cohere
  - Ollama (本地)
  - Groq
  - Replicate
- **统一接口**:
  - `chat()`: 同步对话
  - `chat_streamly()`: 流式对话
  - `embed()`: 文本嵌入
  - `rerank()`: 结果重排序

### 3.4 文档解析系统 (DeepDoc)

#### 架构设计
```
deepdoc/
├── parser/           # 格式解析器
│   ├── pdf_parser.py       # PDF 解析
│   ├── docx_parser.py      # Word 解析
│   ├── pptx_parser.py      # PowerPoint 解析
│   ├── excel_parser.py     # Excel 解析
│   ├── html_parser.py      # HTML 解析
│   ├── markdown_parser.py  # Markdown 解析
│   └── ...
└── vision/           # 视觉识别
    ├── ocr.py              # OCR 引擎
    ├── layout_analysis.py  # 版面分析
    └── table_detection.py  # 表格识别
```

#### 解析流程
```
文档输入
  ↓
格式识别
  ↓
选择对应解析器
  ↓
版面分析
  ↓
内容提取
  ├── 文本
  ├── 图像
  ├── 表格
  └── 元数据
  ↓
OCR 识别 (如需要)
  ↓
结构化输出
  ↓
存储到对象存储
```

#### 支持格式
- **文档**: PDF, Word (.docx, .doc), PowerPoint, Excel
- **图像**: PNG, JPG, TIFF (with OCR)
- **标记语言**: HTML, Markdown, XML
- **邮件**: Outlook MSG
- **其他**: TXT, CSV, JSON

### 3.5 Agent 系统

#### 组件结构
```
agent/
├── canvas.py         # Agent 画布/工作流
├── settings.py       # Agent 配置
├── component/        # Agent 组件
│   ├── begin.py            # 开始节点
│   ├── answer.py           # 回答节点
│   ├── retrieval.py        # 检索节点
│   ├── generate.py         # 生成节点
│   ├── categorize.py       # 分类节点
│   └── ...
├── templates/        # Agent 模板
│   ├── qa_template.py      # 问答模板
│   ├── chat_template.py    # 对话模板
│   └── ...
└── tools/            # Agent 工具
    ├── search.py           # 搜索工具
    ├── calculator.py       # 计算器
    ├── code_executor.py    # 代码执行
    └── ...
```

#### Agent 工作流
```
用户输入
  ↓
开始节点 (Begin)
  ↓
理解意图
  ↓
分类路由 (Categorize)
  ├─→ 知识检索路径
  │     ↓
  │   检索节点 (Retrieval)
  │     ↓
  │   生成答案 (Generate)
  │
  ├─→ 工具调用路径
  │     ↓
  │   工具选择
  │     ↓
  │   工具执行
  │     ↓
  │   结果处理
  │
  └─→ 直接回答路径
        ↓
      回答节点 (Answer)
  ↓
输出结果
```

#### 支持的工具
- **搜索工具**: DuckDuckGo、Google、百度
- **数据工具**: 问财搜索、AKShare 金融数据
- **学术工具**: arXiv、Scholarly
- **计算工具**: Python 代码执行器
- **网页工具**: 网页抓取 (Crawl4AI)
- **自定义工具**: 通过插件系统扩展

### 3.6 图RAG系统 (GraphRAG)

#### 组件结构
```
graphrag/
├── general/          # 通用图RAG
│   ├── entity_extraction.py    # 实体抽取
│   ├── graph_builder.py        # 图构建
│   └── graph_query.py          # 图查询
├── light/            # 轻量级图RAG
└── entity_resolution.py  # 实体消歧
```

#### 工作流程
```
文档输入
  ↓
实体抽取 (NER)
  ↓
实体消歧和链接
  ↓
关系抽取
  ↓
知识图谱构建
  ↓
社区检测
  ↓
图索引
  ↓
用户查询
  ↓
查询分析
  ↓
图遍历和检索
  ↓
子图提取
  ↓
上下文增强
  ↓
LLM 推理
  ↓
响应生成
```

### 3.7 插件系统

#### 架构设计
- **插件管理器**: `plugin/plugin_manager.py`
- **插件接口**: `plugin/common.py`
- **LLM 工具插件**: `plugin/llm_tool_plugin.py`
- **内置插件**: `plugin/embedded_plugins/`

#### 插件生命周期
```
插件注册
  ↓
插件加载
  ↓
插件初始化
  ↓
插件激活
  ↓
插件运行
  ↓
插件卸载
```

#### 插件类型
1. **LLM 工具插件**: 扩展 Agent 可用工具
2. **数据源插件**: 新数据源集成
3. **解析器插件**: 新文档格式支持
4. **模型插件**: 自定义 LLM/Embedding 模型

### 3.8 MCP 服务 (Model Context Protocol)

#### 组件结构
```
mcp/
├── client/           # MCP 客户端
│   └── client.py
└── server/           # MCP 服务端
    └── server.py
```

#### 功能
- 实现 MCP 协议标准
- 提供工具注册和调用
- 支持多种传输方式 (SSE, HTTP)
- 上下文管理和共享

---

## 4. 数据流和交互

### 4.1 典型用户场景：知识库问答

#### 完整数据流
```
用户在前端输入问题
  ↓
前端 (React) 
  ↓ HTTP POST /api/dialog/completion
Nginx 反向代理
  ↓
API Server (Quart)
  ↓ 认证验证
Session 检查
  ↓
Dialog Blueprint (dialog_app.py)
  ↓
业务逻辑层：对话服务
  ↓
RAG 引擎：查询理解
  ↓
检索服务
  ├─→ 向量检索 (Infinity/ES)
  ├─→ 关键词检索 (ES)
  └─→ 混合检索
  ↓
重排序 (Reranker)
  ↓
上下文组装
  ↓
LLM 服务
  ↓ API 调用
外部 LLM (OpenAI/Claude/...)
  ↓
流式响应
  ↓
SSE 推送给前端
  ↓
前端渲染响应
  ↓
展示给用户
```

### 4.2 文档上传和解析流程

```
用户上传文档
  ↓
前端文件选择
  ↓ HTTP POST /api/document/upload
API Server
  ↓
文件临时存储
  ↓
异步任务创建
  ↓ Redis 任务队列
后台任务处理器
  ↓
文档解析 (DeepDoc)
  ↓
文档分块 (Chunking)
  ↓
批量嵌入 (Embedding)
  ↓ 并发调用 Embedding 服务
向量生成
  ↓
索引构建
  ↓ 批量写入
搜索引擎 (ES/Infinity)
  ↓
元数据更新
  ↓ MySQL
数据库
  ↓
文件存储
  ↓ MinIO
对象存储
  ↓
任务完成通知
  ↓
前端轮询/WebSocket 更新
  ↓
显示解析结果
```

### 4.3 Agent 工作流执行

```
用户设计 Agent 流程图
  ↓
保存到数据库 (JSON 格式)
  ↓
用户触发执行
  ↓
Agent 引擎加载流程图
  ↓
从开始节点执行
  ↓
遍历节点图
  ↓ 按依赖顺序
节点逐个执行
  ├─→ 检索节点：调用 RAG
  ├─→ 生成节点：调用 LLM
  ├─→ 分类节点：条件判断
  ├─→ 工具节点：调用外部工具
  └─→ 回答节点：输出结果
  ↓
中间结果传递
  ↓
到达结束节点
  ↓
返回最终结果
```

---

## 5. 外部集成

### 5.1 LLM 服务提供商

#### 集成方式
- **统一抽象**: 通过 `rag/llm/` 模块统一接口
- **配置驱动**: 通过 `conf/llm_factories.json` 配置
- **动态加载**: 运行时根据配置选择 LLM

#### 已集成的 LLM
| 提供商 | 模型示例 | SDK |
|--------|---------|-----|
| OpenAI | GPT-4, GPT-3.5-Turbo | openai |
| Anthropic | Claude 3 Opus/Sonnet | anthropic |
| Google | Gemini Pro | google-genai |
| 阿里云 | 通义千问 | dashscope |
| 百度 | 文心一言 | qianfan |
| 智谱 | GLM-4 | (API) |
| Mistral | Mistral Large | mistralai |
| Cohere | Command | cohere |
| Ollama | Llama 3, Mistral | ollama |
| Groq | Mixtral, Llama | groq |
| Replicate | 各种开源模型 | replicate |

### 5.2 向量数据库和搜索引擎

#### Elasticsearch / OpenSearch
- **用途**: 全文检索 + 向量检索
- **集成**: `elasticsearch-dsl`, `opensearch-py`
- **功能**:
  - 倒排索引搜索
  - KNN 向量搜索
  - 混合检索
  - 聚合分析

#### Infinity
- **用途**: 专用向量数据库
- **集成**: `infinity-sdk`, `infinity-emb`
- **功能**:
  - 高性能向量检索
  - 本地 Embedding 服务
  - 批量嵌入加速

### 5.3 对象存储

#### MinIO
- **用途**: 文件和媒体存储
- **集成**: `minio` SDK
- **存储内容**:
  - 原始文档
  - 解析后的图像
  - 系统备份
- **兼容性**: S3 API 兼容

### 5.4 第三方数据源

#### 云存储和协作平台
| 平台 | 集成 SDK | 用途 |
|------|---------|------|
| Confluence | atlassian-python-api | 知识库同步 |
| Notion | Office365-REST-Python-Client | 笔记同步 |
| Google Drive | google-auth-oauthlib | 文件同步 |
| Slack | slack-sdk | 消息同步 |
| Discord | discord-py | 服务器消息 |
| Box | boxsdk | 云存储 |
| Dropbox | dropbox | 云存储 |
| Jira | jira | 问题跟踪 |
| Moodle | moodlepy | 在线学习 |

#### 网页抓取
- **Crawl4AI**: AI 驱动的智能爬虫
- **Selenium Wire**: 浏览器自动化
- **Firecrawl**: 深度网页抓取

### 5.5 监控和追踪

#### Langfuse
- **用途**: LLM 应用追踪和监控
- **集成**: `langfuse` SDK
- **功能**:
  - LLM 调用追踪
  - Token 使用统计
  - 性能分析
  - 成本跟踪
  - 提示词版本管理

---

## 6. 部署架构

### 6.1 Docker 部署架构

#### 服务拓扑
```
┌─────────────────────────────────────────┐
│           Docker Network (ragflow)       │
│                                          │
│  ┌──────────┐         ┌──────────┐      │
│  │  Nginx   │ ←────→ │ RAGFlow  │      │
│  │  :80     │         │  API     │      │
│  │  :443    │         │  :9380   │      │
│  └──────────┘         └──────────┘      │
│       ↑                    ↓             │
│       │              ┌──────────┐        │
│       │              │  MySQL   │        │
│       │              │  :3306   │        │
│       │              └──────────┘        │
│       │              ┌──────────┐        │
│       │              │   ES/    │        │
│       │              │ OpenSrch │        │
│       │              │  :9200   │        │
│       │              └──────────┘        │
│       │              ┌──────────┐        │
│       │              │  Redis   │        │
│       │              │  :6379   │        │
│       │              └──────────┘        │
│       │              ┌──────────┐        │
│       │              │  MinIO   │        │
│       └──────────────│  :9000   │        │
│                      └──────────┘        │
└─────────────────────────────────────────┘
```

#### 容器列表
| 容器名 | 镜像 | 端口 | 职责 |
|--------|------|------|------|
| nginx | nginx:alpine | 80, 443 | 反向代理、静态资源 |
| ragflow-api | infiniflow/ragflow | 9380 | API 服务器 |
| ragflow-task | infiniflow/ragflow | - | 后台任务处理 |
| mysql | mysql:8.0 | 3306 | 关系数据库 |
| elasticsearch | elastic/elasticsearch | 9200 | 搜索引擎 |
| redis | redis:7-alpine | 6379 | 缓存和队列 |
| minio | minio/minio | 9000, 9001 | 对象存储 |
| infinity | (可选) | 7997 | 向量数据库 |

#### 数据持久化
```
Docker Volumes:
  - mysql_data       → /var/lib/mysql
  - es_data          → /usr/share/elasticsearch/data
  - redis_data       → /data
  - minio_data       → /data
  - ragflow_logs     → /ragflow/logs
```

### 6.2 Kubernetes 部署架构

#### 资源组织
```
Namespace: ragflow
  ├── Deployments
  │   ├── ragflow-api (replicas: 3)
  │   ├── ragflow-task (replicas: 2)
  │   └── nginx (replicas: 2)
  ├── StatefulSets
  │   ├── mysql
  │   ├── elasticsearch
  │   └── redis
  ├── Services
  │   ├── ragflow-api-svc (ClusterIP)
  │   ├── mysql-svc (ClusterIP)
  │   ├── es-svc (ClusterIP)
  │   └── nginx-svc (LoadBalancer)
  ├── ConfigMaps
  │   ├── service-conf
  │   ├── nginx-conf
  │   └── llm-factories
  ├── Secrets
  │   ├── mysql-credentials
  │   ├── llm-api-keys
  │   └── minio-credentials
  └── PersistentVolumeClaims
      ├── mysql-pvc
      ├── es-pvc
      └── minio-pvc
```

#### 服务发现
```
内部服务:
  mysql.ragflow.svc.cluster.local:3306
  elasticsearch.ragflow.svc.cluster.local:9200
  redis.ragflow.svc.cluster.local:6379
  minio.ragflow.svc.cluster.local:9000

外部访问:
  https://ragflow.example.com → nginx-svc (LoadBalancer)
```

### 6.3 高可用部署

#### 多实例配置
```
API 服务器: 3+ 实例
  - 无状态设计
  - Nginx 负载均衡
  - 会话存储在 Redis

任务处理器: 2+ 实例
  - Redis 任务队列
  - 任务锁机制
  - 避免重复处理

数据库: 主从复制
  - 主节点：写入
  - 从节点：读取 (读写分离)
  - 自动故障转移

搜索引擎: 集群部署
  - 3+ 节点
  - 分片和副本
  - 数据冗余

Redis: 哨兵模式
  - 主从复制
  - 自动故障转移
  - 哨兵监控
```

#### 负载均衡策略
- **Nginx**: Round Robin / Least Connections
- **K8s Service**: Round Robin
- **数据库**: 读写分离
- **搜索引擎**: 客户端负载均衡

---

## 7. 数据流和状态管理

### 7.1 数据存储策略

#### 数据分类
| 数据类型 | 存储方式 | 访问频率 | 一致性要求 |
|---------|---------|---------|-----------|
| 用户账户 | MySQL | 中 | 强一致性 |
| 知识库元数据 | MySQL | 高 | 强一致性 |
| 文档元数据 | MySQL | 高 | 强一致性 |
| 对话历史 | MySQL | 高 | 最终一致性 |
| 文档分块 | ES/Infinity | 极高 | 最终一致性 |
| 向量嵌入 | Infinity/ES | 极高 | 最终一致性 |
| 原始文件 | MinIO | 低 | 最终一致性 |
| Session | Redis | 极高 | 强一致性 |
| 任务队列 | Redis | 高 | 强一致性 |
| 缓存 | Redis | 极高 | 弱一致性 |

#### 缓存策略
```
三级缓存:
  L1: 内存缓存 (应用内)
    - LLM 配置
    - 系统配置
    - TTL: 5分钟

  L2: Redis 缓存
    - 用户 Session
    - API 响应
    - 热点数据
    - TTL: 1小时

  L3: CDN 缓存
    - 静态资源
    - 公共文档
    - TTL: 24小时
```

### 7.2 异步任务处理

#### 任务类型
1. **文档解析任务**: 上传后异步解析
2. **索引构建任务**: 批量创建向量索引
3. **数据同步任务**: 第三方数据源同步
4. **定时任务**: 清理过期数据、统计分析
5. **通知任务**: 邮件、Webhook 通知

#### 任务队列架构
```
API Server
  ↓ 创建任务
Redis List (任务队列)
  ↓ 消费任务
Task Worker (多进程)
  ↓ 执行任务
任务处理逻辑
  ↓ 更新状态
MySQL (任务状态表)
  ↓ 完成通知
WebSocket / 轮询
  ↓
前端更新
```

#### 任务调度
- **调度器**: Redis + Python `threading`
- **优先级**: 高/中/低 三级队列
- **重试机制**: 指数退避
- **超时控制**: 任务超时自动取消
- **并发控制**: 进程池 + 协程

### 7.3 实时通信

#### 流式响应 (SSE)
```
客户端
  ↓ HTTP GET /api/dialog/stream
API Server
  ↓ 建立 SSE 连接
保持连接
  ↓ 逐块生成
LLM 流式输出
  ↓ SSE 事件
data: {"chunk": "..."}
  ↓
客户端接收并渲染
```

#### WebSocket (可选)
- **用途**: 实时通知、任务状态更新
- **框架**: Quart WebSocket 支持
- **心跳**: 定时 Ping/Pong

---

## 8. 安全架构

### 8.1 认证和授权

#### 认证机制
- **Session-based**: Flask-Session + Redis
- **Cookie**: HTTP-only, Secure, SameSite
- **Token**: (可选) JWT for API

#### 授权模型
- **RBAC**: 角色基础访问控制
  - 管理员 (Admin)
  - 普通用户 (User)
  - 访客 (Guest)
- **资源权限**: 知识库、文档、对话按用户隔离

### 8.2 数据安全

#### 传输安全
- **HTTPS**: TLS 1.3
- **证书**: Let's Encrypt / 自签名
- **HSTS**: 强制 HTTPS

#### 存储安全
- **密码**: bcrypt 哈希
- **敏感数据**: 加密存储 (pycryptodomex)
- **API Keys**: 环境变量 + Secret 管理

#### 输入验证
- **参数验证**: ajv (前端), marshmallow (后端)
- **XSS 防护**: DOMPurify (前端)
- **SQL 注入**: 双重ORM防护（Peewee + SQLAlchemy参数化查询）
- **CSRF**: Token 验证

### 8.3 API 安全

#### 速率限制
- **限流**: Redis 计数器
- **按用户**: 不同级别不同限额
- **按 IP**: 防止滥用

#### CORS 策略
- **允许源**: 配置化白名单
- **凭证**: 允许 Credentials
- **预检缓存**: 减少 OPTIONS 请求

---

## 9. 可观测性

### 9.1 日志系统

#### 日志级别
- **DEBUG**: 开发调试
- **INFO**: 正常运行信息
- **WARNING**: 警告信息
- **ERROR**: 错误信息
- **CRITICAL**: 严重错误

#### 日志格式
```json
{
  "timestamp": "2025-12-18T10:30:45.123Z",
  "level": "INFO",
  "logger": "api.apps.dialog_app",
  "message": "Dialog completion request",
  "user_id": "user_123",
  "trace_id": "abc-def-ghi",
  "duration_ms": 1234
}
```

#### 日志收集
- **本地**: 文件日志轮转
- **集中式**: ELK Stack (可选)
  - Filebeat → Logstash → Elasticsearch → Kibana

### 9.2 监控指标

#### 系统指标
- CPU 使用率
- 内存使用率
- 磁盘 I/O
- 网络流量

#### 应用指标
- API 请求量 (QPS)
- 响应时间 (P50, P95, P99)
- 错误率
- 并发连接数

#### 业务指标
- 用户活跃度
- 知识库数量
- 文档解析量
- LLM Token 消耗
- 向量检索 QPS

#### 监控方案
- **Prometheus**: 指标采集
- **Grafana**: 可视化看板
- **Langfuse**: LLM 追踪

### 9.3 分布式追踪

#### Trace ID
- 请求入口生成唯一 Trace ID
- 传递给所有下游服务
- 日志中记录 Trace ID

#### 链路追踪
```
HTTP Request [trace_id: xxx]
  ↓
API Gateway
  ↓
Dialog Service [trace_id: xxx]
  ↓
RAG Engine [trace_id: xxx]
  ├─→ ES Query [trace_id: xxx]
  └─→ LLM Call [trace_id: xxx]
  ↓
Response
```

---

## 10. 性能优化

### 10.1 查询优化

#### 向量检索优化
- **索引优化**: HNSW, IVF 索引
- **批量查询**: 合并多个检索请求
- **缓存**: 热点查询结果缓存
- **分片**: 大规模数据分片存储

#### 数据库优化
- **索引**: 频繁查询字段添加索引
- **连接池**: 复用数据库连接
- **读写分离**: 主从架构
- **分页**: 避免大结果集

### 10.2 计算优化

#### 异步处理
- **文档解析**: 异步任务队列
- **批量嵌入**: 并发调用 Embedding API
- **I/O 密集**: asyncio 异步 I/O

#### 并发控制
- **进程池**: CPU 密集任务
- **协程**: I/O 密集任务
- **限流**: 避免资源耗尽

### 10.3 网络优化

#### CDN
- 静态资源 CDN 加速
- 图片压缩和懒加载
- HTTP/2 和 gRPC

#### 缓存
- **浏览器缓存**: Cache-Control
- **反向代理缓存**: Nginx 缓存
- **应用缓存**: Redis 缓存

---

## 11. 扩展性设计

### 11.1 水平扩展

#### 无状态服务
- API Server: 多实例 + 负载均衡
- Task Worker: 多进程处理任务
- Session: 存储在 Redis，非本地

#### 数据库扩展
- **分库分表**: 按租户/时间分片
- **读写分离**: 主从复制
- **缓存**: Redis 集群

#### 搜索引擎扩展
- **分片**: 数据水平分片
- **副本**: 提高可用性和读吞吐
- **集群**: 多节点集群

### 11.2 功能扩展

#### 插件系统
- 新增 LLM 提供商: 实现 LLM 接口
- 新增解析器: 实现 Parser 接口
- 新增 Agent 工具: 注册到工具库

#### 模块化设计
- 低耦合: 模块间通过接口通信
- 高内聚: 模块内功能完整
- 可替换: 接口统一，实现可替换

---

## 12. 架构演进

### 12.1 当前架构 (v0.22.1)
- ✅ 前后端分离
- ✅ 微服务化初步
- ✅ 容器化部署
- ✅ 插件系统
- ✅ Agent 支持
- ✅ MCP 协议

### 12.2 未来演进方向

#### 短期 (3-6个月)
- [ ] 微服务进一步拆分（文档服务、对话服务、Agent 服务独立）
- [ ] gRPC 内部通信
- [ ] 消息队列升级 (Kafka/RabbitMQ)
- [ ] 服务网格 (Service Mesh)

#### 中期 (6-12个月)
- [ ] 多租户 SaaS 架构
- [ ] 弹性伸缩 (Auto-scaling)
- [ ] 边缘计算支持
- [ ] 联邦学习支持

#### 长期 (12+个月)
- [ ] 全球化部署
- [ ] 多区域容灾
- [ ] 实时协作
- [ ] AI 模型自动优化

---

## 13. 总结

### 13.1 架构优势
1. **模块化**: 清晰的分层和模块划分
2. **可扩展**: 支持水平和垂直扩展
3. **高性能**: 异步处理、缓存、并发优化
4. **可维护**: 代码组织良好，易于理解和修改
5. **云原生**: 容器化、微服务化、易于部署
6. **开放性**: 插件系统、MCP 协议支持

### 13.2 技术亮点
- **深度文档理解**: 业界领先的文档解析能力
- **多路召回**: 向量+关键词+混合检索
- **Agent 编排**: 可视化工作流设计
- **图RAG**: 知识图谱增强检索
- **多模态**: 文本、图像、表格统一处理
- **插件生态**: 可扩展的工具和模型支持

---

**文档版本**: 1.0  
**生成日期**: 2025-12-18  
**对应项目版本**: RAGFlow v0.22.1  
**架构图**: 参考 [system-component-architecture.puml](./system-component-architecture.puml)
