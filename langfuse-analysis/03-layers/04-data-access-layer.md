# 数据访问层 (Data Access Layer)

## 1. 层职责

数据访问层是 Langfuse 的数据持久化层，负责：
- 管理数据库连接和客户端
- 执行数据库查询和操作（CRUD）
- 数据模型定义（Schema）
- 数据库迁移管理
- 查询优化和性能监控
- 多数据库协调（PostgreSQL + ClickHouse）

**架构定位**：最底层的数据层，为上层提供数据访问接口，封装所有数据库交互。

---

## 2. 主要组件

### 2.1 数据库系统

Langfuse 使用**多数据库架构**，针对不同场景选择合适的数据库：

| 数据库 | 类型 | 用途 | ORM/Client |
|-------|------|------|-----------|
| **PostgreSQL** | OLTP（关系型） | 元数据、配置、用户数据 | Prisma ORM |
| **ClickHouse** | OLAP（列式） | 观测数据、时序数据、分析 | Kysely Query Builder |
| **Redis** | 缓存/队列 | Session、缓存、任务队列 | ioredis |
| **S3/Blob** | 对象存储 | 原始事件、附件 | AWS SDK |

### 2.2 PostgreSQL 数据访问（Prisma）

#### Prisma Schema
**路径**：`packages/shared/prisma/schema.prisma`

**主要数据模型**：

| 模型 | 说明 | 关键字段 |
|-----|------|---------|
| **User** | 用户账户 | id, email, name, image |
| **Organization** | 组织 | id, name, cloudConfig |
| **Project** | 项目 | id, name, organizationId |
| **ProjectMembership** | 项目成员关系 | userId, projectId, role |
| **ApiKey** | API 密钥 | id, hashedKey, projectId |
| **Trace** | 追踪记录 | id, name, projectId, userId, metadata |
| **Observation** | 观测记录 | id, traceId, type, model, metadata |
| **Score** | 评分 | id, traceId, name, value, comment |
| **Prompt** | 提示词 | id, name, prompt, config, version |
| **Dataset** | 数据集 | id, name, projectId, metadata |
| **DatasetItem** | 数据集项 | id, datasetId, input, expectedOutput |
| **DatasetRun** | 数据集运行 | id, datasetId, name, metadata |
| **ScoreConfig** | 评分配置 | id, projectId, name, dataType |
| **Model** | 模型配置 | id, projectId, modelName, tokenizerId |
| **LlmApiKey** | LLM API 密钥 | id, projectId, provider, hashedKey |
| **Comment** | 评论 | id, projectId, objectType, objectId, content |
| **Posthog** | PostHog 集成 | id, projectId, apiKey |
| **Session** | 会话 | id, projectId, bookmarked |

#### 关系说明
```prisma
model Project {
  id             String   @id @default(cuid())
  name           String
  organizationId String
  
  organization   Organization     @relation(fields: [organizationId], references: [id])
  traces         Trace[]
  prompts        Prompt[]
  datasets       Dataset[]
  scores         Score[]
  members        ProjectMembership[]
  
  @@index([organizationId])
}

model Trace {
  id          String        @id @default(cuid())
  name        String
  projectId   String
  userId      String?
  metadata    Json?
  
  project     Project       @relation(fields: [projectId], references: [id])
  observations Observation[]
  scores      Score[]
  
  @@index([projectId, timestamp])
  @@index([userId])
}
```

#### Prisma Client 配置
**路径**：`packages/shared/src/db.ts`

```typescript
import { PrismaClient } from '@prisma/client'

export const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development'
    ? ['query', 'error', 'warn']
    : ['error'],
})
```

### 2.3 ClickHouse 数据访问（Kysely）

#### 数据表定义
**路径**：`packages/shared/src/tableDefinitions/`

**主要表**：

| 表名 | 说明 | 主要列 |
|-----|------|-------|
| **observations** | 观测数据 | id, trace_id, type, model, usage_details, cost_details |
| **traces** | 追踪数据（冗余） | id, project_id, user_id, metadata |
| **events** | 原始事件 | id, project_id, event_type, event_body |

#### ClickHouse 客户端
**路径**：`packages/shared/src/clickhouse.ts`

```typescript
import { Kysely } from 'kysely';
import { ClickHouseDialect } from 'kysely-clickhouse';

export const clickhouseClient = new Kysely({
  dialect: new ClickHouseDialect({
    url: process.env.CLICKHOUSE_URL,
    username: process.env.CLICKHOUSE_USER,
    password: process.env.CLICKHOUSE_PASSWORD,
  }),
});
```

#### 查询示例
```typescript
// 查询 Observations
const observations = await clickhouseClient
  .selectFrom('observations')
  .selectAll()
  .where('trace_id', '=', traceId)
  .where('project_id', '=', projectId)
  .execute();
```

### 2.4 Redis 数据访问

#### Redis 客户端
**路径**：`packages/shared/src/redis.ts`

```typescript
import Redis from 'ioredis';

export const redis = new Redis(
  process.env.REDIS_CONNECTION_STRING
);
```

