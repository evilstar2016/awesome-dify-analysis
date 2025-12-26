# Langfuse 学习地图 🗺️

> **目标受众**：AI架构进阶人员  
> **学习方式**：系统级视角，深入浅出  
> **基于版本**：Langfuse 3.140.0  
> **最后更新**：2025-12-26

---

## 📚 如何使用本学习地图

### 学习路径说明
- 🟢 **入门级**：快速建立整体认知（1-3天）
- 🟡 **进阶级**：深入理解架构设计（1-2周）
- 🔴 **专家级**：掌握核心实现细节（2-4周）
- ⭐ **推荐优先级**：建议优先学习的内容

### 阅读顺序建议
```
第一阶段（全局认知）→ 第二阶段（架构理解）→ 第三阶段（分层深入）→ 第四阶段（模块实战）→ 第五阶段（数据与集成）
```

---

## 🎯 第一阶段：全局认知（1-3天）

> **目标**：建立对 Langfuse 项目的整体认知，理解其定位、核心价值和技术选型

### 1.1 项目概览 🟢 ⭐⭐⭐⭐⭐
**文档**：[01-overview/project-overview.md](./01-overview/project-overview.md)

**核心问题**：
- ✅ Langfuse 是什么？解决什么问题？
- ✅ 为什么要做 LLM 工程平台？与传统 APM 的区别？
- ✅ 核心功能有哪些？（可观测性、提示词管理、评估系统、数据集）
- ✅ 技术栈选型的考量是什么？

**关键收获**：
- 理解 Langfuse 在 LLM 工程生态中的定位
- 掌握项目的核心能力边界
- 了解 Monorepo 架构和技术栈全貌

**学习输出**：
- [ ] 能用一句话描述 Langfuse 的核心价值
- [ ] 画出 Langfuse 的功能地图
- [ ] 理解为什么选择 Next.js + tRPC + Prisma 技术栈

**预计时间**：2-4小时

---

## 🏗️ 第二阶段：架构理解（3-5天）

> **目标**：从系统架构视角理解 Langfuse 的设计思想和工程实践

### 2.1 系统架构 🟡 ⭐⭐⭐⭐⭐
**文档**：[02-architecture/system-architecture.md](./02-architecture/system-architecture.md)

**核心问题**：
- ✅ Langfuse 采用什么架构模式？为什么这样设计？
- ✅ Web 服务和 Worker 服务如何分工？
- ✅ 为什么需要 PostgreSQL + ClickHouse 双数据库？
- ✅ 系统如何保证可扩展性和高性能？

**关键收获**：
- 理解混合架构模式（Monorepo + 微服务 + 事件驱动）
- 掌握 Web 和 Worker 的职责边界
- 理解 OLTP + OLAP 的数据架构设计
- 学习如何用 BullMQ 实现异步任务处理

**架构图解读**：
```
用户/SDK
   ↓
Web服务（Next.js）─────→ Worker服务（Node.js）
   ↓                        ↓
PostgreSQL              BullMQ队列
   ↓                        ↓
ClickHouse ←───────── 数据同步任务
```

**学习输出**：
- [ ] 画出 Langfuse 的系统架构图
- [ ] 理解为什么 trace 数据要写入两个数据库
- [ ] 掌握异步任务的设计模式

**预计时间**：1-2天

---

