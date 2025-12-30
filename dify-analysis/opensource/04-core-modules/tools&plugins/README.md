# Tools & Plugins 工具与插件模块

## 模块概述

Tools & Plugins 是 Dify 的工具与插件系统，提供了灵活的工具扩展机制。支持内置工具（Builtin）、自定义 API 工具（Custom）、插件工具（Plugin）、MCP 工具和工作流工具（Workflow as Tool）等多种类型，为 Agent 和 Workflow 提供丰富的能力扩展。

**核心代码位置**：`api/core/tools/`

## 目录文件说明

| 文件名 | 描述 |
|--------|------|
| [dify_tools_and_plugins_development_analysis.md](./dify_tools_and_plugins_development_analysis.md) | 工具与插件开发完整分析文档 |
| [dify_tools_and_plugins_development_flow.puml](./dify_tools_and_plugins_development_flow.puml) | 工具开发流程图（PlantUML） |

## 相关模块

- [Agent 智能体](../agent/) - Agent 如何使用工具
- [Workflow 工作流](../workflow/) - 工作流工具节点
- [Model Runtime](../model_runtime/) - Function Calling 集成
