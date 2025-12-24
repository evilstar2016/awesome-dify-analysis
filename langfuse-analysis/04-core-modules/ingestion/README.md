# Ingestion 模块

## 模块概述

Ingestion 模块是 Langfuse 的**数据摄取引擎**，负责接收来自 SDK 的跟踪数据（traces、observations、scores 等），并通过异步流水线将其存储到数据库中。该模块是整个系统的数据入口，设计目标是：
- **高吞吐量**：支持每秒数千个事件的摄取
- **异步处理**：API 快速返回，后台队列处理
- **数据可靠性**：S3 持久化 + Redis 去重缓存
- **灵活合并**：支持乱序到达的 create/update 事件

---

## 核心概念

### 1. **事件类型（Event Types）**

Ingestion 支持多种事件类型，分为4大类：

#### **Trace 事件**
- `trace-create`：创建 trace
- **作用**：初始化跟踪链路，包含 trace ID、metadata、tags 等

#### **Observation 事件**
- `span-create` / `span-update`：创建/更新 span
- `generation-create` / `generation-update`：创建/更新 generation
- `event-create`：创建 event（单点事件）
- `agent-create`, `tool-create`, `chain-create` 等：创建各类 observation
- **作用**：记录执行节点，包含输入/输出、时间、层级关系

#### **Score 事件**
- `score-create`：创建评分
- **作用**：为 trace/observation 添加评分，支持数值和分类评分

#### **Dataset Run Item 事件**
- `dataset-run-item-create`：创建数据集运行项
- **作用**：关联 dataset item 与 trace，用于 evaluation

### 2. **事件 ID 体系**

每个事件都有唯一标识符：
- **事件 ID（event.id）**：SDK 生成的客户端 ID
- **事件体 ID（eventBodyId）**：同一实体的所有事件共享
  - 对于 Trace：`eventBodyId = traceId`
  - 对于 Observation：`eventBodyId = observationId`
  - 对于 Score：`eventBodyId = scoreId`
- **文件 Key（fileKey）**：S3 中的唯一文件名，格式为 `{eventId}-{timestamp}`

### 3. **数据流架构**

```
┌─────────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  SDK Client │ ───> │  Ingestion   │ ───> │ S3 Storage   │ ───> │ Redis Queue  │
│  (Batch)    │      │  API         │      │ (Event File) │      │ (Job)        │
└─────────────┘      └──────────────┘      └──────────────┘      └──────────────┘
                             │                                            │
                             │ 207 Multi-Status                           ▼
                             ▼                                   ┌──────────────┐
                     ┌──────────────┐                            │   Worker     │
                     │  Response    │                            │  (Consumer)  │
                     │  (Success/   │                            └──────────────┘
                     │   Errors)    │                                    │
                     └──────────────┘                                    ▼
                                                                ┌──────────────┐
                                                                │  Download    │
                                                                │  from S3     │
                                                                └──────────────┘
                                                                         │
                                                                         ▼
                                                                ┌──────────────┐
                                                                │ Ingestion    │
                                                                │ Service      │
                                                                │ (Merge)      │
                                                                └──────────────┘
                                                                         │
                                                                         ▼
                                                        ┌────────────────┴────────────────┐
                                                        ▼                                 ▼
                                                ┌──────────────┐                 ┌──────────────┐
                                                │  ClickHouse  │                 │  PostgreSQL  │
                                                │  (Analytics) │                 │  (Metadata)  │
                                                └──────────────┘                 └──────────────┘
```

### 4. **事件合并策略（Event Merging）**

Ingestion 支持同一实体的多个事件合并：
- **Create 事件优先**：初始化所有字段
- **Update 事件覆盖**：只更新提供的字段
- **时间排序**：按事件 timestamp 排序后依次应用
- **非覆盖字段**：某些字段（如 `id`, `project_id`, `created_at`）不可覆盖

**示例场景**：
```
Event 1 (t=0): generation-create { id: "gen-1", input: "Hello", output: null }
Event 2 (t=1): generation-update { id: "gen-1", output: "World" }

合并结果: { id: "gen-1", input: "Hello", output: "World" }
```

### 5. **S3 存储结构**

事件文件按以下层级存储：
```
s3://bucket/{prefix}/{projectId}/{entityType}/{eventBodyId}/{fileKey}.json

示例：
s3://langfuse-events/prod/proj-123/observation/gen-456/evt-789-1734417600000.json
```

