# Dify 扩展新模型提供商开发指南

本文档详细介绍如何为 Dify 扩展新的模型提供商（厂商），包含完整的示例代码和最佳实践。

## 1. 概述

从 Dify 的新版本开始，模型提供商通过**插件系统**进行管理。扩展新模型需要创建一个 Dify 插件，该插件包含：

1. **提供商定义** - 描述提供商的基本信息和凭据配置
2. **模型定义** - 定义支持的模型及其参数
3. **模型实现** - 实现具体的模型调用逻辑

## 2. 插件项目结构

一个完整的模型提供商插件结构如下：

```
my-model-provider/
├── manifest.yaml              # 插件清单文件
├── README.md                  # 说明文档
├── requirements.txt           # Python 依赖
├── provider/
│   ├── __init__.py
│   ├── my_provider.yaml       # 提供商定义
│   ├── my_provider.py         # 提供商实现
│   ├── _assets/               # 图标资源
│   │   ├── icon_small.png
│   │   └── icon_large.png
│   └── models/
│       ├── __init__.py
│       ├── llm/
│       │   ├── __init__.py
│       │   ├── llm.py         # LLM 模型实现
│       │   └── gpt-4.yaml     # 模型定义
│       ├── text_embedding/
│       │   ├── __init__.py
│       │   ├── text_embedding.py
│       │   └── embedding-v1.yaml
│       └── rerank/
│           ├── __init__.py
│           ├── rerank.py
│           └── rerank-v1.yaml
└── tests/
    └── test_provider.py
```

## 3. 提供商定义 (YAML)

### 3.1 提供商配置文件

`provider/my_provider.yaml`:

```yaml
# 提供商标识符
provider: my_provider

# 显示标签（支持多语言）
label:
  en_US: My AI Provider
  zh_Hans: 我的 AI 提供商

# 描述
description:
  en_US: A custom AI model provider for Dify
  zh_Hans: Dify 的自定义 AI 模型提供商

# 图标配置
icon_small:
  en_US: _assets/icon_small.png
  zh_Hans: _assets/icon_small.png
icon_large:
  en_US: _assets/icon_large.png
  zh_Hans: _assets/icon_large.png

# 背景颜色（可选）
background: "#FFFFFF"

# 帮助链接
help:
  title:
    en_US: Get your API Key from My Provider
    zh_Hans: 从 My Provider 获取 API Key
  url:
    en_US: https://my-provider.com/api-keys
    zh_Hans: https://my-provider.com/api-keys

# 支持的模型类型
supported_model_types:
  - llm
  - text-embedding
  - rerank

# 配置方式
configurate_methods:
  - predefined-model      # 预定义模型
  - customizable-model    # 自定义模型（可选）

# 提供商凭据配置
provider_credential_schema:
  credential_form_schemas:
    - variable: api_key
      label:
        en_US: API Key
        zh_Hans: API 密钥
      type: secret-input
      required: true
      placeholder:
        en_US: Enter your API Key
        zh_Hans: 输入您的 API 密钥
        
    - variable: api_base
      label:
        en_US: API Base URL
        zh_Hans: API 基础 URL
      type: text-input
      required: false
      default: https://api.my-provider.com
      placeholder:
        en_US: Enter custom API base URL
        zh_Hans: 输入自定义 API 基础 URL

# 模型凭据配置（用于自定义模型）
model_credential_schema:
  model:
    label:
      en_US: Model Name
      zh_Hans: 模型名称
    placeholder:
      en_US: Enter model name
      zh_Hans: 输入模型名称
  credential_form_schemas:
    - variable: api_key
      label:
        en_US: API Key
        zh_Hans: API 密钥
      type: secret-input
      required: true
    - variable: context_size
      label:
        en_US: Context Size
        zh_Hans: 上下文大小
      type: text-input
      required: false
      default: "4096"
```

### 3.2 模型定义文件

