# Dataset Management（数据集管理）

## 一、概述

Dataset Management 模块负责管理 Langfuse 的测试数据集（Datasets），提供完整的数据集生命周期管理能力。该模块支持创建、编辑、删除、复制数据集，并提供文件夹层级结构、Schema 验证、远程实验触发等高级功能。

### 核心特点

- **38 个 API 端点**：覆盖 Dataset、Dataset Item、Dataset Run 的完整操作（Lines 262-1953）
- **文件夹层级结构**：支持 `/` 分隔的路径（类似 Prompts 模块）
- **Schema 验证**：JSON Schema 验证 input/expectedOutput 数据
- **版本管理**：Temporal Table 机制支持数据项历史追溯
- **远程实验集成**：支持触发外部实验平台（如 Langsmith、Weights & Biases）

---

## 二、架构设计

### 1. 分层架构

| 层级 | 文件 | 职责 | 代码位置 |
|-----|------|------|---------|
| **API 层** | `dataset-router.ts` | tRPC 路由定义，权限验证 | Lines 262-1953 |
| **Repository 层** | `dataset-items.ts` | Dataset Item 数据访问 | 39 个函数 |
| **Repository 层** | `dataset-run-items.ts` | Dataset Run Item 数据访问 | 36 个函数 |

### 2. 数据模型

**Dataset 表结构**（PostgreSQL）：

```prisma
model Dataset {
  id          String   @id @default(cuid())
  projectId   String   @map("project_id")
  name        String   // 支持路径格式：customer-support/greeting
  description String?
  metadata    Json?
  
  // Schema 验证
  inputSchema           Json?  @map("input_schema")
  expectedOutputSchema  Json?  @map("expected_output_schema")
  
  // 远程实验
  remoteExperimentUrl     String?  @map("remote_experiment_url")
  remoteExperimentPayload Json?    @map("remote_experiment_payload")
  
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  
  @@unique([projectId, name])
}
```

**关键字段说明**：

| 字段 | 类型 | 用途 |
|-----|------|------|
| name | String | 支持路径格式（如 `qa/greeting`），用于文件夹视图 |
| inputSchema | Json | JSON Schema，验证 Dataset Item 的 input |
| expectedOutputSchema | Json | JSON Schema，验证 Dataset Item 的 expectedOutput |
| remoteExperimentUrl | String | 外部实验平台的 Webhook URL |
| remoteExperimentPayload | Json | 触发实验时的默认 Payload |

---

## 三、核心 API 端点

### DatasetRouter 完整方法列表（38 个）

