# Traces & Observations 模块

## 模块概述

Traces & Observations 模块是 Langfuse 的**核心基础模块**，提供 LLM 应用的追踪和观测能力。该模块负责记录、存储、查询和分析 LLM 应用的执行过程，是整个可观测性平台的基石。

### 核心价值
- 📊 **完整追踪**: 记录 LLM 应用的完整执行链路
- 🔍 **深度观测**: 捕获每个 LLM 调用的详细信息
- 💰 **成本分析**: 自动计算 Token 使用和成本
- ⚡ **性能监控**: 追踪延迟和执行时间
- 🏷️ **灵活标记**: 支持 tags、环境、版本等元数据

---

## 核心概念

### Trace (追踪)
**定义**: 一个完整的 LLM 应用执行单元，通常对应一次用户请求。

**属性**:
- `id`: 唯一标识符
- `name`: 追踪名称
- `userId`: 关联的用户 ID
- `sessionId`: 关联的会话 ID
- `metadata`: 自定义元数据
- `tags`: 标签列表
- `environment`: 环境标识（dev/staging/prod）
- `release` / `version`: 发布版本
- `input` / `output`: 输入输出数据
- `timestamp`: 创建时间

**数据模型**:
```typescript
export const TraceDomain = z.object({
  id: z.string(),
  name: z.string().nullable(),
  timestamp: z.date(),
  environment: z.string(),
  tags: z.array(z.string()),
  bookmarked: z.boolean(),
  public: z.boolean(),
  release: z.string().nullable(),
  version: z.string().nullable(),
  input: jsonSchema.nullable(),
  output: jsonSchema.nullable(),
  metadata: MetadataDomain,
  createdAt: z.date(),
  updatedAt: z.date(),
  sessionId: z.string().nullable(),
  userId: z.string().nullable(),
  projectId: z.string(),
});
```

### Observation (观测)
**定义**: Trace 内的一个执行单元，表示一次 LLM 调用、函数执行或事件。

**类型**:
1. **GENERATION**: LLM 生成调用（如 OpenAI Chat Completion）
2. **SPAN**: 通用执行跨度（如函数执行、工具调用）
3. **EVENT**: 事件记录（如日志、错误）

**属性**:
- `id`: 唯一标识符
- `traceId`: 所属 Trace ID
- `parentObservationId`: 父 Observation ID（支持嵌套）
- `type`: 类型（GENERATION/SPAN/EVENT）
- `name`: 观测名称
- `model`: 使用的模型（如 gpt-4）
- `input` / `output`: 输入输出
- `metadata`: 元数据
- `startTime` / `endTime`: 开始/结束时间
- `completionStartTime`: 生成开始时间
- `promptTokens` / `completionTokens`: Token 数量
- `totalTokens`: 总 Token 数
- `calculatedInputCost` / `calculatedOutputCost`: 成本
- `level`: 日志级别
- `statusMessage`: 状态消息

### 关系模型
```
Trace (追踪)
  ├─ Observation (GENERATION) - LLM 调用
  │   └─ Observation (SPAN) - 工具调用
  │       └─ Observation (EVENT) - 日志
  ├─ Observation (SPAN) - 另一个函数
  └─ Score[] - 评分列表
```

---

## 主要功能

### 1. Trace 管理
- **创建 Trace**: 通过 SDK 或 API 创建新追踪
- **更新 Trace**: 更新 Trace 的元数据、输出等
- **查询 Trace**: 按 ID、用户、会话、标签等查询
- **搜索 Trace**: 全文搜索 Trace 名称、ID
- **过滤 Trace**: 按时间、环境、版本、标签等过滤
- **删除 Trace**: 删除指定 Trace

### 2. Observation 管理
- **创建 Observation**: 记录 LLM 调用、函数执行、事件
- **更新 Observation**: 更新输出、Token 数、成本等
- **查询 Observation**: 获取 Trace 的所有 Observations
- **嵌套 Observation**: 支持父子关系的嵌套结构

### 3. 成本计算
- **自动计算**: 基于 Token 数和模型价格自动计算成本
- **聚合统计**: 按 Trace、项目、时间段聚合成本
- **多模型支持**: 支持 OpenAI、Anthropic、Azure 等多种模型

