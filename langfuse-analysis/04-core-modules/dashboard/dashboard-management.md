# Dashboard Management（仪表板管理）

## 一、概述

Dashboard Management 模块负责管理 Langfuse 的仪表板生命周期，包括创建、编辑、克隆、删除等操作。该模块基于 tRPC 提供 11 个 API 端点，支持项目级和平台级两种类型的仪表板。

### 核心特点

- **双层架构**：`dashboard-router.ts`（API 层，277 行）+ `DashboardService.ts`（业务层，454 行）
- **11 个 API 端点**：覆盖完整的 CRUD 操作
- **权限控制**：基于 RBAC 的 `dashboards:CUD` 权限
- **布局管理**：JSON 格式存储 Widget 布局，支持 12 列栅格系统
- **过滤系统**：支持全局过滤器与 Widget 过滤器合并

---

## 二、架构设计

### 1. 分层架构

| 层级 | 文件 | 职责 | 代码位置 |
|-----|------|------|---------|
| **API 层** | `dashboard-router.ts` | tRPC 路由定义，权限验证 | Lines 84-360 |
| **Service 层** | `DashboardService.ts` | 业务逻辑，数据库操作 | Lines 19-472 |
| **Repository 层** | `dashboards.ts` | 数据库访问抽象 | - |

### 2. 数据模型

**Dashboard 表结构**（PostgreSQL）：

```prisma
model Dashboard {
  id          String   @id @default(cuid())
  name        String
  description String
  projectId   String?  // null = Langfuse Dashboard
  definition  Json     @default("{ \"widgets\": [] }")
  filters     Json     @default("[]")
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  createdBy   String?
  updatedBy   String?
}
```

**Definition JSON 格式**：

```json
{
  "widgets": [
    {
      "type": "widget",
      "id": "placement-uuid",
      "widgetId": "widget-uuid",
      "x": 0,
      "y": 0,
      "x_size": 6,
      "y_size": 4
    }
  ]
}
```

---

## 三、核心 API 端点

### DashboardRouter 完整方法列表（11 个）

| 方法名 | 类型 | 代码行数 | 行号 | 功能描述 |
|-------|-----|---------|------|---------|
| **chart** | Query | 58 | 85-142 | 执行遗留查询（scores/observations） |
| **scoreHistogram** | Query | 15 | 143-157 | 获取 Score 直方图数据 |
| **executeQuery** | Query | 25 | 158-182 | 执行 Widget 自定义查询 |
| **allDashboards** | Query | 18 | 184-201 | 列出所有仪表板（含权限过滤） |
| **getDashboard** | Query | 23 | 203-225 | 获取单个仪表板详情 |
| **createDashboard** | Mutation | 18 | 227-244 | 创建新仪表板 |
| **updateDashboardDefinition** | Mutation | 18 | 246-263 | 更新布局定义 |
| **updateDashboardMetadata** | Mutation | 19 | 265-283 | 更新名称和描述 |
| **cloneDashboard** | Mutation | 33 | 285-317 | 克隆仪表板 |
| **updateDashboardFilters** | Mutation | 18 | 319-336 | 更新全局过滤器 |
| **delete** | Mutation | 21 | 339-359 | 删除仪表板 |

---

## 四、核心功能详解

### 1. 创建 Dashboard

**API 端点**：`createDashboard`（Lines 227-244）

**实现流程**：

1. **权限验证**（Line 230-234）：
   - 检查 `dashboards:CUD` 权限
   - 调用 `throwIfNoProjectAccess`

2. **业务逻辑**（Line 237-241）：
   - 调用 `DashboardService.createDashboard`（Lines 68-92）
   - 初始化空布局：`{ widgets: [] }`
   - 记录创建者 `createdBy`

3. **返回结果**：完整的 Dashboard 对象

**Service 层实现**（`DashboardService.createDashboard`，Lines 68-92）：

```typescript
// 核心逻辑
await dashboardRepository.insertDashboard({
  name,
  description,
  projectId,
  createdBy: userId,
  definition: { widgets: [] },
  filters: []
});
```

**输入参数**：

| 参数 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| projectId | string | ✅ | 项目 ID |
| name | string | ✅ | 仪表板名称 |
| description | string | ❌ | 描述 |

---

### 2. 更新 Dashboard 布局

**API 端点**：`updateDashboardDefinition`（Lines 246-263）

**触发场景**：
- 拖拽 Widget 改变位置
- 调整 Widget 尺寸
- 添加/删除 Widget

**Service 层实现**（Lines 97-128）：

1. **验证布局格式**：使用 `DashboardDefinitionSchema` 验证
2. **更新数据库**：
   ```typescript
   await dashboardRepository.updateDashboard(dashboardId, {
     definition: input.definition,
     updatedBy: userId
   });
   ```

**输入参数**：

| 参数 | 类型 | 格式 |
|-----|------|------|
| definition | Json | `{ widgets: WidgetPlacement[] }` |

**WidgetPlacement 接口**：

