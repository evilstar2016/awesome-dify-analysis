# Agent 系统模块 (Agent System Module)

## 模块概述

Agent 系统是 RAGFlow 的智能编排核心，提供可视化工作流（Canvas）设计、组件化执行、工具调用、多步推理等能力。通过 DSL (Domain Specific Language) 定义复杂的业务逻辑，实现自动化和智能化的任务处理。

### 核心价值
- **可视化编排**：拖拽式设计 Agent 工作流，零代码构建复杂逻辑
- **组件化架构**：20+ 内置组件，支持自定义扩展
- **工具集成**：20+ 外部工具集成（搜索、金融、代码执行等）
- **多步推理**：支持循环、分支、迭代等控制流
- **状态管理**：全局变量和组件间数据传递

---

## 功能清单

### 1. Canvas 画布设计
- **工作流设计** - 可视化拖拽设计 Agent 流程
- **组件配置** - 配置组件参数和连接关系
- **DSL 生成** - 自动生成执行 DSL
- **版本管理** - 工作流版本控制

### 2. 组件系统
- **控制流组件**
  - Begin (开始节点)
  - Switch (条件分支)
  - Loop/Iteration (循环)
  - Categorize (分类器)
  
- **数据处理组件**
  - Variable Assigner (变量赋值)
  - Data Operations (数据操作)
  - List Operations (列表操作)
  - Excel Processor (Excel 处理)
  
- **AI 组件**
  - LLM (大语言模型)
  - Agent with Tools (工具代理)
  - Retrieval (知识检索)
  
- **集成组件**
  - Webhook (API 调用)
  - Message (消息发送)
  - Docs Generator (文档生成)

### 3. 工具调用 (Tools)
- **搜索工具**
  - DuckDuckGo, Google, Bing
  - Wikipedia, Google Scholar
  - Arxiv, PubMed
  
- **金融工具**
  - Yahoo Finance, Tushare, AKShare
  - Jin10, WenCai
  
- **开发工具**
  - Code Executor (Python/JS)
  - GitHub API
  - SQL Executor
  
- **其他工具**
  - Web Crawler
  - DeepL Translation
  - Weather (QWeather)
  - Email Sender

### 4. 执行引擎
- **图执行** - DAG (有向无环图) 拓扑排序执行
- **异步执行** - 支持并发和异步任务
- **流式输出** - 实时输出执行结果
- **错误处理** - 异常捕获和重试机制

---

## 目录结构

```
agent/
  ├── canvas.py                    # Canvas 画布和图执行引擎
  ├── settings.py                  # Agent 配置
  ├── component/                   # 组件库
  │   ├── base.py                  # 组件基类
  │   ├── begin.py                 # 开始节点
  │   ├── llm.py                   # LLM 组件
  │   ├── agent_with_tools.py      # 工具代理
  │   ├── switch.py                # 条件分支
  │   ├── loop.py                  # 循环组件
  │   ├── iteration.py             # 迭代组件
  │   ├── variable_assigner.py     # 变量赋值
  │   ├── data_operations.py       # 数据操作
  │   ├── list_operations.py       # 列表操作
  │   ├── excel_processor.py       # Excel 处理
  │   ├── webhook.py               # Webhook
  │   ├── message.py               # 消息组件
  │   └── ...                      # 其他组件
  ├── tools/                       # 工具库
  │   ├── base.py                  # 工具基类
  │   ├── retrieval.py             # 检索工具
  │   ├── duckduckgo.py            # DuckDuckGo 搜索
  │   ├── google.py                # Google 搜索
  │   ├── wikipedia.py             # Wikipedia
  │   ├── arxiv.py                 # Arxiv 论文
  │   ├── code_exec.py             # 代码执行
  │   ├── github.py                # GitHub API
  │   ├── yahoofinance.py          # 金融数据
  │   ├── crawler.py               # 网页爬虫
  │   └── ...                      # 其他工具
  └── templates/                   # Agent 模板库
```

---

## 数据模型

### Canvas (画布)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | STRING(32) | Canvas UUID |
| name | STRING(128) | Canvas 名称 |
| tenant_id | STRING(32) | 租户 ID |
| dsl | JSON | DSL 定义 (components, history, path, globals) |
| components | JSON | 组件配置列表 |
| status | ENUM | 状态 (draft, published) |
| create_time | DATETIME | 创建时间 |
| update_time | DATETIME | 更新时间 |

### DSL 结构
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
          "llm_id": "llm_uuid",
          "prompt": "{{sys.query}}"
        }
      },
      "downstream": [],
      "upstream": ["begin"]
    }
  },
  "history": [],
  "path": ["begin"],
  "globals": {
    "sys.query": "",
    "sys.user_id": "",
    "sys.conversation_turns": 0,
    "sys.files": []
  }
}
```

### Component (组件基类)
```python
class ComponentBase:
    component_name: str
    
    def _run(self, history, **kwargs):
        """组件执行逻辑"""
        pass
```

---

## API 接口

### 创建 Canvas
```http
POST /api/v1/canvas
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "智能客服 Agent",
  "dsl": {
    "components": {...},
    "globals": {...}
  }
}
```

### 执行 Canvas
```http
POST /api/v1/canvas/{canvas_id}/run
Authorization: Bearer {token}
Content-Type: application/json

{
  "query": "用户问题",
  "stream": true
}

