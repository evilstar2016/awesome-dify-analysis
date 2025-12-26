# 工作流管理模块深入分析

## 1. 模块概述

工作流管理模块是 FastGPT 的核心能力之一，负责编排和执行复杂的 AI 工作流。该模块通过可视化节点编排的方式，支持多种节点类型、变量系统、条件分支、工具调用等复杂逻辑，实现了灵活的对话和数据处理能力。

### 1.1 核心职责

- **工作流编排**: 支持可视化节点编排，定义节点间的数据流向
- **节点执行**: 按照依赖关系执行各类节点（AI对话、知识库检索、HTTP请求等）
- **变量管理**: 全局变量系统，支持节点间数据传递
- **分支控制**: 条件判断、循环执行等流程控制
- **工具集成**: 支持多种工具调用（MCP、HTTP、代码沙盒等）
- **流式响应**: 实时返回执行结果，支持 SSE 流式输出
- **调试模式**: 支持单步调试，查看节点执行状态

### 1.2 技术特点

- **DAG 执行引擎**: 基于有向无环图的节点执行
- **并发控制**: 支持节点并发执行，可配置最大并发数
- **队列管理**: 采用队列机制管理节点执行顺序
- **交互式节点**: 支持表单输入、用户选择等交互节点
- **错误处理**: 完善的错误捕获和处理机制
- **资源计费**: 实时追踪各节点的资源消耗

## 2. 架构设计

### 2.1 目录结构

```
packages/
├── global/core/workflow/          # 工作流核心类型定义
│   ├── constants.ts              # 常量定义
│   ├── utils.ts                  # 工具函数
│   ├── node/                     # 节点类型定义
│   │   └── constant.ts          # 节点常量
│   ├── runtime/                  # 运行时类型
│   │   ├── type.ts              # 运行时类型定义
│   │   ├── constants.ts         # 运行时常量
│   │   └── utils.ts             # 运行时工具
│   ├── template/                 # 节点模板
│   │   └── system/              # 系统节点模板
│   └── type/                     # 类型定义
│       ├── node.d.ts            # 节点类型
│       ├── edge.d.ts            # 边类型
│       └── io.d.ts              # 输入输出类型
│
├── service/core/workflow/         # 工作流服务层
│   ├── dispatch/                 # 调度器（核心执行引擎）
│   │   ├── index.ts             # 主调度器
│   │   ├── utils.ts             # 调度工具
│   │   ├── constants.ts         # 回调映射
│   │   ├── init/                # 初始化节点
│   │   ├── ai/                  # AI 相关节点
│   │   │   ├── chat.ts         # AI 对话
│   │   │   ├── classifyQuestion.ts  # 问题分类
│   │   │   ├── extract.ts      # 内容提取
│   │   │   └── tool/           # 工具调用
│   │   ├── dataset/             # 知识库节点
│   │   │   ├── search.ts       # 知识库检索
│   │   │   └── concat.ts       # 结果合并
│   │   ├── child/               # 子流程节点
│   │   │   ├── runApp.ts       # 运行应用
│   │   │   └── runTool.ts      # 运行工具
│   │   ├── plugin/              # 插件节点
│   │   ├── loop/                # 循环节点
│   │   ├── interactive/         # 交互节点
│   │   └── tools/               # 工具节点集合
│   │       ├── http468.ts      # HTTP 请求
│   │       ├── answer.ts       # 回答节点
│   │       ├── runIfElse.ts    # 条件分支
│   │       ├── codeSandbox.ts  # 代码沙盒
│   │       ├── runUpdateVar.ts # 变量更新
│   │       └── ...
│   ├── utils.ts                 # 工具函数
│   └── constants.ts             # 常量定义
│
└── web/core/workflow/            # 前端工作流
    └── constants.ts             # 前端常量

projects/app/
├── src/service/core/app/
│   └── workflow.ts              # 工作流服务接口
└── src/pages/api/core/
    └── workflow/
        └── debug.ts             # 调试接口
```

### 2.2 核心类与接口

#### 2.2.1 工作流调度器核心类

