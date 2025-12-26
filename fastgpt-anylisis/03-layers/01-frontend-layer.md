# 前端展示层 (Frontend Layer)

## 概述
前端展示层是 FastGPT 面向用户的交互界面，基于 Next.js 14 + React 18 + Chakra UI 构建，提供应用编排、对话交互、知识库管理等核心功能的可视化操作。

## 技术栈
- **框架**: Next.js 14 (App Router + Pages Router 混合模式)
- **UI 库**: Chakra UI 2.x (主题系统、响应式组件)
- **状态管理**: 
  - React Query (@tanstack/react-query) - 服务端状态管理
  - Zustand - 客户端状态管理
  - use-context-selector - 性能优化的 Context
- **工作流编排**: React Flow 11.x (可视化 Flow 编辑器)
- **国际化**: i18next + next-i18next + react-i18next
- **表单管理**: react-hook-form 7.x
- **其他**: 
  - Framer Motion (动画)
  - Markdown 渲染 (react-markdown + rehype/remark 插件)
  - Monaco Editor (代码编辑)
  - Lexical (富文本编辑)
  - Echarts (图表可视化)

## 核心模块

### 1. 主应用入口 (`projects/app`)

#### 目录结构
```
projects/app/src/
├── pages/              # Next.js 页面路由
│   ├── _app.tsx       # 应用入口，全局 Provider
│   ├── _document.tsx  # HTML 文档结构
│   ├── index.tsx      # 首页
│   ├── account/       # 账户管理页面
│   ├── app/           # 应用管理页面
│   ├── chat/          # 对话页面
│   ├── dataset/       # 知识库管理页面
│   ├── login/         # 登录页面
│   └── api/           # API 路由 (见 API 层文档)
├── components/        # 页面级组件
│   ├── Layout/        # 布局组件 (头部、侧边栏)
│   ├── core/          # 核心业务组件
│   ├── common/        # 通用组件
│   └── support/       # 支持类组件
├── pageComponents/    # 页面专属组件集合
├── service/           # 前端服务层 (API 调用封装)
├── web/               # Web 相关工具 (hooks, utils)
├── global/            # 全局配置与常量
└── types/             # TypeScript 类型定义
```

#### 关键功能模块

**应用管理** (`pages/app/`)
- 应用列表、创建、编辑、删除
- 工作流可视化编排 (React Flow)
- 版本管理与发布
- 应用配置 (模型、知识库、工具)
- 日志查看与调试

**对话交互** (`pages/chat/`)
- 实时对话界面
- 流式响应渲染
- 多模态输入 (文本、语音、图片)
- 对话历史管理
- 引用来源展示
- 反馈与评价

**知识库管理** (`pages/dataset/`)
- 知识库列表与 CRUD
- 文档上传 (多种格式)
- 文档分块查看与编辑
- 训练队列监控
- 测试检索功能

**账户与团队** (`pages/account/`)
- 用户信息管理
- 团队管理与权限
- API Key 管理
- 消费记录与计费
- 外链分享配置

### 2. 共享 UI 组件库 (`packages/web`)

#### 目录结构
```
packages/web/
├── components/        # 跨项目共享组件
│   ├── common/       # 通用 UI 组件
│   │   ├── Icon/     # 图标组件
│   │   ├── MyBox/    # 布局组件
│   │   ├── MyModal/  # 弹窗组件
│   │   ├── MyTooltip/# 提示组件
│   │   └── ...
│   └── core/         # 核心业务组件
│       ├── ai/       # AI 相关组件 (模型选择器)
│       ├── app/      # 应用相关组件
│       ├── chat/     # 对话组件
│       └── workflow/ # 工作流组件
├── hooks/            # 自定义 Hooks
│   ├── useRequest.ts # 请求封装
│   ├── useLoading.ts # 加载状态
│   └── ...
├── context/          # 全局 Context
│   ├── ChatContext.tsx
│   └── ...
├── store/            # Zustand 状态管理
│   ├── app.ts
│   ├── chat.ts
│   └── user.ts
├── i18n/             # 国际化配置
│   ├── en/
│   └── zh-CN/
├── styles/           # 主题与样式
│   ├── theme.ts      # Chakra UI 主题
│   └── global.scss
├── core/             # 核心业务逻辑
│   ├── app/
│   ├── chat/
│   └── dataset/
└── support/          # 支持工具
    ├── permission/   # 权限判断
    └── user/         # 用户相关
```

#### 关键组件

**通用组件** (`components/common/`)
- Icon: 图标库 (基于 @chakra-ui/icons 扩展)
- MyBox: 增强的 Box 组件 (带默认样式)
- MyModal: 弹窗组件 (支持拖拽、自定义头部)
- MyTooltip: 提示框 (增强版 Tooltip)
- MyInput: 输入框 (集成常见场景)
- Loading: 加载动画组件
- Empty: 空状态组件
- Avatar: 头像组件
- Tag: 标签组件
- PermissionTip: 权限提示组件

**工作流组件** (`components/core/workflow/`)
- Flow 编辑器: 基于 React Flow 的可视化编排
- 节点组件: AI 对话、知识库检索、工具调用等节点
- 变量选择器: 支持引用其他节点输出
- 边缘连接器: 节点间连线逻辑
- 调试面板: 变量查看、日志追踪

