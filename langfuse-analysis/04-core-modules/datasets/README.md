# Datasets & Evaluation 模块

## 概述

Datasets & Evaluation 是 Langfuse 的核心评估系统，提供了完整的测试数据管理和模型评估能力。该模块支持创建测试数据集、执行评估运行、对比不同版本的模型输出，是实现 LLM 应用质量保障的关键基础设施。

## 核心概念

### 1. Dataset（数据集）
- **定义**：测试数据的集合，包含多个 Dataset Items
- **用途**：存储黄金标准数据，用于模型评估和回归测试
- **组织方式**：支持文件夹层级结构（类似 Prompts 模块）
- **Schema 验证**：可定义 input/output 的 JSON Schema 进行数据验证

### 2. Dataset Item（数据项）
- **组成部分**：
  - `input`: 输入数据（JSON）
  - `expectedOutput`: 期望输出（JSON，可选）
  - `metadata`: 元数据（JSON）
  - `sourceTraceId`: 源 Trace ID（可从真实流量创建）
  - `sourceObservationId`: 源 Observation ID
- **版本管理**：使用 temporal table 机制（`validFrom`、`validTo`、`sysId`）
- **状态**：ACTIVE / ARCHIVED

### 3. Dataset Run（评估运行）
- **定义**：一次完整的数据集评估执行
- **包含信息**：
  - 运行名称和描述
  - 关联的数据集
  - 执行时间
  - 元数据（如模型配置、prompt 版本等）
- **唯一性约束**：`(datasetId, projectId, name)` 保证同名运行唯一

### 4. Dataset Run Item（运行项）
- **定义**：Dataset Run 中每个 Dataset Item 的执行记录
- **关联关系**：
  - `datasetItemId` → Dataset Item
  - `traceId` → 生成的 Trace
  - `observationId` → 具体的 Observation（可选）
- **数据存储**：
  - PostgreSQL：关系数据（ID、关联关系）
  - ClickHouse：反规范化快照（dataset run 和 item 的 denormalized fields）
  - 性能指标（延迟、成本）在查询时从 observations/traces 表实时计算

## 数据模型

### PostgreSQL Schema

```sql
-- Dataset
CREATE TABLE datasets (
  id VARCHAR,
  project_id VARCHAR,
  name VARCHAR UNIQUE,
  description TEXT,
  metadata JSONB,
  remote_experiment_url VARCHAR,       -- URL for remote experiment trigger
  remote_experiment_payload JSONB,     -- Default payload for remote experiment
  input_schema JSONB,                  -- JSON Schema for input validation
  expected_output_schema JSONB,        -- JSON Schema for output validation
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  PRIMARY KEY (id, project_id),
  UNIQUE (project_id, name)
);

-- Dataset Item (Temporal Table)
CREATE TABLE dataset_items (
  id VARCHAR,
  project_id VARCHAR,
  dataset_id VARCHAR,
  status VARCHAR,                  -- ACTIVE, ARCHIVED
  input JSONB,
  expected_output JSONB,
  metadata JSONB,
  source_trace_id VARCHAR,
  source_observation_id VARCHAR,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  -- Temporal columns
  sys_id VARCHAR DEFAULT (gen_random_uuid()),  -- Database-generated UUID
  valid_from TIMESTAMP DEFAULT NOW(),
  valid_to TIMESTAMP,
  is_deleted BOOLEAN DEFAULT FALSE,
  PRIMARY KEY (id, project_id, valid_from)
);

-- Dataset Run
CREATE TABLE dataset_runs (
  id VARCHAR,
  project_id VARCHAR,
  dataset_id VARCHAR,
  name VARCHAR,
  description TEXT,
  metadata JSONB,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  PRIMARY KEY (id, project_id),
  UNIQUE (dataset_id, project_id, name)
);

-- Dataset Run Item
CREATE TABLE dataset_run_items (
  id VARCHAR,
  project_id VARCHAR,
  dataset_run_id VARCHAR,
  dataset_item_id VARCHAR,
  trace_id VARCHAR,
  observation_id VARCHAR,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  PRIMARY KEY (id, project_id)
);
```

### ClickHouse Table

```sql
-- Dataset Run Items (denormalized snapshot)
CREATE TABLE dataset_run_items_rmt (
  -- primary identifiers
  id String,
  project_id String,
  dataset_run_id String,
  dataset_item_id String,
  dataset_id String,
  trace_id String,
  observation_id Nullable(String),

  -- error field
  error Nullable(String),

  -- timestamps
  created_at DateTime64(3) DEFAULT now(),
  updated_at DateTime64(3) DEFAULT now(),

  -- denormalized immutable dataset run fields
  dataset_run_name String,
  dataset_run_description Nullable(String),
  dataset_run_metadata Map(LowCardinality(String), String),
  dataset_run_created_at DateTime64(3),

  -- denormalized dataset item fields (mutable, but snapshots are relevant)
  dataset_item_input Nullable(String) CODEC(ZSTD(3)), -- json
  dataset_item_expected_output Nullable(String) CODEC(ZSTD(3)), -- json
  dataset_item_metadata Map(LowCardinality(String), String),

  -- clickhouse engine fields
  event_ts DateTime64(3),
  is_deleted UInt8,

  -- For dataset item lookups
  INDEX idx_dataset_item dataset_item_id TYPE bloom_filter(0.001) GRANULARITY 1
) ENGINE = ReplacingMergeTree(event_ts, is_deleted)
ORDER BY (project_id, dataset_id, dataset_run_id, id);
```

