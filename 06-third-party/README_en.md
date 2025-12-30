# Third-party Integration Overview

## Integration Overview

As an enterprise-level LLM application development platform, Dify deeply integrates various third-party services and components to build a complete AI application ecosystem. This document details all third-party integrations in Dify, including implementation methods, configuration methods, and usage guides.

## Integration Categories

### 1. [Large Language Model Integration](./01-大语言模型集成.md)
Integrates 40+ mainstream LLM providers, including OpenAI, Anthropic, Google, local models, etc., supporting unified model calling interfaces.

### 2. [Vector Database Integration](./02-向量数据库集成.md)
Supports 28+ vector databases for knowledge base vector storage and retrieval, including Milvus, Chroma, Qdrant, Weaviate, etc.

### 3. [Object Storage Integration](./03-对象存储集成.md)
Supports various object storage services for file and media resource storage management, including S3, OSS, MinIO, etc.

### 4. [Messaging Platform Integration](./04-消息平台集成.md)
Supports OAuth authentication and Webhook integration, can connect to DingTalk, Enterprise WeChat, Feishu, etc.

### 5. [Document Processing Integration](./05-文档处理集成.md)
Integrates various document parsing and web crawling services, including PDF processing, Firecrawl, Jina Reader, Unstructured, etc.

### 6. [Database and Cache Integration](./06-数据库与缓存集成.md)
Uses PostgreSQL as the main database, Redis as cache and message queue, supports multiple database dialects.

### 7. [AI Enhancement Service Integration](./07-AI增强服务集成.md)
Integrates Rerank models, speech recognition, text-to-speech, and other AI enhancement services.

### 8. [Code Execution Integration](./08-代码执行集成.md)
Provides secure code execution sandbox environment, supports Python and JavaScript code execution.

### 9. [Knowledge Base Integration](./09-知识库集成.md)
Supports external knowledge source access, including Notion, web crawling, etc.

### 10. [Authentication and Monitoring Integration](./10-认证与监控集成.md)
Integrates OAuth authentication, Sentry error monitoring, OpenTelemetry observability, etc.

## Integration Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Dify Platform                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Frontend   │  │  Backend API │  │  Worker/Task │          │
│  │   (Next.js)  │  │   (Flask)    │  │   (Celery)   │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                   │                   │
├─────────┼─────────────────┼───────────────────┼──────────────────┤
│         │                 │                   │                   │
│  ┌──────▼─────────────────▼───────────────────▼────────────┐    │
│  │              Model Runtime Layer                         │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │    │
│  │  │ OpenAI  │ │ Anthropic│ │  Google │ │  Ollama │ ...  │    │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              RAG & Vector Database Layer               │     │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │     │
│  │  │ Milvus  │ │ Qdrant  │ │ Chroma  │ │ Weaviate│ ...  │     │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘      │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              Storage & Cache Layer                     │     │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │     │
│  │  │   S3    │ │   OSS   │ │  Redis  │ │   Local │ ...  │     │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘      │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐     │
│  │          External Services Layer                       │     │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │     │
│  │  │Firecrawl │ │  Jina    │ │  Sentry  │ │  OAuth   │  │     │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

## Configuration Management

### Configuration File Location

All integration configurations are managed through environment variables and configuration classes:

- **Main Configuration**: `api/configs/__init__.py` - DifyConfig class
- **Feature Configuration**: `api/configs/feature/__init__.py` - Various feature module configurations
- **Environment Variables**: `.env.example` - Environment variable template

### Configuration Loading Mechanism

```python
# Use Pydantic Settings to manage configuration
from configs import dify_config

# Access configuration
model_provider = dify_config.DEFAULT_LLM_MODEL_PROVIDER
storage_type = dify_config.STORAGE_TYPE
vector_store = dify_config.VECTOR_STORE
```

### Configuration Priority

1. Environment variables (highest priority)
2. `.env` file
3. Configuration class default values

## Environment Variable Management

### Core Environment Variables

```bash
# Database
DB_USERNAME=postgres
DB_PASSWORD=your_password
DB_HOST=db
DB_PORT=5432
DB_DATABASE=dify

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0

# Storage
STORAGE_TYPE=s3
S3_ENDPOINT=https://s3.amazonaws.com
S3_BUCKET_NAME=dify
S3_ACCESS_KEY=your_access_key
S3_SECRET_KEY=your_secret_key

# Vector Database
VECTOR_STORE=milvus
MILVUS_HOST=milvus
MILVUS_PORT=19530

# Code Execution
CODE_EXECUTION_ENDPOINT=http://sandbox:8194
CODE_EXECUTION_API_KEY=dify-sandbox
```

### Environment Variable Validation

The project uses Pydantic Settings for configuration validation:

```python
class DifyConfig(BaseSettings):
    """Main configuration class with validation"""
    
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        frozen=True,
        extra="ignore",
    )
```

## Integration Architecture Design Principles

### 1. Adapter Pattern

All third-party service integrations use the adapter pattern to provide unified interfaces:

```python
# Model runtime adapter
class AIModel:
    """Base class for all AI model implementations"""
    pass

class LargeLanguageModel(AIModel):
    """Base class for LLM implementations"""
    
    def invoke(self, model: str, credentials: dict, 
               prompt_messages: list, **kwargs) -> LLMResult:
        """Unified invocation interface"""
        pass
```

