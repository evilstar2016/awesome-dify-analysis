# Scores 模块

## 概述

Scores 模块是 Langfuse 的核心评估系统，提供灵活的评分机制用于衡量 LLM 应用的质量。该模块支持多种评分类型（数值、分类、布尔）、多种评分来源（API、标注、模型评估），以及强大的聚合和分析能力。

## 核心概念

### 1. Score（评分）
- **定义**：对 Trace、Observation 或 Session 的质量评估
- **关联对象**：
  - `traceId`: 关联到 Trace
  - `observationId`: 关联到 Observation（可选）
  - `sessionId`: 关联到 Session
  - `datasetRunId`: 关联到 Dataset Run（实验评估）
- **时间戳**：`timestamp` 记录评分时间（可能与创建时间不同）

### 2. Score Config（评分配置）
- **定义**：预定义的评分模板，规定评分的类型、范围和类别
- **用途**：
  - 标准化评分标准
  - 验证评分值的有效性
  - 在 UI 中提供评分输入界面
- **生命周期**：可归档（`isArchived`）但不能删除（保持历史数据完整性）

### 3. Score Source（评分来源）
评分可来自多个来源，反映不同的评估方式：

| Source | 描述 | 用途 |
|--------|------|------|
| `API` | 通过 API 提交 | 程序化评分、自定义 evaluator |
| `ANNOTATION` | 人工标注 | 人工质量评估、标注队列 |
| `EVAL` | 模型评估 | Model-based evaluation（LLM as judge） |
| `FEEDBACK` | 用户反馈 | 终端用户评分（如👍👎） |

### 4. Score Data Type（评分数据类型）

#### NUMERIC（数值型）
- **用途**：连续型评分（如 0-1 的准确度、1-5 的满意度）
- **验证**：
  - `minValue`: 最小值（可选）
  - `maxValue`: 最大值（可选）
  - 值必须在范围内
- **聚合**：计算平均值、最小值、最大值

#### CATEGORICAL（分类型）
- **用途**：离散型评分（如"good", "bad", "neutral"）
- **验证**：
  - `categories`: 预定义类别列表
  - 每个类别包含 `label` 和 `value`（数值映射）
  - Label 和 value 必须唯一
- **聚合**：统计各类别的数量和比例

#### BOOLEAN（布尔型）
- **用途**：二值评分（如"正确/错误"、"相关/不相关"）
- **验证**：
  - 固定 2 个类别：
    - `{ label: "True", value: 1 }`
    - `{ label: "False", value: 0 }`
- **聚合**：统计 True/False 的数量（类似 CATEGORICAL）
- **特殊性**：是 CATEGORICAL 的特殊情况

## 数据模型

### PostgreSQL Schema

```sql
-- Score Config
CREATE TABLE score_configs (
  id VARCHAR PRIMARY KEY,
  project_id VARCHAR NOT NULL,
  name VARCHAR NOT NULL,
  data_type VARCHAR NOT NULL,           -- NUMERIC, CATEGORICAL, BOOLEAN
  is_archived BOOLEAN DEFAULT false,
  min_value FLOAT,                      -- NUMERIC only
  max_value FLOAT,                      -- NUMERIC only
  categories JSONB,                     -- CATEGORICAL/BOOLEAN only
  description TEXT,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  UNIQUE (id, project_id)
);

-- Categories JSON 格式
{
  "categories": [
    { "label": "Excellent", "value": 5 },
    { "label": "Good", "value": 4 },
    { "label": "Fair", "value": 3 },
    { "label": "Poor", "value": 2 },
    { "label": "Very Poor", "value": 1 }
  ]
}
```

### ClickHouse Schema

```sql
-- Scores 表
CREATE TABLE scores (
  id String,
  timestamp DateTime64(3),
  project_id String,
  environment String,                   -- 环境标识（如 production、staging）
  trace_id String,
  observation_id Nullable(String),
  session_id Nullable(String),          -- 通过 migration 添加
  dataset_run_id Nullable(String),      -- 通过 migration 添加
  name String,
  -- 数值型字段
  value Float64,                        -- NUMERIC 类型使用
  -- 分类型字段
  string_value Nullable(String),        -- CATEGORICAL/BOOLEAN 类型使用
  -- 配置关联
  config_id Nullable(String),
  data_type String,                     -- NUMERIC, CATEGORICAL, BOOLEAN
  -- 来源信息
  source String,                        -- API, ANNOTATION, EVAL, FEEDBACK
  author_user_id Nullable(String),      -- ANNOTATION 来源的标注者
  queue_id Nullable(String),            -- 标注队列 ID
  execution_trace_id Nullable(String),  -- EVAL 来源的执行 trace（通过 migration 添加）
  -- 元数据
  comment Nullable(String) CODEC(ZSTD(1)),
  metadata Map(LowCardinality(String), String), -- 通过 migration 添加
  -- 时间戳
  created_at DateTime64(3) DEFAULT now(),
  updated_at DateTime64(3) DEFAULT now(),
  -- 内部字段
  event_ts DateTime64(3),
  is_deleted UInt8,
  -- 索引
  INDEX idx_id id TYPE bloom_filter(0.001) GRANULARITY 1,
  INDEX idx_project_trace_observation (project_id, trace_id, observation_id) TYPE bloom_filter(0.001) GRANULARITY 1,
  INDEX idx_project_session (project_id, session_id) TYPE bloom_filter(0.001) GRANULARITY 1,
  INDEX idx_project_dataset_run (project_id, dataset_run_id) TYPE bloom_filter(0.001) GRANULARITY 1
) ENGINE = ReplacingMergeTree(event_ts, is_deleted)
Partition by toYYYYMM(timestamp)
PRIMARY KEY (
  project_id,
  toDate(timestamp),
  name
)
ORDER BY (
  project_id,
  toDate(timestamp),
  name,
  id
);
```

