# 业务服务层 (Service Layer)

## 概述
业务服务层是 FastGPT 的核心逻辑层，位于 `packages/service/` 目录，负责实现应用、知识库、对话、工作流、插件等核心业务逻辑，以及数据模型操作、第三方服务调用和任务处理。

## 技术栈
- **语言**: TypeScript
- **ORM**: Mongoose (MongoDB)
- **校验**: Zod Schema
- **任务队列**: BullMQ + Redis
- **向量数据库**: PGVector / Milvus / OceanBase (抽象层)
- **对象存储**: MinIO / S3
- **监控**: OpenTelemetry + Pino/Winston

## 核心架构

### 目录结构
```
packages/service/
├── core/                  # 核心业务逻辑
│   ├── ai/               # AI 能力封装
│   ├── app/              # 应用管理
│   ├── chat/             # 对话逻辑
│   ├── dataset/          # 知识库管理
│   ├── plugin/           # 插件系统
│   └── workflow/         # 工作流引擎
├── common/               # 通用基础设施
│   ├── mongo/           # MongoDB 连接与中间件
│   ├── redis/           # Redis 连接管理
│   ├── vectorDB/        # 向量数据库抽象层
│   ├── bullmq/          # 任务队列管理
│   ├── file/            # 文件处理
│   ├── s3/              # 对象存储
│   ├── otel/            # OpenTelemetry 日志
│   ├── cache/           # 缓存管理
│   ├── api/             # API 调用工具
│   ├── security/        # 安全检查
│   └── system/          # 系统工具
├── worker/               # Worker 线程任务
│   ├── readFile/        # 文件读取
│   ├── text2Chunks/     # 文本分块
│   ├── htmlStr2Md/      # HTML 转 Markdown
│   └── countGptMessagesTokens/ # Token 计数
├── support/              # 支持服务
│   ├── user/            # 用户与团队
│   ├── permission/      # 权限管理
│   ├── wallet/          # 钱包与计费
│   ├── openapi/         # OpenAPI 管理
│   └── outLink/         # 外链分享
├── thirdProvider/        # 第三方服务集成
│   ├── fastgptPlugin/   # FastGPT Plugin 运行时
│   └── doc2x/           # 文档解析服务
└── type/                 # 类型定义
```

## 核心模块详解

### 1. 应用管理 (`core/app`)

#### 数据模型 (Schema)
**MongoApp** - 应用表
```typescript
{
  teamId: ObjectId,           // 所属团队
  tmbId: ObjectId,            // 创建者
  parentId: string,           // 父文件夹 ID
  name: string,               // 应用名称
  avatar: string,             // 头像
  intro: string,              // 简介
  type: AppTypeEnum,          // 应用类型 (simple/advanced/plugin/folder/httpPlugin)
  teamTags: string[],         // 团队标签
  version: string,            // 版本号
  modules: FlowModuleItemType[], // 工作流节点 (已弃用,保留兼容)
  edges: Edge[],              // 工作流边 (已弃用,保留兼容)
  chatConfig: AppChatConfigType, // 对话配置
  inheritPermission: boolean, // 是否继承权限
  pluginData: {...},          // 插件数据
  isOwner: boolean,           // 是否所有者 (虚拟字段)
  permission: Permission      // 权限 (虚拟字段)
}
```

**MongoAppVersion** - 应用版本表
```typescript
{
  appId: ObjectId,
  versionName: string,
  nodes: FlowNodeItemType[],  // 工作流节点
  edges: Edge[],              // 工作流边
  chatConfig: AppChatConfigType,
  isPublish: boolean,         // 是否发布
  tmbId: ObjectId,            // 发布者
  time: Date                  // 发布时间
}
```

#### 核心服务
**controller.ts** - 应用控制器
- `getAppById`: 获取应用详情
- `getAppSimpleList`: 获取简单应用列表
- `getAppsByFolderPathNames`: 根据路径名获取应用
- `deleteAppAndRelatedData`: 删除应用及相关数据 (级联删除)
- `getMyApps`: 获取我的应用列表 (支持搜索、分页、类型过滤)

