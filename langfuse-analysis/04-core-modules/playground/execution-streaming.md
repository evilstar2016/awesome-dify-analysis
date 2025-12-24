# Execution Streaming 管理

## 功能概述

Execution Streaming 是 Playground 模块的**核心执行引擎**，负责与 LLM API 交互，支持流式与非流式两种执行模式、工具调用、结构化输出等高级功能。通过 AbortController 实现请求中断，通过 AsyncGenerator 实现流式响应。核心设计：
- **双模式执行**：Streaming（SSE）/ Non-Streaming（JSON）
- **工具集成**：支持多轮 Tool Call + Tool Result 对话
- **结构化输出**：强制 LLM 输出符合 JSON Schema
- **请求中断**：AbortController 实时停止执行

---

## 核心组件

### 1. **handleSubmit 主执行函数**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L307-L444)

#### **执行流程（138 行）**

| 步骤 | 代码行 | 功能 | 耗时（估算） |
|-----|-------|------|-----------|
| **1. 前置验证** | 310-324 | 检查模型配置、消息内容 | < 1ms |
| **2. 状态初始化** | 326-332 | 清空输出、设置执行状态 | < 1ms |
| **3. 消息编译** | 334-340 | 替换变量、处理占位符 | 1-5ms |
| **4. 路由选择** | 342-392 | 根据配置选择执行路径 | < 1ms |
| **5. 流式执行** | 380-390 | SSE 流式处理 | 1-30s |
| **6. 非流式执行** | 391-395 | 一次性返回结果 | 1-10s |
| **7. 输出处理** | 397-426 | 解析 JSON、Tool Calls | 1-10ms |
| **8. 错误处理** | 428-442 | 捕获异常、显示错误 | - |

#### **关键逻辑片段**

**前置验证（Lines 310-324）**：
```typescript
// Lines 310-324: 前置检查（15 行）
if (!modelParams.provider || !modelParams.model) {
  showErrorToast("No model configured");
  return;
}

const hasContent = messages.some(msg => 
  (msg.type === "user" || msg.type === "system") && msg.content.trim()
);
if (!hasContent) {
  showErrorToast("Please add at least one message with content");
  return;
}
```

**执行路径选择（Lines 342-395）**：
```typescript
// Lines 342-395: 执行路径路由（54 行）
let response: string | null = null;

// 路径 1: 工具调用模式
if (tools && tools.length > 0) {
  const completion = await getChatCompletionWithTools(
    projectId,
    modelParams,
    finalMessages,
    tools,
  );
  response = completion;
}
// 路径 2: 结构化输出模式
else if (structuredOutputSchema) {
  response = await getChatCompletionWithStructuredOutput(
    projectId,
    modelParams,
    finalMessages,
    structuredOutputSchema,
  );
}
// 路径 3: 流式模式
else if (streaming) {
  const completionStream = getChatCompletionStream(
    projectId,
    modelParams,
    finalMessages,
    abortControllerRef.current.signal,
  );
  // 逐块处理流式响应
  for await (const chunk of completionStream) {
    setOutput(prev => (prev || "") + chunk);
  }
}
// 路径 4: 非流式模式
else {
  response = await getChatCompletionNonStreaming(
    projectId,
    modelParams,
    finalMessages,
  );
}
```

---

### 2. **getChatCompletionStream 流式函数**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L783-L842)

#### **AsyncGenerator 实现（60 行）**

```typescript
// Lines 783-842: 流式生成器（60 行）
async function* getChatCompletionStream(
  projectId: string,
  modelParams: UIModelParams,
  messages: ChatMessageCompiled[],
  signal?: AbortSignal,
): AsyncGenerator<string, void, undefined> {
  const response = await fetch("/api/public/chat-completions", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      projectId,
      ...modelParams,
      messages,
      stream: true,
    }),
    signal,  // 支持中断
  });

  if (!response.ok || !response.body) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  try {
    while (true) {
      const { done, value } = await reader.read();
      
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split("\n");
      buffer = lines.pop() || "";  // 保留不完整的行

      for (const line of lines) {
        if (line.startsWith("data: ")) {
          const data = line.slice(6);
          
          if (data === "[DONE]") {
            return;
          }

          try {
            const parsed = JSON.parse(data);
            const content = parsed.choices[0]?.delta?.content;
            
            if (content) {
              yield content;  // 生成流式数据块
            }
          } catch (e) {
            // 忽略解析错误
          }
        }
      }
    }
  } finally {
    reader.releaseLock();
  }
}
```

