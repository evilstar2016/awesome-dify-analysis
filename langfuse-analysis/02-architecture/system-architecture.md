# Langfuse 系统架构

## 1. 架构概述

### 1.1 架构模式

Langfuse 采用**混合架构模式**，结合了以下几种架构风格：

| 架构模式 | 应用场景 | 说明 |
|---------|---------|------|
| **Monorepo 架构** | 整体代码组织 | 使用 pnpm workspace + Turborepo 管理多个包 |
| **前后端分离架构** | Web 应用 | Next.js 同时提供前端和后端能力 |
| **微服务架构** | 服务拆分 | Web 服务和 Worker 服务独立部署和扩展 |
| **多数据库架构** | 数据存储 | OLTP（PostgreSQL）+ OLAP（ClickHouse）分离 |
| **事件驱动架构** | 异步处理 | 基于 BullMQ 的消息队列实现异步任务 |
| **分层架构** | 代码组织 | 展示层、API层、业务层、数据层清晰分离 |

### 1.2 架构特点

#### 🎯 核心设计原则
1. **类型安全**：TypeScript 全栈，tRPC 提供端到端类型安全
2. **可扩展性**：独立的 Worker 服务支持水平扩展
3. **高性能**：多数据库架构，针对不同场景优化
4. **可观测性**：内置 OpenTelemetry 追踪和监控
5. **开发友好**：Monorepo 简化开发，共享代码减少重复

#### 🔧 技术栈统一性
- **语言**：全栈 TypeScript
- **运行时**：Node.js 24
- **ORM**：Prisma（PostgreSQL）+ Kysely（ClickHouse）
- **API**：tRPC（内部）+ REST（外部）
- **队列**：BullMQ + Redis

---

## 2. 系统组成

### 2.1 核心服务组件

