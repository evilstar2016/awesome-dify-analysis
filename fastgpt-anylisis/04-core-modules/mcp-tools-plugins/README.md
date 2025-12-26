# MCP 工具与插件模块文档

本目录包含 FastGPT 系统中 MCP (Model Context Protocol) 工具与插件模块的完整分析文档。

## 📚 文档列表

### 1. 模块分析文档
**文件**: `mcp-module-analysis.md`

这是核心分析文档，包含：
- 模块概述和架构定位
- 核心组件详解（MCP Client、MCP Server、工具转换层、工具执行引擎）
- 数据模型和配置格式
- 核心流程说明
- 技术特点和实现细节
- 使用场景和最佳实践
- 扩展点说明

**适合阅读对象**：
- 需要深入了解 MCP 模块架构的开发者
- 计划扩展或维护 MCP 功能的工程师
- 需要集成 MCP 工具的系统架构师

### 2. MCP 客户端调用时序图
**文件**: `mcp-client-call-sequence.puml`

展示 FastGPT 作为 MCP 客户端调用外部 MCP 服务的完整流程：
- 工作流引擎识别和解析 MCP 工具
- 连接管理和复用机制
- 工具调用和结果处理
- 连接清理流程

**关键场景**：
- 工作流中使用外部 MCP 工具
- 连接缓存和性能优化
- 错误处理和重试机制

### 3. MCP 服务端处理时序图
**文件**: `mcp-server-sequence.puml`

展示 FastGPT 作为 MCP 服务端对外提供工具的完整流程：
- 服务初始化和协议适配
- 工具列表获取和权限验证
- 工具调用和工作流执行
- 结果格式化和聊天记录保存

**关键场景**：
- 外部系统集成 FastGPT 能力
- 工作流和插件作为 MCP 工具暴露
- 权限控制和安全机制

### 4. MCP 工具注册与发现时序图
**文件**: `mcp-tool-registration-sequence.puml`

展示 MCP 工具的完整生命周期管理：
- 注册外部 MCP 工具集
- 在工作流中发现和使用工具
- 工作流执行时的工具解析
- 工具配置更新和同步

**关键场景**：
- 添加新的 MCP 工具集
- 工具在工作流编辑器中的展示
- 工具配置的版本管理

### 5. MCP 完整流程时序图
**文件**: `mcp-complete-flow-sequence.puml`

综合展示 MCP 工具的端到端流程：
- **第一部分**：FastGPT 调用外部 MCP 工具
- **第二部分**：外部系统调用 FastGPT MCP 服务
- **第三部分**：工具生命周期管理

这张图展示了 FastGPT 如何同时扮演 MCP 客户端和服务端的双重角色。

## 🎯 快速导航

### 我想了解...

#### MCP 模块的整体架构
→ 阅读 `mcp-module-analysis.md` 的第 1-2 章节

#### 如何在工作流中使用外部 MCP 工具
→ 查看 `mcp-client-call-sequence.puml` 和 `mcp-tool-registration-sequence.puml`

#### 如何将 FastGPT 能力暴露为 MCP 服务
→ 查看 `mcp-server-sequence.puml`

#### MCP 工具的数据结构和配置
→ 阅读 `mcp-module-analysis.md` 的第 3 章节

#### MCP 的安全和性能优化
→ 阅读 `mcp-module-analysis.md` 的第 5 章节和第 10.3 节

#### 完整的端到端流程
→ 查看 `mcp-complete-flow-sequence.puml`

## 🔧 代码位置索引

### 核心代码文件

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| MCP 客户端 | `packages/service/core/app/mcp.ts` | MCPClient 类实现 |
| MCP 服务端 (独立) | `projects/mcp_server/src/index.ts` | 独立 MCP Server |
| MCP 服务端 (集成) | `projects/app/src/pages/api/mcp/app/[key]/mcp.ts` | 集成 MCP Server |
| 工具转换 | `packages/global/core/app/tool/mcpTool/utils.ts` | 运行时节点转换 |
| 工具执行 | `packages/service/core/workflow/dispatch/child/runTool.ts` | dispatchRunTool 函数 |
| 服务端工具处理 | `projects/app/src/service/support/mcp/utils.ts` | getMcpServerTools, callMcpServerTool |
| MCP 密钥管理 | `packages/service/support/mcp/schema.ts` | MongoDB Schema |
| API 端点 | `projects/app/src/pages/api/support/mcp/` | MCP 相关 API |