#### **SSE 流式协议**

**数据格式**：
```
data: {"choices":[{"delta":{"content":"Hello"}}]}

data: {"choices":[{"delta":{"content":" world"}}]}

data: {"choices":[{"delta":{"content":"!"}}]}

data: [DONE]
```

---

### 3. **getChatCompletionNonStreaming 非流式函数**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L844-L882)

#### **实现逻辑（39 行）**

```typescript
// Lines 844-882: 非流式执行（39 行）
async function getChatCompletionNonStreaming(
  projectId: string,
  modelParams: UIModelParams,
  messages: ChatMessageCompiled[],
): Promise<string> {
  const response = await fetch("/api/public/chat-completions", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      projectId,
      ...modelParams,
      messages,
      stream: false,
    }),
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`API error: ${errorText}`);
  }

  const data = await response.json();
  
  // 提取完整输出
  return data.choices[0]?.message?.content || "";
}
```

**响应格式**：
```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "完整的回答内容...",
        "finish_reason": "stop"
      }
    }
  ],
  "usage": {
    "prompt_tokens": 100,
    "completion_tokens": 200,
    "total_tokens": 300
  }
}
```

---

### 4. **getChatCompletionWithTools 工具调用**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L700-L740)

#### **工具调用流程（41 行）**

```typescript
// Lines 700-740: 工具调用执行（41 行）
async function getChatCompletionWithTools(
  projectId: string,
  modelParams: UIModelParams,
  messages: ChatMessageCompiled[],
  tools: PlaygroundTool[],
): Promise<string> {
  const response = await fetch("/api/public/chat-completions", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      projectId,
      ...modelParams,
      messages,
      tools: tools.map(tool => ({
        type: "function",
        function: {
          name: tool.name,
          description: tool.description,
          parameters: tool.parameters,
        },
      })),
      tool_choice: "auto",  // LLM 自动决定是否调用工具
    }),
  });

  const data = await response.json();
  
  // 提取 Tool Calls
  const toolCalls = data.choices[0]?.message?.tool_calls;
  
  if (toolCalls && toolCalls.length > 0) {
    return JSON.stringify(toolCalls, null, 2);
  }
  
  // 无工具调用时返回普通内容
  return data.choices[0]?.message?.content || "";
}
```

#### **Tool Call 响应格式**

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\": \"San Francisco\"}"
            }
          }
        ]
      }
    }
  ]
}
```

---

### 5. **getChatCompletionWithStructuredOutput 结构化输出**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L742-L781)

#### **Schema 强制输出（40 行）**

```typescript
// Lines 742-781: 结构化输出（40 行）
async function getChatCompletionWithStructuredOutput(
  projectId: string,
  modelParams: UIModelParams,
  messages: ChatMessageCompiled[],
  schema: PlaygroundSchema,
): Promise<string> {
  const response = await fetch("/api/public/chat-completions", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      projectId,
      ...modelParams,
      messages,
      response_format: {
        type: "json_schema",
        json_schema: {
          name: schema.name,
          description: schema.description,
          schema: schema.schema,
          strict: true,  // 严格模式：必须符合 schema
        },
      },
    }),
  });

  const data = await response.json();
  const content = data.choices[0]?.message?.content || "{}";
  
  // 验证 JSON
  try {
    const parsed = JSON.parse(content);
    return JSON.stringify(parsed, null, 2);  // 格式化输出
  } catch (e) {
    throw new Error("Invalid JSON output from model");
  }
}
```

#### **Schema 示例**

**定义**：
```json
{
  "name": "user_profile",
  "description": "Extract user profile from text",
  "schema": {
    "type": "object",
    "properties": {
      "name": { "type": "string" },
      "age": { "type": "integer" },
      "email": { "type": "string", "format": "email" }
    },
    "required": ["name", "email"]
  }
}
```

**输出**：
```json
{
  "name": "John Doe",
  "age": 30,
  "email": "john@example.com"
}
```

---

### 6. **请求中断机制**

#### **AbortController 管理**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L109-L112)

```typescript
// Lines 109-112: AbortController 引用
const abortControllerRef = useRef<AbortController | null>(null);

