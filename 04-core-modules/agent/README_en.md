# Agent Module

## Module Overview

Agent is Dify's intelligent agent system, using strategy pattern to support multiple Agent types (Function Calling, ReAct, Plan & Execute). Agent can automatically select and call tools based on user needs, execute reasoning loops, and manage conversation memory to achieve complex task automation.

**Core Code Location**: `api/core/agent/`

## Directory File Description

| Filename | Description |
|----------|-------------|
| [agent_capability_technical_documentation.md](./agent_capability_technical_documentation.md) | Agent capability complete technical documentation |
| [agent_capability_architecture_en.puml](./agent_capability_architecture_en.puml) | Agent architecture diagram (PlantUML) |
| [agent_capability_sequence_diagram_en.puml](./agent_capability_sequence_diagram_en.puml) | Agent execution sequence diagram (PlantUML) |

---

📌 **Related Modules**
- [Workflow](../workflow/) - How Agent triggers workflows
- [Tools & Plugins](../tools&plugins/) - Tools available to Agent
- [Prompt](../prompt/) - Agent's prompt design