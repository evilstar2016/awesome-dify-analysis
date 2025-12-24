# Cache Service 管理

## 功能概述

Cache Service 是 Prompts 模块的**高性能缓存层**，基于 Redis 实现提示词的快速查询。通过多级缓存键设计、自动失效机制、锁定保护等策略，将提示词查询延迟从 ~50ms（数据库）降低到 ~5ms（缓存）。核心特性：
- **分层缓存键**：`projectId:promptName:version/label` 结构
- **自动过期**：TTL 默认 1 小时（可配置）
- **锁定机制**：更新期间防止读取脏数据
- **批量失效**：通过键索引快速清除相关缓存
- **指标监控**：缓存命中率、失效次数统计

---

## 核心组件

### 1. **PromptService 服务类**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L20-L440)

#### **类结构（421 行）**

```typescript
// Lines 20-440: PromptService 类（421 行）
export class PromptService {
  // 属性（Lines 21-22）
  cacheEnabled: boolean;
  ttlSeconds: number = 60 * 60;  // 1 小时

  // 依赖（Lines 26-27）
  private prisma: PrismaClient;
  private redis?: Redis;

  // 构造函数（Lines 24-42）
  constructor(
    prisma: PrismaClient,
    redis?: Redis,
  ) {
    this.prisma = prisma;
    this.redis = redis;
    this.cacheEnabled = Boolean(redis && env.LANGFUSE_CACHE_PROMPT_ENABLED);
    
    if (env.LANGFUSE_CACHE_PROMPT_TTL_SECONDS) {
      this.ttlSeconds = parseInt(env.LANGFUSE_CACHE_PROMPT_TTL_SECONDS);
    }
  }

  // 核心方法（Lines 44-440）
  // - getPrompt: 获取提示词（优先从缓存）
  // - getCachedPrompt: 从缓存读取
  // - cachePrompt: 写入缓存
  // - invalidateCache: 失效缓存
  // - lockCache / unlockCache: 缓存锁定
  // - buildAndResolvePromptGraph: 依赖图解析
}
```

**属性说明**：

| 属性 | 类型 | 默认值 | 作用 |
|-----|------|-------|------|
| **cacheEnabled** | `boolean` | `redis && env.LANGFUSE_CACHE_PROMPT_ENABLED` | 是否启用缓存 |
| **ttlSeconds** | `number` | `3600` (1小时) | 缓存过期时间 |
| **prisma** | `PrismaClient` | - | 数据库客户端 |
| **redis** | `Redis?` | - | Redis 客户端（可选） |

---

### 2. **getPrompt 主查询方法**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L44-L72)

#### **方法实现（29 行）**

```typescript
// Lines 44-72: getPrompt 方法（29 行）
public async getPrompt(params: PromptParams): Promise<PromptResult | null> {
  // 步骤 1: 检查是否使用缓存（Lines 45-55）
  if (await this.shouldUseCache(params)) {
    const cachedPrompt = await this.getCachedPrompt(params);

    this.incrementMetric(
      cachedPrompt
        ? PromptServiceMetrics.PromptCacheHit
        : PromptServiceMetrics.PromptCacheMiss,
    );

    if (cachedPrompt) {
      this.logDebug("Returning cached prompt for params", params);
      return cachedPrompt;
    }
  }

  // 步骤 2: 从数据库查询（Lines 57）
  const dbPrompt = await this.getDbPrompt(params);

  // 步骤 3: 写入缓存（Lines 59-63）
  if ((await this.shouldUseCache(params)) && dbPrompt) {
    await this.cachePrompt({ ...params, prompt: dbPrompt });
    this.logDebug("Successfully cached prompt for params", params);
  }

  this.logDebug("Returning DB prompt for params", params);
  return dbPrompt;
}
```

**工作流**：

