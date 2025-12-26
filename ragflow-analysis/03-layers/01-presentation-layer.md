# 01 - 前端展示层 (Presentation Layer)

## 1. 层职责与定位

### 1.1 职责
前端展示层是 RAGFlow 的用户界面层，负责：
- **用户交互**: 处理用户输入和操作
- **界面渲染**: 展示数据和状态
- **状态管理**: 管理客户端应用状态
- **API 通信**: 与后端 API 服务器交互
- **数据可视化**: 展示图表、流程图等
- **路由管理**: 页面导航和权限控制

### 1.2 定位
- **架构位置**: 最上层，直接面向用户
- **技术栈**: React 18 + TypeScript + UmiJS
- **部署方式**: 静态资源，通过 Nginx 提供服务
- **访问入口**: Web 浏览器 (http://localhost:8000)

---

## 2. 主要组件/模块

### 2.1 目录结构

```
web/
├── src/
│   ├── pages/              # 页面组件（路由页面）
│   │   ├── agent/         # Agent 工作流编辑器
│   │   ├── agents/        # Agent 列表
│   │   ├── dataset/       # 数据集管理 (知识库)
│   │   ├── datasets/      # 数据集列表
│   │   ├── chunk/         # 分块管理
│   │   ├── files/         # 文件管理
│   │   ├── next-chats/    # 对话聊天
│   │   ├── next-search/   # 搜索功能
│   │   ├── next-searches/ # 搜索列表
│   │   ├── memories/      # 记忆管理
│   │   ├── memory/        # 单个记忆
│   │   ├── login-next/    # 登录页面
│   │   ├── user-setting/  # 用户设置
│   │   ├── document-viewer/ # 文档查看器
│   │   ├── admin/         # 管理后台
│   │   └── home/          # 主页
│   ├── components/        # 可复用组件
│   │   ├── chat/         # 聊天相关组件
│   │   ├── knowledge/    # 知识库相关组件
│   │   ├── flow-canvas/  # 流程画布组件
│   │   ├── document-viewer/ # 文档查看器
│   │   └── ...
│   ├── layouts/          # 布局组件
│   │   ├── BasicLayout/  # 基础布局（带侧边栏）
│   │   └── BlankLayout/  # 空白布局（登录页等）
│   ├── services/         # API 服务封装
│   │   ├── knowledge-service.ts  # 知识库服务
│   │   ├── next-chat-service.ts  # 聊天服务
│   │   ├── user-service.ts       # 用户服务
│   │   ├── agent-service.ts      # Agent服务
│   │   ├── file-manager-service.ts # 文件管理服务
│   │   ├── search-service.ts     # 搜索服务
│   │   ├── memory-service.ts     # 记忆服务
│   │   └── ...
│   ├── hooks/            # 自定义 Hooks
│   │   ├── useSetModalState.ts
│   │   ├── useFetchKnowledgeList.ts
│   │   ├── useNavigateWithFromState.ts
│   │   └── ...
│   ├── models/           # 数据模型和状态管理
│   │   └── (UmiJS DVA 模型)
│   ├── utils/            # 工具函数
│   │   ├── request.ts    # HTTP 请求封装
│   │   ├── commonUtil.ts
│   │   └── ...
│   ├── locales/          # 国际化
│   │   ├── en-US/
│   │   ├── zh-CN/
│   │   └── ...
│   ├── constants/        # 常量定义
│   ├── interfaces/       # TypeScript 接口
│   └── assets/           # 静态资源
├── public/               # 公共静态文件
├── .umirc.ts            # UmiJS 配置
├── tailwind.config.js   # Tailwind CSS 配置
├── tsconfig.json        # TypeScript 配置
└── package.json         # 依赖管理
```

### 2.2 核心页面组件

#### 2.2.1 数据集/知识库管理 (`pages/dataset/` 和 `pages/datasets/`)
- **功能**: 知识库列表、创建、编辑、删除
- **核心文件**: `pages/dataset/index.tsx` (单个数据集), `pages/datasets/index.tsx` (列表)
- **关键功能**:
  - 知识库列表展示（卡片视图、列表视图）
  - 知识库创建向导
  - 文档上传和管理
  - 分块策略配置
  - 索引状态监控

#### 2.2.2 对话聊天 (`pages/next-chats/`)
- **功能**: 与知识库进行问答对话
- **核心文件**: `pages/next-chats/index.tsx`
- **关键功能**:
  - 多轮对话交互
  - 流式消息展示
  - Markdown 渲染
  - 引用来源展示
  - 历史对话记录
  - 对话会话管理

#### 2.2.3 Agent 工作流编辑器 (`pages/agent/`)
- **功能**: 可视化设计 Agent 工作流
- **核心文件**: `pages/agent/index.tsx`
- **技术**: XYFlow React (流程图库)
- **关键功能**:
  - 节点拖拽和连接
  - 节点配置面板
  - 流程图保存和加载
  - 工作流执行和调试
  - 模板选择

#### 2.2.4 文件管理 (`pages/files/`)
- **功能**: 文件上传、预览、管理
- **关键功能**:
  - 批量文件上传
  - 文档格式预览
  - 分块结果查看
  - 文档编辑和删除

#### 2.2.5 用户管理 (`pages/user-setting/`)
- **功能**: 用户信息、系统设置
- **关键功能**:
  - 个人信息编辑
  - LLM 模型配置
  - API Key 管理
  - 系统偏好设置

### 2.3 可复用组件库

#### 2.3.1 UI 基础组件
基于以下 UI 库构建：
- **Ant Design 5.x**: 主要 UI 组件库
  - Form, Table, Modal, Button, Input 等
  - Pro Components (高级组件)
- **Radix UI**: 无样式组件库
  - Dialog, Dropdown, Tooltip 等
  - 提供无障碍支持
- **shadcn/ui 风格**: 自定义样式组件

#### 2.3.2 业务组件
- **ChatBox**: 聊天消息框组件
- **DocumentViewer**: 文档查看器
- **ChunkList**: 分块列表展示
- **FlowCanvas**: 流程画布组件
- **MarkdownContent**: Markdown 渲染器
- **FileUploader**: 文件上传组件

### 2.4 服务层 (Services)

#### API 服务封装 (`services/`)
```typescript
// knowledgeService.ts
export const knowledgeService = {
  // 获取知识库列表
  async getKnowledgeList(params) {
    return request('/api/v1/kb/list', { method: 'GET', params });
  },
  
  // 创建知识库
  async createKnowledge(data) {
    return request('/api/v1/kb/create', { method: 'POST', data });
  },
  
  // 删除知识库
  async deleteKnowledge(kbId) {
    return request(`/api/v1/kb/${kbId}`, { method: 'DELETE' });
  }
};

// chatService.ts
export const chatService = {
  // 发送消息（流式）
  async sendMessage(data) {
    return requestStream('/api/v1/dialog/completion', {
      method: 'POST',
      data
    });
  },
  
  // 获取对话历史
  async getConversationHistory(conversationId) {
    return request(`/api/v1/conversation/${conversationId}/messages`);
  }
};
```

### 2.5 状态管理

#### 2.5.1 React Query (服务端状态)
```typescript
// 使用 React Query 管理服务端数据
import { useQuery, useMutation } from '@tanstack/react-query';

// 查询知识库列表
const { data, isLoading } = useQuery({
  queryKey: ['knowledgeList'],
  queryFn: knowledgeService.getKnowledgeList
});

// 创建知识库 Mutation
const createMutation = useMutation({
  mutationFn: knowledgeService.createKnowledge,
  onSuccess: () => {
    queryClient.invalidateQueries(['knowledgeList']);
  }
});
```

#### 2.5.2 Zustand (客户端状态)
```typescript
// 轻量级状态管理
import create from 'zustand';

interface AppState {
  user: User | null;
  theme: 'light' | 'dark';
  setUser: (user: User) => void;
  setTheme: (theme: 'light' | 'dark') => void;
}

export const useAppStore = create<AppState>((set) => ({
  user: null,
  theme: 'light',
  setUser: (user) => set({ user }),
  setTheme: (theme) => set({ theme })
}));
```

### 2.6 自定义 Hooks

#### 常用 Hooks
```typescript
// hooks/useFetchKnowledgeList.ts
export const useFetchKnowledgeList = () => {
  return useQuery({
    queryKey: ['knowledgeList'],
    queryFn: async () => {
      const res = await knowledgeService.getKnowledgeList();
      return res.data;
    }
  });
};

// hooks/useSetModalState.ts
export const useSetModalState = () => {
  const [visible, setVisible] = useState(false);
  const showModal = () => setVisible(true);
  const hideModal = () => setVisible(false);
  return { visible, showModal, hideModal };
};

// hooks/useNavigateWithFromState.ts
export const useNavigateWithFromState = () => {
  const navigate = useNavigate();
  return (path: string, state?: any) => {
    navigate(path, { state: { from: location.pathname, ...state } });
  };
};
```

---

## 3. 对外提供的接口或服务

### 3.1 用户交互界面

#### 3.1.1 主要页面路由
| 路由路径 | 页面 | 说明 |
|---------|------|------|
| `/` | 首页 | 重定向到知识库列表 |
| `/knowledge` | 知识库列表 | 展示所有知识库 |
| `/knowledge/:id` | 知识库详情 | 查看和管理单个知识库 |
| `/chat` | 聊天页面 | 与知识库对话 |
| `/flow` | Agent 工作流 | 设计和执行工作流 |
| `/file-manager` | 文件管理 | 文档上传和管理 |
| `/setting/profile` | 用户设置 | 个人信息和偏好 |
| `/setting/model` | 模型配置 | LLM 模型设置 |
| `/login` | 登录页面 | 用户登录 |

#### 3.1.2 组件导出
```typescript
// 可被其他模块引用的组件
export { ChatBox } from '@/components/chat/ChatBox';
export { DocumentViewer } from '@/components/document-viewer';
export { FlowCanvas } from '@/components/flow-canvas';
export { MarkdownContent } from '@/components/markdown-content';
```

### 3.2 静态资源
- **路径**: `/public/`
- **内容**: 图标、字体、配置文件
- **访问**: 通过 Nginx 提供静态文件服务

---

## 4. 与其他层的交互方式

### 4.1 与 API 服务层交互

#### 4.1.1 HTTP 请求
```typescript
// utils/request.ts - 统一请求封装
import axios from 'axios';

const request = axios.create({
  baseURL: process.env.UMI_APP_API_BASE_URL || 'http://localhost:9380',
  timeout: 30000,
  withCredentials: true, // 携带 Cookie
});

// 请求拦截器
request.interceptors.request.use((config) => {
  // 添加认证 Token
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 响应拦截器
request.interceptors.response.use(
  (response) => response.data,
  (error) => {
    // 统一错误处理
    if (error.response?.status === 401) {
      // 跳转登录
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

#### 4.1.2 SSE 流式请求
```typescript
// 处理流式响应
export const requestStream = (url: string, options: RequestOptions) => {
  return new Promise((resolve, reject) => {
    const eventSource = new EventSource(`${baseURL}${url}`);
    
    let result = '';
    
    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      result += data.chunk;
      options.onMessage?.(data);
    };
    
    eventSource.onerror = (error) => {
      eventSource.close();
      reject(error);
    };
    
    eventSource.addEventListener('done', () => {
      eventSource.close();
      resolve(result);
    });
  });
};
```

#### 4.1.3 WebSocket 连接（可选）
```typescript
// 实时通知
const ws = new WebSocket('ws://localhost:9380/ws');

