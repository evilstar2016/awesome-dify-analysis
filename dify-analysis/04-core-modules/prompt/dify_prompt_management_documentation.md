# Dify Prompt 管理模块深度分析

## 1. 模块概述

Prompt 管理模块位于 `api/core/prompt/` 目录下，是 Dify 平台中负责构建、转换和管理 LLM 提示词的核心组件。该模块为不同类型的应用（Chatbot、Completion、Agent、Workflow）提供了灵活的 Prompt 构建能力。

### 1.1 目录结构

```
api/core/prompt/
├── __init__.py
├── prompt_transform.py              # 基础抽象类
├── simple_prompt_transform.py       # 简单模式 Prompt 转换
├── advanced_prompt_transform.py     # 高级模式 Prompt 转换
├── agent_history_prompt_transform.py # Agent 历史 Prompt 转换
├── entities/
│   ├── __init__.py
│   └── advanced_prompt_entities.py  # 高级 Prompt 实体定义
├── prompt_templates/
│   ├── __init__.py
│   ├── advanced_prompt_templates.py # 高级模板配置
│   ├── common_chat.json            # 通用聊天模板
│   ├── common_completion.json      # 通用补全模板
│   ├── baichuan_chat.json          # 百川聊天模板
│   └── baichuan_completion.json    # 百川补全模板
└── utils/
    ├── __init__.py
    ├── prompt_template_parser.py    # 模板解析器
    ├── prompt_message_util.py       # 消息工具类
    ├── extract_thread_messages.py   # 线程消息提取
    └── get_thread_messages_length.py # 线程消息长度计算
```

## 2. 核心类设计

### 2.1 类继承关系

```
PromptTransform (Base)
    ├── SimplePromptTransform
    ├── AdvancedPromptTransform
    └── AgentHistoryPromptTransform
```

### 2.2 PromptTransform 基类

**文件**: `api/core/prompt/prompt_transform.py`

基类提供了共享的历史消息处理和 token 计算功能：

```python
class PromptTransform:
    """Prompt 转换基类"""
    
    def _append_chat_histories(
        self,
        memory: TokenBufferMemory,
        memory_config: MemoryConfig,
        prompt_messages: list[PromptMessage],
        model_config: ModelConfigWithCredentialsEntity,
    ) -> list[PromptMessage]:
        """追加聊天历史到 prompt 消息列表"""
        
    def _calculate_rest_token(
        self, 
        prompt_messages: list[PromptMessage], 
        model_config: ModelConfigWithCredentialsEntity
    ) -> int:
        """计算剩余可用 token 数量"""
        
    def _get_history_messages_from_memory(
        self,
        memory: TokenBufferMemory,
        memory_config: MemoryConfig,
        max_token_limit: int,
        human_prefix: str | None = None,
        ai_prefix: str | None = None,
    ) -> str:
        """从内存获取历史消息文本（用于 Completion 模式）"""
        
    def _get_history_messages_list_from_memory(
        self, 
        memory: TokenBufferMemory, 
        memory_config: MemoryConfig, 
        max_token_limit: int
    ) -> list[PromptMessage]:
        """从内存获取历史消息列表（用于 Chat 模式）"""
```

**关键功能**:
- **Token 计算**: 基于模型上下文大小和当前消息，计算剩余可用 token
- **历史消息管理**: 支持消息窗口限制和 token 限制
- **多模式支持**: 为 Chat 和 Completion 模式提供不同的历史消息格式

### 2.3 SimplePromptTransform 类

**文件**: `api/core/prompt/simple_prompt_transform.py`

用于 Chatbot 应用的基础模式，支持简单的预设提示词：

```python
class SimplePromptTransform(PromptTransform):
    """简单 Prompt 转换器 - 用于 Chatbot 基础模式"""
    
    def get_prompt(
        self,
        app_mode: AppMode,
        prompt_template_entity: PromptTemplateEntity,
        inputs: Mapping[str, str],
        query: str,
        files: Sequence["File"],
        context: str | None,
        memory: TokenBufferMemory | None,
        model_config: ModelConfigWithCredentialsEntity,
        image_detail_config: ImagePromptMessageContent.DETAIL | None = None,
    ) -> tuple[list[PromptMessage], list[str] | None]:
        """生成 Prompt 消息"""
```

**主要特性**:
- 根据模型模式（Chat/Completion）自动选择处理方法
- 支持从 JSON 模板文件加载 Prompt 规则
- 自动处理上下文注入和历史消息
- 支持特殊变量替换：`#context#`, `#query#`, `#histories#`

