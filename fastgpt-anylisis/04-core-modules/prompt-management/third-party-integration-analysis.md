# FastGPT Prompt管理 - 第三方平台集成分析

## 一、核心问题

**用户问题**: FastGPT是否支持集成第三方prompt管理平台(如LangFuse等)?

**简短回答**: 
- ❌ **不直接支持** - FastGPT目前没有内置对LangFuse、LangSmith等第三方prompt管理平台的直接集成
- ⚠️ **有扩展可能** - 通过现有的`externalProvider`机制和Webhook架构,理论上可以实现集成
- ✅ **有替代方案** - FastGPT有自己的工作流变量和监控机制

## 二、技术调查结果

### 2.1 搜索关键词结果

通过代码库搜索以下关键词,均**未发现**相关集成:

```bash
# 搜索结果
❌ langfuse / Langfuse / LangFuse     → 0 结果
❌ langsmith / LangSmith              → 0 结果  
❌ prompt hub / prompt registry       → 0 结果
❌ prompt platform / prompt store     → 0 结果
```

### 2.2 现有的外部提供商机制

FastGPT确实有一个`externalProvider`机制,但**主要用于OpenAI账户和工作流变量**,而非prompt管理:

```typescript
// packages/global/core/workflow/runtime/type.d.ts
export type ExternalProviderType = {
  openaiAccount?: OpenaiAccountType;        // 外部OpenAI账户配置
  externalWorkflowVariables?: Record<string, string>;  // 外部工作流变量
};
```

**用途**:
- ✅ 使用外部OpenAI API Key
- ✅ 注入外部工作流变量
- ❌ 不支持外部prompt管理

### 2.3 可观测性/监控支持

FastGPT有基本的监控能力,但**不是完整的prompt管理平台集成**:

```typescript
// packages/service/common/otel/trace/register.ts
traceExporter: new OTLPHttpJsonTraceExporter({
  url: `${SignozBaseURL}/v1/traces`
})
```

**特点**:
- ✅ 支持OpenTelemetry traces导出
- ✅ 可对接Signoz等监控平台
- ❌ 仅限于追踪(tracing),不包括prompt版本管理
- ❌ 不支持prompt A/B测试或动态更新

## 三、FastGPT现有Prompt管理能力总结

### 3.1 代码化管理

| 特性 | FastGPT | LangFuse | LangSmith |
|------|---------|----------|-----------|
| Prompt存储 | 代码(TypeScript) | 数据库 | 数据库 |
| 版本控制 | Git | 平台内置 | 平台内置 |
| 动态更新 | ❌ 需要部署 | ✅ 实时 | ✅ 实时 |
| UI管理界面 | ⚠️ 工作流编辑器 | ✅ 专门Prompt Hub | ✅ Prompt Playground |
| API访问 | ❌ | ✅ | ✅ |
| A/B测试 | ❌ | ✅ | ✅ |
| Prompt市场 | ❌ | ✅ | ✅ |

### 3.2 FastGPT的优势

**代码化的优点**:
- ✅ 类型安全(TypeScript)
- ✅ Git版本控制
- ✅ 代码审查流程
- ✅ 与应用逻辑紧密集成

**工作流集成**:
- ✅ Prompt嵌入在工作流节点中
- ✅ 支持变量替换
- ✅ 可视化编辑器
- ✅ 节点级别的prompt自定义

## 四、集成第三方Prompt管理平台的可能方案

### 方案一: 基于externalProvider扩展(难度: ⭐⭐⭐)

**思路**: 扩展现有的`ExternalProviderType`来支持外部prompt获取

```typescript
// 理论上的扩展
export type ExternalProviderType = {
  openaiAccount?: OpenaiAccountType;
  externalWorkflowVariables?: Record<string, string>;
  
  // 新增prompt provider
  promptProvider?: {
    type: 'langfuse' | 'langsmith';
    apiKey: string;
    baseUrl: string;
    projectId?: string;
  };
};
```

