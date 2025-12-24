# Label System 管理

## 功能概述

Label System 是 Prompts 模块的**部署环境管理系统**，通过标签（labels）实现提示词的多环境发布、A/B 测试和版本控制。核心特性包括标签唯一性（同一标签只能存在于一个版本）、受保护标签（protected labels）机制、依赖检查、自动标签移动等。设计理念：
- **唯一性约束**：一个标签只能标记一个版本
- **自动迁移**：设置标签时自动从旧版本移除
- **受保护标签**：需要特殊权限才能操作
- **依赖感知**：删除标签前检查是否被其他提示词依赖
- **实时同步**：标签变更立即失效缓存并触发 webhooks

---

## 核心概念

### Label（标签）

**定义**：用于标识提示词版本的环境或状态的字符串标识符。

**常见标签**：

| 标签 | 用途 | 示例场景 |
|-----|------|---------|
| **latest** | 最新版本（自动添加） | SDK 默认获取 |
| **production** | 生产环境 | 线上服务使用 |
| **staging** | 预发布环境 | 上线前验证 |
| **experiment-A** | A/B 测试 A 组 | 对比测试 |
| **experiment-B** | A/B 测试 B 组 | 对比测试 |
| **rollback-20240101** | 回滚标记 | 应急回滚 |

**数据模型**：
```typescript
type Prompt = {
  id: string;
  name: string;
  version: number;
  labels: string[];  // 标签数组，例如 ["latest", "production"]
  // ...
};
```

---

### Protected Labels（受保护标签）

**定义**：需要特殊权限（`promptProtectedLabels:CUD`）才能添加/删除的标签。

**用途**：
- 保护生产环境标签不被误操作
- 防止非管理员修改关键版本
- 实现严格的发布审批流程

**数据模型**：
```typescript
model PromptProtectedLabels {
  id        String   @id @default(uuid())
  projectId String
  label     String
  createdAt DateTime @default(now())
  
  @@unique([projectId, label])
}
```

---

## 核心组件

### 1. **setLabels 标签设置**