| 步骤 | 操作 | 代码行 | 耗时 |
|-----|------|-------|------|
| 1 | 检查缓存开关 | 45 | < 1ms |
| 2 | 读取缓存 | 46 | ~5ms (Redis) |
| 3 | 记录缓存命中指标 | 48-51 | < 1ms |
| 4 | 返回缓存结果 | 53-55 | - |
| 5 | 查询数据库 | 57 | ~50ms (Postgres) |
| 6 | 写入缓存 | 59-63 | ~5ms (Redis) |
| 7 | 返回数据库结果 | 65-66 | - |

**性能对比**：
- **缓存命中**：~5-10ms
- **缓存未命中**：~50-60ms（数据库查询 + 缓存写入）
- **缓存禁用**：~50ms（仅数据库查询）

---

### 3. **getCachedPrompt 缓存读取**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L141-L154)

#### **方法实现（14 行）**

```typescript
// Lines 141-154: getCachedPrompt 方法（14 行）
private async getCachedPrompt(
  params: PromptParams,
): Promise<PromptResult | null> {
  try {
    // 1. 构建缓存键（Lines 145）
    const key = this.getCacheKey(params);
    
    // 2. 读取并续期（Lines 146）
    const value = await this.redis?.getex(key, "EX", this.ttlSeconds);

    // 3. 解析 JSON（Lines 148）
    if (value) return JSON.parse(value) as PromptResult;
  } catch (e) {
    this.logError("Error getting cached prompt", e);
  }

  return null;
}
```

**关键点**：
- 使用 `GETEX` 命令（Get + Expire）：读取的同时续期 TTL
- 每次读取都会重置过期时间（活跃缓存保持热度）
- 出错时返回 `null`（回退到数据库查询）

---

### 4. **cachePrompt 缓存写入**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L156-L167)

#### **方法实现（12 行）**

```typescript
// Lines 156-167: cachePrompt 方法（12 行）
private async cachePrompt(params: PromptParams & { prompt: PromptResult }) {
  try {
    // 1. 获取键索引键（Lines 158）
    const keyIndexKey = this.getKeyIndexKey(params);
    
    // 2. 构建缓存键（Lines 159）
    const key = this.getCacheKey(params);
    
    // 3. 序列化数据（Lines 160）
    const value = JSON.stringify(params.prompt);

    // 4. 添加到键索引（Lines 162）
    await this.redis?.sadd(keyIndexKey, key);
    
    // 5. 写入缓存（Lines 163）
    await this.redis?.set(key, value, "EX", this.ttlSeconds);
  } catch (e) {
    this.logError("Error caching prompt", e);
  }
}
```

**两步写入**：
1. **键索引**：将缓存键添加到 Set（用于批量失效）
2. **实际缓存**：存储 JSON 数据，设置过期时间

---

### 5. **缓存键设计**

#### **getCacheKey 生成缓存键**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L256-L260)

```typescript
// Lines 256-260: getCacheKey 方法（5 行）
private getCacheKey(params: PromptParams): string {
  const prefix = this.getCacheKeyPrefix(params);
  return `${prefix}:${params.version ?? params.label}`;
}
```

#### **getCacheKeyPrefix 生成键前缀**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L262-L266)

```typescript
// Lines 262-266: getCacheKeyPrefix 方法（5 行）
private getCacheKeyPrefix(params: PromptParams): string {
  return `prompts:${params.projectId}:${params.promptName}`;
}
```

#### **getKeyIndexKey 生成键索引键**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L268-L272)

```typescript
// Lines 268-272: getKeyIndexKey 方法（5 行）
private getKeyIndexKey(params: PromptParams): string {
  return `prompts:keys:${params.projectId}`;
}
```

#### **缓存键结构示例**

| 查询参数 | 缓存键 | 键索引键 |
|---------|-------|---------|
| `projectId=proj_123, promptName=greeting, version=2` | `prompts:proj_123:greeting:2` | `prompts:keys:proj_123` |
| `projectId=proj_123, promptName=greeting, label=production` | `prompts:proj_123:greeting:production` | `prompts:keys:proj_123` |
| `projectId=proj_123, promptName=chat, label=latest` | `prompts:proj_123:chat:latest` | `prompts:keys:proj_123` |

