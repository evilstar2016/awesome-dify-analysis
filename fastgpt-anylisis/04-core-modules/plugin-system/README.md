# FastGPT 插件系统模块文档

## 📚 文档概览

本目录包含 FastGPT 插件系统模块的深度分析文档，涵盖系统插件、团队插件安装、插件标签管理等功能。

## 📖 核心文档

### [plugin-system.md](./plugin-system.md)
**主文档** - 插件系统模块完整分析

**文档章节**:
1. 模块概述 - 插件系统的核心职责
2. 插件类型体系 - 系统插件、工具插件
3. 数据模型设计 - Schema 结构分析
4. 插件安装管理 - 团队插件安装机制
5. 插件标签系统 - 分类与组织
6. 与 MCP/HTTP 工具的关系
7. API 接口

## 🎯 快速导航

### 场景1: 了解插件类型

**推荐阅读路径**:
1. 📖 [plugin-system.md - 第二章](./plugin-system.md#二插件类型体系)

**关键要点**:
- 系统插件：平台提供的官方工具
- 工具插件：工作流工具复用
- MCP工具集：外部MCP服务
- HTTP工具集：外部API封装

### 场景2: 理解插件安装

**推荐阅读路径**:
1. 📖 [plugin-system.md - 第四章](./plugin-system.md#四插件安装管理)

## 🔧 代码位置索引

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| 插件类型定义 | `packages/global/core/plugin/type.ts` | 类型声明 |
| 系统工具Schema | `packages/service/core/plugin/tool/systemToolSchema.ts` | 系统插件 |
| 团队安装Schema | `packages/service/core/plugin/schema/teamInstalledPluginSchema.ts` | 安装记录 |
| 标签Schema | `packages/service/core/plugin/tool/tagSchema.ts` | 插件标签 |
| 插件分组 | `packages/service/core/plugin/tool/pluginGroupSchema.ts` | 插件分组 |

## 📊 核心概念速查

### 插件状态

| 状态 | 枚举值 | 说明 |
|------|--------|------|
| 正常 | `1` | 可正常使用 |
| 即将下线 | `2` | 即将停用 |
| 已下线 | `3` | 已停用 |

## 🔗 相关资源

- [MCP工具模块分析](../mcp-tools-plugins/README.md) - MCP协议工具
- [应用管理模块](../app-management/README.md) - 工具应用类型
- [工作流模块分析](../workflow/README.md) - 工具节点

---

**文档版本**: v1.0  
**创建日期**: 2024-12-26  
**适用版本**: FastGPT v4.9+
