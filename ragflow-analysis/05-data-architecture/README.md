# 数据架构分析

本目录包含 RAGFlow 项目的完整数据架构文档和图示。

## 📁 文件列表

| 文件名 | 说明 | 类型 |
|--------|------|------|
| [data-architecture.md](./data-architecture.md) | **完整的数据架构文档** | Markdown |
| [data-architecture.puml](./data-architecture.puml) | 数据架构组件图 | PlantUML |
| [data-model-er.puml](./data-model-er.puml) | 数据模型ER图 | PlantUML |
| [data-flow.puml](./data-flow.puml) | 数据流程图 | PlantUML |

## 📖 文档概述

### data-architecture.md

这是核心文档，包含以下内容：

1. **概述** - 数据架构设计理念和技术选型
2. **MySQL/PostgreSQL 设计** - 详细的表结构、字段定义、索引策略
3. **向量数据库设计** - Infinity/OceanBase 向量存储方案
4. **Elasticsearch/OpenSearch 设计** - 全文检索索引设计
5. **Redis 设计** - 缓存、任务队列、分布式锁
6. **对象存储设计** - MinIO/S3 文件存储规划
7. **数据一致性保证** - 事务、级联删除、备份策略
8. **性能优化策略** - 各层性能调优方案
9. **监控与维护** - 监控指标、告警规则、日志系统
10. **数据迁移与版本管理** - Schema 迁移和版本控制
11. **安全性设计** - 加密、访问控制、数据隔离
12. **扩展性设计** - 水平和垂直扩展方案

## 🎨 PlantUML 图示

### 1. data-architecture.puml - 数据架构组件图

展示了 RAGFlow 的完整数据架构：
- 应用层：Web UI、API Server、Task Executor
- 数据访问层：ORM、各种客户端
- 数据持久化层：MySQL、Infinity、Elasticsearch、Redis、MinIO

**预览方式**：
```bash
# 使用 PlantUML 渲染
plantuml data-architecture.puml

# 或使用在线查看器
# https://www.plantuml.com/plantuml/uml/
```

### 2. data-model-er.puml - 数据模型 ER 图

详细展示了核心数据表及其关系：
- 用户与租户模型（User, Tenant, UserTenant）
- 知识库模型（Knowledgebase, Document, Task）
- 对话模型（Dialog, Conversation, API4Conversation）
- LLM 模型（LLMFactories, LLM, TenantLLM）
- 文件管理（File, File2Document）
- 连接器（Connector, Connector2Kb）
- 评估系统（EvaluationDataset, EvaluationRun）

### 3. data-flow.puml - 数据流程图

展示了四个主要业务流程：
- **文档处理流程**：上传 → 存储 → 解析 → 向量化 → 索引
- **用户查询流程**：提问 → 检索（向量+全文） → 重排序 → LLM 生成
- **数据同步流程**：定时/Webhook → 增量同步 → 文档更新
- **缓存更新流程**：数据更新 → 缓存失效 → 索引更新

## 🔑 核心数据存储

RAGFlow 使用多种数据存储技术：

### 1. MySQL/PostgreSQL（元数据）
- **用途**：存储业务元数据、用户信息、配置
- **核心表**：30+ 张表
- **连接池**：最大 900 连接
- **ORM**：Peewee

### 2. Infinity（向量数据库）
- **用途**：存储文档向量，支持 KNN 检索
- **索引**：HNSW
- **距离度量**：内积/余弦相似度
- **Collection**：每知识库一个

### 3. Elasticsearch/OpenSearch（全文检索）
- **用途**：存储文档块，支持 BM25 检索
- **索引**：每租户一个
- **分析器**：IK 中文分词
- **版本要求**：ES 8.x+ / OS 2.x+

### 4. NetworkX（图数据库）
- **用途**：存储知识图谱（GraphRAG），实体和关系网络
- **存储方式**：内存图结构
- **算法支持**：PageRank、Leiden 社区检测、最短路径
- **实现方式**：Light GraphRAG / General GraphRAG（Microsoft）