**实现步骤**:

1. **扩展类型定义**
```typescript
// 新增prompt provider配置
interface ExternalPromptProvider {
  type: 'langfuse' | 'langsmith' | 'custom';
  config: {
    apiKey: string;
    baseURL: string;
    projectId?: string;
  };
}
```

2. **创建Prompt获取服务**
```typescript
// packages/service/core/prompt/externalProvider.ts
class ExternalPromptService {
  async getPrompt(promptId: string, version?: string): Promise<string> {
    switch(this.provider.type) {
      case 'langfuse':
        return this.fetchFromLangfuse(promptId, version);
      case 'langsmith':
        return this.fetchFromLangsmith(promptId, version);
      default:
        throw new Error('Unsupported provider');
    }
  }
  
  private async fetchFromLangfuse(promptId: string, version?: string) {
    const response = await fetch(
      `${this.provider.config.baseURL}/api/public/prompts/${promptId}`,
      {
        headers: {
          'Authorization': `Bearer ${this.provider.config.apiKey}`
        }
      }
    );
    return response.json();
  }
}
```

3. **集成到工作流调度**
```typescript
// packages/service/core/workflow/dispatch/ai/chat.ts
async function getChatMessages({
  externalProvider,
  systemPrompt,
  datasetQuotePrompt,
  // ...
}) {
  // 如果配置了外部prompt provider
  if (externalProvider.promptProvider) {
    const externalPromptService = new ExternalPromptService(
      externalProvider.promptProvider
    );
    
    // 从外部平台获取prompt
    if (systemPrompt.startsWith('external:')) {
      const promptId = systemPrompt.replace('external:', '');
      systemPrompt = await externalPromptService.getPrompt(promptId);
    }
  }
  
  // 继续原有逻辑...
}
```

**优点**:
- ✅ 最小侵入性
- ✅ 复用现有架构
- ✅ 支持多种provider

**缺点**:
- ❌ 需要修改核心代码
- ❌ 增加网络调用延迟
- ❌ 需要处理缓存和失败重试

### 方案二: Webhook回调机制(难度: ⭐⭐⭐⭐)

**思路**: 在prompt渲染前通过Webhook调用外部服务

```typescript
// 工作流节点配置
interface PromptWebhookConfig {
  enabled: boolean;
  url: string;
  method: 'GET' | 'POST';
  headers?: Record<string, string>;
  body?: {
    promptId: string;
    version?: string;
    variables?: Record<string, any>;
  };
}
```

**Webhook调用示例**:

```typescript
// Before rendering prompt
if (node.promptWebhookConfig?.enabled) {
  const response = await fetch(node.promptWebhookConfig.url, {
    method: node.promptWebhookConfig.method,
    headers: {
      'Content-Type': 'application/json',
      ...node.promptWebhookConfig.headers
    },
    body: JSON.stringify({
      promptId: node.promptWebhookConfig.body.promptId,
      version: node.promptWebhookConfig.body.version,
      variables: variables
    })
  });
  
  const externalPrompt = await response.json();
  systemPrompt = externalPrompt.content;
}
```

**外部服务示例(LangFuse)**:

```javascript
// 外部服务接收Webhook请求
app.post('/webhook/prompt', async (req, res) => {
  const { promptId, version, variables } = req.body;
  
  // 从LangFuse获取prompt
  const langfuseClient = new Langfuse({
    publicKey: process.env.LANGFUSE_PUBLIC_KEY,
    secretKey: process.env.LANGFUSE_SECRET_KEY
  });
  
  const prompt = await langfuseClient.getPrompt(promptId, version);
  
  // 替换变量
  let content = prompt.prompt;
  Object.keys(variables).forEach(key => {
    content = content.replace(`{{${key}}}`, variables[key]);
  });
  
  res.json({ content });
});
```

**优点**:
- ✅ 灵活性高
- ✅ 支持任意外部服务
- ✅ 可以实现复杂的prompt逻辑

