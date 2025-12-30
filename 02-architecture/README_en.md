# System Architecture

> 🏗️ Dify system-level architecture design and component relationship analysis

## 📋 Directory Overview

This directory contains Dify's **complete system architecture** design documents and architecture diagrams to help you understand Dify's overall design from a macro perspective.

### 📄 Core Documents

- **`dify_componet_architecture.puml`** - Dify complete component architecture (PlantUML source file)
  - Frontend layer: Next.js + React architecture
  - Backend layer: Flask + DDD layered design
  - Core modules: Workflow, Agent, RAG, Model Runtime, etc.
  - Infrastructure: Database, cache, message queue, etc.
  - Third-party integration: LLM, vector database, object storage, etc.

### 🖼️ Architecture Diagram Library (png/)

This directory contains **9 high-definition architecture diagrams** covering the complete system and core scenarios:

#### System-level Architecture Diagrams
1. **`Dify Complete Component Architecture Diagram.png`** ⭐
   - Dify's complete technical architecture
   - Frontend, backend, infrastructure three-layer structure
   - Core modules and dependencies
   - Suitable for: Technology selection, system understanding, team sharing

2. **`Dify Agent Technology Component Architecture Diagram.jpg`**
   - Agent system's technology components
   - Strategy pattern design
   - Tool calling and memory management
   - Suitable for: Understanding Agent architecture

#### Core Scenario Sequence Diagrams
3. **`Dify Workflow Management Core Scenario Sequence Diagram.png`**
   - Workflow creation, orchestration, execution process
   - Frontend-backend interaction sequence
   - Database operation process

4. **`Dify Agent Core Scenario Sequence Diagram.png`**
   - Complete sequence of Agent execution
   - Tool calling process
   - Multi-turn conversation management

#### Document Processing Flow Diagrams
5. **`Dify-PDF Document Processing Sequence Diagram.png`**
   - PDF document upload, parsing, chunking, vectorization process

6. **`Dify-Word Document Processing Sequence Diagram.png`**
   - Word document processing process

7. **`Dify-Excel Document Processing Sequence Diagram.png`**
   - Excel document processing process

8. **`Dify-Notion Document Processing Sequence Diagram.png`**
   - Notion document import process

9. **`Dify-Online Document Import Process.png`**
   - Process of importing online documents via URL

## 🎯 Core Points of Architecture Design

### 1. Layered Architecture

```
┌─────────────────────────────────┐
│  Frontend Layer                 │
│  Next.js 15 + React 19          │
├─────────────────────────────────┤
│  API Gateway Layer              │
│  Flask + RESTful API            │
├─────────────────────────────────┤
│  Business Logic Layer           │
│  DDD + Clean Architecture       │
├─────────────────────────────────┤
│  Data Access Layer              │
│  Repository Pattern             │
├─────────────────────────────────┤
│  Infrastructure Layer           │
│  PostgreSQL + Redis + Celery    │
└─────────────────────────────────┘
```

### 2. Core Modules

- **Workflow Engine**: Visual orchestration and execution
- **Agent System**: Multi-strategy Agent implementation
- **RAG Pipeline**: Complete pipeline from ingestion to retrieval
- **Model Runtime**: Unified multi-model abstraction layer
- **Knowledge Base**: Knowledge base and vector retrieval
- **Observability**: Logging, tracing, monitoring

### 3. Technology Selection

| Layer | Tech Stack | Description |
|-------|------------|-------------|
| Frontend | Next.js 15 + React 19 | App Router architecture |
| Backend | Flask 3.1 + Python 3.11 | Lightweight web framework |
| Architecture Pattern | DDD + Clean Architecture | Domain-driven design |

| Database | PostgreSQL 15+ | Relational database |
| Cache | Redis 7+ | In-memory database |
| Task Queue | Celery + Redis | Asynchronous task processing |
| Vector Database | Weaviate (default) | Supports 28+ types |

### 4. Design Principles

- ✅ **Open-Closed Principle**: Easy to extend without modifying existing code
- ✅ **Single Responsibility**: Clear responsibilities for each module
- ✅ **Dependency Inversion**: Depend on abstractions rather than concrete implementations
- ✅ **Interface Segregation**: Minimize interface dependencies
- ✅ **Composition over Inheritance**: Flexible function composition

## 📖 How to Use These Resources

### For Architects
1. View the **complete component architecture diagram** to understand overall design
2. Read the **PlantUML source file** to understand module relationships
3. Reference design patterns for technology selection

### For Developers
1. View **sequence diagrams** to understand specific processes
2. Locate code positions based on architecture diagrams
3. Understand interaction methods between modules

### For Technical Decision Makers
1. Evaluate the maturity of the tech stack
2. Understand system scalability
3. Assess deployment and operations complexity

## 🔍 Tools for Viewing Architecture Diagrams

### Online Viewing
- **PNG Images**: Open directly with image viewer
- **GitHub**: Preview directly on GitHub

### Editing PlantUML
- **VS Code Plugin**: PlantUML extension
- **Online Tools**: [PlantUML Web Server](http://www.plantuml.com/plantuml)
- **Local Tools**: Graphviz + PlantUML

## 📚 Related Documents

- [01-overview](../01-overview/) - Project overview
- [03-layers/api](../03-layers/api/) - API layer design details
- [04-core-modules](../04-core-modules/) - Core modules in-depth analysis
- [05-data-architecture](../05-data-architecture/) - Data architecture design

---

## 📖 Deeply Understand Design Philosophy

This document provides Dify's **complete architecture diagrams and design descriptions**.

Want to deeply understand **"why design this way"**? We regularly publish:

### 🤔 Design Analysis Series
- **Architecture Evolution**: Dify architecture evolution from v0.1 to v1.0
- **DDD Layering**: Why choose domain-driven design? How to define boundaries?
- **Technology Selection**: Flask vs FastAPI, PostgreSQL vs MongoDB
- **Microservices**: Why didn't Dify adopt microservices architecture?

### 🛠️ Practical Experience Series
- **High Availability Deployment**: Production environment architecture deployment solutions
- **Performance Optimization**: Performance improvement from single machine to distributed
- **Monitoring and Alerting**: Complete observability system setup
- **Containerized Deployment**: Docker + K8s best practices

---

📱 **Follow WeChat Official Account 「柒叔代码阁」**

Regularly publish Dify in-depth content and production practices~

![Official Account QR Code](../qrcode.png)

Not just tell you "what" and "how", but more importantly **"why"** — understanding the thinking behind the design allows you to apply it flexibly.

💬 **Discussion and Exchange**
- GitHub Issue/Discussion: Technical problem discussions
- Official Account Messages