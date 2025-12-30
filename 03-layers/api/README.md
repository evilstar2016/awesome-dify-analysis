# Dify API 流程图文档索引

本目录包含 Dify 后端 API 的完整文档和核心场景的 PlantUML 时序图。

## 文档列表

### API 文档

| 文件 | 说明 |
|------|------|
| [dify_backend_api_documentation.md](./dify_backend_api_documentation.md) | Dify 后端 API 完整文档 |

### 架构图

| 文件 | 说明 |
|------|------|
| [dify_api_architecture.puml](./dify_api_architecture.puml) | Dify API 架构总览 |

### 核心场景时序图

| 文件 | 说明 | 涉及 API |
|------|------|---------|
| [chat_api_sequence.puml](./chat_api_sequence.puml) | Chat/Completion API 调用流程 | `/v1/chat-messages`, `/v1/completion-messages` |
| [workflow_api_sequence.puml](./workflow_api_sequence.puml) | Workflow API 调用流程 | `/v1/workflows/run`, `/v1/workflows/logs` |
| [dataset_api_sequence.puml](./dataset_api_sequence.puml) | Dataset/Knowledge Base API 调用流程 | `/v1/datasets`, `/v1/documents`, `/v1/segments` |
| [file_upload_api_sequence.puml](./file_upload_api_sequence.puml) | 文件上传 API 调用流程 | `/v1/files/upload` |
| [audio_api_sequence.puml](./audio_api_sequence.puml) | 音频转换 API 调用流程 | `/v1/audio-to-text`, `/v1/text-to-audio` |

## PlantUML 查看方式

### 方式一：VS Code 插件

安装 `PlantUML` 插件后，可以在 VS Code 中直接预览 `.puml` 文件。

### 方式二：在线工具

访问 [PlantUML Online Server](http://www.plantuml.com/plantuml/uml/) 粘贴代码预览。

### 方式三：命令行生成

```bash
# 安装 PlantUML
# macOS: brew install plantuml
# Windows: choco install plantuml

# 生成 PNG
plantuml chat_api_sequence.puml

# 生成 SVG
plantuml -tsvg chat_api_sequence.puml
```

## API 类别说明

| API 类别 | URL 前缀 | 认证方式 | 目标用户 |
|---------|---------|---------|---------|
| **Service API** | `/v1/` | Bearer Token | 第三方开发者 |
| **Console API** | `/console/api/` | Session + JWT | 管理员/开发者 |
| **Web API** | `/api/` | Passport Token | 最终用户 |

## 核心概念

### 1. 应用模式 (App Mode)

| 模式 | 说明 | 适用 API |
|-----|------|---------|
| `completion` | 文本补全 | `/v1/completion-messages` |
| `chat` | 基础聊天 | `/v1/chat-messages` |
| `agent_chat` | Agent 聊天 | `/v1/chat-messages` |
| `advanced_chat` | 高级聊天 | `/v1/chat-messages` |
| `workflow` | 工作流 | `/v1/workflows/run` |

### 2. 响应模式 (Response Mode)

| 模式 | 说明 |
|-----|------|
| `blocking` | 阻塞模式，等待完整响应 |
| `streaming` | 流式模式，通过 SSE 返回 |

### 3. 索引技术 (Indexing Technique)

| 技术 | 说明 |
|-----|------|
| `high_quality` | 高质量向量索引 |
| `economy` | 经济型关键词索引 |

## 相关资源

- [Dify 官方文档](https://docs.dify.ai/)
- [API Reference](https://docs.dify.ai/guides/tools)
- [SDK 目录](../../sdks/)

---

📱 **关注公众号「柒叔代码阁」**

定期发布Dify深度内容及生产实践～

![公众号二维码](../../qrcode.png)

💡 **为什么关注？**

不只是告诉你"是什么"和"怎么做"，更重要的是 **"为什么"** — 理解设计背后的思想，才能举一反三。

💬 **交流讨论**
- GitHub Issue/Discussion：技术问题讨论
- 公众号留言