**version/controller.ts** - 版本控制器
- `getAppVersion`: 获取指定版本
- `getLatestVersion`: 获取最新版本
- `publishApp`: 发布新版本
- `getAppVersionList`: 获取版本列表

**plugin/controller.ts** - 插件控制器
- HTTP Plugin 管理
- MCP Plugin 管理
- Plugin 工具调用

### 2. 对话管理 (`core/chat`)

#### 数据模型
**MongoChatConversation** - 对话会话表
```typescript
{
  teamId: ObjectId,
  tmbId: ObjectId,
  appId: ObjectId,
  chatId: string,             // 对话 ID
  title: string,              // 对话标题
  customTitle: string,        // 自定义标题
  top: boolean,               // 是否置顶
  variables: Record<string, any>, // 全局变量
  metadata: Record<string, any>,  // 元数据
  updateTime: Date
}
```

**MongoChatItem** - 对话消息表
```typescript
{
  chatId: string,
  dataId: string,             // 消息 ID
  appId: ObjectId,
  teamId: ObjectId,
  tmbId: ObjectId,
  nodeOutputs: ChatItemType[],// 节点输出
  interactive: {...},         // 交互信息
  metadata: {...}             // 元数据 (引用、工具调用)
}
```

**MongoChatInputGuide** - 输入引导表
```typescript
{
  appId: ObjectId,
  teamId: ObjectId,
  text: string,               // 引导文本
  customPrompt: string,       // 自定义提示词
  datasetIds: ObjectId[]      // 关联知识库
}
```

#### 核心服务
**controller.ts** - 对话控制器
- `getChatHistories`: 获取对话历史列表
- `getConversationById`: 获取对话详情
- `createChatConversation`: 创建新对话
- `updateChatConversation`: 更新对话
- `deleteChatConversation`: 删除对话

**utils.ts** - 工具函数
- `addPreviewUrlToChatItems`: 为消息添加预览 URL
- `filterChatMessageByConfig`: 根据配置过滤消息
- `getChatItems`: 获取对话消息

**feedback/controller.ts** - 反馈控制器
- `createChatFeedback`: 创建反馈
- `updateChatFeedback`: 更新反馈
- `getChatFeedbacks`: 获取反馈列表

### 3. 知识库管理 (`core/dataset`)

#### 数据模型
**MongoDataset** - 知识库表
```typescript
{
  teamId: ObjectId,
  tmbId: ObjectId,
  parentId: string,
  avatar: string,
  name: string,
  intro: string,
  type: DatasetTypeEnum,      // dataset / folder / websiteDataset / apiDataset
  status: DatasetStatusEnum,
  vectorModel: string,         // Embedding 模型
  agentModel: string,          // Agent 模型
  inheritPermission: boolean,
  websiteConfig: {...},        // 网站配置
  apiDatasetConfig: {...},     // API 配置
  externalFileId: string,
  externalFileUrl: string
}
```

**MongoDatasetCollection** - 文档集合表
```typescript
{
  teamId: ObjectId,
  tmbId: ObjectId,
  datasetId: ObjectId,
  parentId: string,
  name: string,
  type: DatasetCollectionTypeEnum, // file / link / virtual / folder
  trainingType: DatasetCollectionTrainingTypeEnum, // manual / qa / auto / segment
  chunkSize: number,
  chunkSplitter: string,
  qaPrompt: string,
  fileId: string,
  rawLink: string,
  externalFileId: string,
  externalFileUrl: string,
  hashRawText: string,
  metadata: {...}
}
```

**MongoDatasetData** - 数据块表
```typescript
{
  teamId: ObjectId,
  tmbId: ObjectId,
  datasetId: ObjectId,
  collectionId: ObjectId,
  chunkIndex: number,
  q: string,                  // 问题 (用于检索)
  a: string,                  // 答案 (实际内容)
  fullTextToken: string,      // 全文分词 Token
  indexes: DatasetDataIndexItemType[], // 向量索引
  updateTime: Date
}
```

