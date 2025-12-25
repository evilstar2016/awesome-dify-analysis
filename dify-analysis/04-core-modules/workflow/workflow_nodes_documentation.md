# Dify 工作流节点完整指南

## 概述

Dify 工作流引擎提供了丰富的节点类型，用于构建复杂的 AI 应用流程。本文档详细介绍每个可用节点的功能、配置参数和使用方法。

节点代码位置：`api/core/workflow/nodes/`

## 工作流类型

Dify 支持三种工作流类型：

| 类型 | 标识 | 说明 | 可用节点 |
|-----|------|------|---------|
| **Workflow** | `workflow` | 标准工作流应用 | 常规节点 |
| **Chat** | `chat` | 对话式工作流 | 常规节点 |
| **RAG Pipeline** | `rag-pipeline` | 知识库处理流水线 | 包含特殊节点 (Datasource, Knowledge Index) |

> ⚠️ **注意**：部分节点仅在特定工作流类型中可见，详见下方"节点可见性"说明。

## 节点分类

### 按功能分类

| 分类 | 节点 | 说明 |
|-----|------|------|
| **基础节点** | Start, End, Answer | 工作流入口、出口和响应 |
| **AI 能力节点** | LLM, Knowledge Retrieval, Question Classifier, Parameter Extractor, Agent | 调用 AI 模型进行推理 |
| **逻辑控制节点** | If-Else, Iteration, Loop | 条件判断和循环控制 |
| **数据处理节点** | Code, Template Transform, Variable Aggregator, Variable Assigner, List Operator, Document Extractor | 数据转换和处理 |
| **外部集成节点** | HTTP Request, Tool | 调用外部 API 和工具 |
| **RAG Pipeline 专用节点** | Datasource, Knowledge Index | 知识库数据导入和索引 |
| **交互节点** | Human Input | 人机交互，等待人工输入 |

### 按执行类型分类

| 执行类型 | 说明 | 包含节点 |
|---------|------|---------|
| **ROOT** | 入口节点，工作流起点 | Start, Datasource |
| **EXECUTABLE** | 可执行节点，处理数据 | LLM, Code, HTTP Request, Tool, Knowledge Retrieval 等 |
| **RESPONSE** | 响应节点，支持流式输出 | Answer, End, Knowledge Index |
| **BRANCH** | 分支节点，控制流程走向 | If-Else, Question Classifier, Human Input |
| **CONTAINER** | 容器节点，管理子图执行 | Iteration, Loop |

### 节点可见性

| 节点 | 常规工作流 | RAG Pipeline | 说明 |
|-----|:----------:|:------------:|------|
| Start | ✅ | ✅ | 所有工作流 |
| End | ✅ | ✅ | 所有工作流 |
| Answer | ✅ | ✅ | 所有工作流 |
| LLM | ✅ | ✅ | 所有工作流 |
| Knowledge Retrieval | ✅ | ❌ | 仅常规工作流 |
| If-Else | ✅ | ✅ | 所有工作流 |
| Code | ✅ | ✅ | 所有工作流 |
| Template Transform | ✅ | ✅ | 所有工作流 |
| Question Classifier | ✅ | ❌ | 仅常规工作流 |
| HTTP Request | ✅ | ✅ | 所有工作流 |
| Tool | ✅ | ✅ | 所有工作流 |
| Variable Aggregator | ✅ | ✅ | 所有工作流 |
| Variable Assigner | ✅ | ✅ | 所有工作流 |
| Iteration | ✅ | ✅ | 所有工作流 |
| Loop | ✅ | ✅ | 所有工作流 |
| Parameter Extractor | ✅ | ❌ | 仅常规工作流 |
| Document Extractor | ✅ | ❌ | 仅常规工作流 |
| List Operator | ✅ | ✅ | 所有工作流 |
| Agent | ✅ | ❌ | 仅常规工作流 |
| **Datasource** | ❌ | ✅ | **仅 RAG Pipeline** |
| **Knowledge Index** | ❌ | ✅ | **仅 RAG Pipeline** |
| **Human Input** | ❌ | ❌ | **后端已实现，前端暂未开放** |

