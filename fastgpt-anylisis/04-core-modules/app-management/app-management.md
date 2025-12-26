# FastGPT 应用管理模块深度分析

## 一、模块概述

### 1.1 核心职责

应用管理模块是 FastGPT 平台的核心模块之一，负责：

- **应用生命周期管理**：创建、编辑、删除、复制应用
- **应用类型支持**：Simple、Agent、Workflow 等多种应用类型
- **版本控制**：应用版本管理与发布
- **权限控制**：应用的访问权限与协作
- **配置管理**：聊天配置、工作流配置、工具配置

### 1.2 技术特点

- **多类型支持**：统一架构支持多种应用类型
- **版本管理**：支持版本发布、回滚、草稿管理
- **工作流存储**：节点和边的持久化存储
- **权限体系**：支持团队级和应用级权限控制

### 1.3 目录结构

```
packages/
├── global/core/app/           # 应用类型定义
│   ├── constants.ts           # 常量定义（AppTypeEnum）
│   ├── type.d.ts              # 类型声明
│   ├── utils.ts               # 工具函数
│   ├── version.d.ts           # 版本类型
│   ├── evaluation/            # 评估相关
│   ├── logs/                  # 日志相关
│   └── tool/                  # 工具相关
│
├── service/core/app/          # 应用服务层
│   ├── schema.ts              # MongoDB Schema
│   ├── controller.ts          # 控制器
│   ├── utils.ts               # 服务工具
│   ├── http.ts                # HTTP工具
│   ├── mcp.ts                 # MCP客户端
│   ├── version/               # 版本管理
│   ├── tool/                  # 工具管理
│   ├── templates/             # 应用模板
│   ├── evaluation/            # 应用评估
│   ├── logs/                  # 应用日志
│   └── provider/              # 提供者管理
│
└── projects/app/src/pages/api/core/app/  # API路由
    ├── create.ts              # 创建应用
    ├── update.ts              # 更新应用
    ├── delete.ts              # 删除应用
    ├── list.ts                # 应用列表
    ├── detail.ts              # 应用详情
    ├── version/               # 版本API
    └── ...
```

---

## 二、应用类型体系

### 2.1 应用类型枚举

**文件位置**: `packages/global/core/app/constants.ts`

```typescript
export enum AppTypeEnum {
  folder = 'folder',           // 文件夹（组织应用）
  toolFolder = 'toolFolder',   // 工具文件夹
  simple = 'simple',           // 简单应用（纯对话）
  agent = 'agent',             // Agent智能体应用
  workflow = 'advanced',       // 工作流应用
  workflowTool = 'plugin',     // 工作流工具（可复用）
  mcpToolSet = 'toolSet',      // MCP工具集
  httpToolSet = 'httpToolSet', // HTTP工具集
  hidden = 'hidden',           // 隐藏应用

  // deprecated
  tool = 'tool',
  httpPlugin = 'httpPlugin'
}
```

### 2.2 应用类型对比

| 类型 | 复杂度 | 工具调用 | 工作流 | 版本管理 | 典型场景 |
|------|--------|----------|--------|----------|----------|
| **simple** | 低 | ❌ | 隐式 | ✅ | 简单问答 |
| **agent** | 中 | ✅ (自动) | 隐式 | ✅ | 工具辅助对话 |
| **workflow** | 高 | ✅ (手动) | 显式 | ✅ | 复杂业务流程 |
| **workflowTool** | 中 | - | 显式 | ✅ | 可复用工作流 |
| **mcpToolSet** | 中 | - | - | ❌ | MCP工具集成 |
| **httpToolSet** | 中 | - | - | ❌ | HTTP API集成 |

### 2.3 可创建应用列表

**文件位置**: `packages/global/core/app/constants.ts`

```typescript
export const AppTypeList = [
  AppTypeEnum.simple,     // 简单应用
  AppTypeEnum.agent,      // Agent应用
  AppTypeEnum.workflow    // 工作流应用
];
```

### 2.4 工具应用列表

```typescript
export const ToolTypeList = [
  AppTypeEnum.mcpToolSet,    // MCP工具集
  AppTypeEnum.httpToolSet    // HTTP工具集
];
```

---

## 三、数据模型设计

### 3.1 应用 Schema

**文件位置**: `packages/service/core/app/schema.ts`

**表名**: `apps`

