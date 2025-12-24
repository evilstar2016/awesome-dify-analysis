# Dashboard & Analytics 模块

## 模块概述

Dashboard & Analytics 模块是 Langfuse 的**数据可视化与分析中心**，提供灵活的多维度数据查询、聚合和可视化能力。该模块允许用户创建自定义仪表板，通过 Widget（小部件）系统展示 Traces、Observations、Scores 等核心数据的时间序列、分布统计和多维分析。

**核心价值**：
- 📊 **灵活的数据探索**：通过拖拽式 Widget 组合，自定义数据分析视图
- 🎯 **多维度聚合**：支持按时间、模型、用户、标签等维度聚合分析
- ⚡ **实时查询**：基于 ClickHouse 的高性能 OLAP 查询引擎
- 🔄 **响应式布局**：基于 react-grid-layout 的自适应网格系统
- 🎨 **丰富的图表类型**：支持 8 种图表类型，满足不同分析场景

---

## 核心概念

### 1. Dashboard（仪表板）

Dashboard 是一个**容器**，用于组织和管理多个 Widget 的布局和配置。

**关键属性**：
```typescript
interface DashboardDomain {
  id: string;                    // Dashboard ID
  name: string;                  // 名称
  description: string;           // 描述
  projectId: string | null;      // 所属项目（null 表示 Langfuse 全局）
  owner: "LANGFUSE" | "PROJECT"; // 所有者类型
  definition: {                  // 布局定义
    widgets: WidgetPlacement[];  // Widget 位置和尺寸
  };
  filters: SingleFilter[];       // 全局过滤器配置
  createdAt: Date;
  updatedAt: Date;
  createdBy: string | null;
  updatedBy: string | null;
}
```

**两类 Dashboard**：
- **Langfuse Dashboard**：系统预置的全局仪表板，所有项目可见，不可编辑
- **Project Dashboard**：项目自定义仪表板，支持 CRUD 操作

### 2. Widget（小部件）

Widget 是**独立的数据查询和可视化单元**，每个 Widget 包含：
- **数据源配置**：View（数据视图）、Dimensions（维度）、Metrics（指标）
- **筛选条件**：Filters（静态过滤）+ Global Filters（全局过滤）
- **可视化配置**：ChartType（图表类型）、ChartConfig（图表配置）

**Widget 结构**：
```typescript
interface WidgetDomain {
  id: string;                        // Widget ID
  name: string;                      // Widget 名称
  description: string;               // Widget 描述
  projectId: string | null;          // 所属项目
  owner: "LANGFUSE" | "PROJECT";     // 所有者
  
  // 数据查询配置
  view: DashboardWidgetViews;        // TRACES | OBSERVATIONS | SCORES_NUMERIC | SCORES_CATEGORICAL
  dimensions: Dimension[];           // 分组维度（如 model, userId）
  metrics: Metric[];                 // 聚合指标（如 count, sum, avg）
  filters: SingleFilter[];           // 静态过滤条件
  
  // 可视化配置
  chartType: DashboardWidgetChartType;  // 图表类型
  chartConfig: ChartConfig;             // 图表特定配置
  
  // 元数据
  createdAt: Date;
  updatedAt: Date;
  createdBy: string | null;
  updatedBy: string | null;
}
```

### 3. WidgetPlacement（Widget 布局）

WidgetPlacement 定义 Widget 在 Dashboard 中的**位置和尺寸**：

```typescript
interface WidgetPlacement {
  type: "widget";
  id: string;           // Placement ID（唯一标识符）
  widgetId: string;     // 关联的 Widget ID
  x: number;            // X 坐标（0-11，12列网格）
  y: number;            // Y 坐标
  x_size: number;       // 宽度（列数）
  y_size: number;       // 高度（行数）
}
```

**网格系统**：
- 12 列响应式布局（Bootstrap-like）
- 动态行高计算：`rowHeight = (containerWidth / 12 * 9) / 16`
- 最小尺寸：`minW: 2, minH: 2`

### 4. Query System（查询系统）

Dashboard 的查询系统基于**统一查询接口**，支持多维度聚合分析：

```typescript
interface QueryType {
  view: ViewType;                   // 数据视图（traces, observations, etc.）
  dimensions: Dimension[];          // 分组维度
  metrics: MetricWithAgg[];         // 聚合指标
  filters: SingleFilter[];          // 筛选条件
  timeDimension: {                  // 时间维度
    granularity: "auto" | "hour" | "day" | "week" | "month";
  } | null;
  fromTimestamp: string;            // 起始时间（ISO 8601）
  toTimestamp: string;              // 结束时间
  orderBy: OrderBy[] | null;        // 排序规则
  chartConfig?: ChartConfig;        // 图表配置
}
```

**执行流程**：
1. 前端构建 `QueryType` 对象
2. tRPC 调用 `dashboard.executeQuery`
3. 后端将 Query 转换为 ClickHouse SQL
4. 执行查询并返回结果
5. 前端转换数据格式并渲染图表

### 5. Filter State（过滤状态）

Dashboard 支持**全局过滤**和**Widget 级过滤**两层过滤机制：