**注意**：
- 该表不直接存储 latency 和 cost 指标
- 这些指标在查询时从 `observations` 和 `traces` 表实时计算
- 使用 `ReplacingMergeTree` 引擎支持更新操作
- 存储了 dataset run 和 dataset item 的反规范化快照数据

## 核心功能

### 1. Dataset 管理

#### 1.1 创建 Dataset
```typescript
// tRPC API: datasetRouter.createDataset
{
  name: "customer-support/greeting",
  description: "Customer greeting test cases",
  metadata: { version: "1.0" },
  inputSchema: {
    type: "object",
    properties: {
      userName: { type: "string" }
    }
  },
  expectedOutputSchema: {
    type: "object",
    properties: {
      greeting: { type: "string" }
    }
  }
}
```

#### 1.2 文件夹视图
- 支持 `/` 分隔的层级结构（如 `customer-support/greeting`）
- 使用 SQL `SPLIT_PART` 和 `CTE` 实现文件夹视图
- 显示文件夹图标和 dataset 图标的混合列表

#### 1.3 Dataset 查询
```typescript
// tRPC API: datasetRouter.allDatasets
{
  projectId,
  pathPrefix: "customer-support",  // 当前文件夹路径
  searchQuery: "greeting",         // 搜索关键词
  page: 0,
  limit: 50
}
```

### 2. Dataset Item 管理

#### 2.1 创建单个 Item
```typescript
// tRPC API: datasetRouter.createDatasetItem
{
  datasetId: "ds-123",
  input: { userName: "Alice" },
  expectedOutput: { greeting: "Hello Alice!" },
  metadata: { difficulty: "easy" }
}
```

#### 2.2 批量创建 Items
```typescript
// tRPC API: datasetRouter.createManyDatasetItems
{
  projectId,
  items: [
    {
      datasetId: "ds-123",
      input: JSON.stringify({ userName: "Alice" }),
      expectedOutput: JSON.stringify({ greeting: "Hello Alice!" }),
      metadata: JSON.stringify({ difficulty: "easy" })
    },
    // ... more items
  ]
}
```

**验证机制**：
- Schema 验证（如果 dataset 定义了 schema）
- 数据清理（sanitize control characters）
- 批量验证报告（返回所有验证错误）

#### 2.3 从 Trace 创建 Item
```typescript
// 从生产流量创建测试数据
{
  datasetId: "ds-123",
  sourceTraceId: "trace-456",
  sourceObservationId: "obs-789"  // 可选
}
```

**自动提取**：
- Input 从 Trace/Observation 的 input 字段提取
- ExpectedOutput 从 output 字段提取
- Metadata 保留原始 metadata

#### 2.4 Item 版本管理
使用 temporal table 机制：
- `validFrom`: 版本生效时间
- `validTo`: 版本失效时间（NULL 表示当前版本）
- `isDeleted`: 软删除标记
- `sysId`: 系统生成的唯一 ID

**查询历史版本**：
```typescript
// 查询 Item 在某个时间点的版本
SELECT * FROM dataset_items
WHERE id = 'item-123'
  AND project_id = 'proj-456'
  AND valid_from <= '2024-01-15'
  AND (valid_to IS NULL OR valid_to > '2024-01-15');
```

### 3. Dataset Run 执行

#### 3.1 创建 Run
```typescript
// tRPC: datasets.createRun
{
  datasetId: "ds-123",
  name: "gpt-4-baseline",
  description: "GPT-4 baseline evaluation",
  metadata: {
    modelName: "gpt-4",
    temperature: 0.7,
    promptVersion: "v2"
  }
}
```

#### 3.2 关联 Trace 到 Run Item
```typescript
// SDK: langfuse.trace()
const trace = langfuse.trace({
  name: "dataset-run-item",
  metadata: {
    datasetItemId: "item-123",
    datasetRunId: "run-456"
  }
});

// 或使用 public API
POST /api/public/dataset-run-items
{
  runId: "run-456",
  datasetItemId: "item-123",
  traceId: "trace-789",
  observationId: "obs-012"  // 可选
}
```

#### 3.3 查询 Run 结果
```typescript
// tRPC: datasets.runsByDatasetId
{
  datasetId: "ds-123",
  filter: [
    {
      column: "avgLatency",
      operator: "less than",
      value: 1000,
      type: "number"
    }
  ],
  page: 0,
  limit: 50
}
```

