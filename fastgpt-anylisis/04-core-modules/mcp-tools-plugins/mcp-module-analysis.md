# MCP工具与插件模块分析

## 1. 模块概述

MCP (Model Context Protocol) 工具与插件模块是 FastGPT 系统中用于集成和管理外部工具的核心组件。该模块基于 Model Context Protocol 标准协议，提供了统一的工具接口规范和执行框架，使得 FastGPT 可以灵活地集成各种外部服务和工具，极大地扩展了系统的能力边界。

### 1.1 核心价值

- **标准化协议**：采用 MCP 标准协议，提供统一的工具定义和调用接口
- **灵活扩展**：支持动态添加和管理工具，无需修改核心代码
- **双向通信**：支持 FastGPT 作为 MCP 服务端和客户端的双重角色
- **工作流集成**：无缝集成到 FastGPT 的工作流引擎中

### 1.2 架构定位

```
┌─────────────────────────────────────────────────────┐
│                  FastGPT 应用层                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ 工作流   │  │  对话    │  │  应用    │          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
└───────┼─────────────┼─────────────┼────────────────┘
        │             │             │
┌───────┼─────────────┼─────────────┼────────────────┐
│       │      MCP 工具与插件模块    │                │
│  ┌────▼─────────────▼─────────────▼─────┐          │
│  │         MCP Client/Server             │          │
│  │  ┌──────────┐      ┌──────────┐      │          │
│  │  │ 工具注册 │      │ 工具调用 │      │          │
│  │  └──────────┘      └──────────┘      │          │
│  └────────────────────────────────────────┘          │
└───────┬─────────────────────────────┬────────────────┘
        │                             │
┌───────▼─────────────┐       ┌───────▼──────────────┐
│   外部 MCP 服务     │       │  FastGPT 内部工具    │
│  (通过 HTTP/SSE)    │       │  (Workflow/Plugin)   │
└─────────────────────┘       └──────────────────────┘
```

## 2. 核心组件

### 2.1 MCP Client（客户端）

**位置**：`packages/service/core/app/mcp.ts`

MCP Client 负责连接外部 MCP 服务并调用其提供的工具。

#### 2.1.1 核心类：MCPClient

```typescript
export class MCPClient {
  private client: Client;
  private url: string;
  private headers: Record<string, any>;
  
  constructor(config: { url: string; headers: Record<string, any> })
  async getTools(): Promise<McpToolConfigType[]>
  async toolCall(params): Promise<any>
  async closeConnection(): Promise<void>
}
```

**主要功能**：

1. **连接管理**：支持两种传输协议
   - `StreamableHTTPClientTransport`：基于 HTTP 流式传输
   - `SSEClientTransport`：基于服务器推送事件（SSE）

2. **工具发现**：通过 `getTools()` 获取远程 MCP 服务提供的工具列表
   - 返回工具名称、描述和输入 Schema
   - 自动标准化工具配置格式

3. **工具调用**：通过 `toolCall()` 执行远程工具
   - 支持参数传递和结果返回
   - 可选的连接保持（避免频繁连接）
   - 超时控制（默认 300 秒）

4. **连接复用**：在工作流执行过程中缓存连接
   - 通过 `mcpClientMemory` 缓存已建立的连接
   - 同一 URL 的多次调用共享连接
   - 工作流结束后统一关闭连接

### 2.2 MCP Server（服务端）

**位置**：`projects/mcp_server/` 和 `projects/app/src/pages/api/mcp/`

FastGPT 可以作为 MCP 服务端，将自身的工作流和插件作为工具暴露给外部系统。

#### 2.2.1 独立 MCP Server

**位置**：`projects/mcp_server/src/index.ts`

这是一个独立的 Express 服务，提供标准的 MCP 协议接口：

- **SSE 连接端点**：`GET /:key/sse`
  - 建立服务器推送事件连接
  - 创建 MCP 服务器实例
  - 注册工具列表和调用处理器

- **消息处理端点**：`POST /:key/messages`
  - 处理客户端消息
  - 路由到对应的 SSE 传输通道

**支持的 MCP 请求**：
1. `ListToolsRequest`：返回可用工具列表
2. `CallToolRequest`：执行指定工具

#### 2.2.2 集成 MCP Server

**位置**：`projects/app/src/pages/api/mcp/app/[key]/mcp.ts`

集成在 FastGPT 主应用中，支持 HTTP Streamable 传输：

```typescript
const server = new Server({
  name: 'fastgpt-mcp-server-http-streamable',
  version: '1.0.0'
});

const transport = new StreamableHTTPServerTransport({
  sessionIdGenerator: undefined
});
```