> 💡 **为什么某些节点在常规工作流中看不到？**
> 
> 1. **Datasource (数据源节点)** 和 **Knowledge Index (知识索引节点)**：
>    - 这两个节点是 **RAG Pipeline 专用节点**
>    - 用于知识库的数据导入和索引流程
>    - 只有在创建 "RAG Pipeline" 类型的工作流时才可见
>    - RAG Pipeline 是 Dify 用于处理知识库数据的特殊工作流类型
>    - 前端类型定义：`BlockEnum.DataSource = 'datasource'`，`BlockEnum.KnowledgeBase = 'knowledge-index'`
> 
> 2. **Human Input (人工输入节点)**：
>    - 后端代码已完整实现（`api/core/workflow/nodes/human_input/`）
>    - 但前端 `BlockEnum` 枚举中没有定义此节点类型
>    - 这表明该功能**尚未在前端开放**，可能是：
>      - 正在开发中的功能
>      - 企业版专属功能
>      - 需要通过配置开启的实验性功能

---

## 1. 开始节点 (Start Node)

**类型标识**: `start`

### 功能描述
工作流的入口节点，用于定义工作流的输入变量。每个工作流必须有且只有一个开始节点。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `variables` | `list[VariableEntity]` | 用户输入变量列表 |

### VariableEntity 结构

每个变量可配置：
- `variable`: 变量名称
- `label`: 显示标签
- `type`: 变量类型 (string, number, select, paragraph, files 等)
- `required`: 是否必填
- `max_length`: 最大长度
- `options`: 选项列表 (用于 select 类型)

### 输出变量

- 所有定义的用户输入变量
- 系统变量 (`sys.query`, `sys.files`, `sys.conversation_id`, `sys.user_id` 等)

### 使用示例

```yaml
type: start
variables:
  - variable: user_input
    label: 用户输入
    type: paragraph
    required: true
  - variable: language
    label: 语言选择
    type: select
    options:
      - 中文
      - English
```

---

## 2. 结束节点 (End Node)

**类型标识**: `end`

### 功能描述
工作流的终止节点，用于收集并输出最终结果。支持流式输出。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `outputs` | `list[VariableSelector]` | 输出变量选择器列表 |

### VariableSelector 结构

```python
class VariableSelector:
    variable: str           # 输出变量名
    value_selector: list[str]  # 变量选择路径 [node_id, variable_name]
```

### 使用示例

```yaml
type: end
outputs:
  - variable: result
    value_selector: [llm_node_1, text]
  - variable: metadata
    value_selector: [code_node_1, output]
```

---

## 3. 回答节点 (Answer Node)

**类型标识**: `answer`

### 功能描述
用于在工作流执行过程中输出中间结果或流式响应。支持模板变量替换和流式输出。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `answer` | `str` | 回答模板字符串，支持变量引用 |

### 模板语法

使用 `{{#node_id.variable_name#}}` 语法引用变量：

```
根据您的问题，这是我的回答：

{{#llm_node_1.text#}}

处理完成！
```

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `answer` | string | 渲染后的回答文本 |
| `files` | array[file] | 提取的文件列表 |

---

## 4. LLM 节点

**类型标识**: `llm`

### 功能描述
调用大语言模型进行推理、生成文本、对话等任务。是工作流中最核心的节点之一。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `model` | `ModelConfig` | 模型配置 |
| `prompt_template` | `list[Message]` | 提示词模板 |
| `prompt_config` | `PromptConfig` | Jinja2 变量配置 |
| `memory` | `MemoryConfig` | 对话记忆配置 |
| `context` | `ContextConfig` | 上下文配置 |
| `vision` | `VisionConfig` | 视觉能力配置 |
| `structured_output` | `dict` | 结构化输出 Schema |
| `reasoning_format` | `str` | 推理输出格式 ("separated" 或 "tagged") |

### ModelConfig 结构

```python
class ModelConfig:
    provider: str              # 模型提供商 (openai, anthropic 等)
    name: str                  # 模型名称 (gpt-4, claude-3 等)
    mode: LLMMode              # 模型模式 (chat, completion)
    completion_params: dict    # 模型参数 (temperature, max_tokens 等)
```

### 消息模板

支持 Chat 模式和 Completion 模式：

**Chat 模式**:
```yaml
prompt_template:
  - role: system
    text: 你是一个有帮助的助手
  - role: user
    text: "{{#start.user_input#}}"
```

**Jinja2 模式**:
```yaml
prompt_template:
  - role: user
    jinja2_text: |
      {% for item in items %}
      - {{ item }}
      {% endfor %}
```

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `text` | string | 模型生成的文本 |
| `reasoning_content` | string | 推理内容 (如果模型支持) |
| `usage` | object | Token 使用统计 |
| `finish_reason` | string | 完成原因 |
| `files` | array[file] | 生成的文件 (多模态输出) |

