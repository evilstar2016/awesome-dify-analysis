# RAGFlow 项目概览文档

## 1. 项目基本信息

### 1.1 基本介绍
- **项目名称**: RAGFlow
- **项目版本**: 0.22.1
- **项目描述**: 基于深度文档理解的开源 RAG（检索增强生成）引擎
- **开源协议**: Apache-2.0
- **仓库地址**: https://github.com/infiniflow/ragflow
- **在线演示**: https://demo.ragflow.io
- **Docker Hub**: https://hub.docker.com/r/infiniflow/ragflow

### 1.2 项目定位
RAGFlow 是一个领先的开源检索增强生成(RAG)引擎，融合了前沿的 RAG 技术与 Agent 能力，为大型语言模型(LLM)创建了卓越的上下文层。它提供了一个可适配任何规模企业的精简 RAG 工作流。通过融合的上下文引擎和预构建的 Agent 模板，RAGFlow 使开发者能够以卓越的效率和精确度将复杂数据转化为高保真、可用于生产的 AI 系统。

### 1.3 核心特性
- ✅ 基于深度文档理解的知识抽取
- ✅ 模板化文档分块策略
- ✅ 带有可追溯引用的回答生成
- ✅ 支持异构数据源（Word、PPT、Excel、PDF、图像、网页等）
- ✅ 自动化且简洁的 RAG 工作流
- ✅ 可配置的 LLM 和 Embedding 模型
- ✅ Agent 工作流和 MCP 支持
- ✅ 多模态文档处理
- ✅ 跨语言查询支持

### 1.4 技术栈

#### 后端技术栈
- **开发语言**: Python 3.12-3.14
- **Web 框架**: Flask/Quart (异步 Web 框架)
- **依赖管理**: uv (现代 Python 包管理器)
- **数据库**: MySQL, PostgreSQL, OceanBase
- **搜索引擎**: Elasticsearch, OpenSearch, Infinity
- **向量存储**: Infinity, Elasticsearch
- **对象存储**: MinIO (S3 兼容)
- **缓存**: Redis
- **任务队列**: Redis

#### 前端技术栈
- **开发语言**: TypeScript
- **UI 框架**: React 18
- **应用框架**: UmiJS
- **UI 组件库**: Ant Design 5.x, Radix UI
- **样式方案**: Tailwind CSS
- **状态管理**: React Query (@tanstack/react-query), Zustand
- **数据可视化**: AntV (G2, G6)
- **流程图**: XYFlow React
- **Markdown**: @uiw/react-markdown-preview

#### 部署与运维
- **容器化**: Docker, Docker Compose
- **编排工具**: Kubernetes (Helm Charts)
- **监控**: Langfuse 集成

---

## 2. 项目目录结构

### 2.1 整体目录树

```
ragflow/
├── api/                    # 后端 API 服务器
│   ├── apps/              # API 蓝图（知识库、对话等功能模块）
│   ├── db/                # 数据库模型和服务（Peewee ORM）
│   ├── common/            # API 共享工具
│   └── utils/             # API 工具函数
├── rag/                    # RAG 核心逻辑
│   ├── llm/               # LLM、Embedding、Rerank 模型抽象
│   ├── app/               # RAG 应用逻辑
│   ├── flow/              # 工作流引擎
│   ├── nlp/               # 自然语言处理
│   ├── prompts/           # 提示词模板
│   ├── svr/               # RAG 服务器
│   └── utils/             # RAG 工具函数（含SQLAlchemy向量数据库连接）
├── deepdoc/               # 文档解析和 OCR 模块
│   ├── parser/            # 各类文档格式解析器
│   └── vision/            # 视觉和 OCR 组件
├── agent/                 # Agent 推理组件
│   ├── component/         # Agent 组件
│   ├── templates/         # Agent 模板
│   └── tools/             # Agent 工具集
├── agentic_reasoning/     # 深度推理能力
├── graphrag/              # 图RAG（知识图谱增强）
│   ├── general/           # 通用图RAG
│   └── light/             # 轻量级图RAG
├── common/                # 全局共享工具
│   └── data_source/       # 数据源集成
├── plugin/                # 插件系统
│   └── embedded_plugins/  # 内置插件
├── mcp/                   # Model Context Protocol
│   ├── client/            # MCP 客户端
│   └── server/            # MCP 服务端
├── web/                   # 前端 Web 应用
│   ├── src/               # 源代码
│   ├── public/            # 静态资源
│   └── .storybook/        # Storybook 配置
├── sdk/                   # Python SDK
├── test/                  # 后端测试
├── docker/                # Docker 部署配置
│   ├── nginx/             # Nginx 配置
│   └── oceanbase/         # OceanBase 配置
├── helm/                  # Kubernetes Helm Charts
├── docs/                  # 项目文档
├── example/               # 示例代码
│   ├── http/              # HTTP API 示例
│   └── sdk/               # SDK 使用示例
├── intergrations/         # 第三方集成
│   ├── chatgpt-on-wechat/ # 微信集成
│   ├── extension_chrome/  # Chrome 扩展
│   └── firecrawl/         # Firecrawl 集成
├── admin/                 # 管理工具
│   ├── client/            # 管理客户端
│   └── server/            # 管理服务器
├── conf/                  # 配置文件
├── sandbox/               # 沙箱环境
└── chat_demo/             # 聊天演示页面
```

