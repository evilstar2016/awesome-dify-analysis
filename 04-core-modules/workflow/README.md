# Workflow 工作流模块

## 模块概述

Workflow 是 Dify 的工作流引擎，采用事件驱动架构，支持灵活的节点编排和异步执行。提供丰富的内置节点类型（LLM、知识库检索、工具调用、条件分支等），支持自定义节点开发，实现复杂的业务流程自动化。

**核心代码位置**：`api/core/workflow/`

## 目录文件说明

| 文件名 | 描述 |
|--------|------|
| [workflow_management_technical_doc.md](./workflow_management_technical_doc.md) | 工作流管理完整技术文档 |
| [workflow_types_documentation.md](./workflow_types_documentation.md) | 工作流类型说明文档 |
| [workflow_nodes_documentation.md](./workflow_nodes_documentation.md) | 工作流节点类型详解 |
| [workflow_component_development_guide.md](./workflow_component_development_guide.md) | 自定义节点开发指南 |
| [workflow_node_development_example.md](./workflow_node_development_example.md) | 节点开发实战示例 |
| [workflow_management_sequence_diagram.puml](./workflow_management_sequence_diagram.puml) | 工作流管理时序图（PlantUML） |
| [workflow_component_development_flow.puml](./workflow_component_development_flow.puml) | 组件开发流程图（PlantUML） |


---

📌 **相关文档**
- [Agent 智能体模块](../agent/) - 了解 Agent 如何调用工作流
- [知识库模块](../knowledgeBase/) - 工作流中的知识检索节点
- [工具插件模块](../tools&plugins/) - 工作流中的工具节点
