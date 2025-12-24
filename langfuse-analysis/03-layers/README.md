# Langfuse 分层架构总览

## 1. 架构概览

Langfuse 采用**分层架构**（Layered Architecture），将系统按职责分为多个层次，每层专注于特定功能，层与层之间通过明确的接口通信。

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    用户/客户端 (Browser/SDK)                  │
└─────────────────────────────────────────────────────────────┘
                              ↓ HTTP/WebSocket
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: 前端展示层 (Frontend Presentation Layer)          │
│  - Next.js Pages                                            │
│  - React Components (shadcn/ui)                             │
│  - React Query (状态管理)                                    │
└─────────────────────────────────────────────────────────────┘
                              ↓ tRPC
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: tRPC API 层 (tRPC API Layer)                      │
│  - tRPC Routers (60+ routers)                               │
│  - Input Validation (Zod schemas)                           │
│  - Middleware (auth, RBAC)                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 业务服务层 (Business Service Layer)               │
│  - Feature Modules (traces, prompts, datasets, evals...)    │
│  - Domain Logic                                             │
│  - Business Rules                                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: 数据访问层 (Data Access Layer)                    │
│  - Prisma ORM (PostgreSQL)                                  │
│  - Kysely (ClickHouse)                                      │
│  - Redis Client                                             │
│  - S3 Client                                                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Infrastructure: 基础设施层                                  │
│  - PostgreSQL (元数据)                                       │
│  - ClickHouse (分析数据)                                     │
│  - Redis (缓存/队列)                                         │
│  - S3 (对象存储)                                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Horizontal: 共享包层 (Shared Packages Layer)               │
│  - Types & Interfaces                                       │
│  - Domain Models                                            │
│  - Utilities (encryption, validation, logger)               │
│  - Database Clients                                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Async: Worker 服务层 (Worker Service Layer)                │
│  - BullMQ Queues (ingestion, evaluation, sync)              │
│  - Background Jobs                                          │
│  - Data Sync (PostgreSQL → ClickHouse)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 各层详解

### Layer 1: 前端展示层
**路径**: `web/src/`  
**文档**: [01-frontend-presentation-layer.md](./01-frontend-presentation-layer.md)

**职责**：
- 用户界面渲染（React + Next.js）
- 用户交互处理
- 状态管理（React Query）
- 路由管理（Next.js Pages Router）
- 样式管理（Tailwind CSS）

**关键技术**：
- Next.js 15.5.9 (Pages Router)
- React 19.2.3
- Tailwind CSS + shadcn/ui
- TanStack Query (React Query)
- Recharts (图表)

**主要目录**：
```
web/src/
├── pages/                  # Next.js 页面
├── components/             # React 组件
└── styles/                 # 全局样式
```

---

### Layer 2: tRPC API 层
**路径**: `web/src/server/api/`  
**文档**: [02-trpc-api-layer.md](./02-trpc-api-layer.md)

**职责**：
- 定义 API 接口（60+ routers）
- 输入验证（Zod schemas）
- 认证和授权（Middleware）
- 错误处理
- API 文档生成

**关键技术**：
- tRPC 11.4.4
- Zod 3.25.62
- NextAuth.js

**主要目录**：
```
web/src/server/api/
├── routers/                # tRPC Routers
│   ├── traces.ts           # Traces API
│   ├── prompts.ts          # Prompts API
│   ├── datasets.ts         # Datasets API
│   └── ...                 # 60+ routers
├── trpc.ts                 # tRPC 配置
└── root.ts                 # Root Router
```

**核心 Routers**：
- traces - 追踪数据
- observations - 观测数据
- scores - 评分
- prompts - 提示词
- datasets - 数据集
- evals - 评估
- models - 模型配置
- dashboard - 仪表板数据

---

### Layer 3: 业务服务层
**路径**: `web/src/features/`, `packages/shared/src/features/`  
**文档**: [03-business-service-layer.md](./03-business-service-layer.md)

**职责**：
- 业务逻辑实现
- 领域模型管理
- 业务规则验证
- 特性模块组织（60+ 特性）

**关键技术**：
- TypeScript
- Domain-Driven Design (DDD)
- Feature-Sliced Design

