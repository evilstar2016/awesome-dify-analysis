# tRPC API 层 (tRPC API Layer)

## 1. 层职责

tRPC API 层是 Langfuse 的核心 API 层，负责：
- 提供类型安全的 API 端点（tRPC Procedures）
- 处理客户端请求的验证和授权
- 调用业务服务层执行业务逻辑
- 返回格式化的响应数据
- 错误处理和转换
- 事务管理和数据一致性保证

**架构定位**：位于前端展示层和业务服务层之间，作为类型安全的桥梁。

---

## 2. 主要组件

### 2.1 tRPC 核心配置

#### tRPC 上下文（Context）
**路径**：`web/src/server/api/trpc.ts`

**职责**：
- 创建请求上下文
- 注入依赖（数据库、session 等）
- 提供认证信息

```typescript
export const createTRPCContext = async (opts: {
  headers: Headers;
}) => {
  const session = await getServerAuthSession();
  
  return {
    session,
    prisma,
    clickhouse,
    // ... 其他依赖
  };
};
```

#### tRPC 实例配置
**路径**：`web/src/server/api/trpc.ts`

**包含**：
- `t` - tRPC 实例
- `publicProcedure` - 公开访问的 Procedure
- `protectedProcedure` - 需要认证的 Procedure
- 中间件配置

### 2.2 API 路由器（Routers）

#### 根路由器
**路径**：`web/src/server/api/root.ts`

**作用**：汇总所有子路由器

```typescript
export const appRouter = createTRPCRouter({
  traces: tracesRouter,
  observations: observationsRouter,
  prompts: promptsRouter,
  datasets: datasetsRouter,
  evals: evalsRouter,
  scores: scoresRouter,
  // ... 更多路由器
});

export type AppRouter = typeof appRouter;
```

#### 子路由器列表
**路径**：`web/src/server/api/routers/`

| 路由器 | 文件 | 功能描述 |
|-------|------|---------|
| **traces** | `traces.ts` | Traces 追踪管理 |
| **observations** | `observations.ts` | Observations 观测数据 |
| **prompts** | `prompts.ts` | Prompts 提示词管理 |
| **datasets** | `datasets.ts` | Datasets 数据集管理 |
| **scores** | `scores.ts` | Scores 评分管理 |
| **scoreConfigs** | `scoreConfigs.ts` | Score 配置 |
| **sessions** | `sessions.ts` | Sessions 会话管理 |
| **models** | `models.ts` | Models 模型配置 |
| **media** | `media.ts` | Media 多媒体处理 |
| **users** | `users.ts` | Users 用户管理 |
| **userAccount** | `userAccount.ts` | User Account 账户管理 |
| **comments** | `comments.ts` | Comments 评论 |
| **commentReactions** | `commentReactions.ts` | Comment Reactions 评论反应 |
| **dashboardWidgets** | `dashboardWidgets.ts` | Dashboard Widgets 仪表盘部件 |
| **auditLogs** | `auditLogs.ts` | Audit Logs 审计日志 |
| **surveys** | `surveys.ts` | Surveys 调查问卷 |
| **notificationPreferences** | `notificationPreferences.ts` | 通知偏好 |
| **tableViewPresets** | `tableViewPresets.ts` | 表格视图预设 |
| **utilities** | `utilities.ts` | 工具类 API |
| **public** | `public.ts` | 公开 API（不需要认证） |
| **generations** | `generations/` | Generations 相关（可能是多个子路由器） |

### 2.3 API 定义（Definitions）

#### 作用
将 Zod schema 和类型定义与路由器解耦

**路径**：`web/src/server/api/definitions/`

**包含**：
- 输入输出的 Zod schema
- 复用的类型定义
- 常量和枚举

### 2.4 API 服务（Services）

#### 作用
封装可复用的业务逻辑，供多个路由器调用

**路径**：`web/src/server/api/services/`

**示例**：
- 权限检查服务
- 数据转换服务
- 查询构建服务

