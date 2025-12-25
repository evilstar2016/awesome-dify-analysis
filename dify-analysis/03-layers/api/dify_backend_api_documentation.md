# Dify 后端 API 完整文档

## 概述

Dify 后端采用 Flask + Flask-RESTx 构建，提供三大类 API：

| API 类别 | URL 前缀 | 说明 | 认证方式 |
|---------|---------|------|---------|
| **Service API** | `/v1/` | 面向第三方集成的公共服务 API | Bearer Token (API Key) |
| **Console API** | `/console/api/` | 管理后台 API | Session + JWT |
| **Web API** | `/api/` | Web 应用 API | Passport Token |

---

## 一、Service API（服务 API）

Service API 是 Dify 对外开放的核心 API，用于第三方应用集成。

### 1.1 认证方式

所有 Service API 请求必须在 Header 中携带 API Token：

```http
Authorization: Bearer {api_key}
```

API Token 分为两类：
- **App Token**: 用于应用相关 API（`app` 类型）
- **Dataset Token**: 用于知识库相关 API（`dataset` 类型）

### 1.2 Chat/Completion API

#### 1.2.1 创建补全消息
```http
POST /v1/completion-messages
```

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| inputs | object | ✓ | 输入参数字典 |
| query | string | ✗ | 查询字符串 |
| files | array | ✗ | 文件附件列表 |
| response_mode | string | ✗ | 响应模式：`blocking` 或 `streaming` |
| user | string | ✓ | 用户标识 |

**响应：**
```json
{
  "id": "message_id",
  "answer": "AI 生成的回答",
  "created_at": 1234567890
}
```

#### 1.2.2 停止补全任务
```http
POST /v1/completion-messages/{task_id}/stop
```

#### 1.2.3 发送聊天消息
```http
POST /v1/chat-messages
```

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| inputs | object | ✓ | 输入参数字典 |
| query | string | ✓ | 聊天查询内容 |
| files | array | ✗ | 文件附件列表 |
| response_mode | string | ✗ | 响应模式：`blocking` 或 `streaming` |
| conversation_id | string | ✗ | 已有会话 ID |
| auto_generate_name | boolean | ✗ | 是否自动生成会话名称，默认 true |
| workflow_id | string | ✗ | 高级聊天的工作流 ID |
| user | string | ✓ | 用户标识 |

**适用模式：** `chat`、`agent_chat`、`advanced_chat`

#### 1.2.4 停止聊天消息生成
```http
POST /v1/chat-messages/{task_id}/stop
```

### 1.3 Workflow API

#### 1.3.1 运行工作流
```http
POST /v1/workflows/run
```

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| inputs | object | ✓ | 工作流输入参数 |
| files | array | ✗ | 文件附件列表 |
| response_mode | string | ✗ | 响应模式：`blocking` 或 `streaming` |
| user | string | ✓ | 用户标识 |

#### 1.3.2 运行指定工作流
```http
POST /v1/workflows/{workflow_id}/run
```

#### 1.3.3 获取工作流运行详情
```http
GET /v1/workflows/run/{workflow_run_id}
```

**响应：**
```json
{
  "id": "run_id",
  "workflow_id": "workflow_id",
  "status": "succeeded",
  "inputs": {},
  "outputs": {},
  "total_steps": 5,
  "total_tokens": 1000,
  "created_at": 1234567890,
  "finished_at": 1234567891,
  "elapsed_time": 1.5
}
```

#### 1.3.4 停止工作流任务
```http
POST /v1/workflows/tasks/{task_id}/stop
```

#### 1.3.5 获取工作流日志
```http
GET /v1/workflows/logs
```

**查询参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| keyword | string | 关键词搜索 |
| status | string | 状态过滤：`succeeded`、`failed`、`stopped` |
| created_at__before | string | 创建时间之前 |
| created_at__after | string | 创建时间之后 |
| page | int | 页码，默认 1 |
| limit | int | 每页数量，默认 20 |

### 1.4 Conversation API

#### 1.4.1 获取会话列表
```http
GET /v1/conversations
```

**查询参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| last_id | string | 上一页最后会话 ID |
| limit | int | 数量限制，1-100，默认 20 |
| sort_by | string | 排序：`created_at`、`-created_at`、`updated_at`、`-updated_at` |
| user | string | 用户标识 |

#### 1.4.2 删除会话
```http
DELETE /v1/conversations/{conversation_id}
```

#### 1.4.3 重命名会话
```http
POST /v1/conversations/{conversation_id}/name
```

**请求参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| name | string | 新会话名称 |
| auto_generate | boolean | 是否自动生成名称 |

#### 1.4.4 获取会话变量
```http
GET /v1/conversations/{conversation_id}/variables
```

#### 1.4.5 更新会话变量
```http
PUT /v1/conversations/{conversation_id}/variables/{variable_id}
```

### 1.5 Message API

#### 1.5.1 获取消息列表
```http
GET /v1/messages
```

**查询参数：**
| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| conversation_id | string | ✓ | 会话 ID |
| first_id | string | ✗ | 分页起始消息 ID |
| limit | int | ✗ | 数量限制，1-100，默认 20 |
| user | string | ✓ | 用户标识 |