```typescript
type FilterState = SingleFilter[];

// 示例：按时间和模型过滤
const globalFilters: FilterState = [
  {
    column: "startTime",
    type: "datetime",
    operator: ">=",
    value: "2024-01-01T00:00:00Z",
  },
  {
    column: "model",
    type: "stringOptions",
    operator: "any of",
    value: ["gpt-4", "claude-3-opus"],
  },
];
```

**过滤优先级**：`Widget Filters + Global Filters` → 合并后传递给查询引擎

### 6. Chart Types（图表类型）

支持 8 种图表类型，适配不同数据分析场景：

| 图表类型 | 用途 | 适用数据 |
|---------|------|---------|
| **LINE_TIME_SERIES** | 折线图（时间序列） | 趋势分析（如 Token 使用量） |
| **BAR_TIME_SERIES** | 柱状图（时间序列） | 时间对比（如每日 Traces 数） |
| **HORIZONTAL_BAR** | 横向柱状图 | 排名对比（如 Top 10 用户） |
| **VERTICAL_BAR** | 纵向柱状图 | 分类对比（如模型分布） |
| **PIE** | 饼图 | 占比分析（如成本分布） |
| **NUMBER** | 大数字卡片 | 单一指标（如总成本） |
| **HISTOGRAM** | 直方图 | 分布分析（如 Latency 分布） |
| **PIVOT_TABLE** | 透视表 | 多维交叉分析 |

---

## 数据模型

### 1. Dashboard 数据模型

**数据库表**：`Dashboard`（PostgreSQL）

```prisma
model Dashboard {
  id          String   @id @default(cuid())
  name        String
  description String
  projectId   String?  @map("project_id")
  
  // JSON 存储布局定义
  definition  Json     @default("{ \"widgets\": [] }")
  filters     Json     @default("[]") @map("filters")
  
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")
  createdBy   String?  @map("created_by")
  updatedBy   String?  @map("updated_by")
  
  project     Project? @relation(fields: [projectId], references: [id])
  
  @@index([projectId])
}
```

**Definition JSON 结构**：
```json
{
  "widgets": [
    {
      "type": "widget",
      "id": "placement-uuid-1",
      "widgetId": "widget-uuid-1",
      "x": 0,
      "y": 0,
      "x_size": 6,
      "y_size": 4
    }
  ]
}
```

### 2. Widget 数据模型

**数据库表**：`DashboardWidget`（PostgreSQL）

```prisma
model DashboardWidget {
  id        String                  @id @default(cuid())
  name      String
  projectId String?                 @map("project_id")
  
  // 数据查询配置
  view       DashboardWidgetViews    // 枚举: TRACES, OBSERVATIONS, SCORES_NUMERIC, SCORES_CATEGORICAL
  dimensions Json                   // Dimension[]
  metrics    Json                   // Metric[]
  filters    Json                   // SingleFilter[]
  
  // 可视化配置
  chartType   DashboardWidgetChartType @map("chart_type")
  chartConfig Json                     @map("chart_config")
  
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  createdBy String?  @map("created_by")
  updatedBy String?  @map("updated_by")
  
  project   Project? @relation(fields: [projectId], references: [id])
  
  @@index([projectId])
}
```

**Enum 类型**：
```prisma
enum DashboardWidgetViews {
  TRACES
  OBSERVATIONS
  SCORES_NUMERIC
  SCORES_CATEGORICAL
}

enum DashboardWidgetChartType {
  LINE_TIME_SERIES
  BAR_TIME_SERIES
  HORIZONTAL_BAR
  VERTICAL_BAR
  PIE
  NUMBER
  HISTOGRAM
  PIVOT_TABLE
}
```

**View 映射关系**（前端 API 与数据库枚举的映射）：
```typescript
// 前端使用字符串格式
const viewMapping = {
  "traces": DashboardWidgetViews.TRACES,
  "observations": DashboardWidgetViews.OBSERVATIONS,
  "scores-numeric": DashboardWidgetViews.SCORES_NUMERIC,
  "scores-categorical": DashboardWidgetViews.SCORES_CATEGORICAL,
};
// 所以在前端 API 调用中使用 "traces", "observations", "scores-numeric", "scores-categorical"
// 而在数据库中存储为枚举值 TRACES, OBSERVATIONS, SCORES_NUMERIC, SCORES_CATEGORICAL
```

### 3. Query Result 数据格式

**查询返回格式**：
```typescript
type DatabaseRow = Record<string, any>;

// 示例返回数据
const result: DatabaseRow[] = [
  {
    time_dimension: "2024-01-01T00:00:00Z",
    model: "gpt-4",
    count_count: 150,              // count(*)
    sum_totalTokens: 50000,        // sum(totalTokens)
    avg_calculatedTotalCost: 0.15, // avg(calculatedTotalCost)
  },
  // ...
];
```

**命名规则**：
- 聚合指标：`{agg}_{measure}`（如 `sum_totalTokens`）
- 时间维度：固定为 `time_dimension`
- 维度字段：保持原字段名（如 `model`, `userId`）

---

## 核心功能

