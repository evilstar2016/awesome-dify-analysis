# Playground 模块

## 概述

Playground 是 Langfuse 的交互式 LLM 测试沙盒，提供多窗口对比环境，用于快速迭代和测试提示词（prompts）、模型参数和工具调用。它是一个功能丰富的前端特性，支持实时聚合对比、流式响应、工具集成和结构化输出。

## 核心概念

### 1. Multi-Window System（多窗口系统）
- **定义**：并排显示多个独立的 Playground 实例用于对比测试
- **隔离性**：每个窗口有独立的状态（消息、模型参数、输出）
- **同步执行**：支持"Run All"功能，并行执行所有窗口
- **窗口限制**：
  - 桌面：最多 10 个窗口
  - 移动：最多 1 个窗口（只显示第一个）
  - 最小宽度：400px

### 2. Window Coordination（窗口协调）
- **注册系统**：每个窗口通过 `useWindowCoordination` hook 注册到全局 registry
- **事件总线**：使用 EventTarget API 进行跨窗口通信（零 React re-render 开销）
- **全局操作**：
  - `executeAllWindows()`: 并行执行所有窗口
  - `stopAllWindows()`: 停止所有正在执行的窗口
  - `getExecutionStatus()`: 获取聚合执行状态
- **句柄接口**：
  ```typescript
  interface PlaygroundHandle {
    handleSubmit: (streaming?: boolean) => Promise<void>;
    stopExecution: () => void;
    getIsStreaming: () => boolean;
    hasModelConfigured: () => boolean;
  }
  ```

### 3. State Isolation（状态隔离）
每个窗口有独立的：
- **Messages**：ChatMessage 数组（System、User、Assistant、Tool Call/Result）
- **Model Parameters**：Provider、Model、Temperature、MaxTokens 等
- **Variables**：Prompt 变量及其值
- **Message Placeholders**：消息占位符填充值
- **Tools**：可用的 LLM 工具定义
- **Structured Output Schema**：结构化输出的 JSON Schema
- **Output**：执行结果（文本、JSON、Tool Calls）

### 4. Playground Cache（状态缓存）
- **存储位置**：LocalStorage（window-specific key）
- **缓存内容**：
  ```typescript
  type PlaygroundCache = {
    messages: (ChatMessage | PlaceholderMessage)[];
    modelParams?: Partial<UIModelParams>;
    output?: string | null;
    promptVariables?: PromptVariable[];
    messagePlaceholders?: PlaceholderMessageFillIn[];
    tools?: PlaygroundTool[];
    structuredOutputSchema?: PlaygroundSchema | null;
  } | null;
  ```
- **持久化时机**：状态变化时自动保存
- **加载时机**：窗口初始化时自动加载

### 5. Streaming Execution（流式执行）
- **支持模式**：
  - **Streaming**：实时流式输出（SSE/Streaming Text Response）
  - **Non-Streaming**：一次性返回完整结果
- **中断机制**：
  - `AbortController` 控制请求取消
  - `stopExecution()` 立即停止流式输出
- **状态同步**：`isStreaming` 状态实时更新 UI

### 6. Tool Integration（工具集成）
- **Tool Definition**：定义 LLM 可用的工具（函数）
- **Tool Call**：LLM 决定调用工具并生成 tool call 消息
- **Tool Result**：用户提供工具执行结果
- **Multi-Turn**：支持多轮工具调用对话
- **ID Mapping**：自动修复空的 tool_call_id（兼容 LangGraph）

### 7. Structured Output（结构化输出）
- **JSON Schema**：定义期望的输出结构
- **强制格式**：LLM 输出必须符合 schema
- **验证**：自动验证输出是否符合 schema
- **用途**：
  - 提取结构化数据
  - 确保输出一致性
  - 简化下游处理

## 数据模型

### 核心类型

#### PlaygroundCache
```typescript
type PlaygroundCache = {
  messages: (ChatMessage | PlaceholderMessage)[];
  modelParams?: Partial<UIModelParams> & Pick<UIModelParams, "provider" | "model">;
  output?: string | null;
  promptVariables?: PromptVariable[];
  messagePlaceholders?: PlaceholderMessageFillIn[];
  tools?: PlaygroundTool[];
  structuredOutputSchema?: PlaygroundSchema | null;
} | null;
```

#### ChatMessage Types
```typescript
type ChatMessage = 
  | { type: "system"; role: "system"; content: string }
  | { type: "user"; role: "user"; content: string }
  | { type: "assistant"; role: "assistant"; content: string }
  | { type: "assistant-tool-call"; role: "assistant"; toolCalls: LLMToolCall[] }
  | { type: "tool-result"; role: "tool"; result: string; toolCallId: string };

type LLMToolCall = {
  id: string;
  name: string;
  arguments: string; // JSON string
};
```

#### UIModelParams
```typescript
type UIModelParams = {
  provider: string;           // "openai", "anthropic", "azure", etc.
  model: string;              // "gpt-4", "claude-3-opus", etc.
  adapter?: string;           // "openai", "anthropic", "langchain"
  temperature?: number;       // 0-2
  maxTokens?: number;         // Max output tokens
  topP?: number;              // Nucleus sampling
  maxCompletionTokens?: number;
  // ... other provider-specific params
};
```

#### PlaygroundTool
```typescript
type PlaygroundTool = LLMToolDefinition & {
  id: string;
  existingLlmTool?: LlmTool;  // Reference to saved tool
};

type LLMToolDefinition = {
  name: string;
  description: string;
  parameters: LLMJSONSchema;   // JSON Schema for parameters
};
```