#### 1.5.2 提交消息反馈
```http
POST /v1/messages/{message_id}/feedbacks
```

**请求参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| rating | string | 评分：`like`、`dislike` 或 `null` |
| content | string | 反馈内容 |
| user | string | 用户标识 |

#### 1.5.3 获取建议问题
```http
GET /v1/messages/{message_id}/suggested
```

#### 1.5.4 获取应用反馈列表
```http
GET /v1/app/feedbacks
```

### 1.6 Audio API

#### 1.6.1 语音转文字
```http
POST /v1/audio-to-text
```

**请求方式：** `multipart/form-data`

**请求参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| file | file | 音频文件 |
| user | string | 用户标识 |

#### 1.6.2 文字转语音
```http
POST /v1/text-to-audio
```

**请求参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| message_id | string | 消息 ID（可选） |
| voice | string | 语音类型 |
| text | string | 要转换的文本 |
| streaming | boolean | 是否流式响应 |
| user | string | 用户标识 |

### 1.7 File API

#### 1.7.1 上传文件
```http
POST /v1/files/upload
```

**请求方式：** `multipart/form-data`

**请求参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| file | file | 要上传的文件 |
| user | string | 用户标识 |

**响应：**
```json
{
  "id": "file_id",
  "name": "file.pdf",
  "size": 1024,
  "extension": "pdf",
  "mime_type": "application/pdf",
  "created_at": 1234567890
}
```

---

## 二、Dataset API（知识库 API）

### 2.1 Dataset（数据集）管理

#### 2.1.1 获取数据集列表
```http
GET /v1/datasets
```

**查询参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| page | int | 页码，默认 1 |
| limit | int | 每页数量，默认 20 |
| keyword | string | 关键词搜索 |
| tag_ids | array | 标签 ID 列表 |
| include_all | boolean | 是否包含所有 |

#### 2.1.2 创建数据集
```http
POST /v1/datasets
```

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| name | string | ✓ | 数据集名称（1-40 字符） |
| description | string | ✗ | 描述 |
| indexing_technique | string | ✗ | 索引技术：`high_quality`、`economy` |
| permission | string | ✗ | 权限：`only_me`、`all_team_members`、`partial_members` |
| provider | string | ✗ | 提供商，默认 `vendor` |
| embedding_model | string | ✗ | 嵌入模型名称 |
| embedding_model_provider | string | ✗ | 嵌入模型提供商 |
| retrieval_model | object | ✗ | 检索模型配置 |

#### 2.1.3 获取数据集详情
```http
GET /v1/datasets/{dataset_id}
```

#### 2.1.4 更新数据集
```http
PATCH /v1/datasets/{dataset_id}
```

#### 2.1.5 删除数据集
```http
DELETE /v1/datasets/{dataset_id}
```

### 2.2 Document（文档）管理

#### 2.2.1 通过文本创建文档
```http
POST /v1/datasets/{dataset_id}/document/create-by-text
```

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| name | string | ✓ | 文档名称 |
| text | string | ✓ | 文档内容 |
| process_rule | object | ✗ | 处理规则 |
| doc_form | string | ✗ | 文档形式，默认 `text_model` |
| doc_language | string | ✗ | 文档语言，默认 `English` |
| indexing_technique | string | ✗ | 索引技术 |
| retrieval_model | object | ✗ | 检索模型配置 |

#### 2.2.2 通过文本更新文档
```http
POST /v1/datasets/{dataset_id}/documents/{document_id}/update-by-text
```

#### 2.2.3 通过文件创建文档
```http
POST /v1/datasets/{dataset_id}/document/create-by-file
```

**请求方式：** `multipart/form-data`

**请求参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| file | file | 要上传的文件 |
| data | string | JSON 格式的配置参数 |

#### 2.2.4 通过文件更新文档
```http
POST /v1/datasets/{dataset_id}/documents/{document_id}/update-by-file
```

#### 2.2.5 获取文档列表
```http
GET /v1/datasets/{dataset_id}/documents
```

**查询参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| page | int | 页码，默认 1 |
| limit | int | 每页数量，默认 20 |
| keyword | string | 关键词搜索 |

#### 2.2.6 获取文档详情
```http
GET /v1/datasets/{dataset_id}/documents/{document_id}
```

**查询参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| metadata | string | `all`、`only`、`without` |

#### 2.2.7 删除文档
```http
DELETE /v1/datasets/{dataset_id}/documents/{document_id}
```

#### 2.2.8 获取文档索引状态
```http
GET /v1/datasets/{dataset_id}/documents/{batch}/indexing-status
```

#### 2.2.9 批量更新文档状态
```http
PATCH /v1/datasets/{dataset_id}/documents/status/{action}
```

**路径参数：**
- `action`: `enable`、`disable`、`archive`、`un_archive`

### 2.3 Segment（分段）管理

#### 2.3.1 创建分段
```http
POST /v1/datasets/{dataset_id}/documents/{document_id}/segments
```

