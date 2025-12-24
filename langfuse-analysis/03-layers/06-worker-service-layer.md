# Worker 服务层 (Worker Service Layer)

## 1. 层职责

Worker 服务层是 Langfuse 的异步处理引擎，负责：
- 处理后台异步任务（BullMQ 队列）
- 执行耗时的数据处理和计算
- 管理数据同步（PostgreSQL → ClickHouse）
- 执行数据集评估和批量处理
- 处理 webhook 和外部集成
- 执行定时任务和后台迁移

**架构定位**：独立的微服务，与 Web 服务解耦，通过 Redis/BullMQ 通信。

---

## 2. 服务结构

### 2.1 目录结构

```
worker/
├── src/
│   ├── app.ts                          # Express 应用
│   ├── index.ts                        # 入口文件
│   ├── database.ts                     # 数据库客户端
│   ├── env.ts                          # 环境变量
│   ├── initialize.ts                   # 初始化
│   ├── instrumentation.ts              # 监控
│   ├── middlewares.ts                  # Express 中间件
│   │
│   ├── api/                            # HTTP API
│   │   ├── health.ts                   # 健康检查
│   │   └── metrics.ts                  # Prometheus 指标
│   │
│   ├── queues/                         # BullMQ 队列
│   │   ├── index.ts                    # 队列注册
│   │   ├── ingestionQueue.ts           # 数据摄取队列
│   │   ├── evaluationQueue.ts          # 评估队列
│   │   ├── datasetQueue.ts             # 数据集队列
│   │   ├── webhookQueue.ts             # Webhook 队列
│   │   ├── batchExportQueue.ts         # 批量导出队列
│   │   └── clickhouseSyncQueue.ts      # ClickHouse 同步队列
│   │
│   ├── services/                       # 业务服务
│   │   ├── IngestionService.ts         # 数据摄取
│   │   ├── EvaluationService.ts        # 评估服务
│   │   ├── DatasetService.ts           # 数据集服务
│   │   ├── WebhookService.ts           # Webhook 服务
│   │   ├── ExportService.ts            # 导出服务
│   │   └── ClickHouseSyncService.ts    # ClickHouse 同步
│   │
│   ├── backgroundMigrations/           # 后台迁移
│   │   ├── index.ts
│   │   └── migrations/
│   │       ├── 001_backfill_costs.ts
│   │       └── 002_sync_old_data.ts
│   │
│   ├── utils/                          # 工具函数
│   │   ├── logger.ts
│   │   └── error-handler.ts
│   │
│   └── __tests__/                      # 测试
│       ├── queues/
│       └── services/
│
├── Dockerfile
├── entrypoint.sh
├── package.json
└── tsconfig.json
```

---

## 3. BullMQ 队列系统

### 3.1 队列架构

```
Web Service
    ↓ (添加任务)
Redis (BullMQ)
    ↓ (消费任务)
Worker Service
    ↓ (处理)
Database (PostgreSQL/ClickHouse)
```

### 3.2 队列注册（queues/index.ts）

```typescript
import { Queue, Worker, QueueScheduler } from 'bullmq';
import IORedis from 'ioredis';
import { env } from '../env';

// Redis 连接
const connection = new IORedis(env.REDIS_CONNECTION_STRING, {
  maxRetriesPerRequest: null,
});

// 队列列表
export const queues = {
  ingestion: new Queue('ingestion', { connection }),
  evaluation: new Queue('evaluation', { connection }),
  dataset: new Queue('dataset', { connection }),
  webhook: new Queue('webhook', { connection }),
  batchExport: new Queue('batch-export', { connection }),
  clickhouseSync: new Queue('clickhouse-sync', { connection }),
};

// 启动所有 Workers
export function startWorkers() {
  // Ingestion Worker
  new Worker('ingestion', ingestionProcessor, {
    connection,
    concurrency: 10,
  });

  // Evaluation Worker
  new Worker('evaluation', evaluationProcessor, {
    connection,
    concurrency: 5,
  });

  // Dataset Worker
  new Worker('dataset', datasetProcessor, {
    connection,
    concurrency: 3,
  });

  // Webhook Worker
  new Worker('webhook', webhookProcessor, {
    connection,
    concurrency: 5,
  });

  // Batch Export Worker
  new Worker('batch-export', batchExportProcessor, {
    connection,
    concurrency: 2,
  });

  // ClickHouse Sync Worker
  new Worker('clickhouse-sync', clickhouseSyncProcessor, {
    connection,
    concurrency: 10,
  });

  logger.info('All workers started');
}
```

---

## 4. 核心队列详解

