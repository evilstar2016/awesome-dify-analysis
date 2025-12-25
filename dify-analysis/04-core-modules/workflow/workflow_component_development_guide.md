# Dify 工作流组件开发指南

本文档详细描述如何在 Dify 中开发一个新的工作流节点组件。

## 目录

- [概述](#概述)
- [架构总览](#架构总览)
- [后端开发](#后端开发)
- [前端开发](#前端开发)
- [开发流程图](#开发流程图)
- [其他注意事项](#其他注意事项)
- [参考资源](#参考资源)

---

## 概述

Dify 工作流采用前后端分离架构，开发一个新的工作流节点组件需要同时在**后端（Python Flask）**和**前端（Next.js + React）**进行开发。

**关键目录结构**：
```
dify/
├── api/                              # 后端 Python 代码
│   └── core/workflow/nodes/          # 工作流节点实现
│       ├── base/                     # 节点基类
│       ├── code/                     # Code 节点示例
│       ├── llm/                      # LLM 节点示例
│       └── node_mapping.py           # 节点类型映射
└── web/                              # 前端 TypeScript/React 代码
    └── app/components/workflow/
        ├── nodes/                    # 节点前端组件
        ├── block-selector/           # 节点选择器
        └── types.ts                  # 类型定义
```

---

## 架构总览

### 核心类关系

```
BaseNodeData (entities.py)      Node (base/node.py)
     ↑                               ↑
     │ 继承                          │ 继承
     │                               │
YourNodeData ◄───────────────► YourNode
(定义配置数据)                  (实现执行逻辑)
```

### 节点执行流程

1. 工作流引擎根据 `NodeType` 从 `NODE_TYPE_CLASSES_MAPPING` 获取节点类
2. 调用 `init_node_data()` 初始化节点配置
3. 调用 `run()` 方法执行节点
4. `run()` 内部调用 `_run()` 获取执行结果
5. 返回 `NodeRunResult` 包含执行状态和输出

---

## 后端开发

### 1. 创建节点目录结构

在 `api/core/workflow/nodes/` 下创建新节点的目录：

```
api/core/workflow/nodes/your_node/
├── __init__.py           # 导出节点类
├── your_node.py          # 节点实现类
├── entities.py           # 节点数据模型
└── exc.py                # 节点特定异常（可选）
```

### 2. 定义节点数据模型 (`entities.py`)

继承 `BaseNodeData`，定义节点需要的配置参数：

```python
from pydantic import BaseModel
from core.workflow.nodes.base.entities import BaseNodeData, VariableSelector

class YourNodeData(BaseNodeData):
    """
    节点配置数据
    """
    # 输入变量选择器
    variables: list[VariableSelector]
    
    # 节点特定配置
    some_config: str
    another_option: bool = False
    
    # 嵌套配置（可选）
    class OutputConfig(BaseModel):
        type: str
        format: str
    
    output_config: OutputConfig | None = None
```

**`BaseNodeData` 已提供的基础字段**：
- `title`: 节点标题
- `desc`: 节点描述
- `version`: 版本号
- `error_strategy`: 错误处理策略
- `retry_config`: 重试配置
- `default_value`: 默认值列表

### 3. 实现节点类 (`your_node.py`)

继承 `Node` 基类，实现核心方法：

```python
from collections.abc import Mapping
from typing import Any

from core.workflow.enums import NodeType, WorkflowNodeExecutionStatus
from core.workflow.nodes.base import Node
from core.workflow.nodes.base.entities import BaseNodeData
from core.workflow.node_events.base import NodeRunResult

from .entities import YourNodeData


class YourNode(Node):
    """
    Your Node 实现
    """
    
    # 必须：指定节点类型（对应 NodeType 枚举）
    node_type = NodeType.YOUR_NODE
    
    # 节点数据实例
    _node_data: YourNodeData
    
    def init_node_data(self, data: Mapping[str, Any]):
        """
        必须实现：初始化节点数据
        """
        self._node_data = YourNodeData.model_validate(data)
    
    def _run(self) -> NodeRunResult:
        """
        必须实现：节点执行逻辑
        
        Returns:
            NodeRunResult: 包含执行状态、输入、输出的结果对象
        """
        # 1. 从变量池获取输入变量
        inputs = {}
        for var_selector in self._node_data.variables:
            variable = self.graph_runtime_state.variable_pool.get(
                var_selector.value_selector
            )
            inputs[var_selector.variable] = variable.to_object() if variable else None
        
        # 2. 执行节点核心逻辑
        try:
            result = self._execute_logic(inputs)
        except Exception as e:
            return NodeRunResult(
                status=WorkflowNodeExecutionStatus.FAILED,
                inputs=inputs,
                error=str(e),
                error_type=type(e).__name__
            )
        
        # 3. 返回成功结果
        return NodeRunResult(
            status=WorkflowNodeExecutionStatus.SUCCEEDED,
            inputs=inputs,
            outputs=result
        )
    
    def _execute_logic(self, inputs: dict) -> dict:
        """
        节点的核心业务逻辑
        """
        # 实现你的业务逻辑
        output = {"result": "processed"}
        return output
    
    @classmethod
    def get_default_config(cls, filters: Mapping[str, object] | None = None) -> Mapping[str, object]:
        """
        可选：返回节点的默认配置
        """
        return {
            "some_config": "default_value",
            "another_option": False,
        }
    
    @classmethod
    def version(cls) -> str:
        """
        必须：返回节点版本号
        """
        return "1"
    
    # ===== 以下为辅助方法，通常直接委托给 _node_data =====
    
    def _get_error_strategy(self):
        return self._node_data.error_strategy
    
    def _get_retry_config(self):
        return self._node_data.retry_config
    
    def _get_title(self) -> str:
        return self._node_data.title
    
    def _get_description(self) -> str | None:
        return self._node_data.desc
    
    def _get_default_value_dict(self) -> dict[str, Any]:
        return self._node_data.default_value_dict
    
    def get_base_node_data(self) -> BaseNodeData:
        return self._node_data
    
    @property
    def retry(self) -> bool:
        return self._node_data.retry_config.retry_enabled
```

### 4. 导出节点 (`__init__.py`)

```python
from .your_node import YourNode

__all__ = ["YourNode"]
```

### 5. 注册节点类型枚举

在 `api/core/workflow/enums.py` 中添加新的节点类型：

```python
class NodeType(StrEnum):
    START = "start"
    END = "end"
    # ... 已有类型
    YOUR_NODE = "your-node"  # 添加新类型
```

### 6. 注册节点映射

在 `api/core/workflow/nodes/node_mapping.py` 中添加：

```python
from core.workflow.nodes.your_node import YourNode

NODE_TYPE_CLASSES_MAPPING: Mapping[NodeType, Mapping[str, type[Node]]] = {
    # ... 已有映射
    NodeType.YOUR_NODE: {
        LATEST_VERSION: YourNode,
        "1": YourNode,
    },
}
```

---

## 前端开发

### 1. 创建节点目录结构

在 `web/app/components/workflow/nodes/` 下创建：

```
web/app/components/workflow/nodes/your-node/
├── node.tsx              # 画布上的节点组件
├── panel.tsx             # 右侧配置面板组件
├── types.ts              # 类型定义
├── default.ts            # 默认配置和验证逻辑
└── use-config.ts         # 配置管理 Hook（可选）
```

### 2. 定义类型 (`types.ts`)

```typescript
import type { CommonNodeType, Variable, VarType } from '@/app/components/workflow/types'

export type YourNodeType = CommonNodeType & {
  variables: Variable[]
  some_config: string
  another_option: boolean
  output_config?: {
    type: string
    format: string
  }
}

// 输出变量类型定义（可选）
export type OutputVar = {
  result: VarType.string
}
```

### 3. 定义默认配置 (`default.ts`)

```typescript
import type { NodeDefault } from '../../types'
import type { YourNodeType } from './types'
import { BlockEnum } from '@/app/components/workflow/types'
import { BlockClassificationEnum } from '@/app/components/workflow/block-selector/types'
import { genNodeMetaData } from '@/app/components/workflow/utils'

const i18nPrefix = 'workflow.errorMsg'

const metaData = genNodeMetaData({
  classification: BlockClassificationEnum.Transform,  // 节点分类
  sort: 1,
  type: BlockEnum.YourNode,
})

const nodeDefault: NodeDefault<YourNodeType> = {
  metaData,
  defaultValue: {
    variables: [],
    some_config: '',
    another_option: false,
  },
  
  // 验证函数
  checkValid(payload: YourNodeType, t: any) {
    let errorMessage = ''
    
    // 验证变量配置
    const { variables, some_config } = payload
    
    if (!errorMessage && variables.filter(v => !v.variable).length > 0)
      errorMessage = t(`${i18nPrefix}.fieldRequired`, { field: t(`${i18nPrefix}.fields.variable`) })
    
    if (!errorMessage && variables.filter(v => !v.value_selector.length).length > 0)
      errorMessage = t(`${i18nPrefix}.fieldRequired`, { field: t(`${i18nPrefix}.fields.variableValue`) })
    
    if (!errorMessage && !some_config)
      errorMessage = t(`${i18nPrefix}.fieldRequired`, { field: 'some_config' })
    
    return {
      isValid: !errorMessage,
      errorMessage,
    }
  },
  
  // 获取输出变量（可选）
  getOutputVars(payload: YourNodeType) {
    return [
      { variable: 'result', type: VarType.string },
    ]
  },
}

export default nodeDefault
```

### 4. 添加节点类型枚举

在 `web/app/components/workflow/types.ts` 中：

```typescript
export enum BlockEnum {
  Start = 'start',
  End = 'end',
  // ... 已有类型
  YourNode = 'your-node',  // 添加新类型（需与后端 NodeType 对应）
}
```

### 5. 实现节点组件 (`node.tsx`)

```typescript
import type { FC } from 'react'
import type { YourNodeType } from './types'
import type { NodeProps } from '@/app/components/workflow/types'

const Node: FC<NodeProps<YourNodeType>> = ({ data }) => {
  const { variables, some_config } = data
  
  return (
    <div className="mb-1 px-3 py-1 space-y-0.5">
      {/* 显示节点预览信息 */}
      <div className="text-xs text-gray-500">
        Config: {some_config || 'Not configured'}
      </div>
      <div className="text-xs text-gray-500">
        Variables: {variables.length}
      </div>
    </div>
  )
}

export default Node
```

### 6. 实现配置面板 (`panel.tsx`)

```typescript
import type { FC } from 'react'
import { useTranslation } from 'react-i18next'
import type { YourNodeType } from './types'
import type { NodePanelProps } from '@/app/components/workflow/types'
import Field from '@/app/components/workflow/nodes/_base/components/field'
import VarReferencePicker from '@/app/components/workflow/nodes/_base/components/variable/var-reference-picker'

const Panel: FC<NodePanelProps<YourNodeType>> = ({ id, data }) => {
  const { t } = useTranslation()
  
  // 使用配置管理 hook（需要自行实现）
  // const { inputs, handleInputChange } = useConfig(id, data)
  
  return (
    <div className="pt-2">
      <Field
        title={t('workflow.nodes.yourNode.someConfig')}
        required
      >
        {/* 配置组件 */}
        <input
          type="text"
          value={data.some_config}
          onChange={(e) => {/* 处理变更 */}}
          className="w-full px-3 py-2 border rounded"
        />
      </Field>
      
      <Field
        title={t('workflow.nodes.yourNode.variables')}
      >
        {/* 变量选择器组件 */}
        <VarReferencePicker
          nodeId={id}
          // ... 其他属性
        />
      </Field>
    </div>
  )
}

export default Panel
```

### 7. 注册节点组件

在 `web/app/components/workflow/nodes/components.ts` 中：

```typescript
import YourNodeNode from './your-node/node'
import YourNodePanel from './your-node/panel'

export const NodeComponentMap: Record<string, ComponentType<any>> = {
  // ... 已有组件
  [BlockEnum.YourNode]: YourNodeNode,
}

export const PanelComponentMap: Record<string, ComponentType<any>> = {
  // ... 已有组件
  [BlockEnum.YourNode]: YourNodePanel,
}
```

### 8. 添加国际化文本

在 `web/i18n/en-US/workflow.ts` 中添加：

```typescript
const translation = {
  nodes: {
    // ... 已有节点
    yourNode: {
      title: 'Your Node',
      description: 'Description of your node',
      someConfig: 'Some Config',
      variables: 'Variables',
    },
  },
}
```

---

## 开发流程图

详见 [workflow_component_development_flow.puml](./workflow_component_development_flow.puml)

---

## 其他注意事项

### 1. 错误处理

- 后端使用 `NodeRunResult` 的 `status` 字段标识执行状态
- 支持 `error_strategy` 配置错误处理策略（fail-branch / default-value）
- 支持 `retry_config` 配置自动重试

### 2. 变量系统

- 输入变量通过 `VariableSelector` 从变量池获取
- 输出变量会自动添加到变量池供下游节点使用
- 支持多种变量类型：string, number, boolean, object, array, file 等

### 3. 测试

- 后端单元测试：`api/tests/unit_tests/core/workflow/nodes/`
- 使用 pytest 框架，遵循 Arrange-Act-Assert 模式

### 4. 节点分类

前端节点在 block-selector 中按分类显示：
- `Default`: 默认分类
- `QuestionUnderstand`: 问题理解类
- `Logic`: 逻辑控制类
- `Transform`: 数据转换类
- `Utilities`: 工具类

---

## 参考资源

### 推荐参考的现有节点

| 节点类型 | 复杂度 | 特点 |
|---------|-------|------|
| `code` | 简单 | 结构清晰，适合入门 |
| `template_transform` | 简单 | Jinja2 模板转换 |
| `http_request` | 中等 | 外部 HTTP 调用 |
| `llm` | 中等 | LLM 模型调用，流式输出 |
| `iteration` | 复杂 | 容器节点，包含子图 |

### 关键文件路径

**后端**：
- 节点基类：`api/core/workflow/nodes/base/node.py`
- 数据基类：`api/core/workflow/nodes/base/entities.py`
- 节点类型枚举：`api/core/workflow/enums.py`
- 节点映射：`api/core/workflow/nodes/node_mapping.py`

**前端**：
- 类型定义：`web/app/components/workflow/types.ts`
- 组件映射：`web/app/components/workflow/nodes/components.ts`
- 节点基础组件：`web/app/components/workflow/nodes/_base/`