**返回数据**：
- Run 基础信息（name, description, metadata）
- 聚合指标（平均延迟、总成本、item 数量）
- Scores 聚合（按 score name 分组）
- Trace scores（从关联的 traces 提取）

### 4. Run Items 查询与对比

#### 4.1 按 Run ID 查询
```typescript
// tRPC: datasets.runItemsByRunId
{
  projectId,
  datasetId: "ds-123",
  datasetRunId: "run-456",
  filter: [
    {
      column: "latency",
      operator: "greater than",
      value: 500
    }
  ],
  page: 0,
  limit: 50
}
```

#### 4.2 按 Item ID 查询（跨 Run 对比）
```typescript
// tRPC: datasets.runItemsByItemId
{
  projectId,
  datasetId: "ds-123",
  datasetItemId: "item-123",
  datasetRunIds: ["run-456", "run-789"],  // 对比两个 runs
  page: 0,
  limit: 50
}
```

**返回格式**：
```typescript
{
  totalRunItems: 2,
  runItems: [
    {
      id: "dri-001",
      datasetItemId: "item-123",
      datasetRunId: "run-456",
      traceId: "trace-789",
      observationId: "obs-012",
      // 从 traces 表 enriched 的数据
      trace: {
        name: "customer-greeting",
        input: { userName: "Alice" },
        output: { greeting: "Hello Alice!" },
        latency: 450,
        totalCost: 0.002,
        scores: [
          { name: "correctness", value: 0.95 }
        ]
      }
    },
    {
      id: "dri-002",
      datasetItemId: "item-123",
      datasetRunId: "run-789",
      // ... similar structure
    }
  ]
}
```

#### 4.3 带 Run Data 的 Dataset Items 查询
```typescript
// tRPC: datasets.datasetItemsWithRunData
{
  projectId,
  datasetId: "ds-123",
  runIds: ["run-456", "run-789"],
  filterByRun: [
    {
      runId: "run-456",
      filters: [
        {
          column: "latency",
          operator: "less than",
          value: 1000
        }
      ]
    }
  ],
  page: 0,
  limit: 50
}
```

**查询逻辑**：
1. 在 ClickHouse 中筛选满足条件的 dataset item IDs
2. 批量查询这些 items 在所有 runs 中的 run items
3. Enrich trace 数据（input/output/scores）
4. 按 dataset item ID 聚合返回

## 技术架构

### API 层

#### tRPC Router
位置：`web/src/features/datasets/server/dataset-router.ts`（1955 行）

**主要 Procedures**：

**Dataset 管理**：
- `hasAny`: 检查项目是否有 dataset
- `allDatasets`: 查询 datasets（支持文件夹视图）
- `allDatasetsMetrics`: 批量获取 dataset 指标
- `byId`: 根据 ID 获取 dataset
- `createDataset`: 创建 dataset
- `updateDataset`: 更新 dataset
- `deleteDataset`: 删除 dataset（队列处理）

**Dataset Item 管理**：
- `baseDatasetItemByDatasetId`: 查询 dataset items
- `itemById`: 根据 ID 获取 item
- `createDatasetItem`: 创建 item
- `createManyDatasetItems`: 批量创建 items
- `updateDatasetItem`: 更新 item
- `deleteDatasetItem`: 删除 item（支持版本管理）

**Dataset Run 管理**：
- `runsByDatasetId`: 查询 dataset 的 runs
- `runsByDatasetIdMetrics`: 获取 runs 的聚合指标
- `runById`: 根据 ID 获取 run
- `createRun`: 创建 run
- `runFilterOptions`: 获取 run 筛选选项

**Dataset Run Item 查询**：
- `runItemsByRunId`: 按 run ID 查询 run items
- `runItemsByItemId`: 按 item ID 查询 run items（跨 run 对比）
- `datasetItemsWithRunData`: 带 run data 的 items 查询
- `countAllDatasetItems`: 统计 run items 数量

#### REST API
位置：`web/src/features/public-api/server/dataset-*.ts`

**端点**：
- `GET /api/public/datasets` - 列出 datasets
- `GET /api/public/datasets/:datasetName` - 获取 dataset
- `POST /api/public/datasets` - 创建 dataset
- `GET /api/public/dataset-items` - 列出 items
- `POST /api/public/dataset-items` - 创建 item
- `GET /api/public/dataset-runs` - 列出 runs
- `POST /api/public/dataset-runs` - 创建 run
- `POST /api/public/dataset-run-items` - 创建 run item

### 服务层

#### Dataset Service
位置：`packages/shared/src/server/repositories/dataset-*.ts`

**主要函数**：

**Dataset Operations**：
```typescript
createDataset(projectId, name, description, metadata, schemas): DatasetMutationResult
upsertDataset(projectId, name, ...): DatasetMutationResult
updateDataset(datasetId, updates): void
deleteDataset(datasetId): void  // 加入队列异步删除
```