### 2.2 核心目录说明

#### 后端核心目录

##### `api/` - API 服务器
- **作用**: 提供 RESTful API 接口，处理前端请求
- **框架**: Quart (异步 Flask 替代品)
- **架构模式**: Blueprint 蓝图模式，模块化 API 设计
- **主要功能**:
  - 用户认证和授权
  - 知识库管理 API
  - 对话管理 API
  - 文档上传和解析 API
  - Agent 工作流 API

##### `rag/` - RAG 核心引擎
- **作用**: 实现 RAG 的核心算法和逻辑
- **主要功能**:
  - 文档索引和分块
  - 向量 Embedding 生成
  - 相似度搜索和检索
  - LLM 集成和提示词管理
  - 多路召回和融合重排序
- **子模块**:
  - `llm/`: 多 LLM 提供商抽象 (OpenAI, Anthropic, Google, 通义千问等)
  - `flow/`: RAG 工作流编排
  - `nlp/`: NLP 工具集
  - `app/`: RAG 应用逻辑

##### `deepdoc/` - 文档深度理解
- **作用**: 实现各类文档格式的解析和理解
- **支持格式**:
  - PDF, Word, PowerPoint, Excel
  - 图像 (OCR)
  - HTML, Markdown
  - 结构化数据
- **核心能力**:
  - 版面分析
  - 表格提取
  - 图像识别
  - 扫描件 OCR

##### `agent/` 和 `agentic_reasoning/` - Agent 系统
- **作用**: 实现智能 Agent 和推理能力
- **功能**:
  - Agent 模板库
  - 工具调用机制
  - 思维链推理
  - 深度研究能力 (Deep Research)
  - MCP 协议支持

##### `graphrag/` - 图RAG
- **作用**: 基于知识图谱的增强检索
- **功能**:
  - 实体提取和解析
  - 关系图构建
  - 图查询分析
  - 社区检测

#### 前端核心目录

##### `web/` - 前端应用
- **技术栈**: React + TypeScript + UmiJS
- **主要页面**:
  - 知识库管理界面
  - 对话聊天界面
  - 文档查看器
  - Agent 工作流编辑器
  - 系统管理后台
- **目录结构**:
  ```
  web/
  ├── src/
  │   ├── components/    # 可复用组件
  │   ├── pages/         # 页面组件
  │   ├── services/      # API 服务
  │   ├── hooks/         # 自定义 Hooks
  │   ├── utils/         # 工具函数
  │   ├── locales/       # 国际化
  │   └── assets/        # 静态资源
  ├── public/            # 公共静态文件
  └── .storybook/        # Storybook 配置
  ```

#### 通用工具目录

##### `common/` - 共享工具库
- **作用**: 被所有模块共享的通用功能
- **包含**:
  - 配置管理 (`config_utils.py`)
  - 日志工具 (`log_utils.py`)
  - 文件处理 (`file_utils.py`)
  - Token 计算 (`token_utils.py`)
  - 加密工具 (`crypto_utils.py`)
  - HTTP 客户端 (`http_client.py`)
  - 异常定义 (`exceptions.py`)

##### `plugin/` - 插件系统
- **作用**: 提供可扩展的插件机制
- **功能**:
  - LLM 工具插件
  - 插件管理器
  - 内置插件集合

##### `mcp/` - Model Context Protocol
- **作用**: 实现 MCP 协议支持
- **包含**:
  - MCP 客户端实现
  - MCP 服务器实现

#### 部署与配置目录

##### `docker/` - Docker 部署
- **作用**: Docker 容器化部署配置
- **包含**:
  - `docker-compose.yml`: 完整服务编排
  - `docker-compose-base.yml`: 基础依赖服务
  - `Dockerfile`: 镜像构建文件
  - `entrypoint.sh`: 容器入口脚本
  - `launch_backend_service.sh`: 后端服务启动脚本
  - `nginx/`: Nginx 反向代理配置