**位置**：[web/src/features/prompts/server/routers/promptRouter.ts](web/src/features/prompts/server/routers/promptRouter.ts#L691-L884)

#### **主流程（194 行）**

```typescript
// Lines 691-884: setLabels 路由（194 行）
setLabels: protectedProjectProcedure
  .input(
    z.object({
      promptId: z.string(),
      projectId: z.string(),
      labels: z.array(z.string()),
    }),
  )
  .mutation(async ({ input, ctx }) => {
    // 步骤 1: 权限验证（Lines 702-706）
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId: input.projectId,
      scope: "prompts:CUD",
    });

    // 步骤 2: 查询目标提示词（Lines 708-716）
    const toBeLabeledPrompt = await ctx.prisma.prompt.findUnique({
      where: {
        id: input.promptId,
        projectId: input.projectId,
      },
    });

    if (!toBeLabeledPrompt) {
      throw new TRPCError({
        code: "NOT_FOUND",
        message: "Prompt not found.",
      });
    }

    // 步骤 3: 计算标签差异（Lines 722-735）
    const newLabelSet = new Set(input.labels);
    const newLabels = [...newLabelSet];

    const removedLabels = [];
    for (const oldLabel of toBeLabeledPrompt.labels) {
      if (!newLabelSet.has(oldLabel)) {
        removedLabels.push(oldLabel);
      }
    }

    const addedLabels = [];
    for (const newLabel of newLabels) {
      if (!toBeLabeledPrompt.labels.includes(newLabel)) {
        addedLabels.push(newLabel);
      }
    }

    // 步骤 4: 受保护标签检查（Lines 737-751）
    const { hasProtectedLabels, protectedLabels } =
      await checkHasProtectedLabels({
        prisma: ctx.prisma,
        projectId: input.projectId,
        labelsToCheck: [...addedLabels, ...removedLabels],
      });

    if (hasProtectedLabels) {
      throwIfNoProjectAccess({
        session: ctx.session,
        projectId: input.projectId,
        scope: "promptProtectedLabels:CUD",
        forbiddenErrorMessage: `You don't have permission...
        Protected labels are: ${protectedLabels.join(", ")}`,
      });
    }

    // 步骤 5: 依赖检查（Lines 753-793）
    if (removedLabels.length > 0) {
      const dependents = await ctx.prisma.$queryRaw<
        {
          parent_name: string;
          parent_version: number;
          child_version: number;
          child_label: string;
        }[]
      >`
        SELECT
          p."name" AS "parent_name",
          p."version" AS "parent_version",
          pd."child_version" AS "child_version",
          pd."child_label" AS "child_label"
        FROM
          prompt_dependencies pd
          INNER JOIN prompts p ON p.id = pd.parent_id
        WHERE
          p.project_id = ${projectId}
          AND pd.project_id = ${projectId}
          AND pd.child_name = ${promptName}
          AND pd."child_label" IS NOT NULL 
          AND pd."child_label" IN (${Prisma.join(removedLabels)})
      `;

      if (dependents.length > 0) {
        const dependencyMessages = dependents
          .map(
            (d) =>
              `${d.parent_name} v${d.parent_version} depends on ${promptName} ${d.child_version ? `v${d.child_version}` : d.child_label}`,
          )
          .join("\\n");

        throw new TRPCError({
          code: "CONFLICT",
          message: `Other prompts are depending on the prompt label...
          ${dependencyMessages}
          Please delete the dependent prompts first.`,
        });
      }
    }

    // 步骤 6: 审计日志（Lines 797-806）
    await auditLog(
      {
        session: ctx.session,
        resourceType: "prompt",
        resourceId: toBeLabeledPrompt.id,
        action: "setLabel",
        after: {
          ...toBeLabeledPrompt,
          labels: newLabels,
        },
      },
      ctx.prisma,
    );

    // 步骤 7: 查询同标签的旧版本（Lines 808-817）
    const previousLabeledPrompts = await ctx.prisma.prompt.findMany({
      where: {
        projectId,
        name: promptName,
        labels: { hasSome: newLabels },
        id: { not: input.promptId },
      },
      orderBy: [{ version: "desc" }],
    });

    touchedPromptIds.push(...previousLabeledPrompts.map((p) => p.id));

    // 步骤 8: 构建事务操作（Lines 821-844）
    const toBeExecuted = [
      // 设置新标签
      ctx.prisma.prompt.update({
        where: {
          id: toBeLabeledPrompt.id,
          projectId,
        },
        data: {
          labels: newLabels,
        },
      }),
    ];

    // 移除旧版本的相同标签
    previousLabeledPrompts.forEach((prevPrompt) => {
      toBeExecuted.push(
        ctx.prisma.prompt.update({
          where: {
            id: prevPrompt.id,
            projectId,
          },
          data: {
            labels: prevPrompt.labels.filter((l) => !newLabels.includes(l)),
          },
        }),
      );
    });

    // 步骤 9: 缓存管理（Lines 846-851）
    const promptService = new PromptService(ctx.prisma, redis);
    await promptService.lockCache({ projectId, promptName });
    await promptService.invalidateCache({ projectId, promptName });

    // 步骤 10: 执行事务（Lines 854）
    await ctx.prisma.$transaction(toBeExecuted);

    // 步骤 11: 解锁缓存（Lines 857）
    await promptService.unlockCache({ projectId, promptName });

    // 步骤 12: 触发 Webhooks（Lines 860-873）
    const updatedPrompts = await ctx.prisma.prompt.findMany({
      where: {
        id: { in: touchedPromptIds },
        projectId,
      },
    });

    await Promise.all(
      updatedPrompts.map(async (prompt) =>
        promptChangeEventSourcing(
          await promptService.resolvePrompt(prompt),
          "updated",
        ),
      ),
    );
  }),
```

**12 步工作流**：

| 步骤 | 操作 | 代码行 | 作用 |
|-----|------|-------|------|
| 1 | 权限验证 | 702-706 | 检查 `prompts:CUD` 权限 |
| 2 | 查询目标 | 708-716 | 获取要设置标签的提示词 |
| 3 | 计算差异 | 722-735 | 找出新增/删除的标签 |
| 4 | 受保护标签检查 | 737-751 | 需要额外权限 |
| 5 | 依赖检查 | 753-793 | 确保不破坏依赖关系 |
| 6 | 审计日志 | 797-806 | 记录标签变更 |
| 7 | 查询旧版本 | 808-817 | 找出同名提示词的其他版本 |
| 8 | 构建事务 | 821-844 | 更新目标 + 移除旧标签 |
| 9 | 缓存管理 | 846-851 | 锁定 + 失效缓存 |
| 10 | 执行事务 | 854 | 原子性提交 |
| 11 | 解锁缓存 | 857 | 释放锁 |
| 12 | 触发 Webhooks | 860-873 | 通知外部系统 |

---

### 2. **标签唯一性机制**

#### **自动移除逻辑**

**问题**：为什么需要标签唯一性？
```
假设允许多个版本有相同标签：
  Version 1: labels = ["production"]
  Version 2: labels = ["production"]
  
SDK 调用 getPrompt("name", { label: "production" })
→ 应该返回哪个版本？模糊不清！
```

**解决方案**：强制唯一性
```typescript
// Lines 808-844: 标签移除逻辑
// 1. 查询所有拥有相同标签的其他版本
const previousLabeledPrompts = await ctx.prisma.prompt.findMany({
  where: {
    projectId,
    name: promptName,
    labels: { hasSome: newLabels },  // 有任意一个相同标签
    id: { not: input.promptId },     // 排除当前版本
  },
});

// 2. 从这些版本中移除冲突标签
previousLabeledPrompts.forEach((prevPrompt) => {
  toBeExecuted.push(
    ctx.prisma.prompt.update({
      where: { id: prevPrompt.id },
      data: {
        labels: prevPrompt.labels.filter((l) => 
          !newLabels.includes(l)  // 移除冲突的标签
        ),
      },
    }),
  );
});
```

**示例**：
```
初始状态：
  Version 1: labels = ["production", "stable"]
  Version 2: labels = ["latest"]

操作：setLabels(Version 2, ["latest", "production"])

结果：
  Version 1: labels = ["stable"]  （"production" 被移除）
  Version 2: labels = ["latest", "production"]
```

---

### 3. **removeLabelsFromPreviousPromptVersions 工具函数**

**位置**：[web/src/features/prompts/server/utils/updatePromptLabels.ts](web/src/features/prompts/server/utils/updatePromptLabels.ts#L2-L42)

#### **函数实现（41 行）**

```typescript
// Lines 2-42: removeLabelsFromPreviousPromptVersions（41 行）
const removeLabelsFromPreviousPromptVersions = async ({
  prisma,
  projectId,
  promptName,
  labelsToRemove,
}: {
  prisma: PrismaClient;
  projectId: string;
  promptName: string;
  labelsToRemove: string[];
}) => {
  // 1. 查询所有拥有这些标签的版本（Lines 17-23）
  const previouslyLabeledPrompts = await prisma.prompt.findMany({
    where: {
      projectId,
      name: promptName,
      labels: { hasSome: labelsToRemove },
    },
    orderBy: [{ version: "desc" }],
  });

  // 2. 记录受影响的提示词 ID（Lines 25-27）
  const touchedPromptIds = previouslyLabeledPrompts.map(
    (prevPrompt) => prevPrompt.id,
  );

  // 3. 构建批量更新操作（Lines 29-40）
  return {
    touchedPromptIds,
    updates: previouslyLabeledPrompts.map((prevPrompt) =>
      prisma.prompt.update({
        where: { id: prevPrompt.id },
        data: {
          labels: prevPrompt.labels.filter(
            (prevLabel) => !labelsToRemove.includes(prevLabel),
          ),
        },
      }),
    ),
  };
};
```

**使用场景**：
```typescript
// 在 createPrompt 中使用
const { touchedPromptIds, updates } = 
  await removeLabelsFromPreviousPromptVersions({
    prisma,
    projectId,
    promptName: name,
    labelsToRemove: ["latest"],  // 新版本会获得 "latest"
  });

create.push(...updates);  // 加入事务
```

---

### 4. **受保护标签管理**

#### **addProtectedLabel 添加受保护标签**

**位置**：[web/src/features/prompts/server/routers/promptRouter.ts](web/src/features/prompts/server/routers/promptRouter.ts#L1235-L1287)

```typescript
// Lines 1235-1287: addProtectedLabel 路由（53 行）
addProtectedLabel: protectedProjectProcedure
  .input(z.object({ projectId: z.string(), label: PromptLabelSchema }))
  .mutation(async ({ input, ctx }) => {
    const { projectId, label } = input;

    // 1. 权限验证（Lines 1240-1245）
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId,
      scope: "promptProtectedLabels:CUD",
      forbiddenErrorMessage:
        "You don't have permission to mark a label as protected...",
    });

    // 2. 权益检查（Lines 1247-1251）
    throwIfNoEntitlement({
      projectId,
      entitlement: "prompt-protected-labels",
      sessionUser: ctx.session.user,
    });

    // 3. 特殊标签检查（Lines 1253-1258）
    if (label === LATEST_PROMPT_LABEL) {
      throw new TRPCError({
        code: "BAD_REQUEST",
        message: `You cannot protect the label '${LATEST_PROMPT_LABEL}' as this would effectively block prompt creation.`,
      });
    }

    // 4. 创建受保护标签（Lines 1260-1271）
    const protectedLabel = await ctx.prisma.promptProtectedLabels.upsert({
      where: {
        projectId_label: {
          projectId,
          label,
        },
      },
      create: {
        projectId,
        label,
      },
      update: {},
    });

    // 5. 审计日志（Lines 1273-1282）
    await auditLog(
      {
        session: ctx.session,
        resourceType: "promptProtectedLabel",
        resourceId: protectedLabel.id,
        action: "create",
        after: protectedLabel.label,
      },
      ctx.prisma,
    );

    return protectedLabel;
  }),
```

**为什么禁止保护 "latest" 标签**：
- `latest` 标签在 `createPrompt` 中自动添加
- 保护 `latest` 会导致无法创建新版本
- 用户会收到 "没有权限" 错误，但实际是系统逻辑问题

---

#### **getProtectedLabels 获取受保护标签列表**

**位置**：[web/src/features/prompts/server/routers/promptRouter.ts](web/src/features/prompts/server/routers/promptRouter.ts#L1209-L1233)

```typescript
// Lines 1209-1233: getProtectedLabels 路由（25 行）
getProtectedLabels: protectedProjectProcedure
  .input(z.object({ projectId: z.string() }))
  .query(async ({ input, ctx }) => {
    const { projectId } = input;

    // 1. 权限验证（Lines 1214-1218）
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId,
      scope: "prompts:read",
    });

    // 2. 权益检查（Lines 1220-1224）
    throwIfNoEntitlement({
      projectId,
      entitlement: "prompt-protected-labels",
      sessionUser: ctx.session.user,
    });

    // 3. 查询受保护标签（Lines 1226-1230）
    const protectedLabels = await ctx.prisma.promptProtectedLabels.findMany({
      where: {
        projectId,
      },
    });

    return protectedLabels.map((l) => l.label);
  }),
```

---

#### **removeProtectedLabel 移除受保护标签**

**位置**：[web/src/features/prompts/server/routers/promptRouter.ts](web/src/features/prompts/server/routers/promptRouter.ts#L1289-L1330)

**关键逻辑**：
```typescript
// Lines 1289-1330: removeProtectedLabel 路由（42 行）
removeProtectedLabel: protectedProjectProcedure
  .input(z.object({ projectId: z.string(), label: z.string() }))
  .mutation(async ({ input, ctx }) => {
    // 权限 + 权益检查
    // ...

    // 删除受保护标签记录
    const protectedLabel = await ctx.prisma.promptProtectedLabels.delete({
      where: {
        projectId_label: {
          projectId: input.projectId,
          label: input.label,
        },
      },
    });

    // 审计日志
    await auditLog({
      session: ctx.session,
      resourceType: "promptProtectedLabel",
      resourceId: protectedLabel.id,
      action: "delete",
      before: protectedLabel.label,
    }, ctx.prisma);

    return protectedLabel;
  }),
```

---

### 5. **依赖检查机制**

#### **为什么需要依赖检查**

**问题场景**：
```
Prompt A (version 2, labels=["production"]):
  "System: @@@langfusePrompt:name=B|label=production@@@"

Prompt B (version 5, labels=["production"]):
  "Context about production environment"

如果尝试删除 Prompt B 的 "production" 标签：
→ Prompt A 的依赖会失效！
→ 运行时会抛出 "Prompt dependency not found" 错误
```

**解决方案**：删除/移除标签前检查依赖

#### **SQL 依赖查询（Lines 753-778）**

```sql
SELECT
  p."name" AS "parent_name",
  p."version" AS "parent_version",
  pd."child_version" AS "child_version",
  pd."child_label" AS "child_label"
FROM
  prompt_dependencies pd
  INNER JOIN prompts p ON p.id = pd.parent_id
WHERE
  p.project_id = ${projectId}
  AND pd.project_id = ${projectId}
  AND pd.child_name = ${promptName}
  AND pd."child_label" IS NOT NULL 
  AND pd."child_label" IN (${Prisma.join(removedLabels)})
```

**查询逻辑**：
1. 从 `prompt_dependencies` 表找所有依赖
2. 过滤出通过标签引用的依赖（`child_label IS NOT NULL`）
3. 检查要删除的标签是否在依赖列表中

**错误信息示例**：
```
Error: Other prompts are depending on the prompt label you are trying to remove:

chat-completion v3 depends on context production
system-prompt v1 depends on context production

Please delete the dependent prompts first.
```

---

### 6. **allLabels 标签列表查询**

**位置**：[web/src/features/prompts/server/routers/promptRouter.ts](web/src/features/prompts/server/routers/promptRouter.ts#L885-L902)

```typescript
// Lines 885-902: allLabels 路由（18 行）
allLabels: protectedProjectProcedure
  .input(z.object({ projectId: z.string() }))
  .query(async ({ input, ctx }) => {
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId: input.projectId,
      scope: "prompts:read",
    });

    // 使用 Prisma 的聚合查询
    const labels = await ctx.prisma.prompt.findMany({
      where: { projectId: input.projectId },
      select: { labels: true },
      distinct: ["id"],
    });

    // 扁平化 + 去重
    const uniqueLabels = [...new Set(labels.flatMap((p) => p.labels))];

    return uniqueLabels;
  }),
```

**返回示例**：
```json
["latest", "production", "staging", "experiment-A", "experiment-B"]
```

---

## 标签生命周期

### **创建时自动添加 "latest"**

```typescript
// Lines 90 in createPrompt
const finalLabels = [...labels, LATEST_PROMPT_LABEL];
```

| 用户输入 | 实际存储 |
|---------|---------|
| `labels: []` | `["latest"]` |
| `labels: ["staging"]` | `["staging", "latest"]` |
| `labels: ["production", "latest"]` | `["production", "latest"]`（去重） |

---

### **标签迁移示例**

**初始状态**：
```
Prompt "chat-completion"
  Version 1: labels = ["latest"]
  Version 2: labels = []
```

**操作 1**：创建 Version 3
```typescript
createPrompt({
  name: "chat-completion",
  labels: [],
  // ...
});
```

**结果**：
```
Version 1: labels = []          （"latest" 被移除）
Version 2: labels = []
Version 3: labels = ["latest"]  （自动添加）
```

**操作 2**：设置 Version 2 为 production
```typescript
setLabels({
  promptId: "version_2_id",
  labels: ["production", "latest"],
});
```

**结果**：
```
Version 1: labels = []
Version 2: labels = ["production", "latest"]
Version 3: labels = []          （"latest" 被移除）
```

---

## 权限控制

### **权限层级**

| 权限 Scope | 操作 | 说明 |
|-----------|------|------|
| **prompts:read** | 查询标签 | 所有人 |
| **prompts:CUD** | 添加/删除普通标签 | 开发者 |
| **promptProtectedLabels:CUD** | 操作受保护标签 | 管理员 |

### **权限检查流程**

```typescript
// 步骤 1: 基础权限检查
throwIfNoProjectAccess({
  session: ctx.session,
  projectId: input.projectId,
  scope: "prompts:CUD",
});

// 步骤 2: 识别要操作的标签
const addedLabels = [...];
const removedLabels = [...];

// 步骤 3: 检查是否包含受保护标签
const { hasProtectedLabels, protectedLabels } =
  await checkHasProtectedLabels({
    prisma: ctx.prisma,
    projectId: input.projectId,
    labelsToCheck: [...addedLabels, ...removedLabels],
  });

// 步骤 4: 如果包含受保护标签，需要额外权限
if (hasProtectedLabels) {
  throwIfNoProjectAccess({
    session: ctx.session,
    projectId: input.projectId,
    scope: "promptProtectedLabels:CUD",
    forbiddenErrorMessage: `Protected labels are: ${protectedLabels.join(", ")}`,
  });
}
```

---

## 使用示例

### **场景 1：发布到生产环境**

```typescript
// 1. 创建新版本
const v3 = await createPrompt({
  name: "greeting",
  prompt: "Hello, {{name}}! Welcome!",
  labels: ["latest"],
  commitMessage: "Add welcome message",
});
// 结果：v3.labels = ["latest"]

// 2. 测试验证后，设置为生产环境
await setLabels({
  promptId: v3.id,
  projectId: "proj_123",
  labels: ["latest", "production"],
});
// 结果：v3.labels = ["latest", "production"]
```

---

### **场景 2：A/B 测试**

```typescript
// 1. 准备两个版本
const vA = await createPrompt({
  name: "recommendation",
  prompt: "Recommend {{count}} items based on {{criteria}}.",
  labels: ["experiment-A"],
});

const vB = await createPrompt({
  name: "recommendation",
  prompt: "Show {{count}} personalized suggestions for {{criteria}}.",
  labels: ["experiment-B"],
});

// 2. 客户端随机选择
const label = Math.random() < 0.5 ? "experiment-A" : "experiment-B";
const prompt = await getPrompt("recommendation", { label });

// 3. 实验结束，选择胜者
await setLabels({
  promptId: vA.id,  // A 组胜出
  projectId: "proj_123",
  labels: ["experiment-A", "production"],
});
```

---

### **场景 3：应急回滚**

```typescript
// 1. 当前生产版本出现问题
const currentProduction = { id: "v5_id", version: 5, labels: ["production"] };

// 2. 立即回滚到稳定版本
await setLabels({
  promptId: "v3_id",  // 之前的稳定版本
  projectId: "proj_123",
  labels: ["production", "rollback-20240124"],
});

// 结果：
// - v5: labels = []（生产标签被移除）
// - v3: labels = ["production", "rollback-20240124"]
```

---

### **场景 4：保护生产环境标签**

```typescript
// 1. 管理员添加受保护标签
await addProtectedLabel({
  projectId: "proj_123",
  label: "production",
});

// 2. 普通开发者尝试修改生产标签
await setLabels({
  promptId: "v5_id",
  labels: ["production"],
});
// ❌ Error: You don't have permission to add/remove a protected label
// Protected labels are: production

// 3. 只有管理员可以操作
// 需要 promptProtectedLabels:CUD 权限
```

---

## 性能优化

### **批量操作优化**

| 策略 | 实现 | 效果 |
|-----|------|------|
| **单次事务** | `prisma.$transaction([...])` | 减少数据库往返 |
| **批量更新** | 合并多个 `update` | 避免多次连接 |
| **并行查询** | `Promise.all([prompts, dependents])` | 减少延迟 |

### **缓存策略**

```typescript
// Lines 846-857: 缓存管理流程
// 1. 锁定缓存（防止读取脏数据）
await promptService.lockCache({ projectId, promptName });

// 2. 失效所有相关缓存
await promptService.invalidateCache({ projectId, promptName });

// 3. 执行标签变更
await prisma.$transaction(toBeExecuted);

// 4. 解锁缓存
await promptService.unlockCache({ projectId, promptName });
```

**为什么要失效整个提示词的缓存**：
- 标签变更可能影响所有版本
- 缓存键包含 label 信息
- 确保下次查询返回最新数据

---

## 监控指标

### **标签使用统计**

| 指标 | 查询方式 | 用途 |
|-----|---------|------|
| **标签数量** | `count(distinct label)` | 监控标签膨胀 |
| **每个标签的版本数** | `group by label` | 发现未清理的标签 |
| **受保护标签数量** | `count(promptProtectedLabels)` | 审计保护策略 |
| **标签变更频率** | `auditLog` + 时间聚合 | 识别频繁变更 |

### **依赖关系统计**

| 指标 | 查询方式 | 用途 |
|-----|---------|------|
| **标签依赖数** | `count(child_label IS NOT NULL)` | 评估标签删除风险 |
| **版本依赖数** | `count(child_version IS NOT NULL)` | 评估版本删除风险 |

---

## 故障排查

### **常见问题**

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| **标签突然消失** | 其他版本设置了相同标签 | 检查审计日志，查看标签迁移历史 |
| **无法删除标签** | 被其他提示词依赖 | 先删除依赖关系或使用版本引用 |
| **权限错误** | 操作受保护标签 | 联系管理员或移除保护 |
| **"latest" 标签丢失** | 手动删除了 | 重新设置标签（会自动恢复） |

---

## 相关文档

- [Version Management 管理](version-management.md) - 版本控制机制
- [Cache Service 管理](cache-service.md) - Redis 缓存服务
- [03-prompt-list-sequence.puml](03-prompt-list-sequence.puml) - 列表查询时序图
- [04-label-management-sequence.puml](04-label-management-sequence.puml) - 标签管理时序图
