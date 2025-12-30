# 07-others - 补充分析文档

本目录包含不属于主体七大分类（overview、architecture、layers、core-modules、data-architecture、third-party）的补充分析文档。

## 📚 文档列表

### 1. [上下文工程概念澄清.md](./上下文工程概念澄清.md)

**类型：** 概念澄清与对比分析

**内容：**
- 📊 **会话级上下文工程**（Session-Level Context Engineering）
  - 定义、时间范围、核心内容、应用场景
  - Dify 的完整支持情况
  
- 📊 **用户级上下文工程**（User-Level Context Engineering）
  - 定义、时间范围、核心内容、应用场景
  - 跨会话用户画像与个性化定制
  - 当前实现状态（基础支持）

- 🔄 **对比分析**：两种上下文工程的区别与适用场景

**阅读对象：** 需要理解 Dify 上下文管理机制的开发者、产品设计者

---

### 2. [Dify知识问答场景-上下文工程应用详解.md](./Dify知识问答场景-上下文工程应用详解.md)

**类型：** 技术深度解析

**内容：**
- 🏗️ **架构设计**：知识检索 → 上下文构建 → Prompt 组装 → LLM 生成
- 🔄 **完整处理流程**（6 个阶段）：
  1. 用户输入与上下文变量准备
  2. 知识库检索与上下文获取
  3. 上下文构建与过滤
  4. Prompt 组装与变量注入
  5. Token 管理与历史对话裁剪
  6. LLM 调用与质量追踪

- 💻 **代码实现分析**：
  - 前端：PromptEditor 组件
  - 后端：上下文处理、向量检索、重排序、Token 管理

- 📊 **最佳实践**

**核心变量：**
```
{{#context#}}   - 知识库检索结果
{{#histories#}} - 对话历史（当前会话）
{{#query#}}     - 用户当前问题
{{自定义}}      - 用户自定义变量
```

**阅读对象：** 需要深入理解 Dify 知识问答内部实现的开发者

---

### 3. [context_engineering_qa_flow.puml](./context_engineering_qa_flow.puml)

**类型：** PlantUML 流程图

**可视化内容：**
- 知识问答场景的完整数据流与组件交互
- 上下文工程的处理步骤
- 系统端到端的流程

**查看方式：**
- 在线：https://www.plantuml.com/plantuml/uml/（复制内容粘贴）
- VS Code：安装 PlantUML 插件，按 Alt + D 预览
- 命令行：`plantuml context_engineering_qa_flow.puml`

**适合读者：** 所有需要快速理解知识问答系统流程的开发者

---

## 🔗 相关主体分析

- [04-core-modules/knowledgeBase](../04-core-modules/knowledgeBase/) - 知识库模块详解
- [04-core-modules/prompt](../04-core-modules/prompt/) - Prompt 管理与模板
- [03-layers/api](../03-layers/api/) - API 层设计与控制器

---

📱 **关注公众号「柒叔代码阁」**

定期发布Dify深度内容及生产实践～

![公众号二维码](../qrcode.png)

💡 **为什么关注？**

不只是告诉你"是什么"和"怎么做"，更重要的是 **"为什么"** — 理解设计背后的思想，才能举一反三。

💬 **交流讨论**
- GitHub Issue/Discussion：技术问题讨论
- 公众号留言


---

## 📝 文档维护

- **最后更新**：2025-12-25
- **内容来源**：Dify 技术分析与深度研究

此目录作为补充，不属于核心分类体系，但提供了重要的概念澄清与实现细节。
