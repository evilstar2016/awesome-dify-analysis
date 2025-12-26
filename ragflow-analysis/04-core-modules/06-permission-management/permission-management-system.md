# RAGFlow 权限管理子系统分析文档

## 1. 系统概述

RAGFlow的权限管理子系统是一个完整的多租户权限控制系统，实现了用户、租户（Tenant）、资源的细粒度隔离和访问控制。系统支持多种认证方式，包括本地密码认证和第三方OAuth/OIDC集成。

### 1.1 核心特性

- ✅ **多租户隔离**：用户-租户多对多关系，支持租户级数据隔离
- ✅ **知识库权限控制**：支持个人（ME）和团队（TEAM）两级权限
- ✅ **角色管理**：Owner、Admin、Normal、Invite四级角色体系
- ✅ **第三方认证**：支持OAuth2、OIDC、GitHub等多种认证方式
- ✅ **资源级权限**：知识库、文档、文件等资源的细粒度权限控制
- ✅ **会话安全**：基于Quart-Auth的会话管理

---

## 2. 架构设计

### 2.1 数据模型

#### 2.1.1 核心实体关系

```
User (用户)
  ↓ 多对多
UserTenant (用户-租户关联)
  ↓ 多对多
Tenant (租户)
  ↓ 一对多
Knowledgebase (知识库)
  ↓ 一对多
Document (文档)
  ↓ 一对多
File (文件)
```

#### 2.1.2 User（用户表）

**文件位置**: `api/db/db_models.py`

**核心字段**:
```python
class User(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)  # 主键
    access_token = CharField(max_length=128)         # 访问令牌
    nickname = CharField(max_length=128)             # 昵称
    password = CharField(max_length=128)             # 密码（加密）
    email = CharField(max_length=128, unique=True)   # 邮箱（唯一）
    is_authenticated = BooleanField(default=True)    # 是否已认证
    is_active = BooleanField(default=True)           # 是否激活
    is_superuser = BooleanField(default=False)       # 是否超级用户
    login_channel = CharField(max_length=16)         # 登录渠道
    status = CharField(max_length=16)                # 状态
```

**设计要点**:
- `id` 作为主键，用于全局用户标识
- `access_token` 用于API访问控制
- `is_superuser` 支持超级管理员权限
- `login_channel` 记录登录来源（password/github/feishu等）

#### 2.1.3 Tenant（租户表）

**核心字段**:
```python
class Tenant(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)  # 租户ID
    name = CharField(max_length=128, unique=True)    # 租户名称（唯一）
    llm_id = CharField(max_length=32)                # 默认LLM模型ID
    embd_id = CharField(max_length=32)               # 默认Embedding模型ID
    credit = IntegerField(default=0)                 # 信用额度
    status = CharField(max_length=16)                # 状态
```

**设计要点**:
- 每个租户有独立的模型配置（LLM、Embedding等）
- 租户是资源隔离的基本单位
- 支持信用额度管理

#### 2.1.4 UserTenant（用户-租户关联表）

**核心字段**:
```python
class UserTenant(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    user_id = CharField(max_length=32)               # 用户ID
    tenant_id = CharField(max_length=32)             # 租户ID
    role = CharField(max_length=16)                  # 角色：owner/admin/normal/invite
    invited_by = CharField(max_length=32)            # 邀请人
    status = CharField(max_length=16)                # 状态
```

**设计要点**:
- 实现用户-租户多对多关系
- 每个用户在租户内有特定角色
- 支持邀请机制追溯

#### 2.1.5 Knowledgebase（知识库表）

**核心字段**:
```python
class Knowledgebase(DataBaseModel):
    id = CharField(max_length=32, primary_key=True)
    tenant_id = CharField(max_length=32)             # 所属租户ID
    name = CharField(max_length=128)                 # 知识库名称
    permission = CharField(max_length=16)            # 权限：me/team
    created_by = CharField(max_length=32)            # 创建者ID
```

**设计要点**:
- `tenant_id` 实现租户级隔离
- `permission` 控制可见范围：
  - `me`: 仅创建者可见
  - `team`: 团队内可见（同租户用户）
- `created_by` 标识知识库所有者

### 2.2 权限枚举定义

**文件位置**: `api/db/__init__.py`

