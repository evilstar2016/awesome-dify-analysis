# 前端展示层 (Frontend/Presentation Layer)

## 1. 层职责

前端展示层是 Langfuse 的用户界面层，负责：
- 渲染用户界面（React 组件）
- 处理用户交互和表单输入
- 管理客户端状态
- 调用 tRPC API 获取和提交数据
- 实现页面路由和导航
- 提供响应式和交互式的用户体验

**架构定位**：作为最顶层，直接面向用户，不包含业务逻辑，只负责展示和用户交互。

---

## 2. 主要组件

### 2.1 页面组件（Pages）

#### 路径
`web/src/pages/`

#### 主要页面模块

| 页面路径 | 文件 | 功能描述 |
|---------|------|---------|
| `/` | `index.tsx` | 首页/登录页 |
| `/project/[projectId]/traces` | `project/[projectId]/traces/` | Traces 追踪列表和详情 |
| `/project/[projectId]/prompts` | `project/[projectId]/prompts/` | 提示词管理 |
| `/project/[projectId]/datasets` | `project/[projectId]/datasets/` | 数据集管理 |
| `/project/[projectId]/evals` | `project/[projectId]/evals/` | 评估配置和结果 |
| `/project/[projectId]/playground` | `project/[projectId]/playground/` | LLM Playground |
| `/project/[projectId]/dashboard` | `project/[projectId]/dashboard/` | 项目仪表盘 |
| `/project/[projectId]/settings` | `project/[projectId]/settings/` | 项目设置 |
| `/organization` | `organization/` | 组织管理 |

#### 页面结构特点
- **动态路由**：使用 Next.js 的文件系统路由
- **SSR/SSG**：支持服务器端渲染和静态生成
- **布局嵌套**：使用 Next.js 布局组件
- **SEO 优化**：Meta 标签和 Open Graph

### 2.2 React 组件（Components）

#### 路径
`web/src/components/`

#### 组件分类

##### UI 基础组件
位于 `web/src/components/ui/`（基于 shadcn/ui）

| 组件 | 文件 | 用途 |
|-----|------|------|
| `Button` | `button.tsx` | 按钮组件 |
| `Input` | `input.tsx` | 输入框 |
| `Select` | `select.tsx` | 下拉选择 |
| `Dialog` | `dialog.tsx` | 对话框 |
| `Table` | `table.tsx` | 表格 |
| `Card` | `card.tsx` | 卡片容器 |
| `Badge` | `badge.tsx` | 徽章标签 |
| `Tabs` | `tabs.tsx` | 标签页 |

##### 业务组件
| 组件类型 | 路径 | 示例 |
|---------|------|------|
| **表格组件** | `components/table/` | DataTable, DataTableToolbar |
| **表单组件** | `components/forms/` | 各种表单输入和验证 |
| **图表组件** | `components/charts/` | 基于 Recharts 的可视化 |
| **布局组件** | `components/layouts/` | Header, Sidebar, Layout |
| **Trace 相关** | `components/trace/` | TraceAggUsageBadge, IOPreview |
| **Prompt 相关** | `components/prompts/` | PromptHistoryNode |

##### 特殊组件
| 组件 | 路径 | 功能 |
|-----|------|------|
| **CodeEditor** | `components/CodeEditor.tsx` | 基于 CodeMirror 的代码编辑器 |
| **JSONView** | `components/JSONView.tsx` | JSON 数据展示 |
| **MarkdownRenderer** | `components/MarkdownRenderer.tsx` | Markdown 渲染 |
| **CommandBar** | `components/CommandBar.tsx` | 命令面板（cmdk） |

### 2.3 客户端状态管理

#### React Query (TanStack Query)
- **路径**：与 tRPC 集成
- **用途**：
  - 服务器状态缓存
  - 自动重新获取
  - 乐观更新
  - 分页和无限滚动