**Dataset Item Operations**：
```typescript
createDatasetItem(projectId, datasetId, input, ...): DatasetMutationResult
upsertDatasetItem(projectId, datasetId, id, ...): DatasetMutationResult
deleteDatasetItem(projectId, datasetId, itemId): void
createManyDatasetItems(projectId, items): BulkCreateResult
validateAllDatasetItems(items, dataset): ValidationResult
```

**Query Operations**：
```typescript
getDatasetItems(projectId, filterState, orderBy, limit, offset): DatasetItem[]
getDatasetItemsCount(projectId, filterState): number
getDatasetItemById(projectId, datasetItemId, datasetId): DatasetItem
getDatasetItemVersionHistory(projectId, itemId): DatasetItemVersion[]
```

**Run Item Operations (ClickHouse)**：
```typescript
getDatasetRunItemsByDatasetIdCh(projectId, datasetId, filter, orderBy, limit, offset): RunItem[]
getDatasetRunItemsCountByDatasetIdCh(projectId, datasetId, filter): number
getDatasetRunsTableMetricsCh(projectId, datasetId, runIds, filter): RunMetrics[]
```

### 数据访问层

#### PostgreSQL Repository
位置：`packages/shared/src/server/repositories/dataset-items.ts`

**使用 Kysely 查询构建器**：
```typescript
// 查询当前版本的 items
DB.selectFrom("dataset_items")
  .where("project_id", "=", projectId)
  .where("dataset_id", "=", datasetId)
  .where("valid_to", "is", null)  // 当前版本
  .where("is_deleted", "=", false)
  .orderBy("created_at", "desc")
  .execute();
```

**版本管理逻辑**：
```typescript
// 更新 item：关闭旧版本，创建新版本
await trx.transaction(async (tx) => {
  // 1. 关闭旧版本
  await tx
    .updateTable("dataset_items")
    .set({ valid_to: new Date() })
    .where("id", "=", itemId)
    .where("valid_to", "is", null)
    .execute();
  
  // 2. 创建新版本
  await tx
    .insertInto("dataset_items")
    .values({
      id: itemId,
      project_id: projectId,
      valid_from: new Date(),
      valid_to: null,
      ...newData
    })
    .execute();
});
```

#### ClickHouse Repository
位置：`packages/shared/src/server/repositories/dataset-run-items.ts`

**性能指标计算逻辑**：
```typescript
// latency 和 cost 从 observations/traces 表实时计算

// Trace-level metrics:
// - latency_ms = dateDiff('millisecond', min(start_time), max(end_time))
// - total_cost = sum(observations.total_cost)
// - 优先级：trace > observation

// Observation-level metrics (当 observation_id 非空):
// - latency = dateDiff('millisecond', start_time, end_time)
// - total_cost = observation.total_cost

// 聚合为 dataset run 级别指标:
// - avg_latency_seconds = AVG(latency_ms) / 1000.0
// - avg_total_cost = AVG(total_cost)
// - total_cost = SUM(total_cost)
```

**查询优化**：
- 使用 `ORDER BY (project_id, dataset_id, dataset_run_id, id)` 加速查询
- 使用 CTEs 组织复杂查询（scores_aggregated, trace_metrics, dataset_run_metrics）
- 批量查询避免 N+1 问题
- 使用 `ReplacingMergeTree` 引擎自动去重

### 文件夹视图实现

#### SQL 生成函数
位置：`dataset-router.ts` 中的 `generateDatasetQuery`

**Root Level（无 pathPrefix）**：
```sql
WITH filtered_datasets AS (
  SELECT * FROM datasets
  WHERE project_id = ?
),
individual_datasets AS (
  -- 没有文件夹的 datasets
  SELECT *, 'dataset' as row_type
  FROM filtered_datasets
  WHERE name NOT LIKE '%/%'
),
folder_representatives AS (
  -- 每个文件夹的代表（显示第一个 dataset）
  SELECT
    id,
    SPLIT_PART(name, '/', 1) as name,  -- 只显示文件夹名
    description,
    metadata,
    'folder' as row_type,
    ROW_NUMBER() OVER (
      PARTITION BY SPLIT_PART(name, '/', 1)
      ORDER BY LENGTH(name) ASC, updated_at DESC
    ) AS rn
  FROM filtered_datasets
  WHERE name LIKE '%/%'
)
SELECT * FROM individual_datasets
UNION ALL
SELECT * FROM folder_representatives WHERE rn = 1
ORDER BY row_type, name;
```