**主要目录**：
```
web/src/features/
├── traces/                 # 追踪特性
├── prompts/                # 提示词特性
├── datasets/               # 数据集特性
├── evals/                  # 评估特性
├── scores/                 # 评分特性
├── playground/             # Playground 特性
├── dashboard/              # 仪表板特性
└── ...                     # 60+ 特性模块

packages/shared/src/features/
├── cost-calculation/       # 成本计算
├── tokenization/           # Token 计数
├── rate-limiting/          # 速率限制
├── ingestion/              # 数据摄取
└── auth/                   # 认证
```

**核心特性模块**：
- **traces**: Trace 追踪和管理
- **prompts**: 提示词版本管理
- **datasets**: 数据集管理和评估
- **evals**: LLM 评估框架
- **scores**: 评分系统
- **playground**: LLM Playground
- **dashboard**: 数据可视化

---

### Layer 4: 数据访问层
**路径**: `packages/shared/src/`, `packages/shared/prisma/`  
**文档**: [04-data-access-layer.md](./04-data-access-layer.md)

**职责**：
- 数据库连接管理
- 数据持久化（CRUD）
- 数据库迁移
- 查询优化
- 多数据库协调（PostgreSQL + ClickHouse）

**关键技术**：
- Prisma 6.17.1 (PostgreSQL ORM)
- Kysely (ClickHouse Query Builder)
- ioredis (Redis 客户端)
- AWS SDK (S3 客户端)

**数据库架构**：

| 数据库 | 类型 | 用途 | 访问方式 |
|-------|------|------|---------|
| **PostgreSQL** | OLTP | 元数据、配置、用户数据 | Prisma ORM |
| **ClickHouse** | OLAP | 观测数据、时序数据、分析 | Kysely |
| **Redis** | 缓存/队列 | Session、缓存、任务队列 | ioredis |
| **S3/Blob** | 对象存储 | 原始事件、附件 | AWS SDK |

**主要目录**：
```
packages/shared/
├── prisma/
│   ├── schema.prisma       # Prisma Schema
│   └── migrations/         # 数据库迁移
├── clickhouse/
│   ├── migrations/         # ClickHouse DDL
│   └── scripts/            # 管理脚本
└── src/
    ├── db.ts               # Prisma 客户端
    ├── clickhouse.ts       # ClickHouse 客户端
    └── tableDefinitions/   # ClickHouse 表定义
```

**核心数据模型**：
- User, Organization, Project
- Trace, Observation, Score
- Prompt, Dataset, DatasetItem
- Model, LlmApiKey, ScoreConfig

---

### Layer 5: 共享包层（横向层）
**路径**: `packages/shared/src/`  
**文档**: [05-shared-packages-layer.md](./05-shared-packages-layer.md)

**职责**：
- 提供跨服务共享代码
- 定义统一的类型系统
- 实现通用工具函数
- 管理加密和安全
- 定义常量和配置

**关键技术**：
- TypeScript
- Zod (运行时验证)
- Crypto (加密)
- Pino (日志)

**主要目录**：
```
packages/shared/src/
├── constants.ts            # 全局常量
├── types.ts                # 通用类型
├── env.ts                  # 环境变量
├── domain/                 # 领域模型
│   ├── traces.ts
│   ├── observations.ts
│   └── scores.ts
├── encryption/             # 加密功能
├── errors/                 # 错误类
├── features/               # 共享特性
│   ├── cost-calculation/
│   ├── tokenization/
│   ├── rate-limiting/
│   └── auth/
├── server/                 # 服务端工具
│   ├── redis/
│   └── clickhouse/
└── utils/                  # 工具函数
    ├── logger.ts
    ├── validation.ts
    └── dates.ts
```

**核心模块**：
- **Types**: 类型系统和 Zod schemas
- **Domain**: 领域模型和逻辑
- **Encryption**: AES-256-GCM 加密
- **Errors**: 自定义错误类层次
- **Features**: 共享业务特性
- **Utils**: 工具函数库

---

### Layer 6: Worker 服务层（异步层）
**路径**: `worker/src/`  
**文档**: [06-worker-service-layer.md](./06-worker-service-layer.md)

**职责**：
- 处理后台异步任务
- 执行耗时的数据处理
- 管理数据同步（PostgreSQL → ClickHouse）
- 执行数据集评估
- 处理 Webhook 和外部集成

