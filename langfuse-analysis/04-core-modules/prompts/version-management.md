# Version Management 管理

## 功能概述

Version Management 是 Prompts 模块的**核心版本控制系统**，负责管理提示词的版本历史、依赖解析、变更追踪。通过自增版本号、commitMessage、依赖图等机制，实现类似 Git 的版本管理能力，支持安全地发布、回滚和并行测试多个提示词版本。核心设计：
- **自增版本号**：同名提示词版本从 1 开始递增
- **依赖解析**：支持嵌套提示词引用（`@@@langfusePrompt:name=xxx|version=1@@@`）
- **变更追踪**：commitMessage + auditLog 完整记录
- **事务保障**：Prisma Transaction 确保原子性
- **缓存管理**：版本变更时自动失效缓存

---

## 核心组件

### 1. **createPrompt 版本创建**

**位置**：[web/src/features/prompts/server/actions/createPrompt.ts](web/src/features/prompts/server/actions/createPrompt.ts#L59-L220)

#### **主流程（162 行）**

```typescript
// Lines 59-220: createPrompt 函数（162 行）
const createPrompt = async ({
  projectId,
  name,
  prompt,
  type = PromptType.Text,
  labels = [],
  config,
  createdBy,
  prisma,
  tags,
  commitMessage,
}: CreatePromptParams) => {
  // 步骤 1: 获取最新版本（Lines 69-71）
  const latestPrompt = await prisma.prompt.findFirst({
    where: { projectId, name },
    orderBy: [{ version: "desc" }],
  });

  // 步骤 2: 类型验证（Lines 73-77）
  if (latestPrompt && latestPrompt.type !== type) {
    throw new InvalidRequestError(
      "Previous versions have different prompt type..."
    );
  }

  // 步骤 3: 变量/占位符冲突检测（Lines 79-88）
  if (type === PromptType.Chat && Array.isArray(prompt)) {
    const { variables, placeholders } =
      extractChatVariableAndPlaceholderNames(prompt);
    const conflictingNames = variables.filter((v) => 
      placeholders.includes(v)
    );
    if (conflictingNames.length > 0) {
      throw new InvalidRequestError(
        `variables and placeholders must be unique...`
      );
    }
  }

  // 步骤 4: 添加 "latest" 标签（Lines 90）
  const finalLabels = [...labels, LATEST_PROMPT_LABEL];

  // 步骤 5: 继承 tags（Lines 93）
  const finalTags = [...new Set(tags ?? latestPrompt?.tags ?? [])];

  // 步骤 6: 解析依赖图（Lines 98-106）
  const promptService = new PromptService(prisma, redis);
  const promptDependencies = parsePromptDependencyTags(prompt);
  
  await promptService.buildAndResolvePromptGraph({
    projectId,
    parentPrompt: {
      id: newPromptId,
      prompt,
      version: latestPrompt?.version ? latestPrompt.version + 1 : 1,
      name,
      labels,
    },
    dependencies: promptDependencies,
  });

  // 步骤 7: 构建事务操作（Lines 116-132）
  const create = [
    prisma.prompt.create({
      data: {
        id: newPromptId,
        prompt,
        name,
        version: latestPrompt?.version ? latestPrompt.version + 1 : 1,
        labels: [...new Set(finalLabels)],
        config,
        commitMessage,
        // ...
      },
    }),
    // 创建依赖关系
    ...promptDependencies.map((dep) =>
      prisma.promptDependency.create({ /* ... */ })
    ),
  ];

  // 步骤 8: 移除旧版本标签（Lines 134-143）
  if (finalLabels.length > 0) {
    const { updates } = await removeLabelsFromPreviousPromptVersions({
      prisma,
      projectId,
      promptName: name,
      labelsToRemove: finalLabels,
    });
    create.push(...updates);
  }

  // 步骤 9: 同步 tags（Lines 145-155）
  if (haveTagsChanged) {
    const { updates } = await updatePromptTagsOnAllVersions({
      prisma,
      projectId,
      promptName: name,
      tags: finalTags,
    });
    create.push(...updates);
  }

  // 步骤 10: 缓存管理（Lines 158-159）
  await promptService.lockCache({ projectId, promptName: name });
  await promptService.invalidateCache({ projectId, promptName: name });

  // 步骤 11: 执行事务（Lines 162-165）
  const [createdPrompt] = await prisma.$transaction(create);

  // 步骤 12: 解锁缓存（Lines 168）
  await promptService.unlockCache({ projectId, promptName: name });

  // 步骤 13: 触发 Webhooks（Lines 175-184）
  await Promise.all([
    ...updatedPrompts.map(async (prompt) =>
      promptChangeEventSourcing(
        await promptService.resolvePrompt(prompt),
        "updated",
      ),
    ),
    promptChangeEventSourcing(
      await promptService.resolvePrompt(createdPrompt),
      "created",
    ),
  ]);

  return createdPrompt;
};
```

**13 步工作流**：

| 步骤 | 操作 | 代码行 | 作用 |
|-----|------|-------|------|
| 1 | 获取最新版本 | 69-71 | 确定新版本号 |
| 2 | 类型验证 | 73-77 | 防止类型不一致 |
| 3 | 冲突检测 | 79-88 | 确保变量/占位符唯一 |
| 4 | 添加标签 | 90 | 自动添加 `latest` |
| 5 | 继承 tags | 93 | 保持版本间 tags 一致 |
| 6 | 解析依赖图 | 98-106 | 验证依赖有效性 |
| 7 | 构建事务 | 116-132 | 创建 Prompt + 依赖 |
| 8 | 移除旧标签 | 134-143 | 确保标签唯一性 |
| 9 | 同步 tags | 145-155 | 更新所有版本的 tags |
| 10 | 缓存管理 | 158-159 | 锁定 + 失效缓存 |
| 11 | 执行事务 | 162-165 | 原子性提交 |
| 12 | 解锁缓存 | 168 | 释放锁 |
| 13 | 触发 Webhooks | 175-184 | 通知外部系统 |

---

### 2. **版本号生成规则**

#### **计算逻辑**

```typescript
// 新版本号 = 最新版本号 + 1（如果存在），否则为 1
const newVersion = latestPrompt?.version 
  ? latestPrompt.version + 1 
  : 1;
```

#### **版本序列示例**

| 操作 | 版本号 | 标签 | commitMessage |
|-----|-------|------|---------------|
| **初次创建** | 1 | `["latest"]` | `"Initial version"` |
| **优化提示词** | 2 | `["latest"]` | `"Improve clarity"` |
| **添加生产标签** | 2 | `["latest", "production"]` | - |
| **继续开发** | 3 | `["latest"]` | `"Add examples"` |
| **回滚操作** | 无新版本 | 修改 v2 标签为 `["production"]` | - |

**关键点**：
- 版本号只增不减
- `latest` 标签自动移动到最新版本
- 回滚通过修改标签实现，不创建新版本

---

### 3. **promptRouter 路由定义**

**位置**：[web/src/features/prompts/server/routers/promptRouter.ts](web/src/features/prompts/server/routers/promptRouter.ts#L76-L1331)

#### **核心路由（1256 行）**

| 路由 | 代码行 | 行数 | 功能 |
|-----|-------|------|------|
| **hasAny** | 77-99 | 23 | 检查是否有任何提示词 |
| **all** | 100-185 | 86 | 分页查询提示词列表 |
| **count** | 186-241 | 56 | 统计提示词数量 |
| **metrics** | 242-259 | 18 | 获取提示词指标 |
| **byId** | 260-279 | 20 | 根据 ID 查询单个提示词 |
| **create** | 280-327 | 48 | 创建新版本 |
| **duplicatePrompt** | 328-369 | 42 | 复制提示词 |
| **filterOptions** | 370-413 | 44 | 获取筛选选项 |
| **delete** | 414-536 | 123 | 删除所有版本 |
| **deleteVersion** | 537-690 | 154 | 删除单个版本 |
| **setLabels** | 691-884 | 194 | 设置标签 |
| **allLabels** | 885-902 | 18 | 获取所有标签 |
| **allNames** | 903-931 | 29 | 获取所有提示词名称 |
| **allVersions** | 1056-1121 | 66 | 获取指定提示词的所有版本 |
| **versionMetrics** | 1122-1165 | 44 | 获取版本使用指标 |
| **resolvePromptGraph** | 1166-1207 | 42 | 解析依赖图 |

#### **create 路由（Lines 280-327）**

```typescript
// Lines 280-327: create 路由（48 行）
create: protectedProjectProcedure
  .input(CreatePromptTRPCSchema)
  .mutation(async ({ input, ctx }) => {
    // 1. 权限验证（Lines 284-287）
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId: input.projectId,
      scope: "prompts:CUD",
    });

    // 2. 受保护标签检查（Lines 289-303）
    const { hasProtectedLabels, protectedLabels } =
      await checkHasProtectedLabels({
        prisma: ctx.prisma,
        projectId: input.projectId,
        labelsToCheck: input.labels,
      });

    if (hasProtectedLabels) {
      throwIfNoProjectAccess({
        session: ctx.session,
        projectId: input.projectId,
        scope: "promptProtectedLabels:CUD",
        forbiddenErrorMessage: `You don't have permission...`,
      });
    }

    // 3. 调用 createPrompt（Lines 305-309）
    const prompt = await createPrompt({
      ...input,
      prisma: ctx.prisma,
      createdBy: ctx.session.user.id,
    });

    // 4. 审计日志（Lines 315-322）
    await auditLog(
      {
        session: ctx.session,
        resourceType: "prompt",
        resourceId: prompt.id,
        action: "create",
        after: prompt,
      },
      ctx.prisma,
    );

    return prompt;
  }),