---

## 3. 对外接口（tRPC Procedures）

### 3.1 Traces API

**路由器**：`tracesRouter`

| Procedure | 类型 | 输入 | 输出 | 说明 |
|-----------|------|------|------|------|
| `all` | Query | `{ projectId, page, limit, filter }` | `{ traces[], totalCount }` | 获取 traces 列表 |
| `byId` | Query | `{ traceId, projectId }` | `Trace` | 获取单个 trace 详情 |
| `filterOptions` | Query | `{ projectId }` | `FilterOptions` | 获取过滤选项 |
| `delete` | Mutation | `{ traceId, projectId }` | `void` | 删除 trace |
| `bookmark` | Mutation | `{ traceId, projectId, bookmarked }` | `void` | 标记/取消标记 |
| `updateTags` | Mutation | `{ traceId, projectId, tags }` | `void` | 更新标签 |

### 3.2 Prompts API

**路由器**：`promptsRouter`

| Procedure | 类型 | 输入 | 输出 | 说明 |
|-----------|------|------|------|------|
| `all` | Query | `{ projectId, page, limit }` | `{ prompts[], totalCount }` | 获取 prompts 列表 |
| `byId` | Query | `{ promptId, version }` | `Prompt` | 获取 prompt 详情 |
| `create` | Mutation | `{ name, prompt, config }` | `Prompt` | 创建新 prompt |
| `update` | Mutation | `{ id, prompt, config }` | `Prompt` | 更新 prompt |
| `delete` | Mutation | `{ id }` | `void` | 删除 prompt |
| `allVersions` | Query | `{ promptName, projectId }` | `PromptVersion[]` | 获取所有版本 |

### 3.3 Datasets API

**路由器**：`datasetsRouter`

| Procedure | 类型 | 输入 | 输出 | 说明 |
|-----------|------|------|------|------|
| `all` | Query | `{ projectId }` | `Dataset[]` | 获取所有数据集 |
| `byId` | Query | `{ datasetId, projectId }` | `Dataset` | 获取数据集详情 |
| `create` | Mutation | `{ name, description, projectId }` | `Dataset` | 创建数据集 |
| `items` | Query | `{ datasetId }` | `DatasetItem[]` | 获取数据集项 |
| `addItem` | Mutation | `{ datasetId, input, expectedOutput }` | `DatasetItem` | 添加数据集项 |
| `runOnDataset` | Mutation | `{ datasetId, runConfig }` | `DatasetRun` | 在数据集上运行评估 |

### 3.4 Scores API

**路由器**：`scoresRouter`

| Procedure | 类型 | 输入 | 输出 | 说明 |
|-----------|------|------|------|------|
| `all` | Query | `{ projectId, traceId }` | `Score[]` | 获取评分列表 |
| `create` | Mutation | `{ traceId, name, value, comment }` | `Score` | 创建评分 |
| `update` | Mutation | `{ scoreId, value, comment }` | `Score` | 更新评分 |
| `delete` | Mutation | `{ scoreId }` | `void` | 删除评分 |

### 3.5 Users & Auth API

**路由器**：`usersRouter`, `userAccountRouter`

| Procedure | 类型 | 输入 | 输出 | 说明 |
|-----------|------|------|------|------|
| `allInProject` | Query | `{ projectId }` | `User[]` | 获取项目用户 |
| `addUserToProject` | Mutation | `{ projectId, userId, role }` | `void` | 添加用户到项目 |
| `updateUserRole` | Mutation | `{ userId, projectId, role }` | `void` | 更新用户角色 |
| `removeUserFromProject` | Mutation | `{ userId, projectId }` | `void` | 移除用户 |
| `currentUser` | Query | `{}` | `User` | 获取当前用户 |

### 3.6 Public API (无需认证)

**路由器**：`publicRouter`

| Procedure | 类型 | 输入 | 输出 | 说明 |
|-----------|------|------|------|------|
| `health` | Query | `{}` | `{ status: "ok" }` | 健康检查 |