**Folder Level（有 pathPrefix）**：
```sql
WITH filtered_datasets AS (
  SELECT * FROM datasets
  WHERE project_id = ?
    AND (name LIKE 'customer-support/%' OR name = 'customer-support')
),
individual_datasets_in_folder AS (
  -- 当前文件夹下的直接 datasets
  SELECT
    id,
    SUBSTRING(name, LENGTH('customer-support/') + 1) as name,
    'dataset' as row_type
  FROM filtered_datasets
  WHERE SUBSTRING(name, LENGTH('customer-support/') + 1) NOT LIKE '%/%'
    AND name != 'customer-support'
),
subfolder_representatives AS (
  -- 子文件夹的代表
  SELECT
    id,
    SPLIT_PART(SUBSTRING(name, LENGTH('customer-support/') + 1), '/', 1) as name,
    'folder' as row_type,
    ROW_NUMBER() OVER (
      PARTITION BY SPLIT_PART(SUBSTRING(name, LENGTH('customer-support/') + 1), '/', 1)
      ORDER BY LENGTH(name) ASC
    ) AS rn
  FROM filtered_datasets
  WHERE SUBSTRING(name, LENGTH('customer-support/') + 1) LIKE '%/%'
)
SELECT * FROM individual_datasets_in_folder
UNION ALL
SELECT * FROM subfolder_representatives WHERE rn = 1;
```

### 异步删除队列

#### Queue Worker
位置：`worker/src/queues/datasetDelete.ts`

**删除流程**：
1. 用户触发删除 → 加入 BullMQ 队列
2. Worker 处理：
   - 删除 Dataset Run Items（PostgreSQL + ClickHouse）
   - 删除 Dataset Runs
   - 删除 Dataset Items（设置 `is_deleted = true`）
   - 删除 Dataset
3. 审计日志记录

**批处理优化**：
```typescript
const BATCH_SIZE = 1000;

// 批量删除 run items
for (let i = 0; i < runItemIds.length; i += BATCH_SIZE) {
  const batch = runItemIds.slice(i, i + BATCH_SIZE);
  await prisma.datasetRunItems.deleteMany({
    where: { id: { in: batch } }
  });
}
```

## 前端集成

### 组件结构
```
web/src/features/datasets/
├── components/
│   ├── DatasetTable.tsx           # Dataset 列表表格
│   ├── DatasetItemTable.tsx       # Dataset Items 表格
│   ├── DatasetRunsTable.tsx       # Runs 列表
│   ├── DatasetRunItemsTable.tsx   # Run Items 表格
│   ├── DatasetFolderView.tsx      # 文件夹视图
│   ├── CreateDatasetButton.tsx    # 创建 dataset 按钮
│   └── DatasetSchemaEditor.tsx    # Schema 编辑器
├── hooks/
│   ├── useDatasets.ts             # Dataset 查询 hook
│   ├── useDatasetItems.ts         # Items 查询 hook
│   └── useDatasetRuns.ts          # Runs 查询 hook
├── lib/
│   ├── validation.ts              # Schema 验证逻辑
│   └── comparison.ts              # Run 对比逻辑
└── utils/
    ├── datasetUtils.ts            # Dataset 工具函数
    └── folderUtils.ts             # 文件夹路径处理
```

### 页面路由
```
/project/[projectId]/datasets                    # Datasets 列表
/project/[projectId]/datasets/[datasetId]        # Dataset 详情
/project/[projectId]/datasets/[datasetId]/items  # Dataset Items
/project/[projectId]/datasets/[datasetId]/runs   # Dataset Runs
/project/[projectId]/datasets/[datasetId]/runs/[runId]  # Run 详情
```

### tRPC Hooks 使用示例

#### 查询 Datasets
```typescript
const { data, isLoading } = api.datasets.allDatasets.useQuery({
  projectId,
  pathPrefix: currentFolder,
  searchQuery: searchTerm,
  page: 0,
  limit: 50
});

// 渲染文件夹和 datasets
{data?.datasets.map((item) => (
  item.row_type === "folder" ? (
    <FolderIcon name={item.name} onClick={() => navigate(item.name)} />
  ) : (
    <DatasetRow dataset={item} />
  )
))}
```

#### 创建 Dataset
```typescript
const createMutation = api.datasets.createDataset.useMutation({
  onSuccess: () => {
    toast.success("Dataset created");
    utils.datasets.allDatasets.invalidate();
  }
});

createMutation.mutate({
  projectId,
  name: "new-dataset",
  description: "Test dataset",
  inputSchema: { type: "object", properties: { ... } }
});
```

#### 查询 Run Items（对比）
```typescript
const { data } = api.datasets.runItemsByItemId.useQuery({
  projectId,
  datasetId,
  datasetItemId: selectedItemId,
  datasetRunIds: [run1Id, run2Id]  // 对比两个 runs
});

// 渲染对比表格
<ComparisonTable>
  <Row>
    <Cell>Input: {data.runItems[0].trace.input}</Cell>
    <Cell>Run 1: {data.runItems[0].trace.output}</Cell>
    <Cell>Run 2: {data.runItems[1].trace.output}</Cell>
  </Row>
  <Row>
    <Cell>Latency</Cell>
    <Cell>{data.runItems[0].trace.latency}ms</Cell>
    <Cell>{data.runItems[1].trace.latency}ms</Cell>
  </Row>
</ComparisonTable>
```

