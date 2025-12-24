# Multi-Window Coordination 管理

## 功能概述

Multi-Window Coordination 是 Playground 模块的**核心协调机制**，实现多个独立 Playground 窗口的同步执行与状态管理。通过基于 EventTarget 的事件总线和全局注册表，实现零 React re-render 开销的跨窗口通信。核心设计：
- **窗口隔离**：每个窗口独立状态（消息、模型参数、输出）
- **全局协调**：统一执行控制（Run All / Stop All）
- **事件驱动**：EventTarget API 实现高性能事件总线
- **动态注册**：窗口动态添加/移除，自动维护注册表

---

## 核心组件

### 1. **useWindowCoordination Hook**

**位置**：[web/src/features/playground/page/hooks/useWindowCoordination.ts](web/src/features/playground/page/hooks/useWindowCoordination.ts#L34-L262)

#### **Hook 结构（229 行）**

| 方法/状态 | 代码行 | 功能 | 返回类型 |
|---------|-------|------|---------|
| **registerWindow** | 58-74 | 注册窗口到全局 registry | `(windowId, handle) => void` |
| **unregisterWindow** | 82-99 | 从 registry 移除窗口 | `(windowId) => void` |
| **executeAllWindows** | 107-186 | 并行执行所有窗口 | `() => void` |
| **stopAllWindows** | 194-207 | 停止所有执行中的窗口 | `() => void` |
| **getExecutionStatus** | 215-237 | 获取执行状态摘要 | `() => string \| null` |
| **isExecutingAll** | 35 | 全局执行状态标志 | `boolean` |
| **hasAnyModelConfigured** | 36 | 是否有窗口配置了模型 | `boolean` |

#### **PlaygroundHandle 接口**

```typescript
// Lines 548-615: 窗口注册逻辑
interface PlaygroundHandle {
  handleSubmit: (streaming?: boolean) => Promise<void>;
  stopExecution: () => void;
  getIsStreaming: () => boolean;
  hasModelConfigured: () => boolean;
}
```

---

### 2. **全局注册表与事件总线**

#### **playgroundWindowRegistry（Map 结构）**

**位置**：[web/src/features/playground/page/hooks/useWindowCoordination.ts](web/src/features/playground/page/hooks/useWindowCoordination.ts#L16-L20)

```typescript
// Lines 16-20: 全局 Map 注册表
const playgroundWindowRegistry = new Map<string, PlaygroundHandle>();

export const getPlaygroundWindowRegistry = () => playgroundWindowRegistry;
export const getWindowCount = () => playgroundWindowRegistry.size;
```

**数据结构**：
```typescript
Map<windowId: string, handle: PlaygroundHandle>
// 示例：
{
  "window-uuid-1": { handleSubmit, stopExecution, ... },
  "window-uuid-2": { handleSubmit, stopExecution, ... },
  "window-uuid-3": { handleSubmit, stopExecution, ... }
}
```

#### **playgroundEventBus（EventTarget）**

**位置**：[web/src/features/playground/page/hooks/useWindowCoordination.ts](web/src/features/playground/page/hooks/useWindowCoordination.ts#L12-L14)

```typescript
// Lines 12-14: 全局事件总线
const playgroundEventBus = new EventTarget();

export const getPlaygroundEventBus = () => playgroundEventBus;
```

**支持的事件类型**：

| 事件名 | 触发时机 | 携带数据 | 监听者 |
|-------|---------|---------|-------|
| `WINDOW_REGISTERED` | 窗口注册时 | `{ windowId }` | 监控组件 |
| `WINDOW_UNREGISTERED` | 窗口移除时 | `{ windowId }` | 监控组件 |
| `EXECUTE_ALL` | 点击 Run All | 无 | 所有窗口 |
| `STOP_ALL` | 点击 Stop All | 无 | 所有窗口 |
| `WINDOW_MODEL_CONFIG_CHANGE` | 模型配置变更 | 无 | useWindowCoordination |

---

### 3. **窗口注册机制**

#### **registerWindow 方法（Lines 58-74）**

```typescript
// Lines 58-74: 注册窗口（17 行）
const registerWindow = useCallback(
  (windowId: string, handle: PlaygroundHandle) => {
    playgroundWindowRegistry.set(windowId, handle);

    // 检查模型配置
    checkModelConfiguration();

    // 分发注册事件
    playgroundEventBus.dispatchEvent(
      new CustomEvent(PLAYGROUND_EVENTS.WINDOW_REGISTERED, {
        detail: { windowId },
      }),
    );
  },
  [checkModelConfiguration],
);
```

#### **注册时序图**

```
PlaygroundProvider (mount)
  │
  ├─ 创建 PlaygroundHandle 对象
  │  └─ { handleSubmit, stopExecution, getIsStreaming, hasModelConfigured }
  │
  ├─ 调用 registerWindow(windowId, handle)
  │  │
  │  ├─ playgroundWindowRegistry.set(windowId, handle)
  │  ├─ checkModelConfiguration()  // 检查是否有模型配置
  │  └─ dispatchEvent(WINDOW_REGISTERED)
  │
  └─ useEffect cleanup
     └─ unregisterWindow(windowId)
```

---

### 4. **窗口移除机制**

#### **unregisterWindow 方法（Lines 82-99）**

```typescript
// Lines 82-99: 移除窗口（18 行）
const unregisterWindow = useCallback(
  (windowId: string) => {
    const wasRegistered = playgroundWindowRegistry.has(windowId);
    playgroundWindowRegistry.delete(windowId);

    if (wasRegistered) {
      // 重新检查模型配置
      checkModelConfiguration();

      // 分发注销事件
      playgroundEventBus.dispatchEvent(
        new CustomEvent(PLAYGROUND_EVENTS.WINDOW_UNREGISTERED, {
          detail: { windowId },
        }),
      );
    }
  },
  [checkModelConfiguration],
);
```

#### **生命周期管理**

```typescript
// Lines 548-615: 窗口生命周期
useEffect(() => {
  const playgroundHandle: PlaygroundHandle = {
    handleSubmit,
    stopExecution,
    getIsStreaming: () => isStreamingRef.current,
    hasModelConfigured: () => !!modelParams.provider && !!modelParams.model,
  };

  // 注册
  registerWindow(windowId, playgroundHandle);

  // 清理
  return () => {
    unregisterWindow(windowId);
  };
}, [windowId, registerWindow, unregisterWindow]);
```

---

### 5. **全局执行控制**

#### **executeAllWindows 方法（Lines 107-186）**

**执行流程（80 行）**：

| 步骤 | 代码行 | 逻辑 | 耗时 |
|-----|-------|------|------|
| **1. 前置检查** | 108-120 | 检查窗口数量、是否已执行、是否有模型配置 | < 1ms |
| **2. 分发事件** | 122-125 | 分发 EXECUTE_ALL 事件到所有窗口 | < 1ms |
| **3. 延迟检查** | 127-135 | 500ms 后检查是否有窗口开始执行 | 500ms |
| **4. 状态更新** | 137-165 | 设置全局执行状态 + 30s 超时保护 | - |
| **5. 监控完成** | 167-183 | 每 500ms 轮询检查是否所有窗口完成 | 持续监控 |

**关键逻辑**：

**前置验证（Lines 108-120）**：
```typescript
// 检查是否为空
if (registeredWindows.length === 0) return;

// 检查是否已在执行
const alreadyExecuting = registeredWindows.some(handle => 
  handle.getIsStreaming()
);
if (alreadyExecuting) return;

// 检查是否有模型配置
const hasModel = registeredWindows.some(handle => 
  handle.hasModelConfigured()
);
if (!hasModel) return;  // UI 已显示警告，无需 toast
```

**执行监控（Lines 167-183）**：
```typescript
// Lines 167-183: 监控执行完成（17 行）
const checkExecutionCompletion = () => {
  const stillExecuting = Array.from(
    playgroundWindowRegistry.values(),
  ).some((handle) => handle.getIsStreaming());

  if (!stillExecuting) {
    setIsExecutingAll(false);
    if (executionTimeoutRef.current) {
      clearTimeout(executionTimeoutRef.current);
      executionTimeoutRef.current = null;
    }
  } else {
    // 继续检查
    setTimeout(checkExecutionCompletion, 500);
  }
};

setTimeout(checkExecutionCompletion, 1000);
```

---

### 6. **停止所有窗口**

#### **stopAllWindows 方法（Lines 194-207）**

```typescript
// Lines 194-207: 停止所有窗口（14 行）
const stopAllWindows = useCallback(() => {
  setIsExecutingAll(false);

  // 清除超时
  if (executionTimeoutRef.current) {
    clearTimeout(executionTimeoutRef.current);
    executionTimeoutRef.current = null;
  }

  // 分发停止事件
  playgroundEventBus.dispatchEvent(
    new CustomEvent(PLAYGROUND_EVENTS.STOP_ALL),
  );
}, []);
```

**事件处理（在 PlaygroundProvider 中）**：
```typescript
// Lines 626-639: 监听 STOP_ALL 事件
useEffect(() => {
  const handleStopAll = () => {
    stopExecution();  // 调用窗口的 stopExecution
  };

  const eventBus = getPlaygroundEventBus();
  eventBus.addEventListener(
    PLAYGROUND_EVENTS.STOP_ALL,
    handleStopAll,
  );

  return () => {
    eventBus.removeEventListener(PLAYGROUND_EVENTS.STOP_ALL, handleStopAll);
  };
}, [stopExecution]);
```

---

### 7. **执行状态汇总**

#### **getExecutionStatus 方法（Lines 215-237）**

```typescript
// Lines 215-237: 获取执行状态（23 行）
const getExecutionStatus = useCallback((): string | null => {
  const registeredWindows = Array.from(playgroundWindowRegistry.values());

  if (registeredWindows.length === 0) {
    return null;
  }

  const executingCount = registeredWindows.filter((handle) =>
    handle.getIsStreaming(),
  ).length;
  const totalCount = registeredWindows.length;

  if (executingCount === 0) {
    return null;
  }

  if (executingCount === totalCount) {
    return `Executing all ${totalCount} windows`;
  }

  return `Executing ${executingCount} of ${totalCount} windows`;
}, []);
```

**状态示例**：

| 场景 | 返回值 |
|-----|-------|
| 无窗口注册 | `null` |
| 无窗口执行 | `null` |
| 全部执行（3/3） | `"Executing all 3 windows"` |
| 部分执行（2/3） | `"Executing 2 of 3 windows"` |

---

### 8. **PlaygroundPage 主页面**

**位置**：[web/src/features/playground/page/index.tsx](web/src/features/playground/page/index.tsx#L34-L104)

#### **页面结构（71 行）**

```typescript
// Lines 34-104: PlaygroundPage 组件
const PlaygroundPage: React.FC = () => {
  const {
    registerWindow,
    unregisterWindow,
    executeAllWindows,
    stopAllWindows,
    getExecutionStatus,
    isExecutingAll,
    hasAnyModelConfigured,
  } = useWindowCoordination();

  // 窗口管理状态
  const [windowIds, setWindowIds] = useState<string[]>([]);
  const [executionStatus, setExecutionStatus] = useState<string | null>(null);

  // 添加窗口
  const addWindow = () => {
    const newId = uuidv4();
    setWindowIds([...windowIds, newId]);
  };

  // 移除窗口
  const removeWindow = (id: string) => {
    if (windowIds.length <= 1) return;
    setWindowIds(windowIds.filter(wId => wId !== id));
  };

  // 执行所有窗口
  const handleExecuteAll = () => {
    executeAllWindows();
    // 更新状态显示
    const intervalId = setInterval(() => {
      setExecutionStatus(getExecutionStatus());
    }, 500);
  };

  return (
    <MultiWindowPlayground
      windowIds={windowIds}
      onAddWindow={addWindow}
      onRemoveWindow={removeWindow}
      onExecuteAll={handleExecuteAll}
      onStopAll={stopAllWindows}
      isExecutingAll={isExecutingAll}
    />
  );
};
```

---

### 9. **MultiWindowPlayground 容器**

**位置**：[web/src/features/playground/page/components/MultiWindowPlayground.tsx](web/src/features/playground/page/components/MultiWindowPlayground.tsx#L45-L112)

#### **布局结构（68 行）**

```typescript
// Lines 45-112: MultiWindowPlayground 组件
const MultiWindowPlayground: React.FC<MultiWindowPlaygroundProps> = ({
  windowIds,
  onAddWindow,
  onRemoveWindow,
  onExecuteAll,
  onStopAll,
  isExecutingAll,
}) => {
  // 响应式布局
  const isMobile = useMediaQuery("(max-width: 768px)");
  const maxWindows = isMobile ? 1 : 10;
  const minWindowWidth = 400;

  return (
    <div className="playground-container">
      {/* 全局工具栏 */}
      <div className="toolbar">
        <Button onClick={onExecuteAll} disabled={isExecutingAll}>
          {isExecutingAll ? "Executing..." : "Run All"}
        </Button>
        <Button onClick={onStopAll}>Stop All</Button>
        <Button onClick={onAddWindow} disabled={windowIds.length >= maxWindows}>
          + Add Window
        </Button>
      </div>

      {/* 窗口网格 */}
      <div className="windows-grid">
        {windowIds.map((windowId, index) => (
          <PlaygroundWindowContent
            key={windowId}
            windowId={windowId}
            onRemove={() => onRemoveWindow(windowId)}
            canRemove={windowIds.length > 1}
          />
        ))}
      </div>
    </div>
  );
};
```

---

## 性能优化

### **零 React Re-render 事件通信**

| 传统方案 | EventTarget 方案 | 优势 |
|---------|-----------------|------|
| Context + useState | EventTarget API | 避免跨组件 re-render |
| 每次事件触发全局 re-render | 只有监听者响应 | 减少 90%+ 渲染开销 |
| 组件深度耦合 | 松耦合事件驱动 | 易于扩展和维护 |

**性能对比（10 窗口场景）**：

| 操作 | Context 方案 | EventTarget 方案 | 提升 |
|-----|-------------|-----------------|------|
| **注册窗口** | 10 次 re-render | 0 次 re-render | ∞ |
| **执行单个窗口** | 10 次 re-render | 1 次 re-render（目标窗口） | 10x |
| **Run All** | 10 次 re-render | 10 次 re-render（并行） | 1x |

---

### **延迟状态检查策略**

**问题**：React 状态更新是异步的，立即检查 `isStreaming` 可能为 `false`

**解决方案**：
```typescript
// Lines 127-135: 延迟 500ms 检查
setTimeout(() => {
  const anyExecuting = Array.from(playgroundWindowRegistry.values())
    .some(handle => handle.getIsStreaming());

  if (!anyExecuting) {
    showErrorToast("No content to execute");
  }
}, 500);
```

---

## 配置参数

### **窗口限制**

| 设备类型 | 最大窗口数 | 最小宽度 | 原因 |
|---------|-----------|---------|------|
| **桌面** | 10 | 400px | 性能限制 + UI 可用性 |
| **移动** | 1 | 100% | 屏幕空间限制 |

### **超时配置**

| 超时类型 | 时长 | 代码行 | 用途 |
|---------|------|-------|------|
| **执行超时** | 30s | 158 | 防止执行状态卡死 |
| **状态检查间隔** | 500ms | 127, 177 | 实时状态监控 |
| **完成检查延迟** | 1s | 183 | 允许窗口启动时间 |

---

## 监控指标

### **关键事件**

| 事件名 | 触发频率 | 监控目标 |
|-------|---------|---------|
| `WINDOW_REGISTERED` | 窗口创建时 | 注册数量、成功率 |
| `WINDOW_UNREGISTERED` | 窗口销毁时 | 注销数量、泄漏检测 |
| `EXECUTE_ALL` | 用户点击 Run All | 执行次数、并发度 |
| `STOP_ALL` | 用户点击 Stop All | 中断次数、原因 |

### **性能 Metrics**

| 指标 | 计算方式 | 正常范围 |
|-----|---------|---------|
| **注册耗时** | registerWindow 执行时间 | < 1ms |
| **执行启动耗时** | executeAllWindows → 首个窗口开始 | < 500ms |
| **状态同步延迟** | 实际完成 → isExecutingAll 更新 | < 1s |
| **内存占用** | windowRegistry.size × 平均窗口大小 | < 50MB（10 窗口） |

---

## 使用示例

### **添加窗口并执行**

```typescript
// 1. 添加新窗口
const newWindowId = uuidv4();
setWindowIds([...windowIds, newWindowId]);

// 2. 窗口自动注册（在 useEffect 中）
// registerWindow(newWindowId, playgroundHandle);

// 3. 执行所有窗口
executeAllWindows();

// 4. 监控执行状态
const status = getExecutionStatus();
console.log(status);  // "Executing 3 of 3 windows"
```

### **动态监听事件**

```typescript
const eventBus = getPlaygroundEventBus();

// 监听窗口注册
eventBus.addEventListener(
  PLAYGROUND_EVENTS.WINDOW_REGISTERED,
  (event: CustomEvent) => {
    console.log("Window registered:", event.detail.windowId);
  }
);

// 监听执行开始
eventBus.addEventListener(
  PLAYGROUND_EVENTS.EXECUTE_ALL,
  () => {
    console.log("All windows executing");
  }
);
```

---

## 故障排查

### **常见问题**

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| **窗口无法执行** | 未配置模型 | 检查 `hasModelConfigured()` 返回值 |
| **执行状态卡死** | 超时未清除 | 检查 `executionTimeoutRef` 是否正常清理 |
| **窗口泄漏** | 未调用 unregisterWindow | 检查 useEffect cleanup 函数 |
| **事件未触发** | EventTarget 监听器未注册 | 检查 addEventListener 调用时机 |

---

## 相关文档

- [Execution Streaming 管理](execution-streaming.md) - 流式执行机制
- [State Management 管理](state-management.md) - 状态管理与缓存
- [01-single-window-execution-sequence.puml](01-single-window-execution-sequence.puml) - 单窗口执行时序图
- [02-multi-window-coordination-sequence.puml](02-multi-window-coordination-sequence.puml) - 多窗口协调时序图