**关键技术**：
- BullMQ (任务队列)
- Redis (队列存储)
- Express (HTTP API)

**主要目录**：
```
worker/src/
├── queues/                 # BullMQ 队列
│   ├── ingestionQueue.ts   # 数据摄取队列
│   ├── evaluationQueue.ts  # 评估队列
│   ├── clickhouseSyncQueue.ts  # ClickHouse 同步
│   └── webhookQueue.ts     # Webhook 队列
├── services/               # 业务服务
│   ├── IngestionService.ts
│   ├── EvaluationService.ts
│   └── ClickHouseSyncService.ts
└── backgroundMigrations/   # 后台迁移
```

**核心队列**：
- **ingestionQueue**: 数据摄取（SDK → DB）
- **evaluationQueue**: LLM 评估和打分
- **clickhouseSyncQueue**: PostgreSQL → ClickHouse 同步
- **webhookQueue**: Webhook 通知
- **batchExportQueue**: 批量导出（CSV/JSON）

---

## 3. 数据流示例

### 3.1 用户查询 Trace 数据流

```
1. User (Browser)
    ↓ HTTP Request
2. Frontend Layer (Next.js Page)
    ↓ tRPC Client
3. tRPC API Layer (traces.getById)
    ↓ Auth Middleware → Router → Procedure
4. Business Service Layer (TraceService)
    ↓ Business Logic
5. Data Access Layer (Prisma)
    ↓ SQL Query
6. Infrastructure (PostgreSQL)
    ↓ Result
7. Data Access Layer (Transform)
    ↓
8. Business Service Layer (Calculate Cost)
    ↓
9. tRPC API Layer (Response)
    ↓
10. Frontend Layer (Display)
    ↓
11. User (Browser)
```

### 3.2 SDK 数据摄取流

```
1. SDK Client
    ↓ HTTP POST /api/ingestion
2. tRPC API Layer (ingestion.create)
    ↓ Validate Event
3. Business Service Layer
    ↓ Add to Queue
4. Worker Service Layer (ingestionQueue)
    ↓ Process Job
5. Data Access Layer
    ↓ Write to PostgreSQL
6. Infrastructure (PostgreSQL)
    ↓ Success
7. Worker Service Layer (clickhouseSyncQueue)
    ↓ Sync to ClickHouse
8. Data Access Layer (Kysely)
    ↓ Write to ClickHouse
9. Infrastructure (ClickHouse)
```

### 3.3 数据集评估流

```
1. User (Browser)
    ↓ Click "Run Evaluation"
2. Frontend Layer
    ↓ tRPC Client
3. tRPC API Layer (datasets.runEvaluation)
    ↓ Create DatasetRun
4. Business Service Layer
    ↓ Add to Queue
5. Worker Service Layer (evaluationQueue)
    ↓ Process Each Item
6. Worker Service Layer (EvaluationService)
    ↓ Run Evaluator (LLM-as-Judge)
7. External Service (OpenAI API)
    ↓ Response
8. Worker Service Layer
    ↓ Calculate Score
9. Data Access Layer (Prisma)
    ↓ Save Score
10. Infrastructure (PostgreSQL)
```

---

## 4. 层间通信规则

### 4.1 通信方向

- **单向依赖**: 上层依赖下层，下层不依赖上层
- **跨层调用**: 禁止跨层调用（如前端直接访问数据层）
- **水平通信**: 同层内模块可以相互调用

```
Frontend ──→ tRPC API ──→ Business Service ──→ Data Access ──→ Infrastructure
    ↑                                                ↑
    └────────────── 禁止直接访问 ───────────────────┘
```

### 4.2 接口契约

- **Frontend ↔ tRPC API**: tRPC Procedures (TypeScript)
- **tRPC API ↔ Business Service**: Function Calls
- **Business Service ↔ Data Access**: Repository Pattern
- **Web ↔ Worker**: BullMQ Jobs (JSON)

---

## 5. 架构优势

### 5.1 关注点分离
- 每层专注于特定职责
- 降低模块耦合度
- 提高代码可维护性

### 5.2 可测试性
- 每层可独立测试
- Mock 层间接口
- 单元测试覆盖率高

### 5.3 可扩展性
- 横向扩展（多实例）
- 垂直扩展（层内优化）
- 微服务解耦（Web + Worker）

