# Workflow Module

## Module Overview

Workflow is Dify's workflow engine, using event-driven architecture, supporting flexible node orchestration and asynchronous execution. Provides rich built-in node types (LLM, knowledge base retrieval, tool calling, conditional branching, etc.), supports custom node development, and achieves complex business process automation.

**Core Code Location**: `api/core/workflow/`

## Directory File Description

| Filename | Description |
|----------|-------------|
| [workflow_management_technical_doc.md](./workflow_management_technical_doc.md) | Workflow management complete technical documentation |
| [workflow_types_documentation.md](./workflow_types_documentation.md) | Workflow types documentation |
| [workflow_nodes_documentation.md](./workflow_nodes_documentation.md) | Workflow node types detailed explanation |
| [workflow_component_development_guide.md](./workflow_component_development_guide.md) | Custom node development guide |
| [workflow_node_development_example.md](./workflow_node_development_example.md) | Node development practical example |
| [workflow_management_sequence_diagram_en.puml](./workflow_management_sequence_diagram_en.puml) | Workflow management sequence diagram (PlantUML) |
| [workflow_component_development_flow_en.puml](./workflow_component_development_flow_en.puml) | Component development flow diagram (PlantUML) |

---

📌 **Related Documentation**
- [Agent Module](../agent/) - Learn how Agent calls workflows
- [Knowledge Base Module](../knowledgeBase/) - Knowledge retrieval nodes in workflows
- [Tools & Plugins Module](../tools&plugins/) - Tool nodes in workflows