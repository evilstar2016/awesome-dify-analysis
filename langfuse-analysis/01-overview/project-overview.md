# Langfuse 项目概览

## 1. 项目基本信息

### 1.1 项目简介
**项目名称**：Langfuse  
**版本**：3.140.0  
**开源协议**：MIT License  
**官方网站**：https://langfuse.com  
**GitHub 仓库**：https://github.com/langfuse/langfuse  
**项目类型**：开源 LLM 工程平台（Full-stack Web Application）

### 1.2 项目描述
Langfuse 是一个**开源 LLM 工程平台**，帮助团队协作**开发、监控、评估和调试** AI 应用程序。该平台提供以下核心能力：

- **LLM 可观测性**：追踪和调试 LLM 调用及应用逻辑
- **提示词管理**：集中管理、版本控制和协作迭代提示词
- **评估系统**：支持 LLM-as-a-judge、用户反馈、人工标注和自定义评估管道
- **数据集管理**：测试集和基准测试，支持持续改进和预部署测试
- **LLM Playground**：交互式测试和迭代工具
- **全面的 API**：OpenAPI 规范，Python/JS/TS SDK

### 1.3 技术栈概览

| 类别 | 技术 |
|-----|------|
| **语言** | TypeScript (Node.js 24) |
| **前端框架** | Next.js 14 (pages router), React 19 |
| **后端框架** | Next.js API Routes, tRPC |
| **认证** | NextAuth.js / Auth.js |
| **ORM** | Prisma |
| **数据验证** | Zod v3 |
| **UI 框架** | Tailwind CSS, shadcn/ui (Radix UI) |
| **构建工具** | Turborepo, pnpm workspace |
| **OLTP 数据库** | PostgreSQL |
| **OLAP 数据库** | ClickHouse |
| **缓存/队列** | Redis / Valkey |
| **对象存储** | S3 / Blob Storage |
| **后台任务** | BullMQ |
| **可观测性** | OpenTelemetry, Winston |
| **API 生成** | Fern (生成 OpenAPI 和 Pydantic 模型) |

---

## 2. 项目目录结构

### 2.1 整体结构（Monorepo）

```
langfuse/
├── web/                          # 主 Web 应用（Next.js）
│   ├── src/
│   │   ├── app/                 # Next.js App 结构
│   │   ├── pages/               # 页面路由和 API 路由
│   │   ├── components/          # React 组件
│   │   ├── server/              # 服务器端逻辑（tRPC 路由）
│   │   ├── features/            # 功能模块
│   │   ├── hooks/               # React Hooks
│   │   ├── utils/               # 工具函数
│   │   ├── constants/           # 常量定义
│   │   ├── ee/                  # 企业版功能
│   │   ├── workers/             # Worker 线程
│   │   ├── __tests__/           # 单元测试
│   │   └── __e2e__/             # E2E 测试
│   ├── public/                  # 静态资源
│   ├── types/                   # 类型定义
│   ├── package.json
│   ├── next.config.mjs
│   ├── tailwind.config.ts
│   └── tsconfig.json
│
├── worker/                       # 后台异步 Worker 服务
│   ├── src/
│   │   ├── queues/              # BullMQ 队列定义和处理器
│   │   ├── services/            # 业务逻辑服务
│   │   ├── features/            # 功能模块
│   │   ├── api/                 # Express API 端点
│   │   ├── backgroundMigrations/# 后台数据迁移
│   │   ├── utils/               # 工具函数
│   │   ├── errors/              # 错误处理
│   │   ├── interfaces/          # 接口定义
│   │   ├── ee/                  # 企业版功能
│   │   ├── scripts/             # 脚本工具
│   │   ├── __tests__/           # 单元测试
│   │   ├── app.ts               # Express 应用入口
│   │   ├── index.ts             # Worker 主入口
│   │   └── database.ts          # 数据库连接
│   ├── package.json
│   └── tsconfig.json
│
├── packages/                     # 共享包（Monorepo）
│   ├── shared/                  # 核心共享代码
│   │   ├── prisma/              # Prisma 数据库 Schema
│   │   │   ├── schema.prisma   # 数据库模型定义
│   │   │   ├── migrations/     # 数据库迁移文件
│   │   │   └── generated/      # Prisma 生成的客户端
│   │   ├── clickhouse/          # ClickHouse 相关
│   │   │   ├── migrations/     # ClickHouse 迁移
│   │   │   └── scripts/        # ClickHouse 脚本
│   │   ├── src/
│   │   │   ├── db.ts           # 数据库客户端
│   │   │   ├── types.ts        # 共享类型
│   │   │   ├── constants.ts    # 共享常量
│   │   │   ├── features/       # 共享功能模块
│   │   │   ├── domain/         # 领域模型
│   │   │   ├── encryption/     # 加密工具
│   │   │   └── errors/         # 错误定义
│   │   └── package.json
│   │
│   ├── config-eslint/           # ESLint 共享配置
│   └── config-typescript/       # TypeScript 共享配置
│
├── ee/                           # 企业版（Enterprise Edition）
│   ├── src/
│   │   ├── index.ts
│   │   └── ee-license-check/   # 许可证检查
│   └── package.json
│
├── fern/                         # API Schema 定义（Fern）
│   ├── fern.config.json
│   └── apis/
│       ├── client/              # 客户端 API
│       ├── server/              # 服务器 API
│       └── organizations/       # 组织 API
│
├── examples/                     # 使用示例
│
├── scripts/                      # 构建和开发脚本
│   └── nuke.sh                  # 数据库重置脚本
│
├── patches/                      # npm 包补丁
│   └── next-auth@4.24.12.patch
│
├── .github/                      # GitHub 配置
│   ├── workflows/               # CI/CD 工作流
│   ├── ISSUE_TEMPLATE/
│   └── PULL_REQUEST_TEMPLATE.md
│
├── docker-compose.yml            # Docker 部署配置
├── docker-compose.dev.yml        # 开发环境 Docker 配置
├── package.json                  # 根 package.json
├── pnpm-workspace.yaml           # pnpm workspace 配置
├── turbo.json                    # Turborepo 配置
├── tsconfig.json                 # TypeScript 根配置
├── .env.example                  # 环境变量模板
├── README.md                     # 项目说明文档
├── CONTRIBUTING.md               # 贡献指南
├── AGENTS.md                     # Codex AI 指南
└── LICENSE                       # MIT 许可证
```