**存储策略**：
- **只存储在 ClickHouse**：无 PostgreSQL scores 表
- **关联数据在 PostgreSQL**：Score Configs、Annotation Queues
- **优势**：ClickHouse 的列式存储和压缩更适合大量评分数据

## 核心功能

### 1. Score Config 管理

#### 1.1 创建 Config

**数值型 Config**：
```typescript
// tRPC: scoreConfigs.create
{
  projectId: "proj-123",
  name: "accuracy",
  dataType: "NUMERIC",
  minValue: 0,
  maxValue: 1,
  description: "Model accuracy score (0-1)"
}
```

**分类型 Config**：
```typescript
{
  projectId: "proj-123",
  name: "quality",
  dataType: "CATEGORICAL",
  categories: [
    { label: "Excellent", value: 5 },
    { label: "Good", value: 4 },
    { label: "Fair", value: 3 },
    { label: "Poor", value: 2 },
    { label: "Very Poor", value: 1 }
  ],
  description: "Overall quality assessment"
}
```

**布尔型 Config**：
```typescript
{
  projectId: "proj-123",
  name: "is_correct",
  dataType: "BOOLEAN",
  categories: [
    { label: "True", value: 1 },
    { label: "False", value: 0 }
  ],
  description: "Correctness check"
}
```

#### 1.2 验证逻辑

**数值型验证**：
```typescript
if (dataType === "NUMERIC") {
  if (minValue !== null && maxValue !== null) {
    if (maxValue <= minValue) {
      throw new Error("maxValue must be greater than minValue");
    }
  }
}
```

**分类型验证**：
```typescript
// 检查唯一性
const uniqueLabels = new Set(categories.map(c => c.label));
const uniqueValues = new Set(categories.map(c => c.value));

if (uniqueLabels.size !== categories.length) {
  throw new Error("Category labels must be unique");
}
if (uniqueValues.size !== categories.length) {
  throw new Error("Category values must be unique");
}
```

**布尔型验证**：
```typescript
if (dataType === "BOOLEAN") {
  if (categories.length !== 2) {
    throw new Error("Boolean must have exactly 2 categories");
  }
  
  const expected = [
    { label: "True", value: 1 },
    { label: "False", value: 0 }
  ];
  
  if (!categoriesMatchExpected(categories, expected)) {
    throw new Error("Boolean categories must be True=1 and False=0");
  }
}
```

#### 1.3 更新 Config（包括归档）
```typescript
// tRPC: scoreConfigs.update
{
  projectId: "proj-123",
  id: "config-456",
  isArchived: true
}
```
- 归档后不可用于新评分
- 历史评分保留 `configId` 引用
- 不删除配置以保持数据完整性
- 更新操作通过 `update` procedure，可以修改 `isArchived`、`name`、`description`、`minValue`、`maxValue`、`categories` 等字段

### 2. Score 创建

#### 2.1 API Score（程序化评分）
```typescript
// SDK
langfuse.score({
  traceId: "trace-123",
  name: "accuracy",
  value: 0.95,
  comment: "High accuracy"
});

// 或关联到 observation
langfuse.score({
  traceId: "trace-123",
  observationId: "obs-456",
  name: "latency_acceptable",
  value: 1,  // boolean: 1 = True
  dataType: "BOOLEAN"
});
```

#### 2.2 Annotation Score（人工标注）
```typescript
// tRPC: scores.createAnnotationScore
{
  projectId: "proj-123",
  scoreTarget: {
    traceId: "trace-123",
    observationId: "obs-456"  // 可选
  },
  name: "quality",
  configId: "config-789",
  dataType: "CATEGORICAL",
  value: 4,
  stringValue: "Good",
  comment: "Well-structured response",
  queueId: "queue-012"  // 来自标注队列
}
```

**特点**：
- `source = "ANNOTATION"`
- `authorUserId` 记录标注者
- 可关联到 Annotation Queue
- 自动验证是否符合 config 规则

#### 2.3 Eval Score（模型评估）
```typescript
// 通过 Evaluator Job 创建
{
  traceId: "trace-123",
  name: "relevance",
  value: 0.87,
  source: "EVAL",
  executionTraceId: "eval-trace-345",  // 评估执行的 trace
  metadata: {
    model: "gpt-4",
    prompt: "Rate relevance 0-1"
  }
}
```

### 3. Score 查询

#### 3.1 列表查询
```typescript
// tRPC: scores.all
{
  projectId: "proj-123",
  filter: [
    {
      column: "name",
      operator: "any of",
      value: ["accuracy", "quality"],
      type: "stringOptions"
    },
    {
      column: "source",
      operator: "any of",
      value: ["ANNOTATION", "API"],
      type: "stringOptions"
    },
    {
      column: "value",
      operator: "greater than",
      value: 0.8,
      type: "number"
    }
  ],
  orderBy: [
    { column: "timestamp", order: "DESC" }
  ],
  page: 0,
  limit: 50
}
```

