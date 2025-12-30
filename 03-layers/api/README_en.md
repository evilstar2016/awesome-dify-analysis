# Dify API Flow Documentation Index

This directory contains Dify backend API complete documentation and PlantUML sequence diagrams for core scenarios.

## Document List

### API Documentation

| File | Description |
|------|-------------|
| [dify_backend_api_documentation.md](./dify_backend_api_documentation.md) | Dify Backend API Complete Documentation |

### Architecture Diagrams

| File | Description |
|------|-------------|
| [dify_api_architecture.puml](./dify_api_architecture.puml) | Dify API Architecture Overview |

### Core Scenario Sequence Diagrams

| File | Description | Involved APIs |
|------|-------------|---------------|
| [chat_api_sequence.puml](./chat_api_sequence.puml) | Chat/Completion API Call Flow | `/v1/chat-messages`, `/v1/completion-messages` |
| [workflow_api_sequence.puml](./workflow_api_sequence.puml) | Workflow API Call Flow | `/v1/workflows/run`, `/v1/workflows/logs` |
| [dataset_api_sequence.puml](./dataset_api_sequence.puml) | Dataset/Knowledge Base API Call Flow | `/v1/datasets`, `/v1/documents`, `/v1/segments` |
| [file_upload_api_sequence.puml](./file_upload_api_sequence.puml) | File Upload API Call Flow | `/v1/files/upload` |
| [audio_api_sequence.puml](./audio_api_sequence.puml) | Audio Conversion API Call Flow | `/v1/audio-to-text`, `/v1/text-to-audio` |

## PlantUML Viewing Methods

### Method 1: VS Code Plugin

After installing the `PlantUML` plugin, you can preview `.puml` files directly in VS Code.

### Method 2: Online Tools

Visit [PlantUML Online Server](http://www.plantuml.com/plantuml/uml/) and paste the code to preview.

### Method 3: Command Line Generation

```bash
# Install PlantUML
# macOS: brew install plantuml
# Windows: choco install plantuml

# Generate PNG
plantuml chat_api_sequence.puml

# Generate SVG
plantuml -tsvg chat_api_sequence.puml
```

## API Category Description

| API Category | URL Prefix | Authentication | Target Users |
|-------------|------------|---------------|--------------|
| **Service API** | `/v1/` | Bearer Token | Third-party Developers |
| **Console API** | `/console/api/` | Session + JWT | Administrators/Developers |
| **Web API** | `/api/` | Passport Token | End Users |

## Core Concepts

### 1. App Mode

| Mode | Description | Applicable APIs |
|------|-------------|-----------------|
| `completion` | Text Completion | `/v1/completion-messages` |
| `chat` | Basic Chat | `/v1/chat-messages` |
| `agent_chat` | Agent Chat | `/v1/chat-messages` |
| `advanced_chat` | Advanced Chat | `/v1/chat-messages` |
| `workflow` | Workflow | `/v1/workflows/run` |

### 2. Response Mode

| Mode | Description |
|------|-------------|
| `blocking` | Blocking mode, wait for complete response |
| `streaming` | Streaming mode, return via SSE |

### 3. Indexing Technique

| Technique | Description |
|-----------|-------------|
| `high_quality` | High-quality vector indexing |
| `economy` | Economy keyword indexing |

## Related Resources

- [Dify Official Documentation](https://docs.dify.ai/)
- [API Reference](https://docs.dify.ai/guides/tools)
- [SDK Directory](../../sdks/)

---

📱 **Follow WeChat Official Account 「柒叔代码阁」**

Regularly publish Dify in-depth content and production practices~

![Official Account QR Code](../../qrcode.png)