# Score Aggregation 管理

## 功能概述

Score Aggregation 是 Scores 模块的**评分聚合引擎**，将多个评分合并为统计摘要，用于仪表盘、表格和报表展示。通过分组聚合、类型推断、键规范化等机制，支持数值型评分的平均值计算和分类型评分的分布统计。核心特性：
- **自动分组**：按 name + source + dataType 分组
- **双模式聚合**：数值型（平均值）和分类型（计数分布）
- **键规范化**：统一评分名称格式（`name-source-dataType`）
- **单值优化**：单个评分时保留完整元数据
- **类型推断**：自动处理 BOOLEAN 类型转换

---

## 核心组件

### 1. **aggregateScores 主聚合函数**

**位置**：[web/src/features/scores/lib/aggregateScores.ts](web/src/features/scores/lib/aggregateScores.ts#L66-L130)

#### **函数实现（65 行）**

```typescript
// Lines 66-130: aggregateScores 函数（65 行）
const aggregateScores = <T extends ScoreToAggregate>(
  scores: T[],
): ScoreAggregate => {
  // 步骤 1: 按聚合键分组（Lines 69-84）
  const groupedScores: Record<string, T[]> = scores.reduce(
    (acc, score) => {
      const key = composeAggregateScoreKey({
        name: score.name,
        source: score.source,
        dataType: score.dataType,
      });
      if (!acc[key]) {
        acc[key] = [];
      }
      acc[key].push(score);
      return acc;
    },
    {} as Record<string, T[]>,
  );

  // 步骤 2: 对每个组计算聚合（Lines 86-128）
  /* IMPORTANT
   * 单值聚合包含额外字段：comment, id, hasMetadata
   * 多值聚合时这些字段为 undefined
   */
  return Object.entries(groupedScores).reduce((acc, [key, scores]) => {
    const aggregateType = resolveAggregateType(scores[0].dataType);
    
    // 分支 A: NUMERIC 聚合（Lines 93-103）
    if (aggregateType === "NUMERIC") {
      const values = scores.map((score) => score.value ?? 0);
      if (!Boolean(values.length)) return acc;
      
      const average = values.reduce((a, b) => a + b, 0) / values.length;
      
      acc[key] = {
        type: aggregateType,
        values,
        average,
        comment: values.length === 1 ? scores[0].comment : undefined,
        id: values.length === 1 ? scores[0].id : undefined,
        hasMetadata: values.length === 1 ? scores[0].hasMetadata : undefined,
        timestamp: values.length === 1 ? scores[0].timestamp : undefined,
      };
    } 
    // 分支 B: CATEGORICAL 聚合（Lines 104-125）
    else {
      const values = scores.map((score) => score.stringValue ?? "n/a");
      if (!Boolean(values.length)) return acc;
      
      const valueCounts = values.reduce(
        (acc, value) => {
          acc[value] = (acc[value] || 0) + 1;
          return acc;
        },
        {} as Record<string, number>,
      );
      
      acc[key] = {
        type: aggregateType,
        values,
        valueCounts: Object.entries(valueCounts).map(([value, count]) => ({
          value,
          count,
        })),
        comment: values.length === 1 ? scores[0].comment : undefined,
        id: values.length === 1 ? scores[0].id : undefined,
        hasMetadata: values.length === 1 ? scores[0].hasMetadata : undefined,
        timestamp: values.length === 1 ? scores[0].timestamp : undefined,
      };
    }
    
    return acc;
  }, {} as ScoreAggregate);
};
```

**2 步工作流**：

| 步骤 | 操作 | 代码行 | 作用 |
|-----|------|-------|------|
| 1 | 分组 | 69-84 | 按 `name-source-dataType` 分组 |
| 2 | 聚合 | 86-128 | 计算统计值（平均/分布） |

---

### 2. **composeAggregateScoreKey 键生成**

**位置**：[web/src/features/scores/lib/aggregateScores.ts](web/src/features/scores/lib/aggregateScores.ts#L17-L29)

#### **函数实现（13 行）**

```typescript
// Lines 17-29: composeAggregateScoreKey 函数（13 行）
const composeAggregateScoreKey = ({
  name,
  source,
  dataType,
}: {
  name: string;
  source: ScoreSourceType;
  dataType: ScoreDataType;
  keyPrefix?: string;
}): string => {
  const formattedName = normalizeScoreName(name);
  return `${formattedName}-${source}-${dataType}`;
};
```

**键格式示例**：

| 输入 | 聚合键 |
|-----|-------|
| `name="quality", source="ANNOTATION", dataType="NUMERIC"` | `quality-ANNOTATION-NUMERIC` |
| `name="user.rating", source="API", dataType="CATEGORICAL"` | `user_rating-API-CATEGORICAL` |
| `name="is-correct", source="EVAL", dataType="BOOLEAN"` | `is_correct-EVAL-BOOLEAN` |

---

### 3. **normalizeScoreName 名称规范化**

**位置**：[web/src/features/scores/lib/aggregateScores.ts](web/src/features/scores/lib/aggregateScores.ts#L13-L15)

#### **函数实现（3 行）**

```typescript
// Lines 13-15: normalizeScoreName 函数（3 行）
const normalizeScoreName = (name: string): string => {
  return name.replaceAll(/[-\.]/g, "_");
};
```

**规范化规则**：
- 替换 `-` → `_`
- 替换 `.` → `_`
- 其他字符保持不变

**示例**：

| 原始名称 | 规范化后 |
|---------|---------|
| `user-rating` | `user_rating` |
| `llm.quality` | `llm_quality` |
| `is-correct-answer` | `is_correct_answer` |
| `quality_score` | `quality_score`（无变化） |

**原因**：
- 避免键中的特殊字符
- 统一命名风格
- 兼容对象属性访问（`obj.quality-score` 语法错误）

---

### 4. **resolveAggregateType 类型推断**

**位置**：[web/src/features/scores/lib/aggregateScores.ts](web/src/features/scores/lib/aggregateScores.ts#L60-L64)

#### **函数实现（5 行）**

```typescript
// Lines 60-64: resolveAggregateType 函数（5 行）
const resolveAggregateType = (
  dataType: ScoreDataType,
): "NUMERIC" | "CATEGORICAL" => {
  return dataType === "BOOLEAN" ? "CATEGORICAL" : dataType;
};
```

**类型映射**：

| 输入 DataType | 输出 AggregateType | 理由 |
|--------------|-------------------|------|
| **NUMERIC** | `NUMERIC` | 数值型，计算平均值 |
| **CATEGORICAL** | `CATEGORICAL` | 分类型，统计分布 |
| **BOOLEAN** | `CATEGORICAL` | 布尔型 = 特殊分类型 |

**为什么 BOOLEAN → CATEGORICAL**：
- BOOLEAN 本质是 2 个类别：`{ "True", "False" }`
- 聚合逻辑相同：统计各类别数量
- 统一处理简化代码

---

## 聚合类型

### **NUMERIC 聚合**

#### **数据结构**

```typescript
type NumericAggregate = {
  type: "NUMERIC";
  values: number[];           // 所有值的数组
  average: number;            // 平均值
  comment?: string;           // 单值时的评论
  id?: string;                // 单值时的 ID
  hasMetadata?: boolean;      // 单值时是否有元数据
  timestamp?: Date;           // 单值时的时间戳
};
```

#### **计算逻辑（Lines 93-103）**

```typescript
// 提取所有值
const values = scores.map((score) => score.value ?? 0);

// 计算平均值
const average = values.reduce((a, b) => a + b, 0) / values.length;

// 构建聚合
acc[key] = {
  type: "NUMERIC",
  values,
  average,
  comment: values.length === 1 ? scores[0].comment : undefined,
  id: values.length === 1 ? scores[0].id : undefined,
  hasMetadata: values.length === 1 ? scores[0].hasMetadata : undefined,
  timestamp: values.length === 1 ? scores[0].timestamp : undefined,
};
```

#### **示例**

**输入**：
```typescript
[
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 4.5 },
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 5.0 },
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 3.5 },
]
```

**输出**：
```typescript
{
  "quality-ANNOTATION-NUMERIC": {
    type: "NUMERIC",
    values: [4.5, 5.0, 3.5],
    average: 4.333,
    comment: undefined,  // 多值，无评论
    id: undefined,
    hasMetadata: undefined,
    timestamp: undefined,
  }
}
```

---

### **CATEGORICAL 聚合**

#### **数据结构**

```typescript
type CategoricalAggregate = {
  type: "CATEGORICAL";
  values: string[];           // 所有值的数组
  valueCounts: Array<{        // 各类别的计数
    value: string;
    count: number;
  }>;
  comment?: string;           // 单值时的评论
  id?: string;                // 单值时的 ID
  hasMetadata?: boolean;      // 单值时是否有元数据
  timestamp?: Date;           // 单值时的时间戳
};
```

#### **计算逻辑（Lines 104-125）**

```typescript
// 提取所有值
const values = scores.map((score) => score.stringValue ?? "n/a");

// 统计各类别数量
const valueCounts = values.reduce(
  (acc, value) => {
    acc[value] = (acc[value] || 0) + 1;
    return acc;
  },
  {} as Record<string, number>,
);

// 构建聚合
acc[key] = {
  type: "CATEGORICAL",
  values,
  valueCounts: Object.entries(valueCounts).map(([value, count]) => ({
    value,
    count,
  })),
  comment: values.length === 1 ? scores[0].comment : undefined,
  id: values.length === 1 ? scores[0].id : undefined,
  hasMetadata: values.length === 1 ? scores[0].hasMetadata : undefined,
  timestamp: values.length === 1 ? scores[0].timestamp : undefined,
};
```

#### **示例**

**输入**：
```typescript
[
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "positive" },
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "positive" },
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "neutral" },
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "positive" },
]
```

**输出**：
```typescript
{
  "sentiment-API-CATEGORICAL": {
    type: "CATEGORICAL",
    values: ["positive", "positive", "neutral", "positive"],
    valueCounts: [
      { value: "positive", count: 3 },
      { value: "neutral", count: 1 },
    ],
    comment: undefined,
    id: undefined,
    hasMetadata: undefined,
    timestamp: undefined,
  }
}
```

---

### **BOOLEAN 聚合（作为 CATEGORICAL 处理）**

#### **示例**

**输入**：
```typescript
[
  { name: "is_correct", source: "EVAL", dataType: "BOOLEAN", stringValue: "True" },
  { name: "is_correct", source: "EVAL", dataType: "BOOLEAN", stringValue: "True" },
  { name: "is_correct", source: "EVAL", dataType: "BOOLEAN", stringValue: "False" },
]
```

**输出**：
```typescript
{
  "is_correct-EVAL-BOOLEAN": {
    type: "CATEGORICAL",  // 注意：类型为 CATEGORICAL
    values: ["True", "True", "False"],
    valueCounts: [
      { value: "True", count: 2 },
      { value: "False", count: 1 },
    ],
    comment: undefined,
    id: undefined,
    hasMetadata: undefined,
    timestamp: undefined,
  }
}
```

---

## 单值优化

### **为什么保留元数据**

```typescript
// Lines 99-102, 121-124: 单值时保留元数据
comment: values.length === 1 ? scores[0].comment : undefined,
id: values.length === 1 ? scores[0].id : undefined,
hasMetadata: values.length === 1 ? scores[0].hasMetadata : undefined,
timestamp: values.length === 1 ? scores[0].timestamp : undefined,
```

**单值场景**：
- Trace 只有一个评分
- 某个评分来源（source）只有一个评分

**保留元数据的作用**：
- **comment**：显示评论内容
- **id**：支持点击跳转到评分详情
- **hasMetadata**：显示元数据图标
- **timestamp**：显示评分时间

**多值场景**：
- 无法确定显示哪个评论/ID
- 元数据字段设为 `undefined`

---

## 分组逻辑

### **分组键组成**

```
聚合键 = normalizeScoreName(name) + "-" + source + "-" + dataType
```

### **为什么按 name + source + dataType 分组**

| 维度 | 作用 | 示例 |
|-----|------|------|
| **name** | 区分不同的评分指标 | `quality` vs `correctness` |
| **source** | 区分评分来源 | `ANNOTATION` vs `API` vs `EVAL` |
| **dataType** | 区分数据类型 | `NUMERIC` vs `CATEGORICAL` |

**示例场景**：
```typescript
// 同名但来源不同 → 分开聚合
[
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 4 },
  { name: "quality", source: "API", dataType: "NUMERIC", value: 3 },
]

// 结果：
{
  "quality-ANNOTATION-NUMERIC": { type: "NUMERIC", average: 4, ... },
  "quality-API-NUMERIC": { type: "NUMERIC", average: 3, ... }
}
```

---

## 使用示例

### **场景 1：数值评分聚合**

```typescript
const scores = [
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 4.5, id: "s1", comment: "Good" },
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 5.0, id: "s2" },
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 3.5, id: "s3" },
];

const aggregated = aggregateScores(scores);

console.log(aggregated);
// {
//   "quality-ANNOTATION-NUMERIC": {
//     type: "NUMERIC",
//     values: [4.5, 5.0, 3.5],
//     average: 4.333,
//     comment: undefined,  // 多个评分，无法确定
//     id: undefined,
//     hasMetadata: undefined,
//     timestamp: undefined,
//   }
// }
```

---

### **场景 2：分类评分聚合**

```typescript
const scores = [
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "positive" },
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "neutral" },
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "positive" },
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "negative" },
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "positive" },
];

const aggregated = aggregateScores(scores);

console.log(aggregated);
// {
//   "sentiment-API-CATEGORICAL": {
//     type: "CATEGORICAL",
//     values: ["positive", "neutral", "positive", "negative", "positive"],
//     valueCounts: [
//       { value: "positive", count: 3 },
//       { value: "neutral", count: 1 },
//       { value: "negative", count: 1 },
//     ],
//     comment: undefined,
//     id: undefined,
//     hasMetadata: undefined,
//     timestamp: undefined,
//   }
// }
```

---

### **场景 3：混合评分聚合**

```typescript
const scores = [
  // 数值评分
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 4.5 },
  { name: "quality", source: "ANNOTATION", dataType: "NUMERIC", value: 5.0 },
  
  // 分类评分
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "positive" },
  { name: "sentiment", source: "API", dataType: "CATEGORICAL", stringValue: "neutral" },
  
  // 布尔评分
  { name: "is_correct", source: "EVAL", dataType: "BOOLEAN", stringValue: "True" },
  { name: "is_correct", source: "EVAL", dataType: "BOOLEAN", stringValue: "False" },
  
  // 同名但不同来源
  { name: "quality", source: "API", dataType: "NUMERIC", value: 3.0 },
];

const aggregated = aggregateScores(scores);

console.log(aggregated);
// {
//   "quality-ANNOTATION-NUMERIC": {
//     type: "NUMERIC",
//     values: [4.5, 5.0],
//     average: 4.75,
//     ...
//   },
//   "quality-API-NUMERIC": {
//     type: "NUMERIC",
//     values: [3.0],
//     average: 3.0,
//     comment: ...,  // 单值，保留元数据
//     id: ...,
//     ...
//   },
//   "sentiment-API-CATEGORICAL": {
//     type: "CATEGORICAL",
//     values: ["positive", "neutral"],
//     valueCounts: [
//       { value: "positive", count: 1 },
//       { value: "neutral", count: 1 },
//     ],
//     ...
//   },
//   "is_correct-EVAL-BOOLEAN": {
//     type: "CATEGORICAL",  // BOOLEAN 转为 CATEGORICAL
//     values: ["True", "False"],
//     valueCounts: [
//       { value: "True", count: 1 },
//       { value: "False", count: 1 },
//     ],
//     ...
//   }
// }
```

---

### **场景 4：单值聚合（保留元数据）**

```typescript
const scores = [
  { 
    name: "quality", 
    source: "ANNOTATION", 
    dataType: "NUMERIC", 
    value: 4.5,
    id: "score_123",
    comment: "Good quality",
    hasMetadata: true,
    timestamp: new Date("2024-01-01"),
  },
];

const aggregated = aggregateScores(scores);

console.log(aggregated);
// {
//   "quality-ANNOTATION-NUMERIC": {
//     type: "NUMERIC",
//     values: [4.5],
//     average: 4.5,
//     comment: "Good quality",       // ✅ 保留
//     id: "score_123",               // ✅ 保留
//     hasMetadata: true,             // ✅ 保留
//     timestamp: 2024-01-01T00:00:00Z, // ✅ 保留
//   }
// }
```

---

## 键解析

### **decomposeAggregateScoreKey 反向解析**

**位置**：[web/src/features/scores/lib/aggregateScores.ts](web/src/features/scores/lib/aggregateScores.ts#L31-L42)

```typescript
// Lines 31-42: decomposeAggregateScoreKey 函数（12 行）
const decomposeAggregateScoreKey = (
  key: string,
): { name: string; source: ScoreSourceType; dataType: ScoreDataType } | null => {
  const parts = key.split("-");
  if (parts.length !== 3) return null;
  
  return {
    name: parts[0],
    source: parts[1] as ScoreSourceType,
    dataType: parts[2] as ScoreDataType,
  };
};
```

**示例**：
```typescript
decomposeAggregateScoreKey("quality-ANNOTATION-NUMERIC");
// 返回：{ name: "quality", source: "ANNOTATION", dataType: "NUMERIC" }

decomposeAggregateScoreKey("user_rating-API-CATEGORICAL");
// 返回：{ name: "user_rating", source: "API", dataType: "CATEGORICAL" }

decomposeAggregateScoreKey("invalid");
// 返回：null（格式错误）
```

---

### **getScoreLabelFromKey 获取显示标签**

**位置**：[web/src/features/scores/lib/aggregateScores.ts](web/src/features/scores/lib/aggregateScores.ts#L44-L58)

```typescript
// Lines 44-58: getScoreLabelFromKey 函数（15 行）
const getScoreLabelFromKey = (key: string): string => {
  const decomposed = decomposeAggregateScoreKey(key);
  
  if (!decomposed) return key;
  
  // 构建友好的标签
  const { name, source } = decomposed;
  
  return source === "ANNOTATION" 
    ? name 
    : `${name} (${source})`;
};
```

**示例**：
```typescript
getScoreLabelFromKey("quality-ANNOTATION-NUMERIC");
// 返回："quality"（标注评分不显示来源）

getScoreLabelFromKey("quality-API-NUMERIC");
// 返回："quality (API)"（非标注评分显示来源）

getScoreLabelFromKey("sentiment-EVAL-CATEGORICAL");
// 返回："sentiment (EVAL)"
```

---

## 性能优化

### **时间复杂度**

| 操作 | 复杂度 | 说明 |
|-----|-------|------|
| **分组** | O(n) | 遍历所有评分 |
| **聚合** | O(n) | 计算每组的统计值 |
| **总计** | O(n) | 线性时间复杂度 |

**n = 评分总数**

---

### **空间复杂度**

| 数据结构 | 复杂度 | 说明 |
|---------|-------|------|
| **groupedScores** | O(n) | 存储所有评分 |
| **聚合结果** | O(g) | g = 分组数量 |
| **总计** | O(n) | 主要开销在输入数据 |

---

### **优化策略**

| 策略 | 实现 | 效果 |
|-----|------|------|
| **单次遍历** | `reduce` 分组 | 避免多次遍历 |
| **提前返回** | `if (!values.length) return acc` | 跳过空组 |
| **类型推断缓存** | `resolveAggregateType` | 避免重复判断 |

---

## 监控指标

### **聚合统计**

| 指标 | 计算方式 | 用途 |
|-----|---------|------|
| **分组数量** | `Object.keys(aggregated).length` | 监控评分多样性 |
| **平均组大小** | `totalScores / groupCount` | 评估评分密度 |
| **单值组比例** | `count(values.length === 1) / groupCount` | 识别稀疏评分 |

---

## 故障排查

### **常见问题**

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| **分组错误** | 键格式不一致 | 检查 `normalizeScoreName` 逻辑 |
| **平均值错误** | `value` 为 null | 使用 `?? 0` 默认值 |
| **类别缺失** | `stringValue` 为 null | 使用 `?? "n/a"` 默认值 |
| **元数据丢失** | 多值聚合 | 正常现象，仅单值保留 |

---

## 相关文档

- [Annotation Scoring 管理](annotation-scoring.md) - 标注评分机制
- [Config Management 管理](config-management.md) - 评分配置管理
- [02-score-aggregation-query-sequence.puml](02-score-aggregation-query-sequence.puml) - 聚合查询时序图
