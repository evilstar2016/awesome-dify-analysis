# 共享包层 (Shared Packages Layer)

## 1. 层职责

共享包层是 Langfuse 的核心代码复用层，负责：
- 提供跨服务（web/worker）的共享代码
- 定义统一的类型系统和接口
- 实现通用工具函数和辅助类
- 提供数据库访问抽象
- 管理加密和安全相关功能
- 定义常量和配置

**架构定位**：横向基础层，为 Web 和 Worker 服务提供共享能力。

---

## 2. 包结构

### 2.1 主要包

| 包名 | 说明 | 依赖者 |
|-----|------|--------|
| **@langfuse/shared** | 核心共享包 | web, worker |
| **@langfuse/config-eslint** | ESLint 配置 | web, worker, shared |
| **@langfuse/config-typescript** | TypeScript 配置 | web, worker, shared |

### 2.2 shared 包目录结构

```
packages/shared/
├── src/
│   ├── constants.ts              # 全局常量
│   ├── db.ts                     # Prisma 客户端
│   ├── env.ts                    # 环境变量
│   ├── eventsTable.ts            # ClickHouse Events 表
│   ├── observationsTable.ts      # ClickHouse Observations 表
│   ├── types.ts                  # 通用类型
│   ├── index.ts                  # 导出入口
│   │
│   ├── domain/                   # 领域模型
│   │   ├── traces.ts             # Trace 领域逻辑
│   │   ├── observations.ts       # Observation 领域逻辑
│   │   ├── scores.ts             # Score 领域逻辑
│   │   └── prompts.ts            # Prompt 领域逻辑
│   │
│   ├── encryption/               # 加密功能
│   │   ├── index.ts
│   │   └── encryption.ts         # 加密/解密工具
│   │
│   ├── errors/                   # 错误类
│   │   ├── index.ts
│   │   ├── LangfuseNotFoundError.ts
│   │   ├── InvalidRequestError.ts
│   │   └── ...
│   │
│   ├── features/                 # 共享特性模块
│   │   ├── auth/                 # 认证相关
│   │   ├── cost-calculation/     # 成本计算
│   │   ├── tokenization/         # Token 计数
│   │   ├── rate-limiting/        # 速率限制
│   │   ├── ingestion/            # 数据摄取
│   │   └── ...
│   │
│   ├── interfaces/               # 接口定义
│   │   ├── index.ts
│   │   ├── ApiDefinitions.ts     # API 接口
│   │   └── ...
│   │
│   ├── server/                   # 服务端工具
│   │   ├── auth/                 # 认证工具
│   │   ├── redis/                # Redis 工具
│   │   ├── clickhouse/           # ClickHouse 工具
│   │   └── ...
│   │
│   ├── tableDefinitions/         # 数据表定义
│   │   ├── index.ts
│   │   ├── ObservationsTable.ts
│   │   ├── TracesTable.ts
│   │   └── ...
│   │
│   └── utils/                    # 工具函数
│       ├── index.ts
│       ├── logger.ts             # 日志工具
│       ├── validation.ts         # 校验工具
│       └── ...
│
├── prisma/                       # Prisma Schema
│   ├── schema.prisma
│   └── migrations/
│
├── clickhouse/                   # ClickHouse 配置
│   ├── migrations/
│   └── scripts/
│
├── scripts/                      # 数据库脚本
│   ├── cleanup.sql
│   └── seeder/
│
├── package.json
└── tsconfig.json
```

---

## 3. 核心模块详解

### 3.1 类型系统（types.ts）

#### 核心类型定义

