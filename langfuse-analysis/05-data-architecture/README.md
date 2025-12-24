# Langfuse 数据架构文档

## 概述

Langfuse 采用**混合数据库架构**，结合了关系型数据库（PostgreSQL）、列式分析数据库（ClickHouse）、缓存系统（Redis）和对象存储（S3），以满足不同的业务需求：

- **PostgreSQL（Prisma ORM）**: 存储元数据、用户信息、配置数据等事务性数据
- **ClickHouse**: 存储和分析海量追踪数据（traces、observations、scores、events）
- **Redis**: 提供缓存加速和异步任务队列
- **S3 兼容对象存储**: 存储媒体文件、批量导出数据等大型二进制对象

这种架构设计实现了**读写分离**和**冷热数据分离**，确保系统在处理大规模追踪数据时仍能保持高性能。

## 数据存储技术栈

| 技术 | 版本要求 | 用途 | 必需性 |
|------|---------|------|--------|
| **PostgreSQL** | 12+ | 主数据库，存储元数据和配置 | ✅ 必需 |
| **ClickHouse** | 23+ | 分析数据库，存储追踪数据 | ✅ 必需 |
| **Redis** | 6+ | 缓存和任务队列 | ⚠️ 可选（推荐生产环境） |
| **S3 Compatible** | - | 对象存储 | ⚠️ 可选（媒体和导出功能需要） |

### 数据库选型理由

1. **PostgreSQL + Prisma**
   - 成熟的关系型数据库，提供强一致性和 ACID 事务
   - Prisma ORM 提供类型安全的数据库访问
   - 支持复杂的关联查询和数据完整性约束
   - 适合存储结构化的元数据

2. **ClickHouse**
   - 列式存储，压缩率高，存储成本低
   - 针对分析查询优化，聚合查询性能优异
   - 支持横向扩展，可处理 PB 级数据
   - 非常适合存储 tracing 这种写多读少的时序数据

3. **Redis**
   - 内存数据库，读写性能极高
   - 支持多种数据结构（String, Hash, List, Set, ZSet）
   - BullMQ 任务队列，提供可靠的异步处理
   - API 密钥缓存，减少数据库查询压力

4. **S3 对象存储**
   - 无限容量，按需扩展
   - 成本低，适合存储大量媒体文件
   - 支持预签名 URL，安全的文件上传下载
   - 兼容多种云服务商（AWS S3、MinIO、Cloudflare R2 等）

## 1. PostgreSQL 数据库设计

### 1.1 连接管理

```typescript
// 连接配置（packages/shared/prisma/schema.prisma）
datasource db {
  provider          = "postgresql"
  url               = env("DATABASE_URL")
  directUrl         = env("DIRECT_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}
```

**连接池配置**：
- 使用 Prisma 内置连接池
- 默认连接池大小：10（可通过 `connection_limit` 参数调整）
- 连接超时：2 秒
- 支持读写分离（通过 `directUrl` 配置）

### 1.2 核心数据模型

Langfuse 的 PostgreSQL 数据库包含 **50+ 张表**，可以分为以下几个模块：

#### 1.2.1 用户与认证模块

**User 表** - 用户基础信息
```prisma
model User {
  id              String    @id @default(cuid())
  name            String?
  email           String?   @unique
  emailVerified   DateTime?
  password        String?
  image           String?
  admin           Boolean   @default(false)
  featureFlags    String[]  @default([])
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // 关联关系
  accounts                  Account[]
  sessions                  Session[]
  organizationMemberships   OrganizationMembership[]
  projectMemberships        ProjectMembership[]
  // ... 更多关联
}
```

**Account 表** - OAuth 账号关联（NextAuth.js）
```prisma
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String  // google, github, azure-ad 等
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@unique([provider, providerAccountId])
  @@index([userId])
}
```