| 字段 | 类型 | 说明 |
|-----|------|------|
| id | string | 唯一放置 ID |
| widgetId | string | Widget 引用 |
| x | number | X 坐标（0-11） |
| y | number | Y 坐标 |
| x_size | number | 宽度（1-12） |
| y_size | number | 高度 |

---

### 3. 克隆 Dashboard

**API 端点**：`cloneDashboard`（Lines 285-317）

**实现流程**：

1. **获取源 Dashboard**（Line 296-302）：
   - 调用 `DashboardService.getDashboard`
   - 验证权限

2. **创建克隆**（Line 309-314）：
   - 名称添加 `" (Clone)"` 后缀
   - 复制完整 `definition`（Widget 布局）
   - 不复制 `filters`（全局过滤器重置为空）

3. **返回新 Dashboard ID**

**Service 层实现**（在 `createDashboard` 中复用）：

```typescript
const clonedDashboard = await DashboardService.createDashboard(
  sourceDashboard.projectId,
  `${sourceDashboard.name} (Clone)`,
  sourceDashboard.description,
  userId
);

await DashboardService.updateDashboardDefinition(
  clonedDashboard.id,
  sourceDashboard.definition
);
```

---

### 4. 更新全局过滤器

**API 端点**：`updateDashboardFilters`（Lines 319-336）

**Filter 数据结构**：

```typescript
type SingleFilter = {
  column: string;
  type: "stringOptions" | "datetime" | "number";
  operator: "any of" | "none of" | "=" | ">" | "<";
  value: any;
};
```

**过滤器合并逻辑**：

1. Dashboard 全局过滤器
2. Widget 自定义过滤器
3. 两者合并后传递给查询引擎

**Service 层实现**（Lines 161-182）：

- 验证 Filter 格式
- 更新 `filters` 字段
- 影响所有挂载的 Widget

---

### 5. 删除 Dashboard

**API 端点**：`delete`（Lines 339-359）

**删除限制**：

- ✅ 可删除：项目级 Dashboard（`projectId != null`）
- ❌ 不可删除：Langfuse Dashboard（`projectId == null`）

**Service 层实现**（Lines 211-221）：

1. **验证所有权**：检查 Dashboard 是否属于该项目
2. **级联删除**：
   - 数据库自动级联删除关联的 `WidgetPlacement` 记录
   - Widget 本身不删除（可被其他 Dashboard 引用）

**错误处理**：

```typescript
if (!dashboard || dashboard.projectId !== projectId) {
  throw new Error("Dashboard not found or access denied");
}
```

---

## 五、查询执行流程

### 1. executeQuery 端点（Lines 158-182）

**调用链**：

```
Dashboard Component
  └─> dashboard.executeQuery (tRPC)
      └─> QueryBuilder.build()
          └─> executeQuery() [queryExecutor.ts]
              └─> queryClickhouse()
```

**输入参数**：

| 参数 | 类型 | 说明 |
|-----|------|------|
| projectId | string | 项目 ID |
| widgetId | string | Widget ID |
| dashboardFilters | SingleFilter[] | 全局过滤器 |
| fromTimestamp | Date | 时间范围起点 |
| toTimestamp | Date | 时间范围终点 |

**输出格式**：

```typescript
type DatabaseRow = Record<string, any>;
// 示例：
[
  {
    time_dimension: "2024-01-01T00:00:00Z",
    model: "gpt-4",
    count_count: 150,
    sum_totalTokens: 50000
  }
]
```

---

### 2. chart 端点（遗留功能，Lines 85-142）

**支持的遗留查询**：

| queryName | 功能 | 数据源 |
|-----------|------|--------|
| score-aggregate | Score 聚合统计 | PostgreSQL |
| observations-usage-by-type-timeseries | 按类型统计 Usage | ClickHouse |
| observations-cost-by-type-timeseries | 按类型统计 Cost | ClickHouse |

**注意**：这些查询未使用新的 QueryBuilder 架构，计划在未来重构。

---

## 六、权限控制

### 1. RBAC 权限检查

**所有 CUD 操作的权限验证**（Lines 230-234 示例）：

```typescript
throwIfNoProjectAccess({
  session: ctx.session,
  projectId: input.projectId,
  scope: "dashboards:CUD"
});
```

**权限范围**：

| 操作类型 | 所需权限 | 适用端点 |
|---------|---------|---------|
| 读取 | `dashboards:read` | getDashboard, allDashboards |
| 创建/更新/删除 | `dashboards:CUD` | create, update, delete |

### 2. Dashboard 类型权限

| Dashboard 类型 | projectId | 可见性 | 可编辑性 |
|---------------|-----------|--------|---------|
| **Langfuse Dashboard** | `null` | 所有项目 | ❌ 不可编辑/删除 |
| **Project Dashboard** | `"proj-xxx"` | 仅当前项目 | ✅ 可编辑/删除 |

---

## 七、DashboardService 方法清单

