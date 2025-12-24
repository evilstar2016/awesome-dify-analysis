# Annotation Scoring 管理

## 功能概述

Annotation Scoring 是 Scores 模块的**人工标注评分系统**，支持质量评估团队对 Trace、Observation 或 Session 进行手动评分。通过标注界面、评分配置、评论功能等，实现结构化的质量评估工作流。核心特性：
- **多目标支持**：Trace、Observation、Session 三种评分目标
- **评分配置**：基于 Score Config 的类型验证
- **评论功能**：支持文本评论和元数据
- **队列集成**：与标注队列（Annotation Queue）关联
- **审计追踪**：完整记录评分历史和修改者

---

## 核心组件

### 1. **createAnnotationScore 创建标注评分**

**位置**：[web/src/server/api/routers/scores.ts](web/src/server/api/routers/scores.ts#L297-L426)

#### **主流程（130 行）**

```typescript
// Lines 297-426: createAnnotationScore 路由（130 行）
createAnnotationScore: protectedProjectProcedure
  .input(CreateAnnotationScoreData)
  .mutation(async ({ input, ctx }) => {
    // 步骤 1: 权限验证（Lines 301-305）
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId: input.projectId,
      scope: "scores:CUD",
    });

    // 步骤 2: 解析评分目标（Lines 307-316）
    const inflatedParams = isTraceScore(input.scoreTarget)
      ? {
          observationId: input.scoreTarget.observationId ?? null,
          traceId: input.scoreTarget.traceId,
          sessionId: null,
        }
      : {
          observationId: null,
          traceId: null,
          sessionId: input.scoreTarget.sessionId,
        };

    // 步骤 3: 验证目标存在性（Lines 318-351）
    if (inflatedParams.traceId) {
      const clickhouseTrace = await getTraceById({
        traceId: inflatedParams.traceId,
        projectId: input.projectId,
        clickhouseFeatureTag: "annotations-trpc",
      });

      if (!clickhouseTrace) {
        throw new LangfuseNotFoundError(
          `No trace with id ${inflatedParams.traceId}...`
        );
      }
    } else if (inflatedParams.sessionId) {
      const traceIdentifiers = await getTracesIdentifierForSession(
        input.projectId,
        inflatedParams.sessionId,
      );
      if (traceIdentifiers.length === 0) {
        throw new LangfuseNotFoundError(
          `No trace referencing session...`
        );
      }
    }

    // 步骤 4: 检查是否存在相同评分（Lines 353-360）
    const clickhouseScore = await searchExistingAnnotationScore(
      input.projectId,
      inflatedParams.observationId,
      inflatedParams.traceId,
      inflatedParams.sessionId,
      input.name,
      input.configId,
      input.dataType,
    );

    const timestamp = input.timestamp ?? new Date();

    // 步骤 5: 构建评分对象（Lines 364-397）
    const score = !!clickhouseScore
      ? {
          // 更新现有评分
          ...clickhouseScore,
          value: input.value,
          stringValue: input.stringValue ?? null,
          comment: input.comment ?? null,
          metadata: {},
          authorUserId: ctx.session.user.id,
          queueId: input.queueId ?? null,
          timestamp,
        }
      : {
          // 创建新评分
          id: input.id ?? v4(),
          projectId: input.projectId,
          environment: input.environment ?? "default",
          ...inflatedParams,
          datasetRunId: null,  // 标注评分不关联 dataset run
          value: input.value,
          stringValue: input.stringValue ?? null,
          dataType: input.dataType ?? null,
          configId: input.configId ?? null,
          name: input.name,
          comment: input.comment ?? null,
          metadata: {},
          authorUserId: ctx.session.user.id,
          source: ScoreSourceEnum.ANNOTATION,
          queueId: input.queueId ?? null,
          executionTraceId: null,
          createdAt: new Date(),
          updatedAt: new Date(),
          timestamp,
        };

    // 步骤 6: 写入 ClickHouse（Lines 399-416）
    await upsertScore({
      id: score.id,
      timestamp: convertDateToClickhouseDateTime(timestamp),
      project_id: input.projectId,
      environment: input.environment ?? "default",
      trace_id: inflatedParams.traceId,
      observation_id: inflatedParams.observationId,
      session_id: inflatedParams.sessionId,
      name: input.name,
      value: input.value,
      source: ScoreSourceEnum.ANNOTATION,
      comment: input.comment,
      author_user_id: ctx.session.user.id,
      config_id: input.configId,
      data_type: input.dataType,
      string_value: input.stringValue,
      queue_id: input.queueId,
      created_at: convertDateToClickhouseDateTime(score.createdAt),
      updated_at: convertDateToClickhouseDateTime(score.updatedAt),
      metadata: score.metadata as Record<string, string>,
    });

    // 步骤 7: 审计日志（Lines 418-423）
    await auditLog({
      session: ctx.session,
      resourceType: "score",
      resourceId: score.id,
      action: "create",
      after: score,
    });

    return validateDbScore(score);
  }),
```

**7 步工作流**：

| 步骤 | 操作 | 代码行 | 作用 |
|-----|------|-------|------|
| 1 | 权限验证 | 301-305 | 检查 `scores:CUD` 权限 |
| 2 | 解析目标 | 307-316 | 区分 Trace/Session 评分 |
| 3 | 验证存在性 | 318-351 | 确保目标存在于 ClickHouse |
| 4 | 检查重复 | 353-360 | 查找相同 name+configId 的评分 |
| 5 | 构建对象 | 364-397 | 创建/更新评分数据 |
| 6 | 写入数据库 | 399-416 | Upsert 到 ClickHouse |
| 7 | 审计日志 | 418-423 | 记录操作历史 |

---

### 2. **updateAnnotationScore 更新标注评分**

**位置**：[web/src/server/api/routers/scores.ts](web/src/server/api/routers/scores.ts#L427-L686)

#### **主流程（260 行）**

```typescript
// Lines 427-686: updateAnnotationScore 路由（260 行）
updateAnnotationScore: protectedProjectProcedure
  .input(UpdateAnnotationScoreData)
  .mutation(async ({ input, ctx }) => {
    // 步骤 1: 权限验证（Lines 431-435）
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId: input.projectId,
      scope: "scores:CUD",
    });

    let updatedScore: ScoreDomain | null | undefined = null;

    // 步骤 2: 查询现有评分（Lines 439-445）
    const score = await getScoreById({
      projectId: input.projectId,
      scoreId: input.id,
      source: ScoreSourceEnum.ANNOTATION,
    });

    // 分支 A: 评分不存在（Lines 447-594）
    if (!score) {
      // A1: 检查是否提供 timestamp（Lines 448-454）
      if (!input.timestamp) {
        throw new LangfuseNotFoundError(
          `No annotation score with id ${input.id}...`
        );
      }

      logger.info(
        `Score ${input.id} not found, upserting with provided timestamp`
      );

      // A2: 验证配置（Lines 458-468）
      const config = await ctx.prisma.scoreConfig.findFirst({
        where: {
          id: input.configId,
          projectId: input.projectId,
        },
      });
      if (!config) {
        throw new LangfuseNotFoundError(
          `No score config with id ${input.configId}...`
        );
      }

      // A3: 解析并验证目标（Lines 471-514）
      const inflatedParams = isTraceScore(input.scoreTarget)
        ? {
            observationId: input.scoreTarget.observationId ?? null,
            traceId: input.scoreTarget.traceId,
            sessionId: null,
          }
        : {
            observationId: null,
            traceId: null,
            sessionId: input.scoreTarget.sessionId,
          };

      if (inflatedParams.traceId) {
        const clickhouseTrace = await getTraceById({
          traceId: inflatedParams.traceId,
          projectId: input.projectId,
          clickhouseFeatureTag: "annotations-trpc",
        });

        if (!clickhouseTrace) {
          throw new LangfuseNotFoundError(
            `No trace with id ${inflatedParams.traceId}...`
          );
        }
      } else if (inflatedParams.sessionId) {
        const traceIdentifiers = await getTracesIdentifierForSession(
          input.projectId,
          inflatedParams.sessionId,
        );
        if (traceIdentifiers.length === 0) {
          throw new LangfuseNotFoundError(
            `No trace referencing session...`
          );
        }
      }

      // A4: Upsert 评分（Lines 516-534）
      await upsertScore({
        id: input.id,
        timestamp: convertDateToClickhouseDateTime(input.timestamp),
        project_id: input.projectId,
        environment: input.environment ?? "default",
        trace_id: inflatedParams.traceId,
        observation_id: inflatedParams.observationId,
        session_id: inflatedParams.sessionId,
        name: input.name,
        value: input.value,
        source: ScoreSourceEnum.ANNOTATION,
        comment: input.comment,
        author_user_id: ctx.session.user.id,
        config_id: input.configId,
        data_type: input.dataType,
        string_value: input.stringValue,
        queue_id: input.queueId,
        created_at: convertDateToClickhouseDateTime(new Date()),
        updated_at: convertDateToClickhouseDateTime(new Date()),
        metadata: {},
      });

      // A5: 构建返回对象（Lines 536-574）
      const baseScore = {
        id: input.id,
        projectId: input.projectId,
        environment: input.environment ?? "default",
        traceId: inflatedParams.traceId,
        observationId: inflatedParams.observationId,
        sessionId: inflatedParams.sessionId,
        datasetRunId: null,
        name: input.name,
        value: input.value,
        dataType: input.dataType,
        configId: input.configId ?? null,
        metadata: {},
        executionTraceId: null,
        createdAt: new Date(),
        updatedAt: new Date(),
        source: ScoreSourceEnum.ANNOTATION,
        comment: input.comment ?? null,
        authorUserId: ctx.session.user.id,
        queueId: input.queueId ?? null,
        timestamp: input.timestamp,
      };

      if (isNumericDataType(baseScore.dataType)) {
        updatedScore = {
          ...baseScore,
          dataType: ScoreDataTypeEnum.NUMERIC,
          stringValue: null,
        };
      } else {
        updatedScore = {
          ...baseScore,
          dataType: input.dataType as "CATEGORICAL" | "BOOLEAN",
          stringValue: input.stringValue!,
        };
      }

      // A6: 审计日志（Lines 576-582）
      await auditLog({
        session: ctx.session,
        resourceType: "score",
        resourceId: input.id,
        action: "update",
        after: updatedScore,
      });
    } 
    // 分支 B: 评分存在（Lines 584-672）
    else {
      // B1: 验证配置（Lines 585-619）
      if (score.configId) {
        const config = await ctx.prisma.scoreConfig.findFirst({
          where: {
            id: score.configId,
            projectId: input.projectId,
          },
        });
        if (!config) {
          throw new LangfuseNotFoundError(
            `No score config with id ${score.configId}...`
          );
        }
        
        try {
          validateConfigAgainstBody({
            body: {
              ...score,
              value: input.value,
              stringValue: isNumericDataType(score.dataType)
                ? null
                : input.stringValue!,
              comment: input.comment ?? null,
            } as ScoreDomain,
            config: config as ScoreConfigDomain,
            context: "ANNOTATION",
          });
        } catch (_error) {
          throw new TRPCError({
            code: "PRECONDITION_FAILED",
            message: "Score does not comply with config schema...",
          });
        }
      }

      // B2: Upsert 评分（Lines 621-638）
      await upsertScore({
        id: input.id,
        project_id: input.projectId,
        timestamp: convertDateToClickhouseDateTime(score.timestamp),
        value: input.value !== null ? input.value : undefined,
        string_value: input.stringValue,
        comment: input.comment,
        author_user_id: ctx.session.user.id,
        queue_id: input.queueId,
        source: ScoreSourceEnum.ANNOTATION,
        name: score.name,
        data_type: score.dataType,
        config_id: score.configId,
        trace_id: score.traceId,
        observation_id: score.observationId,
        session_id: score.sessionId,
        environment: score.environment,
        created_at: convertDateToClickhouseDateTime(score.createdAt),
        updated_at: convertDateToClickhouseDateTime(score.updatedAt),
        metadata: score.metadata as Record<string, string>,
      });

      // B3: 构建返回对象（Lines 640-661）
      const baseScore = {
        ...score,
        value: input.value,
        comment: input.comment ?? null,
        authorUserId: ctx.session.user.id,
        queueId: input.queueId ?? null,
        timestamp: score.timestamp,
      };

      if (isNumericDataType(score.dataType)) {
        updatedScore = {
          ...baseScore,
          dataType: ScoreDataTypeEnum.NUMERIC,
          stringValue: null,
        };
      } else {
        updatedScore = {
          ...baseScore,
          dataType: input.dataType as "CATEGORICAL" | "BOOLEAN",
          stringValue: input.stringValue!,
        };
      }

      // B4: 审计日志（Lines 663-670）
      await auditLog({
        session: ctx.session,
        resourceType: "score",
        resourceId: input.id,
        action: "update",
        before: score,
        after: updatedScore,
      });
    }

    // 返回结果（Lines 674-682）
    if (!updatedScore) {
      throw new InternalServerError(
        `Annotation score could not be updated...`
      );
    }

    return validateDbScore(updatedScore);
  }),
```

**两种分支**：

| 分支 | 场景 | 步骤 |
|-----|------|------|
| **分支 A**（不存在） | ClickHouse 最终一致性延迟 | ① 验证 timestamp<br>② 验证配置<br>③ 验证目标<br>④ Upsert<br>⑤ 构建对象<br>⑥ 审计日志 |
| **分支 B**（存在） | 正常更新流程 | ① 验证配置<br>② Upsert<br>③ 构建对象<br>④ 审计日志 |

---

### 3. **deleteAnnotationScore 删除标注评分**

**位置**：[web/src/server/api/routers/scores.ts](web/src/server/api/routers/scores.ts#L687-L722)

#### **主流程（36 行）**

```typescript
// Lines 687-722: deleteAnnotationScore 路由（36 行）
deleteAnnotationScore: protectedProjectProcedure
  .input(z.object({ projectId: z.string(), id: z.string() }))
  .mutation(async ({ input, ctx }) => {
    // 1. 权限验证（Lines 691-695）
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId: input.projectId,
      scope: "scores:CUD",
    });

    // 2. 查询评分（Lines 697-701）
    const score = await getScoreById({
      projectId: input.projectId,
      scoreId: input.id,
      source: ScoreSourceEnum.ANNOTATION,
    });

    if (!score) {
      throw new LangfuseNotFoundError(
        `No annotation score with id ${input.id}...`
      );
    }

    // 3. 删除评分（Lines 709）
    await deleteScore(input.id, input.projectId);

    // 4. 审计日志（Lines 711-717）
    await auditLog({
      session: ctx.session,
      resourceType: "score",
      resourceId: input.id,
      action: "delete",
      before: score,
    });

    return score;
  }),
```

---

## 评分目标类型

### **ScoreTarget 类型定义**

```typescript
type ScoreTarget = 
  | { traceId: string; observationId?: string }  // Trace 评分
  | { sessionId: string };                       // Session 评分
```

### **目标解析逻辑**

```typescript
// Lines 307-316: inflatedParams 解析
const inflatedParams = isTraceScore(input.scoreTarget)
  ? {
      observationId: input.scoreTarget.observationId ?? null,
      traceId: input.scoreTarget.traceId,
      sessionId: null,
    }
  : {
      observationId: null,
      traceId: null,
      sessionId: input.scoreTarget.sessionId,
    };
```

**解析结果**：

| 输入 | traceId | observationId | sessionId |
|-----|---------|---------------|-----------|
| `{ traceId: "t1" }` | `"t1"` | `null` | `null` |
| `{ traceId: "t1", observationId: "o1" }` | `"t1"` | `"o1"` | `null` |
| `{ sessionId: "s1" }` | `null` | `null` | `"s1"` |

---

## 评分配置验证

### **validateConfigAgainstBody 函数**

```typescript
// 验证评分是否符合配置
validateConfigAgainstBody({
  body: {
    ...score,
    value: input.value,
    stringValue: input.stringValue,
    comment: input.comment,
  } as ScoreDomain,
  config: config as ScoreConfigDomain,
  context: "ANNOTATION",
});
```

**验证规则**：

| 数据类型 | 验证项 | 错误场景 |
|---------|-------|---------|
| **NUMERIC** | `minValue` ≤ `value` ≤ `maxValue` | 值超出范围 |
| **CATEGORICAL** | `stringValue` ∈ `config.categories` | 类别不存在 |
| **BOOLEAN** | `stringValue` ∈ `["True", "False"]` | 非布尔值 |

**示例**：
```typescript
// Config: NUMERIC, minValue=0, maxValue=5
input.value = 6;  // ❌ 抛出错误

// Config: CATEGORICAL, categories=["good", "bad", "neutral"]
input.stringValue = "excellent";  // ❌ 抛出错误

// Config: BOOLEAN
input.stringValue = "maybe";  // ❌ 抛出错误
```

---

## 数据存储

### **ClickHouse 存储**

```typescript
// Lines 399-416: upsertScore 参数
await upsertScore({
  id: score.id,
  timestamp: convertDateToClickhouseDateTime(timestamp),
  project_id: input.projectId,
  environment: input.environment ?? "default",
  trace_id: inflatedParams.traceId,
  observation_id: inflatedParams.observationId,
  session_id: inflatedParams.sessionId,
  name: input.name,
  value: input.value,
  source: ScoreSourceEnum.ANNOTATION,  // 固定为 ANNOTATION
  comment: input.comment,
  author_user_id: ctx.session.user.id,
  config_id: input.configId,
  data_type: input.dataType,
  string_value: input.stringValue,
  queue_id: input.queueId,
  created_at: convertDateToClickhouseDateTime(score.createdAt),
  updated_at: convertDateToClickhouseDateTime(score.updatedAt),
  metadata: score.metadata as Record<string, string>,
});
```

**字段说明**：

| 字段 | 类型 | 说明 |
|-----|------|------|
| **source** | `ANNOTATION` | 标注来源标识（固定值） |
| **author_user_id** | `string` | 标注者用户 ID |
| **queue_id** | `string?` | 标注队列 ID（可选） |
| **comment** | `string?` | 评论文本 |
| **metadata** | `object` | 元数据（当前为空对象） |

---

## 审计日志

### **创建评分审计**

```typescript
// Lines 418-423: 创建审计日志
await auditLog({
  session: ctx.session,
  resourceType: "score",
  resourceId: score.id,
  action: "create",
  after: score,  // 新评分的完整数据
});
```

### **更新评分审计**

```typescript
// Lines 663-670: 更新审计日志
await auditLog({
  session: ctx.session,
  resourceType: "score",
  resourceId: input.id,
  action: "update",
  before: score,       // 更新前的数据
  after: updatedScore, // 更新后的数据
});
```

### **删除评分审计**

```typescript
// Lines 711-717: 删除审计日志
await auditLog({
  session: ctx.session,
  resourceType: "score",
  resourceId: input.id,
  action: "delete",
  before: score,  // 被删除的评分数据
});
```

---

## 重复评分处理

### **searchExistingAnnotationScore 查询**

```typescript
// Lines 353-360: 检查重复评分
const clickhouseScore = await searchExistingAnnotationScore(
  input.projectId,
  inflatedParams.observationId,
  inflatedParams.traceId,
  inflatedParams.sessionId,
  input.name,
  input.configId,
  input.dataType,
);
```

**查询逻辑**：
- 匹配条件：`projectId` + `observationId/traceId/sessionId` + `name` + `configId` + `dataType`
- 返回结果：存在则返回现有评分，不存在则返回 `null`

**处理策略**：

| 场景 | 行为 |
|-----|------|
| **存在相同评分** | 更新现有评分（保留 ID） |
| **不存在** | 创建新评分（生成新 ID） |

**示例**：
```typescript
// 第一次创建
await createAnnotationScore({
  scoreTarget: { traceId: "t1" },
  name: "quality",
  configId: "conf1",
  value: 4,
});
// 结果：创建新评分，id = "score1"

// 第二次创建（相同 traceId + name + configId）
await createAnnotationScore({
  scoreTarget: { traceId: "t1" },
  name: "quality",
  configId: "conf1",
  value: 5,
});
// 结果：更新 "score1"，value = 5
```

---

## 使用示例

### **场景 1：创建 Trace 评分**

```typescript
await trpc.scores.createAnnotationScore.mutate({
  projectId: "proj_123",
  scoreTarget: {
    traceId: "trace_456",
    observationId: "obs_789",  // 可选
  },
  name: "quality",
  configId: "conf_numeric",
  dataType: "NUMERIC",
  value: 4.5,
  comment: "Good quality, minor issues",
  queueId: "queue_001",  // 来自标注队列
  timestamp: new Date(),
});
```

---

### **场景 2：创建 Session 评分**

```typescript
await trpc.scores.createAnnotationScore.mutate({
  projectId: "proj_123",
  scoreTarget: {
    sessionId: "session_abc",
  },
  name: "user_satisfaction",
  configId: "conf_categorical",
  dataType: "CATEGORICAL",
  stringValue: "satisfied",
  comment: "User reported positive experience",
});
```

---

### **场景 3：更新评分**

```typescript
await trpc.scores.updateAnnotationScore.mutate({
  projectId: "proj_123",
  id: "score_123",
  scoreTarget: {
    traceId: "trace_456",
  },
  name: "quality",
  configId: "conf_numeric",
  dataType: "NUMERIC",
  value: 5.0,  // 从 4.5 更新到 5.0
  comment: "Re-evaluated, excellent quality",
});
```

---

### **场景 4：布尔评分**

```typescript
await trpc.scores.createAnnotationScore.mutate({
  projectId: "proj_123",
  scoreTarget: {
    traceId: "trace_456",
  },
  name: "is_correct",
  configId: "conf_boolean",
  dataType: "BOOLEAN",
  stringValue: "True",
  comment: "Answer is factually correct",
});
```

---

### **场景 5：删除评分**

```typescript
await trpc.scores.deleteAnnotationScore.mutate({
  projectId: "proj_123",
  id: "score_123",
});

// 审计日志会记录：
// - action: "delete"
// - before: 被删除的评分完整数据
```

---

## 错误处理

### **常见错误**

| 错误类型 | 原因 | 解决方案 |
|---------|------|---------|
| **LangfuseNotFoundError** | Trace/Session 不存在 | 确保目标已同步到 ClickHouse |
| **LangfuseNotFoundError** | Score Config 不存在 | 使用有效的 configId |
| **PRECONDITION_FAILED** | 评分不符合配置 | 检查值/类别是否在配置范围内 |
| **InternalServerError** | 更新失败 | 检查 ClickHouse 连接 |

---

### **ClickHouse 最终一致性处理**

**问题**：ClickHouse 是最终一致性数据库，写入后可能短时间内查询不到。

**解决方案**：
```typescript
// updateAnnotationScore 中的处理（Lines 447-454）
if (!score) {
  if (!input.timestamp) {
    // ❌ 无 timestamp，无法 upsert
    throw new LangfuseNotFoundError(...);
  }
  
  // ✅ 有 timestamp，可以 upsert（沿着 ordering key）
  await upsertScore({ timestamp: input.timestamp, ... });
}
```

**最佳实践**：
- 客户端创建评分时提供 `timestamp`
- 支持在查询到之前进行更新

---

## 性能优化

### **批量操作优化**

| 策略 | 实现 | 效果 |
|-----|------|------|
| **单次查询** | `searchExistingAnnotationScore` | 避免重复评分 |
| **Upsert 语义** | ClickHouse UPSERT | 幂等操作 |
| **异步审计** | `auditLog` 不阻塞主流程 | 提升响应速度 |

---

### **查询优化**

```typescript
// searchExistingAnnotationScore 查询条件
WHERE project_id = ?
  AND trace_id = ?
  AND observation_id = ?
  AND name = ?
  AND config_id = ?
  AND data_type = ?
  AND source = 'ANNOTATION'
ORDER BY timestamp DESC
LIMIT 1
```

**索引建议**：
- `(project_id, trace_id, name, source)` 复合索引
- `(project_id, session_id, name, source)` 复合索引

---

## 监控指标

### **关键指标**

| 指标 | 查询方式 | 用途 |
|-----|---------|------|
| **标注速度** | `count(action='create') per hour` | 监控标注效率 |
| **修改频率** | `count(action='update') / count(action='create')` | 评估标注质量 |
| **删除率** | `count(action='delete') / total` | 发现异常评分 |
| **队列关联率** | `count(queue_id IS NOT NULL) / total` | 评估队列使用 |

---

## 故障排查

### **常见问题**

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| **创建后查询不到** | ClickHouse 延迟 | 等待 1-2 秒后重试 |
| **重复评分** | `searchExistingAnnotationScore` 失败 | 检查查询条件 |
| **更新失败** | 评分不符合配置 | 验证 value/stringValue |
| **无法删除** | 权限不足 | 检查 `scores:CUD` 权限 |

---

## 相关文档

- [Score Aggregation 管理](score-aggregation.md) - 评分聚合机制
- [Config Management 管理](config-management.md) - 评分配置管理
- [01-annotation-score-creation-sequence.puml](01-annotation-score-creation-sequence.puml) - 创建时序图
