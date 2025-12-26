# FastGPT 插件系统模块深度分析

## 一、模块概述

### 1.1 核心职责

插件系统模块是 FastGPT 的扩展能力核心，负责：

- **系统插件管理**：官方提供的工具插件
- **团队插件安装**：按团队维度管理插件安装
- **插件标签分类**：插件的组织与分类
- **插件状态管理**：插件的启用、禁用、下线
- **费用管理**：插件使用费用配置

### 1.2 技术特点

- **团队隔离**：插件安装按团队维度
- **状态管理**：支持正常、即将下线、已下线状态
- **费用配置**：支持插件费用设置
- **标签系统**：灵活的分类标签

### 1.3 目录结构

```
packages/
├── global/core/plugin/           # 插件类型定义
│   ├── type.ts                   # 基础类型
│   ├── admin/                    # 管理类型
│   │   └── tool/type.ts
│   ├── schema/                   # Schema类型
│   │   └── type.ts
│   └── tool/                     # 工具类型
│       └── type.ts
│
├── service/core/plugin/          # 插件服务层
│   ├── schema/                   # Schema定义
│   │   └── teamInstalledPluginSchema.ts
│   └── tool/                     # 工具管理
│       ├── systemToolSchema.ts   # 系统工具
│       ├── tagSchema.ts          # 标签
│       └── pluginGroupSchema.ts  # 分组
│
└── projects/app/src/pages/api/core/plugin/  # API路由
```

---

## 二、插件类型体系

### 2.1 插件分类

FastGPT 的插件/工具体系包含多种类型：

| 类型 | 说明 | 来源 |
|------|------|------|
| **系统插件** | 官方提供的工具 | 平台内置 |
| **工作流工具** | 用户创建的工作流工具 | 用户创建 |
| **MCP工具集** | 外部MCP服务 | MCP协议 |
| **HTTP工具集** | 外部HTTP API | OpenAPI |

### 2.2 插件状态枚举

**文件位置**: `packages/global/core/plugin/type.ts`

```typescript
export enum PluginStatusEnum {
  Normal = 1,      // 正常使用
  SoonOffline = 2, // 即将下线
  Offline = 3      // 已下线
}

export const PluginStatusMap = {
  [PluginStatusEnum.Normal]: {
    label: '正常',
    tooltip: '',
    tagColor: 'blue'
  },
  [PluginStatusEnum.SoonOffline]: {
    label: '即将下线',
    tooltip: '此工具即将停用，请注意迁移',
    tagColor: 'yellow'
  },
  [PluginStatusEnum.Offline]: {
    label: '已下线',
    tooltip: '此工具已停用',
    tagColor: 'red'
  }
};
```

### 2.3 插件标签类型

```typescript
export const PluginToolTagSchema = z.object({
  tagId: z.string(),
  tagName: I18nUnioStringSchema,  // 支持国际化
  tagOrder: z.number(),
  isSystem: z.boolean()
});

export type SystemPluginToolTagType = z.infer<typeof PluginToolTagSchema>;
```

---

## 三、数据模型设计

### 3.1 系统插件工具 Schema

**文件位置**: `packages/service/core/plugin/tool/systemToolSchema.ts`

**表名**: `system_plugin_tools`

**核心字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `pluginId` | String | 插件唯一标识 |
| `status` | Number | 插件状态 |
| `defaultInstalled` | Boolean | 是否默认安装 |
| `originCost` | Number | 原始费用 |
| `currentCost` | Number | 当前费用 |
| `hasTokenFee` | Boolean | 是否有Token费用 |
| `pluginOrder` | Number | 排序顺序 |
| `systemKeyCost` | Number | 系统密钥费用 |
| `customConfig` | Object | 自定义配置 |
| `inputListVal` | Object | 输入列表值 |

### 3.2 团队安装记录 Schema

**文件位置**: `packages/service/core/plugin/schema/teamInstalledPluginSchema.ts`

**表名**: `team_installed_plugins`

**核心字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `teamId` | ObjectId | 团队ID |
| `pluginType` | String | 插件类型 |
| `pluginId` | String | 插件ID |
| `installed` | Boolean | 是否已安装 |