#### PlaygroundSchema
```typescript
type PlaygroundSchema = {
  id: string;
  name: string;
  description: string;
  schema: LLMJSONSchema;
  existingLlmSchema?: LlmSchema;
};
```

### 存储位置

#### LocalStorage Keys
```typescript
// Window-specific cache
`langfuse:playground:cache:${projectId}:${windowId}`

// Persisted window IDs
`langfuse:playground:windowIds:${projectId}`

// Last execution status
`langfuse:playground:lastExecution:${windowId}`
```

#### Database Tables
- **llm_api_keys**: LLM provider API keys（encrypted）
- **prompts**: Saved prompts（可从 Playground 保存）
- **llm_tools**: Saved LLM tool definitions
- **llm_schemas**: Saved structured output schemas

## 核心功能

### 1. Window Management（窗口管理）

#### 1.1 创建窗口
```typescript
// 添加新窗口（复制最后一个窗口状态）
const addWindow = () => {
  const newWindowId = uuidv4();
  const sourceWindowId = windowIds[windowIds.length - 1];
  
  // 复制源窗口的 cache
  const sourceCache = getPlaygroundCache(sourceWindowId);
  setPlaygroundCache(newWindowId, sourceCache);
  
  // 添加到窗口列表
  setWindowIds([...windowIds, newWindowId]);
};

// 添加窗口并复制特定窗口
const addWindowWithCopy = (sourceWindowId: string) => {
  const newWindowId = uuidv4();
  const sourceCache = getPlaygroundCache(sourceWindowId);
  setPlaygroundCache(newWindowId, sourceCache);
  setWindowIds([...windowIds, newWindowId]);
  return newWindowId;
};
```

#### 1.2 删除窗口
```typescript
const removeWindow = (windowId: string) => {
  // 至少保留一个窗口
  if (windowIds.length <= 1) return;
  
  // 从列表移除
  setWindowIds(windowIds.filter(id => id !== windowId));
  
  // 清除 cache
  localStorage.removeItem(`langfuse:playground:cache:${projectId}:${windowId}`);
};
```

#### 1.3 窗口注册
```typescript
// 每个窗口在 mount 时注册
useEffect(() => {
  const handle: PlaygroundHandle = {
    handleSubmit,
    stopExecution,
    getIsStreaming: () => isStreamingRef.current,
    hasModelConfigured: () => !!modelParams.provider && !!modelParams.model,
  };
  
  registerWindow(windowId, handle);
  
  return () => {
    unregisterWindow(windowId);
  };
}, [windowId, registerWindow, unregisterWindow]);
```

### 2. Message Management（消息管理）

#### 2.1 消息编辑
```typescript
const updateMessage = (messageId: string, updates: Partial<ChatMessage>) => {
  setMessages(messages.map(msg => 
    msg.id === messageId ? { ...msg, ...updates } : msg
  ));
};

const deleteMessage = (messageId: string) => {
  setMessages(messages.filter(msg => msg.id !== messageId));
};

const addMessage = (type: ChatMessageType, role: ChatMessageRole) => {
  const newMessage = createEmptyMessage({ type, role, content: "" });
  setMessages([...messages, newMessage]);
};
```

#### 2.2 变量替换
```typescript
// 提取变量
const extractedVariables = extractVariables(messages);

// 编译消息（替换变量）
const compiledMessages = compileChatMessagesWithIds(messages, promptVariables);

// 变量更新
const updatePromptVariableValue = (variable: string, value: string) => {
  setPromptVariables(prev => 
    prev.map(v => v.name === variable ? { ...v, value } : v)
  );
};
```

#### 2.3 消息占位符
```typescript
// 消息占位符允许在运行时动态插入消息数组
type PlaceholderMessage = {
  type: "placeholder";
  name: string;
  content: string; // Display text like "[[variable_name]]"
};

// 填充占位符
const updateMessagePlaceholderValue = (name: string, value: ChatMessage[]) => {
  setMessagePlaceholders(prev =>
    prev.map(p => p.name === name ? { ...p, value, isUsed: true } : p)
  );
};

// 编译时替换占位符
const compiledMessages = compileChatMessagesWithIds(
  messages,
  promptVariables,
  messagePlaceholders
);
```

### 3. Model Parameter Management（模型参数管理）

#### 3.1 参数配置
```typescript
const { modelParams, setModelParams, updateModelParamValue } = useModelParams(windowId);

// 更新单个参数
updateModelParamValue("temperature", 0.7);
updateModelParamValue("maxTokens", 2000);

// 启用/禁用可选参数
setModelParamEnabled("topP", true);
setModelParamEnabled("maxCompletionTokens", false);
```

#### 3.2 Provider & Model 选择
```typescript
// 获取可用 providers 和 models
const { availableProviders, availableModels, providerModelCombinations } = useModelParams(windowId);

// 切换 provider（自动加载该 provider 的模型列表）
updateModelParamValue("provider", "anthropic");

// 选择 model
updateModelParamValue("model", "claude-3-opus-20240229");
```

#### 3.3 参数验证
```typescript
// 获取最终有效参数（合并默认值、验证范围）
const finalModelParams = getFinalModelParams(modelParams);

// 验证必需参数
if (!finalModelParams.provider || !finalModelParams.model) {
  throw new Error("Provider and model are required");
}
```

### 4. Execution（执行）

