# API 路由层 (API Routes Layer)

## 概述
API 路由层是 FastGPT 的 HTTP 接口层，基于 Next.js API Routes 实现，负责接收前端请求、参数校验、权限认证、调用服务层逻辑，并返回统一格式的响应。所有 API 路由位于 `projects/app/src/pages/api/` 目录下。

## 技术栈
- **框架**: Next.js 14 API Routes
- **中间件**: 自定义中间件链 (NextEntry)
- **校验**: Zod Schema 验证
- **认证**: JWT Token / Cookie Session
- **限流**: IP 频率限制、用户级限流
- **日志**: 请求日志、审计日志 (AuditEventEnum)

## 核心架构

### 中间件系统 (NextEntry)
所有 API 路由通过 `NextEntry()` 包装，自动处理：
- 错误捕获与统一错误响应
- 请求日志记录
- CORS 配置
- 请求体解析
- 响应格式化

```typescript
// 基本用法
async function handler(req: ApiRequestProps<BodyType, QueryType>) {
  // 业务逻辑
  return result;
}
export default NextEntry({ beforeCallback: [] })(handler);

// 带中间件
export default NextEntry({ beforeCallback: [] })(
  authCert({ authToken: true }),      // 认证
  checkTeamAppTypeLimit(),            // 权限校验
  useIPFrequencyLimit({...}),        // 频率限制
  handler
);
```

### 路由结构

```
projects/app/src/pages/api/
├── core/                    # 核心业务 API
│   ├── app/                # 应用管理
│   ├── chat/               # 对话管理
│   ├── dataset/            # 知识库管理
│   ├── ai/                 # AI 能力
│   ├── workflow/           # 工作流
│   └── plugin/             # 插件管理
├── support/                # 支持服务 API
│   ├── user/               # 用户与团队
│   ├── wallet/             # 钱包与计费
│   ├── openapi/            # OpenAPI Key
│   ├── outLink/            # 外链分享
│   └── mcp/                # MCP 管理
├── v1/                     # API v1 (OpenAI 兼容)
│   ├── chat/
│   │   └── completions.ts  # 对话接口
│   └── audio/
│       └── transcriptions.ts # 语音转文字
├── v2/                     # API v2 (增强版)
│   └── chat/
│       └── completions.ts
├── common/                 # 通用 API
│   ├── file/               # 文件管理
│   ├── system/             # 系统信息
│   └── tools/              # 工具接口
├── admin/                  # 管理员 API
│   ├── initv*.ts           # 数据库初始化/迁移
│   └── support/
├── system/                 # 系统级 API (文件访问)
├── marketplace/            # 市场 API (代理)
├── plugin/                 # 插件代理
└── openapi.json.ts         # OpenAPI Schema
```

## 核心模块详解

### 1. 应用管理 API (`/api/core/app`)

#### 基础操作
- **POST /create**: 创建应用
  - 权限: Team 应用创建权限
  - 参数: name, type, modules, edges, chatConfig
  - 返回: appId
  - 审计: CREATE_APP

- **PUT /update**: 更新应用
  - 权限: 应用写权限
  - 支持: 元信息、工作流、配置、移动父目录
  - 审计: UPDATE_APP

- **DELETE /del**: 删除应用
  - 权限: 应用所有者
  - 级联: 删除版本、对话记录、权限
  - 审计: DELETE_APP

- **GET /detail**: 获取应用详情
  - 权限: 应用读权限
  - 返回: 完整配置 + 工作流

- **POST /list**: 获取应用列表
  - 支持: 分页、搜索、类型过滤、文件夹导航
  - 返回: 应用列表 + 最近对话信息

- **POST /copy**: 复制应用
  - 权限: 源应用读权限 + 目标 Team 创建权限
  - 复制: 工作流、配置、版本（可选）

#### 版本管理 (`/version`)
- **POST /publish**: 发布版本
  - 创建新版本快照
  - 自动保存模式 (autoSave)
  - 支持版本命名