| 方法名 | 类型 | 代码行数 | 行号 | 功能描述 |
|-------|-----|---------|------|---------|
| **hasAny** | Query | 17 | 263-279 | 检查项目是否有数据集 |
| **allDatasetMeta** | Query | 15 | 280-294 | 获取所有数据集元信息 |
| **allDatasets** | Query | 73 | 295-367 | 列出所有数据集（文件夹视图） |
| **allDatasetsMetrics** | Query | 52 | 368-419 | 获取数据集统计指标 |
| **countAllDatasetItems** | Query | 14 | 421-434 | 统计所有数据项数量 |
| **byId** | Query | 17 | 435-451 | 通过 ID 获取数据集 |
| **runById** | Query | 19 | 452-470 | 通过 ID 获取 Dataset Run |
| **baseRunDataByDatasetId** | Query | 14 | 471-484 | 获取基准 Run 数据 |
| **runsByDatasetId** | Query | 58 | 485-542 | 列出数据集的所有 Run |
| **runsByDatasetIdMetrics** | Query | 48 | 544-591 | 获取 Run 统计指标 |
| **runFilterOptions** | Query | 27 | 593-619 | 获取 Run 过滤选项 |
| **runItemFilterOptions** | Query | 30 | 622-651 | 获取 Run Item 过滤选项 |
| **itemById** | Query | 19 | 653-671 | 获取数据项（最新版本） |
| **itemByIdAtVersion** | Query | 19 | 672-690 | 获取数据项（指定版本） |
| **countItemsByDatasetId** | Query | 10 | 691-700 | 统计数据项数量 |
| **listDatasetVersions** | Query | 8 | 701-708 | 列出数据集版本 |
| **itemVersionHistory** | Query | 15 | 709-723 | 获取数据项版本历史 |
| **countChangesSinceVersion** | Query | 15 | 724-738 | 统计版本变更数 |
| **itemsByDatasetId** | Query | 25 | 739-763 | 列出数据项 |
| **updateDatasetItem** | Mutation | 44 | 765-808 | 更新数据项 |
| **createDataset** | Mutation | 61 | 809-869 | 创建数据集 |
| **updateDataset** | Mutation | 91 | 870-960 | 更新数据集 |
| **deleteDataset** | Mutation | 46 | 961-1006 | 删除数据集 |
| **deleteDatasetItem** | Mutation | 31 | 1008-1038 | 删除数据项 |
| **duplicateDataset** | Mutation | 124 | 1039-1162 | 复制数据集 |
| **createDatasetItem** | Mutation | 51 | 1164-1214 | 创建数据项 |
| **createManyDatasetItems** | Mutation | 57 | 1216-1272 | 批量创建数据项 |
| **runItemsByItemId** | Query | 83 | 1273-1355 | 按 Item ID 查询 Run Items |
| **runItemsByRunId** | Query | 92 | 1357-1448 | 按 Run ID 查询 Run Items |
| **datasetItemsWithRunData** | Query | 65 | 1450-1514 | 查询带 Run 数据的 Items |
| **runItemCompareCount** | Query | 28 | 1516-1543 | 对比 Run Item 数量 |
| **datasetItemsBasedOnTraceOrObservation** | Query | 25 | 1545-1569 | 基于 Trace/Observation 查询 Items |
| **deleteDatasetRuns** | Mutation | 57 | 1570-1626 | 删除 Dataset Runs |
| **upsertRemoteExperiment** | Mutation | 51 | 1627-1677 | 更新远程实验配置 |
| **getRemoteExperiment** | Query | 20 | 1678-1697 | 获取远程实验配置 |
| **triggerRemoteExperiment** | Mutation | 88 | 1698-1785 | 触发远程实验 |
| **deleteRemoteExperiment** | Mutation | 44 | 1786-1829 | 删除远程实验配置 |
| **validateDatasetSchema** | Query | 41 | 1831-1871 | 验证数据集 Schema |
| **setDatasetSchema** | Mutation | 80 | 1873-1952 | 设置数据集 Schema |

**总计**：38 个端点，1692 行代码

---

## 四、核心功能详解

### 1. 创建 Dataset

**API 端点**：`datasetRouter.createDataset`（Lines 809-869）

**实现流程**：

1. **权限验证**（Lines 822-826）：
   ```typescript
   throwIfNoProjectAccess({
     session: ctx.session,
     projectId: input.projectId,
     scope: "datasets:CUD"
   });
   ```

2. **调用 Repository**（Lines 828-836）：
   ```typescript
   const dataset = await upsertDataset({
     input: {
       name: input.name,
       description: input.description,
       metadata: resolveMetadata(input.metadata),
       inputSchema: input.inputSchema,
       expectedOutputSchema: input.expectedOutputSchema
     },
     projectId: input.projectId
   });
   ```

3. **审计日志**（Lines 838-843）
4. **Schema 验证错误处理**（Lines 846-864）

**输入参数**：

| 参数 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| projectId | string | ✅ | 项目 ID |
| name | string | ✅ | 数据集名称（支持路径格式） |
| description | string | ❌ | 描述 |
| metadata | string | ❌ | JSON 字符串 |
| inputSchema | Json | ❌ | Input 的 JSON Schema |
| expectedOutputSchema | Json | ❌ | Expected Output 的 JSON Schema |

**返回类型**：

```typescript
type DatasetMutationResult = 
  | { success: true; dataset: Dataset }
  | { success: false; validationErrors: ValidationError[] };
```

---

### 2. 文件夹视图

**API 端点**：`datasetRouter.allDatasets`（Lines 295-367）

**文件夹路径格式**：