```typescript
// WorkflowQueue 类 - 工作流队列管理器
class WorkflowQueue {
  // 节点映射
  runtimeNodesMap: Map<string, RuntimeNodeItemType>
  
  // 工作流执行统计
  workflowRunTimes: number
  chatResponses: ChatHistoryItemResType[]
  chatAssistantResponse: AIChatItemValueItemType[]
  chatNodeUsages: ChatNodeUsageType[]
  toolRunResponse: ToolRunResponseItemType
  nodeInteractiveResponse?: {
    entryNodeIds: string[]
    interactiveResponse: InteractiveNodeResponseType
  }
  system_memories: Record<string, any>
  
  // Debug 模式
  debugNextStepRunNodes: RuntimeNodeItemType[]
  debugNodeResponses: WorkflowDebugResponse['nodeResponses']
  
  // 队列控制
  private activeRunQueue: Set<string>           // 待执行节点队列
  private skipNodeQueue: Map<string, {...}>     // 跳过节点队列
  private runningNodeCount: number              // 正在运行的节点数
  private maxConcurrency: number                // 最大并发数
  
  // 核心方法
  addActiveNode(nodeId: string): void           // 添加待执行节点
  private processActiveNode(): void             // 处理下一个节点
  private checkNodeCanRun(node, skippedNodeIdList): Promise<void>
  async nodeRunWithActive(node): Promise<{...}> // 执行节点
  private nodeRunWithSkip(node): {...}          // 跳过节点
  handleInteractiveResult({...}): AIChatItemValueItemType
  getDebugResponse(): WorkflowDebugResponse
}
```

#### 2.2.2 核心类型定义

```typescript
// 运行时节点
interface RuntimeNodeItemType {
  nodeId: string
  name: string
  intro?: string
  avatar?: string
  flowNodeType: FlowNodeTypeEnum
  showStatus?: boolean
  version: string
  inputs: FlowNodeInputItemType[]
  outputs: FlowNodeOutputItemType[]
  isEntry: boolean
  catchError?: boolean
}

// 运行时边
interface RuntimeEdgeItemType {
  source: string
  target: string
  sourceHandle: string
  targetHandle: string
  status: 'waiting' | 'active' | 'skipped'
}

// 节点调度参数
interface ModuleDispatchProps<T> {
  res?: ServerResponse
  mode: 'test' | 'chat' | 'debug'
  node: RuntimeNodeItemType
  runtimeNodes: RuntimeNodeItemType[]
  runtimeEdges: RuntimeEdgeItemType[]
  variables: Record<string, any>
  params: T
  histories: ChatItemType[]
  query: ChatItemType['value']
  chatConfig: AppChatConfigType
  workflowStreamResponse?: WorkflowResponseType
  // ... 其他参数
}

// 节点执行结果
interface DispatchNodeResultType {
  [DispatchNodeResponseKeyEnum.nodeResponse]?: ChatHistoryItemResType
  [DispatchNodeResponseKeyEnum.skipHandleId]?: string[]
  [DispatchNodeResponseKeyEnum.toolResponses]?: any
  [DispatchNodeResponseKeyEnum.assistantResponses]?: AIChatItemValueItemType[]
  [DispatchNodeResponseKeyEnum.interactive]?: InteractiveNodeResponseType
  [DispatchNodeResponseKeyEnum.newVariables]?: Record<string, any>
  nodeDispatchUsages?: ChatNodeUsageType[]
  // ... 其他字段
}
```

### 2.3 核心流程

#### 2.3.1 工作流执行主流程

```
dispatchWorkFlow (入口函数)
  ↓
  1. 权限检查 & 余额验证
  2. 获取用户信息和配置
  3. 设置 SSE 响应头（流式模式）
  4. 处理历史记录
  5. 准备默认变量
  ↓
runWorkflow (核心执行函数)
  ↓
  1. 初始化 WorkflowQueue
  2. 获取入口节点
  3. 将入口节点加入队列
  ↓
WorkflowQueue.processActiveNode (队列处理)
  ↓
  循环处理:
    - 检查队列状态（完成/并发限制）
    - 从 activeRunQueue 取出节点
    - 调用 checkNodeCanRun
      ↓
      - 检查节点运行状态（run/skip/wait）
      - 调用 nodeRunWithActive
        ↓
        - 准备节点输入参数
        - 替换变量引用
        - 调用节点 handler (callbackMap[nodeType])
        - 处理错误捕获
        - 格式化响应数据
        - 更新变量
        - 流式返回结果
      - 或调用 nodeRunWithSkip
      ↓
      - 获取下一批节点
      - 更新边状态
      - 将后续节点加入队列
  ↓
返回执行结果
```

#### 2.3.2 节点状态判断逻辑

```typescript
// 节点状态: run | skip | wait
function checkNodeRunStatus(node, edges): 'run' | 'skip' | 'wait' {
  // 入口节点直接运行
  if (node.isEntry) return 'run'
  
  // 获取所有指向该节点的边
  const sourceEdges = edges.filter(e => e.target === node.nodeId)
  
  // 没有入边 -> skip
  if (sourceEdges.length === 0) return 'skip'
  
  // 检查所有入边状态
  const hasActiveEdge = sourceEdges.some(e => e.status === 'active')
  const allEdgesSkipped = sourceEdges.every(e => e.status === 'skipped')
  
  if (hasActiveEdge) return 'run'
  if (allEdgesSkipped) return 'skip'
  return 'wait'
}
```

#### 2.3.3 变量替换机制