### 2.3 工具转换层

#### 2.3.1 工具配置转换

**位置**：`packages/global/core/app/tool/mcpTool/utils.ts`

提供工具配置到运行时节点的转换：

1. **getMCPToolSetRuntimeNode**：将 MCP 工具集转换为工作流节点
   ```typescript
   {
     nodeId: string,
     flowNodeType: FlowNodeTypeEnum.toolSet,
     toolConfig: {
       mcpToolSet: {
         toolList: McpToolConfigType[],
         headerSecret?: StoreSecretValueType,
         url: string,
         toolId: string
       }
     }
   }
   ```

2. **getMCPToolRuntimeNode**：将单个 MCP 工具转换为工作流节点
   - 自动转换 JSON Schema 为节点输入配置
   - 生成标准的原始响应输出

#### 2.3.2 Schema 转换

**位置**：`projects/app/src/service/support/mcp/utils.ts`

将 FastGPT 工作流/插件转换为 MCP 工具：

1. **pluginNodes2InputSchema**：插件节点转 MCP 输入 Schema
   - 从 `pluginInput` 节点提取参数定义
   - 转换为 JSON Schema 格式
   - 支持枚举、必填等约束

2. **workflow2InputSchema**：工作流配置转 MCP 输入 Schema
   - 固定包含 `question` 字段
   - 可选的文件上传字段
   - 动态变量字段

### 2.4 工具执行引擎

**位置**：`packages/service/core/workflow/dispatch/child/runTool.ts`

核心函数：`dispatchRunTool`

支持三种 MCP 工具执行模式：

#### 2.4.1 新版 MCP 工具（推荐）

```typescript
if (toolConfig?.mcpTool?.toolId) {
  // 1. 解析工具 ID (格式: mcp-{parentId}/{toolName})
  const { pluginId } = splitCombineToolId(toolConfig.mcpTool.toolId);
  const [parentId, toolName] = pluginId.split('/');
  
  // 2. 获取工具集配置
  const tool = await getAppVersionById({ appId: parentId, versionId: version });
  const { headerSecret, url } = tool.nodes[0].toolConfig?.mcpToolSet;
  
  // 3. 复用或创建 MCP 客户端
  const mcpClient = props.mcpClientMemory?.[url] ?? new MCPClient({
    url,
    headers: getSecretValue({ storeSecret: headerSecret })
  });
  props.mcpClientMemory[url] = mcpClient;
  
  // 4. 调用工具（保持连接）
  const result = await mcpClient.toolCall({ 
    toolName, 
    params, 
    closeConnection: false 
  });
}
```

**特点**：
- 支持连接复用，提高性能
- 工具集和工具分离管理
- 支持密钥加密存储

#### 2.4.2 旧版 MCP 工具（兼容）

```typescript
// 直接从工具数据中获取配置
const { name: toolName, url, headerSecret } = toolData || system_toolData;
const mcpClient = new MCPClient({ url, headers: ... });
const result = await mcpClient.toolCall({ toolName, params: restParams });
```

**特点**：
- 每次调用创建新连接
- 工具配置存储在工具本身
- 向后兼容旧版本应用

### 2.5 服务端工具调用

**位置**：`projects/app/src/service/support/mcp/utils.ts`

#### 2.5.1 getMcpServerTools

获取 MCP 密钥关联的所有工具：

```typescript
export const getMcpServerTools = async (key: string): Promise<Tool[]> => {
  // 1. 验证密钥
  const mcp = await MongoMcpKey.findOne({ key });
  
  // 2. 获取关联的应用列表
  const appList = await MongoApp.find({ 
    _id: { $in: mcp.apps.map(app => app.appId) },
    type: { $in: [AppTypeEnum.simple, AppTypeEnum.workflow, AppTypeEnum.workflowTool] }
  });
  
  // 3. 权限过滤
  const permissionAppList = await Promise.all(
    appList.filter(async (app) => {
      await authAppByTmbId({ tmbId: mcp.tmbId, appId: app._id, per: ReadPermissionVal });
    })
  );
  
  // 4. 获取最新版本并转换为 MCP 工具
  const tools = versionList.map<Tool>((version, index) => {
    const isPlugin = !!version.nodes.find(
      node => node.flowNodeType === FlowNodeTypeEnum.pluginInput
    );
    
    return {
      name: mcpApp.toolName,
      description: mcpApp.description,
      inputSchema: isPlugin 
        ? pluginNodes2InputSchema(version.nodes)
        : workflow2InputSchema(version.chatConfig)
    };
  });
}
```