## 使用场景

### 场景 1：从生产流量创建测试数据

**流程**：
1. 在 Traces 页面选择有代表性的 traces
2. 点击 "Add to Dataset" 批量操作
3. 选择目标 dataset 或创建新 dataset
4. 系统自动提取 input/output，创建 dataset items

**代码示例**：
```typescript
// SDK
const trace = langfuse.trace({
  name: "customer-greeting",
  input: { userName: "Alice" },
  output: { greeting: "Hello Alice!" }
});

// 后续在 UI 中选中该 trace，添加到 dataset
// 或使用 API
POST /api/public/dataset-items
{
  datasetId: "ds-123",
  sourceTraceId: trace.id
}
```

### 场景 2：执行评估 Run

**流程**：
1. 创建 Dataset Run
2. 遍历 Dataset Items
3. 对每个 item 执行模型推理
4. 关联生成的 Trace 到 Run Item
5. 计算 scores（可手动或使用 evaluator）
6. 在 UI 中查看聚合结果

**代码示例**：
```typescript
// 创建 run
const run = await langfuse.createDatasetRun({
  datasetId: "ds-123",
  name: "gpt-4-v2-baseline",
  metadata: {
    model: "gpt-4",
    promptVersion: "v2"
  }
});

// 遍历 items
const items = await langfuse.getDatasetItems("ds-123");

for (const item of items) {
  // 执行推理
  const trace = langfuse.trace({
    name: "dataset-run-item",
    input: item.input,
    metadata: {
      datasetItemId: item.id,
      datasetRunId: run.id
    }
  });
  
  const response = await callLLM(item.input);
  
  trace.update({
    output: response
  });
  
  // 关联到 run item
  await langfuse.createDatasetRunItem({
    runId: run.id,
    datasetItemId: item.id,
    traceId: trace.id
  });
  
  // 评分
  trace.score({
    name: "correctness",
    value: calculateScore(response, item.expectedOutput)
  });
}
```

### 场景 3：对比不同模型版本

**流程**：
1. 对同一 dataset 执行多个 runs（不同模型/prompt）
2. 在 UI 中选择要对比的 runs
3. 查看并排对比表格：
   - 每行一个 dataset item
   - 每列一个 run
   - 显示 output、latency、cost、scores
4. 筛选出表现不佳的 items 进行分析

**UI 示例**：
```
Dataset Item ID | Input          | GPT-4 (v1)      | GPT-4 (v2)      | GPT-3.5
----------------|----------------|-----------------|-----------------|----------------
item-001        | "Hello world"  | "Hi there!"     | "Hello!"        | "Hey!"
                |                | Latency: 450ms  | Latency: 380ms  | Latency: 200ms
                |                | Cost: $0.002    | Cost: $0.0018   | Cost: $0.0005
                |                | Score: 0.95     | Score: 0.98     | Score: 0.85
----------------|----------------|-----------------|-----------------|----------------
item-002        | ...            | ...             | ...             | ...
```

### 场景 4：A/B Testing 与实验追踪

**流程**：
1. 创建 dataset 包含真实流量样本
2. 为每个实验配置创建 run：
   - Run A: 控制组配置
   - Run B: 实验组配置
3. 执行 runs 并收集 scores
4. 使用统计检验对比结果
5. 记录 metadata 追踪实验配置

**Metadata 示例**：
```typescript
{
  datasetRunId: "run-456",
  metadata: {
    experimentName: "prompt-optimization-week-12",
    variant: "B",
    promptVersion: "v2.1",
    temperature: 0.7,
    modelName: "gpt-4",
    hypothesis: "Adding examples improves accuracy",
    startDate: "2024-12-01",
    owner: "alice@company.com"
  }
}
```

## 性能优化

### 1. ClickHouse 查询优化

#### 索引设计
```sql
ORDER BY (project_id, dataset_id, trace_id)
```
- `project_id`: 主要分区键，加速租户隔离
- `dataset_id`: 加速按 dataset 查询
- `trace_id`: 加速关联 traces

#### 查询优化
```typescript
// 批量查询避免 N+1
const traceIds = runItems.map(item => item.traceId);
const traces = await clickhouse.query(`
  SELECT * FROM traces
  WHERE project_id = ?
    AND trace_id IN (${traceIds.join(',')})
`);

// 使用 Map 快速关联
const traceMap = new Map(traces.map(t => [t.trace_id, t]));
const enrichedItems = runItems.map(item => ({
  ...item,
  trace: traceMap.get(item.traceId)
}));
```

### 2. PostgreSQL 查询优化

#### 索引策略
```sql
-- Dataset Items
CREATE INDEX idx_dataset_items_current_version 
ON dataset_items (project_id, dataset_id, id) 
WHERE valid_to IS NULL AND is_deleted = false;

-- 加速版本查询
CREATE INDEX idx_dataset_items_temporal 
ON dataset_items (project_id, id, valid_from);

-- 加速 source trace 查询
CREATE INDEX idx_dataset_items_source_trace 
ON dataset_items (source_trace_id) 
USING HASH;
```