```typescript
// 变量引用格式
// 1. {{$nodeId.outputKey$}}  - 引用节点输出
// 2. {{variableName}}         - 引用全局变量

function replaceEditorVariable({
  text,
  nodes,
  variables
}): string {
  // 替换节点输出引用
  text = text.replace(/\{\{\$([^.]+)\.([^$]+)\$\}\}/g, (match, nodeId, key) => {
    const node = nodes.find(n => n.nodeId === nodeId)
    const output = node?.outputs.find(o => o.id === key)
    return output?.value ?? match
  })
  
  // 替换全局变量
  text = text.replace(/\{\{([^}]+)\}\}/g, (match, key) => {
    return variables[key] ?? match
  })
  
  return text
}
```

## 3. 节点类型详解

### 3.1 工作流入口节点 (workflowStart)

**功能**: 工作流起点，接收用户输入和全局变量

**关键代码位置**:
- 模板: `packages/global/core/workflow/template/system/workflowStart.ts`
- 执行: `packages/service/core/workflow/dispatch/init/workflowStart.tsx`

**输入参数**:
```typescript
{
  userChatInput: string      // 用户问题
  variables: object          // 全局变量
  files: FileType[]          // 文件列表
}
```

**输出结果**:
```typescript
{
  userChatInput: string
  [variableKey]: any         // 动态变量输出
  // 系统变量
  cTime: string             // 当前时间
  userId: string            // 用户ID
  chatId: string            // 对话ID
  appId: string             // 应用ID
}
```

### 3.2 AI 对话节点 (aiChat)

**功能**: 调用大语言模型进行对话

**关键代码位置**:
- 模板: `packages/global/core/workflow/template/system/aiChat/index.ts`
- 执行: `packages/service/core/workflow/dispatch/ai/chat.ts`

**配置参数**:
```typescript
{
  model: string              // 模型选择 (gpt-4, gpt-3.5等)
  temperature: number        // 温度 0-1
  maxToken: number           // 最大token数
  systemPrompt: string       // 系统提示词
  history: number            // 历史记录条数
  quoteTemplate: string      // 引用模板
  quotePrompt: string        // 引用提示词
  aiChatDatasetQuote: array  // 知识库引用
  userChatInput: string      // 用户输入
}
```

**核心逻辑**:
```typescript
async function dispatchChatCompletion(props) {
  // 1. 准备对话消息
  const messages = [
    { role: 'system', content: systemPrompt },
    ...historyMessages,
    { role: 'user', content: formatUserInput() }
  ]
  
  // 2. 调用 LLM
  const response = await requestChatCompletion({
    model,
    temperature,
    maxTokens,
    messages,
    stream: true
  })
  
  // 3. 处理流式响应
  for await (const chunk of response) {
    workflowStreamResponse({
      event: SseResponseEventEnum.answer,
      data: { text: chunk }
    })
  }
  
  // 4. 返回结果
  return {
    [DispatchNodeResponseKeyEnum.nodeResponse]: {
      totalPoints: usage.totalPoints,
      model: model,
      tokens: usage.tokens,
      query: userInput,
      textOutput: completeText
    },
    [DispatchNodeResponseKeyEnum.assistantResponses]: [
      { type: 'text', text: { content: completeText } }
    ]
  }
}
```

### 3.3 知识库检索节点 (datasetSearch)

**功能**: 从知识库中检索相关内容

**关键代码位置**:
- 模板: `packages/global/core/workflow/template/system/datasetSearch.ts`
- 执行: `packages/service/core/workflow/dispatch/dataset/search.ts`

**配置参数**:
```typescript
{
  datasets: SelectedDatasetType[]  // 知识库列表
  similarity: number               // 相似度阈值 0-1
  limit: number                    // 返回结果数量
  searchMode: 'embedding' | 'fullText' | 'hybrid'
  usingReRank: boolean             // 是否使用重排序
  datasetSearchUsingExtensionQuery: boolean  // 查询扩展
  datasetSearchExtensionModel: string        // 扩展模型
  datasetSearchExtensionBg: string           // 扩展背景
}
```

**执行流程**:
```typescript
async function dispatchDatasetSearch(props) {
  const { datasets, query, similarity, limit } = props
  
  // 1. 查询扩展（可选）
  const queries = datasetSearchUsingExtensionQuery
    ? await queryExtension(query)
    : [query]
  
  // 2. 向量检索
  const searchResults = await Promise.all(
    queries.map(q => searchDataset({
      datasets,
      query: q,
      similarity,
      limit
    }))
  )
  
  // 3. 重排序（可选）
  const rankedResults = usingReRank
    ? await rerankResults(searchResults)
    : searchResults
  
  // 4. 返回结果
  return {
    quoteQA: formatSearchResults(rankedResults),
    [DispatchNodeResponseKeyEnum.nodeResponse]: {
      searchRes: rankedResults,
      totalPoints: usage
    }
  }
}
```