`provider/models/llm/gpt-4.yaml`:

```yaml
# 模型标识符
model: my-gpt-4

# 显示标签
label:
  en_US: My GPT-4
  zh_Hans: 我的 GPT-4

# 模型类型
model_type: llm

# 模型特性
features:
  - tool-call          # 工具调用
  - multi-tool-call    # 多工具调用
  - agent-thought      # Agent 思考
  - stream-tool-call   # 流式工具调用
  - vision             # 视觉能力（可选）

# 模型属性
model_properties:
  mode: chat           # chat 或 completion
  context_size: 128000 # 上下文窗口大小

# 参数规则
parameter_rules:
  - name: temperature
    use_template: temperature    # 使用内置模板
    label:
      en_US: Temperature
      zh_Hans: 温度
    type: float
    default: 0.7
    min: 0
    max: 2
    precision: 1

  - name: max_tokens
    use_template: max_tokens
    label:
      en_US: Max Tokens
      zh_Hans: 最大 Token 数
    type: int
    default: 4096
    min: 1
    max: 128000

  - name: top_p
    use_template: top_p
    label:
      en_US: Top P
      zh_Hans: Top P
    type: float
    default: 1
    min: 0
    max: 1
    precision: 2

  - name: presence_penalty
    use_template: presence_penalty
    label:
      en_US: Presence Penalty
      zh_Hans: 存在惩罚
    type: float
    default: 0
    min: -2
    max: 2
    precision: 1

  - name: frequency_penalty
    use_template: frequency_penalty
    label:
      en_US: Frequency Penalty
      zh_Hans: 频率惩罚
    type: float
    default: 0
    min: -2
    max: 2
    precision: 1

# 定价配置
pricing:
  input: "0.00003"     # 每 token 输入价格
  output: "0.00006"    # 每 token 输出价格
  unit: "0.001"        # 价格单位（千 token）
  currency: USD
```

## 4. 模型实现

### 4.1 LLM 模型实现

`provider/models/llm/llm.py`:

