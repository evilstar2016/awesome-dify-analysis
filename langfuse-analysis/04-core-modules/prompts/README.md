# Prompts 模块

## 模块概述

Prompts 模块是 Langfuse 的**核心特性模块**之一，提供 LLM 提示词的版本管理、A/B 测试和部署能力。该模块使团队能够集中管理提示词，追踪变更历史，并安全地发布新版本，是实现 LLM 应用 MLOps 的关键组件。

### 核心价值
- 📝 **版本管理**: 提示词的完整版本历史
- 🏷️ **标签系统**: 通过标签（labels）管理部署环境
- 🔄 **A/B 测试**: 支持多版本并行测试
- 🗂️ **文件夹组织**: 分层目录结构管理大量提示词
- 🚀 **即时部署**: 通过 SDK 快速获取最新提示词
- 💾 **Redis 缓存**: 高性能缓存机制
- 🔒 **权限控制**: 支持受保护的标签（protected labels）

---

## 核心概念

### Prompt (提示词)
**定义**: 一个提示词模板，包含内容、配置和元数据。

**属性**:
- `id`: 唯一标识符
- `name`: 提示词名称（支持文件夹路径，如 `folder/subfolder/prompt`）
- `version`: 版本号（自增整数）
- `prompt`: 提示词内容（支持 Mustache 模板语法）
- `type`: 类型（`text` / `chat`）
- `config`: 配置对象（模型参数、温度等）
- `labels`: 标签列表（如 `production`, `staging`, `latest`）
- `tags`: 标签（用于分类和搜索）
- `isActive`: 是否激活（废弃字段，使用 labels 替代）
- `createdBy`: 创建者（`API` / 用户邮箱）
- `commitMessage`: 版本说明
- `projectId`: 所属项目
- `createdAt` / `updatedAt`: 时间戳

**数据模型**:
```typescript
export const PromptDomainSchema = z.object({
  id: z.string(),
  name: z.string(),
  version: z.number(),
  createdAt: z.date(),
  updatedAt: z.date(),
  createdBy: z.string(),
  isActive: z.boolean().nullable(),
  type: z.string().default("text"),
  tags: z.array(z.string()).default([]),
  labels: z.array(z.string()).default([]),
  prompt: jsonSchemaNullable,
  config: jsonSchemaNullable,
  projectId: z.string(),
  commitMessage: z.string().nullable(),
});
```

### Prompt 类型

#### 1. Text Prompt
**用途**: 单一文本提示词  
**格式**: 
```json
{
  "type": "text",
  "prompt": "You are a helpful assistant. {{context}}"
}
```

#### 2. Chat Prompt
**用途**: 多轮对话提示词  
**格式**:
```json
{
  "type": "chat",
  "prompt": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": "{{userMessage}}"
    }
  ]
}
```

### 版本管理机制

**版本号规则**:
- 从 1 开始自增
- 每次创建新版本时自动递增
- 同一 `name` 的所有版本共享版本序列

**版本创建**:
```
Prompt "chat-completion"
  ├─ Version 1 (初始版本)
  ├─ Version 2 (优化提示词)
  ├─ Version 3 (添加示例)
  └─ Version 4 (当前版本)
```

### 标签系统 (Labels)

**标签作用**:
- 标识部署环境（如 `production`, `staging`）
- 支持 A/B 测试（如 `variant-a`, `variant-b`）
- 特殊标签 `latest` 自动指向最新版本

**标签类型**:
1. **普通标签**: 用户可以自由管理
2. **受保护标签**: 需要特殊权限才能修改（如 `production`）

**示例**:
```
Prompt "chat-completion"
  ├─ Version 1: []
  ├─ Version 2: ["staging"]
  ├─ Version 3: ["production"]
  └─ Version 4: ["latest", "staging"]
```

### 文件夹组织

**路径命名**:
- 使用 `/` 分隔文件夹层级
- 示例：`customer-service/greeting`, `internal/debugging/system`

**文件夹视图**:
```
prompts/
├─ customer-service/
│   ├─ greeting
│   ├─ farewell
│   └─ escalation
├─ analytics/
│   ├─ summarization
│   └─ classification
└─ system/
    └─ debug-helper
```

---

## 主要功能

### 1. Prompt 管理
- **创建 Prompt**: 创建新提示词或新版本
- **查询 Prompt**: 按名称、版本、标签查询
- **更新 Prompt**: 创建新版本（不可修改已有版本）
- **删除 Prompt**: 删除指定版本或所有版本
- **复制 Prompt**: 基于现有版本创建新版本
- **搜索 Prompt**: 全文搜索名称、标签、内容