```
┌─────────────────────────────────────────────────────────┐
│                    Langfuse 系统                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────────┐         ┌─────────────────┐       │
│  │  Web 服务        │         │  Worker 服务     │       │
│  │  (Next.js)      │◄───────►│  (Node.js)      │       │
│  │                 │  Queue  │                 │       │
│  │  - UI 渲染      │         │  - 异步任务      │       │
│  │  - API 服务     │         │  - 数据聚合      │       │
│  │  - SSR/SSG      │         │  - 后台迁移      │       │
│  └─────────────────┘         └─────────────────┘       │
│         ▲ ▼                          ▲ ▼                │
│  ┌──────────────────────────────────────────┐          │
│  │         共享包 (packages/shared)          │          │
│  │  - Prisma Schema                         │          │
│  │  - 共享类型和工具                         │          │
│  │  - ClickHouse 迁移                       │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 2.2 基础设施组件

```
┌─────────────────────────────────────────────────┐
│              基础设施层                          │
├─────────────────────────────────────────────────┤
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │PostgreSQL│  │ClickHouse│  │  Redis   │     │
│  │  (OLTP)  │  │  (OLAP)  │  │(Cache/Q) │     │
│  └──────────┘  └──────────┘  └──────────┘     │
│                                                  │
│  ┌──────────────────────────────────────┐      │
│  │      S3 / Blob Storage               │      │
│  │  (原始事件、附件)                      │      │
│  └──────────────────────────────────────┘      │
│                                                  │
└─────────────────────────────────────────────────┘
```

### 2.3 外部集成

```
┌──────────────────────────────────────────┐
│            外部服务                        │
├──────────────────────────────────────────┤
│                                           │
│  ┌──────────┐  ┌──────────┐             │
│  │LLM API   │  │Monitoring│             │
│  │(可选)     │  │Services  │             │
│  │- OpenAI  │  │- Sentry  │             │
│  │- Anthropic│  │- Datadog │             │
│  │- 自定义   │  │- PostHog │             │
│  └──────────┘  └──────────┘             │
│                                           │
└──────────────────────────────────────────┘
```

---

## 3. 核心组件详解

### 3.1 Web 服务（web/）

#### 职责
- 提供用户界面（React + Next.js）
- 处理 HTTP 请求（API Routes）
- 实现 tRPC 类型安全 API
- 用户认证和授权（NextAuth.js）
- 实时数据查询和展示
- 异步任务入队（BullMQ）

#### 主要子模块

| 模块 | 路径 | 功能 |
|-----|------|------|
| **页面路由** | `src/pages/` | Next.js 页面组件和 API 路由 |
| **tRPC 路由** | `src/server/api/routers/` | 类型安全的 API 端点 |
| **React 组件** | `src/components/` | 可复用的 UI 组件 |
| **功能模块** | `src/features/` | 按业务功能组织的代码（traces, prompts, evals） |
| **认证层** | `src/server/auth/` | NextAuth 配置和 API Key 管理 |
| **工具函数** | `src/utils/` | 通用工具函数 |
| **企业功能** | `src/ee/` | 企业版特定功能 |

#### 技术实现
- **框架**：Next.js 15.5.9（pages router）
- **UI**：React 19 + shadcn/ui（Radix UI）+ Tailwind CSS
- **API**：tRPC 11.4 + Zod 验证
- **数据库访问**：Prisma ORM
- **缓存**：React Query（TanStack Query）
- **认证**：NextAuth.js 4.24

#### 代码路径
```
web/
├── src/
│   ├── pages/              # Next.js 页面和 API
│   │   ├── index.tsx       # 首页
│   │   ├── api/            # API 路由
│   │   │   ├── auth/       # 认证 API
│   │   │   └── public/     # 公开 REST API
│   │   └── project/        # 项目页面
│   ├── server/             # 服务器端代码
│   │   ├── api/            # tRPC 路由器
│   │   └── auth/           # 认证逻辑
│   ├── components/         # UI 组件
│   └── features/           # 功能模块
```

### 3.2 Worker 服务（worker/）

#### 职责
- 处理异步后台任务
- 数据聚合和统计
- 事件处理（从 S3 读取）
- 后台数据迁移
- 定时任务执行
- 评估任务执行（LLM-as-a-judge）

#### 主要子模块

| 模块 | 路径 | 功能 |
|-----|------|------|
| **队列处理** | `src/queues/` | BullMQ 队列定义和处理器 |
| **业务服务** | `src/services/` | 核心业务逻辑 |
| **功能模块** | `src/features/` | 业务功能实现（billing, evals） |
| **后台迁移** | `src/backgroundMigrations/` | 数据迁移任务 |
| **脚本工具** | `src/scripts/` | 运维脚本（数据回填等） |
| **API 服务** | `src/api/` | Express HTTP API（健康检查等） |

#### 技术实现
- **框架**：Node.js + Express 5
- **队列**：BullMQ 5.34
- **数据库**：Prisma + Kysely
- **任务调度**：BullMQ 定时任务
- **重试机制**：指数退避策略

#### 代码路径
```
worker/
├── src/
│   ├── queues/             # 队列处理
│   │   ├── ingestionQueue.ts
│   │   ├── evalQueue.ts
│   │   └── ...
│   ├── services/           # 业务服务
│   ├── features/           # 功能模块
│   │   ├── billing/
│   │   └── evals/
│   ├── backgroundMigrations/
│   ├── api/                # Express API
│   └── index.ts            # 入口文件
```

### 3.3 共享包（packages/shared/）

#### 职责
- 提供共享的数据库模型（Prisma）
- 提供共享的类型定义
- 提供通用工具函数
- 管理 ClickHouse 迁移
- 提供加密和安全工具

#### 主要内容

| 模块 | 路径 | 功能 |
|-----|------|------|
| **Prisma Schema** | `prisma/schema.prisma` | 数据库模型定义 |
| **数据库客户端** | `src/db.ts` | Prisma 和 ClickHouse 客户端 |
| **共享类型** | `src/types.ts` | 全局类型定义 |
| **领域模型** | `src/domain/` | 业务领域模型 |
| **加密工具** | `src/encryption/` | 数据加密解密 |
| **ClickHouse** | `clickhouse/migrations/` | OLAP 数据库迁移 |

#### 技术实现
- **ORM**：Prisma 6.17
- **查询构建**：Kysely 0.27（ClickHouse）
- **类型生成**：Prisma 自动生成 TypeScript 类型

#### 代码路径
```
packages/shared/
├── prisma/
│   ├── schema.prisma       # 数据库 Schema
│   ├── migrations/         # Prisma 迁移文件
│   └── generated/          # 生成的客户端
├── clickhouse/
│   ├── migrations/         # ClickHouse DDL
│   └── scripts/            # ClickHouse 脚本
└── src/
    ├── db.ts               # 数据库客户端
    ├── types.ts            # 共享类型
    ├── features/           # 共享功能
    ├── domain/             # 领域模型
    └── encryption/         # 加密工具