**返回数据**：
```typescript
{
  scores: [
    {
      id: "score-001",
      name: "accuracy",
      value: 0.95,
      stringValue: null,
      dataType: "NUMERIC",
      source: "API",
      traceId: "trace-123",
      traceName: "customer-greeting",
      traceUserId: "user-456",
      traceTags: ["production"],
      observationId: null,
      configId: "config-789",
      comment: "High accuracy",
      authorUserId: null,
      authorUserName: null,
      timestamp: "2024-12-15T10:30:00Z",
      hasMetadata: false
    },
    // ... more scores
  ]
}
```

#### 3.2 按 ID 查询
```typescript
// tRPC: scores.byId
{
  projectId: "proj-123",
  scoreId: "score-001"
}

// 返回完整 score（包含 metadata）
{
  id: "score-001",
  // ... all fields
  metadata: {
    model: "gpt-4",
    temperature: 0.7
  }
}
```

#### 3.3 筛选选项
```typescript
// tRPC: scores.filterOptions
{
  projectId: "proj-123",
  timestampFilter: [
    {
      column: "timestamp",
      operator: "greater than",
      value: "2024-12-01",
      type: "datetime"
    }
  ]
}

// 返回所有可用的筛选值
{
  name: [
    { value: "accuracy", count: 1250 },
    { value: "quality", count: 890 }
  ],
  tags: [
    { tag: "production", count: 1500 },
    { tag: "staging", count: 640 }
  ],
  traceName: [
    { value: "customer-greeting", count: 800 }
  ],
  userId: [
    { value: "user-456", count: 120 }
  ],
  stringValue: [
    { value: "Good", count: 450 },
    { value: "Excellent", count: 320 }
  ]
}
```

### 4. Score 更新与删除

#### 4.1 更新 Annotation Score
```typescript
// tRPC: scores.updateAnnotationScore
{
  projectId: "proj-123",
  id: "score-001",
  scoreTarget: {
    traceId: "trace-123"
  },
  name: "quality",
  configId: "config-789",
  value: 5,
  stringValue: "Excellent",
  comment: "Updated after review"
}
```

**特点**：
- 只能更新 ANNOTATION 来源的 scores
- 自动验证是否符合 config
- 更新 `authorUserId` 为当前用户
- 记录审计日志（before/after）

#### 4.2 删除 Score
```typescript
// tRPC: scores.deleteMany
{
  projectId: "proj-123",
  scoreIds: ["score-001", "score-002"],
  isBatchAction: false
}
```

**批量删除**：
```typescript
{
  projectId: "proj-123",
  query: {
    filter: [
      {
        column: "name",
        operator: "equals",
        value: "test_score"
      }
    ]
  },
  isBatchAction: true
}
```
- 使用队列异步处理（ScoreDeleteQueue）
- 需要 `traces:delete` 权限
- 需要 `trace-deletion` entitlement

### 5. Score 聚合

#### 5.1 聚合逻辑

**按 (name, source, dataType) 分组**：
```typescript
const key = `${name}-${source}-${dataType}`;
// 例如: "accuracy-API-NUMERIC"
```

**数值型聚合**：
```typescript
{
  type: "NUMERIC",
  values: [0.95, 0.87, 0.92, 0.89],
  average: 0.9075,
  comment: undefined,      // 多个 scores 时为 undefined
  id: undefined,
  hasMetadata: undefined
}
```

**分类型聚合**：
```typescript
{
  type: "CATEGORICAL",
  values: ["Good", "Excellent", "Good", "Fair", "Good"],
  valueCounts: [
    { value: "Good", count: 3 },
    { value: "Excellent", count: 1 },
    { value: "Fair", count: 1 }
  ],
  comment: undefined,
  id: undefined
}
```

**单个 score 的聚合**（保留额外信息）：
```typescript
{
  type: "NUMERIC",
  values: [0.95],
  average: 0.95,
  comment: "High accuracy",     // 单个时保留
  id: "score-001",              // 单个时保留
  hasMetadata: false,           // 单个时保留
  timestamp: "2024-12-15T10:30:00Z"
}
```

#### 5.2 使用场景

**Trace 详情页**：
```typescript
const trace = await getTraceById(traceId);
const scores = await getScoresForTrace(traceId);
const aggregated = aggregateScores(scores);

// 显示所有聚合后的 scores
Object.entries(aggregated).map(([key, aggregate]) => {
  const { name, source } = decomposeAggregateScoreKey(key);
  return (
    <ScoreDisplay
      name={name}
      source={source}
      aggregate={aggregate}
    />
  );
});
```

**Dataset Run 对比**：
```typescript
const run1Scores = await getScoresForDatasetRun(run1Id);
const run2Scores = await getScoresForDatasetRun(run2Id);

const agg1 = aggregateScores(run1Scores);
const agg2 = aggregateScores(run2Scores);

// 对比聚合结果
compareAggregates(agg1, agg2);
```

### 6. Score Columns（动态列）

用于在 Traces 表格中显示 scores 作为动态列。

