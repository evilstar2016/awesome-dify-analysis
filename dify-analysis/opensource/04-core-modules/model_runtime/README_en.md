# Model Runtime Module

## Module Overview

Model Runtime is Dify's model runtime system, responsible for unified management and calling of various LLM provider models. Through adapter pattern, it achieves unified abstraction for 100+ model providers, supporting multiple model types such as LLM, Embedding, Rerank, Speech2Text, TTS, Moderation, etc.

**Core Code Location**: `api/core/model_runtime/`

## Directory File Description

| Filename | Description |
|----------|-------------|
| [model_runtime_architecture.md](./model_runtime_architecture.md) | Model runtime architecture design documentation |
| [model_provider_extension_guide.md](./model_provider_extension_guide.md) | Model provider extension development guide |

## Related Modules

- [Agent](../agent/) - How Agent calls models
- [Prompt](../prompt/) - Model's prompt construction
- [Workflow](../workflow/) - Model nodes in workflows