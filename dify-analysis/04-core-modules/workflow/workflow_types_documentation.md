# Dify 工作流类型说明文档

> 版本：1.9.2  
> 最后更新：2025年12月

## 目录

1. [概述](#概述)
2. [工作流类型详解](#工作流类型详解)
   - [常规工作流 (Workflow)](#常规工作流-workflow)
   - [Chat 工作流 (Chat)](#chat-工作流-chat)
   - [RAG Pipeline](#rag-pipeline)
3. [技术架构对比](#技术架构对比)
4. [变量系统差异](#变量系统差异)
5. [节点可用性矩阵](#节点可用性矩阵)
6. [应用模式与工作流类型映射](#应用模式与工作流类型映射)
7. [适用场景指南](#适用场景指南)
8. [选型决策树](#选型决策树)

---

## 概述

Dify 平台支持三种工作流类型，每种类型针对不同的使用场景进行了优化：

| 工作流类型 | 标识值 | 主要用途 |
|-----------|--------|---------|
| **常规工作流** | `workflow` | 一次性任务处理、批量数据处理 |
| **Chat 工作流** | `chat` | 多轮对话应用、智能助手 |
| **RAG Pipeline** | `rag-pipeline` | 知识库文档处理、智能检索增强 |

### 类型定义

工作流类型在后端定义于 `api/core/workflow/enums.py` 和 `api/models/workflow.py`：

```python
class WorkflowType(StrEnum):
    WORKFLOW = "workflow"
    CHAT = "chat"
    RAG_PIPELINE = "rag-pipeline"
```

---

## 工作流类型详解

### 常规工作流 (Workflow)

#### 定义

常规工作流是一种**单次执行**的处理流程，适用于不需要上下文记忆的任务。每次执行都是独立的，不保留历史交互信息。

#### 核心特点

| 特性 | 描述 |
|-----|------|
| **执行模式** | 单次执行，无状态 |
| **上下文** | 无对话历史 |
| **变量系统** | 仅支持环境变量 (Environment Variables) |
| **触发方式** | API 调用 / 手动触发 |
| **输出形式** | 结构化数据 / 文本 |

#### 入口节点

使用 **Start** 节点作为入口，可配置多种输入参数类型：
- 文本 (text-input)
- 段落 (paragraph)
- 数字 (number)
- 选择器 (select)
- 文件 (files)

#### 典型结构

```
Start → [处理节点链] → End
```

#### 适用场景

1. **批量数据处理**
   - 文档批量翻译
   - 数据格式转换
   - 内容批量生成

2. **自动化任务**
   - 定时报告生成
   - 数据分析流程
   - 邮件内容生成

3. **API 服务**
   - 无状态 API 接口
   - 数据处理服务
   - 内容审核服务

4. **一次性处理**
   - 表单数据处理
   - 图像分析
   - 文件解析

---

### Chat 工作流 (Chat)

#### 定义

Chat 工作流专为**多轮对话**场景设计，支持会话历史记忆和上下文感知。这是构建智能助手和对话机器人的首选类型。

#### 核心特点

| 特性 | 描述 |
|-----|------|
| **执行模式** | 会话式，有状态 |
| **上下文** | 支持完整对话历史 |
| **变量系统** | 环境变量 + 会话变量 (Conversation Variables) |
| **触发方式** | 用户消息 / API 调用 |
| **输出形式** | 流式文本 / 结构化响应 |

#### 入口节点

使用 **Start** 节点，但具备以下 Chat 专属特性：
- 自动注入 `sys.query`（用户当前输入）
- 自动注入 `sys.conversation_id`（会话标识）
- 支持 `sys.files`（用户上传文件）

#### 会话变量系统

Chat 工作流独有的**会话变量 (Conversation Variables)** 系统：

```python
# 会话变量定义在 Workflow 模型中
_conversation_variables: Mapped[str] = mapped_column(
    "conversation_variables", sa.Text, nullable=False, server_default="{}"
)
```

会话变量特点：
- **跨轮次持久化**：在同一会话内保持状态
- **自动初始化**：首次对话时创建
- **运行时更新**：通过 Variable Assigner 节点修改
- **会话隔离**：不同会话之间相互独立

#### 典型结构

```
Start → LLM (带历史) → [可选处理] → Answer
```

#### 高级功能

1. **对话历史管理**
   - 自动累积对话记录
   - 可配置历史窗口大小
   - 支持历史摘要压缩

2. **流式输出**
   - 实时返回生成内容
   - 支持多个 Answer 节点
   - 支持中间状态展示

3. **会话状态跟踪**
   - 用户偏好记忆
   - 任务进度追踪
   - 多轮收集信息

#### 适用场景

1. **智能客服**
   - 多轮问答
   - 问题诊断
   - 服务引导

2. **个人助手**
   - 日程管理
   - 信息查询
   - 任务规划

3. **教育辅导**
   - 知识问答
   - 学习引导
   - 练习批改

4. **销售顾问**
   - 产品推荐
   - 需求分析
   - 订单处理

---

### RAG Pipeline

#### 定义

RAG Pipeline 是专为**知识库增强**场景设计的工作流类型，用于处理文档索引、知识检索和智能问答。它是 Dify 知识库功能的核心引擎。

#### 核心特点

| 特性 | 描述 |
|-----|------|
| **执行模式** | 数据驱动，可批量 |
| **上下文** | 知识库上下文 |
| **变量系统** | 环境变量 + RAG Pipeline 变量 |
| **触发方式** | 文档导入 / 索引构建 / 检索请求 |
| **输出形式** | 索引数据 / 检索结果 |

#### 专属节点

RAG Pipeline 具有两个**独占节点**，在其他工作流类型中不可用：

| 节点 | 用途 |
|-----|------|
| **Datasource** | 定义数据来源（文件、Notion、网页爬取等） |
| **Knowledge Index** | 执行文档分段、向量化和索引构建 |

#### RAG Pipeline 变量

```python
# RAG Pipeline 专属变量
_rag_pipeline_variables: Mapped[str] = mapped_column(
    "rag_pipeline_variables", db.Text, nullable=False, server_default="{}"
)
```

#### 典型结构

**文档处理流程：**
```
Datasource → [文档处理节点] → Knowledge Index
```

**检索增强流程：**
```
Start → Knowledge Retrieval → LLM → Answer
```

#### 核心能力

1. **多源数据接入**
   - 本地文件上传
   - Notion 集成
   - 网页爬取
   - API 数据源

2. **智能分段策略**
   - 自动分段
   - 自定义分段规则
   - 父子分段模式

3. **向量化处理**
   - 多种 Embedding 模型
   - 批量向量化
   - 增量更新

4. **检索优化**
   - 混合检索
   - 重排序
   - 上下文压缩

#### 适用场景

1. **企业知识库**
   - 文档智能检索
   - 知识问答系统
   - 政策查询助手

2. **文档处理**
   - 大批量文档索引
   - 内容提取分析
   - 多格式文档解析

3. **智能搜索**
   - 语义搜索
   - 精准问答
   - 引用溯源

---

## 技术架构对比

### 执行引擎差异

```
┌─────────────────────────────────────────────────────────────────┐
│                        执行引擎架构                              │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Workflow      │     Chat        │      RAG Pipeline           │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ WorkflowRunner  │ AdvancedChat    │   PipelineRunner            │
│                 │ AppRunner       │                             │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ 无会话状态      │ 维护会话状态    │   处理数据流                │
│ 单次执行        │ 多轮交互        │   批量/流式处理             │
│ 直接返回结果    │ 流式响应        │   索引构建                  │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

### 后端服务对应

| 工作流类型 | 主要服务类 | 执行器 |
|-----------|-----------|--------|
| Workflow | `WorkflowService` | `WorkflowAppRunner` |
| Chat | `WorkflowService` | `AdvancedChatAppRunner` |
| RAG Pipeline | `RagPipelineService` | `PipelineRunner` |

---

## 变量系统差异

### 变量类型支持矩阵

| 变量类型 | Workflow | Chat | RAG Pipeline |
|---------|:--------:|:----:|:------------:|
| 环境变量 (Environment Variables) | ✅ | ✅ | ✅ |
| 会话变量 (Conversation Variables) | ❌ | ✅ | ❌ |
| RAG Pipeline 变量 | ❌ | ❌ | ✅ |
| 系统变量 (sys.*) | ✅ | ✅ | ✅ |

### 变量生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                      变量生命周期对比                            │
├─────────────────────────────────────────────────────────────────┤
│ 环境变量                                                        │
│ ├── 作用域: 工作流级别                                          │
│ ├── 生命周期: 工作流配置期间                                     │
│ └── 用途: 全局配置、API Key、常量                                │
├─────────────────────────────────────────────────────────────────┤
│ 会话变量 (仅 Chat)                                              │
│ ├── 作用域: 会话级别                                            │
│ ├── 生命周期: 会话存续期间                                       │
│ └── 用途: 用户偏好、累积数据、状态追踪                           │
├─────────────────────────────────────────────────────────────────┤
│ RAG Pipeline 变量 (仅 RAG Pipeline)                             │
│ ├── 作用域: Pipeline 执行级别                                   │
│ ├── 生命周期: 单次 Pipeline 执行                                 │
│ └── 用途: 文档元数据、处理参数                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 节点可用性矩阵

### 完整节点可用性

| 节点类型 | Workflow | Chat | RAG Pipeline | 说明 |
|---------|:--------:|:----:|:------------:|------|
| **入口/出口节点** |
| Start | ✅ | ✅ | ✅ | 入口配置不同 |
| End | ✅ | ❌ | ✅ | Chat 使用 Answer |
| Answer | ❌ | ✅ | ❌ | 流式输出 |
| **LLM 节点** |
| LLM | ✅ | ✅ | ✅ | |
| Agent | ✅ | ✅ | ✅ | |
| **数据处理节点** |
| Code | ✅ | ✅ | ✅ | |
| Template Transform | ✅ | ✅ | ✅ | |
| Variable Assigner | ✅ | ✅ | ✅ | |
| Variable Aggregator | ✅ | ✅ | ✅ | |
| List Operator | ✅ | ✅ | ✅ | |
| Document Extractor | ✅ | ✅ | ✅ | |
| **知识库节点** |
| Knowledge Retrieval | ✅ | ✅ | ✅ | |
| **RAG Pipeline 专属** |
| Datasource | ❌ | ❌ | ✅ | 数据源定义 |
| Knowledge Index | ❌ | ❌ | ✅ | 索引构建 |
| **流程控制节点** |
| IF/ELSE | ✅ | ✅ | ✅ | |
| Iteration | ✅ | ✅ | ✅ | |
| Loop | ✅ | ✅ | ✅ | |
| Parallel | ✅ | ✅ | ✅ | |
| **工具节点** |
| Tool | ✅ | ✅ | ✅ | |
| HTTP Request | ✅ | ✅ | ✅ | |
| **其他节点** |
| Parameter Extractor | ✅ | ✅ | ✅ | |
| Question Classifier | ✅ | ✅ | ✅ | |

---

## 应用模式与工作流类型映射

Dify 的**应用模式 (AppMode)** 与**工作流类型 (WorkflowType)** 存在映射关系：

### AppMode 定义

```python
class AppMode(StrEnum):
    COMPLETION = "completion"      # 文本生成
    WORKFLOW = "workflow"          # 常规工作流
    CHAT = "chat"                  # 基础对话
    ADVANCED_CHAT = "advanced-chat" # 高级对话 (Chat 工作流)
    AGENT_CHAT = "agent-chat"      # Agent 对话
    CHANNEL = "channel"            # 渠道
    RAG_PIPELINE = "rag-pipeline"  # RAG Pipeline
```

### 映射规则

```python
@classmethod
def from_app_mode(cls, app_mode: Union[str, "AppMode"]) -> "WorkflowType":
    app_mode = app_mode if isinstance(app_mode, AppMode) else AppMode.value_of(app_mode)
    return cls.WORKFLOW if app_mode == AppMode.WORKFLOW else cls.CHAT
```

### 映射关系表

| AppMode | WorkflowType | 说明 |
|---------|-------------|------|
| `workflow` | WORKFLOW | 1:1 映射 |
| `advanced-chat` | CHAT | 使用 Chat 工作流引擎 |
| `chat` | CHAT | 基础对话模式 |
| `agent-chat` | CHAT | Agent 也使用 Chat 引擎 |
| `rag-pipeline` | RAG_PIPELINE | RAG 专用类型 |

---

## 适用场景指南

### 场景决策矩阵

| 场景特征 | 推荐类型 | 原因 |
|---------|---------|------|
| 需要多轮对话 | Chat | 内置会话管理 |
| 一次性数据处理 | Workflow | 无状态设计更高效 |
| 构建知识库 | RAG Pipeline | 专用节点支持 |
| 需要记忆用户偏好 | Chat | 会话变量支持 |
| 批量文档索引 | RAG Pipeline | 批量处理优化 |
| API 服务接口 | Workflow | 简单直接 |
| 智能客服/助手 | Chat | 自然对话体验 |
| 定时任务 | Workflow | 易于调度触发 |
| 文档问答系统 | Chat + 知识库 | 结合检索与对话 |

### 详细场景示例

#### 场景 1: 智能客服系统

**推荐类型**: Chat 工作流

**理由**:
- 需要多轮对话理解用户问题
- 需要记住对话上下文
- 需要流式输出提升体验

**典型配置**:
```
Start → Knowledge Retrieval → LLM (with history) → Answer
```

#### 场景 2: 文档批量翻译

**推荐类型**: 常规 Workflow

**理由**:
- 单次处理，无需上下文
- 批量处理效率更高
- 结构化输出便于后处理

**典型配置**:
```
Start → Document Extractor → LLM → End
```

#### 场景 3: 企业知识库构建

**推荐类型**: RAG Pipeline

**理由**:
- 需要专用的 Datasource 节点
- 需要 Knowledge Index 进行索引
- 支持多种数据源接入

**典型配置**:
```
Datasource → Code (预处理) → Knowledge Index
```

#### 场景 4: 数据分析报告生成

**推荐类型**: 常规 Workflow

**理由**:
- 一次性任务
- 需要多步骤数据处理
- 输出结构化报告

**典型配置**:
```
Start → HTTP Request → Code (分析) → LLM (报告) → End
```

---

## 选型决策树

```
                    ┌─────────────────────┐
                    │   开始选择工作流类型  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │ 是否需要构建知识库？ │
                    └──────────┬──────────┘
                          ┌────┴────┐
                         是        否
                          │         │
               ┌──────────▼───┐     │
               │ RAG Pipeline │     │
               └──────────────┘     │
                               ┌────▼────────────────┐
                               │ 是否需要多轮对话？   │
                               └────┬────────────────┘
                               ┌────┴────┐
                              是        否
                               │         │
                    ┌──────────▼───┐  ┌──▼──────────┐
                    │ Chat 工作流  │  │ 常规 Workflow│
                    └──────────────┘  └─────────────┘
```

### 快速参考卡片

| 如果你需要... | 选择 |
|-------------|------|
| 多轮对话、上下文记忆 | **Chat 工作流** |
| 一次性任务处理 | **常规 Workflow** |
| 文档索引、知识库构建 | **RAG Pipeline** |
| 用户偏好存储 | **Chat 工作流** (会话变量) |
| 批量数据处理 | **常规 Workflow** |
| API 无状态服务 | **常规 Workflow** |
| 智能问答系统 | **Chat 工作流** + 知识库 |

---

## 附录

### 相关代码文件

| 文件路径 | 说明 |
|---------|------|
| `api/core/workflow/enums.py` | WorkflowType 枚举定义 |
| `api/models/workflow.py` | Workflow 模型和变量系统 |
| `api/models/model.py` | AppMode 枚举定义 |
| `api/services/workflow_service.py` | Workflow 服务实现 |
| `api/services/rag_pipeline/rag_pipeline.py` | RAG Pipeline 服务 |
| `api/core/app/apps/workflow/app_runner.py` | Workflow 执行器 |
| `api/core/app/apps/advanced_chat/app_runner.py` | Chat 执行器 |
| `api/core/app/apps/pipeline/pipeline_runner.py` | Pipeline 执行器 |

### 版本更新说明

| 版本 | 更新内容 |
|-----|---------|
| 1.9.x | RAG Pipeline 类型增强，新增自定义模板支持 |
| 1.8.x | 会话变量系统完善 |
| 1.7.x | Loop 节点支持 |
| 1.6.x | Agent 节点增强 |

---

*文档生成时间: 2025年12月*  
*基于 Dify 版本: 1.9.2*