### 4. 数据聚合和分析
- **统计指标**: Trace 数量、Observation 数量、Token 使用量、总成本
- **分组聚合**: 按名称、标签、用户、会话分组统计
- **时间序列**: 按时间维度分析趋势
- **评分聚合**: 关联 Scores 进行质量分析

### 5. UI 表格查询
- **分页查询**: 支持大数据量的分页展示
- **多维排序**: 按时间、成本、Token 数等排序
- **高级过滤**: 支持复杂的过滤条件组合
- **评论过滤**: 支持按评论内容过滤

---

## 技术架构

### 数据存储

#### PostgreSQL (元数据)
**表**: `traces` (LegacyPrismaTrace)  
**用途**: 旧版数据存储，逐步迁移到 ClickHouse  
**Schema**: 见 `packages/shared/prisma/schema.prisma`

**注意**: 主要的 traces 和 observations 数据现在存储在 ClickHouse 中，PostgreSQL 主要用于项目元数据和配置。

```sql
-- ClickHouse traces 表结构
CREATE TABLE traces (
    `id` String,
    `timestamp` DateTime64(3),
    `name` String,
    `user_id` Nullable(String),
    `metadata` Map(LowCardinality(String), String),
    `release` Nullable(String),
    `version` Nullable(String),
    `project_id` String,
    `public` Bool,
    `bookmarked` Bool,
    `tags` Array(String),
    `input` Nullable(String) CODEC(ZSTD(3)),
    `output` Nullable(String) CODEC(ZSTD(3)),
    `session_id` Nullable(String),
    `created_at` DateTime64(3),
    `updated_at` DateTime64(3),
    `event_ts` DateTime64(3),
    `is_deleted` UInt8,
    ...
) ENGINE = ReplacingMergeTree(event_ts, is_deleted)
```

#### ClickHouse (分析数据)
**表**: `traces`, `observations`  
**用途**: 存储用于分析和聚合的数据副本  
**特点**: 列式存储，查询性能高

**同步机制**:
- PostgreSQL 写入后，通过 Worker 异步同步到 ClickHouse
- 支持批量写入，提高性能
- 最终一致性

### API 架构

#### tRPC Routers
**路径**: `web/src/server/api/routers/traces.ts`

**主要 Procedures**:

| Procedure | 输入 | 输出 | 说明 |
|----------|------|------|------|
| `hasTracingConfigured` | projectId | boolean | 检查是否配置了追踪 |
| `all` | TraceFilterOptions | traces[] | 查询 Traces 列表 |
| `countAll` | TraceFilterOptions | totalCount | 统计 Traces 数量 |
| `metrics` | traceIds, filter | metrics[] | 获取 Traces 指标 |
| `byId` | traceId | trace | 获取单个 Trace |
| `byIdWithObservationsAndScores` | traceId | trace + observations + scores | 获取完整 Trace 数据 |
| `filterOptions` | projectId | filterOptions | 获取过滤选项 |
| `deleteMany` | traceIds[] | void | 批量删除 Traces |
| `bookmark` | traceId, bookmarked | void | 标记/取消标记 |
| `publish` | traceId, public | void | 发布/取消发布 |
| `updateTags` | traceId, tags | void | 更新标签 |

#### Public API (REST)
**路径**: `web/src/features/public-api/server/traces.ts`

**端点**:
- `GET /api/public/traces` - 查询 Traces
- `GET /api/public/traces/:traceId` - 获取单个 Trace
- `POST /api/public/traces` - 创建 Trace (通常通过 SDK)
- `PATCH /api/public/traces/:traceId` - 更新 Trace

### 服务层
**路径**: `packages/shared/src/server/services/`

**主要服务**:
- `getTracesTable`: 查询 Traces 表格数据
- `getTracesTableCount`: 统计 Traces 数量
- `getTracesTableMetrics`: 获取 Traces 指标
- `getTraceById`: 查询单个 Trace
- `upsertTrace`: 创建或更新 Trace
- `getObservationsForTrace`: 获取 Trace 的所有 Observations
- `getScoresForTraces`: 获取 Traces 的评分
- `getTracesGroupedByName`: 按名称分组
- `getTracesGroupedByTags`: 按标签分组
- `getTracesGroupedByUsers`: 按用户分组
- `traceDeletionProcessor`: 删除 Trace 处理器

