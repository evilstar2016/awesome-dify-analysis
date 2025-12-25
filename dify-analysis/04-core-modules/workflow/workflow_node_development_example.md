# Dify 工作流节点开发示例

本文档提供一个完整的工作流节点开发示例 —— **文本处理节点（Text Processor Node）**，该节点可以对输入文本进行多种处理操作（如大小写转换、去除空白、截取等）。

---

## 目录

- [需求说明](#需求说明)
- [后端实现](#后端实现)
- [前端实现](#前端实现)
- [测试用例](#测试用例)
- [完整文件清单](#完整文件清单)

---

## 需求说明

### 功能描述
创建一个文本处理节点，支持以下操作：
- **uppercase**: 转换为大写
- **lowercase**: 转换为小写
- **trim**: 去除首尾空白
- **truncate**: 截取指定长度
- **reverse**: 反转文本

### 输入输出
- **输入**: 一个文本变量
- **输出**: 处理后的文本

### 配置项
- `text_variable`: 输入文本变量选择器
- `operation`: 处理操作类型
- `max_length`: 截取长度（仅 truncate 操作使用）

---

## 后端实现

### 1. 目录结构

```
api/core/workflow/nodes/text_processor/
├── __init__.py
├── text_processor_node.py
├── entities.py
└── exc.py
```

### 2. 异常定义 (`exc.py`)

```python
"""
Text Processor Node 异常定义
"""


class TextProcessorNodeError(Exception):
    """Text Processor 节点基础异常"""
    pass


class InvalidOperationError(TextProcessorNodeError):
    """无效的操作类型"""
    pass


class EmptyInputError(TextProcessorNodeError):
    """输入为空"""
    pass
```

### 3. 数据模型 (`entities.py`)

```python
"""
Text Processor Node 数据模型
"""
from enum import StrEnum

from pydantic import Field

from core.workflow.nodes.base.entities import BaseNodeData, VariableSelector


class TextOperation(StrEnum):
    """文本处理操作类型"""
    UPPERCASE = "uppercase"
    LOWERCASE = "lowercase"
    TRIM = "trim"
    TRUNCATE = "truncate"
    REVERSE = "reverse"


class TextProcessorNodeData(BaseNodeData):
    """
    文本处理节点配置数据
    """
    # 输入文本变量
    text_variable: VariableSelector = Field(
        ...,
        description="输入文本的变量选择器"
    )
    
    # 处理操作
    operation: TextOperation = Field(
        default=TextOperation.TRIM,
        description="文本处理操作类型"
    )
    
    # 截取长度（仅 truncate 使用）
    max_length: int = Field(
        default=100,
        ge=1,
        le=10000,
        description="截取的最大长度"
    )
    
    # 截取时是否添加省略号
    add_ellipsis: bool = Field(
        default=True,
        description="截取时是否在末尾添加省略号"
    )
```

### 4. 节点实现 (`text_processor_node.py`)

```python
"""
Text Processor Node 实现

该节点用于对文本进行各种处理操作。
"""
from collections.abc import Mapping
from typing import Any

from core.workflow.enums import NodeType, WorkflowNodeExecutionStatus
from core.workflow.nodes.base import Node
from core.workflow.nodes.base.entities import BaseNodeData
from core.workflow.node_events.base import NodeRunResult

from .entities import TextOperation, TextProcessorNodeData
from .exc import EmptyInputError, InvalidOperationError


class TextProcessorNode(Node):
    """
    文本处理节点
    
    支持的操作:
    - uppercase: 转换为大写
    - lowercase: 转换为小写
    - trim: 去除首尾空白
    - truncate: 截取指定长度
    - reverse: 反转文本
    """
    
    # 节点类型
    node_type = NodeType.TEXT_PROCESSOR
    
    # 节点数据
    _node_data: TextProcessorNodeData
    
    def init_node_data(self, data: Mapping[str, Any]) -> None:
        """
        初始化节点数据
        
        Args:
            data: 节点配置数据字典
        """
        self._node_data = TextProcessorNodeData.model_validate(data)
    
    def _run(self) -> NodeRunResult:
        """
        执行节点逻辑
        
        Returns:
            NodeRunResult: 节点执行结果
        """
        # 1. 获取输入文本
        text_variable = self.graph_runtime_state.variable_pool.get(
            self._node_data.text_variable.value_selector
        )
        
        input_text = text_variable.to_object() if text_variable else None
        
        inputs = {
            "text": input_text,
            "operation": self._node_data.operation,
        }
        
        # 2. 验证输入
        if input_text is None or not isinstance(input_text, str):
            return NodeRunResult(
                status=WorkflowNodeExecutionStatus.FAILED,
                inputs=inputs,
                error="Input text is empty or not a string",
                error_type=EmptyInputError.__name__
            )
        
        # 3. 执行处理
        try:
            result = self._process_text(
                text=input_text,
                operation=self._node_data.operation,
                max_length=self._node_data.max_length,
                add_ellipsis=self._node_data.add_ellipsis
            )
        except InvalidOperationError as e:
            return NodeRunResult(
                status=WorkflowNodeExecutionStatus.FAILED,
                inputs=inputs,
                error=str(e),
                error_type=InvalidOperationError.__name__
            )
        
        # 4. 返回结果
        return NodeRunResult(
            status=WorkflowNodeExecutionStatus.SUCCEEDED,
            inputs=inputs,
            outputs={
                "result": result,
                "original_length": len(input_text),
                "result_length": len(result),
            }
        )
    
    def _process_text(
        self,
        text: str,
        operation: TextOperation,
        max_length: int,
        add_ellipsis: bool
    ) -> str:
        """
        执行文本处理
        
        Args:
            text: 输入文本
            operation: 处理操作
            max_length: 最大长度（truncate 使用）
            add_ellipsis: 是否添加省略号
            
        Returns:
            str: 处理后的文本
            
        Raises:
            InvalidOperationError: 操作类型无效
        """
        match operation:
            case TextOperation.UPPERCASE:
                return text.upper()
            
            case TextOperation.LOWERCASE:
                return text.lower()
            
            case TextOperation.TRIM:
                return text.strip()
            
            case TextOperation.TRUNCATE:
                if len(text) <= max_length:
                    return text
                if add_ellipsis:
                    return text[:max_length - 3] + "..."
                return text[:max_length]
            
            case TextOperation.REVERSE:
                return text[::-1]
            
            case _:
                raise InvalidOperationError(f"Unknown operation: {operation}")
    
    @classmethod
    def get_default_config(
        cls,
        filters: Mapping[str, object] | None = None
    ) -> Mapping[str, object]:
        """
        获取节点默认配置
        
        Args:
            filters: 过滤条件
            
        Returns:
            默认配置字典
        """
        return {
            "operation": TextOperation.TRIM,
            "max_length": 100,
            "add_ellipsis": True,
        }
    
    @classmethod
    def version(cls) -> str:
        """节点版本"""
        return "1"
    
    # ===== 辅助方法 =====
    
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
    
    @classmethod
    def _extract_variable_selector_to_variable_mapping(
        cls,
        *,
        graph_config: Mapping[str, Any],
        node_id: str,
        node_data: Mapping[str, Any],
    ) -> Mapping[str, list[str]]:
        """
        提取变量选择器到变量的映射
        
        用于工作流引擎识别节点依赖的变量
        """
        typed_node_data = TextProcessorNodeData.model_validate(node_data)
        
        return {
            f"{node_id}.text": typed_node_data.text_variable.value_selector
        }
```

### 5. 模块导出 (`__init__.py`)

```python
"""
Text Processor Node

文本处理节点，支持多种文本操作。
"""
from .text_processor_node import TextProcessorNode

__all__ = ["TextProcessorNode"]
```

### 6. 注册节点类型

在 `api/core/workflow/enums.py` 中添加：

```python
class NodeType(StrEnum):
    # ... 已有类型
    TEXT_PROCESSOR = "text-processor"
```

### 7. 注册节点映射

在 `api/core/workflow/nodes/node_mapping.py` 中添加：

```python
from core.workflow.nodes.text_processor import TextProcessorNode

NODE_TYPE_CLASSES_MAPPING: Mapping[NodeType, Mapping[str, type[Node]]] = {
    # ... 已有映射
    NodeType.TEXT_PROCESSOR: {
        LATEST_VERSION: TextProcessorNode,
        "1": TextProcessorNode,
    },
}
```

---

## 前端实现

### 1. 目录结构

```
web/app/components/workflow/nodes/text-processor/
├── types.ts
├── default.ts
├── node.tsx
├── panel.tsx
└── use-config.ts
```

### 2. 类型定义 (`types.ts`)

```typescript
import type { CommonNodeType, Variable, VarType } from '@/app/components/workflow/types'

/**
 * 文本处理操作类型
 */
export enum TextOperation {
  Uppercase = 'uppercase',
  Lowercase = 'lowercase',
  Trim = 'trim',
  Truncate = 'truncate',
  Reverse = 'reverse',
}

/**
 * 操作类型选项（用于下拉选择）
 */
export const TEXT_OPERATION_OPTIONS = [
  { value: TextOperation.Uppercase, label: 'uppercase' },
  { value: TextOperation.Lowercase, label: 'lowercase' },
  { value: TextOperation.Trim, label: 'trim' },
  { value: TextOperation.Truncate, label: 'truncate' },
  { value: TextOperation.Reverse, label: 'reverse' },
]

/**
 * 文本处理节点类型
 */
export type TextProcessorNodeType = CommonNodeType & {
  // 输入文本变量
  text_variable: Variable
  
  // 处理操作
  operation: TextOperation
  
  // 截取长度
  max_length: number
  
  // 是否添加省略号
  add_ellipsis: boolean
}

/**
 * 输出变量定义
 */
export const OUTPUT_VARS = [
  { variable: 'result', type: 'string' as VarType },
  { variable: 'original_length', type: 'number' as VarType },
  { variable: 'result_length', type: 'number' as VarType },
]
```

### 3. 默认配置 (`default.ts`)

```typescript
import type { NodeDefault, VarType } from '../../types'
import { BlockEnum } from '@/app/components/workflow/types'
import { BlockClassificationEnum } from '@/app/components/workflow/block-selector/types'
import { genNodeMetaData } from '@/app/components/workflow/utils'
import { OUTPUT_VARS, TextOperation, type TextProcessorNodeType } from './types'

const i18nPrefix = 'workflow.errorMsg'

/**
 * 节点元数据
 */
const metaData = genNodeMetaData({
  classification: BlockClassificationEnum.Transform,
  sort: 2,
  type: BlockEnum.TextProcessor,
})

/**
 * 节点默认配置
 */
const nodeDefault: NodeDefault<TextProcessorNodeType> = {
  metaData: {
    ...metaData,
    title: 'Text Processor',
    author: 'Dify',
    description: 'Process text with various operations like uppercase, lowercase, trim, truncate, and reverse.',
  },
  
  // 默认值
  defaultValue: {
    text_variable: {
      variable: '',
      value_selector: [],
    },
    operation: TextOperation.Trim,
    max_length: 100,
    add_ellipsis: true,
  },
  
  // 验证函数
  checkValid(payload: TextProcessorNodeType, t: any) {
    let errorMessage = ''
    const { text_variable, operation, max_length } = payload
    
    // 验证输入变量
    if (!errorMessage && (!text_variable || !text_variable.value_selector?.length)) {
      errorMessage = t(`${i18nPrefix}.fieldRequired`, { 
        field: t('workflow.nodes.textProcessor.textVariable') 
      })
    }
    
    // 验证操作类型
    if (!errorMessage && !operation) {
      errorMessage = t(`${i18nPrefix}.fieldRequired`, { 
        field: t('workflow.nodes.textProcessor.operation') 
      })
    }
    
    // 验证截取长度（仅 truncate 操作）
    if (!errorMessage && operation === TextOperation.Truncate) {
      if (!max_length || max_length < 1) {
        errorMessage = t(`${i18nPrefix}.fieldRequired`, { 
          field: t('workflow.nodes.textProcessor.maxLength') 
        })
      }
    }
    
    return {
      isValid: !errorMessage,
      errorMessage,
    }
  },
  
  // 获取输出变量
  getOutputVars(payload: TextProcessorNodeType) {
    return OUTPUT_VARS.map(v => ({
      variable: v.variable,
      type: v.type as VarType,
    }))
  },
}

export default nodeDefault
```

### 4. 节点组件 (`node.tsx`)

```tsx
import type { FC } from 'react'
import { useTranslation } from 'react-i18next'
import type { NodeProps } from '@/app/components/workflow/types'
import { TEXT_OPERATION_OPTIONS, type TextProcessorNodeType } from './types'

/**
 * 画布上的节点预览组件
 */
const TextProcessorNode: FC<NodeProps<TextProcessorNodeType>> = ({ data }) => {
  const { t } = useTranslation()
  const { text_variable, operation, max_length } = data
  
  // 获取操作标签
  const operationLabel = TEXT_OPERATION_OPTIONS.find(
    opt => opt.value === operation
  )?.label || operation
  
  // 是否有配置输入变量
  const hasInput = text_variable?.value_selector?.length > 0
  
  return (
    <div className="mb-1 px-3 py-1 space-y-0.5">
      {/* 输入变量 */}
      <div className="flex items-center text-xs text-gray-500">
        <span className="mr-1">Input:</span>
        {hasInput ? (
          <span className="text-gray-700 font-medium">
            {text_variable.value_selector.join('.')}
          </span>
        ) : (
          <span className="text-orange-500">
            {t('workflow.nodes.textProcessor.notConfigured')}
          </span>
        )}
      </div>
      
      {/* 操作类型 */}
      <div className="flex items-center text-xs text-gray-500">
        <span className="mr-1">Operation:</span>
        <span className="px-1.5 py-0.5 bg-blue-100 text-blue-700 rounded text-xs font-medium">
          {operationLabel}
        </span>
        
        {/* truncate 显示长度 */}
        {operation === 'truncate' && (
          <span className="ml-2 text-gray-400">
            (max: {max_length})
          </span>
        )}
      </div>
    </div>
  )
}

export default TextProcessorNode
```

### 5. 配置面板 (`panel.tsx`)

```tsx
import type { FC } from 'react'
import { useCallback, useMemo } from 'react'
import { useTranslation } from 'react-i18next'
import type { NodePanelProps } from '@/app/components/workflow/types'
import Field from '@/app/components/workflow/nodes/_base/components/field'
import VarReferencePicker from '@/app/components/workflow/nodes/_base/components/variable/var-reference-picker'
import Select from '@/app/components/base/select'
import Input from '@/app/components/base/input'
import Switch from '@/app/components/base/switch'
import { VarType } from '@/app/components/workflow/types'
import { TEXT_OPERATION_OPTIONS, TextOperation, type TextProcessorNodeType } from './types'
import useConfig from './use-config'

/**
 * 右侧配置面板组件
 */
const TextProcessorPanel: FC<NodePanelProps<TextProcessorNodeType>> = ({ 
  id, 
  data 
}) => {
  const { t } = useTranslation()
  
  // 使用配置 hook
  const {
    inputs,
    handleInputChange,
    handleVarChange,
  } = useConfig(id, data)
  
  // 是否显示截取相关配置
  const showTruncateOptions = inputs.operation === TextOperation.Truncate
  
  // 操作选项
  const operationOptions = useMemo(() => {
    return TEXT_OPERATION_OPTIONS.map(opt => ({
      value: opt.value,
      name: t(`workflow.nodes.textProcessor.operations.${opt.label}`),
    }))
  }, [t])
  
  return (
    <div className="pt-2 space-y-4">
      {/* 输入变量选择 */}
      <Field
        title={t('workflow.nodes.textProcessor.textVariable')}
        required
        tooltip={t('workflow.nodes.textProcessor.textVariableTip')}
      >
        <VarReferencePicker
          nodeId={id}
          readonly={false}
          value={inputs.text_variable?.value_selector || []}
          onChange={(value) => handleVarChange('text_variable', value)}
          filterVar={(varPayload) => {
            // 只允许选择字符串类型变量
            return varPayload.type === VarType.string
          }}
        />
      </Field>
      
      {/* 操作类型选择 */}
      <Field
        title={t('workflow.nodes.textProcessor.operation')}
        required
      >
        <Select
          value={inputs.operation}
          items={operationOptions}
          onSelect={(item) => handleInputChange('operation', item.value)}
          allowSearch={false}
        />
      </Field>
      
      {/* 截取相关配置 */}
      {showTruncateOptions && (
        <>
          {/* 最大长度 */}
          <Field
            title={t('workflow.nodes.textProcessor.maxLength')}
            required
          >
            <Input
              type="number"
              value={inputs.max_length}
              onChange={(e) => handleInputChange('max_length', parseInt(e.target.value) || 100)}
              min={1}
              max={10000}
              className="w-full"
            />
          </Field>
          
          {/* 添加省略号 */}
          <Field
            title={t('workflow.nodes.textProcessor.addEllipsis')}
          >
            <div className="flex items-center">
              <Switch
                checked={inputs.add_ellipsis}
                onChange={(checked) => handleInputChange('add_ellipsis', checked)}
              />
              <span className="ml-2 text-xs text-gray-500">
                {t('workflow.nodes.textProcessor.addEllipsisTip')}
              </span>
            </div>
          </Field>
        </>
      )}
      
      {/* 输出变量说明 */}
      <Field
        title={t('workflow.nodes.textProcessor.outputVars')}
      >
        <div className="space-y-1 text-xs text-gray-500">
          <div>
            <span className="font-mono text-gray-700">result</span>
            <span className="ml-2">- {t('workflow.nodes.textProcessor.outputResult')}</span>
          </div>
          <div>
            <span className="font-mono text-gray-700">original_length</span>
            <span className="ml-2">- {t('workflow.nodes.textProcessor.outputOriginalLength')}</span>
          </div>
          <div>
            <span className="font-mono text-gray-700">result_length</span>
            <span className="ml-2">- {t('workflow.nodes.textProcessor.outputResultLength')}</span>
          </div>
        </div>
      </Field>
    </div>
  )
}

export default TextProcessorPanel
```

### 6. 配置 Hook (`use-config.ts`)

```typescript
import { useCallback } from 'react'
import { useStoreApi } from 'reactflow'
import produce from 'immer'
import type { TextProcessorNodeType } from './types'
import type { ValueSelector } from '@/app/components/workflow/types'
import { useNodeDataUpdate } from '@/app/components/workflow/hooks'

/**
 * 配置管理 Hook
 */
const useConfig = (id: string, data: TextProcessorNodeType) => {
  const store = useStoreApi()
  const { handleNodeDataUpdate } = useNodeDataUpdate()
  
  /**
   * 处理普通配置项变更
   */
  const handleInputChange = useCallback((
    key: keyof TextProcessorNodeType,
    value: any
  ) => {
    const newData = produce(data, (draft: any) => {
      draft[key] = value
    })
    handleNodeDataUpdate({ id, data: newData })
  }, [id, data, handleNodeDataUpdate])
  
  /**
   * 处理变量选择器变更
   */
  const handleVarChange = useCallback((
    key: 'text_variable',
    value: ValueSelector
  ) => {
    const newData = produce(data, (draft: any) => {
      draft[key] = {
        ...draft[key],
        value_selector: value,
      }
    })
    handleNodeDataUpdate({ id, data: newData })
  }, [id, data, handleNodeDataUpdate])
  
  return {
    inputs: data,
    handleInputChange,
    handleVarChange,
  }
}

export default useConfig
```

### 7. 注册组件

在 `web/app/components/workflow/types.ts` 中：

```typescript
export enum BlockEnum {
  // ... 已有类型
  TextProcessor = 'text-processor',
}
```

在 `web/app/components/workflow/nodes/components.ts` 中：

```typescript
import TextProcessorNode from './text-processor/node'
import TextProcessorPanel from './text-processor/panel'

export const NodeComponentMap: Record<string, ComponentType<any>> = {
  // ... 已有组件
  [BlockEnum.TextProcessor]: TextProcessorNode,
}

export const PanelComponentMap: Record<string, ComponentType<any>> = {
  // ... 已有组件
  [BlockEnum.TextProcessor]: TextProcessorPanel,
}
```

### 8. 国际化文件

在 `web/i18n/en-US/workflow.ts` 中添加：

```typescript
const translation = {
  nodes: {
    // ... 已有节点
    textProcessor: {
      title: 'Text Processor',
      description: 'Process text with various operations',
      textVariable: 'Input Text',
      textVariableTip: 'Select the text variable to process',
      operation: 'Operation',
      operations: {
        uppercase: 'Uppercase',
        lowercase: 'Lowercase',
        trim: 'Trim',
        truncate: 'Truncate',
        reverse: 'Reverse',
      },
      maxLength: 'Max Length',
      addEllipsis: 'Add Ellipsis',
      addEllipsisTip: 'Add "..." at the end when truncating',
      outputVars: 'Output Variables',
      outputResult: 'Processed text result',
      outputOriginalLength: 'Original text length',
      outputResultLength: 'Result text length',
      notConfigured: 'Not configured',
    },
  },
}
```

---

## 测试用例

### 后端单元测试

创建 `api/tests/unit_tests/core/workflow/nodes/test_text_processor_node.py`：

```python
"""
Text Processor Node 单元测试
"""
import pytest
from unittest.mock import MagicMock, patch

from core.workflow.enums import WorkflowNodeExecutionStatus
from core.workflow.nodes.text_processor import TextProcessorNode
from core.workflow.nodes.text_processor.entities import TextOperation, TextProcessorNodeData


class TestTextProcessorNode:
    """Text Processor 节点测试类"""
    
    @pytest.fixture
    def node_config(self):
        """基础节点配置"""
        return {
            "title": "Test Text Processor",
            "desc": "Test node",
            "text_variable": {
                "variable": "input_text",
                "value_selector": ["start", "text"],
            },
            "operation": "trim",
            "max_length": 100,
            "add_ellipsis": True,
        }
    
    @pytest.fixture
    def mock_node(self, node_config):
        """创建模拟节点"""
        node = TextProcessorNode.__new__(TextProcessorNode)
        node.init_node_data(node_config)
        return node

    class TestProcessText:
        """测试文本处理逻辑"""
        
        def test_uppercase(self, mock_node):
            """测试大写转换"""
            result = mock_node._process_text(
                text="hello world",
                operation=TextOperation.UPPERCASE,
                max_length=100,
                add_ellipsis=True
            )
            assert result == "HELLO WORLD"
        
        def test_lowercase(self, mock_node):
            """测试小写转换"""
            result = mock_node._process_text(
                text="HELLO WORLD",
                operation=TextOperation.LOWERCASE,
                max_length=100,
                add_ellipsis=True
            )
            assert result == "hello world"
        
        def test_trim(self, mock_node):
            """测试去除空白"""
            result = mock_node._process_text(
                text="  hello world  ",
                operation=TextOperation.TRIM,
                max_length=100,
                add_ellipsis=True
            )
            assert result == "hello world"
        
        def test_truncate_with_ellipsis(self, mock_node):
            """测试截取（带省略号）"""
            result = mock_node._process_text(
                text="hello world this is a long text",
                operation=TextOperation.TRUNCATE,
                max_length=15,
                add_ellipsis=True
            )
            assert result == "hello world ..."
            assert len(result) == 15
        
        def test_truncate_without_ellipsis(self, mock_node):
            """测试截取（不带省略号）"""
            result = mock_node._process_text(
                text="hello world this is a long text",
                operation=TextOperation.TRUNCATE,
                max_length=15,
                add_ellipsis=False
            )
            assert result == "hello world thi"
            assert len(result) == 15
        
        def test_truncate_short_text(self, mock_node):
            """测试截取短文本（不需要截取）"""
            result = mock_node._process_text(
                text="hello",
                operation=TextOperation.TRUNCATE,
                max_length=100,
                add_ellipsis=True
            )
            assert result == "hello"
        
        def test_reverse(self, mock_node):
            """测试反转"""
            result = mock_node._process_text(
                text="hello",
                operation=TextOperation.REVERSE,
                max_length=100,
                add_ellipsis=True
            )
            assert result == "olleh"

    class TestNodeDataValidation:
        """测试节点数据验证"""
        
        def test_valid_config(self, node_config):
            """测试有效配置"""
            data = TextProcessorNodeData.model_validate(node_config)
            assert data.operation == TextOperation.TRIM
            assert data.max_length == 100
        
        def test_invalid_max_length(self, node_config):
            """测试无效的最大长度"""
            node_config["max_length"] = 0
            with pytest.raises(Exception):
                TextProcessorNodeData.model_validate(node_config)
        
        def test_default_values(self, node_config):
            """测试默认值"""
            del node_config["max_length"]
            del node_config["add_ellipsis"]
            data = TextProcessorNodeData.model_validate(node_config)
            assert data.max_length == 100
            assert data.add_ellipsis is True
```

---

## 完整文件清单

### 后端文件

| 文件路径 | 说明 |
|---------|------|
| `api/core/workflow/nodes/text_processor/__init__.py` | 模块导出 |
| `api/core/workflow/nodes/text_processor/text_processor_node.py` | 节点实现 |
| `api/core/workflow/nodes/text_processor/entities.py` | 数据模型 |
| `api/core/workflow/nodes/text_processor/exc.py` | 异常定义 |
| `api/core/workflow/enums.py` | 添加 NodeType.TEXT_PROCESSOR |
| `api/core/workflow/nodes/node_mapping.py` | 添加节点映射 |
| `api/tests/unit_tests/core/workflow/nodes/test_text_processor_node.py` | 单元测试 |

### 前端文件

| 文件路径 | 说明 |
|---------|------|
| `web/app/components/workflow/nodes/text-processor/types.ts` | 类型定义 |
| `web/app/components/workflow/nodes/text-processor/default.ts` | 默认配置 |
| `web/app/components/workflow/nodes/text-processor/node.tsx` | 节点组件 |
| `web/app/components/workflow/nodes/text-processor/panel.tsx` | 配置面板 |
| `web/app/components/workflow/nodes/text-processor/use-config.ts` | 配置 Hook |
| `web/app/components/workflow/types.ts` | 添加 BlockEnum.TextProcessor |
| `web/app/components/workflow/nodes/components.ts` | 注册组件 |
| `web/i18n/en-US/workflow.ts` | 国际化文本 |

---

## 使用示例

### 工作流配置示例

```json
{
  "nodes": [
    {
      "id": "start",
      "type": "start",
      "data": {
        "title": "Start",
        "variables": [
          {
            "variable": "user_input",
            "type": "string",
            "required": true
          }
        ]
      }
    },
    {
      "id": "text_processor_1",
      "type": "text-processor",
      "data": {
        "title": "Clean Input",
        "text_variable": {
          "variable": "user_input",
          "value_selector": ["start", "user_input"]
        },
        "operation": "trim",
        "max_length": 100,
        "add_ellipsis": true
      }
    },
    {
      "id": "text_processor_2",
      "type": "text-processor",
      "data": {
        "title": "To Uppercase",
        "text_variable": {
          "variable": "cleaned_text",
          "value_selector": ["text_processor_1", "result"]
        },
        "operation": "uppercase",
        "max_length": 100,
        "add_ellipsis": true
      }
    },
    {
      "id": "end",
      "type": "end",
      "data": {
        "title": "End",
        "outputs": [
          {
            "variable": "processed_text",
            "value_selector": ["text_processor_2", "result"]
          }
        ]
      }
    }
  ],
  "edges": [
    { "source": "start", "target": "text_processor_1" },
    { "source": "text_processor_1", "target": "text_processor_2" },
    { "source": "text_processor_2", "target": "end" }
  ]
}
```

### 执行结果示例

**输入**：
```
"  Hello World  "
```

**输出**：
```json
{
  "processed_text": "HELLO WORLD"
}
```