**优势**：
- **按项目隔离**：便于数据删除和访问控制
- **按实体分组**：同一 trace 的所有事件在同一目录
- **时间排序**：文件名包含时间戳，便于按序读取

### 6. **Queue 分片（Sharding）**

为了提高并发处理能力，Ingestion Queue 支持分片：
- **分片数量**：由 `LANGFUSE_INGESTION_QUEUE_SHARD_COUNT` 环境变量控制
- **分片键**：`{projectId}-{eventBodyId}`，确保同一实体的事件在同一分片
- **队列名称**：
  - Shard 0: `ingestion-queue`
  - Shard 1+: `ingestion-queue-1`, `ingestion-queue-2`, ...

**作用**：
- 避免 Redis 单队列瓶颈
- 保证同一实体的事件顺序处理

---

## 数据模型

### 1. **Ingestion Event Schema**

核心接口定义：`packages/shared/src/server/ingestion/types.ts`

```typescript
// 事件基础结构
{
  id: string;              // 事件唯一 ID
  type: string;            // 事件类型（如 "trace-create"）
  timestamp: string;       // 事件时间戳（ISO 8601）
  metadata?: object;       // 元数据
  body: {                  // 事件内容（根据 type 变化）
    // Trace body
    id?: string;
    name?: string;
    userId?: string;
    metadata?: object;
    tags?: string[];
    
    // Observation body
    traceId?: string;
    parentObservationId?: string;
    startTime?: string;
    endTime?: string;
    input?: any;
    output?: any;
    model?: string;
    usage?: { input: number, output: number, total: number };
    
    // Score body
    traceId?: string;
    observationId?: string;
    name: string;
    value: number | string;
    dataType: "NUMERIC" | "CATEGORICAL";
    comment?: string;
  }
}
```

### 2. **Queue Job 结构**

```typescript
// IngestionQueue Job Payload
{
  id: string;                    // Job ID
  timestamp: Date;               // Job 创建时间
  name: "IngestionJob";
  payload: {
    data: {
      type: string;              // trace | observation | score | dataset_run_item
      eventBodyId: string;       // 实体 ID
      fileKey: string;           // S3 文件 key
      skipS3List: boolean;       // 是否跳过 S3 list 操作（性能优化）
      forwardToEventsTable: boolean; // 是否转发到 events 表（实验性）
    },
    authCheck: {
      validKey: true;
      scope: {
        projectId: string;
        accessLevel: "project" | "scores";
      }
    }
  }
}
```

### 3. **ClickHouse 记录结构**

最终写入 ClickHouse 的记录（简化版）：

```typescript
// Trace Record
{
  id: string;
  project_id: string;
  name: string;
  user_id: string;
  session_id: string;
  metadata: Record<string, string>;
  tags: string[];
  bookmarked: boolean;
  public: boolean;
  timestamp: number;           // 微秒时间戳
  created_at: number;
  updated_at: number;
  event_ts: number;
  is_deleted: 0 | 1;
}

// Observation Record
{
  id: string;
  trace_id: string;
  parent_observation_id: string;
  project_id: string;
  type: string;              // SPAN | GENERATION | EVENT
  name: string;
  start_time: number;
  end_time: number;
  completion_start_time: number;
  model: string;
  model_parameters: object;
  input: any;
  output: any;
  metadata: Record<string, string>;
  level: string;             // DEFAULT | DEBUG | WARNING | ERROR
  status_message: string;
  version: string;
  provided_usage_details: object;
  provided_cost_details: object;
  usage_details: object;     // 增强后的 usage（包含 token 分解）
  cost_details: object;      // 增强后的 cost
  total_cost: number;
  prompt_id: string;
  prompt_name: string;
  prompt_version: number;
  created_at: number;
  updated_at: number;
  event_ts: number;
  is_deleted: 0 | 1;
}

// Score Record
{
  id: string;
  project_id: string;
  trace_id: string;
  observation_id: string;
  name: string;
  value: number;
  string_value: string;
  data_type: string;         // NUMERIC | CATEGORICAL
  source: string;            // API | ANNOTATION | EVAL
  comment: string;
  config_id: string;
  queue_id: string;
  execution_trace_id: string;
  metadata: Record<string, string>;
  timestamp: number;
  created_at: number;
  updated_at: number;
  event_ts: number;
  is_deleted: 0 | 1;
}
```

---

## 核心功能

### 1. **API 端点处理**