##### `helm/` - Kubernetes 部署
- **作用**: K8s Helm Chart 配置
- **包含**:
  - Chart 定义
  - Values 配置模板

##### `conf/` - 配置文件
- **作用**: 服务配置和映射文件
- **包含**:
  - `service_conf.yaml`: 服务配置模板
  - `llm_factories.json`: LLM 工厂配置
  - `mapping.json`: ES 映射配置
  - `infinity_mapping.json`: Infinity 向量库映射

#### 其他目录

##### `sdk/` - Python SDK
- **作用**: 提供 Python SDK 供外部调用

##### `test/` - 测试代码
- **作用**: 后端单元测试和集成测试
- **框架**: pytest

##### `docs/` - 文档
- **作用**: 项目文档和使用指南
- **包含**:
  - 快速开始指南
  - 配置说明
  - API 参考
  - 开发指南

##### `example/` - 示例代码
- **作用**: API 使用示例和 SDK 示例

##### `intergrations/` - 第三方集成
- **作用**: 与外部系统的集成示例
- **包含**:
  - ChatGPT-on-WeChat 集成
  - Chrome 扩展
  - Firecrawl 集成

---

## 3. 主要功能模块

### 3.1 核心功能模块

#### 知识库管理 (Knowledge Base)
- **位置**: `api/apps/` (API 层), `rag/` (核心逻辑层)
- **功能**:
  - 知识库创建、编辑、删除
  - 文档上传和管理
  - 文档解析和分块
  - 向量索引构建
  - 知识检索优化

#### 文档解析 (Document Parsing)
- **位置**: `deepdoc/parser/`
- **支持格式**:
  - PDF (pdfplumber, pypdf)
  - Word (python-docx, mammoth)
  - PowerPoint (python-pptx, aspose-slides)
  - Excel (python-calamine, openpyxl)
  - 图像 (OpenCV, OCR)
  - HTML/Markdown
- **核心技术**:
  - 版面分析
  - 表格识别
  - 图像提取
  - OCR 文字识别
  - 多模态理解

#### RAG 检索引擎 (RAG Engine)
- **位置**: `rag/`
- **功能**:
  - 向量检索
  - 关键词检索
  - 混合检索
  - 多路召回
  - 融合重排序
  - 上下文组装
- **支持的检索引擎**:
  - Elasticsearch
  - OpenSearch
  - Infinity (向量数据库)

#### 对话管理 (Chat/Conversation)
- **位置**: `api/apps/`, `rag/app/`
- **功能**:
  - 多轮对话管理
  - 上下文维护
  - 流式响应
  - 引用追溯
  - 对话历史

#### LLM 集成 (LLM Integration)
- **位置**: `rag/llm/`
- **支持的 LLM 提供商**:
  - OpenAI (GPT-4, GPT-3.5)
  - Anthropic (Claude)
  - Google (Gemini)
  - 阿里云通义千问 (Qianfan)
  - 百度文心一言
  - 智谱 AI (GLM)
  - Mistral AI
  - Cohere
  - Ollama (本地部署)
  - Replicate
  - Groq

#### Embedding 模型
- **位置**: `rag/llm/`
- **支持**:
  - Infinity Embedding Service
  - OpenAI Embeddings
  - Google Embeddings
  - 本地 Embedding 模型 (onnxruntime)

### 3.2 高级功能模块

#### Agent 系统 (Agentic System)
- **位置**: `agent/`, `agentic_reasoning/`
- **功能**:
  - Agent 工作流编排
  - 工具调用 (Tool Calling)
  - 思维链推理
  - 深度研究 (Deep Research)
  - 代码执行器 (Python/JavaScript Sandbox)
  - Agent 模板库
- **核心组件**:
  - Canvas (工作流画布)
  - Component (可组合组件)
  - Templates (预置模板)
  - Tools (工具集)

#### 图RAG (GraphRAG)
- **位置**: `graphrag/`
- **功能**:
  - 实体抽取和识别
  - 实体消歧 (Entity Resolution)
  - 关系抽取
  - 知识图谱构建
  - 图查询分析
  - 社区检测
- **变体**:
  - General GraphRAG: 通用知识图谱
  - Light GraphRAG: 轻量级实现

#### Model Context Protocol (MCP)
- **位置**: `mcp/`
- **功能**:
  - MCP 服务器实现
  - MCP 客户端实现
  - 工具注册和调用
  - 上下文管理