### 2.2 核心目录说明

#### 2.2.1 web/ - 主 Web 应用
**作用**：面向用户的主要 Web 应用程序，基于 Next.js 14

**关键子目录**：
- `src/pages/`: Next.js 页面路由和 API 路由（pages router 模式）
- `src/server/`: 服务器端逻辑，包括 tRPC 路由定义
- `src/components/`: 可复用的 React 组件
- `src/features/`: 按功能组织的业务逻辑（如 traces、prompts、evaluations）
- `src/ee/`: 企业版功能模块

**技术特点**：
- Next.js 15.5.9 (pages router)
- tRPC 用于类型安全的 API
- NextAuth.js 用于身份认证
- Prisma ORM 用于数据库访问
- shadcn/ui + Tailwind CSS 用于 UI

#### 2.2.2 worker/ - 后台 Worker 服务
**作用**：处理异步后台任务，包括数据聚合、事件处理、后台迁移等

**关键子目录**：
- `src/queues/`: BullMQ 队列定义和任务处理器
- `src/services/`: 核心业务逻辑服务
- `src/backgroundMigrations/`: 后台数据迁移任务
- `src/scripts/`: 工具脚本（如数据回填）

**技术特点**：
- BullMQ 用于任务队列
- Redis 用于队列存储
- 独立的 Node.js 进程
- 支持并发任务处理

#### 2.2.3 packages/shared/ - 共享代码库
**作用**：Web 和 Worker 之间共享的核心代码

**关键内容**：
- **Prisma Schema**：统一的数据库模型定义
- **ClickHouse**：OLAP 数据库的迁移和脚本
- **Domain Models**：领域模型和业务逻辑
- **Utilities**：通用工具函数
- **Types**：共享的 TypeScript 类型定义
- **Encryption**：数据加密工具

#### 2.2.4 fern/ - API Schema
**作用**：使用 Fern 工具定义 API Schema，自动生成 OpenAPI 规范和 SDK

**包含**：
- 客户端 API 定义
- 服务器 API 定义
- 组织管理 API 定义
- 自动生成 Python SDK 和 JS/TS SDK

---

## 3. 主要功能模块

### 3.1 核心业务功能

