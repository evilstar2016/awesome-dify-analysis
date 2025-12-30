# Agent 智能体模块

## 模块概述

Agent 是 Dify 的智能体系统，采用策略模式支持多种 Agent 类型（Function Calling、ReAct、Plan & Execute）。Agent 能够根据用户需求自动选择和调用工具、执行推理循环，并管理对话记忆，实现复杂的任务自动化。

**核心代码位置**：`api/core/agent/`

## 目录文件说明

| 文件名 | 描述 |
|--------|------|
| [agent_capability_technical_documentation.md](./agent_capability_technical_documentation.md) | Agent 能力完整技术文档 |
| [agent_capability_architecture.puml](./agent_capability_architecture.puml) | Agent 架构图（PlantUML） |
| [agent_capability_sequence_diagram.puml](./agent_capability_sequence_diagram.puml) | Agent 执行时序图（PlantUML） |


---

📌 **相关模块**
- [Workflow 工作流](../workflow/) - Agent 如何触发工作流
- [Tools & Plugins](../tools&plugins/) - Agent 可用的工具
- [Prompt 提示词](../prompt/) - Agent 的提示词设计