**文件**：`web/src/pages/api/public/ingestion.ts`

**核心函数**：`handler(req, res)`

**处理流程**：
1. **认证**：
   - 调用 `ApiAuthService.verifyAuthHeaderAndReturnScope()`
   - 验证 API key 和权限范围
   - 检查 ingestion 是否被暂停（usage threshold）

2. **限流**：
   - 调用 `RateLimitService.rateLimitRequest()`
   - 按项目 + endpoint 限流

3. **验证**：
   - 使用 Zod schema 验证请求体
   - 批量验证所有事件的格式

4. **批处理**：
   - 调用 `processEventBatch(events, authCheck)`
   - 返回 207 Multi-Status 响应

### 2. **批量事件处理**

**文件**：`packages/shared/src/server/ingestion/processEventBatch.ts`

**核心函数**：`processEventBatch(input, authCheck, options)`

**参数**：
- `input`: 事件数组
- `authCheck`: 认证信息
- `options`: 可选配置
  - `delay`: 延迟时间
  - `source`: 来源（"api" 或 "otel"）
  - `isLangfuseInternal`: 是否内部事件
  - `forwardToEventsTable`: 是否转发到 events 表

**处理步骤**：

**步骤 1：验证和认证**
- 使用 `createIngestionEventSchema()` 验证每个事件
- 调用 `isAuthorized()` 检查访问权限
- 过滤 SDK_LOG 事件（仅记录不处理）

**步骤 2：排序和分组**
- 调用 `sortBatch()` 按时间和事件类型排序
  - Create 事件优先
  - Update 事件排后
  - 同类事件按 timestamp 升序
- 按 `eventBodyId` 分组

**步骤 3：S3 上传**
- 调用 `StorageService.uploadJson()`
- 路径：`{prefix}/{projectId}/{entityType}/{eventBodyId}/{fileKey}.json`
- 并发上传所有事件
- 检测 S3 SlowDown 错误并标记项目

**步骤 4：队列入队**
- 获取分片队列：`IngestionQueue.getInstance({ shardingKey })`
- 对每个 eventBodyId 组：
  - 检查采样配置：`isTraceIdInSample()`（可能跳过事件）
  - 添加 job：`queue.add(QueueJobs.IngestionJob, payload, { delay })`
  - delay 计算考虑日期边界（避免重复）

**步骤 5：返回结果**
- 调用 `aggregateBatchResult()` 汇总成功/失败
- 返回 `{ successes: [], errors: [] }`

### 3. **队列管理**

**文件**：`packages/shared/src/server/redis/ingestionQueue.ts`

**核心类**：`IngestionQueue` 和 `SecondaryIngestionQueue`

**IngestionQueue 方法**：

- **`getInstance({ shardingKey, shardName })`**：
  - 计算分片索引：`getShardIndex(shardingKey, SHARD_COUNT)`
  - 返回 BullMQ Queue 实例
  - 配置：`removeOnComplete=true`, `attempts=6`, `exponential backoff`

- **`getShardNames()`**：
  - 返回所有分片队列名称数组

**SecondaryIngestionQueue**：
- 用于 S3 SlowDown 的项目
- 单例队列，独立处理避免影响主队列
- 配置：`removeOnComplete=true`, `removeOnFail=100_000`, `attempts=5`, `exponential backoff`

### 4. **Worker 处理**

**文件**：`worker/src/queues/ingestionQueue.ts`

**核心函数**：`ingestionQueueProcessorBuilder(enableRedirectToSecondaryQueue)`

**处理流程**：

**步骤 1：记录元数据**
- 写入 `BlobStorageFileLog` 表（如果启用）
- 记录文件路径、entity 信息

**步骤 2：去重检查**
- 使用 Redis 检查 `recently-processed` key
- 如果存在则跳过（5 分钟缓存）

**步骤 3：下载事件**
- 两种模式：
  - **Direct download**（`skipS3List=true`）：直接下载指定文件
  - **List & download**：列出目录下所有文件并批量下载
- 并发下载：`LANGFUSE_S3_CONCURRENT_READS` 控制并发数

**步骤 4：合并和写入**
- 调用 `IngestionService.mergeAndWrite()`
- 传入所有事件和首次写入时间

**步骤 5：设置缓存**
- 将处理过的文件 key 存入 Redis（5 分钟过期）

**错误处理**：
- 检测 S3 SlowDown：调用 `markProjectS3Slowdown()`
- 记录错误并抛出（触发重试）

