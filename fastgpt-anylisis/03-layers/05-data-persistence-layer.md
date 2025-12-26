# 数据持久化层 (Data Persistence Layer)

## 概述
数据持久化层负责 FastGPT 的数据存储与管理，包括 MongoDB（结构化数据）、向量数据库（Embedding 向量）、Redis（缓存与队列）和 MinIO/S3（对象存储）。

## 技术栈
- **MongoDB**: 主数据库（应用、知识库、对话记录、用户等）
- **向量数据库**: PGVector / Milvus / OceanBase（向量检索）
- **Redis**: 缓存、任务队列（BullMQ）、Session
- **MinIO/S3**: 对象存储（文件、图片、语音）

## 核心架构

### 1. MongoDB

#### 连接管理
```typescript
// 主连接
export const connectionMongo = new Mongoose();
connectionMongo.connect(MONGO_URL);

// 日志连接（读写分离）
export const connectionLogMongo = new Mongoose();
connectionLogMongo.connect(MONGO_LOG_URL);
```

#### 核心数据表

**MongoApp** - 应用表
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,           // 团队 ID
  tmbId: ObjectId,            // 创建者
  parentId: string,           // 父文件夹
  name: string,
  avatar: string,
  intro: string,
  type: AppTypeEnum,
  teamTags: string[],
  version: string,
  chatConfig: {...},
  inheritPermission: boolean,
  createTime: Date,
  updateTime: Date
}
索引:
- { teamId: 1, parentId: 1 }
- { teamId: 1, type: 1 }
- { teamId: 1, name: 'text' }
```

**MongoAppVersion** - 应用版本表
```typescript
{
  _id: ObjectId,
  appId: ObjectId,
  versionName: string,
  nodes: FlowNodeItemType[],
  edges: Edge[],
  chatConfig: {...},
  isPublish: boolean,
  tmbId: ObjectId,
  time: Date
}
索引:
- { appId: 1, time: -1 }
```

**MongoDataset** - 知识库表
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,
  tmbId: ObjectId,
  parentId: string,
  avatar: string,
  name: string,
  intro: string,
  type: DatasetTypeEnum,
  status: DatasetStatusEnum,
  vectorModel: string,
  agentModel: string,
  websiteConfig: {...},
  apiDatasetConfig: {...},
  inheritPermission: boolean,
  createTime: Date,
  updateTime: Date
}
索引:
- { teamId: 1, parentId: 1 }
- { teamId: 1, type: 1 }
```

**MongoDatasetCollection** - 文档集合表
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,
  tmbId: ObjectId,
  datasetId: ObjectId,
  parentId: string,
  name: string,
  type: DatasetCollectionTypeEnum,
  trainingType: TrainingTypeEnum,
  chunkSize: number,
  chunkSplitter: string,
  fileId: string,
  rawLink: string,
  hashRawText: string,
  createTime: Date,
  updateTime: Date,
  trainingAmount: number,
  canWrite: boolean
}
索引:
- { teamId: 1, datasetId: 1 }
- { teamId: 1, updateTime: -1 }
- { hashRawText: 1 }
```

**MongoDatasetData** - 数据块表
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,
  tmbId: ObjectId,
  datasetId: ObjectId,
  collectionId: ObjectId,
  chunkIndex: number,
  q: string,                  // 问题（用于检索）
  a: string,                  // 答案（实际内容）
  fullTextToken: string,      // 全文分词
  indexes: [{
    dataId: string,
    defaultIndex: boolean,
    type: IndexTypeEnum
  }],
  updateTime: Date
}
索引:
- { teamId: 1, datasetId: 1, collectionId: 1 }
- { teamId: 1, datasetId: 1, updateTime: -1 }
- { 'indexes.dataId': 1 }
- 全文索引: { fullTextToken: 'text', q: 'text' }
```