#### 客户端 Hooks
- **路径**：`web/src/hooks/`
- **主要 Hooks**：
  - `useLocalStorage` - 本地存储
  - `useDebounce` - 防抖
  - `useClickhouse` - ClickHouse 查询
  - `useQueryParams` - URL 参数管理
  - 各种业务 Hooks

### 2.4 路由和导航

#### Next.js Pages Router
- **类型**：文件系统路由
- **动态路由**：`[projectId]`, `[traceId]` 等
- **API 路由**：`pages/api/` 目录

#### 导航组件
- **Header**：顶部导航栏
- **Sidebar**：侧边栏菜单
- **Breadcrumbs**：面包屑导航
- **ProjectNavigation**：项目级导航

### 2.5 样式系统

#### Tailwind CSS
- **配置**：`tailwind.config.ts`
- **主题**：支持亮色/暗色模式
- **工具类**：完整的 Tailwind 工具类集合

#### CSS Modules
- **使用场景**：特殊样式需求
- **命名规范**：`.module.css`

#### 全局样式
- **路径**：`web/src/styles/`
- **文件**：`globals.css`, `markdown.css` 等

---

## 3. 对外接口

### 3.1 HTTP 端点（浏览器访问）

| 路由 | 类型 | 说明 |
|-----|------|------|
| `/` | SSR | 首页/登录 |
| `/project/[projectId]/*` | SSR/CSR | 项目页面 |
| `/auth/*` | SSR | 认证页面 |
| `/onboarding` | SSR | 新用户引导 |

### 3.2 客户端 API 调用（tRPC）

前端通过 tRPC 客户端调用后端 API：

```typescript
// 示例：调用 traces API
const { data } = trpc.traces.all.useQuery({
  projectId: "project-123",
  page: 1,
  limit: 50,
});

// 示例：创建 prompt
const createPrompt = trpc.prompts.create.useMutation({
  onSuccess: () => {
    // 成功回调
  },
});
```

**特点**：
- 端到端类型安全
- 自动类型推导
- 内置缓存和重新验证
- 乐观更新支持

---

## 4. 与其他层的交互

### 4.1 下层依赖（tRPC API 层）

**交互方式**：
```
前端组件
  ↓ (tRPC Client)
tRPC API Router
  ↓ (Procedure Call)
业务逻辑层
```

**数据流**：
1. 用户触发操作（点击、提交表单）
2. React 组件调用 tRPC Hook
3. tRPC 客户端发送 HTTP 请求
4. 服务器处理并返回数据
5. React Query 缓存并更新组件状态
6. 组件重新渲染

### 4.2 上层交互（用户）

**用户交互方式**：
- 鼠标点击、键盘输入
- 表单提交
- 拖拽操作（Dashboard 部件）
- 实时搜索和过滤

### 4.3 跨层依赖（共享层）

**使用的共享资源**：
- **类型定义**：`@langfuse/shared` 的类型
- **常量**：`web/src/constants/`
- **工具函数**：`web/src/utils/`

---

## 5. 关键代码文件路径

### 5.1 核心配置文件

| 文件 | 路径 | 作用 |
|-----|------|------|
| **Next.js 配置** | `web/next.config.mjs` | Next.js 构建配置 |
| **Tailwind 配置** | `web/tailwind.config.ts` | Tailwind CSS 配置 |
| **TypeScript 配置** | `web/tsconfig.json` | TypeScript 编译配置 |
| **PostCSS 配置** | `web/postcss.config.cjs` | CSS 处理配置 |
| **组件配置** | `web/components.json` | shadcn/ui 组件配置 |

### 5.2 入口文件

| 文件 | 路径 | 作用 |
|-----|------|------|
| **App 入口** | `web/src/pages/_app.tsx` | Next.js App 组件 |
| **Document** | `web/src/pages/_document.tsx` | HTML Document 配置 |
| **tRPC 客户端** | `web/src/utils/api.ts` | tRPC 客户端设置 |

### 5.3 主要页面目录