**Session 表** - 用户会话
```prisma
model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

**关键特性**：
- 支持多种认证方式（用户名密码、OAuth、SSO）
- 级联删除：删除用户时自动清理关联数据
- 邮箱唯一性约束

#### 1.2.2 组织与项目模块

**Organization 表** - 多租户组织
```prisma
model Organization {
  id                               String   @id @default(cuid())
  name                             String
  createdAt                        DateTime @default(now())
  updatedAt                        DateTime @updatedAt
  cloudConfig                      Json?    // Langfuse Cloud 配置
  metadata                         Json?
  
  // 计费相关（云版本）
  cloudBillingCycleAnchor          DateTime?
  cloudCurrentCycleUsage           Int?
  cloudFreeTierUsageThresholdState String?
  aiFeaturesEnabled                Boolean  @default(false)
  
  // 关联关系
  organizationMemberships OrganizationMembership[]
  projects                Project[]
  apiKeys                 ApiKey[]
}
```

**Project 表** - 项目（数据隔离单元）
```prisma
model Project {
  id            String    @id @default(cuid())
  orgId         String
  name          String
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  deletedAt     DateTime? // 软删除
  retentionDays Int?      // 数据保留天数
  metadata      Json?
  
  organization Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)
  
  // 关联的所有项目数据
  projectMembers    ProjectMembership[]
  apiKeys           ApiKey[]
  dataset           Dataset[]
  sessions          TraceSession[]
  Prompt            Prompt[]
  Model             Model[]
  scoreConfig       ScoreConfig[]
  // ... 50+ 个关联
  
  @@index([orgId])
}
```

**OrganizationMembership 表** - 组织成员关系
```prisma
model OrganizationMembership {
  id           String   @id @default(cuid())
  orgId        String
  userId       String
  role         Role     // OWNER, ADMIN, MEMBER, VIEWER
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  
  organization Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)
  user         User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  ProjectMemberships ProjectMembership[]
  
  @@unique([orgId, userId])
  @@index([userId])
}
```

**ProjectMembership 表** - 项目成员关系
```prisma
model ProjectMembership {
  orgMembershipId String
  projectId       String
  userId          String
  role            Role
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  organizationMembership OrganizationMembership @relation(...)
  project                Project                @relation(...)
  user                   User                   @relation(...)
  
  @@id([projectId, userId])
  @@index([userId])
  @@index([projectId])
}
```

**数据隔离设计**：
- 所有业务数据都属于某个 Project
- Project 属于 Organization
- 通过 `projectId` 进行数据隔离
- 软删除机制（`deletedAt`），支持数据恢复

#### 1.2.3 API 密钥模块

**ApiKey 表**
```prisma
enum ApiKeyScope {
  ORGANIZATION  // 组织级，可访问所有项目
  PROJECT       // 项目级，只能访问单个项目
}

