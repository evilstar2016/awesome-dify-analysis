# FastGPT 第三方集成分析

本目录整理了 FastGPT 中所有支持的第三方服务集成。

## 文档列表

1. [01-大语言模型集成.md](./01-大语言模型集成.md) - LLM 提供商集成（OpenAI、OneAPI 等）
2. [02-向量数据库集成.md](./02-向量数据库集成.md) - 向量存储服务（PostgreSQL PGVector、Milvus、OceanBase）
3. [03-对象存储集成.md](./03-对象存储集成.md) - 文件存储服务（S3 兼容存储、MinIO）
4. [04-消息平台集成.md](./04-消息平台集成.md) - 即时通讯平台（飞书、钉钉、企业微信）
5. [05-文档处理集成.md](./05-文档处理集成.md) - 文档解析服务（Doc2X）
6. [06-数据库与缓存集成.md](./06-数据库与缓存集成.md) - MongoDB、Redis
7. [07-AI增强服务集成.md](./07-AI增强服务集成.md) - TTS、STT、OCR、重排序
8. [08-代码执行集成.md](./08-代码执行集成.md) - 代码沙箱服务
9. [09-知识库集成.md](./09-知识库集成.md) - 语雀、飞书云文档
10. [10-认证与支付集成.md](./10-认证与支付集成.md) - OAuth、支付网关

## 集成架构概览

FastGPT 的第三方集成分为以下几个层次：

### 1. 核心基础设施层
- **数据库**：MongoDB（主数据存储）、Redis（缓存和队列）
- **向量数据库**：PostgreSQL with PGVector、Milvus、OceanBase
- **对象存储**：S3 兼容存储（MinIO）

### 2. AI 服务层
- **LLM 提供商**：OpenAI、OneAPI（支持多种模型）
- **Embedding 模型**：向量化服务
- **TTS/STT**：语音合成与识别
- **OCR**：文档识别
- **Rerank**：结果重排序

### 3. 文档处理层
- **PDF 解析**：Doc2X、自定义解析服务、插件式解析器
- **知识库同步**：飞书云文档、语雀

### 4. 通讯集成层
- **消息平台**：飞书机器人、企业微信、钉钉、微信公众号
- **通知服务**：邮件、短信

### 5. 认证与支付层
- **OAuth**：GitHub、Google、微信、Microsoft
- **支付网关**：微信支付、支付宝、银行转账

### 6. 扩展服务层
- **代码沙箱**：Python/JavaScript 代码执行环境
- **MCP Server**：Model Context Protocol 服务器代理

## 配置方式

所有第三方服务的配置主要通过以下方式：

1. **环境变量配置**（`config.json`）
2. **团队级别配置**（Team OpenAI Account）
3. **工作流级别配置**（External Provider Variables）
4. **应用级别配置**（API Dataset Server）

详细配置方法请参阅各个具体文档。
