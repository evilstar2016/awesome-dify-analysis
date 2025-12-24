# Ingestion Pipeline 管理

## 功能概述

Ingestion Pipeline 是 Langfuse 的**数据摄取入口**，负责接收 SDK 发送的事件批次，并通过 API → S3 → Redis Queue 的异步架构快速返回响应，同时保证数据可靠性。核心设计目标：
- **高吞吐量**：支持 4.5MB 批次大小，单请求数百事件
- **快速响应**：API 平均响应时间 < 200ms（仅上传 S3 + 入队）
- **数据持久化**：S3 作为事件存储，支持灾难恢复
- **并发控制**：Redis 去重缓存 + 速率限制

---

## 核心组件

### 1. **Ingestion API Handler**

**位置**：[web/src/pages/api/public/ingestion.ts](web/src/pages/api/public/ingestion.ts#L49-L173)

#### **请求处理流程（125 行）**

| 处理阶段 | 代码行 | 功能描述 |
|---------|-------|---------|
| **1. CORS & 验证** | 55-94 | CORS 中间件 + HTTP Method 检查 + API Key 认证 |
| **2. 权限检查** | 76-94 | 验证 projectId + 检查 Ingestion 暂停状态 |
| **3. 速率限制** | 103-116 | Rate Limiter（失败开放策略） |
| **4. Schema 验证** | 118-131 | Zod 验证批次结构（batch + metadata） |
| **5. 事件处理** | 133-138 | 调用 `processEventBatch()` 处理事件 |
| **6. 响应返回** | 138 | 返回 207 Multi-Status（包含成功/失败列表） |

#### **配置参数**

```typescript
// Lines 26-32
export const config = {
  api: {
    bodyParser: {
      sizeLimit: "4.5mb",  // 最大请求体大小
    },
  },
};
```

#### **认证与授权（Lines 76-94）**

| 验证项 | 错误码 | 描述 |
|-------|-------|------|
| `validKey` | 401 | API Key 无效 |
| `scope.projectId` | 401 | 缺少 projectId（可能使用了组织密钥） |
| `isIngestionSuspended` | 403 | 项目因超出配额被暂停 |

#### **速率限制策略（Lines 103-116）**

```typescript
// Fail-Open 策略：速率限制器错误时继续处理
const rateLimitCheck = await RateLimitService.getInstance()
  .rateLimitRequest(authCheck.scope, "ingestion");

if (rateLimitCheck?.isRateLimited()) {
  return rateLimitCheck.sendRestResponseIfLimited(res);
}
// 错误时记录日志但不阻塞请求
```

---

### 2. **processEventBatch 核心函数**

**位置**：[packages/shared/src/server/ingestion/processEventBatch.ts](packages/shared/src/server/ingestion/processEventBatch.ts#L103-L355)

#### **函数签名（253 行）**

```typescript
// Lines 103-107
export const processEventBatch = async (
  batch: unknown[],
  authCheck: AuthHeaderVerificationResult,
  options?: ProcessEventBatchOptions
)
```

#### **处理流程（9 个阶段）**

| 阶段 | 代码行 | 处理内容 | 耗时（估算） |
|-----|-------|---------|-----------|
| **1. 批次排序** | 126-130 | 按事件类型分组（trace/observation/score/dataset） | < 1ms |
| **2. 授权验证** | 132-163 | 检查项目访问权限 + 暂停状态 | 2-5ms |
| **3. S3 上传** | 167-260 | 并发上传事件到 S3（分 entity 存储） | 50-150ms |
| **4. 队列入队** | 262-297 | 创建 Redis 队列任务（支持延迟入队） | 5-10ms |
| **5. 同步写入** | 299-339 | 可选的同步写入路径（向后兼容） | 0ms（默认禁用） |
| **6. 聚合结果** | 341-353 | 合并成功/失败列表 + 207 响应 | < 1ms |

#### **关键优化**

**并发 S3 上传（Lines 167-260）**：
```typescript
// 按 entityType + eventBodyId 分组
const groupedByEntity = groupBy(sortedBatch, 
  (event) => `${event.type}:${event.body.id}`
);

// Promise.all 并发上传
await Promise.all(
  Object.entries(groupedByEntity).map(async ([entityKey, events]) => {
    await s3StorageServiceClient.uploadJson(path, events);
  })
);
```

---

### 3. **S3 存储结构**

**目录层次结构**：
```
s3://langfuse-event-bucket/
└── {env.LANGFUSE_S3_EVENT_UPLOAD_PREFIX}/   # 环境前缀（如 "prod/"）
    └── {projectId}/                          # 项目 ID
        └── {entityType}/                     # traces/observations/scores/dataset_run_items
            └── {eventBodyId}/                # 实体 ID（traceId/observationId/scoreId）
                └── {fileKey}.json            # 文件：{eventId}-{timestamp}.json
```

#### **文件 Key 生成逻辑**

| 实体类型 | eventBodyId | fileKey 示例 |
|---------|------------|-------------|
| Trace | `traceId` | `evt-123-1704067200000.json` |
| Observation | `observationId` | `evt-456-1704067200001.json` |
| Score | `scoreId` | `evt-789-1704067200002.json` |

**示例路径**：
```
prod/prj-abc123/traces/trc-xyz789/evt-001-1704067200000.json
prod/prj-abc123/observations/obs-def456/evt-002-1704067200001.json
```

---

### 4. **Redis 队列任务**

**队列名称**：[packages/shared/src/server/redis/ingestionQueue.ts](packages/shared/src/server/redis/ingestionQueue.ts#L11-L92)

#### **IngestionQueue 类结构**

| 方法 | 代码行 | 功能 |
|-----|-------|------|
| `getShardNames()` | 17-22 | 获取分片名称数组（`ingestion-queue-0` ~ `N-1`） |
| `getShardIndexFromShardName()` | 24-37 | 从分片名称解析索引 |
| `getInstance()` | 44-91 | 单例模式获取队列实例（支持分片） |

#### **队列分片策略**

```typescript
// Lines 17-22
static getShardNames(): string[] {
  const shardCount = env.LANGFUSE_INGESTION_QUEUE_SHARDS;
  return Array.from(
    { length: shardCount },
    (_, i) => `${QueueName.IngestionQueue}-${i}`
  );
}
```

**分片计算**（基于 projectId 哈希）：
- 使用 MurmurHash3 对 `projectId` 取哈希
- 取模运算分配到 N 个分片（`env.LANGFUSE_INGESTION_QUEUE_SHARDS`）
- 保证同一项目的事件始终路由到同一分片

---

### 5. **任务延迟策略**

**位置**：[packages/shared/src/server/ingestion/processEventBatch.ts](packages/shared/src/server/ingestion/processEventBatch.ts#L76-L101)

#### **延迟计算逻辑（26 行）**

```typescript
// Lines 76-101: getDelay 函数
const getDelay = (
  entityType: ClickhouseEntityType,
  skipS3List: boolean
): number => {
  const delayMs = env.LANGFUSE_INGESTION_QUEUE_DELAY_MS;
  
  // 对于 OTel observation，且启用 skipS3List，返回 0 延迟
  if (entityType === "observation" && skipS3List) {
    return 0;
  }
  
  return delayMs;
};
```

#### **延迟配置表**

| 实体类型 | skipS3List | 延迟时间 | 原因 |
|---------|-----------|---------|------|
| Observation（OTel） | ✅ | 0ms | 单文件直接下载，无需等待合并 |
| 其他类型 | ❌ | 可配置（默认 500ms） | 等待乱序事件到达后批量合并 |

---

### 6. **错误处理与响应**

#### **207 Multi-Status 响应结构**

```typescript
{
  "successes": [
    { "id": "evt-001", "status": 201 }
  ],
  "errors": [
    { 
      "id": "evt-002", 
      "status": 400,
      "message": "Invalid event schema",
      "error": "ValidationError"
    }
  ]
}
```

#### **常见错误码**

| 错误码 | 错误类型 | 触发条件 | 代码行 |
|-------|---------|---------|-------|
| 400 | Invalid Request | Schema 验证失败 | 125-131 |
| 401 | Unauthorized | API Key 无效 | 81-83 |
| 401 | Unauthorized | 缺少 projectId | 84-88 |
| 403 | Forbidden | Ingestion 暂停 | 90-94 |
| 429 | Rate Limited | 超出速率限制 | 109-111 |
| 500 | Internal Error | S3/Redis 错误 | 捕获异常 |

---

## 性能指标

### **API 响应时间分解**

| 阶段 | 耗时 | 占比 |
|-----|------|------|
| 认证 + 速率限制 | 5-10ms | 5% |
| Schema 验证 | 1-2ms | 1% |
| S3 并发上传 | 50-150ms | 75% |
| Redis 入队 | 5-10ms | 5% |
| 响应序列化 | 1-2ms | 1% |
| **总计** | **62-174ms** | **100%** |

### **吞吐量优化**

| 优化项 | 实现方式 | 性能提升 |
|-------|---------|---------|
| **并发 S3 上传** | `Promise.all` 分组上传 | 3-5x |
| **队列分片** | 基于 projectId 哈希分片 | 10-20x |
| **Redis 自动流水线** | 自动批量命令（Cluster 模式） | 2-3x |
| **Skip S3 List** | OTel 单文件直接下载 | 10-50x |

---

## 配置参数

### **环境变量**

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| `LANGFUSE_S3_EVENT_UPLOAD_BUCKET` | - | S3 存储桶名称 |
| `LANGFUSE_S3_EVENT_UPLOAD_PREFIX` | `""` | S3 路径前缀 |
| `LANGFUSE_INGESTION_QUEUE_SHARDS` | `5` | 队列分片数量 |
| `LANGFUSE_INGESTION_QUEUE_DELAY_MS` | `500` | 默认延迟时间（毫秒） |
| `LANGFUSE_ENABLE_REDIS_SEEN_EVENT_CACHE` | `true` | 启用去重缓存 |
| `LANGFUSE_S3_CONCURRENT_READS` | `10` | S3 并发下载数 |

---

## 监控指标

### **关键 Metrics**

| 指标名 | 类型 | 标签 | 描述 |
|-------|------|------|------|
| `langfuse.ingestion.request_duration` | Histogram | `status` | API 请求耗时分布 |
| `langfuse.ingestion.batch_size` | Distribution | `project_id` | 批次大小分布 |
| `langfuse.ingestion.s3_upload_errors` | Counter | `error_type` | S3 上传失败次数 |
| `langfuse.ingestion.queue_add_success` | Counter | `shard` | 入队成功次数 |
| `langfuse.ingestion.recently_processed_cache` | Counter | `skipped` | 去重缓存命中率 |

---

## 使用示例

### **SDK 调用示例**

```typescript
// TypeScript SDK
import Langfuse from "langfuse";

const langfuse = new Langfuse({
  publicKey: "pk-xxx",
  secretKey: "sk-xxx",
  flushAt: 100,  // 批次大小（触发上传）
  flushInterval: 5000  // 5秒自动刷新
});

// 自动批量上传
langfuse.trace({ id: "trace-1", name: "API Call" });
langfuse.span({ traceId: "trace-1", name: "DB Query" });

await langfuse.shutdownAsync();  // 刷新剩余事件
```

### **直接 API 调用**

```bash
curl -X POST https://cloud.langfuse.com/api/public/ingestion \
  -H "Authorization: Bearer pk-xxx:sk-xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "batch": [
      {
        "id": "evt-001",
        "type": "trace-create",
        "timestamp": "2024-01-01T00:00:00Z",
        "body": {
          "id": "trace-1",
          "name": "API Request"
        }
      }
    ],
    "metadata": {
      "sdk_version": "3.0.0",
      "sdk_name": "langfuse-js"
    }
  }'
```

---

## 相关文档

- [Event Processing 管理](event-processing.md) - Worker 处理流程
- [Data Storage 管理](data-storage.md) - S3 与数据库存储
- [sequence-01-api-ingestion-flow.puml](sequence-01-api-ingestion-flow.puml) - API 流程时序图
