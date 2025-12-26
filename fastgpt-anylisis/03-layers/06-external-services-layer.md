# 外部服务集成层 (External Services Layer)

## 概述
外部服务集成层负责 FastGPT 与第三方服务的集成，包括 AI 模型服务、插件运行时、文档处理、语音服务、搜索引擎和外部平台集成（钉钉、飞书、企微等）。

## 核心模块

### 1. AI 模型服务

#### OpenAI
**配置**
```typescript
{
  baseURL: 'https://api.openai.com/v1',
  apiKey: process.env.OPENAI_API_KEY,
  models: ['gpt-4', 'gpt-3.5-turbo', 'text-embedding-ada-002']
}
```

**接口封装**
- Chat Completions (对话)
- Embeddings (向量化)
- Audio Transcriptions (语音转文字)
- Audio Speech (文字转语音)

#### Azure OpenAI
**配置**
```typescript
{
  baseURL: 'https://{resource}.openai.azure.com',
  apiKey: process.env.AZURE_OPENAI_API_KEY,
  apiVersion: '2024-02-15-preview',
  deployment: 'gpt-4'
}
```

#### One API（聚合服务）
**特性**
- 统一多个模型提供商（OpenAI、Azure、Claude、文心一言等）
- 统一鉴权与计费
- 负载均衡与故障转移

**配置**
```typescript
{
  baseURL: process.env.ONE_API_URL,
  apiKey: process.env.ONE_API_KEY
}
```

### 2. FastGPT Plugin 运行时

#### 架构
- 独立服务（Docker 容器）
- 支持多语言（Node.js、Python、Go）
- 沙盒隔离执行

#### 插件包结构
```
plugin.zip
├── manifest.json       # 插件清单
├── index.js           # 入口文件
├── package.json       # 依赖声明
└── README.md
```

**manifest.json**
```json
{
  "id": "plugin-xxx",
  "name": "插件名称",
  "version": "1.0.0",
  "author": "作者",
  "description": "插件描述",
  "entry": "index.js",
  "tools": [{
    "name": "toolName",
    "description": "工具描述",
    "parameters": {...}
  }]
}
```

#### 插件调用流程
1. 工作流执行到插件节点
2. 加载插件代码
3. 在沙盒环境执行
4. 返回执行结果
5. 记录调用日志与费用

### 3. 文档处理服务

#### doc2x
**功能**
- PDF 解析（文字 + 图片 + 表格）
- DOCX 解析
- 保留格式与结构

**API 调用**
```typescript
const result = await axios.post('https://api.doc2x.com/parse', {
  file: fileBuffer,
  type: 'pdf',
  options: {
    extractImages: true,
    extractTables: true
  }
});
```

#### PDF 处理插件
**pdf-marker**
- 基于 PyMuPDF
- 高精度文本提取
- 支持 OCR

**pdf-mineru**
- AI 驱动的 PDF 解析
- 识别复杂布局
- 提取公式与图表

**pdf-mistral**
- 基于 Mistral AI
- 理解 PDF 内容语义
- 智能分块

### 4. 语音服务

#### STT (Speech-to-Text) - SenseVoice
**功能**
- 多语言识别（中英日韩等）
- 高准确率
- 实时转写

**API 调用**
```typescript
const text = await axios.post('http://sensevoice:8000/transcribe', {
  audio: audioBase64,
  language: 'zh'
});
```

#### TTS (Text-to-Speech) - CosyVoice
**功能**
- 自然语音合成
- 多音色支持
- 情感表达

**API 调用**
```typescript
const audio = await axios.post('http://cosyvoice:8000/synthesize', {
  text: '你好，欢迎使用 FastGPT',
  speaker: 'zh-CN-XiaoxiaoNeural'
});
```

### 5. 搜索引擎 - Searxng

#### 功能
- 元搜索引擎（聚合多个搜索源）
- 隐私保护
- 无广告

#### 配置
```yaml
server:
  port: 8080
search:
  safe_search: 1
  autocomplete: 'google'
engines:
  - name: google
    weight: 1
  - name: bing
    weight: 1
```

#### 调用示例
```typescript
const results = await axios.get('http://searxng:8080/search', {
  params: {
    q: '搜索关键词',
    format: 'json',
    lang: 'zh-CN'
  }
});
```

### 6. 外部平台集成

#### 飞书（Feishu/Lark）
**Webhook 集成**
```typescript
// 接收飞书回调
app.post('/api/support/outLink/feishu/:token', async (req, res) => {
  const { challenge, event } = req.body;
  
  // 验证挑战
  if (challenge) {
    return res.json({ challenge });
  }
  
  // 处理消息事件
  if (event.type === 'message') {
    const message = event.message;
    // 调用 FastGPT 对话接口
    const reply = await chatCompletion({...});
    // 回复消息
    await sendFeishuMessage(reply);
  }
});
```

