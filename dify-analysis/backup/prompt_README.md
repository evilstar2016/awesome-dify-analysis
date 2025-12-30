# Prompt 提示词模块

> 📝 Dify 提示词管理与转换系统的架构设计与实现原理

## 📋 本模块内容

本模块深入分析 Dify 的 Prompt 提示词系统，包括：

- 提示词模板管理
- 多模型提示词转换
- 变量注入与参数化
- 对话历史管理

## 🏗️ 架构概览

Dify 的 Prompt 系统采用**模板方法模式**，实现了灵活的提示词构建与转换：

### 提示词类型
1. **Simple Prompt**：简单提示词模板（Chatbot 基础模式）
2. **Advanced Prompt**：高级提示词（支持 Jinja2，Workflow LLM 节点）
3. **Agent Prompt**：智能体专用提示词
4. **Chat Prompt**：对话式提示词
5. **Completion Prompt**：补全式提示词

### 核心组件
```
PromptTransform（提示词转换器）
    ├── SimplePromptTransform（简单转换）
    ├── AdvancedPromptTransform（高级转换）
    └── AgentHistoryPromptTransform（Agent历史转换）

PromptTemplateParser（模板解析器）
    ├── 变量提取与验证
    ├── Jinja2 模板渲染
    └── 条件逻辑处理

PromptMessageUtil（消息工具）
    ├── 消息格式转换
    ├── Token 长度计算
    └── 消息历史截断
```

### 使用场景

| 应用类型 | 转换器 | 说明 |
|---------|--------|------|
| Chatbot (Basic) | SimplePromptTransform | 使用预设 JSON 模板 |
| Chatbot (Advanced) | AdvancedPromptTransform | 自定义多轮对话模板 |
| Completion App | SimplePromptTransform | 单次补全任务 |
| Workflow LLM | AdvancedPromptTransform | 工作流中的 LLM 节点 |
| Agent | AgentHistoryPromptTransform | Agent 历史消息管理 |

## 🎯 核心能力

### 1. 模板管理
- 支持多种模板格式（Chat/Completion）
- Jinja2 模板引擎支持
- 变量占位符与默认值
- 模板版本管理

### 2. 模型适配
- 不同模型的消息格式转换
- System/User/Assistant 角色适配
- 特殊模型格式处理（如百川、文心等）
- 提示词长度自动调整

### 3. 变量处理
- 动态变量注入（用户变量、特殊变量、Workflow 变量）
- 类型验证与转换
- 文件变量处理（多模态内容）
- 对话上下文变量（`{{#context#}}`, `{{#query#}}`, `{{#histories#}}`）

### 4. 历史管理
- 对话历史提取
- 消息窗口滑动
- Token 预算管理
- 历史压缩与摘要

---

## 📊 深度内容

| 维度 | 本文档（GitHub） | 完整版（公众号） |
|------|----------------|----------------|
| 架构设计 | ✅ 模板方法模式架构 | ✅ + 为什么选择模板方法模式 |
| 源码实现 | ⚠️ 核心类概述 | ✅ PromptTransform 完整源码解读 |
| 模板引擎 | ⚠️ 基本机制 | ✅ Jinja2 集成完整解析 |
| 变量注入 | ✅ 变量语法与系统 | ✅ 变量系统底层实现 |
| 历史管理 | ❌ | ✅ 对话历史优化策略 |
| 多模型适配 | ❌ | ✅ 各大模型提示词格式对比 |

## 📖 完整版内容

关注公众号 **「柒叔代码阁」**，回复 **"Prompt"** 获取：

- 🔍 PromptTransform 源码逐行解读（4500字）
- 🛠️ 如何实现自定义提示词模板
- ⚡ 提示词优化技巧与最佳实践
- ⚠️ 提示词渲染失败的 6 种场景及解决方案
- 🎯 企业级提示词工程实战案例

<!-- ![公众号二维码](../../qrcode.png) -->

---

## 📝 快速参考

### 变量语法

```
{{variable_name}}      # 用户自定义变量
{{#context#}}          # 上下文（RAG 检索结果）
{{#query#}}            # 用户查询
{{#histories#}}        # 对话历史
{{#node_id.var#}}      # Workflow 变量（高级模式）
```

### 模板示例

```python
# 简单模式 - 系统提示词
"你是一个有帮助的助手。用户问题：{{#query#}}"

# 高级模式 - Chat 消息
[
    {"role": "system", "text": "你是专业的客服助手"},
    {"role": "user", "text": "{{#query#}}"}
]
```

## 📁 详细文档

| 文件 | 描述 |
|------|------|
| [dify_prompt_management_documentation.md](./dify_prompt_management_documentation.md) | Prompt 管理模块完整技术文档 |
| [prompt_module_class_diagram.puml](./prompt_module_class_diagram.puml) | 模块类图（PlantUML） |
| [simple_prompt_transform_sequence.puml](./simple_prompt_transform_sequence.puml) | SimplePromptTransform 时序图 |
| [advanced_prompt_transform_sequence.puml](./advanced_prompt_transform_sequence.puml) | AdvancedPromptTransform 时序图 |
| [agent_history_prompt_transform_sequence.puml](./agent_history_prompt_transform_sequence.puml) | AgentHistoryPromptTransform 时序图 |

## 🔗 相关模块

- [Agent 智能体](../agent/) - Agent 的提示词设计
- [Model Runtime](../model_runtime/) - 模型调用与提示词
- [Workflow 工作流](../workflow/) - 工作流中的提示词节点
