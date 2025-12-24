# 第三方集成方案总览

## 1. 集成概述

Langfuse 作为一个开源 LLM 工程平台，深度集成了多个第三方服务和技术栈，以提供完整的 LLM 应用监控、评估和管理能力。项目采用**适配器模式**和**工厂模式**来实现灵活的第三方服务集成，确保系统的可扩展性和可维护性。

### 1.1 集成策略

- **松耦合设计**：通过抽象接口层隔离第三方服务，便于切换和升级
- **配置化管理**：所有第三方服务通过环境变量和配置文件管理
- **多提供商支持**：关键服务（如 LLM、存储）支持多个提供商，用户可自由选择
- **降级策略**：对于非核心服务，提供默认实现或降级方案

### 1.2 集成架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Langfuse Application                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  LLM 层      │  │  数据层      │  │  消息层      │          │
│  │              │  │              │  │              │          │
│  │ • OpenAI    │  │ • PostgreSQL │  │ • Slack      │          │
│  │ • Anthropic │  │ • ClickHouse │  │ • Webhook    │          │
│  │ • Vertex AI │  │ • Redis      │  │ • Email      │          │
│  │ • Bedrock   │  │ • S3/Azure   │  │              │          │
│  │ • Gemini    │  │ • GCS        │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  认证层      │  │  可观测性    │  │  支付层      │          │
│  │              │  │              │  │              │          │
│  │ • NextAuth  │  │ • OpenTelem. │  │ • Stripe     │          │
│  │ • OAuth2.0  │  │ • PostHog    │  │              │          │
│  │ • SSO       │  │ • Sentry     │  │              │          │
│  │ • API Keys  │  │ • Winston    │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## 2. 集成类别

| 类别 | 文档 | 集成数量 | 优先级 |
|-----|------|---------|--------|
| 大语言模型集成 | [01-大语言模型集成.md](./01-大语言模型集成.md) | 5+ | ⭐⭐⭐ 核心 |
| 数据库与缓存集成 | [02-数据库与缓存集成.md](./02-数据库与缓存集成.md) | 4 | ⭐⭐⭐ 核心 |
| 对象存储集成 | [03-对象存储集成.md](./03-对象存储集成.md) | 3 | ⭐⭐⭐ 核心 |
| 消息通知集成 | [04-消息通知集成.md](./04-消息通知集成.md) | 3 | ⭐⭐ 重要 |
| 认证授权集成 | [05-认证授权集成.md](./05-认证授权集成.md) | 8+ | ⭐⭐⭐ 核心 |
| 可观测性集成 | [06-可观测性集成.md](./06-可观测性集成.md) | 5 | ⭐⭐ 重要 |
| 支付集成 | [07-支付集成.md](./07-支付集成.md) | 1 | ⭐⭐ 重要 |
| AI增强服务集成 | [08-AI增强服务集成.md](./08-AI增强服务集成.md) | 2 | ⭐ 可选 |

> **注意**: Langfuse 作为 LLM 可观测性平台,其核心功能聚焦于追踪、评估和分析。以下类别在项目中**不适用**:
> - ❌ 向量数据库集成 (Langfuse 不提供 RAG 或向量搜索功能)
> - ❌ 文档处理集成 (无 PDF 解析、OCR 等文档处理)
> - ❌ 代码执行集成 (无沙箱环境或代码执行功能)
> - ❌ 知识库集成 (不直接集成外部知识源)

## 3. 统一接口设计

### 3.1 LLM 提供商统一接口

Langfuse 通过 **Langchain** 作为统一的 LLM 集成层，提供一致的接口：

```typescript
// 所有 LLM 提供商都通过 Langchain 的 ChatModel 接口
interface LLMProvider {
  chat(messages: ChatMessage[], options: ChatOptions): Promise<ChatResponse>;
  stream(messages: ChatMessage[], options: ChatOptions): AsyncIterable<Chunk>;
}

// 支持的提供商
- ChatOpenAI (OpenAI)
- AzureChatOpenAI (Azure OpenAI)
- ChatAnthropic (Anthropic Claude)
- ChatVertexAI (Google Vertex AI)
- ChatBedrockConverse (AWS Bedrock)
- ChatGoogleGenerativeAI (Google Gemini)
```