### 4.1 数据摄取队列（ingestionQueue.ts）

#### 职责
- 接收来自 SDK 的事件
- 验证和转换数据
- 写入 PostgreSQL
- 触发 ClickHouse 同步

#### 任务类型
```typescript
export type IngestionJob = {
  projectId: string;
  event: {
    type: 'trace-create' | 'generation-create' | 'span-create' | 'event-create' | 'score-create';
    body: Record<string, any>;
  };
  timestamp: Date;
};
```

#### 处理器
```typescript
import { Job } from 'bullmq';
import { IngestionService } from '../services/IngestionService';

export async function ingestionProcessor(job: Job<IngestionJob>) {
  const { projectId, event } = job.data;
  
  logger.info(`Processing ingestion job`, {
    jobId: job.id,
    projectId,
    eventType: event.type,
  });
  
  const service = new IngestionService();
  
  try {
    // 1. 验证事件
    const validated = await service.validateEvent(event);
    
    // 2. 写入 PostgreSQL
    const result = await service.writeToPostgres(projectId, validated);
    
    // 3. 触发 ClickHouse 同步
    await queues.clickhouseSync.add('sync', {
      projectId,
      entityType: event.type,
      entityId: result.id,
    });
    
    logger.info(`Ingestion job completed`, { jobId: job.id });
  } catch (error) {
    logger.error(`Ingestion job failed`, {
      jobId: job.id,
      error: error.message,
    });
    throw error;
  }
}
```

### 4.2 评估队列（evaluationQueue.ts）

#### 职责
- 执行数据集评估
- 运行评估器（Evaluators）
- 计算评分和指标
- 生成评估报告

#### 任务类型
```typescript
export type EvaluationJob = {
  datasetRunId: string;
  projectId: string;
  evaluatorConfigs: Array<{
    id: string;
    type: 'llm-as-judge' | 'code' | 'external';
    config: Record<string, any>;
  }>;
};
```

#### 处理器
```typescript
export async function evaluationProcessor(job: Job<EvaluationJob>) {
  const { datasetRunId, projectId, evaluatorConfigs } = job.data;
  
  const service = new EvaluationService();
  
  // 1. 获取 Dataset Run
  const datasetRun = await prisma.datasetRun.findUnique({
    where: { id: datasetRunId },
    include: {
      datasetRunItems: {
        include: {
          trace: {
            include: {
              observations: true,
            },
          },
        },
      },
    },
  });
  
  if (!datasetRun) {
    throw new Error(`DatasetRun ${datasetRunId} not found`);
  }
  
  // 2. 逐项评估
  for (const item of datasetRun.datasetRunItems) {
    for (const evaluatorConfig of evaluatorConfigs) {
      try {
        // 执行评估器
        const score = await service.runEvaluator(
          evaluatorConfig,
          item.trace
        );
        
        // 保存评分
        await prisma.score.create({
          data: {
            traceId: item.trace.id,
            name: evaluatorConfig.id,
            value: score.value,
            comment: score.comment,
            dataType: score.dataType,
            projectId,
          },
        });
      } catch (error) {
        logger.error(`Evaluator failed`, {
          evaluatorId: evaluatorConfig.id,
          traceId: item.trace.id,
          error: error.message,
        });
      }
    }
  }
  
  // 3. 更新状态
  await prisma.datasetRun.update({
    where: { id: datasetRunId },
    data: {
      status: 'COMPLETED',
      completedAt: new Date(),
    },
  });
  
  logger.info(`Evaluation completed`, {
    datasetRunId,
    itemCount: datasetRun.datasetRunItems.length,
  });
}
```

### 4.3 ClickHouse 同步队列（clickhouseSyncQueue.ts）

#### 职责
- 将 PostgreSQL 数据同步到 ClickHouse
- 处理批量写入
- 计算聚合指标
- 数据去重和清理

#### 任务类型
```typescript
export type ClickHouseSyncJob = {
  projectId: string;
  entityType: 'trace' | 'observation' | 'score';
  entityId: string;
};
```

#### 处理器
```typescript
export async function clickhouseSyncProcessor(job: Job<ClickHouseSyncJob>) {
  const { projectId, entityType, entityId } = job.data;
  
  const service = new ClickHouseSyncService();
  
  switch (entityType) {
    case 'observation':
      // 1. 从 PostgreSQL 读取
      const observation = await prisma.observation.findUnique({
        where: { id: entityId },
        include: {
          trace: true,
        },
      });
      
      if (!observation) {
        logger.warn(`Observation ${entityId} not found`);
        return;
      }
      
      // 2. 转换为 ClickHouse 格式
      const chObservation = service.transformObservation(observation);
      
      // 3. 写入 ClickHouse
      await clickhouseClient
        .insertInto('observations')
        .values(chObservation)
        .execute();
      
      logger.info(`Synced observation to ClickHouse`, {
        observationId: entityId,
      });
      break;
    
    case 'trace':
      // 类似处理
      break;
    
    case 'score':
      // 类似处理
      break;
  }
}
```