#### 插件系统 (Plugin System)
- **位置**: `plugin/`
- **功能**:
  - 插件加载和管理
  - LLM 工具插件
  - 自定义插件开发
  - 插件市场 (embedded_plugins)

### 3.3 数据源集成

#### 本地文件上传
- 支持拖拽上传
- 批量上传
- 格式自动识别

#### 第三方数据源同步
- **位置**: `common/data_source/`, `api/apps/`
- **支持平台**:
  - Confluence (atlassian-python-api)
  - Notion (Office365-REST-Python-Client)
  - Google Drive (google-auth-oauthlib)
  - Discord (discord-py)
  - Slack (slack-sdk)
  - Jira (jira)
  - S3 (boto3, mypy-boto3-s3)
  - Box (boxsdk)
  - Dropbox (dropbox)
  - Moodle (moodlepy)

#### Web 抓取
- **依赖**: Crawl4AI, selenium-wire
- **功能**:
  - 网页内容抓取
  - 动态页面支持
  - 智能内容提取

### 3.4 前端功能模块

#### 知识库管理界面
- 知识库列表
- 文档管理
- 分块策略配置
- 索引状态监控

#### 对话聊天界面
- 多轮对话
- 流式输出
- 引用展示
- 历史记录
- Markdown 渲染

#### Agent 工作流编辑器
- 可视化流程编排
- 节点拖拽
- 连线配置
- 实时预览
- 调试工具

#### 文档查看器
- 多格式预览
- 分块可视化
- 高亮标注
- 引用定位

#### 系统管理
- 用户管理
- 权限配置
- 模型配置
- 系统监控

---

## 4. 核心依赖分析

### 4.1 后端核心依赖

#### Web 框架和服务器
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| quart-cors | 0.8.0 | CORS 支持 |
| quart-auth | 0.11.0 | 认证授权 |
| flask-cors | 6.0.2 | Flask CORS |
| flask-login | 0.6.3 | 用户登录管理 |
| flask-session | 0.8.0 | Session 管理 |
| flask-mail | >=0.10.0 | 邮件发送 |
| flasgger | >=0.9.7.1 | API 文档 (Swagger) |

#### LLM 和 AI 服务
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| anthropic | 0.34.1 | Anthropic Claude API |
| google-genai | >=1.41.0 | Google Gemini API |
| google-generativeai | >=0.8.1 | Google 生成式 AI |
| openai | (通过SDK调用) | OpenAI API |
| cohere | 5.6.2 | Cohere API |
| mistralai | 0.4.2 | Mistral AI |
| groq | 0.9.0 | Groq API |
| ollama | >=0.5.0 | Ollama 本地模型 |
| replicate | 0.31.0 | Replicate API |
| dashscope | 1.20.11 | 阿里云通义千问 |
| qianfan | 0.4.6 | 百度千帆 |

#### 向量数据库和搜索
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| elasticsearch-dsl | 8.12.0 | Elasticsearch DSL |
| opensearch-py | 2.7.1 | OpenSearch 客户端 |
| infinity-sdk | 0.6.11 | Infinity 向量库 SDK |
| infinity-emb | >=0.0.66 | Infinity Embedding 服务 |
| pyobvector | 0.2.18 | OceanBase 向量扩展 |

#### 文档解析和处理
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| pdfplumber | 0.10.4 | PDF 解析 |
| pypdf | 6.4.0 | PDF 处理 |
| pypdf2 | >=3.0.1 | PDF 工具 |
| python-docx | >=1.1.2 | Word 文档解析 |
| python-pptx | >=1.0.2 | PowerPoint 解析 |
| mammoth | >=1.11.0 | Word 转 HTML |
| python-calamine | >=0.4.0 | Excel 读取 (高性能) |
| extract-msg | >=0.39.0 | Outlook MSG 解析 |
| aspose-slides | 24.7.0 | PPT 高级处理 |
| reportlab | >=4.4.1 | PDF 生成 |

#### 图像和 OCR
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| opencv-python | 4.10.0.84 | 计算机视觉 |
| opencv-python-headless | 4.10.0.84 | 无GUI的OpenCV |
| onnxruntime | 1.23.2 | ONNX 模型推理 |
| onnxruntime-gpu | 1.23.2 | GPU 加速推理 |

#### 数据存储和ORM
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| peewee | (通过playhouse) | **应用层ORM** - 业务数据模型 (User, Knowledgebase等) |
| sqlalchemy | (间接依赖) | **向量层ORM** - OceanBase向量数据库操作 |
| psycopg2-binary | >=2.9.11 | PostgreSQL 驱动 |
| pyodbc | >=5.2.0 | ODBC 数据库连接 |
| minio | 7.2.4 | MinIO 对象存储 |

