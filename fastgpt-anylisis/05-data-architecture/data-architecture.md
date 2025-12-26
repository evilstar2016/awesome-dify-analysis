# FastGPT 数据架构文档

## 概述

FastGPT 采用多层次、多数据库的数据架构设计，充分利用不同数据库的特性满足不同的业务场景需求。整体架构包括：
- **MongoDB**: 主数据库，存储业务数据（应用、知识库、对话、用户等）
- **向量数据库**: PGVector / Milvus / OceanBase，用于向量检索
- **Redis**: 缓存、任务队列、会话管理
- **MinIO/S3**: 对象存储，存储文件、图片等资源

---

## 数据库架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         FastGPT 数据层                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   前端层     │  │   API路由    │  │   服务层     │         │
│  │   (Web UI)   │──│   (Next.js)  │──│  (Business)  │         │
│  └──────────────┘  └──────────────┘  └──────┬───────┘         │
│                                              │                  │
├──────────────────────────────────────────────┼──────────────────┤
│                        数据访问层            │                  │
│                                              │                  │
│  ┌───────────────────────────────────────────┴─────────────┐   │
│  │                    数据持久化层                         │   │
│  │                                                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐ │   │
│  │  │ MongoDB  │  │ VectorDB │  │ Redis  │  │ MinIO/S3 │ │   │
│  │  │          │  │          │  │        │  │          │ │   │
│  │  │ 主数据   │  │ 向量检索 │  │ 缓存   │  │ 对象存储 │ │   │
│  │  │          │  │          │  │ 队列   │  │          │ │   │
│  │  └──────────┘  └──────────┘  └────────┘  └──────────┘ │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. MongoDB 数据库设计

### 1.1 连接管理

FastGPT 使用 Mongoose 进行 MongoDB 连接管理，支持读写分离：

```typescript
// 主连接（读写）
export const connectionMongo = new Mongoose();
connectionMongo.connect(MONGO_URL);

// 日志连接（写）
export const connectionLogMongo = new Mongoose();
connectionLogMongo.connect(MONGO_LOG_URL);
```

**连接配置参数：**
- maxConnecting: 30 (最大连接数)
- maxPoolSize: 30 (连接池大小)
- minPoolSize: 20 (最小连接数)
- connectTimeoutMS: 60000 (60秒)
- socketTimeoutMS: 60000 (60秒)
- maxIdleTimeMS: 300000 (5分钟)
- retryWrites: true (重试写入)
- retryReads: true (重试读取)

### 1.2 核心数据模型

#### 1.2.1 用户与团队模型

**users（用户表）**
```typescript
{
  _id: ObjectId,
  status: String,              // 用户状态
  username: String,            // 用户名（唯一）
  password: String,            // 密码（加密）
  passwordUpdateTime: Date,
  createTime: Date,
  timezone: String,            // 时区（默认：Asia/Shanghai）
  language: String,            // 语言（默认：zh_CN）
  lastLoginTmbId: ObjectId,    // 最后登录团队成员ID
  inviterId: ObjectId,         // 邀请人ID
  phonePrefix: Number,
  contact: String,
  openaiAccount: {
    key: String,
    baseUrl: String
  }
}

索引:
- { username: 1 } (唯一索引)
- { createTime: -1 }
```

**teams（团队表）**
```typescript
{
  _id: ObjectId,
  name: String,                // 团队名称
  ownerId: ObjectId,           // 所有者ID (ref: users)
  avatar: String,              // 团队头像
  createTime: Date,
  balance: Number,             // 账户余额
  teamDomain: String,          // 团队域名
  limit: {
    lastExportDatasetTime: Date,
    lastWebsiteSyncTime: Date
  },
  lafAccount: Object,          // Laf账户配置
  openaiAccount: Object,       // OpenAI账户配置
  externalWorkflowVariables: Object,
  notificationAccount: String
}

索引:
- { name: 1 }
- { ownerId: 1 }
```

**team_members（团队成员表）**
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,            // ref: teams
  userId: ObjectId,            // ref: users
  role: String,                // 角色: owner/admin/member
  status: String,              // 状态: active/inactive/pending
  createTime: Date,
  defaultTeam: Boolean         // 是否默认团队
}

