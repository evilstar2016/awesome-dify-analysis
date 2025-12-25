## Trace集成示例：Langfuse 集成指南

本章节以 Langfuse 为例，详细说明如何配置和集成追踪组件到 Dify 应用。

### 快速开始

#### 步骤 1：环境变量配置

在 Dify 项目的 `.env` 文件中添加以下配置：

```bash
# Langfuse 启用开关
LANGFUSE_ENABLED=true

# Langfuse API 端点（可选，默认为云端）
LANGFUSE_URL=https://cloud.langfuse.com

# 从 Langfuse 控制面板获取的 API 凭证
LANGFUSE_API_KEY=pk_lf_xxxxxxxxxxxxxxxxxxxxx
LANGFUSE_SECRET_KEY=sk_lf_xxxxxxxxxxxxxxxxxxxxx

# 可选：项目密钥（如果在 Langfuse 中配置了）
LANGFUSE_PROJECT_KEY=default
```

#### 步骤 2：验证配置

在应用启动时，Dify 会自动：
1. 读取环境变量
2. 初始化 Langfuse 连接
3. 在系统日志中显示连接状态

你可以通过查看启动日志确认连接是否成功：

```bash
# 启动 Dify 后端
uv run --project api dev/start-api

# 查看日志输出，应该看到类似信息：
# [INFO] Langfuse connection established
# [INFO] Tracing enabled for Langfuse provider
```

### 中级配置

#### 自托管 Langfuse

如果使用自托管的 Langfuse 实例，需要配置自定义 URL：

```bash
# 指向自托管的 Langfuse 服务器
LANGFUSE_URL=https://your-langfuse-instance.com

# 其他配置保持不变
LANGFUSE_API_KEY=pk_xxx
LANGFUSE_SECRET_KEY=sk_xxx
```

#### 条件启用追踪

根据环境选择性启用追踪：

```bash
# 开发环境：启用追踪
ENVIRONMENT=development
LANGFUSE_ENABLED=true

# 生产环境：启用追踪
ENVIRONMENT=production
LANGFUSE_ENABLED=true

# 测试环境：禁用追踪（减少日志噪音）
ENVIRONMENT=test
LANGFUSE_ENABLED=false
```

### 代码中的集成方式

#### 方式 1：自动追踪（推荐）

Dify 提供的 `OpsTraceManager` 类会自动处理追踪，开发者只需在适当的位置调用即可：

```python
from core.ops.ops_trace_manager import OpsTraceManager
from core.ops.entities.trace_entity import TraceTaskTypeEnum

# 追踪消息处理
trace_task = OpsTraceManager.trace_message_run(
    message_id=message.id,
    user_id=user_id,
    app_id=app_id,
    conversation_id=conversation_id,
    message_text=message.content,
    answer=response_text
)

# 追踪工作流执行
trace_task = OpsTraceManager.trace_workflow_run(
    workflow_run_id=workflow_run.id,
    workflow_run_inputs=workflow_run.inputs,
    workflow_run_outputs=workflow_run.outputs,
    user_id=user_id,
    app_id=app_id,
    conversation_id=conversation_id
)
```

#### 方式 2：手动配置（高级）

对于需要更细粒度控制的场景，可以直接使用 Langfuse DataTrace：

```python
from core.ops.langfuse_trace.langfuse_trace import LangFuseDataTrace
from core.ops.entities.trace_entity import MessageTraceInfo
from datetime import datetime

# 创建 Langfuse 数据追踪器
trace_manager = LangFuseDataTrace()

# 创建追踪信息
trace_info = MessageTraceInfo(
    trace_id=\"msg-12345\",
    tenant_id=\"tenant-001\",
    app_id=\"app-001\",
    user_id=\"user-001\",
    message_id=\"msg-001\",
    start_time=datetime.now(),
    end_time=datetime.now(),
    message_text=\"用户问题\",
    answer=\"AI 回答\",
    metadata={
        \"conversation_id\": \"conv-001\",
        \"model\": \"gpt-4\"
    }
)

# 手动发送追踪
trace_manager.message_trace(trace_info)
```

### 常见集成场景

#### 场景 1：聊天应用集成

在消息处理的完成回调中追踪：

```python
# 位置：controllers/web/conversation.py

def post_message(conversation_id, message_text):
    try:
        # 处理消息
        response = generate_response(message_text)
        
        # 保存到数据库
        message = save_message(conversation_id, message_text, response)
        
        # 追踪消息执行
        OpsTraceManager.trace_message_run(
            message_id=message.id,
            user_id=current_user.id,
            app_id=app_id,
            conversation_id=conversation_id,
            message_text=message_text,
            answer=response
        )
        
        return {\"message_id\": message.id, \"answer\": response}
    
    except Exception as e:
        # 错误也会被追踪
        logger.error(f\"Message processing failed: {e}\")
        raise
```

