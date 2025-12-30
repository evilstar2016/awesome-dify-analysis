# Tools & Plugins Module

## Module Overview

Tools & Plugins is Dify's tools and plugins system, providing flexible tool extension mechanisms. Supports multiple types including built-in tools (Builtin), custom API tools (Custom), plugin tools (Plugin), MCP tools, and workflow tools (Workflow as Tool), providing rich capability extensions for Agent and Workflow.

**Core Code Location**: `api/core/tools/`

## Directory File Description

| Filename | Description |
|----------|-------------|
| [dify_tools_and_plugins_development_analysis.md](./dify_tools_and_plugins_development_analysis.md) | Tools and plugins development complete analysis documentation |
| [dify_tools_and_plugins_development_flow.puml](./dify_tools_and_plugins_development_flow.puml) | Tool development flow diagram (PlantUML) |

## Related Modules

- [Agent](../agent/) - How Agent uses tools
- [Workflow](../workflow/) - Workflow tool nodes
- [Model Runtime](../model_runtime/) - Function Calling integration