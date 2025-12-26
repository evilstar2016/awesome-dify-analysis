# Agent 开发指南 (Agent Development Guide)

## 概述

本指南面向希望开发和扩展 RAGFlow Agent 系统的开发者，涵盖组件开发、工具集成、Canvas 设计、调试技巧等内容。

---

## 目录

1. [环境准备](#环境准备)
2. [组件开发](#组件开发)
3. [工具开发](#工具开发)
4. [Canvas 设计](#canvas-设计)
5. [调试与测试](#调试与测试)
6. [最佳实践](#最佳实践)
7. [常见问题](#常见问题)

---

## 环境准备

### 安装依赖

```bash
# 克隆项目
git clone https://github.com/infiniflow/ragflow.git
cd ragflow

# 安装 Python 环境
uv sync --python 3.12 --all-extras

# 下载依赖
uv run download_deps.py

# 启动基础服务
cd docker
docker compose -f docker-compose-base.yml up -d
```

### 配置开发环境

```bash
# 设置环境变量
export PYTHONPATH=$(pwd)
export RAGFLOW_ENV=development

# 激活虚拟环境
source .venv/bin/activate
```

---

## 组件开发

### 1. 组件基类

所有组件必须继承 `ComponentBase` 基类：

```python
from agent.component.base import ComponentBase

class ComponentBase:
    component_name: str = "Base"
    
    def __init__(self, component_id: str, params: dict):
        self.component_id = component_id
        self._param = params
    
    def _run(self, history: list, **kwargs) -> dict:
        """
        组件执行逻辑
        
        Args:
            history: 对话历史
            **kwargs: 全局变量和上游输出
        
        Returns:
            dict: 组件输出
        """
        raise NotImplementedError
```

### 2. 创建自定义组件

#### 示例：文本转换组件

```python
from agent.component.base import ComponentBase
import re

class TextTransformComponent(ComponentBase):
    """文本转换组件：大小写转换、去除特殊字符等"""
    
    component_name = "TextTransform"
    
    def _run(self, history, **kwargs):
        # 1. 获取输入参数
        text = self._param.get("text", "")
        operation = self._param.get("operation", "lowercase")
        
        # 2. 参数验证
        if not text:
            raise ValueError("text parameter is required")
        
        # 3. 执行转换
        result = self._transform(text, operation)
        
        # 4. 返回结果
        return {
            "transformed_text": result,
            "original_length": len(text),
            "result_length": len(result)
        }
    
    def _transform(self, text: str, operation: str) -> str:
        """执行具体转换"""
        operations = {
            "lowercase": lambda x: x.lower(),
            "uppercase": lambda x: x.upper(),
            "remove_special": lambda x: re.sub(r'[^a-zA-Z0-9\s]', '', x),
            "remove_spaces": lambda x: re.sub(r'\s+', '', x),
        }
        
        if operation not in operations:
            raise ValueError(f"Unsupported operation: {operation}")
        
        return operations[operation](text)
```

### 3. 注册组件

在 `agent/component/__init__.py` 中注册：

```python
from agent.component.text_transform import TextTransformComponent

component_class = {
    "Begin": Begin,
    "LLM": LLM,
    "TextTransform": TextTransformComponent,  # 新增
    # ... 其他组件
}
```

### 4. 组件配置定义

创建组件配置文件 `agent/component/text_transform.json`：

```json
{
  "component_name": "TextTransform",
  "display_name": "文本转换",
  "description": "对文本进行各种转换操作",
  "category": "数据处理",
  "icon": "text",
  "params": [
    {
      "name": "text",
      "display_name": "输入文本",
      "type": "string",
      "required": true,
      "description": "需要转换的文本"
    },
    {
      "name": "operation",
      "display_name": "转换操作",
      "type": "select",
      "required": true,
      "default": "lowercase",
      "options": [
        {"label": "转小写", "value": "lowercase"},
        {"label": "转大写", "value": "uppercase"},
        {"label": "移除特殊字符", "value": "remove_special"},
        {"label": "移除空格", "value": "remove_spaces"}
      ]
    }
  ],
  "inputs": ["text"],
  "outputs": ["transformed_text", "original_length", "result_length"]
}
```

---

## 工具开发

### 1. 工具基类

继承 `ToolBase` 创建自定义工具：

```python
from agent.tools.base import ToolBase

class ToolBase:
    name: str
    description: str
    parameters: dict
    
    def run(self, **kwargs):
        raise NotImplementedError
```

### 2. 创建自定义工具

#### 示例：天气查询工具

```python
from agent.tools.base import ToolBase
import requests

class WeatherTool(ToolBase):
    """天气查询工具"""
    
    name = "weather_query"
    description = "查询指定城市的天气信息"
    parameters = {
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "城市名称，如：北京、上海"
            },
            "date": {
                "type": "string",
                "description": "日期，格式：YYYY-MM-DD，默认今天"
            }
        },
        "required": ["city"]
    }
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.weatherapi.com/v1"
    
    def run(self, city: str, date: str = None, **kwargs):
        """
        查询天气
        
        Args:
            city: 城市名称
            date: 日期（可选）
        
        Returns:
            dict: 天气信息
        """
        try:
            # 构建请求
            url = f"{self.base_url}/forecast.json"
            params = {
                "key": self.api_key,
                "q": city,
                "days": 1,
                "lang": "zh"
            }
            
            if date:
                params["dt"] = date
            
            # 发送请求
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            
            data = response.json()
            
            # 解析结果
            location = data["location"]
            current = data["current"]
            
            result = {
                "city": location["name"],
                "temperature": current["temp_c"],
                "condition": current["condition"]["text"],
                "humidity": current["humidity"],
                "wind_speed": current["wind_kph"],
                "update_time": current["last_updated"]
            }
            
            return result
            
        except requests.RequestException as e:
            return {"error": f"天气查询失败: {str(e)}"}
        except KeyError as e:
            return {"error": f"数据解析失败: {str(e)}"}
```

### 3. 注册工具

在 `agent/tools/__init__.py` 中注册：

```python
from agent.tools.weather import WeatherTool

def load_tools(tool_names: list, config: dict) -> dict:
    """加载工具实例"""
    tools = {}
    
    for name in tool_names:
        if name == "weather_query":
            tools[name] = WeatherTool(api_key=config.get("weather_api_key"))
        # ... 其他工具
    
    return tools
```

### 4. 在 Agent 中使用工具

```python
from agent.component.agent_with_tools import AgentWithTools

# 配置 Agent
agent_config = {
    "llm_id": "llm_uuid",
    "tools": ["weather_query", "duckduckgo", "calculator"],
    "max_iterations": 10,
    "system_prompt": "你是一个智能助手，可以使用工具回答问题。"
}

# 创建 Agent 组件
agent = AgentWithTools(component_id="agent_0", params=agent_config)

# 执行
result = agent._run(
    history=[],
    **{"sys.query": "北京今天天气怎么样？"}
)
```

---

## Canvas 设计

### 1. 基础 Canvas 结构

```json
{
  "components": {
    "begin": {
      "obj": {
        "component_name": "Begin",
        "params": {}
      },
      "downstream": ["llm_0"],
      "upstream": []
    },
    "llm_0": {
      "obj": {
        "component_name": "LLM",
        "params": {
          "llm_id": "gpt-4",
          "prompt": "回答问题：{{sys.query}}"
        }
      },
      "downstream": [],
      "upstream": ["begin"]
    }
  },
  "globals": {
    "sys.query": "",
    "sys.user_id": "",
    "sys.conversation_turns": 0
  }
}
```

### 2. 设计模式

#### 模式1：顺序执行

```
Begin → Retrieval → LLM → Message
```

适用场景：简单的 RAG 问答流程

#### 模式2：条件分支

```
Begin → Categorize → {
  type1 → Handler1 → Message
  type2 → Handler2 → Message
  type3 → Handler3 → Message
}
```

适用场景：根据用户意图路由到不同处理流程

#### 模式3：循环处理

```
Begin → Loop(items) → {
  IterationItem → Process → Aggregate
} → Message
```

适用场景：批量处理文档或数据

#### 模式4：并行执行

```
Begin → {
  Branch1 → Process1
  Branch2 → Process2
  Branch3 → Process3
} → Merge → Message
```

适用场景：需要从多个来源获取数据

### 3. 变量引用

在组件参数中引用变量：

```json
{
  "prompt": "用户问题：{{sys.query}}\n\n参考内容：{{retrieval_0.output.chunks}}\n\n请回答问题。"
}
```

支持的引用方式：
- `{{sys.query}}` - 系统变量
- `{{component_id.output.field}}` - 组件输出
- `{{component_id.output.list[0]}}` - 数组访问
- `{{component_id.output.nested.field}}` - 嵌套对象

---

## 调试与测试

### 1. 单元测试

创建测试文件 `agent/test/test_text_transform.py`：

```python
import unittest
from agent.component.text_transform import TextTransformComponent

class TestTextTransformComponent(unittest.TestCase):
    
    def setUp(self):
        self.component = TextTransformComponent(
            component_id="test_transform",
            params={
                "text": "Hello World!",
                "operation": "lowercase"
            }
        )
    
    def test_lowercase(self):
        result = self.component._run(history=[])
        self.assertEqual(result["transformed_text"], "hello world!")
    
    def test_uppercase(self):
        self.component._param["operation"] = "uppercase"
        result = self.component._run(history=[])
        self.assertEqual(result["transformed_text"], "HELLO WORLD!")
    
    def test_remove_special(self):
        self.component._param["text"] = "Hello, World! 123"
        self.component._param["operation"] = "remove_special"
        result = self.component._run(history=[])
        self.assertEqual(result["transformed_text"], "Hello World 123")

if __name__ == "__main__":
    unittest.main()
```

运行测试：

```bash
uv run pytest agent/test/test_text_transform.py -v
```

### 2. 集成测试

测试 Canvas 执行：

```python
import asyncio
from agent.canvas import Canvas

async def test_canvas():
    dsl = {
        "components": {
            "begin": {...},
            "text_transform_0": {
                "obj": {
                    "component_name": "TextTransform",
                    "params": {
                        "text": "{{sys.query}}",
                        "operation": "uppercase"
                    }
                },
                "downstream": [],
                "upstream": ["begin"]
            }
        },
        "globals": {
            "sys.query": "hello world"
        }
    }
    
    canvas = Canvas(dsl)
    result = await canvas.run()
    
    assert result["text_transform_0"]["transformed_text"] == "HELLO WORLD"
    print("✅ Canvas 测试通过")

asyncio.run(test_canvas())
```

### 3. 调试技巧

#### 启用详细日志

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# 在组件中添加日志
class MyComponent(ComponentBase):
    def _run(self, history, **kwargs):
        self.logger.debug(f"输入参数: {self._param}")
        result = self.process()
        self.logger.info(f"输出结果: {result}")
        return result
```

#### 使用断点调试

```python
import pdb

class MyComponent(ComponentBase):
    def _run(self, history, **kwargs):
        # 设置断点
        pdb.set_trace()
        
        # 检查变量
        print(f"history: {history}")
        print(f"kwargs: {kwargs}")
        
        return result
```

---

## 最佳实践

### 1. 组件设计原则

✅ **单一职责**：每个组件只做一件事  
✅ **参数验证**：严格验证输入参数  
✅ **错误处理**：优雅处理异常情况  
✅ **可测试性**：编写单元测试  
✅ **文档完善**：添加详细注释和文档

### 2. 性能优化

```python
class OptimizedComponent(ComponentBase):
    
    def __init__(self, component_id, params):
        super().__init__(component_id, params)
        # 预加载资源
        self._cache = {}
        self._model = self._load_model()
    
    def _run(self, history, **kwargs):
        # 使用缓存
        cache_key = self._get_cache_key(kwargs)
        if cache_key in self._cache:
            return self._cache[cache_key]
        
        # 批量处理
        results = self._batch_process(data)
        
        # 缓存结果
        self._cache[cache_key] = results
        return results
    
    def _batch_process(self, data):
        """批量处理提升性能"""
        batch_size = 32
        results = []
        for i in range(0, len(data), batch_size):
            batch = data[i:i+batch_size]
            results.extend(self._process_batch(batch))
        return results
```

### 3. 安全性

```python
class SecureComponent(ComponentBase):
    
    def _run(self, history, **kwargs):
        # 1. 输入验证
        text = self._sanitize_input(self._param.get("text"))
        
        # 2. 权限检查
        if not self._check_permission(kwargs.get("sys.user_id")):
            raise PermissionError("无权限执行该操作")
        
        # 3. 限流
        if not self._rate_limit_check(kwargs.get("sys.user_id")):
            raise Exception("请求过于频繁")
        
        # 4. 执行业务逻辑
        result = self._process(text)
        
        # 5. 输出过滤
        return self._filter_sensitive_data(result)
```

---

## 常见问题

### Q1: 组件间如何传递复杂数据？

使用全局变量存储：

```python
# 组件 A
def _run(self, history, **kwargs):
    complex_data = {
        "results": [...],
        "metadata": {...}
    }
    # 数据会自动保存到 globals["component_a.output"]
    return complex_data

# 组件 B
def _run(self, history, **kwargs):
    # 通过参数引用
    # params: {"data": "{{component_a.output.results}}"}
    data = self._param.get("data")
    return process(data)
```

### Q2: 如何处理异步操作？

```python
import asyncio

class AsyncComponent(ComponentBase):
    
    async def _run_async(self, history, **kwargs):
        # 异步操作
        results = await asyncio.gather(
            self._fetch_data_1(),
            self._fetch_data_2(),
            self._fetch_data_3()
        )
        return {"results": results}
    
    def _run(self, history, **kwargs):
        # 同步包装
        loop = asyncio.get_event_loop()
        return loop.run_until_complete(
            self._run_async(history, **kwargs)
        )
```

### Q3: 如何限制组件执行时间？

```python
import signal
from contextlib import contextmanager

@contextmanager
def timeout(seconds):
    def timeout_handler(signum, frame):
        raise TimeoutError("组件执行超时")
    
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(seconds)
    try:
        yield
    finally:
        signal.alarm(0)

class TimeLimitedComponent(ComponentBase):
    
    def _run(self, history, **kwargs):
        max_time = self._param.get("max_time", 30)
        
        try:
            with timeout(max_time):
                result = self._long_running_task()
            return result
        except TimeoutError:
            return {"error": "执行超时"}
```

---

## 资源链接

- [组件 API 文档](../components-api.md)
- [工具 API 文档](../tools-api.md)
- [Canvas DSL 规范](../canvas-dsl-spec.md)
- [示例项目](../examples/)

---

**作者**: RAGFlow 开发团队  
**最后更新**: 2025-12-19  
**版本**: 1.0