```typescript
// API 响应类型
export type ApiResponse<T> = {
  data: T;
  meta?: {
    page?: number;
    limit?: number;
    totalItems?: number;
    totalPages?: number;
  };
};

// Trace 类型
export type TraceWithDetails = Trace & {
  observations: Observation[];
  scores: Score[];
  session?: Session | null;
};

// Observation 类型
export type ObservationWithCost = Observation & {
  calculatedTotalCost: number | null;
  calculatedInputCost: number | null;
  calculatedOutputCost: number | null;
};

// 评分类型
export type ScoreDataType = 'NUMERIC' | 'CATEGORICAL' | 'BOOLEAN';

// 提示词配置
export type PromptConfig = {
  modelName: string;
  temperature?: number;
  maxTokens?: number;
  topP?: number;
};
```

#### Zod Schemas（运行时验证）

```typescript
import { z } from 'zod';

export const traceCreateSchema = z.object({
  id: z.string().optional(),
  name: z.string().optional(),
  userId: z.string().optional(),
  metadata: z.record(z.any()).optional(),
  release: z.string().optional(),
  version: z.string().optional(),
});

export type TraceCreate = z.infer<typeof traceCreateSchema>;
```

### 3.2 常量（constants.ts）

```typescript
// Observation 类型
export const OBSERVATION_TYPES = [
  'GENERATION',
  'SPAN',
  'EVENT',
] as const;

// 评分数据类型
export const SCORE_DATA_TYPES = [
  'NUMERIC',
  'CATEGORICAL',
  'BOOLEAN',
] as const;

// 模型 Provider
export const MODEL_PROVIDERS = [
  'openai',
  'anthropic',
  'azure',
  'google',
] as const;

// 默认限制
export const DEFAULT_PAGE_SIZE = 50;
export const MAX_PAGE_SIZE = 1000;

// Token 价格（USD per 1M tokens）
export const DEFAULT_TOKEN_COSTS = {
  'gpt-4': { input: 30, output: 60 },
  'gpt-3.5-turbo': { input: 0.5, output: 1.5 },
  'claude-3-opus': { input: 15, output: 75 },
};
```

### 3.3 领域模型（domain/）

#### Trace 领域逻辑

**路径**：`packages/shared/src/domain/traces.ts`

```typescript
export class TraceDomain {
  // 计算 Trace 的总成本
  static calculateTotalCost(trace: TraceWithDetails): number {
    return trace.observations.reduce(
      (sum, obs) => sum + (obs.calculatedTotalCost ?? 0),
      0
    );
  }

  // 计算 Trace 的总 Token 数
  static calculateTotalTokens(trace: TraceWithDetails): {
    input: number;
    output: number;
    total: number;
  } {
    // ...
  }

  // 获取 Trace 的时间范围
  static getTimeRange(trace: TraceWithDetails): {
    start: Date;
    end: Date;
    duration: number;
  } {
    // ...
  }
}
```

#### Observation 领域逻辑

**路径**：`packages/shared/src/domain/observations.ts`

```typescript
export class ObservationDomain {
  // 计算 Observation 的成本
  static calculateCost(
    observation: Observation,
    model: Model
  ): {
    inputCost: number;
    outputCost: number;
    totalCost: number;
  } {
    const inputCost =
      (observation.inputTokens ?? 0) * model.inputPrice / 1_000_000;
    const outputCost =
      (observation.outputTokens ?? 0) * model.outputPrice / 1_000_000;
    
    return {
      inputCost,
      outputCost,
      totalCost: inputCost + outputCost,
    };
  }

  // 计算延迟
  static calculateLatency(observation: Observation): number {
    if (!observation.startTime || !observation.endTime) return 0;
    return observation.endTime.getTime() - observation.startTime.getTime();
  }
}
```

### 3.4 加密模块（encryption/）

**路径**：`packages/shared/src/encryption/encryption.ts`

```typescript
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = process.env.ENCRYPTION_KEY; // 32 bytes

export function encrypt(text: string): {
  encrypted: string;
  iv: string;
  tag: string;
} {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, Buffer.from(KEY, 'hex'), iv);
  
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const tag = cipher.getAuthTag();
  
  return {
    encrypted,
    iv: iv.toString('hex'),
    tag: tag.toString('hex'),
  };
}

export function decrypt(encrypted: string, iv: string, tag: string): string {
  const decipher = crypto.createDecipheriv(
    ALGORITHM,
    Buffer.from(KEY, 'hex'),
    Buffer.from(iv, 'hex')
  );
  
  decipher.setAuthTag(Buffer.from(tag, 'hex'));
  
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return decrypted;
}
```