---

## 4. 与其他层的交互

### 4.1 上层依赖（前端展示层）

**交互方式**：
```
React 组件
  ↓
tRPC React Query Hooks
  ↓
HTTP POST /api/trpc/[procedure]
  ↓
tRPC Server Handler
  ↓
tRPC Procedure
```

**示例**：
```typescript
// 前端调用
const { data } = trpc.traces.all.useQuery({
  projectId: "abc",
  page: 1,
  limit: 50,
});

// 后端处理
export const tracesRouter = createTRPCRouter({
  all: protectedProcedure
    .input(z.object({
      projectId: z.string(),
      page: z.number(),
      limit: z.number(),
    }))
    .query(async ({ input, ctx }) => {
      // 调用业务逻辑
      return await getTraces(ctx.prisma, input);
    }),
});
```

### 4.2 下层依赖（业务服务层）

**交互方式**：
```
tRPC Procedure
  ↓
Feature Modules (web/src/features/)
  ↓
Shared Services (@langfuse/shared)
  ↓
Database (Prisma / Kysely)
```

**数据流**：
1. tRPC Procedure 接收请求
2. 验证输入（Zod）
3. 检查权限（middleware）
4. 调用 Feature 模块
5. Feature 模块调用共享服务
6. 访问数据库
7. 返回结果

### 4.3 横向依赖（认证层）

**NextAuth 集成**：
```typescript
// 获取 session
const session = await getServerAuthSession();

// 在 context 中提供
export const createTRPCContext = async () => {
  return {
    session,
    prisma,
  };
};

// 在 procedure 中使用
export const protectedProcedure = t.procedure
  .use(async ({ ctx, next }) => {
    if (!ctx.session?.user) {
      throw new TRPCError({ code: "UNAUTHORIZED" });
    }
    return next({
      ctx: {
        session: { ...ctx.session, user: ctx.session.user },
      },
    });
  });
```

### 4.4 数据库层交互

**Prisma（PostgreSQL）**：
```typescript
// 在 procedure 中使用
.query(async ({ ctx }) => {
  const traces = await ctx.prisma.trace.findMany({
    where: { projectId },
    include: { scores: true },
  });
  return traces;
})
```

**Kysely（ClickHouse）**：
```typescript
// 通过共享服务
import { queryClickhouse } from "@langfuse/shared/src/clickhouse";

.query(async ({ input }) => {
  const observations = await queryClickhouse({
    query: "SELECT * FROM observations WHERE traceId = ?",
    params: [input.traceId],
  });
  return observations;
})
```

---

## 5. 关键代码文件路径

### 5.1 核心配置

| 文件 | 路径 | 作用 |
|-----|------|------|
| **tRPC 配置** | `web/src/server/api/trpc.ts` | tRPC 实例和中间件 |
| **根路由器** | `web/src/server/api/root.ts` | 汇总所有子路由器 |
| **认证** | `web/src/server/auth.ts` | NextAuth 配置 |
| **数据库** | `web/src/server/db.ts` | Prisma 客户端 |

### 5.2 路由器目录

```
web/src/server/api/routers/
├── traces.ts                    # Traces API
├── observations.ts              # Observations API
├── prompts.ts                   # Prompts API
├── datasets.ts                  # Datasets API
├── scores.ts                    # Scores API
├── scoreConfigs.ts              # Score 配置 API
├── sessions.ts                  # Sessions API
├── models.ts                    # Models API
├── media.ts                     # Media API
├── users.ts                     # Users API
├── userAccount.ts               # User Account API
├── comments.ts                  # Comments API
├── commentReactions.ts          # Comment Reactions API
├── dashboardWidgets.ts          # Dashboard Widgets API
├── auditLogs.ts                 # Audit Logs API
├── surveys.ts                   # Surveys API
├── notificationPreferences.ts   # Notification Preferences API
├── tableViewPresets.ts          # Table View Presets API
├── utilities.ts                 # Utilities API
├── public.ts                    # Public API
└── generations/                 # Generations 相关
```