| 功能模块 | 描述 | 主要实现位置 |
|---------|------|------------|
| **Traces（追踪）** | LLM 调用追踪和可观测性 | `web/src/features/traces/`, `web/src/pages/api/public/traces.ts` |
| **Observations（观测）** | 详细的执行观测数据 | `web/src/features/observations/`, ClickHouse 存储 |
| **Prompts（提示词管理）** | 版本控制的提示词管理 | `web/src/features/prompts/` |
| **Evaluations（评估）** | LLM 输出评估和打分 | `web/src/features/evals/`, `worker/src/features/evals/` |
| **Datasets（数据集）** | 测试集和基准测试管理 | `web/src/features/datasets/` |
| **Playground（调试工具）** | 交互式 LLM 测试 | `web/src/features/playground/` |
| **Projects（项目管理）** | 多项目和组织管理 | `web/src/features/projects/` |
| **API Keys（密钥管理）** | API 密钥生成和管理 | `web/src/server/auth/apiKeys` |
| **Billing（计费）** | 使用量跟踪和计费 | `web/src/features/billing/`, `worker/src/features/billing/` |
| **Dashboard（仪表盘）** | 数据可视化和分析 | `web/src/features/dashboard/` |
| **Public API** | RESTful API 接口 | `web/src/pages/api/public/` |

### 3.2 基础设施功能

| 功能模块 | 描述 | 主要实现位置 |
|---------|------|------------|
| **Authentication（认证）** | 用户登录和权限管理 | `web/src/server/auth/`, NextAuth.js |
| **Authorization（授权）** | 基于角色的访问控制（RBAC） | `web/src/features/rbac/` |
| **Ingestion（数据摄取）** | 高吞吐量的事件摄取 | `web/src/features/ingest/`, S3 + Worker |
| **Queue Processing（队列处理）** | 异步任务处理 | `worker/src/queues/` |
| **Data Export（数据导出）** | 导出追踪数据 | `worker/src/features/export/` |
| **Analytics（分析）** | ClickHouse 数据分析 | ClickHouse 查询 |
| **Monitoring（监控）** | OpenTelemetry 可观测性 | `web/src/instrumentation.ts`, `worker/src/instrumentation.ts` |

### 3.3 企业功能（EE）

| 功能模块 | 描述 | 实现位置 |
|---------|------|---------|
| **License Check（许可证检查）** | 企业版许可证验证 | `ee/src/ee-license-check/` |
| **SSO/SAML** | 单点登录支持 | `web/src/ee/` |
| **Advanced RBAC** | 高级权限控制 | `web/src/ee/` |
| **Self-Hosting Enhancements** | 企业部署增强 | `ee/` |

---

## 4. 核心依赖分析

### 4.1 主要框架依赖（Web）

| 依赖包 | 版本 | 用途 |
|-------|------|------|
| **next** | 15.5.9 | Next.js 框架（核心） |
| **react** | 19.2.3 | React 库 |
| **react-dom** | 19.2.3 | React DOM 渲染 |
| **@trpc/server** | ^11.4.4 | tRPC 服务器端 |
| **@trpc/client** | ^11.4.4 | tRPC 客户端 |
| **@trpc/react-query** | ^11.4.4 | tRPC + React Query 集成 |
| **next-auth** | ^4.24.12 | 身份认证 |
| **prisma** | ^6.17.1 | Prisma ORM |

### 4.2 UI/前端依赖

| 依赖包 | 用途 |
|-------|------|
| **@radix-ui/react-*** | Radix UI 组件库（无样式组件） |
| **tailwindcss** | Tailwind CSS 框架 |
| **lucide-react** | 图标库 |
| **recharts** | 图表库 |
| **cmdk** | 命令面板组件 |
| **react-hook-form** | 表单管理 |
| **@tanstack/react-table** | 表格组件 |
| **react-markdown** | Markdown 渲染 |
| **@uiw/react-codemirror** | 代码编辑器 |

### 4.3 数据库和存储

| 依赖包 | 用途 |
|-------|------|
| **prisma** | PostgreSQL ORM |
| **kysely** | ClickHouse 查询构建器 |
| **ioredis** | Redis 客户端 |
| **@aws-sdk/client-s3** | S3 对象存储客户端 |
| **pg** | PostgreSQL 驱动（Worker） |

### 4.4 后台任务和队列（Worker）

| 依赖包 | 用途 |
|-------|------|
| **bullmq** | 任务队列系统 |
| **ioredis** | Redis 客户端 |
| **express** | HTTP 服务器（Worker API） |
| **backoff** | 重试机制 |
| **exponential-backoff** | 指数退避策略 |

### 4.5 LLM 和 AI 相关