ws.onmessage = (event) => {
  const notification = JSON.parse(event.data);
  // 显示通知
  message.info(notification.message);
};
```

### 4.2 数据流向

```
用户操作
  ↓
UI 组件事件
  ↓
调用 Service 方法
  ↓
HTTP/SSE 请求
  ↓ (通过 Nginx)
API 服务器
  ↓
响应数据
  ↓
更新 React Query 缓存
  ↓
组件重新渲染
  ↓
界面更新
```

### 4.3 错误处理
```typescript
// 统一错误处理
const handleError = (error: any) => {
  if (error.response) {
    // API 错误
    const { status, data } = error.response;
    switch (status) {
      case 400:
        message.error(data.message || '请求参数错误');
        break;
      case 401:
        message.error('请先登录');
        navigate('/login');
        break;
      case 403:
        message.error('没有权限');
        break;
      case 500:
        message.error('服务器错误');
        break;
    }
  } else if (error.request) {
    // 网络错误
    message.error('网络连接失败');
  } else {
    message.error(error.message);
  }
};
```

---

## 5. 关键代码文件路径

### 5.1 核心配置文件
| 文件 | 路径 | 说明 |
|------|------|------|
| UmiJS 配置 | `web/.umirc.ts` | 应用框架配置 |
| TypeScript 配置 | `web/tsconfig.json` | TypeScript 编译配置 |
| Tailwind 配置 | `web/tailwind.config.js` | 样式配置 |
| 包管理 | `web/package.json` | 依赖和脚本 |
| 环境变量 | `web/.env` | 环境配置 |
| ESLint 配置 | `web/.eslintrc.js` | 代码规范 |
| Prettier 配置 | `web/.prettierrc` | 格式化配置 |

### 5.2 核心业务文件
| 功能 | 路径 | 说明 |
|------|------|------|
| 知识库页面 | `web/src/pages/knowledge/` | 知识库管理 |
| 对话页面 | `web/src/pages/chat/` | 聊天界面 |
| 工作流页面 | `web/src/pages/flow/` | Agent 编辑器 |
| API 服务 | `web/src/services/` | API 封装 |
| 自定义 Hooks | `web/src/hooks/` | 可复用逻辑 |
| 工具函数 | `web/src/utils/` | 辅助函数 |
| 国际化 | `web/src/locales/` | 多语言 |

### 5.3 组件文件
| 组件 | 路径 | 说明 |
|------|------|------|
| 聊天组件 | `web/src/components/chat/` | 对话相关组件 |
| 文档查看器 | `web/src/components/document-viewer/` | 文档预览 |
| 流程画布 | `web/src/components/flow-canvas/` | 工作流编辑 |
| Markdown 渲染 | `web/src/components/markdown-content/` | Markdown 显示 |

---

## 6. 技术实现细节

### 6.1 技术栈

#### 6.1.1 核心框架
- **React 18**: UI 框架（函数式组件 + Hooks）
- **TypeScript**: 类型安全
- **UmiJS**: React 应用框架
  - 约定式路由
  - 插件体系
  - 构建优化

#### 6.1.2 UI 组件库
- **Ant Design 5.x**: 企业级 UI 库
- **Radix UI**: 无样式组件（高可访问性）
- **Tailwind CSS**: 原子化 CSS 框架
- **@ant-design/icons**: 图标库

#### 6.1.3 状态管理
- **React Query**: 服务端状态管理
  - 自动缓存
  - 自动重新请求
  - 乐观更新
- **Zustand**: 轻量级客户端状态
- **Immer**: 不可变数据

#### 6.1.4 数据可视化
- **AntV G2**: 图表库
- **AntV G6**: 图可视化
- **XYFlow**: 流程图编辑器

#### 6.1.5 工具库
- **Axios**: HTTP 客户端
- **ahooks**: React Hooks 工具集
- **dayjs**: 日期处理
- **i18next**: 国际化
- **dompurify**: XSS 防护

### 6.2 路由设计

#### UmiJS 约定式路由
```typescript
// .umirc.ts
export default {
  routes: [
    {
      path: '/',
      component: '@/layouts/BasicLayout',
      routes: [
        { path: '/', redirect: '/knowledge' },
        { path: '/knowledge', component: '@/pages/knowledge' },
        { path: '/knowledge/:id', component: '@/pages/knowledge/detail' },
        { path: '/chat', component: '@/pages/chat' },
        { path: '/flow', component: '@/pages/flow' },
        { path: '/setting', component: '@/pages/user-setting' },
      ],
    },
    {
      path: '/login',
      component: '@/layouts/BlankLayout',
      routes: [
        { path: '/login', component: '@/pages/login' },
      ],
    },
  ],
};
```

### 6.3 样式方案

#### 6.3.1 Tailwind CSS
```tsx
// 原子化样式
<div className="flex items-center justify-between p-4 bg-white rounded-lg shadow-md">
  <h2 className="text-xl font-bold text-gray-800">知识库</h2>
  <button className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
    创建
  </button>