### 5.3 辅助目录

```
web/src/server/api/
├── definitions/                 # Zod schemas 和类型定义
├── services/                    # 可复用的服务逻辑
└── utils/                       # 工具函数
```

---

## 6. 技术实现细节

### 6.1 tRPC 技术栈

| 技术 | 版本 | 用途 |
|-----|------|------|
| **@trpc/server** | 11.4.4 | tRPC 服务器端 |
| **@trpc/client** | 11.4.4 | tRPC 客户端 |
| **@trpc/react-query** | 11.4.4 | tRPC React Query 集成 |
| **@trpc/next** | 11.4.4 | tRPC Next.js 集成 |
| **Zod** | 3.25.62 | 输入输出验证 |
| **SuperJSON** | 2.2.2 | 序列化（支持 Date, Map, Set） |

### 6.2 Procedure 类型

#### Query Procedure
**用途**：查询数据（GET 语义）

```typescript
export const tracesRouter = createTRPCRouter({
  all: protectedProcedure
    .input(z.object({ projectId: z.string() }))
    .query(async ({ input, ctx }) => {
      // 只读操作
      return await ctx.prisma.trace.findMany({
        where: { projectId: input.projectId },
      });
    }),
});
```

#### Mutation Procedure
**用途**：修改数据（POST/PUT/DELETE 语义）

```typescript
export const tracesRouter = createTRPCRouter({
  delete: protectedProcedure
    .input(z.object({ traceId: z.string() }))
    .mutation(async ({ input, ctx }) => {
      // 写操作
      await ctx.prisma.trace.delete({
        where: { id: input.traceId },
      });
    }),
});
```

### 6.3 中间件（Middleware）

#### 认证中间件
```typescript
const enforceUserIsAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.session?.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({
    ctx: {
      session: { ...ctx.session, user: ctx.session.user },
    },
  });
});

export const protectedProcedure = t.procedure.use(enforceUserIsAuthed);
```

#### 权限检查中间件
```typescript
const checkProjectAccess = t.middleware(async ({ ctx, input, next }) => {
  const hasAccess = await verifyProjectAccess(
    ctx.session.user.id,
    input.projectId
  );
  
  if (!hasAccess) {
    throw new TRPCError({ code: "FORBIDDEN" });
  }
  
  return next();
});
```

#### 日志中间件
```typescript
const logMiddleware = t.middleware(async ({ path, type, next }) => {
  const start = Date.now();
  const result = await next();
  const duration = Date.now() - start;
  
  console.log(`${type} ${path} - ${duration}ms`);
  
  return result;
});
```

### 6.4 错误处理

#### tRPC 错误类型
```typescript
import { TRPCError } from "@trpc/server";

// 常见错误码
throw new TRPCError({
  code: "UNAUTHORIZED",      // 401
  code: "FORBIDDEN",          // 403
  code: "NOT_FOUND",          // 404
  code: "BAD_REQUEST",        // 400
  code: "INTERNAL_SERVER_ERROR", // 500
  message: "自定义错误消息",
});
```

#### 错误转换
```typescript
// 将 Prisma 错误转换为 tRPC 错误
try {
  await ctx.prisma.trace.findUniqueOrThrow({
    where: { id: input.traceId },
  });
} catch (error) {
  if (error.code === "P2025") {
    throw new TRPCError({
      code: "NOT_FOUND",
      message: "Trace not found",
    });
  }
  throw error;
}
```

### 6.5 输入输出验证

#### Zod Schema 定义
```typescript
// 定义输入 schema
const createPromptInput = z.object({
  name: z.string().min(1).max(100),
  prompt: z.string(),
  config: z.object({
    model: z.string(),
    temperature: z.number().min(0).max(2),
  }),
  projectId: z.string().cuid(),
});

// 在 procedure 中使用
.input(createPromptInput)
.mutation(async ({ input }) => {
  // input 已经过验证
});
```