| 方法名 | 代码行数 | 行号 | 功能描述 |
|-------|---------|------|---------|
| **listDashboards** | 41 | 23-63 | 列出所有仪表板 |
| **createDashboard** | 25 | 68-92 | 创建新仪表板 |
| **updateDashboardDefinition** | 32 | 97-128 | 更新布局定义 |
| **updateDashboard** | 24 | 133-156 | 更新元数据 |
| **updateDashboardFilters** | 22 | 161-182 | 更新全局过滤器 |
| **getDashboard** | 20 | 187-206 | 获取单个仪表板 |
| **deleteDashboard** | 11 | 211-221 | 删除仪表板 |
| **listWidgets** | 41 | 226-266 | 列出所有 Widget |
| **createWidget** | 26 | 271-296 | 创建 Widget |
| **getWidget** | 20 | 301-320 | 获取 Widget 详情 |
| **updateWidget** | 29 | 325-353 | 更新 Widget |
| **deleteWidget** | 37 | 359-395 | 删除 Widget |
| **copyWidgetToProject** | 72 | 400-471 | 复制 Widget 到项目 |

**总计**：13 个方法，454 行代码

---

## 八、布局系统

### 1. 栅格系统

**12 列栅格布局**（React Grid Layout）：

| 参数 | 说明 | 取值范围 |
|-----|------|---------|
| x | 列起点 | 0-11 |
| x_size | 列宽度 | 1-12 |
| y | 行起点 | 0-∞ |
| y_size | 行高度 | 1-∞ |

**典型布局示例**：

```json
{
  "widgets": [
    { "x": 0, "y": 0, "x_size": 6, "y_size": 4 },  // 左侧半屏
    { "x": 6, "y": 0, "x_size": 6, "y_size": 4 },  // 右侧半屏
    { "x": 0, "y": 4, "x_size": 12, "y_size": 6 }  // 全屏
  ]
}
```

### 2. 拖拽更新流程

1. **用户拖拽**：`DashboardGrid.tsx` 监听 `onLayoutChange`
2. **本地更新**：立即更新 UI 状态
3. **持久化**：调用 `updateDashboardDefinition` mutation
4. **冲突处理**：基于 `updatedAt` 字段的乐观锁

---

## 九、性能优化

### 1. 查询优化

- **Shadow Testing**：双查询并行执行，对比优化效果（在 `executeQuery` 中）
- **单层查询优化**：满足条件时跳过嵌套查询
- **时间粒度自适应**：根据时间范围自动选择 hour/day/week

### 2. 缓存策略

- **前端缓存**：tRPC 客户端缓存 Dashboard 元数据
- **数据库索引**：`projectId` 字段建立索引

---

## 十、最佳实践

### 1. Dashboard 设计

| 实践 | 说明 |
|-----|------|
| ✅ 合理命名 | 使用描述性名称如 "Model Performance Dashboard" |
| ✅ 精简 Widget | 单个 Dashboard 不超过 10 个 Widget |
| ✅ 复用 Widget | 使用 Langfuse Widget 作为模板 |
| ❌ 避免过度过滤 | 过多的全局过滤器影响性能 |

### 2. 布局规范

- **标准卡片**：`x_size: 6`, `y_size: 4`（半屏）
- **大数字卡片**：`x_size: 3`, `y_size: 3`（小卡片）
- **表格/图表**：`x_size: 12`, `y_size: 6`（全屏）

### 3. 过滤器设计

- **全局过滤器**：适用于时间范围、用户 ID 等通用维度
- **Widget 过滤器**：适用于特定 Widget 的细分维度（如模型类型）

---

## 十一、错误处理

| 错误类型 | HTTP 状态码 | 错误信息 |
|---------|------------|---------|
| Dashboard 不存在 | 404 | "Dashboard not found" |
| 权限不足 | 403 | "Access denied" |
| 删除 Langfuse Dashboard | 403 | "Cannot delete Langfuse dashboard" |
| 布局格式错误 | 400 | "Invalid definition format" |

---

## 十二、相关文件

| 文件路径 | 行数 | 职责 |
|---------|------|------|
| [web/src/features/dashboard/server/dashboard-router.ts](web/src/features/dashboard/server/dashboard-router.ts#L84-L360) | 277 | tRPC 路由定义 |
| [packages/shared/src/server/services/DashboardService/DashboardService.ts](packages/shared/src/server/services/DashboardService/DashboardService.ts#L19-L472) | 454 | 业务逻辑层 |
| [packages/shared/src/server/repositories/dashboards.ts](packages/shared/src/server/repositories/dashboards.ts) | - | 数据库访问层 |
| [web/src/features/dashboard/components/DashboardGrid.tsx](web/src/features/dashboard/components/DashboardGrid.tsx) | - | 前端布局组件 |

---

## 十三、时序图参考

参见同目录下的 `.puml` 文件：

- `02-dashboard-widget-creation-sequence.puml`：Dashboard 创建流程
- `01-widget-query-execution-sequence.puml`：查询执行流程