```

---

### 4. **allVersions 版本列表查询**

**位置**：[web/src/features/prompts/server/routers/promptRouter.ts](web/src/features/prompts/server/routers/promptRouter.ts#L1056-L1121)

#### **查询逻辑（66 行）**

```typescript
// Lines 1056-1121: allVersions 路由（66 行）
allVersions: protectedProjectProcedure
  .input(
    z.object({
      projectId: z.string(),
      name: z.string(),
      ...optionalPaginationZod,
    }),
  )
  .query(async ({ input, ctx }) => {
    // 1. 权限验证（Lines 1067-1071）
    throwIfNoProjectAccess({
      session: ctx.session,
      projectId: input.projectId,
      scope: "prompts:read",
    });

    // 2. 并行查询版本和总数（Lines 1072-1087）
    const [prompts, totalCount] = await Promise.all([
      ctx.prisma.prompt.findMany({
        where: {
          projectId: input.projectId,
          name: input.name,
        },
        ...(input.limit && input.page
          ? { take: input.limit, skip: input.page * input.limit }
          : undefined),
        orderBy: [{ version: "desc" }],  // 降序排列
      }),
      ctx.prisma.prompt.count({
        where: {
          projectId: input.projectId,
          name: input.name,
        },
      }),
    ]);

    // 3. 查询创建者信息（Lines 1089-1102）
    const userIds = prompts
      .map((p) => p.createdBy)
      .filter((id) => id !== "API");
      
    const users = await ctx.prisma.user.findMany({
      select: { id: true, name: true },
      where: {
        id: { in: userIds },
        organizationMemberships: {
          some: { orgId: ctx.session.orgId },
        },
      },
    });

    // 4. 关联创建者名称（Lines 1104-1113）
    const joinedPromptAndUsers = prompts.map((p) => {
      const user = users.find((u) => u.id === p.createdBy);
      if (!user && p.createdBy === "API") {
        return { ...p, creator: "API" };
      }
      return { ...p, creator: user?.name };
    });

    return { promptVersions: joinedPromptAndUsers, totalCount };
  }),
