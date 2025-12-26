# FastGPT 知识库管理模块文档

## 📚 文档概览

本目录包含 FastGPT 知识库管理模块的深度分析文档，涵盖知识库创建、文档处理、向量化、检索等核心功能。

## 📖 核心文档

### [knowledge-base-management.md](./knowledge-base-management.md)
**主文档** - 知识库管理模块完整分析

**文档章节**:
1. 概述 - 知识库管理的核心职责
2. 数据模型设计 - Dataset/Collection/Data 结构
3. 知识库类型 - 普通/网站/API/飞书/语雀
4. 文档训练流程 - 分块、向量化、索引
5. 检索机制 - 向量检索、混合检索、重排序
6. API 接口 - 知识库管理相关API

### [embedding-and-retrieval.md](./embedding-and-retrieval.md)
**向量化与检索** - Embedding生成与检索策略

### [document-processing/](./document-processing/)
**文档处理子模块** - 各类文档格式解析

## 🖼️ 时序图

| 时序图 | 文件 | 说明 |
|--------|------|------|
| 知识库创建 | [01-dataset-create-sequence.puml](./01-dataset-create-sequence.puml) | 创建知识库流程 |
| 文档训练 | [02-document-upload-training-sequence.puml](./02-document-upload-training-sequence.puml) | 文档上传与训练 |
| 知识库检索 | [03-dataset-search-sequence.puml](./03-dataset-search-sequence.puml) | 检索流程 |
| 知识库删除 | [04-dataset-delete-sequence.puml](./04-dataset-delete-sequence.puml) | 删除流程 |
| Embedding生成 | [05-embedding-generation-sequence.puml](./05-embedding-generation-sequence.puml) | 向量生成 |
| 混合检索 | [06-hybrid-retrieval-sequence.puml](./06-hybrid-retrieval-sequence.puml) | 向量+全文 |
| 向量存储 | [07-vector-storage-sequence.puml](./07-vector-storage-sequence.puml) | 向量数据库操作 |

## 📂 文档处理子模块

位于 `document-processing/` 目录：

| 文档 | 说明 |
|------|------|
| [00-overview.md](./document-processing/00-overview.md) | 文档处理概览 |
| [01-txt-md-parser.md](./document-processing/01-txt-md-parser.md) | TXT/MD 解析 |
| [02-html-parser.md](./document-processing/02-html-parser.md) | HTML 解析 |
| [03-pdf-parser.md](./document-processing/03-pdf-parser.md) | PDF 解析（三级策略） |
| [04-docx-parser.md](./document-processing/04-docx-parser.md) | Word 解析 |
| [05-xlsx-parser.md](./document-processing/05-xlsx-parser.md) | Excel 解析 |
| [06-csv-parser.md](./document-processing/06-csv-parser.md) | CSV 解析 |
| [07-pptx-parser.md](./document-processing/07-pptx-parser.md) | PPT 解析 |

## 🎯 快速导航

### 场景1: 了解知识库架构

**推荐阅读路径**:
1. 📖 [knowledge-base-management.md](./knowledge-base-management.md) - 第1-2章
2. 🖼️ [01-dataset-create-sequence.puml](./01-dataset-create-sequence.puml)

### 场景2: 理解文档处理

**推荐阅读路径**:
1. 📖 [document-processing/00-overview.md](./document-processing/00-overview.md)
2. 📖 选择具体文档类型的详解

### 场景3: 理解检索机制

**推荐阅读路径**:
1. 📖 [embedding-and-retrieval.md](./embedding-and-retrieval.md)
2. 🖼️ [06-hybrid-retrieval-sequence.puml](./06-hybrid-retrieval-sequence.puml)

## 🔧 代码位置索引

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| 知识库类型 | `packages/global/core/dataset/constants.ts` | 类型定义 |
| 知识库Schema | `packages/service/core/dataset/datasetSchema.ts` | 数据模型 |
| 文档集合Schema | `packages/service/core/dataset/collectionSchema.ts` | 集合模型 |
| 数据块Schema | `packages/service/core/dataset/dataSchema.ts` | 数据块模型 |
| 文档处理 | `packages/service/worker/file/read/` | 文件解析 |
| 向量数据库 | `packages/service/common/vectorDB/` | 向量存储 |
| 检索逻辑 | `packages/service/core/dataset/search/` | 检索服务 |

## 📊 核心概念速查

### 知识库类型

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| 文件夹 | `folder` | 组织层级 |
| 普通知识库 | `dataset` | 本地文件 |
| 网站爬取 | `websiteDataset` | 网页抓取 |
| API知识库 | `apiDataset` | 动态API |
| 飞书文档 | `feishu` | 飞书集成 |
| 语雀文档 | `yuque` | 语雀集成 |

### 训练类型

| 类型 | 说明 |
|------|------|
| chunk | 直接分块向量化 |
| qa | 生成问答对 |
| image | 图片索引 |

## 🔗 相关资源

- [工作流模块](../workflow/README.md) - 知识库检索节点
- [Agent模块](../agent/README.md) - 知识库工具调用

---

**文档版本**: v1.0  
**适用版本**: FastGPT v4.9+