#### 场景 2：工作流执行追踪

在工作流完成后追踪整个执行过程：

```python
# 位置：core/workflow/workflow_engine.py

def execute_workflow(workflow_run):
    try:
        # 执行工作流
        result = workflow_run.execute()
        workflow_run.status = \"succeeded\"
    
    except Exception as e:
        workflow_run.status = \"failed\"
        workflow_run.error = str(e)
        raise
    
    finally:
        # 工作流执行后追踪（无论成功失败）
        OpsTraceManager.trace_workflow_run(
            workflow_run_id=workflow_run.id,
            workflow_run_inputs=workflow_run.inputs,
            workflow_run_outputs=workflow_run.outputs,
            user_id=workflow_run.user_id,
            app_id=workflow_run.app_id,
            conversation_id=workflow_run.conversation_id
        )
        
        db.session.commit()
```

#### 场景 3：关联消息和工作流的追踪

当消息通过工作流处理时，记录两者的关系：

```python
def process_message_with_workflow(message_id, workflow_id):
    # 执行工作流
    workflow_run = execute_workflow(workflow_id)
    
    # 追踪时关联消息 ID
    OpsTraceManager.trace_workflow_run(
        workflow_run_id=workflow_run.id,
        workflow_run_inputs=workflow_run.inputs,
        workflow_run_outputs=workflow_run.outputs,
        user_id=user_id,
        app_id=app_id,
        conversation_id=conversation_id,
        message_id=message_id  # 关键：关联到原始消息
    )
```

在 Langfuse 中，这样会自动将该工作流追踪链接到消息追踪。

### 注意事项

#### 1. 凭证安全性

⚠️ **重要**：永远不要在代码中硬编码 API 密钥

```python
# ✗ 错误做法
langfuse_key = \"sk_lf_xxxxx\"  # 不要这样做！

# ✓ 正确做法
langfuse_key = os.getenv(\"LANGFUSE_SECRET_KEY\")
```

使用环境变量和密钥管理服务：
- 本地开发：`.env` 文件（添加到 `.gitignore`）
- Docker：环境变量或 Docker Secrets
- Kubernetes：Secrets 或 ConfigMaps
- 生产环境：密钥管理服务（AWS Secrets Manager、Azure Key Vault 等）

#### 2. 异步处理和延迟

追踪数据通过 Celery 异步发送，这意味着：

```python
# 追踪调用立即返回，不会阻塞应用
trace_task = OpsTraceManager.trace_message_run(...)

# 但数据发送可能在几秒后才完成
# 这是设计好的，不会影响用户体验
```

**验证追踪状态**：
- 查看 Celery 日志确认任务已发送
- 在 Langfuse 控制面板中查看追踪是否到达
- 通常延迟为 2-5 秒

#### 3. 数据大小和配额限制

Langfuse 对单个请求有大小限制，避免发送过大的对象：

```python
# ✗ 不好的做法：发送完整的大型对象
metadata = {
    \"full_document_content\": very_large_pdf_text,  # 可能超过限制
    \"all_search_results\": complete_search_results
}

# ✓ 好的做法：发送摘要或关键信息
metadata = {
    \"document_pages\": 45,
    \"document_language\": \"chinese\",
    \"search_results_count\": 10,
    \"top_result_score\": 0.95
}
```

#### 4. 性能影响

由于是异步处理，对应用性能的影响极小：

```python
# 追踪不会减慢响应时间
@app.route(\"/api/message\")
def post_message():
    start_time = time.time()
    
    # 处理消息
    response = generate_response(message_text)
    
    # 追踪（异步，耗时 < 1ms）
    OpsTraceManager.trace_message_run(...)
    
    elapsed = time.time() - start_time
    # 追踪调用几乎不增加总延迟
```

#### 5. 错误处理

确保追踪失败不会影响应用主流程：

```python
def process_request():
    try:
        # 主业务逻辑
        result = main_business_logic()
    
    except Exception as e:
        # 业务错误被正确处理
        logger.error(f\"Business error: {e}\")
        raise
    
    finally:
        try:
            # 追踪是最后的操作
            OpsTraceManager.trace_message_run(...)
        except Exception as e:
            # 追踪失败不应该导致请求失败
            logger.warning(f\"Tracing failed (non-critical): {e}\")
            # 继续，不重新抛出异常
```

#### 6. 元数据规范

遵循一致的元数据结构便于在 Langfuse 中查询：