**缺点**:
- ❌ 需要部署额外服务
- ❌ 增加系统复杂度
- ❌ 网络延迟和可靠性问题

### 方案三: MCP Server集成(难度: ⭐⭐⭐⭐⭐)

**思路**: 利用FastGPT已有的MCP(Model Context Protocol)能力

FastGPT已经支持MCP Server,理论上可以开发一个专门的Prompt管理MCP Server:

```typescript
// Prompt管理MCP Server
const server = new Server({
  name: "prompt-manager",
  version: "1.0.0",
}, {
  capabilities: {
    tools: {}
  }
});

// 注册工具
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "get_prompt",
        description: "Get prompt from external platform",
        inputSchema: {
          type: "object",
          properties: {
            platform: { type: "string", enum: ["langfuse", "langsmith"] },
            promptId: { type: "string" },
            version: { type: "string" }
          },
          required: ["platform", "promptId"]
        }
      }
    ]
  };
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_prompt") {
    const { platform, promptId, version } = request.params.arguments;
    
    // 从外部平台获取prompt
    const prompt = await fetchPromptFromPlatform(platform, promptId, version);
    
    return {
      content: [
        {
          type: "text",
          text: prompt
        }
      ]
    };
  }
});
```

**在工作流中使用**:
```
1. 添加MCP工具节点
2. 调用 get_prompt 工具
3. 将返回的prompt传递给AI Chat节点
```

**优点**:
- ✅ 利用现有MCP架构
- ✅ 标准化的协议
- ✅ 可复用性强

**缺点**:
- ❌ 需要开发专门的MCP Server
- ❌ 工作流变得更复杂
- ❌ 性能开销

### 方案四: Prompt代理中间件(难度: ⭐⭐⭐⭐)

**思路**: 在LLM调用层添加Prompt拦截和替换逻辑

```typescript
// packages/service/core/ai/llm/middleware/promptProxy.ts
export class PromptProxyMiddleware {
  async beforeLLMCall(params: {
    messages: ChatMessage[];
    externalProvider?: ExternalProviderType;
  }): Promise<ChatMessage[]> {
    const { messages, externalProvider } = params;
    
    if (!externalProvider?.promptProvider) {
      return messages;
    }
    
    // 遍历消息,替换带有external:前缀的内容
    return Promise.all(messages.map(async (msg) => {
      if (msg.role === 'system' && msg.content.startsWith('external:')) {
        const promptRef = msg.content.replace('external:', '');
        const [platform, promptId, version] = promptRef.split(':');
        
        // 从外部平台获取
        const externalContent = await this.fetchExternalPrompt(
          platform,
          promptId,
          version
        );
        
        return {
          ...msg,
          content: externalContent
        };
      }
      return msg;
    }));
  }
  
  private async fetchExternalPrompt(
    platform: string,
    promptId: string,
    version?: string
  ): Promise<string> {
    // 实现具体的获取逻辑
  }
}
```

**集成到LLM调用链**:
```typescript
// packages/service/core/ai/llm/createLLMResponse.ts
async function createLLMResponse({
  messages,
  externalProvider,
  // ...
}) {
  // 应用prompt代理中间件
  const promptProxy = new PromptProxyMiddleware();
  const processedMessages = await promptProxy.beforeLLMCall({
    messages,
    externalProvider
  });
  
  // 继续原有LLM调用逻辑
  const response = await llmClient.chat({
    messages: processedMessages,
    // ...
  });
  
  return response;
}
```

**优点**:
- ✅ 对工作流层透明
- ✅ 集中管理prompt获取逻辑
- ✅ 易于添加缓存和监控

**缺点**:
- ❌ 需要定义prompt引用规范
- ❌ 调试可能变复杂
- ❌ 需要处理异步加载

## 五、推荐方案对比

