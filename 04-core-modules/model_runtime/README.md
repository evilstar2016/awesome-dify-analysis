# Model Runtime 模型运行时模块

## 模块概述

Model Runtime 是 Dify 的模型运行时系统，负责统一管理和调用各种 LLM 提供商的模型。通过适配器模式实现了对 100+ 种模型提供商的统一抽象，支持 LLM、Embedding、Rerank、Speech2Text、TTS、Moderation 等多种模型类型。

**核心代码位置**：`api/core/model_runtime/`

## 目录文件说明

| 文件名 | 描述 |
|--------|------|
| [model_runtime_architecture.md](./model_runtime_architecture.md) | 模型运行时架构设计文档 |
| [model_provider_extension_guide.md](./model_provider_extension_guide.md) | 模型提供商扩展开发指南 |

## 相关模块

- [Agent 智能体](../agent/) - Agent 如何调用模型
- [Prompt 提示词](../prompt/) - 模型的提示词构建
- [Workflow 工作流](../workflow/) - 工作流中的模型节点
