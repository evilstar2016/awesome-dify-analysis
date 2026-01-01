# Prompt Module

## Module Overview

Prompt is Dify's prompt management and transformation system, responsible for constructing and transforming prompts sent to LLMs. Through template method pattern, it implements Simple, Advanced, Agent three types of prompt transformers, supports Jinja2 templates, variable injection, conversation history management, etc., and adapts to different application scenarios (Chatbot, Completion, Workflow, Agent).

**Core Code Location**: `api/core/prompt/`

## Directory File Description

| Filename | Description |
|----------|-------------|
| [dify_prompt_management_documentation.md](./dify_prompt_management_documentation.md) | Prompt management module complete technical documentation |
| [prompt_module_class_diagram_en.puml](./prompt_module_class_diagram_en.puml) | Module class diagram (PlantUML) |
| [simple_prompt_transform_sequence_en.puml](./simple_prompt_transform_sequence_en.puml) | SimplePromptTransform sequence diagram |
| [advanced_prompt_transform_sequence_en.puml](./advanced_prompt_transform_sequence_en.puml) | AdvancedPromptTransform sequence diagram |
| [agent_history_prompt_transform_sequence_en.puml](./agent_history_prompt_transform_sequence_en.puml) | AgentHistoryPromptTransform sequence diagram |
| [prompt_template_parser_sequence_en.puml](./prompt_template_parser_sequence_en.puml) | PromptTemplateParser parsing process sequence diagram |
| [prompt_module_integration_sequence_en.puml](./prompt_module_integration_sequence_en.puml) | Prompt module and application layer integration sequence diagram |

## Related Modules

- [Agent](../agent/) - Agent's prompt design
- [Model Runtime](../model_runtime/) - Model calling and prompts
- [Workflow](../workflow/) - Prompts in workflow nodes