### 5. **事件合并与存储**

**文件**：`worker/src/services/IngestionService/index.ts`

**核心类**：`IngestionService`

**主方法**：`mergeAndWrite(eventType, projectId, eventBodyId, createdAt, events, forwardToEventsTable)`

**处理分支**：
- `eventType === "trace"` → `processTraceEventList()`
- `eventType === "observation"` → `processObservationEventList()`
- `eventType === "score"` → `processScoreEventList()`
- `eventType === "dataset_run_item"` → `processDatasetRunItemEventList()`

#### **Trace 处理流程**

**函数**：`processTraceEventList(params)`

**步骤**：
1. **时间排序**：调用 `toTimeSortedEventList()` 按 timestamp 排序
2. **读取现有记录**：
   - 调用 `getClickhouseRecord()` 查询 ClickHouse
   - 查询条件：`id = eventBodyId AND project_id = projectId`
3. **合并逻辑**：
   - 调用 `mergeTracesBasedOnTimestamps()` 合并所有事件
   - 使用 `overwriteObject()` 按字段合并
   - 非覆盖字段：`id`, `project_id`, `created_at`
4. **数据增强**：
   - 转换 metadata 为扁平结构
   - 合并 tags 并去重
5. **写入 ClickHouse**：
   - 调用 `clickHouseWriter.addToQueue(TableName.Traces, record)`

#### **Observation 处理流程**

**函数**：`processObservationEventList(params)`

**步骤**：
1. **时间排序**：同 Trace
2. **读取现有记录**：查询 ClickHouse observations 表
3. **合并逻辑**：
   - 调用 `mergeObservationsBasedOnTimestamps()`
   - 特殊处理：`completion_start_time`, `end_time`
4. **数据增强**：
   - **Model 匹配**：调用 `findModel()` 查找匹配的模型
   - **Prompt 查找**：调用 `promptService.getPrompt()` 获取 prompt 信息
   - **Usage 增强**：
     - 如果没有 usage，调用 `tokenCountAsync()` 计算 token
     - 调用 `matchPricingTier()` 计算 cost
   - **Metadata 扁平化**：转换为 path-based arrays
5. **写入 ClickHouse**：
   - 调用 `clickHouseWriter.addToQueue(TableName.Observations, record)`
6. **转换为 Trace**（如果需要）：
   - 调用 `convertTraceToStagingObservation()` 生成 staging_observations 记录

#### **Score 处理流程**

**函数**：`processScoreEventList(params)`

**步骤**：
1. **时间排序**：同上
2. **读取现有记录**：查询 ClickHouse scores 表
3. **验证和转换**：
   - 调用 `validateAndInflateScore()` 验证 score body
   - 支持 numeric 和 categorical 两种类型
4. **合并逻辑**：
   - 调用 `mergeScoresBasedOnTimestamps()`
   - 允许更新 value, comment, metadata
5. **写入 ClickHouse**：
   - 调用 `clickHouseWriter.addToQueue(TableName.Scores, record)`

#### **Dataset Run Item 处理流程**

**函数**：`processDatasetRunItemEventList(params)`

**步骤**：
1. **查询关联数据**：
   - 查询 `datasetRuns` 表获取 run 元数据
   - 调用 `getDatasetItemById()` 获取 dataset item 数据
2. **数据增强**：
   - 合并 run 信息（name, description, metadata, createdAt）
   - 合并 item 信息（version, input, expectedOutput, metadata）
3. **写入 ClickHouse**：
   - 调用 `clickHouseWriter.addToQueue(TableName.DatasetRunItems, record)`

### 6. **ClickHouse 批量写入**

**文件**：`worker/src/services/ClickhouseWriter/index.ts`

**核心类**：`ClickhouseWriter`

**方法**：

- **`addToQueue(table, record)`**：
  - 将记录添加到内存队列
  - 队列按表分组
  - 当队列长度达到 batchSize 时自动触发 flush

- **`flush(tableName, fullQueue)`**：
  - 周期性调用（`CLICKHOUSE_FLUSH_INTERVAL_MS`）
  - 批量写入指定表的记录
  - 调用 `writeToClickhouse({ table, records })`
  - 支持错误重试、批次拆分和记录截断

**优势**：
- 减少网络开销（批量写入）
- 提高写入吞吐量
- 自动处理失败重试

---

## 技术架构

### 1. **分层架构**