**核心字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 应用唯一标识 |
| `parentId` | ObjectId | 父应用/文件夹ID |
| `teamId` | ObjectId | 所属团队 |
| `tmbId` | ObjectId | 创建者成员ID |
| `name` | String | 应用名称 |
| `type` | AppTypeEnum | 应用类型 |
| `version` | 'v1'\|'v2' | 应用版本格式 |
| `avatar` | String | 应用头像 |
| `intro` | String | 应用简介 |
| `templateId` | String | 创建来源模板ID |
| `modules` | Array | 工作流节点列表 |
| `edges` | Array | 工作流边列表 |
| `chatConfig` | Object | 聊天配置 |
| `pluginData` | Object | 工具/插件配置 |
| `scheduledTriggerConfig` | Object | 定时触发配置 |
| `inheritPermission` | Boolean | 是否继承权限 |
| `updateTime` | Date | 更新时间 |

### 3.2 索引设计

```typescript
AppSchema.index({ type: 1 });
AppSchema.index({ teamId: 1, updateTime: -1 });
AppSchema.index({ teamId: 1, type: 1 });
AppSchema.index(
  { scheduledTriggerConfig: 1, scheduledTriggerNextTime: -1 },
  { partialFilterExpression: { scheduledTriggerConfig: { $exists: true } } }
);
```

### 3.3 应用类型声明

**文件位置**: `packages/global/core/app/type.d.ts`

```typescript
export type AppSchema = {
  _id: string;
  parentId?: ParentIdType;
  teamId: string;
  tmbId: string;
  type: AppTypeEnum;
  version?: 'v1' | 'v2';
  name: string;
  avatar: string;
  intro: string;
  templateId?: string;
  updateTime: Date;
  modules: StoreNodeItemType[];
  edges: StoreEdgeItemType[];
  pluginData?: {
    nodeVersion?: string;
    pluginUniId?: string;
    apiSchemaStr?: string;
    customHeaders?: string;
  };
  chatConfig: AppChatConfigType;
  scheduledTriggerConfig?: AppScheduledTriggerConfigType | null;
  scheduledTriggerNextTime?: Date;
  inheritPermission?: boolean;
  favourite?: boolean;
  quick?: boolean;
};
```

---

## 四、应用版本管理

### 4.1 版本管理机制

FastGPT 支持应用版本管理，允许用户：

- **发布版本**：将当前工作流发布为正式版本
- **版本回滚**：回滚到历史版本
- **草稿管理**：编辑中的工作流作为草稿保存

### 4.2 版本数据结构

**目录**: `packages/service/core/app/version/`

版本管理涉及的核心概念：
- **当前版本**：正在对外服务的版本
- **草稿版本**：编辑中但未发布的版本
- **历史版本**：已发布的历史版本列表

### 4.3 版本操作流程

```
编辑工作流 → 保存草稿 → 发布版本 → 对外服务
                ↑           ↓
                └── 回滚版本 ←┘
```

---

## 五、应用配置系统

### 5.1 聊天配置 (ChatConfig)

**文件位置**: `packages/global/core/app/type.d.ts`

```typescript
export type AppChatConfigType = {
  welcomeText?: string;           // 欢迎语
  variables?: VariableItemType[]; // 全局变量
  autoExecute?: AppAutoExecuteConfigType;  // 自动执行
  questionGuide?: AppQGConfigType;         // 问题引导
  ttsConfig?: AppTTSConfigType;            // TTS配置
  whisperConfig?: AppWhisperConfigType;    // 语音识别
  scheduledTriggerConfig?: AppScheduledTriggerConfigType; // 定时触发
  chatInputGuide?: ChatInputGuideConfigType; // 输入引导
  fileSelectConfig?: AppFileSelectConfigType; // 文件选择
  instruction?: string;           // 使用说明
};
```

### 5.2 变量系统

应用支持定义全局变量，供工作流节点使用：

| 变量类型 | 说明 |
|----------|------|
| `input` | 文本输入 |
| `textarea` | 多行文本 |
| `select` | 下拉选择 |
| `numberInput` | 数字输入 |
| `custom` | 自定义类型 |
| `external` | 外部变量 |

### 5.3 文件选择配置

```typescript
export type AppFileSelectConfigType = {
  canSelectFile?: boolean;     // 允许选择文件
  canSelectImg?: boolean;      // 允许选择图片
  maxFiles?: number;           // 最大文件数
  customExtensions?: string[]; // 自定义扩展名
};
```

### 5.4 默认配置

**文件位置**: `packages/global/core/app/constants.ts`

```typescript
export const defaultChatInputGuideConfig = {
  open: false,
  textList: [],
  customUrl: ''
};

export const defaultTTSConfig = {
  type: 'web' as const
};

export const defaultWhisperConfig = {
  open: false,
  autoSend: false,
  autoTTSResponse: false
};

export const defaultQGConfig = {
  open: false
};
```

---

## 六、工具应用

### 6.1 HTTP 工具集

HTTP工具集允许将外部 HTTP API 封装为工具供 Agent 调用。