### 3.2 存储服务统一接口

```typescript
interface StorageService {
  uploadFile(params: UploadFile): Promise<string>;
  getSignedDownloadUrl(key: string): Promise<string>;
  getSignedUploadUrl(key: string): Promise<string>;
  deleteFile(key: string): Promise<void>;
}

// 支持的提供商
- S3StorageService (AWS S3)
- AzureBlobStorageService (Azure Blob Storage)
- GoogleCloudStorageService (Google Cloud Storage)
```

### 3.3 消息服务统一接口

```typescript
interface MessagingService {
  sendMessage(params: MessageParams): Promise<MessageResponse>;
  validateConfiguration(): Promise<boolean>;
}

// 支持的提供商
- SlackService
- WebhookService
- EmailService (SMTP)
```

## 4. 配置管理

### 4.1 环境变量清单

#### 数据库配置
```bash
# PostgreSQL
DATABASE_URL="postgresql://..."

# ClickHouse
CLICKHOUSE_URL="http://..."
CLICKHOUSE_USER="default"
CLICKHOUSE_PASSWORD="..."

# Redis
REDIS_HOST="localhost"
REDIS_PORT="6379"
REDIS_AUTH="..."
```

#### LLM 连接配置
```bash
# 用户自定义 LLM 连接 (加密存储在数据库中)
LANGFUSE_LLM_CONNECTION_OPENAI_KEY="sk-..."
LANGFUSE_LLM_CONNECTION_ANTHROPIC_KEY="sk-ant-..."

# 内部评估使用 (可选)
LANGFUSE_INTERNAL_OPENAI_API_KEY="sk-..."
```

#### 对象存储配置
```bash
# S3 / R2 / MinIO
LANGFUSE_S3_EVENT_UPLOAD_BUCKET="..."
LANGFUSE_S3_EVENT_UPLOAD_REGION="..."
LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID="..."
LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY="..."
LANGFUSE_S3_EVENT_UPLOAD_ENDPOINT="..."  # 可选，用于 MinIO/R2

# Azure Blob Storage
LANGFUSE_AZURE_BLOB_CONTAINER_NAME="..."
AZURE_STORAGE_CONNECTION_STRING="..."

# Google Cloud Storage
LANGFUSE_GCS_BUCKET_NAME="..."
GCS_CREDENTIALS_PATH="..."
```

#### 认证配置
```bash
# NextAuth
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="..."

# OAuth Providers
AUTH_GOOGLE_CLIENT_ID="..."
AUTH_GOOGLE_CLIENT_SECRET="..."
AUTH_GITHUB_CLIENT_ID="..."
AUTH_GITHUB_CLIENT_SECRET="..."

# SSO (企业版)
AUTH_OKTA_CLIENT_ID="..."
AUTH_OKTA_ISSUER="..."
AUTH_JUMPCLOUD_CLIENT_ID="..."
```

#### 消息通知配置
```bash
# Email (SMTP)
SMTP_CONNECTION_URL="smtp://..."
EMAIL_FROM_ADDRESS="noreply@..."

# Slack
SLACK_CLIENT_ID="..."
SLACK_CLIENT_SECRET="..."

# Webhook
# (在应用内配置)
```

#### 支付配置
```bash
# Stripe
STRIPE_SECRET_KEY="sk_..."
STRIPE_WEBHOOK_SIGNING_SECRET="whsec_..."
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="pk_..."
```

#### 可观测性配置
```bash
# OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT="http://..."
OTEL_SERVICE_NAME="langfuse-web"

# PostHog
NEXT_PUBLIC_POSTHOG_KEY="..."
NEXT_PUBLIC_POSTHOG_HOST="https://..."

# Sentry
NEXT_PUBLIC_SENTRY_DSN="..."
SENTRY_ORG="..."
SENTRY_PROJECT="..."

# Datadog (可选)
DD_AGENT_HOST="..."
DD_TRACE_ENABLED="true"
```

### 4.2 配置加载顺序