### 2.2 部署架构 🟡 ⭐⭐⭐
**文档**：[02-architecture/system-architecture.md](./02-architecture/system-architecture.md#4-部署架构)

**核心问题**：
- ✅ 如何进行本地开发？
- ✅ 生产环境如何部署？（Docker、Kubernetes、云服务）
- ✅ 如何保证服务的高可用？

**关键收获**：
- 理解单机、分布式、云托管三种部署模式
- 掌握 Docker Compose 本地开发环境配置
- 学习如何用 Kubernetes 实现水平扩展

**学习输出**：
- [ ] 搭建本地开发环境
- [ ] 理解生产部署的最佳实践

**预计时间**：1天

---

## 🏛️ 第三阶段：分层架构深入（1-2周）

> **目标**：理解 Langfuse 的分层架构设计，掌握各层的职责和交互方式

### 3.1 分层架构总览 🟡 ⭐⭐⭐⭐⭐
**文档**：[03-layers/README.md](./03-layers/README.md)

**核心问题**：
- ✅ Langfuse 如何进行代码分层？
- ✅ 每一层的职责是什么？
- ✅ 层与层之间如何通信？

**关键收获**：
- 理解 6 层架构设计
- 掌握清晰的职责边界
- 学习如何用分层架构降低耦合

**架构图**：
```
前端展示层（React + Next.js）
      ↓ tRPC
tRPC API层（60+ routers）
      ↓
业务服务层（Feature Modules）
      ↓
数据访问层（Prisma + Kysely）
      ↓
基础设施层（PostgreSQL + ClickHouse + Redis + S3）

横向：共享包层（types, utils, domain models）
异步：Worker 服务层（BullMQ queues）
```

**学习输出**：
- [ ] 画出完整的分层架构图
- [ ] 理解每一层的输入输出
- [ ] 掌握 tRPC 端到端类型安全的实现原理

**预计时间**：1天

---

### 3.2 前端展示层 🟢 ⭐⭐⭐⭐
**文档**：[03-layers/01-frontend-presentation-layer.md](./03-layers/01-frontend-presentation-layer.md)

**核心问题**：
- ✅ 如何用 Next.js Pages Router 组织路由？
- ✅ 如何用 shadcn/ui 构建一致的 UI？
- ✅ React Query 如何管理服务端状态？
- ✅ 如何实现 SSR 和 SSG？

**关键收获**：
- 理解 Next.js 的文件路由系统
- 掌握 React Query 的缓存策略
- 学习 shadcn/ui 的组件设计模式
- 理解 Server Components vs Client Components

**关键目录**：
```
web/src/
├── pages/              # 页面路由
│   ├── project/        # 项目相关页面
│   ├── traces/         # 追踪页面
│   └── settings/       # 设置页面
├── components/         # React 组件
│   ├── ui/             # shadcn/ui 基础组件
│   └── layouts/        # 布局组件
└── hooks/              # 自定义 Hooks
```

**学习输出**：
- [ ] 理解 Next.js 的渲染策略
- [ ] 掌握 React Query 的最佳实践
- [ ] 能够基于 shadcn/ui 开发新组件

**预计时间**：2-3天

---

### 3.3 tRPC API 层 🟡 ⭐⭐⭐⭐⭐
**文档**：[03-layers/02-trpc-api-layer.md](./03-layers/02-trpc-api-layer.md)

**核心问题**：
- ✅ 什么是 tRPC？为什么选择 tRPC？
- ✅ 如何定义 tRPC Router？
- ✅ 如何实现端到端类型安全？
- ✅ 如何处理认证和权限？

**关键收获**：
- 理解 tRPC 的核心理念和优势
- 掌握 Zod Schema 验证
- 学习 Context、Middleware 的设计模式
- 理解 RBAC 权限控制的实现

**核心概念**：
```typescript
// tRPC Procedure 结构
export const traceRouter = createTRPCRouter({
  byId: protectedProjectProcedure
    .input(z.object({ traceId: z.string() }))
    .query(async ({ input, ctx }) => {
      // 业务逻辑
    }),
});
```

**学习输出**：
- [ ] 理解 tRPC 的工作原理
- [ ] 能够创建新的 tRPC Router
- [ ] 掌握 Middleware 的使用场景

**预计时间**：2-3天

---

### 3.4 业务服务层 🟡 ⭐⭐⭐⭐
**文档**：[03-layers/03-business-service-layer.md](./03-layers/03-business-service-layer.md)

**核心问题**：
- ✅ 业务逻辑如何组织？
- ✅ 如何实现复杂的业务规则？
- ✅ 如何处理事务和并发？

**关键收获**：
- 理解 Feature Modules 的组织方式
- 掌握领域驱动设计（DDD）的思想
- 学习如何封装复杂的业务逻辑

**核心模块**：
- Traces 管理
- Prompts 版本控制
- Datasets 管理
- Scores 计算
- Evaluation 执行

**学习输出**：
- [ ] 理解如何拆分业务模块
- [ ] 掌握事务处理的最佳实践
- [ ] 能够实现新的业务功能

**预计时间**：2-3天

---

### 3.5 数据访问层 🟡 ⭐⭐⭐⭐
**文档**：[03-layers/04-data-access-layer.md](./03-layers/04-data-access-layer.md)

**核心问题**：
- ✅ 如何用 Prisma 访问 PostgreSQL？
- ✅ 如何用 Kysely 查询 ClickHouse？
- ✅ 如何优化数据库查询性能？
- ✅ 如何实现数据同步？

**关键收获**：
- 掌握 Prisma ORM 的高级用法
- 理解 ClickHouse 的查询优化技巧
- 学习数据库连接池管理
- 理解 PostgreSQL → ClickHouse 的同步机制

**学习输出**：
- [ ] 能够编写高效的 Prisma 查询
- [ ] 理解 ClickHouse 的物化视图
- [ ] 掌握数据同步的设计模式

**预计时间**：2-3天

---

### 3.6 共享包层 🟢 ⭐⭐⭐
**文档**：[03-layers/05-shared-packages-layer.md](./03-layers/05-shared-packages-layer.md)

**核心问题**：
- ✅ 如何在 Monorepo 中共享代码？
- ✅ 哪些代码应该放在共享包？
- ✅ 如何保证类型安全？

**关键收获**：
- 理解 pnpm workspace 的工作原理
- 掌握共享包的设计原则
- 学习如何管理依赖关系

**核心包**：
```
packages/shared/
├── prisma/           # 数据库 Schema
├── clickhouse/       # ClickHouse 配置
└── src/
    ├── features/     # 共享功能模块
    ├── server/       # 服务端工具
    └── utils/        # 通用工具函数
```

**学习输出**：
- [ ] 理解 Monorepo 的优势
- [ ] 能够创建新的共享包

**预计时间**：1天

---

### 3.7 Worker 服务层 🟡 ⭐⭐⭐⭐⭐
**文档**：[03-layers/06-worker-service-layer.md](./03-layers/06-worker-service-layer.md)

**核心问题**：
- ✅ Worker 服务的职责是什么？
- ✅ 如何用 BullMQ 处理异步任务？
- ✅ 如何保证任务的可靠性？
- ✅ 如何实现数据同步？

**关键收获**：
- 理解异步任务的设计模式
- 掌握 BullMQ 的核心概念（Queue、Worker、Job）
- 学习如何实现失败重试和死信队列
- 理解 PostgreSQL → ClickHouse 的实时同步

**核心队列**：
```
ingestion-queue          # 数据摄入
evaluation-queue         # 评估执行
trace-delete-queue       # 数据删除
legacy-ingestion-queue   # 兼容性队列
batch-export-queue       # 批量导出
postgres-clickhouse-sync # 数据同步
```

**学习输出**：
- [ ] 理解 BullMQ 的工作原理
- [ ] 掌握如何设计可靠的异步任务
- [ ] 能够创建新的 Worker Queue

**预计时间**：2-3天

---

## 🔧 第四阶段：核心模块实战（2-4周）

> **目标**：深入核心业务模块，理解功能实现和最佳实践

### 4.1 Traces & Observations 模块 🔴 ⭐⭐⭐⭐⭐
**文档**：[04-core-modules/traces/](./04-core-modules/traces/)

**核心问题**：
- ✅ 什么是 Trace 和 Observation？
- ✅ 如何设计追踪数据模型？
- ✅ 如何高效存储和查询海量追踪数据？
- ✅ 如何计算 LLM 调用成本？

**关键收获**：
- 理解分布式追踪的核心概念
- 掌握 Trace 的数据结构和关系
- 学习如何用 ClickHouse 存储时序数据
- 理解成本计算和聚合统计

**核心流程**：
1. SDK 发送 Trace 事件 → API 接收
2. 数据验证和转换 → 写入 PostgreSQL
3. 异步任务 → 同步到 ClickHouse
4. 前端查询 → 从 ClickHouse 读取
5. 成本计算 → 聚合统计

**学习输出**：
- [ ] 理解 Trace 的完整生命周期
- [ ] 掌握 Observation 的层级关系
- [ ] 能够实现新的统计维度

**预计时间**：3-5天

---

### 4.2 Prompts 模块 🔴 ⭐⭐⭐⭐⭐
**文档**：[04-core-modules/prompts/](./04-core-modules/prompts/)

**核心问题**：
- ✅ 如何实现提示词版本管理？
- ✅ 如何渲染动态模板（Mustache/Jinja2）？
- ✅ 如何支持多模型配置？
- ✅ 如何实现提示词的 A/B 测试？

**关键收获**：
- 理解提示词管理的核心价值
- 掌握版本控制和发布策略
- 学习模板引擎的实现原理
- 理解 Production Label 的作用

**核心概念**：
```
Prompt (提示词)
  ↓ 1:N
PromptVersion (版本)
  ↓ labels
Production Label (生产标签)
```

**学习输出**：
- [ ] 能够实现提示词的 CRUD
- [ ] 理解版本发布流程
- [ ] 掌握模板变量的注入机制

**预计时间**：2-3天

---

### 4.3 Datasets & Evaluation 模块 🔴 ⭐⭐⭐⭐⭐
**文档**：[04-core-modules/datasets/](./04-core-modules/datasets/)

**核心问题**：
- ✅ 如何管理测试数据集？
- ✅ 如何设计评估流程？
- ✅ 如何配置 Evaluator（LLM-as-a-judge）？
- ✅ 如何可视化评估结果？

**关键收获**：
- 理解 LLM 评估的核心方法论
- 掌握数据集的组织方式
- 学习如何用 LLM 评估 LLM
- 理解评估结果的聚合和对比

**核心流程**：
```
创建 Dataset → 添加 DatasetItem → 运行 Evaluation → 生成 Score → 可视化结果
```

**学习输出**：
- [ ] 能够设计评估场景
- [ ] 掌握 Evaluator 的配置方法
- [ ] 理解评估结果的统计分析

**预计时间**：3-4天

---

### 4.4 Scores 模块 🟡 ⭐⭐⭐⭐
**文档**：[04-core-modules/scores/](./04-core-modules/scores/)

**核心问题**：
- ✅ 如何设计评分系统？
- ✅ 如何聚合和统计评分？
- ✅ 如何支持多种评分类型？

**关键收获**：
- 理解评分的数据模型
- 掌握评分的来源和类型
- 学习如何计算平均分和分布

**评分来源**：
- 用户反馈（Thumbs up/down）
- 人工标注（Manual annotation）
- LLM 评估（LLM-as-a-judge）
- 自定义评估器（Custom evaluator）

**学习输出**：
- [ ] 理解评分的完整生命周期
- [ ] 掌握评分的聚合计算
- [ ] 能够实现自定义评分类型

**预计时间**：2天

---

### 4.5 Playground 模块 🟡 ⭐⭐⭐⭐
**文档**：[04-core-modules/playground/](./04-core-modules/playground/)

**核心问题**：
- ✅ 如何实现多模型对比？
- ✅ 如何调用不同的 LLM API？
- ✅ 如何实时展示流式响应？
- ✅ 如何保存和分享测试结果？

**关键收获**：
- 理解 Playground 的交互设计
- 掌握 LLM API 的统一封装
- 学习流式响应的实现
- 理解如何与 Prompts 集成

**学习输出**：
- [ ] 能够集成新的 LLM 模型
- [ ] 理解流式传输的实现
- [ ] 掌握提示词测试的最佳实践

**预计时间**：2-3天

---

### 4.6 Dashboard & Analytics 模块 🟡 ⭐⭐⭐⭐
**文档**：[04-core-modules/dashboard/](./04-core-modules/dashboard/)

**核心问题**：
- ✅ 如何设计分析仪表板？
- ✅ 如何高效查询聚合数据？
- ✅ 如何可视化统计图表？

**关键收获**：
- 理解 OLAP 查询优化
- 掌握 ClickHouse 的聚合函数
- 学习如何用 Recharts 可视化数据

**核心指标**：
- Trace 数量趋势
- Token 使用量
- 成本统计
- 延迟分布
- 错误率

**学习输出**：
- [ ] 能够设计新的统计指标
- [ ] 掌握 ClickHouse 的查询优化
- [ ] 理解图表的交互设计

**预计时间**：2天

---

### 4.7 Ingestion 模块 🟡 ⭐⭐⭐⭐
**文档**：[04-core-modules/ingestion/](./04-core-modules/ingestion/)

**核心问题**：
- ✅ SDK 如何发送数据到服务端？
- ✅ 如何验证和转换数据？
- ✅ 如何保证数据一致性？
- ✅ 如何处理大批量数据？

**关键收获**：
- 理解数据摄入的完整流程
- 掌握 Zod Schema 验证
- 学习如何设计幂等性 API
- 理解批量写入的优化策略

**学习输出**：
- [ ] 理解 SDK 的工作原理
- [ ] 掌握数据验证的最佳实践
- [ ] 能够优化摄入性能

**预计时间**：2-3天

---

### 4.8 Authentication & RBAC 模块 🟢 ⭐⭐⭐
**文档**：[04-core-modules/authentication-rbac/](./04-core-modules/authentication-rbac/)

**核心问题**：
- ✅ 如何实现用户认证？（Email、SSO、OAuth）
- ✅ 如何设计权限模型？
- ✅ 如何管理 API Key？
- ✅ 如何实现多租户隔离？

**关键收获**：
- 理解 NextAuth.js 的认证流程
- 掌握 RBAC 权限模型
- 学习 API Key 的生成和验证
- 理解组织和项目的隔离机制

**权限模型**：
```
Organization (组织)
  ↓ 1:N
Project (项目)
  ↓ 1:N
Resources (资源)

Roles: OWNER, ADMIN, MEMBER, VIEWER
```

**学习输出**：
- [ ] 理解认证和授权的区别
- [ ] 掌握 RBAC 的实现原理
- [ ] 能够实现新的权限规则

**预计时间**：2天

---

## 📊 第五阶段：数据与集成（1-2周）

> **目标**：理解数据架构设计和第三方集成策略

### 5.1 数据架构 🔴 ⭐⭐⭐⭐⭐
**文档**：[05-data-architecture/README.md](./05-data-architecture/README.md)

**核心问题**：
- ✅ 为什么需要 PostgreSQL + ClickHouse 双数据库？
- ✅ 如何设计数据模型？（ER 图）
- ✅ 如何实现数据同步？
- ✅ 如何优化查询性能？

**关键收获**：
- 理解 OLTP vs OLAP 的区别
- 掌握数据库的选型原则
- 学习数据建模的最佳实践
- 理解冷热数据分离策略

**数据流**：
```
SDK/API
  ↓ Write
PostgreSQL (元数据 + 最近数据)
  ↓ Async Sync
ClickHouse (历史数据 + 聚合查询)
  ↓ Read
Dashboard/Analytics
```

**学习输出**：
- [ ] 理解双数据库架构的优势
- [ ] 掌握数据同步的实现
- [ ] 能够设计新的数据表

**预计时间**：3-4天

---

### 5.2 PostgreSQL 数据模型 🟡 ⭐⭐⭐⭐
**文档**：[05-data-architecture/README.md](./05-data-architecture/README.md#1-postgresql-数据库设计)

**核心问题**：
- ✅ 有哪些核心数据表？
- ✅ 表之间的关联关系是什么？
- ✅ 如何用 Prisma Schema 定义模型？
- ✅ 如何处理数据库迁移？

**关键收获**：
- 掌握 50+ 张表的结构和关系
- 理解 Prisma Schema 的设计
- 学习数据库索引优化
- 理解外键约束和级联删除

**核心表**：
- User, Organization, Project（组织结构）
- Trace, Observation, Score（追踪数据）
- Prompt, Dataset, Evaluation（业务数据）
- ApiKey, Session（认证数据）

**学习输出**：
- [ ] 能够绘制完整的 ER 图
- [ ] 理解数据表的设计原则
- [ ] 掌握 Prisma Migration

**预计时间**：2-3天

---

### 5.3 ClickHouse 数据模型 🟡 ⭐⭐⭐⭐
**文档**：[05-data-architecture/README.md](./05-data-architecture/README.md#2-clickhouse-数据库设计)

**核心问题**：
- ✅ ClickHouse 存储哪些数据？
- ✅ 如何设计 ClickHouse 的表结构？
- ✅ 如何优化 ClickHouse 查询性能？
- ✅ 如何实现实时数据同步？

**关键收获**：
- 理解列式存储的优势
- 掌握 ClickHouse 的表引擎
- 学习分区和索引策略
- 理解物化视图的应用

**核心表**：
- observations（观测数据）
- traces（追踪数据）
- scores（评分数据）

**学习输出**：
- [ ] 理解 ClickHouse 的数据模型
- [ ] 掌握查询优化技巧
- [ ] 能够设计新的聚合查询

**预计时间**：2-3天

---

### 5.4 Redis 缓存与队列 🟢 ⭐⭐⭐
**文档**：[05-data-architecture/README.md](./05-data-architecture/README.md#3-redis-缓存设计)

**核心问题**：
- ✅ Redis 用于哪些场景？
- ✅ 如何设计缓存策略？
- ✅ 如何用 BullMQ 管理任务队列？

**关键收获**：
- 理解缓存的使用场景
- 掌握缓存失效策略
- 学习 BullMQ 的任务管理

**缓存场景**：
- API Key 缓存
- 用户会话
- 计算结果缓存

**学习输出**：
- [ ] 理解缓存的最佳实践
- [ ] 掌握 BullMQ 的使用

**预计时间**：1天

---

### 5.5 S3 对象存储 🟢 ⭐⭐⭐
**文档**：[05-data-architecture/README.md](./05-data-architecture/README.md#4-s3-对象存储设计)

**核心问题**：
- ✅ S3 存储哪些数据？
- ✅ 如何实现安全的文件上传？
- ✅ 如何支持多种对象存储服务？

**关键收获**：
- 理解对象存储的应用场景
- 掌握预签名 URL 的生成
- 学习如何兼容不同的云服务

**存储内容**：
- 媒体文件（图片、音频、视频）
- 批量导出数据
- 备份文件

**学习输出**：
- [ ] 理解对象存储的优势
- [ ] 掌握文件上传的安全实践

**预计时间**：1天

---

### 5.6 第三方集成 🟢 ⭐⭐⭐
**文档**：[06-ThirdParty/](./06-ThirdParty/)

**核心问题**：
- ✅ 集成了哪些第三方服务？
- ✅ 如何设计集成架构？
- ✅ 如何保证集成的稳定性？

**关键收获**：
- 理解第三方集成的设计模式
- 掌握 API 封装和错误处理
- 学习如何适配多种服务商

**集成类别**：
1. **LLM 集成**：OpenAI, Anthropic, Azure OpenAI, 自定义模型
2. **数据库与缓存**：PostgreSQL, ClickHouse, Redis
3. **对象存储**：AWS S3, MinIO, Cloudflare R2
4. **消息通知**：Email (SMTP, SendGrid)
5. **认证授权**：NextAuth.js, OAuth, SAML
6. **可观测性**：Sentry, PostHog, OpenTelemetry
7. **支付**：Stripe
8. **AI 增强**：LangChain, LlamaIndex

**学习输出**：
- [ ] 理解集成的架构设计
- [ ] 掌握 API 封装的最佳实践
- [ ] 能够集成新的第三方服务

**预计时间**：2-3天

---

## 🎓 学习路径总结

### 快速路径（1周，入门级） 🟢
适合快速了解 Langfuse 的核心价值和架构设计

```
Day 1: 项目概览 → 系统架构
Day 2-3: 分层架构总览 → 前端展示层
Day 4-5: tRPC API 层 → 业务服务层
Day 6-7: 核心模块速览（Traces + Prompts）
```

**学习成果**：
- ✅ 理解 Langfuse 的定位和核心功能
- ✅ 掌握整体架构和技术栈
- ✅ 能够读懂核心代码
- ✅ 可以进行简单的二次开发

---

### 标准路径（3-4周，进阶级） 🟡
适合深入理解 Langfuse 的设计思想和工程实践

```
Week 1: 全局认知 + 架构理解
  - 项目概览 + 系统架构 + 部署架构

Week 2: 分层架构深入
  - 前端展示层 + tRPC API 层 + 业务服务层
  - 数据访问层 + 共享包层 + Worker 服务层

Week 3: 核心模块实战（上）
  - Traces & Observations
  - Prompts
  - Datasets & Evaluation

Week 4: 核心模块实战（下）
  - Scores + Playground + Dashboard
  - Ingestion + Authentication & RBAC
```

**学习成果**：
- ✅ 掌握完整的架构设计和分层思想
- ✅ 理解核心业务模块的实现细节
- ✅ 能够进行复杂的功能开发
- ✅ 可以基于 Langfuse 进行架构创新

---

### 专家路径（6-8周，专家级） 🔴
适合全面掌握 Langfuse 的源码和最佳实践

```
Week 1-2: 全局认知 + 架构理解 + 分层架构深入

Week 3-4: 核心模块实战（全部模块）

Week 5-6: 数据与集成
  - 数据架构 + PostgreSQL + ClickHouse
  - Redis + S3 + 第三方集成

Week 7-8: 源码深入 + 性能优化
  - 阅读核心模块源码
  - 性能优化和扩展性设计
  - 生产环境最佳实践
```

**学习成果**：
- ✅ 完全理解 Langfuse 的设计思想和实现细节
- ✅ 能够进行架构级优化和扩展
- ✅ 可以基于 Langfuse 打造企业级平台
- ✅ 能够为社区贡献代码

---

## 🔍 学习方法建议

### 1. 理论与实践结合
- 📖 **先读文档**：理解设计思想和架构原理
- 💻 **再看代码**：验证理解，发现细节
- 🛠️ **动手实践**：搭建环境，运行项目，修改代码
- 📝 **输出总结**：写博客、画架构图、做分享

### 2. 由浅入深
- 🔍 **第一遍**：快速浏览，建立全局认知
- 🧐 **第二遍**：重点阅读，理解核心设计
- 🔬 **第三遍**：深入源码，掌握实现细节

### 3. 问题驱动学习
- ❓ **带着问题读**：为什么这样设计？如何实现的？有什么优势？
- 🎯 **关注核心问题**：可扩展性、高性能、类型安全、可观测性
- 💡 **思考改进方向**：如果是我来设计，会怎么做？

### 4. 对比学习
- 🔁 **横向对比**：Langfuse vs LangSmith vs Weights & Biases
- 📊 **纵向对比**：不同版本的演进，技术选型的变化
- 🌐 **技术栈对比**：tRPC vs GraphQL, Prisma vs Drizzle

### 5. 社区参与
- 💬 **GitHub Discussions**：提问、讨论、分享
- 🐛 **Issue & PR**：报告问题、贡献代码
- 📚 **博客分享**：输出学习心得，帮助他人

---

## 📈 学习检验清单

### 入门级检验 🟢
- [ ] 能够用 3 句话介绍 Langfuse 的核心价值
- [ ] 能够画出 Langfuse 的系统架构图
- [ ] 能够搭建本地开发环境
- [ ] 能够理解 Web 服务和 Worker 服务的分工
- [ ] 能够创建一个简单的 tRPC Router

### 进阶级检验 🟡
- [ ] 能够画出完整的分层架构图
- [ ] 能够解释 tRPC 端到端类型安全的原理
- [ ] 能够理解 Trace 的完整生命周期
- [ ] 能够实现一个新的业务功能模块
- [ ] 能够优化 ClickHouse 查询性能
- [ ] 能够设计一个新的异步任务队列

### 专家级检验 🔴
- [ ] 能够从架构层面分析 Langfuse 的优劣势
- [ ] 能够设计 Langfuse 的扩展性方案
- [ ] 能够进行生产环境的性能优化
- [ ] 能够为 Langfuse 贡献核心代码
- [ ] 能够基于 Langfuse 打造定制化平台
- [ ] 能够撰写高质量的架构分析文章

---

## 🎯 学习资源推荐

### 官方资源
- 📘 **官方文档**：https://langfuse.com/docs
- 💻 **GitHub 仓库**：https://github.com/langfuse/langfuse
- 🎥 **官方视频**：YouTube 频道
- 💬 **Discord 社区**：官方技术支持

### 技术栈学习
- 📗 **Next.js**：https://nextjs.org/docs
- 📕 **tRPC**：https://trpc.io/docs
- 📙 **Prisma**：https://www.prisma.io/docs
- 📘 **ClickHouse**：https://clickhouse.com/docs
- 📓 **BullMQ**：https://docs.bullmq.io

### 进阶阅读
- 📚 《分布式追踪系统设计》
- 📚 《LLM 应用工程实战》
- 📚 《高性能 Web 架构设计》
- 📚 《数据密集型应用系统设计》

---

## 💡 学习建议

### 对于初学者
1. **不要急于求成**：先建立全局认知，再深入细节
2. **动手实践**：光看不练假把式，必须运行和修改代码
3. **做好笔记**：记录关键概念、架构图、代码示例
4. **善用工具**：VSCode、PlantUML、Excalidraw

### 对于进阶者
1. **关注设计模式**：理解为什么这样设计，有什么优势
2. **对比学习**：与其他项目对比，找出差异和创新点
3. **性能优化**：关注性能瓶颈和优化策略
4. **参与社区**：提问、讨论、贡献代码

### 对于专家级
1. **源码深入**：阅读核心模块的完整源码
2. **架构演进**：研究项目的历史演进和技术选型变化
3. **扩展创新**：基于 Langfuse 实现新的功能或平台
4. **知识输出**：写博客、做分享、贡献开源

---

## 🚀 开始学习

现在，根据您的技术背景和学习目标，选择合适的学习路径：

- 🟢 **快速路径（1周）**：快速了解核心价值和架构设计
- 🟡 **标准路径（3-4周）**：深入理解设计思想和工程实践
- 🔴 **专家路径（6-8周）**：全面掌握源码和最佳实践

**推荐起点**：
👉 开始阅读 [01-overview/project-overview.md](./01-overview/project-overview.md)

---

**祝学习愉快！** 🎉

如有问题，欢迎在 GitHub Discussions 提问，或加入 Discord 社区交流。