```python
class UserTenantRole(StrEnum):
    OWNER = 'owner'     # 拥有者：最高权限
    ADMIN = 'admin'     # 管理员：管理权限
    NORMAL = 'normal'   # 普通用户：基本权限
    INVITE = 'invite'   # 受邀用户：待审核

class TenantPermission(StrEnum):
    ME = 'me'          # 个人私有
    TEAM = 'team'      # 团队共享
```

---

## 3. 多租户隔离机制

### 3.1 数据隔离策略

#### 3.1.1 知识库级隔离

**实现文件**: `api/common/check_team_permission.py`

```python
def check_kb_team_permission(kb: dict | Knowledgebase, other: str) -> bool:
    """
    检查知识库团队权限
    
    参数:
        kb: 知识库对象或字典
        other: 请求用户的ID
    
    返回:
        bool: 是否有权限访问
    """
    kb = kb.to_dict() if isinstance(kb, Knowledgebase) else kb
    kb_tenant_id = kb["tenant_id"]
    
    # 1. 检查是否为知识库所属租户
    if kb_tenant_id == other:
        return True
    
    # 2. 检查知识库权限类型
    if kb["permission"] != TenantPermission.TEAM:
        return False
    
    # 3. 检查用户是否加入该租户
    joined_tenants = TenantService.get_joined_tenants_by_user_id(other)
    return any(tenant["tenant_id"] == kb_tenant_id for tenant in joined_tenants)
```

**权限判断逻辑**:
1. **租户匹配**: 如果用户ID等于知识库的tenant_id，直接授权
2. **权限检查**: 如果知识库权限不是"team"，拒绝访问
3. **团队成员**: 检查用户是否加入知识库所属租户

#### 3.1.2 文件级隔离

```python
def check_file_team_permission(file: dict | File, other: str) -> bool:
    """
    检查文件团队权限
    
    逻辑：
    1. 检查文件所属租户
    2. 查找文件关联的所有知识库
    3. 对每个知识库进行权限检查
    """
    file_tenant_id = file["tenant_id"]
    if file_tenant_id == other:
        return True
    
    file_id = file["id"]
    kb_ids = [kb_info["kb_id"] for kb_info in FileService.get_kb_id_by_file_id(file_id)]
    
    for kb_id in kb_ids:
        ok, kb = KnowledgebaseService.get_by_id(kb_id)
        if not ok:
            continue
        if check_kb_team_permission(kb, other):
            return True
    
    return False
```

### 3.2 索引隔离

**搜索引擎索引命名**:
```python
# 每个租户有独立的索引
index_name = search.index_name(tenant_id)  # 例如: ragflow_tenant_abc123

# 向量搜索隔离
settings.docStoreConn.search(
    query, 
    search.index_name(kb.tenant_id),  # 租户级索引
    [kb_id]                            # 知识库级过滤
)
```

**设计优势**:
- ✅ 租户数据物理隔离
- ✅ 防止跨租户数据泄露
- ✅ 支持独立的索引优化

### 3.3 用户访问控制流程

```
用户请求
    ↓
登录验证 (login_required)
    ↓
获取用户所属租户列表
    ↓
过滤可访问的知识库
    ├─ tenant_id in user_tenants ✓
    └─ permission == TEAM ✓
    ↓
返回授权资源
```

---

## 4. 认证系统

### 4.1 本地密码认证

**实现文件**: `api/apps/user_app.py`

**登录流程**:
```python
@manager.route("/login", methods=["POST"])
async def login():
    # 1. 获取登录凭证
    json_body = await get_request_json()
    email = json_body.get("email", "")
    password = json_body.get("password")
    
    # 2. 查询用户
    users = UserService.query(email=email)
    if not users:
        return get_json_result(code=RetCode.AUTHENTICATION_ERROR, 
                              message=f"Email: {email} is not registered!")
    
    # 3. 密码解密与验证
    password = decrypt(password)
    user = UserService.query_user(email, password)
    
    # 4. 检查用户状态
    if user and user.is_active == "0":
        return get_json_result(code=RetCode.FORBIDDEN, 
                              message="This account has been disabled!")
    
    # 5. 生成会话
    if user:
        user.access_token = get_uuid()
        login_user(user)  # Flask-Login会话管理
        user.save()
        return construct_response(data=user.to_json(), auth=user.get_id())
```

**密码安全**:
- 传输加密：前端发送加密密码
- 存储加密：数据库存储加密后的密码
- 会话令牌：每次登录生成新的access_token