#### 6.1 查询 Score Columns
```typescript
// tRPC: scores.getScoreColumns
{
  projectId: "proj-123",
  filter: [/* trace filters */],
  fromTimestamp: "2024-12-01",
  toTimestamp: "2024-12-15"
}

// 返回所有存在的 score 组合
{
  scoreColumns: [
    {
      key: "accuracy-API-NUMERIC",
      name: "accuracy",
      source: "API",
      dataType: "NUMERIC"
    },
    {
      key: "quality-ANNOTATION-CATEGORICAL",
      name: "quality",
      source: "ANNOTATION",
      dataType: "CATEGORICAL"
    }
  ]
}
```

#### 6.2 渲染动态列
```typescript
// 前端使用 score columns 创建表格列
scoreColumns.map(({ key, name, source, dataType }) => {
  return {
    id: key,
    header: getScoreLabelFromKey(key),  // "📊 accuracy (api)"
    cell: (trace) => {
      const scores = trace.scores;
      const aggregate = aggregateScores(
        scores.filter(s => 
          s.name === name && 
          s.source === source && 
          s.dataType === dataType
        )
      );
      return <ScoreCell aggregate={aggregate[key]} />;
    }
  };
});
```

## 技术架构

### API 层

#### tRPC Router: scores
位置：`web/src/server/api/routers/scores.ts`（791 行）

**主要 Procedures**：

**查询**：
- `all`: 列表查询（支持 filter、排序、分页）
- `byId`: 根据 ID 查询单个 score
- `countAll`: 统计 scores 数量
- `filterOptions`: 获取筛选选项
- `hasAny`: 检查项目是否有 scores
- `getScoreMetadataById`: 获取 score 的 metadata

**Annotation Scores**：
- `createAnnotationScore`: 创建标注评分
- `updateAnnotationScore`: 更新标注评分
- `deleteAnnotationScore`: 删除标注评分

**批量操作**：
- `deleteMany`: 批量删除 scores

**Score Columns**：
- `getScoreColumns`: 获取动态 score 列
- `getScoreKeysAndProps`: （已弃用）获取 score keys

#### tRPC Router: scoreConfigs
位置：`web/src/server/api/routers/scoreConfigs.ts`

**主要 Procedures**：
- `all`: 列表查询 configs（支持分页）
- `byId`: 根据 ID 查询 config
- `create`: 创建 config
- `update`: 更新 config（包括归档操作，通过 `isArchived` 字段）

#### REST API
位置：`web/src/features/public-api/server/scores.ts`

**端点**：
- `GET /api/public/scores` - 列出 scores
- `GET /api/public/scores/:scoreId` - 获取 score
- `POST /api/public/scores` - 创建 score
- `PATCH /api/public/scores/:scoreId` - 更新 score
- `DELETE /api/public/scores/:scoreId` - 删除 score
- `GET /api/public/score-configs` - 列出 configs
- `POST /api/public/score-configs` - 创建 config

### 服务层

#### Score Repository
位置：`packages/shared/src/server/repositories/scores.ts`

**主要函数**：

**查询**：
```typescript
getScoresUiTable(props): Score[]
getScoresUiCount(projectId, filter, orderBy): number
getScoreById(projectId, scoreId, source?): Score
getScoreMetadataById(projectId, scoreId): Metadata
hasAnyScore(projectId): boolean
getScoresForTraces(props): Score[]
getScoresForObservations(props): Score[]
getScoresForSessions(props): Score[]
getScoresForDatasetRuns(props): Score[]
```

**创建/更新**：
```typescript
upsertScore(scoreData): void
searchExistingAnnotationScore(projectId, ...): Score | null
```

**删除**：
```typescript
deleteScores(projectId, scoreIds): void
deleteScoresByProjectId(projectId): void
deleteScoresByTraceIds(projectId, traceIds): void
```

**筛选选项**：
```typescript
getScoreNames(projectId, timestampFilter): { name, count }[]
getScoreStringValues(projectId, timestampFilter): { value, count }[]
getScoresGroupedByNameSourceType(projectId, filter, from, to): { name, source, dataType, count }[]
```

**聚合查询**：
```typescript
getNumericScoresGroupedByName(props): AggregatedScore[]
getCategoricalScoresGroupedByName(props): AggregatedScore[]
getAggregatedScoresForPrompts(props): AggregatedScore[]
```

**注意**：Score Config 的验证逻辑在 shared package 的 domain 层（`packages/shared/src/domain/score-configs.ts`）中实现，而非单独的 service 文件。

### 数据访问层

#### ClickHouse Repository
位置：`packages/shared/src/server/repositories/scores.ts`

**表名**：`scores`

**主要列**：
- `id`, `project_id`, `environment`, `trace_id`, `observation_id`
- `session_id`, `dataset_run_id`, `name`
- `value` (Float64), `string_value` (Nullable(String))
- `config_id`, `data_type`, `source`
- `author_user_id`, `queue_id`, `execution_trace_id`
- `comment`, `metadata` (Map)
- `timestamp`, `created_at`, `updated_at`
- `event_ts`, `is_deleted` (内部字段)

**查询优化**：
```sql
-- PRIMARY KEY & ORDER BY 索引
PRIMARY KEY (project_id, toDate(timestamp), name)
ORDER BY (project_id, toDate(timestamp), name, id)

-- 查询优化示例
SELECT *
FROM scores
WHERE project_id = ?
  AND trace_id IN (?, ?, ...)
  AND name = ?
ORDER BY timestamp DESC
LIMIT 100;

-- 利用 bloom_filter 索引
SELECT *
FROM scores
WHERE project_id = ?
  AND session_id = ?
LIMIT 100;
```

