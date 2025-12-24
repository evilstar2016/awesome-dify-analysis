# 业务服务层 (Business Service Layer)

## 1. 层职责

业务服务层是 Langfuse 的核心业务逻辑层，负责：
- 实现核心业务功能和规则
- 封装可复用的业务逻辑
- 协调多个数据访问操作
- 处理复杂的业务流程
- 执行业务验证和计算
- 管理领域模型和业务实体

**架构定位**：位于 tRPC API 层和数据访问层之间，包含所有业务规则和领域逻辑。

---

## 2. 主要组件

### 2.1 功能模块（Features）

#### Web 功能模块
**路径**：`web/src/features/`

业务功能按照领域划分为独立的 feature 模块：

| 功能模块 | 目录 | 功能描述 |
|---------|------|---------|
| **traces** | `trace-graph-view/` | Traces 追踪和图形化展示 |
| **prompts** | `prompts/` | 提示词版本管理 |
| **datasets** | `datasets/` | 数据集和测试用例管理 |
| **evals** | `evals/` | 评估配置和执行逻辑 |
| **scores** | `scores/`, `score-configs/`, `score-analytics/` | 评分系统 |
| **playground** | `playground/` | LLM Playground 功能 |
| **dashboard** | `dashboard/` | 仪表盘和数据可视化 |
| **projects** | `projects/` | 项目管理 |
| **organizations** | `organizations/` | 组织管理 |
| **auth** | `auth/`, `auth-credentials/` | 认证和授权 |
| **rbac** | `rbac/` | 基于角色的访问控制 |
| **models** | `models/` | 模型配置管理 |
| **media** | `media/` | 多媒体文件处理 |
| **comments** | `comments/` | 评论系统 |
| **notifications** | `notifications/` | 通知系统 |
| **experiments** | `experiments/` | 实验和 A/B 测试 |
| **automations** | `automations/` | 自动化工作流 |
| **batch-exports** | `batch-exports/` | 批量数据导出 |
| **llm-api-key** | `llm-api-key/` | LLM API 密钥管理 |
| **mcp** | `mcp/` | Model Context Protocol 集成 |
| **widgets** | `widgets/` | Dashboard 小部件 |

#### 共享功能模块
**路径**：`packages/shared/src/features/`

跨 Web 和 Worker 共享的业务逻辑：

| 功能模块 | 功能描述 |
|---------|---------|
| **entitlements** | 权限和许可证管理 |
| **usage-metering** | 使用量计量 |
| **ingestion** | 数据摄取逻辑 |
| **query-builder** | 查询构建器 |

### 2.2 领域模型（Domain Models）

#### 路径
`packages/shared/src/domain/`

#### 核心领域实体

| 领域 | 文件 | 说明 |
|-----|------|------|
| **Trace** | `trace.ts` | 追踪实体和业务规则 |
| **Observation** | `observation.ts` | 观测实体 |
| **Prompt** | `prompt.ts` | 提示词实体 |
| **Score** | `score.ts` | 评分实体 |
| **Dataset** | `dataset.ts` | 数据集实体 |
| **Project** | `project.ts` | 项目实体 |
| **User** | `user.ts` | 用户实体 |

#### 领域模型特点
- **封装业务规则**：实体方法包含业务逻辑
- **不可变性**：使用不可变模式
- **类型安全**：TypeScript 类型定义
- **验证逻辑**：内置数据验证

### 2.3 业务服务（Services）

#### Web 服务
**路径**：`web/src/server/api/services/`

可复用的业务服务：

| 服务 | 功能 |
|-----|------|
| **权限服务** | 检查用户权限和访问控制 |
| **查询构建服务** | 构建复杂的数据库查询 |
| **数据转换服务** | 格式化和转换数据 |
| **通知服务** | 发送通知 |

#### Worker 服务
**路径**：`worker/src/services/`

后台任务相关的业务服务：

| 服务 | 功能 |
|-----|------|
| **摄取服务** | 处理数据摄取 |
| **聚合服务** | 数据聚合和统计 |
| **评估服务** | 执行 LLM 评估 |
| **导出服务** | 数据导出 |