| 依赖包 | 用途 |
|-------|------|
| **langchain** | LangChain 集成 |
| **@langchain/core** | LangChain 核心库 |
| **ai** | Vercel AI SDK |
| **@anthropic-ai/tokenizer** | Anthropic Token 计数 |
| **tiktoken** | OpenAI Token 计数 |
| **langfuse** | Langfuse SDK（自引用） |

### 4.6 监控和可观测性

| 依赖包 | 用途 |
|-------|------|
| **@opentelemetry/sdk-node** | OpenTelemetry SDK |
| **@opentelemetry/instrumentation-*** | 各种自动插桩 |
| **@sentry/nextjs** | Sentry 错误监控 |
| **dd-trace** | Datadog APM |
| **posthog-js** | PostHog 产品分析 |

### 4.7 验证和类型安全

| 依赖包 | 用途 |
|-------|------|
| **zod** | 运行时类型验证 |
| **typescript** | TypeScript 编译器 |
| **@t3-oss/env-nextjs** | 环境变量验证 |

### 4.8 工具和辅助

| 依赖包 | 用途 |
|-------|------|
| **lodash** | 工具函数库 |
| **date-fns** | 日期处理 |
| **decimal.js** | 精确小数计算 |
| **uuid** | UUID 生成 |
| **nanoid** | 短 ID 生成 |
| **csv-parse** | CSV 解析 |

---

## 5. 构建与部署

### 5.1 开发命令

#### 完整开发环境启动
```bash
# 完整开发工作流（推荐）
pnpm run dx
# 包括：安装依赖 → 重置基础设施 → 启动 Docker → 迁移数据库 → 填充示例数据 → 启动开发服务器

# 强制重置版本
pnpm run dx-f

# 跳过基础设施（如 Docker 已运行）
pnpm run dx:skip-infra
```

#### 独立开发服务器
```bash
# 启动所有服务（Web + Worker）
pnpm run dev

# 仅启动 Web 应用
pnpm run dev:web

# 仅启动 Worker
pnpm run dev:worker

# Web 应用（不使用 Turbopack）
pnpm run dev:web-no-turbo
```

### 5.2 基础设施管理

#### Docker 基础设施
```bash
# 启动基础设施（PostgreSQL, Redis, ClickHouse）
pnpm run infra:dev:up

# 停止基础设施
pnpm run infra:dev:down

# 清除基础设施（删除卷）
pnpm run infra:dev:prune
```

#### 数据库管理
```bash
# 生成 Prisma 客户端
pnpm run db:generate

# 运行数据库迁移
pnpm run db:migrate

# 填充数据库
pnpm run db:seed

# 填充示例数据
pnpm run db:seed:examples

# 完全重置（危险！删除所有数据）
pnpm run nuke
```

#### ClickHouse 管理
```bash
# 启动 ClickHouse
pnpm --filter=shared run ch:up

# 停止 ClickHouse
pnpm --filter=shared run ch:down

# 重置 ClickHouse
pnpm --filter=shared run ch:reset
```

### 5.3 构建与生产

```bash
# 构建所有包
pnpm run build

# 启动生产服务器
pnpm run start
```

### 5.4 代码质量

```bash
# 代码检查
pnpm run lint

# 代码格式化
pnpm run format

# 检查格式
pnpm run format:check
```

### 5.5 测试

```bash
# 运行所有测试
pnpm run test

# 运行 Web 测试
pnpm --filter=web run test

# 运行 Worker 测试
pnpm --filter=worker run test

# E2E 测试
pnpm --filter=web run test:e2e
```

### 5.6 部署方式

#### Docker Compose（推荐自托管）
```bash
# 使用 Docker Compose 部署
docker-compose up -d
```

**配置文件**：
- `docker-compose.yml` - 生产部署
- `docker-compose.dev.yml` - 开发环境
- `docker-compose.dev-azure.yml` - Azure 开发环境
- `docker-compose.dev-redis-cluster.yml` - Redis 集群模式

#### Kubernetes
支持 Helm Charts 部署（详见官方文档）

#### Langfuse Cloud
官方托管的 SaaS 服务：https://cloud.langfuse.com

---

## 6. 环境配置

### 6.1 必需的环境变量

创建 `.env` 文件（参考 `.env.example`）：