| 维度 | 方案一<br/>ExternalProvider扩展 | 方案二<br/>Webhook回调 | 方案三<br/>MCP Server | 方案四<br/>Prompt代理 |
|------|------|------|------|------|
| **实现难度** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **侵入性** | 中 | 低 | 低 | 中 |
| **灵活性** | 高 | 最高 | 高 | 中 |
| **性能影响** | 中等 | 较大 | 较大 | 中等 |
| **维护成本** | 中 | 高 | 高 | 中 |
| **推荐度** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |

### 推荐方案: 方案一(ExternalProvider扩展)

**理由**:
1. ✅ 与现有架构契合度最高
2. ✅ 实现难度适中
3. ✅ 性能影响可控
4. ✅ 便于后续优化(如添加缓存)

## 六、具体实现LangFuse集成示例

### 6.1 安装依赖

```bash
cd packages/service
pnpm add langfuse
```

### 6.2 类型定义

```typescript
// packages/global/core/workflow/runtime/type.d.ts
import type { Langfuse } from 'langfuse';

export type ExternalPromptProviderConfig = {
  type: 'langfuse' | 'langsmith';
  langfuse?: {
    publicKey: string;
    secretKey: string;
    baseUrl?: string;
  };
  langsmith?: {
    apiKey: string;
    baseUrl?: string;
  };
};

export type ExternalProviderType = {
  openaiAccount?: OpenaiAccountType;
  externalWorkflowVariables?: Record<string, string>;
  promptProvider?: ExternalPromptProviderConfig;  // 新增
};
```

### 6.3 创建Prompt服务

```typescript
// packages/service/core/prompt/externalProvider/index.ts
import { Langfuse } from 'langfuse';
import type { ExternalPromptProviderConfig } from '@fastgpt/global/core/workflow/runtime/type';

export class ExternalPromptService {
  private langfuseClient?: Langfuse;
  
  constructor(private config: ExternalPromptProviderConfig) {
    if (config.type === 'langfuse' && config.langfuse) {
      this.langfuseClient = new Langfuse({
        publicKey: config.langfuse.publicKey,
        secretKey: config.langfuse.secretKey,
        baseUrl: config.langfuse.baseUrl
      });
    }
  }
  
  async getPrompt(promptId: string, version?: string): Promise<string> {
    switch(this.config.type) {
      case 'langfuse':
        return this.getFromLangfuse(promptId, version);
      case 'langsmith':
        return this.getFromLangsmith(promptId, version);
      default:
        throw new Error(`Unsupported prompt provider: ${this.config.type}`);
    }
  }
  
  private async getFromLangfuse(
    promptId: string, 
    version?: string
  ): Promise<string> {
    if (!this.langfuseClient) {
      throw new Error('LangFuse client not initialized');
    }
    
    const prompt = await this.langfuseClient.getPrompt(promptId, version);
    return prompt.prompt;
  }
  
  private async getFromLangsmith(
    promptId: string,
    version?: string
  ): Promise<string> {
    // LangSmith implementation
    const response = await fetch(
      `${this.config.langsmith?.baseUrl}/api/v1/prompts/${promptId}`,
      {
        headers: {
          'X-API-Key': this.config.langsmith?.apiKey || ''
        }
      }
    );
    
    const data = await response.json();
    return data.prompt;
  }
}
```

### 6.4 集成到AI Chat节点