```

### 3.4 企业版（ee/）

#### 职责
- 企业版许可证检查
- 企业特定功能
- 高级 SSO/SAML 支持
- 高级权限控制

#### 实现方式
- 模块化设计，可选加载
- 许可证验证机制
- 与核心代码解耦

---

## 4. 数据架构

### 4.1 多数据库架构

Langfuse 使用**读写分离**和**OLTP/OLAP 分离**的数据架构：

#### PostgreSQL（OLTP - 事务处理）
**用途**：
- 用户、项目、组织数据
- API Keys 和 Sessions
- 提示词（Prompts）配置
- 评估（Evaluations）配置
- 元数据和关系

**特点**：
- ACID 事务保证
- 复杂关系查询
- 实时读写
- 相对低吞吐量

#### ClickHouse（OLAP - 分析处理）
**用途**：
- Traces（追踪记录）
- Observations（观测数据）
- 事件日志
- 时序数据
- 聚合指标

**特点**：
- 高吞吐量写入
- 列式存储
- 快速分析查询
- 时序数据优化

#### Redis（缓存和队列）
**用途**：
- Session 缓存
- API 响应缓存
- BullMQ 任务队列
- 速率限制

#### S3/Blob Storage（对象存储）
**用途**：
- 原始摄取事件
- 多模态附件（图片、文件）
- 数据导出文件

### 4.2 数据流向

#### 写入流程（Ingestion）
```
SDK/API
  ↓
Web Server (验证、初步处理)
  ↓
S3 (原始事件存储)
  ↓
Redis Queue (异步任务队列)
  ↓
Worker (解析、转换)
  ↓
ClickHouse (观测数据) + PostgreSQL (元数据)
```

#### 查询流程
```
用户请求
  ↓
Web Server (tRPC API)
  ↓
Prisma (PostgreSQL - 元数据)
  ↓
Kysely (ClickHouse - 分析数据)
  ↓
数据聚合、关联
  ↓
返回给前端
```

---

## 5. 组件依赖关系

### 5.1 服务间依赖

```
┌──────────┐
│  用户     │
└─────┬────┘
      ▼
┌──────────────┐
│  Web 服务     │◄─────────┐
│  (Next.js)   │          │
└──────┬───────┘          │
       │                  │
       ├──► PostgreSQL    │ 共享数据库
       ├──► ClickHouse    │
       ├──► Redis ────────┤
       └──► S3            │
                          │
┌──────────────┐          │
│ Worker 服务   │──────────┘
│ (Node.js)    │
└──────┬───────┘
       │
       ├──► PostgreSQL
       ├──► ClickHouse
       ├──► Redis (队列消费)
       ├──► S3
       └──► LLM API (可选)
```

### 5.2 包依赖关系

```
┌────────────────┐      ┌────────────────┐
│   web 包       │      │   worker 包     │
└───────┬────────┘      └────────┬───────┘
        │                        │
        └────────────┬───────────┘
                     ▼
           ┌─────────────────┐
           │  shared 包      │
           │                 │
           │ - Prisma Schema │
           │ - 类型定义       │
           │ - 工具函数       │
           └─────────────────┘
```

**依赖规则**：
- `web` 依赖 `shared`
- `worker` 依赖 `shared`
- `web` 和 `worker` 不直接依赖
- `shared` 不依赖其他包

### 5.3 模块内部依赖

#### Web 服务内部
```
pages/           (页面组件)
  ↓
components/      (UI 组件)
  ↓
server/api/      (tRPC API)
  ↓
features/        (业务逻辑)
  ↓
utils/           (工具函数)
```

#### Worker 服务内部
```
index.ts         (入口)
  ↓
queues/          (队列处理器)
  ↓
services/        (业务服务)
  ↓
features/        (业务逻辑)
  ↓
utils/           (工具函数)
```

---

## 6. 数据流详解

### 6.1 Trace 摄取流程

```
1. SDK/API 调用
   ↓
2. Web Server 接收
   - 验证 API Key
   - 初步数据验证
   - 写入 S3（原始事件）
   ↓
3. 异步入队（Redis）
   - 创建 ingestion 任务
   ↓
4. Worker 处理
   - 从队列获取任务
   - 从 S3 读取原始事件
   - 解析和转换
   - 数据增强
   ↓
5. 写入数据库
   - ClickHouse（Traces, Observations）
   - PostgreSQL（元数据关联）
   ↓
6. 触发后续任务
   - 计费更新
   - 数据聚合
   - 通知触发
```

### 6.2 仪表盘查询流程

```
1. 用户请求仪表盘
   ↓
2. tRPC API 调用
   - 认证和授权检查
   ↓
3. 查询 PostgreSQL
   - 获取项目元数据
   - 获取用户权限
   ↓
4. 查询 ClickHouse
   - 分析 Traces 数据
   - 计算聚合指标
   ↓