#### 使用场景
- **Session 缓存**：NextAuth session
- **API 响应缓存**：减少数据库查询
- **BullMQ 任务队列**：异步任务
- **速率限制**：API 限流

### 2.5 S3 对象存储

#### S3 客户端
**路径**：`packages/shared/src/s3.ts`

```typescript
import { S3Client } from '@aws-sdk/client-s3';

export const s3Client = new S3Client({
  region: process.env.S3_REGION,
  endpoint: process.env.S3_ENDPOINT,
  credentials: {
    accessKeyId: process.env.S3_ACCESS_KEY_ID,
    secretAccessKey: process.env.S3_SECRET_ACCESS_KEY,
  },
});
```

#### 使用场景
- **原始事件存储**：Ingestion 事件
- **多媒体附件**：图片、文件
- **数据导出**：导出的 CSV/JSON 文件

---

## 3. 数据模型详解

### 3.1 核心数据模型（PostgreSQL）

#### User & Organization
```
Organization (组织)
  ├─ Project (项目)
  │   ├─ ProjectMembership (成员)
  │   ├─ ApiKey (API 密钥)
  │   ├─ Trace (追踪)
  │   ├─ Prompt (提示词)
  │   └─ Dataset (数据集)
  └─ User (用户)
```

#### Trace & Observation
```
Trace (追踪)
  ├─ Observation (观测) - 1:N
  ├─ Score (评分) - 1:N
  └─ Session (会话) - N:1
  
Observation 类型:
  - GENERATION (生成)
  - SPAN (跨度)
  - EVENT (事件)
```

#### Prompt 版本管理
```
Prompt
  ├─ version: 1 (初始版本)
  ├─ version: 2 (修改后)
  └─ version: 3 (当前版本)
  
每次修改创建新版本
```

#### Dataset & Evaluation
```
Dataset (数据集)
  ├─ DatasetItem (测试项) - 1:N
  └─ DatasetRun (运行记录) - 1:N
  
DatasetRun
  ├─ DatasetRunItem (运行项) - 1:N
  └─ 关联的 Traces
```

### 3.2 数据关系图

查看完整的 ERD：
**路径**：`packages/shared/prisma/database.svg`

---

## 4. 数据库迁移

### 4.1 Prisma 迁移（PostgreSQL）

#### 迁移文件
**路径**：`packages/shared/prisma/migrations/`

#### 创建迁移
```bash
# 开发环境
pnpm --filter=shared run db:migrate

# 生产环境
pnpm --filter=shared run db:deploy
```

#### 迁移命名
```
20231201120000_add_user_table/
  └─ migration.sql
```

### 4.2 ClickHouse 迁移

#### 迁移文件
**路径**：`packages/shared/clickhouse/migrations/`

**格式**：
```
001_create_observations_table.sql
002_add_cost_details_column.sql
```

#### 执行迁移
```bash
# 自动执行（通过脚本）
pnpm --filter=shared run ch:migrate
```

---

## 5. 查询模式和最佳实践

### 5.1 Prisma 查询模式

#### 基本查询
```typescript
// 查询单个
const trace = await prisma.trace.findUnique({
  where: { id: traceId },
});

// 查询多个
const traces = await prisma.trace.findMany({
  where: { projectId },
  orderBy: { timestamp: 'desc' },
  take: 50,
});
```

#### 关联查询
```typescript
// Include 关联数据
const trace = await prisma.trace.findUnique({
  where: { id: traceId },
  include: {
    observations: true,
    scores: true,
    project: {
      include: {
        organization: true,
      },
    },
  },
});
```

#### 聚合查询
```typescript
// 统计
const count = await prisma.trace.count({
  where: { projectId },
});

// 分组统计
const stats = await prisma.observation.groupBy({
  by: ['model'],
  where: { traceId },
  _count: true,
  _sum: {
    totalCost: true,
  },
});
```

#### 事务
```typescript
await prisma.$transaction(async (tx) => {
  const trace = await tx.trace.create({
    data: { /* ... */ },
  });
  
  await tx.observation.createMany({
    data: observations.map(obs => ({
      traceId: trace.id,
      ...obs,
    })),
  });
});
```

### 5.2 ClickHouse 查询模式

#### 基本查询
```typescript
import { clickhouseClient } from '@langfuse/shared/src/clickhouse';

const observations = await clickhouseClient
  .selectFrom('observations')
  .select(['id', 'type', 'model', 'totalCost'])
  .where('trace_id', '=', traceId)
  .execute();
```

#### 聚合查询
```typescript
const stats = await clickhouseClient
  .selectFrom('observations')
  .select([
    'model',
    sql`count(*)`.as('count'),
    sql`sum(totalCost)`.as('total_cost'),
  ])
  .where('project_id', '=', projectId)
  .groupBy('model')
  .execute();
```

#### 时序查询
```typescript
const timeline = await clickhouseClient
  .selectFrom('observations')
  .select([
    sql`toStartOfHour(timestamp)`.as('hour'),
    sql`count(*)`.as('count'),
  ])
  .where('project_id', '=', projectId)
  .where('timestamp', '>=', startDate)
  .groupBy(sql`toStartOfHour(timestamp)`)
  .orderBy('hour', 'asc')
  .execute();
```