```
┌─────────────────────────────────────────────────────────────┐
│                      API Layer (Web)                         │
│  - Authentication & Rate Limiting                            │
│  - Request Validation (Zod)                                  │
│  - Response Formatting (207 Multi-Status)                    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Processing Layer (Shared)                  │
│  - processEventBatch(): 批量处理核心逻辑                       │
│  - Event Validation & Sorting                                │
│  - S3 Storage: 事件持久化                                     │
│  - Queue Enqueue: BullMQ job 创建                             │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Queue Layer (Redis)                      │
│  - Sharded Ingestion Queues                                  │
│  - Job Scheduling & Retry                                    │
│  - Deduplication Cache                                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Worker Layer (Consumer)                    │
│  - ingestionQueueProcessor: Job 处理器                        │
│  - S3 Download: 批量下载事件文件                              │
│  - Event Merging: 合并 create/update 事件                    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Service Layer (Ingestion)                   │
│  - IngestionService: 核心合并和增强逻辑                        │
│  - Model Matching & Tokenization                             │
│  - Prompt Lookup & Usage Calculation                         │
│  - Metadata Flattening                                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Storage Layer (Database)                   │
│  - ClickHouse: 批量写入（traces, observations, scores）       │
│  - PostgreSQL: 关联查询（prompts, models, datasets）         │
└─────────────────────────────────────────────────────────────┘
```

### 2. **关键技术栈**

| 组件 | 技术 | 作用 |
|------|------|------|
| **API** | Next.js API Routes | HTTP 入口，快速响应 |
| **验证** | Zod | 事件 schema 验证 |
| **认证** | Custom ApiAuthService | API key 验证 |
| **限流** | RateLimitService | 防止滥用 |
| **存储** | AWS S3 / Azure Blob | 事件持久化 |
| **队列** | BullMQ (Redis) | 异步任务管理 |
| **Worker** | Node.js (worker container) | 后台处理 |
| **数据库** | ClickHouse + PostgreSQL | OLAP + OLTP |
| **批量写入** | ClickhouseWriter | 性能优化 |

### 3. **性能优化策略**

#### **API 层优化**
- **快速响应**：API 只负责验证和入队，立即返回
- **批量处理**：单次请求支持多个事件（减少 HTTP 开销）
- **限流保护**：基于 Redis 的滑动窗口限流

#### **存储层优化**
- **S3 持久化**：API 层上传，Worker 层下载（解耦）
- **路径设计**：按 `projectId/entityType/eventBodyId` 分组（便于查询和删除）
- **SlowDown 检测**：动态切换到 Secondary Queue

#### **队列层优化**
- **分片队列**：避免 Redis 单队列瓶颈
- **去重缓存**：Redis key 缓存已处理文件（5 分钟）
- **延迟入队**：日期边界附近延迟处理（避免重复）

#### **Worker 层优化**
- **批量下载**：并发下载多个 S3 文件
- **跳过 List**：直接下载已知文件（OTEL 优化）
- **事件合并**：内存中合并多个事件（减少 DB 写入）

#### **数据库层优化**
- **批量写入**：ClickhouseWriter 定期 flush（减少 INSERT 次数）
- **异步写入**：Worker 写入 ClickHouse，不阻塞 API
- **分区表**：ClickHouse 按 `project_id` 和日期分区

---

## 使用场景

### 1. **SDK 批量上报**

**场景**：应用程序通过 SDK 批量发送 traces

```typescript
// SDK 示例（伪代码）
langfuse.flush(); // 触发批量上传

// 内部实现
const batch = {
  batch: [
    { id: "evt-1", type: "trace-create", body: { id: "trace-1", ... } },
    { id: "evt-2", type: "span-create", body: { id: "span-1", traceId: "trace-1", ... } },
    { id: "evt-3", type: "generation-create", body: { id: "gen-1", traceId: "trace-1", ... } },
  ]
};
fetch("/api/public/ingestion", { method: "POST", body: JSON.stringify(batch) });
```

**Ingestion 处理**：
1. API 验证所有事件格式
2. 上传 3 个文件到 S3
3. 创建 3 个 queue jobs（可能在不同分片）
4. 返回 `{ successes: [{ id: "evt-1", status: 200 }, ...], errors: [] }`
5. Worker 异步处理，合并写入 ClickHouse

### 2. **乱序事件处理**

**场景**：Update 事件先于 Create 事件到达