**请求参数：**
```json
{
  "segments": [
    {
      "content": "分段内容",
      "answer": "可选的答案",
      "keywords": ["关键词1", "关键词2"]
    }
  ]
}
```

#### 2.3.2 获取分段列表
```http
GET /v1/datasets/{dataset_id}/documents/{document_id}/segments
```

**查询参数：**
| 参数 | 类型 | 说明 |
|-----|------|------|
| status | string | 状态过滤 |
| keyword | string | 关键词搜索 |
| page | int | 页码 |
| limit | int | 每页数量 |

#### 2.3.3 获取分段详情
```http
GET /v1/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}
```

#### 2.3.4 更新分段
```http
POST /v1/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}
```

#### 2.3.5 删除分段
```http
DELETE /v1/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}
```

### 2.4 Child Chunk（子块）管理

#### 2.4.1 创建子块
```http
POST /v1/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}/child_chunks
```

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| content | string | ✓ | 子块内容 |

#### 2.4.2 获取子块列表
```http
GET /v1/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}/child_chunks
```

#### 2.4.3 更新子块
```http
PATCH /v1/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}/child_chunks/{child_chunk_id}
```

#### 2.4.4 删除子块
```http
DELETE /v1/datasets/{dataset_id}/documents/{document_id}/segments/{segment_id}/child_chunks/{child_chunk_id}
```

### 2.5 Hit Testing（检索测试）

#### 2.5.1 执行检索测试
```http
POST /v1/datasets/{dataset_id}/hit-testing
```
或
```http
POST /v1/datasets/{dataset_id}/retrieve
```

### 2.6 Tags（标签）管理

#### 2.6.1 获取标签列表
```http
GET /v1/datasets/tags
```

#### 2.6.2 创建标签
```http
POST /v1/datasets/tags
```

#### 2.6.3 更新标签
```http
PATCH /v1/datasets/tags
```

#### 2.6.4 删除标签
```http
DELETE /v1/datasets/tags
```

#### 2.6.5 绑定标签
```http
POST /v1/datasets/tags/binding
```

#### 2.6.6 解绑标签
```http
POST /v1/datasets/tags/unbinding
```

#### 2.6.7 获取数据集标签
```http
GET /v1/datasets/{dataset_id}/tags
```

---

## 三、错误处理

### 3.1 HTTP 状态码

| 状态码 | 说明 |
|-------|------|
| 200 | 成功 |
| 201 | 创建成功 |
| 204 | 删除成功（无返回内容） |
| 400 | 请求参数错误 |
| 401 | 未授权（Token 无效） |
| 403 | 权限不足 |
| 404 | 资源不存在 |
| 409 | 资源冲突 |
| 413 | 文件过大 |
| 415 | 不支持的文件类型 |
| 429 | 请求频率超限 |
| 500 | 服务器内部错误 |

### 3.2 错误响应格式

```json
{
  "code": "error_code",
  "message": "错误描述",
  "status": 400
}
```

### 3.3 常见错误码

| 错误码 | 说明 |
|-------|------|
| `app_unavailable` | 应用不可用 |
| `completion_request_error` | 补全请求错误 |
| `conversation_completed` | 会话已结束 |
| `not_chat_app` | 非聊天应用 |
| `not_workflow_app` | 非工作流应用 |
| `provider_not_initialize` | 模型提供商未初始化 |
| `provider_quota_exceeded` | 配额超限 |
| `dataset_name_duplicate` | 数据集名称重复 |
| `dataset_in_use` | 数据集正在使用中 |
| `document_indexing` | 文档正在索引中 |

---

## 四、速率限制

### 4.1 知识库 API 限制

- 对于知识库相关 API，存在每分钟请求数限制
- 超过限制将返回 403 错误
- 限制基于租户级别的订阅计划

### 4.2 云版本资源限制

| 资源类型 | 说明 |
|---------|------|
| members | 团队成员数量限制 |
| apps | 应用数量限制 |
| vector_space | 向量空间容量限制 |
| documents | 文档上传数量限制 |

---

## 五、最佳实践

### 5.1 流式响应处理

对于 `response_mode: "streaming"` 的请求，服务器将返回 Server-Sent Events (SSE) 格式的流式响应：

```javascript
const eventSource = new EventSource('/v1/chat-messages', {
  headers: {
    'Authorization': 'Bearer {api_key}'
  }
});

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  // 处理流式数据
};
```

### 5.2 用户标识最佳实践

- 始终提供唯一的用户标识 (`user` 参数)
- 用户标识用于会话管理和用量统计
- 如果不提供，系统将使用默认会话 ID

### 5.3 文件上传注意事项

- 支持的文件类型取决于应用配置
- 文件大小限制取决于订阅计划
- 上传的文件会关联到租户和用户

---

## 六、SDK 支持

Dify 提供多语言 SDK：

- **Python SDK**: `sdks/python-client/`
- **Node.js SDK**: `sdks/nodejs-client/`
- **PHP SDK**: `sdks/php-client/`

详细使用说明请参考各 SDK 目录下的 README 文件。