**设计优势**：
- **层级清晰**：`prompts:projectId:promptName:version/label`
- **高效失效**：通过 `projectId` 批量删除
- **版本/标签混合**：同一键格式支持两种查询方式

---

### 6. **shouldUseCache 缓存开关**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L129-L139)

#### **方法实现（11 行）**

```typescript
// Lines 129-139: shouldUseCache 方法（11 行）
private async shouldUseCache(params: PromptParams): Promise<boolean> {
  // 1. 全局开关检查（Lines 130）
  if (!this.cacheEnabled) return false;

  // 2. 锁定状态检查（Lines 132）
  const isLocked = await this.isCacheLocked(params);

  if (isLocked) {
    this.logInfo("Cache is locked for params", params);
  }

  // 3. 返回结果（Lines 138）
  return !isLocked;
}
```

**两级检查**：
1. **全局开关**：`cacheEnabled` (环境变量控制)
2. **锁定状态**：更新期间不使用缓存

---

### 7. **锁定机制**

#### **lockCache 锁定缓存**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L175-L189)

```typescript
// Lines 175-189: lockCache 方法（15 行）
public async lockCache(
  params: Pick<PromptParams, "projectId" | "promptName">,
): Promise<void> {
  if (!this.cacheEnabled) return;

  // 1. 获取锁键（Lines 180）
  const lockKey = this.getLockKey(params);

  try {
    // 2. 设置锁（30 秒过期）（Lines 183）
    await this.redis?.setex(lockKey, 30, "locked");
  } catch (e) {
    this.logError("Error locking cache key prefix", lockKey, e);
    throw e;
  }
}
```

#### **getLockKey 生成锁键**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L221-L226)

```typescript
// Lines 221-226: getLockKey 方法（6 行）
private getLockKey(
  params: Pick<PromptParams, "projectId" | "promptName">,
): string {
  return `prompts:lock:${params.projectId}:${params.promptName}`;
}
```

**锁键示例**：
```
prompts:lock:proj_123:greeting
```

#### **unlockCache 解锁缓存**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L191-L205)

```typescript
// Lines 191-205: unlockCache 方法（15 行）
public async unlockCache(
  params: Pick<PromptParams, "projectId" | "promptName">,
): Promise<void> {
  if (!this.cacheEnabled) return;

  const lockKey = this.getLockKey(params);

  try {
    await this.redis?.del(lockKey);
  } catch (e) {
    this.logError("Error unlocking cache key prefix", lockKey, e);
    throw e;
  }
}
```

#### **isCacheLocked 检查锁定状态**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L207-L219)

```typescript
// Lines 207-219: isCacheLocked 方法（13 行）
private async isCacheLocked(
  params: Pick<PromptParams, "projectId" | "promptName">,
): Promise<boolean> {
  try {
    const lockKey = this.getLockKey(params);
    const value = await this.redis?.get(lockKey);

    return Boolean(value);
  } catch (e) {
    this.logError("Error checking cache lock", e);
    return false;  // 出错时假设未锁定
  }
}
```

---

### 8. **invalidateCache 批量失效**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L228-L254)

#### **方法实现（27 行）**