- **GET /list**: 版本列表
  - 分页查询
  - 版本号、时间、发布者

- **GET /detail**: 版本详情
  - 完整工作流配置
  - 用于回滚

- **GET /latest**: 最新版本
  - 获取最新发布的版本

- **PUT /update**: 更新版本信息
  - 修改版本名称

#### 工具管理
**HTTP Tools** (`/httpTools`)
- **POST /create**: 创建 HTTP 工具应用
- **PUT /update**: 更新工具配置 (baseUrl, schema, headers)
- **POST /getApiSchemaByUrl**: 从 URL 解析 OpenAPI Schema
- **POST /runTool**: 测试运行工具

**MCP Tools** (`/mcpTools`)
- **POST /create**: 创建 MCP 工具应用
- **PUT /update**: 更新 MCP 工具配置
- **POST /getChildren**: 获取 MCP 子节点

#### 文件夹管理 (`/folder`)
- **POST /create**: 创建文件夹
- **GET /path**: 获取文件夹路径 (面包屑)

#### 其他
- **GET /getBasicInfo**: 批量获取应用基本信息
- **POST /getChatLogs**: 获取对话日志 (分页、筛选、搜索)
- **POST /exportChatLogs**: 导出对话日志 (CSV/Excel)
- **GET /template/list**: 获取模板列表
- **GET /template/detail**: 获取模板详情
- **POST /resumeInheritPermission**: 恢复继承权限

### 2. 对话管理 API (`/api/core/chat`)

#### 核心接口
- **POST /init**: 初始化对话
  - 创建新对话或加载历史
  - 返回: chatId, history, app info

- **POST /chatTest**: 测试对话 (调试模式)
  - 不保存记录
  - 返回详细执行日志

#### 历史管理
- **POST /getHistories**: 获取对话列表
  - 按应用分组
  - 支持搜索

- **POST /getPaginationRecords**: 分页获取对话记录
  - 单个对话的消息列表

- **DELETE /delHistory**: 删除对话
- **DELETE /clearHistories**: 清空对话历史
- **PUT /updateHistory**: 更新对话标题

#### 引用与反馈
- **GET /quote/getQuote**: 获取引用详情
- **GET /quote/getCollectionQuote**: 获取文档集合引用
- **POST /feedback/updateUserFeedback**: 用户反馈 (赞/踩)
- **PUT /feedback/adminUpdate**: 管理员更新反馈
- **POST /feedback/closeCustom**: 关闭自定义反馈

#### 输入引导 (`/inputGuide`)
- **POST /create**: 创建输入引导
- **POST /list**: 查询引导列表
- **POST /query**: 模糊查询
- **PUT /update**: 更新引导
- **DELETE /delete**: 删除引导
- **DELETE /deleteAll**: 删除所有引导
- **GET /countTotal**: 统计数量

#### 文件与语音
- **POST /presignChatFilePostUrl**: 获取文件上传预签名 URL
- **POST /presignChatFileGetUrl**: 获取文件访问 URL
- **GET /item/getSpeech**: 获取语音合成结果

#### 外链对话
- **POST /outLink/init**: 外链对话初始化
- **POST /team/init**: 团队对话初始化

### 3. 知识库管理 API (`/api/core/dataset`)

#### 知识库 CRUD
- **POST /create**: 创建知识库
  - 支持: 普通知识库、文件夹、外部知识库
  - 参数: name, avatar, intro, type, vectorModel

- **POST /createWithFiles**: 创建知识库并上传文件
  - 一键创建 + 导入文档

- **GET /detail**: 获取知识库详情
- **PUT /update**: 更新知识库配置
- **DELETE /delete**: 删除知识库 (级联删除文档、向量)
- **POST /list**: 知识库列表
- **GET /paths**: 获取路径 (面包屑)
- **POST /exportAll**: 导出全部数据
- **POST /searchTest**: 测试检索功能

