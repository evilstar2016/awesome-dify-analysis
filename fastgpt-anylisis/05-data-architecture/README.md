# FastGPT 数据架构分析

本目录包含 FastGPT 项目的完整数据架构分析文档和图示。

## 📄 文档清单

### 核心文档
- **[data-architecture.md](data-architecture.md)** - 完整的数据架构文档
  - 数据库架构概述（MongoDB、VectorDB、Redis、MinIO/S3）
  - 核心数据模型详解（10+核心表结构）
  - 数据一致性与事务处理
  - 性能优化策略
  - 监控与维护
  - 安全性设计
  - 扩展性设计

## 🎨 架构图示

### 1. 数据架构组件图
- **[data-architecture.puml](data-architecture.puml)** - 总体数据架构
  - 四大数据存储层
  - 数据访问层
  - 应用层
  - 组件间的连接关系

### 2. 数据模型 ER 图
- **[data-model-er.puml](data-model-er.puml)** - 实体关系图
  - 详细的表结构定义（字段、类型、约束）
  - 主键、外键、索引标注
  - 表之间的关系（1对1、1对多、多对多）
  - 包含用户、团队、应用、知识库、对话等核心模型

### 3. 数据流程图
- **[data-flow.puml](data-flow.puml)** - 业务数据流
  - 知识库数据流（文档上传→分块→向量化→存储）
  - 对话数据流（消息→检索→LLM→保存）
  - 应用数据流（创建→版本→缓存）
  - 跨数据库的数据流转

### 4. 向量检索流程图
- **[vector-search-flow.puml](vector-search-flow.puml)** - 向量检索详细流程
  - 查询预处理
  - 文本向量化
  - 向量相似度搜索
  - 重排序（Rerank）
  - 结果返回

## 📊 数据架构概览

FastGPT 采用多数据库协同的数据架构：

```
┌──────────────────────────────────────────────────────┐
│              FastGPT 数据架构                        │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌───────┐ │
│  │ MongoDB  │  │ VectorDB │  │ Redis  │  │ MinIO │ │
│  │  主数据  │  │  向量检索│  │  缓存  │  │ 对象  │ │
│  │          │  │          │  │  队列  │  │ 存储  │ │
│  └──────────┘  └──────────┘  └────────┘  └───────┘ │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### MongoDB - 主数据库
- **用户与团队**: users, teams, team_members
- **应用**: apps, app_versions
- **知识库**: datasets, dataset_collections, dataset_datas, dataset_trainings
- **对话**: chats, chat_items
- **系统**: system_configs, system_logs

### 向量数据库
- **PGVector**: PostgreSQL + pgvector 扩展
- **Milvus**: 专业向量数据库
- **OceanBase**: 支持向量检索的分布式数据库

### Redis
- **缓存**: 系统配置、模型列表、团队信息、用户会话
- **任务队列**: BullMQ（训练队列、评估队列）
- **分布式锁**: 任务处理锁

### MinIO/S3
- **头像**: fastgpt-avatar
- **对话文件**: fastgpt-chat
- **知识库文件**: fastgpt-dataset
- **临时文件**: fastgpt-temp
- **插件资源**: fastgpt-plugin

## 🔑 核心数据模型关系

```
users (用户)
  └─ teams (团队)
      └─ team_members (成员)
          ├─ apps (应用)
          │   ├─ app_versions (版本)
          │   └─ chats (对话)
          │       └─ chat_items (消息)
          │
          └─ datasets (知识库)
              ├─ dataset_collections (文档集合)
              │   └─ dataset_datas (数据块)
              │       └─ dataset_vector (向量数据)
              │
              └─ dataset_trainings (训练队列)
```

## 💡 数据架构特点

1. **多租户隔离**: 通过 teamId 实现数据隔离
2. **读写分离**: MongoDB 主从分离，日志独立库
3. **缓存优化**: Redis 多层缓存策略
4. **向量检索**: 支持三种向量数据库，余弦相似度检索
5. **对象存储**: 文件与数据分离，预签名 URL 访问
6. **事务保证**: MongoDB 事务支持，级联删除策略
7. **性能优化**: 索引优化、批量操作、连接池管理
8. **水平扩展**: 分片、集群、分区支持

## 🛠️ 如何使用这些文档

### 开发者
- 了解表结构和字段定义
- 理解数据流和业务逻辑
- 学习数据访问模式
- 掌握性能优化技巧

### 架构师
- 评估架构设计
- 规划扩展方案
- 优化数据库性能
- 制定备份策略

### 运维工程师
- 配置数据库环境
- 监控数据库性能
- 执行备份和恢复
- 处理故障和告警

## 📚 相关文档

- [系统架构](../02-architecture/system-architecture.md)
- [数据持久化层](../03-layers/05-data-persistence-layer.md)
- [第三方集成](../06-ThirdParty/README.md)

## 🔧 工具推荐

### 查看 PUML 图
- **在线工具**: [PlantUML Online](http://www.plantuml.com/plantuml)
- **VS Code 插件**: PlantUML
- **IDEA 插件**: PlantUML integration

### 数据库工具
- **MongoDB**: MongoDB Compass, Studio 3T
- **PostgreSQL**: pgAdmin, DBeaver
- **Redis**: RedisInsight, Another Redis Desktop Manager
- **MinIO**: MinIO Console

---

**最后更新**: 2025-12-17