</div>
```

#### 6.3.2 CSS Modules
```tsx
// 组件样式隔离
import styles from './index.module.less';

<div className={styles.container}>
  <div className={styles.header}>...</div>
</div>
```

### 6.4 性能优化

#### 6.4.1 代码分割
```typescript
// 路由懒加载
const ChatPage = lazy(() => import('@/pages/chat'));
const FlowPage = lazy(() => import('@/pages/flow'));
```

#### 6.4.2 组件优化
```tsx
// React.memo 避免不必要渲染
export const ChatMessage = React.memo(({ message }) => {
  return <div>{message.content}</div>;
});

// useMemo 缓存计算结果
const sortedList = useMemo(() => {
  return list.sort((a, b) => b.timestamp - a.timestamp);
}, [list]);

// useCallback 缓存函数
const handleClick = useCallback(() => {
  doSomething();
}, [dependency]);
```

#### 6.4.3 虚拟滚动
```tsx
// 大列表优化
import { List } from 'react-virtualized';

<List
  width={800}
  height={600}
  rowCount={items.length}
  rowHeight={50}
  rowRenderer={({ index, key, style }) => (
    <div key={key} style={style}>
      {items[index]}
    </div>
  )}
/>
```

### 6.5 国际化

```typescript
// locales/zh-CN/knowledge.ts
export default {
  'knowledge.title': '知识库',
  'knowledge.create': '创建知识库',
  'knowledge.delete': '删除',
};