5. 数据合并和处理
   - 关联元数据
   - 格式化输出
   ↓
6. 返回前端渲染
```

### 6.3 评估执行流程

```
1. 用户触发评估
   ↓
2. Web Server 创建评估任务
   - 验证配置
   - 入队（Redis）
   ↓
3. Worker 处理评估任务
   - 获取评估配置
   - 获取目标 Traces
   ↓
4. 调用 LLM API（如配置）
   - LLM-as-a-judge
   - 获取评分
   ↓
5. 保存评估结果
   - 写入 PostgreSQL
   - 更新统计数据
   ↓
6. 通知用户（可选）
```

---

## 7. 外部集成点

### 7.1 必需的外部系统

| 系统 | 用途 | 访问方式 |
|-----|------|---------|
| **PostgreSQL** | 主数据库 | Prisma ORM |
| **ClickHouse** | 分析数据库 | Kysely + HTTP |
| **Redis** | 缓存和队列 | ioredis |

### 7.2 可选的外部系统

| 系统 | 用途 | 访问方式 | 使用场景 |
|-----|------|---------|---------|
| **S3/Blob Storage** | 对象存储 | AWS SDK | 高吞吐量摄取、附件存储 |
| **OpenAI** | LLM API | HTTP + API Key | Playground、评估 |
| **Anthropic** | LLM API | HTTP + API Key | Playground、评估 |
| **自定义 LLM** | LLM Gateway | HTTP | 企业自托管 |
| **Sentry** | 错误监控 | Sentry SDK | 生产环境监控 |
| **Datadog** | APM | dd-trace | 性能监控 |
| **PostHog** | 产品分析 | PostHog SDK | 用户行为分析 |
| **Stripe** | 支付 | Stripe SDK | 计费和订阅 |

### 7.3 SDK 集成

Langfuse 提供 SDK 供用户集成：

| SDK | 语言 | 仓库 |
|-----|------|------|
| **langfuse-python** | Python | github.com/langfuse/langfuse-python |
| **langfuse-js** | JavaScript/TypeScript | github.com/langfuse/langfuse-js |
| **langfuse (npm)** | Browser/Node.js | 与 langfuse-js 相同 |

**集成方式**：
```python
# Python SDK 示例
from langfuse import Langfuse
langfuse = Langfuse(
    public_key="pk-...",
    secret_key="sk-...",
    host="https://cloud.langfuse.com"
)
```

---

## 8. 架构优势与挑战

### 8.1 架构优势

#### ✅ 类型安全
- 全栈 TypeScript
- tRPC 端到端类型推导
- Prisma 类型化查询
- Zod 运行时验证

#### ✅ 高性能
- OLAP/OLTP 分离
- 异步任务队列
- 多级缓存策略
- ClickHouse 列式存储

#### ✅ 可扩展
- 独立的 Worker 服务
- 水平扩展能力
- 无状态设计
- 微服务架构

#### ✅ 开发效率
- Monorepo 代码共享
- Turborepo 并行构建
- 热重载开发
- 统一工具链

#### ✅ 可维护性
- 清晰的模块边界
- 功能模块化组织
- 完善的类型系统
- 统一的代码风格

### 8.2 架构挑战

#### ⚠️ 复杂度
- 多数据库管理复杂
- 分布式系统调试困难
- 需要理解多种技术栈

#### ⚠️ 部署要求
- 需要多个基础设施组件
- 资源需求较高
- 配置复杂

#### ⚠️ 数据一致性
- 跨数据库的数据一致性
- 异步任务的最终一致性
- 需要仔细设计补偿机制

---

## 9. 架构演进建议

### 9.1 短期优化
- 优化 ClickHouse 查询性能
- 增加缓存层（Redis）
- 改进异步任务监控

### 9.2 中期演进
- 考虑引入 GraphQL（可选）
- 微服务进一步拆分（如需要）
- 实现更细粒度的权限控制

### 9.3 长期规划
- 多租户架构优化
- 边缘计算支持
- 实时流处理能力

---

## 10. 相关文档

- **PlantUML 架构图**：[system-component-architecture.puml](./system-component-architecture.puml)
- **分层架构说明**：[../03-layers/](../03-layers/)
- **核心模块详解**：[../04-core-modules/](../04-core-modules/)
- **数据库 Schema**：[../../packages/shared/prisma/schema.prisma](../../packages/shared/prisma/schema.prisma)
- **数据库 ERD**：[../../packages/shared/prisma/database.svg](../../packages/shared/prisma/database.svg)

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0  

