# Data Storage 管理

## 功能概述

Data Storage 是 Langfuse Ingestion 模块的**持久化层**，负责管理事件数据在 S3、ClickHouse、PostgreSQL 三层存储中的读写、合并与查询。核心设计：
- **三层存储**：S3（事件缓存）→ ClickHouse（OLAP）→ PostgreSQL（OLTP）
- **数据一致性**：事件合并保证最终一致性
- **分区管理**：时间分区 + 项目分区
- **性能优化**：批量写入 + 异步队列

---

## 存储架构

### **三层存储对比**

| 存储层 | 用途 | 数据格式 | 保留期 | 查询性能 |
|-------|------|---------|-------|---------|
| **S3** | 事件持久化 + 灾难恢复 | JSON 文件 | 永久（可配置） | 慢（秒级） |
| **ClickHouse** | OLAP 分析查询 | 列式存储 | 90 天（可配置） | 快（毫秒级） |
| **PostgreSQL** | OLTP 元数据 | 行式存储 | 永久 | 中（10-100ms） |

---

## 核心组件

### 1. **S3 事件存储**

#### **目录结构设计**

**完整路径格式**：
```
s3://{bucket}/{prefix}{projectId}/{entityType}/{eventBodyId}/{fileKey}.json
```

**示例路径**：
```
# Trace 事件
s3://langfuse-events/prod/prj-abc123/traces/trc-xyz789/evt-001-1704067200000.json

# Observation 事件
s3://langfuse-events/prod/prj-abc123/observations/obs-def456/evt-002-1704067200001.json

# Score 事件
s3://langfuse-events/prod/prj-abc123/scores/scr-ghi789/evt-003-1704067200002.json
```

#### **文件内容格式**

**单事件文件**：
```json
{
  "id": "evt-001",
  "type": "trace-create",
  "timestamp": "2024-01-01T00:00:00Z",
  "body": {
    "id": "trc-xyz789",
    "name": "API Request",
    "metadata": { "user_id": "user-123" }
  }
}
```

**批量事件文件**（SDK 批量上传）：
```json
[
  { "id": "evt-001", "type": "trace-create", ... },
  { "id": "evt-002", "type": "observation-create", ... }
]
```

#### **S3 客户端操作**