### 聚合逻辑

#### Aggregate Service
位置：`web/src/features/scores/lib/aggregateScores.ts`

**核心函数**：
```typescript
// 组合 key
composeAggregateScoreKey({ name, source, dataType }): string
// 例如: "accuracy-API-NUMERIC"

// 解析 key
decomposeAggregateScoreKey(key): { name, source, dataType }

// 标准化名称（替换 - 和 . 为 _）
normalizeScoreName(name): string

// 聚合 scores
aggregateScores(scores): ScoreAggregate

// 确定聚合类型（Boolean → Categorical）
resolveAggregateType(dataType): "NUMERIC" | "CATEGORICAL"

// 生成 label
getScoreLabelFromKey(key): string
// 例如: "📊 accuracy (api)"
```

**聚合算法**：
```typescript
export const aggregateScores = (scores: Score[]): ScoreAggregate => {
  // 1. 按 (name, source, dataType) 分组
  const grouped = scores.reduce((acc, score) => {
    const key = composeAggregateScoreKey({
      name: score.name,
      source: score.source,
      dataType: score.dataType
    });
    if (!acc[key]) acc[key] = [];
    acc[key].push(score);
    return acc;
  }, {});

  // 2. 对每组计算聚合
  return Object.entries(grouped).reduce((acc, [key, scores]) => {
    const aggregateType = resolveAggregateType(scores[0].dataType);
    
    if (aggregateType === "NUMERIC") {
      const values = scores.map(s => s.value ?? 0);
      const average = values.reduce((a, b) => a + b, 0) / values.length;
      acc[key] = {
        type: "NUMERIC",
        values,
        average,
        // 单个 score 时保留额外信息
        comment: values.length === 1 ? scores[0].comment : undefined,
        id: values.length === 1 ? scores[0].id : undefined,
        hasMetadata: values.length === 1 ? scores[0].hasMetadata : undefined,
        timestamp: values.length === 1 ? scores[0].timestamp : undefined
      };
    } else {
      const values = scores.map(s => s.stringValue ?? "n/a");
      const valueCounts = values.reduce((acc, value) => {
        acc[value] = (acc[value] || 0) + 1;
        return acc;
      }, {});
      acc[key] = {
        type: "CATEGORICAL",
        values,
        valueCounts: Object.entries(valueCounts).map(([value, count]) => ({
          value,
          count
        })),
        comment: values.length === 1 ? scores[0].comment : undefined,
        id: values.length === 1 ? scores[0].id : undefined,
        hasMetadata: values.length === 1 ? scores[0].hasMetadata : undefined,
        timestamp: values.length === 1 ? scores[0].timestamp : undefined
      };
    }
    
    return acc;
  }, {});
};
```

### 队列处理

#### Score Delete Queue
位置：`worker/src/queues/scoreDelete.ts`

**队列任务**：
```typescript
type ScoreDeletePayload = {
  projectId: string;
  scoreIds: string[];
};

// Worker 处理
const worker = new Worker(queueName, async (job: Job<ScoreDeletePayload>) => {
  const { projectId, scoreIds } = job.data.payload;
  
  // 批量删除
  await deleteScores(projectId, scoreIds);
  
  logger.info(`Deleted ${scoreIds.length} scores from project ${projectId}`);
});
```

## 前端集成

### 组件结构
```
web/src/features/scores/
├── components/
│   ├── ScoreTable.tsx              # Scores 列表表格
│   ├── ScoreCell.tsx               # Score 单元格显示
│   ├── ScoreDisplay.tsx            # Score 详情显示
│   ├── ScoreForm.tsx               # Score 创建/编辑表单
│   ├── ScoreConfigForm.tsx         # Config 创建/编辑表单
│   ├── ScoreAggregateDisplay.tsx   # 聚合 scores 显示
│   └── ScoreFilterPanel.tsx        # 筛选面板
├── hooks/
│   ├── useScores.ts                # Scores 查询 hook
│   ├── useScoreConfigs.ts          # Configs 查询 hook
│   └── useScoreAggregation.ts      # 聚合 hook
├── lib/
│   ├── aggregateScores.ts          # 聚合逻辑
│   ├── scoreColumns.ts             # 动态列生成
│   └── helpers.ts                  # 辅助函数
└── types.ts                        # TypeScript 类型
```

### 页面路由
```
/project/[projectId]/scores                      # Scores 列表
/project/[projectId]/scores/[scoreId]            # Score 详情
/project/[projectId]/settings/scores             # Score Configs 管理
/project/[projectId]/settings/scores/[configId]  # Config 详情
```

### tRPC Hooks 使用示例

#### 查询 Scores
```typescript
const { data, isLoading } = api.scores.all.useQuery({
  projectId,
  filter: [
    {
      column: "name",
      operator: "any of",
      value: ["accuracy", "quality"],
      type: "stringOptions"
    }
  ],
  orderBy: [
    { column: "timestamp", order: "DESC" }
  ],
  page: 0,
  limit: 50
});

// 渲染表格
<ScoreTable scores={data?.scores} />
```

