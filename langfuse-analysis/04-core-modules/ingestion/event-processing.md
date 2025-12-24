# Event Processing 管理

## 功能概述

Event Processing 是 Langfuse Ingestion 模块的**核心处理引擎**，负责从 Redis 队列消费任务，从 S3 下载事件，执行事件合并与增强，最终写入 ClickHouse 和 PostgreSQL。核心设计：
- **事件合并**：支持乱序到达的 create/update 事件
- **增量更新**：非覆盖字段保留原值
- **成本计算**：自动计算 LLM 使用成本
- **双库写入**：ClickHouse（分析）+ PostgreSQL（元数据）

---

## 核心组件

### 1. **Worker Queue Consumer**

**位置**：[worker/src/queues/ingestionQueue.ts](worker/src/queues/ingestionQueue.ts#L28-L303)

#### **ingestionQueueProcessorBuilder 函数（276 行）**

| 处理阶段 | 代码行 | 功能描述 | 耗时（估算） |
|---------|-------|---------|-----------|
| **1. 任务初始化** | 36-57 | 提取任务参数 + OpenTelemetry 追踪 | < 1ms |
| **2. 文件日志记录** | 59-83 | 写入 ClickHouse 文件日志表 | 2-5ms |
| **3. 去重检查** | 85-105 | Redis 缓存检查（5 分钟 TTL） | 1-2ms |
| **4. 二级队列路由** | 107-131 | 检查是否需要路由到二级队列 | 5-10ms |
| **5. S3 文件下载** | 133-221 | 批量下载事件文件 | 100-5000ms |
| **6. 缓存标记** | 223-242 | 标记已处理事件（去重） | 5-10ms |
| **7. 事件合并** | 244-267 | 调用 IngestionService 合并写入 | 50-500ms |
| **8. 错误处理** | 269-303 | 捕获 S3 SlowDown 错误 + 异常追踪 | - |

#### **关键优化策略**

**Skip S3 List 优化（Lines 153-174）**：
```typescript
// 对于 OTel observation，直接下载单文件，跳过 S3 list 操作
const shouldSkipS3List = 
  job.data.payload.data.skipS3List && 
  job.data.payload.data.fileKey;

if (shouldSkipS3List) {
  // 直接下载：{prefix}/{fileKey}.json
  const filePath = `${s3Prefix}${job.data.payload.data.fileKey}.json`;
  const file = await s3Client.download(filePath);
  events.push(...parsedFile);
} else {
  // 列出所有文件后批量下载
  eventFiles = await s3Client.listFiles(s3Prefix);
}
```

**并发下载优化（Lines 176-210）**：
```typescript
// 按 S3_CONCURRENT_READS 分批并发下载
const S3_CONCURRENT_READS = env.LANGFUSE_S3_CONCURRENT_READS;
const batches = chunk(eventFiles, S3_CONCURRENT_READS);

for (const batch of batches) {
  const batchEvents = await Promise.all(
    batch.map(downloadAndParseFile)
  );
  events.push(...batchEvents.flat());
}
```

#### **二级队列路由（Lines 107-131）**

| 条件 | 触发方式 | 目的 |
|-----|---------|------|
| **环境配置** | `LANGFUSE_SECONDARY_INGESTION_QUEUE_ENABLED_PROJECT_IDS` | 手动指定项目 |
| **S3 SlowDown** | Redis Flag `s3-slowdown:{projectId}` | 自动检测并隔离 |

**路由逻辑**：
```typescript
if (enableRedirectToSecondaryQueue && (shouldRedirectEnv || shouldRedirectSlowdown)) {
  const secondaryQueue = getQueue(QueueName.IngestionSecondaryQueue);
  await secondaryQueue.add(QueueName.IngestionSecondaryQueue, job.data);
  return;  // 提前退出，避免重复处理
}
```

---

### 2. **IngestionService 核心服务**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L212-L1737)

#### **类结构（1526 行）**

| 属性/方法 | 代码行 | 功能 |
|---------|-------|------|
| **构造函数** | 215-222 | 注入 Redis/Prisma/ClickHouse |
| **mergeAndWrite** | 224-270 | 主入口：分发到不同实体类型处理 |
| **processTraceEventList** | 634-774 (141 行) | 处理 Trace 事件 |
| **processObservationEventList** | 776-910 (135 行) | 处理 Observation 事件 |
| **processScoreEventList** | 532-632 (101 行) | 处理 Score 事件 |
| **processDatasetRunItemEventList** | 441-530 (90 行) | 处理 Dataset Run Item 事件 |
| **mergeTraceRecords** | 936-958 (23 行) | 合并 Trace 记录 |
| **mergeObservationRecords** | 960-995 (36 行) | 合并 Observation 记录 |
| **mergeScoreRecords** | 912-934 (23 行) | 合并 Score 记录 |
| **getGenerationUsage** | 1070-1156 (87 行) | 计算 LLM 使用量 |
| **calculateUsageCosts** | 1288-1360 (73 行) | 计算使用成本 |

#### **mergeAndWrite 主流程（47 行）**

```typescript
// Lines 224-270
async mergeAndWrite(
  entityType: ClickhouseEntityType,
  projectId: string,
  entityId: string,
  createdAtTimestamp: Date,
  events: IngestionEventType[],
  writeToStagingTables: boolean
) {
  // 按实体类型分发
  switch (entityType) {
    case "trace":
      await this.processTraceEventList(...);
      break;
    case "observation":
      await this.processObservationEventList(...);
      break;
    case "score":
      await this.processScoreEventList(...);
      break;
    case "dataset_run_item":
      await this.processDatasetRunItemEventList(...);
      break;
  }
}
```

---

### 3. **Trace 事件处理**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L634-L774)

#### **处理流程（141 行）**

| 步骤 | 代码行 | 功能 | 数据操作 |
|-----|-------|------|---------|
| **1. 时间排序** | 650-651 | 按 timestamp 升序排序 | - |
| **2. 映射到记录** | 653-657 | 转换为 ClickHouse 记录格式 | 调用 `mapTraceEventsToRecords()` |
| **3. 计算 IO** | 662-669 | 从最后一个非空记录提取 input/output | - |
| **4. 获取 ClickHouse 记录** | 680-690 | 查询现有记录 | ClickHouse SELECT |
| **5. 合并记录** | 699-702 | 执行增量合并 | 调用 `mergeTraceRecords()` |
| **6. Session 关联** | 712-715 | 根据 sessionId 关联 | Prisma 查询 |
| **7. 包装观察（可选）** | 736-739 | 创建 staging observation | 向后兼容 |
| **8. 写入数据库** | 757-773 | ClickHouse + Trace Upsert Queue | 异步写入 |

#### **关键代码片段**

**IO 计算（Lines 662-669）**：
```typescript
// 从时间排序后的记录中找最后一个非空 input/output
const reversedRawRecords = [...traceRecords].reverse();
const finalIO = {
  input: reversedRawRecords.find((r) => r.input)?.input ?? null,
  output: reversedRawRecords.find((r) => r.output)?.output ?? null,
};
```

**Trace Upsert Queue（Lines 757-773）**：
```typescript
// 发送到 TraceUpsert 队列（PostgreSQL 异步更新）
const traceUpsertQueue = getQueue(QueueName.TraceUpsert);
await traceUpsertQueue.add(
  QueueName.TraceUpsert,
  {
    id: entityId,
    projectId,
    timestamp: new Date(),
    payload: { /* trace 数据 */ },
  },
  { jobId: shardingKey }  // 基于 traceId 的幂等性
);
```

---

### 4. **Observation 事件处理**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L776-L910)

#### **处理流程（135 行）**

| 步骤 | 代码行 | 功能 | 复杂度 |
|-----|-------|------|-------|
| **1. 时间排序** | 792-793 | 按 timestamp 排序 | O(n log n) |
| **2. 计算最早开始时间** | 796-804 | 找最早的 startTime | O(n) |
| **3. 判断观察类型** | 795 | SPAN/GENERATION/EVENT | - |
| **4. 获取 ClickHouse 记录** | 806 | 查询现有记录 + 关联 Prompt | ClickHouse SELECT |
| **5. 映射事件到记录** | 829-834 | 转换为 ClickHouse 格式 | - |
| **6. 合并记录** | 836-840 | 增量合并 | 调用 `mergeObservationRecords()` |
| **7. 提取最终 IO** | 847-855 | 从反向列表提取 input/output/metadata | - |
| **8. 成本计算** | 857-864 | 计算 LLM 使用成本（仅 GENERATION） | 调用 `getGenerationUsage()` |
| **9. 包装 Trace** | 868-881 | 为 legacy observation 创建 trace | 向后兼容 |
| **10. 写入数据库** | 900-909 | ClickHouse + Staging Table | 异步写入 |

#### **Prompt 关联逻辑（Lines 806）**

```typescript
const { clickhouseObservationRecord, prompt } = await this.getClickhouseRecord(
  clickhouseEntityType,
  projectId,
  entityId,
  undefined
);
```

**查询逻辑**：
- 查找 Observation 记录中的 `promptName` + `promptVersion`
- 从 Prompt 表获取完整 Prompt 数据
- 用于后续的 Prompt Token 计算

---

### 5. **事件合并策略**

#### **mergeRecords 通用逻辑**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L997-L1018)

```typescript
// Lines 997-1018: mergeRecords 方法（22 行）
private mergeRecords<T extends Record<string, any>>(
  records: T[]
): T {
  let result: Partial<T> = {};
  
  for (const record of records) {
    for (const [key, value] of Object.entries(record)) {
      // 跳过 null/undefined
      if (value === null || value === undefined) continue;
      
      // 跳过不可覆盖字段
      if (immutableEntityKeys.includes(key)) {
        result[key] = result[key] ?? value;
      } else {
        result[key] = value;  // 后续值覆盖
      }
    }
  }
  
  return result as T;
}
```

#### **不可覆盖字段（immutableEntityKeys）**

| 字段 | 原因 |
|-----|------|
| `id` | 实体主键，不可变 |
| `project_id` | 项目隔离，不可变 |
| `created_at` | 创建时间，不可变 |
| `timestamp` | 首次记录时间，不可变 |

#### **合并示例**

**输入事件**：
```typescript
Event 1 (t=0): { id: "obs-1", input: "Hello", output: null }
Event 2 (t=1): { id: "obs-1", output: "World", metadata: { model: "gpt-4" } }
```

**合并结果**：
```typescript
{
  id: "obs-1",           // 保留首次值（不可覆盖）
  input: "Hello",        // Event 1 提供
  output: "World",       // Event 2 覆盖
  metadata: { model: "gpt-4" }  // Event 2 提供
}
```

---

### 6. **成本计算引擎**

#### **getGenerationUsage 方法（87 行）**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L1070-L1156)

| 计算步骤 | 代码行 | 逻辑 |
|---------|-------|------|
| **1. 提取 usage_details** | 1104-1107 | 从事件中提取已提供的 usage |
| **2. 查找定价模型** | 1096 | 从 `internalModel` 获取价格表 |
| **3. 匹配定价层级** | 1115-1118 | 根据 projectId 匹配 PricingTier |
| **4. 计算 cost_details** | 1134-1138 | 应用价格表计算成本 |
| **5. 返回结果** | 1150-1154 | 返回 usage + cost + tier 信息 |

#### **calculateUsageCosts 方法（73 行）**

**位置**：[worker/src/services/IngestionService/index.ts](worker/src/services/IngestionService/index.ts#L1288-L1360)

**计算公式**：
```typescript
// Lines 1331-1335
for (const [key, units] of Object.entries(finalCostEntries)) {
  const price = modelPrices[key];
  if (price) {
    finalTotalCost += units * price;
  }
}
```

#### **成本详情字段**

| 字段 | 类型 | 描述 | 示例 |
|-----|------|------|------|
| `input` | number | 输入 token 成本 | 0.003 |
| `output` | number | 输出 token 成本 | 0.006 |
| `total` | number | 总成本（USD） | 0.009 |

---

### 7. **数据库写入策略**

#### **双库写入架构**

| 数据库 | 用途 | 写入方式 | 延迟 |
|-------|------|---------|------|
| **ClickHouse** | OLAP 分析查询 | 直接写入（ClickhouseWriter） | < 100ms |
| **PostgreSQL** | 事务元数据 | 异步队列（TraceUpsert/ObservationUpsert） | 100-1000ms |

#### **ClickHouse 写入（Lines 900-904）**

```typescript
// Lines 900-904: Staging Record 写入
const stagingRecord = {
  id: entityId,
  project_id: projectId,
  type: observationType,
  ...finalObservationRecord,
};

this.clickHouseWriter.addToQueue(
  TableName.StagingObservations,
  stagingRecord
);
```

#### **PostgreSQL 异步更新（Lines 757-773）**

```typescript
// Trace Upsert Queue
await traceUpsertQueue.add(
  QueueName.TraceUpsert,
  { id: entityId, projectId, payload: { ... } },
  { jobId: shardingKey }  // 幂等性保证
);
```

---

### 8. **去重与缓存机制**

#### **Recently Processed Cache（Lines 85-105）**

**Redis Key 格式**：
```
langfuse:ingestion:recently-processed:{projectId}:{eventType}:{eventBodyId}:{fileKey}
```

**TTL**：5 分钟

**工作流程**：
```typescript
// 1. 检查缓存
const exists = await redis.exists(key);
if (exists) {
  logger.debug("Skipping already processed event");
  return;  // 跳过处理
}

// 2. 处理事件
await new IngestionService(...).mergeAndWrite(...);

// 3. 写入缓存
await redis.set(key, "1", "EX", 60 * 5);
```

#### **缓存命中场景**

| 场景 | 描述 | 发生频率 |
|-----|------|---------|
| **快速重试** | SDK 在 5 分钟内重传同一事件 | 低（< 1%） |
| **队列重试** | Redis 队列任务失败后自动重试 | 中（1-5%） |
| **并发处理** | 队列分片中的任务竞争 | 极低（< 0.1%） |

---

## 性能指标

### **事件处理耗时分布**

| 事件类型 | 平均耗时 | P95 耗时 | P99 耗时 |
|---------|---------|---------|---------|
| Trace（单文件） | 50ms | 120ms | 200ms |
| Trace（多文件） | 500ms | 2s | 5s |
| Observation（OTel） | 80ms | 150ms | 300ms |
| Observation（传统） | 300ms | 1s | 3s |
| Score | 30ms | 80ms | 150ms |

### **S3 下载优化对比**

| 场景 | 文件数 | 原始耗时 | 优化后耗时 | 提升 |
|-----|-------|---------|-----------|------|
| **单文件（Skip List）** | 1 | 100ms（List + Download） | 20ms（Direct Download） | **5x** |
| **并发下载（10 并发）** | 100 | 10s（串行） | 1s（并发） | **10x** |
| **批量处理（分批）** | 1000 | 100s（串行） | 15s（10 并发 × 100 批） | **6.7x** |

---

## 监控指标

### **关键 Metrics**

| 指标名 | 类型 | 标签 | 描述 |
|-------|------|------|------|
| `langfuse.ingestion.count_files_distribution` | Distribution | `kind` | 每个任务处理的文件数量分布 |
| `langfuse.ingestion.s3_file_size_bytes` | Histogram | `skippedS3List` | S3 文件大小分布 |
| `langfuse.ingestion.event.count_files` | Span Attribute | - | 单个任务处理的文件数 |
| `langfuse.ingestion.s3_all_files_size_bytes` | Span Attribute | - | 单个任务下载的总字节数 |
| `langfuse.ingestion.recently_processed_cache` | Counter | `skipped` | 去重缓存命中统计 |

---

## 配置参数

### **环境变量**

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| `LANGFUSE_S3_CONCURRENT_READS` | `10` | S3 并发下载数 |
| `LANGFUSE_ENABLE_REDIS_SEEN_EVENT_CACHE` | `true` | 启用去重缓存 |
| `LANGFUSE_ENABLE_BLOB_STORAGE_FILE_LOG` | `true` | 写入文件日志表 |
| `LANGFUSE_SECONDARY_INGESTION_QUEUE_ENABLED_PROJECT_IDS` | `""` | 二级队列项目 ID（逗号分隔） |
| `LANGFUSE_EXPERIMENT_INSERT_INTO_EVENTS_TABLE` | `false` | 实验性：写入事件表 |

---

## 相关文档

- [Ingestion Pipeline 管理](ingestion-pipeline.md) - API 摄取流程
- [Data Storage 管理](data-storage.md) - S3 与数据库存储
- [sequence-02-worker-processing-flow.puml](sequence-02-worker-processing-flow.puml) - Worker 处理时序图