```python
# 推荐的元数据结构
metadata = {
    # 必需
    \"app_id\": app_id,
    \"user_id\": user_id,
    \"tenant_id\": tenant_id,
    
    # 推荐
    \"environment\": \"production\",
    \"version\": app_version,
    \"region\": deployment_region,
    
    # 可选：业务相关
    \"feature_flag_enabled\": True,
    \"experiment_group\": \"control\",
    \"user_tier\": \"premium\"
}
```

#### 7. 隐私合规

遵守数据隐私法规（GDPR、CCPA 等）：

```python
# ✗ 不要追踪个人信息
metadata = {
    \"user_name\": user.name,        # PII
    \"user_email\": user.email,      # PII
    \"user_phone\": user.phone,      # PII
}

# ✓ 应该只追踪去标识化的信息
metadata = {
    \"user_id\": user.id,            # 安全
    \"user_tier\": user.tier,        # 聚合信息
    \"region\": user.region,         # 去标识化
}
```

### 调试技巧

#### 启用详细日志

```python
# 在 api/configs/logging.py 中配置
import logging

logging.getLogger('core.ops').setLevel(logging.DEBUG)
logging.getLogger('langfuse').setLevel(logging.DEBUG)
```

#### 验证追踪配置

```python
# 在应用启动时运行的诊断脚本
from core.ops.ops_trace_manager import OpsTraceManager

def verify_tracing_config():
    try:
        # 尝试获取追踪实例
        trace_instance = OpsTraceManager.get_tracing_instance()
        print(f\"✓ Langfuse connection verified\")
        print(f\"  URL: {trace_instance.config.host}\")
        print(f\"  Project: {trace_instance.config.project_key}\")
    except Exception as e:
        print(f\"✗ Langfuse connection failed: {e}\")
        print(f\"  Please check your LANGFUSE_* environment variables\")
```

#### 手动测试追踪

```python
# 在 Python REPL 中测试
from core.ops.ops_trace_manager import OpsTraceManager
from datetime import datetime

# 发送测试追踪
trace_task = OpsTraceManager.trace_message_run(
    message_id=\"test-123\",
    user_id=\"test-user\",
    app_id=\"test-app\",
    conversation_id=\"test-conv\",
    message_text=\"Test message\",
    answer=\"Test response\"
)

print(\"Trace submitted successfully\")
print(f\"Trace ID: {trace_task.trace_id}\")

# 检查 Langfuse 控制面板，应该在几秒内看到追踪
```

### 故障排除

#### 问题：追踪未在 Langfuse 中出现

**可能原因**：
1. 环境变量配置不正确
2. 网络连接问题
3. API 密钥无效
4. Celery Worker 未运行

**解决步骤**：
```bash
# 1. 验证环境变量
echo $LANGFUSE_ENABLED
echo $LANGFUSE_API_KEY

# 2. 检查 Celery Worker 状态
ps aux | grep celery

# 3. 查看 Celery 日志
tail -f logs/celery.log

# 4. 检查网络连接
curl -H \"Authorization: Bearer $LANGFUSE_SECRET_KEY\" \\
     https://cloud.langfuse.com/api/health
```

#### 问题：API 密钥被拒绝

**检查清单**：
- ✓ 凭证来自正确的 Langfuse 项目
- ✓ API 密钥和 Secret 密钥配对正确
- ✓ 密钥未被复制时引入空格或特殊字符
- ✓ 在 Langfuse 控制面板中确认密钥仍然有效

#### 问题：Celery 任务超时

如果追踪总是超时，调整 Celery 配置：

```python
# api/configs/celery.py
CELERY_TASK_TIME_LIMIT = 300  # 5 分钟
CELERY_TASK_SOFT_TIME_LIMIT = 240  # 4 分钟的软限制

# 或针对追踪任务
app.conf.task_routes = {
    'tasks.ops_trace_task.*': {'queue': 'trace'},
}
app.conf.task_time_limits = {
    'tasks.ops_trace_task.*': (300, 240),
}
```

---

## 配置和环境变量

### Langfuse 配置

```bash
# .env 文件示例
LANGFUSE_ENABLED=true
LANGFUSE_URL=https://cloud.langfuse.com
LANGFUSE_API_KEY=pk_xxx
LANGFUSE_SECRET_KEY=sk_xxx
```

### 追踪配置

```python
# api/configs/feature.py 中的特性开关
FEATURE_FLAGS = {
    "ENABLE_TRACING": True,           # 启用追踪
    "ENABLE_LANGFUSE": True,          # 启用 Langfuse 集成
    "TRACE_ASYNC": True,              # 异步发送追踪数据
    "TRACE_LLM_CALLS": True,          # 追踪 LLM 调用
    "TRACE_WORKFLOW_NODES": True,     # 追踪工作流节点
}
```