1. `.env` 文件 (本地开发)
2. `.env.local` (本地覆盖，不提交到版本控制)
3. `.env.production` (生产环境)
4. 系统环境变量 (最高优先级)

### 4.3 敏感信息管理

- **API Keys**: 使用加密存储在数据库中 (用户自定义的 LLM 连接)
- **环境变量**: 通过 Kubernetes Secrets 或云服务 Secret Manager 管理
- **配置加密**: 使用 `@langfuse/shared/encryption` 模块进行加密解密

## 5. 依赖版本管理

### 5.1 核心依赖版本

| 依赖包 | 版本 | 用途 |
|-------|------|------|
| `@langchain/openai` | ^0.5.13 | OpenAI 集成 |
| `@langchain/anthropic` | ^0.3.32 | Anthropic 集成 |
| `@langchain/google-vertexai` | ^0.2.12 | Google Vertex AI 集成 |
| `@langchain/aws` | ^0.1.15 | AWS Bedrock 集成 |
| `@langchain/google-genai` | ^0.2.12 | Google Gemini 集成 |
| `@prisma/client` | ^6.17.1 | PostgreSQL ORM |
| `@clickhouse/client` | ^1.13.0 | ClickHouse 客户端 |
| `ioredis` | ^5.8.2 | Redis 客户端 |
| `bullmq` | ^5.34.10 | 队列管理 |
| `@aws-sdk/client-s3` | ^3.675.0 | AWS S3 客户端 |
| `@azure/storage-blob` | ^12.26.0 | Azure Blob Storage |
| `@google-cloud/storage` | ^7.17.0 | Google Cloud Storage |
| `next-auth` | ^4.24.12 | 认证框架 |
| `stripe` | ^17.4.0 (worker) / ^18.5.0 (web) | 支付处理 |
| `@slack/web-api` | ^7.10.0 | Slack 集成 |
| `nodemailer` | ^7.0.11 | 邮件发送 |
| `@opentelemetry/sdk-node` | ^0.53.0 | OpenTelemetry SDK |
| `posthog-js` | ^1.273.1 | PostHog 客户端 |
| `posthog-node` | ^5.8.4 | PostHog 服务端 |

### 5.2 依赖更新策略

- **主要依赖** (如 Langchain, Prisma): 谨慎更新，需要充分测试
- **安全补丁**: 及时更新
- **次要版本**: 定期更新，关注 breaking changes
- **pnpm**: 使用 lockfile 确保依赖版本一致性

## 6. 扩展指南

### 6.1 添加新的 LLM 提供商

1. **添加 Langchain 依赖**
   ```bash
   pnpm add @langchain/new-provider --filter=@langfuse/shared
   ```

2. **更新 LLM 适配器**
   
   在 `packages/shared/src/server/llm/fetchLLMCompletion.ts` 中添加新的 case：
   ```typescript
   case LLMAdapter.NewProvider:
     const newProviderClient = new ChatNewProvider({
       apiKey: decrypt(llmConnection.secretKey),
       ...
     });
     return newProviderClient;
   ```

3. **更新配置 Schema**
   
   在 `packages/shared/src/interfaces/customLLMProviderConfigSchemas.ts` 中添加配置验证：
   ```typescript
   export const NewProviderConfigSchema = z.object({
     apiKey: z.string(),
     ...
   });
   ```

4. **添加 UI 配置**
   
   在 Web 应用中添加提供商选择和配置表单

### 6.2 添加新的存储提供商

1. **实现 StorageService 接口**
   
   在 `packages/shared/src/server/services/StorageService.ts` 中添加新类：
   ```typescript
   export class NewStorageService implements StorageService {
     async uploadFile(params: UploadFile): Promise<string> {
       // 实现上传逻辑
     }
     // ... 其他方法
   }
   ```

2. **更新工厂方法**
   ```typescript
   export class StorageServiceFactory {
     static create(): StorageService {
       if (env.NEW_STORAGE_ENABLED) {
         return new NewStorageService();
       }
       // ... 现有逻辑
     }
   }
   ```

### 6.3 添加新的认证提供商

