# Observability Module

## Module Overview

Observability is Dify's observability system, responsible for tracking and monitoring various behaviors during application runtime. Through asynchronous queues and batch processing mechanisms, it supports full-link tracing for conversations, workflows, LLM calls, tool calls, etc., and can report tracing data to 8 mainstream tracing providers including Langfuse, LangSmith, Opik, Weave, Arize Phoenix, Alibaba Cloud, Tencent Cloud, etc.

**Core Code Location**: `api/core/ops/`

## Directory File Description

| Filename | Description |
|----------|-------------|
| [跟踪系统完整分析.md](./跟踪系统完整分析.md) | Tracing system complete technical analysis documentation |
| [Trace集成示例-Langfuse 集成指南.md](./Trace集成示例-Langfuse%20集成指南.md) | Langfuse tracing provider integration guide |

## Related Modules

- [Agent](../agent/) - Agent execution tracing
- [Workflow](../workflow/) - Workflow execution tracing
- [Tools & Plugins](../tools&plugins/) - Tool call tracing