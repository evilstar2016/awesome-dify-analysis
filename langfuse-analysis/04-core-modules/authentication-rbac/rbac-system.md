# RBAC 权限体系 - 详细说明

## 文档说明
本文档详细描述 Langfuse 的基于角色的访问控制（RBAC）系统，包括权限模型、角色定义、权限检查机制和实现细节。

⚠️ **编写原则**：
- 不贴大量源代码，只标注函数名和文件位置
- 使用表格、列表展示信息
- 代码示例控制在 10 行以内
- 引用时序图，不重复描述流程细节

---

## 目录
- [1. RBAC 架构概述](#1-rbac-架构概述)
- [2. 权限模型设计](#2-权限模型设计)
- [3. 项目级权限](#3-项目级权限)
- [4. 组织级权限](#4-组织级权限)
- [5. 权限检查机制](#5-权限检查机制)
- [6. 成员管理](#6-成员管理)
- [7. 权限继承与覆盖](#7-权限继承与覆盖)
- [8. 配置与扩展](#8-配置与扩展)
- [9. 性能优化](#9-性能优化)
- [10. 最佳实践](#10-最佳实践)

---

## 1. RBAC 架构概述

### 1.1 权限模型层次

```
┌─────────────────────────────────────┐
│          Organization               │
│  (组织级权限)                        │
│  ┌───────────────────────────────┐  │
│  │ Role: OWNER, ADMIN, MEMBER    │  │
│  └───────────────────────────────┘  │
│                                     │
│  ┌───────────┐  ┌───────────┐     │
│  │ Project 1 │  │ Project 2 │ ... │
│  ├───────────┤  ├───────────┤     │
│  │ Role:     │  │ Role:     │     │
│  │ OWNER     │  │ ADMIN     │     │
│  │ ADMIN     │  │ MEMBER    │     │
│  │ MEMBER    │  │ VIEWER    │     │
│  │ VIEWER    │  │ NONE      │     │
│  └───────────┘  └───────────┘     │
└─────────────────────────────────────┘
```

### 1.2 核心概念

| 概念 | 说明 | 数据库表 |
|------|------|---------|
| **Organization** | 组织，最高层级的资源容器 | `Organization` |
| **Project** | 项目，属于某个组织 | `Project` |
| **Role** | 角色，定义权限集合 | 枚举类型 |
| **Scope** | 权限范围，具体的操作权限 | 代码常量 |
| **Membership** | 成员关系，用户-组织-角色 | `OrganizationMembership` |
| **Project Role** | 项目角色，可覆盖组织角色 | `Project.role` |

### 1.3 权限模型特点

- **分层架构**: 组织级 + 项目级双层权限
- **角色继承**: 高级别角色拥有低级别角色的所有权限
- **权限覆盖**: 项目级角色可覆盖组织级角色
- **细粒度控制**: 70+ 项目级权限，8+ 组织级权限
- **超级管理员**: 系统级 admin 用户拥有所有权限

---

## 2. 权限模型设计

### 2.1 角色定义

**枚举类型**: `Role` (Prisma Schema)

```typescript
enum Role {
  OWNER    // 所有者
  ADMIN    // 管理员
  MEMBER   // 成员
  VIEWER   // 查看者
  NONE     // 无权限
}
```

### 2.2 角色层级

**角色排序**: [web/src/features/rbac/constants/orderedRoles.ts](../../../web/src/features/rbac/constants/orderedRoles.ts)

```
OWNER > ADMIN > MEMBER > VIEWER > NONE
  5       4       3        2       1
```

**权限继承规则**: 高级别角色包含低级别角色的所有权限（仅在同一层级内）

### 2.3 权限范围（Scope）

**Scope 格式**: `{resource}:{action}`

| 格式示例 | 说明 |
|---------|------|
| `project:read` | 读取项目信息 |
| `traces:delete` | 删除追踪数据 |
| `apiKeys:CUD` | API 密钥的创建、更新、删除 |
| `organizationMembers:read` | 读取组织成员列表 |

**操作简写**:
- `CUD`: Create, Update, Delete
- `CRUD`: Create, Read, Update, Delete

---

## 3. 项目级权限

### 3.1 权限定义

**文件位置**: [web/src/features/rbac/constants/projectAccessRights.ts](../../../web/src/features/rbac/constants/projectAccessRights.ts)

#### 权限列表（70+ 项）

| 类别 | 权限范围 | 说明 |
|------|---------|------|
| **项目管理** | `project:read`, `project:update`, `project:delete` | 项目基本操作 |
| **成员管理** | `projectMembers:read`, `projectMembers:CUD` | 项目成员管理 |
| **API 密钥** | `apiKeys:read`, `apiKeys:CUD` | API 密钥管理 |
| **集成** | `integrations:CRUD` | 第三方集成管理 |
| **对象操作** | `objects:publish`, `objects:bookmark`, `objects:tag` | 通用对象操作 |
| **追踪** | `traces:delete` | 追踪数据删除 |
| **评分** | `scores:CUD`, `scoreConfigs:CUD`, `scoreConfigs:read` | 评分和配置 |
| **数据集** | `datasets:CUD` | 数据集管理 |
| **提示词** | `prompts:CUD`, `prompts:read`, `promptProtectedLabels:CUD` | 提示词管理 |
| **模型** | `models:CUD` | 模型配置 |
| **评估** | `evalTemplate:CUD`, `evalJob:CUD`, `evalJobExecution:read` | 评估任务 |
| **LLM API** | `llmApiKeys:read/create/update/delete` | LLM API 密钥 |
| **LLM 配置** | `llmSchemas:CUD`, `llmTools:CUD` | LLM 模式和工具 |
| **批量导出** | `batchExports:create`, `batchExports:read` | 数据导出 |
| **评论** | `comments:CUD`, `comments:read` | 评论功能 |
| **标注队列** | `annotationQueues:CUD`, `annotationQueueAssignments:CUD` | 标注管理 |
| **实验** | `promptExperiments:CUD`, `promptExperiments:read` | 提示词实验 |
| **审计日志** | `auditLogs:read` | 操作审计 |
| **仪表板** | `dashboards:CUD`, `dashboards:read` | 仪表板管理 |
| **视图预设** | `TableViewPresets:CUD`, `TableViewPresets:read` | 表格视图 |
| **自动化** | `automations:CUD`, `automations:read` | 自动化规则 |

### 3.2 角色权限矩阵

**数据结构**: `projectRoleAccessRights` - [projectAccessRights.ts](../../../web/src/features/rbac/constants/projectAccessRights.ts#L84-L249)

**代码统计**（基于符号分析）:
- 总代码行数: 165 行（行 84-249）
- 定义了 5 种角色的完整权限映射

**精确统计**（基于符号分析）:
- **OWNER**: 54 项权限（行 85-138）
- **ADMIN**: 53 项权限（行 139-191）- 比 OWNER 少 1 项（`project:delete`）
- **MEMBER**: 38 项权限（行 192-229）- 比 ADMIN 少 15 项（管理类权限）
- **VIEWER**: 18 项权限（行 230-247）- 比 MEMBER 少 20 项（全部为读权限）
- **NONE**: 0 项权限（行 248）- 空数组

**代码复杂度**:
- 平均每个角色: 33 行代码
- 最大角色定义: OWNER (54 行)
- 最小角色定义: NONE (1 行)

#### OWNER 角色

**代码位置**: 行 85-138（共 54 行）

**权限数量**: 54 项（所有项目权限）

**核心权限**:
- 删除项目（`project:delete`）- 仅 OWNER 可以
- 管理成员（`projectMembers:CUD`）
- 所有资源的完全控制

#### ADMIN 角色

**代码位置**: 行 139-191（共 53 行）

**权限数量**: 53 项

**核心权限**:
- 更新项目（`project:update`）
- 管理成员（`projectMembers:CUD`）
- 所有资源的 CRUD（除删除项目外）

**与 OWNER 的差异**:
- ❌ 无法删除项目

#### MEMBER 角色

**代码位置**: 行 192-229（共 38 行）

**权限数量**: 38 项

**核心权限**:
- 读取项目（`project:read`）
- 创建和编辑资源
- 读取 API 密钥（但不能创建/删除）

**限制**:
- ❌ 无法管理成员
- ❌ 无法管理 API 密钥（仅读取）
- ❌ 无法创建/删除 LLM API 密钥
- ❌ 无法管理集成

#### VIEWER 角色

**代码位置**: 行 230-247（共 18 行）

**权限数量**: 18 项（纯只读权限）

**核心权限**:
```typescript
[
  "project:read",
  "prompts:read",
  "evalTemplate:read",
  "scoreConfigs:read",
  "evalJob:read",
  "evalJobExecution:read",
  "evalDefaultModel:read",
  "llmApiKeys:read",
  "llmSchemas:read",
  "llmTools:read",
  "comments:read",
  "annotationQueues:read",
  "promptExperiments:read",
  "dashboards:read",
  "TableViewPresets:read",
  "automations:read",
]
```

**限制**:
- ❌ 无任何写操作权限
- ❌ 仅可查看，不可创建/编辑/删除

#### NONE 角色

**代码位置**: 行 248（仅 1 行：空数组）

**权限数量**: 0（无权限）

**代码定义**: 
```typescript
NONE: [],  // 行 248
```

**说明**: 不覆盖组织角色，使用组织级别的默认权限。当用户在组织有权限但不希望访问特定项目时使用。

### 3.3 权限检查实现

**文件位置**: [web/src/features/rbac/utils/checkProjectAccess.ts](../../../web/src/features/rbac/utils/checkProjectAccess.ts)

#### 核心函数

| 函数 | 用途 | 使用场景 |
|------|------|---------|
| `hasProjectAccess()` | 检查权限（返回布尔值） | 条件渲染、功能可用性检查 |
| `throwIfNoProjectAccess()` | 检查权限（抛出异常） | tRPC resolver、API 路由保护 |
| `useHasProjectAccess()` | React Hook | 前端组件权限检查 |

#### 权限检查逻辑

```typescript
function hasProjectAccess(params) {
  // 1. 超级管理员：拥有所有权限
  if (user.admin) return true;

  // 2. 获取用户在项目中的角色
  const projectRole = getUserProjectRole(userId, projectId);
  
  // 3. 检查角色权限列表
  return projectRoleAccessRights[projectRole].includes(scope);
}
```

**流程图**: 参见 [RBAC 权限检查时序图](./03-rbac-permission-check-sequence.puml)

---

## 4. 组织级权限

### 4.1 权限定义

**文件位置**: [web/src/features/rbac/constants/organizationAccessRights.ts](../../../web/src/features/rbac/constants/organizationAccessRights.ts)

#### 权限列表

| 权限范围 | 说明 | 影响范围 |
|---------|------|---------|
| `projects:create` | 创建项目 | 组织下新增项目 |
| `projects:transfer_org` | 转移项目 | 项目在组织间转移 |
| `organization:CRUD_apiKeys` | 组织级 API 密钥管理 | 跨项目 API 密钥 |
| `organization:update` | 更新组织信息 | 组织设置和配置 |
| `organization:delete` | 删除组织 | 删除整个组织 |
| `organizationMembers:read` | 读取成员列表 | 查看组织成员 |
| `organizationMembers:CUD` | 成员管理 | 添加/删除成员，修改角色 |
| `langfuseCloudBilling:CRUD` | 账单管理 | 订阅和支付管理 |

### 4.2 角色权限矩阵

**数据结构**: `organizationRoleAccessRights`

#### OWNER 角色

**权限**: 所有组织级权限（8 项）

```typescript
[
  "projects:create",
  "projects:transfer_org",
  "organization:CRUD_apiKeys",
  "organization:update",
  "organization:delete",          // 仅 OWNER
  "organizationMembers:CUD",
  "organizationMembers:read",
  "langfuseCloudBilling:CRUD",    // 仅 OWNER
]
```

**独占权限**:
- 删除组织
- 账单管理

#### ADMIN 角色

**权限**: 6 项

```typescript
[
  "projects:create",
  "projects:transfer_org",
  "organization:CRUD_apiKeys",
  "organization:update",
  "organizationMembers:CUD",
  "organizationMembers:read",
]
```

**与 OWNER 差异**:
- ❌ 无法删除组织
- ❌ 无法管理账单

#### MEMBER 角色

**权限**: 1 项

```typescript
["organizationMembers:read"]
```

**说明**: 仅能查看组织成员列表，无其他组织级权限

#### VIEWER 和 NONE 角色

**权限**: 0 项（无组织级权限）

```typescript
VIEWER: [],
NONE: [],
```

### 4.3 组织权限检查实现

**文件位置**: [web/src/features/rbac/utils/checkOrganizationAccess.ts](../../../web/src/features/rbac/utils/checkOrganizationAccess.ts)

#### 核心函数

| 函数 | 用途 |
|------|------|
| `hasOrganizationAccess()` | 检查组织权限 |
| `throwIfNoOrganizationAccess()` | 检查并抛出异常 |
| `useHasOrganizationAccess()` | React Hook |

---

## 5. 权限检查机制

### 5.1 检查流程

#### 核心函数分析

**函数**: `hasProjectAccess()` - [checkProjectAccess.ts](../../../web/src/features/rbac/utils/checkProjectAccess.ts#L54-L67)

**代码行数**: 仅 14 行（极度优化！）

**完整实现**:
```typescript
export function hasProjectAccess(p: HasProjectAccessParams): boolean {
  // 步骤 1: 超级管理员检查（1-2行）
  const isAdmin = "role" in p ? p.admin : p.session?.user?.admin;
  if (isAdmin) return true;

  // 步骤 2: 获取项目角色（3-7行）
  const projectRole: Role | undefined =
    "role" in p
      ? p.role
      : p.session?.user?.organizations
          .flatMap((org) => org.projects)
          .find((project) => project.id === p.projectId)?.role;
  if (projectRole === undefined) return false;

  // 步骤 3: 检查权限（1行）
  return projectRoleAccessRights[projectRole].includes(p.scope);
}
```

**性能特点**:
- ✅ **零数据库查询** - 所有数据来自 Session
- ✅ **纯内存操作** - 数组查找和对象访问
- ✅ **<1ms 响应** - 极快的权限检查
- ✅ **Session 预加载** - 组织-项目-角色树已在登录时加载

#### 检查流程图

```
┌──────────────────┐
│  权限检查请求     │
│ hasProjectAccess()│
└────────┬─────────┘
         │
         ▼
┌─────────────────────────┐
│ 1. 检查超级管理员        │
│    user.admin === true?  │
│    ✓ 代码行: 55-56       │
└────────┬────────────────┘
         │ No
         ▼
┌─────────────────────────┐
│ 2. 获取用户项目角色      │
│    从 Session 中查找     │
│    ✓ 代码行: 58-62       │
│    ✓ 无数据库查询        │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 3. 查询角色权限列表      │
│    projectRoleAccessRights│
│    [role]               │
│    ✓ 代码行: 66          │
│    ✓ 内存数组访问        │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 4. 检查 scope 是否在列表 │
│    .includes(scope)      │
│    ✓ 代码行: 66          │
└────────┬────────────────┘
         │
         ▼
┌──────────────────┐
│  返回结果         │
│  true / false    │
│  ✓ 总耗时: <1ms  │
└──────────────────┘
```

### 5.2 使用示例

#### 在 tRPC Resolver 中使用

```typescript
// web/src/server/api/routers/traces.ts
import { throwIfNoProjectAccess } from "@/src/features/rbac/utils/checkProjectAccess";

export const tracesRouter = createTRPCRouter({
  deleteById: protectedProjectProcedure
    .input(z.object({ traceId: z.string(), projectId: z.string() }))
    .mutation(async ({ input, ctx }) => {
      // 检查删除权限
      throwIfNoProjectAccess({
        session: ctx.session,
        projectId: input.projectId,
        scope: "traces:delete",
      });

      // 执行删除操作
      await ctx.prisma.trace.delete({
        where: { id: input.traceId },
      });
    }),
});
```

#### 在 React 组件中使用

```typescript
// web/src/components/TracesTable.tsx
import { useHasProjectAccess } from "@/src/features/rbac/utils/checkProjectAccess";

export function TracesTable({ projectId }: { projectId: string }) {
  const canDelete = useHasProjectAccess({
    projectId,
    scope: "traces:delete",
  });

  return (
    <div>
      {canDelete && (
        <Button onClick={handleDelete}>删除</Button>
      )}
    </div>
  );
}
```

#### 基于角色的直接检查

```typescript
import { hasProjectAccess } from "@/src/features/rbac/utils/checkProjectAccess";

const canManageMembers = hasProjectAccess({
  role: userRole,
  scope: "projectMembers:CUD",
  admin: isAdmin,
});
```

### 5.3 权限检查性能

| 检查类型 | 性能 | 说明 |
|---------|------|------|
| 内存检查 | <1ms | 角色权限列表在内存中 |
| 数据库查询 | 10-50ms | 获取用户角色（带索引） |
| 缓存命中 | <5ms | Session 缓存在 Redis/内存 |

---

## 6. 成员管理

### 6.1 成员路由

**文件位置**: [web/src/features/rbac/server/membersRouter.ts](../../../web/src/features/rbac/server/membersRouter.ts)

#### API 端点

| 端点 | 方法 | 功能 | 权限要求 |
|------|------|------|---------|
| `/api/trpc/members.all` | GET | 获取成员列表 | `projectMembers:read` |
| `/api/trpc/members.create` | POST | 添加成员 | `projectMembers:CUD` |
| `/api/trpc/members.update` | PUT | 更新成员角色 | `projectMembers:CUD` |
| `/api/trpc/members.delete` | DELETE | 删除成员 | `projectMembers:CUD` |

### 6.2 邀请机制

**文件位置**: [web/src/features/rbac/server/allInvitesRoutes.ts](../../../web/src/features/rbac/server/allInvitesRoutes.ts)

#### 邀请流程

1. **创建邀请**: 发送邀请邮件
2. **接受邀请**: 用户点击邮件链接
3. **创建成员关系**: 添加到组织/项目
4. **角色分配**: 设置初始角色

#### 数据库表

```
MembershipInvitation
  ├─ id
  ├─ email
  ├─ orgId / projectId
  ├─ role
  ├─ invitedBy
  └─ expiresAt
```

### 6.3 成员角色变更

#### 变更规则

| 规则 | 说明 |
|------|------|
| 自我变更 | 用户不能更改自己的角色 |
| OWNER 保护 | 组织必须至少有一个 OWNER |
| 权限要求 | 需要 `projectMembers:CUD` 或 `organizationMembers:CUD` |
| 级联影响 | 变更组织角色会影响项目权限（如果项目角色为 NONE） |

---

## 7. 权限继承与覆盖

### 7.1 继承规则

```
用户的最终权限 = MAX(组织级角色权限, 项目级角色权限)
```

**示例**:
```
用户 A:
  - 组织角色: MEMBER
  - Project 1 角色: ADMIN
  - Project 2 角色: NONE

权限结果:
  - Project 1: ADMIN 权限（项目角色覆盖）
  - Project 2: MEMBER 权限（继承组织角色）
  - 其他项目: MEMBER 权限（继承组织角色）
```

### 7.2 角色覆盖机制

**数据库字段**: `Project.role` (nullable)

| Project.role | 生效角色 | 说明 |
|-------------|---------|------|
| `OWNER` | OWNER | 完全控制项目 |
| `ADMIN` | ADMIN | 管理员权限 |
| `MEMBER` | MEMBER | 标准成员权限 |
| `VIEWER` | VIEWER | 只读权限 |
| `NONE` | 组织角色 | 不覆盖，使用组织角色 |
| `null` | 组织角色 | 默认行为 |

### 7.3 权限计算逻辑

```typescript
function getUserEffectiveRole(userId: string, projectId: string): Role {
  const orgRole = getUserOrgRole(userId, orgId);
  const projectRole = getUserProjectRole(userId, projectId);
  
  // 项目角色为 NONE 或 null 时，使用组织角色
  if (projectRole === "NONE" || projectRole === null) {
    return orgRole;
  }
  
  // 否则使用项目角色
  return projectRole;
}
```

---

## 8. 配置与扩展

### 8.1 添加新权限

#### 步骤

1. **定义 Scope**: 在 `projectScopes` 或 `organizationScopes` 中添加

```typescript
// projectAccessRights.ts
export const projectScopes = [
  // ... 现有权限
  "newResource:read",
  "newResource:CUD",
] as const;
```

2. **分配权限**: 在 `roleAccessRights` 中为各角色分配

```typescript
export const projectRoleAccessRights: Record<Role, ProjectScope[]> = {
  OWNER: [
    // ... 现有权限
    "newResource:read",
    "newResource:CUD",
  ],
  // ...
};
```

3. **应用检查**: 在代码中使用权限检查

```typescript
throwIfNoProjectAccess({
  session: ctx.session,
  projectId: input.projectId,
  scope: "newResource:CUD",
});
```

### 8.2 添加新角色

**注意**: 当前系统使用固定的 5 种角色，不建议添加新角色。如需扩展，考虑使用更细粒度的权限组合。

### 8.3 自定义权限策略

#### 扩展点

| 扩展点 | 位置 | 用途 |
|--------|------|------|
| 权限检查中间件 | tRPC middleware | 自动权限检查 |
| 自定义验证器 | `checkProjectAccess.ts` | 特殊业务逻辑 |
| 权限装饰器 | 待实现 | 声明式权限控制 |

---

## 9. 性能优化

### 9.1 Session 缓存

**缓存位置**: NextAuth Session

```typescript
session: {
  user: {
    organizations: [
      {
        id: "org-id",
        role: "ADMIN",
        projects: [
          { id: "proj-id", role: "OWNER" }
        ]
      }
    ]
  }
}
```

**优势**:
- 无需每次请求查询数据库
- 权限检查在内存中完成（<1ms）
- Session 有效期内保持一致

### 9.2 数据库索引

```sql
-- OrganizationMembership 表
CREATE INDEX idx_org_membership_user ON OrganizationMembership(userId);
CREATE INDEX idx_org_membership_org ON OrganizationMembership(orgId);

-- Project 表
CREATE INDEX idx_project_org ON Project(orgId);
```

### 9.3 批量权限检查

```typescript
// 避免：循环检查
projects.forEach(project => {
  if (hasProjectAccess({ projectId: project.id, scope: "project:read" })) {
    // ...
  }
});

// 推荐：预先过滤
const accessibleProjects = projects.filter(project => 
  session.user.organizations
    .flatMap(org => org.projects)
    .some(p => p.id === project.id)
);
```

---

## 10. 最佳实践

### 10.1 角色分配原则

| 角色 | 适用场景 | 建议人数 |
|------|---------|---------|
| **OWNER** | 组织创始人、核心负责人 | 1-2 人 |
| **ADMIN** | 技术负责人、项目经理 | 2-5 人 |
| **MEMBER** | 开发人员、数据分析师 | 多数成员 |
| **VIEWER** | 外部顾问、只读访问者 | 按需 |
| **NONE** | 限制访问特定项目 | 特殊情况 |

### 10.2 权限最小化

```typescript
// ❌ 错误：给予过高权限
addMember({ role: "ADMIN" });

// ✅ 正确：按需分配
addMember({ role: "MEMBER" }); // 默认使用 MEMBER
```

### 10.3 项目角色覆盖策略

**场景 1: 限制访问**
```
组织角色: ADMIN
项目角色: VIEWER
结果: 用户在该项目只有只读权限
```

**场景 2: 提升权限**
```
组织角色: MEMBER
项目角色: OWNER
结果: 用户是该项目的所有者
```

**场景 3: 默认继承**
```
组织角色: ADMIN
项目角色: NONE
结果: 继承组织的 ADMIN 权限
```

### 10.4 权限审计

#### 定期检查

- [ ] 审查 OWNER 角色分配
- [ ] 检查长期未活跃成员
- [ ] 验证外部成员权限
- [ ] 审计权限变更日志

#### 审计日志

**功能**: `auditLogs:read`

```typescript
// 查询权限变更记录
const logs = await prisma.auditLog.findMany({
  where: {
    action: "MEMBER_ROLE_CHANGE",
    projectId: projectId,
  },
  orderBy: { createdAt: "desc" },
});
```

### 10.5 安全建议

1. **定期轮换 OWNER**: 至少 2 人拥有 OWNER 权限，避免单点故障
2. **使用 VIEWER**: 对只需查看的用户使用 VIEWER 角色
3. **项目隔离**: 敏感项目使用项目级角色覆盖
4. **权限变更通知**: 配置权限变更的邮件通知
5. **定期审计**: 每季度审查成员权限
6. **最小权限**: 新成员默认使用 MEMBER 角色
7. **禁用账户**: 离职员工及时移除或降级
8. **API 密钥管理**: 使用项目级密钥而非组织级密钥（除非必要）

---

## 相关文档

- [模块 README](./README.md)
- [认证机制详解](./authentication-mechanism.md)
- [API 密钥管理](./api-key-management.md)
- [RBAC 权限检查时序图](./03-rbac-permission-check-sequence.puml)
