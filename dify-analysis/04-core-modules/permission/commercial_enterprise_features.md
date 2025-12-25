# Dify 权限系统 - 商业版/企业版功能详细分析

> 📅 生成日期：2025-12-19  
> 🔍 基于代码版本：main 分支  
> 📦 分析范围：账号权限管理模块中的商业化功能

---

## 📊 目录

1. [功能概述](#功能概述)
2. [云版本计费功能 (Cloud Edition Billing)](#云版本计费功能)
3. [企业版功能 (Enterprise Edition)](#企业版功能)
4. [许可证管理 (License Management)](#许可证管理)
5. [配置与环境变量](#配置与环境变量)
6. [代码实现详解](#代码实现详解)
7. [权限装饰器](#权限装饰器)
8. [限制与配额](#限制与配额)

---

## 功能概述

Dify 的权限系统支持三种部署模式：

| 部署模式 | 说明 | 标识变量 |
|----------|------|---------|
| **开源自托管版** | 完全免费，功能有限 | `EDITION=SELF_HOSTED` |
| **云版本 + 计费** | SaaS 模式，按订阅收费 | `BILLING_ENABLED=true` |
| **企业版** | 私有部署，许可证授权 | `ENTERPRISE_ENABLED=true` |

### 版本功能对比

| 功能类别 | 开源版 | 云版本 | 企业版 |
|---------|--------|--------|--------|
| **基础权限管理** | ✅ | ✅ | ✅ |
| **成员数量限制** | 无限制 | 按订阅计划 | 按许可证 |
| **工作区数量** | 无限制 | 无限制 | 按许可证 |
| **SSO 单点登录** | ❌ | ❌ | ✅ |
| **自定义品牌** | 部分 | 付费计划 | ✅ |
| **WebApp 访问控制** | ❌ | ❌ | ✅ |
| **知识库限流** | ❌ | ✅ | ✅ |
| **插件管理器** | ❌ | ❌ | ✅ |
| **账户冻结机制** | ❌ | ✅ | ❌ |

---

## 云版本计费功能

### 1. 计费系统架构

```python
# 配置位置: configs/dify_config.py
BILLING_ENABLED = os.getenv("BILLING_ENABLED", "false").lower() == "true"
BILLING_API_URL = os.getenv("BILLING_API_URL", "")
BILLING_API_SECRET_KEY = os.getenv("BILLING_API_SECRET_KEY", "")
```

#### 核心服务：`BillingService`

**代码位置**：`api/services/billing_service.py`

```python
class BillingService:
    """与计费 API 通信的服务"""
    
    base_url = os.environ.get("BILLING_API_URL")
    secret_key = os.environ.get("BILLING_API_SECRET_KEY")
    
    @classmethod
    def get_info(cls, tenant_id: str):
        """获取租户的计费信息"""
        params = {"tenant_id": tenant_id}
        billing_info = cls._send_request("GET", "/subscription/info", params=params)
        return billing_info
    
    @classmethod
    def get_subscription(cls, plan: str, interval: str, ...):
        """获取订阅支付链接"""
        params = {"plan": plan, "interval": interval, ...}
        return cls._send_request("GET", "/subscription/payment-link", params=params)
    
    @classmethod
    def is_email_in_freeze(cls, email: str) -> bool:
        """检查邮箱是否在冻结期（30天内删除的账户）"""
        params = {"email": email}
        response = cls._send_request("GET", "/account/in-freeze", params=params)
        return bool(response.get("data", False))
```

### 2. 订阅计划模型

**代码位置**：`api/services/feature_service.py`

```python
class SubscriptionModel(BaseModel):
    plan: str = "sandbox"  # 计划类型
    interval: str = ""     # 计费周期

class BillingModel(BaseModel):
    enabled: bool = False
    subscription: SubscriptionModel = SubscriptionModel()
```

**订阅计划类型**：
- `sandbox`：免费沙盒计划（功能受限）
- `professional`：专业版
- `team`：团队版
- `enterprise`：企业版（通过计费 API）

### 3. 计费相关限制

#### 3.1 成员数量限制

```python
# 代码位置: services/feature_service.py
class FeatureModel(BaseModel):
    members: LimitationModel = LimitationModel(size=0, limit=1)
    # size: 当前成员数
    # limit: 最大允许数（0=无限制）
```

**限制检查装饰器**：
```python
# 代码位置: controllers/console/wraps.py
@cloud_edition_billing_resource_check("members")
def add_member_api():
    """添加成员前检查配额"""
    # 如果 members.size >= members.limit，返回 403 错误
```

#### 3.2 应用数量限制

```python
class FeatureModel(BaseModel):
    apps: LimitationModel = LimitationModel(size=0, limit=10)
    # sandbox 计划默认限制 10 个应用
```

#### 3.3 向量空间限制

```python
class FeatureModel(BaseModel):
    vector_space: LimitationModel = LimitationModel(size=0, limit=5)
    # 限制知识库存储空间（单位：MB/GB）
```

#### 3.4 文档上传配额

```python
class FeatureModel(BaseModel):
    documents_upload_quota: LimitationModel = LimitationModel(size=0, limit=50)
    # 限制文档上传数量
```

#### 3.5 标注配额限制

```python
class FeatureModel(BaseModel):
    annotation_quota_limit: LimitationModel = LimitationModel(size=0, limit=10)
    # 限制标注数据条数
```

### 4. 知识库速率限制

**🔥 商业版特有功能**

```python
@cloud_edition_billing_rate_limit_check("knowledge")
def knowledge_api():
    """知识库 API 限流"""
    # 沙盒计划：10 次/分钟
    # 付费计划：根据订阅级别调整
```

**实现机制**：
```python
# 使用 Redis 存储请求记录
key = f"rate_limit_{tenant_id}"
redis_client.zadd(key, {current_time: current_time})
redis_client.zremrangebyscore(key, 0, current_time - 60000)  # 清理1分钟前的记录

request_count = redis_client.zcard(key)
if request_count > knowledge_rate_limit.limit:
    # 记录限流日志
    rate_limit_log = RateLimitLog(
        tenant_id=current_tenant_id,
        subscription_plan=knowledge_rate_limit.subscription_plan,
        operation="knowledge",
    )
    db.session.add(rate_limit_log)
    db.session.commit()
    abort(403, "Sorry, you have reached the knowledge base request rate limit...")
```

### 5. 账户冻结机制

**🔥 商业版特有功能**

防止用户通过删除账户后重新注册来绕过付费：

```python
# 注册时检查
if dify_config.BILLING_ENABLED and BillingService.is_email_in_freeze(email):
    raise AccountRegisterError(
        description=(
            "This email account has been deleted within the past "
            "30 days and is temporarily unavailable for new account registration"
        )
    )
```

**应用场景**：
- 注册新账户
- 邀请成员加入
- OAuth 登录注册

### 6. 教育优惠计划

**代码位置**：`services/billing_service.py`

```python
class BillingService:
    class EducationIdentity:
        """教育身份验证"""
        
        @classmethod
        def verify(cls, account_id: str, account_email: str):
            """验证教育邮箱"""
            params = {"account_id": account_id}
            return BillingService._send_request("GET", "/education/verify", params=params)
        
        @classmethod
        def activate(cls, account: Account, token: str, institution: str, role: str):
            """激活教育优惠"""
            json = {
                "institution": institution,
                "token": token,
                "role": role,
            }
            return BillingService._send_request("POST", "/education/", json=json, params=...)
```

**功能**：
- 教育邮箱验证（.edu 域名）
- 提供优惠订阅计划
- 速率限制：10 次/分钟

---

## 企业版功能

### 1. 企业版架构

```python
# 配置
ENTERPRISE_ENABLED = os.getenv("ENTERPRISE_ENABLED", "false").lower() == "true"
```

#### 核心服务：`EnterpriseService`

**代码位置**：`api/services/enterprise/enterprise_service.py`

```python
class EnterpriseService:
    """与企业版 API 通信"""
    
    @classmethod
    def get_info(cls):
        """获取企业版全局信息"""
        return EnterpriseRequest.send_request("GET", "/info")
    
    @classmethod
    def get_workspace_info(cls, tenant_id: str):
        """获取工作区信息（包含成员限制）"""
        return EnterpriseRequest.send_request("GET", f"/workspace/{tenant_id}/info")
```

### 2. 企业版独占功能

#### 2.1 SSO 单点登录

**🔥 企业版独占**

```python
class SystemFeatureModel(BaseModel):
    sso_enforced_for_signin: bool = False  # 强制 SSO 登录
    sso_enforced_for_signin_protocol: str = ""  # SAML/OAuth/OIDC
```

**功能特性**：
- 支持 SAML 2.0
- 支持 OAuth 2.0
- 支持 OIDC (OpenID Connect)
- 可强制用户使用 SSO 登录
- WebApp 应用级 SSO 控制

#### 2.2 自定义品牌

**🔥 企业版独占**

```python
class BrandingModel(BaseModel):
    enabled: bool = False
    application_title: str = ""      # 自定义应用标题
    login_page_logo: str = ""        # 登录页 Logo
    workspace_logo: str = ""         # 工作区 Logo
    favicon: str = ""                # 网站图标
```

**权限控制**：
```python
if dify_config.ENTERPRISE_ENABLED:
    system_features.branding.enabled = True
```

#### 2.3 WebApp 访问控制

**🔥 企业版独占**

```python
class WebAppAuthModel(BaseModel):
    enabled: bool = False
    allow_sso: bool = False
    sso_config: WebAppAuthSSOModel = WebAppAuthSSOModel()
    allow_email_code_login: bool = False
    allow_email_password_login: bool = False
```

**功能**：
- 控制 WebApp 的访问方式
- 支持 SSO 验证后访问
- 支持私有应用（需权限验证）
- 批量检查用户权限

```python
class EnterpriseService:
    class WebAppAuth:
        @classmethod
        def is_user_allowed_to_access_webapp(cls, user_id: str, app_id: str):
            """检查用户是否有权限访问应用"""
            params = {"userId": user_id, "appId": app_id}
            data = EnterpriseRequest.send_request("GET", "/webapp/permission", params=params)
            return data.get("result", False)
        
        @classmethod
        def get_app_access_mode_by_id(cls, app_id: str) -> WebAppSettings:
            """获取应用访问模式"""
            # 返回: public / private / private_all / sso_verified
```

**访问模式**：
| 模式 | 说明 |
|------|------|
| `public` | 公开访问，任何人可访问 |
| `private` | 私有访问，需授权用户 |
| `private_all` | 完全私有，仅内部用户 |
| `sso_verified` | 需 SSO 验证 |

#### 2.4 插件管理器

**🔥 企业版独占**

```python
class PluginManagerModel(BaseModel):
    enabled: bool = False

# 启用条件
if dify_config.ENTERPRISE_ENABLED:
    system_features.plugin_manager.enabled = True
```

**功能**：
- 统一管理插件安装
- 控制插件安装范围
- 限制插件来源

```python
class PluginInstallationScope(StrEnum):
    NONE = "none"  # 禁止所有插件
    OFFICIAL_ONLY = "official_only"  # 仅官方插件
    OFFICIAL_AND_SPECIFIC_PARTNERS = "official_and_specific_partners"  # 官方+特定合作伙伴
    ALL = "all"  # 所有插件

class PluginInstallationPermissionModel(BaseModel):
    plugin_installation_scope: PluginInstallationScope = PluginInstallationScope.ALL
    restrict_to_marketplace_only: bool = False  # 强制从市场安装
```

#### 2.5 知识管道发布功能

**🔥 企业版独占**

```python
class KnowledgePipeline(BaseModel):
    publish_enabled: bool = False

# 启用条件
if dify_config.ENTERPRISE_ENABLED:
    features.knowledge_pipeline.publish_enabled = True
```

#### 2.6 WebApp 版权标识

```python
# 企业版/付费版都会启用
if dify_config.ENTERPRISE_ENABLED:
    features.webapp_copyright_enabled = True
# 或
if features.billing.subscription.plan != "sandbox":
    features.webapp_copyright_enabled = True
```

#### 2.7 禁用邮箱修改

**🔥 企业版独占**

```python
if dify_config.ENTERPRISE_ENABLED:
    system_features.enable_change_email = False
```

**原因**：企业版通常与 SSO 集成，邮箱由身份提供商管理。

---

## 许可证管理

### 1. 许可证状态

```python
class LicenseStatus(StrEnum):
    NONE = "none"          # 无许可证
    INACTIVE = "inactive"  # 未激活
    ACTIVE = "active"      # 已激活
    EXPIRING = "expiring"  # 即将过期
    EXPIRED = "expired"    # 已过期
    LOST = "lost"          # 丢失

class LicenseModel(BaseModel):
    status: LicenseStatus = LicenseStatus.NONE
    expired_at: str = ""  # 过期时间
    workspaces: LicenseLimitationModel = ...  # 工作区限制
```

### 2. 许可证验证装饰器

**🔥 企业版独占**

```python
@enterprise_license_required
def protected_api():
    """需要有效许可证才能访问"""
    # 如果许可证无效/过期/丢失，返回 401 错误
    # 并强制用户登出
```

**实现**：
```python
def enterprise_license_required(view):
    @wraps(view)
    def decorated(*args, **kwargs):
        settings = FeatureService.get_system_features()
        if settings.license.status in [
            LicenseStatus.INACTIVE,
            LicenseStatus.EXPIRED,
            LicenseStatus.LOST
        ]:
            raise UnauthorizedAndForceLogout(
                "Your license is invalid. Please contact your administrator."
            )
        return view(*args, **kwargs)
    return decorated
```

### 3. 工作区数量限制

**🔥 企业版独占**

```python
class LicenseLimitationModel(BaseModel):
    enabled: bool = False  # 是否启用限制
    size: int = 0          # 已使用数量
    limit: int = 0         # 最大数量（0=无限制）
    
    def is_available(self, required: int = 1) -> bool:
        """检查是否还有可用配额"""
        if not self.enabled or self.limit == 0:
            return True  # 未启用或无限制
        return (self.limit - self.size) >= required
```

**应用场景**：
```python
class LicenseModel(BaseModel):
    workspaces: LicenseLimitationModel = LicenseLimitationModel(
        enabled=False,
        size=0,
        limit=0
    )

# 企业版从 API 获取限制
if dify_config.ENTERPRISE_ENABLED:
    workspace_info = EnterpriseService.get_workspace_info(tenant_id)
    features.workspace_members.size = workspace_info["WorkspaceMembers"]["used"]
    features.workspace_members.limit = workspace_info["WorkspaceMembers"]["limit"]
    features.workspace_members.enabled = workspace_info["WorkspaceMembers"]["enabled"]
```

---

## 配置与环境变量

### 完整配置清单

```bash
# ================== 部署模式 ==================
EDITION=SELF_HOSTED  # 或 CLOUD

# ================== 云版本计费 ==================
BILLING_ENABLED=false              # 是否启用计费
BILLING_API_URL=                   # 计费 API 地址
BILLING_API_SECRET_KEY=            # 计费 API 密钥

# ================== 企业版 ==================
ENTERPRISE_ENABLED=false           # 是否启用企业版
ENTERPRISE_API_URL=                # 企业版 API 地址
ENTERPRISE_API_SECRET_KEY=         # 企业版 API 密钥

# ================== 功能开关 ==================
EDUCATION_ENABLED=false            # 教育优惠
MARKETPLACE_ENABLED=false          # 插件市场
CAN_REPLACE_LOGO=false             # 允许替换 Logo
MODEL_LB_ENABLED=false             # 模型负载均衡
DATASET_OPERATOR_ENABLED=false     # 数据集操作员角色

# ================== 邮箱功能 ==================
ENABLE_EMAIL_CODE_LOGIN=false      # 邮箱验证码登录
ENABLE_EMAIL_PASSWORD_LOGIN=true   # 邮箱密码登录
ENABLE_SOCIAL_OAUTH_LOGIN=false    # 社交登录

# ================== 注册限制 ==================
ALLOW_REGISTER=true                # 允许注册
ALLOW_CREATE_WORKSPACE=true        # 允许创建工作区
```

---

## 权限装饰器

### 1. 企业版专属装饰器

#### `@only_edition_enterprise`

```python
@only_edition_enterprise
def enterprise_only_api():
    """仅企业版可访问"""
    # 如果不是企业版，返回 404
```

**代码**：
```python
def only_edition_enterprise(view):
    @wraps(view)
    def decorated(*args, **kwargs):
        if not dify_config.ENTERPRISE_ENABLED:
            abort(404)
        return view(*args, **kwargs)
    return decorated
```

#### `@enterprise_license_required`

```python
@enterprise_license_required
def licensed_api():
    """需要有效企业许可证"""
    # 验证许可证状态
```

### 2. 云版本计费装饰器

#### `@cloud_edition_billing_enabled`

```python
@cloud_edition_billing_enabled
def billing_api():
    """需要启用计费功能"""
    # 如果未启用计费，返回 403
```

#### `@cloud_edition_billing_resource_check(resource)`

```python
@cloud_edition_billing_resource_check("members")
def add_member():
    """检查成员配额"""
    # 如果超过限制，返回 403
```

**支持的资源类型**：
- `members`：成员数量
- `apps`：应用数量
- `vector_space`：向量空间
- `documents`：文档数量
- `workspace_custom`：自定义品牌
- `annotation`：标注配额

#### `@cloud_edition_billing_knowledge_limit_check(resource)`

```python
@cloud_edition_billing_knowledge_limit_check("add_segment")
def add_segment():
    """沙盒计划限制"""
    # 沙盒计划不能添加分段
```

#### `@cloud_edition_billing_rate_limit_check(resource)`

```python
@cloud_edition_billing_rate_limit_check("knowledge")
def knowledge_query():
    """知识库速率限制"""
    # 检查请求频率
```

### 3. 自托管版装饰器

#### `@only_edition_self_hosted`

```python
@only_edition_self_hosted
def self_hosted_only():
    """仅自托管版可访问"""
    # 如果不是自托管版，返回 404
```

---

## 限制与配额

### 对比表

| 限制项 | 开源版 | 云版本(sandbox) | 云版本(付费) | 企业版 |
|-------|--------|----------------|-------------|--------|
| **成员数量** | 无限制 | 1 | 按计划 | 按许可证 |
| **应用数量** | 无限制 | 10 | 按计划 | 无限制 |
| **向量空间** | 无限制 | 5 MB | 按计划 | 无限制 |
| **文档上传** | 无限制 | 50 | 按计划 | 无限制 |
| **标注配额** | 无限制 | 10 | 按计划 | 无限制 |
| **知识库速率** | 无限制 | 10/分钟 | 按计划 | 按配置 |
| **工作区数量** | 无限制 | 无限制 | 无限制 | 按许可证 |
| **工作区转移** | ✅ | ❌(sandbox) | ✅ | ✅ |

---

## 代码实现详解

### 特性获取流程

```python
class FeatureService:
    @classmethod
    def get_features(cls, tenant_id: str) -> FeatureModel:
        """获取租户的所有特性"""
        features = FeatureModel()
        
        # 1. 从环境变量获取基础配置
        cls._fulfill_params_from_env(features)
        
        # 2. 如果启用计费，从计费 API 获取
        if dify_config.BILLING_ENABLED and tenant_id:
            cls._fulfill_params_from_billing_api(features, tenant_id)
        
        # 3. 如果启用企业版，从企业 API 获取
        if dify_config.ENTERPRISE_ENABLED:
            features.webapp_copyright_enabled = True
            features.knowledge_pipeline.publish_enabled = True
            cls._fulfill_params_from_workspace_info(features, tenant_id)
        
        return features
```

### 计费信息获取

```python
@classmethod
def _fulfill_params_from_billing_api(cls, features, tenant_id):
    """从计费 API 获取限制信息"""
    billing_info = BillingService.get_info(tenant_id)
    
    features.billing.enabled = billing_info["enabled"]
    features.billing.subscription.plan = billing_info["subscription"]["plan"]
    features.billing.subscription.interval = billing_info["subscription"]["interval"]
    
    # 成员限制
    if "members" in billing_info:
        features.members.size = billing_info["members"]["size"]
        features.members.limit = billing_info["members"]["limit"]
    
    # 应用限制
    if "apps" in billing_info:
        features.apps.size = billing_info["apps"]["size"]
        features.apps.limit = billing_info["apps"]["limit"]
    
    # 向量空间限制
    if "vector_space" in billing_info:
        features.vector_space.size = billing_info["vector_space"]["size"]
        features.vector_space.limit = billing_info["vector_space"]["limit"]
    
    # 文档配额
    if "documents_upload_quota" in billing_info:
        features.documents_upload_quota.size = billing_info["documents_upload_quota"]["size"]
        features.documents_upload_quota.limit = billing_info["documents_upload_quota"]["limit"]
    
    # 标注配额
    if "annotation_quota_limit" in billing_info:
        features.annotation_quota_limit.size = billing_info["annotation_quota_limit"]["size"]
        features.annotation_quota_limit.limit = billing_info["annotation_quota_limit"]["limit"]
    
    # 品牌定制
    if "can_replace_logo" in billing_info:
        features.can_replace_logo = billing_info["can_replace_logo"]
    
    # 知识管道发布
    if "knowledge_pipeline_publish_enabled" in billing_info:
        features.knowledge_pipeline.publish_enabled = billing_info["knowledge_pipeline_publish_enabled"]
```

### 企业版信息获取

```python
@classmethod
def _fulfill_params_from_enterprise(cls, features: SystemFeatureModel):
    """从企业 API 获取配置"""
    enterprise_info = EnterpriseService.get_info()
    
    # SSO 配置
    if "SSOEnforcedForSignin" in enterprise_info:
        features.sso_enforced_for_signin = enterprise_info["SSOEnforcedForSignin"]
    
    if "SSOEnforcedForSigninProtocol" in enterprise_info:
        features.sso_enforced_for_signin_protocol = enterprise_info["SSOEnforcedForSigninProtocol"]
    
    # 登录方式控制
    if "EnableEmailCodeLogin" in enterprise_info:
        features.enable_email_code_login = enterprise_info["EnableEmailCodeLogin"]
    
    if "EnableEmailPasswordLogin" in enterprise_info:
        features.enable_email_password_login = enterprise_info["EnableEmailPasswordLogin"]
    
    # 注册和工作区创建
    if "IsAllowRegister" in enterprise_info:
        features.is_allow_register = enterprise_info["IsAllowRegister"]
    
    if "IsAllowCreateWorkspace" in enterprise_info:
        features.is_allow_create_workspace = enterprise_info["IsAllowCreateWorkspace"]
    
    # 品牌定制
    if "Branding" in enterprise_info:
        features.branding.application_title = enterprise_info["Branding"].get("applicationTitle", "")
        features.branding.login_page_logo = enterprise_info["Branding"].get("loginPageLogo", "")
        features.branding.workspace_logo = enterprise_info["Branding"].get("workspaceLogo", "")
        features.branding.favicon = enterprise_info["Branding"].get("favicon", "")
    
    # WebApp 认证
    if "WebAppAuth" in enterprise_info:
        features.webapp_auth.allow_sso = enterprise_info["WebAppAuth"].get("allowSso", False)
        features.webapp_auth.allow_email_code_login = enterprise_info["WebAppAuth"].get("allowEmailCodeLogin", False)
        features.webapp_auth.allow_email_password_login = enterprise_info["WebAppAuth"].get("allowEmailPasswordLogin", False)
    
    # 许可证信息
    if "License" in enterprise_info:
        license_info = enterprise_info["License"]
        features.license.status = LicenseStatus(license_info.get("status", LicenseStatus.INACTIVE))
        features.license.expired_at = license_info.get("expiredAt", "")
        
        if "workspaces" in license_info:
            features.license.workspaces.enabled = license_info["workspaces"]["enabled"]
            features.license.workspaces.limit = license_info["workspaces"]["limit"]
            features.license.workspaces.size = license_info["workspaces"]["used"]
    
    # 插件安装权限
    if "PluginInstallationPermission" in enterprise_info:
        plugin_info = enterprise_info["PluginInstallationPermission"]
        features.plugin_installation_permission.plugin_installation_scope = plugin_info["pluginInstallationScope"]
        features.plugin_installation_permission.restrict_to_marketplace_only = plugin_info["restrictToMarketplaceOnly"]
```

---

## 总结

### 商业化功能分类

#### 🟢 云版本独占
- 订阅计划管理
- 资源配额限制（成员/应用/向量空间等）
- 知识库速率限制
- 账户冻结机制（30天）
- 教育优惠计划

#### 🔵 企业版独占
- SSO 单点登录（SAML/OAuth/OIDC）
- 自定义品牌（Logo/标题/图标）
- WebApp 访问控制
- 插件管理器
- 知识管道发布
- 许可证管理
- 工作区数量限制
- 禁用邮箱修改

#### 🟡 共享功能（云版本+企业版）
- WebApp 版权标识移除
- 高级文档处理
- 模型负载均衡

### 已有文档中的商业版功能引用

根据搜索结果，以下文档中提到了商业版功能：

1. **`account_permission_management.md`** (第 506 行)
   ```markdown
   BILLING_ENABLED=true              # 是否启用计费（影响成员限制）
   ```
   **建议**：✏️ 添加标识 `[商业版功能]`

2. **`account_permission_management_implementation_details.md`** (第 346, 352-353 行)
   ```python
   @cloud_edition_billing_resource_check("members")
   
   if dify_config.BILLING_ENABLED:
       BillingService.clean_billing_info_cache(tenant.id)
   ```
   **建议**：✏️ 添加标识 `[云版本计费功能]`

### 标识建议

在已有文档中，应该用醒目的标识标注商业版功能：

```markdown
### 成员限制检查 [☁️ 云版本计费功能]

在添加成员时，系统会检查订阅计划的成员配额限制：

\`\`\`python
@cloud_edition_billing_resource_check("members")
def add_member_api():
    # ...
\`\`\`
```

或使用表格：

| 功能 | 开源版 | 云版本 | 企业版 |
|------|--------|--------|--------|
| 成员数量限制 | 无限制 | ✅ 按计划 | ✅ 按许可证 |

---

**📌 本文档完整涵盖了 Dify 权限系统中所有的商业版/企业版功能，可作为独立参考文档使用。**