### 使用示例

```yaml
type: llm
model:
  provider: openai
  name: gpt-4o
  mode: chat
  completion_params:
    temperature: 0.7
    max_tokens: 2000
prompt_template:
  - role: system
    text: 你是一个专业的翻译助手
  - role: user
    text: "请将以下文本翻译成英文：{{#start.text#}}"
context:
  enabled: true
  variable_selector: [knowledge_retrieval_1, result]
vision:
  enabled: true
  configs:
    detail: high
```

---

## 5. 知识检索节点 (Knowledge Retrieval Node)

**类型标识**: `knowledge-retrieval`

### 功能描述
从知识库中检索相关文档片段，支持语义检索、关键词检索和混合检索。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `query_variable_selector` | `list[str]` | 查询变量选择器 |
| `dataset_ids` | `list[str]` | 知识库 ID 列表 |
| `retrieval_mode` | `str` | 检索模式 (single/multiple) |
| `single_retrieval_config` | `SingleRetrievalConfig` | 单库检索配置 |
| `multiple_retrieval_config` | `MultipleRetrievalConfig` | 多库检索配置 |
| `metadata_filtering_conditions` | `list[Condition]` | 元数据过滤条件 |

### 检索配置

**MultipleRetrievalConfig**:
```python
class MultipleRetrievalConfig:
    top_k: int                          # 返回结果数量
    score_threshold: float              # 分数阈值
    reranking_mode: str                 # 重排序模式
    reranking_enable: bool              # 是否启用重排序
    reranking_model: RerankingModelConfig  # 重排序模型配置
    weights: WeightedScoreConfig        # 混合检索权重
```

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `result` | array[object] | 检索结果列表 |
| `result[].content` | string | 文档内容 |
| `result[].title` | string | 文档标题 |
| `result[].score` | float | 相关性分数 |
| `result[].metadata` | object | 元数据 |

### 使用示例

```yaml
type: knowledge-retrieval
query_variable_selector: [start, user_input]
dataset_ids:
  - dataset_123
  - dataset_456
retrieval_mode: multiple
multiple_retrieval_config:
  top_k: 5
  score_threshold: 0.5
  reranking_enable: true
  reranking_model:
    provider: cohere
    model: rerank-english-v2.0
```

---

## 6. 条件分支节点 (If-Else Node)

**类型标识**: `if-else`

### 功能描述
根据条件判断选择不同的执行分支，支持多条件组合和多分支。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `cases` | `list[Case]` | 条件分支列表 |
| `logical_operator` | `str` | 逻辑运算符 (and/or) - 已废弃 |

### Case 结构

```python
class Case:
    case_id: str                    # 分支唯一标识
    logical_operator: Literal["and", "or"]  # 条件间逻辑关系
    conditions: list[Condition]     # 条件列表
```

### Condition 结构

```python
class Condition:
    variable_selector: list[str]    # 变量选择器
    comparison_operator: str        # 比较运算符
    value: Any                      # 比较值
```

### 支持的比较运算符

| 运算符 | 说明 | 适用类型 |
|-------|------|---------|
| `contains` | 包含 | string, array |
| `not contains` | 不包含 | string, array |
| `start with` | 开头是 | string |
| `end with` | 结尾是 | string |
| `is` | 等于 | string |
| `is not` | 不等于 | string |
| `empty` | 为空 | string, array |
| `not empty` | 不为空 | string, array |
| `=` | 等于 | number |
| `≠` | 不等于 | number |
| `>` | 大于 | number |
| `<` | 小于 | number |
| `≥` | 大于等于 | number |
| `≤` | 小于等于 | number |

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `result` | boolean | 条件判断结果 |
| `selected_case_id` | string | 选中的分支 ID |

### 使用示例

```yaml
type: if-else
cases:
  - case_id: positive
    logical_operator: and
    conditions:
      - variable_selector: [llm_1, text]
        comparison_operator: contains
        value: 积极
  - case_id: negative
    logical_operator: and
    conditions:
      - variable_selector: [llm_1, text]
        comparison_operator: contains
        value: 消极
# 默认分支 case_id: false
```

---

## 7. 代码节点 (Code Node)

**类型标识**: `code`

### 功能描述
执行自定义 Python 或 JavaScript 代码，用于数据处理、转换和复杂逻辑实现。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `code_language` | `str` | 编程语言 (python3, javascript) |
| `code` | `str` | 代码内容 |
| `variables` | `list[VariableSelector]` | 输入变量列表 |
| `outputs` | `dict[str, Output]` | 输出变量定义 |
| `dependencies` | `list[Dependency]` | 依赖包列表 |