### 1. Dashboard 管理

#### 1.1 创建 Dashboard

**流程**：
1. 用户点击 "New Dashboard" 按钮
2. 填写名称和描述
3. 调用 `dashboard.createDashboard` mutation
4. 跳转到新创建的 Dashboard 页面

**API**：
```typescript
// tRPC Mutation
api.dashboard.createDashboard.useMutation({
  projectId: "proj-123",
  name: "Model Performance",
  description: "Track model latency and cost",
});

// 后端逻辑
DashboardService.createDashboard(
  projectId,
  name,
  description,
  userId,
  { widgets: [] } // 初始空布局
);
```

#### 1.2 编辑 Dashboard 元数据

**支持编辑**：
- 名称（name）
- 描述（description）

**API**：
```typescript
api.dashboard.updateDashboardMetadata.useMutation({
  dashboardId: "dash-123",
  projectId: "proj-123",
  name: "Updated Name",
  description: "Updated Description",
});
```

#### 1.3 更新 Dashboard 布局

**触发时机**：
- 拖拽 Widget 改变位置
- 调整 Widget 尺寸
- 添加/删除 Widget

**API**：
```typescript
api.dashboard.updateDashboardDefinition.useMutation({
  dashboardId: "dash-123",
  projectId: "proj-123",
  definition: {
    widgets: [
      {
        type: "widget",
        id: "placement-1",
        widgetId: "widget-1",
        x: 0,
        y: 0,
        x_size: 6,
        y_size: 4,
      },
      // ...
    ],
  },
});
```

#### 1.4 克隆 Dashboard

**功能**：复制 Dashboard 及其所有 Widget 布局

**API**：
```typescript
api.dashboard.cloneDashboard.useMutation({
  dashboardId: "dash-123",
  projectId: "proj-123",
});

// 后端逻辑：
// 1. 获取源 Dashboard
// 2. 创建新 Dashboard，name 添加 " (Clone)" 后缀
// 3. 复制 definition（Widget 布局）
```

#### 1.5 删除 Dashboard

**限制**：
- 只能删除 Project Dashboard
- Langfuse Dashboard 不可删除

**API**：
```typescript
api.dashboard.delete.useMutation({
  dashboardId: "dash-123",
  projectId: "proj-123",
});
```

### 2. Widget 管理

#### 2.1 创建 Widget

**流程**：
1. 进入 Widget 创建页面
2. 配置数据查询（View, Dimensions, Metrics, Filters）
3. 选择图表类型和配置
4. 保存并添加到 Dashboard

**API**：
```typescript
// Step 1: 创建 Widget
const widget = await api.dashboardWidgets.create.mutate({
  projectId: "proj-123",
  name: "Token Usage by Model",
  view: "observations",
  dimensions: [{ field: "model" }],
  metrics: [
    { measure: "totalTokens", agg: "sum" },
  ],
  filters: [
    {
      column: "type",
      type: "stringOptions",
      operator: "any of",
      value: ["GENERATION"],
    },
  ],
  chartType: "LINE_TIME_SERIES",
  chartConfig: { type: "LINE_TIME_SERIES" },
});

// Step 2: 添加到 Dashboard
const placement = {
  type: "widget",
  id: uuidv4(),
  widgetId: widget.id,
  x: 0,
  y: 0,
  x_size: 12,
  y_size: 4,
};

await api.dashboard.updateDashboardDefinition.mutate({
  dashboardId: "dash-123",
  definition: {
    widgets: [...existingWidgets, placement],
  },
});
```

#### 2.2 编辑 Widget

**可编辑内容**：
- Widget 名称
- 数据查询配置（View, Dimensions, Metrics, Filters）
- 图表类型和配置

**API**：
```typescript
api.dashboardWidgets.update.useMutation({
  widgetId: "widget-123",
  projectId: "proj-123",
  name: "Updated Widget",
  view: "traces",
  dimensions: [{ field: "userId" }],
  metrics: [{ measure: "count", agg: "count" }],
  filters: [],
  chartType: "HORIZONTAL_BAR",
  chartConfig: {
    type: "HORIZONTAL_BAR",
    row_limit: 20,
  },
});
```

#### 2.3 复制 Widget

**场景**：
- 从 Langfuse Widget 复制到项目
- 在项目内复制 Widget 进行微调

**API**：
```typescript
api.dashboardWidgets.copyToProject.useMutation({
  widgetId: "widget-123",
  projectId: "proj-123",
  dashboardId: "dash-123",
  placementId: "placement-123",
});

// 后端逻辑：
// 1. 查询源 Widget
// 2. 创建新 Widget（owner = PROJECT）
// 3. 返回新 Widget ID 用于添加到 Dashboard
```

#### 2.4 删除 Widget

**限制**：
- 只能删除未被任何 Dashboard 使用的 Widget
- 正在使用的 Widget 需要先从所有 Dashboard 移除

**API**：
```typescript
// 从 Dashboard 移除 Widget
api.dashboard.updateDashboardDefinition.mutate({
  definition: {
    widgets: widgets.filter(w => w.id !== placementId),
  },
});

// 删除 Widget（如果未被使用）
api.dashboardWidgets.delete.mutate({
  widgetId: "widget-123",
  projectId: "proj-123",
});
```

