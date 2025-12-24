# State Management 管理

## 功能概述

State Management 是 Playground 模块的**状态持久化系统**，负责管理窗口的消息、模型参数、变量、工具、输出等状态，并通过 LocalStorage 实现跨会话持久化。通过 PlaygroundContext 提供统一的状态管理接口，支持消息编辑、变量替换、占位符填充等复杂操作。核心设计：
- **分层状态**：Messages、ModelParams、Variables、Tools、Output 独立管理
- **自动持久化**：状态变化实时同步到 LocalStorage
- **窗口隔离**：每个窗口独立缓存（window-specific key）
- **智能编译**：运行时变量替换 + 占位符展开

---

## 核心组件

### 1. **PlaygroundProvider 状态容器**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L89-L693)

#### **状态结构（605 行）**

| 状态 | 代码行 | 类型 | 初始值 | 作用 |
|-----|-------|------|-------|------|
| **messages** | 108 | `ChatMessage[]` | `[]` | 对话消息列表 |
| **modelParams** | 122 | `UIModelParams` | `{}` | 模型配置参数 |
| **promptVariables** | 96 | `PromptVariable[]` | `[]` | Prompt 变量及值 |
| **messagePlaceholders** | 97 | `PlaceholderMessageFillIn[]` | `[]` | 消息占位符填充 |
| **tools** | 105 | `PlaygroundTool[]` | `[]` | LLM 工具定义 |
| **structuredOutputSchema** | 106 | `PlaygroundSchema \| null` | `null` | 结构化输出 Schema |
| **output** | 100 | `string \| null` | `null` | 执行输出文本 |
| **outputJson** | 102 | `any \| null` | `null` | 解析后的 JSON 输出 |
| **outputToolCalls** | 101 | `LLMToolCall[] \| null` | `null` | 工具调用列表 |
| **isStreaming** | 103 | `boolean` | `false` | 是否正在执行 |

#### **Provider 结构**

```typescript
// Lines 89-693: PlaygroundProvider 组件（605 行）
export const PlaygroundProvider: React.FC<{ 
  children: ReactNode; 
  windowId: string; 
}> = ({ children, windowId }) => {
  // 1. 状态定义（Lines 93-108）
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [modelParams, setModelParams] = useState<UIModelParams>({});
  const [promptVariables, setPromptVariables] = useState<PromptVariable[]>([]);
  // ... 其他状态

  // 2. 缓存加载（Lines 141-199）
  useEffect(() => {
    const cache = getPlaygroundCache(projectId, windowId);
    if (cache) {
      setMessages(cache.messages);
      setModelParams(cache.modelParams);
      // ... 加载其他缓存
    }
  }, [windowId]);

  // 3. 缓存同步（Lines 516-533）
  useEffect(() => {
    setPlaygroundCache(projectId, windowId, {
      messages,
      modelParams,
      promptVariables,
      // ... 其他状态
    });
  }, [messages, modelParams, promptVariables, ...]);

  // 4. 窗口注册（Lines 548-615）
  useEffect(() => {
    registerWindow(windowId, {
      handleSubmit,
      stopExecution,
      getIsStreaming,
      hasModelConfigured,
    });
  }, [windowId]);

  return (
    <PlaygroundContext.Provider value={{ /* 所有状态和方法 */ }}>
      {children}
    </PlaygroundContext.Provider>
  );
};
```

---

### 2. **PlaygroundCache 缓存机制**

#### **缓存结构**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L95)

```typescript
// Lines 95: PlaygroundCache 类型
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

#### **LocalStorage Key 格式**

```typescript
// Window-specific cache key
const cacheKey = `langfuse:playground:cache:${projectId}:${windowId}`;