```python
"""
LLM 模型实现示例

该文件展示了如何实现一个完整的 LLM 模型提供商。
"""

import json
import logging
from collections.abc import Generator
from decimal import Decimal
from typing import Optional, Union

import requests

from dify_plugin.entities.model import (
    AIModelEntity,
    FetchFrom,
    ModelPropertyKey,
    ModelType,
    ParameterRule,
    ParameterType,
)
from dify_plugin.entities.model.llm import (
    LLMMode,
    LLMResult,
    LLMResultChunk,
    LLMResultChunkDelta,
    LLMUsage,
)
from dify_plugin.entities.model.message import (
    AssistantPromptMessage,
    PromptMessage,
    PromptMessageRole,
    PromptMessageTool,
    SystemPromptMessage,
    ToolPromptMessage,
    UserPromptMessage,
)
from dify_plugin.errors.model import (
    CredentialsValidateFailedError,
    InvokeAuthorizationError,
    InvokeBadRequestError,
    InvokeConnectionError,
    InvokeError,
    InvokeRateLimitError,
    InvokeServerUnavailableError,
)
from dify_plugin.interfaces.model.large_language_model import LargeLanguageModel

logger = logging.getLogger(__name__)


class MyProviderLargeLanguageModel(LargeLanguageModel):
    """
    自定义 LLM 模型实现类
    
    继承 LargeLanguageModel 基类，实现核心调用方法。
    """

    def _invoke(
        self,
        model: str,
        credentials: dict,
        prompt_messages: list[PromptMessage],
        model_parameters: dict,
        tools: Optional[list[PromptMessageTool]] = None,
        stop: Optional[list[str]] = None,
        stream: bool = True,
        user: Optional[str] = None,
    ) -> Union[LLMResult, Generator[LLMResultChunk, None, None]]:
        """
        调用 LLM 模型的核心方法
        
        Args:
            model: 模型名称
            credentials: 凭据信息，包含 api_key 等
            prompt_messages: 提示消息列表
            model_parameters: 模型参数，如 temperature, max_tokens 等
            tools: 工具列表（用于 function calling）
            stop: 停止词列表
            stream: 是否流式返回
            user: 用户标识
            
        Returns:
            流式返回 Generator[LLMResultChunk]
            非流式返回 LLMResult
        """
        # 1. 获取凭据
        api_key = credentials.get("api_key")
        api_base = credentials.get("api_base", "https://api.my-provider.com")
        
        if not api_key:
            raise InvokeAuthorizationError("API Key is required")

        # 2. 构建请求
        headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        }
        
        # 3. 转换消息格式
        messages = self._convert_prompt_messages(prompt_messages)
        
        # 4. 构建请求体
        payload = {
            "model": model,
            "messages": messages,
            "stream": stream,
            **model_parameters,
        }
        
        # 添加工具（如果有）
        if tools:
            payload["tools"] = self._convert_tools(tools)
            
        # 添加停止词（如果有）
        if stop:
            payload["stop"] = stop

        # 5. 发送请求
        try:
            response = requests.post(
                f"{api_base}/v1/chat/completions",
                headers=headers,
                json=payload,
                stream=stream,
                timeout=600,
            )
            response.raise_for_status()
        except requests.exceptions.ConnectionError as e:
            raise InvokeConnectionError(str(e))
        except requests.exceptions.Timeout as e:
            raise InvokeConnectionError(f"Request timeout: {e}")
        except requests.exceptions.HTTPError as e:
            self._handle_http_error(e.response)

        # 6. 处理响应
        if stream:
            return self._handle_stream_response(model, response, prompt_messages)
        else:
            return self._handle_sync_response(model, response, prompt_messages)

    def _convert_prompt_messages(self, prompt_messages: list[PromptMessage]) -> list[dict]:
        """将 PromptMessage 转换为 API 请求格式"""
        messages = []
        
        for message in prompt_messages:
            if isinstance(message, SystemPromptMessage):
                messages.append({
                    "role": "system",
                    "content": message.content,
                })
            elif isinstance(message, UserPromptMessage):
                # 处理多模态内容
                if isinstance(message.content, list):
                    content_parts = []
                    for item in message.content:
                        if item.type == "text":
                            content_parts.append({
                                "type": "text",
                                "text": item.data,
                            })
                        elif item.type == "image":
                            content_parts.append({
                                "type": "image_url",
                                "image_url": {"url": item.data},
                            })
                    messages.append({
                        "role": "user",
                        "content": content_parts,
                    })
                else:
                    messages.append({
                        "role": "user",
                        "content": message.content,
                    })
            elif isinstance(message, AssistantPromptMessage):
                msg = {
                    "role": "assistant",
                    "content": message.content or "",
                }
                # 添加工具调用
                if message.tool_calls:
                    msg["tool_calls"] = [
                        {
                            "id": tc.id,
                            "type": tc.type,
                            "function": {
                                "name": tc.function.name,
                                "arguments": tc.function.arguments,
                            },
                        }
                        for tc in message.tool_calls
                    ]
                messages.append(msg)
            elif isinstance(message, ToolPromptMessage):
                messages.append({
                    "role": "tool",
                    "tool_call_id": message.tool_call_id,
                    "content": message.content,
                })
                
        return messages

    def _convert_tools(self, tools: list[PromptMessageTool]) -> list[dict]:
        """将 PromptMessageTool 转换为 API 请求格式"""
        return [
            {
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": tool.parameters,
                },
            }
            for tool in tools
        ]

    def _handle_stream_response(
        self,
        model: str,
        response: requests.Response,
        prompt_messages: list[PromptMessage],
    ) -> Generator[LLMResultChunk, None, None]:
        """处理流式响应"""
        full_content = ""
        usage = None
        
        for line in response.iter_lines():
            if not line:
                continue
                
            line = line.decode("utf-8")
            if line.startswith("data: "):
                data = line[6:]
                if data == "[DONE]":
                    break
                    
                try:
                    chunk_data = json.loads(data)
                except json.JSONDecodeError:
                    continue
                    
                choice = chunk_data.get("choices", [{}])[0]
                delta = choice.get("delta", {})
                content = delta.get("content", "")
                
                if content:
                    full_content += content
                    
                # 处理工具调用
                tool_calls = []
                if delta.get("tool_calls"):
                    for tc in delta["tool_calls"]:
                        tool_calls.append(
                            AssistantPromptMessage.ToolCall(
                                id=tc.get("id", ""),
                                type=tc.get("type", "function"),
                                function=AssistantPromptMessage.ToolCall.ToolCallFunction(
                                    name=tc.get("function", {}).get("name", ""),
                                    arguments=tc.get("function", {}).get("arguments", ""),
                                ),
                            )
                        )
                
                # 获取使用量（通常在最后一个 chunk）
                if chunk_data.get("usage"):
                    usage_data = chunk_data["usage"]
                    usage = LLMUsage(
                        prompt_tokens=usage_data.get("prompt_tokens", 0),
                        completion_tokens=usage_data.get("completion_tokens", 0),
                        total_tokens=usage_data.get("total_tokens", 0),
                        prompt_unit_price=Decimal("0.00003"),
                        completion_unit_price=Decimal("0.00006"),
                        prompt_price_unit=Decimal("0.001"),
                        completion_price_unit=Decimal("0.001"),
                        prompt_price=Decimal("0"),
                        completion_price=Decimal("0"),
                        total_price=Decimal("0"),
                        currency="USD",
                        latency=0.0,
                    )
                
                yield LLMResultChunk(
                    model=model,
                    prompt_messages=prompt_messages,
                    delta=LLMResultChunkDelta(
                        index=0,
                        message=AssistantPromptMessage(
                            content=content,
                            tool_calls=tool_calls,
                        ),
                        usage=usage,
                        finish_reason=choice.get("finish_reason"),
                    ),
                )

    def _handle_sync_response(
        self,
        model: str,
        response: requests.Response,
        prompt_messages: list[PromptMessage],
    ) -> LLMResult:
        """处理同步响应"""
        data = response.json()
        
        choice = data.get("choices", [{}])[0]
        message = choice.get("message", {})
        content = message.get("content", "")
        
        # 处理工具调用
        tool_calls = []
        if message.get("tool_calls"):
            for tc in message["tool_calls"]:
                tool_calls.append(
                    AssistantPromptMessage.ToolCall(
                        id=tc["id"],
                        type=tc["type"],
                        function=AssistantPromptMessage.ToolCall.ToolCallFunction(
                            name=tc["function"]["name"],
                            arguments=tc["function"]["arguments"],
                        ),
                    )
                )
        
        # 获取使用量
        usage_data = data.get("usage", {})
        usage = LLMUsage(
            prompt_tokens=usage_data.get("prompt_tokens", 0),
            completion_tokens=usage_data.get("completion_tokens", 0),
            total_tokens=usage_data.get("total_tokens", 0),
            prompt_unit_price=Decimal("0.00003"),
            completion_unit_price=Decimal("0.00006"),
            prompt_price_unit=Decimal("0.001"),
            completion_price_unit=Decimal("0.001"),
            prompt_price=Decimal("0"),
            completion_price=Decimal("0"),
            total_price=Decimal("0"),
            currency="USD",
            latency=0.0,
        )
        
        return LLMResult(
            model=model,
            prompt_messages=prompt_messages,
            message=AssistantPromptMessage(
                content=content,
                tool_calls=tool_calls,
            ),
            usage=usage,
        )

    def _handle_http_error(self, response: requests.Response):
        """处理 HTTP 错误"""
        status_code = response.status_code
        error_msg = response.text
        
        if status_code == 401:
            raise InvokeAuthorizationError(f"Authentication failed: {error_msg}")
        elif status_code == 429:
            raise InvokeRateLimitError(f"Rate limit exceeded: {error_msg}")
        elif status_code >= 500:
            raise InvokeServerUnavailableError(f"Server error: {error_msg}")
        else:
            raise InvokeBadRequestError(f"Bad request: {error_msg}")

    def validate_credentials(self, model: str, credentials: dict) -> None:
        """
        验证模型凭据
        
        Args:
            model: 模型名称
            credentials: 凭据信息
            
        Raises:
            CredentialsValidateFailedError: 凭据验证失败
        """
        api_key = credentials.get("api_key")
        api_base = credentials.get("api_base", "https://api.my-provider.com")
        
        if not api_key:
            raise CredentialsValidateFailedError("API Key is required")
        
        try:
            # 发送测试请求
            response = requests.get(
                f"{api_base}/v1/models",
                headers={"Authorization": f"Bearer {api_key}"},
                timeout=10,
            )
            response.raise_for_status()
        except requests.exceptions.RequestException as e:
            raise CredentialsValidateFailedError(f"Failed to validate credentials: {e}")

    def get_num_tokens(
        self,
        model: str,
        credentials: dict,
        prompt_messages: list[PromptMessage],
        tools: Optional[list[PromptMessageTool]] = None,
    ) -> int:
        """
        计算 token 数量
        
        可以使用 tiktoken 或其他分词器进行估算
        """
        # 简单估算：每 4 个字符约等于 1 个 token
        total_chars = 0
        for message in prompt_messages:
            if isinstance(message.content, str):
                total_chars += len(message.content)
            elif isinstance(message.content, list):
                for item in message.content:
                    if hasattr(item, "data"):
                        total_chars += len(item.data)
        
        return total_chars // 4

    def get_customizable_model_schema(
        self,
        model: str,
        credentials: dict,
    ) -> Optional[AIModelEntity]:
        """
        获取自定义模型的 Schema
        
        用于支持用户自定义模型配置
        """
        return AIModelEntity(
            model=model,
            label={"en_US": model, "zh_Hans": model},
            model_type=ModelType.LLM,
            fetch_from=FetchFrom.CUSTOMIZABLE_MODEL,
            features=[],
            model_properties={
                ModelPropertyKey.MODE: LLMMode.CHAT.value,
                ModelPropertyKey.CONTEXT_SIZE: int(credentials.get("context_size", 4096)),
            },
            parameter_rules=[
                ParameterRule(
                    name="temperature",
                    label={"en_US": "Temperature", "zh_Hans": "温度"},
                    type=ParameterType.FLOAT,
                    default=0.7,
                    min=0,
                    max=2,
                    precision=1,
                ),
                ParameterRule(
                    name="max_tokens",
                    label={"en_US": "Max Tokens", "zh_Hans": "最大 Token 数"},
                    type=ParameterType.INT,
                    default=4096,
                    min=1,
                    max=int(credentials.get("context_size", 4096)),
                ),
            ],
        )

    @property
    def _invoke_error_mapping(self) -> dict[type[InvokeError], list[type[Exception]]]:
        """错误映射表"""
        return {
            InvokeConnectionError: [
                requests.exceptions.ConnectionError,
                requests.exceptions.Timeout,
            ],
            InvokeServerUnavailableError: [InvokeServerUnavailableError],
            InvokeRateLimitError: [InvokeRateLimitError],
            InvokeAuthorizationError: [InvokeAuthorizationError],
            InvokeBadRequestError: [InvokeBadRequestError],
        }
```