### 4.4 Webhook 队列（webhookQueue.ts）

#### 职责
- 发送 Webhook 通知
- 重试失败的请求
- 记录 Webhook 历史

#### 任务类型
```typescript
export type WebhookJob = {
  projectId: string;
  event: {
    type: 'trace.created' | 'score.created' | 'evaluation.completed';
    data: Record<string, any>;
  };
  webhookUrl: string;
  signature: string;
};
```

#### 处理器
```typescript
import axios from 'axios';

export async function webhookProcessor(job: Job<WebhookJob>) {
  const { projectId, event, webhookUrl, signature } = job.data;
  
  try {
    const response = await axios.post(webhookUrl, event, {
      headers: {
        'Content-Type': 'application/json',
        'X-Langfuse-Signature': signature,
      },
      timeout: 10000, // 10s timeout
    });
    
    if (response.status >= 200 && response.status < 300) {
      logger.info(`Webhook sent successfully`, {
        projectId,
        eventType: event.type,
        webhookUrl,
      });
    } else {
      throw new Error(`Unexpected status ${response.status}`);
    }
  } catch (error) {
    logger.error(`Webhook failed`, {
      projectId,
      eventType: event.type,
      webhookUrl,
      error: error.message,
      attempt: job.attemptsMade,
    });
    
    // BullMQ 会自动重试
    throw error;
  }
}
```

### 4.5 批量导出队列（batchExportQueue.ts）

#### 职责
- 导出 Traces 到 CSV/JSON
- 生成报告
- 上传到 S3

#### 任务类型
```typescript
export type BatchExportJob = {
  projectId: string;
  exportId: string;
  format: 'csv' | 'json';
  filters: {
    startDate?: Date;
    endDate?: Date;
    traceIds?: string[];
  };
};
```

#### 处理器
```typescript
import { createObjectCsvWriter } from 'csv-writer';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

export async function batchExportProcessor(job: Job<BatchExportJob>) {
  const { projectId, exportId, format, filters } = job.data;
  
  const service = new ExportService();
  
  // 1. 查询数据
  const traces = await service.fetchTraces(projectId, filters);
  
  // 2. 生成文件
  let fileContent: Buffer;
  let contentType: string;
  
  if (format === 'csv') {
    const csvWriter = createObjectCsvWriter({
      path: `/tmp/${exportId}.csv`,
      header: [
        { id: 'id', title: 'ID' },
        { id: 'name', title: 'Name' },
        { id: 'timestamp', title: 'Timestamp' },
        // ...
      ],
    });
    
    await csvWriter.writeRecords(traces);
    fileContent = await fs.readFile(`/tmp/${exportId}.csv`);
    contentType = 'text/csv';
  } else {
    fileContent = Buffer.from(JSON.stringify(traces, null, 2));
    contentType = 'application/json';
  }
  
  // 3. 上传到 S3
  const s3Client = new S3Client({});
  await s3Client.send(
    new PutObjectCommand({
      Bucket: env.S3_BUCKET_NAME,
      Key: `exports/${projectId}/${exportId}.${format}`,
      Body: fileContent,
      ContentType: contentType,
    })
  );
  
  // 4. 更新状态
  await prisma.batchExport.update({
    where: { id: exportId },
    data: {
      status: 'COMPLETED',
      url: `s3://.../${exportId}.${format}`,
      completedAt: new Date(),
    },
  });
  
  logger.info(`Export completed`, {
    exportId,
    traceCount: traces.length,
  });
}
```

---

## 5. 后台迁移（backgroundMigrations/）

### 5.1 迁移系统

#### 职责
- 执行耗时的数据迁移
- 不阻塞正常服务
- 支持断点续传

#### 迁移示例：成本回填
**路径**：`worker/src/backgroundMigrations/migrations/001_backfill_costs.ts`

```typescript
export class BackfillCostsMigration {
  async run() {
    const batchSize = 1000;
    let offset = 0;
    
    while (true) {
      // 1. 分批读取 Observations
      const observations = await prisma.observation.findMany({
        where: {
          calculatedTotalCost: null, // 未计算成本的
        },
        include: {
          trace: {
            include: {
              project: true,
            },
          },
        },
        take: batchSize,
        skip: offset,
      });
      
      if (observations.length === 0) {
        break; // 完成
      }
      
      // 2. 计算成本
      for (const obs of observations) {
        const model = await prisma.model.findFirst({
          where: {
            projectId: obs.trace.project.id,
            modelName: obs.model,
          },
        });
        
        if (model) {
          const cost = calculateObservationCost(obs, model);
          
          await prisma.observation.update({
            where: { id: obs.id },
            data: {
              calculatedInputCost: cost.inputCost,
              calculatedOutputCost: cost.outputCost,
              calculatedTotalCost: cost.totalCost,
            },
          });
        }
      }
      
      logger.info(`Processed ${observations.length} observations`);
      offset += batchSize;
      
      // 3. 限流（避免过载）
      await sleep(1000); // 休息 1 秒
    }
    
    logger.info(`Cost backfill completed`);
  }
}
```

---

## 6. 业务服务详解

### 6.1 数据摄取服务（IngestionService.ts）

```typescript
export class IngestionService {
  // 验证事件
  async validateEvent(event: any): Promise<ValidatedEvent> {
    switch (event.type) {
      case 'trace-create':
        return traceCreateSchema.parse(event.body);
      case 'generation-create':
        return generationCreateSchema.parse(event.body);
      // ...
      default:
        throw new InvalidRequestError(`Unknown event type: ${event.type}`);
    }
  }