### 支持的输出类型

- `string` - 字符串
- `number` - 数字
- `boolean` - 布尔值
- `object` - 对象
- `array[string]` - 字符串数组
- `array[number]` - 数字数组
- `array[boolean]` - 布尔数组
- `array[object]` - 对象数组

### Python 代码模板

```python
def main(arg1: str, arg2: dict) -> dict:
    # 处理逻辑
    result = process_data(arg1, arg2)
    
    return {
        "output": result,
        "count": len(result)
    }
```

### JavaScript 代码模板

```javascript
function main(arg1, arg2) {
    // 处理逻辑
    const result = processData(arg1, arg2);
    
    return {
        output: result,
        count: result.length
    };
}
```

### 使用示例

```yaml
type: code
code_language: python3
variables:
  - variable: text
    value_selector: [llm_1, text]
  - variable: config
    value_selector: [start, config]
code: |
  import json
  
  def main(text: str, config: dict) -> dict:
      words = text.split()
      word_count = len(words)
      
      return {
          "word_count": word_count,
          "words": words[:10]
      }
outputs:
  word_count:
    type: number
  words:
    type: array[string]
```

---

## 8. HTTP 请求节点 (HTTP Request Node)

**类型标识**: `http-request`

### 功能描述
发送 HTTP 请求到外部 API，支持各种请求方法、认证方式和请求体格式。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `method` | `str` | 请求方法 (GET, POST, PUT, DELETE 等) |
| `url` | `str` | 请求 URL (支持变量) |
| `authorization` | `HttpRequestNodeAuthorization` | 认证配置 |
| `headers` | `str` | 请求头 (Key: Value 格式) |
| `params` | `str` | URL 参数 (Key: Value 格式) |
| `body` | `HttpRequestNodeBody` | 请求体配置 |
| `timeout` | `HttpRequestNodeTimeout` | 超时配置 |
| `ssl_verify` | `bool` | 是否验证 SSL 证书 |

### 认证类型

```python
class HttpRequestNodeAuthorization:
    type: Literal["no-auth", "api-key"]
    config: HttpRequestNodeAuthorizationConfig  # 包含 type, api_key, header
```

### 请求体类型

- `none` - 无请求体
- `form-data` - 表单数据
- `x-www-form-urlencoded` - URL 编码表单
- `raw-text` - 原始文本
- `json` - JSON 格式
- `binary` - 二进制数据

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `status_code` | number | HTTP 状态码 |
| `body` | string | 响应体 |
| `headers` | object | 响应头 |
| `files` | array[file] | 响应文件 (如果是文件下载) |

### 使用示例

```yaml
type: http-request
method: POST
url: "https://api.example.com/v1/process"
authorization:
  type: api-key
  config:
    type: bearer
    api_key: "{{#start.api_key#}}"
headers: |
  Content-Type: application/json
  X-Custom-Header: value
body:
  type: json
  data:
    - key: input
      type: text
      value: "{{#llm_1.text#}}"
timeout:
  connect: 10
  read: 60
  write: 30
```

---

## 9. 工具节点 (Tool Node)

**类型标识**: `tool`

### 功能描述
调用内置工具或插件工具，如 Google 搜索、天气查询、代码执行等。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `provider_id` | `str` | 工具提供者 ID |
| `provider_type` | `ToolProviderType` | 提供者类型 (builtin, api, workflow) |
| `provider_name` | `str` | 提供者名称 |
| `tool_name` | `str` | 工具名称 |
| `tool_label` | `str` | 工具标签 |
| `tool_configurations` | `dict` | 工具配置 |
| `tool_parameters` | `dict[str, ToolInput]` | 工具参数 |

### ToolInput 结构

```python
class ToolInput:
    value: Any                              # 参数值
    type: Literal["mixed", "variable", "constant"]  # 输入类型
```

### 输出变量

根据工具不同，输出变量也不同。通常包含：

| 变量 | 类型 | 说明 |
|-----|------|------|
| `text` | string | 工具返回的文本 |
| `files` | array[file] | 工具返回的文件 |
| `json` | array[object] | 工具返回的 JSON 数据 |

### 使用示例

```yaml
type: tool
provider_id: google
provider_type: builtin
provider_name: google
tool_name: google_search
tool_label: Google 搜索
tool_parameters:
  query:
    type: variable
    value: [start, search_query]
  num_results:
    type: constant
    value: 5
```