```typescript
// Lines 228-254: invalidateCache 方法（27 行）
public async invalidateCache(
  params: Pick<PromptParams, "projectId" | "promptName">,
): Promise<void> {
  if (!this.cacheEnabled) return;

  // 1. 获取新键索引（Lines 233-234）
  const keyIndexKey = this.getKeyIndexKey(params);
  const keys = await this.redis?.smembers(keyIndexKey);

  // 2. 向后兼容：处理旧键索引（Lines 236-245）
  /*
   * 之前的键索引格式：projectId + promptName
   * 现在的格式：仅 projectId（支持批量失效）
   * 为了兼容，同时删除旧格式的键索引
   */
  const legacyKeyIndexKey = `${keyIndexKey}:${params.promptName}`;
  const legacyKeys = await this.redis?.smembers(legacyKeyIndexKey);

  // 3. 批量删除（Lines 247-252）
  const keysToDelete = [
    ...(keys ?? []),
    keyIndexKey,
    ...(legacyKeys ?? []),
    legacyKeyIndexKey,
  ];
  await safeMultiDel(this.redis, keysToDelete);
}
```

**失效流程**：

| 步骤 | 操作 | 删除的键 |
|-----|------|---------|
| 1 | 读取键索引 | - |
| 2 | 收集所有缓存键 | `prompts:proj_123:greeting:*` |
| 3 | 收集旧格式键 | `prompts:keys:proj_123:greeting` |
| 4 | 批量删除 | 所有缓存键 + 键索引 |

**向后兼容说明**：
- **旧版本**：`prompts:keys:projectId:promptName` （每个提示词独立索引）
- **新版本**：`prompts:keys:projectId` （项目级索引，支持依赖失效）
- **迁移期**：同时删除两种格式

---

## 缓存生命周期

### **完整流程示例**

#### **场景 1：首次查询（缓存未命中）**

```typescript
// 1. 调用 getPrompt
const prompt = await promptService.getPrompt({
  projectId: "proj_123",
  promptName: "greeting",
  label: "production",
});

// 内部流程：
// ① shouldUseCache() → true（缓存开启，无锁定）
// ② getCachedPrompt() → null（首次查询，缓存为空）
// ③ incrementMetric(PromptCacheMiss) → 记录缓存未命中
// ④ getDbPrompt() → 从数据库查询（~50ms）
// ⑤ cachePrompt() → 写入缓存
//    - sadd("prompts:keys:proj_123", "prompts:proj_123:greeting:production")
//    - set("prompts:proj_123:greeting:production", JSON.stringify(prompt), "EX", 3600)
// ⑥ 返回结果
```

**耗时**：~55ms（数据库 + 缓存写入）

---

#### **场景 2：后续查询（缓存命中）**

```typescript
// 2. 再次调用 getPrompt（相同参数）
const prompt = await promptService.getPrompt({
  projectId: "proj_123",
  promptName: "greeting",
  label: "production",
});

// 内部流程：
// ① shouldUseCache() → true
// ② getCachedPrompt()
//    - getCacheKey() → "prompts:proj_123:greeting:production"
//    - redis.getex("prompts:proj_123:greeting:production", "EX", 3600) → 返回数据并续期
// ③ incrementMetric(PromptCacheHit) → 记录缓存命中
// ④ 直接返回缓存结果
```

**耗时**：~5ms（仅 Redis 查询）

---

#### **场景 3：版本更新（锁定 + 失效）**

```typescript
// 3. 创建新版本
await promptService.lockCache({ projectId: "proj_123", promptName: "greeting" });
// 锁键：prompts:lock:proj_123:greeting（30秒过期）

await promptService.invalidateCache({ projectId: "proj_123", promptName: "greeting" });
// 删除键：
// - prompts:proj_123:greeting:1
// - prompts:proj_123:greeting:2
// - prompts:proj_123:greeting:production
// - prompts:proj_123:greeting:latest
// - prompts:keys:proj_123

await prisma.$transaction([/* 创建版本 */]);

await promptService.unlockCache({ projectId: "proj_123", promptName: "greeting" });
// 删除锁键：prompts:lock:proj_123:greeting

// 后续查询会回到场景 1（缓存未命中）
```

---

#### **场景 4：锁定期间查询**