Response (SSE):
data: {"component_id": "llm_0", "event": "running", "content": "正在思考..."}
data: {"component_id": "llm_0", "event": "completed", "content": "答案内容"}
data: [DONE]
```

### 获取组件列表
```http
GET /api/v1/canvas/components
Authorization: Bearer {token}

Response:
{
  "code": 0,
  "data": [
    {
      "component_name": "LLM",
      "display_name": "大语言模型",
      "category": "AI",
      "params": [
        {"name": "llm_id", "type": "string", "required": true},
        {"name": "prompt", "type": "string", "required": true}
      ]
    }
  ]
}
```

---

## 技术栈

### 执行引擎
- **asyncio** - 异步执行
- **ThreadPoolExecutor** - 线程池
- **topological_sort** - 拓扑排序

### 组件框架
- **ABC (Abstract Base Class)** - 抽象基类
- **动态加载** - 反射机制加载组件
- **装饰器模式** - 组件注册

### 工具集成
- **HTTP Client** - requests, httpx
- **搜索引擎 API** - DuckDuckGo, Google
- **金融数据 API** - yfinance, akshare
- **代码执行** - RestrictedPython, sandbox

---

## 依赖关系

```
Agent 系统模块 (canvas.py)
    │
    ├─→ ComponentBase (base.py)
    │   ├─→ Begin, LLM, Switch, Loop, ...
    │   └─→ 20+ 内置组件
    │
    ├─→ ToolBase (tools/base.py)
    │   ├─→ Retrieval, DuckDuckGo, GitHub, ...
    │   └─→ 20+ 外部工具
    │
    ├─→ Graph (canvas.py)
    │   ├─→ DSL 解析
    │   ├─→ 拓扑排序
    │   └─→ 异步执行
    │
    ├─→ LLMService
    │   └─→ LLM 模型调用
    │
    └─→ DialogService
        └─→ 对话历史管理
```

---

## 核心流程

### Canvas 执行流程 (详见时序图)
1. **加载 DSL** - 读取 Canvas 配置
2. **初始化组件** - 实例化所有组件
3. **拓扑排序** - 构建执行顺序
4. **执行节点** - 按顺序执行组件
5. **数据传递** - 组件间参数传递
6. **流式输出** - 实时返回执行结果
7. **错误处理** - 异常捕获和回滚

### 工具调用流程
1. **Agent 决策** - LLM 判断是否需要调用工具
2. **工具选择** - 从工具库选择合适工具
3. **参数提取** - 从 LLM 输出提取参数
4. **工具执行** - 调用外部 API 或服务
5. **结果返回** - 将工具结果返回给 LLM
6. **答案生成** - LLM 基于工具结果生成最终答案

---

## 配置参数

### 全局变量 (globals)
```json
{
  "sys.query": "用户输入",
  "sys.user_id": "user_uuid",
  "sys.conversation_turns": 5,
  "sys.files": []
}
```

### 组件通用参数
- **component_name** - 组件类型
- **component_id** - 组件唯一标识
- **inputs** - 输入参数配置
- **outputs** - 输出参数定义

### LLM 组件参数
```json
{
  "llm_id": "llm_uuid",
  "prompt": "提示词模板 {{sys.query}}",
  "temperature": 0.7,
  "max_tokens": 2048,
  "system_prompt": "系统提示词"
}
```

### Agent with Tools 参数
```json
{
  "llm_id": "llm_uuid",
  "tools": ["duckduckgo", "wikipedia", "calculator"],
  "max_iterations": 10,
  "system_prompt": "你是一个智能助手..."
}
```

---

## 性能监控

### 指标项
- **Canvas 执行时间** - 总执行耗时
- **组件执行时间** - 每个组件耗时
- **工具调用次数** - 工具使用统计
- **LLM Token 消耗** - Token 使用量
- **并发执行数** - 同时执行的 Canvas 数

### 日志记录
- Canvas 执行日志
- 组件运行日志
- 工具调用日志
- 错误和异常日志

---

## 组件开发指南

### 创建自定义组件
```python
from agent.component.base import ComponentBase

class MyCustomComponent(ComponentBase):
    component_name = "MyCustom"
    
    def _run(self, history, **kwargs):
        # 获取输入参数
        input_text = self._param.get("input")
        
        # 执行业务逻辑
        result = process(input_text)
        
        # 返回结果
        return {"output": result}
```

### 注册组件
```python
# 在 agent/component/__init__.py 中注册
from .my_custom import MyCustomComponent

component_class["MyCustom"] = MyCustomComponent
```

---

## 常见问题

### Q1: 如何设计复杂的条件分支？
使用 Switch 组件根据条件路由到不同分支：
```json
{
  "component_name": "Switch",
  "params": {
    "condition": "{{sys.user_type}}",
    "cases": {
      "vip": "vip_flow",
      "normal": "normal_flow"
    }
  }
}
```

### Q2: 如何实现循环处理？
使用 Loop 组件配合 Iteration 组件：
```json
{
  "component_name": "Loop",
  "params": {
    "items": "{{sys.files}}",
    "body": "iteration_0"
  }
}
```

### Q3: 工具调用失败怎么办？
- 检查 API Key 配置
- 查看工具调用日志
- 启用重试机制
- 配置降级方案

---

## 相关文档

- [Canvas 执行流程时序图](./01-canvas-execution-sequence.puml)
- [工具调用流程时序图](./02-tool-calling-sequence.puml)
- [组件通信时序图](./03-component-communication-sequence.puml)
- [Agent 推理流程时序图](./04-agent-reasoning-sequence.puml)
- [Agent 开发指南](./agent-development-guide.md)
