# KnowledgeBase Module

## Module Overview

KnowledgeBase is Dify's knowledge base and RAG (Retrieval-Augmented Generation) system, responsible for document parsing, chunking, vectorization, indexing, and retrieval. Supports multiple document formats (PDF, Word, Excel, Notion, etc.), integrates multiple vector databases, and provides flexible retrieval strategies and Rerank optimization.

**Core Code Location**: `api/core/rag/`

## Directory File Description

### Main Directory Files

| Filename | Description |
|----------|-------------|
| [index_method.md](./index_method.md) | Index method documentation |
| [index_compare.md](./index_compare.md) | Index method comparison analysis |
| [rerank_logic.md](./rerank_logic.md) | Rerank reordering logic documentation |

### document_process Subdirectory

| Filename | Description |
|----------|-------------|
| [dify_document_types_analysis.md](./document_process/dify_document_types_analysis.md) | Dify supported document types analysis |
| [pdf_processing_technical_analysis.md](./document_process/pdf_processing_technical_analysis.md) | PDF document processing technical analysis |
| [word_document_processing_analysis.md](./document_process/word_document_processing_analysis.md) | Word document processing analysis |
| [online_document_import_analysis.md](./document_process/online_document_import_analysis.md) | Online document import analysis |
| [pdf_document_import_sequence_en.puml](./document_process/pdf_document_import_sequence_en.puml) | PDF document import sequence diagram |
| [word_document_import_flow_en.puml](./document_process/word_document_import_flow_en.puml) | Word document import flow diagram |
| [excel_document_processing_flow_en.puml](./document_process/excel_document_processing_flow_en.puml) | Excel document processing flow diagram |
| [excel_import_flow_en.puml](./document_process/excel_import_flow_en.puml) | Excel import flow diagram |
| [notion_integration_flow_en.puml](./document_process/notion_integration_flow_en.puml) | Notion integration flow diagram |
| [online_document_import_flow_en.puml](./document_process/online_document_import_flow_en.puml) | Online document import flow diagram |

## Related Modules

- [Model Runtime](../model_runtime/) - Embedding model integration
- [Workflow](../workflow/) - Knowledge base retrieval node
- [Agent](../agent/) - Agent tool integration