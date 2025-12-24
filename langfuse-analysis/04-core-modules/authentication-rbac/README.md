# Authentication & RBAC（认证与基于角色的访问控制）

## 模块概述

Authentication & RBAC 模块是 Langfuse 系统的核心安全模块，负责用户身份认证和权限控制。该模块实现了完整的多租户认证体系，支持多种认证方式（包括用户名密码、OAuth、SSO），以及基于角色的细粒度访问控制（RBAC）。

系统采用两级权限模型：
- **组织级权限（Organization-level RBAC）**: 管理组织资源和成员
- **项目级权限（Project-level RBAC）**: 管理项目资源和操作

## 主要功能

1. **多种认证方式**
   - 用户名密码认证（Credentials）
   - OAuth 认证（Google, GitHub, GitLab 等）
   - 企业 SSO（Azure AD, Okta, Auth0, Keycloak 等）
   - 邮箱验证码认证（Email Magic Link）
   - 自定义 OIDC 提供商

2. **API 密钥认证**
   - 项目级 API 密钥（PROJECT scope）
   - 组织级 API 密钥（ORGANIZATION scope）
   - 基于 SHA-256 的快速密钥验证
   - Redis 缓存加速

3. **基于角色的访问控制（RBAC）**
   - 五种内置角色：OWNER, ADMIN, MEMBER, VIEWER, NONE
   - 56 项目级操作权限
   - 8+ 组织级操作权限
   - 细粒度资源访问控制

4. **会话管理**
   - NextAuth.js 集成
   - JWT token 管理
   - Cookie 安全配置
   - 多设备登录支持

5. **企业级功能（EE）**
   - 多租户 SSO
   - 域名强制 SSO
   - 组织创建白名单
   - SSO 域名黑名单

## 目录结构

```
web/src/
├── server/
│   ├── auth.ts                              # NextAuth 配置和认证选项
│   └── utils/cookies.ts                     # Cookie 配置
├── features/
│   ├── auth/                                # 认证功能
│   │   ├── lib/
│   │   │   └── createProjectMembershipsOnSignup.ts
│   │   └── constants.ts
│   ├── auth-credentials/                    # 用户名密码认证
│   │   ├── lib/credentialsServerUtils.ts    # 密码加密验证
│   │   └── server/signupApiHandler.ts       # 注册处理
│   ├── rbac/                                # RBAC 权限控制
│   │   ├── constants/
│   │   │   ├── projectAccessRights.ts       # 项目级权限定义
│   │   │   ├── organizationAccessRights.ts  # 组织级权限定义
│   │   │   └── orderedRoles.ts              # 角色排序
│   │   ├── utils/
│   │   │   ├── checkProjectAccess.ts        # 项目权限检查
│   │   │   └── checkOrganizationAccess.ts   # 组织权限检查
│   │   ├── server/
│   │   │   ├── membersRouter.ts             # 成员管理路由
│   │   │   ├── allMembersRoutes.ts
│   │   │   └── allInvitesRoutes.ts
│   │   └── components/                      # 前端组件
│   └── public-api/
│       └── server/
│           └── apiAuth.ts                   # API 密钥认证服务
├── pages/
│   └── api/
│       └── auth/
│           └── [...nextauth].ts             # NextAuth API 路由
└── ee/features/
    └── multi-tenant-sso/                    # 企业多租户 SSO
        └── utils.ts

packages/shared/src/server/
├── auth/
│   ├── apiKeys.ts                           # API 密钥生成和管理
│   ├── userProjectRoleAuth.ts               # 用户项目角色查询
│   ├── customSsoProvider.ts                 # 自定义 SSO 提供商
│   ├── gitHubEnterpriseProvider.ts
│   └── jumpcloudProvider.ts
└── ...
```

## 核心流程

- [流程1：用户登录认证](./01-user-login-sequence.puml) - 用户使用凭证或 OAuth 登录
- [流程2：API 密钥认证](./02-api-key-auth-sequence.puml) - API 请求的密钥验证
- [流程3：RBAC 权限检查](./03-rbac-permission-check-sequence.puml) - 资源访问权限验证
- [流程4：SSO 单点登录](./04-sso-login-sequence.puml) - 企业 SSO 登录流程

## 相关文档

- [认证机制详解](./authentication-mechanism.md)
- [RBAC 权限体系](./rbac-system.md)
- [API 密钥管理](./api-key-management.md)

## 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| NextAuth.js | 4.24.x | 认证框架 |
| Prisma | 最新 | 数据库 ORM |
| bcrypt | 最新 | 密码加密 |
| Redis | 最新 | 缓存（可选） |
| crypto | Node.js 内置 | 密钥哈希 |

## 依赖关系

### 依赖模块
- **数据库层（Prisma）**: 用户、组织、项目、API 密钥数据存储
- **缓存层（Redis）**: API 密钥缓存，提高验证性能
- **邮件服务**: 发送验证码和重置密码邮件
- **企业功能（EE）**: 多租户 SSO 配置