**使用场景**：
- API Key 加密存储
- LLM API Key 加密
- 敏感配置加密

### 3.5 错误类（errors/）

**错误类层次**：

```
Error (标准 JS 错误)
  └─ LangfuseError (基类)
      ├─ LangfuseNotFoundError (404)
      ├─ InvalidRequestError (400)
      ├─ UnauthorizedError (401)
      ├─ ForbiddenError (403)
      ├─ MethodNotAllowedError (405)
      ├─ TooManyRequestsError (429)
      └─ InternalServerError (500)
```

**示例**：

```typescript
export class LangfuseError extends Error {
  public readonly httpCode: number;
  
  constructor(message: string, httpCode: number = 500) {
    super(message);
    this.name = this.constructor.name;
    this.httpCode = httpCode;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class LangfuseNotFoundError extends LangfuseError {
  constructor(message: string = 'Resource not found') {
    super(message, 404);
  }
}

export class InvalidRequestError extends LangfuseError {
  constructor(message: string = 'Invalid request') {
    super(message, 400);
  }
}
```

---

## 4. 共享特性模块（features/）

### 4.1 成本计算（cost-calculation/）

**路径**：`packages/shared/src/features/cost-calculation/`

```typescript
export function calculateObservationCost(
  observation: Observation,
  model: Model
): number {
  const inputCost = (observation.inputTokens ?? 0) * model.inputPrice / 1_000_000;
  const outputCost = (observation.outputTokens ?? 0) * model.outputPrice / 1_000_000;
  
  return inputCost + outputCost;
}

export function calculateTraceCost(trace: TraceWithDetails): number {
  return trace.observations.reduce(
    (sum, obs) => sum + (obs.calculatedTotalCost ?? 0),
    0
  );
}
```

### 4.2 Token 计数（tokenization/）

**路径**：`packages/shared/src/features/tokenization/`

**支持的 Tokenizer**：
- `gpt-4` / `gpt-3.5-turbo` → `cl100k_base`
- `gpt-3` → `p50k_base`
- Claude → 估算（1 token ≈ 4 chars）

```typescript
import { encoding_for_model } from 'tiktoken';

export function countTokens(text: string, model: string): number {
  try {
    const encoder = encoding_for_model(model);
    const tokens = encoder.encode(text);
    encoder.free();
    return tokens.length;
  } catch (error) {
    // Fallback：估算
    return Math.ceil(text.length / 4);
  }
}
```

### 4.3 速率限制（rate-limiting/）

**路径**：`packages/shared/src/features/rate-limiting/`

**基于 Redis 的滑动窗口算法**：

```typescript
import { redis } from '@langfuse/shared/src/server/redis';

export async function checkRateLimit(
  key: string,
  limit: number,
  windowSeconds: number
): Promise<{
  allowed: boolean;
  remaining: number;
  resetAt: Date;
}> {
  const now = Date.now();
  const windowStart = now - windowSeconds * 1000;
  
  // 移除过期记录
  await redis.zremrangebyscore(key, 0, windowStart);
  
  // 获取当前窗口内的请求数
  const count = await redis.zcard(key);
  
  if (count < limit) {
    // 添加新请求
    await redis.zadd(key, now, `${now}:${Math.random()}`);
    await redis.expire(key, windowSeconds);
    
    return {
      allowed: true,
      remaining: limit - count - 1,
      resetAt: new Date(now + windowSeconds * 1000),
    };
  }
  
  return {
    allowed: false,
    remaining: 0,
    resetAt: new Date(now + windowSeconds * 1000),
  };
}
```