**🎯 双ORM架构说明**:
- **Peewee**: 用于 `api/db/` 中的业务数据模型（轻量、简洁，适合CRUD）
- **SQLAlchemy**: 用于 `rag/utils/ob_conn.py` 中的向量数据库操作（强大、灵活，支持复杂SQL和向量操作）
- **设计原因**: 职责分离 - 业务层使用简单ORM，向量层使用强大ORM

#### 第三方集成
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| atlassian-python-api | 4.0.7 | Confluence/Jira 集成 |
| Office365-REST-Python-Client | 2.6.2 | Office 365 集成 |
| google-auth-oauthlib | >=1.2.0 | Google OAuth |
| slack-sdk | 3.37.0 | Slack 集成 |
| discord-py | 2.3.2 | Discord 集成 |
| jira | 3.10.5 | Jira 集成 |
| boxsdk | >=10.1.0 | Box 云存储 |
| dropbox | 12.0.2 | Dropbox 集成 |
| moodlepy | >=0.23.0 | Moodle 集成 |

#### 数据处理和分析
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| cn2an | 0.5.22 | 中文数字转换 |
| editdistance | 0.8.1 | 编辑距离计算 |
| roman-numbers | 1.0.2 | 罗马数字处理 |
| ranx | 0.3.20 | 信息检索评估 |
| graspologic | (git) | 图算法库 |

#### 网络爬虫和搜索
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| Crawl4AI | >=0.4.0 | AI 驱动网页爬取 |
| selenium-wire | 5.1.0 | Web 自动化 |
| duckduckgo-search | >=7.2.0 | DuckDuckGo 搜索 |
| google-search-results | 2.4.2 | Google 搜索 API |
| scholarly | 1.7.11 | 学术搜索 |
| arxiv | 2.1.3 | arXiv 论文 API |
| pywencai | >=0.13.1 | 问财搜索 |
| akshare | >=1.15.78 | 金融数据 |

#### 工具和实用库
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| beartype | >=0.20.0 | 运行时类型检查 |
| pyclipper | >=1.4.0 | 多边形裁剪 |
| pycryptodomex | 3.20.0 | 加密库 |
| demjson3 | 3.0.6 | 容错 JSON 解析 |
| json-repair | 0.35.0 | JSON 修复 |
| ormsgpack | 1.5.0 | MessagePack 序列化 |
| ruamel-yaml | >=0.18.6 | YAML 处理 |
| markdown | 3.6 | Markdown 解析 |
| markdown-to-json | 2.1.1 | Markdown 转 JSON |
| markdownify | >=1.2.0 | HTML 转 Markdown |
| html-text | 0.6.2 | HTML 文本提取 |
| readability-lxml | >=0.8.4 | 网页正文提取 |

#### 其他核心依赖
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| mcp | >=1.19.0 | Model Context Protocol |
| langfuse | >=2.60.0 | LLM 追踪和监控 |
| pluginlib | 0.9.4 | 插件管理 |
| mini-racer | >=0.12.4 | JavaScript 运行时 |
| ffmpeg-python | >=0.2.0 | 音视频处理 |
| opendal | >=0.45.0 | 统一数据访问层 |
| pypandoc | >=1.16 | Pandoc 文档转换 |
| papaparse | (通过前端) | CSV 解析 |

### 4.2 前端核心依赖

#### 核心框架
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| react | 18.x | React 核心库 |
| umi | (UmiJS) | React 应用框架 |
| typescript | (通过UmiJS) | TypeScript 支持 |

#### UI 组件库
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| antd | ^5.12.7 | Ant Design UI 库 |
| @ant-design/pro-components | ^2.6.46 | Pro 组件 |
| @ant-design/pro-layout | ^7.17.16 | Pro 布局 |
| @ant-design/icons | ^5.2.6 | Ant Design 图标 |
| @radix-ui/react-* | (多个) | Radix UI 组件集 |

#### 状态管理和数据获取
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| @tanstack/react-query | ^5.40.0 | 服务端状态管理 |
| @tanstack/react-query-devtools | ^5.51.5 | React Query 调试 |
| zustand | (若使用) | 轻量状态管理 |
| immer | ^10.1.1 | 不可变数据 |

#### 数据可视化
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| @antv/g2 | ^5.2.10 | AntV 图表库 |
| @antv/g6 | ^5.0.10 | AntV 图可视化 |
| @xyflow/react | ^12.3.6 | 流程图和节点编辑器 |