// locales/en-US/knowledge.ts
export default {
  'knowledge.title': 'Knowledge Base',
  'knowledge.create': 'Create Knowledge Base',
  'knowledge.delete': 'Delete',
};

// 组件中使用
import { useIntl } from 'umi';

const { formatMessage } = useIntl();
<h1>{formatMessage({ id: 'knowledge.title' })}</h1>
```

### 6.6 表单处理

```tsx
// 使用 Ant Design Form
import { Form, Input, Button } from 'antd';

const CreateKnowledgeForm = () => {
  const [form] = Form.useForm();
  
  const onFinish = async (values) => {
    await knowledgeService.createKnowledge(values);
    message.success('创建成功');
  };
  
  return (
    <Form form={form} onFinish={onFinish}>
      <Form.Item
        name="name"
        label="知识库名称"
        rules={[{ required: true, message: '请输入名称' }]}
      >
        <Input placeholder="输入知识库名称" />
      </Form.Item>
      
      <Form.Item>
        <Button type="primary" htmlType="submit">
          创建
        </Button>
      </Form.Item>
    </Form>
  );
};
```

### 6.7 流式响应处理

```typescript
// 处理 SSE 流式数据
export const useChatStream = () => {
  const [content, setContent] = useState('');
  const [loading, setLoading] = useState(false);
  
  const sendMessage = async (message: string) => {
    setLoading(true);
    setContent('');
    
    try {
      await chatService.sendMessageStream(message, {
        onMessage: (chunk) => {
          setContent(prev => prev + chunk);
        },
      });
    } finally {
      setLoading(false);
    }
  };
  
  return { content, loading, sendMessage };
};
```

---

## 7. 设计模式与最佳实践

### 7.1 组件设计原则
- **单一职责**: 每个组件只负责一个功能
- **可复用性**: 通用组件提取到 `components/`
- **组合优于继承**: 使用组合模式构建复杂组件
- **Props 类型化**: 所有 Props 必须有 TypeScript 类型定义

### 7.2 状态管理原则
- **服务端状态**: 使用 React Query
- **客户端状态**: 使用 Zustand 或 Context
- **局部状态**: 使用 useState
- **避免 Prop Drilling**: 使用 Context 或状态管理库

### 7.3 错误边界
```tsx
// 错误边界组件
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  
  componentDidCatch(error, errorInfo) {
    console.error('Error:', error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return <div>出错了，请刷新页面</div>;
    }
    return this.props.children;
  }
}
```

---

## 8. 构建与部署

### 8.1 开发环境
```bash
# 安装依赖
npm install

