# Dify Prompt 管理模块分析

本目录包含 Dify Prompt 管理模块的深度分析文档和架构图。

## 📁 文档结构

| 文件 | 描述 |
|------|------|
| [dify_prompt_management_documentation.md](./dify_prompt_management_documentation.md) | Prompt 管理模块完整技术文档 |
| [prompt_module_class_diagram.puml](./prompt_module_class_diagram.puml) | 模块类图（PlantUML） |
| [simple_prompt_transform_sequence.puml](./simple_prompt_transform_sequence.puml) | SimplePromptTransform 时序图 |
| [advanced_prompt_transform_sequence.puml](./advanced_prompt_transform_sequence.puml) | AdvancedPromptTransform 时序图 |
| [agent_history_prompt_transform_sequence.puml](./agent_history_prompt_transform_sequence.puml) | AgentHistoryPromptTransform 时序图 |
| [prompt_template_parser_sequence.puml](./prompt_template_parser_sequence.puml) | PromptTemplateParser 解析流程时序图 |
| [prompt_module_integration_sequence.puml](./prompt_module_integration_sequence.puml) | Prompt 模块与应用层集成时序图 |

## 🏗️ 模块概览

Prompt 管理模块是 Dify 平台的核心组件，负责构建、转换和管理发送给 LLM 的提示词。

### 核心类

```
PromptTransform (基类)
├── SimplePromptTransform      # Chatbot 基础模式
├── AdvancedPromptTransform    # Workflow LLM 节点
└── AgentHistoryPromptTransform # Agent 应用
```

### 使用场景

| 应用类型 | 转换器 | 说明 |
|---------|--------|------|
| Chatbot (Basic) | SimplePromptTransform | 使用预设 JSON 模板 |
| Chatbot (Advanced) | AdvancedPromptTransform | 自定义多轮对话模板 |
| Completion App | SimplePromptTransform | 单次补全任务 |
| Workflow LLM | AdvancedPromptTransform | 工作流中的 LLM 节点 |
| Agent | AgentHistoryPromptTransform | Agent 历史消息管理 |

## 📊 查看图表

使用 PlantUML 渲染 `.puml` 文件：

1. **VS Code 插件**: 安装 PlantUML 扩展
2. **在线工具**: [PlantUML Online Server](http://www.plantuml.com/plantuml/uml/)
3. **命令行**: `java -jar plantuml.jar filename.puml`

## 🔑 关键特性

- **多模式支持**: Chat 和 Completion 模型模式
- **模板系统**: Basic（变量替换）和 Jinja2（高级模板）
- **内存管理**: Token 限制和消息窗口
- **变量系统**: 用户变量、特殊变量、Workflow 变量
- **文件处理**: 多模态内容（图片、音频等）

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

## 🔗 相关模块

- `core/memory/` - 对话记忆管理
- `core/model_runtime/` - 模型运行时
- `core/app/` - 应用运行器
- `core/workflow/nodes/llm/` - Workflow LLM 节点