### 2. 标签管理
- **添加标签**: 为版本添加标签
- **移除标签**: 从版本移除标签
- **转移标签**: 将标签从一个版本转移到另一个版本
- **受保护标签**: 限制关键标签的修改权限

### 3. 版本比较
- **Diff 视图**: 对比两个版本的差异
- **历史记录**: 查看完整的版本变更历史
- **回滚**: 基于旧版本创建新版本

### 4. SDK 集成
- **获取 Prompt**: 通过 SDK 获取指定版本或标签的提示词
- **模板渲染**: 使用变量渲染 Mustache 模板
- **缓存优化**: Redis 缓存加速获取

### 5. 使用统计
- **观测计数**: 统计使用该 Prompt 的 Observations 数量
- **质量指标**: 关联 Scores 分析质量

---

## 技术架构

### 数据存储

#### PostgreSQL Schema
**表**: `prompts`  
**Schema**: 见 `packages/shared/prisma/schema.prisma`

```prisma
model Prompt {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @default(now()) @updatedAt @map("updated_at")

  projectId String  @map("project_id")
  project   Project @relation(fields: [projectId], references: [id], onDelete: Cascade)

  createdBy String @map("created_by")

  prompt           Json                 // 提示词内容（必填）
  name             String               // 支持路径，如 "folder/subfolder/prompt"
  version          Int                  // 版本号
  type             String               @default("text")
  isActive         Boolean?             @map("is_active") // 已废弃
  config           Json                 @default("{}") @db.Json
  tags             String[]             @default([])
  labels           String[]             @default([])
  commitMessage    String?              @map("commit_message")
  PromptDependency PromptDependency[]

  @@unique([projectId, name, version])
  @@index([projectId, id])
  @@index([createdAt])
  @@index([updatedAt])
  @@index([tags(ops: ArrayOps)], type: Gin)
  @@map("prompts")
}
```

#### 相关表

**PromptDependency 表**:
用于存储 Prompt 之间的依赖关系，支持 Prompt 嵌套和引用。

```prisma
model PromptDependency {
  id           String  @id @default(cuid())
  projectId    String  @map("project_id")
  parentId     String  @map("parent_id")
  parent       Prompt  @relation(fields: [parentId], references: [id])
  childName    String  @map("child_name")
  childLabel   String? @map("child_label")
  childVersion Int?    @map("child_version")
  
  @@index([projectId, parentId])
  @@index([projectId, childName])
  @@map("prompt_dependencies")
}
```

**PromptProtectedLabels 表**:
存储受保护的 Label 列表，只有特定权限的用户才能修改带有受保护 Label 的 Prompt。

```prisma
model PromptProtectedLabels {
  id        String  @id @default(cuid())
  projectId String  @map("project_id")
  label     String
  
  @@unique([projectId, label])
  @@map("prompt_protected_labels")
}
```

#### ClickHouse (分析数据)
**表**: `prompts`  
**用途**: 用于高性能查询和分析

**同步**: Worker 异步同步到 ClickHouse

### 缓存策略

#### Redis 缓存
**路径**: `packages/shared/src/server/services/PromptService.ts`

**缓存 Key 格式**:
```
prompt:<project-id>:<prompt-name>:<version-or-label>
```

**示例**:
```
prompt:proj123:chat-completion:1
prompt:proj123:chat-completion:production
prompt:proj123:chat-completion:latest
```

**缓存机制**:
1. **写入时**: 
   - 获取 Redis 锁
   - 删除该 Prompt 的所有缓存 Key
   - 写入 PostgreSQL
   - 释放锁

2. **读取时**:
   - 检查是否存在锁
   - 如无锁，从缓存读取并刷新 TTL
   - 如有锁或缓存未命中，从 PostgreSQL 读取并写入缓存

3. **TTL**: 默认 1 小时

**优势**:
- 极低延迟（< 1ms）
- 减轻数据库压力
- 支持高并发访问

### API 架构

#### tRPC Routers
**路径**: `web/src/features/prompts/server/routers/promptRouter.ts`

**主要 Procedures**:

| Procedure | 输入 | 输出 | 说明 |
|----------|------|------|------|
| `hasAny` | projectId | boolean | 检查是否有任何 Prompt |
| `all` | filter, orderBy, pagination | prompts[], totalCount | 查询 Prompts 列表 |
| `count` | projectId, searchQuery | totalCount | 统计 Prompts 数量 |
| `metrics` | promptNames | observationCount[] | 获取使用统计 |
| `byId` | promptId | prompt | 获取单个 Prompt |
| `allVersions` | name | prompts[] | 获取所有版本 |
| `allNames` | projectId, type? | prompt names | 获取所有 Prompt 名称 |
| `allLabels` | projectId | labels[] | 获取所有 Labels |
| `allPromptMeta` | projectId | prompts metadata | 获取所有 Prompts 元数据 |
| `create` | name, prompt, config, labels | prompt | 创建新 Prompt |
| `duplicatePrompt` | promptId, name, isSingleVersion | prompt | 复制 Prompt |
| `delete` | promptName | void | 删除 Prompt（所有版本）|
| `deleteVersion` | promptVersionId | void | 删除单个 Prompt 版本 |
| `updateTags` | name, tags | void | 更新 Tags |
| `setLabels` | promptId, labels | void | 设置 Labels |
| `filterOptions` | projectId | filter options | 获取筛选选项 |
| `versionMetrics` | promptIds | metrics[] | 获取版本指标 |
| `resolvePromptGraph` | promptId | prompt graph | 解析 Prompt 依赖图 |
| `getPromptLinkOptions` | projectId | link options | 获取 Prompt 链接选项 |
| `getProtectedLabels` | projectId | labels[] | 获取受保护的 Labels |
| `addProtectedLabel` | projectId, label | label | 添加受保护的 Label |
| `removeProtectedLabel` | projectId, label | success | 移除受保护的 Label |

#### Public API (REST)
**路径**: `web/src/pages/api/public/prompts.ts`

**端点**:
- `GET /api/public/prompts` - 查询 Prompts 元数据
- `GET /api/public/prompts/:promptName` - 获取 Prompt（支持 version/label 参数）
- `POST /api/public/prompts` - 创建 Prompt

### 服务层
**路径**: `packages/shared/src/server/services/PromptService/index.ts`

**主要方法**:
- `getPrompt(projectId, name, version?, label?)` - 获取 Prompt（带缓存）
- `createPrompt(input)` - 创建新 Prompt
- `invalidateCache(projectId, name)` - 清除缓存
- `acquireLock(projectId, name)` - 获取锁
- `releaseLock(projectId, name)` - 释放锁

### 变更事件溯源
**路径**: `web/src/features/prompts/server/promptChangeEventSourcing.ts`

**功能**:
- 记录所有 Prompt 变更事件
- 支持历史回溯
- 审计日志

---

## 目录结构

```
# 前端
web/src/pages/project/[projectId]/prompts/
├── index.tsx                       # Prompts 列表页
└── [promptName]/
    ├── [promptVersion].tsx         # Prompt 详情页
    └── index.tsx                   # Prompt 所有版本

web/src/features/prompts/
├── components/                     # React 组件
├── hooks/                          # 自定义 Hooks
├── server/                         # 后端逻辑
│   ├── actions/
│   │   ├── createPrompt.ts         # 创建 Prompt
│   │   └── getPromptsMeta.ts       # 获取元数据
│   ├── routers/
│   │   └── promptRouter.ts         # tRPC Router
│   └── handlers/
│       └── promptsHandler.ts       # REST API Handler
├── utils.ts                        # 工具函数
└── README.md                       # 缓存策略文档

# 后端服务
packages/shared/src/server/
├── services/
│   └── PromptService.ts            # Prompt 服务（含缓存）
└── repositories/
    └── prompts.ts                  # Prompt 仓储

# 领域模型
packages/shared/src/domain/
└── prompts.ts                      # Prompt 领域模型

# 数据表定义
packages/shared/src/tableDefinitions/
└── promptsTable.ts                 # ClickHouse 表定义

# 测试
web/src/__tests__/
├── prompts.v1.servertest.ts        # V1 API 测试
├── prompts.v2.servertest.ts        # V2 API 测试
├── prompts-trpc.servertest.ts      # tRPC 测试
└── promptCache.servertest.ts       # 缓存测试
```

---

## 核心流程

### 流程索引
1. [Prompt 创建流程](./01-prompt-creation-sequence.puml) - 创建新版本和标签管理
2. [Prompt 获取流程（SDK）](./02-prompt-fetch-sequence.puml) - SDK 获取 Prompt（含缓存）
3. [Prompt 查询流程（UI）](./03-prompt-list-sequence.puml) - UI 列表查询和文件夹视图
4. [标签管理流程](./04-label-management-sequence.puml) - 添加/移除/转移标签

