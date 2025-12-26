# 共享层 (Shared Layer)

## 概述
共享层位于 `packages/global/` 目录，提供跨端共享的类型定义、常量、工具函数、Schema 校验和 SDK 封装，是前端、服务端和其他模块的共同依赖基础。

## 技术栈
- **语言**: TypeScript
- **校验**: Zod Schema
- **SDK**: OpenAI SDK 封装
- **工具**: lodash, dayjs, nanoid

## 核心架构

### 目录结构
```
packages/global/
├── common/                # 通用工具与类型
│   ├── error/            # 错误定义
│   ├── file/             # 文件工具
│   ├── string/           # 字符串工具
│   ├── time/             # 时间工具
│   ├── system/           # 系统工具
│   └── parentFolder/     # 文件夹工具
├── core/                  # 核心业务模块
│   ├── ai/               # AI 相关
│   ├── app/              # 应用相关
│   ├── chat/             # 对话相关
│   ├── dataset/          # 知识库相关
│   ├── plugin/           # 插件相关
│   └── workflow/         # 工作流相关
├── support/              # 支持模块
│   ├── user/             # 用户相关
│   ├── permission/       # 权限相关
│   ├── wallet/           # 钱包相关
│   └── outLink/          # 外链相关
├── openapi/              # OpenAPI 定义
└── sdk/                  # SDK 封装
```

## 核心模块

### 1. 通用工具 (`common/`)

#### 错误定义 (`error/`)
- **CommonErrEnum**: 通用错误码
- **DatasetErrEnum**: 知识库错误码
- **AppErrEnum**: 应用错误码
- **UserError**: 自定义错误类

#### 字符串工具 (`string/`)
- `getNanoid`: 生成唯一 ID
- `hashStr`: 字符串哈希
- `simpleText`: 简化文本

#### 时间工具 (`time/`)
- `getSystemTime`: 获取系统时间
- `formatTime`: 格式化时间
- Timezone 处理

#### 文件工具 (`file/`)
- `getFileIcon`: 获取文件图标
- `getFileExtension`: 获取文件扩展名
- `parseDataChunk`: 解析数据块

### 2. 核心业务模块 (`core/`)

#### AI 相关 (`core/ai`)
**常量**
- `ChatRoleEnum`: 对话角色 (system/user/assistant)
- `ModelProvider`: 模型提供商
- `FunctionCallStatusEnum`: 函数调用状态

**类型**
```typescript
type LLMModelItemType = {
  model: string;
  name: string;
  maxToken: number;
  maxResponse: number;
  quoteMaxToken: number;
  price: number;
};
```

**Schema**
- `LLMModelSchema`: 模型配置校验

#### 应用相关 (`core/app`)
**常量**
- `AppTypeEnum`: 应用类型
- `AppFolderTypeList`: 文件夹类型
- `ToolTypeList`: 工具类型

**类型**
```typescript
type AppSchema = {
  _id: string;
  teamId: string;
  name: string;
  type: AppTypeEnum;
  modules: FlowModuleItemType[];
  edges: Edge[];
  chatConfig: AppChatConfigType;
};
```

**Utils**
- `getDefaultAppForm`: 获取默认应用表单
- `removeUnauthModels`: 移除未授权模型

#### 对话相关 (`core/chat`)
**常量**
- `ChatItemValueTypeEnum`: 消息值类型
- `ChatStatusEnum`: 对话状态

**类型**
```typescript
type ChatItemType = {
  obj: ChatRoleEnum;
  value: ChatItemValueItemType[];
  dataId?: string;
};
```

**Utils**
- `formatChatValue2InputType`: 格式化对话值
- `getMaxHistoryLimitFromNodes`: 获取历史限制

#### 知识库相关 (`core/dataset`)
**常量**
- `DatasetTypeEnum`: 知识库类型
- `DatasetCollectionTypeEnum`: 集合类型
- `TrainingModeEnum`: 训练模式