#### 创建 Annotation Score
```typescript
const createMutation = api.scores.createAnnotationScore.useMutation({
  onSuccess: () => {
    toast.success("Score created");
    utils.scores.all.invalidate();
  }
});

const handleSubmit = (values) => {
  createMutation.mutate({
    projectId,
    scoreTarget: {
      traceId: selectedTrace.id
    },
    name: values.name,
    configId: values.configId,
    dataType: config.dataType,
    value: values.value,
    stringValue: values.stringValue,
    comment: values.comment
  });
};
```

#### 聚合显示
```typescript
import { aggregateScores } from "@/src/features/scores/lib/aggregateScores";

const trace = api.traces.byId.useQuery({ traceId });
const aggregated = aggregateScores(trace.data?.scores || []);

// 渲染聚合结果
{Object.entries(aggregated).map(([key, aggregate]) => {
  const { name, source } = decomposeAggregateScoreKey(key);
  return (
    <ScoreAggregateDisplay
      key={key}
      name={name}
      source={source}
      aggregate={aggregate}
    />
  );
})}
```

## 使用场景

### 场景 1：API 评分（自定义 Evaluator）

**流程**：
1. SDK 创建 trace
2. 自定义 evaluator 计算分数
3. SDK 提交 score

**代码示例**：
```typescript
// 1. 创建 trace
const trace = langfuse.trace({
  name: "customer-support",
  input: { question: "How to reset password?" },
  output: { answer: "Click 'Forgot Password'..." }
});

// 2. 自定义评估
const accuracy = evaluateAccuracy(trace.output, expectedAnswer);
const relevance = evaluateRelevance(trace.output, trace.input);

// 3. 提交 scores
trace.score({
  name: "accuracy",
  value: accuracy,
  comment: `Similarity: ${accuracy}`
});

trace.score({
  name: "relevance",
  value: relevance
});
```

### 场景 2：人工标注（Annotation Queue）

**流程**：
1. 管理员创建标注队列并分配 score configs
2. 标注者从队列获取待标注 items
3. 标注者对每个 item 评分
4. Scores 记录 `queueId` 和 `authorUserId`

**代码示例**：
```typescript
// 创建标注队列
const queue = await prisma.annotationQueue.create({
  data: {
    name: "Quality Review",
    projectId,
    scoreConfigIds: ["config-quality", "config-correctness"]
  }
});

// 标注者获取 item
const item = await getNextAnnotationQueueItem(queueId, userId);

// 标注者评分
await api.scores.createAnnotationScore.mutate({
  projectId,
  scoreTarget: {
    traceId: item.objectId  // 假设 objectType = "trace"
  },
  name: "quality",
  configId: "config-quality",
  dataType: "CATEGORICAL",
  value: 4,
  stringValue: "Good",
  comment: "Clear and helpful",
  queueId: queue.id
});

// 标记 item 为已完成
await completeAnnotationQueueItem(item.id);
```

### 场景 3：Model-based Evaluation

**流程**：
1. Evaluator Job 配置（定期或触发）
2. Job 执行：遍历 traces 并调用 LLM as judge
3. LLM 返回评分
4. Job 创建 scores（source = "EVAL"）

**代码示例**：
```typescript
// Job 配置
const evaluatorConfig = {
  name: "Relevance Evaluator",
  projectId,
  targetType: "trace",
  scoreConfigId: "config-relevance",
  llmConfig: {
    model: "gpt-4",
    temperature: 0,
    prompt: `Rate the relevance of the answer to the question.
Question: {{input.question}}
Answer: {{output.answer}}

Return a score from 0 (not relevant) to 1 (highly relevant).`
  }
};

// Job 执行逻辑（简化）
async function runEvaluator(traceId: string) {
  const trace = await getTraceById(traceId);
  
  // 创建 evaluation trace
  const evalTrace = langfuse.trace({
    name: "evaluation-execution",
    input: { traceId, evaluatorName: "Relevance Evaluator" }
  });
  
  // 调用 LLM
  const prompt = renderPrompt(evaluatorConfig.llmConfig.prompt, trace);
  const response = await callLLM(prompt);
  const relevance = parseFloat(response);
  
  evalTrace.update({ output: { relevance } });
  
  // 创建 score
  await langfuse.score({
    traceId: trace.id,
    name: "relevance",
    value: relevance,
    source: "EVAL",
    configId: "config-relevance",
    executionTraceId: evalTrace.id  // 关联 evaluation trace
  });
}
```

### 场景 4：用户反馈

**流程**：
1. 在应用中显示👍👎按钮
2. 用户点击后通过 API 提交 score
3. Score 记录 `source = "FEEDBACK"`

**代码示例**：
```typescript
// 前端（用户应用）
const handleFeedback = async (isPositive: boolean) => {
  await fetch("/api/public/scores", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${apiKey}`
    },
    body: JSON.stringify({
      traceId: currentTraceId,
      name: "user_feedback",
      dataType: "BOOLEAN",
      value: isPositive ? 1 : 0,
      stringValue: isPositive ? "True" : "False",
      source: "FEEDBACK"
    })
  });
  
  toast.success("Thank you for your feedback!");
};

// 渲染
<div>
  <button onClick={() => handleFeedback(true)}>👍</button>
  <button onClick={() => handleFeedback(false)}>👎</button>
</div>
```

### 场景 5：Dataset Run 评估分析

**流程**：
1. 执行 dataset run
2. 对每个 run item 评分
3. 聚合 scores 对比不同 runs

**代码示例**：
```typescript
// 获取 runs 的 scores
const run1Scores = await getScoresForDatasetRun(run1Id);
const run2Scores = await getScoresForDatasetRun(run2Id);