#### 输出类型推导
```typescript
// 服务器端
.query(async () => {
  return {
    id: "abc",
    name: "Test",
    createdAt: new Date(),
  };
})

// 客户端自动获得类型
const { data } = trpc.traces.byId.useQuery();
// data: { id: string; name: string; createdAt: Date; }
```

### 6.6 Context 依赖注入

#### Context 定义
```typescript
export const createTRPCContext = async (opts: {
  headers: Headers;
}) => {
  const session = await getServerAuthSession();
  
  return {
    session,
    prisma,
    clickhouse,
    redis,
    s3,
    // 可以注入任何依赖
  };
};

export type Context = Awaited<ReturnType<typeof createTRPCContext>>;
```

#### 在 Procedure 中使用
```typescript
.query(async ({ ctx }) => {
  // ctx.prisma - Prisma 客户端
  // ctx.session - 用户 session
  // ctx.clickhouse - ClickHouse 客户端
  
  const traces = await ctx.prisma.trace.findMany();
  return traces;
})
```

### 6.7 批处理（Batching）

**自动批处理**：
```typescript
// 多个并发请求会自动批处理
const [user, posts, comments] = await Promise.all([
  trpc.users.byId.query({ id: 1 }),
  trpc.posts.all.query({ userId: 1 }),
  trpc.comments.all.query({ userId: 1 }),
]);

// tRPC 会将这些请求批处理为单个 HTTP 请求
```

### 6.8 数据转换（SuperJSON）

**自动序列化**：
```typescript
// 服务器返回
.query(async () => {
  return {
    date: new Date(),
    map: new Map([["key", "value"]]),
    set: new Set([1, 2, 3]),
  };
})

// 客户端自动反序列化
const { data } = trpc.example.useQuery();
// data.date 仍然是 Date 对象
// data.map 仍然是 Map 对象
```

---

## 7. 设计模式和最佳实践

### 7.1 路由器组织

#### 按功能拆分
```typescript
// 不要将所有 API 放在一个文件
// ❌ 不好
export const appRouter = createTRPCRouter({
  getAllTraces: procedure.query(...),
  getTraceById: procedure.query(...),
  getAllPrompts: procedure.query(...),
  // ... 100+ procedures
});

// ✅ 好
export const appRouter = createTRPCRouter({
  traces: tracesRouter,
  prompts: promptsRouter,
  datasets: datasetsRouter,
});
```

#### 嵌套路由器
```typescript
// generations/ 目录包含多个子路由器
export const generationsRouter = createTRPCRouter({
  list: generationsListRouter,
  detail: generationsDetailRouter,
  stats: generationsStatsRouter,
});
```

### 7.2 输入验证

#### 复用 Schema
```typescript
// 定义可复用的 schema
const projectIdSchema = z.string().cuid();
const paginationSchema = z.object({
  page: z.number().min(1),
  limit: z.number().min(1).max(100),
});

// 在多个 procedure 中复用
.input(z.object({
  projectId: projectIdSchema,
  ...paginationSchema.shape,
}))
```

#### 严格验证
```typescript
// 使用 Zod 的高级特性
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const dateSchema = z.coerce.date();
const enumSchema = z.enum(["VIEWER", "MEMBER", "ADMIN"]);
```

### 7.3 权限控制

#### 细粒度权限
```typescript
.mutation(async ({ input, ctx }) => {
  // 检查项目访问权限
  await verifyProjectAccess(ctx.session.user.id, input.projectId);
  
  // 检查资源拥有权
  const trace = await ctx.prisma.trace.findUnique({
    where: { id: input.traceId },
  });
  
  if (trace.projectId !== input.projectId) {
    throw new TRPCError({ code: "FORBIDDEN" });
  }
  
  // 执行操作
})
```

### 7.4 事务管理