// Window IDs array
const windowIdsKey = `langfuse:playground:windowIds:${projectId}`;
```

#### **缓存加载（Lines 141-199）**

```typescript
// Lines 141-199: 从 LocalStorage 加载缓存（59 行）
useEffect(() => {
  const loadCache = () => {
    const cacheKey = `langfuse:playground:cache:${projectId}:${windowId}`;
    const cachedData = localStorage.getItem(cacheKey);
    
    if (!cachedData) {
      setCacheLoaded(true);
      return;
    }

    try {
      const cache: PlaygroundCache = JSON.parse(cachedData);
      
      // 恢复消息
      if (cache.messages) {
        setMessages(cache.messages);
      }
      
      // 恢复模型参数
      if (cache.modelParams) {
        setModelParams(cache.modelParams);
      }
      
      // 恢复变量
      if (cache.promptVariables) {
        setPromptVariables(cache.promptVariables);
      }
      
      // 恢复工具
      if (cache.tools) {
        setTools(cache.tools);
      }
      
      // 恢复结构化输出 Schema
      if (cache.structuredOutputSchema) {
        setStructuredOutputSchema(cache.structuredOutputSchema);
      }
      
      // 恢复占位符
      if (cache.messagePlaceholders) {
        setMessagePlaceholders(cache.messagePlaceholders);
      }
      
      // 恢复输出
      if (cache.output !== undefined) {
        setOutput(cache.output);
      }
      
      setCacheLoaded(true);
    } catch (e) {
      console.error("Failed to load playground cache", e);
      setCacheLoaded(true);
    }
  };

  loadCache();
}, [windowId, projectId]);
```

#### **缓存同步（Lines 516-533）**

```typescript
// Lines 516-533: 状态变化时同步到 LocalStorage（18 行）
useEffect(() => {
  // 等待初始加载完成
  if (!cacheLoaded) return;

  const cache: PlaygroundCache = {
    messages,
    modelParams,
    output,
    promptVariables,
    messagePlaceholders,
    tools,
    structuredOutputSchema,
  };

  const cacheKey = `langfuse:playground:cache:${projectId}:${windowId}`;
  localStorage.setItem(cacheKey, JSON.stringify(cache));
}, [
  messages,
  modelParams,
  output,
  promptVariables,
  messagePlaceholders,
  tools,
  structuredOutputSchema,
  cacheLoaded,
  projectId,
  windowId,
]);
```

---

### 3. **消息管理**

#### **消息类型定义**

```typescript
type ChatMessage = 
  | { type: "system"; role: "system"; content: string; id: string }
  | { type: "user"; role: "user"; content: string; id: string }
  | { type: "assistant"; role: "assistant"; content: string; id: string }
  | { type: "assistant-tool-call"; role: "assistant"; toolCalls: LLMToolCall[]; id: string }
  | { type: "tool-result"; role: "tool"; result: string; toolCallId: string; id: string };

