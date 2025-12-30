# Core Modules

> ⚙️ Dify core functional modules architecture design and implementation analysis

## 📋 Directory Overview

This directory contains in-depth technical analysis of Dify's **8 major core modules**, each with independent documentation and architecture diagrams.

## 🗂️ Module List

### 🔄 [Workflow Engine](./workflow/)
**Event-driven workflow orchestration and execution system**

Core Capabilities:
- Visual workflow orchestration
- Multiple node types (LLM, knowledge retrieval, code execution, condition judgment, etc.)
- Event-driven architecture
- Asynchronous execution and state management
- Support for serial, parallel, conditional branching

**Recommended Reading**:
- [Workflow Management Technical Documentation](./workflow/workflow_management_technical_doc.md)
- [Workflow Types Documentation](./workflow/workflow_types_documentation.md)
- [Node Development Guide](./workflow/workflow_component_development_guide.md)

---

### 🤖 [Agent System](./agent/)
**Multi-strategy Agent implementation and tool calling mechanism**

Core Capabilities:
- Strategy Pattern: Function Calling, ReAct, Plan & Execute
- Tool auto-discovery and calling
- Chain of Thought
- Multi-turn conversation management
- Memory system (short-term + long-term)

**Design Highlights**:
- Uses strategy pattern, easy to extend new Agent types
- Runtime dynamic strategy switching
- Decorator pattern to enhance Agent capabilities

---

### 📚 [Knowledge Base System](./knowledgeBase/)
**Complete RAG (Retrieval-Augmented Generation) implementation**

Core Capabilities:
- Multi-format document parsing (PDF, Word, Markdown, TXT, HTML)
- Intelligent text chunking (fixed length, semantic chunking, recursive chunking)
- Vectorization and index building
- Multiple retrieval strategies (semantic retrieval, keyword retrieval, hybrid retrieval)
- Re-ranking optimization

**Tech Stack**:
- Supports 28+ vector databases
- Multiple Embedding models
- Document processing pipeline (Extractor → Splitter → Embedding → Retrieval)

---

### 🔌 [Model Runtime](./model_runtime/)
**Unified multi-model abstraction layer**

Core Capabilities:
- Unified model calling interface
- Supports 40+ mainstream LLM providers
- Load balancing and routing
- Token calculation and optimization
- Error handling and retry mechanism

**Design Patterns**:
- Factory Pattern: Create different model providers
- Adapter Pattern: Unify different model APIs
- Strategy Pattern: Different calling strategies

---

### 💬 [Prompt Engineering](./prompt/)
**Prompt management and optimization system**

Core Capabilities:
- Prompt template system
- Variable management and injection
- Multi-turn conversation context management
- Prompt version control
- Prompt optimization suggestions

**Application Scenarios**:
- Application-level Prompt management
- Agent Prompt construction
- Workflow node Prompt configuration

---

### 🛠️ [Tools & Plugins System](./tools&plugins/)
**Tool calling and plugin extension mechanism**

Core Capabilities:
- Tool auto-registration and discovery
- Parameter validation and type checking
- Tool execution and result parsing
- Custom tool development
- Tool marketplace (community contributions)

**Built-in Tools**:
- API calling tools
- Database query tools
- File operation tools
- Code execution tools

---

### 📊 [Observability System](./observability/)
**Complete logging, tracing, monitoring solution**

Core Capabilities:
- Structured Logging
- Distributed Tracing
- Metrics Collection
- Error Tracking
- Performance Profiling

**Technology Integration**:
- OpenTelemetry standard
- Sentry error monitoring
- Custom logging system

---

### 🔐 [Permission Management System](./permission/)
**RBAC permission model and access control**

Core Capabilities:
- Role-Based Access Control (RBAC)
- Resource-level permission management
- Multi-tenant isolation
- API-level permission checking
- Fine-grained permission control

**Permission Model**:
- User
- Role
- Permission
- Resource

---

## 🏗️ Inter-Module Relationships