**工作流程**:
1. 判断模型模式（Chat 或 Completion）
2. 加载对应的 Prompt 模板规则
3. 解析用户自定义变量和特殊变量
4. 构建系统提示词
5. 追加聊天历史（如有）
6. 添加用户查询消息

### 2.4 AdvancedPromptTransform 类

**文件**: `api/core/prompt/advanced_prompt_transform.py`

用于 Workflow LLM 节点的高级模式：

```python
class AdvancedPromptTransform(PromptTransform):
    """高级 Prompt 转换器 - 用于 Workflow LLM 节点"""
    
    def __init__(
        self,
        with_variable_tmpl: bool = False,
        image_detail_config: ImagePromptMessageContent.DETAIL = ImagePromptMessageContent.DETAIL.LOW,
    ):
        self.with_variable_tmpl = with_variable_tmpl
        self.image_detail_config = image_detail_config

    def get_prompt(
        self,
        *,
        prompt_template: Sequence[ChatModelMessage] | CompletionModelPromptTemplate,
        inputs: Mapping[str, str],
        query: str,
        files: Sequence[File],
        context: str | None,
        memory_config: MemoryConfig | None,
        memory: TokenBufferMemory | None,
        model_config: ModelConfigWithCredentialsEntity,
        image_detail_config: ImagePromptMessageContent.DETAIL | None = None,
    ) -> list[PromptMessage]:
        """生成高级 Prompt 消息"""
```

**主要特性**:
- 支持 Chat 和 Completion 两种模型模式
- 支持 Basic 和 Jinja2 两种模板编辑类型
- 支持变量模板语法（`{{#node.variable#}}`）
- 完整的内存配置支持（角色前缀、窗口配置、查询模板）

**编辑类型**:
1. **Basic**: 使用 `PromptTemplateParser` 进行简单变量替换
2. **Jinja2**: 使用 `Jinja2Formatter` 进行高级模板渲染

### 2.5 AgentHistoryPromptTransform 类

**文件**: `api/core/prompt/agent_history_prompt_transform.py`

专门用于 Agent 应用的历史消息处理：

```python
class AgentHistoryPromptTransform(PromptTransform):
    """Agent 历史 Prompt 转换器"""
    
    def __init__(
        self,
        model_config: ModelConfigWithCredentialsEntity,
        prompt_messages: list[PromptMessage],
        history_messages: list[PromptMessage],
        memory: TokenBufferMemory | None = None,
    ):
        self.model_config = model_config
        self.prompt_messages = prompt_messages
        self.history_messages = history_messages
        self.memory = memory

    def get_prompt(self) -> list[PromptMessage]:
        """获取处理后的 Prompt 消息"""
```

**主要特性**:
- 专注于 Agent 场景的历史消息裁剪
- 保留系统消息不变
- 从后向前遍历消息，确保最新的对话优先保留
- 按完整的用户-助手对话单元进行裁剪

## 3. 实体类定义

### 3.1 ChatModelMessage

```python
class ChatModelMessage(BaseModel):
    """Chat 模型消息实体"""
    text: str                                         # 消息文本
    role: PromptMessageRole                           # 角色（user/system/assistant）
    edition_type: Literal["basic", "jinja2"] | None = None  # 编辑类型
```

### 3.2 CompletionModelPromptTemplate

```python
class CompletionModelPromptTemplate(BaseModel):
    """Completion 模型 Prompt 模板"""
    text: str                                         # 模板文本
    edition_type: Literal["basic", "jinja2"] | None = None  # 编辑类型
```

### 3.3 MemoryConfig

```python
class MemoryConfig(BaseModel):
    """内存配置"""
    
    class RolePrefix(BaseModel):
        """角色前缀配置"""
        user: str           # 用户前缀
        assistant: str      # 助手前缀

    class WindowConfig(BaseModel):
        """窗口配置"""
        enabled: bool       # 是否启用窗口限制
        size: int | None = None  # 窗口大小

    role_prefix: RolePrefix | None = None      # 角色前缀
    window: WindowConfig                        # 窗口配置
    query_prompt_template: str | None = None   # 查询 Prompt 模板
```

## 4. 工具类详解

### 4.1 PromptTemplateParser

**文件**: `api/core/prompt/utils/prompt_template_parser.py`

模板变量解析和替换引擎：