---

## 10. 问题分类器节点 (Question Classifier Node)

**类型标识**: `question-classifier`

### 功能描述
使用 LLM 对用户问题进行意图分类，根据分类结果选择不同的执行分支。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `query_variable_selector` | `list[str]` | 查询变量选择器 |
| `model` | `ModelConfig` | 分类使用的模型 |
| `classes` | `list[ClassConfig]` | 分类类别列表 |
| `instruction` | `str` | 额外分类指令 |
| `memory` | `MemoryConfig` | 对话记忆配置 |
| `vision` | `VisionConfig` | 视觉能力配置 |

### ClassConfig 结构

```python
class ClassConfig:
    id: str      # 类别唯一标识 (用作分支 handle)
    name: str    # 类别名称/描述
```

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `class_name` | string | 分类结果名称 |
| `class_id` | string | 分类结果 ID |

### 使用示例

```yaml
type: question-classifier
query_variable_selector: [start, user_input]
model:
  provider: openai
  name: gpt-4o-mini
  mode: chat
classes:
  - id: product_inquiry
    name: 产品咨询
  - id: tech_support
    name: 技术支持
  - id: complaint
    name: 投诉建议
  - id: other
    name: 其他问题
instruction: |
  请仔细分析用户的问题，判断其意图类别。
  如果无法确定，请选择"其他问题"。
```

---

## 11. 模板转换节点 (Template Transform Node)

**类型标识**: `template-transform`

### 功能描述
使用 Jinja2 模板引擎进行文本转换，支持复杂的模板逻辑。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `variables` | `list[VariableSelector]` | 模板变量列表 |
| `template` | `str` | Jinja2 模板内容 |

### Jinja2 语法示例

```jinja2
{% for item in items %}
- 项目: {{ item.name }}
  描述: {{ item.description }}
{% endfor %}

{% if condition %}
满足条件的内容
{% else %}
不满足条件的内容
{% endif %}

总数: {{ items | length }}
```

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `output` | string | 模板渲染结果 |

### 使用示例

```yaml
type: template-transform
variables:
  - variable: results
    value_selector: [knowledge_retrieval_1, result]
  - variable: query
    value_selector: [start, user_input]
template: |
  用户问题: {{ query }}
  
  相关文档:
  {% for doc in results %}
  ---
  来源: {{ doc.title }}
  内容: {{ doc.content }}
  {% endfor %}
```

---

## 12. 迭代节点 (Iteration Node)

**类型标识**: `iteration`

### 功能描述
对数组类型数据进行循环迭代处理，每次迭代可以执行复杂的子流程。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `iterator_selector` | `list[str]` | 迭代器变量选择器 |
| `output_selector` | `list[str]` | 输出变量选择器 |
| `is_parallel` | `bool` | 是否并行执行 |
| `parallel_nums` | `int` | 并行数量 (默认 10) |
| `error_handle_mode` | `ErrorHandleMode` | 错误处理模式 |
| `flatten_output` | `bool` | 是否展平输出数组 |
| `start_node_id` | `str` | 迭代子图的起始节点 |

### 错误处理模式

| 模式 | 说明 |
|-----|------|
| `terminated` | 遇到错误终止整个迭代 |
| `continue-on-error` | 忽略错误继续执行 |
| `remove-abnormal-output` | 移除异常输出，继续执行 |

### 特殊节点

- **iteration-start**: 迭代起始节点，提供当前迭代项
- 在迭代内部可通过 `[iteration_node_id, item]` 访问当前项
- 通过 `[iteration_node_id, index]` 访问当前索引

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `output` | array | 所有迭代的输出结果 |

### 使用示例

```yaml
type: iteration
iterator_selector: [code_1, items]
output_selector: [llm_inside, text]
is_parallel: true
parallel_nums: 5
error_handle_mode: continue-on-error
flatten_output: true
# 内部包含 iteration-start 节点和处理节点
```

---

## 13. 循环节点 (Loop Node)

**类型标识**: `loop`

### 功能描述
基于条件的循环执行，支持 while 循环模式，可定义循环变量和终止条件。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `loop_count` | `int` | 最大循环次数 |
| `break_conditions` | `list[Condition]` | 终止条件列表 |
| `logical_operator` | `str` | 条件逻辑运算符 (and/or) |
| `loop_variables` | `list[LoopVariableData]` | 循环变量定义 |
| `outputs` | `dict` | 输出变量定义 |

### LoopVariableData 结构