### 4.2 文本嵌入模型实现

`provider/models/text_embedding/text_embedding.py`:

```python
"""
文本嵌入模型实现示例
"""

import logging
from typing import Optional

import requests

from dify_plugin.entities.model.text_embedding import TextEmbeddingResult
from dify_plugin.errors.model import (
    CredentialsValidateFailedError,
    InvokeAuthorizationError,
    InvokeConnectionError,
)
from dify_plugin.interfaces.model.text_embedding_model import TextEmbeddingModel

logger = logging.getLogger(__name__)


class MyProviderTextEmbeddingModel(TextEmbeddingModel):
    """文本嵌入模型实现"""

    def _invoke(
        self,
        model: str,
        credentials: dict,
        texts: list[str],
        user: Optional[str] = None,
    ) -> TextEmbeddingResult:
        """
        调用文本嵌入模型
        
        Args:
            model: 模型名称
            credentials: 凭据信息
            texts: 待嵌入的文本列表
            user: 用户标识
            
        Returns:
            TextEmbeddingResult: 嵌入结果
        """
        api_key = credentials.get("api_key")
        api_base = credentials.get("api_base", "https://api.my-provider.com")
        
        if not api_key:
            raise InvokeAuthorizationError("API Key is required")

        try:
            response = requests.post(
                f"{api_base}/v1/embeddings",
                headers={
                    "Authorization": f"Bearer {api_key}",
                    "Content-Type": "application/json",
                },
                json={
                    "model": model,
                    "input": texts,
                },
                timeout=60,
            )
            response.raise_for_status()
        except requests.exceptions.ConnectionError as e:
            raise InvokeConnectionError(str(e))
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 401:
                raise InvokeAuthorizationError("Invalid API Key")
            raise

        data = response.json()
        
        # 提取嵌入向量
        embeddings = []
        for item in data.get("data", []):
            embeddings.append(item["embedding"])
        
        # 获取使用量
        usage = data.get("usage", {})
        
        return TextEmbeddingResult(
            model=model,
            embeddings=embeddings,
            usage={
                "tokens": usage.get("total_tokens", 0),
                "total_tokens": usage.get("total_tokens", 0),
            },
        )

    def validate_credentials(self, model: str, credentials: dict) -> None:
        """验证凭据"""
        try:
            self._invoke(model, credentials, ["test"])
        except Exception as e:
            raise CredentialsValidateFailedError(str(e))

    def get_num_tokens(
        self,
        model: str,
        credentials: dict,
        texts: list[str],
    ) -> list[int]:
        """计算每个文本的 token 数量"""
        # 简单估算
        return [len(text) // 4 for text in texts]
```