// 聚合
const agg1 = aggregateScores(run1Scores);
const agg2 = aggregateScores(run2Scores);

// 对比
const comparison = compareAggregates(agg1, agg2);

// 渲染对比表
<ComparisonTable>
  <Row>
    <Cell>Metric</Cell>
    <Cell>Baseline</Cell>
    <Cell>Optimized</Cell>
    <Cell>Δ</Cell>
  </Row>
  {comparison.map(({ name, baseline, optimized, delta }) => (
    <Row key={name}>
      <Cell>{name}</Cell>
      <Cell>{baseline.average.toFixed(2)}</Cell>
      <Cell>{optimized.average.toFixed(2)}</Cell>
      <Cell style={{ color: delta > 0 ? "green" : "red" }}>
        {delta > 0 ? "↑" : "↓"} {Math.abs(delta * 100).toFixed(1)}%
      </Cell>
    </Row>
  ))}
</ComparisonTable>
```

## 性能优化

### 1. ClickHouse 查询优化

#### 索引设计
```sql
PRIMARY KEY (project_id, toDate(timestamp), name)
ORDER BY (project_id, toDate(timestamp), name, id)
```
- `project_id`: 主要分区键，加速租户隔离
- `toDate(timestamp)`: 按日期分区，加速时间范围查询
- `name`: 加速按评分名称查询
- `id`: 确保排序唯一性

**Bloom Filter 索引**：
- `idx_id`: 加速按 ID 精确查询
- `idx_project_trace_observation`: 加速按项目、trace、observation 组合查询
- `idx_project_session`: 加速按项目和 session 查询
- `idx_project_dataset_run`: 加速按项目和 dataset run 查询

#### 查询优化
```typescript
// 批量查询 scores for multiple traces
const traceIds = ["trace-1", "trace-2", ...];

const scores = await clickhouse.query(`
  SELECT *
  FROM scores
  WHERE project_id = ?
    AND trace_id IN (${traceIds.join(',')})
  ORDER BY timestamp DESC
`);

// 使用 Map 快速查找
const scoresByTrace = scores.reduce((acc, score) => {
  if (!acc[score.trace_id]) acc[score.trace_id] = [];
  acc[score.trace_id].push(score);
  return acc;
}, {});
```

### 2. 聚合缓存

#### 前端缓存
```typescript
// 使用 React Query 缓存聚合结果
const { data: aggregated } = useQuery(
  ["scores-aggregated", traceId],
  async () => {
    const scores = await api.scores.all.fetch({ traceId });
    return aggregateScores(scores);
  },
  {
    staleTime: 60000,      // 1 分钟内不重新请求
    cacheTime: 300000      // 5 分钟后清除缓存
  }
);
```

#### 服务端缓存（Redis）
```typescript
const cacheKey = `scores:aggregated:${projectId}:${traceId}`;

// 尝试从缓存获取
const cached = await redis.get(cacheKey);
if (cached) return JSON.parse(cached);

// 查询并缓存
const scores = await getScoresForTrace(projectId, traceId);
const aggregated = aggregateScores(scores);

await redis.setex(cacheKey, 300, JSON.stringify(aggregated));  // 5 分钟 TTL

return aggregated;
```

### 3. 批量操作优化

#### 批量创建 Scores
```typescript
const BATCH_SIZE = 1000;

// 分批 upsert
for (let i = 0; i < scores.length; i += BATCH_SIZE) {
  const batch = scores.slice(i, i + BATCH_SIZE);
  
  await clickhouse.insert({
    table: "scores",
    values: batch,
    format: "JSONEachRow"
  });
}
```

#### 批量删除优化
```typescript
// 使用队列异步处理
await scoreDeleteQueue.add({
  projectId,
  scoreIds: [/* 大量 IDs */]
});

// Worker 中批量删除
const BATCH_SIZE = 500;

for (let i = 0; i < scoreIds.length; i += BATCH_SIZE) {
  const batch = scoreIds.slice(i, i + BATCH_SIZE);
  
  await clickhouse.command({
    query: `
      DELETE FROM scores
      WHERE project_id = ?
        AND id IN (${batch.map(() => '?').join(',')})
    `,
    query_params: [projectId, ...batch]
  });
}
```

### 4. 动态列优化

#### 延迟加载 Score Columns
```typescript
// 只加载可见的 score columns
const visibleColumns = useMemo(() => {
  return scoreColumns.filter(col => 
    selectedScoreNames.includes(col.name)
  );
}, [scoreColumns, selectedScoreNames]);