**MongoDatasetTraining** - 训练队列表
```typescript
{
  teamId: ObjectId,
  tmbId: ObjectId,
  datasetId: ObjectId,
  collectionId: ObjectId,
  billId: string,
  mode: DatasetTrainingModeEnum,
  prompt: string,
  q: string,
  a: string,
  chunkIndex: number,
  weight: number,
  model: string,
  lockTime: Date,
  retryTimes: number
}
```

#### 核心服务
**controller.ts** - 知识库控制器
- `getDatasetById`: 获取知识库详情
- `getDatasets`: 获取知识库列表
- `createDataset`: 创建知识库
- `updateDataset`: 更新知识库
- `deleteDatasetById`: 删除知识库

**collection/controller.ts** - 文档集合控制器
- `getCollectionById`: 获取集合详情
- `getCollections`: 获取集合列表
- `createCollection`: 创建集合
- `updateCollection`: 更新集合
- `deleteCollectionById`: 删除集合
- `syncCollection`: 同步集合 (外部数据源)

**data/controller.ts** - 数据块控制器
- `insertData2Dataset`: 插入数据块
- `updateData`: 更新数据块
- `deleteDataById`: 删除数据块
- `getDatasetDataList`: 获取数据块列表
- `searchDatasetData`: 检索数据块 (向量 + 全文)

**training/controller.ts** - 训练控制器
- `createTrainingBill`: 创建训练账单
- `pushDataToTrainingQueue`: 推送数据到训练队列
- `generateVector`: 生成向量 (Embedding)
- `generateQA`: 生成问答对 (AI 辅助)

### 4. AI 能力封装 (`core/ai`)

#### 核心服务
**ai/service.ts** - AI 服务
- `chatCompletion`: 对话补全 (支持流式)
- `embedding`: 文本向量化
- `rerank`: 重排序
- `textToSpeech`: 文本转语音
- `speechToText`: 语音转文字

**model/controller.ts** - 模型控制器
- `getMyModels`: 获取可用模型列表
- `getModelById`: 获取模型详情
- `checkModelAvailable`: 检查模型是否可用

**config/controller.ts** - 配置控制器
- `getAIApi`: 获取 AI API 配置
- `updateAIApi`: 更新 AI API 配置

#### 模型适配器
- OpenAI Adapter
- Azure Adapter
- Custom Model Adapter
- One API Adapter

### 5. 工作流引擎 (`core/workflow`)

#### 核心组件
**dispatch/index.ts** - 工作流调度器
```typescript
// 主入口
export async function dispatchWorkFlow(props: ChatDispatchProps) {
  // 1. 初始化运行时环境
  // 2. 解析工作流 (nodes + edges)
  // 3. 执行节点调度
  // 4. 处理变量传递
  // 5. 返回执行结果
}
```

**节点调度逻辑**
1. 按拓扑排序执行节点
2. 检查节点运行状态 (是否跳过)
3. 获取节点输入变量 (从前置节点输出)
4. 调用节点 handler
5. 存储节点输出
6. 根据边继续下一个节点

**节点类型** (`dispatch/`)
- `entry/` - 工作流入口节点
- `aiChat/` - AI 对话节点
- `datasetSearch/` - 知识库检索节点
- `classifyQuestion/` - 问题分类节点
- `contentExtract/` - 内容提取节点
- `httpRequest/` - HTTP 请求节点
- `runPlugin/` - 插件运行节点
- `tools/` - 工具节点集合
  - `http468.ts` - HTTP 工具 (4.6.8 版本)
  - `stopTool.ts` - 停止工具
  - `answerNode.ts` - 回答节点
  - `variable.ts` - 变量赋值
  - `customFeedback.ts` - 自定义反馈
  - `codeSandbox.ts` - 代码沙盒
  - `mcpTool.ts` - MCP 工具

**变量系统** (`runtime/utils.ts`)
- `getReferenceVariableValue`: 获取引用变量值
- `replaceEditorVariable`: 替换编辑器变量
- `valueTypeFormat`: 变量类型转换