#### 工具库
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| axios | ^1.12.0 | HTTP 客户端 |
| ahooks | ^3.7.10 | React Hooks 库 |
| dayjs | ^1.11.10 | 日期处理 |
| i18next | ^23.7.16 | 国际化 |
| i18next-browser-languagedetector | ^8.0.0 | 语言检测 |
| classnames | ^2.5.1 | 样式类名工具 |
| clsx | ^2.1.1 | 类名拼接 |
| class-variance-authority | ^0.7.0 | 样式变体管理 |

#### Markdown 和编辑器
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| @uiw/react-markdown-preview | ^5.1.3 | Markdown 预览 |
| @lexical/react | ^0.23.1 | 富文本编辑器 |
| @monaco-editor/react | ^4.6.0 | Monaco 代码编辑器 |

#### 表格和数据处理
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| @tanstack/react-table | ^8.20.5 | 表格组件 |
| @js-preview/excel | ^1.7.14 | Excel 预览 |
| papaparse | (类型定义) | CSV 解析 |

#### 样式和UI工具
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| tailwindcss | (配置) | Tailwind CSS |
| @tailwindcss/line-clamp | ^0.4.4 | 文本截断 |
| dompurify | ^3.1.6 | XSS 防护 |

#### 表单和验证
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| @hookform/resolvers | ^3.9.1 | React Hook Form 解析器 |
| ajv | ^8.17.1 | JSON Schema 验证 |
| ajv-formats | ^3.0.1 | AJV 格式验证 |

#### 其他工具
| 依赖包 | 版本 | 用途 |
|--------|------|------|
| human-id | ^4.1.1 | 人性化 ID 生成 |
| input-otp | ^1.4.1 | OTP 输入组件 |
| cmdk | ^1.0.4 | 命令面板 |
| eventsource-parser | ^1.1.2 | SSE 事件解析 |

---

## 5. 构建与部署

### 5.1 开发环境搭建

#### 前置要求
- **Python**: 3.12-3.14
- **Node.js**: 16.x+ (推荐 18.x+)
- **Docker**: 24.0.0+
- **Docker Compose**: v2.26.1+
- **uv**: 最新版本 (Python 包管理器)
- **gVisor**: (可选) 用于代码执行沙箱

#### 后端环境配置

```bash
# 1. 克隆仓库
git clone https://github.com/infiniflow/ragflow.git
cd ragflow

# 2. 安装 Python 依赖
uv sync --python 3.12 --all-extras

# 3. 下载额外依赖
uv run download_deps.py

# 4. 启动依赖服务 (MySQL, ES, Redis, MinIO)
docker compose -f docker/docker-compose-base.yml up -d

# 5. 配置环境变量
# 复制并编辑 docker/.env 文件

# 6. 运行数据库迁移
bash docker/migration.sh

# 7. 启动后端服务
bash docker/launch_backend_service.sh
```

#### 前端环境配置

```bash
# 1. 进入前端目录
cd web

# 2. 安装依赖
npm install

# 3. 启动开发服务器
npm run dev
```

### 5.2 构建命令

#### 后端构建
```bash
# Python 项目无需单独构建
# 通过 uv 管理依赖和环境

# 运行测试
uv run pytest

# 代码检查和格式化
ruff check .
ruff format .
```

#### 前端构建
```bash
cd web

# 开发模式
npm run dev          # 启动开发服务器 (端口 8000)
npm start            # 同上

# 生产构建
npm run build        # 构建生产版本

# 代码质量
npm run lint         # ESLint 检查
npm run test         # 运行测试

# Storybook
npm run storybook    # 启动 Storybook (端口 6006)
npm run build-storybook  # 构建 Storybook
```

### 5.3 Docker 部署

#### 使用 Docker Compose (推荐)

```bash
# 1. 进入 docker 目录
cd docker

# 2. 配置环境变量
cp .env.example .env
# 编辑 .env 文件

# 3. 启动所有服务
docker compose -f docker-compose.yml up -d

# 4. 查看日志
docker compose logs -f

# 5. 停止服务
docker compose down
```

**服务包括**:
- `ragflow-api`: API 服务器
- `ragflow-task`: 后台任务处理
- `mysql`: 数据库
- `elasticsearch` / `opensearch` / `infinity`: 搜索引擎
- `redis`: 缓存和队列
- `minio`: 对象存储
- `nginx`: 反向代理

#### 仅启动基础服务

```bash
# 启动 MySQL, ES, Redis, MinIO
docker compose -f docker/docker-compose-base.yml up -d
```

#### 构建自定义镜像