**对话组件** (`components/core/chat/`)
- ChatBox: 对话容器
- ChatItem: 单条消息组件
- QuoteModal: 引用来源弹窗
- InputGuide: 输入引导
- VoiceInput: 语音输入组件
- FileUpload: 文件上传组件

### 3. React Flow 编排器

#### 核心功能
- **节点拖拽**: 从侧边栏拖拽节点到画布
- **节点配置**: 点击节点打开配置面板
- **连线**: 节点间通过 Handle 连接
- **变量系统**: 支持引用其他节点的输出变量
- **布局**: 自动布局算法 (Dagre)
- **缩放平移**: 画布缩放与平移
- **快捷键**: 删除、复制、粘贴等操作
- **状态持久化**: 保存到服务端

#### 节点类型
- **系统节点**: 工作流开始、用户引导、AI 对话
- **功能节点**: 知识库检索、HTTP 请求、代码执行
- **工具节点**: Plugin 工具、MCP 工具、自定义工具
- **逻辑节点**: 条件判断、变量赋值、循环
- **交互节点**: 用户输入、文本输出

## 状态管理架构

### React Query (服务端状态)
- 应用列表、知识库列表等数据缓存
- 自动重新获取 (staleTime, refetchInterval)
- 乐观更新 (Optimistic Updates)
- 请求去重与缓存共享

### Zustand (客户端状态)
- 用户信息 (`useUserStore`)
- 应用编辑状态 (`useAppStore`)
- 对话状态 (`useChatStore`)
- 全局 UI 状态 (侧边栏展开、主题)

### Context (跨组件共享)
- ChatContext: 对话上下文 (历史记录、配置)
- PermissionContext: 权限上下文
- use-context-selector: 性能优化，避免不必要的重渲染

## 路由架构

### Pages Router (主要路由)
```
/                    # 首页
/login               # 登录页
/account             # 账户管理
/app/list            # 应用列表
/app/detail          # 应用详情 (工作流编排)
/chat/[chatId]       # 对话页面
/dataset/list        # 知识库列表
/dataset/detail      # 知识库详情
```

### 路由守卫
- 未登录跳转到 `/login`
- 权限校验 (Team 权限、功能权限)
- 页面级 Loading 状态

## 国际化 (i18n)

### 配置
- 默认语言: 中文 (zh-CN)
- 支持语言: 英文 (en)、日语 (ja)
- 命名空间: app, chat, dataset, user, common, workflow, plugin

### 使用方式
```typescript
// 页面级
export async function getServerSideProps(context) {
  return {
    props: {
      ...(await serverSideTranslations(context.locale, ['app', 'common']))
    }
  };
}

// 组件内
const { t } = useTranslation();
<Button>{t('common:save')}</Button>
```

### 翻译规范
- Key 格式: `namespace:key_name` (小写+下划线)
- 静态文件使用 `i18nT` 函数
- 动态插值使用 `{{variable}}`

## 样式系统

### Chakra UI 主题
- 色彩系统: 主色调 (#7d09f1)、语义色
- 组件变体: 自定义 Button、Input、Modal 等变体
- 响应式断点: sm, md, lg, xl, 2xl
- 暗黑模式: 支持 (useColorMode)

### 全局样式
- Normalize CSS
- 自定义滚动条样式
- Markdown 内容样式
- 代码高亮样式

## 性能优化

### 代码分割
- 动态导入: `next/dynamic`
- 路由级别代码分割 (Next.js 自动)
- 组件级懒加载 (React.lazy)

### 渲染优化
- React.memo: 避免不必要的重渲染
- useMemo / useCallback: 缓存计算结果与函数
- 虚拟滚动: 长列表渲染 (react-window)
- use-context-selector: Context 性能优化

### 资源优化
- Image 优化: next/image (自动 WebP、懒加载)
- 字体优化: 本地字体文件
- Icon 按需加载: 动态导入图标

## 关键交互流程

### 工作流编排流程
1. 用户进入应用详情页
2. 加载应用配置与工作流数据
3. 渲染 React Flow 画布
4. 用户拖拽/配置节点
5. 实时保存到本地状态
6. 点击保存按钮，调用 API 保存到服务端
7. 版本管理与发布

### 对话交互流程
1. 用户输入消息
2. 调用 `/api/v1/chat/completions` (流式接口)
3. 通过 SSE 接收流式响应
4. 实时渲染 AI 回复内容
5. 展示引用来源、工具调用信息
6. 保存对话记录到本地状态

### 知识库训练流程
1. 用户上传文档
2. 调用上传 API，文件存储到 MinIO/S3
3. 创建文档集合，触发训练任务
4. 轮询或 WebSocket 获取训练进度
5. 训练完成后，展示分块数据
6. 支持编辑、删除分块

## 待完善功能
- 离线模式支持
- PWA 改造
- 移动端适配优化
- 更多快捷键支持
- 工作流模板市场
- 实时协作编辑

## 相关文档
- [API 路由层](./02-api-routes-layer.md)
- [共享层](./04-shared-layer.md)
- [工作流引擎](./07-workflow-engine.md)
