# KnowledgeBase 知识库模块

## 模块概述

KnowledgeBase 是 Dify 的知识库与 RAG（检索增强生成）系统，负责文档的解析、分片、向量化、索引和检索。支持多种文档格式（PDF、Word、Excel、Notion 等），集成多种向量数据库，提供灵活的检索策略和 Rerank 优化。

**核心代码位置**：`api/core/rag/`

## 目录文件说明

### 主目录文件

| 文件名 | 描述 |
|--------|------|
| [index_method.md](./index_method.md) | 索引方法说明文档 |
| [index_compare.md](./index_compare.md) | 索引方法对比分析 |
| [rerank_logic.md](./rerank_logic.md) | Rerank 重排序逻辑说明 |

### document_process 子目录

| 文件名 | 描述 |
|--------|------|
| [dify_document_types_analysis.md](./document_process/dify_document_types_analysis.md) | Dify 支持的文档类型分析 |
| [pdf_processing_technical_analysis.md](./document_process/pdf_processing_technical_analysis.md) | PDF 文档处理技术分析 |
| [word_document_processing_analysis.md](./document_process/word_document_processing_analysis.md) | Word 文档处理分析 |
| [online_document_import_analysis.md](./document_process/online_document_import_analysis.md) | 在线文档导入分析 |
| [pdf_document_import_sequence.puml](./document_process/pdf_document_import_sequence.puml) | PDF 文档导入时序图 |
| [word_document_import_flow.puml](./document_process/word_document_import_flow.puml) | Word 文档导入流程图 |
| [excel_document_processing_flow.puml](./document_process/excel_document_processing_flow.puml) | Excel 文档处理流程图 |
| [excel_import_flow.puml](./document_process/excel_import_flow.puml) | Excel 导入流程图 |
| [notion_integration_flow.puml](./document_process/notion_integration_flow.puml) | Notion 集成流程图 |
| [online_document_import_flow.puml](./document_process/online_document_import_flow.puml) | 在线文档导入流程图 |

## 相关模块

- [Model Runtime](../model_runtime/) - Embedding 模型集成
- [Workflow 工作流](../workflow/) - 知识库检索节点
- [Agent 智能体](../agent/) - Agent 工具集成