```bash
# 构建标准镜像
docker build -t ragflow:latest .

# 使用 TEI 加速
docker build -f Dockerfile_tei -t ragflow:tei .

# Scratch 构建 (OceanBase 9)
docker build -f Dockerfile.scratch.oc9 -t ragflow:oc9 .
```

### 5.4 Kubernetes 部署

#### 使用 Helm Charts

```bash
# 1. 添加 Helm 仓库 (如果有)
# helm repo add ragflow <repo-url>

# 2. 安装
cd helm
helm install ragflow . \
  --namespace ragflow \
  --create-namespace \
  --values values.yaml

# 3. 升级
helm upgrade ragflow . --namespace ragflow

# 4. 卸载
helm uninstall ragflow --namespace ragflow
```

#### 自定义配置

编辑 `helm/values.yaml`:
```yaml
replicaCount: 2
image:
  repository: infiniflow/ragflow
  tag: v0.22.1
service:
  type: LoadBalancer
  port: 80
```

### 5.5 生产部署建议

#### 资源要求
- **CPU**: >= 4 cores (推荐 8 cores)
- **内存**: >= 16 GB (推荐 32 GB)
- **磁盘**: >= 50 GB (推荐 SSD)
- **GPU**: (可选) 用于 Embedding 加速

#### 高可用配置
- **API 服务器**: 多实例 + 负载均衡
- **数据库**: 主从复制或集群
- **搜索引擎**: 集群部署
- **Redis**: 哨兵模式或集群
- **MinIO**: 分布式部署

#### 性能优化
- **缓存策略**: Redis 缓存热点数据
- **CDN**: 静态资源使用 CDN
- **数据库索引**: 优化查询性能
- **异步任务**: 使用任务队列处理耗时操作
- **连接池**: 数据库和 Redis 连接池

#### 监控和日志
- **日志收集**: ELK Stack 或类似方案
- **指标监控**: Prometheus + Grafana
- **链路追踪**: Langfuse 集成
- **告警**: 关键指标告警

---

## 6. 环境配置

### 6.1 环境变量配置

#### 必需环境变量

```bash
# 服务配置
RAGFLOW_API_HOST=0.0.0.0
RAGFLOW_API_PORT=8080

# 数据库配置
MYSQL_HOST=mysql
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=infini_rag_flow
MYSQL_DATABASE=rag_flow

# Redis 配置
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=infini_rag_flow
REDIS_DB=1

# Elasticsearch 配置
ES_HOST=elasticsearch
ES_PORT=9200
ES_USER=elastic
ES_PASSWORD=infini_rag_flow

# MinIO 配置
MINIO_HOST=minio
MINIO_PORT=9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin

# LLM API Keys (按需配置)
OPENAI_API_KEY=sk-xxx
ANTHROPIC_API_KEY=sk-ant-xxx
GOOGLE_API_KEY=xxx
DASHSCOPE_API_KEY=xxx  # 阿里云通义千问
```

#### 可选环境变量

```bash
# Embedding 服务
INFINITY_HOST=infinity
INFINITY_PORT=7997

# 日志级别
LOG_LEVEL=INFO

# 沙箱配置 (gVisor)
ENABLE_SANDBOX=true

# 监控
LANGFUSE_PUBLIC_KEY=xxx
LANGFUSE_SECRET_KEY=xxx
LANGFUSE_HOST=https://cloud.langfuse.com
```

### 6.2 配置文件

#### 后端配置

**`conf/service_conf.yaml`** - 主配置文件
```yaml
# MySQL 配置
mysql:
  host: mysql
  port: 3306
  user: root
  password: infini_rag_flow
  database: rag_flow

# Redis 配置
redis:
  host: redis
  port: 6379
  password: infini_rag_flow
  db: 1

# Elasticsearch 配置
es:
  hosts:
    - http://elasticsearch:9200
  user: elastic
  password: infini_rag_flow

# MinIO 配置
minio:
  endpoint: minio:9000
  access_key: minioadmin
  secret_key: minioadmin
  secure: false

# LLM 工厂配置
llm_factories: conf/llm_factories.json
```

**`conf/llm_factories.json`** - LLM 配置
```json
{
  "OpenAI": {
    "api_base": "https://api.openai.com/v1",
    "api_key": "${OPENAI_API_KEY}",
    "models": ["gpt-4", "gpt-3.5-turbo"]
  },
  "Anthropic": {
    "api_key": "${ANTHROPIC_API_KEY}",
    "models": ["claude-3-opus", "claude-3-sonnet"]
  }
}
```

