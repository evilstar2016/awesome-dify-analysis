# 工作流引擎 (Workflow Engine)

## 概述
工作流引擎是 FastGPT 的核心能力之一，负责解析和执行用户通过可视化编排的工作流。引擎支持多种节点类型、变量系统、条件分支、工具调用等复杂逻辑。

## 核心架构

### 执行流程
```
1. 加载工作流配置（nodes + edges）
2. 初始化运行时环境
   - 全局变量
   - 系统变量
   - 用户上下文
3. 解析节点依赖关系（DAG）
4. 按拓扑顺序执行节点
   - 获取节点输入（从前置节点输出）
   - 调用节点 handler
   - 存储节点输出
   - 处理条件分支
5. 流式返回结果（SSE）
6. 记录执行日志与消费
```

### 核心文件
```
packages/service/core/workflow/dispatch/
├── index.ts                 # 工作流主调度器 (dispatchWorkFlow)
├── type.d.ts                # 类型定义
├── utils.ts                 # 工具函数
├── constants.ts             # 常量定义
├── ai/                      # AI 相关节点
│   ├── chat.ts              # AI 对话节点
│   ├── classifyQuestion.ts  # 问题分类节点
│   ├── extract.ts           # 内容提取节点
│   └── tool/                # 工具调用
│       ├── index.ts         # 工具调用入口
│       ├── toolCall.ts      # 工具执行
│       └── utils.ts         # 工具辅助函数
├── child/                   # 子工作流
│   ├── runApp.ts            # 运行应用
│   └── runTool.ts           # 运行工具
├── dataset/                 # 知识库节点
│   ├── search.ts            # 知识库检索
│   └── concat.ts            # 结果合并
├── init/                    # 初始化节点
│   ├── systemConfig.tsx     # 系统配置
│   └── workflowStart.tsx    # 工作流入口
├── interactive/             # 交互节点
│   ├── formInput.ts         # 表单输入
│   └── userSelect.ts        # 用户选择
├── loop/                    # 循环节点
│   ├── runLoop.ts           # 循环执行
│   ├── runLoopStart.ts      # 循环开始
│   └── runLoopEnd.ts        # 循环结束
├── plugin/                  # 插件节点
│   ├── run.ts               # 运行插件
│   ├── runInput.ts          # 插件输入
│   └── runOutput.ts         # 插件输出
└── tools/                   # 工具节点集合
    ├── answer.ts            # 回答节点
    ├── codeSandbox.ts       # 代码沙盒
    ├── customFeedback.ts    # 自定义反馈
    ├── http468.ts           # HTTP 请求
    ├── queryExternsion.ts   # 查询扩展
    ├── readFiles.ts         # 读取文件
    ├── runIfElse.ts         # 条件判断
    ├── runLaf.ts            # Laf 执行
    ├── runUpdateVar.ts      # 变量更新
    └── textEditor.ts        # 文本编辑
```

## 节点类型详解

### 1. 工作流入口节点 (Entry)
**功能**: 工作流起点，接收用户输入

**输入**
- 用户问题 (userChatInput)
- 全局变量 (variables)
- 文件列表 (files)

**输出**
- 用户问题
- 全局变量
- 系统变量 (userId, chatId, appId 等)

### 2. AI 对话节点 (AI Chat)
**功能**: 调用 LLM 生成回复

**配置**
```typescript
{
  model: 'gpt-4',              // 模型选择
  temperature: 0.7,            // 温度
  maxTokens: 2000,             // 最大 Token
  systemPrompt: '...',         // 系统提示词
  userChatInput: '{{question}}', // 用户输入（变量引用）
  history: 6,                  // 历史记录条数
  quoteQA: '{{quoteQA}}',     // 知识库引用
  tools: [...]                 // 工具列表（函数调用）
}
```

**输出**
- AI 回复 (text)
- 工具调用结果 (toolCalls)
- Token 消耗 (usage)

### 3. 知识库检索节点 (Dataset Search)
**功能**: 从知识库检索相关内容