type PlaceholderMessage = {
  type: "placeholder";
  name: string;
  content: string;  // Display text like "[[variable_name]]"
  id: string;
};
```

#### **addMessage 方法（Lines 234-278）**

```typescript
// Lines 234-278: 添加消息（45 行）
const addMessage = useCallback(
  (
    type: ChatMessageType,
    role?: ChatMessageRole,
    content?: string,
  ) => {
    const newMessage = createEmptyMessage({
      type,
      role: role || "user",
      content: content || "",
    });

    setMessages((prev) => [...prev, newMessage]);

    // 更新变量（如果消息中包含新变量）
    const extractedVars = extractVariables([newMessage]);
    if (extractedVars.length > 0) {
      updatePromptVariables(extractedVars);
    }

    capture("playground_message_added", {
      message_type: type,
      message_role: role,
    });
  },
  [capture, updatePromptVariables],
);
```

#### **updateMessage 方法（Lines 280-289）**

```typescript
// Lines 280-289: 更新消息（10 行）
const updateMessage = useCallback(
  (messageId: string, updates: Partial<ChatMessage>) => {
    setMessages((prev) =>
      prev.map((msg) =>
        msg.id === messageId ? { ...msg, ...updates } : msg,
      ),
    );
  },
  [],
);
```

#### **deleteMessage 方法（Lines 300-305）**

```typescript
// Lines 300-305: 删除消息（6 行）
const deleteMessage = useCallback((messageId: string) => {
  setMessages((prev) => prev.filter((msg) => msg.id !== messageId));
}, []);
```

---

### 4. **变量替换系统**

#### **变量提取与更新（Lines 201-230）**

```typescript
// Lines 201-230: 更新 Prompt 变量（30 行）
const updatePromptVariables = useCallback(
  (extractedVariables: string[]) => {
    setPromptVariables((prev) => {
      // 保留已存在变量的值
      const existingVars = new Map(prev.map((v) => [v.name, v.value]));

      // 为新变量添加空值
      const newVars: PromptVariable[] = extractedVariables.map((name) => ({
        name,
        value: existingVars.get(name) || "",
        isUsed: true,
      }));

      // 合并新旧变量（去重）
      const allVars = [...prev, ...newVars];
      const uniqueVars = Array.from(
        new Map(allVars.map((v) => [v.name, v])).values(),
      );

      return uniqueVars;
    });
  },
  [],
);
```

#### **变量更新（Lines 448-455）**

```typescript
// Lines 448-455: 更新变量值（8 行）
const updatePromptVariableValue = useCallback(
  (variable: string, value: string) => {
    setPromptVariables((prev) =>
      prev.map((v) => (v.name === variable ? { ...v, value } : v)),
    );
  },
  [],
);
```

#### **变量编译（getFinalMessages）**

**位置**：[web/src/features/playground/page/context/index.tsx](web/src/features/playground/page/context/index.tsx#L919-L980)

```typescript
// Lines 919-980: 编译最终消息（62 行）
function getFinalMessages(
  messages: (ChatMessage | PlaceholderMessage)[],
  promptVariables: PromptVariable[],
  messagePlaceholders: PlaceholderMessageFillIn[],
): ChatMessageCompiled[] {
  let compiledMessages: ChatMessage[] = [];

  for (const msg of messages) {
    // 处理占位符消息
    if (msg.type === "placeholder") {
      const placeholder = messagePlaceholders.find(
        (p) => p.name === msg.name,
      );
      
      if (placeholder?.value) {
        // 展开占位符为消息数组
        compiledMessages.push(...placeholder.value);
      }
      continue;
    }

    // 普通消息：替换变量
    const compiledContent = replaceVariables(
      msg.content,
      promptVariables,
    );

    compiledMessages.push({
      ...msg,
      content: compiledContent,
    });
  }

  return compiledMessages;
}

// 变量替换函数
function replaceVariables(
  content: string,
  variables: PromptVariable[],
): string {
  let result = content;
  
  for (const variable of variables) {
    const pattern = new RegExp(`{{\\s*${variable.name}\\s*}}`, "g");
    result = result.replace(pattern, variable.value);
  }
  
  return result;
}
```

**示例**：
```typescript
// 输入
messages: [
  { type: "user", content: "Hello {{name}}, today is {{date}}" }
]
variables: [
  { name: "name", value: "Alice" },
  { name: "date", value: "2024-01-01" }
]

// 输出
[
  { type: "user", content: "Hello Alice, today is 2024-01-01" }
]
```

---

### 5. **消息占位符**

#### **占位符结构**

```typescript
type PlaceholderMessageFillIn = {
  name: string;              // 占位符名称，如 "history"
  value: ChatMessage[];      // 要插入的消息数组
  isUsed: boolean;           // 是否被使用
};
```

#### **占位符更新（Lines 461-468）**

```typescript
// Lines 461-468: 更新占位符值（8 行）
const updateMessagePlaceholderValue = useCallback(
  (name: string, value: ChatMessage[]) => {
    setMessagePlaceholders((prev) =>
      prev.map((p) => (p.name === name ? { ...p, value, isUsed: true } : p)),
    );
  },
  [],
);
```

#### **占位符删除（Lines 470-472）**

```typescript
// Lines 470-472: 删除占位符（3 行）
const deleteMessagePlaceholder = useCallback((name: string) => {
  setMessagePlaceholders((prev) => prev.filter((p) => p.name !== name));
}, []);
```

#### **占位符展开示例**

**输入**：
```typescript
messages: [
  { type: "system", content: "You are a helpful assistant" },
  { type: "placeholder", name: "history", content: "[[history]]" },
  { type: "user", content: "What did we discuss?" }
]

