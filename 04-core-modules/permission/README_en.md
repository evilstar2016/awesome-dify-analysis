# Permission Module

## Module Overview

Permission is Dify's permission management system, using Role-Based Access Control (RBAC) model. It defines five tenant roles: Owner, Admin, Editor, Normal, Dataset Operator, implements fine-grained permission verification through decorator pattern, and supports multi-tenant isolation and resource-level permission control.

**Core Code Location**:
- `api/models/account.py` - Role definitions
- `api/controllers/console/wraps.py` - Permission decorators
- `api/controllers/console/workspace/` - Workspace permission control

## Directory File Description

| Filename | Description |
|----------|-------------|
| [account_permission_management_INDEX.md](./account_permission_management_INDEX.md) | Permission management documentation index |
| [account_permission_management.md](./account_permission_management.md) | Permission management core documentation |
| [account_permission_management_implementation_details.md](./account_permission_management_implementation_details.md) | Permission implementation details documentation |
| [account_permission_management_quick_reference.md](./account_permission_management_quick_reference.md) | Permission quick reference manual |
| [account_permission_management_architecture.puml](./account_permission_management_architecture.puml) | Permission architecture diagram (PlantUML) |
| [account_permission_management_matrix.puml](./account_permission_management_matrix.puml) | Permission matrix diagram (PlantUML) |
| [account_permission_management_sequence_1_login.puml](./account_permission_management_sequence_1_login.puml) | Login sequence diagram |
| [account_permission_management_sequence_2_invite.puml](./account_permission_management_sequence_2_invite.puml) | Invite member sequence diagram |
| [account_permission_management_sequence_3_role_update.puml](./account_permission_management_sequence_3_role_update.puml) | Role update sequence diagram |
| [account_permission_management_sequence_4_owner_transfer.puml](./account_permission_management_sequence_4_owner_transfer.puml) | Owner transfer sequence diagram |
| [commercial_enterprise_features.md](./commercial_enterprise_features.md) | Commercial enterprise features documentation |
| [Q&A.md](./Q&A.md) | Permission related FAQs |

## Related Modules

- [Account Service](../../../api/services/account_service.py) - Account and tenant management
- [Console Controllers](../../../api/controllers/console/) - Console permission control