### 类型定义

| 类型 | 文件路径 | 说明 |
|-----|---------|------|
| McpToolConfigType | `packages/global/core/app/tool/mcpTool/type.d.ts` | MCP 工具配置 |
| McpToolSetDataType | 同上 | MCP 工具集数据 |
| McpKeyType | `packages/global/support/mcp/type.d.ts` | MCP 密钥类型 |
| toolCallProps | `projects/app/src/service/support/mcp/type.d.ts` | 工具调用参数 |

## 🏗️ 架构概览

```
FastGPT MCP 模块架构

┌─────────────────────────────────────────────────┐
│                 应用层                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ 工作流   │  │  对话    │  │  应用    │      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
└───────┼─────────────┼─────────────┼────────────┘
        │             │             │
┌───────┼─────────────┼─────────────┼────────────┐
│       │       MCP 工具与插件模块   │            │
│  ┌────▼─────────────▼─────────────▼─────┐      │
│  │         双向 MCP 接口                 │      │
│  │  ┌──────────┐      ┌──────────┐      │      │
│  │  │MCP Client│      │MCP Server│      │      │
│  │  │(调用外部)│      │(对外服务)│      │      │
│  │  └──────────┘      └──────────┘      │      │
│  │  ┌──────────┐      ┌──────────┐      │      │
│  │  │ 工具发现 │      │ 工具注册 │      │      │
│  │  └──────────┘      └──────────┘      │      │
│  │  ┌──────────────────────────────┐    │      │
│  │  │      工具执行引擎             │    │      │
│  │  └──────────────────────────────┘    │      │
│  └────────────────────────────────────────┘      │
└───────┬─────────────────────────────┬──────────┘
        │                             │
┌───────▼─────────────┐       ┌───────▼─────────┐
│  外部 MCP 服务      │       │  FastGPT 内部   │
│ (第三方工具)        │       │  (工作流/插件)  │
└─────────────────────┘       └─────────────────┘
```

## 📊 关键概念

### MCP (Model Context Protocol)
一个开放的标准协议，用于 AI 应用与工具服务之间的通信。FastGPT 完整实现了 MCP 协议的客户端和服务端功能。

### 双重角色
- **MCP Client**：FastGPT 可以调用外部 MCP 服务提供的工具
- **MCP Server**：FastGPT 可以将自身的工作流和插件作为工具暴露给外部系统

### 工具集 vs 工具
- **工具集 (ToolSet)**：一组相关工具的集合，对应一个 MCP 服务端点
- **工具 (Tool)**：具体的可执行单元，有明确的输入输出 Schema

### 连接复用
在工作流执行期间，同一 URL 的 MCP 连接会被缓存和复用，减少连接开销，提高性能。

## 🚀 使用示例

### 场景 1：在工作流中调用天气查询工具

1. 创建 MCP 工具集应用，配置天气服务 URL
2. FastGPT 自动获取该服务提供的工具列表
3. 在工作流中拖入"获取当前天气"工具
4. 配置输入参数（如城市名称）
5. 执行工作流，获取天气数据

### 场景 2：将 FastGPT 对话能力暴露给 Claude Desktop

1. 在 FastGPT 中创建 MCP 密钥，关联对话应用
2. 在 Claude Desktop 配置中添加 FastGPT MCP Server
3. Claude 可以列出 FastGPT 提供的对话工具
4. 用户在 Claude 中调用该工具，实际执行 FastGPT 工作流
5. 返回 FastGPT 的对话结果

## 🔗 相关资源

- [Model Context Protocol 官方文档](https://modelcontextprotocol.io/)
- [FastGPT 官方文档](https://doc.fastgpt.in/)
- [MCP SDK GitHub](https://github.com/modelcontextprotocol/sdk)

## 📝 更新日志

| 日期 | 版本 | 说明 |
|-----|------|------|
| 2025-12-11 | v1.0 | 初始版本，包含完整的模块分析和时序图 |

## 👥 贡献者

本文档由 GitHub Copilot 基于 FastGPT 源码深入分析生成。

---

**注意**：本文档基于 FastGPT 当前代码结构编写，具体实现细节可能随版本更新而变化。建议结合实际代码阅读以获取最准确的信息。