```typescript
// 时间线
t=0: generation-update 到达（evt-2）
t=1: generation-create 到达（evt-1）
```

**Ingestion 处理**：
1. 两个事件都上传到 S3 的同一目录：`proj-123/observation/gen-1/`
2. Worker 从 S3 下载所有文件
3. `IngestionService` 按 timestamp 排序：`[evt-1, evt-2]`
4. 依次合并：先 create，再 update
5. 最终结果正确

### 3. **高频更新场景**

**场景**：同一 generation 快速更新多次（如 streaming）

```typescript
// 快速发送 10 次 update
for (let i = 0; i < 10; i++) {
  langfuse.generation.update({ id: "gen-1", output: chunk[i] });
}
```

**Ingestion 优化**：
1. 所有事件上传到同一目录：`proj-123/observation/gen-1/`
2. Queue 使用相同 sharding key，确保顺序处理
3. Worker 一次性下载所有 10 个文件
4. 合并为单条记录（只有最后的 output）
5. 单次写入 ClickHouse（减少 10 倍写入）

### 4. **大规模并发摄取**

**场景**：1000 个并发请求，每个请求 100 个事件

**Ingestion 扩展**：
1. **API 层**：多个 Web 实例（负载均衡）
2. **S3 层**：高并发上传（无瓶颈）
3. **Queue 层**：8 个分片队列（分散压力）
4. **Worker 层**：多个 Worker 实例（水平扩展）
5. **DB 层**：ClickHouse 分布式表（分片写入）

**吞吐量**：
- API 响应：< 200ms（仅验证和入队）
- 总吞吐：100,000 events/sec（理论值）

### 5. **错误恢复**

**场景**：Worker 崩溃，部分事件未处理

**恢复机制**：
1. BullMQ 自动重试（最多 6 次，指数退避）
2. S3 文件持久化（不丢失数据）
3. Redis 去重缓存防止重复处理
4. 失败任务记录到 `failed` 队列（可手动重放）

---

## 配置项

### 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `LANGFUSE_S3_EVENT_UPLOAD_BUCKET` | - | S3 bucket 名称 |
| `LANGFUSE_S3_EVENT_UPLOAD_PREFIX` | `""` | S3 路径前缀 |
| `LANGFUSE_S3_EVENT_UPLOAD_REGION` | `us-east-1` | AWS 区域 |
| `LANGFUSE_S3_CONCURRENT_READS` | `5` | Worker 并发下载数 |
| `LANGFUSE_INGESTION_QUEUE_SHARD_COUNT` | `1` | 队列分片数 |
| `LANGFUSE_INGESTION_QUEUE_DELAY_MS` | `5000` | 默认入队延迟 |
| `LANGFUSE_ENABLE_REDIS_SEEN_EVENT_CACHE` | `false` | 是否启用去重缓存 |
| `LANGFUSE_ENABLE_BLOB_STORAGE_FILE_LOG` | `false` | 是否记录文件日志 |
| `LANGFUSE_EXPERIMENT_INSERT_INTO_EVENTS_TABLE` | `false` | 是否写入 events 表（实验性） |
| `CLICKHOUSE_FLUSH_INTERVAL_MS` | `5000` | ClickHouse 批量写入间隔 |

### Queue 配置

```typescript
// Ingestion Queue 配置（BullMQ）
{
  removeOnComplete: true,        // 完成后删除 job
  removeOnFail: 100_000,         // 保留最近 10 万失败 job
  attempts: 6,                   // 最多重试 6 次
  backoff: {
    type: "exponential",         // 指数退避
    delay: 5000                  // 初始延迟 5 秒
  }
}
```

---

## 目录结构

```
ingestion/
├── web/src/pages/api/public/
│   └── ingestion.ts                    # API 入口（认证、验证、批处理）
│
├── packages/shared/src/server/
│   ├── ingestion/
│   │   ├── types.ts                    # 事件 schema 定义（Zod）
│   │   ├── processEventBatch.ts        # 核心批处理逻辑
│   │   ├── sampling.ts                 # 采样配置
│   │   ├── modelMatch.ts               # Model 匹配逻辑
│   │   └── validateAndInflateScore.ts  # Score 验证
│   │
│   ├── redis/
│   │   ├── ingestionQueue.ts           # 队列管理（分片）
│   │   └── otelIngestionQueue.ts       # OTEL 专用队列
│   │
│   └── services/
│       └── StorageService.ts           # S3 上传/下载封装
│
└── worker/src/
    ├── queues/
    │   ├── ingestionQueue.ts           # Worker processor（S3 下载、调度）
    │   └── otelIngestionQueue.ts       # OTEL processor
    │
    └── services/
        ├── IngestionService/
        │   ├── index.ts                # 核心合并逻辑（主类）
        │   └── utils.ts                # 辅助函数（合并、转换）
        │
        └── ClickhouseWriter.ts         # 批量写入管理
```