```python
class LoopVariableData:
    label: str                              # 变量标签
    var_type: SegmentType                   # 变量类型
    value_type: Literal["variable", "constant"]  # 值类型
    value: Any                              # 初始值或变量引用
```

### 特殊节点

- **loop-start**: 循环起始节点
- **loop-end**: 循环结束节点

### 使用示例

```yaml
type: loop
loop_count: 10
break_conditions:
  - variable_selector: [loop_1, result]
    comparison_operator: "="
    value: "done"
logical_operator: or
loop_variables:
  - label: counter
    var_type: number
    value_type: constant
    value: 0
```

---

## 14. 变量聚合节点 (Variable Aggregator Node)

**类型标识**: `variable-aggregator`

### 功能描述
从多个分支中选择第一个有值的变量进行输出，常用于分支合并。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `output_type` | `str` | 输出类型 |
| `variables` | `list[list[str]]` | 变量选择器列表 |
| `advanced_settings` | `AdvancedSettings` | 高级设置 (分组) |

### 高级设置 - 分组

```python
class AdvancedSettings:
    group_enabled: bool     # 是否启用分组
    groups: list[Group]     # 分组列表
    
class Group:
    output_type: SegmentType    # 输出类型
    variables: list[list[str]]  # 变量列表
    group_name: str             # 分组名称
```

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `output` | any | 聚合后的变量值 |

### 使用示例

```yaml
type: variable-aggregator
output_type: string
variables:
  - [branch_1, result]
  - [branch_2, result]
  - [branch_3, result]
# 返回第一个非空的 result
```

---

## 15. 变量赋值节点 (Variable Assigner Node)

**类型标识**: `assigner`

### 功能描述
对会话变量进行赋值操作，支持多种操作类型。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `items` | `list[VariableOperationItem]` | 操作项列表 |

### VariableOperationItem 结构

```python
class VariableOperationItem:
    variable_selector: Sequence[str]    # 目标变量选择器
    input_type: InputType               # 输入类型 (VARIABLE, CONSTANT)
    operation: Operation                # 操作类型
    value: Any                          # 值
```

### 支持的操作

| 操作 | 说明 | 适用类型 |
|-----|------|---------|
| `overwrite` | 覆盖 | 所有类型 |
| `clear` | 清空 | 所有类型 |
| `append` | 追加 | array, string |
| `extend` | 扩展 | array |
| `set` | 设置 (对象属性) | object |
| `add` | 加法 | number |
| `subtract` | 减法 | number |
| `multiply` | 乘法 | number |
| `divide` | 除法 | number |

### 使用示例

```yaml
type: assigner
items:
  - variable_selector: [conversation, message_history]
    input_type: VARIABLE
    operation: append
    value: [llm_1, text]
  - variable_selector: [conversation, turn_count]
    input_type: CONSTANT
    operation: add
    value: 1
```

---

## 16. 参数提取节点 (Parameter Extractor Node)

**类型标识**: `parameter-extractor`

### 功能描述
使用 LLM 从文本中提取结构化参数，支持 Function Calling 和 JSON 模式。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `query_variable_selector` | `list[str]` | 输入变量选择器 |
| `model` | `ModelConfig` | 使用的模型 |
| `parameters` | `list[ParameterConfig]` | 参数定义列表 |
| `instruction` | `str` | 提取指令 |
| `memory` | `MemoryConfig` | 对话记忆配置 |
| `reasoning_mode` | `str` | 推理模式 |

### ParameterConfig 结构

```python
class ParameterConfig:
    name: str               # 参数名称
    type: SegmentType       # 参数类型
    description: str        # 参数描述
    required: bool          # 是否必填
    options: list[str]      # 可选值 (用于 select 类型)
```

### 支持的参数类型

- `string` - 字符串
- `number` - 数字
- `boolean` - 布尔值
- `array[string]` - 字符串数组
- `array[number]` - 数字数组
- `array[object]` - 对象数组
- `array[boolean]` - 布尔数组

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `__is_success` | boolean | 提取是否成功 |
| `__reason` | string | 失败原因 |
| `{param_name}` | any | 提取的参数值 |

### 使用示例

```yaml
type: parameter-extractor
query_variable_selector: [start, user_input]
model:
  provider: openai
  name: gpt-4o
  mode: chat
parameters:
  - name: product_name
    type: string
    description: 产品名称
    required: true
  - name: quantity
    type: number
    description: 购买数量
    required: true
  - name: delivery_date
    type: string
    description: 期望送达日期
    required: false
instruction: |
  从用户的订单描述中提取产品信息。
  如果用户没有指定数量，默认为 1。
```