### 4.2 第三方认证集成

#### 4.2.1 OAuth2 架构

**实现文件**: `api/apps/auth/oauth.py`

**核心类**:
```python
class OAuthClient:
    """OAuth2客户端基类"""
    
    def __init__(self, config):
        self.client_id = config["client_id"]
        self.client_secret = config["client_secret"]
        self.authorization_url = config["authorization_url"]
        self.token_url = config["token_url"]
        self.userinfo_url = config["userinfo_url"]
        self.redirect_uri = config["redirect_uri"]
    
    def get_authorization_url(self, state=None):
        """生成授权URL"""
        params = {
            "client_id": self.client_id,
            "redirect_uri": self.redirect_uri,
            "response_type": "code",
            "scope": self.scope
        }
        return f"{self.authorization_url}?{urlencode(params)}"
    
    def exchange_code_for_token(self, code):
        """用授权码换取访问令牌"""
        payload = {
            "client_id": self.client_id,
            "client_secret": self.client_secret,
            "code": code,
            "redirect_uri": self.redirect_uri,
            "grant_type": "authorization_code"
        }
        response = sync_request("POST", self.token_url, data=payload)
        return response.json()
    
    def fetch_user_info(self, access_token):
        """获取用户信息"""
        headers = {"Authorization": f"Bearer {access_token}"}
        response = sync_request("GET", self.userinfo_url, headers=headers)
        return UserInfo(**response.json())
```

#### 4.2.2 OIDC 支持

**实现文件**: `api/apps/auth/oidc.py`

```python
class OIDCClient(OAuthClient):
    """OpenID Connect客户端"""
    
    def __init__(self, config):
        # 自动发现OIDC配置
        self.issuer = config["issuer"]
        oidc_metadata = self._load_oidc_metadata(self.issuer)
        
        config.update({
            'authorization_url': oidc_metadata['authorization_endpoint'],
            'token_url': oidc_metadata['token_endpoint'],
            'userinfo_url': oidc_metadata['userinfo_endpoint'],
            'jwks_uri': oidc_metadata['jwks_uri']
        })
        super().__init__(config)
    
    def _load_oidc_metadata(self, issuer):
        """加载OIDC元数据"""
        metadata_url = f"{issuer}/.well-known/openid-configuration"
        response = sync_request("GET", metadata_url)
        return response.json()
    
    def parse_id_token(self, id_token):
        """解析并验证ID Token (JWT)"""
        # 1. 获取JWKS
        jwks_cli = jwt.PyJWKClient(self.jwks_uri)
        signing_key = jwks_cli.get_signing_key_from_jwt(id_token).key
        
        # 2. 验证签名
        decoded_token = jwt.decode(
            id_token,
            key=signing_key,
            algorithms=["RS256"],
            audience=self.client_id,
            issuer=self.issuer
        )
        return decoded_token
```

**OIDC 优势**:
- ✅ 自动配置发现
- ✅ JWT令牌验证
- ✅ 标准化用户信息

#### 4.2.3 GitHub OAuth

**实现文件**: `api/apps/auth/github.py`

```python
class GithubOAuthClient(OAuthClient):
    """GitHub专用OAuth客户端"""
    
    def __init__(self, config):
        config.update({
            "authorization_url": "https://github.com/login/oauth/authorize",
            "token_url": "https://github.com/login/oauth/access_token",
            "userinfo_url": "https://api.github.com/user",
            "scope": "user:email"
        })
        super().__init__(config)
    
    def fetch_user_info(self, access_token):
        """获取GitHub用户信息"""
        headers = {"Authorization": f"Bearer {access_token}"}
        
        # 获取用户基本信息
        response = sync_request("GET", self.userinfo_url, headers=headers)
        user_info = response.json()
        
        # 获取主邮箱
        email_response = sync_request("GET", f"{self.userinfo_url}/emails", headers=headers)
        emails = email_response.json()
        primary_email = next((e for e in emails if e["primary"]), None)["email"]
        
        return UserInfo(
            email=primary_email,
            username=user_info.get("login"),
            nickname=user_info.get("name"),
            avatar_url=user_info.get("avatar_url")
        )
```

### 4.3 OAuth认证流程

**实现文件**: `api/apps/user_app.py`