### 4.4 数据摄取（ingestion/）

**路径**：`packages/shared/src/features/ingestion/`

**功能**：
- 接收来自 SDK 的事件
- 验证事件格式
- 写入数据库（Prisma + ClickHouse）
- 异步处理（通过 BullMQ）

```typescript
export async function ingestEvent(
  projectId: string,
  event: IngestionEvent
): Promise<void> {
  // 1. 验证
  const validated = ingestionEventSchema.parse(event);
  
  // 2. 写入 PostgreSQL（元数据）
  const trace = await prisma.trace.upsert({
    where: { id: validated.traceId },
    create: { /* ... */ },
    update: { /* ... */ },
  });
  
  // 3. 异步写入 ClickHouse（分析数据）
  await ingestionQueue.add('write-to-clickhouse', {
    projectId,
    event: validated,
  });
}
```

### 4.5 认证（auth/）

**路径**：`packages/shared/src/features/auth/`

```typescript
export async function verifyApiKey(
  hashedKey: string
): Promise<ApiKey | null> {
  return prisma.apiKey.findUnique({
    where: { hashedKey },
    include: {
      project: {
        include: {
          organization: true,
        },
      },
    },
  });
}

export async function verifyUserSession(
  sessionToken: string
): Promise<User | null> {
  const session = await prisma.session.findUnique({
    where: { sessionToken },
    include: { user: true },
  });
  
  if (!session || session.expires < new Date()) {
    return null;
  }
  
  return session.user;
}
```

---

## 5. 服务端工具（server/）

### 5.1 Redis 工具（server/redis/）

```typescript
import Redis from 'ioredis';

export const redis = new Redis(process.env.REDIS_CONNECTION_STRING);

// 缓存工具
export async function getOrSetCache<T>(
  key: string,
  ttlSeconds: number,
  fetcher: () => Promise<T>
): Promise<T> {
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }
  
  const data = await fetcher();
  await redis.setex(key, ttlSeconds, JSON.stringify(data));
  
  return data;
}
```

### 5.2 ClickHouse 工具（server/clickhouse/）

```typescript
import { Kysely } from 'kysely';
import { ClickHouseDialect } from 'kysely-clickhouse';

export const clickhouseClient = new Kysely({
  dialect: new ClickHouseDialect({
    url: process.env.CLICKHOUSE_URL,
  }),
});

// 批量插入工具
export async function batchInsertObservations(
  observations: Observation[]
): Promise<void> {
  await clickhouseClient
    .insertInto('observations')
    .values(observations)
    .execute();
}
```

---

## 6. 工具函数（utils/）

### 6.1 日志工具（utils/logger.ts）

```typescript
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  transport:
    process.env.NODE_ENV === 'development'
      ? { target: 'pino-pretty' }
      : undefined,
});
```

### 6.2 校验工具（utils/validation.ts）

```typescript
import { z } from 'zod';

export function validateWithZod<T>(
  schema: z.ZodSchema<T>,
  data: unknown
): T {
  try {
    return schema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new InvalidRequestError(
        `Validation error: ${error.errors.map(e => e.message).join(', ')}`
      );
    }
    throw error;
  }
}
```

### 6.3 日期工具（utils/dates.ts）

```typescript
export function formatDate(date: Date): string {
  return date.toISOString();
}

export function parseDate(dateString: string): Date {
  const date = new Date(dateString);
  if (isNaN(date.getTime())) {
    throw new InvalidRequestError('Invalid date format');
  }
  return date;
}

export function getDateRange(
  period: 'day' | 'week' | 'month'
): { start: Date; end: Date } {
  const end = new Date();
  const start = new Date();
  
  switch (period) {
    case 'day':
      start.setDate(start.getDate() - 1);
      break;
    case 'week':
      start.setDate(start.getDate() - 7);
      break;
    case 'month':
      start.setMonth(start.getMonth() - 1);
      break;
  }
  
  return { start, end };
}
```