### 3. 查询执行

#### 3.1 executeQuery API

**最核心的 API**，执行任意数据查询：

```typescript
const result = await api.dashboard.executeQuery.useQuery({
  projectId: "proj-123",
  query: {
    view: "observations",
    dimensions: [{ field: "model" }],
    metrics: [
      { measure: "totalTokens", aggregation: "sum" },
      { measure: "calculatedTotalCost", aggregation: "sum" },
    ],
    filters: [
      {
        column: "type",
        type: "stringOptions",
        operator: "any of",
        value: ["GENERATION"],
      },
    ],
    timeDimension: { granularity: "day" },
    fromTimestamp: "2024-01-01T00:00:00Z",
    toTimestamp: "2024-01-31T23:59:59Z",
    orderBy: [
      { field: "sum_totalTokens", direction: "desc" },
    ],
  },
});
```

**返回数据示例**：
```typescript
[
  {
    time_dimension: "2024-01-01T00:00:00Z",
    model: "gpt-4",
    sum_totalTokens: 50000,
    sum_calculatedTotalCost: 1.25,
  },
  {
    time_dimension: "2024-01-01T00:00:00Z",
    model: "gpt-3.5-turbo",
    sum_totalTokens: 120000,
    sum_calculatedTotalCost: 0.24,
  },
  // ...
]
```

#### 3.2 Legacy Chart API

**遗留 API**，用于特定图表查询（逐步迁移到 executeQuery）：

```typescript
// score-aggregate（Score 聚合）
api.dashboard.chart.useQuery({
  projectId: "proj-123",
  queryName: "score-aggregate",
  filter: globalFilters,
});

// observations-usage-by-type-timeseries
api.dashboard.chart.useQuery({
  projectId: "proj-123",
  from: "traces_observations",
  select: [
    { column: "totalTokens", agg: "SUM" },
    { column: "type" },
  ],
  groupBy: [
    { type: "datetime", column: "startTime", temporalUnit: "day" },
    { type: "string", column: "type" },
  ],
  queryName: "observations-usage-by-type-timeseries",
});
```

#### 3.3 scoreHistogram API

**专用于 Score 分布分析**：

```typescript
const histogram = api.dashboard.scoreHistogram.useQuery({
  projectId: "proj-123",
  filter: [
    {
      column: "name",
      type: "string",
      operator: "=",
      value: "quality",
    },
  ],
  limit: 10000,
});

// 返回：直方图数据（bins, counts）
```

### 4. 数据转换

#### 4.1 时间序列数据转换

**目标格式**：
```typescript
interface TimeSeriesChartDataPoint {
  ts: number;  // Unix timestamp (ms)
  values: {
    label: string;
    value: number;
  }[];
}
```

**转换逻辑**：
```typescript
const transformedData = queryResult.data.map((item) => ({
  ts: new Date(item.time_dimension).getTime(),
  values: [
    {
      label: item.model,  // 维度作为 label
      value: Number(item.sum_totalTokens),  // 指标值
    },
  ],
}));
```

#### 4.2 分类数据转换

**用于柱状图、饼图等**：

```typescript
const transformedData = queryResult.data.map((item) => ({
  dimension: item.model || item.userId || "n/a",
  metric: Number(item.count_count),
}));
```

#### 4.3 Pivot Table 数据

**保留原始数据结构**，由 PivotTable 组件处理：

```typescript
const transformedData = queryResult.data.map((item) => ({
  ...item,  // 保留所有字段
  dimension: dimensions[0]?.field ?? "dimension",
  metric: 0, // placeholder
  time_dimension: item.time_dimension,
}));
```

### 5. 全局过滤

#### 5.1 Date Range Filter

**顶部日期范围选择器**，影响所有 Widget：

```typescript
const { dateRange } = useDashboardDateRange();
// dateRange: { from: Date, to: Date }

// 每个 Widget 查询时自动应用
const query = {
  fromTimestamp: dateRange.from.toISOString(),
  toTimestamp: dateRange.to.toISOString(),
  // ...
};
```

#### 5.2 Filter State

**URL 参数化过滤器**，支持页面刷新保持：

```typescript
const [filterState, setFilterState] = useFilterState();

// 添加过滤条件
setFilterState([
  ...filterState,
  {
    column: "userId",
    type: "string",
    operator: "=",
    value: "user-123",
  },
]);

// 应用到所有 Widget
const query = {
  filters: [
    ...widgetFilters,
    ...mapLegacyUiTableFilterToView(view, filterState),
  ],
};
```

---

## 技术架构

### 1. 前端架构

```
Dashboard Page
├── DashboardGrid (react-grid-layout)
│   ├── DashboardWidget (Widget 容器)
│   │   ├── Data Query (tRPC)
│   │   ├── Data Transformation
│   │   └── Chart Rendering
│   │       ├── BaseTimeSeriesChart (Recharts)
│   │       ├── HorizontalBarChart
│   │       ├── PieChart
│   │       ├── BigNumber
│   │       └── PivotTable
│   └── Layout Management
│       ├── Drag & Drop
│       ├── Resize
│       └── Responsive Breakpoints
├── Global Filters
│   ├── Date Range Picker
│   └── Filter Builder
└── Dashboard Actions
    ├── Add Widget Dialog
    ├── Edit Dashboard
    └── Clone Dashboard
```

