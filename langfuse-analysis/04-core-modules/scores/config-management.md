# Config Management 管理

## 功能概述

Config Management 是 Scores 模块的**评分配置管理系统**，通过模板化配置（Score Configs）统一评分标准，支持数值范围、分类类别、评论模板等约束。核心特性：
- **模板化验证**：统一评分标准（范围、类别、数据类型）
- **配置归档**：软删除机制，防止误操作
- **值映射**：分类标签 → 数值映射（如 "good" → 5）
- **审计日志**：完整记录配置变更历史
- **灵活验证**：支持 INGESTION 和 ANNOTATION 两种上下文

---

## 核心组件

### 1. **scoreConfigsRouter TRPC 路由**

**位置**：[web/src/server/api/routers/scoreConfigs.ts](web/src/server/api/routers/scoreConfigs.ts#L50-L185)

#### **路由实现（136 行）**

```typescript
// Lines 50-185: scoreConfigsRouter（136 行）
const scoreConfigsRouter = createTRPCRouter({
  // 路由 1: 获取所有配置（Lines 51-83）
  all: protectedProjectProcedure
    .input(ScoreConfigAllInputPaginated)
    .query(async ({ input, ctx }) => {
      throwIfNoProjectAccess({
        session: ctx.session,
        projectId: input.projectId,
        scope: "scoreConfigs:read",
      });

      const [configs, totalCount] = await Promise.all([
        ctx.prisma.scoreConfig.findMany({
          where: {
            projectId: input.projectId,
          },
          ...(input.limit !== undefined && input.page !== undefined
            ? { take: input.limit, skip: input.page * input.limit }
            : undefined),
          orderBy: {
            createdAt: "desc",
          },
        }),
        ctx.prisma.scoreConfig.count({
          where: {
            projectId: input.projectId,
          },
        }),
      ]);

      return {
        configs: filterAndValidateDbScoreConfigList(configs, traceException),
        totalCount,
      };
    }),
    
  // 路由 2: 创建配置（Lines 84-107）
  create: protectedProjectProcedure
    .input(ScoreConfigCreateInput)
    .mutation(async ({ input, ctx }) => {
      throwIfNoProjectAccess({
        session: ctx.session,
        projectId: input.projectId,
        scope: "scoreConfigs:CUD",
      });

      const config = await ctx.prisma.scoreConfig.create({
        data: {
          ...input,
        },
      });

      await auditLog({
        session: ctx.session,
        resourceType: "scoreConfig",
        resourceId: config.id,
        action: "create",
        after: config,
      });

      return validateDbScoreConfig(config);
    }),
    
  // 路由 3: 更新配置（Lines 108-157）
  update: protectedProjectProcedure
    .input(ScoreConfigUpdateInput)
    .mutation(async ({ input, ctx }) => {
      throwIfNoProjectAccess({
        session: ctx.session,
        projectId: input.projectId,
        scope: "scoreConfigs:CUD",
      });

      const existingConfig = await ctx.prisma.scoreConfig.findFirst({
        where: {
          id: input.id,
          projectId: input.projectId,
        },
      });
      if (!existingConfig) {
        throw new LangfuseNotFoundError(
          "No score config with this id in this project.",
        );
      }

      // 合并输入并验证架构合规性
      const result = validateDbScoreConfigSafe({ ...existingConfig, ...input });

      if (!result.success) {
        throw new InvalidRequestError(
          result.error.issues.map((issue) => issue.message).join(", "),
        );
      }

      const config = await ctx.prisma.scoreConfig.update({
        where: {
          id: input.id,
          projectId: input.projectId,
        },
        data: { ...input },
      });

      await auditLog({
        session: ctx.session,
        resourceType: "scoreConfig",
        resourceId: config.id,
        action: "update",
        before: existingConfig,
        after: config,
      });

      return validateDbScoreConfig(config);
    }),
    
  // 路由 4: 按 ID 获取配置（Lines 158-185）
  byId: protectedProjectProcedure
    .input(
      z.object({
        id: z.string(),
        projectId: z.string(),
      }),
    )
    .query(async ({ input, ctx }) => {
      throwIfNoProjectAccess({
        session: ctx.session,
        projectId: input.projectId,
        scope: "scoreConfigs:read",
      });

      const config = await ctx.prisma.scoreConfig.findFirst({
        where: {
          id: input.id,
          projectId: input.projectId,
        },
      });

      if (!config) {
        throw new Error("No score config with this id in this project.");
      }

      return validateDbScoreConfig(config);
    }),
});
```

**4 个路由**：

| 路由 | 方法 | 代码行 | 作用 |
|-----|------|-------|------|
| `all` | query | 51-83 | 获取所有配置（分页） |
| `create` | mutation | 84-107 | 创建新配置 |
| `update` | mutation | 108-157 | 更新配置（含验证） |
| `byId` | query | 158-185 | 按 ID 获取配置 |

---

### 2. **validateConfigAgainstBody 验证函数**

**位置**：[packages/shared/src/server/ingestion/validateAndInflateScore.ts](packages/shared/src/server/ingestion/validateAndInflateScore.ts#L157-L200)

#### **函数实现（44 行估算）**

```typescript
// Lines 157-200: validateConfigAgainstBody（约 44 行）
export function validateConfigAgainstBody({
  body,
  config,
  context,
}: ValidateConfigAgainstBodyParams): void {
  const { maxValue, minValue, categories, dataType: configDataType } = config;

  // 验证 1: 数据类型匹配
  if (body.dataType && body.dataType !== configDataType) {
    throw new InvalidRequestError(
      `Data type mismatch based on config: expected ${configDataType}, got ${body.dataType}`,
    );
  }

  // 验证 2: 配置未归档
  if (config.isArchived) {
    throw new InvalidRequestError(
      "Config is archived and cannot be used to create new scores. Please restore the config first.",
    );
  }

  // 验证 3: 名称匹配
  if (config.name !== body.name) {
    throw new InvalidRequestError(
      `Name mismatch based on config: expected ${config.name}, got ${body.name}`,
    );
  }

  // 步骤 4: 提取评分值
  const relevantDataType = configDataType ?? body.dataType;
  const scoreValue =
    context === "INGESTION"
      ? resolveScoreValueIngestion(body)
      : resolveScoreValueAnnotation(body);

  // 验证 5: 范围/类别验证
  const rangeValidation = ScorePropsAgainstConfig.safeParse({
    value: scoreValue,
    dataType: relevantDataType,
    ...(maxValue !== null && maxValue !== undefined && { maxValue }),
    ...(minValue !== null && minValue !== undefined && { minValue }),
    ...(categories && { categories }),
  });

  if (!rangeValidation.success) {
    const errorDetails = rangeValidation.error.issues
      .map((error) => `${error.path.join(".")} - ${error.message}`)
      .join(", ");

    throw new InvalidRequestError(
      `Score value does not comply with config: ${errorDetails}`,
    );
  }
}
```

**5 步验证流程**：

| 步骤 | 验证内容 | 错误信息 |
|-----|---------|---------|
| 1 | 数据类型匹配 | `Data type mismatch: expected NUMERIC, got CATEGORICAL` |
| 2 | 配置未归档 | `Config is archived and cannot be used` |
| 3 | 名称匹配 | `Name mismatch: expected quality, got correctness` |
| 4 | 提取评分值 | 根据上下文（INGESTION/ANNOTATION）提取 |
| 5 | 范围/类别验证 | `Score value 10 exceeds max value 5` |

---

## 配置数据模型

### **ScoreConfig Schema**

```typescript
type ScoreConfig = {
  id: string;
  projectId: string;
  name: string;                   // 评分名称
  dataType: ScoreDataType;        // NUMERIC | CATEGORICAL | BOOLEAN
  minValue?: number | null;       // 最小值（NUMERIC）
  maxValue?: number | null;       // 最大值（NUMERIC）
  categories?: Category[] | null; // 类别列表（CATEGORICAL）
  description?: string | null;    // 描述信息
  isArchived: boolean;            // 是否归档
  createdAt: Date;
  updatedAt: Date;
};

type Category = {
  label: string;                  // 类别标签（如 "good"）
  value: number;                  // 映射数值（如 5）
};
```

---

## 配置类型

### **NUMERIC 配置**

#### **数据结构**

```typescript
type NumericScoreConfig = {
  name: "quality";
  dataType: "NUMERIC";
  minValue: 0;
  maxValue: 5;
  categories: null;
  description: "Quality rating from 0 to 5";
};
```

#### **验证规则**

| 规则 | 说明 | 示例 |
|-----|------|------|
| **范围检查** | `minValue <= value <= maxValue` | `value: 3` ✅, `value: 10` ❌ |
| **数值类型** | 必须为 number | `value: 4.5` ✅, `value: "high"` ❌ |

#### **使用示例**

```typescript
// 配置
const config: NumericScoreConfig = {
  name: "quality",
  dataType: "NUMERIC",
  minValue: 0,
  maxValue: 5,
};

// 有效评分
validateConfigAgainstBody({
  body: { name: "quality", dataType: "NUMERIC", value: 4.5 },
  config,
  context: "INGESTION",
});  // ✅ 通过

// 无效评分：超出范围
validateConfigAgainstBody({
  body: { name: "quality", dataType: "NUMERIC", value: 10 },
  config,
  context: "INGESTION",
});  // ❌ 抛出异常："Score value 10 exceeds max value 5"
```

---

### **CATEGORICAL 配置**

#### **数据结构**

```typescript
type CategoricalScoreConfig = {
  name: "sentiment";
  dataType: "CATEGORICAL";
  minValue: null;
  maxValue: null;
  categories: [
    { label: "positive", value: 5 },
    { label: "neutral", value: 3 },
    { label: "negative", value: 1 },
  ];
  description: "Sentiment analysis";
};
```

#### **验证规则**

| 规则 | 说明 | 示例 |
|-----|------|------|
| **类别匹配** | 值必须在类别列表中 | `"positive"` ✅, `"unknown"` ❌ |
| **标签 → 值映射** | 自动映射到对应数值 | `"positive"` → `5` |

#### **使用示例**

```typescript
// 配置
const config: CategoricalScoreConfig = {
  name: "sentiment",
  dataType: "CATEGORICAL",
  categories: [
    { label: "positive", value: 5 },
    { label: "neutral", value: 3 },
    { label: "negative", value: 1 },
  ],
};

// 有效评分
validateConfigAgainstBody({
  body: { name: "sentiment", dataType: "CATEGORICAL", value: "positive" },
  config,
  context: "INGESTION",
});  // ✅ 通过，映射为 value: 5, stringValue: "positive"

// 无效评分：类别不存在
validateConfigAgainstBody({
  body: { name: "sentiment", dataType: "CATEGORICAL", value: "unknown" },
  config,
  context: "INGESTION",
});  // ❌ 抛出异常："Category 'unknown' not found in config"
```

---

### **BOOLEAN 配置**

#### **数据结构**

```typescript
type BooleanScoreConfig = {
  name: "is_correct";
  dataType: "BOOLEAN";
  minValue: null;
  maxValue: null;
  categories: null;
  description: "Correctness check";
};
```

#### **验证规则**

| 规则 | 说明 | 示例 |
|-----|------|------|
| **布尔值** | 接受 `true/false` 或 `1/0` | `true` ✅, `"maybe"` ❌ |
| **自动转换** | `1 → "True"`, `0 → "False"` | `value: 1` → `stringValue: "True"` |

#### **使用示例**

```typescript
// 配置
const config: BooleanScoreConfig = {
  name: "is_correct",
  dataType: "BOOLEAN",
};

// 有效评分
validateConfigAgainstBody({
  body: { name: "is_correct", dataType: "BOOLEAN", value: 1 },
  config,
  context: "INGESTION",
});  // ✅ 通过，转换为 value: 1, stringValue: "True"

// 有效评分：布尔值
validateConfigAgainstBody({
  body: { name: "is_correct", dataType: "BOOLEAN", value: true },
  config,
  context: "INGESTION",
});  // ✅ 通过
```

---

## 验证上下文

### **INGESTION 上下文**

**场景**：API ingestion 时验证评分

```typescript
// 提取值的方式
function resolveScoreValueIngestion(
  body: ScoreEventType["body"],
): string | number | null {
  return body.value;  // 直接从 body.value 提取
}

// 示例
validateConfigAgainstBody({
  body: { name: "quality", value: 4.5, dataType: "NUMERIC" },
  config: { name: "quality", dataType: "NUMERIC", minValue: 0, maxValue: 5 },
  context: "INGESTION",
});
```

---

### **ANNOTATION 上下文**

**场景**：手动标注时验证评分

```typescript
// 提取值的方式
function resolveScoreValueAnnotation(
  body: ScoreDomain,
): string | number | null {
  switch (body.dataType) {
    case ScoreDataType.NUMERIC:
    case ScoreDataType.BOOLEAN:
      return body.value;        // NUMERIC/BOOLEAN 使用 value
    case ScoreDataType.CATEGORICAL:
      return body.stringValue;  // CATEGORICAL 使用 stringValue
  }
}

// 示例
validateConfigAgainstBody({
  body: { 
    name: "sentiment", 
    dataType: "CATEGORICAL", 
    stringValue: "positive",  // 注意：使用 stringValue
  },
  config: { 
    name: "sentiment", 
    dataType: "CATEGORICAL", 
    categories: [{ label: "positive", value: 5 }] 
  },
  context: "ANNOTATION",
});
```

**上下文差异**：

| 字段 | INGESTION | ANNOTATION |
|-----|-----------|-----------|
| **NUMERIC 提取** | `body.value` | `body.value` |
| **CATEGORICAL 提取** | `body.value` | `body.stringValue` |
| **BOOLEAN 提取** | `body.value` | `body.value` |

---

## 配置归档

### **归档机制**

```typescript
// Lines 600-620 (updateAnnotationScore): 验证配置未归档
if (score.configId) {
  const config = await ctx.prisma.scoreConfig.findFirst({
    where: {
      id: score.configId,
      projectId: input.projectId,
    },
  });
  
  if (!config) {
    throw new LangfuseNotFoundError(
      `No score config with id ${score.configId} in project ${input.projectId}`,
    );
  }
  
  try {
    validateConfigAgainstBody({
      body: { ...score, value: input.value },
      config: config as ScoreConfigDomain,
      context: "ANNOTATION",
    });
  } catch (_error) {
    throw new TRPCError({
      code: "PRECONDITION_FAILED",
      message: "Score does not comply with config schema. Please adjust or delete score.",
    });
  }
}
```

**归档规则**：

| 状态 | 创建新评分 | 更新现有评分 | 查询配置 |
|-----|-----------|-------------|---------|
| **isArchived: false** | ✅ 允许 | ✅ 允许 | ✅ 返回 |
| **isArchived: true** | ❌ 拒绝 | ⚠️ 警告 | ✅ 返回（标记归档） |

---

## 值映射

### **分类标签 → 数值映射**

**位置**：[packages/shared/src/server/ingestion/validateAndInflateScore.ts](packages/shared/src/server/ingestion/validateAndInflateScore.ts#L77-L84)

```typescript
// Lines 77-84: mapStringValueToNumericValue（8 行）
function mapStringValueToNumericValue(
  config: ScoreConfigDomain,
  label: string,
): number {
  return (
    config.categories?.find((category) => category.label === label)?.value ?? 0
  );
}
```

**映射示例**：

```typescript
const config = {
  name: "quality",
  dataType: "CATEGORICAL",
  categories: [
    { label: "excellent", value: 5 },
    { label: "good", value: 4 },
    { label: "poor", value: 2 },
  ],
};

mapStringValueToNumericValue(config, "excellent");  // 返回：5
mapStringValueToNumericValue(config, "good");       // 返回：4
mapStringValueToNumericValue(config, "unknown");    // 返回：0（默认值）
```

**映射流程（inflateScoreBody）**：

| 输入类型 | 数据类型 | value | stringValue |
|---------|---------|-------|------------|
| **number** | NUMERIC | `4.5` | `null` |
| **number** | BOOLEAN | `1` | `"True"` |
| **string** | CATEGORICAL | `5`（映射） | `"good"` |

---

## 审计日志

### **配置创建审计**

```typescript
// Lines 98-103 (create): 审计日志
await auditLog({
  session: ctx.session,
  resourceType: "scoreConfig",
  resourceId: config.id,
  action: "create",
  after: config,
});
```

---

### **配置更新审计**

```typescript
// Lines 145-151 (update): 审计日志
await auditLog({
  session: ctx.session,
  resourceType: "scoreConfig",
  resourceId: config.id,
  action: "update",
  before: existingConfig,
  after: config,
});
```

**审计字段**：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `resourceType` | `"scoreConfig"` | 资源类型 |
| `resourceId` | `string` | 配置 ID |
| `action` | `"create" \| "update"` | 操作类型 |
| `before` | `ScoreConfig` | 更新前的配置（仅 update） |
| `after` | `ScoreConfig` | 更新后的配置 |
| `session` | `Session` | 操作用户信息 |

---

## 使用示例

### **场景 1：创建 NUMERIC 配置**

```typescript
const result = await trpc.scoreConfigs.create({
  projectId: "proj_123",
  name: "quality",
  dataType: "NUMERIC",
  minValue: 0,
  maxValue: 5,
  description: "Quality rating from 0 to 5",
});

console.log(result);
// {
//   id: "cfg_456",
//   projectId: "proj_123",
//   name: "quality",
//   dataType: "NUMERIC",
//   minValue: 0,
//   maxValue: 5,
//   categories: null,
//   description: "Quality rating from 0 to 5",
//   isArchived: false,
//   createdAt: "2024-01-01T00:00:00Z",
//   updatedAt: "2024-01-01T00:00:00Z",
// }
```

---

### **场景 2：创建 CATEGORICAL 配置**

```typescript
const result = await trpc.scoreConfigs.create({
  projectId: "proj_123",
  name: "sentiment",
  dataType: "CATEGORICAL",
  categories: [
    { label: "positive", value: 5 },
    { label: "neutral", value: 3 },
    { label: "negative", value: 1 },
  ],
  description: "Sentiment analysis",
});

console.log(result);
// {
//   id: "cfg_789",
//   name: "sentiment",
//   dataType: "CATEGORICAL",
//   categories: [
//     { label: "positive", value: 5 },
//     { label: "neutral", value: 3 },
//     { label: "negative", value: 1 },
//   ],
//   ...
// }
```

---

### **场景 3：验证评分（INGESTION）**

```typescript
// 配置
const config = await trpc.scoreConfigs.byId({
  projectId: "proj_123",
  id: "cfg_456",
});

// 验证评分
try {
  validateConfigAgainstBody({
    body: {
      name: "quality",
      dataType: "NUMERIC",
      value: 4.5,
    },
    config,
    context: "INGESTION",
  });
  console.log("✅ 验证通过");
} catch (error) {
  console.error("❌ 验证失败:", error.message);
}
```

---

### **场景 4：验证评分（ANNOTATION）**

```typescript
// 配置
const config = await trpc.scoreConfigs.byId({
  projectId: "proj_123",
  id: "cfg_789",
});

// 验证评分
try {
  validateConfigAgainstBody({
    body: {
      name: "sentiment",
      dataType: "CATEGORICAL",
      stringValue: "positive",  // 注意：使用 stringValue
    },
    config,
    context: "ANNOTATION",
  });
  console.log("✅ 验证通过，映射值为:", 5);
} catch (error) {
  console.error("❌ 验证失败:", error.message);
}
```

---

### **场景 5：更新配置**

```typescript
const result = await trpc.scoreConfigs.update({
  projectId: "proj_123",
  id: "cfg_456",
  maxValue: 10,  // 修改最大值 5 → 10
});

console.log(result);
// {
//   id: "cfg_456",
//   name: "quality",
//   maxValue: 10,  // 已更新
//   updatedAt: "2024-01-02T00:00:00Z",  // 更新时间
//   ...
// }
```

---

### **场景 6：归档配置**

```typescript
// 归档配置
await trpc.scoreConfigs.update({
  projectId: "proj_123",
  id: "cfg_456",
  isArchived: true,
});

// 尝试使用归档配置创建评分
try {
  validateConfigAgainstBody({
    body: { name: "quality", value: 4.5 },
    config: { ...config, isArchived: true },
    context: "INGESTION",
  });
} catch (error) {
  console.error(error.message);
  // "Config is archived and cannot be used to create new scores. Please restore the config first."
}
```

---

## 错误处理

### **常见错误**

| 错误类型 | 错误信息 | 原因 |
|---------|---------|------|
| **数据类型不匹配** | `Data type mismatch: expected NUMERIC, got CATEGORICAL` | body.dataType ≠ config.dataType |
| **范围超限** | `Score value 10 exceeds max value 5` | value > maxValue |
| **类别不存在** | `Category 'unknown' not found in config` | stringValue 不在 categories 中 |
| **配置已归档** | `Config is archived and cannot be used` | isArchived = true |
| **名称不匹配** | `Name mismatch: expected quality, got correctness` | body.name ≠ config.name |
| **配置不存在** | `No score config with this id in this project` | configId 无效 |

---

### **错误处理示例**

```typescript
try {
  validateConfigAgainstBody({
    body: { name: "quality", value: 10, dataType: "NUMERIC" },
    config: { name: "quality", dataType: "NUMERIC", minValue: 0, maxValue: 5 },
    context: "INGESTION",
  });
} catch (error) {
  if (error instanceof InvalidRequestError) {
    console.error("❌ 验证失败:", error.message);
    // "Score value 10 exceeds max value 5"
  }
}
```

---

## 数据流

### **配置创建 → 评分验证流程**

```
┌──────────────┐
│ 1. 创建配置  │
│ (create)     │
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│ 2. 保存到 Prisma │
│ (scoreConfig)    │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ 3. 审计日志      │
│ (auditLog)       │
└──────┬───────────┘
       │
       ▼
┌──────────────────────────┐
│ 4. 创建评分时验证        │
│ (validateConfigAgainstBody) │
└──────┬───────────────────┘
       │
       ├─ ✅ 验证通过 → 创建评分
       │
       └─ ❌ 验证失败 → 抛出异常
```

---

## 权限控制

### **权限范围**

| 操作 | 权限 | 说明 |
|-----|------|------|
| **查询配置** | `scoreConfigs:read` | 读取配置列表/详情 |
| **创建配置** | `scoreConfigs:CUD` | 创建新配置 |
| **更新配置** | `scoreConfigs:CUD` | 修改现有配置 |
| **归档配置** | `scoreConfigs:CUD` | 软删除配置 |

---

## 监控指标

### **配置统计**

| 指标 | 计算方式 | 用途 |
|-----|---------|------|
| **配置总数** | `totalCount` | 监控配置数量 |
| **归档配置数** | `count(isArchived=true)` | 识别废弃配置 |
| **配置使用率** | `scores.configId != null / totalScores` | 评估配置覆盖率 |
| **验证失败率** | `validationErrors / totalScoreAttempts` | 监控验证错误 |

---

## 故障排查

### **常见问题**

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| **评分创建失败** | 值超出范围 | 检查 `minValue/maxValue` |
| **类别未找到** | 标签不在类别列表 | 检查 `categories` |
| **配置归档错误** | `isArchived=true` | 恢复配置或创建新配置 |
| **名称不匹配** | `body.name ≠ config.name` | 使用正确的配置 ID |
| **数据类型错误** | `NUMERIC` vs `CATEGORICAL` | 检查 `dataType` 一致性 |

---

## 相关文档

- [Annotation Scoring 管理](annotation-scoring.md) - 标注评分机制
- [Score Aggregation 管理](score-aggregation.md) - 评分聚合算法
- [01-score-create-sequence.puml](01-score-create-sequence.puml) - 评分创建时序图