**类型**
```typescript
type DatasetItemType = {
  _id: string;
  name: string;
  vectorModel: string;
  agentModel: string;
};
```

**Search**
- `searchDatasetData`: 数据检索逻辑

#### 工作流相关 (`core/workflow`)
**节点类型**
- `FlowNodeTypeEnum`: 节点类型枚举
- `FlowNodeInputTypeEnum`: 输入类型

**类型**
```typescript
type FlowNodeItemType = {
  nodeId: string;
  name: string;
  intro?: string;
  avatar?: string;
  flowNodeType: FlowNodeTypeEnum;
  inputs: FlowNodeInputItemType[];
  outputs: FlowNodeOutputItemType[];
};
```

**运行时**
- `RuntimeNodeItemType`: 运行时节点
- `DispatchNodeResultType`: 节点执行结果

**Utils**
- `getHandleId`: 获取连接点 ID
- `checkNodeRunStatus`: 检查节点运行状态
- `getReferenceVariableValue`: 获取引用变量

### 3. 支持模块 (`support/`)

#### 用户相关 (`support/user`)
**权限常量**
- `PermissionTypeEnum`: 权限类型
- `PerResourceTypeEnum`: 资源类型
- `ReadPermissionVal`: 读权限值
- `WritePermissionVal`: 写权限值

**类型**
```typescript
type UserType = {
  _id: string;
  username: string;
  avatar: string;
  timezone: string;
};
```

#### 钱包相关 (`support/wallet`)
**类型**
```typescript
type BillSchema = {
  teamId: string;
  tmbId: string;
  appId: string;
  total: number;
  list: BillItemType[];
};
```

### 4. OpenAPI 定义 (`openapi/`)

#### Schema 定义
- Chat Completions API Schema
- Dataset API Schema
- App API Schema

#### 响应类型
- `ChatCompletionResponse`
- `DatasetSearchResponse`

### 5. SDK 封装 (`sdk/`)

#### Plugin SDK
```typescript
import { FastGPTPluginSDK } from '@fastgpt-sdk/plugin';

const sdk = new FastGPTPluginSDK({
  apiKey: 'xxx',
  baseURL: 'xxx'
});
```

## Zod Schema 校验

### 应用 Schema
```typescript
export const AppUpdateSchema = z.object({
  name: z.string().min(1).max(30),
  avatar: z.string().optional(),
  type: z.nativeEnum(AppTypeEnum),
  intro: z.string().max(500).optional()
});
```

### 知识库 Schema
```typescript
export const DatasetUpdateSchema = z.object({
  name: z.string().min(1).max(30),
  avatar: z.string().optional(),
  intro: z.string().max(500).optional(),
  vectorModel: z.string()
});
```

## 工具函数示例

### 字符串工具
```typescript
import { getNanoid } from '@fastgpt/global/common/string/tools';
const id = getNanoid(12); // 生成 12 位 ID
```

### 时间工具
```typescript
import { getSystemTime } from '@fastgpt/global/common/time/timezone';
const now = getSystemTime('Asia/Shanghai');
```

### 错误处理
```typescript
import { UserError } from '@fastgpt/global/common/error/utils';
import { CommonErrEnum } from '@fastgpt/global/common/error/code/common';

throw new UserError({
  message: CommonErrEnum.fileNotFound,
  statusText: '文件不存在'
});
```

## 类型安全

### 严格类型定义
- 所有 API 请求/响应类型化
- 枚举类型替代字符串常量
- 泛型类型复用

### 类型导出
```typescript
// 前端使用
import type { AppSchema } from '@fastgpt/global/core/app/type';

// 服务端使用
import type { ChatDispatchProps } from '@fastgpt/global/core/workflow/runtime/type';
```

## 相关文档
- [前端展示层](./01-frontend-layer.md)
- [API 路由层](./02-api-routes-layer.md)
- [业务服务层](./03-service-layer.md)