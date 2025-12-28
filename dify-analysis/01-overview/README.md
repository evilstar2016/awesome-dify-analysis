# Dify 项目概览（Overview）

本页概述 Dify 的目标、能力、架构、技术栈与目录结构，帮助你快速从“了解项目”到“开始开发”。

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
# Dify：一本入门读物 风格的项目概览

这不是一份枯燥的技术手册，而是一本短小的导读，带你用通俗的语言读懂 Dify：它为什么存在、能做什么、核心部件如何协同，以及如何开始动手。

如果你只是想快速把握要点，先读“快速航标”；如果想深入实践，请沿着“下一步阅读”中的目录继续探索。

---

## 快速航标（1 分钟读完）
- **目标**：为开发者提供一套完整的 LLM 应用开发平台，覆盖从提示工程、检索增强生成（RAG）、智能体编排到模型管理与可观测性。
- **适合谁**：产品经理想验证 AI 功能；工程师需要可复用的 LLM 运行时；研究者想把模型快速部署成服务。
- **核心构成**：工作流画布、Prompt IDE、RAG 管道、Agent 编排、模型运行时、LLMOps 与 SDK/API。

---

## 以故事说清楚：为什么需要 Dify？

想象一个场景：你要把一个新型 LLM 功能接入客服、文档问答和自动化流程。每个场景都需要不同的提示、检索策略、工具调用顺序、模型选择与监控。传统方法是为每个场景重写逻辑、搭建单独管道。

Dify 的价值就在于把这些共同问题抽象成可视化的构件与运行时：设计一次工作流、复用提示、在运行时切换模型并统一观察行为。它把“实验→生产”的成本降下来，让团队更多精力用在产品优化上，而不是重复构建基础设施。

---

## 核心概念——像讲故事一样理解技术

- **工作流（Workflow）**：把复杂的交互拆成节点，把节点以可视化方式连接。你可以在画布上调试每一步，像搭积木一样组合能力。
- **提示词 IDE（Prompt IDE）**：把提示从散落的字符串变成可管理的模块，支持变量、版本和测试用例，便于复用与审计。
- **RAG（检索增强生成）管道**：从文档提取、分片、生成向量、检索与重排，形成一条可观测的流水线，提升答案的准确性与可解释性。
- **智能体（Agent）**：将模型的“对话能力”与外部工具（数据库、搜索引擎、执行器）结合，支持 Function Calling / ReAct 风格的策略，实现复杂任务自动化。
- **模型运行时（Model Runtime）**：统一抽象不同模型，做负载均衡、超时、重试与模型替换，使上层业务无需关心底层模型差异。
- **LLMOps（可观测性）**：收集提示、模型调用、延迟、成本与错误，支持追溯与报警，帮助工程团队快速定位问题。

---

## 架构一瞥（用最少的术语说明系统如何组织）

- **后端**：基于 Python 的后端服务（Flask 风格的组织，采用 DDD / Clean 架构思想），数据库使用 PostgreSQL，缓存与队列用 Redis 与 Celery，进程由 Gunicorn 管理。
- **前端**：使用 Next.js + React（TypeScript）构建交互式 UI，提供工作流画布与 Prompt IDE。
- **部署**：容器化（Docker / Docker Compose），常见于云环境或私有部署；Nginx 做反向代理与证书管理。
- **向量存储**：默认集成 Weaviate，亦支持 Qdrant、Milvus 等流行向量数据库，便于做向量检索与相似度搜索。

想看源码层级与模块位于何处？请参阅仓库摘录：

```
dify/
├── api/        # 后端 API 服务（领域层、服务层、适配层）
├── web/        # 前端 Web 应用（Next.js）
├── docker/     # 容器与部署配置
├── dev/        # 本地开发与 CI 脚本
├── sdks/       # 多语言 SDK
└── analysis/   # 本份分析文档
```

后端常见分层：`controllers` → `services` → `core/domain` → `repositories`，并配套 `libs`、`extensions` 与 `migrations`。

---

## 常见问题（以产品视角回答）

- **Q：我需要多少工程投入才能试用？**
  - A：基础试用只需运行后端与向量数据库，搭建一个小型文档索引即可验证 RAG 效果，通常数小时到一天。
- **Q：能否替换模型或接入私有模型？**
  - A：支持接入多种模型提供者与本地/私有部署的模型，通过统一运行时抽象替换非常方便。
- **Q：如何保证安全与审计？**
  - A：提示、调用与结果可以记录到日志，配合权限模块与审计链路可以满足合规需求。

---

## 典型应用场景（几行话描述）

- 文档问答：将企业文档做向量化，支持自然语言问答与证据回溯。
- 智能流程机器人：基于 Agent 调度外部任务（下单、查询、执行脚本）。
- 嵌入式产品功能：把模型能力作为后端服务，暴露给移动端或第三方系统。

---

## 开始动手（最小可行步骤）

1. 克隆仓库并阅读此目录：参见 [analysis/04-core-modules](../04-core-modules) 与 [analysis/03-layers/api](../03-layers/api)。
2. 启动依赖（示例用 Docker Compose）：

```bash
# 在仓库根目录运行（示例）
docker compose -f docker/docker-compose.yaml up -d
```

3. 启动后端与前端开发服务器：后端参考 `api/` 下的启动脚本，前端使用 `pnpm dev`。

4. 按需接入向量数据库（Weaviate/Qdrant）并导入样例文档，验证 RAG 流程。

---

## 下一步阅读（按兴趣分道）

- 系统设计与模块：参见 [analysis/04-core-modules](../04-core-modules)
- API 层与序列图：参见 [analysis/03-layers/api](../03-layers/api)
- 部署样例：参见 [docker/](../../docker)

---

## 参考与索引

- 技术栈、测试与 LLMOps 细节可见本仓库的相关分析文章与脚本。

— 最后更新：2025-12-25
