# API 密钥管理 - 详细说明

## 文档说明
本文档详细描述 Langfuse 的 API 密钥管理系统，包括密钥生成、验证、缓存、生命周期管理和安全实践。

⚠️ **编写原则**：
- 不贴大量源代码，只标注函数名和文件位置
- 使用表格、列表展示信息
- 代码示例控制在 10 行以内
- 引用时序图，不重复描述流程细节

---

## 目录
- [1. API 密钥体系概述](#1-api-密钥体系概述)
- [2. 密钥生成机制](#2-密钥生成机制)
- [3. 密钥验证流程](#3-密钥验证流程)
- [4. 缓存策略](#4-缓存策略)
- [5. 密钥管理 API](#5-密钥管理-api)
- [6. 认证模式](#6-认证模式)
- [7. 安全机制](#7-安全机制)
- [8. 配置参数](#8-配置参数)
- [9. 性能优化](#9-性能优化)
- [10. 最佳实践](#10-最佳实践)

---

## 1. API 密钥体系概述

### 1.1 密钥架构

```
┌─────────────────────────────────────┐
│         API 密钥系统                 │
├─────────────────────────────────────┤
│                                     │
│  ┌──────────────┐  ┌─────────────┐ │
│  │项目级密钥     │  │组织级密钥    │ │
│  │(PROJECT)     │  │(ORGANIZATION)│ │
│  └──────┬───────┘  └──────┬──────┘ │
│         │                 │        │
│         └────────┬────────┘        │
│                  │                 │
│         ┌────────▼────────┐        │
│         │  ApiAuthService │        │
│         └────────┬────────┘        │
│                  │                 │
│         ┌────────▼────────┐        │
│         │  Redis 缓存层    │        │
│         └────────┬────────┘        │
│                  │                 │
│         ┌────────▼────────┐        │
│         │   数据库层       │        │
│         │   (Prisma)      │        │
│         └─────────────────┘        │
└─────────────────────────────────────┘
```

### 1.2 核心组件

| 组件 | 文件路径 | 职责 |
|------|---------|------|
| **密钥生成** | [packages/shared/src/server/auth/apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts) | 生成公私钥对，哈希处理 |
| **认证服务** | [web/src/features/public-api/server/apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts) | 验证 API 密钥，权限检查 |
| **密钥路由** | [web/src/server/api/routers/llmApiKeys.ts](../../../web/src/server/api/routers/llmApiKeys.ts) | 密钥 CRUD 操作 |

### 1.3 密钥类型

| 类型 | Scope | 用途 | 认证方式 |
|------|-------|------|---------|
| **项目密钥** | `PROJECT` | SDK 调用、Trace 上报 | Basic Auth / Bearer |
| **组织密钥** | `ORGANIZATION` | 跨项目操作、批量查询 | Basic Auth 仅 |

### 1.4 密钥格式

```
公钥（Public Key）:  pk-lf-{uuid}
私钥（Secret Key）:  sk-lf-{uuid}

示例:
pk-lf-1a2b3c4d-5e6f-7890-abcd-ef1234567890
sk-lf-9z8y7x6w-5v4u-3t2s-1r0q-p0o9i8u7y6t5
```

**设计特点**:
- UUID 保证全局唯一性
- `lf` 前缀标识 Langfuse
- 公钥可公开，私钥严格保密

---

## 2. 密钥生成机制

### 2.1 生成流程

**核心函数**: `createAndAddApiKeysToDb()` - [packages/shared/src/server/auth/apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts)

#### 生成步骤

| 步骤 | 操作 | 函数 |
|------|------|------|
| 1. 生成密钥对 | 创建公钥和私钥 | `generateKeySet()` |
| 2. Bcrypt 哈希 | 使用 bcrypt 哈希私钥 | `hashSecretKey()` |
| 3. SHA-256 哈希 | 生成快速验证哈希 | `createShaHash()` |
| 4. 生成显示密钥 | 截取前后字符 | `getDisplaySecretKey()` |
| 5. 保存到数据库 | 存储密钥信息 | `prisma.apiKey.create()` |
| 6. 返回密钥 | 返回完整私钥（仅此一次） | - |

#### 代码位置

**generateKeySet()** - [apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts#L15-L20)
```typescript
async function generateKeySet() {
  return {
    pk: `pk-lf-${randomUUID()}`,
    sk: `sk-lf-${randomUUID()}`,
  };
}
```

### 2.2 哈希策略

#### 双重哈希设计

```
            私钥 (sk-lf-xxx)
                 │
         ┌───────┴────────┐
         │                │
         ▼                ▼
    Bcrypt 哈希       SHA-256 哈希
    (安全慢速)        (快速验证)
         │                │
         ▼                ▼
  hashedSecretKey   fastHashedSecretKey
  (数据库存储)      (数据库存储 + Redis)
```

#### Bcrypt 哈希

**函数**: `hashSecretKey()` - [apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts#L12-L16)

| 参数 | 值 | 说明 |
|------|-----|------|
| 哈希算法 | bcrypt | 密码学安全哈希 |
| 轮数（rounds） | 11 | 计算复杂度 |
| Salt | 自动生成 | 每个密钥独立 salt |
| 计算时间 | ~200ms | 防暴力破解 |

#### SHA-256 快速哈希

**函数**: `createShaHash()` - [apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts#L25-L31)

```typescript
function createShaHash(privateKey: string, salt: string): string {
  const hash = crypto
    .createHash("sha256")
    .update(privateKey)
    .update(crypto.createHash("sha256").update(salt, "utf8").digest("hex"))
    .digest("hex");
  return hash;
}
```

| 特性 | 说明 |
|------|------|
| 算法 | SHA-256 |
| Salt | 全局 `SALT` 环境变量 |
| 计算时间 | <1ms |
| 用途 | 快速验证 + Redis 缓存键 |

### 2.3 显示密钥

**函数**: `getDisplaySecretKey()` - [apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts#L7-L10)

```typescript
function getDisplaySecretKey(secretKey: string) {
  return secretKey.slice(0, 6) + "..." + secretKey.slice(-4);
}
```

**示例**:
```
完整私钥: sk-lf-1a2b3c4d-5e6f-7890-abcd-ef1234567890
显示密钥: sk-lf-...7890
```

**用途**: UI 显示，帮助用户识别密钥而不泄露完整值

### 2.4 数据库存储

**表**: `ApiKey`

**创建函数**: `createAndAddApiKeysToDb()` - [apiKeys.ts](../../../packages/shared/src/server/auth/apiKeys.ts#L38-L85)

**函数代码行数**: 48 行

| 字段 | 类型 | 说明 | 示例 | 创建位置 |
|------|------|------|------|----------|
| `id` | String | 主键 | `clx1y2z3...` | 自动生成 |
| `publicKey` | String | 公钥（明文） | `pk-lf-xxx` | 行 53-55 |
| `hashedSecretKey` | String | Bcrypt 哈希 | `$2a$11$...` | 行 57 |
| `displaySecretKey` | String | 显示用 | `sk-lf-...7890` | 行 58 |
| `fastHashedSecretKey` | String | SHA-256 哈希 | `a3f5d9e...` | 行 60 |
| `scope` | Enum | 密钥范围 | `PROJECT` / `ORGANIZATION` | 参数传入 |
| `projectId` | String | 项目 ID（PROJECT 类型） | `proj-id` | 行 62-63 |
| `orgId` | String | 组织 ID（ORGANIZATION 类型） | `org-id` | 行 62-63 |
| `note` | String | 备注 | `Production SDK` | 参数传入 |
| `createdAt` | DateTime | 创建时间 | - | 自动生成 |

**存储流程**（行 38-85）:
1. 验证 SALT 环境变量（行 48-50）
2. 生成或使用预定义密钥对（行 53-55）
3. 三重哈希处理（行 57-60）
4. 确定实体类型（PROJECT/ORGANIZATION）（行 62-63）
5. 数据库插入（行 65-73）
6. 返回完整私钥（仅此一次）（行 75-82）

---

## 3. 密钥验证流程

### 3.1 验证架构

**核心类**: `ApiAuthService` - [web/src/features/public-api/server/apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts)

#### ApiAuthService 完整结构

**类定义**: 行 28-490（共 463 行代码）

| 类别 | 成员 | 说明 | 代码行数 |
|------|------|------|----------|
| **属性** | `prisma` | Prisma 客户端实例 | - |
| | `redis` | Redis 客户端实例（可选） | - |
| **构造函数** | `constructor()` | 初始化 prisma 和 redis | 4 行 |
| **核心方法** | `verifyAuthHeaderAndReturnScope()` | API 密钥验证主方法 | 176 行 |
| **缓存管理** | `fetchApiKeyAndAddToRedis()` | 获取密钥并缓存 | 38 行 |
| | `fetchApiKeyFromRedis()` | 从 Redis 读取 | 35 行 |
| | `addApiKeyToRedis()` | 写入 Redis 缓存 | 19 行 |
| | `invalidateCachedApiKeys()` | 失效指定密钥缓存 | 3 行 |
| | `invalidateCachedProjectApiKeys()` | 失效项目密钥缓存 | 3 行 |
| | `invalidateCachedOrgApiKeys()` | 失效组织密钥缓存 | 3 行 |
| **辅助方法** | `extractBasicAuthCredentials()` | 解析 Basic Auth | 11 行 |
| | `findDbKeyOrThrow()` | 数据库查询密钥 | 14 行 |
| | `extractOrgIdAndCloudConfig()` | 提取组织配置 | 52 行 |
| | `convertToRedisRepresentation()` | 转换为 Redis 格式 | 43 行 |
| | `createRedisKey()` | 生成 Redis 键名 | 3 行 |
| | `deleteApiKey()` | 删除密钥 | 26 行 |

**总计**: 13 个方法，涵盖验证、缓存、辅助功能

#### 验证流程图

参见 [API 密钥认证时序图](./02-api-key-auth-sequence.puml)

### 3.2 Basic Auth 验证

**认证头格式**: `Authorization: Basic {base64(pk:sk)}`

#### 验证步骤（完整决策树）

**基于 176 行代码的 `verifyAuthHeaderAndReturnScope()` 方法分析：**

```
验证入口
├─ 检查 authHeader 是否存在
│  └─ 否 → 返回 "No authorization header"
│
├─ Basic Auth 分支 (authHeader.startsWith("Basic "))
│  ├─ 1. extractBasicAuthCredentials() → 提取公私钥 (<1ms)
│  ├─ 2. createShaHash(secretKey, SALT) → SHA-256 哈希 (<1ms)
│  ├─ 3. fetchApiKeyAndAddToRedis(hash) → 尝试缓存
│  │  ├─ Redis 查询 (1-5ms)
│  │  ├─ 缓存命中 → 跳到步骤 7
│  │  └─ 缓存未命中 → 继续
│  ├─ 4. 检查 fastHashedSecretKey 是否存在
│  │  ├─ 存在 → 快速验证成功，跳到步骤 7
│  │  └─ 不存在 → 降级到 bcrypt 验证
│  ├─ 5. prisma.apiKey.findUnique(publicKey) → 数据库查询 (20-50ms)
│  │  └─ 未找到 → 缓存 API_KEY_NON_EXISTENT，返回错误
│  ├─ 6. verifySecretKey(bcrypt) → 慢速验证 (~200ms)
│  │  ├─ 验证失败 → 返回 "Invalid credentials"
│  │  └─ 验证成功 → 哈希迁移
│  │     ├─ createShaHash() → 生成快速哈希
│  │     ├─ prisma.apiKey.update() → 更新 fastHashedSecretKey
│  │     └─ convertToRedisRepresentation() → 转换格式
│  ├─ 7. 验证计划类型 isPlan(plan)
│  ├─ 8. addUserToSpan() → 添加追踪信息
│  └─ 9. 返回完整权限 (accessLevel: "project" | "organization")
│
└─ Bearer Auth 分支 (authHeader.startsWith("Bearer "))
   ├─ 1. 提取 publicKey (replace "Bearer ", "")
   ├─ 2. findDbKeyOrThrow(publicKey) → 数据库查询
   ├─ 3. 检查 scope !== "ORGANIZATION"
   │  └─ 是组织密钥 → 抛出错误
   ├─ 4. extractOrgIdAndCloudConfig() → 提取配置
   ├─ 5. addUserToSpan() → 添加追踪信息
   └─ 6. 返回受限权限 (accessLevel: "scores" only)
```

| 步骤 | 操作 | 方法名 | 代码位置 | 耗时 |
|------|------|--------|---------|------|
| 1. 提取凭证 | Base64 解码获取公私钥 | `extractBasicAuthCredentials()` | 行 262-272 | <1ms |
| 2. 计算哈希 | SHA-256 哈希私钥 | `createShaHash()` | shared/apiKeys.ts | <1ms |
| 3. 查询缓存 | Redis 查询密钥信息 | `fetchApiKeyFromRedis()` | 行 348-382 | 1-5ms |
| 4. 快速验证 | 比对 fastHashedSecretKey | - | 行 114-115 | <1ms |
| 5. 降级查询 | 根据 publicKey 查数据库 | `prisma.apiKey.findUnique()` | 行 116-121 | 20-50ms |
| 6. Bcrypt 验证 | 慢速哈希验证 | `verifySecretKey()` | 行 138-141 | ~200ms |
| 7. 哈希迁移 | 更新为快速哈希 | `prisma.apiKey.update()` | 行 145-150 | 10-20ms |
| 8. 更新缓存 | 写入 Redis | `addApiKeyToRedis()` | 行 328-346 | 1-5ms |
| 9. 返回权限 | 返回项目/组织上下文 | - | 行 165-183 | <1ms |

#### 代码位置

**verifyAuthHeaderAndReturnScope()** - [apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts#L100-L260)

```typescript
if (authHeader.startsWith("Basic ")) {
  // 1. 提取公私钥
  const { username: publicKey, password: secretKey } =
    this.extractBasicAuthCredentials(authHeader);

  // 2. 计算快速哈希
  const hashFromProvidedKey = createShaHash(secretKey, salt);

  // 3. 从 Redis 或数据库获取密钥
  const apiKey = await this.fetchApiKeyAndAddToRedis(hashFromProvidedKey);

  // 4. 验证通过，返回权限范围
  return {
    validKey: true,
    scope: {
      projectId: apiKey.projectId,
      orgId: apiKey.orgId,
      scope: apiKey.scope,
      // ...
    },
  };
}
```

### 3.3 Bearer Auth 验证

**认证头格式**: `Authorization: Bearer {publicKey}`

**限制**: 仅支持项目级密钥（PROJECT scope），且仅能访问评分（scores）功能

#### 验证步骤

| 步骤 | 操作 | 位置 |
|------|------|------|
| 1. 提取公钥 | 从 Bearer token 获取 | - |
| 2. 查询数据库 | 根据公钥查询密钥信息 | `findDbKeyOrThrow()` |
| 3. 检查范围 | 验证是否为 PROJECT 类型 | - |
| 4. 返回权限 | 返回受限权限（scores only） | - |

**代码位置**: [apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts#L200-L230)

```typescript
if (authHeader.startsWith("Bearer ")) {
  const publicKey = authHeader.replace("Bearer ", "");
  
  const dbKey = await this.findDbKeyOrThrow(publicKey);
  
  // 组织密钥不支持 Bearer 认证
  if (dbKey.scope === "ORGANIZATION") {
    throw new Error("Cannot use organization key with bearer auth");
  }
  
  return {
    validKey: true,
    scope: {
      projectId: dbKey.projectId,
      accessLevel: "scores",  // 仅评分访问
      // ...
    },
  };
}
```

### 3.4 验证结果

**返回类型**: `AuthHeaderVerificationResult`

```typescript
interface AuthHeaderVerificationResult {
  validKey: boolean;
  error?: string;
  scope?: {
    projectId?: string;
    orgId: string;
    scope: ApiKeyScope;
    accessLevel: "project" | "organization" | "scores";
    plan: string;
    apiKeyId: string;
    publicKey: string;
    rateLimitOverrides: RateLimitOverride[];
    isIngestionSuspended: boolean;
  };
}
```

---

## 4. 缓存策略

### 4.1 Redis 缓存架构

#### 缓存键格式

```
langfuse:api-key:{fastHashedSecretKey}
```

**示例**:
```
langfuse:api-key:a3f5d9e2c1b7...
```

#### 缓存数据结构

**类型**: `CachedApiKey`

```typescript
{
  id: string;
  publicKey: string;
  projectId?: string;
  orgId: string;
  scope: "PROJECT" | "ORGANIZATION";
  fastHashedSecretKey: string;
  plan: string;
  rateLimitOverrides: Array;
  isIngestionSuspended: boolean;
}
```

### 4.2 缓存操作

#### 查询缓存

**函数**: `fetchApiKeyFromRedis()` - [apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts#L350)

| 场景 | 结果 | 操作 |
|------|------|------|
| 缓存命中 | 返回密钥信息 | 直接使用 |
| 缓存未命中 | 返回 null | 查询数据库 |
| 密钥不存在 | 返回 `API_KEY_NON_EXISTENT` | 记录并缓存（避免重复查询） |

#### 写入缓存

**函数**: `addApiKeyToRedis()` - [apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts#L320)

**配置**:
```typescript
{
  key: "langfuse:api-key:{hash}",
  value: JSON.stringify(apiKey),
  ttl: env.LANGFUSE_CACHE_API_KEY_TTL_SECONDS, // 默认 3600 秒（1 小时）
}
```

#### 缓存失效

**场景和方法**:

| 场景 | 方法 | 触发时机 |
|------|------|---------|
| 删除密钥 | `invalidateCachedApiKeys()` | 密钥被删除 |
| 项目变更 | `invalidateCachedProjectApiKeys()` | 项目转移组织 |
| 组织变更 | `invalidateCachedOrgApiKeys()` | 组织计划变更 |

**实现位置**: [apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts#L40-L52)

### 4.3 缓存性能

#### 性能指标

| 指标 | 无缓存 | 有缓存 | 提升 |
|------|--------|--------|------|
| 验证时间 | 50-250ms | 2-10ms | 10-50x |
| 数据库查询 | 每次请求 | 缓存未命中时 | 99%+ 减少 |
| 缓存命中率 | - | ~95% | - |

#### 监控指标

```typescript
// 埋点统计
recordIncrement("langfuse.api_key.cache_hit", 1);
recordIncrement("langfuse.api_key.cache_miss", 1);
```

---

## 5. 密钥管理 API

### 5.1 密钥 CRUD 操作

#### tRPC 路由

**文件**: [web/src/server/api/routers/llmApiKeys.ts](../../../web/src/server/api/routers/llmApiKeys.ts)

| 操作 | Procedure | 权限要求 |
|------|-----------|---------|
| 列出密钥 | `llmApiKeys.all` | `apiKeys:read` |
| 创建密钥 | `llmApiKeys.create` | `apiKeys:CUD` |
| 删除密钥 | `llmApiKeys.delete` | `apiKeys:CUD` |

### 5.2 创建密钥

**Procedure**: `create`

#### 输入参数

```typescript
{
  projectId: string;           // 项目 ID（项目密钥）
  orgId?: string;              // 组织 ID（组织密钥）
  scope: "PROJECT" | "ORGANIZATION";  // 密钥范围
  note?: string;               // 备注说明
}
```

#### 返回结果

```typescript
{
  id: string;
  publicKey: string;
  secretKey: string;          // ⚠️ 仅返回一次！
  displaySecretKey: string;
  note: string | null;
  createdAt: Date;
}
```

**⚠️ 重要提示**: `secretKey` 仅在创建时返回一次，务必安全保存！

### 5.3 删除密钥

**Procedure**: `delete`

#### 输入参数

```typescript
{
  id: string;              // 密钥 ID
  projectId?: string;      // 项目 ID（项目密钥）
  orgId?: string;          // 组织 ID（组织密钥）
  scope: ApiKeyScope;      // 密钥范围
}
```

#### 操作步骤

1. 验证密钥存在且属于指定项目/组织
2. 从 Redis 缓存中删除
3. 从数据库中删除
4. 返回成功/失败

**实现位置**: [apiAuth.ts](../../../web/src/features/public-api/server/apiAuth.ts#L60-L87)

### 5.4 查询密钥列表

**Procedure**: `all`

#### 输入参数

```typescript
{
  projectId?: string;      // 项目 ID（查询项目密钥）
  orgId?: string;          // 组织 ID（查询组织密钥）
  scope: ApiKeyScope;      // 密钥范围
}
```

#### 返回结果

```typescript
Array<{
  id: string;
  publicKey: string;
  displaySecretKey: string;  // 显示用（sk-lf-...xxx）
  note: string | null;
  createdAt: Date;
  lastUsedAt: Date | null;
}>
```

**注意**: 不返回完整私钥，仅返回显示密钥

---

## 6. 认证模式

### 6.1 Basic Auth（完整访问）

#### 适用场景

- SDK 客户端（Python, JavaScript, Go 等）
- 服务端应用
- 数据上报和查询

#### 使用方式

```bash
# 公钥和私钥组合认证
curl -X POST https://cloud.langfuse.com/api/public/ingestion \
  -u pk-lf-xxx:sk-lf-yyy \
  -H "Content-Type: application/json" \
  -d '{"batch": [...]}'
```

**编码格式**: Base64(`publicKey:secretKey`)

#### 权限范围

| Scope | 访问权限 |
|-------|---------|
| PROJECT | 项目所有资源的 CRUD |
| ORGANIZATION | 组织所有项目的资源 |

### 6.2 Bearer Auth（受限访问）

#### 适用场景

- 评分提交（Scores）
- 公开集成
- 受限权限的第三方服务

#### 使用方式

```bash
# 仅使用公钥
curl -X POST https://cloud.langfuse.com/api/public/scores \
  -H "Authorization: Bearer pk-lf-xxx" \
  -H "Content-Type: application/json" \
  -d '{"name": "quality", "value": 0.9}'
```

#### 限制

- ✅ 仅支持项目密钥（PROJECT scope）
- ❌ 不支持组织密钥
- ✅ 仅能访问评分（scores）端点
- ❌ 无法访问其他资源

### 6.3 认证模式对比

| 特性 | Basic Auth | Bearer Auth |
|------|-----------|-------------|
| 认证材料 | 公钥 + 私钥 | 仅公钥 |
| 安全性 | 高（双重验证） | 中（单一验证） |
| 访问范围 | 完整权限 | 仅评分 |
| 支持 Scope | PROJECT / ORGANIZATION | 仅 PROJECT |
| 典型用途 | SDK、完整集成 | 评分提交、受限集成 |

---

## 7. 安全机制

### 7.1 密钥生成安全

#### 随机性保证

```typescript
// 使用 Node.js crypto.randomUUID()
const uuid = randomUUID(); // 符合 RFC 4122 v4
```

| 特性 | 说明 |
|------|------|
| 熵源 | 系统加密随机数生成器 |
| 唯一性 | UUID v4 碰撞概率极低 |
| 可预测性 | 密码学安全，不可预测 |

#### 哈希安全

**Bcrypt**:
- 自动 Salt 生成
- 慢速哈希（防暴力破解）
- 轮数可配置（默认 11）

**SHA-256 + Salt**:
- 全局 Salt（环境变量 `SALT`）
- 双重 SHA-256（私钥 + Salt）
- 快速验证用，不直接暴露

### 7.2 传输安全

#### HTTPS 强制

```bash
# 生产环境必须使用 HTTPS
NEXTAUTH_URL=https://your-domain.com
```

**保护措施**:
- TLS 1.2+ 加密
- 防中间人攻击
- 证书验证

#### 请求头安全

```
Authorization: Basic {base64}
```

**Base64 编码**: 注意，Base64 不是加密，仅是编码。必须配合 HTTPS 使用。

### 7.3 存储安全

#### 数据库

| 字段 | 存储方式 | 安全性 |
|------|---------|-------|
| `publicKey` | 明文 | 可公开 |
| `hashedSecretKey` | Bcrypt 哈希 | 高安全性 |
| `fastHashedSecretKey` | SHA-256 哈希 | 中安全性（需 Salt） |
| `displaySecretKey` | 部分字符 | 低风险 |

**注意**: 即使数据库泄露，攻击者也无法还原原始私钥。

#### Redis 缓存

**缓存内容**: 密钥元数据 + 快速哈希

**安全措施**:
- 不缓存原始私钥
- 不缓存 Bcrypt 哈希
- TTL 自动过期
- 支持 Redis AUTH

### 7.4 密钥生命周期

#### 创建

- 生成时显示完整私钥
- 用户必须保存
- 无法再次查看

#### 使用

- 每次请求验证
- 记录最后使用时间（可选）
- 监控异常调用

#### 删除

- 立即从数据库删除
- 立即从 Redis 删除
- 不可恢复

#### 轮换（建议）

- 定期生成新密钥
- 删除旧密钥
- 更新客户端配置

---

## 8. 配置参数

### 8.1 密钥相关配置

| 环境变量 | 必需 | 默认值 | 说明 |
|---------|------|--------|------|
| `SALT` | ✓ | - | SHA-256 哈希盐值（32+ 字符） |
| `LANGFUSE_CACHE_API_KEY_ENABLED` | × | `false` | 是否启用 Redis 缓存 |
| `LANGFUSE_CACHE_API_KEY_TTL_SECONDS` | × | `3600` | 缓存 TTL（秒） |

### 8.2 Redis 配置

```bash
# Redis 连接
REDIS_CONNECTION_STRING=redis://localhost:6379
# 或
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_AUTH=password
```

### 8.3 生成 SALT

```bash
# 推荐方式：使用 openssl
openssl rand -hex 32

# 或使用 Node.js
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

**要求**:
- 至少 32 字符
- 高熵值
- 生产环境保密
- 不可轻易变更（会导致所有密钥失效）

---

## 9. 性能优化

### 9.1 验证性能对比

| 验证方式 | 平均耗时 | 说明 |
|---------|---------|------|
| SHA-256（Redis 缓存命中） | 2-5ms | 最快路径 |
| SHA-256（数据库查询） | 20-50ms | 缓存未命中 |
| Bcrypt 验证（首次） | 200-300ms | 降级路径 |
| Bcrypt + 哈希迁移 | 200-350ms | 一次性成本 |

### 9.2 缓存优化

#### 缓存策略

```
请求 1: 缓存未命中 → 数据库查询 (50ms) → 写入缓存
请求 2-N: 缓存命中 → 直接返回 (2ms)
```

**收益**:
```
无缓存：1000 请求 = 50,000ms = 50 秒
有缓存：1000 请求 = 50ms + 999×2ms = 2,048ms ≈ 2 秒
性能提升：24x
```

#### 缓存预热

```typescript
// 应用启动时预加载热点密钥
async function warmupApiKeyCache() {
  const hotKeys = await prisma.apiKey.findMany({
    where: { /* 最近使用的密钥 */ },
    take: 100,
  });
  
  for (const key of hotKeys) {
    await addApiKeyToRedis(key.fastHashedSecretKey, key);
  }
}
```

### 9.3 数据库优化

#### 索引策略

```sql
-- 公钥索引（唯一）
CREATE UNIQUE INDEX idx_api_key_public ON ApiKey(publicKey);

-- 快速哈希索引（唯一）
CREATE UNIQUE INDEX idx_api_key_fast_hash ON ApiKey(fastHashedSecretKey);

-- 项目密钥查询
CREATE INDEX idx_api_key_project ON ApiKey(projectId, scope);

-- 组织密钥查询
CREATE INDEX idx_api_key_org ON ApiKey(orgId, scope);
```

#### 查询优化

```typescript
// ✅ 推荐：使用索引查询
const key = await prisma.apiKey.findUnique({
  where: { fastHashedSecretKey: hash },
});

// ❌ 避免：全表扫描
const keys = await prisma.apiKey.findMany({
  where: { note: { contains: "production" } },
});
```

---

## 10. 最佳实践

### 10.1 密钥管理原则

#### 创建密钥

- ✅ 为每个环境创建独立密钥（开发/测试/生产）
- ✅ 为每个应用/服务创建独立密钥
- ✅ 使用有意义的备注（`note` 字段）
- ❌ 不要共享密钥
- ❌ 不要在代码中硬编码密钥

#### 存储密钥

```bash
# ✅ 推荐：环境变量
LANGFUSE_PUBLIC_KEY=pk-lf-xxx
LANGFUSE_SECRET_KEY=sk-lf-yyy

# ✅ 推荐：密钥管理服务
# AWS Secrets Manager
# Azure Key Vault
# HashiCorp Vault

# ❌ 避免：代码中硬编码
const apiKey = "sk-lf-xxx"; // 不要这样做！
```

#### 轮换密钥

**建议周期**:
```
开发环境: 按需
测试环境: 每 90 天
生产环境: 每 90 天或发生泄露时立即轮换
```

**轮换步骤**:
1. 创建新密钥
2. 更新应用配置（支持双密钥并行）
3. 验证新密钥工作正常
4. 删除旧密钥

### 10.2 安全检查清单

- [ ] 生成强随机 `SALT`（32+ 字符）
- [ ] `SALT` 存储在安全位置（密钥管理服务）
- [ ] 生产环境强制 HTTPS
- [ ] 密钥不在代码仓库中
- [ ] 密钥不在日志中打印
- [ ] 定期审计密钥使用情况
- [ ] 删除未使用的密钥
- [ ] 监控异常 API 调用
- [ ] 设置速率限制
- [ ] 记录密钥创建和删除操作

### 10.3 故障排查

#### 密钥验证失败

**症状**: API 返回 401 Unauthorized

**排查步骤**:

1. **检查密钥格式**
```bash
# 公钥格式
pk-lf-{uuid}

# 私钥格式
sk-lf-{uuid}
```

2. **检查认证头**
```bash
# Basic Auth
Authorization: Basic {base64(pk:sk)}

# Bearer Auth
Authorization: Bearer pk-lf-xxx
```

3. **验证密钥存在**
```sql
SELECT publicKey, displaySecretKey, scope 
FROM ApiKey 
WHERE publicKey = 'pk-lf-xxx';
```

4. **检查 SALT 配置**
```bash
echo $SALT
# 必须与创建密钥时的 SALT 一致
```

5. **测试 Redis 连接**
```bash
redis-cli PING
# 应返回 PONG
```

#### 性能问题

**症状**: API 响应缓慢

**排查步骤**:

1. **检查缓存命中率**
```typescript
// 查看监控指标
langfuse.api_key.cache_hit
langfuse.api_key.cache_miss
```

2. **Redis 健康检查**
```bash
redis-cli INFO stats
```

3. **数据库慢查询**
```sql
-- 检查索引是否存在
SHOW INDEXES FROM ApiKey;
```

### 10.4 监控和告警

#### 关键指标

| 指标 | 描述 | 告警阈值 |
|------|------|---------|
| 验证失败率 | 无效密钥尝试 | >5% |
| 验证延迟 | 平均验证时间 | >100ms |
| 缓存命中率 | Redis 缓存命中率 | <80% |
| 密钥创建量 | 每日新建密钥数 | 异常增长 |
| 密钥删除量 | 每日删除密钥数 | 批量删除 |

#### 日志示例

```typescript
logger.info("API key created", {
  publicKey: key.publicKey,
  scope: key.scope,
  projectId: key.projectId,
  createdBy: user.id,
});

logger.warn("API key verification failed", {
  publicKey: publicKey,
  reason: "Invalid credentials",
  ip: request.ip,
});
```

---

## 相关文档

- [模块 README](./README.md)
- [认证机制详解](./authentication-mechanism.md)
- [RBAC 权限体系](./rbac-system.md)
- [API 密钥认证时序图](./02-api-key-auth-sequence.puml)