**配置**
```typescript
{
  datasets: ['datasetId1', 'datasetId2'], // 知识库列表
  searchMode: 'embedding',     // 检索模式：embedding/fulltext/hybrid
  similarity: 0.5,             // 相似度阈值
  limit: 10,                   // 返回数量
  rerank: {
    model: 'bge-reranker',
    topK: 5
  },
  queryExtension: true         // 查询扩展
}
```

**输出**
- 检索结果列表 (quoteQA)
- 相似度分数 (scores)
- 来源信息 (sources)

### 4. 问题分类节点 (Classify Question)
**功能**: 对用户问题进行分类

**配置**
```typescript
{
  model: 'gpt-3.5-turbo',
  systemPrompt: '根据用户问题分类...',
  categories: [
    { key: 'product', value: '产品咨询' },
    { key: 'technical', value: '技术支持' },
    { key: 'other', value: '其他' }
  ]
}
```

**输出**
- 分类结果 (category)
- 置信度 (confidence)
- 分支路由 (通过 edge 连接不同后续节点)

### 5. 内容提取节点 (Content Extract)
**功能**: 从文本中提取结构化信息

**配置**
```typescript
{
  model: 'gpt-4',
  extractFields: [
    { key: 'name', desc: '姓名', required: true },
    { key: 'phone', desc: '电话', required: false }
  ],
  description: '从文本中提取用户信息'
}
```

**输出**
- 提取结果 (extractedData)
- 原始文本 (rawText)

### 6. HTTP 请求节点 (HTTP Request)
**功能**: 调用外部 HTTP API

**配置**
```typescript
{
  method: 'POST',
  url: 'https://api.example.com/endpoint',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer {{apiKey}}'
  },
  body: {
    query: '{{question}}'
  },
  timeout: 30000
}
```

**输出**
- 响应状态 (status)
- 响应体 (body)
- 响应头 (headers)

### 7. 代码执行节点 (Code Sandbox)
**功能**: 执行用户代码（Python/JavaScript）

**配置**
```typescript
{
  language: 'python',
  code: `
def process(input_data):
    # 用户代码
    return result
`,
  inputs: {
    input_data: '{{userInput}}'
  }
}
```

**输出**
- 执行结果 (output)
- 错误信息 (error)

### 8. 变量赋值节点 (Variable)
**功能**: 创建或更新变量

**配置**
```typescript
{
  variables: [
    { key: 'userName', value: '{{extractedData.name}}' },
    { key: 'count', value: '{{add count 1}}' }  // 支持表达式
  ]
}
```

**输出**
- 更新后的变量

### 9. 条件判断节点 (If/Else)
**功能**: 根据条件分支

**配置**
```typescript
{
  condition: '{{category}} == "product"',
  ifTrue: 'nodeId1',    // 满足条件跳转
  ifFalse: 'nodeId2'    // 不满足跳转
}
```

### 10. 回答节点 (Answer)
**功能**: 输出最终回复

**配置**
```typescript
{
  text: '{{aiResponse}}\n\n引用来源：{{quoteQA}}'
}
```

**输出**
- 最终回复文本

### 11. MCP 工具节点 (MCP Tool)
**功能**: 调用 MCP 协议工具

**配置**
```typescript
{
  mcpServerId: 'mcp-server-id',
  toolName: 'search',
  parameters: {
    query: '{{question}}'
  }
}
```

**输出**
- 工具执行结果

### 12. 插件节点 (Plugin)
**功能**: 运行已安装的插件

**配置**
```typescript
{
  pluginId: 'plugin-xxx',
  toolName: 'customTool',
  inputs: {
    param1: '{{value1}}',
    param2: '{{value2}}'
  }
}
```

**输出**
- 插件返回结果

## 变量系统

### 变量类型
1. **系统变量**: userId, chatId, appId, timezone 等
2. **全局变量**: 用户定义的持久化变量
3. **节点输出变量**: 每个节点的输出可被后续节点引用
4. **临时变量**: 节点内部计算的临时值

### 变量引用语法
```typescript
// 直接引用
{{variableName}}

// 嵌套引用
{{nodeId.outputKey}}

// 条件表达式
{{if condition then value1 else value2}}

// 数组索引
{{arrayVar[0]}}

// 对象属性
{{objectVar.property}}
```