messagePlaceholders: [
  {
    name: "history",
    value: [
      { type: "user", content: "Tell me about AI" },
      { type: "assistant", content: "AI stands for..." }
    ]
  }
]
```

**输出（编译后）**：
```typescript
[
  { type: "system", content: "You are a helpful assistant" },
  { type: "user", content: "Tell me about AI" },
  { type: "assistant", content: "AI stands for..." },
  { type: "user", content: "What did we discuss?" }
]
```

---

### 6. **模型参数管理**

#### **UIModelParams 结构**

```typescript
type UIModelParams = {
  provider: string;            // "openai", "anthropic", "azure"
  model: string;               // "gpt-4", "claude-3-opus"
  adapter?: string;            // "openai", "anthropic", "langchain"
  temperature?: number;        // 0-2
  maxTokens?: number;          // Max output tokens
  topP?: number;               // Nucleus sampling (0-1)
  maxCompletionTokens?: number;
  frequencyPenalty?: number;   // -2.0 to 2.0
  presencePenalty?: number;    // -2.0 to 2.0
  stop?: string[];             // Stop sequences
};
```

#### **参数更新（Lines 126-127）**

```typescript
// Lines 126-127: 更新模型参数
const updateModelParamValue = useCallback(
  (key: keyof UIModelParams, value: any) => {
    setModelParams((prev) => ({ ...prev, [key]: value }));
    
    // 通知窗口协调器
    const eventBus = getPlaygroundEventBus();
    eventBus.dispatchEvent(
      new CustomEvent(PLAYGROUND_EVENTS.WINDOW_MODEL_CONFIG_CHANGE)
    );
  },
  [],
);
```

#### **参数启用控制（Lines 127）**

```typescript
// Lines 127: 启用/禁用可选参数
const setModelParamEnabled = useCallback(
  (key: keyof UIModelParams, enabled: boolean) => {
    setModelParams((prev) => {
      const updated = { ...prev };
      if (!enabled) {
        delete updated[key];  // 禁用时删除参数
      }
      return updated;
    });
  },
  [],
);
```

---

### 7. **工具与 Schema 管理**

#### **PlaygroundTool 结构**

```typescript
type PlaygroundTool = LLMToolDefinition & {
  id: string;
  existingLlmTool?: LlmTool;  // 引用已保存的工具
};

type LLMToolDefinition = {
  name: string;
  description: string;
  parameters: LLMJSONSchema;   // JSON Schema for parameters
};
```

#### **工具状态（Lines 105）**

```typescript
// Lines 105: 工具状态
const [tools, setTools] = useState<PlaygroundTool[]>([]);

// 添加工具
const addTool = (tool: PlaygroundTool) => {
  setTools(prev => [...prev, tool]);
};

// 删除工具
const removeTool = (toolId: string) => {
  setTools(prev => prev.filter(t => t.id !== toolId));
};
```

#### **PlaygroundSchema 结构**

```typescript
type PlaygroundSchema = {
  id: string;
  name: string;
  description: string;
  schema: LLMJSONSchema;
  existingLlmSchema?: LlmSchema;
};
```

#### **Schema 状态（Lines 106）**

```typescript
// Lines 106: 结构化输出 Schema
const [structuredOutputSchema, setStructuredOutputSchema] = 
  useState<PlaygroundSchema | null>(null);
```

---

### 8. **输出状态管理**

#### **输出类型**

| 输出状态 | 代码行 | 类型 | 用途 |
|---------|-------|------|------|
| **output** | 100 | `string \| null` | 原始文本输出 |
| **outputJson** | 102 | `any \| null` | 解析后的 JSON |
| **outputToolCalls** | 101 | `LLMToolCall[] \| null` | 工具调用列表 |

#### **输出更新逻辑**

```typescript
// 流式更新（逐块追加）
setOutput(prev => (prev || "") + chunk);

// 非流式更新（一次性设置）
setOutput(response);

// JSON 解析
const parsed = getOutputJson(output);
setOutputJson(parsed);