#### Prisma 事务
```typescript
.mutation(async ({ input, ctx }) => {
  return await ctx.prisma.$transaction(async (tx) => {
    // 在事务中执行多个操作
    const trace = await tx.trace.create({
      data: { name: input.name },
    });
    
    await tx.observation.createMany({
      data: input.observations.map(obs => ({
        traceId: trace.id,
        ...obs,
      })),
    });
    
    return trace;
  });
})
```

### 7.5 性能优化

#### 选择性加载
```typescript
.query(async ({ input, ctx }) => {
  return await ctx.prisma.trace.findMany({
    where: { projectId: input.projectId },
    select: {
      id: true,
      name: true,
      timestamp: true,
      // 只选择需要的字段
    },
  });
})
```

#### 数据库索引
```typescript
// 确保查询使用的字段有索引
// 在 Prisma schema 中定义
model Trace {
  id        String   @id
  projectId String
  timestamp DateTime
  
  @@index([projectId, timestamp])
}
```

### 7.6 错误日志

#### 结构化日志
```typescript
.mutation(async ({ input, ctx }) => {
  try {
    // 业务逻辑
  } catch (error) {
    logger.error("Failed to create trace", {
      userId: ctx.session.user.id,
      projectId: input.projectId,
      error: error.message,
    });
    throw error;
  }
})
```

---

## 8. 安全性

### 8.1 认证
- **NextAuth Session**：验证用户身份
- **API Key**：支持程序化访问（通过 REST API，非 tRPC）
- **JWT Token**：Session 令牌

### 8.2 授权
- **项目级权限**：用户必须属于项目才能访问
- **角色检查**：VIEWER, MEMBER, ADMIN 权限
- **资源级权限**：检查资源所有权

### 8.3 输入验证
- **Zod Schema**：严格的输入验证
- **SQL 注入防护**：Prisma 参数化查询
- **XSS 防护**：输入清理

### 8.4 速率限制
- **中间件**：可选的速率限制中间件
- **Redis**：使用 Redis 存储速率限制计数器

---

## 9. 监控和可观测性

### 9.1 OpenTelemetry 集成
```typescript
import { trace } from "@opentelemetry/api";

.query(async ({ input }) => {
  const span = trace.getActiveSpan();
  span?.setAttribute("projectId", input.projectId);
  
  // 执行查询
});
```

### 9.2 错误追踪
- **Sentry 集成**：自动捕获 tRPC 错误
- **错误上下文**：包含用户和请求信息

### 9.3 性能监控
- **响应时间**：监控 procedure 执行时间
- **慢查询**：记录超过阈值的查询

---

## 10. 与 REST API 的对比

| 特性 | tRPC | REST API |
|-----|------|----------|
| **类型安全** | ✅ 端到端类型安全 | ❌ 需要手动维护类型 |
| **API 文档** | ✅ 自动生成 | ❌ 需要手动编写 |
| **客户端代码** | ✅ 自动生成 | ❌ 需要手动编写 |
| **运行时验证** | ✅ Zod | ❌ 需要手动验证 |
| **开发速度** | ✅ 快速 | ❌ 较慢 |
| **公开 API** | ❌ 不适合 | ✅ 标准化 |
| **跨语言** | ❌ 仅 TypeScript | ✅ 任何语言 |

**注意**：Langfuse 同时提供 tRPC（内部）和 REST API（公开，位于 `web/src/pages/api/public/`）。

---

## 11. 总结

tRPC API 层是 Langfuse 内部 API 的核心，通过类型安全、自动验证和优秀的开发体验，大大提高了前后端协作效率。这一层作为前端和业务逻辑之间的桥梁，确保了数据的正确性和安全性。

**关键优势**：
- ✅ 端到端类型安全
- ✅ 自动输入输出验证
- ✅ 优秀的开发体验
- ✅ 内置错误处理
- ✅ 高性能（批处理、缓存）

**适用场景**：
- ✅ TypeScript 全栈应用
- ✅ 内部 API（非公开）
- ✅ 快速迭代开发

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0