**关键组件**：

1. **DashboardGrid**（`web/src/features/widgets/components/DashboardGrid.tsx`）
   - 基于 `react-grid-layout` 的响应式网格
   - 支持拖拽和调整尺寸
   - 12 列布局，动态行高

2. **DashboardWidget**（`web/src/features/widgets/components/DashboardWidget.tsx`）
   - Widget 渲染容器
   - 处理数据查询和转换
   - 根据 chartType 渲染不同图表

3. **BaseTimeSeriesChart**（`web/src/features/dashboard/components/BaseTimeSeriesChart.tsx`）
   - 基于 Recharts 的时间序列图表基类
   - 支持多条线/柱对比
   - 自定义 Tooltip

4. **PivotTable**（`web/src/features/widgets/chart-library/PivotTable.tsx`）
   - 透视表组件
   - 支持动态排序
   - 多列展示

### 2. 后端架构

```
tRPC Router
├── dashboardRouter
│   ├── allDashboards (list)
│   ├── getDashboard (get by id)
│   ├── createDashboard
│   ├── updateDashboardDefinition (layout)
│   ├── updateDashboardMetadata (name, desc)
│   ├── cloneDashboard
│   ├── delete
│   ├── executeQuery ⭐ (核心查询 API)
│   ├── chart (legacy API)
│   └── scoreHistogram
└── dashboardWidgetRouter
    ├── all (list)
    ├── get (get by id)
    ├── create
    ├── update
    ├── copyToProject
    └── delete
```

**核心服务**：

1. **DashboardService**（`packages/shared/src/server/services/DashboardService/`）
   ```typescript
   class DashboardService {
     // Dashboard CRUD
     static listDashboards(props): Promise<DashboardListResponse>
     static getDashboard(id, projectId): Promise<DashboardDomain>
     static createDashboard(projectId, name, desc, userId): Promise<DashboardDomain>
     static updateDashboardDefinition(id, projectId, definition, userId)
     static updateDashboard(id, projectId, name, desc, userId)
     static deleteDashboard(id, projectId)
     
     // Widget CRUD
     static listWidgets(props): Promise<WidgetListResponse>
     static getWidget(id, projectId): Promise<WidgetDomain>
     static createWidget(input): Promise<WidgetDomain>
     static updateWidget(id, input): Promise<WidgetDomain>
     static deleteWidget(id, projectId)
     static copyWidgetToProject(widgetId, projectId): Promise<WidgetDomain>
   }
   ```

2. **Query Execution**
   - 查询构建：前端 `QueryType` → 后端 SQL 生成
   - 过滤转换：`createFilterFromFilterState()` → ClickHouse WHERE 子句
   - 聚合构建：Dimensions + Metrics → GROUP BY + Aggregations
   - 时间分桶：`timeDimension.granularity` → `toStartOfDay()` / `toStartOfHour()`

### 3. 数据流

```
┌─────────────────────────────────────────────────────────┐
│ 1. User Interaction                                     │
│    - 选择 Dashboard                                      │
│    - 设置 Date Range / Filters                          │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Frontend (DashboardWidget)                           │
│    - 获取 Widget 配置（view, dimensions, metrics）       │
│    - 构建 QueryType 对象                                 │
│    - 合并 Widget Filters + Global Filters               │
│    - 调用 api.dashboard.executeQuery.useQuery()         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 3. tRPC API (dashboardRouter.executeQuery)             │
│    - 权限校验（dashboards:read）                         │
│    - 调用 executeQuery(projectId, query)                │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Query Builder                                        │
│    - 解析 QueryType                                      │
│    - 映射 View → ClickHouse Table                       │
│    - 构建 SELECT, GROUP BY, WHERE, ORDER BY             │
│    - 应用时间分桶（toStartOfDay, etc.）                  │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 5. ClickHouse                                           │
│    - 执行 OLAP 查询                                      │
│    - 返回聚合结果                                         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Data Transformation                                  │
│    - 解析查询结果                                         │
│    - 转换为图表数据格式                                    │
│    - 应用 chartConfig 配置                               │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 7. Chart Rendering                                      │
│    - BaseTimeSeriesChart (Recharts)                    │
│    - PivotTable (Custom Table)                          │
│    - BigNumber, PieChart, etc.                          │
└─────────────────────────────────────────────────────────┘
```

### 4. 查询优化

**性能优化策略**：

1. **ClickHouse OLAP 引擎**
   - 列式存储，聚合查询高效
   - 时间分区，快速范围查询
   - 预聚合表（Materialized Views）

2. **查询批处理**
   - `skipBatch: true`：跳过 tRPC 批处理，独立执行
   - 避免多个 Widget 查询阻塞