```env
# 数据库连接
DATABASE_URL=postgresql://user:password@localhost:5432/langfuse
CLICKHOUSE_URL=http://localhost:8123
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=

# Redis
REDIS_CONNECTION_STRING=redis://localhost:6379

# NextAuth
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=<生成随机字符串>

# S3 存储（可选）
S3_ENDPOINT=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
S3_BUCKET_NAME=

# 应用配置
NODE_ENV=development
LANGFUSE_DEFAULT_PROJECT_ROLE=OWNER

# 可选：LLM API Keys（用于 Playground 和 Evals）
OPENAI_API_KEY=
ANTHROPIC_API_KEY=

# 可选：监控
SENTRY_DSN=
POSTHOG_API_KEY=
```

### 6.2 测试环境配置

创建 `.env.test` 文件用于测试：
```env
DATABASE_URL=postgresql://user:password@localhost:5432/langfuse_test
# ... 其他测试配置
```

### 6.3 关键配置文件

| 文件 | 作用 |
|-----|------|
| `.env` | 开发环境变量 |
| `.env.test` | 测试环境变量 |
| `next.config.mjs` | Next.js 配置 |
| `turbo.json` | Turborepo 任务配置 |
| `pnpm-workspace.yaml` | pnpm workspace 配置 |
| `packages/shared/prisma/schema.prisma` | 数据库 Schema |
| `tsconfig.json` | TypeScript 配置 |
| `tailwind.config.ts` | Tailwind CSS 配置 |

---

## 7. 开发工作流

### 7.1 首次设置

```bash
# 1. 克隆仓库
git clone https://github.com/langfuse/langfuse.git
cd langfuse

# 2. 安装依赖（要求 Node.js 24 + pnpm）
pnpm install

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env 文件

# 4. 启动完整开发环境
pnpm run dx
```

### 7.2 日常开发

```bash
# 启动开发服务器（假设基础设施已运行）
pnpm run dev

# Web 应用运行在：http://localhost:3000
# Worker 在后台运行
```

### 7.3 数据库修改工作流

```bash
# 1. 修改 packages/shared/prisma/schema.prisma

# 2. 生成迁移
pnpm --filter=shared run db:migrate

# 3. 生成 Prisma 客户端
pnpm run db:generate

# 4. 重启开发服务器
```

### 7.4 代码提交工作流

```bash
# 1. 格式化代码
pnpm run format

# 2. 检查 Lint
pnpm run lint

# 3. 运行测试
pnpm run test

# 4. 提交（使用 Conventional Commits 格式）
git commit -m "feat: add new feature"
```

---

## 8. 架构特点总结

### 8.1 Monorepo 架构优势
- **代码共享**：Web 和 Worker 共享核心代码
- **统一依赖管理**：pnpm workspace + Turborepo
- **类型安全**：TypeScript 跨包类型检查
- **统一构建**：Turborepo 并行构建和缓存

### 8.2 多数据库架构
- **PostgreSQL（OLTP）**：事务性数据，关系型查询
- **ClickHouse（OLAP）**：高吞吐量写入，分析查询
- **Redis**：缓存和任务队列
- **S3**：对象存储（原始事件、附件）

### 8.3 异步处理架构
- **BullMQ** 队列系统
- **独立 Worker 进程**
- **可扩展**：支持多 Worker 实例
- **可靠**：任务重试和错误处理

### 8.4 类型安全栈
- **TypeScript**：全栈类型安全
- **tRPC**：端到端类型安全 API
- **Prisma**：类型安全的数据库访问
- **Zod**：运行时类型验证

### 8.5 可观测性
- **OpenTelemetry**：分布式追踪
- **Sentry**：错误监控
- **Datadog**：APM（可选）
- **PostHog**：产品分析

---

## 9. 项目规模和成熟度

- **代码库大小**：大型项目（数万行 TypeScript 代码）
- **活跃开发**：持续更新（版本 3.140.0）
- **社区支持**：活跃的 GitHub 和 Discord 社区
- **生产就绪**：Battle-tested，用于生产环境
- **开源成熟度**：MIT 许可证，完善的贡献指南

---

## 10. 相关资源

- **官方文档**：https://langfuse.com/docs
- **GitHub 仓库**：https://github.com/langfuse/langfuse
- **Discord 社区**：https://langfuse.com/discord
- **在线 Demo**：https://langfuse.com/demo
- **部署指南**：https://langfuse.com/docs/deployment/self-host
- **贡献指南**：[CONTRIBUTING.md](../../CONTRIBUTING.md)
- **API 文档**：https://api.reference.langfuse.com/

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0  