**索引**:
```typescript
TeamInstalledPluginSchema.index(
  { teamId: 1, pluginId: 1 }, 
  { unique: true }
);
```

---

## 四、插件安装管理

### 4.1 安装机制

FastGPT 采用团队维度的插件安装机制：

```
系统插件池
    ↓
团队管理员选择安装
    ↓
写入 team_installed_plugins
    ↓
团队成员可使用
```

### 4.2 默认安装

部分核心插件设置为默认安装（`defaultInstalled: true`），新团队自动获得这些插件。

### 4.3 安装状态查询

```typescript
// 查询团队已安装的插件
const installedPlugins = await MongoTeamInstalledPlugin.find({
  teamId: teamId,
  installed: true
});
```

---

## 五、插件标签系统

### 5.1 标签结构

插件标签用于分类组织：

```typescript
{
  tagId: 'search',
  tagName: {
    en: 'Search',
    'zh-CN': '搜索'
  },
  tagOrder: 1,
  isSystem: true
}
```

### 5.2 系统标签 vs 自定义标签

| 类型 | isSystem | 说明 |
|------|----------|------|
| 系统标签 | `true` | 平台预设的分类 |
| 自定义标签 | `false` | 用户自定义分类 |

---

## 六、与其他工具类型的关系

### 6.1 工具类型对比

| 特性 | 系统插件 | 工作流工具 | MCP工具集 | HTTP工具集 |
|------|----------|------------|-----------|------------|
| 来源 | 官方 | 用户 | 外部MCP | 外部API |
| 管理方式 | 安装 | 创建 | 配置 | 配置 |
| 计费 | 平台 | 无 | 无 | 无 |
| 状态管理 | 有 | 无 | 无 | 无 |

### 6.2 统一工具接口

所有工具类型在工作流中统一通过工具节点调用，详见：
- [MCP工具模块](../mcp-tools-plugins/README.md)
- [工作流模块](../workflow/README.md)

---

## 七、API 接口

### 7.1 插件管理 API

**路由前缀**: `/api/core/plugin/`

| 接口 | 方法 | 说明 |
|------|------|------|
| `/list` | GET | 获取可用插件列表 |
| `/install` | POST | 安装插件 |
| `/uninstall` | POST | 卸载插件 |
| `/tag/list` | GET | 获取标签列表 |

### 7.2 系统工具 API (Admin)

| 接口 | 方法 | 说明 |
|------|------|------|
| `/admin/tool/list` | GET | 获取系统工具列表 |
| `/admin/tool/update` | PUT | 更新工具配置 |
| `/admin/tag/create` | POST | 创建标签 |

---

## 八、费用管理

### 8.1 费用类型

| 字段 | 说明 |
|------|------|
| `originCost` | 原始费用（基准价） |
| `currentCost` | 当前费用（实际价） |
| `hasTokenFee` | 是否有额外Token费用 |
| `systemKeyCost` | 使用系统密钥的费用 |

### 8.2 费用计算

- 基础费用：`currentCost`
- Token费用：如果 `hasTokenFee=true`，额外计算Token消耗
- 系统密钥：使用官方密钥额外收取 `systemKeyCost`

---

## 九、与应用的关系

```
┌─────────────────────────────────────────────────┐
│                  插件系统模块                    │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐     │
│  │系统插件   │ │ 插件安装  │ │ 插件标签  │     │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘     │
└────────┼─────────────┼─────────────┼───────────┘
         │             │             │
         ▼             ▼             ▼
┌─────────────────────────────────────────────────┐
│                  应用管理模块                    │
│  ┌──────────────────────────────────────────┐  │
│  │  工具应用: MCP工具集 / HTTP工具集         │  │
│  │  工作流工具: workflowTool                 │  │
│  └──────────────────────────────────────────┘  │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│                  工作流引擎                      │
│           (工具节点统一调用)                     │
└─────────────────────────────────────────────────┘
```

---

**文档版本**: v1.0  
**创建日期**: 2024-12-26  
**基于 FastGPT 版本**: v4.9+
