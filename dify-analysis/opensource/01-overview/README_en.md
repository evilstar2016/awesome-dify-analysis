# Dify Project Overview

This page provides an overview of Dify's goals, capabilities, architecture, tech stack, and directory structure to help you quickly go from "understanding the project" to "starting development".

## 🎯 Project Goals
- Open-source LLM application development platform: Use intuitive interface and strongly-typed backend to quickly build AI applications from prototype to production.
- Integrate workflow, RAG, agents, model management, and observability, providing APIs for business system integration (Backend-as-a-Service).

## 🔑 Core Capabilities
- Workflow: Build and test complex AI processes on a visual canvas.
- Prompt IDE: Intuitive Prompt creation and variable management.
- RAG Pipeline: Complete capabilities from document ingestion, splitting, embedding to retrieval and reranking.
- Agent: Tool orchestration and execution based on Function Calling / ReAct.
- Model Management: Support hundreds of proprietary/open-source LLMs, seamless switching and load balancing.
- LLMOps: Logging and performance monitoring, error tracking and observability.
- SDK and Open API: Convenient for external system integration and automation.

## 🏗️ Architecture Overview
- Backend: Python Flask + DDD/Clean Architecture, PostgreSQL, Redis, Celery, Gunicorn.
- Frontend: Next.js + React + TypeScript, Tailwind CSS, pnpm.
- Deployment: Docker containerization, Nginx reverse proxy; supports multi-cloud environments.
- Vector Database: Default Weaviate, also supports Qdrant, Milvus, etc.

## 🧰 Tech Stack (Key Versions)
- Backend: Python 3.11~3.12, Flask 3.1, uv package management, Celery, Redis.
- Frontend: TypeScript, Next.js 15, React 19, Tailwind CSS, pnpm.
- Toolchain: Ruff / MyPy (backend), ESLint + Prettier / TypeScript (frontend), pytest / Jest (testing).
- Monitoring: Sentry, OpenTelemetry (integrated as needed).

## 📁 Repository Structure (Excerpt)
```
dify/
├── api/        # Backend API service (Flask, DDD)
├── web/        # Frontend web application (Next.js App Router)
├── docker/     # Docker & Compose configuration
├── dev/        # Development and script tools
├── sdks/       # Multi-language SDKs
└── analysis/   # Project analysis documentation
```

Backend (api) layering example: controllers (API endpoints), core (domain logic: workflow / rag / agent / model_runtime, etc.), services, models, repositories, libs, extensions, migrations, tests.

## 🔍 Core Modules Quick View
- Workflow Engine: Visual orchestration and execution, supports single-step/free node execution.
- RAG System: Extractor / Splitter / Embedding / Retrieval / Rerank full pipeline.
- Agent: FunctionCallAgentRunner and other runners with tool call management.
- Model Runtime: Unified model abstraction and load-balanced model management.
- Observability and Permission: Classified as core modules for parallel maintenance and retrieval with business modules.

For more details, see the following directories:
- [analysis/04-core-modules](../04-core-modules) (Agent, Workflow, Prompt, KnowledgeBase, Model Runtime, Observability, Permission)
- [analysis/03-layers/api](../03-layers/api) (API architecture and sequence diagrams)

## 🚀 Development and Quality
- Backend (API):
  - Run: `uv run --project api <command>`
  - Quality checks: `make lint`, `make type-check`, `uv run --project api --dev dev/pytest/pytest_unit_tests.sh`
- Frontend (Web):
  - Quality: `pnpm lint`, `pnpm lint:fix`, `pnpm test`

## 📦 Deployment and Running (Example)
- Using Docker Compose: See [docker/](../../docker) and `docker-compose.yaml`.
- Reverse proxy and certificates: See [docker/nginx](../../docker/nginx) and `certbot/`.

---

📱 **Follow WeChat Official Account 「柒叔代码阁」**

Regularly publish Dify in-depth content and production practices~

![Official Account QR Code](../qrcode.png)

💡 **Why follow?**

Not just tell you "what" and "how", but more importantly **"why"** — understanding the thinking behind the design allows you to apply it flexibly.

💬 **Discussion and Exchange**
- GitHub Issue/Discussion: Technical problem discussions
- Official Account Messages