#### 4.1 单窗口执行
```typescript
const handleSubmit = async (streaming = true) => {
  setIsStreaming(true);
  setOutput("");
  setOutputToolCalls([]);
  setOutputJson("");
  
  const abortController = new AbortController();
  abortControllerRef.current = abortController;
  
  try {
    // 编译消息（替换变量和占位符）
    const compiledMessages = compileChatMessagesWithIds(
      messages,
      promptVariables,
      messagePlaceholders
    );
    
    // 构建请求体
    const body = {
      projectId,
      messages: compiledMessages,
      modelParams: getFinalModelParams(modelParams),
      tools: tools.map(t => t.definition),
      structuredOutputSchema: structuredOutputSchema?.schema,
      streaming,
    };
    
    // 发送请求
    const response = await fetch("/api/chatCompletion", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
      signal: abortController.signal,
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || "Request failed");
    }
    
    if (streaming) {
      // 流式处理
      const reader = response.body?.getReader();
      const decoder = new TextDecoder();
      
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const chunk = decoder.decode(value);
        setOutput(prev => prev + chunk);
      }
    } else {
      // 非流式处理
      const result = await response.json();
      
      if (result.content) {
        setOutput(result.content);
      }
      if (result.toolCalls) {
        setOutputToolCalls(result.toolCalls);
      }
      if (result.structuredOutput) {
        setOutputJson(JSON.stringify(result.structuredOutput, null, 2));
      }
    }
    
    // 记录成功执行（PostHog analytics）
    capture("playground:execution:success", {
      projectId,
      provider: modelParams.provider,
      model: modelParams.model,
      streaming,
      hasTools: tools.length > 0,
      hasStructuredOutput: !!structuredOutputSchema,
    });
    
  } catch (error) {
    if (error.name === "AbortError") {
      console.log("Execution aborted");
    } else {
      showErrorToast(error.message);
      capture("playground:execution:error", { error: error.message });
    }
  } finally {
    setIsStreaming(false);
    abortControllerRef.current = null;
  }
};
```

#### 4.2 全局执行（Run All）
```typescript
const executeAllWindows = async () => {
  setIsExecutingAll(true);
  
  // 获取所有注册的窗口
  const registeredWindows = Array.from(playgroundWindowRegistry.entries());
  
  // 检查是否有窗口配置了模型
  const windowsWithModels = registeredWindows.filter(([_, handle]) =>
    handle.hasModelConfigured()
  );
  
  if (windowsWithModels.length === 0) {
    showErrorToast("No windows have models configured");
    setIsExecutingAll(false);
    return;
  }
  
  // 并行执行所有窗口
  const executions = windowsWithModels.map(([windowId, handle]) =>
    handle.handleSubmit(true).catch(error => {
      console.error(`Window ${windowId} execution failed:`, error);
    })
  );
  
  await Promise.all(executions);
  
  setIsExecutingAll(false);
  
  // 分发完成事件
  playgroundEventBus.dispatchEvent(
    new CustomEvent(PLAYGROUND_EVENTS.EXECUTE_ALL, { detail: { completed: true } })
  );
};
```

#### 4.3 停止执行
```typescript
const stopExecution = () => {
  if (abortControllerRef.current) {
    abortControllerRef.current.abort();
    abortControllerRef.current = null;
  }
  setIsStreaming(false);
};

const stopAllWindows = () => {
  playgroundWindowRegistry.forEach(handle => {
    if (handle.getIsStreaming()) {
      handle.stopExecution();
    }
  });
  
  setIsExecutingAll(false);
};
```

### 5. Tool Management（工具管理）

#### 5.1 添加工具
```typescript
const addTool = () => {
  const newTool: PlaygroundTool = {
    id: uuidv4(),
    name: "",
    description: "",
    parameters: {
      type: "object",
      properties: {},
      required: [],
    },
  };
  setTools([...tools, newTool]);
};
```

#### 5.2 编辑工具
```typescript
const updateTool = (toolId: string, updates: Partial<PlaygroundTool>) => {
  setTools(tools.map(tool =>
    tool.id === toolId ? { ...tool, ...updates } : tool
  ));
};
```

#### 5.3 处理 Tool Calls
```typescript
// LLM 返回 tool calls
const handleToolCalls = (toolCalls: LLMToolCall[]) => {
  // 添加 assistant tool call 消息
  const assistantMessage: ChatMessageWithId = {
    id: uuidv4(),
    type: ChatMessageType.AssistantToolCall,
    role: ChatMessageRole.Assistant,
    toolCalls,
  };
  setMessages([...messages, assistantMessage]);
  
  // 为每个 tool call 添加空的 tool result 消息
  const toolResultMessages = toolCalls.map(tc => ({
    id: uuidv4(),
    type: ChatMessageType.ToolResult,
    role: ChatMessageRole.Tool,
    toolCallId: tc.id,
    result: "", // User fills this in
    _originalRole: tc.name, // For ID mapping fix
  }));
  setMessages(prev => [...prev, ...toolResultMessages]);
};
```

### 6. Structured Output（结构化输出）

#### 6.1 配置 Schema
```typescript
const setStructuredOutputSchema = (schema: PlaygroundSchema | null) => {
  // 清除工具（结构化输出与工具互斥）
  if (schema) {
    setTools([]);
  }
  
  setStructuredOutputSchema(schema);
};

// Schema 示例
const exampleSchema: PlaygroundSchema = {
  id: uuidv4(),
  name: "UserProfile",
  description: "Extract user profile information",
  schema: {
    type: "object",
    properties: {
      name: { type: "string" },
      age: { type: "number" },
      interests: {
        type: "array",
        items: { type: "string" }
      }
    },
    required: ["name", "age"],
  },
};
```