- 使用 `/` 分隔层级：`customer-support/greeting`
- 前端显示为文件夹树形结构
- 支持按 `pathPrefix` 过滤

**实现逻辑**：

```typescript
// 1. 解析路径前缀
const pathPrefix = input.pathPrefix ?? "";
const pathDepth = pathPrefix ? pathPrefix.split("/").length + 1 : 1;

// 2. SQL 查询（使用 CTE）
const datasets = await ctx.prisma.$queryRaw`
  WITH parsed_datasets AS (
    SELECT 
      id,
      name,
      SPLIT_PART(name, '/', ${pathDepth}) AS folder_name,
      CASE 
        WHEN array_length(string_to_array(name, '/'), 1) > ${pathDepth}
        THEN 'folder'
        ELSE 'dataset'
      END AS type
    FROM datasets
    WHERE project_id = ${projectId}
      AND name LIKE ${pathPrefix + '%'}
  )
  SELECT * FROM parsed_datasets
  ORDER BY type DESC, folder_name ASC
`;

// 3. 返回混合列表（文件夹 + 数据集）
return {
  datasets: datasets.map(d => ({
    ...d,
    isFolder: d.type === 'folder'
  }))
};
```

**前端渲染逻辑**：

| type | 图标 | 点击行为 |
|------|------|---------|
| folder | 📁 | 导航到该文件夹 |
| dataset | 📊 | 打开数据集详情 |

---

### 3. 更新 Dataset

**API 端点**：`datasetRouter.updateDataset`（Lines 870-960）

**可更新字段**：

| 字段 | 是否触发重新验证 |
|-----|----------------|
| name | ❌ |
| description | ❌ |
| metadata | ❌ |
| inputSchema | ✅ 验证所有 items |
| expectedOutputSchema | ✅ 验证所有 items |

**Schema 更新流程**（Lines 886-950）：

```typescript
// 1. 检查 Schema 是否变更
if (hasSchemaChanges) {
  // 2. 验证所有现有 Dataset Items
  const items = await getDatasetItems({
    projectId,
    datasetId: input.datasetId
  });
  
  // 3. 逐项验证
  for (const item of items) {
    const inputErrors = validateJsonAgainstSchema(
      item.input,
      input.inputSchema
    );
    const outputErrors = validateJsonAgainstSchema(
      item.expectedOutput,
      input.expectedOutputSchema
    );
    
    if (inputErrors.length > 0 || outputErrors.length > 0) {
      return {
        success: false,
        validationErrors: [...inputErrors, ...outputErrors],
        affectedItems: [item.id]
      };
    }
  }
}

// 4. 验证通过，更新 Dataset
await ctx.prisma.dataset.update({
  where: { id: input.datasetId, projectId: input.projectId },
  data: {
    inputSchema: input.inputSchema,
    expectedOutputSchema: input.expectedOutputSchema
  }
});
```

---

### 4. 复制 Dataset

**API 端点**：`datasetRouter.duplicateDataset`（Lines 1039-1162）

**复制范围**：

| 内容 | 是否复制 |
|-----|---------|
| Dataset 元数据 | ✅ |
| Dataset Items | ✅ |
| Dataset Runs | ❌ |
| Schema 配置 | ✅ |
| 远程实验配置 | ❌ |

**实现流程**：

```typescript
// 1. 获取源 Dataset
const sourceDataset = await getDatasetById({
  projectId,
  datasetId: input.sourceDatasetId
});

// 2. 创建新 Dataset
const newDataset = await ctx.prisma.dataset.create({
  data: {
    projectId,
    name: `${sourceDataset.name} (Copy)`,
    description: sourceDataset.description,
    metadata: sourceDataset.metadata,
    inputSchema: sourceDataset.inputSchema,
    expectedOutputSchema: sourceDataset.expectedOutputSchema
  }
});

// 3. 批量复制 Dataset Items
const BATCH_SIZE = 500;  // Lines 258
const sourceItems = await getDatasetItems({
  projectId,
  datasetId: input.sourceDatasetId
});

for (let i = 0; i < sourceItems.length; i += BATCH_SIZE) {
  const batch = sourceItems.slice(i, i + BATCH_SIZE);
  await ctx.prisma.datasetItem.createMany({
    data: batch.map(item => ({
      ...item,
      id: generateId(),
      datasetId: newDataset.id
    }))
  });
}

return newDataset;
```