#### 文档集合管理 (`/collection`)
- **POST /create**: 创建文档集合
- **POST /create/localFile**: 本地文件上传
- **POST /create/link**: 网页链接导入
- **POST /create/text**: 文本直接导入
- **POST /create/fileId**: 通过文件 ID 导入
- **POST /create/apiCollection**: API 知识库集合
- **POST /create/images**: 图片导入
- **POST /create/template**: 从模板创建
- **POST /create/backup**: 从备份恢复

- **GET /detail**: 集合详情
- **PUT /update**: 更新集合
- **DELETE /delete**: 删除集合
- **POST /list**: 集合列表
- **POST /listV2**: 集合列表 v2 (增强)
- **GET /paths**: 集合路径
- **POST /sync**: 同步外部数据
- **GET /export**: 导出集合数据
- **GET /trainingDetail**: 训练详情

#### 数据块管理 (`/data`)
- **POST /insertData**: 手动插入数据块
- **POST /pushData**: 推送数据 (用于 API 知识库)
- **POST /insertImages**: 插入图片数据
- **GET /detail**: 数据块详情
- **PUT /update**: 更新数据块
- **DELETE /delete**: 删除数据块
- **POST /list**: 数据块列表
- **POST /v2/list**: 数据块列表 v2
- **GET /getQuoteData**: 获取引用数据
- **GET /getPermission**: 获取数据权限

#### 训练队列 (`/training`)
- **GET /getDatasetTrainingQueue**: 获取训练队列
- **GET /getTrainingDataDetail**: 训练数据详情
- **PUT /updateTrainingData**: 更新训练数据
- **DELETE /deleteTrainingData**: 删除训练数据
- **GET /getTrainingError**: 获取训练错误信息
- **POST /rebuildEmbedding**: 重建 Embedding

#### API 知识库 (`/apiDataset`)
- **POST /list**: API 知识库列表
- **POST /getCatalog**: 获取目录结构
- **POST /getPathNames**: 获取路径名称
- **POST /listExistId**: 检查 ID 是否存在

#### 其他
- **POST /folder/create**: 创建文件夹
- **POST /file/getPreviewChunks**: 预览文件分块
- **POST /presignDatasetFilePostUrl**: 文件上传预签名

### 4. AI 能力 API (`/api/core/ai`)

#### 模型管理 (`/model`)
- **POST /list**: 模型列表
- **GET /getMyModels**: 我的模型
- **GET /detail**: 模型详情
- **POST /update**: 更新模型配置
- **POST /updateDefault**: 更新默认模型
- **DELETE /delete**: 删除模型
- **POST /test**: 测试模型可用性
- **GET /getConfigJson**: 获取配置 JSON
- **POST /updateWithJson**: 通过 JSON 更新配置
- **GET /getDefaultConfig**: 获取默认配置

#### AI 工具
- **POST /optimizePrompt**: 优化提示词
- **POST /token**: 计算 Token 数量

#### Agent
- **POST /agent/createQuestionGuide**: 生成问题引导
- **POST /agent/v2/createQuestionGuide**: 生成问题引导 v2

### 5. 工作流 API (`/api/core/workflow`)
- **POST /debug**: 工作流调试
  - 单步执行
  - 返回详细执行日志与变量

- **POST /optimizeCode**: 代码优化 (AI 辅助)

### 6. 插件管理 API (`/api/core/plugin`)

#### 管理员 API (`/admin`)
- **POST /installWithUrl**: 从 URL 安装插件包
- **POST /marketplace/installed**: 已安装插件列表

**插件包管理** (`/pkg`)
- **POST /parse**: 解析插件包
- **POST /confirm**: 确认安装
- **DELETE /delete**: 删除插件包
- **POST /presign**: 获取上传预签名

**工具管理** (`/tool`)
- **POST /create**: 创建工具
- **GET /list**: 工具列表
- **GET /detail**: 工具详情
- **PUT /update**: 更新工具
- **DELETE /delete**: 删除工具
- **PUT /updateOrder**: 更新排序