```typescript
// packages/service/core/workflow/dispatch/ai/chat.ts
import { ExternalPromptService } from '../../../prompt/externalProvider';

export const dispatchChatCompletion = async (props: ChatProps) => {
  const {
    externalProvider,
    // ...
  } = props;
  
  let systemPrompt = props.systemPrompt;
  
  // 如果配置了外部prompt provider且prompt以external:开头
  if (
    externalProvider.promptProvider && 
    systemPrompt?.startsWith('external:')
  ) {
    try {
      const promptService = new ExternalPromptService(
        externalProvider.promptProvider
      );
      
      // 解析prompt引用: external:promptId:version
      const [, promptId, version] = systemPrompt.split(':');
      
      // 从外部平台获取prompt
      systemPrompt = await promptService.getPrompt(promptId, version);
      
      console.log('[ExternalPrompt] Loaded from:', 
        externalProvider.promptProvider.type, 
        promptId, 
        version || 'latest'
      );
    } catch (error) {
      console.error('[ExternalPrompt] Failed to load:', error);
      // 回退到原始prompt
    }
  }
  
  // 继续原有逻辑...
  const messages = await getChatMessages({
    systemPrompt,  // 使用处理后的prompt
    // ...
  });
  
  // ...
};
```

### 6.5 配置示例

```typescript
// Team配置中添加prompt provider
{
  "_id": "team123",
  "name": "My Team",
  "externalWorkflowVariables": {},
  "promptProvider": {
    "type": "langfuse",
    "langfuse": {
      "publicKey": "pk-lf-xxx",
      "secretKey": "sk-lf-xxx",
      "baseUrl": "https://cloud.langfuse.com"
    }
  }
}
```

### 6.6 使用示例

在工作流的AI Chat节点中,System Prompt字段填写:

```
external:my-system-prompt-v1:latest
```

或使用特定版本:

```
external:my-system-prompt-v1:v2.0
```

### 6.7 添加缓存优化

```typescript
// packages/service/core/prompt/externalProvider/cache.ts
import { LRUCache } from 'lru-cache';

const promptCache = new LRUCache<string, string>({
  max: 500,  // 最多缓存500个prompt
  ttl: 1000 * 60 * 10,  // 10分钟过期
});

export class ExternalPromptService {
  async getPrompt(promptId: string, version?: string): Promise<string> {
    const cacheKey = `${this.config.type}:${promptId}:${version || 'latest'}`;
    
    // 检查缓存
    const cached = promptCache.get(cacheKey);
    if (cached) {
      console.log('[ExternalPrompt] Cache hit:', cacheKey);
      return cached;
    }
    
    // 从外部平台获取
    const prompt = await this._fetchPrompt(promptId, version);
    
    // 存入缓存
    promptCache.set(cacheKey, prompt);
    
    return prompt;
  }
  
  private async _fetchPrompt(
    promptId: string, 
    version?: string
  ): Promise<string> {
    // ... 原有获取逻辑
  }
}
```

## 七、监控和追踪集成

虽然FastGPT不直接支持LangFuse的prompt管理,但可以将LLM调用追踪发送到LangFuse:

```typescript
// packages/service/core/ai/llm/createLLMResponse.ts
import { Langfuse } from 'langfuse';

export async function createLLMResponse({
  messages,
  externalProvider,
  // ...
}) {
  const langfuse = externalProvider.promptProvider?.type === 'langfuse'
    ? new Langfuse(externalProvider.promptProvider.langfuse)
    : null;
  
  // 创建trace
  const trace = langfuse?.trace({
    name: 'fastgpt-chat',
    userId: runningUserInfo.tmbId,
    metadata: {
      appId: runningAppInfo.id,
      appName: runningAppInfo.name
    }
  });
  
  // 创建generation span
  const generation = trace?.generation({
    name: 'llm-call',
    model: modelConstantsData.model,
    input: messages,
    metadata: {
      temperature,
      maxTokens: max_tokens
    }
  });
  
  try {
    // 调用LLM
    const response = await llmClient.chat({
      messages,
      // ...
    });
    
    // 记录输出
    generation?.end({
      output: response.choices[0].message.content,
      usage: {
        inputTokens: response.usage?.prompt_tokens,
        outputTokens: response.usage?.completion_tokens,
        totalTokens: response.usage?.total_tokens
      }
    });
    
    return response;
  } catch (error) {
    generation?.end({
      level: 'ERROR',
      statusMessage: error.message
    });
    throw error;
  } finally {
    await langfuse?.flushAsync();
  }
}
```

