# 对话系统模块 (Chat/Dialog Module)

## 模块概述

对话系统是 RAGFlow 的核心交互模块，负责处理用户查询、检索相关内容、生成答案，并管理对话历史。该模块支持同步对话、流式对话、多轮对话管理，实现了完整的 RAG 对话流程。

### 核心价值
- **智能对话**：基于 RAG 的智能问答，结合知识库内容生成准确答案
- **流式响应**：支持 SSE 流式输出，提升用户体验
- **上下文管理**：多轮对话上下文跟踪，保持对话连贯性
- **引用溯源**：答案标注内容来源，支持引用验证

---

## 功能清单

### 1. 对话管理 (Dialog Management)
- **创建对话** - 创建新的对话会话
- **对话配置** - 配置 LLM 模型、检索参数、提示词
- **对话列表** - 获取用户的所有对话
- **对话删除** - 删除对话及其所有消息

### 2. 消息管理 (Message Management)
- **发送消息** - 用户发送问题
- **获取消息** - 获取对话的所有消息
- **消息反馈** - 点赞/点踩反馈
- **消息删除** - 删除指定消息

### 3. 对话模式 (Chat Modes)
- **同步对话** - 等待完整答案后返回
- **流式对话** - SSE 流式输出答案
- **工具调用** - 支持 Function Calling 集成外部工具

### 4. 检索集成 (Retrieval Integration)
- **向量检索** - 从知识库检索相关分块
- **重排序** - 对检索结果进行重排序
- **上下文组装** - 构建 LLM 提示词
- **引用标注** - 标注答案来源

---

## 目录结构

```
api/apps/dialog_app.py          # 对话 API 端点
api/db/services/
  ├── dialog_service.py         # 对话服务 (CRUD)
  └── conversation_service.py   # 消息服务
rag/app/
  ├── ragchat.py                # RAG 对话引擎
  └── retrieval.py              # 检索模块
rag/llm/
  ├── chat_model.py             # LLM 封装
  └── chat.py                   # 对话接口
```

---

## 数据模型

### Dialog (对话)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | STRING(32) | 对话 UUID |
| name | STRING(128) | 对话名称 |
| description | TEXT | 对话描述 |
| tenant_id | STRING(32) | 租户 ID |
| kb_ids | JSON | 关联知识库 ID 列表 |
| llm_id | STRING(32) | LLM 模型 ID |
| llm_setting | JSON | LLM 配置 (temperature, top_p, max_tokens) |
| prompt_config | JSON | 提示词配置 |
| similarity_threshold | FLOAT | 相似度阈值 (0.2) |
| vector_similarity_weight | FLOAT | 向量权重 (0.3) |
| top_k | INT | 向量检索数量 (1024) |
| top_n | INT | 重排序后数量 (8) |
| rerank_id | STRING(32) | 重排模型 ID |
| status | ENUM | 对话状态 (active, archived) |
| icon | STRING | 对话图标 |
| create_time | DATETIME | 创建时间 |
| update_time | DATETIME | 更新时间 |

### Conversation (对话轮次)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | STRING(32) | 对话轮次 UUID |
| dialog_id | STRING(32) | 所属对话 ID |
| user_id | STRING(255) | 用户 ID |
| name | STRING(255) | 对话轮次名称 |
| message | JSON | 消息列表 (包含 user/assistant 消息) |
| reference | JSON | 引用的分块列表 (默认 []) |

---

## API 接口

### 创建对话
```http
POST /api/v1/dialog/set
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "技术支持对话",
  "kb_ids": ["kb_uuid1", "kb_uuid2"],
  "llm_id": "llm_uuid",
  "llm_setting": {
    "temperature": 0.7,
    "top_p": 0.9,
    "max_tokens": 2048
  },
  "similarity_threshold": 0.2,
  "top_k": 1024,
  "top_n": 8
}
```

### 同步对话
```http
POST /api/v1/conversation/completion
Authorization: Bearer {token}
Content-Type: application/json

{
  "conversation_id": "conv_uuid",
  "messages": [
    {"role": "user", "content": "如何使用 RAGFlow?"}
  ],
  "stream": false
}

Response:
{
  "code": 0,
  "data": {
    "answer": "RAGFlow 使用步骤...",
    "reference": [
      {
        "doc_name": "用户手册.pdf",
        "page": 5,
        "content": "...",
        "similarity": 0.85
      }
    ],
    "doc_aggs": [
      {
        "doc_id": "doc_uuid",
        "doc_name": "用户手册.pdf",
        "count": 3
      }
    ]
  }
}
```

### 流式对话
```http
POST /api/v1/conversation/completion
Authorization: Bearer {token}
Content-Type: application/json

{
  "conversation_id": "conv_uuid",
  "messages": [
    {"role": "user", "content": "如何使用 RAGFlow?"}
  ],
  "stream": true
}

Response (SSE):
data: {"answer": "RAG", "reference": [...]}
data: {"answer": "Flow ", "reference": [...]}
data: {"answer": "使用", "reference": [...]}
data: {"answer": "步骤", "reference": [...]}
data: [DONE]
```