#### 6.2 处理结构化输出
```typescript
// 发送请求时包含 schema
const response = await fetch("/api/chatCompletion", {
  method: "POST",
  body: JSON.stringify({
    ...body,
    structuredOutputSchema: structuredOutputSchema?.schema,
  }),
});

// 接收结构化输出
const result = await response.json();
if (result.structuredOutput) {
  setOutputJson(JSON.stringify(result.structuredOutput, null, 2));
  
  // 验证是否符合 schema
  const isValid = validateAgainstSchema(result.structuredOutput, schema);
  if (!isValid) {
    showErrorToast("Output does not match schema");
  }
}
```

### 7. Save to Prompt（保存为 Prompt）

```typescript
const saveToPrompt = async () => {
  const promptData = {
    name: `Playground - ${new Date().toISOString()}`,
    project_id: projectId,
    type: "chat",
    prompt: {
      messages: messages.map(msg => ({
        role: msg.role,
        content: msg.content,
      })),
      model_params: modelParams,
      tools: tools.map(t => ({
        name: t.name,
        description: t.description,
        parameters: t.parameters,
      })),
      structured_output_schema: structuredOutputSchema?.schema,
    },
    labels: ["playground"],
  };
  
  const response = await fetch(`/api/public/prompts`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(promptData),
  });
  
  if (response.ok) {
    showSuccessToast("Prompt saved successfully");
  } else {
    showErrorToast("Failed to save prompt");
  }
};
```

## 技术架构

### 前端架构

#### 组件层次
```
PlaygroundPage (页面级状态管理)
  ├─ MultiWindowPlayground (多窗口容器)
  │   ├─ PlaygroundProvider (窗口1 - 状态隔离)
  │   │   └─ PlaygroundWindowContent
  │   │       ├─ Messages (消息编辑器)
  │   │       ├─ ModelParameters (模型参数)
  │   │       ├─ Variables (变量管理)
  │   │       ├─ MessagePlaceholders (占位符)
  │   │       ├─ PlaygroundTools (工具管理)
  │   │       ├─ StructuredOutputSchemaSection
  │   │       └─ GenerationOutput (输出显示)
  │   ├─ PlaygroundProvider (窗口2)
  │   ├─ PlaygroundProvider (窗口3)
  │   └─ ...
  └─ Header Controls
      ├─ Run All Button
      ├─ Add Window Button
      └─ Reset Playground Button
```

#### 状态管理

**Page-Level State**（页面级状态）：
- `windowIds: string[]` - 活动窗口 ID 列表
- `isExecutingAll: boolean` - 全局执行状态

**Window-Level State**（窗口级状态 - per Provider）：
- `messages: ChatMessageWithId[]`
- `modelParams: UIModelParams`
- `promptVariables: PromptVariable[]`
- `messagePlaceholders: PlaceholderMessageFillIn[]`
- `tools: PlaygroundTool[]`
- `structuredOutputSchema: PlaygroundSchema | null`
- `output: string`
- `outputToolCalls: LLMToolCall[]`
- `outputJson: string`
- `isStreaming: boolean`

#### Hooks 架构

```typescript
// 全局协调
useWindowCoordination(): WindowCoordinationReturn
  - registerWindow(windowId, handle)
  - unregisterWindow(windowId)
  - executeAllWindows()
  - stopAllWindows()
  - getExecutionStatus()

// 窗口持久化
usePersistedWindowIds(): {
  windowIds: string[];
  isLoaded: boolean;
  addWindowWithCopy(sourceId?): string;
  removeWindowId(windowId): void;
}

// 状态缓存
usePlaygroundCache(windowId): {
  playgroundCache: PlaygroundCache;
  setPlaygroundCache(cache): void;
}

// 模型参数
useModelParams(windowId): ModelParamsContext

// 快捷键
useCommandEnter(enabled, callback)
  - 监听 Cmd+Enter / Ctrl+Enter
  - 触发 "Run All"
```

#### Event Bus（事件总线）

使用 EventTarget API 实现零 React re-render 的跨窗口通信：

```typescript
const playgroundEventBus = new EventTarget();

// 事件类型
const PLAYGROUND_EVENTS = {
  EXECUTE_ALL: "playground:execute-all",
  STOP_ALL: "playground:stop-all",
  WINDOW_REGISTERED: "playground:window-registered",
  WINDOW_UNREGISTERED: "playground:window-unregistered",
  WINDOW_EXECUTION_STATE_CHANGE: "playground:window-execution-state-change",
  WINDOW_MODEL_CONFIG_CHANGE: "playground:window-model-config-change",
};

// 分发事件
playgroundEventBus.dispatchEvent(
  new CustomEvent(PLAYGROUND_EVENTS.EXECUTE_ALL, { detail: { windowId } })
);

// 监听事件
playgroundEventBus.addEventListener(PLAYGROUND_EVENTS.EXECUTE_ALL, (event) => {
  console.log("Execute all triggered", event.detail);
});
```

### 后端架构

#### API 路由

**路由文件**：`web/src/app/api/chatCompletion/route.ts`

```typescript
// 路由配置
export const dynamic = "force-dynamic";
export const maxDuration = 120; // 最长执行时间120秒

// POST 处理器
export const POST = chatCompletionHandler;
```

**请求/响应格式**：
```
POST /api/chatCompletion
  ├─ Request Body:
  │   ├─ projectId: string
  │   ├─ messages: ChatMessage[]
  │   ├─ modelParams: UIModelParams
  │   ├─ tools?: LLMToolDefinition[]
  │   ├─ structuredOutputSchema?: LLMJSONSchema
  │   └─ streaming: boolean
  └─ Response:
      ├─ Streaming: StreamingTextResponse (SSE)
      └─ Non-Streaming: JSON { content, toolCalls, structuredOutput }
```