// 分页加载 traces 和 scores
const { data: traces } = api.traces.all.useInfiniteQuery({
  projectId,
  limit: 50,
  includeScores: true
});
```

## 最佳实践

### 1. Score Config 设计
- **使用描述性名称**：如 `response_quality` 而非 `score1`
- **添加描述**：说明评分标准和用途
- **定义范围**：数值型使用 minValue/maxValue 限制范围
- **预定义类别**：分类型明确定义所有可能的类别
- **考虑聚合**：设计时考虑如何聚合和对比

### 2. Score 命名规范
- **小写下划线**：使用 `snake_case`（如 `response_quality`）
- **避免特殊字符**：不使用 `-` 和 `.`（会被标准化为 `_`）
- **语义清晰**：名称应明确表达评分内容
- **一致性**：同一评分标准使用相同名称

### 3. Source 选择
- **API**：程序化评分、自定义 evaluator
- **ANNOTATION**：人工标注、质量审核
- **EVAL**：模型评估、LLM as judge
- **FEEDBACK**：用户反馈、终端用户评分

### 4. Config 使用
- **创建前规划**：设计好评分标准再创建 config
- **归档而非删除**：保持历史数据完整性
- **版本管理**：config 名称可包含版本（如 `quality_v2`）
- **文档化**：在 description 中详细说明评分标准

### 5. 性能考虑
- **批量操作**：大量 scores 使用批量 API
- **异步处理**：删除操作使用队列
- **索引友好**：查询时包含 `project_id` 和 `trace_id`
- **聚合缓存**：频繁访问的聚合结果使用缓存

## 常见问题

### Q1: Score 和 Score Config 的关系？
A: Score Config 是可选的模板：
- **有 Config**：Score 必须符合 config 规则（类型、范围、类别）
- **无 Config**：Score 可自由定义值，但缺少验证和标准化
- Config 提供 UI 界面（下拉选择、滑动条）
- Config 支持归档但不删除（保持历史数据完整性）

### Q2: 为什么 Boolean 是 Categorical 的特殊情况？
A: 设计考虑：
- Boolean 本质上是只有 2 个类别的 Categorical
- 聚合逻辑相同（统计数量而非平均）
- `resolveAggregateType("BOOLEAN")` 返回 `"CATEGORICAL"`
- 固定类别：`[{ label: "True", value: 1 }, { label: "False", value: 0 }]`

### Q3: Score 的 value 和 stringValue 如何选择？
A: 根据 dataType：
- **NUMERIC**：使用 `value`（Float），`stringValue = null`
- **CATEGORICAL**：使用 `stringValue`（类别 label），`value`（类别对应的数值）
- **BOOLEAN**：使用 `stringValue`（"True"/"False"），`value`（1/0）

### Q4: Annotation Score 和 API Score 有什么区别？
A: 主要区别：
| 特性 | API Score | Annotation Score |
|------|-----------|------------------|
| Source | `API` | `ANNOTATION` |
| 创建方式 | SDK/REST API | tRPC (UI) |
| authorUserId | 可选 | 必需（当前用户） |
| queueId | 无 | 可关联标注队列 |
| 更新 | 可通过 API | 需通过 `updateAnnotationScore` |
| 删除 | 批量删除 | 需通过 `deleteAnnotationScore` |

### Q5: 如何实现 Score 的版本管理？
A: 推荐方案：
- **Config 层面**：创建新 config（如 `quality_v1`, `quality_v2`）
- **Score 层面**：scores 记录 `configId`，关联到特定版本
- **查询时**：可按 `configId` 筛选特定版本的 scores
- **对比时**：对比不同 configId 的 scores

### Q6: Score 聚合的 key 为什么包含 source 和 dataType？
A: 精确区分：
- **同名不同 source**：`accuracy-API` vs `accuracy-ANNOTATION`
- **同名不同 dataType**：`quality-NUMERIC` vs `quality-CATEGORICAL`
- **避免冲突**：确保聚合时不会混淆不同类型的 scores
- **分别显示**：UI 中可分别展示不同来源/类型的评分

### Q7: 如何处理 Score 的时间戳？
A: 三个时间字段：
- **timestamp**：评分时间（业务时间，可回溯）
- **createdAt**：记录创建时间（系统时间）
- **updatedAt**：记录更新时间（系统时间）

使用建议：
- 查询时优先使用 `timestamp`（反映评分实际时间）
- 排序时使用 `timestamp`
- 审计时参考 `createdAt` 和 `updatedAt`

### Q8: Score 删除为什么需要 trace-deletion entitlement？
A: 权限控制：
- Score 是评估数据的核心部分
- 删除 scores 类似删除 traces（破坏性操作）
- 需要高级权限防止误删
- 批量删除使用队列确保可追踪

## 相关模块

- **Traces 模块**：Scores 主要关联到 Traces
- **Observations 模块**：Scores 可关联到特定 Observations
- **Datasets 模块**：Dataset Runs 使用 Scores 进行评估
- **Annotation Queues 模块**：人工标注创建 Annotation Scores
- **Evaluators 模块**：模型评估创建 Eval Scores
- **Dashboard 模块**：聚合展示 Scores 统计

## 目录结构

```
web/src/features/scores/
├── components/              # UI 组件
│   ├── ScoreTable.tsx
│   ├── ScoreCell.tsx
│   └── ...
├── hooks/                   # React Hooks
│   ├── useScores.ts
│   └── useScoreConfigs.ts
├── lib/                     # 核心逻辑
│   ├── aggregateScores.ts   # 聚合逻辑
│   ├── scoreColumns.ts      # 动态列
│   └── helpers.ts
└── types.ts                 # TypeScript 类型

web/src/server/api/routers/
├── scores.ts                # Scores tRPC Router (791 lines)
└── scoreConfigs.ts          # Score Configs tRPC Router

packages/shared/src/
├── domain/
│   └── scores.ts            # Score Domain 类型和验证
├── server/
│   └── repositories/
│       └── scores.ts        # Score Repository（ClickHouse 查询和操作）

worker/src/queues/
└── scoreDelete.ts           # 异步删除 worker
```