### 3.4 HTTP 请求节点 (http468)

**功能**: 发送 HTTP 请求，支持各种请求方式和参数

**关键代码位置**:
- 模板: `packages/global/core/workflow/template/system/http468.ts`
- 执行: `packages/service/core/workflow/dispatch/tools/http468.ts`

**配置参数**:
```typescript
{
  system_httpMethod: 'GET' | 'POST' | 'PUT' | 'DELETE'
  system_httpReqUrl: string
  system_httpHeader: Array<{key: string, value: string, type: string}>
  system_httpParams: Array<{key: string, value: string}>
  system_httpJsonBody: string
  system_httpFormBody: Array<{key: string, value: string}>
  system_httpContentType: ContentTypes
  system_httpTimeout: number
  [NodeInputKeyEnum.addInputParam]: Record<string, any>  // 动态参数
}
```

**核心功能**:
```typescript
async function dispatchHttp468(props) {
  const { httpMethod, httpReqUrl, httpHeader, httpParams } = props
  
  // 1. 变量替换
  const url = replaceEditorVariable({ text: httpReqUrl, nodes, variables })
  
  // 2. 构建请求头
  const headers = httpHeader.reduce((acc, item) => {
    acc[replaceVariables(item.key)] = replaceVariables(item.value)
    return acc
  }, {})
  
  // 3. 构建请求体
  const body = buildRequestBody({ httpContentType, httpJsonBody, httpFormBody })
  
  // 4. 发送请求
  const response = await axios({
    method: httpMethod,
    url,
    headers,
    params: httpParams,
    data: body,
    timeout: httpTimeout * 1000
  })
  
  // 5. JSONPath 提取结果
  const results = node.outputs.reduce((acc, output) => {
    const path = output.key.startsWith('$') ? output.key : `$.${output.key}`
    acc[output.key] = JSONPath({ path, json: response.data })
    return acc
  }, {})
  
  return {
    data: results,
    [DispatchNodeResponseKeyEnum.nodeResponse]: {
      httpResult: response.data
    }
  }
}
```

### 3.5 条件分支节点 (ifElse)

**功能**: 根据条件判断执行不同分支

**关键代码位置**:
- 模板: `packages/global/core/workflow/template/system/ifElse/index.ts`
- 执行: `packages/service/core/workflow/dispatch/tools/runIfElse.ts`

**配置参数**:
```typescript
{
  ifElseList: Array<{
    condition: 'AND' | 'OR'
    list: Array<{
      variable: string
      operator: '===' | '!==' | '>' | '<' | 'includes' | 'notIncludes' | ...
      value: any
    }>
  }>
}
```

**判断逻辑**:
```typescript
function dispatchIfElse(props) {
  const { ifElseList, variables } = props
  
  // 遍历条件列表
  for (const [index, item] of ifElseList.entries()) {
    const { condition, list } = item
    
    // 计算条件结果
    const results = list.map(({ variable, operator, value }) => {
      const varValue = getReferenceVariableValue({ value: variable, variables })
      return compareValues(varValue, operator, value)
    })
    
    // AND / OR 逻辑
    const pass = condition === 'AND'
      ? results.every(r => r)
      : results.some(r => r)
    
    if (pass) {
      // 返回对应分支的 handle
      return {
        [DispatchNodeResponseKeyEnum.skipHandleId]: 
          getAllHandlesExcept(index)
      }
    }
  }
  
  // 所有条件都不满足，走 else 分支
  return {
    [DispatchNodeResponseKeyEnum.skipHandleId]: 
      getAllConditionHandles()
  }
}
```

### 3.6 循环节点 (loop)

**功能**: 遍历数组，对每个元素执行子流程

**关键文件**:
- `loop/loopStart.ts`: 循环开始节点
- `loop/loop.ts`: 循环体节点  
- `loop/loopEnd.ts`: 循环结束节点

**执行流程**:
```
loopStart (开始)
  ↓ 设置循环变量
  ├→ loop (循环体) → 执行子流程
  │   ↓
  │   loopEnd (结束) 
  │   ↓
  └─ (继续循环或结束)
```

### 3.7 交互节点

#### 3.7.1 表单输入 (formInput)

**功能**: 暂停工作流，等待用户填写表单

**执行逻辑**:
```typescript
function dispatchFormInput(props) {
  return {
    [DispatchNodeResponseKeyEnum.interactive]: {
      type: 'formInput',
      params: {
        description: '请填写以下信息',
        formInputs: [
          { label: '姓名', key: 'name', required: true },
          { label: '年龄', key: 'age', type: 'number' }
        ]
      }
    }
  }
}
```

#### 3.7.2 用户选择 (userSelect)

**功能**: 暂停工作流，等待用户选择选项

**配置参数**:
```typescript
{
  description: string
  options: Array<{
    label: string
    value: string
  }>
}
```