---

## 常见问题

### 1. **为什么使用 S3 而不是直接写数据库？**

**答**：
- **解耦**：API 快速响应，不阻塞客户端
- **持久化**：事件永久保存，支持审计和重放
- **削峰**：S3 作为缓冲层，避免数据库过载
- **成本**：S3 存储成本低，适合大规模数据

### 2. **如何保证事件不丢失？**

**答**：
- **S3 持久化**：API 成功上传后即持久化
- **Queue 重试**：BullMQ 自动重试失败任务（6 次）
- **去重缓存**：避免重复处理同一文件
- **Monitoring**：失败任务可在 BullMQ dashboard 查看

### 3. **如何处理大量 Update 事件？**

**答**：
- **批量下载**：Worker 一次性下载所有文件
- **内存合并**：在内存中合并所有 update
- **单次写入**：最终只写一条记录到 ClickHouse
- **性能**：相比逐个处理，减少 90% 数据库写入

### 4. **为什么需要队列分片？**

**答**：
- **瓶颈**：单队列 Redis 性能有限（约 10k ops/s）
- **分片**：多队列并行处理，线性扩展
- **顺序**：同一 `eventBodyId` 在同一分片，保证顺序
- **配置**：根据负载调整 `LANGFUSE_INGESTION_QUEUE_SHARD_COUNT`

### 5. **如何调试 Ingestion 问题？**

**答**：
1. **API 响应**：检查返回的 `errors` 数组
2. **S3 文件**：验证文件是否上传成功
3. **Queue Dashboard**：查看 BullMQ job 状态
4. **Worker 日志**：查看 Worker 处理日志
5. **ClickHouse 查询**：验证数据是否写入

### 6. **Update 事件必须等 Create 事件吗？**

**答**：
- **不需要**：Ingestion 支持乱序处理
- **自动排序**：Worker 下载所有文件后按时间排序
- **合并逻辑**：即使 update 先到，最终结果也正确
- **注意**：过早的 update（超过缓存时间）可能丢失

### 7. **如何限制 Ingestion 速率？**

**答**：
- **API 限流**：`RateLimitService` 按项目限流
- **Usage 限制**：超过 threshold 自动暂停 ingestion
- **Queue 延迟**：日期边界附近延迟处理
- **Worker 限速**：控制 Worker 实例数量

### 8. **S3 SlowDown 怎么处理？**

**答**：
- **检测**：`isS3SlowDownError()` 识别 503 错误
- **标记**：`markProjectS3Slowdown()` 标记项目（Redis）
- **切换**：后续请求路由到 `SecondaryIngestionQueue`
- **恢复**：标记自动过期（30 分钟）

### 9. **ClickHouse 写入失败怎么办？**

**答**：
- **重试**：ClickhouseWriter 自动重试
- **队列**：失败记录回到 BullMQ 队列重试
- **监控**：设置 ClickHouse 写入失败告警
- **恢复**：从 S3 重放事件

### 10. **如何迁移历史事件？**

**答**：
- **脚本**：`worker/src/scripts/replayIngestionEvents/`
- **步骤**：
  1. 列出 S3 文件
  2. 批量创建 queue jobs
  3. Worker 自动处理
- **限制**：按时间范围、项目过滤
- **监控**：观察 queue 深度和处理速度

---

## 总结

Ingestion 模块是 Langfuse 的**数据摄取核心**，通过**异步流水线 + S3 持久化 + 队列分片**实现高吞吐、高可靠的事件摄取。关键设计包括：

1. **快速响应**：API 仅验证和入队（< 200ms）
2. **持久化**：S3 作为事件源（Event Source）
3. **异步处理**：Worker 后台合并和写入
4. **灵活合并**：支持乱序 create/update 事件
5. **水平扩展**：队列分片 + 多 Worker 实例

该模块为 Langfuse 的**实时监控**、**性能分析**、**成本追踪**等功能提供了坚实的数据基础。