model ApiKey {
  id                  String      @id @default(cuid())
  createdAt           DateTime    @default(now())
  note                String?
  publicKey           String      @unique
  hashedSecretKey     String      @unique  // bcrypt 慢速哈希
  fastHashedSecretKey String?     @unique  // SHA-256 快速哈希
  displaySecretKey    String                // 显示用的部分密钥
  lastUsedAt          DateTime?
  expiresAt           DateTime?
  
  projectId    String?
  project      Project?      @relation(...)
  orgId        String?
  organization Organization? @relation(...)
  scope        ApiKeyScope   @default(PROJECT)
  
  @@index([orgId])
  @@index([projectId])
  @@index([publicKey])
  @@index([hashedSecretKey])
  @@index([fastHashedSecretKey])
}
```

**密钥验证策略**：
1. 优先使用 `fastHashedSecretKey`（SHA-256）快速验证
2. 如果不存在，降级使用 `hashedSecretKey`（bcrypt）验证
3. 验证通过后，生成 `fastHashedSecretKey` 并更新
4. Redis 缓存密钥信息，减少数据库查询

#### 1.2.4 Prompt 管理模块

**Prompt 表** - 提示词模板
```prisma
model Prompt {
  id          String    @id @default(cuid())
  projectId   String
  name        String
  version     Int
  prompt      String    @db.Text  // JSON 格式的提示词内容
  type        String    @default("text")
  isActive    Boolean   @default(false)
  config      Json?     // LLM 配置（model, temperature 等）
  tags        String[]  @default([])
  labels      String[]  @default([])
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdBy   String
  updatedBy   String
  
  project Project @relation(...)
  
  @@unique([projectId, name, version])
  @@index([projectId, name])
}
```

**PromptProtectedLabels 表** - 受保护的标签
```prisma
model PromptProtectedLabels {
  projectId String
  label     String
  
  project Project @relation(...)
  
  @@id([projectId, label])
}
```

#### 1.2.5 数据集管理模块

**Dataset 表**
```prisma
model Dataset {
  id          String   @id @default(cuid())
  projectId   String
  name        String
  description String?
  metadata    Json?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  project      Project       @relation(...)
  datasetItems DatasetItem[]
  datasetRuns  DatasetRun[]
  
  @@unique([projectId, name])
  @@index([projectId])
}
```

**DatasetItem 表**
```prisma
model DatasetItem {
  id               String   @id @default(cuid())
  datasetId        String
  input            Json?
  expectedOutput   Json?
  metadata         Json?
  sourceObservationId String?
  sourceTraceId       String?
  status              String  @default("ACTIVE")
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt
  
  dataset Dataset @relation(...)
  
  @@index([datasetId])
}
```

#### 1.2.6 评分配置模块

**ScoreConfig 表**
```prisma
model ScoreConfig {
  id          String   @id @default(cuid())
  projectId   String
  name        String
  dataType    String   // NUMERIC, CATEGORICAL, BOOLEAN
  isArchived  Boolean  @default(false)
  
  // 类别型评分配置
  categories  Json?    // [{value: 1, label: "Good"}, ...]
  
  // 数值型评分配置
  minValue    Float?
  maxValue    Float?
  
  description String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  project Project @relation(...)
  
  @@unique([projectId, name])
  @@index([projectId])
}
```

#### 1.2.7 评估模块

**EvalTemplate 表** - 评估模板
```prisma
model EvalTemplate {
  id          String   @id @default(cuid())
  projectId   String
  name        String
  version     Int
  prompt      String   @db.Text
  model       String
  modelParams Json
  outputSchema Json
  provider    String
  vars        String[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  project Project @relation(...)
  
  @@unique([projectId, name, version])
  @@index([projectId])
}
```

**JobConfiguration 表** - 评估任务配置
```prisma
model JobConfiguration {
  id              String   @id @default(cuid())
  projectId       String
  jobType         String   // EVAL_EXECUTION, DATASET_RUN
  status          String   @default("ACTIVE")
  evalTemplateId  String?
  targetObject    String?  // observations, traces
  filter          Json?
  samplingRate    Float    @default(1.0)
  delay           Int      @default(0)
  variableMapping Json?
  scoreName       String?
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  project Project @relation(...)
  
  @@index([projectId])
}
```

#### 1.2.8 LLM 集成模块

**LlmApiKeys 表** - LLM API 密钥配置
```prisma
model LlmApiKeys {
  id                String   @id @default(cuid())
  projectId         String
  provider          String   // openai, anthropic, azure 等
  adapter           String   // 接口适配器类型
  displaySecretKey  String
  secretKey         String   // 加密存储
  baseURL           String?
  customModels      String[] @default([])
  withDefaultModels Boolean  @default(true)
  extraHeaders      String?
  extraHeaderKeys   String[] @default([])
  config            Json?
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  project Project @relation(...)
  
  @@unique([projectId, provider])
}
```

**Model 表** - 模型配置
```prisma
model Model {
  id                String   @id @default(cuid())
  projectId         String
  modelName         String
  matchPattern      String
  startDate         DateTime?
  unit              String    // TOKENS, CHARACTERS, REQUESTS 等
  inputPrice        Decimal?
  outputPrice       Decimal?
  totalPrice        Decimal?
  tokenizerId       String?
  tokenizerConfig   Json?
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  project Project @relation(...)
  
  @@unique([projectId, modelName, startDate, unit])
  @@index([projectId])
}
```

#### 1.2.9 集成模块

**PosthogIntegration 表**
```prisma
model PosthogIntegration {
  id             String   @id @default(cuid())
  projectId      String   @unique
  apiKey         String
  host           String
  enabled        Boolean  @default(true)
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  project Project @relation(...)
}
```

**SlackIntegration 表**
```prisma
model SlackIntegration {
  projectId      String   @id @unique
  workspaceId    String
  accessToken    String
  teamName       String?
  webhookUrl     String?
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  project Project @relation(...)
}
```

**BlobStorageIntegration 表** - 自定义对象存储
```prisma
model BlobStorageIntegration {
  id            String   @id @default(cuid())
  projectId     String   @unique
  provider      String   // s3, azure-blob, gcs 等
  endpoint      String
  region        String?
  bucket        String
  accessKeyId   String
  secretAccessKey String
  enabled       Boolean  @default(true)
  
  project Project @relation(...)
}
```

### 1.3 数据关系总览

```
Organization (1) ─┬─ (N) OrganizationMembership ─ (1) User
                  │
                  └─ (N) Project ─┬─ (N) ProjectMembership ─ (1) User
                                   │
                                   ├─ (N) ApiKey
                                   ├─ (N) Dataset ─ (N) DatasetItem
                                   ├─ (N) Prompt
                                   ├─ (N) Model
                                   ├─ (N) ScoreConfig
                                   ├─ (N) EvalTemplate
                                   ├─ (N) JobConfiguration
                                   ├─ (N) LlmApiKeys
                                   ├─ (1) PosthogIntegration
                                   ├─ (1) SlackIntegration
                                   └─ (1) BlobStorageIntegration
```

### 1.4 索引策略

**主要索引**：

1. **外键索引**：所有外键字段都有索引（自动创建）
   - `User.id`, `Organization.id`, `Project.id` 等

2. **查询优化索引**：
   - `ApiKey`: `publicKey`, `hashedSecretKey`, `fastHashedSecretKey`
   - `Project`: `orgId`
   - `OrganizationMembership`: `userId`, `orgId`
   - `ProjectMembership`: `userId`, `projectId`

3. **唯一性约束索引**：
   - `User.email`
   - `ApiKey.publicKey`
   - `Prompt(projectId, name, version)`
   - `Dataset(projectId, name)`

**性能考虑**：
- 使用 `@@index` 而非 `@unique`，除非需要唯一性约束
- 复合索引顺序：最常用的查询条件放在前面
- 避免过多索引，影响写入性能

## 2. ClickHouse 数据库设计

### 2.1 为什么使用 ClickHouse？

ClickHouse 用于存储 Langfuse 的核心业务数据：**追踪数据（Tracing Data）**

**数据特点**：
- 写入量大：每秒可能有数千次 trace/observation 写入
- 数据量大：单个项目可能产生数十亿条记录
- 查询模式：主要是聚合查询和时间范围查询
- 更新少：追踪数据写入后很少更新

**ClickHouse 优势**：
- 列式存储，压缩率高（通常 10:1）
- 针对 OLAP 查询优化，聚合性能优异
- 支持水平扩展，可处理 PB 级数据
- 写入性能高，支持批量插入

### 2.2 核心表结构

ClickHouse 中有以下核心表：

#### traces 表 - 追踪主表
```sql
CREATE TABLE traces (
  id String,
  timestamp DateTime64(3),
  project_id String,
  name String,
  user_id String,
  metadata String,  -- JSON
  tags Array(String),
  input String,     -- JSON
  output String,    -- JSON
  session_id String,
  release String,
  version String,
  public Boolean,
  bookmarked Boolean,
  -- ... 更多字段
  
  INDEX idx_timestamp timestamp TYPE minmax GRANULARITY 1,
  INDEX idx_project_id project_id TYPE bloom_filter GRANULARITY 1
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (project_id, timestamp, id);
```

#### observations 表 - 观察记录（LLM 调用）
```sql
CREATE TABLE observations (
  id String,
  trace_id String,
  project_id String,
  type String,       -- SPAN, GENERATION, EVENT
  name String,
  start_time DateTime64(3),
  end_time DateTime64(3),
  completion_start_time DateTime64(3),
  model String,
  model_parameters String,  -- JSON
  input String,             -- JSON
  output String,            -- JSON
  metadata String,          -- JSON
  level String,
  status_message String,
  version String,
  prompt_id String,
  prompt_name String,
  prompt_version Int32,
  -- 成本和使用量
  input_cost Decimal(20, 10),
  output_cost Decimal(20, 10),
  total_cost Decimal(20, 10),
  input_units Int64,
  output_units Int64,
  total_units Int64,
  
  INDEX idx_trace_id trace_id TYPE bloom_filter GRANULARITY 1,
  INDEX idx_project_id project_id TYPE bloom_filter GRANULARITY 1
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(start_time)
ORDER BY (project_id, start_time, trace_id, id);
```

#### scores 表 - 评分数据
```sql
CREATE TABLE scores (
  id String,
  timestamp DateTime64(3),
  project_id String,
  trace_id String,
  observation_id String,
  name String,
  value Float64,
  string_value String,
  source String,       -- API, ANNOTATION, EVAL
  comment String,
  author_user_id String,
  config_id String,
  data_type String,    -- NUMERIC, CATEGORICAL, BOOLEAN
  
  INDEX idx_project_id project_id TYPE bloom_filter GRANULARITY 1
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (project_id, timestamp, id);
```

#### events 表 - 事件日志
```sql
CREATE TABLE events (
  id String,
  timestamp DateTime64(3),
  project_id String,
  type String,
  name String,
  data String,  -- JSON
  
  INDEX idx_project_id project_id TYPE bloom_filter GRANULARITY 1
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (project_id, timestamp, id);
```

#### dataset_run_items_rmt 表 - 数据集运行结果（反规范化快照）
```sql
CREATE TABLE dataset_run_items_rmt (
  -- primary identifiers
  id String,
  project_id String,
  dataset_run_id String,
  dataset_item_id String,
  dataset_id String,
  trace_id String,
  observation_id Nullable(String),
  error Nullable(String),
  
  -- timestamps
  created_at DateTime64(3) DEFAULT now(),
  updated_at DateTime64(3) DEFAULT now(),
  
  -- denormalized dataset run fields (immutable)
  dataset_run_name String,
  dataset_run_description Nullable(String),
  dataset_run_metadata Map(LowCardinality(String), String),
  dataset_run_created_at DateTime64(3),
  
  -- denormalized dataset item fields (mutable snapshots)
  dataset_item_input Nullable(String) CODEC(ZSTD(3)),
  dataset_item_expected_output Nullable(String) CODEC(ZSTD(3)),
  dataset_item_metadata Map(LowCardinality(String), String),
  
  -- clickhouse engine fields
  event_ts DateTime64(3),
  is_deleted UInt8,
  
  INDEX idx_dataset_item dataset_item_id TYPE bloom_filter(0.001) GRANULARITY 1
) ENGINE = ReplacingMergeTree(event_ts, is_deleted)
ORDER BY (project_id, dataset_id, dataset_run_id, id);

-- 注意：latency 和 cost 指标不存储在此表，而是查询时从 observations/traces 表计算
```

### 2.3 ClickHouse 性能优化

**分区策略**：
- 按月分区：`PARTITION BY toYYYYMM(timestamp)`
- 便于数据保留策略（TTL）
- 查询时可以跳过不相关的分区

**排序键**：
- 主排序键：`(project_id, timestamp, id)`
- 数据隔离：project_id 作为第一列
- 时间范围查询：timestamp 作为第二列

**索引策略**：
- 主键索引：自动创建，基于 ORDER BY
- Bloom Filter 索引：用于字符串字段的精确匹配
- MinMax 索引：用于时间范围查询

**压缩**：
- 默认使用 LZ4 压缩
- 列式存储天然高压缩率
- 典型压缩比：10:1 到 20:1

**批量写入**：
- 使用批量插入而非单条插入
- 默认批次大小：10,000 条
- 异步写入，通过队列缓冲

### 2.4 数据保留策略

ClickHouse 支持 TTL（Time To Live）自动清理旧数据：

```sql
ALTER TABLE traces 
MODIFY TTL timestamp + INTERVAL project_retention_days DAY;
```

- 每个项目可配置不同的保留天数
- 自动清理过期数据，释放存储空间
- 后台自动执行，不影响查询性能

## 3. Redis 数据设计

### 3.1 Redis 使用场景

Langfuse 使用 Redis 实现两大功能：
1. **缓存加速**：减少数据库查询压力
2. **任务队列**：异步处理和后台任务

### 3.2 连接配置

```typescript
// 支持三种部署模式
- 单机模式：REDIS_CONNECTION_STRING
- 集群模式：REDIS_CLUSTER_NODES
- 哨兵模式：REDIS_SENTINEL_NODES + REDIS_SENTINEL_MASTER_NAME

// 连接选项
{
  enableReadyCheck: true,
  maxRetriesPerRequest: null,
  enableAutoPipelining: true,  // 自动管道化
  keyPrefix: 'langfuse:',      // Key 前缀
  retryStrategy: exponential backoff,
  reconnectOnError: true
}
```

### 3.3 缓存策略

#### 3.3.1 API 密钥缓存

**Key 格式**：
```
langfuse:api-key:hash:{sha256_hash}
```

**缓存内容**：
```json
{
  "id": "key-id",
  "projectId": "proj-id",
  "orgId": "org-id",
  "scope": "PROJECT",
  "plan": "pro",
  "rateLimitOverrides": [],
  "isIngestionSuspended": false
}
```

**缓存策略**：
- TTL：24 小时
- 验证流程：
  1. 计算密钥的 SHA-256 哈希
  2. 查询 Redis 缓存
  3. 缓存命中：直接返回
  4. 缓存未命中：查询数据库并缓存

**失效策略**：
- 密钥删除时主动失效
- 项目/组织更新时批量失效
- 定时刷新（通过 TTL 自动过期）

#### 3.3.2 项目配置缓存

**Key 格式**：
```
langfuse:project:config:{project_id}
```

**缓存内容**：
- 项目元数据
- 集成配置
- Feature flags

### 3.4 任务队列（BullMQ）

Langfuse 使用 BullMQ 实现了 **20+ 个任务队列**：

#### 核心队列列表

| 队列名称 | 用途 | 优先级 |
|---------|------|--------|
| **ingestionQueue** | 追踪数据摄取 | 高 |
| **otelIngestionQueue** | OTEL 数据摄取 | 高 |
| **traceUpsert** | Trace 更新 | 高 |
| **webhookQueue** | Webhook 发送 | 中 |
| **createEvalQueue** | 创建评估任务 | 中 |
| **evalExecutionQueue** | 执行评估 | 中 |
| **batchExport** | 批量导出 | 低 |
| **batchActionQueue** | 批量操作 | 低 |
| **traceDelete** | Trace 删除 | 低 |
| **projectDelete** | 项目删除 | 低 |
| **datasetDelete** | 数据集删除 | 低 |
| **scoreDelete** | 评分删除 | 低 |
| **datasetRunItemUpsert** | 数据集运行项更新 | 中 |
| **cloudUsageMeteringQueue** | 用量计费 | 低 |
| **cloudSpendAlertQueue** | 费用告警 | 低 |
| **cloudFreeTierUsageThresholdQueue** | 免费额度告警 | 低 |
| **postHogIntegrationQueue** | PostHog 集成 | 低 |
| **mixpanelIntegrationQueue** | Mixpanel 集成 | 低 |
| **notificationQueue** | 通知发送 | 中 |
| **entityChangeQueue** | 实体变更事件 | 中 |

#### 队列配置

**默认选项**：
```typescript
{
  defaultJobOptions: {
    attempts: 5,              // 最多重试 5 次
    backoff: {
      type: 'exponential',
      delay: 5000             // 初始延迟 5 秒
    },
    removeOnComplete: 100,    // 保留最近 100 个完成的任务
    removeOnFail: 1000        // 保留最近 1000 个失败的任务
  }
}
```

**优先级队列**：
- 支持 1-10 优先级
- 高优先级任务优先处理
- 摄取队列使用最高优先级

#### 任务重试策略

1. **指数退避**：
   - 第 1 次重试：5 秒后
   - 第 2 次重试：25 秒后
   - 第 3 次重试：125 秒后
   - ...

2. **死信队列（DLQ）**：
   - 超过最大重试次数的任务进入 DLQ
   - 可以手动重新处理
   - 定期分析失败原因

### 3.5 Redis Key 命名规范

```
langfuse:{category}:{subcategory}:{identifier}

示例：
- langfuse:api-key:hash:abc123
- langfuse:project:config:proj-123
- langfuse:queue:ingestion:job:456
- langfuse:cache:user:user-789
- langfuse:lock:trace-upsert:trace-001
```

**命名原则**：
- 统一前缀：`langfuse:`
- 分层结构：便于批量操作
- 可读性：清晰表达用途
- 避免冲突：唯一标识

## 4. S3 对象存储设计

### 4.1 存储场景

Langfuse 使用 S3 兼容对象存储（支持 AWS S3、MinIO、Cloudflare R2 等）存储大型二进制对象：

1. **媒体文件**：图片、音频、视频等
2. **批量导出**：CSV、JSON 导出文件
3. **核心数据备份**：PostgreSQL 和 ClickHouse 数据导出

### 4.2 Bucket 规划

#### 4.2.1 Media Bucket - 媒体存储

**用途**：存储 trace 和 observation 关联的媒体文件

**Key 规范**：
```
media/{project_id}/{media_id}/{filename}

示例：
media/proj-abc123/media-xyz789/screenshot.png
```

**访问控制**：
- 私有存储
- 通过预签名 URL 访问
- URL 有效期：15 分钟

**生命周期**：
- 跟随 trace 数据保留策略
- 项目删除时批量清理

#### 4.2.2 Export Bucket - 导出存储

**用途**：存储批量导出的数据文件

**Key 规范**：
```
exports/{project_id}/{export_id}/{filename}

示例：
exports/proj-abc123/export-20231215/traces_export.csv
```

**访问控制**：
- 私有存储
- 预签名 URL 下载
- URL 有效期：24 小时

**生命周期**：
- 自动清理：30 天后删除
- 可配置保留时间

#### 4.2.3 Backup Bucket - 备份存储

**用途**：数据库备份和灾难恢复

**Key 规范**：
```
backups/{database}/{date}/{backup_file}

示例：
backups/postgresql/2023-12-15/full_backup.sql.gz
backups/clickhouse/2023-12-15/traces_partition_202312.tar.gz
```

**访问控制**：
- 最高权限保护
- 加密存储
- 版本控制启用

**生命周期**：
- 完整备份：保留 90 天
- 增量备份：保留 30 天

### 4.3 S3 客户端配置

```typescript
// packages/shared/src/server/s3/index.ts

export function getS3MediaStorageClient() {
  return new S3Client({
    region: env.S3_REGION,
    endpoint: env.S3_ENDPOINT,
    credentials: {
      accessKeyId: env.S3_ACCESS_KEY_ID,
      secretAccessKey: env.S3_SECRET_ACCESS_KEY
    },
    forcePathStyle: env.S3_FORCE_PATH_STYLE === 'true'
  });
}

export function getS3EventStorageClient() {
  return new S3Client({
    region: env.S3_EVENT_UPLOAD_REGION,
    endpoint: env.S3_EVENT_UPLOAD_ENDPOINT,
    credentials: {
      accessKeyId: env.S3_EVENT_UPLOAD_ACCESS_KEY_ID,
      secretAccessKey: env.S3_EVENT_UPLOAD_SECRET_ACCESS_KEY
    }
  });
}
```

### 4.4 预签名 URL 机制

**上传流程**：
```typescript
// 1. 客户端请求上传 URL
POST /api/media/upload-url
{
  "projectId": "proj-123",
  "fileName": "image.png",
  "contentType": "image/png"
}

// 2. 服务端生成预签名 URL
const presignedUrl = await s3Client.getSignedUrl('putObject', {
  Bucket: 'langfuse-media',
  Key: `media/${projectId}/${mediaId}/${fileName}`,
  Expires: 900, // 15 分钟
  ContentType: contentType
});

// 3. 客户端直接上传到 S3
PUT {presignedUrl}
Content-Type: image/png
Body: <file content>

// 4. 上传完成后通知服务端
POST /api/media/confirm-upload
{
  "mediaId": "media-xyz",
  "traceId": "trace-abc"
}
```

**下载流程**：
```typescript
// 1. 请求下载 URL
GET /api/media/{mediaId}/download-url

// 2. 服务端验证权限并生成 URL
const presignedUrl = await s3Client.getSignedUrl('getObject', {
  Bucket: 'langfuse-media',
  Key: `media/${projectId}/${mediaId}/${fileName}`,
  Expires: 900
});

// 3. 客户端使用 URL 下载
GET {presignedUrl}
```

**安全性**：
- 所有文件私有存储
- 必须通过预签名 URL 访问
- URL 短期有效（15 分钟 - 24 小时）
- 服务端验证用户权限后才生成 URL

## 5. 数据一致性保证

### 5.1 跨数据库事务

Langfuse 不使用分布式事务，而是通过以下策略保证最终一致性：

**写入流程**：
```
1. 写入 PostgreSQL（元数据）→ 成功
2. 发送消息到 Redis 队列 → 异步
3. Worker 从队列消费 → 写入 ClickHouse
4. 重试机制保证最终写入成功
```

**一致性级别**：
- PostgreSQL：强一致性（ACID）
- ClickHouse：最终一致性
- Redis：最终一致性

### 5.2 级联删除策略

**软删除**：
- Project 表有 `deletedAt` 字段
- 软删除时只标记，不立即删除数据
- 保留期后通过后台任务清理

**硬删除流程**：
```
1. 标记项目为删除状态（软删除）
2. 异步任务清理：
   - PostgreSQL：级联删除所有关联数据
   - ClickHouse：删除该项目的所有分区
   - Redis：清除缓存
   - S3：删除该项目的所有对象
3. 删除确认
```

**级联关系**：
```sql
-- Prisma Schema 中定义
Project @relation(fields: [...], onDelete: Cascade)
```
- 删除 Organization → 删除所有 Project
- 删除 Project → 删除所有相关数据
- 删除 User → 删除所有关联关系

### 5.3 数据备份策略

**PostgreSQL 备份**：
- 每日全量备份
- 每小时增量备份
- WAL 归档，支持时间点恢复
- 备份存储在 S3

**ClickHouse 备份**：
- 按分区备份
- 每周全量备份
- 利用 S3 存储备份
- 压缩后存储，节省成本

**Redis 备份**：
- RDB 快照：每 6 小时
- AOF 日志：实时
- 备份存储在 S3

## 6. 性能优化策略

### 6.1 查询优化

**PostgreSQL 查询优化**：
1. 使用 Prisma 的连接池
2. 避免 N+1 查询（使用 `include` 预加载关联）
3. 复杂查询使用原生 SQL
4. 合理使用索引
5. 定期 `ANALYZE` 更新统计信息

**ClickHouse 查询优化**：
1. 利用分区裁剪（按时间范围查询）
2. 使用 `PREWHERE` 提前过滤
3. 避免 `SELECT *`，只查询需要的列
4. 使用物化视图加速聚合查询
5. 合理设置 `max_threads` 参数

### 6.2 写入优化

**批量写入**：
- PostgreSQL：使用 Prisma 的 `createMany`
- ClickHouse：批量插入（默认 10,000 条）
- 减少网络往返次数

**异步写入**：
- 核心数据同步写入 PostgreSQL
- 追踪数据异步写入 ClickHouse
- 通过队列缓冲，平滑峰值

**连接复用**：
- 使用连接池
- Keep-Alive 保持长连接
- 减少连接建立开销

### 6.3 缓存优化

**多级缓存**：
1. 应用内存缓存：热点数据
2. Redis 缓存：API 密钥、配置
3. CDN 缓存：静态资源

**缓存预热**：
- 启动时加载常用配置
- 定期刷新热点数据

**缓存失效**：
- 主动失效：数据更新时
- 被动失效：TTL 过期
- 批量失效：使用 Key 前缀

## 7. 监控与维护

### 7.1 监控指标

**数据库监控**：
- 连接数
- QPS（每秒查询数）
- 慢查询
- 磁盘使用率
- 复制延迟

**ClickHouse 监控**：
- 插入速率
- 查询延迟
- 磁盘空间
- 分区数量
- 压缩率

**Redis 监控**：
- 内存使用
- 命中率
- 键空间大小
- 队列长度
- 慢查询

**S3 监控**：
- 存储用量
- 请求数
- 带宽使用
- 错误率

### 7.2 日志系统

**应用日志**：
- 结构化日志（JSON）
- 分级：DEBUG, INFO, WARN, ERROR
- 包含请求 ID、用户 ID、项目 ID
- 集中收集到日志系统

**数据库日志**：
- PostgreSQL：慢查询日志
- ClickHouse：查询日志
- Redis：慢命令日志

### 7.3 定期维护

**PostgreSQL**：
```sql
-- 每周执行
VACUUM ANALYZE;

-- 重建索引（必要时）
REINDEX DATABASE langfuse;
```

**ClickHouse**：
```sql
-- 合并分区（自动）
OPTIMIZE TABLE traces FINAL;

-- 清理旧数据（TTL 自动）
ALTER TABLE traces MODIFY TTL timestamp + INTERVAL 90 DAY;
```

**Redis**：
- 监控内存使用
- 定期清理过期键
- 备份 RDB 文件

## 8. 数据迁移与版本管理

### 8.1 Schema 迁移

**Prisma Migrate**：
```bash
# 创建迁移
pnpm prisma migrate dev --name add_new_field

# 应用迁移（生产）
pnpm prisma migrate deploy

# 回滚迁移
pnpm prisma migrate resolve --rolled-back migration_name
```

**ClickHouse 迁移**：
- 使用 ALTER TABLE 添加字段
- 新旧表并存，逐步迁移数据
- 使用物化视图进行数据转换

### 8.2 数据版本控制

**迁移文件**：
```
packages/shared/prisma/migrations/
├── 20231201_init/
│   └── migration.sql
├── 20231205_add_ai_features/
│   └── migration.sql
└── migration_lock.toml
```

**版本追踪**：
- 每个迁移有唯一 ID
- 记录在 `_prisma_migrations` 表
- 保证迁移顺序执行

## 9. 安全性设计

### 9.1 数据加密

**传输加密**：
- PostgreSQL：SSL/TLS 连接
- ClickHouse：HTTPS
- Redis：TLS（可选）
- S3：HTTPS

**存储加密**：
- PostgreSQL：支持表空间加密
- S3：服务端加密（SSE-S3 或 SSE-KMS）
- 敏感字段：应用层加密（如 API 密钥）

### 9.2 访问控制

**数据库用户权限**：
- 最小权限原则
- 应用使用专用数据库用户
- 限制 DDL 权限
- 定期审计权限

**S3 访问策略**：
- IAM 角色授权
- Bucket 策略限制
- 预签名 URL 时间限制
- CORS 配置

### 9.3 数据隔离

**多租户隔离**：
- 所有数据通过 `project_id` 隔离
- 查询必须包含项目 ID 过滤
- Row-Level Security（可选）

**敏感数据保护**：
- 密码：bcrypt 哈希
- API 密钥：单向哈希 + 部分显示
- LLM 密钥：加密存储
- PII 数据：可配置脱敏

## 10. 扩展性设计

### 10.1 水平扩展

**应用层**：
- 无状态设计
- 负载均衡
- 多实例部署

**数据库层**：
- PostgreSQL：读写分离、主从复制
- ClickHouse：集群部署、分片
- Redis：集群模式、哨兵模式

### 10.2 垂直扩展

**单机优化**：
- 增加 CPU 核心数
- 增加内存容量
- 使用 SSD 存储
- 调整数据库参数

### 10.3 存储扩展

**ClickHouse 分片**：
```sql
-- 按项目 ID 分片
CREATE TABLE traces_distributed AS traces
ENGINE = Distributed(cluster, database, traces, cityHash64(project_id));
```

**S3 无限扩展**：
- 对象存储天然支持扩展
- 按需付费
- 无需预先规划容量

## 附录

### A. 环境变量配置

```bash
# PostgreSQL
DATABASE_URL=postgresql://user:pass@host:5432/langfuse
DIRECT_URL=postgresql://user:pass@host:5432/langfuse
SHADOW_DATABASE_URL=postgresql://user:pass@host:5432/langfuse_shadow

# ClickHouse
CLICKHOUSE_URL=http://clickhouse-host:8123
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=password
CLICKHOUSE_CLUSTER_ENABLED=false

# Redis
REDIS_CONNECTION_STRING=redis://localhost:6379
REDIS_ENABLE_AUTO_PIPELINING=true
REDIS_KEY_PREFIX=langfuse:
# 或集群模式
REDIS_CLUSTER_NODES=host1:6379,host2:6379,host3:6379
# 或哨兵模式
REDIS_SENTINEL_NODES=host1:26379,host2:26379
REDIS_SENTINEL_MASTER_NAME=mymaster

# S3
S3_ENDPOINT=https://s3.amazonaws.com
S3_REGION=us-east-1
S3_ACCESS_KEY_ID=your-access-key
S3_SECRET_ACCESS_KEY=your-secret-key
S3_BUCKET_NAME=langfuse-media
S3_FORCE_PATH_STYLE=false

# S3 事件存储（可选，用于批量导出）
S3_EVENT_UPLOAD_ENABLED=true
S3_EVENT_UPLOAD_BUCKET=langfuse-exports
S3_EVENT_UPLOAD_REGION=us-east-1
S3_EVENT_UPLOAD_ENDPOINT=https://s3.amazonaws.com
S3_EVENT_UPLOAD_ACCESS_KEY_ID=your-access-key
S3_EVENT_UPLOAD_SECRET_ACCESS_KEY=your-secret-key
```

### B. 数据量级参考

**典型项目规模**：

| 规模 | 每日 Traces | 每日 Observations | 每日 Scores | PostgreSQL | ClickHouse | Redis |
|------|-------------|-------------------|-------------|------------|------------|-------|
| 小型 | < 1K | < 10K | < 1K | < 1GB | < 10GB | < 100MB |
| 中型 | 1K - 100K | 10K - 1M | 1K - 100K | 1-10GB | 10GB - 1TB | 100MB - 1GB |
| 大型 | 100K - 1M | 1M - 10M | 100K - 1M | 10-100GB | 1TB - 10TB | 1-10GB |
| 超大型 | > 1M | > 10M | > 1M | > 100GB | > 10TB | > 10GB |

### C. 相关文档

- [数据架构组件图](./data-architecture.puml)
- [数据模型 ER 图](./data-model-er.puml)
- [数据流程图](./data-flow.puml)
- [Prisma Schema 完整定义](../../packages/shared/prisma/schema.prisma)
- [ClickHouse Schema](../../packages/shared/src/server/clickhouse/schema.ts)
- [Redis 队列配置](../../packages/shared/src/server/redis/)

---

**最后更新**: 2025-12-17
