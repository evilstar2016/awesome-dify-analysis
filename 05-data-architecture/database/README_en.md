# Database Module

## Module Overview

Database is Dify's data persistence layer, designed based on **PostgreSQL**, using multi-tenant architecture. Uses **SQLAlchemy ORM** for data access, **Alembic** for database version migration management. The overall data architecture uses `Tenant` as the top-level entity, supporting core business domains such as applications, conversations, knowledge bases, workflows, etc.

**Core Code Location**: `api/models/`, `api/migrations/`

## Architecture Features

- 🏢 **Multi-tenant Design**: Uses Tenant as top-level entity to achieve data isolation
- 🔐 **Fine-grained Permissions**: Supports 5 types of role permission management (owner/admin/editor/normal/dataset_operator)
- 📊 **Domain-driven**: Organizes data models by business domains (account domain, application domain, conversation domain, knowledge base domain, etc.)
- 🔄 **Version Migration**: Manages database Schema changes through Alembic
- 🎯 **Performance Optimization**: Reasonable index design and query optimization strategies

## Directory File Description

| Filename | Description |
|----------|-------------|
| [dify_data_architecture.md](./dify_data_architecture.md) | Database architecture complete technical documentation |
| [dify_data_architecture.puml](./dify_data_architecture.puml) | Database ER diagram (PlantUML source code) |
| [dify_data_architecture_overview.puml](./dify_data_architecture_overview.puml) | Database architecture overview diagram (PlantUML source code) |

---

📌 **Related Documentation**
- [Application Module](../../04-core-modules/) - Learn how applications use the database
- [Knowledge Base Module](../../04-core-modules/knowledgeBase/) - Understand RAG data flow
- [Workflow Module](../../04-core-modules/workflow/) - Workflow data persistence
- [System Architecture](../../02-architecture/) - Overall architecture design

---

📱 **Follow WeChat Official Account 「柒叔代码阁」**

Regularly publish Dify in-depth content and production practices~

![Official Account QR Code](../../qrcode.png)

💡 **Why follow?**

Not just tell you "what" and "how", but more importantly **"why"** — understanding the thinking behind the design allows you to apply it flexibly.

💬 **Discussion and Exchange**
- GitHub Issue/Discussion: Technical problem discussions
- Official Account Messages