```

**查询优化**：
- 并行查询数据和总数（减少延迟）
- 按版本号降序排列（最新版本在前）
- 可选分页支持（`limit` + `page`）
- 批量查询创建者（避免 N+1 查询）

---

### 5. **依赖图解析**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L274-L419)

#### **buildAndResolvePromptGraph 方法（146 行）**

```typescript
// Lines 274-419: buildAndResolvePromptGraph（146 行）
public async buildAndResolvePromptGraph(params: {
  projectId: string;
  parentPrompt: PartialPrompt;
  dependencies?: ParsedPromptDependencyTag[];
}): Promise<ResolvedPromptGraph> {
  const { projectId, parentPrompt, dependencies } = params;

  // 初始化依赖图（Lines 281-286）
  const graph: PromptGraph = {
    root: {
      name: parentPrompt.name,
      version: parentPrompt.version,
      id: parentPrompt.id,
    },
    dependencies: {},
  };
  const seen = new Set<string>();

  // 递归解析函数（Lines 288-394）
  const resolve = async (
    currentPrompt: PartialPrompt,
    deps: ParsedPromptDependencyTag[] | undefined,
    level: number,
  ) => {
    // 1. 深度检查（Lines 294-298）
    if (level >= MAX_PROMPT_NESTING_DEPTH) {
      throw Error(`Maximum nesting depth exceeded (${MAX_PROMPT_NESTING_DEPTH})`);
    }

    // 2. 循环依赖检查（Lines 300-309）
    if (
      seen.has(currentPrompt.id) ||
      (currentPrompt.name === parentPrompt.name &&
       currentPrompt.id !== parentPrompt.id)
    ) {
      throw Error(`Circular dependency detected...`);
    }

    seen.add(currentPrompt.id);

    // 3. 获取依赖列表（Lines 313-334）
    let promptDependencies = deps;
    if (!deps) {
      promptDependencies = (
        await this.prisma.promptDependency.findMany({
          where: { projectId, parentId: currentPrompt.id },
          select: {
            childName: true,
            childLabel: true,
            childVersion: true,
          },
        })
      ).map((dep) => ({
        name: dep.childName,
        ...(dep.childVersion
          ? { type: "version", version: dep.childVersion }
          : { type: "label", label: dep.childLabel }),
      }));
    }

    // 4. 解析依赖内容（Lines 336-388）
    if (promptDependencies && promptDependencies.length) {
      let resolvedPrompt = JSON.stringify(currentPrompt.prompt);

      for (const dep of promptDependencies) {
        // 查询依赖的提示词（Lines 345-352）
        const depPrompt = await this.prisma.prompt.findFirst({
          where: {
            projectId,
            name: dep.name,
            ...(dep.type === "version"
              ? { version: dep.version }
              : { labels: { has: dep.label } }),
          },
        });

        if (!depPrompt)
          throw Error(`Prompt dependency not found: ${dep.name}`);
        if (depPrompt.type !== "text")
          throw Error(`Prompt dependency is not a text prompt`);

        // 记录依赖关系（Lines 358-364）
        graph.dependencies[currentPrompt.id] ??= [];
        graph.dependencies[currentPrompt.id].push({
          id: depPrompt.id,
          name: depPrompt.name,
          version: depPrompt.version,
        });

        // 递归解析（Lines 367）
        const resolvedDepPrompt = await resolve(
          depPrompt,
          undefined,
          level + 1,
        );

        // 替换依赖标记（Lines 369-382）
        const versionPattern = `@@@langfusePrompt:name=${escapeRegex(depPrompt.name)}\\|version=${escapeRegex(depPrompt.version)}@@@`;
        const labelPatterns = depPrompt.labels.map(
          (label) =>
            `@@@langfusePrompt:name=${escapeRegex(depPrompt.name)}\\|label=${escapeRegex(label)}@@@`,
        );
        const combinedPattern = [versionPattern, ...labelPatterns].join("|");
        const regex = new RegExp(combinedPattern, "g");

        const replaceValue = JSON.stringify(resolvedDepPrompt)
          .slice(1, -1) // 去除外层引号
          .replace(/\$/g, "$$$$"); // 转义 $ 符号

        resolvedPrompt = resolvedPrompt.replace(regex, replaceValue);
      }

      seen.delete(currentPrompt.id);
      return JSON.parse(resolvedPrompt);
    } else {
      seen.delete(currentPrompt.id);
      return currentPrompt.prompt;
    }
  };

  // 开始解析（Lines 396）
  const resolvedPrompt = await resolve(parentPrompt, dependencies, 0);

  return {
    graph: Object.keys(graph.dependencies).length > 0 ? graph : null,
    resolvedPrompt,
  };
}
```

**依赖解析机制**：

| 检查 | 错误类型 | 描述 |
|-----|---------|------|
| **深度限制** | `MAX_PROMPT_NESTING_DEPTH` | 防止无限嵌套（默认 10 层） |
| **循环依赖** | `seen.has(id)` | 检测 A → B → A 循环 |
| **同名版本** | `name === parent.name && id !== parent.id` | 禁止引用自身其他版本 |
| **依赖缺失** | `!depPrompt` | 引用的提示词不存在 |
| **类型检查** | `type !== "text"` | 只能引用 text 类型提示词 |

---

### 6. **依赖标记语法**

#### **版本引用**

```
@@@langfusePrompt:name=greeting|version=2@@@
```

#### **标签引用**

```
@@@langfusePrompt:name=greeting|label=production@@@
```

#### **使用示例**

**Prompt A (name="base-prompt", version=1)**:
```text
You are a helpful assistant.
Context: @@@langfusePrompt:name=context|label=latest@@@
```

**Prompt B (name="context", version=3, labels=["latest"])**:
```text
The current date is {{date}}.
```

**解析后的 Prompt A**:
```text
You are a helpful assistant.
Context: The current date is {{date}}.
```

---

## 变更追踪

### **commitMessage 机制**

#### **数据模型**

```typescript
type Prompt = {
  id: string;
  name: string;
  version: number;
  commitMessage: string | null;  // 版本变更说明
  createdBy: string;             // 创建者 ID 或 "API"
  createdAt: Date;
  updatedAt: Date;
  // ...
};
```

#### **使用场景**

| 场景 | commitMessage 示例 |
|-----|-------------------|
| **初次创建** | `"Initial version"` |
| **优化提示词** | `"Improve clarity and add examples"` |
| **修复问题** | `"Fix variable name typo"` |
| **API 创建** | `null`（通常由 API 创建时不提供） |

---

### **Audit Log 审计日志**

#### **日志结构**

```typescript
await auditLog(
  {
    session: ctx.session,
    resourceType: "prompt",        // 资源类型
    resourceId: prompt.id,         // 提示词 ID
    action: "create",              // 操作：create/update/delete
    after: prompt,                 // 变更后状态
  },
  ctx.prisma,
);
```

#### **记录的操作**

| 操作 | action | 记录内容 |
|-----|--------|---------|
| **创建版本** | `create` | 完整的新版本数据 |
| **设置标签** | `setLabel` | 标签变更前后对比 |
| **删除版本** | `delete` | 被删除的版本信息 |
| **更新 tags** | `updateTags` | tags 变更 |

---

## 事务管理

### **原子性保障**

```typescript
// Lines 162-165: 事务执行
const toBeExecuted = [
  prisma.prompt.create({ /* 创建新版本 */ }),
  ...dependencyCreates,        // 创建依赖关系
  ...labelRemovalUpdates,      // 移除旧版本标签
  ...tagUpdateOperations,      // 同步 tags
];