3. **缓存策略**
   - tRPC Query 默认缓存（React Query）
   - 相同查询参数复用结果

4. **按需加载**
   - Widget 可见时才执行查询（`enabled` 条件）
   - 懒加载图表库（dynamic import）

---

## UI 组件

### 1. Chart Components（图表组件）

| 组件 | 路径 | 用途 |
|------|------|------|
| **BaseTimeSeriesChart** | `dashboard/components/BaseTimeSeriesChart.tsx` | 时间序列图表基类（折线图、柱状图） |
| **TracesTimeSeriesChart** | `dashboard/components/TracesTimeSeriesChart.tsx` | Traces 和 Observations 时间趋势 |
| **ModelUsageChart** | `dashboard/components/ModelUsageChart.tsx` | 模型使用量和成本时间序列 |
| **LatencyChart** | `dashboard/components/LatencyChart.tsx` | Latency 时间趋势（支持按模型分组） |
| **ChartScores** | `dashboard/components/ChartScores.tsx` | Score 时间序列 |
| **TracesBarListChart** | `dashboard/components/TracesBarListChart.tsx` | Traces Top N 排名 |
| **UserChart** | `dashboard/components/UserChart.tsx` | 用户维度分析 |
| **PivotTable** | `widgets/chart-library/PivotTable.tsx` | 透视表（支持动态排序） |

### 2. Table Components（表格组件）

| 组件 | 路径 | 用途 |
|------|------|------|
| **DashboardTable** | `dashboard/components/DashboardTable.tsx` | Dashboard 列表（支持克隆、编辑、删除） |
| **ModelCostTable** | `dashboard/components/ModelCostTable.tsx` | 模型成本排名表 |
| **LatencyTables** | `dashboard/components/LatencyTables.tsx` | Latency 分位数表 |
| **ScoresTable** | `dashboard/components/ScoresTable.tsx` | Score 统计表 |
| **DashboardTable (Card)** | `dashboard/components/cards/DashboardTable.tsx` | 通用表格卡片 |

### 3. Utility Components（工具组件）

| 组件 | 路径 | 用途 |
|------|------|------|
| **DashboardCard** | `dashboard/components/cards/DashboardCard.tsx` | Widget 卡片容器 |
| **TotalMetric** | `dashboard/components/TotalMetric.tsx` | 总计指标显示 |
| **TabComponent** | `dashboard/components/TabsComponent.tsx` | Tab 切换（如 Cost vs Usage） |
| **ModelSelector** | `dashboard/components/ModelSelector.tsx` | 模型多选下拉框 |
| **Tooltip** | `dashboard/components/Tooltip.tsx` | 图表自定义 Tooltip |
| **EditDashboardDialog** | `dashboard/components/EditDashboardDialog.tsx` | Dashboard 编辑对话框 |
| **SelectDashboardDialog** | `dashboard/components/SelectDashboardDialog.tsx` | Dashboard 选择对话框 |

### 4. Grid Components（网格组件）

| 组件 | 路径 | 用途 |
|------|------|------|
| **DashboardGrid** | `widgets/components/DashboardGrid.tsx` | 响应式网格布局容器 |
| **DashboardWidget** | `widgets/components/DashboardWidget.tsx` | Widget 查询和渲染逻辑 |

---

## 使用场景

### 场景 1：监控模型性能

**需求**：实时监控 GPT-4 和 Claude 的 Latency 和成本

**步骤**：
1. 创建 Dashboard "Model Performance"
2. 添加 Widget：
   - **Latency Time Series**（LINE_TIME_SERIES）
     - View: `observations`
     - Dimensions: `[{ field: "model" }]`
     - Metrics: `[{ measure: "latency", agg: "avg" }]`
     - Filters: `model in ["gpt-4", "claude-3-opus"]`
   
   - **Cost Time Series**（BAR_TIME_SERIES）
     - View: `observations`
     - Dimensions: `[{ field: "model" }]`
     - Metrics: `[{ measure: "calculatedTotalCost", agg: "sum" }]`
   
   - **Total Cost**（NUMBER）
     - Metrics: `[{ measure: "calculatedTotalCost", agg: "sum" }]`

3. 设置 Date Range 为 "Last 7 Days"
4. 结果：实时查看模型性能趋势

### 场景 2：分析用户行为

**需求**：Top 20 活跃用户及其 Token 消耗

**步骤**：
1. 创建 Widget "Top Users"
2. 配置：
   - ChartType: `HORIZONTAL_BAR`
   - View: `traces`
   - Dimensions: `[{ field: "userId" }]`
   - Metrics: `[{ measure: "count", agg: "count" }]`
   - ChartConfig: `{ type: "HORIZONTAL_BAR", row_limit: 20 }`
   - OrderBy: `[{ field: "count_count", direction: "desc" }]`

3. 添加到 Dashboard
4. 结果：横向柱状图展示 Top 20 用户

### 场景 3：Score 分布分析

**需求**：分析 "quality" Score 的数值分布

**步骤**：
1. 创建 Widget "Quality Score Distribution"
2. 配置：
   - ChartType: `HISTOGRAM`
   - 使用 `scoreHistogram` API
   - Filters: `[{ column: "name", value: "quality" }]`
   - ChartConfig: `{ type: "HISTOGRAM", bins: 20 }`