索引:
- { teamId: 1, userId: 1 } (唯一索引)
- { userId: 1 }
```

#### 1.2.2 应用模型

**apps（应用表）**
```typescript
{
  _id: ObjectId,
  parentId: ObjectId,          // 父文件夹ID (ref: apps)
  teamId: ObjectId,            // ref: teams
  tmbId: ObjectId,             // 创建者 (ref: team_members)
  name: String,                // 应用名称
  type: String,                // 应用类型: workflow/simple/plugin
  version: String,             // 版本: v1/v2
  avatar: String,              // 应用图标
  intro: String,               // 应用简介
  templateId: String,          // 模板ID
  updateTime: Date,
  
  // Workflow数据
  modules: Array,              // 工作流节点
  edges: Array,                // 工作流连线
  chatConfig: {
    welcomeText: String,
    variables: Array,
    questionGuide: Object,
    ttsConfig: Object,
    whisperConfig: Object,
    scheduledTriggerConfig: Object,
    chatInputGuide: Object,
    fileSelectConfig: Object,
    instruction: String,
    autoExecute: Object
  },
  
  // Plugin工具配置
  pluginData: {
    nodeVersion: String,
    pluginUniId: String,
    apiSchemaStr: String,
    customHeaders: String
  },
  
  scheduledTriggerConfig: {
    cronString: String,
    timezone: String,
    defaultPrompt: String
  }
}

索引:
- { teamId: 1, parentId: 1 }
- { teamId: 1, type: 1 }
- { teamId: 1, name: 'text' } (全文索引)
- { templateId: 1 }
```

**app_versions（应用版本表）**
```typescript
{
  _id: ObjectId,
  appId: ObjectId,             // ref: apps
  versionName: String,         // 版本名称
  nodes: Array,                // 工作流节点
  edges: Array,                // 工作流连线
  chatConfig: Object,          // 聊天配置
  isPublish: Boolean,          // 是否已发布
  tmbId: ObjectId,             // 创建者
  time: Date                   // 版本时间
}

索引:
- { appId: 1, time: -1 }
- { appId: 1, isPublish: 1 }
```

#### 1.2.3 知识库模型

**datasets（知识库表）**
```typescript
{
  _id: ObjectId,
  parentId: ObjectId,          // 父文件夹ID (ref: datasets)
  teamId: ObjectId,            // ref: teams
  tmbId: ObjectId,             // 创建者 (ref: team_members)
  type: String,                // 类型: dataset/folder/websiteDataset/apiDataset
  avatar: String,              // 知识库图标
  name: String,                // 知识库名称
  updateTime: Date,
  intro: String,               // 简介
  
  // 向量配置
  vectorModel: String,         // 向量模型（默认：text-embedding-3-small）
  agentModel: String,          // Agent模型
  
  // 数据处理配置
  trainingType: String,        // 训练类型
  chunkTriggerType: String,    // 分块触发类型
  chunkTriggerMinSize: Number,
  dataEnhanceCollectionName: Boolean,
  imageIndex: Boolean,
  autoIndexes: Boolean,
  indexPrefixTitle: Boolean,
  chunkSettingMode: String,
  chunkSplitMode: String,
  paragraphChunkAIMode: String,
  paragraphChunkDeep: Number,
  paragraphChunkMinSize: Number,
  chunkSize: Number,
  chunkSplitter: String,
  indexSize: Number,
  qaPrompt: String,
  
  // 特殊类型配置
  websiteConfig: Object,       // 网站爬虫配置
  apiDatasetConfig: Object     // API数据源配置
}

索引:
- { teamId: 1, parentId: 1 }
- { teamId: 1, type: 1 }
- { teamId: 1, name: 'text' } (全文索引)
```

**dataset_collections（文档集合表）**
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,            // ref: teams
  tmbId: ObjectId,             // 创建者
  datasetId: ObjectId,         // ref: datasets
  parentId: ObjectId,          // 父集合ID
  name: String,                // 集合名称
  type: String,                // 类型: file/link/text/apiImport
  trainingType: String,        // 训练类型
  chunkSize: Number,           // 分块大小
  chunkSplitter: String,       // 分块分隔符
  fileId: String,              // 文件ID（MinIO/S3）
  rawLink: String,             // 原始链接
  hashRawText: String,         // 原始文本哈希（去重）
  createTime: Date,
  updateTime: Date,
  trainingAmount: Number,      // 训练数量
  canWrite: Boolean            // 是否可写
}

索引:
- { teamId: 1, datasetId: 1 }
- { teamId: 1, updateTime: -1 }
- { hashRawText: 1 }
- { datasetId: 1, parentId: 1 }
```

