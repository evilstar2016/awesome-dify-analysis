# Dify 项目概览（Overview）

本页概述 Dify 的目标、能力、架构、技术栈与目录结构，帮助你快速从“了解项目”到“开始开发”。内容依据 Serena 项目记忆整理（project_overview / project_structure / tech_stack / dify_backend_architecture_analysis / dify_backend_api_analysis）。

## 🎯 项目目标
- 开源的 LLM 应用开发平台：用直观界面与强类型后端，快速构建从原型到生产的 AI 应用。
- 集成工作流、RAG、智能体、模型管理与可观测性，提供 API 以便业务系统集成（Backend-as-a-Service）。

## 🔑 核心能力
- 工作流（Workflow）：在可视化画布上搭建与测试复杂 AI 流程。
- 提示词 IDE：直观的 Prompt 制作与变量管理。
- RAG 管道：从文档摄取、切分、嵌入到检索与重排的完整能力。
- 智能体（Agent）：基于 Function Calling / ReAct 的工具编排与执行。
- 模型管理：支持数百个专有/开源 LLM，无缝切换与负载均衡。
- LLMOps：日志与性能监控，错误追踪与可观测性。
- SDK 与开放 API：便于外部系统集成与自动化。

## 🏗️ 架构概览
- 后端：Python Flask + DDD/Clean Architecture，PostgreSQL、Redis、Celery，Gunicorn。
- 前端：Next.js + React + TypeScript，Tailwind CSS，pnpm。
- 部署：Docker 容器化，Nginx 反向代理；支持多云环境。
- 向量数据库：默认 Weaviate，亦支持 Qdrant、Milvus 等。

## 🧰 技术栈（关键版本）
- 后端：Python 3.11~3.12、Flask 3.1、uv 包管理、Celery、Redis。
- 前端：TypeScript、Next.js 15、React 19、Tailwind CSS、pnpm。
- 工具链：Ruff / MyPy（后端），ESLint + Prettier / TypeScript（前端），pytest / Jest（测试）。
- 监控：Sentry、OpenTelemetry（按需集成）。

## 📁 仓库结构（摘录）
```
dify/
├── api/        # 后端 API 服务（Flask, DDD）
├── web/        # 前端 Web 应用（Next.js App Router）
├── docker/     # Docker & Compose 配置
├── dev/        # 开发与脚本工具
├── sdks/       # 多语言 SDK
└── analysis/   # 项目分析文档
```

后端（api）分层示例：controllers（API端点）、core（领域逻辑：workflow / rag / agent / model_runtime 等）、services、models、repositories、libs、extensions、migrations、tests。

## 🔍 核心模块速览
- 工作流引擎：可视化编排与执行，支持单步/自由节点运行。
- RAG 系统：Extractor / Splitter / Embedding / Retrieval / Rerank 全链路。
- 智能体：FunctionCallAgentRunner 等运行器与工具调用管理。
- 模型运行时：统一模型抽象与负载均衡模型管理。
- 可观测性与权限：归类为核心模块，便于与业务模块并行维护与检索。

更多细节可在以下目录查看：
- [analysis/04-core-modules](../04-core-modules)（Agent、Workflow、Prompt、KnowledgeBase、Model Runtime、Observability、Permission）
- [analysis/03-layers/api](../03-layers/api)（API 架构与序列图）

## 🚀 开发与质量
- 后端（API）：
  - 运行：`uv run --project api <command>`
  - 质量检查：`make lint`、`make type-check`、`uv run --project api --dev dev/pytest/pytest_unit_tests.sh`
- 前端（Web）：
  - 质量：`pnpm lint`、`pnpm lint:fix`、`pnpm test`

## 📦 部署与运行（示例）
- 使用 Docker Compose：见 [docker/](../../docker) 与 `docker-compose.yaml`。
- 反向代理与证书：见 [docker/nginx](../../docker/nginx) 与 `certbot/`。

## 📎 参考与延伸
- 项目结构与架构分析：Serena 记忆（project_structure、dify_backend_architecture_analysis）。
- API 控制器与路由：Serena 记忆（dify_backend_api_analysis）。
- 技术栈与工具链：Serena 记忆（tech_stack）。

— 最后更新：2025-12-25