---

## 17. 文档提取节点 (Document Extractor Node)

**类型标识**: `document-extractor`

### 功能描述
从上传的文件中提取文本内容，支持多种文档格式。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `variable_selector` | `list[str]` | 文件变量选择器 |

### 支持的文件格式

| 格式 | 扩展名 |
|-----|--------|
| 纯文本 | .txt, .md, .markdown |
| PDF | .pdf |
| Word | .doc, .docx |
| Excel | .xls, .xlsx, .csv |
| PowerPoint | .ppt, .pptx |
| 电子书 | .epub |
| 邮件 | .eml, .msg |
| 字幕 | .vtt |
| 配置 | .json, .yaml, .yml, .properties |

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `text` | string/array[string] | 提取的文本内容 |

### 使用示例

```yaml
type: document-extractor
variable_selector: [start, files]
# 如果输入是单个文件，输出 text 为字符串
# 如果输入是文件数组，输出 text 为字符串数组
```

---

## 18. 列表操作节点 (List Operator Node)

**类型标识**: `list-operator`

### 功能描述
对数组数据进行过滤、排序、限制数量和提取操作。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `variable` | `list[str]` | 输入数组变量选择器 |
| `filter_by` | `FilterBy` | 过滤配置 |
| `order_by` | `OrderByConfig` | 排序配置 |
| `limit` | `Limit` | 数量限制配置 |
| `extract_by` | `ExtractConfig` | 提取配置 |

### FilterBy 结构

```python
class FilterBy:
    enabled: bool                       # 是否启用
    conditions: list[FilterCondition]   # 过滤条件

class FilterCondition:
    key: str                    # 过滤字段
    comparison_operator: FilterOperator  # 比较运算符
    value: str | list[str] | bool       # 比较值
```

### 过滤运算符

**字符串条件**:
- `contains` / `not contains`
- `start with` / `end with`
- `is` / `is not`
- `in` / `not in`
- `empty` / `not empty`

**数字条件**:
- `=` / `≠`
- `<` / `>` / `≤` / `≥`

### OrderByConfig 结构

```python
class OrderByConfig:
    enabled: bool       # 是否启用
    key: str           # 排序字段
    value: Order       # 排序方向 (asc/desc)
```

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `result` | array | 处理后的数组 |
| `first_record` | any | 第一条记录 |
| `last_record` | any | 最后一条记录 |

### 使用示例

```yaml
type: list-operator
variable: [knowledge_retrieval_1, result]
filter_by:
  enabled: true
  conditions:
    - key: score
      comparison_operator: ">"
      value: "0.8"
order_by:
  enabled: true
  key: score
  value: desc
limit:
  enabled: true
  size: 5
```

---

## 19. Agent 节点

**类型标识**: `agent`

### 功能描述
调用智能代理策略，支持多轮工具调用和复杂任务处理。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `agent_strategy_provider_name` | `str` | 代理策略提供者 |
| `agent_strategy_name` | `str` | 代理策略名称 |
| `agent_strategy_label` | `str` | 代理策略标签 |
| `memory` | `MemoryConfig` | 对话记忆配置 |
| `agent_parameters` | `dict[str, AgentInput]` | 代理参数 |

### AgentInput 结构

```python
class AgentInput:
    value: Any                              # 参数值
    type: Literal["mixed", "variable", "constant"]  # 输入类型
```

### 输出变量

| 变量 | 类型 | 说明 |
|-----|------|------|
| `text` | string | Agent 响应文本 |
| `files` | array[file] | 生成的文件 |
| `usage` | object | Token 使用统计 |

### 使用示例

```yaml
type: agent
agent_strategy_provider_name: langgenius
agent_strategy_name: function_call
agent_parameters:
  query:
    type: variable
    value: [start, user_input]
  tools:
    type: constant
    value:
      - provider: google
        tool: google_search
      - provider: calculator
        tool: calculate
memory:
  role_prefix:
    user: "用户"
    assistant: "助手"
  window:
    enabled: true
    size: 10
```

---

## 20. 人工输入节点 (Human Input Node)

**类型标识**: `human-input`

### 功能描述
暂停工作流执行，等待人工审核或输入，支持分支选择。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `required_variables` | `list[str]` | 必需的输入变量列表 |
| `pause_reason` | `str` | 暂停原因说明 |

### 工作流程