---

## 使用场景

### 场景 1: 版本迭代
```typescript
// 1. 创建初始版本
POST /api/public/prompts
{
  "name": "chat-greeting",
  "prompt": "Hello! How can I help you?",
  "config": { "temperature": 0.7 }
}
// → Version 1, labels: ["latest"]

// 2. 创建优化版本
POST /api/public/prompts
{
  "name": "chat-greeting",
  "prompt": "Hi there! 👋 How may I assist you today?",
  "config": { "temperature": 0.7 },
  "labels": ["latest", "staging"]
}
// → Version 2, labels: ["latest", "staging"]
// Version 1 的 "latest" 自动移除

// 3. 推送到生产环境
PATCH /api/public/prompts/chat-greeting/labels
{
  "version": 2,
  "labels": ["production"]
}
// → Version 2, labels: ["latest", "staging", "production"]
```

### 场景 2: A/B 测试
```typescript
// 创建 Variant A
POST /api/public/prompts
{
  "name": "email-subject",
  "prompt": "Generate a professional email subject",
  "labels": ["variant-a"]
}

// 创建 Variant B
POST /api/public/prompts
{
  "name": "email-subject",
  "prompt": "Create a catchy email subject line",
  "labels": ["variant-b"]
}

// SDK 随机获取
const variantLabel = Math.random() < 0.5 ? "variant-a" : "variant-b";
const prompt = await langfuse.getPrompt("email-subject", { label: variantLabel });
```

### 场景 3: SDK 集成
```typescript
import { Langfuse } from "langfuse";

const langfuse = new Langfuse({
  publicKey: "pk-...",
  secretKey: "sk-...",
});

// 获取最新版本
const prompt = await langfuse.getPrompt("chat-completion");
console.log(prompt.prompt); // 提示词内容
console.log(prompt.version); // 版本号
console.log(prompt.config); // 配置参数

// 获取 production 版本
const prodPrompt = await langfuse.getPrompt("chat-completion", {
  label: "production",
});

// 模板渲染
const rendered = prompt.compile({ userName: "Alice", context: "..." });

// 使用提示词
const response = await openai.chat.completions.create({
  model: prompt.config.model ?? "gpt-4",
  messages: rendered,
  temperature: prompt.config.temperature ?? 0.7,
});

// 追踪使用
const generation = trace.generation({
  name: "chat-completion",
  model: "gpt-4",
  promptName: prompt.name,
  promptVersion: prompt.version,
  input: rendered,
  output: response,
});
```

---

## 性能优化

### 1. Redis 缓存
- **缓存命中率**: > 95%
- **响应时间**: < 1ms (缓存命中)
- **TTL**: 1 小时

### 2. 数据库索引
```sql
-- PostgreSQL
CREATE UNIQUE INDEX ON prompts(project_id, name, version);  -- 唯一约束
CREATE INDEX ON prompts(project_id, id);
CREATE INDEX ON prompts(created_at);
CREATE INDEX ON prompts(updated_at);
CREATE INDEX ON prompts USING GIN (tags array_ops);  -- GIN索引用于数组搜索
```

### 3. 查询优化
- 使用 `projectId + name + version` 唯一索引
- 标签查询使用 GIN 索引（数组类型）
- 文件夹查询使用 `LIKE` 模式匹配

### 4. 并发控制
- 使用 Redis 锁避免并发冲突
- 幂等性设计（相同输入产生相同结果）
- Optimistic locking（乐观锁）

---

## 关键技术点

### 1. 版本号生成
```typescript
// 获取当前最大版本号
const maxVersion = await prisma.prompt.aggregate({
  where: { projectId, name },
  _max: { version: true },
});

const newVersion = (maxVersion._max.version ?? 0) + 1;
```

### 2. Label 自动管理
```typescript
// 添加 label 时，自动从其他版本移除
if (input.labels.includes("latest")) {
  // 移除其他版本的 "latest" 标签
  await prisma.prompt.updateMany({
    where: {
      projectId,
      name,
      version: { not: newVersion },
    },
    data: {
      labels: { set: Prisma.sql`array_remove(labels, 'latest')` },
    },
  });
}
```