**发送消息**
```typescript
await axios.post('https://open.feishu.cn/open-apis/bot/v2/hook/' + webhookUrl, {
  msg_type: 'text',
  content: {
    text: '回复内容'
  }
});
```

#### 钉钉（DingTalk）
**Webhook 集成**
```typescript
app.post('/api/support/outLink/dingtalk/:token', async (req, res) => {
  const { text, msgtype } = req.body;
  
  // 处理文本消息
  if (msgtype === 'text') {
    const reply = await chatCompletion({
      messages: [{ role: 'user', content: text.content }]
    });
    
    await sendDingTalkMessage({
      msgtype: 'text',
      text: { content: reply }
    });
  }
});
```

#### 企业微信（WeCom）
**应用配置**
```typescript
{
  corpId: process.env.WECOM_CORP_ID,
  agentId: process.env.WECOM_AGENT_ID,
  secret: process.env.WECOM_SECRET
}
```

**消息处理**
```typescript
app.post('/api/support/outLink/wecom/:token', async (req, res) => {
  const { ToUserName, FromUserName, Content } = req.body;
  
  const reply = await chatCompletion({...});
  
  await sendWeComMessage({
    touser: FromUserName,
    msgtype: 'text',
    text: { content: reply }
  });
});
```

#### 微信公众号（Official Account）
**配置**
```typescript
{
  appId: process.env.WECHAT_APPID,
  appSecret: process.env.WECHAT_SECRET,
  token: process.env.WECHAT_TOKEN,
  encodingAESKey: process.env.WECHAT_AES_KEY
}
```

**消息处理**
```typescript
app.post('/api/support/outLink/offiaccount/:token', async (req, res) => {
  const xml = req.body;
  
  // 解析 XML 消息
  const message = await parseXML(xml);
  
  // 处理文本消息
  if (message.MsgType === 'text') {
    const reply = await chatCompletion({...});
    
    // 返回 XML 格式回复
    res.type('xml');
    res.send(buildXMLReply({
      ToUserName: message.FromUserName,
      FromUserName: message.ToUserName,
      Content: reply
    }));
  }
});
```

### 7. MCP (Model Context Protocol)

#### MCP 客户端
**功能**
- 调用外部 MCP 服务端的工具
- 动态发现工具列表
- 上下文共享

**调用流程**
```typescript
// 1. 连接 MCP 服务端
const mcpClient = new MCPClient({
  serverUrl: 'http://mcp-server:3000'
});

// 2. 获取工具列表
const tools = await mcpClient.getTools();

// 3. 调用工具
const result = await mcpClient.runTool({
  name: 'search',
  parameters: { query: '搜索内容' }
});
```

#### MCP 服务端
**功能**
- 暴露 FastGPT 能力为工具
- 支持第三方调用
- 工具注册与发现

**工具定义**
```typescript
{
  name: 'knowledge_base_search',
  description: '知识库检索',
  parameters: {
    type: 'object',
    properties: {
      datasetId: { type: 'string' },
      query: { type: 'string' }
    }
  }
}
```

### 8. AI Proxy

#### 功能
- 聚合国内模型服务（文心一言、通义千问、智谱 AI 等）
- 统一接口格式
- 计费与限流

#### 配置
```typescript
{
  baseURL: process.env.AIPROXY_URL,
  apiKey: process.env.AIPROXY_KEY,
  models: ['ernie-bot-4', 'qwen-max', 'glm-4']
}
```

## 集成最佳实践

### 1. 错误处理
```typescript
try {
  const result = await callExternalService();
} catch (error) {
  // 记录错误日志
  logger.error('External service error:', error);
  
  // 降级处理
  if (error.code === 'TIMEOUT') {
    return fallbackResult;
  }
  
  // 重试机制
  if (error.code === 'RATE_LIMIT') {
    await sleep(1000);
    return retry();
  }
  
  throw error;
}
```

### 2. 超时控制
```typescript
const result = await Promise.race([
  callExternalService(),
  new Promise((_, reject) => 
    setTimeout(() => reject(new Error('Timeout')), 30000)
  )
]);
```

### 3. 缓存策略
```typescript
// 缓存模型列表
const cachedModels = await redis.get('models');
if (cachedModels) {
  return JSON.parse(cachedModels);
}

const models = await fetchModels();
await redis.setex('models', 3600, JSON.stringify(models));
return models;
```

### 4. 监控与日志
```typescript
// 记录调用日志
await addLog({
  type: 'external_service',
  service: 'openai',
  method: 'chat_completion',
  duration: endTime - startTime,
  tokens: result.usage.total_tokens,
  cost: calculateCost(result.usage)
});
```

## 相关文档
- [业务服务层](./03-service-layer.md)
- [数据持久化层](./05-data-persistence-layer.md)
- [工作流引擎](./07-workflow-engine.md)
