# Observability 可观测性模块

## 模块概述

Observability 是 Dify 的可观测性系统，负责追踪和监控应用运行时的各种行为。通过异步队列和批处理机制，支持对话、工作流、LLM 调用、工具调用等全链路追踪，并可将追踪数据上报到 Langfuse、LangSmith、Opik、Weave、Arize Phoenix、阿里云、腾讯云等 8 种主流追踪提供商。

**核心代码位置**：`api/core/ops/`

## 目录文件说明

| 文件名 | 描述 |
|--------|------|
| [跟踪系统完整分析.md](./跟踪系统完整分析.md) | 追踪系统完整技术分析文档 |
| [Trace集成示例-Langfuse 集成指南.md](./Trace集成示例-Langfuse%20集成指南.md) | Langfuse 追踪提供商集成指南 |

## 相关模块

- [Agent 智能体](../agent/) - Agent 执行追踪
- [Workflow 工作流](../workflow/) - 工作流执行追踪
- [Tools & Plugins](../tools&plugins/) - 工具调用追踪