```python
# 标准变量正则：{{variable_name}} 或 {{#special#}}
REGEX = re.compile(r"\{\{([a-zA-Z_][a-zA-Z0-9_]{0,29}|#histories#|#query#|#context#)\}\}")

# 带变量模板的正则：支持 {{#node.variable#}} 格式
WITH_VARIABLE_TMPL_REGEX = re.compile(
    r"\{\{([a-zA-Z_][a-zA-Z0-9_]{0,29}|#[a-zA-Z0-9_]{1,50}\.[a-zA-Z0-9_\.]{1,100}#|#histories#|#query#|#context#)\}\}"
)

class PromptTemplateParser:
    """Prompt 模板解析器"""
    
    def __init__(self, template: str, with_variable_tmpl: bool = False):
        self.template = template
        self.with_variable_tmpl = with_variable_tmpl
        self.regex = WITH_VARIABLE_TMPL_REGEX if with_variable_tmpl else REGEX
        self.variable_keys = self.extract()  # 提取所有变量键

    def extract(self):
        """提取模板中的所有变量键"""
        return re.findall(self.regex, self.template)

    def format(self, inputs: Mapping[str, str], remove_template_variables: bool = True) -> str:
        """格式化模板，替换变量"""
```

**模板规则**:
1. 变量必须用 `{{}}` 包裹
2. 变量名：字母/下划线开头，最多 30 字符
3. 特殊变量：`{{#histories#}}`, `{{#query#}}`, `{{#context#}}`
4. Workflow 变量：`{{#node_id.variable_name#}}`

### 4.2 PromptMessageUtil

**文件**: `api/core/prompt/utils/prompt_message_util.py`

消息序列化工具：

```python
class PromptMessageUtil:
    @staticmethod
    def prompt_messages_to_prompt_for_saving(
        model_mode: str, 
        prompt_messages: Sequence[PromptMessage]
    ):
        """将 Prompt 消息转换为可保存格式"""
```

**功能**:
- 将 `PromptMessage` 对象序列化为字典格式
- 处理文本和文件内容（图片、音频等）
- 支持工具调用消息的序列化
- 文件内容会被截断以减少存储空间

### 4.3 extract_thread_messages

**文件**: `api/core/prompt/utils/extract_thread_messages.py`

线程消息提取：

```python
def extract_thread_messages(messages: Sequence[Message]) -> list[Message]:
    """
    从消息列表中提取当前线程的消息
    支持消息重新生成场景，基于 parent_message_id 追溯
    """
```

**功能**:
- 基于 `parent_message_id` 构建消息链
- 支持消息重新生成（regenerate）场景
- 确保只返回当前对话线程的消息

## 5. Prompt 模板配置

### 5.1 JSON 模板文件

**common_chat.json**（通用聊天模板）:
```json
{
  "human_prefix": "Human",
  "assistant_prefix": "Assistant",
  "context_prompt": "Use the following context as your learned knowledge...",
  "histories_prompt": "Here is the chat histories between human and assistant...",
  "system_prompt_orders": ["context_prompt", "pre_prompt", "histories_prompt"],
  "query_prompt": "\n\nHuman: {{#query#}}\n\nAssistant: ",
  "stops": ["\nHuman:", "</histories>"]
}
```

**common_completion.json**（通用补全模板）:
```json
{
  "context_prompt": "Use the following context as your learned knowledge...",
  "system_prompt_orders": ["context_prompt", "pre_prompt"],
  "query_prompt": "{{#query#}}",
  "stops": null
}
```

### 5.2 模板组装顺序

1. **context_prompt**: 上下文提示（如有）
2. **pre_prompt**: 用户预设提示词
3. **histories_prompt**: 历史对话（Chat 模式）
4. **query_prompt**: 用户查询

## 6. 调用关系与使用场景

### 6.1 应用层调用

| 应用类型 | 使用的转换器 | 调用位置 |
|---------|-------------|---------|
| Chatbot (Basic Mode) | SimplePromptTransform | BaseAppRunner |
| Chatbot (Advanced Mode) | AdvancedPromptTransform | BaseAppRunner |
| Completion App | SimplePromptTransform | BaseAppRunner |
| Workflow LLM Node | AdvancedPromptTransform | LLMNode |
| Agent (CoT) | AgentHistoryPromptTransform | CotAgentRunner |
| Agent (FC) | AgentHistoryPromptTransform | FCAgentRunner |
| Question Classifier | AdvancedPromptTransform | QuestionClassifierNode |
| Parameter Extractor | AdvancedPromptTransform | ParameterExtractorNode |

### 6.2 BaseAppRunner 中的使用