  // 写入 PostgreSQL
  async writeToPostgres(
    projectId: string,
    event: ValidatedEvent
  ): Promise<{ id: string }> {
    switch (event.type) {
      case 'trace-create':
        const trace = await prisma.trace.upsert({
          where: { id: event.id ?? cuid() },
          create: {
            projectId,
            name: event.name,
            metadata: event.metadata,
            // ...
          },
          update: {
            name: event.name,
            metadata: event.metadata,
            // ...
          },
        });
        return { id: trace.id };
      
      case 'generation-create':
        const generation = await prisma.observation.create({
          data: {
            type: 'GENERATION',
            traceId: event.traceId,
            name: event.name,
            model: event.model,
            input: event.input,
            output: event.output,
            // ...
          },
        });
        return { id: generation.id };
      
      // ...
    }
  }
}
```

### 6.2 评估服务（EvaluationService.ts）

```typescript
export class EvaluationService {
  // 运行评估器
  async runEvaluator(
    config: EvaluatorConfig,
    trace: TraceWithObservations
  ): Promise<Score> {
    switch (config.type) {
      case 'llm-as-judge':
        return this.runLLMEvaluator(config, trace);
      
      case 'code':
        return this.runCodeEvaluator(config, trace);
      
      case 'external':
        return this.runExternalEvaluator(config, trace);
      
      default:
        throw new Error(`Unknown evaluator type: ${config.type}`);
    }
  }

  // LLM 评估器
  private async runLLMEvaluator(
    config: LLMEvaluatorConfig,
    trace: TraceWithObservations
  ): Promise<Score> {
    // 1. 构造提示词
    const prompt = this.buildEvaluationPrompt(config, trace);
    
    // 2. 调用 LLM
    const response = await callLLM({
      model: config.model,
      messages: [{ role: 'user', content: prompt }],
    });
    
    // 3. 解析结果
    const score = this.parseEvaluationResponse(response, config.scoreType);
    
    return score;
  }

  // 代码评估器
  private async runCodeEvaluator(
    config: CodeEvaluatorConfig,
    trace: TraceWithObservations
  ): Promise<Score> {
    // 执行用户提供的评估代码
    const fn = new Function('trace', config.code);
    const result = fn(trace);
    
    return {
      value: result.value,
      comment: result.comment,
      dataType: config.scoreType,
    };
  }
}
```

---

## 7. 错误处理和重试

### 7.1 重试策略

**BullMQ 重试配置**：
```typescript
new Worker('ingestion', ingestionProcessor, {
  connection,
  concurrency: 10,
  
  // 重试配置
  settings: {
    backoff: {
      type: 'exponential',
      delay: 1000, // 初始延迟 1s
    },
    attempts: 3, // 最多重试 3 次
  },
});
```

### 7.2 错误监控

```typescript
worker.on('failed', (job, error) => {
  logger.error(`Job failed`, {
    jobId: job.id,
    queue: job.queueName,
    error: error.message,
    attemptsMade: job.attemptsMade,
  });
  
  // 发送到监控系统（如 Sentry）
  Sentry.captureException(error, {
    extra: {
      jobId: job.id,
      queue: job.queueName,
    },
  });
});
```

---

## 8. HTTP API（健康检查和指标）

### 8.1 健康检查（api/health.ts）

```typescript
import { Router } from 'express';