### 领域模型
**路径**: `packages/shared/src/domain/traces.ts`

**定义**:
- `TraceDomain`: Trace 领域模型
- `MetadataDomain`: 元数据模型
- Zod schemas 用于运行时验证

---

## 目录结构

```
# 前端
web/src/pages/project/[projectId]/
├── traces.tsx                      # Traces 列表页
└── traces/
    └── [traceId].tsx               # Trace 详情页

web/src/components/
├── table/                          # Traces 表格组件
└── trace/                          # Trace 详情组件

# 后端 API
web/src/server/api/routers/
└── traces.ts                       # tRPC Router

web/src/features/public-api/
├── server/traces.ts                # REST API 实现
└── types/traces.ts                 # API 类型定义

# 服务层
packages/shared/src/server/
├── services/
│   └── traces-ui-table-service.ts  # UI 表格服务
├── repositories/
│   ├── traces.ts                   # Traces 仓储
│   └── traces_converters.ts        # 数据转换器
└── queues/
    └── traceDeletionQueue.ts       # 删除队列

# 领域模型
packages/shared/src/domain/
└── traces.ts                       # Trace 领域模型

# 数据表定义
packages/shared/src/tableDefinitions/
└── tracesTable.ts                  # ClickHouse 表定义

# 测试
web/src/__tests__/async/
├── traces-api.servertest.ts        # API 测试
├── traces-trpc.servertest.ts       # tRPC 测试
└── traces-ui-table.servertest.ts   # UI 表格测试
```

---

## 核心流程

### 流程索引
1. [Trace 创建流程](./01-trace-creation-sequence.puml) - SDK 到数据库的完整链路
2. [Trace 查询流程](./02-trace-query-sequence.puml) - 用户查询 Traces 列表
3. [Trace 详情加载流程](./03-trace-detail-sequence.puml) - 加载单个 Trace 的完整信息
4. [成本计算流程](./04-cost-calculation-sequence.puml) - Token 计数和成本计算
5. [ClickHouse 写入流程](./05-clickhouse-sync-sequence.puml) - 数据异步写入 ClickHouse

---

## 数据流

### 写入流程
```
SDK Client
    ↓ HTTP POST
Ingestion API
    ↓ Validate
Business Logic
    ↓ Transform
Prisma (PostgreSQL)
    ↓ Write Success
BullMQ Queue
    ↓ Async Job
Worker Service
    ↓ Sync
ClickHouse
```

### 查询流程
```
Frontend
    ↓ tRPC Client
tRPC Router (traces.all)
    ↓ Auth Middleware
Traces Service
    ↓ Query ClickHouse
ClickHouse
    ↓ Results
Traces Service (Transform)
    ↓ Response
Frontend (Display)
```

---

## 性能优化

### 1. 数据库索引
```sql
-- PostgreSQL
CREATE INDEX idx_traces_project_timestamp ON traces(project_id, timestamp DESC);
CREATE INDEX idx_traces_user_id ON traces(user_id);
CREATE INDEX idx_traces_session_id ON traces(session_id);
CREATE INDEX idx_observations_trace_id ON observations(trace_id);
```

### 2. 分页策略
- 使用游标分页（cursor-based）
- 默认页大小：50
- 最大页大小：1000

### 3. 查询优化
- ClickHouse 用于聚合查询
- PostgreSQL 用于精确查询
- Redis 缓存热点数据

### 4. 异步处理
- ClickHouse 同步异步化
- 成本计算异步化
- 批量操作队列化

---

## 关键技术点

### 1. 多数据库协调
- **PostgreSQL**: 作为主数据库，保证数据一致性
- **ClickHouse**: 作为分析数据库，提供高性能聚合查询
- **同步机制**: Worker 通过 BullMQ 异步同步