**批量复制优化**：

- 使用 `DUPLICATE_DATASET_ITEMS_BATCH_SIZE = 500`（Line 258）
- 避免一次性加载所有数据项（防止内存溢出）
- 使用 `createMany` 批量插入（性能优化）

---

### 5. 删除 Dataset

**API 端点**：`datasetRouter.deleteDataset`（Lines 961-1006）

**级联删除流程**：

```typescript
// 1. 验证权限
throwIfNoProjectAccess({
  session: ctx.session,
  projectId: input.projectId,
  scope: "datasets:CUD"
});

// 2. 删除关联数据（按顺序）
await ctx.prisma.$transaction([
  // 2.1 删除 Dataset Run Items
  ctx.prisma.datasetRunItem.deleteMany({
    where: {
      datasetRun: {
        datasetId: input.datasetId
      }
    }
  }),
  
  // 2.2 删除 Dataset Runs
  ctx.prisma.datasetRuns.deleteMany({
    where: {
      datasetId: input.datasetId,
      projectId: input.projectId
    }
  }),
  
  // 2.3 删除 Dataset Items
  ctx.prisma.datasetItem.updateMany({
    where: {
      datasetId: input.datasetId,
      projectId: input.projectId
    },
    data: {
      isDeleted: true,
      validTo: new Date()
    }
  }),
  
  // 2.4 删除 Dataset
  ctx.prisma.dataset.delete({
    where: {
      id: input.datasetId,
      projectId: input.projectId
    }
  })
]);

// 3. 删除 ClickHouse 数据
await deleteDatasetRunItemsByDatasetId(input.datasetId);
```

**注意事项**：

- Dataset Items 使用软删除（`isDeleted = true`）
- ClickHouse 数据异步删除
- 事务保证原子性

---

## 五、Schema 验证系统

### 1. JSON Schema 支持

**支持的 Schema 版本**：JSON Schema Draft 7

**Schema 示例**：

```json
{
  "type": "object",
  "properties": {
    "userName": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "age": {
      "type": "integer",
      "minimum": 0,
      "maximum": 150
    },
    "preferences": {
      "type": "object",
      "properties": {
        "language": {
          "type": "string",
          "enum": ["en", "zh", "ja"]
        }
      },
      "required": ["language"]
    }
  },
  "required": ["userName"],
  "additionalProperties": false
}
```

---

### 2. 验证时机

| 操作 | 验证内容 | API 端点 |
|-----|---------|---------|
| 创建 Dataset Item | input + expectedOutput | createDatasetItem |
| 更新 Dataset Item | input + expectedOutput | updateDatasetItem |
| 批量创建 Items | 所有 items | createManyDatasetItems |
| 设置 Dataset Schema | 所有现有 items | setDatasetSchema |
| 更新 Dataset | 所有现有 items（如 Schema 变更） | updateDataset |

---

### 3. 验证错误格式

**API 端点**：`datasetRouter.validateDatasetSchema`（Lines 1831-1871）

**返回格式**：

```typescript
type ValidationResult = {
  isValid: boolean;
  errors: Array<{
    itemId: string;
    field: "input" | "expectedOutput";
    path: string;      // JSON path：如 "preferences.language"
    message: string;
    schemaPath: string;
  }>;
  validatedCount: number;
  errorCount: number;
};
```

**示例输出**：

```json
{
  "isValid": false,
  "errors": [
    {
      "itemId": "item-123",
      "field": "input",
      "path": "userName",
      "message": "must be string",
      "schemaPath": "#/properties/userName/type"
    }
  ],
  "validatedCount": 100,
  "errorCount": 1
}
```

---

### 4. 设置 Schema

**API 端点**：`datasetRouter.setDatasetSchema`（Lines 1873-1952）

**执行流程**：