### 4.3 Rerank 模型实现

`provider/models/rerank/rerank.py`:

```python
"""
Rerank 模型实现示例
"""

import logging
from typing import Optional

import requests

from dify_plugin.entities.model.rerank import RerankDocument, RerankResult
from dify_plugin.errors.model import (
    CredentialsValidateFailedError,
    InvokeAuthorizationError,
    InvokeConnectionError,
)
from dify_plugin.interfaces.model.rerank_model import RerankModel

logger = logging.getLogger(__name__)


class MyProviderRerankModel(RerankModel):
    """Rerank 模型实现"""

    def _invoke(
        self,
        model: str,
        credentials: dict,
        query: str,
        docs: list[str],
        score_threshold: Optional[float] = None,
        top_n: Optional[int] = None,
        user: Optional[str] = None,
    ) -> RerankResult:
        """
        调用 Rerank 模型
        
        Args:
            model: 模型名称
            credentials: 凭据信息
            query: 查询文本
            docs: 待排序的文档列表
            score_threshold: 分数阈值
            top_n: 返回前 N 个结果
            user: 用户标识
            
        Returns:
            RerankResult: 重排序结果
        """
        api_key = credentials.get("api_key")
        api_base = credentials.get("api_base", "https://api.my-provider.com")
        
        if not api_key:
            raise InvokeAuthorizationError("API Key is required")

        payload = {
            "model": model,
            "query": query,
            "documents": docs,
        }
        
        if top_n is not None:
            payload["top_n"] = top_n

        try:
            response = requests.post(
                f"{api_base}/v1/rerank",
                headers={
                    "Authorization": f"Bearer {api_key}",
                    "Content-Type": "application/json",
                },
                json=payload,
                timeout=60,
            )
            response.raise_for_status()
        except requests.exceptions.ConnectionError as e:
            raise InvokeConnectionError(str(e))
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 401:
                raise InvokeAuthorizationError("Invalid API Key")
            raise

        data = response.json()
        
        # 构建结果
        rerank_docs = []
        for item in data.get("results", []):
            score = item.get("relevance_score", 0)
            
            # 应用分数阈值过滤
            if score_threshold is not None and score < score_threshold:
                continue
                
            rerank_docs.append(
                RerankDocument(
                    index=item["index"],
                    text=docs[item["index"]],
                    score=score,
                )
            )
        
        return RerankResult(
            model=model,
            docs=rerank_docs,
        )

    def validate_credentials(self, model: str, credentials: dict) -> None:
        """验证凭据"""
        try:
            self._invoke(
                model=model,
                credentials=credentials,
                query="test query",
                docs=["test document"],
            )
        except Exception as e:
            raise CredentialsValidateFailedError(str(e))
```