### 6. 插件系统 (`core/plugin`)

#### 数据模型
**MongoPlugin** - 插件表
```typescript
{
  teamId: ObjectId,
  name: string,
  avatar: string,
  intro: string,
  updateTime: Date,
  isInstalled: boolean,
  currentCost: number,
  metadata: {
    apiSchemaStr: string,
    customHeaders: string
  }
}
```

#### 核心服务
**controller.ts** - 插件控制器
- `getPluginById`: 获取插件详情
- `getTeamPlugins`: 获取团队插件列表
- `installPlugin`: 安装插件
- `uninstallPlugin`: 卸载插件
- `runPlugin`: 运行插件

**parse/controller.ts** - 插件解析
- `parsePluginPackage`: 解析插件包
- `validatePluginManifest`: 校验插件清单

### 7. Worker 任务处理 (`worker/`)

#### 核心 Worker
**readFile/** - 文件读取
- 支持格式: PDF, DOCX, XLSX, CSV, TXT, MD, HTML
- 提取文本内容
- 处理编码

**text2Chunks/** - 文本分块
- 递归分块策略
- 按 Token 限制
- 支持自定义分隔符

**htmlStr2Md/** - HTML 转 Markdown
- 基于 Turndown
- 清理无效标签
- 保留图片链接

**countGptMessagesTokens/** - Token 计数
- 基于 tiktoken
- 支持多模型

### 8. 通用基础设施 (`common/`)

#### MongoDB 管理 (`mongo/`)
**index.ts** - 连接管理
```typescript
export const connectionMongo = (() => {
  if (!global.mongodb) {
    global.mongodb = new Mongoose();
  }
  return global.mongodb;
})();

// 主从分离
export const connectionLogMongo = (() => {
  if (!global.mongodbLog) {
    global.mongodbLog = new Mongoose();
  }
  return global.mongodbLog;
})();
```

**中间件**
- 查询性能监控
- 慢查询日志
- 自动添加 teamId (隔离)

#### 向量数据库抽象层 (`vectorDB/`)
**controller.ts** - 统一接口
```typescript
export abstract class VectorDBAdapter {
  abstract createCollection(params): Promise<void>;
  abstract insertData(params): Promise<void>;
  abstract updateData(params): Promise<void>;
  abstract deleteData(params): Promise<void>;
  abstract searchData(params): Promise<SearchResult[]>;
}
```

**实现**
- `pg/` - PostgreSQL + PGVector
- `milvus/` - Milvus
- `oceanbase/` - OceanBase Vector

#### Redis 管理 (`redis/`)
**index.ts** - 连接管理
```typescript
export const newQueueRedisConnection = () => {
  const redis = new Redis(REDIS_URL);
  return redis;
};

export const newWorkerRedisConnection = () => {
  const redis = new Redis(REDIS_URL, {
    maxRetriesPerRequest: null
  });
  return redis;
};

export const getGlobalRedisConnection = () => {
  // 单例模式
};
```

#### 任务队列 (`bullmq/`)
**队列定义**
- `trainingQueue` - 训练队列
- `fileProcessQueue` - 文件处理队列
- `syncQueue` - 数据同步队列

**Worker 处理**
```typescript
// trainingQueue
worker.process(async (job) => {
  const { datasetId, collectionId, data } = job.data;
  // 1. 生成 Embedding
  // 2. 存储向量
  // 3. 更新状态
});
```

#### 文件处理 (`file/`)
**controller.ts**
- `uploadFile`: 上传文件
- `readFile`: 读取文件内容
- `parseFile`: 解析文件 (PDF/DOCX/XLSX)
- `splitText`: 文本分块

**gridfs/controller.ts** - GridFS 存储
- 大文件分块存储
- 流式读写

#### 对象存储 (`s3/`)
**controller.ts** - MinIO/S3 操作
- `uploadFile`: 上传文件
- `getFile`: 获取文件
- `deleteFile`: 删除文件
- `generatePresignedUrl`: 生成预签名 URL

**sources/** - 分类存储
- `avatar/` - 头像
- `chat/` - 对话文件
- `dataset/` - 知识库文件
- `temp/` - 临时文件

#### 监控日志 (`otel/`)
**winston/index.ts** - Winston 日志
- OpenTelemetry Transport
- 日志级别: info, warn, error
- 日志导出到 OTLP

**pino/index.ts** - Pino 日志
- 高性能日志
- 结构化日志
- OpenTelemetry 集成

### 9. 支持服务 (`support/`)

#### 用户与团队 (`user/`)
**controller.ts**
- `getUserByToken`: 通过 Token 获取用户
- `updateUser`: 更新用户信息
- `getTeamMembers`: 获取团队成员

**team/controller.ts**
- `createTeam`: 创建团队
- `updateTeam`: 更新团队
- `inviteMember`: 邀请成员
- `removeMember`: 移除成员

**audit/controller.ts** - 审计日志
- `addAuditLog`: 添加审计日志
- `getAuditLogs`: 获取审计日志

#### 权限管理 (`permission/`)
**auth/common.ts** - 通用认证
- `authCert`: 基础认证
- `authUserPer`: 用户权限校验
- `authTeamPer`: 团队权限校验

**app/auth.ts** - 应用权限
- `authApp`: 应用权限校验 (读/写/所有者)
- `authAppByTmbId`: 通过成员 ID 校验

**dataset/auth.ts** - 知识库权限
- `authDataset`: 知识库权限校验

**controller.ts** - 权限控制器
- `getResourcePermission`: 获取资源权限
- `updateResourcePermission`: 更新资源权限
- `removeResourcePermission`: 移除资源权限

#### 钱包与计费 (`wallet/`)
**controller.ts**
- `createBill`: 创建账单
- `updateBalance`: 更新余额
- `getWalletBalance`: 获取钱包余额
- `getUsageRecords`: 获取消费记录

**usage/controller.ts** - 使用量统计
- `createChatUsageRecord`: 创建对话消费记录
- `pushChatItemUsage`: 推送消息消费
- `updateUsageRecord`: 更新消费记录

### 10. 第三方服务集成 (`thirdProvider/`)

#### FastGPT Plugin 运行时
**fastgptPlugin/controller.ts**
- `runPluginCode`: 运行插件代码
- `validatePluginSandbox`: 验证沙盒安全

#### 文档解析服务
**doc2x/controller.ts**
- `parseDocument`: 解析文档 (PDF/DOCX)
- `extractImages`: 提取图片

## 数据流示例

### 对话流程
```
1. API 调用 chatCompletion
2. 加载应用配置与工作流
3. dispatchWorkFlow 启动
4. 执行节点:
   - 工作流入口 → 知识库检索 → AI 对话 → 回答输出
5. 每个节点:
   - 获取输入变量
   - 调用节点 handler
   - 返回输出变量
6. 流式返回结果 (SSE)
7. 保存对话记录
8. 记录消费
```

### 知识库训练流程
```
1. 上传文件 → MinIO/S3
2. 创建文档集合
3. pushDataToTrainingQueue → BullMQ
4. Worker 处理:
   - readFile (Worker 线程读取文件)
   - text2Chunks (分块)
   - generateVector (生成 Embedding)
5. 存储:
   - 数据块 → MongoDB
   - 向量 → VectorDB (PGVector/Milvus)
6. 更新训练状态
```

## 性能优化

### 数据库优化
- 索引优化 (MongoDB)
- 聚合查询优化
- 分页查询
- 主从读写分离

### 缓存策略
- Redis 缓存热点数据
- 应用配置缓存
- 模型列表缓存

### 并发控制
- BullMQ 并发数控制
- Worker 线程池
- 限流 (Rate Limiting)

## 相关文档
- [API 路由层](./02-api-routes-layer.md)
- [共享层](./04-shared-layer.md)
- [数据持久化层](./05-data-persistence-layer.md)
- [工作流引擎](./07-workflow-engine.md)