### 被依赖模块
- **所有 API 路由**: 依赖认证和权限检查
- **tRPC 路由**: 使用 session 和权限中间件
- **前端组件**: 使用 useSession 和权限检查 hooks

## 角色权限矩阵

### 组织级角色权限

| 角色 | 权限范围 |
|------|---------|
| **OWNER** | 完全控制：创建/删除组织、管理成员、管理 API 密钥、项目转移、账单管理 |
| **ADMIN** | 管理权限：创建项目、管理成员、管理 API 密钥、项目转移 |
| **MEMBER** | 基础权限：查看组织成员 |
| **VIEWER** | 无组织级权限 |
| **NONE** | 无组织级权限 |

### 项目级角色权限

| 角色 | 权限范围 |
|------|---------|
| **OWNER** | 完全控制：70+ 项操作权限，包括删除项目、管理成员、所有资源的 CRUD |
| **ADMIN** | 管理权限：除删除项目外的所有权限，包括成员管理、资源 CRUD |
| **MEMBER** | 标准权限：创建和编辑资源，无管理权限 |
| **VIEWER** | 只读权限：仅查看项目资源 |
| **NONE** | 无项目权限 |

## API 接口

### 认证相关
- `POST /api/auth/signin` - 用户登录
- `POST /api/auth/signout` - 用户登出
- `POST /api/auth/signup` - 用户注册
- `POST /api/auth/callback/[provider]` - OAuth 回调

### API 密钥管理
- `GET /api/trpc/projectApiKey.byProjectId` - 获取项目 API 密钥
- `POST /api/trpc/projectApiKey.create` - 创建 API 密钥
- `DELETE /api/trpc/projectApiKey.delete` - 删除 API 密钥

### 成员管理
- `GET /api/trpc/members.all` - 获取成员列表
- `POST /api/trpc/members.create` - 添加成员
- `PUT /api/trpc/members.update` - 更新成员角色
- `DELETE /api/trpc/members.delete` - 删除成员

## 配置选项

### 环境变量（认证相关）

```bash
# NextAuth 配置
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key

# 密码认证
AUTH_DISABLE_USERNAME_PASSWORD=false  # 是否禁用密码登录
SALT=your-salt                        # 密钥哈希盐值

# OAuth 提供商
AUTH_GOOGLE_CLIENT_ID=xxx
AUTH_GOOGLE_CLIENT_SECRET=xxx
AUTH_GITHUB_CLIENT_ID=xxx
AUTH_GITHUB_CLIENT_SECRET=xxx

# 企业 SSO
AUTH_CUSTOM_CLIENT_ID=xxx
AUTH_CUSTOM_CLIENT_SECRET=xxx
AUTH_CUSTOM_ISSUER=https://your-idp.com

# 组织创建限制
LANGFUSE_ALLOWED_ORGANIZATION_CREATORS=user1@example.com,user2@example.com

# Redis（可选，用于 API 密钥缓存）
REDIS_CONNECTION_STRING=redis://localhost:6379
```

## 安全考虑

1. **密码安全**
   - 使用 bcrypt 进行密码哈希
   - 不存储明文密码
   - 支持密码强度策略

2. **API 密钥安全**
   - 密钥使用 SHA-256 快速哈希
   - 密钥仅在创建时显示一次
   - 支持密钥过期和撤销

3. **会话安全**
   - HTTP-Only Cookie
   - CSRF 保护
   - Secure Cookie（生产环境）

4. **SSO 安全**
   - OIDC 标准协议
   - Token 验证
   - 域名强制策略

5. **权限隔离**
   - 最小权限原则
   - 资源级访问控制
   - 操作审计日志

## 扩展点

1. **自定义认证提供商**
   - 可添加新的 OAuth 提供商
   - 支持自定义 OIDC 配置
   - 可扩展认证回调逻辑

2. **自定义权限**
   - 可扩展 ProjectScope 和 OrganizationScope
   - 可添加新的角色类型
   - 可实现自定义权限检查逻辑

3. **审计和监控**
   - 登录事件追踪
   - API 调用审计
   - 权限变更日志

## 测试覆盖

- **单元测试**: `web/src/__tests__/api-auth.servertest.ts`
- **E2E 测试**: `web/src/__e2e__/auth.spec.ts`
- **测试场景**:
  - 密码登录
  - OAuth 登录
  - API 密钥验证
  - 权限检查
  - 会话管理

## 注意事项

1. **首次部署**: 需要配置 `NEXTAUTH_SECRET` 和 `SALT`
2. **OAuth 配置**: 需要在各提供商创建应用并配置回调 URL
3. **Redis 可选**: 不使用 Redis 时 API 密钥验证会直接查询数据库
4. **域名 SSO**: 企业版功能，需要配置多租户 SSO
5. **角色变更**: 更改用户角色后，需要用户重新登录才能生效