**工具标签** (`/tool/tag`)
- **POST /create**: 创建标签
- **PUT /update**: 更新标签
- **DELETE /delete**: 删除标签
- **PUT /updateOrder**: 更新标签排序

**工具应用** (`/tool/app`)
- **POST /create**: 创建工具应用
- **GET /systemApps**: 系统工具应用

#### 团队 API (`/team`)
- **POST /list**: 团队插件列表
- **POST /toggleInstall**: 安装/卸载插件
- **GET /toolDetail**: 工具详情

#### 工具标签
- **POST /toolTag/list**: 工具标签列表

### 7. 用户与团队 API (`/api/support/user`)

#### 账户管理 (`/account`)
- **POST /loginByPassword**: 密码登录
- **POST /tokenLogin**: Token 登录
- **POST /preLogin**: 预登录 (检查账号状态)
- **POST /loginout**: 登出
- **PUT /update**: 更新用户信息
- **PUT /updatePasswordByOld**: 修改密码
- **GET /checkPswExpired**: 检查密码是否过期
- **POST /resetExpiredPsw**: 重置过期密码

#### 团队管理 (`/team`)
- **PUT /update**: 更新团队信息

**限制检查** (`/limit`)
- **POST /datasetSizeLimit**: 知识库容量限制
- **POST /exportDatasetLimit**: 导出限制
- **POST /webSyncLimit**: 网页同步限制

**计划管理** (`/plan`)
- **GET /getTeamPlanStatus**: 获取团队计划状态

**第三方集成** (`/thirtdParty`)
- **POST /checkUsage**: 检查第三方使用量

### 8. 钱包与计费 API (`/api/support/wallet`)

**消费管理** (`/usage`)
- **POST /createTrainingUsage**: 创建训练消费记录

### 9. OpenAPI 管理 (`/api/support/openapi`)
- **POST /create**: 创建 API Key
- **PUT /update**: 更新 API Key
- **DELETE /delete**: 删除 API Key
- **POST /list**: API Key 列表
- **GET /health**: 健康检查

### 10. 外链分享 API (`/api/support/outLink`)
- **POST /create**: 创建外链
- **PUT /update**: 更新外链配置
- **DELETE /delete**: 删除外链
- **POST /list**: 外链列表

#### 外链集成
- **POST /feishu/[token]**: 飞书回调
- **POST /dingtalk/[token]**: 钉钉回调
- **POST /wecom/[token]**: 企微回调
- **POST /offiaccount/[token]**: 公众号回调

### 11. MCP 管理 API (`/api/support/mcp`)
- **POST /create**: 创建 MCP 配置
- **PUT /update**: 更新 MCP 配置
- **DELETE /delete**: 删除 MCP 配置
- **POST /list**: MCP 列表

**客户端** (`/client`)
- **POST /getTools**: 获取 MCP 工具列表
- **POST /runTool**: 运行 MCP 工具

**服务端** (`/server`)
- **POST /toolList**: MCP 服务端工具列表
- **POST /toolCall**: MCP 服务端工具调用

### 12. OpenAI 兼容接口

#### v1
- **POST /v1/chat/completions**: 对话接口 (兼容 OpenAI)
  - 支持流式 (stream: true)
  - 支持函数调用 (tools)
  - 支持视觉模型 (vision)

- **POST /v1/audio/transcriptions**: 语音转文字
  - 兼容 Whisper API

#### v2
- **POST /v2/chat/completions**: 增强版对话接口
  - 扩展字段支持
  - 更详细的响应信息

### 13. 通用 API (`/api/common`)

#### 文件管理 (`/file`)
- **POST /presignAvatarPostUrl**: 头像上传预签名
- **POST /presignTempFilePostUrl**: 临时文件上传预签名
- **GET /read/[filename]**: 读取文件

