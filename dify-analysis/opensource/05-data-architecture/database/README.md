# Database 数据库架构模块

## 模块概述

Database 是 Dify 的数据持久化层，基于 **PostgreSQL** 设计，采用多租户架构。使用 **SQLAlchemy ORM** 进行数据访问，**Alembic** 管理数据库版本迁移。整体数据架构以 `Tenant` 为顶层实体，支撑应用、对话、知识库、工作流等核心业务域。

**核心代码位置**：`api/models/`、`api/migrations/`

## 架构特点

- 🏢 **多租户设计**：以 Tenant 为顶层实体，实现数据隔离
- 🔐 **细粒度权限**：支持 5 种角色权限管理（owner/admin/editor/normal/dataset_operator）
- 📊 **领域驱动**：按业务域组织数据模型（账户域、应用域、对话域、知识库域等）
- 🔄 **版本迁移**：通过 Alembic 管理数据库 Schema 变更
- 🎯 **性能优化**：合理的索引设计和查询优化策略

## 目录文件说明

| 文件名 | 描述 |
|--------|------|
| [dify_data_architecture.md](./dify_data_architecture.md) | 数据库架构完整技术文档 |
| [dify_data_architecture.puml](./dify_data_architecture.puml) | 数据库 ER 图（PlantUML 源码） |
| [dify_data_architecture_overview.puml](./dify_data_architecture_overview.puml) | 数据库架构概览图（PlantUML 源码） |

---

📌 **相关文档**
- [应用模块](../../04-core-modules/) - 了解应用如何使用数据库
- [知识库模块](../../04-core-modules/knowledgeBase/) - 理解 RAG 数据流
- [工作流模块](../../04-core-modules/workflow/) - 工作流数据持久化
- [系统架构](../../02-architecture/) - 整体架构设计
