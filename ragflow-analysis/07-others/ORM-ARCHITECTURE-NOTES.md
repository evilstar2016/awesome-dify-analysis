# RAGFlow 双ORM架构说明

## 📌 核心发现

RAGFlow 采用**双ORM架构设计**，这是一个**合理的架构决策**，而非技术债务。

---

## 🎯 架构概览

```
┌─────────────────────────────────────────┐
│        应用层 (Application Layer)        │
│                                          │
│    api/apps/  (使用 Peewee ORM)         │
│    └── User, Tenant, Knowledgebase,     │
│        Document, Dialog, Conversation   │
│                                          │
│    连接到: MySQL/PostgreSQL (业务数据)   │
└─────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────┐
│      向量检索层 (Vector Search Layer)    │
│                                          │
│  rag/utils/ob_conn.py                   │
│  (使用 SQLAlchemy + pyobvector)          │
│    └── 向量表结构定义                     │
│        向量搜索、全文搜索                  │
│                                          │
│  连接到: OceanBase (向量数据+文档chunks)  │
└─────────────────────────────────────────┘
```

---

## 🔑 两种ORM的职责分工

| ORM | 使用位置 | 用途 | 特点 |
|-----|---------|------|------|
| **Peewee** | `api/db/` | 业务数据模型 | 轻量、简单、适合CRUD |
| **SQLAlchemy** | `rag/utils/ob_conn.py` | 向量数据库 | 强大、灵活、支持复杂SQL |

---

## 📂 代码位置

### 1. Peewee ORM（应用层）

**文件**: `api/db/db_models.py`

```python
from peewee import CharField, DateTimeField, BooleanField, TextField

class User(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    nickname = CharField(max_length=100, ...)
    email = CharField(max_length=255, ...)
    # ...
```

**服务层**: `api/db/services/*.py`
- `user_service.py`
- `knowledgebase_service.py`
- `document_service.py`
- `dialog_service.py`
- `conversation_service.py`
- 等20+个服务类

**管理的模型**:
- User（用户）
- Tenant（租户）
- Knowledgebase（知识库）
- Document（文档）
- Dialog（对话）
- Conversation（会话）
- Task（任务）
- APIToken（API令牌）
- 等

---

### 2. SQLAlchemy（向量层）

**文件**: `rag/utils/ob_conn.py`

```python
from sqlalchemy import text, Column, String, Integer, JSON, Double
from sqlalchemy.dialects.mysql import LONGTEXT, TEXT

# 定义向量数据表结构
column_definitions: list[Column] = [
    Column("id", String(256), primary_key=True),
    Column("kb_id", String(256), nullable=False, index=True),
    Column("content_with_weight", LONGTEXT, ...),
    Column("q_768_vec", VECTOR, ...),  # 向量字段
    ...
]
```

**核心类**: `OBConnection`
- 使用 `pyobvector.ObVecClient`（内部基于SQLAlchemy）
- 执行向量搜索、全文搜索
- 管理OceanBase向量索引

---

## 💡 为什么使用双ORM？

### Peewee的优势（应用层）
✅ **轻量级** - 学习曲线平缓，代码简洁  
✅ **高效CRUD** - 适合频繁的增删改查操作  
✅ **连接池简单** - PooledMySQLDatabase 开箱即用  
✅ **代码可读性** - 查询语法直观

**示例**:
```python
# Peewee 简洁的查询
users = User.select().where(User.tenant_id == tenant_id).paginate(page, size)
```

---

### SQLAlchemy的优势（向量层）
🎯 **强大的SQL能力** - 支持复杂的原生SQL  
🎯 **向量扩展集成** - pyobvector 内部依赖 SQLAlchemy  
🎯 **Column定义灵活** - 可定义复杂的向量、数组、JSON字段  
🎯 **低层控制** - text() 方法执行原生SQL

**示例**:
```python
# SQLAlchemy 执行复杂的向量搜索
fulltext_search = "MATCH (content) AGAINST ('query' IN NATURAL LANGUAGE MODE)"
vector_search = "cosine_distance(q_768_vec, '[0.1, 0.2, ...]')"
```

---

## 🔄 数据流示例

### 1. 用户创建知识库（使用Peewee）
```python
# api/db/services/knowledgebase_service.py
from api.db.db_models import Knowledgebase

kb = Knowledgebase.create(
    id=kb_id,
    tenant_id=tenant_id,
    name="我的知识库",
    ...
)
```

### 2. 文档分块后存入向量库（使用SQLAlchemy）
```python
# rag/utils/ob_conn.py
from sqlalchemy import text

# 通过 ObVecClient 插入向量数据
self.client.perform_raw_vector_search(
    table_name=table_name,
    query_vector=embedding,
    ...
)
```

---

## 📊 配置初始化

**文件**: `common/settings.py` (第253-255行)

```python
# 根据配置选择向量数据库
elif lower_case_doc_engine == "oceanbase":
    OB = get_base_config("oceanbase", {})
    docStoreConn = rag.utils.ob_conn.OBConnection()  # ← 使用SQLAlchemy
```

---

## ✅ 架构优势

1. **职责清晰**
   - 应用层专注业务逻辑
   - 向量层专注检索性能

2. **技术适配**
   - 轻量ORM适合CRUD
   - 强大ORM适合复杂查询

3. **可维护性**
   - 两个ORM不冲突（操作不同数据库）
   - 没有循环依赖

4. **性能优化**
   - Peewee连接池管理MySQL/PostgreSQL
   - SQLAlchemy+pyobvector优化向量检索

---

## 🚀 最佳实践

这种**双ORM架构**在以下场景中是**业界最佳实践**：

- ✅ 微服务架构（不同服务使用不同ORM）
- ✅ 多数据源系统（业务库 + 向量库）
- ✅ 性能敏感应用（轻量ORM + 强大ORM）
- ✅ 遗留系统集成（新旧ORM并存）

---

## 📝 已修复的文档

根据调查结果，已修复以下文档：

1. ✅ `anylisis/01-overview/project-overview.md`
   - 更新目录树注释
   - 添加双ORM架构说明
   - 在依赖包中明确标注两种ORM用途

2. ✅ `anylisis/02-architecture/system-architecture.md`
   - 更新数据访问层组件说明
   - 更新架构图中的ORM描述
   - 修正SQL注入防护说明

3. ✅ `anylisis/02-architecture/README.md`
   - 更新数据访问层描述
   - 添加OceanBase到基础设施层

4. ✅ `anylisis/02-architecture/system-component-architecture.puml`
   - 添加双ORM架构注释
   - 分离数据库服务和向量数据服务
   - 更新基础设施层包含OceanBase
   - 更新连接关系

---

## 🎓 总结

RAGFlow的双ORM架构是一个**深思熟虑的设计决策**，体现了：

1. **架构清晰性** - 业务层与检索层分离
2. **技术适配性** - 为不同场景选择最佳工具
3. **可扩展性** - 易于添加新的数据源或ORM
4. **性能优化** - 两种ORM各司其职，发挥最大优势

这不是技术债务，而是**架构成熟度的体现**！ 🎯

---

**文档版本**: 1.0  
**创建日期**: 2025-12-19  
**维护者**: RAGFlow 团队
