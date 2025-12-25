# Dify 账号权限管理 - 实现细节补充

## 目录
1. [令牌管理详解](#令牌管理详解)
2. [数据库约束](#数据库约束)
3. [缓存策略](#缓存策略)
4. [错误处理](#错误处理)
5. [集成点](#集成点)
6. [扩展性](#扩展性)

---

## 令牌管理详解

### 1. JWT Access Token（访问令牌）

#### 生成流程
```python
def get_account_jwt_token(account: Account) -> str:
    payload = {
        'account_id': account.id,
        'current_tenant_id': account.current_tenant_id,
        'role': account.role,
        'iat': current_timestamp,
        'exp': current_timestamp + token_expire_time
    }
    token = PassportService().issue(payload)
    return token
```

#### 包含信息
| 字段 | 说明 | 用途 |
|------|------|------|
| `account_id` | 账号 UUID | 标识请求用户 |
| `current_tenant_id` | 当前工作区 ID | 标识请求上下文 |
| `role` | 用户在工作区的角色 | 权限检查 |
| `iat` | 签发时间 | 令牌有效性检查 |
| `exp` | 过期时间 | 令牌生命周期 |

#### 验证机制
```python
# 每个需要认证的 API 端点都使用 @login_required 装饰器
# 装饰器执行以下步骤：
1. 从 Cookie 或 Header 提取 JWT token
2. 验证签名有效性
3. 验证是否过期
4. 验证账号状态（非 BANNED/CLOSED）
5. 加载 Account 和 Tenant 对象
6. 注入到请求上下文 (current_user)
```

---

### 2. Refresh Token（刷新令牌）

#### 特点
- **随机生成**：32 字节随机值，无结构化数据
- **服务端存储**：Redis 中存储映射关系
- **长期有效**：默认 30 天有效期
- **关键交换**：用于获取新的 access_token

#### 存储格式
```python
Redis Key: "refresh_token:{token_hash}"
Redis Value: account_id
Redis TTL: REFRESH_TOKEN_EXPIRE_DAYS (30 days)

# 同时维护反向映射用于登出
Redis Key: "account_refresh_token:{account_id}"
Redis Value: refresh_token
Redis TTL: REFRESH_TOKEN_EXPIRE_DAYS
```

#### 刷新流程
```python
def refresh_token(refresh_token: str) -> TokenPair:
    # 1. 验证 refresh_token 有效性
    account_id = redis_client.get(f"refresh_token:{hash(refresh_token)}")
    if not account_id:
        raise ValueError("Invalid refresh token")
    
    # 2. 加载账号
    account = AccountService.load_user(account_id)
    if not account:
        raise ValueError("Invalid account")
    
    # 3. 生成新的 access_token
    new_access_token = AccountService.get_account_jwt_token(account)
    
    # 4. 生成新的 refresh_token
    new_refresh_token = _generate_refresh_token()
    
    # 5. 更新 Redis 中的映射
    AccountService._delete_refresh_token(refresh_token, account.id)
    AccountService._store_refresh_token(new_refresh_token, account.id)
    
    # 6. 生成新的 CSRF token
    csrf_token = generate_csrf_token(account.id)
    
    return TokenPair(
        access_token=new_access_token,
        refresh_token=new_refresh_token,
        csrf_token=csrf_token
    )
```

---

### 3. CSRF Token（跨站请求伪造防护）

#### 实现方式
```python
def generate_csrf_token(account_id: str) -> str:
    # 基于 account_id 生成 CSRF token
    # 存储在 Cookie 中（HttpOnly=False，客户端可读）
    payload = {
        'account_id': account_id,
        'timestamp': current_timestamp
    }
    return sign_and_encode(payload)
```

#### 验证规则
- 所有状态改变请求（POST, PUT, DELETE）必须包含有效的 CSRF token
- Token 来源：
  - Cookie 中的 `csrf_token`
  - 或请求头中的 `X-CSRF-Token`
  - 或请求体中的 `csrf_token` 字段
- 验证失败返回 403 Forbidden

---

## 数据库约束

### Account 表
```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    password VARCHAR(255),
    password_salt VARCHAR(255),
    avatar VARCHAR(255),
    interface_language VARCHAR(255),
    interface_theme VARCHAR(255),
    timezone VARCHAR(255),
    last_login_at TIMESTAMP,
    last_login_ip VARCHAR(255),
    last_active_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(16) DEFAULT 'active',
    initialized_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX account_email_idx (email)
);
```

**关键约束**：
- `email` 唯一性：防止重复注册
- `status` 默认值：'active'
- `last_active_at` 自动更新：用于账号活跃度追踪

---

### TenantAccountJoin 表
```sql
CREATE TABLE tenant_account_joins (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    account_id UUID NOT NULL,
    current BOOLEAN DEFAULT false,
    role VARCHAR(16) DEFAULT 'normal',
    invited_by UUID,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(tenant_id, account_id),
    INDEX tenant_account_join_account_id_idx (account_id),
    INDEX tenant_account_join_tenant_id_idx (tenant_id)
);
```

**关键设计**：
- `(tenant_id, account_id)` 组合唯一：确保一个账号在一个工作区内只有一条记录
- `current` 字段：标记账号当前选中的工作区
- `invited_by` 字段：追踪邀请来源
- 索引优化：快速查询账号所属工作区和工作区成员

---

### AccountIntegrate 表
```sql
CREATE TABLE account_integrates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    account_id UUID NOT NULL,
    provider VARCHAR(16) NOT NULL,
    open_id VARCHAR(255) NOT NULL,
    encrypted_token VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(account_id, provider),
    UNIQUE(provider, open_id)
);
```

**关键约束**：
- `(account_id, provider)` 唯一：一个账号只能关联一个 OAuth 提供商
- `(provider, open_id)` 唯一：防止 OAuth ID 复用
- 支持多个 OAuth 提供商集成

---

## 缓存策略

### Redis 缓存结构

| 键前缀 | 键格式 | 值 | TTL | 用途 |
|--------|--------|-----|-----|------|
| `refresh_token:` | `refresh_token:{token_hash}` | `account_id` | 30d | 刷新令牌验证 |
| `account_refresh_token:` | `account_refresh_token:{account_id}` | `refresh_token` | 30d | 登出清除 |
| `invitation_token:` | `invitation_token:{token_hash}` | JSON | 24h | 邀请码验证 |
| `owner_transfer_token:` | `owner_transfer_token:{token_hash}` | JSON | 1h | 所有权转移验证 |
| `login_error_limit:` | `login_error_limit:{email}` | `count` | 1h | 登录失败限制 |
| `reset_password_limit:` | `reset_password_limit:{email}` | `count` | 1m | 密码重置限制 |
| `rate_limit:` | `rate_limit:{tenant_id}` | sorted_set | 1m | 知识库速率限制 |

### 缓存策略

#### 读写模式
```
新令牌生成 → Redis 存储 → 
  ↓
令牌验证 → Redis 查询 → 
  ↓
令牌刷新 → 删除旧令牌 + 存储新令牌 →
  ↓
登出 → 删除所有相关令牌
```

#### 容错机制
```python
# Redis 连接失败时的降级方案
from extensions.ext_redis import redis_fallback

try:
    token = redis_client.get(key)
except RedisException:
    # 使用本地内存缓存或直接验证（开销较大）
    token = redis_fallback.get(key)
```

---

## 错误处理

### 异常体系

#### 认证异常
```python
class AccountPasswordError(Exception):
    """密码错误或账号不存在"""
    pass

class AccountLoginError(Exception):
    """账号被禁用或其他登录失败"""
    pass

class InvalidTokenError(Exception):
    """令牌无效或过期"""
    pass
```

#### 权限异常
```python
class NoPermissionError(Exception):
    """操作者权限不足"""
    pass

class CannotOperateSelfError(Exception):
    """不能对自己执行操作"""
    pass

class MemberNotInTenantError(Exception):
    """成员不在工作区"""
    pass
```

#### 工作区异常
```python
class TenantNotFoundError(Exception):
    """工作区不存在"""
    pass

class AccountNotLinkTenantError(Exception):
    """账号未链接到工作区"""
    pass
```

### 错误映射

| 异常 | HTTP 状态码 | 响应格式 |
|------|------------|---------|
| `NoPermissionError` | 403 | `{code: "forbidden", message: "..."}` |
| `CannotOperateSelfError` | 400 | `{code: "cannot-operate-self", message: "..."}` |
| `MemberNotInTenantError` | 404 | `{code: "member-not-found", message: "..."}` |
| `InvalidTokenError` | 400 | `{code: "invalid-token", message: "..."}` |
| `AccountPasswordError` | 401 | `{code: "auth-failed", message: "..."}` |
| `RoleAlreadyAssignedError` | 400 | `{code: "error", message: "..."}` |

---

## 集成点

### 1. 邮件系统集成

#### 集成方式
```python
# 使用 Celery 任务队列发送邮件
from tasks.mail_invite_member_task import send_invite_member_mail_task

send_invite_member_mail_task.delay(
    to=email,
    token=invitation_token,
    workspace_name=workspace_name,
    language=language
)
```

#### 邮件类型
| 邮件类型 | 触发场景 | 任务函数 |
|---------|---------|---------|
| 邀请邮件 | 邀请成员 | `send_invite_member_mail_task` |
| 密码重置 | 重置密码 | `send_reset_password_mail_task` |
| 所有权转移 | 转移所有权 | `send_owner_transfer_confirm_task` |
| 账号删除确认 | 删除账号 | `send_account_deletion_verification_code` |

---

### 2. 计费系统集成

#### 集成点
```python
# 邀请成员前检查计费限额
@cloud_edition_billing_resource_check("members")
def post(self):
    # 在这个装饰器中检查成员数量限制
    pass

# 成员数量变化时清除缓存
if dify_config.BILLING_ENABLED:
    BillingService.clean_billing_info_cache(tenant.id)
```

#### 影响功能
- 成员邀请数量限制
- 应用创建数量限制
- 存储空间限制
- 知识库请求速率限制

---

### 3. OAuth 集成

#### 支持的 OAuth 提供商
- Google OAuth 2.0
- GitHub OAuth
- 其他自定义 OAuth 2.0 提供商

#### 集成流程
```python
# 1. 获取 OAuth 认证信息
provider, open_id = oauth_service.get_auth_info(...)

# 2. 链接账号与 OAuth 提供商
AccountService.link_account_integrate(provider, open_id, account)

# 3. 存储加密的访问令牌
account_integrate.encrypted_token = encrypt_token(token)
```

---

### 4. 审计日志（可选扩展）

#### 建议的审计事件
```python
# 账号相关
- account_created
- account_login
- account_logout
- account_password_changed
- account_email_changed

# 权限相关
- member_invited
- member_removed
- member_role_updated
- owner_transferred

# 工作区相关
- workspace_created
- workspace_updated
- workspace_archived
```

#### 实现示例
```python
def update_member_role(tenant, member, new_role, operator):
    # ... 权限检查和更新逻辑 ...
    
    # 记录审计日志
    AuditService.log(
        event='member_role_updated',
        tenant_id=tenant.id,
        operator_id=operator.id,
        target_id=member.id,
        details={'old_role': old_role, 'new_role': new_role},
        timestamp=now()
    )
```

---

## 扩展性

### 1. 添加新角色

#### 步骤
```python
# 1. 修改 TenantAccountRole 枚举
class TenantAccountRole(enum.StrEnum):
    NEW_ROLE = "new_role"
    
    @staticmethod
    def is_valid_role(role: str) -> bool:
        return role in {
            ..., TenantAccountRole.NEW_ROLE
        }

# 2. 更新权限矩阵
# 在 TenantService.check_member_permission() 中添加权限检查
perms = {
    "action": [TenantAccountRole.OWNER, TenantAccountRole.NEW_ROLE],
}

# 3. 更新前端权限映射
# 在相关 API 文档中更新权限表

# 4. 数据迁移
# 添加 Alembic 迁移脚本确保数据库兼容
```

---

### 2. 添加新的权限检查

#### 模式
```python
# 在装饰器中添加新的检查
def new_permission_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        from libs.login import current_user
        
        if not current_user.has_new_permission:
            raise Forbidden()
        return f(*args, **kwargs)
    
    return decorated_function

# 在 Account 模型中添加属性
class Account(UserMixin, TypeBase):
    @property
    def has_new_permission(self):
        return TenantAccountRole.is_new_permission_role(self.role)
```

---

### 3. 扩展 OAuth 提供商

#### 步骤
```python
# 1. 在 OAuth 配置中注册新提供商
OAUTH_PROVIDERS = {
    'new_provider': {
        'client_id': '...',
        'client_secret': '...',
        'authorize_url': 'https://...',
        'token_url': 'https://...',
    }
}

# 2. 实现 OAuth 流程
class NewProviderOAuth:
    def authorize(self, code):
        # 交换访问令牌
        pass
    
    def get_user_info(self, access_token):
        # 获取用户信息
        pass

# 3. 集成到认证流程
auth_service.link_with_oauth('new_provider', ...)
```

---

### 4. 添加自定义权限策略

#### 示例：基于时间的权限
```python
class TimeBasedPermission:
    """基于时间的权限策略"""
    
    def can_perform(self, user, action, tenant):
        # 检查业务时间
        if not is_business_hours():
            return False
        
        # 检查配额
        if self.is_quota_exceeded(user, action):
            return False
        
        return True

# 集成到权限检查流程
def check_member_permission(tenant, operator, member, action):
    # 基本权限检查
    if not basic_permission_check():
        raise NoPermissionError()
    
    # 自定义权限检查
    if not TimeBasedPermission().can_perform(operator, action, tenant):
        raise TimeBasedPermissionError()
```

---

## 总结

Dify 的账号权限管理系统设计遵循以下原则：

1. **安全第一**：多层认证、加密存储、速率限制
2. **清晰分层**：模型、服务、控制器的职责分明
3. **可维护性**：使用装饰器、服务方法实现关注点分离
4. **可扩展性**：支持添加新角色、权限检查、OAuth 提供商
5. **容错性**：优雅的异常处理、降级方案、缓存策略
6. **可集成性**：集成邮件、计费、OAuth、审计等系统

这个架构为企业级应用提供了坚实的基础，同时保持了足够的灵活性以应对未来的需求变化。