**dataset_datas（数据块表）**
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,            // ref: teams
  tmbId: ObjectId,             // 创建者
  datasetId: ObjectId,         // ref: datasets
  collectionId: ObjectId,      // ref: dataset_collections
  chunkIndex: Number,          // 块索引
  q: String,                   // 问题（用于检索）
  a: String,                   // 答案（实际内容）
  fullTextToken: String,       // 全文分词（用于全文检索）
  indexes: [{                  // 向量索引
    dataId: String,            // 向量数据库中的ID
    defaultIndex: Boolean,     // 是否默认索引
    type: String               // 索引类型: vector/fullText
  }],
  updateTime: Date
}

索引:
- { teamId: 1, datasetId: 1, collectionId: 1 }
- { teamId: 1, datasetId: 1, updateTime: -1 }
- { 'indexes.dataId': 1 }
- { fullTextToken: 'text', q: 'text' } (全文索引)
```

**dataset_trainings（训练队列表）**
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,            // ref: teams
  tmbId: ObjectId,             // 创建者
  datasetId: ObjectId,         // ref: datasets
  collectionId: ObjectId,      // ref: dataset_collections
  billId: String,              // 账单ID
  mode: String,                // 训练模式: chunk/qa/auto
  prompt: String,              // 提示词
  q: String,                   // 问题
  a: String,                   // 答案
  chunkIndex: Number,          // 块索引
  weight: Number,              // 权重
  model: String,               // 模型
  lockTime: Date,              // 锁定时间（任务处理锁）
  retryTimes: Number           // 重试次数
}

索引:
- { teamId: 1, lockTime: 1 }
- { billId: 1 }
- { datasetId: 1, collectionId: 1 }
```

#### 1.2.4 对话模型

**chats（对话会话表）**
```typescript
{
  _id: ObjectId,
  chatId: String,              // 对话ID（唯一标识）
  teamId: ObjectId,            // ref: teams
  tmbId: ObjectId,             // ref: team_members
  appId: ObjectId,             // ref: apps
  createTime: Date,
  updateTime: Date,
  title: String,               // 对话标题
  customTitle: String,         // 自定义标题
  top: Boolean,                // 是否置顶
  source: String,              // 来源: online/share/api
  sourceName: String,          // 来源名称
  shareId: String,             // 分享ID
  outLinkUid: String,          // 外链用户ID
  
  // 变量配置
  variableList: Array,
  welcomeText: String,
  variables: Object,           // 变量值
  pluginInputs: Array,
  metadata: Object             // 元数据
}

索引:
- { chatId: 1 } (唯一索引)
- { tmbId: 1, appId: 1, top: -1, updateTime: -1 }
- { appId: 1, chatId: 1 }
- { appId: 1, tmbId: 1, outLinkUid: 1 }
```

**chat_items（对话消息表）**
```typescript
{
  _id: ObjectId,
  chatId: String,              // ref: chats.chatId
  dataId: String,              // 消息ID（唯一标识）
  appId: ObjectId,             // ref: apps
  teamId: ObjectId,            // ref: teams
  tmbId: ObjectId,             // ref: team_members
  nodeOutputs: Array,          // 节点输出数据
  interactive: Object,         // 交互数据
  metadata: Object,            // 元数据
  createTime: Date
}

索引:
- { chatId: 1, dataId: 1 }
- { appId: 1, createTime: -1 }
- { teamId: 1, createTime: -1 }
```

#### 1.2.5 插件与工具模型

**plugins（插件表）**
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,            // ref: teams
  tmbId: ObjectId,             // 创建者
  name: String,                // 插件名称
  avatar: String,
  intro: String,
  type: String,                // 类型: custom/system
  version: String,
  modules: Array,              // 插件节点
  edges: Array,                // 插件连线
  inputConfig: Array,          // 输入配置
  outputConfig: Array          // 输出配置
}
```

#### 1.2.6 系统配置模型

**system_configs（系统配置表）**
```typescript
{
  _id: ObjectId,
  key: String,                 // 配置键（唯一）
  value: Mixed,                // 配置值
  updateTime: Date
}