### 变量处理
```typescript
// 替换变量
const processedText = replaceEditorVariable({
  text: 'Hello {{userName}}, your order is {{orderId}}',
  variables: {
    userName: 'Alice',
    orderId: '12345'
  }
});
// 结果: "Hello Alice, your order is 12345"
```

## 边（Edge）管理

### 边的类型
1. **顺序边**: 默认连接，按顺序执行
2. **条件边**: 根据节点输出决定下一步
3. **工具调用边**: AI 函数调用触发

### 边的筛选
```typescript
// 根据条件过滤边
const nextEdges = filterWorkflowEdges({
  edges: allEdges,
  nodeId: currentNodeId,
  runningStatus: nodeResult.status
});
```

## 执行控制

### 节点状态
```typescript
enum NodeRunStatusEnum {
  WAIT = 'wait',        // 等待执行
  RUNNING = 'running',  // 执行中
  SUCCESS = 'success',  // 成功
  SKIP = 'skip',        // 跳过
  ERROR = 'error'       // 错误
}
```

### 跳过逻辑
- 条件不满足跳过
- 前置节点失败跳过
- 循环次数达到上限跳过

### 错误处理
```typescript
try {
  const result = await executeNode(node);
} catch (error) {
  // 记录错误
  nodeResult.error = error.message;
  nodeResult.status = 'error';
  
  // 是否继续执行
  if (node.continueOnError) {
    return nodeResult;
  } else {
    throw error;
  }
}
```

## 流式响应（SSE）

### 事件类型
```typescript
enum SseResponseEventEnum {
  ANSWER = 'answer',          // 回答流
  TOOL_CALL = 'toolCall',     // 工具调用
  TOOL_RESPONSE = 'toolResponse', // 工具响应
  WORKFLOW_START = 'workflowStart',
  WORKFLOW_END = 'workflowEnd',
  NODE_START = 'nodeStart',
  NODE_END = 'nodeEnd',
  ERROR = 'error'
}
```

### 流式发送
```typescript
// 发送节点开始事件
res.write(`event: nodeStart\ndata: ${JSON.stringify({
  nodeId: node.nodeId,
  name: node.name
})}\n\n`);

// 流式发送 AI 回复
for await (const chunk of aiStream) {
  res.write(`event: answer\ndata: ${JSON.stringify({
    text: chunk.content
  })}\n\n`);
}

// 发送节点结束事件
res.write(`event: nodeEnd\ndata: ${JSON.stringify({
  nodeId: node.nodeId,
  outputs: nodeResult
})}\n\n`);
```

## 调试模式

### 调试信息
```typescript
{
  nodeId: string,
  nodeName: string,
  startTime: number,
  endTime: number,
  duration: number,
  status: NodeRunStatusEnum,
  inputs: Record<string, any>,
  outputs: Record<string, any>,
  error?: string,
  logs: string[]
}
```

### 单步调试
- 暂停执行
- 查看当前变量
- 修改变量值
- 继续/重试

## 性能优化

### 并发执行
- 无依赖节点并发执行
- 独立分支并行处理

### 缓存策略
- 知识库检索结果缓存
- HTTP 请求结果缓存（可选）

### 超时控制
- 节点级超时
- 工作流整体超时

## 使用示例

### 简单对话流程
```
[工作流入口] → [知识库检索] → [AI 对话] → [回答]
```

### 问题分类流程
```
[工作流入口] → [问题分类]
                    ├─ [产品咨询] → [AI 对话 A] → [回答]
                    ├─ [技术支持] → [AI 对话 B] → [回答]
                    └─ [其他] → [AI 对话 C] → [回答]
```

### 复杂业务流程
```
[工作流入口] → [内容提取] → [HTTP 请求(查询订单)]
                                    ↓
                              [条件判断]
                          ├─ [已支付] → [AI 对话] → [回答]
                          └─ [未支付] → [变量赋值] → [HTTP 请求(发送提醒)] → [回答]
```

## 相关文档
- [前端展示层](./01-frontend-layer.md)
- [API 路由层](./02-api-routes-layer.md)
- [业务服务层](./03-service-layer.md)