```typescript
// 4. 在锁定期间调用 getPrompt
const prompt = await promptService.getPrompt({
  projectId: "proj_123",
  promptName: "greeting",
  label: "production",
});

// 内部流程：
// ① shouldUseCache()
//    - cacheEnabled → true
//    - isCacheLocked() → true（发现锁键存在）
//    - 返回 false
// ② 跳过缓存，直接查询数据库
// ③ 不写入缓存（因为 shouldUseCache 返回 false）
// ④ 返回数据库结果
```

**保护机制**：
- 更新期间不读取缓存（防止读到不一致的数据）
- 更新期间不写入缓存（防止覆盖正在失效的数据）

---

## 性能优化

### **缓存命中率优化**

| 策略 | 实现 | 效果 |
|-----|------|------|
| **活跃续期** | `GETEX` 命令 | 热门提示词保持缓存 |
| **合理 TTL** | 默认 1 小时 | 平衡内存和命中率 |
| **批量预热** | 启动时加载常用提示词 | 提升初始命中率 |

### **键索引优化**

**旧设计（每个提示词独立索引）**：
```
prompts:keys:proj_123:greeting → Set { "key1", "key2" }
prompts:keys:proj_123:chat → Set { "key3", "key4" }
```

**新设计（项目级索引）**：
```
prompts:keys:proj_123 → Set { "key1", "key2", "key3", "key4" }
```

**优势**：
- **批量失效**：删除 `proj_123` 的所有缓存只需一次 `SMEMBERS`
- **依赖处理**：提示词依赖其他提示词时，可以一次性失效所有相关缓存
- **内存节省**：减少索引键数量

---

### **批量删除优化**

**位置**：使用 `safeMultiDel` 工具函数

```typescript
// 安全批量删除（分批处理，避免阻塞）
await safeMultiDel(this.redis, keysToDelete);

// 内部实现（假设）：
// 1. 将键列表分批（每批 100 个）
// 2. 使用 pipeline 批量删除
// 3. 避免单次删除大量键导致 Redis 阻塞
```

---

## 监控指标

### **PromptServiceMetrics 指标枚举**

```typescript
enum PromptServiceMetrics {
  PromptCacheHit = "langfuse.prompt_cache.hit",
  PromptCacheMiss = "langfuse.prompt_cache.miss",
}
```

### **incrementMetric 指标记录**