#### 处理流程

**chatCompletionHandler**：
```typescript
1. validateChatCompletionBody(body)
   - Zod schema 验证

2. authorizeRequestOrThrow(projectId)
   - 检查用户权限
   - 获取 userId

3. 查询 LLM API Key
   - prisma.llmApiKeys.findFirst({ projectId, provider })
   - 解密 API key

4. fetchLLMCompletion(params)
   - 根据 provider 调用相应的 LLM API
   - 支持 OpenAI、Anthropic、Azure、Gemini 等
   - 传递 messages、modelParams、tools、schema

5. 处理响应
   - Streaming: 返回 StreamingTextResponse
   - Non-Streaming: 返回 JSON
   - Tool Calls: 返回 toolCalls 数组
   - Structured Output: 返回 structuredOutput

6. 记录分析数据
   - PosthogCallbackHandler 记录使用情况
   - 包含 provider、model、tokens、latency 等
```

#### LLM API 调用

**fetchLLMCompletion**

**文件路径**：`packages/shared/src/server/llm/fetchLLMCompletion.ts`

**函数签名**：
```typescript
export async function fetchLLMCompletion(params: {
  llmConnection: LLMApiKey;
  messages: ChatMessage[];
  modelParams: UIModelParams;
  tools?: LLMToolDefinition[];
  structuredOutputSchema?: LLMJSONSchema;
  streaming: boolean;
  callbacks?: BaseCallbackHandler[];
  maxRetries?: number;
  traceSinkParams?: TraceSinkParams;
  shouldUseLangfuseAPIKey?: boolean;
}): Promise<
  | string
  | IterableReadableStream<Uint8Array>
  | Record<string, unknown>
  | ToolCallResponse
>
```

**支持的适配器**：
```typescript
// 根据 modelParams.adapter 选择适配器
- LLMAdapter.OpenAI      → ChatOpenAI
- LLMAdapter.Anthropic   → ChatAnthropic
- LLMAdapter.Azure       → AzureChatOpenAI
- LLMAdapter.Bedrock     → ChatBedrockConverse
- LLMAdapter.VertexAI    → ChatVertexAI
- LLMAdapter.GoogleAIStudio → ChatGoogleGenerativeAI
```

**Provider 适配器示例（OpenAI）**：
```typescript
async function fetchOpenAICompletion(params) {
  const openai = new OpenAI({
    apiKey: params.llmConnection.secretKey,
  });
  
  const requestParams = {
    model: params.modelParams.model,
    messages: params.messages.map(msg => ({
      role: msg.role,
      content: msg.content,
      ...(msg.toolCalls && { tool_calls: msg.toolCalls }),
      ...(msg.toolCallId && { tool_call_id: msg.toolCallId }),
    })),
    temperature: params.modelParams.temperature,
    max_tokens: params.modelParams.maxTokens,
    ...(params.tools && { tools: params.tools }),
    ...(params.structuredOutputSchema && {
      response_format: {
        type: "json_schema",
        json_schema: params.structuredOutputSchema,
      },
    }),
    stream: params.streaming,
  };
  
  if (params.streaming) {
    const stream = await openai.chat.completions.create(requestParams);
    return async function* () {
      for await (const chunk of stream) {
        const content = chunk.choices[0]?.delta?.content || "";
        if (content) yield content;
      }
    }();
  } else {
    const response = await openai.chat.completions.create(requestParams);
    
    return {
      content: response.choices[0].message.content,
      toolCalls: response.choices[0].message.tool_calls,
      structuredOutput: response.choices[0].message.content
        ? JSON.parse(response.choices[0].message.content)
        : null,
    };
  }
}
```

### 数据流

#### 执行请求流
```
User Action (Click Run)
  ↓
PlaygroundContext.handleSubmit()
  ↓
POST /api/chatCompletion
  ↓
chatCompletionHandler
  ↓
authorizeRequestOrThrow (Check permissions)
  ↓
Fetch LLM API Key from DB
  ↓
fetchLLMCompletion (Call LLM)
  ↓
LLM Provider (OpenAI/Anthropic/etc)
  ↓
Stream Response / JSON Response
  ↓
Update UI State (output/outputToolCalls/outputJson)
  ↓
Cache to LocalStorage
  ↓
PostHog Analytics
```

#### 多窗口协调流
```
User Action (Click Run All)
  ↓
useWindowCoordination.executeAllWindows()
  ↓
playgroundEventBus.dispatchEvent(EXECUTE_ALL)
  ↓
For each registered window:
  playgroundWindowRegistry.get(windowId).handleSubmit()
  ↓
Parallel Execution (Promise.all)
  ↓
Each window updates its own state
  ↓
Global state: isExecutingAll = false
```

## UI 组件

### 核心组件

#### PlaygroundPage
- **路径**：`web/src/features/playground/page/index.tsx`
- **职责**：
  - 页面级窗口状态管理
  - Header 控件集成
  - 全局执行控制
- **Props**：无（使用路由参数 `projectId`）

#### MultiWindowPlayground
- **路径**：`web/src/features/playground/page/components/MultiWindowPlayground.tsx`
- **职责**：
  - 渲染多个窗口
  - 响应式布局（水平滚动 + Grid）
  - 窗口操作（复制、删除）
- **Props**：
  ```typescript
  interface MultiWindowPlaygroundProps {
    windowState: MultiWindowState;
    onRemoveWindow: (windowId: string) => void;
    onAddWindow: (sourceWindowId?: string) => void;
  }
  ```