**MongoDatasetTraining** - 训练队列表
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,
  tmbId: ObjectId,
  datasetId: ObjectId,
  collectionId: ObjectId,
  billId: string,
  mode: TrainingModeEnum,
  prompt: string,
  q: string,
  a: string,
  chunkIndex: number,
  weight: number,
  model: string,
  lockTime: Date,
  retryTimes: number
}
索引:
- { teamId: 1, lockTime: 1 }
- { billId: 1 }
```

**MongoChatConversation** - 对话会话表
```typescript
{
  _id: ObjectId,
  chatId: string,
  teamId: ObjectId,
  tmbId: ObjectId,
  appId: ObjectId,
  title: string,
  customTitle: string,
  top: boolean,
  variables: {...},
  metadata: {...},
  createTime: Date,
  updateTime: Date
}
索引:
- { chatId: 1 }
- { appId: 1, updateTime: -1 }
- { teamId: 1, tmbId: 1, updateTime: -1 }
```

**MongoChatItem** - 对话消息表
```typescript
{
  _id: ObjectId,
  chatId: string,
  dataId: string,
  appId: ObjectId,
  teamId: ObjectId,
  tmbId: ObjectId,
  nodeOutputs: ChatItemType[],
  interactive: {...},
  metadata: {...},
  createTime: Date
}
索引:
- { chatId: 1, dataId: 1 }
- { appId: 1, createTime: -1 }
```

**MongoUser** - 用户表
```typescript
{
  _id: ObjectId,
  username: string,
  password: string,
  avatar: string,
  timezone: string,
  openaiAccount: {...},
  createTime: Date,
  lastLoginTime: Date
}
索引:
- { username: 1 } (唯一)
```

**MongoTeam** - 团队表
```typescript
{
  _id: ObjectId,
  name: string,
  avatar: string,
  ownerId: ObjectId,
  balance: number,
  createTime: Date,
  limit: {...}
}
```

**MongoTeamMember** - 团队成员表
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,
  userId: ObjectId,
  role: TeamMemberRoleEnum,
  status: TeamMemberStatusEnum,
  createTime: Date
}
索引:
- { teamId: 1, userId: 1 } (唯一)
- { userId: 1 }
```

### 2. 向量数据库

#### PGVector (PostgreSQL)

**连接配置**
```typescript
const pool = new Pool({
  host: process.env.PG_HOST,
  port: process.env.PG_PORT,
  database: process.env.PG_DATABASE,
  user: process.env.PG_USER,
  password: process.env.PG_PASSWORD
});
```

**表结构**
```sql
CREATE TABLE dataset_vector (
  id SERIAL PRIMARY KEY,
  dataset_id VARCHAR(24),
  collection_id VARCHAR(24),
  data_id VARCHAR(24),
  vector vector(1536),  -- 维度根据模型调整
  create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_dataset_id ON dataset_vector(dataset_id);
CREATE INDEX idx_data_id ON dataset_vector(data_id);
CREATE INDEX idx_vector_hnsw ON dataset_vector USING hnsw (vector vector_cosine_ops);
```

**检索操作**
```typescript
// 向量检索
const result = await pool.query(`
  SELECT data_id, 1 - (vector <=> $1) as score
  FROM dataset_vector
  WHERE dataset_id = $2
  ORDER BY vector <=> $1
  LIMIT $3
`, [queryVector, datasetId, limit]);
```

#### Milvus

**Collection 结构**
```typescript
{
  collectionName: `ds_${datasetId}`,
  fields: [
    { name: 'id', dataType: DataType.Int64, isPrimary: true, autoID: true },
    { name: 'data_id', dataType: DataType.VarChar, maxLength: 24 },
    { name: 'vector', dataType: DataType.FloatVector, dim: 1536 }
  ]
}
```

**索引配置**
```typescript
{
  index_type: 'HNSW',
  metric_type: 'COSINE',
  params: { M: 16, efConstruction: 256 }
}
```

**检索操作**
```typescript
const result = await milvusClient.search({
  collection_name: `ds_${datasetId}`,
  vectors: [queryVector],
  search_params: {
    anns_field: 'vector',
    topk: limit,
    metric_type: 'COSINE',
    params: JSON.stringify({ ef: 64 })
  }
});
```

### 3. Redis

#### 连接管理
```typescript
// 任务队列连接
const queueRedis = new Redis(REDIS_URL);

// Worker 连接
const workerRedis = new Redis(REDIS_URL, {
  maxRetriesPerRequest: null
});

// 全局缓存连接
const cacheRedis = new Redis(REDIS_URL);
```

#### 数据结构