#### 文件夹查询优化
```sql
-- 使用 CTE 和 UNION ALL 避免多次全表扫描
-- 使用 ROW_NUMBER() 窗口函数选择文件夹代表
-- 使用 SPLIT_PART 和 SUBSTRING 进行字符串操作（PostgreSQL 内置优化）
```

### 3. 缓存策略

#### 应用层缓存
```typescript
// Dataset schema 缓存（不常变化）
const datasetSchemaCache = new LRU<string, JSONSchema>({
  max: 1000,
  ttl: 3600000  // 1 hour
});

// Run metrics 缓存（计算密集）
const runMetricsCache = new LRU<string, RunMetrics>({
  max: 500,
  ttl: 300000  // 5 minutes
});
```

#### tRPC 查询缓存
```typescript
// 前端使用 React Query 自动缓存
const { data } = api.datasets.allDatasets.useQuery(
  { projectId, pathPrefix },
  {
    staleTime: 60000,      // 1 分钟内不重新请求
    cacheTime: 300000,     // 5 分钟后清除缓存
    refetchOnWindowFocus: false
  }
);
```

### 4. 批量操作优化

#### 批量创建 Items
```typescript
const BATCH_SIZE = 100;

// 分批插入，避免单个事务过大
for (let i = 0; i < items.length; i += BATCH_SIZE) {
  const batch = items.slice(i, i + BATCH_SIZE);
  
  await prisma.$transaction(async (tx) => {
    // 验证
    const validated = validateBatch(batch);
    
    // 插入
    await tx.datasetItem.createMany({
      data: validated
    });
  });
}
```

#### 批量删除优化
```typescript
// 使用 queue 异步处理
await addToDeleteDatasetQueue({
  datasetId,
  projectId
});

// Worker 中批量删除
const deleteInBatches = async (ids: string[], batchSize = 1000) => {
  for (let i = 0; i < ids.length; i += batchSize) {
    const batch = ids.slice(i, i + batchSize);
    await prisma.datasetRunItems.deleteMany({
      where: { id: { in: batch } }
    });
  }
};
```

## 最佳实践

### 1. Dataset 组织
- **使用文件夹**：按功能/场景组织 datasets（如 `customer-support/greeting`）
- **命名规范**：使用描述性名称（如 `gpt4-baseline-2024-12`）
- **添加 metadata**：记录 dataset 的用途、创建时间、数据来源等

### 2. Schema 定义
- **定义 input schema**：确保测试数据格式一致
- **定义 output schema**：验证模型输出格式
- **使用严格模式**：`additionalProperties: false` 防止意外字段

### 3. Run 命名
- **包含模型信息**：如 `gpt-4-v2-baseline`
- **包含日期**：如 `2024-12-15-experiment-1`
- **包含实验变量**：如 `temperature-0.9-topk-50`

### 4. 评估策略
- **多维度评估**：不仅看 output 正确性，还要看延迟、成本
- **使用 scores**：定义多个 score 维度（correctness, relevance, fluency）
- **设置基线**：始终有一个稳定的基线 run 用于对比
- **自动化评估**：使用 evaluator（model-based 或 rule-based）

### 5. 数据质量
- **从真实流量采样**：确保测试数据代表性
- **定期更新**：随着产品演进更新测试数据
- **多样性**：覆盖 edge cases 和常见场景
- **去重**：避免重复的测试样本

## 常见问题

### Q1: Dataset Item 的版本管理如何工作？
A: 使用 temporal table 机制：
- 每次更新创建新版本（新行）
- `validFrom` 记录版本生效时间
- `validTo` 标记版本失效时间（NULL = 当前版本）
- `isDeleted = true` 实现软删除
- 查询时使用 `WHERE valid_to IS NULL` 获取当前版本

### Q2: 为什么 Run Items 同时存在 PostgreSQL 和 ClickHouse？
A: 职责分离：
- **PostgreSQL**：存储关系数据（IDs、关联关系），用于事务写入
- **ClickHouse**：存储反规范化快照数据（dataset run 和 item 的 denormalized fields）
- 性能指标（latency、cost）不直接存储，而是查询时从 `observations` 和 `traces` 表实时计算
- Scores 从 `scores` 表关联查询
- ClickHouse 查询性能更好，适合聚合分析和复杂筛选
- PostgreSQL 保证事务一致性和关系完整性

### Q3: 如何高效对比多个 Runs？
A: 使用 `runItemsByItemId` API：
1. 传入 `datasetItemIds` 和 `datasetRunIds`
2. ClickHouse 批量查询所有 run items
3. Enrich trace 数据（input/output/scores）
4. 按 item ID 分组返回
5. 前端渲染并排对比表格

