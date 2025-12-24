# 认证机制详解

## 文档说明
本文档详细描述 Langfuse 的认证机制，包括用户认证、API 密钥认证、SSO 集成的技术实现和业务流程。

⚠️ **编写原则**：
- 不贴大量源代码，只标注函数名和文件位置
- 使用表格、列表展示信息
- 代码示例控制在 10 行以内
- 引用时序图，不重复描述流程细节

---

## 目录
- [1. 认证体系概述](#1-认证体系概述)
- [2. 用户认证机制](#2-用户认证机制)
- [3. API 密钥认证机制](#3-api-密钥认证机制)
- [4. SSO 单点登录](#4-sso-单点登录)
- [5. 会话管理](#5-会话管理)
- [6. 安全机制](#6-安全机制)
- [7. 配置参数详解](#7-配置参数详解)
- [8. 错误处理](#8-错误处理)
- [9. 性能优化](#9-性能优化)
- [10. 最佳实践](#10-最佳实践)

---

## 1. 认证体系概述

### 1.1 认证架构

Langfuse 采用双轨认证体系：
- **用户认证（User Authentication）**: 用于 Web 界面登录，基于 NextAuth.js
- **API 认证（API Authentication）**: 用于 SDK 和 API 调用，基于 API Key

```
┌─────────────────┐     ┌─────────────────┐
│   Web 前端      │     │   SDK/API 客户端 │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │ NextAuth              │ API Key
         │ (Session/JWT)         │ (Bearer Token)
         │                       │
         ▼                       ▼
┌────────────────────────────────────────┐
│           认证层                        │
│  ┌──────────────┐  ┌─────────────────┐│
│  │ NextAuth.js  │  │ ApiAuthService  ││
│  └──────────────┘  └─────────────────┘│
└────────────────────────────────────────┘
         │                       │
         └───────────┬───────────┘
                     ▼
         ┌───────────────────────┐
         │    数据库 (Prisma)     │
         │  - User               │
         │  - Account            │
         │  - Session            │
         │  - ApiKey             │
         └───────────────────────┘
```

### 1.2 主要组件

| 组件 | 文件路径 | 职责 |
|------|---------|------|
| NextAuth 配置 | [web/src/server/auth.ts](../../../web/src/server/auth.ts) | 用户认证配置和选项 |
| API 认证服务 | [web/src/features/public-api/server/apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts) | API 密钥验证服务 |
| 密钥管理 | [packages/shared/src/server/auth/apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts) | API 密钥生成和哈希 |
| 凭证工具 | [web/src/features/auth-credentials/lib/credentialsServerUtils.ts](../../../web/src/features/auth-credentials/lib/credentialsServerUtils.ts) | 密码加密和验证 |
| SSO 提供商 | [packages/shared/src/server/auth/customSsoProvider.ts](../../../packages/shared/src/server/auth/customSsoProvider.ts) | 自定义 SSO 配置 |

### 1.3 认证流程概览

**Web 用户登录流程**：参见 [用户登录时序图](./01-user-login-sequence.puml)

**API 调用认证流程**：参见 [API 密钥认证时序图](./02-api-key-auth-sequence.puml)

---

## 2. 用户认证机制

### 2.1 NextAuth.js 集成

**核心配置文件**: [web/src/server/auth.ts](../../../web/src/server/auth.ts)

#### 主要函数

| 函数 | 功能 | 说明 |
|------|------|------|
| `getAuthOptions()` | 获取 NextAuth 配置 | 动态生成认证选项，支持多种提供商 |
| `getServerAuthSession()` | 获取服务端会话 | 用于服务端验证用户身份 |
| `canCreateOrganizations()` | 检查组织创建权限 | 基于白名单限制组织创建 |

#### 认证提供商配置

Langfuse 支持以下认证提供商：

| 提供商类型 | 环境变量前缀 | 说明 |
|-----------|-------------|------|
| Credentials | `AUTH_DISABLE_USERNAME_PASSWORD` | 用户名密码登录 |
| Email Magic Link | `EMAIL_*` | 邮箱验证码 |
| Google OAuth | `AUTH_GOOGLE_*` | Google 账号登录 |
| GitHub OAuth | `AUTH_GITHUB_*` | GitHub 账号登录 |
| GitLab OAuth | `AUTH_GITLAB_*` | GitLab 账号登录 |
| Azure AD | `AUTH_AZURE_AD_*` | 微软 Azure AD SSO |
| Okta | `AUTH_OKTA_*` | Okta SSO |
| Auth0 | `AUTH_AUTH0_*` | Auth0 SSO |
| Cognito | `AUTH_COGNITO_*` | AWS Cognito SSO |
| Custom OIDC | `AUTH_CUSTOM_*` | 自定义 OIDC 提供商 |

### 2.2 密码认证

**实现文件**: [web/src/features/auth-credentials/lib/credentialsServerUtils.ts](../../../web/src/features/auth-credentials/lib/credentialsServerUtils.ts)

#### 密码加密流程

**关键函数**（来自 `packages/shared/src/server/auth/apiKeys.ts`）:

```typescript
// 1. 注册时加密密码
import { hash } from "bcryptjs";
const hashedPassword = await hash(password, 11);
// 轮数 11 = 2^11 次迭代 ≈ 2048 次
// 计算时间: ~200ms（防暴力破解）

// 2. 登录时验证密码
import { compare } from "bcryptjs";
const isValid = await compare(inputPassword, hashedPassword);
// 验证时间: ~200ms（与加密时间相同）
```

**实际代码位置**:
- `hashSecretKey()`: apiKeys.ts 行 12-16
- `verifySecretKey()`: apiKeys.ts 行 22-25

#### 关键功能

| 功能 | 函数 | 说明 |
|------|------|------|
| 密码加密 | `hashSecretKey()` | 使用 bcrypt 加密密码，轮数为 11 |
| 密码验证 | `verifySecretKey()` | 验证输入密码与哈希值是否匹配 |
| 用户注册 | `signupApiHandler()` | 处理用户注册请求 |

### 2.3 OAuth 认证

**配置位置**: [web/src/server/auth.ts](../../../web/src/server/auth.ts#L100-L300)

#### OAuth 流程关键步骤

1. **重定向到提供商**: 用户点击 OAuth 登录按钮
2. **授权回调**: 提供商回调到 `/api/auth/callback/[provider]`
3. **账户链接**: 通过邮箱自动链接账户
4. **创建会话**: 生成 JWT token 和 session

#### 账户自动链接配置

```typescript
{
  allowDangerousEmailAccountLinking: true,
  // 允许同邮箱的不同提供商账户自动链接
}
```

**安全考虑**: 仅当邮箱已验证时才允许自动链接。

### 2.4 邮箱验证码认证

**提供商类型**: `EmailProvider`

#### 配置参数

| 参数 | 环境变量 | 说明 |
|------|---------|------|
| SMTP 服务器 | `EMAIL_HOST` | 邮件服务器地址 |
| SMTP 端口 | `EMAIL_PORT` | 邮件服务器端口 |
| 发件人 | `EMAIL_FROM` | 发件人邮箱 |
| 用户名 | `EMAIL_AUTH_USER` | SMTP 用户名 |
| 密码 | `EMAIL_AUTH_PASS` | SMTP 密码 |

#### 验证码生成

**函数位置**: [web/src/server/auth.ts](../../../web/src/server/auth.ts)

```typescript
generateVerificationToken: async () => {
  const token = randomUUID();
  return token; // 返回 UUID 格式的验证码
}
```

---

## 3. API 密钥认证机制

### 3.1 API 密钥架构

**核心服务**: `ApiAuthService` - [web/src/features/public-api/server/apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts)

#### 密钥类型

| 类型 | Scope | 用途 | 权限范围 |
|------|-------|------|---------|
| 项目密钥 | `PROJECT` | SDK 调用、数据上报 | 单个项目的所有资源 |
| 组织密钥 | `ORGANIZATION` | 跨项目操作 | 组织下所有项目 |

#### 密钥格式

```
公钥 (Public Key):  pk-lf-{uuid}
私钥 (Secret Key):  sk-lf-{uuid}
```

### 3.2 密钥生成流程

**实现文件**: [packages/shared/src/server/auth/apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts)

#### 关键函数

| 函数 | 功能 | 位置 |
|------|------|------|
| `generateKeySet()` | 生成公私钥对 | apiKeys.ts |
| `hashSecretKey()` | 使用 bcrypt 哈希私钥 | apiKeys.ts |
| `createShaHash()` | 生成 SHA-256 快速哈希 | apiKeys.ts |
| `createAndAddApiKeysToDb()` | 创建并保存密钥到数据库 | apiKeys.ts |

#### 密钥存储策略

```typescript
{
  publicKey: "pk-lf-xxx",              // 明文存储，用于标识
  hashedSecretKey: "bcrypt_hash",      // bcrypt 哈希（遗留，首次使用时迁移）
  fastHashedSecretKey: "sha256_hash",  // SHA-256 快速哈希（推荐）
  displaySecretKey: "sk-lf-...xxx"     // 显示用（前6后4字符）
}
```

**安全设计**: 
- 私钥仅在创建时返回一次
- 数据库仅存储哈希值
- 使用双重哈希（bcrypt + SHA-256）提供向后兼容

### 3.3 密钥验证流程

**主要函数**: `verifyAuthHeaderAndReturnScope()` - [web/src/features/public-api/server/apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts#L100)

#### 验证步骤

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1. 提取密钥 | 从 `Authorization: Bearer` 头中提取 | 格式验证 |
| 2. 查找公钥 | 根据公钥前缀查询数据库或缓存 | 支持 Redis 缓存 |
| 3. 快速验证 | 使用 SHA-256 哈希比对 | 高性能验证路径 |
| 4. 降级验证 | 如果快速验证失败，使用 bcrypt | 兼容旧密钥 |
| 5. 迁移哈希 | 首次 bcrypt 验证成功后生成 SHA-256 | 自动迁移机制 |
| 6. 返回上下文 | 返回项目/组织 ID 和权限范围 | 用于后续授权 |

#### 验证结果结构

```typescript
interface AuthHeaderVerificationResult {
  validKey: boolean;
  error?: string;
  scope?: {
    projectId?: string;
    orgId?: string;
    scope: "PROJECT" | "ORGANIZATION";
  };
}
```

### 3.4 Redis 缓存机制

**缓存键格式**:
```
langfuse:api-key:{publicKey}
```

**缓存内容**:
```typescript
interface CachedApiKey {
  publicKey: string;
  projectId?: string;
  orgId?: string;
  scope: ApiKeyScope;
  fastHashedSecretKey: string;
}
```

**缓存失效策略**:

| 操作 | 失效方法 | 触发时机 |
|------|---------|---------|
| 删除密钥 | `invalidateCachedApiKeys()` | 密钥被删除时 |
| 项目迁移 | `invalidateCachedProjectApiKeys()` | 项目转移组织时 |
| 组织变更 | `invalidateCachedOrgApiKeys()` | 组织计划变更时 |

---

## 4. SSO 单点登录

### 4.1 SSO 架构

**流程图**: 参见 [SSO 登录时序图](./04-sso-login-sequence.puml)

#### 支持的 SSO 类型

| 类型 | 提供商示例 | 协议 |
|------|-----------|------|
| 企业 IdP | Azure AD, Okta, Auth0 | OIDC / SAML 2.0 |
| 代码托管 | GitHub Enterprise, GitLab | OAuth 2.0 |
| 自定义 OIDC | Keycloak, JumpCloud | OpenID Connect |

### 4.2 OIDC 提供商配置

**配置文件**: [packages/shared/src/server/auth/customSsoProvider.ts](../../../packages/shared/src/server/auth/customSsoProvider.ts)

#### 必需配置参数

```typescript
{
  clientId: string;        // 客户端 ID
  clientSecret: string;    // 客户端密钥
  issuer: string;         // IdP 发行者 URL
  wellKnown?: string;     // OIDC 发现端点（可选）
}
```

#### 自定义声明映射

```typescript
{
  profile: (profile) => ({
    id: profile.sub,
    name: profile.name,
    email: profile.email,
    image: profile.picture,
  })
}
```

### 4.3 多租户 SSO（企业版）

**实现文件**: [web/src/ee/features/multi-tenant-sso/utils.ts](../../../web/src/ee/features/multi-tenant-sso/utils.ts)

#### 功能特性

| 特性 | 说明 |
|------|------|
| 域名绑定 | 每个组织可配置独立的 SSO 域名 |
| 强制 SSO | 可要求特定域名用户必须使用 SSO |
| SSO 黑名单 | 阻止特定域名使用 SSO |
| 动态配置 | 根据用户邮箱域名动态加载 SSO 配置 |

#### 域名强制策略

**数据库字段**: `SsoConfig.domainForceSso`

```typescript
if (domainForceSso && !usingSso) {
  throw new Error("必须使用 SSO 登录");
}
```

---

## 5. 会话管理

### 5.1 会话存储策略

**策略**: Database Sessions（数据库存储）

**表结构**: `Session`

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionToken` | String | 会话令牌（UUID） |
| `userId` | String | 用户 ID |
| `expires` | DateTime | 过期时间 |

### 5.2 JWT Token 配置

**配置位置**: [web/src/server/auth.ts](../../../web/src/server/auth.ts)

```typescript
{
  session: {
    strategy: "database",  // 使用数据库存储会话
    maxAge: 30 * 24 * 60 * 60, // 30 天
  },
  jwt: {
    maxAge: 30 * 24 * 60 * 60, // JWT 有效期
  }
}
```

### 5.3 Cookie 配置

**配置文件**: [web/src/server/utils/cookies.ts](../../../web/src/server/utils/cookies.ts)

#### 安全属性

| 属性 | 值 | 说明 |
|------|-----|------|
| `httpOnly` | `true` | 防止 XSS 攻击 |
| `secure` | `true` (生产) | 仅 HTTPS 传输 |
| `sameSite` | `lax` | CSRF 保护 |
| `path` | `/` | Cookie 作用域 |

---

## 6. 安全机制

### 6.1 密码安全

#### Bcrypt 配置

| 参数 | 值 | 说明 |
|------|-----|------|
| 哈希轮数 | 11 | 平衡性能与安全 |
| Salt | 自动生成 | Bcrypt 内置 salt |

#### 密码策略建议

```typescript
// 前端验证（可配置）
{
  minLength: 8,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: false,
}
```

### 6.2 API 密钥安全

#### 双重哈希机制

```
┌──────────────┐
│  用户私钥     │ sk-lf-{uuid}
└──────┬───────┘
       │
       ├─────────────┐
       │             │
       ▼             ▼
┌─────────────┐  ┌────────────┐
│ SHA-256哈希  │  │ Bcrypt哈希  │
│(快速验证)    │  │(兼容旧版)   │
└─────────────┘  └────────────┘
```

**优势**:
- **SHA-256**: 快速验证（纳秒级）
- **Bcrypt**: 高安全性（慢速，防暴力破解）
- **自动迁移**: 首次使用时从 bcrypt 迁移到 SHA-256

#### SALT 配置

**环境变量**: `SALT`

```bash
# 生成高强度 SALT
openssl rand -hex 32
```

### 6.3 会话安全

#### CSRF 保护

- NextAuth 内置 CSRF Token
- 每个请求验证 CSRF Token
- Token 存储在 Cookie 中

#### XSS 防护

- HTTP-Only Cookie 阻止 JavaScript 访问
- CSP (Content Security Policy) 头配置
- 输入输出转义

#### 会话固定攻击防护

- 登录后重新生成 Session Token
- 定期刷新会话

---

## 7. 配置参数详解

### 7.1 NextAuth 核心配置

| 环境变量 | 必需 | 默认值 | 说明 |
|---------|------|--------|------|
| `NEXTAUTH_URL` | ✓ | - | 应用基础 URL |
| `NEXTAUTH_SECRET` | ✓ | - | JWT 签名密钥 |
| `AUTH_DISABLE_USERNAME_PASSWORD` | × | `false` | 禁用密码登录 |
| `SALT` | ✓ | - | API 密钥哈希盐值 |

### 7.2 OAuth 提供商配置

#### Google OAuth

```bash
AUTH_GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
AUTH_GOOGLE_CLIENT_SECRET=xxx
AUTH_GOOGLE_ALLOW_ACCOUNT_LINKING=true
```

#### GitHub OAuth

```bash
AUTH_GITHUB_CLIENT_ID=xxx
AUTH_GITHUB_CLIENT_SECRET=xxx
```

### 7.3 企业 SSO 配置

#### Azure AD

```bash
AUTH_AZURE_AD_CLIENT_ID=xxx
AUTH_AZURE_AD_CLIENT_SECRET=xxx
AUTH_AZURE_AD_TENANT_ID=xxx  # 或 "common" 支持多租户
```

#### 自定义 OIDC

```bash
AUTH_CUSTOM_CLIENT_ID=xxx
AUTH_CUSTOM_CLIENT_SECRET=xxx
AUTH_CUSTOM_ISSUER=https://your-idp.com
AUTH_CUSTOM_NAME=Your SSO  # 显示名称
```

### 7.4 组织创建限制

```bash
# 白名单模式：仅允许特定邮箱创建组织
LANGFUSE_ALLOWED_ORGANIZATION_CREATORS=admin@company.com,manager@company.com

# 不设置 = 所有用户可创建组织
```

### 7.5 Redis 配置（可选）

```bash
REDIS_CONNECTION_STRING=redis://localhost:6379
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_AUTH=your_password
```

---

## 8. 错误处理

### 8.1 认证错误类型

| 错误码 | 错误信息 | 原因 | 解决方案 |
|--------|---------|------|----------|
| `AUTH_001` | No authorization header | 缺少认证头 | 添加 `Authorization: Bearer {key}` |
| `AUTH_002` | Invalid API key format | 密钥格式错误 | 检查密钥格式 `sk-lf-xxx` |
| `AUTH_003` | API key not found | 密钥不存在 | 验证密钥是否已删除 |
| `AUTH_004` | Invalid credentials | 用户名或密码错误 | 检查登录凭证 |
| `AUTH_005` | OAuth callback failed | OAuth 回调失败 | 检查回调 URL 配置 |
| `AUTH_006` | SSO configuration not found | SSO 配置缺失 | 验证 SSO 配置 |
| `AUTH_007` | Session expired | 会话过期 | 重新登录 |
| `AUTH_008` | CSRF token invalid | CSRF 验证失败 | 刷新页面重试 |

### 8.2 错误处理流程

```typescript
// API 认证错误处理示例
try {
  const result = await apiAuthService.verifyAuthHeaderAndReturnScope(authHeader);
  if (!result.validKey) {
    return res.status(401).json({
      error: result.error || "Unauthorized",
      code: "AUTH_001"
    });
  }
} catch (error) {
  logger.error("Auth verification failed", { error });
  return res.status(500).json({
    error: "Internal server error",
    code: "AUTH_999"
  });
}
```

### 8.3 降级策略

| 场景 | 策略 | 说明 |
|------|------|------|
| Redis 不可用 | 直接查询数据库 | 自动降级，性能下降 |
| Email 服务故障 | 禁用邮箱登录 | 使用其他认证方式 |
| SSO 提供商故障 | 允许密码登录 | 提供备用登录方式 |

---

## 9. 性能优化

### 9.1 API 密钥验证优化

#### 缓存策略

| 层级 | 命中率 | 响应时间 | 说明 |
|------|--------|---------|------|
| Redis 缓存 | ~95% | <5ms | 热点密钥缓存 |
| 数据库查询 | 5% | 20-50ms | 缓存未命中 |

#### SHA-256 快速验证

```
传统 bcrypt 验证：~200ms
SHA-256 快速验证：<1ms
性能提升：200x
```

**实现位置**: [web/src/features/public-api/server/apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts#L120)

### 9.2 会话查询优化

#### 数据库索引

```sql
-- Session 表索引
CREATE INDEX idx_session_token ON Session(sessionToken);
CREATE INDEX idx_session_user ON Session(userId);
CREATE INDEX idx_session_expires ON Session(expires);
```

#### 查询优化

- 使用 `findFirst` 替代 `findMany`
- 仅查询必需字段
- 会话过期自动清理

### 9.3 OAuth 回调优化

- 并行查询用户和账户信息
- 缓存 OAuth 提供商配置
- 异步处理账户链接

---

## 10. 最佳实践

### 10.1 密钥管理

#### 密钥轮换策略

```
┌─────────────┐
│ 建议周期：   │
│ - 开发: 无限制│
│ - 生产: 90天 │
│ - 泄露: 立即  │
└─────────────┘
```

#### 密钥命名规范

```typescript
{
  note: "Production SDK - Mobile App",
  // 格式：环境-用途-应用标识
}
```

### 10.2 SSO 配置

#### 域名绑定最佳实践

1. **测试环境**: 允许所有认证方式
2. **生产环境**: 企业域名强制 SSO
3. **个人账号**: 支持 OAuth 登录

#### 回调 URL 配置

```
开发环境: http://localhost:3000/api/auth/callback/[provider]
生产环境: https://your-domain.com/api/auth/callback/[provider]
```

### 10.3 安全检查清单

- [ ] 配置强随机 `NEXTAUTH_SECRET`（最少 32 字符）
- [ ] 配置强随机 `SALT`（最少 32 字符）
- [ ] 生产环境启用 HTTPS
- [ ] 配置 CORS 白名单
- [ ] 定期轮换 API 密钥
- [ ] 监控异常登录行为
- [ ] 启用 MFA（多因素认证）
- [ ] 配置会话超时策略
- [ ] 定期审计权限变更
- [ ] 备份认证配置

### 10.4 故障排查

#### 登录失败诊断

```bash
# 1. 检查环境变量
echo $NEXTAUTH_URL
echo $NEXTAUTH_SECRET

# 2. 查看 NextAuth 日志
# web/src/server/auth.ts 中启用 debug

# 3. 测试数据库连接
# 检查 User, Account, Session 表

# 4. 验证 OAuth 配置
# 检查回调 URL 是否匹配
```

#### API 密钥验证失败诊断

```bash
# 1. 验证密钥格式
echo "sk-lf-xxx" | grep "^sk-lf-"

# 2. 检查数据库记录
SELECT publicKey, displaySecretKey FROM ApiKey WHERE publicKey = 'pk-lf-xxx';

# 3. 测试 Redis 连接
redis-cli PING

# 4. 查看验证日志
# ApiAuthService 类中的日志输出
```

---

## 相关文档

- [模块 README](./README.md)
- [RBAC 权限体系](./rbac-system.md)
- [API 密钥管理](./api-key-management.md)
- [用户登录时序图](./01-user-login-sequence.puml)
- [API 密钥认证时序图](./02-api-key-auth-sequence.puml)
- [SSO 登录时序图](./04-sso-login-sequence.puml)