#### 2.5.2 callMcpServerTool

执行 MCP 工具调用：

```typescript
export const callMcpServerTool = async ({ key, toolName, inputs }: toolCallProps) => {
  // 1. 获取应用配置
  const app = await MongoApp.findOne({ ... });
  
  // 2. 构建用户查询
  const userQuestion = isPlugin 
    ? serverGetWorkflowToolRunUserQuery({ pluginInputs, variables })
    : { obj: ChatRoleEnum.Human, value: [{ type: ChatItemValueTypeEnum.text, ... }] };
  
  // 3. 执行工作流
  const { flowResponses, assistantResponses, ... } = await dispatchWorkFlow({
    mode: 'chat',
    usageSource: UsageSourceEnum.mcp,
    runtimeNodes,
    runtimeEdges,
    variables,
    query: userQuestion.value,
    ...
  });
  
  // 4. 保存聊天记录
  await saveChat({ chatId, appId, source: ChatSourceEnum.mcp, ... });
  
  // 5. 返回结果
  return isPlugin 
    ? JSON.stringify(pluginOutput)
    : assistantResponses.map(item => item?.text?.content).join('\n');
}
```

## 3. 数据模型

### 3.1 MCP 工具配置

```typescript
// MCP 工具基础配置
type McpToolConfigType = {
  name: string;                    // 工具名称
  description: string;             // 工具描述
  inputSchema: JSONSchemaInputType; // 输入参数 Schema
};

// MCP 工具集数据
type McpToolSetDataType = {
  url: string;                     // MCP 服务端点 URL
  headerSecret?: StoreSecretValueType; // 认证头部密钥
  toolList: McpToolConfigType[];   // 工具列表
};

// MCP 工具数据（包含连接信息）
type McpToolDataType = McpToolConfigType & {
  url: string;
  headerSecret?: StoreSecretValueType;
};
```

### 3.2 MCP 密钥配置

```typescript
// 存储在 MongoDB 中
type McpKeyType = {
  _id: ObjectId;
  key: string;           // API 密钥
  teamId: ObjectId;      // 所属团队
  tmbId: ObjectId;       // 创建者
  apps: Array<{
    appId: ObjectId;     // 关联的应用 ID
    toolName: string;    // 工具名称
    description: string; // 工具描述
  }>;
};
```

### 3.3 工作流节点配置

```typescript
// 节点工具配置
type NodeToolConfig = {
  // MCP 工具集节点
  mcpToolSet?: {
    toolId: string;                    // 工具集 App ID
    url: string;                       // MCP 服务 URL
    headerSecret?: StoreSecretValueType;
    toolList: McpToolConfigType[];     // 工具列表
  };
  
  // MCP 工具节点
  mcpTool?: {
    toolId: string; // 格式: mcp-{parentId}/{toolName}
  };
};
```

## 4. 核心流程

### 4.1 MCP 客户端调用流程

```
1. 工作流引擎识别 MCP 工具节点
   ↓
2. 解析工具 ID，获取父工具集 ID 和工具名称
   ↓
3. 从父工具集获取 MCP 服务连接信息（URL、Headers）
   ↓
4. 检查 mcpClientMemory 是否已有该 URL 的连接
   ├─ 有：复用现有连接
   └─ 无：创建新的 MCPClient 实例并缓存
   ↓
5. 调用 mcpClient.toolCall()
   ├─ getConnection(): 建立 HTTP/SSE 连接
   ├─ client.callTool(): 发送工具调用请求
   └─ 返回结果（保持连接不关闭）
   ↓
6. 处理返回结果，生成节点输出
   ↓
7. 工作流结束后，批量关闭所有 MCP 连接
```

### 4.2 MCP 服务端处理流程

```
外部系统
   ↓ 请求工具列表
1. GET /support/mcp/server/toolList?key={key}
   ↓
2. getMcpServerTools(key)
   ├─ 验证 MCP 密钥
   ├─ 查询关联的应用列表
   ├─ 过滤权限（ReadPermission）
   ├─ 获取应用最新版本
   └─ 转换为 MCP Tool Schema
   ↓
3. 返回 Tool[] 列表
   
外部系统
   ↓ 调用工具
4. POST /support/mcp/server/toolCall
   body: { key, toolName, inputs }
   ↓
5. callMcpServerTool({ key, toolName, inputs })
   ├─ 查找对应的应用
   ├─ 获取应用最新版本
   ├─ 构建运行时上下文
   │   ├─ 插件：使用 serverGetWorkflowToolRunUserQuery
   │   └─ 工作流：构建对话消息
   ├─ dispatchWorkFlow() 执行工作流
   ├─ saveChat() 保存聊天记录
   └─ 格式化返回结果
       ├─ 插件：返回 pluginOutput
       └─ 工作流：返回 assistantResponses 文本
   ↓
6. 返回执行结果
```