## 5. 提供商实现

`provider/my_provider.py`:

```python
"""
提供商实现
"""

import logging

from dify_plugin.errors.model import CredentialsValidateFailedError
from dify_plugin.interfaces.model import ModelProvider

logger = logging.getLogger(__name__)


class MyModelProvider(ModelProvider):
    """自定义模型提供商"""

    def validate_provider_credentials(self, credentials: dict) -> None:
        """
        验证提供商凭据
        
        Args:
            credentials: 凭据信息
            
        Raises:
            CredentialsValidateFailedError: 验证失败
        """
        api_key = credentials.get("api_key")
        
        if not api_key:
            raise CredentialsValidateFailedError("API Key is required")
        
        # 可以发送测试请求验证凭据
        # 或者调用某个模型的 validate_credentials 方法
        try:
            from .models.llm.llm import MyProviderLargeLanguageModel
            
            llm = MyProviderLargeLanguageModel()
            llm.validate_credentials("my-gpt-4", credentials)
        except Exception as e:
            raise CredentialsValidateFailedError(f"Validation failed: {e}")
```

## 6. 插件清单文件

`manifest.yaml`:

```yaml
version: 1.0.0
name: my-model-provider
author: Your Name
description:
  en_US: My custom model provider for Dify
  zh_Hans: Dify 的自定义模型提供商

icon: provider/_assets/icon_small.png

type: model-provider
plugins:
  - provider/my_provider.yaml

label:
  en_US: My Model Provider
  zh_Hans: 我的模型提供商

created_at: 2024-01-01T00:00:00Z

resource:
  memory: 256
  permission:
    tool:
      enabled: false
    model:
      enabled: true
      llm: true
      text_embedding: true
      rerank: true
      tts: false
      speech2text: false
      moderation: false
    node:
      enabled: false
    endpoint:
      enabled: false
    app:
      enabled: false
    storage:
      enabled: false
      size: 0

meta:
  version: 1.0.0
  arch:
    - amd64
    - arm64
  runner:
    language: python
    version: "3.12"
    entrypoint: "main"
```