// 初始化
abortControllerRef.current = new AbortController();
```

#### **stopExecution 方法**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L446-L457)

```typescript
// Lines 446-457: 停止执行（12 行）
const stopExecution = useCallback(() => {
  if (abortControllerRef.current) {
    abortControllerRef.current.abort();  // 中断请求
    abortControllerRef.current = null;
  }
  
  setIsStreaming(false);
  isStreamingRef.current = false;
  
  capture("playground_execution_stopped", {
    manual_stop: true,
  });
}, [capture]);
```

#### **中断传播链**

```
用户点击 Stop Button
  │
  ├─ stopExecution()
  │  └─ abortController.abort()
  │
  ├─ fetch() 接收 AbortSignal
  │  └─ throw AbortError
  │
  ├─ for await 循环中断
  │  └─ 停止 yield 数据块
  │
  └─ setIsStreaming(false)
     └─ UI 更新为停止状态
```

---

### 7. **输出解析与处理**

#### **getOutputJson 方法**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L884-L916)

```typescript
// Lines 884-916: 解析输出 JSON（33 行）
function getOutputJson(output: string | null): any | null {
  if (!output) return null;

  try {
    // 尝试直接解析
    return JSON.parse(output);
  } catch (e) {
    // 尝试提取 Markdown 代码块中的 JSON
    const codeBlockRegex = /```(?:json)?\s*([\s\S]*?)```/;
    const match = output.match(codeBlockRegex);
    
    if (match && match[1]) {
      try {
        return JSON.parse(match[1].trim());
      } catch (e2) {
        return null;
      }
    }
    
    return null;
  }
}
```

#### **Tool Call ID 修复**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L132-L137)

```typescript
// Lines 132-137: 修复空 tool_call_id（6 行）
const toolCallIds = useMemo(() => {
  const ids = new Map<string, string>();
  // 为空 ID 生成唯一 ID（兼容 LangGraph）
  messages.forEach(msg => {
    if (msg.type === "tool-result" && !msg.toolCallId) {
      ids.set(msg.id, uuidv4());
    }
  });
  return ids;
}, [messages]);
```

---

## 执行模式对比

### **流式 vs 非流式**

| 特性 | 流式（Streaming） | 非流式（Non-Streaming） |
|-----|------------------|---------------------|
| **响应延迟** | 低（首字节 < 1s） | 高（完整响应 5-30s） |
| **用户体验** | 实时看到输出 | 等待完整结果 |
| **网络开销** | 低（边生成边传输） | 高（一次性传输） |
| **错误处理** | 可中途停止 | 必须等待完成 |
| **内存占用** | 低（逐块处理） | 高（完整存储） |
| **适用场景** | 长文本生成、对话 | 短文本、结构化输出 |

### **工具调用模式**

| 模式 | 请求参数 | 响应类型 | 用途 |
|-----|---------|---------|------|
| **无工具** | `tools: undefined` | 普通文本 | 标准对话 |
| **Auto 工具** | `tool_choice: "auto"` | 文本或 Tool Calls | LLM 自动决定 |
| **强制工具** | `tool_choice: { "name": "get_weather" }` | Tool Calls | 必须调用指定工具 |
| **结构化输出** | `response_format: { "type": "json_schema" }` | JSON | 数据提取 |

---

## 性能指标

### **执行耗时分解**

| 阶段 | 流式模式 | 非流式模式 |
|-----|---------|-----------|
| **网络往返** | 100-300ms | 100-300ms |
| **首字节延迟** | 500-1000ms | - |
| **完整响应** | 5-30s（渐进式） | 5-30s（一次性） |
| **解析处理** | < 10ms/块 | 10-50ms |

### **吞吐量**

| 指标 | 数值 | 单位 |
|-----|------|------|
| **流式吞吐** | 50-200 | 字符/秒 |
| **并发执行** | 10 | 窗口数 |
| **内存占用** | 5-20 | MB/窗口 |

---

## 错误处理

### **异常类型**

| 错误类型 | 触发条件 | 处理方式 |
|---------|---------|---------|
| **AbortError** | 用户点击 Stop | 静默处理，不显示错误 |
| **NetworkError** | 网络断开 | 显示错误 toast |
| **APIError** | 模型返回错误 | 解析错误信息 + toast |
| **ValidationError** | 输入验证失败 | 显示具体字段错误 |
| **JSONParseError** | 结构化输出解析失败 | 显示原始输出 |

### **错误处理代码（Lines 428-442）**

```typescript
// Lines 428-442: 异常捕获（15 行）
catch (error) {
  if (error instanceof DOMException && error.name === "AbortError") {
    // 用户主动停止，不显示错误
    return;
  }

  const errorMessage = error instanceof Error 
    ? error.message 
    : "Unknown error occurred";
  
  showErrorToast("Execution failed", errorMessage);
  
  capture("playground_execution_error", {
    error_type: error?.constructor?.name,
    error_message: errorMessage,
  });
}
```

---

## 配置参数

### **模型参数（UIModelParams）**

| 参数 | 类型 | 默认值 | 范围 | 说明 |
|-----|------|-------|------|------|
| `provider` | string | - | - | openai/anthropic/azure |
| `model` | string | - | - | gpt-4/claude-3-opus |
| `temperature` | number | 1.0 | 0-2 | 创造性（0=确定性，2=随机） |
| `maxTokens` | number | 2048 | 1-128K | 最大输出 token 数 |
| `topP` | number | 1.0 | 0-1 | Nucleus sampling |
| `stream` | boolean | true | - | 是否启用流式 |

### **超时配置**

| 超时类型 | 默认值 | 环境变量 |
|---------|-------|---------|
| **API 请求超时** | 60s | `PLAYGROUND_API_TIMEOUT` |
| **流式读取超时** | 120s | `PLAYGROUND_STREAM_TIMEOUT` |

---

## 监控指标

### **关键 Metrics**

| 指标名 | 类型 | 标签 | 描述 |
|-------|------|------|------|
| `playground_execution_started` | Counter | `mode`, `provider` | 执行开始次数 |
| `playground_execution_completed` | Counter | `mode`, `provider` | 执行完成次数 |
| `playground_execution_duration` | Histogram | `mode`, `provider` | 执行耗时分布 |
| `playground_execution_error` | Counter | `error_type`, `provider` | 执行失败次数 |
| `playground_execution_stopped` | Counter | `manual_stop` | 手动停止次数 |
| `playground_streaming_throughput` | Histogram | `provider` | 流式吞吐量（字符/秒） |

---

## 使用示例

### **流式执行**

```typescript
// 启用流式模式
const handleStreamingSubmit = async () => {
  setIsStreaming(true);
  setOutput("");

  const stream = getChatCompletionStream(
    projectId,
    modelParams,
    messages,
    abortController.signal
  );

  for await (const chunk of stream) {
    setOutput(prev => prev + chunk);
  }

  setIsStreaming(false);
};
```

### **工具调用多轮对话**

```typescript
// 第 1 轮：LLM 决定调用工具
const toolCallsResponse = await getChatCompletionWithTools(
  projectId,
  modelParams,
  messages,
  [weatherTool]
);