### 4.3 工具注册流程

```
1. 创建/更新 MCP 密钥
   POST /support/mcp/create
   ↓
2. 配置关联应用
   {
     apps: [
       { appId, toolName, description },
       ...
     ]
   }
   ↓
3. 保存到 MongoDB (mcp_keys 集合)
   ↓
4. 返回密钥 key
   ↓
5. 外部系统使用 key 访问工具
```

## 5. 技术特点

### 5.1 协议适配

支持多种 MCP 传输协议：

1. **HTTP Streamable**：适合 Web 环境
   - 客户端：`StreamableHTTPClientTransport`
   - 服务端：`StreamableHTTPServerTransport`

2. **SSE（Server-Sent Events）**：适合长连接场景
   - 客户端：`SSEClientTransport`
   - 服务端：`SSEServerTransport`

3. **自动降级**：客户端先尝试 HTTP，失败后自动切换到 SSE

### 5.2 安全机制

1. **密钥认证**
   - 通过 `key` 参数验证访问权限
   - 支持自定义 HTTP Headers（如 Bearer Token）

2. **权限控制**
   - 工具访问需要应用读取权限
   - 基于 TeamMember ID 进行权限校验

3. **密钥加密存储**
   - `StoreSecretValueType` 支持加密存储敏感信息
   - 运行时通过 `getSecretValue()` 解密

### 5.3 性能优化

1. **连接复用**
   - 工作流执行期间复用 MCP 客户端连接
   - 减少连接建立开销

2. **缓存机制**
   - 通过 `mcpClientMemory` 缓存连接
   - Key 为 MCP 服务 URL

3. **批量关闭**
   - 工作流结束后统一关闭所有连接
   - 避免连接泄漏

### 5.4 错误处理

1. **工具调用错误**
   - 支持 `catchError` 配置
   - 区分系统错误和自定义错误字段

2. **连接错误**
   - 自动重试机制（`retryFn`）
   - 详细的日志记录

3. **兼容性处理**
   - 新旧版本 MCP 工具共存
   - 向后兼容旧版配置格式

## 6. 使用场景

### 6.1 FastGPT 调用外部工具

**场景**：在工作流中集成第三方 MCP 服务

1. 创建 MCP 工具集应用（`AppTypeEnum.mcpToolSet`）
2. 配置 MCP 服务 URL 和认证信息
3. 从远程获取工具列表并保存
4. 在工作流中使用这些工具节点

**优势**：
- 无需编写适配代码
- 自动处理参数映射
- 统一的工具管理界面

### 6.2 外部系统调用 FastGPT

**场景**：将 FastGPT 的工作流/插件暴露为 MCP 服务

1. 创建 MCP 密钥，关联应用
2. 外部系统连接 FastGPT MCP Server
3. 列出可用工具（FastGPT 的应用）
4. 调用工具执行对话或插件逻辑

**优势**：
- FastGPT 能力标准化输出
- 支持多系统集成
- 完整的执行日志

### 6.3 工具链编排

**场景**：多个 MCP 工具在工作流中协同

1. 配置多个 MCP 工具集
2. 在工作流中串联调用
3. 数据在工具间流转
4. 所有工具共享一个执行上下文

**优势**：
- 统一的执行引擎
- 自动的数据格式转换
- 可视化编排界面

## 7. 扩展点

### 7.1 自定义传输协议

可以实现新的传输层适配器：

```typescript
import { Transport } from '@modelcontextprotocol/sdk/shared/transport.js';

class CustomTransport extends Transport {
  async start() { ... }
  async send(message: JSONRPCMessage) { ... }
  async close() { ... }
}
```

### 7.2 工具类型扩展

除了 MCP 工具，系统还支持：

- **System Tools**：系统内置工具
- **HTTP Tools**：基于 HTTP 的工具
- **Workflow Tools**：工作流作为工具

可以参考 MCP 工具的实现模式扩展新的工具类型。

### 7.3 Schema 增强

可以扩展 JSON Schema 转换逻辑：

```typescript
// 自定义 Schema 转换
export const customJsonSchema2NodeInput = (jsonSchema: JSONSchemaInputType) => {
  // 添加自定义字段类型支持
  // 处理特殊的验证规则
  // ...
};
```