## 7. 测试

### 7.1 单元测试

`tests/test_provider.py`:

```python
"""
模型提供商单元测试
"""

import pytest
from unittest.mock import MagicMock, patch

from provider.models.llm.llm import MyProviderLargeLanguageModel
from provider.models.text_embedding.text_embedding import MyProviderTextEmbeddingModel
from dify_plugin.entities.model.message import (
    SystemPromptMessage,
    UserPromptMessage,
)


class TestLLMModel:
    """LLM 模型测试"""

    def setup_method(self):
        self.llm = MyProviderLargeLanguageModel()
        self.credentials = {
            "api_key": "test-api-key",
            "api_base": "https://api.my-provider.com",
        }

    @patch("requests.post")
    def test_invoke_sync(self, mock_post):
        """测试同步调用"""
        # 模拟响应
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            "choices": [
                {
                    "message": {
                        "content": "Hello! How can I help you?",
                    },
                    "finish_reason": "stop",
                }
            ],
            "usage": {
                "prompt_tokens": 10,
                "completion_tokens": 8,
                "total_tokens": 18,
            },
        }
        mock_post.return_value = mock_response

        # 调用
        result = self.llm._invoke(
            model="my-gpt-4",
            credentials=self.credentials,
            prompt_messages=[
                SystemPromptMessage(content="You are a helpful assistant."),
                UserPromptMessage(content="Hello!"),
            ],
            model_parameters={"temperature": 0.7},
            stream=False,
        )

        # 验证
        assert result.message.content == "Hello! How can I help you?"
        assert result.usage.total_tokens == 18

    def test_validate_credentials_missing_key(self):
        """测试缺少 API Key"""
        with pytest.raises(Exception):
            self.llm.validate_credentials("my-gpt-4", {})


class TestTextEmbeddingModel:
    """文本嵌入模型测试"""

    def setup_method(self):
        self.model = MyProviderTextEmbeddingModel()
        self.credentials = {
            "api_key": "test-api-key",
        }

    @patch("requests.post")
    def test_invoke(self, mock_post):
        """测试嵌入调用"""
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            "data": [
                {"embedding": [0.1, 0.2, 0.3]},
                {"embedding": [0.4, 0.5, 0.6]},
            ],
            "usage": {"total_tokens": 10},
        }
        mock_post.return_value = mock_response

        result = self.model._invoke(
            model="embedding-v1",
            credentials=self.credentials,
            texts=["Hello", "World"],
        )

        assert len(result.embeddings) == 2
        assert result.embeddings[0] == [0.1, 0.2, 0.3]
```