1. 工作流执行到该节点时暂停
2. 用户可以查看当前状态并提供输入
3. 用户选择分支或确认继续
4. 工作流恢复执行

### 分支选择

通过以下键名传递分支选择：
- `edge_source_handle`
- `selected_branch`
- `branch`
- `branch_id`

### 使用示例

```yaml
type: human-input
required_variables:
  - "human_input.approval"
  - "human_input.comments"
pause_reason: 请审核 AI 生成的内容，确认是否正确。
```

---

## 21. 数据源节点 (Datasource Node)

**类型标识**: `datasource`

### 功能描述
从外部数据源获取数据，支持在线文档、云存储等。用于 RAG Pipeline。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `plugin_id` | `str` | 插件 ID |
| `provider_name` | `str` | 提供者名称 |
| `provider_type` | `str` | 提供者类型 |
| `datasource_name` | `str` | 数据源名称 |
| `datasource_configurations` | `dict` | 数据源配置 |
| `datasource_parameters` | `dict[str, DatasourceInput]` | 数据源参数 |

### 支持的数据源类型

- 本地文件
- 在线文档 (Notion, Google Docs 等)
- 云存储 (Google Drive, Dropbox 等)
- 网页抓取

### 使用示例

```yaml
type: datasource
plugin_id: notion
provider_name: notion
provider_type: online_document
datasource_name: notion_page
datasource_configurations:
  workspace_id: "xxx"
datasource_parameters:
  page_id:
    type: variable
    value: [start, page_id]
```

---

## 22. 知识索引节点 (Knowledge Index Node)

**类型标识**: `knowledge-index`

### 功能描述
将数据写入知识库，支持多种索引方式。用于 RAG Pipeline。

### 配置参数

| 参数 | 类型 | 说明 |
|-----|------|------|
| `index_chunk_variable_selector` | `list[str]` | 分块变量选择器 |
| `chunk_structure` | `str` | 分块结构类型 |
| `index_method` | `IndexMethod` | 索引方式配置 |
| `retrieval_setting` | `RetrievalSetting` | 检索设置 |

### 索引方式

```python
class IndexMethod:
    indexing_technique: Literal["high_quality", "economy"]
    embedding_setting: EmbeddingSetting
    economy_setting: EconomySetting
```

### 使用示例

```yaml
type: knowledge-index
index_chunk_variable_selector: [code_1, chunks]
chunk_structure: general
index_method:
  indexing_technique: high_quality
  embedding_setting:
    embedding_provider_name: openai
    embedding_model_name: text-embedding-3-small
retrieval_setting:
  search_method: semantic_search
  top_k: 4
  score_threshold: 0.5
```

---

## 错误处理

所有节点都支持统一的错误处理配置：

### 错误策略

| 策略 | 说明 |
|-----|------|
| `fail-branch` | 执行失败分支 |
| `default-value` | 使用默认值 |

### 重试配置

```python
class RetryConfig:
    max_retries: int        # 最大重试次数
    retry_interval: float   # 重试间隔 (秒)
    retry_enabled: bool     # 是否启用重试
```

### 配置示例

```yaml
error_strategy: fail-branch
retry_config:
  retry_enabled: true
  max_retries: 3
  retry_interval: 1.0
```

---

## 系统变量

在工作流中可以访问以下系统变量：

| 变量路径 | 说明 |
|---------|------|
| `sys.query` | 用户查询 |
| `sys.files` | 上传的文件 |
| `sys.conversation_id` | 会话 ID |
| `sys.user_id` | 用户 ID |
| `sys.dialogue_count` | 对话轮次 |
| `sys.app_id` | 应用 ID |
| `sys.workflow_id` | 工作流 ID |
| `sys.workflow_run_id` | 工作流执行 ID |

---

## 最佳实践

### 1. 变量引用

- 使用 `{{#node_id.variable#}}` 在模板中引用变量
- 使用 `[node_id, variable]` 作为变量选择器

### 2. 错误处理

- 对关键节点配置重试策略
- 使用 If-Else 节点处理异常情况
- 为 HTTP 请求配置超时

### 3. 性能优化

- 使用迭代节点的并行模式处理大量数据
- 合理配置知识检索的 top_k 和分数阈值
- 避免在循环中进行不必要的 LLM 调用

### 4. 模块化设计

- 将复杂逻辑封装在代码节点中
- 使用工具节点调用外部服务
- 利用模板转换节点格式化输出

---

## 版本信息

- 文档版本：1.0
- 更新日期：2024-12
- 适用于：Dify v1.x