```
web/src/pages/
├── index.tsx                    # 首页
├── auth/                        # 认证相关页面
├── onboarding.tsx               # 新用户引导
├── project/
│   └── [projectId]/
│       ├── traces/              # Traces 页面
│       ├── prompts/             # Prompts 页面
│       ├── datasets/            # Datasets 页面
│       ├── evals/               # Evaluations 页面
│       ├── playground/          # Playground 页面
│       ├── dashboard/           # Dashboard 页面
│       └── settings/            # 设置页面
└── organization/                # 组织管理
```

### 5.4 主要组件目录

```
web/src/components/
├── ui/                          # 基础 UI 组件（shadcn/ui）
├── layouts/                     # 布局组件
├── table/                       # 表格相关
├── charts/                      # 图表组件
├── trace/                       # Trace 相关组件
├── prompts/                     # Prompt 相关组件
└── forms/                       # 表单组件
```

---

## 6. 技术实现细节

### 6.1 前端技术栈

| 技术 | 版本 | 用途 |
|-----|------|------|
| **React** | 19.2.3 | UI 框架 |
| **Next.js** | 15.5.9 | React 框架 |
| **TypeScript** | 5.7.2 | 类型安全 |
| **Tailwind CSS** | 3.4.17 | CSS 框架 |
| **shadcn/ui** | - | UI 组件库 |
| **Radix UI** | - | 无样式组件库 |
| **tRPC** | 11.4.4 | 类型安全 API |
| **React Query** | 5.85.1 | 服务器状态管理 |
| **Zod** | 3.25.62 | 运行时验证 |

### 6.2 状态管理策略

#### 服务器状态
- **工具**：React Query (TanStack Query)
- **策略**：
  - 自动缓存和重新验证
  - Stale-while-revalidate 模式
  - 乐观更新

#### 客户端状态
- **工具**：React Hooks (useState, useReducer)
- **策略**：
  - 组件本地状态
  - Context API（少量全局状态）
  - URL 状态（查询参数）

#### 表单状态
- **工具**：react-hook-form
- **验证**：Zod schema

### 6.3 性能优化

#### 代码分割
```typescript
// 动态导入
const CodeEditor = dynamic(() => import("@/components/CodeEditor"), {
  ssr: false,
  loading: () => <Skeleton />,
});
```

#### 虚拟滚动
- **工具**：@tanstack/react-virtual
- **应用**：大型列表和表格

#### 图片优化
- **工具**：next/image
- **特性**：自动优化、懒加载

#### 预加载
- **Prefetching**：tRPC 数据预取
- **Link Prefetching**：Next.js Link 组件

### 6.4 渲染模式

#### SSR（服务器端渲染）
- 首屏渲染快速
- SEO 友好
- 用于公开页面

#### CSR（客户端渲染）
- 交互性强
- 适合认证后页面
- 使用 tRPC 获取数据

#### 混合模式
- 页面框架 SSR
- 数据 CSR
- 最佳性能和用户体验

### 6.5 主题系统

#### 实现方式
```typescript
// 使用 next-themes
import { ThemeProvider } from "next-themes";

// 支持的主题
themes: ["light", "dark", "system"]
```

#### CSS 变量
```css
/* Tailwind CSS 变量 */
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  /* ... */
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  /* ... */
}
```

### 6.6 国际化（i18n）

**当前状态**：主要为英文，部分组件支持多语言

**实现方式**：
- 使用 Next.js i18n 路由（如需要）
- 内联文本（目前）

### 6.7 错误处理

#### 错误边界
```typescript
// React Error Boundary
class ErrorBoundary extends React.Component {
  // 捕获组件树错误
}
```

#### 全局错误处理
- **工具**：Sentry
- **集成**：`@sentry/nextjs`

#### API 错误处理
```typescript
// tRPC 错误处理
const { error, isError } = trpc.traces.all.useQuery();

if (isError) {
  toast.error(error.message);
}
```

### 6.8 表单处理

#### 表单库
```typescript
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";

const form = useForm({
  resolver: zodResolver(formSchema),
});
```