---

## 7. 环境变量管理（env.ts）

**使用 Zod 进行类型安全的环境变量验证**：

```typescript
import { z } from 'zod';

const envSchema = z.object({
  // Database
  DATABASE_URL: z.string().url(),
  CLICKHOUSE_URL: z.string().url(),
  REDIS_CONNECTION_STRING: z.string(),
  
  // Encryption
  ENCRYPTION_KEY: z.string().length(64), // 32 bytes hex
  
  // Auth
  NEXTAUTH_URL: z.string().url(),
  NEXTAUTH_SECRET: z.string().min(32),
  
  // S3
  S3_ENDPOINT: z.string().url().optional(),
  S3_ACCESS_KEY_ID: z.string().optional(),
  S3_SECRET_ACCESS_KEY: z.string().optional(),
  S3_BUCKET_NAME: z.string().optional(),
  
  // Observability
  LANGFUSE_TRACING_SAMPLE_RATE: z.coerce.number().min(0).max(1).default(0.1),
});

export const env = envSchema.parse(process.env);
```

---

## 8. 数据库客户端（db.ts）

```typescript
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log:
      process.env.NODE_ENV === 'development'
        ? ['query', 'error', 'warn']
        : ['error'],
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

---

## 9. 表定义（tableDefinitions/）

**ClickHouse 表的 TypeScript 定义**：

**路径**：`packages/shared/src/tableDefinitions/ObservationsTable.ts`

```typescript
export interface ObservationsTable {
  id: string;
  trace_id: string;
  project_id: string;
  type: 'GENERATION' | 'SPAN' | 'EVENT';
  name: string;
  start_time: Date;
  end_time: Date | null;
  model: string | null;
  input_tokens: number | null;
  output_tokens: number | null;
  total_cost: number | null;
  metadata: string; // JSON string
}
```

---

## 10. 配置包

### 10.1 ESLint 配置（@langfuse/config-eslint）

**路径**：`packages/config-eslint/`

```javascript
// library.js
module.exports = {
  extends: ['eslint:recommended', 'plugin:@typescript-eslint/recommended'],
  parser: '@typescript-eslint/parser',
  plugins: ['@typescript-eslint'],
  rules: {
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/no-unused-vars': [
      'error',
      { argsIgnorePattern: '^_' },
    ],
  },
};

// next.js
module.exports = {
  extends: ['next/core-web-vitals', './library.js'],
};
```

### 10.2 TypeScript 配置（@langfuse/config-typescript）

**路径**：`packages/config-typescript/`

```json
// base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}

// nextjs.json
{
  "extends": "./base.json",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }]
  }
}
```

---

## 11. 测试支持

### 11.1 测试工具

**路径**：`packages/shared/src/test/`

```typescript
// test/helpers.ts
export function createMockTrace(overrides?: Partial<Trace>): Trace {
  return {
    id: 'trace-1',
    name: 'Test Trace',
    projectId: 'project-1',
    timestamp: new Date(),
    ...overrides,
  };
}

export function createMockObservation(
  overrides?: Partial<Observation>
): Observation {
  return {
    id: 'obs-1',
    traceId: 'trace-1',
    type: 'GENERATION',
    name: 'Test Generation',
    ...overrides,
  };
}
```

---

## 12. 构建和发布

### 12.1 构建配置

**package.json**：
```json
{
  "name": "@langfuse/shared",
  "version": "1.0.0",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "db:migrate": "prisma migrate dev",
    "db:deploy": "prisma migrate deploy",
    "db:generate": "prisma generate",
    "ch:migrate": "ts-node clickhouse/scripts/migrate.ts"
  },
  "dependencies": {
    "@prisma/client": "^6.17.1",
    "kysely": "^0.27.3",
    "kysely-clickhouse": "^0.3.0",
    "ioredis": "^5.3.2",
    "zod": "^3.25.62",
    "tiktoken": "^1.0.10"
  }
}
```

### 12.2 在其他包中使用

**web/package.json**：
```json
{
  "dependencies": {
    "@langfuse/shared": "workspace:*"
  }
}
```

**导入示例**：
```typescript
// Web 服务
import { prisma } from '@langfuse/shared/src/db';
import { TraceDomain } from '@langfuse/shared/src/domain/traces';
import { LangfuseNotFoundError } from '@langfuse/shared/src/errors';