### 3.8 代码沙盒节点 (sandbox)

**功能**: 执行 JavaScript/Python 代码

**关键代码位置**:
- 模板: `packages/global/core/workflow/template/system/sandbox/index.ts`
- 执行: `packages/service/core/workflow/dispatch/tools/codeSandbox.ts`

**配置参数**:
```typescript
{
  language: 'javascript' | 'python'
  code: string
  variables: Record<string, any>
}
```

### 3.9 工具调用节点

#### 3.9.1 运行工具 (runTool)

**功能**: 调用 MCP 工具或 HTTP 工具

**支持的工具类型**:
- MCP 工具 (Model Context Protocol)
- HTTP 工具集
- 系统工具

#### 3.9.2 运行应用 (runApp)

**功能**: 调用其他工作流应用

**执行逻辑**:
```typescript
async function dispatchRunApp(props) {
  const { appId, variables } = props
  
  // 递归调用 runWorkflow
  const result = await runWorkflow({
    ...props,
    appId,
    variables,
    workflowDispatchDeep: props.workflowDispatchDeep + 1
  })
  
  return result
}
```

## 4. 变量系统

### 4.1 变量类型

#### 4.1.1 全局变量
- 用户定义的工作流全局变量
- 可在任意节点引用和修改
- 通过 `{{variableName}}` 引用

#### 4.1.2 系统变量
```typescript
{
  cTime: string        // 当前时间
  userId: string       // 用户ID
  chatId: string       // 对话ID
  appId: string        // 应用ID
  timezone: string     // 时区
  histories: array     // 历史记录
}
```

#### 4.1.3 节点输出变量
- 每个节点的输出可被后续节点引用
- 通过 `{{$nodeId.outputKey$}}` 引用

### 4.2 变量引用语法

```typescript
// 1. 全局变量引用
"用户名是: {{userName}}"

// 2. 节点输出引用
"检索结果: {{$datasetSearch.quoteQA$}}"

// 3. 动态输入引用
"{{addInputParam.customField}}"

// 4. 嵌套引用
"处理 {{$node1.data$}} 的结果: {{$node2.result$}}"
```

### 4.3 变量类型转换

```typescript
function valueTypeFormat(value: any, valueType: WorkflowIOValueTypeEnum) {
  switch (valueType) {
    case 'string':
      return String(value)
    case 'number':
      return Number(value)
    case 'boolean':
      return Boolean(value)
    case 'object':
      return typeof value === 'object' ? value : JSON.parse(value)
    case 'arrayString':
    case 'arrayNumber':
    case 'arrayBoolean':
    case 'arrayObject':
      return Array.isArray(value) ? value : [value]
    case 'any':
    default:
      return value
  }
}
```

## 5. 队列管理机制

### 5.1 并发控制

```typescript
class WorkflowQueue {
  private maxConcurrency = 10       // 最大并发数
  private runningNodeCount = 0      // 正在运行的节点数
  private activeRunQueue = new Set<string>()  // 待执行队列
  
  private processActiveNode() {
    // 1. 检查完成条件
    if (this.activeRunQueue.size === 0 && this.runningNodeCount === 0) {
      return this.resolve()
    }
    
    // 2. 检查并发限制
    if (this.runningNodeCount >= this.maxConcurrency) {
      return
    }
    
    // 3. 取出一个节点执行
    const nodeId = this.activeRunQueue.keys().next().value
    this.activeRunQueue.delete(nodeId)
    this.runningNodeCount++
    
    this.checkNodeCanRun(node).finally(() => {
      this.runningNodeCount--
      this.processActiveNode()  // 继续处理下一个
    })
  }
}
```

### 5.2 跳过节点队列

当节点的所有入边都被跳过时，该节点也应该被跳过：

```typescript
private skipNodeQueue = new Map<string, {
  node: RuntimeNodeItemType
  skippedNodeIdList: Set<string>  // 已跳过的前置节点
}>()

private processSkipNodes() {
  const skipItem = this.skipNodeQueue.values().next().value
  if (skipItem) {
    this.skipNodeQueue.delete(skipItem.node.nodeId)
    this.checkNodeCanRun(skipItem.node, skipItem.skippedNodeIdList)
      .finally(() => this.processActiveNode())
  }
}
```

### 5.3 边状态管理

```typescript
type EdgeStatus = 'waiting' | 'active' | 'skipped'

// 边状态转换
function updateEdgeStatus(edges, sourceNodeId, skipHandleIds) {
  edges.forEach(edge => {
    if (edge.source === sourceNodeId) {
      edge.status = skipHandleIds.includes(edge.sourceHandle)
        ? 'skipped'
        : 'active'
    }
  })
}
```

## 6. 错误处理机制

### 6.1 节点错误捕获