### 获取对话消息
```http
GET /api/v1/conversation/list
Authorization: Bearer {token}
Query: dialog_id={dialog_id}

Response:
{
  "code": 0,
  "data": {
    "messages": [
      {
        "role": "user",
        "content": "如何使用 RAGFlow?",
        "create_time": "2024-01-01 10:00:00"
      },
      {
        "role": "assistant",
        "content": "RAGFlow 使用步骤...",
        "reference": [...],
        "create_time": "2024-01-01 10:00:05"
      }
    ],
    "total": 2
  }
}
```

---

## 技术栈

### 后端框架
- **Flask/Quart** - Web 框架
- **Peewee** - ORM
- **Celery** - 异步任务

### LLM 集成
- **OpenAI API** - GPT-4, GPT-3.5
- **Azure OpenAI** - Azure 部署
- **开源模型** - Qwen, Llama, Baichuan
- **本地部署** - vLLM, TGI, Ollama

### 流式技术
- **SSE (Server-Sent Events)** - 服务器推送
- **Generator** - Python 生成器
- **Stream Buffer** - 流式缓冲

---

## 依赖关系

```
对话系统模块 (dialog_app.py)
    │
    ├─→ DialogService (dialog_service.py)
    │   └─→ MySQL (Dialog 表)
    │
    ├─→ ConversationService (conversation_service.py)
    │   └─→ MySQL (Conversation 表)
    │
    ├─→ RAGChat (ragchat.py)
    │   ├─→ Retrieval (retrieval.py)
    │   │   ├─→ Elasticsearch/Infinity
    │   │   └─→ Reranker
    │   │
    │   └─→ ChatModel (chat_model.py)
    │       ├─→ OpenAI API
    │       ├─→ Azure OpenAI
    │       └─→ 本地模型
    │
    └─→ KnowledgebaseService
        └─→ MySQL (Knowledgebase 表)
```

---

## 核心流程

### 对话流程 (详见时序图)
1. **用户提问** - 前端发送问题到 API
2. **检索相关内容** - 从知识库检索 TopK 分块
3. **重排序** - 使用重排模型优化结果
4. **构建提示词** - 组装系统提示词 + 上下文 + 问题
5. **调用 LLM** - 调用 LLM 生成答案
6. **流式输出** - SSE 流式返回答案片段
7. **保存消息** - 存储问题和答案到数据库
8. **标注引用** - 标注答案中引用的分块

### 多轮对话管理
- **上下文窗口** - 保持最近 N 轮对话
- **Token 限制** - 动态裁剪上下文以适应 Token 限制
- **记忆机制** - 提取关键信息作为长期记忆

---

## 配置参数

### LLM 配置 (llm_setting)
```json
{
  "model_name": "gpt-4",
  "temperature": 0.7,
  "top_p": 0.9,
  "max_tokens": 2048,
  "presence_penalty": 0.0,
  "frequency_penalty": 0.0
}
```

### 检索配置
- **similarity_threshold** (0.0-1.0) - 相似度阈值，过滤低相关分块
- **vector_similarity_weight** (0.0-1.0) - 向量搜索权重 (vs BM25)
- **top_k** (int) - 向量检索初始数量
- **top_n** (int) - 重排序后最终数量

### 提示词配置 (prompt_config)
```json
{
  "system": "你是一个专业的 AI 助手...",
  "prologue": "你好！我是 RAGFlow 助手...",
  "quote": true,
  "empty_response": "抱歉，我没有找到相关信息。"
}
```

---

## 性能监控

### 指标项
- **对话 QPS** - 每秒对话请求数
- **平均响应时间** - 从提问到首字输出 (TTFT)
- **检索耗时** - 向量检索 + 重排序时间
- **LLM 耗时** - LLM 推理时间
- **Token 消耗** - 输入/输出 Token 统计

### 日志记录
- 对话请求日志
- 检索结果日志
- LLM 调用日志
- 错误和异常日志

---

## 常见问题

### Q1: 如何提高答案质量？
- 优化提示词模板
- 调整相似度阈值
- 使用更好的重排模型
- 提高知识库内容质量

### Q2: 如何处理长对话？
- 使用滑动窗口保留最近 N 轮
- 提取关键信息作为摘要
- 使用长上下文模型 (100K+)

### Q3: 如何支持多模态对话？
- 集成 GPT-4V/Claude Vision
- 支持图片上传和识别
- 结合 OCR 和多模态检索

---

## 相关文档

- [创建对话时序图](./01-create-dialog-sequence.puml)
- [同步对话时序图](./02-sync-chat-sequence.puml)
- [流式对话时序图](./03-stream-chat-sequence.puml)
- [上下文检索时序图](./04-context-retrieval-sequence.puml)
- [对话管理详细文档](./dialog-management.md) *(待创建)*
