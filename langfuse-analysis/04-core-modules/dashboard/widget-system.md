# Widget System（Widget 系统）

## 一、概述

Widget System 是 Dashboard 模块的核心数据可视化组件系统，负责管理 Widget 的创建、配置、渲染和交互。系统支持 8 种图表类型，通过灵活的 Dimension + Metric 配置模型实现多维数据分析。

### 核心特点

- **6 个 Widget API**：覆盖 CRUD + 复制操作（Lines 76-240）
- **8 种图表类型**：从折线图到透视表的完整图表库
- **4 种数据视图**：Traces、Observations、Scores（Numeric/Categorical）
- **灵活查询配置**：Dimension（维度）+ Metric（指标）+ Filter（过滤器）
- **实时渲染**：基于 React 的高性能图表渲染

---

## 二、架构设计

### 1. Widget 数据模型

**数据库表结构**（PostgreSQL）：

```prisma
model DashboardWidget {
  id        String   @id @default(cuid())
  name      String
  projectId String?  // null = Langfuse Widget
  
  // 数据查询配置
  view       DashboardWidgetViews    // 枚举
  dimensions Json                   // Dimension[]
  metrics    Json                   // Metric[]
  filters    Json                   // SingleFilter[]
  
  // 可视化配置
  chartType   DashboardWidgetChartType
  chartConfig Json
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  createdBy String?
  updatedBy String?
}
```

### 2. Widget API Router（Lines 75-240）