---

## 3. 核心业务功能

### 3.1 Traces 追踪系统

#### 功能模块
`web/src/features/trace-graph-view/`

#### 核心业务逻辑
- **Trace 创建**：接收 SDK 数据，创建 Trace 记录
- **Trace 查询**：复杂的过滤、排序、分页
- **Trace 关联**：关联 Observations、Scores、Sessions
- **Trace 可视化**：树形结构和图形展示
- **Trace 导出**：批量导出功能

#### 业务规则
```typescript
// Trace 必须属于一个项目
// Trace 可以有多个 Observations
// Trace 可以有多个 Scores
// Trace 可以被标记为 Bookmarked
// Trace 可以有 Tags
```

### 3.2 Prompts 提示词管理

#### 功能模块
`web/src/features/prompts/`

#### 核心业务逻辑
- **版本控制**：自动版本管理
- **Prompt 缓存**：客户端和服务器端缓存
- **Prompt 模板**：支持变量替换
- **Prompt 测试**：在 Playground 中测试
- **Prompt 发布**：版本发布和回滚

#### 业务规则
```typescript
// Prompt 有唯一的名称（在项目内）
// 每次修改创建新版本
// 可以标记某个版本为 Production
// 支持变量 {{variable}} 语法
// 可以关联到多个 Traces
```

### 3.3 Evaluations 评估系统

#### 功能模块
`web/src/features/evals/`

#### 核心业务逻辑
- **评估配置**：定义评估规则和标准
- **LLM-as-a-Judge**：使用 LLM 进行自动评估
- **人工评估**：人工打分和反馈
- **批量评估**：在数据集上批量运行
- **评估报告**：生成评估结果报告

#### 业务规则
```typescript
// 评估可以是人工或自动
// 自动评估需要配置 LLM 和 Prompt
// 评估结果生成 Score
// 可以在数据集上批量评估
// 评估可以被调度为定时任务
```

### 3.4 Datasets 数据集管理

#### 功能模块
`web/src/features/datasets/`

#### 核心业务逻辑
- **数据集创建**：定义测试用例集合
- **数据集项管理**：添加、编辑、删除测试项
- **数据集运行**：在数据集上运行评估
- **结果追踪**：追踪每次运行的结果
- **数据集版本**：数据集变更历史

#### 业务规则
```typescript
// Dataset 包含多个 DatasetItem
// DatasetItem 有 input 和 expectedOutput
// 可以从 Traces 创建 DatasetItem
// DatasetRun 记录每次运行的结果
// 支持 A/B 对比测试
```

### 3.5 Scores 评分系统

#### 功能模块
- `web/src/features/scores/`
- `web/src/features/score-configs/`
- `web/src/features/score-analytics/`

#### 核心业务逻辑
- **Score 配置**：定义评分标准和范围
- **Score 创建**：手动或自动创建评分
- **Score 聚合**：计算平均分、分布等
- **Score 分析**：时序分析和趋势
- **Score 可视化**：图表展示

#### 业务规则
```typescript
// Score 关联到 Trace 或 Observation
// Score 有名称、值、注释
// Score 可以是数值、分类或布尔
// ScoreConfig 定义 Score 的元数据
// 支持自定义 Score 类型
```

### 3.6 Playground LLM 调试工具

#### 功能模块
`web/src/features/playground/`

#### 核心业务逻辑
- **Prompt 测试**：实时测试 Prompt
- **模型对比**：并行测试多个模型
- **参数调整**：调整温度、Top-P 等参数
- **结果保存**：保存测试结果为 Trace
- **提示词优化**：迭代优化 Prompt

#### 业务规则
```typescript
// Playground 会话不持久化（可选）
// 可以加载历史 Trace 到 Playground
// 可以保存 Playground 结果为新 Trace
// 支持多个 LLM Provider
// 需要配置 API Key
```

### 3.7 Dashboard 仪表盘

#### 功能模块
`web/src/features/dashboard/`