## 8. 配置示例

### 8.1 MCP 工具集节点配置

```json
{
  "nodeId": "mcpToolSet123",
  "flowNodeType": "toolSet",
  "name": "Weather Tools",
  "avatar": "core/app/type/mcpTools",
  "toolConfig": {
    "mcpToolSet": {
      "toolId": "64f1a2b3c4d5e6f7g8h9i0j1",
      "url": "https://weather-mcp.example.com/mcp",
      "headerSecret": {
        "type": "encrypted",
        "value": "encrypted_api_key_here"
      },
      "toolList": [
        {
          "name": "getCurrentWeather",
          "description": "Get current weather for a location",
          "inputSchema": {
            "type": "object",
            "properties": {
              "location": {
                "type": "string",
                "description": "City name or coordinates"
              },
              "unit": {
                "type": "string",
                "enum": ["celsius", "fahrenheit"],
                "description": "Temperature unit"
              }
            },
            "required": ["location"]
          }
        }
      ]
    }
  }
}
```

### 8.2 MCP 工具节点配置

```json
{
  "nodeId": "mcpTool456",
  "flowNodeType": "tool",
  "name": "Get Current Weather",
  "avatar": "core/app/type/mcpToolsFill",
  "toolConfig": {
    "mcpTool": {
      "toolId": "mcp-64f1a2b3c4d5e6f7g8h9i0j1/getCurrentWeather"
    }
  },
  "inputs": [
    {
      "key": "location",
      "valueType": "string",
      "label": "Location",
      "required": true
    },
    {
      "key": "unit",
      "valueType": "string",
      "label": "Temperature Unit",
      "enum": "celsius\nfahrenheit"
    }
  ],
  "outputs": [
    {
      "key": "rawResponse",
      "label": "Raw Response",
      "valueType": "any"
    }
  ]
}
```

## 9. 监控与日志

### 9.1 日志记录

系统在关键位置记录日志：

```typescript
// MCP Client
addLog.debug(`[MCP Client] Call tool: ${toolName}`, params);
addLog.error(`[MCP Client] Failed to call tool ${toolName}:`, error);

// MCP Server
addLog.info(`Call tool: ${name} with args: ${JSON.stringify(args)}`);
addLog.debug('[MCP server] Close connection');
```

### 9.2 使用统计

MCP 工具调用会记录使用量：

```typescript
{
  source: UsageSourceEnum.mcp,
  teamId: runningUserInfo.teamId,
  tmbId: runningUserInfo.tmbId,
  // ...
}
```

### 9.3 聊天记录

每次 MCP 服务端工具调用都会保存完整的聊天记录：

```typescript
await saveChat({
  chatId,
  appId,
  source: ChatSourceEnum.mcp,
  userContent: userQuestion,
  aiContent: aiResponse,
  durationSeconds,
  // ...
});
```

## 10. 最佳实践

### 10.1 工具设计

1. **清晰的命名**：工具名称应该简洁且具有描述性
2. **完整的描述**：提供详细的工具功能说明
3. **规范的 Schema**：使用标准的 JSON Schema 定义输入
4. **合理的粒度**：一个工具完成一个明确的功能

### 10.2 错误处理

1. **使用 catchError**：在工具节点配置中启用错误捕获
2. **提供有意义的错误信息**：帮助用户快速定位问题
3. **区分错误类型**：系统错误 vs 业务逻辑错误

### 10.3 性能优化

1. **启用连接复用**：在工作流中设置 `closeConnection: false`
2. **合理的超时设置**：根据工具特性调整超时时间
3. **异步处理**：耗时操作使用后台任务

### 10.4 安全建议

1. **使用加密存储**：敏感信息通过 `StoreSecretValueType` 存储
2. **最小权限原则**：只授予必要的应用访问权限
3. **定期轮换密钥**：更新 MCP API 密钥
4. **审计日志**：定期检查工具调用日志

## 11. 总结

MCP 工具与插件模块是 FastGPT 系统的重要扩展机制，它通过标准化的 MCP 协议实现了：

1. **灵活的工具集成**：支持任意符合 MCP 协议的外部服务
2. **双向能力输出**：既可以调用外部工具，也可以对外提供工具
3. **无缝工作流集成**：MCP 工具可以像内置节点一样在工作流中使用
4. **完善的生命周期管理**：从工具发现、配置、调用到监控的完整链路

该模块的设计充分体现了**开放、标准、可扩展**的架构理念，为 FastGPT 的能力扩展提供了坚实的基础。