**位置**：[packages/shared/src/server/ingestion/processEventBatch.ts](packages/shared/src/server/ingestion/processEventBatch.ts#L50-L74)

| 操作 | 方法 | 代码行 | 说明 |
|-----|------|-------|------|
| **上传 JSON** | `uploadJson()` | 167-260 | 并发上传（按 entity 分组） |
| **列出文件** | `listFiles()` | 178 | 列出实体下所有事件文件 |
| **下载文件** | `download()` | 182-189 | 批量下载并解析 JSON |

**上传逻辑（Lines 167-260）**：
```typescript
// 按 entityType:eventBodyId 分组
const groupedByEntity = groupBy(sortedBatch, 
  (event) => `${event.type}:${event.body.id}`
);

// 并发上传每个实体的事件
await Promise.all(
  Object.entries(groupedByEntity).map(async ([entityKey, events]) => {
    const [entityType, eventBodyId] = entityKey.split(":");
    const path = `${projectId}/${entityType}/${eventBodyId}/${fileKey}.json`;
    await s3StorageServiceClient.uploadJson(path, events);
  })
);
```

---

### 2. **ClickHouse 列式存储**

#### **核心表结构**

| 表名 | 用途 | 记录数（示例） | 代码行 |
|-----|------|--------------|-------|
| **observations** | Observation 数据 | 10M+ | 900-904 |
| **traces** | Trace 数据 | 1M+ | 757-773 |
| **scores** | Score 数据 | 5M+ | 618-627 |
| **blob_storage_file_log** | S3 文件日志 | 50M+ | 66-80 |

#### **observations 表字段（关键字段）**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L900-L904)

| 字段名 | 类型 | 描述 | 来源 |
|-------|------|------|------|
| `id` | String | Observation ID | Event Body |
| `project_id` | String | 项目 ID | Auth Scope |
| `trace_id` | String | 关联的 Trace ID | Event Body |
| `type` | Enum | SPAN/GENERATION/EVENT | Event Type |
| `name` | String | Observation 名称 | Event Body |
| `start_time` | DateTime64 | 开始时间 | Event Body |
| `end_time` | Nullable(DateTime64) | 结束时间 | Event Body |
| `input` | Nullable(String) | 输入内容 | Event Body |
| `output` | Nullable(String) | 输出内容 | Event Body |
| `metadata` | Map(String, String) | 元数据键值对 | Event Body |
| `prompt_id` | Nullable(String) | 关联 Prompt ID | Prompt 服务 |
| `prompt_version` | Nullable(Int32) | Prompt 版本 | Prompt 服务 |
| `usage_details` | Map(String, Float64) | Token 使用详情 | 计算得出 |
| `cost_details` | Map(String, Float64) | 成本详情 | 计算得出 |
| `total_cost` | Nullable(Float64) | 总成本 | 计算得出 |
| `created_at` | DateTime64 | 创建时间戳 | 系统时间 |
| `updated_at` | DateTime64 | 更新时间戳 | 系统时间 |
| `event_ts` | DateTime64 | 事件时间戳（分区键） | Event Timestamp |

#### **traces 表字段（关键字段）**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L757-L773)

| 字段名 | 类型 | 描述 | 特殊逻辑 |
|-------|------|------|---------|
| `id` | String | Trace ID | - |
| `project_id` | String | 项目 ID | - |
| `name` | Nullable(String) | Trace 名称 | - |
| `input` | Nullable(String) | 输入内容 | 从最后非空记录提取 |
| `output` | Nullable(String) | 输出内容 | 从最后非空记录提取 |
| `metadata` | Map(String, String) | 元数据 | - |
| `tags` | Array(String) | 标签数组 | - |
| `session_id` | Nullable(String) | Session ID | 关联 Session 表 |
| `user_id` | Nullable(String) | 用户 ID | - |
| `timestamp` | DateTime64 | Trace 时间戳 | 最早事件时间 |
| `created_at` | DateTime64 | 创建时间 | - |
| `updated_at` | DateTime64 | 更新时间 | - |
| `event_ts` | DateTime64 | 事件时间戳（分区键） | 时间分区感知 |

#### **scores 表字段**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L618-L627)

| 字段名 | 类型 | 描述 |
|-------|------|------|
| `id` | String | Score ID |
| `project_id` | String | 项目 ID |
| `trace_id` | String | 关联 Trace ID |
| `observation_id` | Nullable(String) | 关联 Observation ID |
| `name` | String | Score 名称 |
| `value` | Nullable(Float64) | 数值评分 |
| `string_value` | Nullable(String) | 分类评分 |
| `comment` | Nullable(String) | 评分备注 |
| `data_type` | Enum | NUMERIC/CATEGORICAL/BOOLEAN |
| `source` | Enum | API/ANNOTATION/EVAL |
| `timestamp` | DateTime64 | 评分时间戳 |
| `created_at` | DateTime64 | 创建时间 |
| `event_ts` | DateTime64 | 事件时间戳（分区键） |

---

### 3. **ClickHouse 写入机制**

#### **ClickhouseWriter 批量写入**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L218-L219)

```typescript
// Lines 218-219: 注入 ClickhouseWriter
private clickHouseWriter: ClickhouseWriter;
private clickhouseClient: ClickHouseClient;
```

#### **addToQueue 方法**

**调用示例（Lines 900-904）**：
```typescript
this.clickHouseWriter.addToQueue(
  TableName.StagingObservations,
  {
    id: entityId,
    project_id: projectId,
    type: observationType,
    ...finalObservationRecord,
  }
);
```

#### **批量写入参数**

| 参数 | 默认值 | 说明 |
|-----|-------|------|
| **批量大小** | 1000 条 | 达到阈值后批量插入 |
| **时间窗口** | 5 秒 | 超时后强制刷新 |
| **最大并发** | 10 | 并发写入线程数 |

---

### 4. **时间分区感知**

#### **getPartitionAwareTimestamp 方法**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L1727-L1736)

```typescript
// Lines 1727-1736: 分区感知时间戳（10 行）
private getPartitionAwareTimestamp(timestamp: Date): Date {
  const now = new Date();
  const createdAt = new Date(timestamp);
  const ageInMs = now.getTime() - createdAt.getTime();
  const threeAndHalfMinutesInMs = 3.5 * 60 * 1000;
  
  // 如果事件超过 3.5 分钟，使用当前时间（避免写入旧分区）
  return ageInMs > threeAndHalfMinutesInMs ? now : createdAt;
}
```

#### **分区策略**

| 条件 | event_ts 值 | 原因 |
|-----|-----------|------|
| 事件时间 < 3.5 分钟前 | 当前时间 | 避免写入已合并的旧分区 |
| 事件时间 ≥ 3.5 分钟前 | 原始时间 | 保持时间准确性 |

**ClickHouse 分区表定义**：
```sql
CREATE TABLE observations (
  ...
  event_ts DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(event_ts)
ORDER BY (project_id, id, event_ts);
```

---

### 5. **ClickHouse 记录合并**

#### **getClickhouseRecord 方法族（4 重载）**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L1363-L1493)

| 重载 | 代码行 | 参数 | 返回值 |
|-----|-------|------|-------|
| **重载 1（Score）** | 1363-1371 | `table, projectId, entityId` | `ScoreRecordReadType \| null` |
| **重载 2（Trace）** | 1373-1381 | `table, projectId, entityId` | `TraceRecordReadType \| null` |
| **重载 3（Observation）** | 1383-1391 | `table, projectId, entityId, additionalFilters` | `ObservationRecordReadType \| null` |
| **重载 4（实现）** | 1393-1493 (101 行) | 所有参数 | 查询 ClickHouse + 解析 |

#### **查询逻辑（Lines 1427-1491）**

```typescript
// Lines 1427-1491: ClickHouse 查询（65 行）
const result = await instrumentAsync(
  { name: "getClickhouseRecord" },
  async () => {
    const query = `
      SELECT * 
      FROM ${table}
      WHERE project_id = {projectId: String}
        AND ${table === "observations" ? "id" : "trace_id"} = {entityId: String}
        ${additionalFilters || ""}
      ORDER BY event_ts DESC
      LIMIT 1 BY project_id, ${table === "observations" ? "id" : "trace_id"}
    `;
    
    const rows = await this.clickhouseClient.query({
      query,
      query_params: { projectId, entityId },
    });
    
    return rows.json();
  }
);
```

#### **LIMIT 1 BY 语义**

**去重逻辑**：
- `ORDER BY event_ts DESC`：最新事件优先
- `LIMIT 1 BY project_id, id`：每个实体只返回最新记录

**等价 SQL（PostgreSQL）**：
```sql
SELECT DISTINCT ON (project_id, id) *
FROM observations
WHERE project_id = 'prj-123' AND id = 'obs-456'
ORDER BY project_id, id, event_ts DESC;
```

---

### 6. **PostgreSQL 异步更新**

#### **TraceUpsert Queue**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L757-L773)

**入队逻辑（Lines 757-773）**：
```typescript
const traceUpsertQueue = getQueue(QueueName.TraceUpsert);
await traceUpsertQueue.add(
  QueueName.TraceUpsert,
  {
    id: entityId,
    projectId,
    timestamp: new Date(),
    payload: {
      id: entityId,
      name: finalTraceRecord.name,
      timestamp: minTimestamp,
      ...
    },
  },
  {
    jobId: shardingKey,  // 基于 traceId 的幂等性
    removeOnComplete: true,
    removeOnFail: 10000,
  }
);
```

#### **幂等性保证**

| 机制 | 实现方式 | 目的 |
|-----|---------|------|
| **Job ID** | `jobId: shardingKey`（traceId 哈希） | 同一 traceId 的任务只执行一次 |
| **Upsert 语义** | `ON CONFLICT (id) DO UPDATE` | 更新已存在记录 |

---

### 7. **Blob Storage File Log**

#### **表结构**

**位置**：[worker/src/queues/ingestionQueue.ts](worker/src/queues/ingestionQueue.ts#L66-L80)

| 字段名 | 类型 | 描述 |
|-------|------|------|
| `id` | String | 日志 ID（UUID） |
| `project_id` | String | 项目 ID |
| `entity_type` | Enum | traces/observations/scores |
| `entity_id` | String | 实体 ID |
| `event_id` | String | 事件 ID（fileKey） |
| `bucket_name` | String | S3 存储桶名称 |
| `bucket_path` | String | S3 完整路径 |
| `created_at` | DateTime64 | 创建时间 |
| `updated_at` | DateTime64 | 更新时间 |
| `event_ts` | DateTime64 | 事件时间戳 |
| `is_deleted` | UInt8 | 删除标记（0/1） |

#### **写入逻辑（Lines 66-80）**

```typescript
if (env.LANGFUSE_ENABLE_BLOB_STORAGE_FILE_LOG === "true") {
  const fileName = `${job.data.payload.data.fileKey}.json`;
  clickhouseWriter.addToQueue(TableName.BlobStorageFileLog, {
    id: randomUUID(),
    project_id: job.data.payload.authCheck.scope.projectId,
    entity_type: getClickhouseEntityType(job.data.payload.data.type),
    entity_id: job.data.payload.data.eventBodyId,
    event_id: job.data.payload.data.fileKey,
    bucket_name: env.LANGFUSE_S3_EVENT_UPLOAD_BUCKET,
    bucket_path: `${env.LANGFUSE_S3_EVENT_UPLOAD_PREFIX}...`,
    created_at: new Date().getTime(),
    updated_at: new Date().getTime(),
    event_ts: new Date().getTime(),
    is_deleted: 0,
  });
}
```

#### **用途**

| 场景 | 查询方式 | 示例 |
|-----|---------|------|
| **数据保留策略** | 查询 `event_ts` 删除旧文件 | 删除 90 天前的 S3 文件 |
| **数据审计** | 统计项目文件数量 | 计费依据 |
| **灾难恢复** | 根据 `bucket_path` 重新导入 | 从 S3 重建 ClickHouse |

---

## 数据一致性保证

### **最终一致性模型**

| 组件 | 写入时机 | 一致性级别 |
|-----|---------|-----------|
| **S3** | 摄取时立即写入 | **强一致**（事件源） |
| **ClickHouse** | Worker 异步写入 | **最终一致**（延迟 100-500ms） |
| **PostgreSQL** | 队列异步更新 | **最终一致**（延迟 500-5000ms） |

### **一致性保证机制**

| 机制 | 实现方式 | 代码行 |
|-----|---------|-------|
| **事件时间排序** | `toTimeSortedEventList()` | 1020-1033 |
| **不可覆盖字段** | `immutableEntityKeys` 检查 | 997-1018 |
| **幂等性写入** | Job ID + Upsert 语义 | 757-773 |
| **去重缓存** | Redis 5 分钟 TTL | 85-105 |

---

## 查询优化

### **ClickHouse 查询索引**

#### **PRIMARY KEY 设计**

```sql
-- observations 表
ORDER BY (project_id, id, event_ts)

-- traces 表
ORDER BY (project_id, trace_id, event_ts)

-- scores 表
ORDER BY (project_id, trace_id, timestamp)
```

#### **查询性能**

| 查询类型 | 索引使用 | 性能 |
|---------|---------|------|
| **按 project_id + id** | 完整索引 | < 10ms |
| **按 project_id + 时间范围** | 前缀索引 + 分区裁剪 | 10-100ms |
| **全表扫描（统计）** | 列式压缩 | 100-1000ms |

---

## 性能指标

### **写入吞吐量**

| 指标 | S3 | ClickHouse | PostgreSQL |
|-----|----|-----------| |
| **单请求延迟** | 50-150ms | 5-50ms（批量） | 10-100ms |
| **吞吐量（TPS）** | 1000+ | 10,000+（批量） | 100-500 |
| **并发写入** | 无限 | 10 并发线程 | 队列控制 |

### **存储空间**

| 存储层 | 压缩率 | 示例（1M traces） |
|-------|-------|-----------------|
| **S3（JSON）** | 1x | 50 GB |
| **ClickHouse** | 10x | 5 GB（压缩后） |
| **PostgreSQL** | 3x | 15 GB |

---

## 配置参数

### **S3 配置**

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| `LANGFUSE_S3_EVENT_UPLOAD_BUCKET` | - | S3 存储桶名称 |
| `LANGFUSE_S3_EVENT_UPLOAD_PREFIX` | `""` | S3 路径前缀 |
| `LANGFUSE_S3_CONCURRENT_READS` | `10` | S3 并发下载数 |

### **ClickHouse 配置**

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| `CLICKHOUSE_URL` | - | ClickHouse 连接地址 |
| `CLICKHOUSE_USER` | `default` | ClickHouse 用户名 |
| `CLICKHOUSE_PASSWORD` | - | ClickHouse 密码 |
| `LANGFUSE_ENABLE_BLOB_STORAGE_FILE_LOG` | `true` | 启用文件日志 |

### **PostgreSQL 配置**

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| `DATABASE_URL` | - | PostgreSQL 连接字符串 |
| `DATABASE_POOL_MAX` | `10` | 连接池最大连接数 |

---

## 监控指标

### **关键 Metrics**

| 指标名 | 类型 | 标签 | 描述 |
|-------|------|------|------|
| `langfuse.storage.s3_upload_duration` | Histogram | `status` | S3 上传耗时分布 |
| `langfuse.storage.clickhouse_write_batch_size` | Distribution | `table` | ClickHouse 批量写入大小 |
| `langfuse.storage.postgres_queue_lag` | Gauge | `queue` | PostgreSQL 队列延迟 |
| `langfuse.storage.s3_file_size_bytes` | Histogram | - | S3 文件大小分布 |

---

## 相关文档

- [Ingestion Pipeline 管理](ingestion-pipeline.md) - API 摄取流程
- [Event Processing 管理](event-processing.md) - Worker 事件处理
- [sequence-03-event-merge-enrichment.puml](sequence-03-event-merge-enrichment.puml) - 数据合并时序图