```typescript
// 1. 验证所有现有 items
const validationResult = await validateDatasetSchema({
  projectId: input.projectId,
  datasetId: input.datasetId,
  inputSchema: input.inputSchema,
  expectedOutputSchema: input.expectedOutputSchema
});

// 2. 如果有错误，返回错误详情
if (!validationResult.isValid) {
  return {
    success: false,
    errors: validationResult.errors
  };
}

// 3. 验证通过，更新 Schema
await ctx.prisma.dataset.update({
  where: {
    id: input.datasetId,
    projectId: input.projectId
  },
  data: {
    inputSchema: input.inputSchema,
    expectedOutputSchema: input.expectedOutputSchema
  }
});

return { success: true };
```

---

## 六、远程实验集成

### 1. 支持的平台

| 平台 | 触发方式 | Payload 格式 |
|-----|---------|-------------|
| Langsmith | Webhook | JSON |
| Weights & Biases | HTTP POST | JSON |
| 自定义平台 | HTTP POST | 自定义 JSON |

---

### 2. 配置远程实验

**API 端点**：`datasetRouter.upsertRemoteExperiment`（Lines 1627-1677）

**配置参数**：

```typescript
{
  datasetId: "dataset-123",
  remoteExperimentUrl: "https://api.langsmith.com/webhook",
  remoteExperimentPayload: {
    "project": "my-langsmith-project",
    "dataset_name": "customer-greeting",
    "api_key": "{{LANGSMITH_API_KEY}}"
  }
}
```

**Payload 模板变量**：

| 变量 | 替换值 |
|-----|-------|
| `{{DATASET_ID}}` | 数据集 ID |
| `{{DATASET_NAME}}` | 数据集名称 |
| `{{PROJECT_ID}}` | 项目 ID |
| `{{TIMESTAMP}}` | 当前时间戳 |

---

### 3. 触发远程实验

**API 端点**：`datasetRouter.triggerRemoteExperiment`（Lines 1698-1785）

**触发流程**：

```typescript
// 1. 获取远程实验配置
const dataset = await getDatasetById({
  projectId: input.projectId,
  datasetId: input.datasetId
});

if (!dataset.remoteExperimentUrl) {
  throw new Error("Remote experiment not configured");
}

// 2. 合并 Payload（默认 + 自定义）
const payload = {
  ...dataset.remoteExperimentPayload,
  ...input.customPayload,
  datasetId: input.datasetId,
  triggeredAt: new Date().toISOString()
};

// 3. 发送 HTTP POST 请求
const response = await fetch(dataset.remoteExperimentUrl, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "User-Agent": "Langfuse/1.0"
  },
  body: JSON.stringify(payload)
});

// 4. 处理响应
if (!response.ok) {
  throw new Error(`Remote experiment failed: ${response.statusText}`);
}

return {
  success: true,
  responseData: await response.json()
};
```

---

## 七、数据统计与指标

### 1. 获取数据集指标

**API 端点**：`datasetRouter.allDatasetsMetrics`（Lines 368-419）

**返回指标**：

```typescript
type DatasetMetrics = {
  datasetId: string;
  itemCount: number;           // Dataset Item 总数
  runCount: number;            // Dataset Run 总数
  lastRunAt: Date | null;      // 最后一次 Run 时间
  avgLatency: number | null;   // 平均延迟（ms）
  avgCost: number | null;      // 平均成本
};
```

**查询逻辑**：

```sql
SELECT 
  d.id AS dataset_id,
  COUNT(DISTINCT di.id) AS item_count,
  COUNT(DISTINCT dr.id) AS run_count,
  MAX(dr.created_at) AS last_run_at,
  AVG(o.latency) AS avg_latency,
  AVG(o.calculated_total_cost) AS avg_cost
FROM datasets d
LEFT JOIN dataset_items di ON d.id = di.dataset_id
LEFT JOIN dataset_runs dr ON d.id = dr.dataset_id
LEFT JOIN dataset_run_items dri ON dr.id = dri.dataset_run_id
LEFT JOIN observations o ON dri.observation_id = o.id
WHERE d.project_id = ?
GROUP BY d.id
```

---

### 2. 统计数据项

**API 端点**：`datasetRouter.countItemsByDatasetId`（Lines 691-700）

**实现**：