```typescript
interface RuntimeNodeItemType {
  catchError?: boolean  // 是否捕获错误
}

// 错误处理逻辑
if (node.catchError) {
  // 捕获错误，走错误分支
  const errorHandle = getHandleId(node.nodeId, 'source_catch', 'right')
  return {
    error: { [NodeOutputKeyEnum.error]: errorMessage },
    [DispatchNodeResponseKeyEnum.skipHandleId]: 
      allHandles.filter(h => h !== errorHandle)
  }
} else {
  // 不捕获错误，跳过所有后续节点
  return {
    [DispatchNodeResponseKeyEnum.skipHandleId]: allHandles
  }
}
```

### 6.2 全局错误处理

```typescript
try {
  const result = await callbackMap[node.flowNodeType](dispatchData)
  return result
} catch (error) {
  addLog.error('Node execution error', { nodeId: node.nodeId, error })
  
  // 返回错误响应
  return {
    [DispatchNodeResponseKeyEnum.nodeResponse]: {
      error: getErrText(error)
    },
    [DispatchNodeResponseKeyEnum.skipHandleId]: 
      allEdges.map(e => e.sourceHandle)
  }
}
```

## 7. 流式响应机制

### 7.1 SSE 配置

```typescript
// 设置 SSE 响应头
res.setHeader('Content-Type', 'text/event-stream;charset=utf-8')
res.setHeader('Access-Control-Allow-Origin', '*')
res.setHeader('X-Accel-Buffering', 'no')
res.setHeader('Cache-Control', 'no-cache, no-transform')
res.setHeader('Connection', 'keep-alive')

// 心跳保活
const heartbeatTimer = setInterval(() => {
  workflowStreamResponse?.({
    event: SseResponseEventEnum.answer,
    data: textAdaptGptResponse({ text: '' })
  })
}, 10000)
```

### 7.2 事件类型

```typescript
enum SseResponseEventEnum {
  answer = 'answer',                    // AI 回答内容
  flowNodeStatus = 'flowNodeStatus',    // 节点状态
  flowNodeResponse = 'flowNodeResponse',// 节点响应
  interactive = 'interactive',          // 交互请求
  workflowDuration = 'workflowDuration' // 工作流耗时
}
```

### 7.3 流式数据格式

```typescript
// SSE 数据格式
function responseWrite({ res, event, data }) {
  if (!res || res.closed) return
  
  const response = event
    ? `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`
    : `data: ${JSON.stringify(data)}\n\n`
  
  res.write(response)
}
```

## 8. 调试模式

### 8.1 调试功能

```typescript
interface WorkflowDebugResponse {
  memoryEdges: RuntimeEdgeItemType[]    // 边状态快照
  memoryNodes: RuntimeNodeItemType[]    // 节点状态快照
  entryNodeIds: string[]                // 下一步执行的节点
  nodeResponses: {                      // 节点执行结果
    [nodeId: string]: {
      nodeId: string
      type: 'run' | 'skip'
      response?: ChatHistoryItemResType
      interactiveResponse?: InteractiveNodeResponseType
    }
  }
  skipNodeQueue: Array<{                // 跳过节点队列
    id: string
    skippedNodeIdList: string[]
  }>
}
```

### 8.2 单步执行

```typescript
// Debug 模式下暂停执行，记录下一步节点
if (isDebugMode) {
  this.debugNextStepRunNodes = this.debugNextStepRunNodes.concat(nextStepActiveNodes)
  // 不自动执行，等待用户操作
} else {
  // 正常模式自动执行
  nextStepActiveNodes.forEach(node => {
    this.addActiveNode(node.nodeId)
  })
}
```

### 8.3 断点续传

```typescript
// 从上次中断的地方继续执行
async function continueDebug(props: {
  memoryEdges: RuntimeEdgeItemType[]
  entryNodeIds: string[]
  skipNodeQueue: Array<{...}>
}) {
  const workflowQueue = new WorkflowQueue({
    defaultSkipNodeQueue: props.skipNodeQueue
  })
  
  // 恢复边状态
  runtimeEdges = props.memoryEdges
  
  // 从入口节点继续
  props.entryNodeIds.forEach(nodeId => {
    workflowQueue.addActiveNode(nodeId)
  })
  
  return await workflowQueue.run()
}
```

## 9. 交互式工作流

### 9.1 交互类型

```typescript
interface InteractiveNodeResponseType {
  type: 'formInput' | 'userSelect' | 'paymentPause'
  params: {
    description?: string
    formInputs?: FormInputItem[]
    options?: SelectOption[]
  }
}
```

### 9.2 交互流程

```
用户发起请求
  ↓
执行到交互节点
  ↓
暂停工作流，返回交互请求
  ↓
前端展示表单/选项
  ↓
用户提交表单/选择选项
  ↓
携带交互结果继续请求
  ↓
从交互节点继续执行
  ↓
返回最终结果
```

### 9.3 交互数据传递