**位置**：[packages/shared/src/server/services/PromptService/index.ts](packages/shared/src/server/services/PromptService/index.ts#L433-L439)

```typescript
// Lines 433-439: incrementMetric 方法（7 行）
private incrementMetric(metric: string): void {
  try {
    this.metricIncrementer?.(metric);
  } catch (e) {
    this.logError("Error incrementing metric", metric, e);
  }
}
```

### **关键指标**

| 指标 | 计算方式 | 用途 |
|-----|---------|------|
| **缓存命中率** | `hits / (hits + misses)` | 评估缓存效果 |
| **平均查询延迟** | `sum(latency) / count` | 监控性能 |
| **缓存大小** | `redis.dbsize()` | 监控内存使用 |
| **锁定次数** | `count(lockCache)` | 监控更新频率 |
| **失效次数** | `count(invalidateCache)` | 监控缓存失效 |

**告警阈值建议**：
- 命中率 < 80% → 检查 TTL 配置
- 平均延迟 > 50ms → 检查 Redis 性能
- 锁定次数 > 100/min → 版本更新过于频繁

---

## 配置管理

### **环境变量**

| 变量 | 类型 | 默认值 | 说明 |
|-----|------|-------|------|
| **LANGFUSE_CACHE_PROMPT_ENABLED** | `boolean` | `true` | 是否启用缓存 |
| **LANGFUSE_CACHE_PROMPT_TTL_SECONDS** | `number` | `3600` | 缓存过期时间（秒） |
| **REDIS_HOST** | `string` | `localhost` | Redis 主机 |
| **REDIS_PORT** | `number` | `6379` | Redis 端口 |
| **REDIS_AUTH** | `string` | - | Redis 密码 |

### **配置示例**

```bash
# 开发环境（短 TTL，快速验证）
LANGFUSE_CACHE_PROMPT_ENABLED=true
LANGFUSE_CACHE_PROMPT_TTL_SECONDS=300  # 5 分钟

# 生产环境（长 TTL，高命中率）
LANGFUSE_CACHE_PROMPT_ENABLED=true
LANGFUSE_CACHE_PROMPT_TTL_SECONDS=7200  # 2 小时

# 禁用缓存（调试）
LANGFUSE_CACHE_PROMPT_ENABLED=false
```

---

## 故障排查

### **常见问题**

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| **缓存未命中率高** | TTL 过短 | 增加 `LANGFUSE_CACHE_PROMPT_TTL_SECONDS` |
| **查询返回旧数据** | 缓存未失效 | 检查 `invalidateCache` 是否调用 |
| **Redis 内存溢出** | 缓存键过多 | 减少 TTL 或增加 Redis 内存 |
| **锁定超时** | 事务执行时间过长 | 优化事务逻辑或增加锁超时时间 |
| **缓存键泄漏** | 键索引未更新 | 检查 `cachePrompt` 是否正确调用 `sadd` |

### **调试工具**

#### **查看缓存内容**

```bash
# 查看所有提示词缓存键
redis-cli KEYS "prompts:*"

# 查看键索引
redis-cli SMEMBERS "prompts:keys:proj_123"

# 查看缓存内容
redis-cli GET "prompts:proj_123:greeting:production"

# 查看锁状态
redis-cli GET "prompts:lock:proj_123:greeting"
```

#### **手动失效缓存**

```bash
# 删除特定提示词的所有缓存
redis-cli DEL $(redis-cli KEYS "prompts:proj_123:greeting:*")

# 删除项目的所有缓存
redis-cli DEL $(redis-cli SMEMBERS "prompts:keys:proj_123")

# 删除所有提示词缓存
redis-cli DEL $(redis-cli KEYS "prompts:*")
```

---

## 使用示例

### **场景 1：启用缓存**

```typescript
import { PromptService } from "@langfuse/shared/src/server/services/PromptService";
import { prisma } from "@/lib/prisma";
import { redis } from "@/lib/redis";

const promptService = new PromptService(prisma, redis);

// 查询提示词（自动使用缓存）
const prompt = await promptService.getPrompt({
  projectId: "proj_123",
  promptName: "greeting",
  label: "production",
});
```

---

### **场景 2：禁用缓存**

```typescript
// 方式 1: 环境变量
process.env.LANGFUSE_CACHE_PROMPT_ENABLED = "false";

// 方式 2: 不传入 Redis 客户端
const promptService = new PromptService(prisma);  // redis = undefined

// 此时所有查询都会直接访问数据库
```

---

### **场景 3：手动失效缓存**

```typescript
// 更新提示词后失效缓存
await promptService.lockCache({ projectId: "proj_123", promptName: "greeting" });

await promptService.invalidateCache({ projectId: "proj_123", promptName: "greeting" });

await prisma.prompt.update({
  where: { id: "prompt_id" },
  data: { labels: ["production"] },
});

await promptService.unlockCache({ projectId: "proj_123", promptName: "greeting" });
```

---

### **场景 4：批量预热缓存**

```typescript
// 启动时预加载热门提示词
const hotPrompts = [
  { projectId: "proj_123", promptName: "greeting", label: "production" },
  { projectId: "proj_123", promptName: "chat", label: "production" },
  { projectId: "proj_123", promptName: "summary", label: "production" },
];

await Promise.all(
  hotPrompts.map((params) => promptService.getPrompt(params))
);
// 所有热门提示词已加载到缓存
```

---

## 相关文档

- [Version Management 管理](version-management.md) - 版本控制机制
- [Label System 管理](label-system.md) - 标签管理机制
- [02-prompt-fetch-sequence.puml](02-prompt-fetch-sequence.puml) - 获取时序图