索引:
- { key: 1 } (唯一索引)
```

**system_logs（系统日志表）**
```typescript
{
  _id: ObjectId,
  level: String,               // 日志级别: info/warn/error
  message: String,             // 日志消息
  context: Object,             // 上下文
  createTime: Date
}

索引:
- { createTime: -1 }
- { level: 1, createTime: -1 }
```

### 1.3 数据关系图

```
users ───┬─── teams (ownerId)
         │
         └─── team_members (userId)
                │
                ├─── apps (tmbId)
                │     │
                │     ├─── app_versions (appId)
                │     │
                │     └─── chats (appId)
                │           │
                │           └─── chat_items (chatId)
                │
                └─── datasets (tmbId)
                      │
                      ├─── dataset_collections (datasetId)
                      │     │
                      │     └─── dataset_datas (collectionId)
                      │           │
                      │           └─── [向量数据库]
                      │
                      └─── dataset_trainings (datasetId)
```

---

## 2. 向量数据库设计

FastGPT 支持三种向量数据库：PGVector（PostgreSQL）、Milvus、OceanBase，通过统一的抽象接口进行访问。

### 2.1 PGVector（PostgreSQL）

**表结构：**
```sql
CREATE TABLE dataset_vector (
  id SERIAL PRIMARY KEY,
  team_id VARCHAR(24),
  dataset_id VARCHAR(24),
  collection_id VARCHAR(24),
  data_id VARCHAR(24),
  vector vector(1536),        -- 向量维度根据模型调整
  create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 索引
CREATE INDEX idx_team_id ON dataset_vector(team_id);
CREATE INDEX idx_dataset_id ON dataset_vector(dataset_id);
CREATE INDEX idx_data_id ON dataset_vector(data_id);
CREATE INDEX idx_vector_hnsw ON dataset_vector 
  USING hnsw (vector vector_cosine_ops);
```

**检索操作：**
```sql
SELECT data_id, 1 - (vector <=> $1) as score
FROM dataset_vector
WHERE team_id = $2 AND dataset_id = $3
ORDER BY vector <=> $1
LIMIT $4
```

### 2.2 Milvus

**Collection 结构：**
```typescript
{
  collectionName: `ds_${datasetId}`,
  fields: [
    { name: 'id', dataType: DataType.Int64, isPrimary: true, autoID: true },
    { name: 'team_id', dataType: DataType.VarChar, maxLength: 24 },
    { name: 'data_id', dataType: DataType.VarChar, maxLength: 24 },
    { name: 'vector', dataType: DataType.FloatVector, dim: 1536 }
  ]
}
```

**索引配置：**
```typescript
{
  index_type: 'HNSW',
  metric_type: 'COSINE',
  params: { M: 16, efConstruction: 256 }
}
```

### 2.3 向量数据流程

```
1. 文档上传
   ↓
2. 文本分块（dataset_collections）
   ↓
3. 数据块入库（dataset_datas）
   ↓
4. 向量化任务（dataset_trainings）
   ↓
5. 生成向量（Embedding API）
   ↓
6. 向量入库（VectorDB）
   ↓
7. 更新索引引用（dataset_datas.indexes）
```

---

## 3. Redis 数据设计

### 3.1 缓存 Key 规范

```
# 系统配置
fastgpt:cache:system_config               # TTL: 1小时

# 模型配置
fastgpt:cache:models                      # TTL: 1小时
fastgpt:cache:model:{modelId}             # TTL: 1小时

# 团队缓存
fastgpt:cache:team:{teamId}               # TTL: 30分钟
fastgpt:cache:team_vector_count:{teamId}  # TTL: 1小时

# 用户缓存
fastgpt:cache:user:{userId}               # TTL: 30分钟
fastgpt:session:{sessionId}               # TTL: 24小时

# 分布式锁
fastgpt:lock:{lockKey}                    # TTL: 动态

# 频率限制
fastgpt:limit:{userId}:{action}           # TTL: 动态
```

### 3.2 任务队列（BullMQ）

**训练队列（trainingQueue）：**
```typescript
{
  name: 'trainingQueue',
  data: {
    teamId: string,
    datasetId: string,
    collectionId: string,
    dataId: string,
    trainingData: {
      q: string,
      a: string,
      chunkIndex: number
    }
  },
  opts: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    removeOnComplete: true,
    removeOnFail: false
  }
}
```

**评估队列（evalQueue）：**
```typescript
{
  name: 'evalQueue',
  data: {
    teamId: string,
    appId: string,
    evalId: string,
    testData: Array
  }
}
```

### 3.3 缓存策略

- **系统配置**: 1小时 TTL，配置更新时主动失效
- **模型列表**: 1小时 TTL，新模型添加时主动失效
- **用户信息**: 30分钟 TTL，LRU淘汰
- **团队信息**: 30分钟 TTL，LRU淘汰
- **会话数据**: 24小时 TTL，过期自动清理
- **向量计数**: 1小时 TTL，增量更新

---

## 4. MinIO/S3 对象存储

### 4.1 Bucket 规划

```
fastgpt-avatar      # 用户/团队头像
fastgpt-chat        # 对话文件（上传的文件、图片）
fastgpt-dataset     # 知识库文件（文档、图片）
fastgpt-temp        # 临时文件（TTL: 24小时）
fastgpt-plugin      # 插件资源
```

### 4.2 对象 Key 规范

```
# 头像
avatar/{userId}/{timestamp}.{ext}
avatar/team/{teamId}/{timestamp}.{ext}

