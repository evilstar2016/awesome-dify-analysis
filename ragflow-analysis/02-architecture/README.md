# RAGFlow 系统架构文档目录

本目录包含 RAGFlow 项目的系统架构分析文档和架构图。

## 📄 文档列表

### 1. 系统架构文档
- **文件**: [system-architecture.md](./system-architecture.md)
- **内容**:
  - 架构概述和设计原则
  - 系统分层架构详解
  - 核心组件详细说明
  - 数据流和交互流程
  - 外部集成说明
  - 部署架构方案
  - 安全架构设计
  - 可观测性方案
  - 性能优化策略
  - 扩展性设计
  - 架构演进规划

## 📊 架构图

### 2. 系统组件架构图 (PlantUML)
- **文件**: [system-component-architecture.puml](./system-component-architecture.puml)
- **说明**: 完整的系统组件架构图，展示各层级组件和依赖关系
- **包含层次**:
  - 展示层 (Web前端、移动端、第三方集成)
  - 网关层 (Nginx)
  - API服务层 (Quart + Blueprints)
  - 业务逻辑层 (RAG引擎、Agent系统、文档解析、图RAG)
  - 数据访问层 (Peewee+SQLAlchemy双ORM、搜索客户端、缓存客户端)
  - 基础设施层 (MySQL、ES、Redis、MinIO、OceanBase)
  - 外部服务 (LLM提供商、数据源、Embedding服务)

### 3. 知识库问答数据流图 (PlantUML)
- **文件**: [qa-data-flow.puml](./qa-data-flow.puml)
- **说明**: 展示完整的知识库问答流程时序图
- **流程阶段**:
  1. 用户提问阶段
  2. 业务处理阶段
  3. 检索阶段 (向量检索 + 关键词检索)
  4. LLM 推理阶段
  5. 流式响应阶段
  6. 后处理阶段 (引用追溯、保存历史)

### 4. 文档上传解析流程图 (PlantUML)
- **文件**: [document-upload-flow.puml](./document-upload-flow.puml)
- **说明**: 展示文档上传、解析、分块和索引的完整异步流程
- **流程阶段**:
  1. 文档上传阶段
  2. 异步解析阶段 (格式识别、内容提取、OCR)
  3. 分块和索引阶段 (分块策略、向量生成、索引构建)
  4. 前端更新阶段

## 🔍 如何查看 PlantUML 图

### 方法一：使用在线工具
1. 访问 [PlantUML Online Editor](http://www.plantuml.com/plantuml/uml/)
2. 复制 `.puml` 文件内容
3. 粘贴到编辑器中即可查看

### 方法二：使用 VS Code 插件
1. 安装 VS Code 插件: `PlantUML`
2. 安装 Graphviz: `choco install graphviz` (Windows) 或 `brew install graphviz` (Mac)
3. 打开 `.puml` 文件
4. 按 `Alt+D` 预览图表

### 方法三：生成图片
```bash
# 安装 PlantUML
# Mac: brew install plantuml
# Windows: choco install plantuml

# 生成 PNG 图片
plantuml system-component-architecture.puml
plantuml qa-data-flow.puml
plantuml document-upload-flow.puml

# 生成 SVG 图片（推荐，矢量图）
plantuml -tsvg system-component-architecture.puml
plantuml -tsvg qa-data-flow.puml
plantuml -tsvg document-upload-flow.puml
```

## 📐 架构设计亮点

### 1. 模块化设计
- **前后端分离**: React SPA + Python RESTful API
- **分层清晰**: 展示层、API层、业务逻辑层、数据层分离
- **Blueprint 模式**: API 模块化组织

### 2. 高性能架构
- **异步处理**: Quart 异步框架 + asyncio
- **缓存策略**: 三级缓存 (内存、Redis、CDN)
- **并发优化**: 进程池 + 协程池
- **流式响应**: SSE 流式输出，降低首字节延迟

### 3. 可扩展性
- **水平扩展**: 无状态服务设计，支持多实例
- **插件系统**: 可扩展的工具和模型支持
- **微服务化**: 核心组件可独立部署和扩展

### 4. 云原生
- **容器化**: Docker + Docker Compose
- **编排支持**: Kubernetes Helm Charts
- **服务发现**: K8s Service
- **配置管理**: ConfigMap + Secrets

### 5. 数据处理
- **多路召回**: 向量检索 + 关键词检索 + 混合检索
- **重排序**: Reranker 提高检索精度
- **异步任务**: Redis 任务队列处理耗时操作
- **批量优化**: Embedding 批量生成

### 6. 安全性
- **认证授权**: Session + RBAC
- **传输加密**: HTTPS/TLS
- **数据加密**: 密码哈希、敏感数据加密
- **输入验证**: XSS/SQL 注入防护

## 🔗 相关文档

- [项目概览文档](../01-overview/project-overview.md)
- [开发文档](../../README.md)
- [部署文档](../../docker/README.md)
- [API 文档](http://localhost:9380/api/docs) (运行后访问)

## 📝 更新日志

- **2025-12-18**: 初始版本，完整的系统架构分析
  - 创建系统架构文档
  - 创建组件架构图
  - 创建数据流时序图
  - 创建文档处理流程图

---

**文档维护**: 架构文档应随项目演进持续更新  
**反馈渠道**: 如有问题或建议，请提交 Issue 或 PR