### 7.2 运行测试

```bash
# 安装测试依赖
pip install pytest pytest-cov

# 运行测试
pytest tests/ -v

# 运行并生成覆盖率报告
pytest tests/ --cov=provider --cov-report=html
```

## 8. 部署和使用

### 8.1 打包插件

```bash
# 安装 dify-plugin-daemon CLI
pip install dify-plugin-daemon

# 打包插件
dify-plugin pack ./my-model-provider

# 输出: my-model-provider-1.0.0.difypkg
```

### 8.2 安装插件

1. 登录 Dify 控制台
2. 进入 "插件" 页面
3. 点击 "安装插件"
4. 上传 `.difypkg` 文件
5. 配置凭据

### 8.3 使用新模型

安装并配置完成后，新的模型提供商和模型将自动出现在：

- 模型设置页面
- 应用编排页面的模型选择器
- 知识库的嵌入模型选择器

## 9. 最佳实践

### 9.1 错误处理

1. **使用统一错误类型**：将底层 SDK 错误映射到 Dify 统一错误
2. **提供详细错误信息**：在错误消息中包含有用的调试信息
3. **实现重试逻辑**：对于临时性错误，考虑实现指数退避重试

### 9.2 性能优化

1. **连接池**：使用 `requests.Session()` 复用 HTTP 连接
2. **超时设置**：设置合理的请求超时时间
3. **流式处理**：优先使用流式响应减少首字节延迟

### 9.3 安全考虑

1. **凭据安全**：不要在日志中打印 API Key
2. **输入验证**：验证所有用户输入
3. **HTTPS**：确保所有 API 调用使用 HTTPS

## 10. 参考资源

- [Dify 插件开发文档](https://docs.dify.ai/plugins)
- [Dify Model Runtime 源码](https://github.com/langgenius/dify/tree/main/api/core/model_runtime)
- [现有模型提供商示例](https://github.com/langgenius/dify-official-plugins)