# 对话文件
chat/{teamId}/{chatId}/{fileId}.{ext}

# 知识库文件
dataset/{datasetId}/{collectionId}/{fileId}.{ext}

# 临时文件
temp/{sessionId}/{timestamp}_{fileId}.{ext}

# 插件资源
plugin/{pluginId}/{version}/{filename}
```

### 4.3 预签名 URL

```typescript
// 上传预签名（5分钟）
const uploadUrl = await s3Client.presignedPutObject(
  bucket, objectKey, 60 * 5
);

// 下载预签名（24小时）
const downloadUrl = await s3Client.presignedGetObject(
  bucket, objectKey, 60 * 60 * 24
);
```

---

## 5. 数据一致性保证

### 5.1 事务处理

**MongoDB 事务示例：**
```typescript
const session = await connectionMongo.startSession();
await session.withTransaction(async () => {
  // 1. 删除应用
  await MongoApp.deleteOne({ _id: appId }, { session });
  
  // 2. 删除应用版本
  await MongoAppVersion.deleteMany({ appId }, { session });
  
  // 3. 删除权限记录
  await MongoResourcePermission.deleteMany(
    { resourceId: appId }, 
    { session }
  );
});
```

### 5.2 级联删除策略

**删除应用：**
1. 删除应用记录（apps）
2. 删除所有版本（app_versions）
3. 删除权限记录（permissions）
4. 删除对话记录（chats, chat_items）
5. 清理Redis缓存

**删除知识库：**
1. 删除知识库记录（datasets）
2. 删除文档集合（dataset_collections）
3. 删除数据块（dataset_datas）
4. 删除向量（VectorDB）
5. 删除文件（MinIO/S3）
6. 清理Redis缓存
7. 更新团队向量计数

### 5.3 数据备份策略

- **MongoDB**: 
  - 全量备份：每天 02:00
  - 增量备份：Oplog实时备份
  - 保留周期：30天

- **向量数据库**:
  - 导出备份：每周一次
  - 保留周期：4周

- **MinIO/S3**:
  - 对象版本控制：启用
  - 生命周期：temp bucket 24小时自动删除

---

## 6. 性能优化策略

### 6.1 MongoDB 优化

**索引优化：**
- 复合索引：根据查询模式创建（teamId + updateTime）
- 全文索引：name字段支持模糊搜索
- 覆盖索引：只查询索引字段时直接返回

**分片策略：**
```javascript
sh.shardCollection("fastgpt.dataset_datas", { teamId: 1, _id: 1 })
```

**读写分离：**
- 查询操作：从库（readPreference: 'secondaryPreferred'）
- 写入操作：主库（readPreference: 'primary'）

### 6.2 向量数据库优化

**索引选择：**
- HNSW：高性能（推荐）
- IVF_FLAT：内存友好
- IVF_SQ8：压缩存储

**批量操作：**
```typescript
// 批量插入向量（100个一批）
for (let i = 0; i < vectors.length; i += 100) {
  await insertVectors(vectors.slice(i, i + 100));
}
```

### 6.3 Redis 优化

**Pipeline 批量操作：**
```typescript
const pipeline = redis.pipeline();
for (const key of keys) {
  pipeline.get(key);
}
const results = await pipeline.exec();
```

**Lua 脚本原子操作：**
```typescript
const script = `
  local count = redis.call('INCR', KEYS[1])
  redis.call('EXPIRE', KEYS[1], ARGV[1])
  return count
`;
await redis.eval(script, 1, key, ttl);
```

### 6.4 对象存储优化

- **分片上传**: 文件 >100MB 使用分片上传
- **CDN加速**: 公开资源通过CDN分发
- **压缩**: 图片自动压缩（JPEG 85%, PNG optimized）

---

## 7. 监控与维护

### 7.1 监控指标

**MongoDB：**
- 查询耗时 >1s 告警
- 慢查询日志
- 连接数 >80% 告警
- 磁盘空间 >85% 告警

**向量数据库：**
- 检索耗时 >500ms 告警
- 索引大小监控
- 向量数量统计

**Redis：**
- 内存使用 >80% 告警
- 缓存命中率 <50% 告警
- 队列长度 >1000 告警

**MinIO/S3：**
- 存储容量监控
- 请求量统计
- 上传/下载失败率

### 7.2 日志系统

**日志级别：**
- ERROR: 错误日志
- WARN: 警告日志
- INFO: 信息日志
- DEBUG: 调试日志

**日志存储：**
- MongoDB (system_logs)
- OpenTelemetry (分布式追踪)
- Pino/Winston (结构化日志)

---

## 8. 数据迁移与版本管理

### 8.1 数据迁移策略

**版本升级流程：**
1. 备份当前数据
2. 运行迁移脚本
3. 验证数据完整性
4. 更新索引
5. 清理旧数据

### 8.2 Schema 版本控制

```typescript
// 应用 Schema 版本
{
  version: 'v2',  // v1 -> v2 迁移
  // 迁移逻辑...
}
```

---

## 9. 安全性设计

### 9.1 数据加密

- **传输加密**: TLS/SSL
- **存储加密**: MongoDB/MinIO 加密
- **密码加密**: bcrypt hash
- **敏感数据**: AES-256-GCM 加密

### 9.2 访问控制

- **MongoDB**: 用户名/密码 + IP白名单
- **Redis**: AUTH密码验证
- **MinIO/S3**: AccessKey/SecretKey
- **VectorDB**: 独立认证

### 9.3 数据隔离

- **多租户隔离**: teamId分隔
- **数据权限**: 基于团队成员角色
- **API限流**: 基于用户/团队

---

## 10. 扩展性设计

### 10.1 水平扩展

- **MongoDB**: 分片（Sharding）
- **Redis**: 集群（Cluster）
- **VectorDB**: 分区（Partitioning）
- **MinIO**: 分布式部署

### 10.2 垂直扩展

- 增加服务器配置
- 优化索引和查询
- 缓存优化

---

## 附录

### A. 环境变量配置

```env
# MongoDB
MONGO_URL=mongodb://username:password@host:27017/fastgpt
MONGO_LOG_URL=mongodb://username:password@host:27017/fastgpt_logs
DB_MAX_LINK=30

# Redis
REDIS_URL=redis://:password@host:6379

# 向量数据库
PG_URL=postgresql://user:password@host:5432/fastgpt
# 或
MILVUS_ADDRESS=host:19530
# 或
OCEANBASE_URL=...

# MinIO/S3
MINIO_ENDPOINT=http://host:9000
MINIO_ACCESS_KEY=accesskey
MINIO_SECRET_KEY=secretkey
MINIO_BUCKET_PREFIX=fastgpt-
```

### B. 数据量级参考

- 小型部署: <1000用户, <100万向量
- 中型部署: 1000-10000用户, 100万-1000万向量
- 大型部署: >10000用户, >1000万向量

### C. 相关文档

- [系统架构文档](./system-architecture.md)
- [数据持久化层文档](../03-layers/05-data-persistence-layer.md)
- [服务层文档](../03-layers/03-service-layer.md)