```python
@manager.route("/oauth/login", methods=["GET"])
async def oauth_login():
    """OAuth登录入口"""
    channel = request.args.get("channel")  # github/feishu/oidc
    
    # 1. 获取OAuth配置
    config = get_oauth_config(channel)
    
    # 2. 创建OAuth客户端
    client = get_auth_client(config)
    
    # 3. 生成授权URL
    state = generate_state()  # CSRF防护
    auth_url = client.get_authorization_url(state=state)
    
    # 4. 重定向到认证服务器
    return redirect(auth_url)

@manager.route("/oauth/callback/<channel>", methods=["GET"])
async def oauth_callback(channel):
    """OAuth回调处理"""
    code = request.args.get("code")
    state = request.args.get("state")
    
    # 1. 验证state（CSRF防护）
    if not verify_state(state):
        return get_json_result(code=RetCode.AUTHENTICATION_ERROR)
    
    # 2. 获取OAuth客户端
    config = get_oauth_config(channel)
    client = get_auth_client(config)
    
    # 3. 用授权码换取令牌
    token_response = await client.async_exchange_code_for_token(code)
    access_token = token_response["access_token"]
    
    # 4. 获取用户信息
    user_info = await client.async_fetch_user_info(access_token)
    
    # 5. 查询或创建用户
    users = UserService.query(email=user_info.email)
    if not users:
        # 自动注册新用户
        user = create_user_from_oauth(user_info, channel)
    else:
        user = users[0]
    
    # 6. 创建会话
    login_user(user)
    return redirect("/")
```

### 4.4 支持的认证渠道

| 渠道 | 协议 | 实现类 | 特性 |
|------|------|--------|------|
| **Password** | 本地 | - | 基础密码认证 |
| **GitHub** | OAuth2 | `GithubOAuthClient` | 社区账号集成 |
| **Feishu** | OAuth2 | `OAuthClient` | 企业IM集成 |
| **OIDC** | OIDC | `OIDCClient` | 标准化企业认证 |
| **Custom OAuth** | OAuth2 | `OAuthClient` | 自定义OAuth服务 |

---

## 5. 服务层实现

### 5.1 TenantService（租户服务）

**文件位置**: `api/db/services/user_service.py`

**核心方法**:

```python
class TenantService:
    model = Tenant
    
    @classmethod
    def get_joined_tenants_by_user_id(cls, user_id):
        """
        获取用户加入的所有租户
        
        SQL逻辑:
        SELECT t.* FROM tenant t
        JOIN user_tenant ut ON t.id = ut.tenant_id
        WHERE ut.user_id = ? AND ut.status = 'VALID'
        """
        tenants = cls.model.select().join(
            UserTenant,
            on=(cls.model.id == UserTenant.tenant_id)
        ).where(
            UserTenant.user_id == user_id,
            UserTenant.status == StatusEnum.VALID.value
        )
        return [t.to_dict() for t in tenants]
```

### 5.2 UserTenantService（用户-租户关联服务）

```python
class UserTenantService:
    model = UserTenant
    
    @classmethod
    def get_tenants_by_user_id(cls, user_id):
        """获取用户的租户关联（包含角色信息）"""
        relations = cls.model.select().where(
            cls.model.user_id == user_id,
            cls.model.status == StatusEnum.VALID.value
        )
        return [r.to_dict() for r in relations]
    
    @classmethod
    def get_by_tenant_id(cls, tenant_id):
        """获取租户的所有成员"""
        members = cls.model.select().join(
            User,
            on=(cls.model.user_id == User.id)
        ).where(
            cls.model.tenant_id == tenant_id,
            cls.model.status == StatusEnum.VALID.value
        )
        return [m.to_dict() for m in members]
```

### 5.3 KnowledgebaseService（知识库服务）

**文件位置**: `api/db/services/knowledgebase_service.py`