---

## 6. 性能优化

### 6.1 数据库索引

#### PostgreSQL 索引
**在 Prisma Schema 中定义**：
```prisma
model Trace {
  id        String   @id
  projectId String
  timestamp DateTime
  
  @@index([projectId, timestamp])
  @@index([userId])
}
```

#### ClickHouse 索引
```sql
CREATE TABLE observations (
  id String,
  trace_id String,
  project_id String,
  timestamp DateTime
) ENGINE = MergeTree()
ORDER BY (project_id, trace_id, timestamp)
PARTITION BY toYYYYMM(timestamp);
```

### 6.2 查询优化

#### 使用 Select 减少数据传输
```typescript
// ❌ 不好：查询所有字段
const traces = await prisma.trace.findMany();

// ✅ 好：只查询需要的字段
const traces = await prisma.trace.findMany({
  select: {
    id: true,
    name: true,
    timestamp: true,
  },
});
```

#### 避免 N+1 查询
```typescript
// ❌ 不好：N+1 查询
const traces = await prisma.trace.findMany();
for (const trace of traces) {
  const scores = await prisma.score.findMany({
    where: { traceId: trace.id },
  });
}

// ✅ 好：使用 include
const traces = await prisma.trace.findMany({
  include: {
    scores: true,
  },
});
```

#### 批量操作
```typescript
// ❌ 不好：多次单独插入
for (const item of items) {
  await prisma.observation.create({ data: item });
}

// ✅ 好：批量插入
await prisma.observation.createMany({
  data: items,
});
```

### 6.3 连接池

#### Prisma 连接池配置
```env
DATABASE_URL="postgresql://...?connection_limit=10"
```

#### ClickHouse 连接池
```typescript
const clickhouseClient = new Kysely({
  dialect: new ClickHouseDialect({
    // 连接池配置
    pool: {
      min: 2,
      max: 10,
    },
  }),
});
```

---

## 7. 数据一致性

### 7.1 事务保证（PostgreSQL）

**ACID 保证**：
- **Atomicity**：事务全部成功或全部失败
- **Consistency**：数据一致性约束
- **Isolation**：事务隔离级别
- **Durability**：数据持久化

### 7.2 最终一致性（ClickHouse）

**特点**：
- 异步写入
- 最终一致性
- 适合追加写入，不适合频繁更新

**设计考虑**：
- PostgreSQL 存储元数据（强一致性）
- ClickHouse 存储分析数据（最终一致性）
- 通过 Worker 异步同步数据

---

## 8. 监控和维护

### 8.1 查询监控

#### Prisma 日志
```typescript
const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
    { level: 'error', emit: 'stdout' },
  ],
});

prisma.$on('query', (e) => {
  console.log('Query: ' + e.query);
  console.log('Duration: ' + e.duration + 'ms');
});
```

#### 慢查询监控
- 记录超过阈值的查询
- 使用 OpenTelemetry 追踪

### 8.2 数据库健康检查

```typescript
export async function checkDatabaseHealth() {
  try {
    await prisma.$queryRaw`SELECT 1`;
    await clickhouseClient.selectFrom('observations').select('1').limit(1).execute();
    return { status: 'healthy' };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}
```

---

## 9. 备份和恢复

### 9.1 PostgreSQL 备份
- 定期全量备份
- WAL 归档（Point-in-Time Recovery）
- 自动备份脚本

### 9.2 ClickHouse 备份
- 使用 `BACKUP` 命令
- S3 备份存储
- 增量备份

### 9.3 灾难恢复计划
- RTO（Recovery Time Objective）
- RPO（Recovery Point Objective）
- 恢复流程文档

---

## 10. 关键代码文件路径

### 10.1 Prisma 相关

```
packages/shared/prisma/
├── schema.prisma                # 数据库 Schema
├── migrations/                  # 迁移文件
│   ├── 20231201120000_init/
│   └── 20231202130000_add_scores/
└── generated/                   # 生成的 Prisma Client
```

### 10.2 ClickHouse 相关

```
packages/shared/
├── clickhouse/
│   ├── migrations/              # ClickHouse DDL
│   └── scripts/                 # 管理脚本
└── src/
    ├── tableDefinitions/        # 表定义（TypeScript）
    └── clickhouse.ts            # 客户端配置
```

### 10.3 数据库客户端

```
packages/shared/src/
├── db.ts                        # Prisma 客户端
├── clickhouse.ts                # ClickHouse 客户端
├── redis.ts                     # Redis 客户端
└── s3.ts                        # S3 客户端
```

---

## 11. 总结

数据访问层是 Langfuse 的基础设施核心，通过多数据库架构实现了高性能和高可用性。Prisma 提供类型安全的 ORM，ClickHouse 提供高性能分析能力，Redis 提供缓存和队列支持。

**关键特点**：
- ✅ 多数据库架构（OLTP + OLAP）
- ✅ 类型安全的数据访问（Prisma）
- ✅ 高性能分析查询（ClickHouse）
- ✅ 完善的迁移管理
- ✅ 性能优化和监控

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0