3. 结果：直方图展示 Score 分布（如大部分在 0.8-1.0 区间）

### 场景 4：多维透视分析

**需求**：按用户和模型交叉分析 Token 消耗

**步骤**：
1. 创建 Widget "User-Model Pivot"
2. 配置：
   - ChartType: `PIVOT_TABLE`
   - View: `observations`
   - Dimensions: `[{ field: "userId" }, { field: "model" }]`
   - Metrics: `[{ measure: "totalTokens", agg: "sum" }]`
   - ChartConfig: `{ 
       type: "PIVOT_TABLE", 
       row_limit: 100,
       defaultSort: { column: "sum_totalTokens", order: "DESC" }
     }`

3. 结果：透视表，行=用户，列包含各模型的 Token 消耗

### 场景 5：成本优化分析

**需求**：对比不同模型的成本效率（Cost per 1K Tokens）

**步骤**：
1. 创建 Widget "Cost Efficiency"
2. 配置：
   - ChartType: `VERTICAL_BAR`
   - View: `observations`
   - Dimensions: `[{ field: "model" }]`
   - Metrics: `[
       { measure: "calculatedTotalCost", agg: "sum" },
       { measure: "totalTokens", agg: "sum" }
     ]`

3. 前端计算：`costEfficiency = (sum_calculatedTotalCost / sum_totalTokens) * 1000`
4. 结果：柱状图对比各模型的 Cost per 1K Tokens

---

## 性能优化

### 1. 查询优化

**策略**：
- **时间范围限制**：建议最多查询 90 天数据
- **Limit 限制**：非时间序列查询默认 limit 1000
- **索引利用**：
  - ClickHouse 按 `project_id` 和 `timestamp` 分区
  - 过滤条件优先使用索引字段

**示例**：
```typescript
// ✅ Good: 明确的时间范围
const query = {
  fromTimestamp: "2024-01-01T00:00:00Z",
  toTimestamp: "2024-01-31T23:59:59Z",
  // ...
};

// ❌ Bad: 无时间限制
const query = {
  fromTimestamp: "1970-01-01T00:00:00Z",
  toTimestamp: new Date().toISOString(),
};
```

### 2. 前端优化

**策略**：
- **skipBatch: true**：大查询跳过批处理，避免阻塞
- **debounce**：用户调整过滤器时，延迟 300ms 再查询
- **virtualList**：大数据表格使用虚拟滚动（如 PivotTable）

**示例**：
```typescript
const queryResult = api.dashboard.executeQuery.useQuery(
  { projectId, query },
  {
    trpc: {
      context: { skipBatch: true }, // 独立执行
    },
    enabled: Boolean(projectId),
  }
);
```

### 3. 布局优化

**响应式断点**：
```typescript
// 小屏幕禁用拖拽
const isSmallScreen = useMediaQuery("(max-width: 1024px)");
const layout = widgets.map((w) => ({
  ...w,
  isDraggable: canEdit && !isSmallScreen,
}));
```

**动态行高**：
```typescript
const handleWidthChange = (containerWidth: number) => {
  const calculatedRowHeight = ((containerWidth / 12) * 9) / 16;
  setRowHeight(calculatedRowHeight);
};
```

---

## 常见问题（FAQ）

### 1. Dashboard 和 Widget 的关系？

**Dashboard** 是容器，**Widget** 是数据单元：
- 一个 Dashboard 包含多个 WidgetPlacement
- 一个 Widget 可以被多个 Dashboard 复用
- Dashboard 只存储 Widget 的位置和尺寸，不存储查询配置

### 2. 为什么有 Langfuse Dashboard 和 Project Dashboard？

- **Langfuse Dashboard**：系统预置的通用仪表板，所有项目可见，提供开箱即用的分析视图
- **Project Dashboard**：项目自定义，满足特定业务需求，支持完全自定义

### 3. 如何理解 View、Dimension 和 Metric？

- **View**：数据源（如 `traces`, `observations`），对应 ClickHouse 中的表或视图
- **Dimension**：分组维度（如 `model`, `userId`），对应 SQL 的 `GROUP BY`
- **Metric**：聚合指标（如 `sum(totalTokens)`），对应 SQL 的 `SUM()`, `AVG()` 等

### 4. executeQuery 和 chart API 有什么区别？

- **executeQuery**：新的统一查询接口，支持任意 View/Dimension/Metric 组合，推荐使用
- **chart**：遗留 API，用于特定图表查询（如 `score-aggregate`），逐步迁移中

### 5. 全局过滤器和 Widget 过滤器如何合并？

**合并规则**：`Widget Filters + Global Filters`

```typescript
const finalFilters = [
  ...widget.filters,  // Widget 静态过滤
  ...mapLegacyUiTableFilterToView(view, globalFilterState),  // 全局过滤
];
```

**优先级**：如果两者有冲突，都会应用（AND 逻辑）

### 6. 为什么时间维度字段固定为 `time_dimension`？