#### 验证策略
- **客户端**：Zod schema 实时验证
- **服务器**：tRPC 服务器端验证
- **双重验证**：确保数据安全

---

## 7. 设计模式和最佳实践

### 7.1 组件设计模式

#### 组合模式
```typescript
// 示例：Card 组件
<Card>
  <CardHeader>
    <CardTitle>Title</CardTitle>
  </CardHeader>
  <CardContent>Content</CardContent>
</Card>
```

#### Render Props
```typescript
// 示例：DataTable
<DataTable
  renderRow={(row) => <CustomRow data={row} />}
/>
```

#### Custom Hooks
```typescript
// 封装逻辑
function useTraces(projectId: string) {
  return trpc.traces.all.useQuery({ projectId });
}
```

### 7.2 代码组织

#### 按功能组织
```
features/
├── traces/
│   ├── components/
│   ├── hooks/
│   └── utils/
└── prompts/
    ├── components/
    ├── hooks/
    └── utils/
```

#### 组件共置
```
TraceView/
├── index.tsx
├── TraceView.tsx
├── TraceViewHeader.tsx
├── TraceViewTable.tsx
└── useTraceView.ts
```

### 7.3 类型安全

#### tRPC 类型推导
```typescript
// 自动类型推导
const { data } = trpc.traces.all.useQuery();
// data 的类型自动推导
```

#### Zod Schema
```typescript
const traceSchema = z.object({
  id: z.string(),
  name: z.string(),
  timestamp: z.date(),
});
```

### 7.4 性能监控

- **工具**：React DevTools, Lighthouse
- **指标**：LCP, FID, CLS
- **优化**：代码分割、懒加载、虚拟滚动

---

## 8. 与后端的契约

### 8.1 API 契约（tRPC）

**类型定义**：
```typescript
// 服务器定义
export const tracesRouter = router({
  all: publicProcedure
    .input(z.object({ projectId: z.string() }))
    .query(async ({ input }) => {
      // 实现
    }),
});

// 客户端自动获得类型
const traces = trpc.traces.all.useQuery({ projectId });
// traces 类型自动推导
```

**优势**：
- 端到端类型安全
- 重构友好
- 自动补全

### 8.2 数据格式

- **请求格式**：JSON
- **响应格式**：JSON
- **日期格式**：ISO 8601
- **分页**：Cursor-based 或 Offset-based

---

## 9. 关键特性

### 9.1 实时更新
- **轮询**：React Query 自动重新获取
- **Websocket**：（如需要）实时推送

### 9.2 搜索和过滤
- **客户端过滤**：小数据集
- **服务器端过滤**：大数据集
- **Debounce**：优化搜索性能

### 9.3 数据可视化
- **图表库**：Recharts
- **表格**：TanStack Table
- **树形视图**：@mui/x-tree-view

### 9.4 交互体验
- **加载状态**：Skeleton, Spinner
- **空状态**：Empty state 组件
- **错误状态**：Error boundary
- **成功提示**：Toast notifications (sonner)

---

## 10. 开发工作流

### 10.1 本地开发
```bash
# 启动开发服务器
pnpm --filter=web run dev

# 访问
http://localhost:3000
```

### 10.2 热重载
- **Fast Refresh**：React 组件热重载
- **Turbopack**：Next.js 15 使用 Turbopack（可选）

### 10.3 调试工具
- **React DevTools**：组件树和状态
- **tRPC Panel**：API 调试
- **Browser DevTools**：网络、性能

---

## 11. 总结

前端展示层是 Langfuse 用户体验的核心，采用现代化的 React + Next.js 技术栈，通过 tRPC 实现类型安全的全栈开发体验。层次分明、组件化设计、性能优化和良好的开发体验是该层的主要特点。

**关键优势**：
- ✅ 端到端类型安全
- ✅ 优秀的开发体验
- ✅ 丰富的 UI 组件库
- ✅ 高性能渲染
- ✅ 响应式设计

---

**文档编写时间**：2025-12-17  
**项目版本**：3.140.0