#### PlaygroundProvider
- **路径**：`web/src/features/playground/page/context/index.tsx`
- **职责**：
  - 窗口级状态管理（Context API）
  - 消息、参数、变量、工具管理
  - 执行逻辑
  - 状态缓存
- **Props**：
  ```typescript
  interface PlaygroundProviderProps {
    children: React.ReactNode;
    windowId?: string; // 用于状态隔离
  }
  ```

#### Messages
- **路径**：`web/src/features/playground/page/components/Messages.tsx`
- **职责**：
  - 消息列表渲染
  - 消息编辑（添加、删除、修改）
  - 消息类型切换（System/User/Assistant/Tool）
- **集成**：
  - ChatMessageComponent（通用消息组件）
  - 支持 Markdown 渲染
  - 支持代码高亮

#### ModelParameters
- **路径**：`web/src/components/ModelParameters/index.tsx`
- **职责**：
  - 模型参数配置 UI
  - Provider/Model 选择下拉框
  - 参数滑块（Temperature、MaxTokens、TopP）
  - 可选参数切换
- **复用性**：在 Prompts 模块和 Playground 模块共享

#### Variables
- **路径**：`web/src/features/playground/page/components/Variables.tsx`
- **职责**：
  - 显示提取的变量
  - 变量值输入
  - 变量删除

#### PlaygroundTools
- **路径**：`web/src/features/playground/page/components/PlaygroundTools/`
- **职责**：
  - 工具列表管理
  - 工具创建/编辑对话框
  - JSON Schema 编辑器
  - 从保存的 LlmTool 加载

#### StructuredOutputSchemaSection
- **路径**：`web/src/features/playground/page/components/StructuredOutputSchemaSection.tsx`
- **职责**：
  - Schema 配置
  - Schema 创建/编辑对话框
  - JSON Schema 验证
  - 从保存的 LlmSchema 加载

#### GenerationOutput
- **路径**：`web/src/features/playground/page/components/GenerationOutput.tsx`
- **职责**：
  - 显示 LLM 输出
  - 支持多种输出格式：
    - 文本（Markdown 渲染）
    - JSON（格式化显示）
    - Tool Calls（可视化显示）
  - 复制到剪贴板
  - 流式输出动画

### 交互流程

#### 添加新窗口
```
1. User clicks "Add Window" button
2. PlaygroundPage.addWindow() called
3. Generate new windowId (UUID)
4. Copy cache from last window
5. Add windowId to windowIds array
6. MultiWindowPlayground re-renders with new window
7. New PlaygroundProvider mounts
8. New window registers with coordination system
9. Auto-scroll to new window
```

#### 执行单个窗口
```
1. User clicks "Run" button in window
2. PlaygroundContext.handleSubmit(streaming=true) called
3. Compile messages (replace variables, placeholders)
4. Validate modelParams (provider, model required)
5. POST /api/chatCompletion with body
6. If streaming:
   - Read response stream
   - Update output state incrementally
   - Show loading animation
7. If non-streaming:
   - Wait for full response
   - Update output state once
8. Handle tool calls:
   - Parse toolCalls from response
   - Add assistant-tool-call message
   - Add empty tool-result messages
9. Handle structured output:
   - Parse JSON from response
   - Validate against schema
   - Display formatted JSON
10. Save state to cache
11. Record analytics (PostHog)
```

#### Run All 窗口
```
1. User clicks "Run All" button
2. PlaygroundPage.handleExecuteAll() called
3. useWindowCoordination.executeAllWindows() called
4. Get all registered windows from registry
5. Filter windows with configured models
6. If no windows with models:
   - Show error toast
   - Return early
7. For each window:
   - Call window.handleSubmit(streaming=true)
   - Wrap in Promise for parallel execution
8. Promise.all() waits for all windows
9. Update global isExecutingAll state
10. Dispatch completion event on event bus
```

## 使用场景

### 场景 1：Prompt 快速迭代

**用例**：用户想测试不同的 system prompt 对输出的影响

**流程**：
```
1. 在第一个窗口输入基础 system prompt
   "You are a helpful assistant."

2. 点击"Add Window"创建第二个窗口（自动复制第一个窗口）

3. 修改第二个窗口的 system prompt
   "You are a concise assistant. Keep answers brief."

4. 继续添加第三个窗口
   "You are a creative assistant. Use metaphors."

5. 点击"Run All"并行执行所有窗口

6. 对比三个窗口的输出，选择最佳 prompt

7. 点击最佳窗口的"Save to Prompt"保存
```

**优势**：
- 快速复制和修改
- 并行执行节省时间
- 直观的并排对比

### 场景 2：模型对比

**用例**：对比 GPT-4 和 Claude-3 在同一任务上的表现

**流程**：
```
1. 第一个窗口：
   - Provider: OpenAI
   - Model: gpt-4-turbo
   - Messages: [system prompt, user query]

2. 添加第二个窗口（复制）

3. 修改第二个窗口：
   - Provider: Anthropic
   - Model: claude-3-opus-20240229
   - Messages保持不变

4. Run All执行

5. 对比输出：
   - 质量（准确性、相关性）
   - 速度（latency）
   - 成本（tokens used）
   - 风格（简洁 vs 详细）
```

### 场景 3：参数调优

**用例**：找到最佳的 temperature 设置

**流程**：
```
1. 第一个窗口：temperature = 0 (确定性)
2. 第二个窗口：temperature = 0.5 (平衡)
3. 第三个窗口：temperature = 1.0 (创意)

Run All后对比：
- Temperature 0: 一致、事实性强，但可能过于机械
- Temperature 0.5: 平衡，既准确又自然
- Temperature 1.0: 创意、多样，但可能偶尔不准确

根据需求选择合适的设置
```