```python
class KnowledgebaseService:
    model = Knowledgebase
    
    @classmethod
    def get_by_tenant_ids(cls, tenant_ids, user_id, page, page_size, 
                          orderby, desc, keywords, parser_id):
        """
        获取多个租户的知识库（支持权限过滤）
        
        权限逻辑:
        1. 返回tenant_id在tenant_ids中的知识库
        2. 如果permission='me'，只返回created_by=user_id的
        3. 如果permission='team'，返回所有同租户知识库
        """
        query = cls.model.select().where(
            cls.model.tenant_id.in_(tenant_ids),
            cls.model.status == StatusEnum.VALID.value
        )
        
        # 关键词搜索
        if keywords:
            query = query.where(cls.model.name.contains(keywords))
        
        # 权限过滤
        kbs = []
        for kb in query:
            if kb.permission == TenantPermission.ME and kb.created_by != user_id:
                continue  # 私有知识库，非创建者跳过
            kbs.append(kb.to_dict())
        
        # 分页
        total = len(kbs)
        if page and page_size:
            kbs = kbs[(page-1)*page_size : page*page_size]
        
        return kbs, total
```

---

## 6. API层权限控制

### 6.1 装饰器验证

**常见装饰器**:
```python
@manager.route('/create', methods=['POST'])
@login_required                    # 要求用户登录
@validate_request("name")          # 验证请求参数
@not_allowed_parameters("tenant_id")  # 禁止修改租户ID
async def create_kb():
    # tenant_id自动设置为当前用户ID
    tenant_id = current_user.id
    kb = KnowledgebaseService.create_with_name(
        name=req["name"],
        tenant_id=tenant_id,  # 强制使用当前用户租户
        created_by=current_user.id
    )
```

### 6.2 知识库列表权限过滤

**实现文件**: `api/apps/kb_app.py`

```python
@manager.route('/list', methods=['GET'])
@login_required
async def list_kbs():
    # 1. 获取用户加入的所有租户
    tenants = TenantService.get_joined_tenants_by_user_id(current_user.id)
    tenant_ids = [t["tenant_id"] for t in tenants]
    
    # 2. 查询这些租户的知识库（自动过滤权限）
    kbs, total = KnowledgebaseService.get_by_tenant_ids(
        tenant_ids, 
        current_user.id,  # 用于ME权限过滤
        page_number, 
        items_per_page
    )
    
    # 3. 返回结果
    return get_json_result(data={"knowledgebases": kbs, "total": total})
```

### 6.3 知识库更新权限

```python
@manager.route('/update', methods=['POST'])
@login_required
@validate_request("kb_id")
@not_allowed_parameters("tenant_id", "created_by")  # 禁止修改关键字段
async def update_kb():
    kb_id = req["kb_id"]
    
    # 1. 查询知识库
    tenants = UserTenantService.query(user_id=current_user.id)
    for tenant in tenants:
        if KnowledgebaseService.query(tenant_id=tenant.tenant_id, id=kb_id):
            break
    else:
        # 用户不属于任何包含该知识库的租户
        return get_json_result(code=RetCode.OPERATING_ERROR, 
                              message="You don't own the dataset.")
    
    # 2. 更新知识库
    KnowledgebaseService.update_by_id(kb_id, req)
    return get_json_result(data=True)
```

### 6.4 删除操作权限

```python
@manager.route('/rm', methods=['POST'])
@login_required
@validate_request("kb_id")
async def rm_kb():
    req = await get_request_json()
    kb_ids = req["kb_id"]
    
    # 权限检查：只能删除自己租户的知识库
    kbs = []
    for kb_id in kb_ids:
        for tenant in UserTenantService.query(user_id=current_user.id):
            kb_list = KnowledgebaseService.query(
                tenant_id=tenant.tenant_id, 
                id=kb_id
            )
            if kb_list:
                kbs.extend(kb_list)
                break
    
    if not kbs:
        return get_json_result(code=RetCode.OPERATING_ERROR)
    
    # 删除知识库（包括索引隔离）
    for kb in kbs:
        # 删除租户隔离的索引
        settings.docStoreConn.delete(
            {"kb_id": kb.id}, 
            search.index_name(kb.tenant_id),  # 租户级索引
            kb.id
        )
        # 删除数据库记录
        KnowledgebaseService.delete_by_id(kb.id)
    
    return get_json_result(data=True)
```

---

## 7. 安全机制

### 7.1 会话管理

**实现**: Quart-Auth + Flask-Login

```python
from quart_auth import login_user, logout_user, current_user

# 登录
user.access_token = get_uuid()
login_user(user)

# 登出
@manager.route('/logout', methods=['GET'])
async def log_out():
    logout_user()
    return get_json_result(data=True)
```

### 7.2 密码加密

**传输加密**:
- 前端使用公钥加密密码
- 后端使用私钥解密