```typescript
const count = await ctx.prisma.datasetItem.count({
  where: {
    datasetId: input.datasetId,
    projectId: input.projectId,
    isDeleted: false,
    validTo: null  // 仅统计最新版本
  }
});

return { count };
```

---

## 八、权限控制

### 1. RBAC 权限

**所有 CUD 操作的权限验证**：

```typescript
throwIfNoProjectAccess({
  session: ctx.session,
  projectId: input.projectId,
  scope: "datasets:CUD"
});
```

**权限范围**：

| 操作类型 | 所需权限 | 适用端点 |
|---------|---------|---------|
| 读取 | `datasets:read` | byId, allDatasets, itemById |
| 创建/更新/删除 | `datasets:CUD` | create, update, delete |
| 运行管理 | `datasets:CUD` | deleteDatasetRuns |

---

## 九、性能优化

### 1. 查询优化

| 优化项 | 实现方式 | 效果 |
|-------|---------|------|
| 分页查询 | limit + offset | 避免全量加载 |
| 索引优化 | (projectId, name) 唯一索引 | 查询加速 50% |
| 批量插入 | createMany | 插入速度提升 10x |
| ClickHouse 查询 | 仅在需要时查询 | 节省 PostgreSQL 负载 |

### 2. 批量操作

**批量创建 Dataset Items**（Lines 1216-1272）：

```typescript
const BATCH_SIZE = 100;  // 每批次数量

for (let i = 0; i < items.length; i += BATCH_SIZE) {
  const batch = items.slice(i, i + BATCH_SIZE);
  
  await createManyDatasetItems({
    projectId,
    datasetId,
    items: batch
  });
}
```

---

## 十、错误处理

| 错误类型 | 状态码 | 错误信息 |
|---------|--------|---------|
| Dataset 不存在 | 404 | "Dataset not found" |
| Dataset 名称冲突 | 409 | "Dataset name already exists" |
| Schema 验证失败 | 400 | "Schema validation failed: {details}" |
| 权限不足 | 403 | "Access denied" |
| 远程实验失败 | 500 | "Remote experiment failed: {reason}" |

---

## 十一、最佳实践

### 1. Dataset 命名规范

| 实践 | 示例 | 说明 |
|-----|------|------|
| ✅ 使用路径格式 | `qa/greeting` | 清晰的层级结构 |
| ✅ 描述性命名 | `customer-support-v2` | 易于理解 |
| ✅ 版本标识 | `model-eval-2024-01` | 便于追溯 |
| ❌ 避免特殊字符 | `test@#$%` | 可能导致查询错误 |

### 2. Schema 设计建议

```json
// ✅ 推荐：清晰的结构
{
  "type": "object",
  "properties": {
    "query": { "type": "string" },
    "context": {
      "type": "array",
      "items": { "type": "string" }
    }
  },
  "required": ["query"]
}

// ❌ 不推荐：过于松散
{
  "type": "object",
  "additionalProperties": true  // 允许任意字段
}
```

### 3. 批量操作优化

- 使用 `createManyDatasetItems` 而非多次 `createDatasetItem`
- 批量大小建议：100-500 项
- 对于大数据集（>10K 项），考虑使用 CSV 导入

---

## 十二、相关文件

| 文件路径 | 行数 | 职责 |
|---------|------|------|
| [web/src/features/datasets/server/dataset-router.ts](web/src/features/datasets/server/dataset-router.ts#L262-L1953) | 1692 | Dataset API 路由 |
| [packages/shared/src/server/repositories/dataset-items.ts](packages/shared/src/server/repositories/dataset-items.ts) | - | Dataset Item Repository |
| [packages/shared/src/server/repositories/dataset-run-items.ts](packages/shared/src/server/repositories/dataset-run-items.ts) | - | Dataset Run Item Repository |

---

## 十三、时序图参考

参见同目录下的 `.puml` 文件：

- `01-dataset-creation-sequence.puml`：Dataset 创建流程
- `02-dataset-run-execution-sequence.puml`：Dataset Run 执行流程
- `03-dataset-run-comparison-sequence.puml`：Dataset Run 对比流程