// Tool Calls 提取
const toolCalls = extractToolCalls(output);
setOutputToolCalls(toolCalls);
```

---

## 缓存策略

### **缓存时机**

| 操作 | 触发时机 | 延迟 |
|-----|---------|------|
| **消息编辑** | `messages` state 变化 | < 10ms（React batching） |
| **参数调整** | `modelParams` state 变化 | < 10ms |
| **变量更新** | `promptVariables` state 变化 | < 10ms |
| **输出完成** | `output` state 变化 | < 10ms |

### **缓存失效**

| 场景 | 处理方式 |
|-----|---------|
| **窗口关闭** | 保留缓存（下次打开恢复） |
| **手动清除** | 调用 `localStorage.removeItem(cacheKey)` |
| **项目切换** | 不同 projectId，缓存隔离 |

---

## 性能优化

### **状态更新优化**

| 优化策略 | 实现方式 | 提升 |
|---------|---------|------|
| **useCallback 缓存** | 所有回调函数包裹 | 避免子组件重渲染 |
| **useState 函数式更新** | `setState(prev => ...)` | 避免闭包陷阱 |
| **useRef 存储** | `isStreamingRef` | 避免状态更新触发 re-render |
| **useMemo 计算缓存** | `toolCallIds` | 减少重复计算 |

### **LocalStorage 优化**

| 策略 | 效果 |
|-----|------|
| **批量更新** | React batching 自动合并 setState |
| **延迟写入** | useEffect 依赖数组触发 |
| **压缩存储** | JSON.stringify（浏览器自动压缩） |

---

## 监控指标

### **状态大小**

| 指标 | 计算方式 | 正常范围 |
|-----|---------|---------|
| **消息数量** | `messages.length` | < 100 |
| **缓存大小** | `localStorage.getItem(key).length` | < 1MB |
| **变量数量** | `promptVariables.length` | < 50 |
| **工具数量** | `tools.length` | < 20 |

### **性能指标**

| 指标 | 测量方式 | 目标值 |
|-----|---------|-------|
| **缓存加载耗时** | `performance.now()` | < 50ms |
| **状态更新耗时** | React DevTools Profiler | < 10ms |
| **LocalStorage 写入耗时** | `performance.now()` | < 5ms |

---

## 使用示例

### **添加消息并更新变量**

```typescript
// 1. 添加包含变量的消息
addMessage("user", "user", "Hello {{name}}, today is {{date}}");

// 2. 自动提取变量（在 addMessage 内部）
// updatePromptVariables(["name", "date"]);

// 3. 用户填写变量值
updatePromptVariableValue("name", "Alice");
updatePromptVariableValue("date", "2024-01-01");

// 4. 执行时自动编译
const finalMessages = getFinalMessages(
  messages,
  promptVariables,
  messagePlaceholders
);
// 结果: [{ type: "user", content: "Hello Alice, today is 2024-01-01" }]
```

### **使用消息占位符**

```typescript
// 1. 添加占位符消息
addMessage("placeholder", undefined, "[[conversation_history]]");

// 2. 填充占位符值
updateMessagePlaceholderValue("conversation_history", [
  { type: "user", content: "What is AI?" },
  { type: "assistant", content: "AI stands for Artificial Intelligence..." }
]);

// 3. 执行时自动展开
const finalMessages = getFinalMessages(messages, [], messagePlaceholders);
// 占位符被展开为实际消息数组
```

### **跨窗口复制状态**

```typescript
// 1. 获取源窗口缓存
const sourceCache = getPlaygroundCache(projectId, sourceWindowId);

// 2. 创建新窗口并复制缓存
const newWindowId = uuidv4();
setPlaygroundCache(projectId, newWindowId, sourceCache);

// 3. 添加到窗口列表
setWindowIds([...windowIds, newWindowId]);
```

---

## 故障排查

### **常见问题**

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| **缓存未加载** | `cacheLoaded` 为 false | 检查 useEffect 加载逻辑 |
| **变量未替换** | 变量名不匹配 | 检查 `{{variable}}` 格式 |
| **占位符未展开** | `isUsed` 为 false | 检查占位符是否填充值 |
| **状态不同步** | 未包含在 useEffect 依赖 | 添加到缓存同步依赖数组 |
| **LocalStorage 溢出** | 缓存过大（> 5MB） | 限制消息数量或清理旧缓存 |

---

## 相关文档

- [Multi-Window Coordination 管理](multi-window-coordination.md) - 多窗口协调机制
- [Execution Streaming 管理](execution-streaming.md) - 执行与流式处理
- [02-multi-window-coordination-sequence.puml](02-multi-window-coordination-sequence.puml) - 多窗口协调时序图