const [createdPrompt] = await prisma.$transaction(toBeExecuted);
```

**事务内容**：
1. 创建新版本记录
2. 创建依赖关系（`PromptDependency` 表）
3. 更新旧版本标签（确保标签唯一性）
4. 同步所有版本的 tags

**失败回滚**：
- 任何一步失败，整个事务回滚
- 缓存锁会在失败时释放（通过 `unlockCache`）
- 不会产生部分更新的脏数据

---

### **缓存同步**

```typescript
// Lines 158-168: 缓存管理流程
// 1. 锁定缓存（防止并发读取过期数据）
await promptService.lockCache({ projectId, promptName: name });

// 2. 失效缓存（删除所有相关缓存键）
await promptService.invalidateCache({ projectId, promptName: name });

// 3. 执行事务
const [createdPrompt] = await prisma.$transaction(create);

// 4. 解锁缓存
await promptService.unlockCache({ projectId, promptName: name });
```

**为什么需要锁**：
- **问题**：事务执行期间，其他请求可能读取旧缓存
- **解决**：锁定期间，`getPrompt` 会跳过缓存直接查数据库
- **时长**：通常 < 100ms（事务执行时间）

---

## 性能优化

### **查询优化**

| 优化策略 | 实现方式 | 提升 |
|---------|---------|------|
| **并行查询** | `Promise.all([prompts, totalCount])` | 减少 50% 延迟 |
| **批量查询用户** | `user.findMany({ where: { id: { in: userIds } } })` | 避免 N+1 查询 |
| **索引优化** | `projectId + name + version` 复合索引 | 加速版本查找 |
| **版本号排序** | `orderBy: [{ version: "desc" }]` | 数据库层排序 |

### **事务优化**

| 策略 | 效果 |
|-----|------|
| **批量操作** | 合并多个 `update` 到一个事务 |
| **减少往返** | 一次事务完成所有变更 |
| **选择性更新** | 只更新变化的字段 |

---

## 监控指标

### **版本统计**

| 指标 | 查询方式 | 用途 |
|-----|---------|------|
| **总版本数** | `count({ where: { projectId, name } })` | 监控版本增长 |
| **平均版本数** | `count / distinctNames` | 衡量迭代频率 |
| **最新版本号** | `max(version)` | 快速获取最新版本 |

### **性能指标**

| 指标 | 目标值 | 测量方式 |
|-----|-------|---------|
| **版本创建耗时** | < 500ms | `performance.now()` |
| **依赖解析耗时** | < 200ms | 递归调用计数 |
| **事务执行耗时** | < 100ms | Prisma metrics |
| **缓存失效耗时** | < 50ms | Redis metrics |

---

## 故障排查

### **常见问题**

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| **版本号跳跃** | 事务回滚后重试 | 正常现象，版本号不回收 |
| **类型不一致** | 尝试创建不同类型的同名提示词 | 使用不同名称 |
| **循环依赖** | A 引用 B，B 引用 A | 检查依赖图 |
| **依赖缺失** | 引用的提示词不存在 | 先创建被依赖的提示词 |
| **标签冲突** | 多个版本有相同标签 | 系统会自动移除旧版本标签 |

---

## 使用示例

### **创建第一个版本**

```typescript
const prompt = await createPrompt({
  projectId: "proj_123",
  name: "greeting",
  prompt: "Hello, {{name}}!",
  type: PromptType.Text,
  labels: ["latest"],
  commitMessage: "Initial version",
  createdBy: "user_456",
  prisma,
});