```python
from common.crypto_utils import decrypt

password = json_body.get("password")
try:
    password = decrypt(password)  # RSA解密
except BaseException:
    return get_json_result(code=RetCode.SERVER_ERROR, 
                          message="Fail to crypt password")
```

**存储加密**:
- 数据库存储加密后的密码
- 使用单向哈希算法

### 7.3 CSRF防护

**OAuth State参数**:
```python
# 生成state
state = generate_random_state()
session['oauth_state'] = state

# 验证state
if request.args.get('state') != session.get('oauth_state'):
    return get_json_result(code=RetCode.AUTHENTICATION_ERROR)
```

### 7.4 SQL注入防护

**使用ORM**:
```python
# ✓ 安全：使用Peewee ORM
users = UserService.query(email=email)

# ✗ 不安全：直接拼接SQL
# query = f"SELECT * FROM user WHERE email='{email}'"
```

### 7.5 XSS防护

**输入验证**:
```python
from api.validation import validate_request

@validate_request("email", "password")
async def login():
    # 自动验证参数存在性和类型
    pass
```

---

## 8. 多租户最佳实践

### 8.1 租户隔离检查清单

✅ **数据库层**:
- 每条记录都有 `tenant_id` 字段
- 所有查询都包含租户过滤条件

✅ **搜索引擎层**:
- 每个租户有独立索引
- 索引命名包含tenant_id

✅ **对象存储层**:
- 每个知识库有独立bucket
- Bucket命名包含kb_id

✅ **API层**:
- 所有接口都验证用户-租户关系
- 禁止通过API修改tenant_id

### 8.2 权限设计模式

**分层权限模型**:
```
系统级 (Superuser)
    ↓
租户级 (Tenant Owner/Admin)
    ↓
资源级 (Knowledgebase Owner)
    ↓
对象级 (Document/File)
```

**权限检查顺序**:
1. 用户是否登录
2. 用户是否属于该租户
3. 资源权限类型（ME/TEAM）
4. 用户在租户内的角色

### 8.3 第三方认证集成指南

**步骤**:
1. 在OAuth提供商注册应用，获取client_id和client_secret
2. 配置回调URL: `https://your-domain/v1/user/oauth/callback/<channel>`
3. 添加配置到系统（环境变量或配置文件）
4. 重启服务

**配置示例**:
```yaml
oauth:
  github:
    client_id: "your_github_client_id"
    client_secret: "your_github_client_secret"
    redirect_uri: "https://ragflow.io/v1/user/oauth/callback/github"
  
  oidc:
    issuer: "https://your-idp.com"
    client_id: "your_oidc_client_id"
    client_secret: "your_oidc_client_secret"
    redirect_uri: "https://ragflow.io/v1/user/oauth/callback/oidc"
```

---

## 9. 时序图

请参考以下PlantUML图文件：
- `permission-auth-flow.puml`: 认证流程时序图
- `permission-oauth-flow.puml`: OAuth认证流程
- `permission-kb-access.puml`: 知识库访问权限检查
- `permission-architecture.puml`: 权限系统架构图

---

## 10. 未来扩展

### 10.1 潜在优化

- [ ] 细粒度角色权限（RBAC）
- [ ] 基于属性的访问控制（ABAC）
- [ ] 多因子认证（MFA）
- [ ] 审计日志增强
- [ ] SAML 2.0支持
- [ ] LDAP/AD集成

### 10.2 性能优化

- [ ] 权限缓存（Redis）
- [ ] 租户连接池隔离
- [ ] 批量权限检查
- [ ] 权限预计算

---

## 11. 总结

RAGFlow的权限管理子系统是一个设计良好的多租户权限控制系统，核心特点包括：

1. **完善的多租户隔离**：从数据库到索引的全方位隔离
2. **灵活的权限模型**：支持ME和TEAM两级权限
3. **丰富的认证方式**：本地密码、OAuth2、OIDC、GitHub等
4. **清晰的角色体系**：Owner、Admin、Normal、Invite四级角色
5. **安全的设计**：密码加密、CSRF防护、会话管理
6. **易于扩展**：模块化设计，支持自定义OAuth集成

系统在保证安全性的同时，也提供了良好的用户体验和开发者友好性。

---

**文档版本**: 1.0  
**生成日期**: 2025-12-23  
**对应项目版本**: RAGFlow v0.22.1