### 5. Redis（缓存/队列）
- **用途**：缓存、任务队列、分布式锁
- **数据结构**：String、Hash、Sorted Set、Streams
- **实现**：Valkey（Redis 兼容）

### 6. MinIO/S3（对象存储）
- **用途**：存储原始文件、图片、解析结果、图数据序列化文件
- **Bucket**：每租户一个
- **访问控制**：预签名 URL

## 📊 数据关系概览

```
Tenant（租户）
  ├── Knowledgebase（知识库）
  │    ├── Document（文档）
  │    │    └── Task（任务）
  │    ├── [ES] Chunks（文档块）
  │    │    └── [Infinity] Vectors（向量）
  │    └── [NetworkX] Knowledge Graph（知识图谱）
  │         ├── Entities（实体）
  │         ├── Relations（关系）
  │         └── Communities（社区）
  │
  ├── Dialog（对话应用）
  │    ├── Conversation（对话记录）
  │    └── API4Conversation（API 调用记录）
  │
  └── TenantLLM（LLM 配置）
```

## 🔄 典型数据流

### 文档上传流程
```
1. 用户上传 → 2. API 验证 → 3. 保存元数据(MySQL)
   ↓                ↓                ↓
4. 上传文件(MinIO) → 5. 创建任务 → 6. 入队(Redis)
   ↓
7. 任务执行器拉取 → 8. 解析文档 → 9. 向量化(Infinity)
   ↓                                ↓
10. 索引(Elasticsearch) ← ────────── 11. 更新进度(MySQL)
```

### 查询流程
```
1. 用户提问 → 2. 查询向量化
   ↓
3. 并行检索
   ├── Infinity: 向量检索 Top 1024
   └── Elasticsearch: 全文检索 Top 1024
   ↓
4. 融合 → 5. 重排序 Top 6 → 6. LLM 生成 → 7. 返回答案
```

## 📈 性能指标

### 数据规模参考

| 规模 | 用户数 | 知识库 | 文档数 | 文档块数 | 存储 |
|------|--------|--------|--------|----------|------|
| 小型 | < 100 | < 100 | < 10K | < 1M | < 100GB |
| 中型 | 100-1K | < 1K | < 100K | < 10M | < 1TB |
| 大型 | > 1K | > 1K | > 100K | > 10M | > 1TB |

### 性能优化要点

1. **数据库优化**
   - 索引策略：所有外键和高频查询字段
   - 连接池：900 最大连接
   - 慢查询监控：> 1s

2. **向量检索优化**
   - HNSW 参数：M=16, ef=200
   - 批量插入：500-1000 条/批
   - 定期 Compact

3. **全文检索优化**
   - Refresh Interval: 30s
   - 禁用副本（单节点）
   - Query Cache

4. **缓存优化**
   - 多级缓存：内存 + Redis
   - TTL：300-3600s
   - 命中率：> 90%

## 🛡️ 安全性

- **数据加密**：敏感字段 AES-256 加密
- **访问控制**：JWT + API Key 认证
- **租户隔离**：通过 tenant_id 强制过滤
- **软删除**：标记 status='0'，不物理删除

## 🔧 维护建议

### 日常维护
- 监控慢查询（> 1s）
- 检查队列堆积（> 1000）
- 监控磁盘使用率（< 80%）
- 检查缓存命中率（> 90%）

### 定期维护
- **每日**：数据库全量备份
- **每周**：Infinity 索引 Compact
- **每月**：清理过期数据（status='0'）
- **季度**：性能基准测试

## 📚 相关文档

- [项目概览](../01-overview/project-overview.md)
- [架构设计](../02-architecture/architecture-overview.md)
- [核心模块](../04-core-modules/)
- [API 文档](../04-core-modules/api-layer.md)
- [部署指南](../../docker/README.md)

## 🤝 贡献

如发现文档有误或需要补充，请提交 Issue 或 Pull Request。

---

**文档版本**: v1.0  
**生成日期**: 2024-12-19  
**维护者**: RAGFlow Team