```
┌─────────────────────────────────────────────────────┐
│                   Application Layer                 │
│  Workflow / Agent / Knowledge Base                  │
└───────────┬─────────────────────────┬───────────────┘
            │                         │
            ↓                         ↓
┌───────────────────┐     ┌──────────────────────┐
│  Model Runtime    │     │  Tools & Plugins     │
│  (Model Abstract) │     │  (Tool Calling)      │
└───────────────────┘     └──────────────────────┘
            │                         │
            └──────────┬──────────────┘
                       ↓
            ┌──────────────────┐
            │  Prompt Engine   │
            │  (Prompt Mgmt)   │
            └──────────────────┘
                       │
        ┌──────────────┴──────────────┐
        ↓                             ↓
┌──────────────┐            ┌─────────────────┐
│ Observability│            │   Permission    │
│ (Observability)│          │   (Permission)  │
└──────────────┘            └─────────────────┘
```

## 🎯 Core Design Principles

### 1. Modular Design
- Each module has single responsibility, clear boundaries
- Modules communicate through interfaces, reducing coupling
- Support independent development, testing, deployment

### 2. Extensibility
- Use design patterns like strategy pattern, factory pattern
- Support plugin extension
- Easy to add new features without modifying existing code

### 3. High Cohesion, Low Coupling
- Related functions aggregated in the same module
- Minimize dependencies between modules
- Decouple through events or messages

### 4. Domain-Driven Design (DDD)
- Core modules correspond to business domains
- Clear domain boundaries
- Rich domain models

## 📚 Reading Suggestions

### For Beginners
1. First read [Workflow](./workflow/) to understand workflow concepts
2. Then look at [Agent](./agent/) to understand Agent implementation
3. Then learn [Knowledge Base](./knowledgeBase/) to master RAG

### For Architects
1. Focus on **design patterns** of each module
2. Understand **dependencies** between modules
3. Learn Dify's **architecture evolution** process

### For Developers
1. View detailed documentation of modules of interest
2. Read core code implementation
3. Reference examples for secondary development

## 🔗 Quick Navigation

| Module | Core Document | Keywords |
|--------|---------------|----------|
| [Workflow](./workflow/) | Workflow management, node development | `Workflow` or `工作流` |
| [Agent](./agent/) | Strategy pattern, tool calling | `Agent` or `智能体` |
| [Knowledge Base](./knowledgeBase/) | RAG, vector retrieval | `知识库` or `RAG` |
| [Model Runtime](./model_runtime/) | Model abstraction, load balancing | `模型运行时` or `模型` |
| [Prompt](./prompt/) | Template management, variable injection | `Prompt` or `提示词` |
| [Tools & Plugins](./tools&plugins/) | Tool registration, plugin development | `工具` or `插件` |
| [Observability](./observability/) | Logging, tracing, monitoring | `可观测性` or `监控` |
| [Permission](./permission/) | RBAC, permission control | `权限` or `RBAC` |

---

## 📖 Deeply Understand Design Philosophy

This directory provides an **overview of 8 major core modules**.

Want to deeply understand **"why design this way"**? We regularly publish:

### 🤔 Design Analysis Series
- **Why did Agent choose strategy pattern?** - Trade-off analysis of 3 solutions
- **Workflow's event-driven architecture evolution** - From v0.1 to v1.0
- **Why doesn't Knowledge Base use LangChain's RAG?** - Technology selection considerations
- **Model Runtime's unified abstraction design** - How to support 40+ models
- **Prompt template system's 3 reconstructions** - Architecture evolution journey
- **Tool calling security sandbox design** - Balancing flexibility and security
- **Why choose OpenTelemetry for observability?** - Standardization selection
- **Permission system evolution: From simple to enterprise-level** - RBAC practice

### 🛠️ Practical Experience Series
- **3 scenarios of Workflow execution timeout** - With complete solutions
- **Agent stuck in infinite loop: Detection and circuit breaker** - Production issue practice
- **7 reasons for inaccurate Knowledge Base retrieval** - Tuning guide
- **Rate limiting strategy for large model Token overrun** - Cost control
- **Large document processing OOM optimization** - Memory management practice
- **Vector database performance comparison real test** - Selection reference
- **5 big pitfalls of multi-tenant data isolation** - Enterprise-level must-read
- **Production deployment Checklist** - 21 key configurations

---

📱 **Follow WeChat Official Account 「柒叔代码阁」**

Regularly publish Dify in-depth content and production practices~

![Official Account QR Code](../qrcode.png)

💡 **Why follow?**

Not just tell you "what" and "how", but more importantly **"why"** — understanding the thinking behind the design allows you to apply it flexibly.

💬 **Discussion and Exchange**
- GitHub Issue/Discussion: Technical problem discussions
- Official Account Messages