### Q4: Dataset 的文件夹视图如何实现？
A: 使用 SQL CTE 和字符串函数：
- `SPLIT_PART(name, '/', 1)` 提取文件夹名
- `SUBSTRING(name, LENGTH(prefix) + 1)` 计算相对路径
- `ROW_NUMBER()` 选择每个文件夹的代表 dataset
- `UNION ALL` 合并文件夹和 datasets
- `row_type` 字段标记类型（'folder' / 'dataset'）

### Q5: 如何从 Trace 快速创建 Dataset Item？
A: 使用 `sourceTraceId`：
```typescript
// 创建时指定 sourceTraceId
createDatasetItem({
  datasetId,
  sourceTraceId: "trace-123",
  sourceObservationId: "obs-456"  // 可选
});

// 系统自动提取：
// - input from trace.input / observation.input
// - expectedOutput from trace.output / observation.output
// - metadata from trace.metadata
```

### Q6: Dataset 删除为什么是异步的？
A: 性能和用户体验：
- 大 dataset 可能有数万个 items 和 run items
- 同步删除会导致请求超时
- 使用 BullMQ 队列异步处理：
  1. 立即返回 200 OK
  2. Worker 批量删除（batch size = 1000）
  3. 删除完成后记录审计日志

### Q7: 如何确保 Schema 验证性能？
A: 多层优化：
- 批量验证时使用并行验证（Promise.all）
- 缓存 compiled schema（使用 Ajv）
- 验证失败时提前返回（fail-fast）
- 大型 JSON 使用流式验证

### Q8: Run Items 的 latency 和 cost 如何计算？
A: **实时计算，不存储在 dataset_run_items_rmt 表中**：

**优先级策略**：
- **Trace-level**（当 observation_id 为 NULL）：
  - `latency_ms = dateDiff('millisecond', min(observations.start_time), max(observations.end_time))`
  - `total_cost = SUM(observations.total_cost)`
  
- **Observation-level**（当 observation_id 非空）：
  - `latency = dateDiff('millisecond', observation.start_time, observation.end_time)`
  - `total_cost = observation.total_cost`

**聚合逻辑**（在 `dataset_run_metrics` CTE 中）：
```typescript
// 平均 latency（优先 trace，其次 observation）
CASE
  WHEN trace_avg_latency IS NOT NULL THEN trace_avg_latency
  ELSE obs_avg_latency
END as avg_latency_seconds

// 平均和总 cost（优先 trace，其次 observation）
CASE
  WHEN trace_avg_cost IS NOT NULL THEN trace_avg_cost
  ELSE COALESCE(obs_avg_cost, 0)
END as avg_total_cost
```

**性能优化**：
- 使用 `observations_filtered` CTE 预筛选相关时间范围
- JOIN trace_metrics 批量计算
- 避免 N+1 查询

### Q9: 如何实现跨 Run 的统计对比？
A: 使用聚合查询：
```typescript
// 1. 查询所有 runs 的 metrics
const metrics = await getDatasetRunsTableMetricsCh({
  projectId,
  datasetId,
  runIds: [run1Id, run2Id]
});

// 2. 计算聚合指标
metrics.map(run => ({
  runId: run.id,
  avgLatency: run.avgLatency,
  avgCost: run.avgCost,
  avgScore: aggregateScores(run.scores).avg
}));

// 3. 前端渲染对比图表
```

## 相关模块

- **Traces 模块**：Dataset Items 可从 Traces 创建，Run Items 关联 Traces
- **Scores 模块**：Run Items 的评估依赖 Scores
- **Prompts 模块**：Run metadata 中记录 prompt 版本
- **Playground 模块**：可使用 Dataset Items 作为测试输入
- **Evaluators 模块**：自动化评估 Run Items（model-based evaluation）

## 目录结构

```
web/src/features/datasets/
├── components/              # UI 组件
│   ├── DatasetTable.tsx
│   ├── DatasetItemTable.tsx
│   ├── DatasetRunsTable.tsx
│   └── ...
├── hooks/                   # React Hooks
│   ├── useDatasets.ts
│   └── useDatasetItems.ts
├── server/                  # 后端逻辑
│   ├── dataset-router.ts    # tRPC Router (1955 lines)
│   ├── service.ts           # 业务逻辑
│   └── actions/
│       └── createDataset.ts
└── utils/                   # 工具函数
    └── datasetUtils.ts

packages/shared/src/
├── domain/
│   ├── dataset-items.ts     # Domain 类型
│   └── dataset-run-items.ts
├── server/repositories/
│   ├── dataset-items.ts     # PostgreSQL 查询
│   ├── dataset-run-items.ts # ClickHouse 查询
│   └── ...
├── tableDefinitions/
│   ├── datasetItemsTable.ts
│   └── datasetRunItemsTable.ts
└── server/redis/
    └── datasetDelete.ts     # 删除队列

worker/src/queues/
└── datasetDelete.ts         # 异步删除 worker

web/src/features/public-api/server/
├── dataset-runs.ts          # REST API
└── dataset-run-items.ts
```
