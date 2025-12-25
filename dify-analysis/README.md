# Dify 项目分析文档

本目录包含了 Dify 项目的全面技术分析文档，按照模块化的方式组织。

## 📁 目录结构

### [01-overview](./01-overview) - 项目概述
- Dify 项目目标、能力、架构与技术栈速览
- 仓库结构与核心模块介绍
- 开发、质量与部署指南

### [02-architecture](./02-architecture) - 系统架构
- 组件架构设计
- 架构图与资源文件（png）

### [03-layers](./03-layers) - 分层设计
- API 层架构与接口设计
- 各类 API 的序列图与文档

### [04-core-modules](./04-core-modules) - 核心模块
- Agent 智能体模块
- Workflow 工作流模块
- Prompt 提示词模块
- Tools & Plugins 工具与插件
- Model Runtime 模型运行时
- KnowledgeBase 知识库模块
 - Observability 可观测性模块
 - Permission 权限管理模块

### [05-data-architecture](./05-data-architecture) - 数据架构
- 数据库设计与架构
- 数据流转分析

### [06-third-party](./06-third-party) - 第三方集成
- 大语言模型集成
- 向量数据库集成
- 对象存储集成
- 消息平台集成
- 文档处理集成
- 数据库与缓存集成
- AI增强服务集成
- 代码执行集成
- 知识库集成
- 认证与监控集成

### [07-others](./07-others) - 其他补充文档
- 会话级上下文工程（Context Engineering）深度分析
- 知识问答场景中的上下文处理流程图
- 用户级与会话级上下文工程的概念对比

## 📖 使用指南

1. **新手入门**：建议从 `01-overview` 开始了解项目整体概况
2. **架构理解**：查看 `02-architecture` 了解系统架构设计
3. **深入模块**：根据需要深入 `04-core-modules` 中的具体模块
4. **集成开发**：需要集成第三方服务时参考 `06-third-party`

## 🔍 文档类型说明

- `.puml` - PlantUML 图表文件，描述架构和流程
- `.md` - Markdown 技术文档，详细说明和指南

## 🤝 贡献

欢迎补充和完善分析文档，请确保：
- 文档放置在合适的目录下
- 使用清晰的命名规范
- 包含必要的图表和说明