### 场景 4：工具调用测试

**用例**：测试 LLM 是否正确调用工具

**流程**：
```
1. 定义工具：
   Tool: get_weather
   Parameters: { location: string, unit: "celsius" | "fahrenheit" }

2. 测试消息：
   User: "What's the weather in San Francisco?"

3. 执行后 LLM 返回 tool call：
   {
     id: "call_123",
     name: "get_weather",
     arguments: '{"location": "San Francisco", "unit": "fahrenheit"}'
   }

4. 填写 tool result：
   Tool Result: "72°F, Sunny"

5. 再次执行，LLM 生成自然语言响应：
   "The weather in San Francisco is 72°F and sunny."
```

### 场景 5：结构化输出提取

**用例**：从自然语言中提取结构化数据

**流程**：
```
1. 配置 Structured Output Schema：
   {
     "type": "object",
     "properties": {
       "product": { "type": "string" },
       "price": { "type": "number" },
       "features": {
         "type": "array",
         "items": { "type": "string" }
       }
     },
     "required": ["product", "price"]
   }

2. 输入 User Message：
   "The iPhone 15 Pro costs $999 and features 
    a titanium design, A17 Pro chip, and USB-C."

3. 执行后获得结构化输出：
   {
     "product": "iPhone 15 Pro",
     "price": 999,
     "features": [
       "titanium design",
       "A17 Pro chip",
       "USB-C"
     ]
   }

4. 验证输出符合 schema

5. 可直接用于下游系统
```

## 性能优化

### 1. State Isolation（状态隔离）

**问题**：多个窗口共享状态会导致不必要的 re-render

**解决方案**：
- 每个窗口有独立的 `PlaygroundProvider`
- 使用 `windowId` 作为 Context key
- LocalStorage 缓存也按 `windowId` 隔离

**效果**：
- 一个窗口的状态变化不影响其他窗口
- 降低 React re-render 开销

### 2. Event Bus（事件总线）

**问题**：跨窗口通信会触发多次 React state 更新

**解决方案**：
- 使用 EventTarget API（原生浏览器 API）
- 零 React re-render 开销
- 事件订阅/分发在 React 外部

**效果**：
- 全局协调无性能损失
- 窗口间通信零延迟

### 3. Streaming Optimization（流式优化）

**问题**：流式输出频繁更新 state 导致卡顿

**解决方案**：
```typescript
// 使用 ref 存储流式状态，减少 re-render
const outputRef = useRef("");
const isStreamingRef = useRef(false);

// 批量更新（debounce）
const debouncedSetOutput = useMemo(
  () => debounce((value: string) => setOutput(value), 50),
  []
);

// 流式更新
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  
  const chunk = decoder.decode(value);
  outputRef.current += chunk;
  
  // 批量更新而非每次都 setState
  debouncedSetOutput(outputRef.current);
}
```

**效果**：
- 减少 50% 的 re-render 次数
- 流式输出更流畅

### 4. Cache Strategy（缓存策略）

**LocalStorage 缓存**：
```typescript
// 自动保存（debounced）
useEffect(() => {
  const saveCache = debounce(() => {
    const cache: PlaygroundCache = {
      messages,
      modelParams,
      promptVariables,
      messagePlaceholders,
      tools,
      structuredOutputSchema,
      output,
    };
    localStorage.setItem(
      `langfuse:playground:cache:${projectId}:${windowId}`,
      JSON.stringify(cache)
    );
  }, 1000);
  
  saveCache();
  
  return () => saveCache.cancel();
}, [messages, modelParams, /* ... other deps */]);
```

**效果**：
- 页面刷新不丢失状态
- 批量写入减少 I/O

### 5. Lazy Loading（延迟加载）

```typescript
// 窗口内容延迟渲染（移动端只渲染第一个）
{windowState.windowIds.map((windowId, index) => {
  const isFirstWindow = index === 0;
  
  return (
    <div
      key={windowId}
      className={isFirstWindow ? "flex-1" : "hidden md:block"}
    >
      <PlaygroundProvider windowId={windowId}>
        <PlaygroundWindowContent {...props} />
      </PlaygroundProvider>
    </div>
  );
})}
```

**效果**：
- 移动端性能优化
- 按需渲染窗口

## 常见问题

### Q1: 为什么使用 EventTarget 而不是 React Context 进行跨窗口通信？

A: 性能和解耦考虑：
- **性能**：EventTarget 不触发 React re-render，零额外开销
- **解耦**：窗口间无需知道彼此存在，通过事件总线协调
- **灵活性**：支持任意数量的窗口，无需修改 Context 结构
- **原生 API**：浏览器原生支持，无需额外依赖

### Q2: 窗口数量限制为什么是 10？

A: 基于性能和用户体验：
- **性能**：10 个窗口同时执行不会导致浏览器卡顿
- **用户体验**：超过 10 个窗口很难同时查看和对比
- **响应式布局**：10 个窗口在标准显示器可以合理显示
- **可配置**：通过 `MULTI_WINDOW_CONFIG.MAX_WINDOWS` 可调整

### Q3: 为什么工具调用和结构化输出互斥？

A: LLM API 限制：
- **OpenAI**：`tools` 和 `response_format.json_schema` 不能同时使用
- **Anthropic**：类似限制
- **设计选择**：避免混淆和错误，UI 强制互斥
- **未来支持**：部分 providers 可能支持同时使用，需分别处理

### Q4: 如何处理不同 Provider 的参数差异？