**缓存 Key 规范**
```
fastgpt:cache:system_config         # 系统配置
fastgpt:cache:models                # 模型列表
fastgpt:cache:team:{teamId}         # 团队信息
fastgpt:cache:user:{userId}         # 用户信息
fastgpt:session:{sessionId}         # 用户会话
fastgpt:lock:{lockKey}              # 分布式锁
```

**任务队列（BullMQ）**
```typescript
// 训练队列
const trainingQueue = new Queue('trainingQueue', {
  connection: queueRedis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 }
  }
});

// 任务数据结构
{
  datasetId: string,
  collectionId: string,
  trainingData: {
    q: string,
    a: string,
    chunkIndex: number
  }
}
```

**缓存策略**
- TTL: 系统配置 1小时，模型列表 1小时，用户信息 30分钟
- LRU: 自动淘汰最少使用的 Key
- Pub/Sub: 配置更新通知

### 4. MinIO / S3

#### Bucket 规划
```
fastgpt-avatar          # 头像
fastgpt-chat            # 对话文件
fastgpt-dataset         # 知识库文件
fastgpt-temp            # 临时文件
fastgpt-plugin          # 插件资源
```

#### 对象 Key 规范
```
avatar/{userId}/{timestamp}.{ext}
chat/{teamId}/{chatId}/{fileId}.{ext}
dataset/{datasetId}/{collectionId}/{fileId}.{ext}
temp/{sessionId}/{fileId}.{ext}
plugin/{pluginId}/{version}/{file}
```

#### 预签名 URL
```typescript
// 上传预签名（POST）
const presignedUrl = await s3Client.presignedPutObject(
  bucket,
  objectKey,
  60 * 5  // 5分钟过期
);

// 下载预签名（GET）
const presignedUrl = await s3Client.presignedGetObject(
  bucket,
  objectKey,
  60 * 60 * 24  // 24小时过期
);
```

## 数据一致性

### 事务处理（MongoDB）
```typescript
const session = await connectionMongo.startSession();
await session.withTransaction(async () => {
  // 1. 删除应用
  await MongoApp.deleteOne({ _id: appId }, { session });
  // 2. 删除版本
  await MongoAppVersion.deleteMany({ appId }, { session });
  // 3. 删除权限
  await MongoResourcePermission.deleteMany({ resourceId: appId }, { session });
});
```

### 级联删除
**删除应用**
1. 删除应用记录
2. 删除所有版本
3. 删除权限记录
4. 删除对话记录
5. 清理缓存

**删除知识库**
1. 删除知识库记录
2. 删除文档集合
3. 删除数据块（MongoDB）
4. 删除向量（VectorDB）
5. 删除文件（MinIO/S3）
6. 清理缓存

### 数据备份
- MongoDB: 定期全量备份 + Oplog 增量备份
- 向量数据库: 导出向量数据定期备份
- MinIO/S3: 对象版本控制 + 定期快照

## 性能优化

### MongoDB 优化
- **索引优化**: 根据查询模式创建复合索引
- **分片**: 按 teamId 分片实现水平扩展
- **读写分离**: 查询走从库，写入走主库
- **聚合优化**: 使用 `$project` 减少返回字段

### 向量数据库优化
- **索引类型**: HNSW（高性能）、IVF_FLAT（内存友好）
- **批量操作**: 批量插入向量提升性能
- **分区策略**: 按 datasetId 分区

### Redis 优化
- **Pipeline**: 批量操作减少网络往返
- **Lua 脚本**: 原子操作保证一致性
- **连接池**: 复用连接减少开销

### 对象存储优化
- **分片上传**: 大文件分片上传
- **CDN 加速**: 静态资源通过 CDN 分发
- **压缩**: 图片自动压缩

## 监控与维护

### 监控指标
- MongoDB: 查询耗时、慢查询、连接数
- 向量数据库: 检索耗时、索引大小
- Redis: 内存使用、命中率、队列长度
- MinIO/S3: 存储容量、请求量

### 告警规则
- MongoDB 慢查询 > 1s
- Redis 内存使用 > 80%
- 向量检索耗时 > 500ms
- 对象存储容量 > 阈值

## 相关文档
- [业务服务层](./03-service-layer.md)
- [共享层](./04-shared-layer.md)
- [外部服务层](./06-external-services-layer.md)