```python
# 简单模式
if prompt_template_entity.prompt_type == PromptType.SIMPLE:
    prompt_transform = SimplePromptTransform()
    prompt_messages, stop = prompt_transform.get_prompt(
        app_mode=AppMode.value_of(app_record.mode),
        prompt_template_entity=prompt_template_entity,
        inputs=inputs,
        query=query or "",
        files=files,
        context=context,
        memory=memory,
        model_config=model_config,
    )
# 高级模式
else:
    prompt_transform = AdvancedPromptTransform()
    prompt_messages = prompt_transform.get_prompt(
        prompt_template=prompt_template,
        inputs=inputs,
        query=query or "",
        files=files,
        context=context,
        memory_config=memory_config,
        memory=memory,
        model_config=model_config,
    )
```

## 7. 设计模式与架构特点

### 7.1 策略模式

`PromptTransform` 及其子类实现了策略模式：
- 定义统一的 `get_prompt` 接口
- 不同子类实现不同的 Prompt 构建策略
- 调用方无需关心具体实现

### 7.2 模板方法模式

基类提供了模板方法（如 `_append_chat_histories`），子类可以复用或覆盖。

### 7.3 关注点分离

- **实体层**: 定义数据结构（`ChatModelMessage`, `MemoryConfig`）
- **工具层**: 提供通用功能（解析、序列化）
- **转换层**: 实现业务逻辑（Prompt 构建）

### 7.4 可扩展性设计

- 支持多种模型提供商（通过模板文件）
- 支持多种编辑类型（Basic/Jinja2）
- 支持自定义变量和特殊变量

## 8. 核心流程示例

### 8.1 SimplePromptTransform 流程

1. 接收应用参数（app_mode, inputs, query, context, memory）
2. 判断模型模式（Chat/Completion）
3. 加载 Prompt 规则（从 JSON 文件）
4. 构建模板变量映射
5. 解析并替换模板变量
6. 追加历史消息（如有内存）
7. 添加用户查询
8. 返回 PromptMessage 列表

### 8.2 AdvancedPromptTransform 流程

1. 接收 Prompt 模板（ChatModelMessage[] 或 CompletionModelPromptTemplate）
2. 遍历模板消息
3. 根据 edition_type 选择解析器（PromptTemplateParser 或 Jinja2Formatter）
4. 替换变量（context, query, 自定义变量）
5. 构建对应角色的 PromptMessage
6. 追加内存中的历史消息
7. 添加查询和文件
8. 返回 PromptMessage 列表

## 9. 与其他模块的交互

### 9.1 Memory 模块

- `TokenBufferMemory`: 提供历史消息的存储和检索
- 支持 token 限制和消息数量限制
- 自动处理线程消息提取

### 9.2 Model Runtime 模块

- `PromptMessage`: 消息实体基类
- `SystemPromptMessage`, `UserPromptMessage`, `AssistantPromptMessage`: 角色消息
- `ImagePromptMessageContent`, `TextPromptMessageContent`: 内容类型

### 9.3 File 模块

- `file_manager.to_prompt_message_content()`: 将文件转换为 Prompt 内容
- 支持图片、音频等多模态内容

### 9.4 Workflow 模块

- `VariablePool`: 变量池，用于 Workflow 变量解析
- 支持 `{{#node_id.variable#}}` 格式的变量引用

## 10. 最佳实践

### 10.1 Prompt 模板设计

1. 使用清晰的变量命名
2. 合理使用特殊变量（context, histories, query）
3. 考虑 token 限制，避免过长的模板

### 10.2 内存配置

1. 根据场景选择合适的窗口大小
2. 设置合理的角色前缀（用于 Completion 模式）
3. 使用 query_prompt_template 自定义查询格式

### 10.3 性能优化

1. 模板文件会被缓存（`prompt_file_contents`）
2. 使用 token 计算避免上下文溢出
3. 合理裁剪历史消息

## 11. 总结

Dify 的 Prompt 管理模块采用了清晰的分层架构和策略模式，为不同类型的应用提供了灵活且强大的 Prompt 构建能力。通过基类抽象、模板系统和工具类的组合，实现了代码复用和功能扩展的平衡。

核心优势：
- **模式支持全面**: 覆盖 Chat/Completion 模型模式
- **模板灵活**: 支持 Basic 和 Jinja2 两种模板类型
- **内存管理智能**: 自动处理 token 限制和消息裁剪
- **扩展性强**: 易于添加新的模型提供商和模板格式