A: 统一抽象 + 动态映射：
```typescript
// 统一的 UIModelParams
type UIModelParams = {
  provider: string;
  model: string;
  temperature?: number;
  maxTokens?: number;
  topP?: number;
  // ... 通用参数
};

// Provider-specific 映射
const mapToProviderParams = (params: UIModelParams, provider: string) => {
  if (provider === "openai") {
    return {
      model: params.model,
      temperature: params.temperature,
      max_tokens: params.maxTokens,
      top_p: params.topP,
    };
  } else if (provider === "anthropic") {
    return {
      model: params.model,
      temperature: params.temperature,
      max_tokens_to_sample: params.maxTokens,
      top_p: params.topP,
    };
  }
  // ... 其他 providers
};
```

### Q5: 为什么需要 tool_call_id 修复逻辑？

A: 兼容性问题：
- **LangGraph**：返回的 tool result 可能缺少 `tool_call_id`
- **OpenAI 要求**：tool result 必须有 `tool_call_id` 才能关联
- **修复策略**：
  1. 查找最近的 assistant-tool-call 消息
  2. 匹配 tool name（存储在 `_originalRole`）
  3. 复制对应的 tool call id
  4. 填充到 tool result 的 `toolCallId`

### Q6: 如何避免缓存冲突？

A: 命名空间隔离：
```typescript
// Window-specific cache
`langfuse:playground:cache:${projectId}:${windowId}`

// 而不是全局cache
`langfuse:playground:cache:${projectId}` // ❌ 会冲突

// 窗口 IDs 列表（项目级别）
`langfuse:playground:windowIds:${projectId}`
```

### Q7: 流式输出如何停止？

A: AbortController API：
```typescript
const abortController = new AbortController();
abortControllerRef.current = abortController;

// 发送请求时传递 signal
fetch("/api/chatCompletion", {
  signal: abortController.signal,
  // ...
});

// 停止时调用 abort
const stopExecution = () => {
  if (abortControllerRef.current) {
    abortControllerRef.current.abort();
  }
  setIsStreaming(false);
};
```

### Q8: 如何处理 LLM API 错误？

A: 分层错误处理：
```typescript
try {
  // 执行请求
} catch (error) {
  if (error.name === "AbortError") {
    // 用户主动停止，静默处理
    console.log("Execution aborted");
  } else if (error instanceof InvalidRequestError) {
    // 请求参数错误
    showErrorToast(error.message);
  } else if (error instanceof UnauthorizedError) {
    // 认证错误
    showErrorToast("API key is invalid or missing");
  } else if (error.response?.status === 429) {
    // Rate limit
    showErrorToast("Rate limit exceeded. Please try again later.");
  } else {
    // 其他错误
    showErrorToast(error.message || "An error occurred");
  }
  
  // 记录到 PostHog
  capture("playground:execution:error", {
    error: error.message,
    provider: modelParams.provider,
  });
}
```

## 相关模块

- **Prompts 模块**：Playground 可保存到 Prompts，Prompts 可跳转到 Playground
- **Models 模块**：管理可用的 LLM models 和 API keys
- **LLM Tools 模块**：保存和管理可复用的工具定义
- **LLM Schemas 模块**：保存和管理可复用的结构化输出 schemas
- **Projects 模块**：Playground 隔离在项目级别
- **RBAC 模块**：控制 Playground 访问权限（`playground:create` 权限）

## 目录结构

```
web/
├── src/
│   ├── pages/
│   │   └── project/
│   │       └── [projectId]/
│   │           └── playground.tsx       # Next.js 页面路由入口
│   ├── app/
│   │   └── api/
│   │       └── chatCompletion/
│   │           └── route.ts            # API 路由（Next.js App Router）
│   └── features/
│       └── playground/
│           ├── page/                   # 前端 UI
│           │   ├── index.tsx           # PlaygroundPage 主页面组件
│           │   ├── types.ts            # TypeScript 类型定义
│           │   ├── components/         # UI 组件
│           │   │   ├── MultiWindowPlayground.tsx
│           │   │   ├── Messages.tsx
│           │   │   ├── Variables.tsx
│           │   │   ├── MessagePlaceholders.tsx
│           │   │   ├── GenerationOutput.tsx
│           │   │   ├── SaveToPromptButton.tsx
│           │   │   ├── ResetPlaygroundButton.tsx
│           │   │   ├── PlaygroundTools/
│           │   │   │   ├── index.tsx
│           │   │   │   ├── CreateOrEditLLMToolDialog.tsx
│           │   │   │   └── ToolsList.tsx
│           │   │   └── StructuredOutputSchemaSection.tsx
│           │   ├── context/            # React Context
│           │   │   └── index.tsx       # PlaygroundProvider
│           │   ├── hooks/              # Custom Hooks
│           │   │   ├── useWindowCoordination.ts
│           │   │   ├── usePersistedWindowIds.ts
│           │   │   ├── usePlaygroundCache.ts
│           │   │   ├── useModelParams.ts
│           │   │   ├── useCommandEnter.ts
│           │   │   ├── usePlaygroundWindowSize.ts
│           │   │   └── useNamingConflicts.ts
│           │   └── storage/            # LocalStorage 封装
│           │       ├── keys.ts
│           │       └── windowStorage.ts
│           └── server/                 # 后端 API Handler
│               ├── chatCompletionHandler.ts    # 主请求处理器
│               ├── validateChatCompletionBody.ts
│               ├── authorizeRequest.ts
│               └── analytics/
│                   └── posthogCallback.ts      # 分析回调
└── packages/
    └── shared/
        └── src/
            └── server/
                └── llm/
                    └── fetchLLMCompletion.ts   # LLM API 调用核心逻辑
```