**`api/settings.py`** - API 设置
```python
# Flask/Quart 配置
SECRET_KEY = os.getenv("SECRET_KEY", "your-secret-key")
SESSION_TYPE = "redis"
PERMANENT_SESSION_LIFETIME = 86400  # 24小时

# CORS 配置
CORS_ORIGINS = ["*"]

# 文件上传
MAX_CONTENT_LENGTH = 100 * 1024 * 1024  # 100MB
UPLOAD_FOLDER = "/tmp/ragflow"
```

#### 前端配置

**`web/.env`** - 前端环境变量
```bash
# API 地址
UMI_APP_API_BASE_URL=http://localhost:8080

# 公共路径
PUBLIC_PATH=/
```

**`web/.umirc.ts`** - UmiJS 配置
```typescript
export default {
  title: 'RAGFlow',
  hash: true,
  routes: [
    // 路由配置
  ],
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
    },
  },
  // ... 其他配置
};
```

**`web/tailwind.config.js`** - Tailwind 配置
```javascript
module.exports = {
  content: [
    './src/**/*.{js,jsx,ts,tsx}',
  ],
  theme: {
    extend: {
      // 自定义主题
    },
  },
  plugins: [],
};
```

### 6.3 开发与生产环境差异

#### 开发环境
- **调试**: 启用详细日志 (`LOG_LEVEL=DEBUG`)
- **热重载**: 前端自动刷新，后端监听文件变化
- **Mock 数据**: 可选 Mock API
- **Source Maps**: 启用源码映射
- **工具**: React DevTools, Query DevTools

#### 生产环境
- **优化**: 代码压缩、Tree Shaking
- **日志**: 适度日志 (`LOG_LEVEL=INFO`)
- **缓存**: 启用全部缓存策略
- **CDN**: 静态资源 CDN 分发
- **监控**: 完整的监控和告警
- **备份**: 定期数据备份
- **安全**: HTTPS、CORS 限制、认证授权

### 6.4 数据持久化

#### Docker Volumes

```yaml
volumes:
  mysql_data:      # MySQL 数据
  es_data:         # Elasticsearch 数据
  redis_data:      # Redis 数据
  minio_data:      # MinIO 对象存储
  ragflow_logs:    # 应用日志
```

#### 备份策略

```bash
# MySQL 备份
docker exec ragflow-mysql mysqldump -u root -p rag_flow > backup.sql

# MinIO 备份
mc mirror minio/ragflow /backup/minio/

# Elasticsearch 快照
# 配置快照仓库并创建快照
```

---

## 7. 项目特点总结

### 7.1 技术亮点
1. **深度文档理解**: 不同于简单的文本提取，实现了版面分析、表格识别等深度理解
2. **模板化分块**: 提供多种分块策略模板，可视化调整
3. **多路召回**: 向量检索 + 关键词检索 + 融合重排序
4. **Agent 系统**: 完整的 Agent 工作流编排和工具调用
5. **图RAG**: 知识图谱增强的检索能力
6. **MCP 支持**: 支持 Model Context Protocol
7. **多模态**: 支持图像、表格等多模态内容处理
8. **可扩展**: 插件系统支持自定义扩展

### 7.2 架构特点
- **微服务架构**: API、任务处理、前端分离
- **容器化部署**: 完整的 Docker 和 K8s 支持
- **异步处理**: Quart 异步框架 + Redis 任务队列
- **多租户**: 支持多用户、多知识库隔离
- **水平扩展**: 各组件可独立扩展

### 7.3 开发体验
- **现代工具链**: uv (Python), UmiJS (前端)
- **类型安全**: TypeScript + Python Type Hints
- **代码质量**: ruff (Python), ESLint (前端)
- **组件化**: 前端组件化开发，后端 Blueprint 模块化
- **文档完善**: 详细的 README 和 API 文档

### 7.4 生产就绪
- **性能优化**: 缓存、连接池、异步 I/O
- **可观测性**: 日志、监控、链路追踪
- **安全性**: 认证授权、加密、XSS 防护
- **高可用**: 支持集群部署和容灾
- **国际化**: 多语言支持

---

## 8. 快速链接

- **项目仓库**: https://github.com/infiniflow/ragflow
- **在线演示**: https://demo.ragflow.io
- **文档**: https://ragflow.io/docs/dev/
- **Docker Hub**: https://hub.docker.com/r/infiniflow/ragflow
- **Discord**: https://discord.gg/NjYzJD3GM3
- **Twitter**: https://twitter.com/infiniflowai

---

**文档版本**: 1.0  
**生成日期**: 2025-12-18  
**对应项目版本**: RAGFlow v0.22.1