// 添加 Tool Call 消息
addMessage("assistant-tool-call", toolCalls);

// 用户提供 Tool Result
addMessage("tool-result", {
  toolCallId: "call_abc123",
  result: JSON.stringify({ temperature: 72, condition: "sunny" })
});

// 第 2 轮：LLM 基于 Tool Result 生成最终回答
const finalResponse = await getChatCompletionNonStreaming(
  projectId,
  modelParams,
  updatedMessages
);
```

### **结构化输出提取**

```typescript
const extractUserInfo = async (text: string) => {
  const schema = {
    name: "user_info",
    description: "Extract user information",
    schema: {
      type: "object",
      properties: {
        name: { type: "string" },
        email: { type: "string" }
      },
      required: ["name", "email"]
    }
  };

  const jsonOutput = await getChatCompletionWithStructuredOutput(
    projectId,
    modelParams,
    [{ role: "user", content: text }],
    schema
  );

  return JSON.parse(jsonOutput);
};
```

---

## 相关文档

- [Multi-Window Coordination 管理](multi-window-coordination.md) - 多窗口协调机制
- [State Management 管理](state-management.md) - 状态管理与缓存
- [01-single-window-execution-sequence.puml](01-single-window-execution-sequence.puml) - 单窗口执行时序图
- [03-tool-call-multi-turn-sequence.puml](03-tool-call-multi-turn-sequence.puml) - 工具调用多轮时序图