## 八、总结与建议

### 8.1 当前状态

**FastGPT的Prompt管理现状**:
- ✅ 有完善的代码化prompt管理
- ✅ 有工作流级别的prompt编辑
- ✅ 支持版本化(通过Git)
- ❌ 无第三方prompt平台集成
- ❌ 无运行时动态prompt更新
- ❌ 无prompt A/B测试

### 8.2 集成建议

**如果你需要第三方prompt管理平台**:

1. **短期方案**: 使用Webhook方式
   - 部署一个中间服务
   - 从LangFuse/LangSmith获取prompt
   - 通过API返回给FastGPT

2. **中期方案**: 实现ExternalProvider扩展
   - Fork FastGPT仓库
   - 按照方案一实现
   - 提交PR贡献给社区

3. **长期方案**: 等待官方支持
   - 在FastGPT GitHub提Issue
   - 说明需求和使用场景
   - 等待官方实现或社区贡献

### 8.3 替代方案

**如果暂时不需要第三方集成**:

1. **使用FastGPT现有能力**:
   - 在工作流中自定义prompt
   - 使用全局变量传递prompt内容
   - 通过Git管理prompt版本

2. **混合方案**:
   - FastGPT管理prompt结构
   - LangFuse追踪LLM调用
   - 手动同步prompt内容

3. **外部管理**:
   - 在LangFuse中管理prompt
   - 手动复制到FastGPT工作流
   - 定期同步更新

### 8.4 技术可行性评估

| 需求 | FastGPT原生支持 | 通过扩展实现难度 | 建议方案 |
|------|----------------|-----------------|---------|
| Prompt版本管理 | ⚠️ Git | ⭐⭐⭐ | Git + 代码化管理 |
| 运行时动态更新 | ❌ | ⭐⭐⭐⭐ | ExternalProvider扩展 |
| Prompt A/B测试 | ❌ | ⭐⭐⭐⭐⭐ | 暂不建议,复杂度太高 |
| LLM调用追踪 | ⚠️ 基础支持 | ⭐⭐ | 使用OpenTelemetry |
| Prompt性能分析 | ❌ | ⭐⭐⭐⭐ | 集成LangFuse tracing |
| 团队协作管理 | ✅ | - | 使用Git工作流 |

### 8.5 最终建议

**对于大多数用户**:
- ✅ 使用FastGPT内置的prompt管理能力
- ✅ 通过Git进行版本控制
- ✅ 在工作流编辑器中自定义prompt
- ✅ 使用OpenTelemetry追踪到外部监控平台

**对于需要高级prompt管理的用户**:
- 🔧 考虑实现ExternalProvider扩展方案
- 🔧 或使用Webhook中间服务方案
- 🔧 向FastGPT社区提出feature request

**对于企业用户**:
- 💼 可以基于FastGPT进行定制开发
- 💼 实现完整的prompt管理集成
- 💼 贡献回社区造福其他用户

## 九、参考资源

### 相关文档
- [FastGPT Workflow 工作流文档](https://doc.fastgpt.in/docs/workflow/)
- [LangFuse Prompt Management](https://langfuse.com/docs/prompts)
- [LangSmith Hub](https://docs.smith.langchain.com/hub)
- [OpenTelemetry Protocol](https://opentelemetry.io/docs/specs/otlp/)

### 代码位置
- `packages/global/core/workflow/runtime/type.d.ts` - ExternalProvider类型定义
- `packages/service/core/workflow/dispatch/ai/chat.ts` - AI Chat节点调度
- `packages/service/support/user/team/utils.ts` - Team配置获取
- `packages/global/core/ai/prompt/` - Prompt模块

### 社区讨论
- 建议在FastGPT GitHub仓库提Issue讨论此功能需求
- 加入FastGPT社区交流群获取更多信息

---

**文档版本**: 1.0  
**最后更新**: 2024年12月  
**基于FastGPT版本**: 4.9.7