### 2. 成本计算
```typescript
// 成本计算公式
inputCost = (promptTokens / 1_000_000) * model.inputPrice
outputCost = (completionTokens / 1_000_000) * model.outputPrice
totalCost = inputCost + outputCost
```

### 3. 嵌套结构
- Observations 支持父子关系
- 通过 `parentObservationId` 建立层次结构
- 前端递归渲染树形结构

### 4. 实时更新
- SDK 支持流式更新（streaming）
- 前端使用 React Query 轮询或 WebSocket
- 数据库使用 Upsert 避免冲突

---

## 依赖关系

### 依赖的模块
- **Scores 模块**: 关联评分数据
- **Sessions 模块**: 关联会话
- **Models 模块**: 获取模型配置（价格、Tokenizer）
- **Projects 模块**: 项目隔离
- **Auth 模块**: 认证和权限

### 被依赖的模块
- **Dashboard 模块**: 使用 Traces 数据生成图表
- **Datasets 模块**: 从 Traces 创建数据集
- **Evals 模块**: 评估 Traces 的质量
- **Playground 模块**: 查看历史 Traces
- **Exports 模块**: 导出 Traces 数据

---

## 配置参数

### 环境变量
```env
# PostgreSQL
DATABASE_URL=postgresql://...

# ClickHouse
CLICKHOUSE_URL=https://...
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=...

# Redis (BullMQ)
REDIS_CONNECTION_STRING=redis://...

# 数据保留
LANGFUSE_RETENTION_DAYS=90
```

### 项目级配置
- `retentionDays`: 数据保留天数
- `cloudConfig`: 云配置（rate limits, quotas）

---

## 最佳实践

### 1. SDK 使用
```typescript
import { Langfuse } from "langfuse";

const langfuse = new Langfuse({
  publicKey: "pk-...",
  secretKey: "sk-...",
});

// 创建 Trace
const trace = langfuse.trace({
  name: "chat-completion",
  userId: "user-123",
  metadata: { language: "en" },
  tags: ["production"],
});

// 创建 Generation
const generation = trace.generation({
  name: "gpt-4-call",
  model: "gpt-4",
  input: messages,
});

// 更新输出
generation.end({
  output: response,
});
```

### 2. 命名规范
- Trace name: 描述性名称，如 "chat-completion", "rag-query"
- Observation name: 具体操作，如 "gpt-4-call", "vector-search"

### 3. 元数据使用
- 使用 `metadata` 存储结构化数据
- 使用 `tags` 存储可过滤的标签
- 使用 `environment` 区分环境

### 4. 成本控制
- 配置模型价格
- 监控 Token 使用
- 设置预算告警

---

## 常见问题

### Q: Trace 和 Observation 的区别？
A: Trace 是一个完整的执行单元（如一次用户请求），Observation 是 Trace 内的子单元（如一次 LLM 调用）。一个 Trace 可以包含多个 Observations。

### Q: 如何处理大量 Traces？
A: Langfuse 使用 ClickHouse 进行高性能分析，支持数百万级 Traces。同时提供数据保留策略，自动清理过期数据。

### Q: 成本计算准确吗？
A: 成本基于 Token 数和模型价格计算，Token 数使用 tiktoken 库计算，准确度高。价格需要在 Models 配置中设置。

### Q: 支持哪些模型？
A: 支持 OpenAI、Anthropic、Azure OpenAI、Google等主流模型。可以在 Models 配置中自定义模型价格。

---

## 技术栈

- **前端**: Next.js 15 + React 19 + TanStack Query
- **API**: tRPC 11
- **ORM**: Prisma 6
- **数据库**: PostgreSQL 15 + ClickHouse 24
- **队列**: BullMQ
- **验证**: Zod 3

---

## 参考资源

- [Prisma Schema](../../../packages/shared/prisma/schema.prisma)
- [tRPC Router](../../../web/src/server/api/routers/traces.ts)
- [Domain Model](../../../packages/shared/src/domain/traces.ts)
- [API Types](../../../web/src/features/public-api/types/traces.ts)
- [UI Table Service](../../../packages/shared/src/server/services/traces-ui-table-service.ts)

---

**文档编写时间**: 2025-12-17  
**项目版本**: 3.140.0
