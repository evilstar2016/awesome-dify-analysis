# Langfuse 核心模块分析

## 模块识别

基于步骤2和步骤3的架构分析，识别出以下核心模块（按业务重要性排序）：

### 1. **Traces & Observations 模块** ⭐⭐⭐⭐⭐
**目录**: `traces/`  
**重要性**: 最核心的模块，LLM 追踪和观测的基础  
**功能**: Trace 创建、查询、分析，Observation 管理，成本计算

### 2. **Prompts 模块** ⭐⭐⭐⭐⭐
**目录**: `prompts/`  
**重要性**: 提示词管理是 LLM 应用的核心能力  
**功能**: 提示词版本管理、模板渲染、A/B 测试

### 3. **Datasets & Evaluation 模块** ⭐⭐⭐⭐⭐
**目录**: `datasets/`  
**重要性**: LLM 评估和测试的基础设施  
**功能**: 数据集管理、评估执行、评估器配置、结果分析

### 4. **Scores 模块** ⭐⭐⭐⭐
**目录**: `scores/`  
**重要性**: 质量评分是 LLM 监控的关键  
**功能**: 评分管理、评分配置、聚合统计

### 5. **Playground 模块** ⭐⭐⭐⭐ ✅
**目录**: `playground/`  
**重要性**: 交互式 LLM 测试环境  
**功能**: 多模型对比、提示词测试、实时调用  
**状态**: 已完成 - README + 3 个时序图

### 6. **Dashboard & Analytics 模块** ⭐⭐⭐⭐ ✅
**目录**: `dashboard/`  
**重要性**: 数据可视化和分析  
**功能**: 仪表板、图表、统计报表  
**状态**: 已完成 - README + 3 个时序图

### 7. **Ingestion 模块** ⭐⭐⭐⭐
**目录**: `packages/shared/src/features/ingestion/`  
**重要性**: SDK 数据接入的入口  
**功能**: 事件接收、验证、写入、异步处理

### 8. **Authentication & RBAC 模块** ⭐⭐⭐
**目录**: `web/src/features/auth/`, `web/src/features/rbac/`  
**重要性**: 安全和权限管理  
**功能**: 用户认证、角色权限、API Key 管理

---

## 模块分析顺序

### 第一批：核心数据模块（已完成）
- [x] Traces & Observations
- [ ] Scores
- [ ] Prompts

### 第二批：评估和测试模块
- [ ] Datasets & Evaluation
- [ ] Playground

### 第三批：分析和展示模块
- [ ] Dashboard & Analytics
- [ ] Ingestion

### 第四批：基础设施模块
- [ ] Authentication & RBAC

---

## 文档结构

每个模块包含：

```
{module-name}/
├── README.md                           # 模块概述
├── {feature}-management.md             # 功能详细说明
├── 01-{flow-name}-sequence.puml        # 核心流程时序图1
├── 02-{flow-name}-sequence.puml        # 核心流程时序图2
└── ...
```

---

## 分析方法

### 1. 模块结构分析
- 使用 Serena MCP 的 `list_dir` 探索目录结构
- 使用 `get_symbols_overview` 了解主要文件
- 识别模块边界和依赖关系

### 2. 代码路径追踪
- 使用 `find_symbol` 查找关键函数/类
- 使用 `find_referencing_symbols` 查找调用关系
- 使用 `search_for_pattern` 搜索特定模式

### 3. 业务流程梳理
- 确定入口点（API 路由、tRPC procedures）
- 追踪执行路径
- 识别关键决策点
- 记录数据变换
- 标注异常处理

### 4. 时序图绘制
- 使用统一的 PlantUML 样式（参考 puml-styles.md）
- 标注主要参与者
- 绘制完整调用链路
- 添加关键注释

---

## 技术栈总结

### 前端技术栈
- **框架**: Next.js 15 (Pages Router)
- **UI**: React 19 + shadcn/ui
- **状态**: TanStack Query

### 后端技术栈
- **API**: tRPC 11
- **ORM**: Prisma 6
- **认证**: NextAuth.js
- **队列**: BullMQ

### 数据库技术栈
- **OLTP**: PostgreSQL 15+
- **OLAP**: ClickHouse 24+
- **缓存**: Redis 7+
- **存储**: S3/Blob

---

## 分析进度

| 模块 | README | 时序图 | 状态 |
|-----|--------|--------|------|
| Traces & Observations | ✅ 15KB | ✅ (3个) | ✅ 完成 |
| Prompts | ✅ 16KB | ✅ (4个) | ✅ 完成 |
| Datasets & Evaluation | ✅ 20KB | ✅ (3个) | ✅ 完成 |
| Scores | ✅ 25KB | ✅ (3个) | ✅ 完成 |
| **Playground** | ✅ 52KB | ✅ (3个) | ✅ 完成 |
| **Dashboard & Analytics** | ✅ 36KB | ✅ (3个) | ✅ **完成** |
| Ingestion | ⏳ | ⏳ | 未开始 |
| Authentication & RBAC | ⏳ | ⏳ | 未开始 |

**完成进度**: 6/8 模块 (75%)

---

**文档编写时间**: 2025-12-17  
**项目版本**: 3.140.0