#### 核心业务逻辑
- **指标计算**：实时计算关键指标
- **数据聚合**：按时间、模型等维度聚合
- **图表渲染**：多种图表类型支持
- **自定义布局**：拖拽调整部件位置
- **数据刷新**：定时或手动刷新

#### 业务规则
```typescript
// Dashboard 可以有多个 Widgets
// Widget 支持拖拽和调整大小
// 数据从 ClickHouse 查询
// 支持时间范围过滤
// 可以导出图表和数据
```

---

## 4. 与其他层的交互

### 4.1 上层依赖（tRPC API 层）

**交互方式**：
```
tRPC API Procedure
  ↓ 调用
Feature Module Function
  ↓ 使用
Domain Model
  ↓ 返回
业务数据
```

**示例**：
```typescript
// tRPC Procedure
export const tracesRouter = createTRPCRouter({
  byId: protectedProcedure
    .input(z.object({ traceId: z.string() }))
    .query(async ({ input, ctx }) => {
      // 调用业务逻辑
      return await getTraceById(ctx.prisma, input.traceId);
    }),
});

// 业务逻辑 (features/traces)
export async function getTraceById(
  prisma: PrismaClient,
  traceId: string
) {
  const trace = await prisma.trace.findUnique({
    where: { id: traceId },
    include: {
      observations: true,
      scores: true,
    },
  });
  
  // 业务规则：计算总费用
  const totalCost = calculateTotalCost(trace);
  
  return {
    ...trace,
    totalCost,
  };
}
```

### 4.2 下层依赖（数据访问层）

**交互方式**：
```
Feature Module
  ↓
Prisma Client (PostgreSQL)
Kysely Client (ClickHouse)
  ↓
数据库
```

**数据访问模式**：
- **Repository 模式**（部分）：封装数据访问
- **Active Record 模式**：Prisma ORM
- **Query Builder 模式**：Kysely

### 4.3 横向依赖（共享服务）

**共享资源**：
```
Feature Module
  ↓
Shared Utils (@langfuse/shared)
  - 类型定义
  - 工具函数
  - 常量
  - 验证器
```

---

## 5. 关键代码文件路径

### 5.1 Web 功能模块

```
web/src/features/
├── trace-graph-view/            # Traces 可视化
├── prompts/                     # Prompts 管理
├── datasets/                    # Datasets 管理
├── evals/                       # Evaluations
├── scores/                      # Scores
├── playground/                  # Playground
├── dashboard/                   # Dashboard
├── projects/                    # Projects
├── organizations/               # Organizations
├── auth/                        # 认证
├── rbac/                        # RBAC
├── models/                      # Models
├── media/                       # Media
├── comments/                    # Comments
├── notifications/               # Notifications
├── experiments/                 # Experiments
├── automations/                 # Automations
├── batch-exports/               # Batch Exports
├── llm-api-key/                 # LLM API Keys
└── mcp/                         # MCP Integration
```

### 5.2 共享功能模块

```
packages/shared/src/
├── features/                    # 共享业务逻辑
├── domain/                      # 领域模型
├── server/                      # 服务器端工具
└── utils/                       # 工具函数
```

### 5.3 Worker 服务

```
worker/src/
├── services/                    # 业务服务
├── features/                    # 功能模块
│   ├── billing/                # 计费
│   └── evals/                  # 评估
└── queues/                      # 队列处理器
```

---

## 6. 技术实现细节

### 6.1 业务逻辑组织

#### Feature 模块结构
```
features/prompts/
├── index.ts                     # 导出
├── components/                  # UI 组件（如果有）
├── server/                      # 服务器端逻辑
│   ├── prompt-service.ts       # 业务服务
│   ├── prompt-repository.ts    # 数据访问（可选）
│   └── prompt-validation.ts    # 业务验证
└── utils/                       # 工具函数
```

#### 关注点分离
- **Controller**：tRPC Procedure（API 层）
- **Service**：业务逻辑（本层）
- **Repository**：数据访问（数据层）