### 3. 文件夹视图生成
```sql
-- 生成文件夹和提示词的混合视图
WITH folders AS (
  SELECT DISTINCT
    split_part(name, '/', 1) as segment,
    'folder' as row_type
  FROM prompts
  WHERE project_id = ?
    AND name LIKE ?
)
SELECT * FROM folders
UNION ALL
SELECT name, 'prompt' as row_type
FROM prompts
WHERE project_id = ?
  AND name LIKE ?
ORDER BY row_type, segment
```

### 4. 受保护标签
```typescript
// 检查受保护标签
const protectedLabels = await prisma.projectSettings.findUnique({
  where: { projectId },
  select: { protectedPromptLabels: true },
});

if (labelsToAdd.some(l => protectedLabels.includes(l))) {
  // 检查用户是否有 prompts:protected_label 权限
  throwIfNoProjectAccess({
    session,
    projectId,
    scope: "prompts:protected_label",
  });
}
```

---

## 依赖关系

### 依赖的模块
- **Projects 模块**: 项目隔离
- **Auth 模块**: 认证和权限
- **Redis**: 缓存支持
- **Observations 模块**: 使用统计

### 被依赖的模块
- **Playground 模块**: 测试提示词
- **Datasets 模块**: 使用提示词生成数据
- **Traces 模块**: 追踪提示词使用
- **Analytics 模块**: 分析提示词效果

---

## 配置参数

### 环境变量
```env
# Redis (缓存)
REDIS_CONNECTION_STRING=redis://...

# PostgreSQL
DATABASE_URL=postgresql://...

# 缓存 TTL (秒)
PROMPT_CACHE_TTL=3600
```

### 项目级配置
```typescript
{
  protectedPromptLabels: ["production", "prod"], // 受保护的标签
}
```

---

## 最佳实践

### 1. 命名规范
- **文件夹**: 使用小写和连字符，如 `customer-service/`
- **提示词**: 描述性名称，如 `greeting`, `error-handling`
- **完整路径**: `customer-service/greeting`

### 2. 版本管理
- 每次修改创建新版本，不修改已有版本
- 使用 `commitMessage` 记录变更原因
- 定期清理不再使用的旧版本

### 3. 标签使用
- `latest`: 自动指向最新版本（SDK 默认）
- `production` / `prod`: 生产环境
- `staging` / `dev`: 测试环境
- `variant-*`: A/B 测试变体

### 4. 提示词内容
- 使用 Mustache 模板语法：`{{variable}}`
- Chat prompt 使用标准格式：`[{role, content}, ...]`
- 在 `config` 中存储模型参数

### 5. 性能优化
- 优先使用标签（`latest`, `production`）而非固定版本号
- 利用 Redis 缓存，避免频繁数据库查询
- 批量查询时使用 `IN` 查询而非多次单独查询

---

## 常见问题

### Q: 如何回滚到旧版本？
A: 不直接回滚，而是基于旧版本创建新版本。使用 `duplicate` 功能或直接创建新版本时复制旧版本内容。

### Q: Label 和 Tag 的区别？
A: **Label** 用于版本标识和部署管理（如 `production`, `latest`），支持自动转移。**Tag** 用于分类和搜索，不影响版本选择。

### Q: 如何实现 A/B 测试？
A: 创建同一 `name` 的多个版本，使用不同的 label（如 `variant-a`, `variant-b`）。在应用中随机选择 label 获取提示词。

### Q: 缓存如何更新？
A: 创建/更新 Prompt 时自动清除该 name 的所有缓存。下次读取时会从数据库重新加载并写入缓存。

### Q: 支持多少个版本？
A: 理论上无限制，但建议定期清理不再使用的旧版本以保持性能。

---

## 技术栈

- **前端**: Next.js 15 + React 19 + TanStack Query
- **API**: tRPC 11 + REST API
- **ORM**: Prisma 6
- **数据库**: PostgreSQL 15 + ClickHouse 24
- **缓存**: Redis 7
- **验证**: Zod 3
- **模板**: Mustache.js

---

## 参考资源

- [Prisma Schema](../../../packages/shared/prisma/schema.prisma)
- [tRPC Router](../../../web/src/features/prompts/server/routers/promptRouter.ts)
- [Domain Model](../../../packages/shared/src/domain/prompts.ts)
- [Prompt Service](../../../packages/shared/src/server/services/PromptService/index.ts)
- [Public API](../../../web/src/pages/api/public/prompts.ts)
- [缓存策略文档](../../../web/src/features/prompts/README.md)

---

**文档编写时间**: 2025-12-17  
**项目版本**: 3.140.0