### 5.4 技术选型灵活
- 每层可独立选择技术栈
- 易于替换底层实现
- 支持渐进式迁移

---

## 6. 技术栈总结

### 6.1 前端技术栈
- **框架**: Next.js 15.5.9 (Pages Router)
- **UI 库**: React 19.2.3
- **组件库**: shadcn/ui (Radix UI + Tailwind)
- **状态管理**: TanStack Query (React Query)
- **样式**: Tailwind CSS
- **图表**: Recharts

### 6.2 后端技术栈
- **API 框架**: tRPC 11.4.4
- **ORM**: Prisma 6.17.1
- **查询构建器**: Kysely
- **认证**: NextAuth.js
- **校验**: Zod 3.25.62
- **队列**: BullMQ

### 6.3 数据库技术栈
- **OLTP**: PostgreSQL 15+
- **OLAP**: ClickHouse 24+
- **缓存**: Redis 7+
- **存储**: S3/Azure Blob

### 6.4 基础设施技术栈
- **运行时**: Node.js 24
- **包管理**: pnpm + Turborepo
- **容器**: Docker + Docker Compose
- **监控**: OpenTelemetry + Prometheus

---

## 7. 文档导航

| 层名称 | 文档路径 | 说明 |
|-------|---------|------|
| **前端展示层** | [01-frontend-presentation-layer.md](./01-frontend-presentation-layer.md) | React + Next.js UI 层 |
| **tRPC API 层** | [02-trpc-api-layer.md](./02-trpc-api-layer.md) | API 接口定义层 |
| **业务服务层** | [03-business-service-layer.md](./03-business-service-layer.md) | 业务逻辑层 |
| **数据访问层** | [04-data-access-layer.md](./04-data-access-layer.md) | 数据持久化层 |
| **共享包层** | [05-shared-packages-layer.md](./05-shared-packages-layer.md) | 横向共享代码层 |
| **Worker 服务层** | [06-worker-service-layer.md](./06-worker-service-layer.md) | 异步任务处理层 |

---

## 8. 架构演进建议

### 8.1 短期优化
- ✅ 进一步解耦业务逻辑（从 tRPC routers 提取到 services）
- ✅ 完善单元测试和集成测试
- ✅ 优化 ClickHouse 查询性能
- ✅ 增强 API 文档（OpenAPI）

### 8.2 中期规划
- 🔄 考虑迁移到 App Router（Next.js）
- 🔄 引入 GraphQL 支持（补充 tRPC）
- 🔄 实现更细粒度的微服务拆分
- 🔄 增加 CQRS 模式（读写分离）

### 8.3 长期愿景
- 🚀 多租户架构优化
- 🚀 全球化部署（CDN + 边缘计算）
- 🚀 实时数据流（WebSocket/SSE）
- 🚀 AI 驱动的智能分析

---

## 9. 最佳实践

### 9.1 代码组织
- ✅ 按层、按特性组织代码
- ✅ 使用 Barrel Exports (`index.ts`)
- ✅ 遵循命名约定（PascalCase/camelCase）

### 9.2 类型安全
- ✅ 使用 TypeScript strict 模式
- ✅ Zod schemas 运行时验证
- ✅ 避免使用 `any`

### 9.3 错误处理
- ✅ 统一错误类层次
- ✅ 结构化错误日志
- ✅ 友好的错误提示

### 9.4 性能优化
- ✅ 数据库查询优化（索引、分页）
- ✅ 缓存策略（Redis）
- ✅ 异步处理（BullMQ）
- ✅ 代码分割（Next.js dynamic import）

---

## 10. 总结

Langfuse 的分层架构是一个**现代化、可扩展、高性能**的设计，通过清晰的层次划分和职责分离，实现了代码的高度模块化和可维护性。

**核心亮点**：
- ✅ **清晰的层次划分**：前端、API、业务、数据、基础设施
- ✅ **微服务架构**：Web + Worker 解耦
- ✅ **多数据库架构**：OLTP (PostgreSQL) + OLAP (ClickHouse)
- ✅ **类型安全**：TypeScript + Zod + tRPC
- ✅ **异步处理**：BullMQ 队列系统
- ✅ **横向共享**：@langfuse/shared 包
- ✅ **可观测性**：OpenTelemetry + Prometheus

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0  
**架构风格**：分层架构 (Layered Architecture)