**文件位置**：[web/src/server/api/routers/dashboardWidgets.ts](web/src/server/api/routers/dashboardWidgets.ts#L75-L240)

| 方法名 | 类型 | 代码行数 | 行号 | 功能描述 |
|-------|-----|---------|------|---------|
| **create** | Mutation | 21 | 76-96 | 创建新 Widget |
| **all** | Query | 18 | 98-115 | 列出所有 Widget |
| **get** | Query | 27 | 117-143 | 获取单个 Widget 详情 |
| **update** | Mutation | 31 | 145-175 | 更新 Widget 配置 |
| **copyToProject** | Mutation | 26 | 177-202 | 复制 Widget 到项目 |
| **delete** | Mutation | 35 | 205-239 | 删除 Widget 及关联 |

---

## 三、核心 API 详解

### 1. 创建 Widget

**API 端点**：`dashboardWidgetRouter.create`（Lines 76-96）

**实现流程**：

1. **权限验证**（Lines 77-81）：
   ```typescript
   throwIfNoProjectAccess({
     session: ctx.session,
     projectId: input.projectId,
     scope: "dashboards:CUD"
   });
   ```

2. **View 映射**（Line 88）：
   - 前端：`"traces"` / `"observations"` / `"scores-numeric"` / `"scores-categorical"`
   - 数据库：`TRACES` / `OBSERVATIONS` / `SCORES_NUMERIC` / `SCORES_CATEGORICAL`
   - 映射对象：`viewMapping`（定义在 Lines 14-19）

3. **调用 Service 层**（Lines 87-89）：
   ```typescript
   const widget = await DashboardService.createWidget(
     input.projectId,
     { ...input, view: viewMapping[input.view] },
     ctx.session.user?.id
   );
   ```

**输入参数**：

| 参数 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| projectId | string | ✅ | 项目 ID |
| name | string | ✅ | Widget 名称 |
| view | string | ✅ | 数据视图（traces/observations/scores-*） |
| dimensions | Json | ✅ | 维度配置数组 |
| metrics | Json | ✅ | 指标配置数组 |
| filters | Json | ✅ | 过滤条件数组 |
| chartType | string | ✅ | 图表类型 |
| chartConfig | Json | ✅ | 图表配置对象 |

**Service 层实现**（`DashboardService.createWidget`，Lines 271-296）：

- **验证配置**：检查 dimensions/metrics/filters 格式
- **插入数据库**：调用 `dashboardRepository.insertWidget`
- **返回完整对象**：包含生成的 Widget ID

---

### 2. 更新 Widget

**API 端点**：`dashboardWidgetRouter.update`（Lines 145-175）

**可更新字段**：

| 字段 | 说明 | 是否触发重新查询 |
|-----|------|----------------|
| name | Widget 名称 | ❌ |
| view | 数据视图 | ✅ |
| dimensions | 维度配置 | ✅ |
| metrics | 指标配置 | ✅ |
| filters | 过滤条件 | ✅ |
| chartType | 图表类型 | ❌ |
| chartConfig | 图表配置 | ❌ |

**Service 层实现**（Lines 325-353）：

1. **获取现有 Widget**：验证权限和所有权
2. **合并更新**：使用 Prisma 的 `update` 方法
3. **更新时间戳**：自动更新 `updatedAt` 和 `updatedBy`

---

### 3. 复制 Widget

**API 端点**：`dashboardWidgetRouter.copyToProject`（Lines 177-202）

**使用场景**：
- 从 Langfuse Widget 复制到项目（`projectId = null` → `projectId = "proj-xxx"`）
- 在项目内复制 Widget 进行微调

**Service 层实现**（`DashboardService.copyWidgetToProject`，Lines 400-471）：

```typescript
// 核心逻辑
const sourceWidget = await getWidget(widgetId);

const newWidget = await createWidget(targetProjectId, {
  name: `${sourceWidget.name} (Copy)`,
  view: sourceWidget.view,
  dimensions: sourceWidget.dimensions,
  metrics: sourceWidget.metrics,
  filters: sourceWidget.filters,
  chartType: sourceWidget.chartType,
  chartConfig: sourceWidget.chartConfig
}, userId);

return newWidget.id;
```

**复制策略**：

| 源 Widget | 目标 Widget | 操作 |
|----------|-----------|------|
| Langfuse Widget（projectId = null） | 项目 Widget | ✅ 允许复制 |
| 项目 Widget A | 项目 Widget B | ✅ 允许复制 |
| 项目 Widget | Langfuse Widget | ❌ 不允许（权限限制） |

---

### 4. 删除 Widget

**API 端点**：`dashboardWidgetRouter.delete`（Lines 205-239）

**删除流程**：

1. **验证权限**（Lines 206-211）
2. **检查所有权**（Lines 213-220）
3. **级联删除**（Service 层 Lines 359-395）：
   - 删除 Widget 本身
   - 自动清理所有 Dashboard 中的 Widget Placement 记录

**Service 层实现要点**：

```typescript
// 1. 查找所有引用该 Widget 的 Dashboard
const dashboardsWithWidget = await findDashboardsByWidgetId(widgetId);

// 2. 从每个 Dashboard 的 definition 中移除 Placement
for (const dashboard of dashboardsWithWidget) {
  const updatedDefinition = {
    ...dashboard.definition,
    widgets: dashboard.definition.widgets.filter(
      w => w.widgetId !== widgetId
    )
  };
  await updateDashboardDefinition(dashboard.id, updatedDefinition);
}

// 3. 删除 Widget
await dashboardRepository.deleteWidget(widgetId);
```

---

## 四、数据查询配置

### 1. View（数据视图）

**4 种视图类型**：

| View 类型 | 前端值 | 数据库枚举 | 数据源表 | 适用场景 |
|----------|--------|-----------|---------|---------|
| **Traces** | `"traces"` | `TRACES` | `traces` | Trace 级别分析 |
| **Observations** | `"observations"` | `OBSERVATIONS` | `observations` | Observation 级别分析 |
| **Scores (Numeric)** | `"scores-numeric"` | `SCORES_NUMERIC` | `scores` | 数值型评分 |
| **Scores (Categorical)** | `"scores-categorical"` | `SCORES_CATEGORICAL` | `scores` | 分类型评分 |

**View 映射对象**（Lines 14-19）：

```typescript
const viewMapping = {
  "traces": DashboardWidgetViews.TRACES,
  "observations": DashboardWidgetViews.OBSERVATIONS,
  "scores-numeric": DashboardWidgetViews.SCORES_NUMERIC,
  "scores-categorical": DashboardWidgetViews.SCORES_CATEGORICAL
};
```

---

### 2. Dimensions（维度）

**定义**：用于数据分组的字段

**配置格式**：

```typescript
type Dimension = {
  field: string;           // 字段名
  label?: string;          // 显示标签（可选）
  timeBucket?: "hour" | "day" | "week" | "month";  // 时间分桶
};
```

**常见维度示例**：

| field | 说明 | 适用 View |
|-------|------|----------|
| `model` | 模型名称 | Observations |
| `userId` | 用户 ID | Traces |
| `name` | Trace 名称 | Traces |
| `timestamp` | 时间戳（需配置 timeBucket） | 所有 |
| `scoreName` | 评分名称 | Scores |
| `type` | Observation 类型 | Observations |

**时间维度特殊处理**：

- 前端传递：`{ field: "timestamp", timeBucket: "day" }`
- QueryBuilder 处理：自动转换为 ClickHouse 的 `toStartOfDay(timestamp)` 函数

---

### 3. Metrics（指标）

**定义**：用于数据聚合的度量

**配置格式**：

```typescript
type Metric = {
  measure: string;          // 度量字段
  agg: AggregationType;     // 聚合类型
  label?: string;           // 显示标签（可选）
};

type AggregationType = 
  | "count"      // 计数
  | "sum"        // 求和
  | "avg"        // 平均
  | "min"        // 最小值
  | "max"        // 最大值
  | "median"     // 中位数
  | "p50" | "p90" | "p95" | "p99";  // 百分位数
```

**常见指标示例**：

| measure | agg | 结果字段名 | 说明 |
|---------|-----|-----------|------|
| `count` | `count` | `count_count` | 记录数 |
| `totalTokens` | `sum` | `sum_totalTokens` | Token 总和 |
| `calculatedTotalCost` | `avg` | `avg_calculatedTotalCost` | 平均成本 |
| `latency` | `p95` | `p95_latency` | 95 分位延迟 |

**命名规则**（在 QueryBuilder 中生成）：

- 格式：`{agg}_{measure}`
- 示例：`sum_totalTokens`、`avg_latency`、`count_count`

---

### 4. Filters（过滤器）

**配置格式**：

```typescript
type SingleFilter = {
  column: string;                          // 字段名
  type: "stringOptions" | "datetime" | "number";
  operator: "any of" | "none of" | "=" | ">" | "<" | ">=";
  value: string[] | Date | number;
};
```

**过滤器示例**：

```json
[
  {
    "column": "type",
    "type": "stringOptions",
    "operator": "any of",
    "value": ["GENERATION", "SPAN"]
  },
  {
    "column": "calculatedTotalCost",
    "type": "number",
    "operator": ">",
    "value": 0.01
  }
]
```

**过滤器合并优先级**：

1. **Widget Filters**（Widget 自定义过滤器）
2. **Dashboard Global Filters**（仪表板全局过滤器）
3. 两者合并后传递给 QueryBuilder

---

## 五、图表类型系统

### 1. 支持的图表类型（8 种）

| 图表类型 | chartType 值 | 适用场景 | 必需配置 |
|---------|-------------|---------|---------|
| **折线图（时间序列）** | `LINE_TIME_SERIES` | 趋势分析 | 时间维度 + 1 指标 |
| **柱状图（时间序列）** | `BAR_TIME_SERIES` | 时间对比 | 时间维度 + 1 指标 |
| **横向柱状图** | `HORIZONTAL_BAR` | 排名对比 | 1 分类维度 + 1 指标 |
| **纵向柱状图** | `VERTICAL_BAR` | 分类对比 | 1 分类维度 + 1 指标 |
| **饼图** | `PIE` | 占比分析 | 1 分类维度 + 1 指标 |
| **大数字卡片** | `NUMBER` | 单一指标 | 1 指标，0 维度 |
| **直方图** | `HISTOGRAM` | 分布分析 | 1 数值维度 |
| **透视表** | `PIVOT_TABLE` | 多维交叉 | 多维度 + 多指标 |

### 2. chartConfig 配置详解

**LINE_TIME_SERIES 配置**：

```json
{
  "type": "LINE_TIME_SERIES",
  "stacking": false,               // 是否堆叠
  "showLegend": true,              // 显示图例
  "yAxisLabel": "Total Tokens"     // Y 轴标签
}
```

**HORIZONTAL_BAR 配置**：

```json
{
  "type": "HORIZONTAL_BAR",
  "row_limit": 20,                 // 显示行数（默认 10）
  "order_by": "count_count",       // 排序字段
  "order_direction": "DESC"        // 排序方向
}
```

**PIVOT_TABLE 配置**：

```json
{
  "type": "PIVOT_TABLE",
  "row_limit": 100,
  "columnOrder": ["model", "userId", "count_count"]
}
```

**NUMBER 配置**：

```json
{
  "type": "NUMBER",
  "prefix": "$",                   // 前缀符号
  "suffix": "",                    // 后缀符号
  "decimals": 2,                   // 小数位数
  "comparisonEnabled": true,       // 启用对比
  "comparisonPeriod": "previous"   // 对比周期
}
```

---

## 六、前端渲染流程

### 1. Widget 渲染组件

**主组件**：[web/src/features/widgets/components/DashboardWidget.tsx](web/src/features/widgets/components/DashboardWidget.tsx)

**渲染流程**：

```
DashboardWidget
  ├─> 1. 获取 Widget 配置（useQuery）
  ├─> 2. 合并过滤器（Widget Filters + Dashboard Filters）
  ├─> 3. 执行查询（dashboard.executeQuery）
  ├─> 4. 数据转换（transformData）
  └─> 5. 渲染图表（ChartComponent）
```

### 2. 图表渲染库

**使用的图表库**：

| 图表类型 | 渲染库 | 备注 |
|---------|--------|------|
| LINE/BAR_TIME_SERIES | Recharts | 响应式 SVG 图表 |
| HORIZONTAL/VERTICAL_BAR | Recharts | - |
| PIE | Recharts | - |
| NUMBER | 自定义组件 | 纯 CSS + React |
| HISTOGRAM | Recharts | 使用 BarChart 实现 |
| PIVOT_TABLE | Tanstack Table | 高性能虚拟滚动 |

---

## 七、Widget 与 Dashboard 关联

### 1. Widget Placement 机制

**数据存储**：
- Widget 配置存储在 `DashboardWidget` 表
- Widget 布局存储在 `Dashboard.definition` JSON 字段

**Placement 对象格式**：

```json
{
  "type": "widget",
  "id": "placement-uuid-123",      // 唯一放置 ID
  "widgetId": "widget-uuid-456",   // Widget 引用
  "x": 0,
  "y": 0,
  "x_size": 6,
  "y_size": 4
}
```

### 2. 多 Dashboard 共享 Widget

**场景**：同一个 Widget 可以被多个 Dashboard 引用

**实现**：

```json
// Dashboard A
{
  "definition": {
    "widgets": [
      { "id": "placement-1", "widgetId": "shared-widget-1", ... }
    ]
  }
}

// Dashboard B
{
  "definition": {
    "widgets": [
      { "id": "placement-2", "widgetId": "shared-widget-1", ... }
    ]
  }
}
```

**好处**：
- ✅ 集中管理：修改 Widget 配置，所有 Dashboard 同步更新
- ✅ 节省存储：避免重复存储相同配置
- ⚠️ 注意事项：删除 Widget 会影响所有引用的 Dashboard

---

## 八、Langfuse Widget vs. Project Widget

### 1. 两种 Widget 类型

| 属性 | Langfuse Widget | Project Widget |
|-----|----------------|---------------|
| **projectId** | `null` | `"proj-xxx"` |
| **可见性** | 所有项目可见 | 仅当前项目可见 |
| **可编辑性** | ❌ 不可编辑/删除 | ✅ 可编辑/删除 |
| **用途** | 官方模板库 | 项目自定义 |
| **创建者** | Langfuse 团队 | 项目用户 |

### 2. Widget 复制工作流

**典型工作流**：

1. **浏览 Langfuse Widget**：用户在 Widget 库中浏览官方模板
2. **复制到项目**：调用 `copyToProject` API
3. **自定义配置**：修改 dimensions/metrics/filters
4. **添加到 Dashboard**：选择复制后的 Widget 添加到仪表板

**代码示例**：

```typescript
// 1. 复制 Widget
const newWidgetId = await api.dashboardWidgets.copyToProject.mutate({
  widgetId: "langfuse-widget-123",
  projectId: "proj-456"
});

// 2. 添加到 Dashboard
await api.dashboard.updateDashboardDefinition.mutate({
  dashboardId: "dash-789",
  definition: {
    widgets: [
      ...existingWidgets,
      {
        type: "widget",
        id: uuidv4(),
        widgetId: newWidgetId,
        x: 0, y: 0, x_size: 6, y_size: 4
      }
    ]
  }
});
```

---

## 九、性能优化

### 1. 查询优化

- **字段裁剪**：仅查询必需的 dimensions/metrics 字段
- **分页加载**：PIVOT_TABLE 使用虚拟滚动
- **缓存策略**：Widget 配置缓存在前端（tRPC 缓存）

### 2. 渲染优化

- **按需加载**：Dashboard 仅渲染可见 Widget
- **数据转换缓存**：使用 `useMemo` 缓存转换结果
- **Debounce 交互**：拖拽布局时防抖更新

---

## 十、错误处理

| 错误类型 | 状态码 | 错误信息 |
|---------|--------|---------|
| Widget 不存在 | 404 | "Widget not found" |
| 权限不足 | 403 | "Access denied" |
| 配置格式错误 | 400 | "Invalid dimensions/metrics format" |
| View 类型不支持 | 400 | "Unsupported view type" |
| 删除被引用 Widget | 400 | "Widget is in use by dashboards" |

---

## 十一、最佳实践

### 1. Widget 设计原则

| 原则 | 说明 |
|-----|------|
| ✅ 单一职责 | 每个 Widget 聚焦一个分析目标 |
| ✅ 合理聚合 | 避免过多维度导致数据过于分散 |
| ✅ 性能优先 | 限制数据量（row_limit）和时间范围 |
| ❌ 避免重复 | 优先复用 Langfuse Widget |

### 2. 图表选择建议

| 分析目标 | 推荐图表 | 备注 |
|---------|---------|------|
| 趋势分析 | LINE_TIME_SERIES | 适合连续数据 |
| 对比分析 | HORIZONTAL_BAR | Top N 排名 |
| 占比分析 | PIE | 不超过 7 个分类 |
| 单一指标 | NUMBER | KPI 监控 |
| 多维分析 | PIVOT_TABLE | 适合数据探索 |

### 3. 配置优化

```typescript
// ✅ 推荐：限制行数
{
  "chartType": "HORIZONTAL_BAR",
  "chartConfig": {
    "row_limit": 20  // 避免过多数据
  }
}

// ❌ 不推荐：无限制
{
  "chartType": "PIVOT_TABLE",
  "chartConfig": {}  // 可能返回海量数据
}
```

---

## 十二、相关文件

| 文件路径 | 行数 | 职责 |
|---------|------|------|
| [web/src/server/api/routers/dashboardWidgets.ts](web/src/server/api/routers/dashboardWidgets.ts#L75-L240) | 166 | Widget API 路由 |
| [packages/shared/src/server/services/DashboardService/DashboardService.ts](packages/shared/src/server/services/DashboardService/DashboardService.ts#L271-L471) | 201 | Widget Service 层 |
| [web/src/features/widgets/components/DashboardWidget.tsx](web/src/features/widgets/components/DashboardWidget.tsx) | - | Widget 渲染组件 |
| [web/src/features/widgets/components/WidgetForm.tsx](web/src/features/widgets/components/WidgetForm.tsx) | - | Widget 配置表单 |

---

## 十三、时序图参考

参见同目录下的 `.puml` 文件：

- `02-dashboard-widget-creation-sequence.puml`：Widget 创建流程
- `01-widget-query-execution-sequence.puml`：Widget 查询执行流程
