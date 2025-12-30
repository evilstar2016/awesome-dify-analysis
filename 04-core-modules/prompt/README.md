# Prompt 提示词模块

## 模块概述

Prompt 是 Dify 的提示词管理与转换系统，负责构建和转换发送给 LLM 的提示词。通过模板方法模式实现了 Simple、Advanced、Agent 三种提示词转换器，支持 Jinja2 模板、变量注入、对话历史管理等功能，适配不同应用场景（Chatbot、Completion、Workflow、Agent）。

**核心代码位置**：`api/core/prompt/`

## 目录文件说明

| 文件名 | 描述 |
|--------|------|
| [dify_prompt_management_documentation.md](./dify_prompt_management_documentation.md) | Prompt 管理模块完整技术文档 |
| [prompt_module_class_diagram.puml](./prompt_module_class_diagram.puml) | 模块类图（PlantUML） |
| [simple_prompt_transform_sequence.puml](./simple_prompt_transform_sequence.puml) | SimplePromptTransform 时序图 |
| [advanced_prompt_transform_sequence.puml](./advanced_prompt_transform_sequence.puml) | AdvancedPromptTransform 时序图 |
| [agent_history_prompt_transform_sequence.puml](./agent_history_prompt_transform_sequence.puml) | AgentHistoryPromptTransform 时序图 |
| [prompt_template_parser_sequence.puml](./prompt_template_parser_sequence.puml) | PromptTemplateParser 解析流程时序图 |
| [prompt_module_integration_sequence.puml](./prompt_module_integration_sequence.puml) | Prompt 模块与应用层集成时序图 |

## 相关模块

- [Agent 智能体](../agent/) - Agent 的提示词设计
- [Model Runtime](../model_runtime/) - 模型调用与提示词
- [Workflow 工作流](../workflow/) - 工作流中的提示词节点