### 6.2 领域驱动设计（DDD）元素

#### 实体（Entity）
```typescript
// 有唯一标识的对象
class Trace {
  constructor(
    public readonly id: string,
    public name: string,
    public projectId: string
  ) {}
  
  // 业务方法
  updateName(newName: string) {
    if (!newName || newName.length > 100) {
      throw new Error("Invalid name");
    }
    this.name = newName;
  }
}
```

#### 值对象（Value Object）
```typescript
// 无标识，只有值
class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: string
  ) {}
  
  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("Currency mismatch");
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

#### 聚合根（Aggregate Root）
```typescript
// Trace 是聚合根，管理 Observations
class Trace {
  private observations: Observation[] = [];
  
  addObservation(obs: Observation) {
    // 业务规则：验证 Observation
    if (obs.traceId !== this.id) {
      throw new Error("Invalid observation");
    }
    this.observations.push(obs);
  }
}
```

### 6.3 业务规则实现

#### 验证规则
```typescript
// Zod schema 用于输入验证
export const createPromptSchema = z.object({
  name: z.string().min(1).max(100),
  prompt: z.string().min(1),
  config: z.object({
    model: z.string(),
    temperature: z.number().min(0).max(2),
  }),
});

// 业务验证（更复杂的规则）
export function validatePromptBusiness(prompt: Prompt) {
  // 业务规则：Prompt 名称不能与已有重复
  // 业务规则：Config 必须与模型兼容
  // ...
}
```

#### 计算规则
```typescript
// 业务计算：总费用
export function calculateTotalCost(trace: Trace): number {
  return trace.observations.reduce((sum, obs) => {
    return sum + (obs.totalCost ?? 0);
  }, 0);
}

// 业务计算：Token 统计
export function calculateTotalTokens(trace: Trace) {
  return {
    promptTokens: sum(trace.observations, "promptTokens"),
    completionTokens: sum(trace.observations, "completionTokens"),
  };
}
```

### 6.4 事务处理

#### Prisma 事务
```typescript
export async function createDatasetWithItems(
  prisma: PrismaClient,
  data: CreateDatasetInput
) {
  return await prisma.$transaction(async (tx) => {
    // 创建 Dataset
    const dataset = await tx.dataset.create({
      data: {
        name: data.name,
        description: data.description,
        projectId: data.projectId,
      },
    });
    
    // 批量创建 DatasetItems
    await tx.datasetItem.createMany({
      data: data.items.map(item => ({
        datasetId: dataset.id,
        input: item.input,
        expectedOutput: item.expectedOutput,
      })),
    });
    
    return dataset;
  });
}
```

### 6.5 异步任务入队

#### 从 Web 入队
```typescript
import { Queue } from "bullmq";

export async function triggerEvaluation(
  evalConfig: EvalConfig,
  traceIds: string[]
) {
  const queue = new Queue("eval-queue");
  
  for (const traceId of traceIds) {
    await queue.add("evaluate-trace", {
      evalConfigId: evalConfig.id,
      traceId,
    });
  }
}
```

#### Worker 处理
```typescript
// worker/src/queues/evalQueue.ts
import { Worker } from "bullmq";

const worker = new Worker("eval-queue", async (job) => {
  const { evalConfigId, traceId } = job.data;
  
  // 执行评估业务逻辑
  await executeEvaluation(evalConfigId, traceId);
});
```

---

## 7. 设计模式和最佳实践

### 7.1 常用设计模式

#### Strategy Pattern（策略模式）
```typescript
// 不同的评分策略
interface ScoringStrategy {
  calculate(trace: Trace): number;
}

class AverageScoringStrategy implements ScoringStrategy {
  calculate(trace: Trace): number {
    return average(trace.scores.map(s => s.value));
  }
}

