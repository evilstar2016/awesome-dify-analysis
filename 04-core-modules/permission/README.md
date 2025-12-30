# Permission 权限控制模块

## 模块概述

Permission 是 Dify 的权限管理系统，采用基于角色的访问控制（RBAC）模型。定义了 Owner、Admin、Editor、Normal、Dataset Operator 五种租户角色，通过装饰器模式实现了细粒度的权限验证，支持多租户隔离和资源级权限控制。

**核心代码位置**：
- `api/models/account.py` - 角色定义
- `api/controllers/console/wraps.py` - 权限装饰器
- `api/controllers/console/workspace/` - 工作空间权限控制

## 目录文件说明

| 文件名 | 描述 |
|--------|------|
| [account_permission_management_INDEX.md](./account_permission_management_INDEX.md) | 权限管理文档索引 |
| [account_permission_management.md](./account_permission_management.md) | 权限管理核心文档 |
| [account_permission_management_implementation_details.md](./account_permission_management_implementation_details.md) | 权限实现细节文档 |
| [account_permission_management_quick_reference.md](./account_permission_management_quick_reference.md) | 权限快速参考手册 |
| [account_permission_management_architecture.puml](./account_permission_management_architecture.puml) | 权限架构图（PlantUML） |
| [account_permission_management_matrix.puml](./account_permission_management_matrix.puml) | 权限矩阵图（PlantUML） |
| [account_permission_management_sequence_1_login.puml](./account_permission_management_sequence_1_login.puml) | 登录时序图 |
| [account_permission_management_sequence_2_invite.puml](./account_permission_management_sequence_2_invite.puml) | 邀请成员时序图 |
| [account_permission_management_sequence_3_role_update.puml](./account_permission_management_sequence_3_role_update.puml) | 角色更新时序图 |
| [account_permission_management_sequence_4_owner_transfer.puml](./account_permission_management_sequence_4_owner_transfer.puml) | 所有者转移时序图 |
| [commercial_enterprise_features.md](./commercial_enterprise_features.md) | 商业版企业功能说明 |
| [Q&A.md](./Q&A.md) | 权限相关常见问题 |

## 相关模块

- [Account Service](../../../api/services/account_service.py) - 账户与租户管理
- [Console Controllers](../../../api/controllers/console/) - 控制台权限控制