# 启动开发服务器
npm run dev  # 运行在 http://localhost:8000

# 代码检查
npm run lint

# 运行测试
npm run test
```

### 8.2 生产构建
```bash
# 构建生产版本
npm run build

# 输出目录: web/dist/
# 包含优化后的静态文件
```

### 8.3 部署
- **静态托管**: Nginx 提供静态文件服务
- **CDN**: 静态资源上传到 CDN
- **Gzip 压缩**: Nginx 开启 Gzip
- **缓存策略**: HTML 不缓存，JS/CSS/图片长期缓存

---

## 9. 总结

### 9.1 层的特点
- ✅ **用户友好**: 直观的界面设计
- ✅ **响应式**: 支持不同屏幕尺寸
- ✅ **高性能**: 代码分割、虚拟滚动优化
- ✅ **类型安全**: TypeScript 全覆盖
- ✅ **国际化**: 支持多语言
- ✅ **可维护**: 模块化、组件化设计

### 9.2 未来优化方向
- [ ] PWA 支持（离线访问）
- [ ] 移动端优化
- [ ] 暗黑模式
- [ ] 更多数据可视化
- [ ] 性能监控集成

---

**文档版本**: 1.0  
**最后更新**: 2025-12-18  
**维护者**: RAGFlow 团队