```typescript
interface WorkflowInteractiveResponseType extends InteractiveNodeResponseType {
  entryNodeIds: string[]          // 交互节点列表
  memoryEdges: RuntimeEdgeItemType[]
  nodeOutputs: NodeOutputItemType[]  // 当前所有节点输出
  skipNodeQueue: Array<{...}>
  usageId?: string
}

// 续传请求
function continueWorkflow(props: {
  lastInteractive: WorkflowInteractiveResponseType
  interactiveResult: any  // 用户输入的结果
}) {
  // 将结果注入到交互节点
  // 从 entryNodeIds 继续执行
}
```

## 10. 资源计费

### 10.1 消耗追踪

```typescript
interface ChatNodeUsageType {
  nodeId: string
  nodeName: string
  model?: string
  tokens?: number
  totalPoints: number  // 消耗积分
  charsLength?: number
}

// 实时记录消耗
function pushStore({ nodeDispatchUsages }) {
  if (nodeDispatchUsages) {
    // 立即写入数据库，避免工作流消耗大量积分
    pushChatItemUsage({
      teamId,
      usageId,
      nodeUsages: nodeDispatchUsages
    })
    
    this.chatNodeUsages = this.chatNodeUsages.concat(nodeDispatchUsages)
  }
}
```

### 10.2 余额检查

```typescript
private async checkTeamBlance() {
  try {
    await checkTeamAIPoints(data.runningUserInfo.teamId)
  } catch (error) {
    if (error === TeamErrEnum.aiPointsNotEnough) {
      // 余额不足，暂停工作流
      return {
        [DispatchNodeResponseKeyEnum.interactive]: {
          type: 'paymentPause',
          params: {
            description: '余额不足，请充值后继续'
          }
        }
      }
    }
  }
}
```

## 11. 性能优化

### 11.1 并发执行

- 支持无依赖节点并发执行
- 可配置最大并发数（默认 10）
- 自动管理节点执行队列

### 11.2 流式输出

- AI 节点支持流式返回
- 减少用户等待时间
- 降低服务器内存占用

### 11.3 变量引用优化

```typescript
// 延迟计算变量值
// 只在节点真正需要时才替换变量
function replaceEditorVariable({ text, nodes, variables }) {
  // 只替换当前节点用到的变量
  // 避免全量替换
}
```

### 11.4 资源控制

```typescript
// 最大运行次数限制
const WORKFLOW_MAX_RUN_TIMES = 100

// 运行深度限制（防止无限递归）
if (data.workflowDispatchDeep > 20) {
  return // 终止执行
}
```

## 12. API 接口

### 12.1 工作流执行接口

**路由**: `POST /api/v1/chat/completions` 或 `POST /api/v2/chat/completions`

**请求参数**:
```typescript
{
  chatId?: string
  appId: string
  messages: ChatCompletionMessageParam[]
  variables?: Record<string, any>
  stream?: boolean
  detail?: boolean
}
```

**响应格式**:
```typescript
// 流式响应 (stream=true)
event: answer
data: {"text": "...", "finish_reason": null}

event: flowNodeResponse
data: {"nodeId": "...", "moduleName": "...", ...}

event: answer
data: {"text": null, "finish_reason": "stop"}

data: [DONE]

// 非流式响应
{
  choices: [{
    message: { role: 'assistant', content: '...' }
  }],
  responseData: [...],  // 节点执行结果（detail=true时返回）
  usage: { totalPoints: 100 }
}
```

### 12.2 工作流调试接口

**路由**: `POST /api/core/workflow/debug`

**请求参数**:
```typescript
{
  nodes: StoreNodeItemType[]
  edges: StoreEdgeItemType[]
  variables: Record<string, any>
  chatId: string
  appId: string
  // 续传参数（单步执行）
  memoryEdges?: RuntimeEdgeItemType[]
  entryNodeIds?: string[]
  skipNodeQueue?: Array<{...}>
}
```

**响应**:
```typescript
{
  finishMessages: AIChatItemValueItemType[]
  debugResponse: WorkflowDebugResponse
}
```

## 13. 扩展开发

### 13.1 自定义节点开发

#### 步骤1: 定义节点模板

```typescript
// packages/global/core/workflow/template/system/customNode.ts
export const CustomNodeTemplate: FlowNodeTemplateType = {
  id: FlowNodeTypeEnum.customNode,
  flowNodeType: FlowNodeTypeEnum.customNode,
  templateType: FlowNodeTemplateTypeEnum.other,
  avatar: '/icons/custom.svg',
  name: '自定义节点',
  intro: '自定义功能节点',
  inputs: [
    {
      key: NodeInputKeyEnum.customInput,
      renderTypeList: [FlowNodeInputTypeEnum.input],
      label: '输入参数',
      valueType: WorkflowIOValueTypeEnum.string
    }
  ],
  outputs: [
    {
      key: NodeOutputKeyEnum.customOutput,
      label: '输出结果',
      valueType: WorkflowIOValueTypeEnum.string
    }
  ]
}
```