1. **添加 NextAuth Provider**
   
   在 `web/src/pages/api/auth/[...nextauth].ts` 中添加：
   ```typescript
   import NewProvider from "next-auth/providers/new-provider";
   
   providers: [
     NewProvider({
       clientId: env.AUTH_NEW_CLIENT_ID,
       clientSecret: env.AUTH_NEW_CLIENT_SECRET,
     }),
   ]
   ```

2. **更新环境变量配置**
   ```bash
   AUTH_NEW_CLIENT_ID="..."
   AUTH_NEW_CLIENT_SECRET="..."
   ```

## 7. 测试策略

### 7.1 集成测试

- **Mock 外部服务**: 使用 `msw` (Mock Service Worker) 模拟第三方 API
- **测试覆盖**: 确保关键集成路径有测试覆盖
- **环境隔离**: 使用 `.env.test` 配置测试环境

### 7.2 端到端测试

- **Playwright**: 测试完整的用户流程，包括第三方服务交互
- **测试数据**: 使用测试账号和 Sandbox 环境

## 8. 故障排查

### 8.1 常见问题

#### LLM 连接失败
- **检查 API Key**: 确保密钥正确且未过期
- **检查网络**: 确保服务器可以访问 LLM API
- **检查速率限制**: 查看是否触发 API 速率限制

#### 存储上传失败
- **检查权限**: 确保 IAM 角色有足够权限
- **检查网络**: 确保可以访问存储服务
- **检查配置**: 验证 bucket 名称和区域设置

#### 认证问题
- **检查 Callback URL**: 确保 OAuth 回调 URL 配置正确
- **检查会话**: 验证 session 配置和 cookie 设置
- **检查数据库**: 确保 session 表正常工作

### 8.2 日志和监控

- **结构化日志**: 使用 Winston 记录详细的第三方服务调用日志
- **错误追踪**: 使用 Sentry 捕获和分析错误
- **性能监控**: 使用 OpenTelemetry 追踪第三方服务调用性能

## 9. 安全最佳实践

1. **API Key 管理**
   - 使用环境变量存储敏感信息
   - 在数据库中加密存储用户提供的 API Keys
   - 定期轮换密钥

2. **网络安全**
   - 使用 HTTPS 进行所有外部通信
   - 配置适当的 CORS 策略
   - 使用代理服务器 (可选)

3. **权限控制**
   - 遵循最小权限原则
   - 使用 IAM 角色而非固定凭证 (云环境)
   - 定期审计权限配置

4. **数据保护**
   - 不在日志中记录敏感信息
   - 对传输中的数据加密
   - 遵守数据合规要求 (GDPR, CCPA 等)

## 10. 成本优化

### 10.1 LLM 成本
- **缓存策略**: 缓存 LLM 响应以减少 API 调用
- **批量处理**: 批量评估以减少调用次数
- **模型选择**: 根据任务复杂度选择合适的模型

### 10.2 存储成本
- **生命周期策略**: 配置对象存储的生命周期规则
- **压缩**: 压缩大型文件
- **存储层级**: 使用冷存储保存历史数据

### 10.3 数据库成本
- **查询优化**: 优化 ClickHouse 和 PostgreSQL 查询
- **数据保留**: 设置合理的数据保留策略
- **索引管理**: 维护适当的索引

## 11. 合规性考虑

### 11.1 数据驻留
- **区域部署**: 支持在特定区域部署以满足数据驻留要求
- **提供商选择**: 选择符合合规要求的第三方服务

### 11.2 数据处理协议
- **DPA 协议**: 与第三方服务签订数据处理协议
- **隐私政策**: 确保隐私政策涵盖所有第三方集成
- **用户同意**: 获取必要的用户同意

## 12. 相关资源

- **官方文档**: [langfuse.com/docs](https://langfuse.com/docs)
- **GitHub 仓库**: [github.com/langfuse/langfuse](https://github.com/langfuse/langfuse)
- **社区支持**: [Discord](https://discord.gg/7NXusRtqYU)
- **问题追踪**: [GitHub Issues](https://github.com/langfuse/langfuse/issues)

---

**文档版本**: v1.0  
**更新日期**: 2025-12-17  
**维护者**: Langfuse Engineering Team