const router = Router();

router.get('/health', async (req, res) => {
  try {
    // 检查数据库连接
    await prisma.$queryRaw`SELECT 1`;
    
    // 检查 Redis 连接
    await redis.ping();
    
    // 检查队列状态
    const queueHealth = await checkQueuesHealth();
    
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      queues: queueHealth,
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message,
    });
  }
});

export default router;
```

### 8.2 Prometheus 指标（api/metrics.ts）

```typescript
import { Router } from 'express';
import { Registry, Counter, Gauge } from 'prom-client';

const router = Router();
const register = new Registry();

// 定义指标
const jobsProcessed = new Counter({
  name: 'worker_jobs_processed_total',
  help: 'Total number of jobs processed',
  labelNames: ['queue', 'status'],
  registers: [register],
});

const jobDuration = new Gauge({
  name: 'worker_job_duration_seconds',
  help: 'Job processing duration',
  labelNames: ['queue'],
  registers: [register],
});

// 暴露指标端点
router.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

export default router;
```

---

## 9. 部署和扩展

### 9.1 Docker 部署

**Dockerfile**：
```dockerfile
FROM node:24-alpine

WORKDIR /app

# 安装依赖
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# 复制代码
COPY . .

# 构建
RUN pnpm run build

# 入口
CMD ["node", "dist/index.js"]
```

### 9.2 水平扩展

**优势**：
- 多个 Worker 实例可以并行处理任务
- BullMQ 自动负载均衡
- 无状态设计，易于扩展

**示例**：
```yaml
# docker-compose.yml
services:
  worker:
    image: langfuse-worker
    deploy:
      replicas: 3 # 启动 3 个 Worker 实例
    environment:
      - REDIS_CONNECTION_STRING=redis://redis:6379
      - DATABASE_URL=postgresql://...
```

### 9.3 资源配置

**CPU 密集型任务**（如评估）：
- 增加 CPU 资源
- 降低并发数

**I/O 密集型任务**（如同步）：
- 增加并发数
- 优化数据库连接池

---

## 10. 监控和可观测性

### 10.1 日志

**结构化日志**：
```typescript
logger.info('Processing job', {
  jobId: job.id,
  queue: job.queueName,
  projectId: job.data.projectId,
  duration: Date.now() - startTime,
});
```

### 10.2 指标

**关键指标**：
- 任务处理速率
- 任务失败率
- 队列长度
- 处理延迟

### 10.3 OpenTelemetry

**Trace 追踪**：
```typescript
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('worker');

export async function ingestionProcessor(job: Job<IngestionJob>) {
  return tracer.startActiveSpan('ingestion-job', async (span) => {
    span.setAttribute('job.id', job.id);
    span.setAttribute('project.id', job.data.projectId);
    
    try {
      // 处理任务
      await processJob(job);
      
      span.setStatus({ code: SpanStatusCode.OK });
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

---

## 11. 关键文件路径总结

```
worker/
├── src/
│   ├── index.ts                        # 入口文件
│   ├── app.ts                          # Express 应用
│   ├── database.ts                     # 数据库客户端
│   │
│   ├── queues/                         # BullMQ 队列
│   │   ├── index.ts                    # 队列注册
│   │   ├── ingestionQueue.ts           # 数据摄取队列
│   │   ├── evaluationQueue.ts          # 评估队列
│   │   ├── clickhouseSyncQueue.ts      # ClickHouse 同步
│   │   └── webhookQueue.ts             # Webhook 队列
│   │
│   ├── services/                       # 业务服务
│   │   ├── IngestionService.ts
│   │   ├── EvaluationService.ts
│   │   └── ClickHouseSyncService.ts
│   │
│   ├── backgroundMigrations/           # 后台迁移
│   │   └── migrations/
│   │
│   └── api/                            # HTTP API
│       ├── health.ts                   # 健康检查
│       └── metrics.ts                  # Prometheus 指标
│
└── Dockerfile
```

---

## 12. 总结

Worker 服务层是 Langfuse 的异步处理引擎，通过 BullMQ 队列系统实现了高效的后台任务处理。它与 Web 服务解耦，支持水平扩展，并提供完善的监控和错误处理机制。

**关键特点**：
- ✅ 异步任务处理（BullMQ）
- ✅ 微服务解耦（独立部署）
- ✅ 水平扩展（多实例）
- ✅ 可靠性保证（重试机制）
- ✅ 完善的监控（日志 + 指标 + Trace）

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0