// Worker 服务
import { clickhouseClient } from '@langfuse/shared/src/server/clickhouse';
import { logger } from '@langfuse/shared/src/utils/logger';
```

---

## 13. 版本管理

### 13.1 语义化版本

遵循 [SemVer](https://semver.org/)：
- **Major (1.x.x)**：Breaking changes
- **Minor (x.1.x)**：新特性（向后兼容）
- **Patch (x.x.1)**：Bug 修复

### 13.2 变更日志

**CHANGELOG.md**：
```markdown
# Changelog

## [1.2.0] - 2024-01-15
### Added
- 新增 Token 计数功能
- 新增速率限制模块

### Changed
- 优化成本计算性能
- 更新 Prisma schema

### Fixed
- 修复加密模块内存泄漏
```

---

## 14. 最佳实践

### 14.1 代码组织

✅ **好**：
```typescript
// 按功能模块组织
packages/shared/src/features/cost-calculation/
├── index.ts
├── calculate.ts
├── types.ts
└── __tests__/
    └── calculate.test.ts
```

❌ **不好**：
```typescript
// 所有代码放在一个文件
packages/shared/src/utils.ts (10000+ lines)
```

### 14.2 类型安全

✅ **好**：
```typescript
// 使用 Zod 运行时验证
const schema = z.object({
  name: z.string(),
  age: z.number().min(0),
});

const validated = schema.parse(input);
```

❌ **不好**：
```typescript
// 使用 any
function process(data: any) {
  // ...
}
```

### 14.3 错误处理

✅ **好**：
```typescript
if (!trace) {
  throw new LangfuseNotFoundError(`Trace ${id} not found`);
}
```

❌ **不好**：
```typescript
if (!trace) {
  throw new Error('Not found');
}
```

---

## 15. 关键文件路径总结

```
packages/shared/
├── src/
│   ├── constants.ts                     # 全局常量
│   ├── db.ts                            # Prisma 客户端
│   ├── env.ts                           # 环境变量
│   ├── types.ts                         # 通用类型
│   ├── index.ts                         # 导出入口
│   │
│   ├── domain/                          # 领域模型
│   │   ├── traces.ts
│   │   ├── observations.ts
│   │   └── scores.ts
│   │
│   ├── encryption/                      # 加密功能
│   │   └── encryption.ts
│   │
│   ├── errors/                          # 错误类
│   │   ├── LangfuseError.ts
│   │   └── LangfuseNotFoundError.ts
│   │
│   ├── features/                        # 共享特性
│   │   ├── cost-calculation/
│   │   ├── tokenization/
│   │   ├── rate-limiting/
│   │   ├── ingestion/
│   │   └── auth/
│   │
│   ├── server/                          # 服务端工具
│   │   ├── redis/
│   │   └── clickhouse/
│   │
│   └── utils/                           # 工具函数
│       ├── logger.ts
│       ├── validation.ts
│       └── dates.ts
│
└── prisma/
    └── schema.prisma                    # 数据库 Schema
```

---

## 16. 总结

共享包层是 Langfuse 的核心基础设施，通过提供统一的类型系统、工具函数、领域模型和数据访问抽象，实现了代码的高度复用和一致性。

**关键特点**：
- ✅ 类型安全（TypeScript + Zod）
- ✅ 模块化设计（按功能组织）
- ✅ 统一的错误处理
- ✅ 强大的工具库（加密、日志、验证）
- ✅ 跨服务共享（web + worker）

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0