// 结果：version = 1, labels = ["latest"]
```

### **创建新版本（迭代）**

```typescript
const prompt = await createPrompt({
  projectId: "proj_123",
  name: "greeting",  // 同名
  prompt: "Hello, {{name}}! Welcome to {{app}}.",
  type: PromptType.Text,
  labels: ["latest"],
  commitMessage: "Add app variable",
  createdBy: "user_456",
  prisma,
});

// 结果：version = 2, labels = ["latest"]
// 副作用：version 1 的 "latest" 标签被移除
```

### **发布到生产环境**

```typescript
await setLabels({
  promptId: "prompt_v2_id",
  projectId: "proj_123",
  labels: ["latest", "production"],
});

// 结果：
// - version 2: labels = ["latest", "production"]
// - version 1: labels = []（如果之前有标签会被移除）
```

### **使用依赖**

```typescript
// 创建基础提示词
await createPrompt({
  name: "system-context",
  prompt: "You are a helpful assistant. Today is {{date}}.",
  // ...
});

// 创建依赖于基础提示词的新提示词
await createPrompt({
  name: "chat-prompt",
  prompt: "@@@langfusePrompt:name=system-context|label=latest@@@\n\nUser: {{message}}",
  // ...
});

// 解析后的 chat-prompt:
// "You are a helpful assistant. Today is {{date}}.\n\nUser: {{message}}"
```

---

## 相关文档

- [Label System 管理](label-system.md) - 标签管理机制
- [Cache Service 管理](cache-service.md) - Redis 缓存服务
- [01-prompt-creation-sequence.puml](01-prompt-creation-sequence.puml) - 创建时序图
- [02-prompt-fetch-sequence.puml](02-prompt-fetch-sequence.puml) - 获取时序图