**配置结构**:

```typescript
export type HttpToolConfigType = {
  name: string;                    // 工具名称
  description: string;             // 工具描述
  inputSchema: JSONSchemaInputType;  // 输入Schema
  outputSchema: JSONSchemaOutputType; // 输出Schema
  path: string;                    // API路径
  method: string;                  // HTTP方法
  staticParams?: Array<{ key: string; value: string }>;
  staticHeaders?: Array<{ key: string; value: string }>;
  staticBody?: {
    type: ContentTypes;
    content?: string;
    formData?: Array<{ key: string; value: string }>;
  };
  headerSecret?: StoreSecretValueType;
};
```

### 6.2 MCP 工具集

MCP (Model Context Protocol) 工具集支持连接外部 MCP 服务：

**相关文件**: `packages/service/core/app/mcp.ts`

MCP 工具集特点：
- 动态工具发现
- 连接复用
- 工具调用转发

详细信息请参考 [MCP工具模块分析](../mcp-tools-plugins/README.md)

### 6.3 工作流工具

工作流工具 (`workflowTool`) 允许将一个工作流封装为可复用工具：

- 可被其他 Agent/工作流调用
- 支持定义输入输出参数
- 支持版本管理

---

## 七、API 接口

### 7.1 应用管理 API

**路由前缀**: `/api/core/app/`

| 接口 | 方法 | 说明 |
|------|------|------|
| `/create` | POST | 创建应用 |
| `/update` | PUT | 更新应用 |
| `/delete` | DELETE | 删除应用 |
| `/list` | GET | 获取应用列表 |
| `/detail` | GET | 获取应用详情 |
| `/copy` | POST | 复制应用 |
| `/move` | PUT | 移动应用 |

### 7.2 版本管理 API

**路由前缀**: `/api/core/app/version/`

| 接口 | 方法 | 说明 |
|------|------|------|
| `/publish` | POST | 发布版本 |
| `/list` | GET | 获取版本列表 |
| `/rollback` | POST | 回滚版本 |
| `/detail` | GET | 获取版本详情 |

### 7.3 工具管理 API

**路由前缀**: `/api/core/app/tool/`

| 接口 | 方法 | 说明 |
|------|------|------|
| `/http/create` | POST | 创建HTTP工具 |
| `/mcp/list` | GET | 获取MCP工具列表 |

---

## 八、最佳实践

### 8.1 应用类型选择

| 需求 | 推荐类型 |
|------|----------|
| 简单问答机器人 | Simple |
| 需要查询知识库 | Agent |
| 需要调用外部API | Agent + HTTP工具 |
| 复杂多步骤流程 | Workflow |
| 可复用的流程片段 | Workflow Tool |

### 8.2 工作流设计原则

1. **模块化设计**：将复杂流程拆分为可复用的工具
2. **变量命名规范**：使用有意义的变量名
3. **错误处理**：在关键节点添加错误处理
4. **版本管理**：定期发布稳定版本

### 8.3 性能优化建议

1. **减少节点数量**：合并可合并的节点
2. **合理使用并行**：利用工作流并行执行能力
3. **缓存复用**：使用变量缓存中间结果
4. **Token优化**：控制历史记录长度

---

## 九、核心控制器函数

### 9.1 应用管理函数

**文件位置**: `packages/service/core/app/controller.ts`

| 函数 | 说明 |
|------|------|
| `beforeUpdateAppFormat` | 更新前格式化应用数据 |
| `findAppAndAllChildren` | 查找应用及所有子应用 |
| `getAppBasicInfoByIds` | 批量获取应用基础信息 |
| `onDelOneApp` | 删除单个应用 |

### 9.2 工具函数

**文件位置**: `packages/global/core/app/utils.ts`

| 函数 | 说明 |
|------|------|
| `getUploadFileType` | 获取上传文件类型配置 |

---

## 十、与其他模块的关系

```
┌─────────────────────────────────────────────────┐
│                  应用管理模块                    │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐     │
│  │  Simple   │ │   Agent   │ │ Workflow  │     │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘     │
└────────┼─────────────┼─────────────┼───────────┘
         │             │             │
         ▼             ▼             ▼
┌─────────────────────────────────────────────────┐
│                  工作流引擎                      │
│  (所有应用类型底层都基于工作流实现)              │
└────────┬────────────────────────────┬───────────┘
         │                            │
         ▼                            ▼
┌─────────────────┐          ┌─────────────────┐
│   知识库模块    │          │   对话管理模块   │
└─────────────────┘          └─────────────────┘
```

---

**文档版本**: v1.0  
**创建日期**: 2024-12-26  
**基于 FastGPT 版本**: v4.9+