#### 步骤2: 实现节点处理器

```typescript
// packages/service/core/workflow/dispatch/custom/customNode.ts
export async function dispatchCustomNode(
  props: ModuleDispatchProps<{
    customInput: string
  }>
): Promise<DispatchNodeResultType> {
  const { params, node } = props
  
  // 实现节点逻辑
  const result = await processCustomLogic(params.customInput)
  
  return {
    data: {
      [NodeOutputKeyEnum.customOutput]: result
    },
    [DispatchNodeResponseKeyEnum.nodeResponse]: {
      totalPoints: 0,
      customOutput: result
    }
  }
}
```

#### 步骤3: 注册节点处理器

```typescript
// packages/service/core/workflow/dispatch/constants.ts
export const callbackMap: Record<
  FlowNodeTypeEnum,
  (props: ModuleDispatchProps<any>) => Promise<DispatchNodeResultType>
> = {
  // ... 其他节点
  [FlowNodeTypeEnum.customNode]: dispatchCustomNode
}
```

### 13.2 节点开发最佳实践

1. **输入验证**: 始终验证输入参数
2. **错误处理**: 使用 try-catch 捕获异常
3. **资源计费**: 记录节点消耗的资源
4. **流式输出**: 对于耗时操作，支持流式返回
5. **幂等性**: 节点应该是幂等的，相同输入产生相同输出

## 14. 测试

### 14.1 单元测试

```typescript
// test/cases/service/core/app/workflow/workflowDispatch.test.ts
describe('workflow dispatch', () => {
  test('should execute workflow correctly', async () => {
    const result = await runWorkflow({
      runtimeNodes: testNodes,
      runtimeEdges: testEdges,
      variables: { input: 'test' }
    })
    
    expect(result.assistantResponses).toBeDefined()
    expect(result.workflowRunTimes).toBeGreaterThan(0)
  })
  
  test('should handle node error', async () => {
    const result = await runWorkflow({
      runtimeNodes: errorNodes,
      runtimeEdges: testEdges
    })
    
    expect(result.flowResponses[0].error).toBeDefined()
  })
})
```

### 14.2 集成测试

测试完整的工作流执行流程，包括：
- 多节点串联执行
- 条件分支跳转
- 循环执行
- 错误捕获
- 交互式暂停与恢复

## 15. 监控与日志

### 15.1 执行日志

```typescript
addLog.debug('Run node', {
  maxRunTimes: data.maxRunTimes,
  appId: data.runningAppInfo.id,
  nodeId: node.nodeId,
  nodeName: node.name
})

addLog.error('workflow error', {
  error: dispatchRes.responseData.error,
  nodeId: node.nodeId
})
```

### 15.2 性能监控

```typescript
const startTime = Date.now()

// ... 执行工作流

const durationSeconds = +((Date.now() - startTime) / 1000).toFixed(2)

workflowStreamResponse?.({
  event: SseResponseEventEnum.workflowDuration,
  data: { durationSeconds }
})
```

## 16. 总结

工作流管理模块是 FastGPT 的核心引擎，通过以下关键设计实现了强大且灵活的工作流编排能力：

### 16.1 核心优势

1. **可视化编排**: 通过节点和边的方式定义工作流，降低使用门槛
2. **丰富的节点类型**: 覆盖 AI 对话、知识库检索、HTTP 请求、条件分支等多种场景
3. **灵活的变量系统**: 支持全局变量、节点输出引用、动态输入等
4. **并发执行**: 无依赖节点可并发执行，提升性能
5. **错误处理**: 完善的错误捕获和处理机制
6. **流式响应**: 实时返回执行结果，优化用户体验
7. **交互式节点**: 支持表单输入、用户选择等交互场景
8. **调试功能**: 支持单步执行、断点续传等调试能力
9. **资源计费**: 精确追踪各节点的资源消耗

### 16.2 技术亮点

- **队列管理**: 采用双队列（active + skip）管理节点执行
- **DAG 调度**: 基于有向无环图的依赖分析和拓扑排序
- **边状态管理**: 通过边状态（waiting/active/skipped）控制执行流
- **深度控制**: 防止无限递归，支持嵌套工作流调用
- **SSE 流式**: 实现实时响应，降低延迟

### 16.3 应用场景

- 智能客服对话流程
- 文档处理工作流
- 数据分析管道
- 自动化任务编排
- 多模态交互应用
- 复杂业务流程自动化

### 16.4 未来展望

- 支持更多节点类型（如图像处理、音频处理）
- 优化执行效率（更智能的并发控制）
- 增强调试能力（可视化执行轨迹）
- 支持工作流版本管理
- 增加工作流模板市场