### 2. Factory Pattern

Use factory pattern to dynamically create integration instances:

```python
class ModelProviderFactory:
    """Factory for creating model provider instances"""
    
    @staticmethod
    def get_provider_instance(provider: str):
        """Get provider instance by name"""
        pass
```

### 3. Configuration Driven

All integrations are configuration-driven, supporting dynamic switching:

```python
# Vector database factory
class VectorFactory:
    @staticmethod
    def create(vector_type: str, **kwargs):
        """Create vector store based on type"""
        if vector_type == VectorType.MILVUS:
            return MilvusVector(**kwargs)
        elif vector_type == VectorType.QDRANT:
            return QdrantVector(**kwargs)
        # ...
```

### 4. Error Handling

Unified error handling mechanism:

```python
# Model invocation error
class InvokeError(Exception):
    """Base class for model invocation errors"""
    pass

# Configuration validation error
class ProviderCredentialValidationError(Exception):
    """Provider credential validation error"""
    pass
```

## Monitoring and Logging

### Logging Configuration

```python
# Log level configured through environment variables
LOG_LEVEL=INFO
LOG_FILE=/var/log/dify/app.log
```

### Performance Monitoring

- **Sentry**: Error monitoring and performance tracking
- **OpenTelemetry**: Distributed tracing and metrics collection
- **Redis**: Cache performance monitoring

### Health Checks

```python
# Database health check
GET /api/health

# Response example
{
    "status": "healthy",
    "database": "connected",
    "redis": "connected",
    "storage": "available"
}
```

## Extension Guide

### Adding a New LLM Provider

1. Create provider directory in `api/core/model_runtime/model_providers/`
2. Implement provider class inheriting from `AIModelProvider`
3. Implement model class inheriting from `LargeLanguageModel`
4. Add configuration YAML file
5. Update `_position.yaml` to add provider order

### Adding a New Vector Database

1. Create vector database directory in `api/core/rag/datasource/vdb/`
2. Implement vector class inheriting from `BaseVector`
3. Add type to `VectorType` enum
4. Add creation logic in `VectorFactory`

### Adding a New Storage Service

1. Create storage class file in `api/extensions/storage/`
2. Implement storage class inheriting from `BaseStorage`
3. Add type to `StorageType` enum
4. Add creation logic in storage factory

## Testing Strategy

### Unit Testing

```bash
# Run all tests
uv run --project api --dev dev/pytest/pytest_unit_tests.sh

# Run specific module tests
pytest api/tests/unit_tests/core/model_runtime/
```

### Integration Testing

Integration tests run in CI environment, testing interactions with real third-party services.

### Mock Strategy

For third-party service calls, use Mock for unit testing:

```python
from unittest.mock import Mock, patch

@patch('openai.ChatCompletion.create')
def test_openai_invoke(mock_create):
    mock_create.return_value = {...}
    # Test logic
```

## Security

### API Key Management

- Use environment variables to store sensitive information
- Support encrypted storage of keys in database
- Provide key rotation mechanism

### Network Security

- Support SSRF proxy protection
- SSL/TLS encrypted connections
- IP whitelist restrictions

### Data Privacy

- Support data masking
- Comply with GDPR requirements
- Provide data deletion interface

## Performance Optimization

### Connection Pooling

- Database connection pool (SQLAlchemy)
- Redis connection pool
- HTTP client connection pool

### Caching Strategy

- Redis cache model responses
- Local cache configuration information
- CDN cache static resources

### Concurrency Control

- Celery asynchronous task queue
- Rate limiting and circuit breaker mechanisms
- Load balancing

## Troubleshooting

### Common Issues

1. **Connection Timeout**: Check network configuration and firewall rules
2. **Authentication Failure**: Verify API keys and credential configuration
3. **Quota Limits**: Check provider quotas and usage
4. **Data Inconsistency**: Clear cache and retry

### Log Analysis

```bash
# View application logs
docker logs dify-api

# View Celery worker logs
docker logs dify-worker

# View error logs
tail -f /var/log/dify/error.log
```

## Version Compatibility

| Component | Minimum Version | Recommended Version |
|-----------|-----------------|---------------------|
| Python | 3.10 | 3.11+ |
| PostgreSQL | 12 | 15+ |
| Redis | 6.0 | 7.0+ |
| Node.js | 18 | 20+ |

## Reference Resources

- [Dify Official Documentation](https://docs.dify.ai)
- [Model Provider Documentation](./01-大语言模型集成.md)
- [Vector Database Documentation](./02-向量数据库集成.md)
- [Deployment Guide](../docker/README.md)

## Update Log

- 2025-12-11: Initial version, contains documentation for all major integration categories
- Support 40+ LLM providers
- Support 28+ vector databases
- Support 12+ object storage services

---

📱 **Follow WeChat Official Account 「柒叔代码阁」**

Regularly publish Dify in-depth content and production practices~

![Official Account QR Code](../qrcode.png)

💡 **Why follow?**

Not just tell you "what" and "how", but more importantly **"why"** — understanding the thinking behind the design allows you to apply it flexibly.

💬 **Discussion and Exchange**
- GitHub Issue/Discussion: Technical problem discussions
- Official Account Messages