class WeightedScoringStrategy implements ScoringStrategy {
  calculate(trace: Trace): number {
    // 加权平均
  }
}
```

#### Factory Pattern（工厂模式）
```typescript
// 创建不同类型的导出器
class ExporterFactory {
  static create(type: "csv" | "json" | "xlsx") {
    switch (type) {
      case "csv":
        return new CsvExporter();
      case "json":
        return new JsonExporter();
      case "xlsx":
        return new XlsxExporter();
    }
  }
}
```

#### Observer Pattern（观察者模式）
```typescript
// 事件发布订阅
class TraceCreatedEvent {
  constructor(public trace: Trace) {}
}

// 订阅者
eventBus.on("trace.created", async (event: TraceCreatedEvent) => {
  // 触发自动评估
  await triggerAutoEvaluation(event.trace);
});
```

### 7.2 业务逻辑组织原则

#### 单一职责原则（SRP）
```typescript
// ❌ 不好：一个函数做太多事情
function createTraceAndNotify(data) {
  // 创建 Trace
  // 发送通知
  // 更新统计
  // 触发 Webhook
}

// ✅ 好：职责分离
function createTrace(data) { /* ... */ }
function sendNotification(trace) { /* ... */ }
function updateStats(trace) { /* ... */ }
function triggerWebhook(trace) { /* ... */ }
```

#### 依赖倒置原则（DIP）
```typescript
// 依赖抽象，不依赖具体实现
interface StorageProvider {
  upload(file: File): Promise<string>;
}

class S3StorageProvider implements StorageProvider {
  async upload(file: File) { /* ... */ }
}

// 业务逻辑依赖接口
class MediaService {
  constructor(private storage: StorageProvider) {}
  
  async uploadMedia(file: File) {
    const url = await this.storage.upload(file);
    return url;
  }
}
```

### 7.3 错误处理

#### 业务异常
```typescript
export class BusinessError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 400
  ) {
    super(message);
  }
}

export class TraceNotFoundError extends BusinessError {
  constructor(traceId: string) {
    super(`Trace ${traceId} not found`, "TRACE_NOT_FOUND", 404);
  }
}

// 使用
if (!trace) {
  throw new TraceNotFoundError(traceId);
}
```

### 7.4 数据验证

#### 多层验证
```typescript
// 1. 输入验证（Zod）- API 层
const input = createTraceSchema.parse(req.body);

// 2. 业务验证 - 业务层
function validateTraceBusiness(trace: Trace) {
  if (trace.name.includes("invalid")) {
    throw new BusinessError("Invalid trace name");
  }
}

// 3. 数据库约束 - 数据层
// Prisma schema 定义 unique, not null 等
```

---

## 8. 性能优化

### 8.1 查询优化
- 使用 Prisma `include` 减少 N+1 查询
- 使用 `select` 只查询需要的字段
- 批量操作使用 `createMany`, `updateMany`

### 8.2 缓存策略
- React Query 客户端缓存
- Redis 服务器端缓存
- Prompt 强缓存（不常变化）

### 8.3 异步处理
- 耗时操作入队到 Worker
- 使用 BullMQ 队列
- 避免阻塞 Web 请求

---

## 9. 测试策略

### 9.1 单元测试
```typescript
describe("calculateTotalCost", () => {
  it("should sum observation costs", () => {
    const trace = {
      observations: [
        { totalCost: 0.01 },
        { totalCost: 0.02 },
      ],
    };
    expect(calculateTotalCost(trace)).toBe(0.03);
  });
});
```

### 9.2 集成测试
```typescript
describe("Trace Service", () => {
  it("should create trace with observations", async () => {
    const trace = await createTraceWithObservations({
      name: "Test",
      observations: [/* ... */],
    });
    expect(trace).toBeDefined();
    expect(trace.observations).toHaveLength(2);
  });
});
```

---

## 10. 总结

业务服务层是 Langfuse 的核心，包含所有业务规则和领域逻辑。通过清晰的模块划分、领域驱动设计和设计模式，实现了高内聚、低耦合的业务架构。

**关键特点**：
- ✅ 功能模块化组织
- ✅ 领域模型封装业务规则
- ✅ 类型安全的业务逻辑
- ✅ 清晰的分层和依赖
- ✅ 可测试性强

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0