#### 系统信息 (`/system`)
- **GET /getInitData**: 获取初始化数据 (配置、模型列表)
- **POST /unlockTask**: 解锁任务

#### 工具 (`/tools`)
- **POST /urlFetch**: URL 抓取

#### 埋点 (`/tracks`)
- **POST /push**: 推送埋点数据

### 14. 管理员 API (`/api/admin`)
- **POST /initv*****: 数据库初始化/迁移脚本
- **POST /clearInvalidData**: 清理无效数据
- **POST /resetMilvus**: 重置 Milvus

#### 应用注册 (`/support/appRegistration`)
- **POST /create**: 创建应用注册

### 15. 系统级 API (`/api/system`)
- **GET /file/[jwt]**: 通过 JWT 访问文件
- **GET /img/[...id]**: 图片访问
- **GET /plugin/[...path]**: 插件资源代理

### 16. 代理 API
- **ANY /marketplace/[...path]**: 市场 API 代理
- **ANY /plugin/[...pluginRequestPath]**: 插件 API 代理
- **ANY /lafApi/[...path]**: Laf API 代理
- **ANY /proApi/[...path]**: Pro API 代理
- **ANY /aiproxy/[...path]**: AI Proxy 代理

## 认证与权限

### 认证方式
1. **Cookie Session**: 前端登录后通过 Cookie 携带 Session
2. **JWT Token**: API 调用通过 Header 携带 `Authorization: Bearer <token>`
3. **OpenAPI Key**: `/api/v1|v2` 接口通过 `apikey` 参数或 Header

### 权限校验
- **authCert**: 基础认证 (验证用户身份)
- **authApp**: 应用权限校验 (读/写/所有者)
- **authDataset**: 知识库权限校验
- **authUserPer**: 用户权限校验 (Team 级权限)
- **authTeamSpaceToken**: 团队空间 Token 校验

### 权限级别
- **Read (读)**: 可查看
- **Write (写)**: 可编辑
- **Owner (所有者)**: 可删除、转移

## 限流策略

### IP 限流
```typescript
useIPFrequencyLimit({ 
  id: 'export-chat-logs',  // 唯一标识
  seconds: 60,             // 时间窗口
  limit: 10,               // 次数限制
  force: true              // 强制限制
})
```

### 用户限流
- 按 Team 限制并发请求
- 按 API Key 限制调用频率
- 计费限制 (余额不足拒绝)

## 错误处理

### 统一错误码
```typescript
CommonErrEnum.unAuthorization       // 401 未授权
CommonErrEnum.forbidden             // 403 无权限
CommonErrEnum.fileNotFound          // 404 文件不存在
CommonErrEnum.invalidParams         // 400 参数错误
CommonErrEnum.serverError           // 500 服务器错误
```

### 错误响应格式
```json
{
  "code": 403,
  "statusText": "Forbidden",
  "message": "无权限访问该资源",
  "data": null
}
```

## 日志与审计

### 请求日志
- 记录所有 API 请求
- 包含: 用户、路径、参数、耗时、IP

### 审计日志 (AuditEventEnum)
关键操作记录：
- CREATE_APP / UPDATE_APP / DELETE_APP
- CREATE_DATASET / UPDATE_DATASET / DELETE_DATASET
- EXPORT_CHAT_LOGS
- CREATE_OPENAPI_KEY / DELETE_OPENAPI_KEY
- ...

## 性能优化

### 缓存策略
- React Query 缓存前端请求
- Redis 缓存热点数据 (模型列表、配置)

### 数据库优化
- 索引优化 (MongoDB)
- 分页查询 (避免一次性加载大量数据)
- 聚合查询优化

### 响应优化
- 流式响应 (对话接口)
- 数据压缩 (gzip)
- 按需返回字段

## 相关文档
- [前端展示层](./01-frontend-layer.md)
- [业务服务层](./03-service-layer.md)
- [共享层](./04-shared-layer.md)