**统一化处理**：
- ClickHouse 查询结果中，时间分桶字段统一命名为 `time_dimension`
- 避免前端需要根据不同 View 处理不同字段名（如 `timestamp`, `startTime`）

### 7. Pivot Table 的排序是如何实现的？

**前端状态管理**：
```typescript
const [sortState, setSortState] = useState<OrderByState | null>(defaultSort);

// 排序状态传递给查询
const query = {
  orderBy: sortState ? [
    { field: sortState.column, direction: sortState.order.toLowerCase() }
  ] : null,
};
```

**后端应用**：在 ClickHouse 查询的 `ORDER BY` 子句中应用

### 8. 如何处理大数据量的图表？

**限制策略**：
- **row_limit**：ChartConfig 中设置 `row_limit: 1000`
- **时间聚合**：使用 `timeDimension.granularity = "day"` 减少数据点
- **Top N**：使用 `LIMIT` + `ORDER BY` 只展示 Top 20/50

### 9. Dashboard 布局如何持久化？

**存储在 `definition` JSON 字段**：
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

**更新时机**：
- 拖拽 Widget 后，`onLayoutChange` 回调触发
- 调用 `updateDashboardDefinition` mutation 保存

### 10. 如何添加新的图表类型？

**步骤**：
1. 在 `DashboardWidgetChartType` enum 中添加类型
2. 创建 ChartConfig Schema（如 `NewChartConfig`）
3. 在 `ChartConfigSchema` 中添加 discriminated union
4. 实现前端图表组件
5. 在 `DashboardWidget` 中添加渲染逻辑

---

## 相关模块

- **Traces & Observations**：Dashboard 主要数据源
- **Scores**：Score 分析图表的数据来源
- **Ingestion**：数据写入 ClickHouse，供 Dashboard 查询
- **Query Builder**：将 QueryType 转换为 ClickHouse SQL

---

## 目录结构

```
web/src/features/dashboard/
├── components/
│   ├── BaseTimeSeriesChart.tsx       # 时间序列图表基类
│   ├── TracesTimeSeriesChart.tsx     # Traces 时间序列
│   ├── ModelUsageChart.tsx           # 模型使用量图表
│   ├── LatencyChart.tsx              # Latency 图表
│   ├── ChartScores.tsx               # Score 图表
│   ├── UserChart.tsx                 # 用户分析图表
│   ├── TracesBarListChart.tsx        # Top N 排名
│   ├── ModelCostTable.tsx            # 成本表格
│   ├── LatencyTables.tsx             # Latency 表格
│   ├── ScoresTable.tsx               # Score 表格
│   ├── DashboardTable.tsx            # Dashboard 列表
│   ├── EditDashboardDialog.tsx       # 编辑对话框
│   ├── SelectDashboardDialog.tsx     # 选择对话框
│   ├── ModelSelector.tsx             # 模型选择器
│   ├── TotalMetric.tsx               # 总计指标
│   ├── TabsComponent.tsx             # Tab 组件
│   ├── Tooltip.tsx                   # 自定义 Tooltip
│   ├── hooks.ts                      # 自定义 Hooks
│   ├── cards/
│   │   ├── DashboardCard.tsx         # Widget 卡片容器
│   │   ├── DashboardTable.tsx        # 表格卡片
│   │   └── ChevronButton.tsx         # 展开按钮
│   └── score-analytics/              # Score 分析组件
│       ├── ScoreAnalytics.tsx
│       ├── NumericScoreTimeSeriesChart.tsx
│       ├── CategoricalScoreChart.tsx
│       └── NumericScoreHistogram.tsx
├── lib/
│   ├── dashboard-utils.ts            # 工具函数
│   └── score-analytics-utils.ts      # Score 分析工具
├── server/
│   └── dashboard-router.ts           # tRPC Router
└── utils/
    └── getColorsForCategories.ts     # 颜色工具

web/src/features/widgets/
├── components/
│   ├── DashboardGrid.tsx             # 网格布局容器
│   ├── DashboardWidget.tsx           # Widget 查询和渲染
│   ├── SelectWidgetDialog.tsx        # Widget 选择对话框
│   ├── WidgetForm.tsx                # Widget 创建/编辑表单
│   └── WidgetTable.tsx               # Widget 列表
├── chart-library/
│   └── PivotTable.tsx                # 透视表组件
└── utils.ts                          # 工具函数

web/src/server/api/routers/
└── dashboardWidgets.ts               # Widget tRPC Router

packages/shared/src/server/services/DashboardService/
├── DashboardService.ts               # 核心服务类
└── types.ts                          # 类型定义

packages/shared/src/server/queries/
├── index.ts                          # 查询工具导出
├── clickhouse-sql/
│   ├── factory.ts                    # 过滤器工厂
│   ├── event-query-builder.ts        # 查询构建器
│   ├── clickhouse-filter.ts          # 过滤器类
│   └── orderby-factory.ts            # 排序工厂
└── types.ts                          # 查询类型定义
```

---

**文档版本**: 1.0  
**最后更新**: 2024-12-17  
**模块状态**: ✅ 已完成
