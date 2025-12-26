# 03 - API 服务层 (API Service Layer)

## 1. 层职责与定位

### 1.1 职责
API 服务层是 RAGFlow 后端的核心，负责：
- **接口定义**: 提供 RESTful API 接口
- **请求路由**: 根据 URL 路径分发到对应的 Blueprint
- **认证授权**: 验证用户身份和权限
- **参数校验**: 验证请求参数合法性
- **业务编排**: 协调业务逻辑层的多个服务
- **响应封装**: 统一封装 API 响应格式
- **错误处理**: 捕获异常并返回友好错误信息
- **流式响应**: 支持 SSE (Server-Sent Events) 流式输出

### 1.2 定位
- **架构位置**: 网关层和业务逻辑层之间
- **技术选型**: Quart (异步 Flask 替代品)
- **设计模式**: Blueprint 模块化设计
- **部署方式**: Docker 容器
- **访问端口**:
  - API 服务: 9380
  - Admin 服务: 9381

---

## 2. 主要组件/模块

### 2.1 应用入口

#### 2.1.1 主服务器 (`ragflow_server.py`)

**路径**: `api/ragflow_server.py`

**核心功能**:
```python
# 初始化日志、配置、数据库
init_root_logger("ragflow_server")
init_web_db()
init_web_data()
init_superuser()

# 启动后台线程
threading.Thread(target=update_progress, daemon=True).start()

# 启动 Quart 应用
from api.apps import app
app.run(host="0.0.0.0", port=9380, debug=False)
```

**关键特性**:
- ✅ 异步 WSGI 服务器 (Quart)
- ✅ 多线程后台任务 (文档处理进度更新)
- ✅ 信号处理 (优雅关闭)
- ✅ Debugpy 支持 (远程调试)
- ✅ Redis 分布式锁 (防止重复执行)

#### 2.1.2 应用工厂 (`api/apps/__init__.py`)

**路径**: `api/apps/__init__.py`

**Blueprint 注册**:
```python
from quart import Quart
from api.apps.user_app import user_app
from api.apps.kb_app import kb_app
from api.apps.dialog_app import dialog_app
# ...更多 Blueprint

app = Quart(__name__)

# 注册 Blueprint
app.register_blueprint(user_app, url_prefix='/api/v1/user')
app.register_blueprint(kb_app, url_prefix='/api/v1/kb')
app.register_blueprint(dialog_app, url_prefix='/api/v1/dialog')
# ...
```

### 2.2 Blueprint 模块结构

**目录**: `api/apps/`

**主要 Blueprint**:

| Blueprint | 文件 | URL 前缀 | 功能 |
|-----------|------|----------|------|
| `user_app` | `user_app.py` | `/api/v1/user` | 用户管理 (注册、登录、资料) |
| `kb_app` | `kb_app.py` | `/api/v1/kb` | 知识库管理 (CRUD、任务) |
| `dialog_app` | `dialog_app.py` | `/api/v1/dialog` | 对话管理 (创建、删除、聊天) |
| `canvas_app` | `canvas_app.py` | `/api/v1/canvas` | Agent 画布编辑 |
| `document_app` | `document_app.py` | `/api/v1/document` | 文档管理 (上传、删除、解析) |
| `chunk_app` | `chunk_app.py` | `/api/v1/chunk` | 文档分块管理 |
| `file_app` | `file_app.py` | `/api/v1/file` | 文件上传下载 |
| `conversation_app` | `conversation_app.py` | `/api/v1/conversation` | 对话会话管理 |
| `llm_app` | `llm_app.py` | `/api/v1/llm` | LLM 模型管理 |
| `system_app` | `system_app.py` | `/api/v1/system` | 系统配置和健康检查 |
| `search_app` | `search_app.py` | `/api/v1/search` | 全局搜索 |
| `tenant_app` | `tenant_app.py` | `/api/v1/tenant` | 租户管理 |
| `plugin_app` | `plugin_app.py` | `/api/v1/plugin` | 插件管理 |
| `evaluation_app` | `evaluation_app.py` | `/api/v1/evaluation` | RAG 评估 |
| `mcp_server_app` | `mcp_server_app.py` | `/api/v1/mcp` | MCP 服务管理 |
| `memories_app` | `memories_app.py` | `/api/v1/memories` | Agent 记忆管理 |
| `langfuse_app` | `langfuse_app.py` | `/api/v1/langfuse` | Langfuse 集成 |
| `connector_app` | `connector_app.py` | `/api/v1/connector` | 数据源连接器 |

### 2.3 核心 Blueprint 示例

#### 2.3.1 知识库 Blueprint (`kb_app.py`)

**典型接口**:
```python
from quart import Blueprint, request
from api.db.services.knowledgebase_service import KnowledgebaseService
from api.utils import get_uuid, get_error_data_result, get_data_error_result

kb_app = Blueprint("kb", __name__)

@kb_app.route('/create', methods=['POST'])
async def create():
    """创建知识库"""
    req = await request.get_json()
    kb_id = get_uuid()
    
    # 参数校验
    if not req.get("name"):
        return get_error_data_result(message="Name is required")
    
    # 调用服务层
    kb = KnowledgebaseService.save(
        id=kb_id,
        name=req["name"],
        tenant_id=req["tenant_id"],
        embd_id=req.get("embd_id"),
        # ...
    )
    
    return get_data_error_result(data=kb.to_dict())

@kb_app.route('/list', methods=['GET'])
async def list_kbs():
    """获取知识库列表"""
    tenant_id = request.args.get("tenant_id")
    page = int(request.args.get("page", 1))
    page_size = int(request.args.get("page_size", 10))
    
    kbs, total = KnowledgebaseService.get_list(
        tenant_id=tenant_id,
        page=page,
        page_size=page_size
    )
    
    return get_data_error_result(data={
        "kbs": [kb.to_dict() for kb in kbs],
        "total": total
    })

@kb_app.route('/delete/<kb_id>', methods=['DELETE'])
async def delete(kb_id):
    """删除知识库"""
    success = KnowledgebaseService.delete(kb_id)
    
    if not success:
        return get_error_data_result(message="Delete failed")
    
    return get_data_error_result(data={"success": True})
```

#### 2.3.2 对话 Blueprint (`dialog_app.py`)

**流式响应示例**:
```python
from quart import Response
import asyncio

@dialog_app.route('/stream', methods=['POST'])
async def stream_chat():
    """SSE 流式对话"""
    req = await request.get_json()
    dialog_id = req["dialog_id"]
    question = req["question"]
    
    async def generate():
        """生成流式响应"""
        # 调用 RAG 引擎
        from rag.chat import RAGChat
        chat = RAGChat(dialog_id=dialog_id)
        
        # 流式生成答案
        async for chunk in chat.stream(question):
            # SSE 格式: data: {json}\n\n
            yield f"data: {json.dumps(chunk)}\n\n"
            await asyncio.sleep(0.01)  # 避免阻塞
        
        # 发送结束信号
        yield "data: [DONE]\n\n"
    
    return Response(
        generate(),
        mimetype='text/event-stream',
        headers={
            'Cache-Control': 'no-cache',
            'X-Accel-Buffering': 'no',  # 禁用 Nginx 缓冲
        }
    )
```

#### 2.3.3 文件 Blueprint (`file_app.py`)

**文件上传示例**:
```python
from werkzeug.utils import secure_filename
from api.utils import minio_client

@file_app.route('/upload', methods=['POST'])
async def upload():
    """上传文件到 MinIO"""
    files = await request.files
    uploaded_files = []
    
    for file in files.getlist('file'):
        # 安全文件名
        filename = secure_filename(file.filename)
        
        # 生成唯一路径
        file_id = get_uuid()
        object_name = f"files/{file_id}/{filename}"
        
        # 上传到 MinIO
        minio_client.put_object(
            bucket_name="ragflow",
            object_name=object_name,
            data=file.stream,
            length=-1,
            part_size=10*1024*1024
        )
        
        uploaded_files.append({
            "file_id": file_id,
            "filename": filename,
            "object_name": object_name
        })
    
    return get_data_error_result(data=uploaded_files)
```

### 2.4 认证与授权

#### 2.4.1 认证装饰器 (`api/apps/auth/auth.py`)

**JWT 认证**:
```python
from functools import wraps
from quart import request
import jwt

def token_required(f):
    @wraps(f)
    async def decorated(*args, **kwargs):
        token = request.headers.get('Authorization')
        
        if not token:
            return get_error_data_result(message="Token is missing", code=401)
        
        try:
            # 解析 JWT
            token = token.replace("Bearer ", "")
            data = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
            
            # 注入用户信息
            kwargs['user_id'] = data['user_id']
            kwargs['tenant_id'] = data['tenant_id']
            
        except jwt.ExpiredSignatureError:
            return get_error_data_result(message="Token expired", code=401)
        except jwt.InvalidTokenError:
            return get_error_data_result(message="Invalid token", code=401)
        
        return await f(*args, **kwargs)
    
    return decorated
```

**使用示例**:
```python
@kb_app.route('/create', methods=['POST'])
@token_required
async def create(user_id, tenant_id):
    # user_id 和 tenant_id 由装饰器注入
    # ...
```

#### 2.4.2 权限控制

**基于角色的访问控制 (RBAC)**:
```python
def admin_required(f):
    @wraps(f)
    async def decorated(user_id, tenant_id, *args, **kwargs):
        from api.db.services.user_service import UserService
        
        user = UserService.get_by_id(user_id)
        
        if not user or not user.is_superuser:
            return get_error_data_result(message="Admin access required", code=403)
        
        return await f(user_id, tenant_id, *args, **kwargs)
    
    return decorated
```

### 2.5 SDK 接口 (`api/apps/sdk/`)

**RESTful SDK API**:
- `sdk_app.py`: SDK 专用接口 (更简洁的参数)
- 兼容外部调用 (Python SDK, HTTP 客户端)

**典型接口**:
```python
@sdk_app.route('/v1/dataset/create', methods=['POST'])
@api_key_required
async def sdk_create_dataset(tenant_id):
    """SDK: 创建数据集 (知识库)"""
    req = await request.get_json()
    
    # 更简洁的参数映射
    kb = KnowledgebaseService.save(
        name=req["name"],
        tenant_id=tenant_id,
        # 自动使用默认配置
    )
    
    return {"code": 0, "data": {"dataset_id": kb.id}}
```

---

## 3. 对外提供的接口或服务

### 3.1 RESTful API

#### 3.1.1 接口规范

**URL 格式**:
```
https://{domain}/api/v1/{module}/{action}
```

**请求头**:
```http
Content-Type: application/json
Authorization: Bearer {access_token}
```

**响应格式**:
```json
{
  "code": 0,              // 0 表示成功
  "message": "Success",
  "data": { ... }
}
```

#### 3.1.2 核心接口列表

| 类别 | 接口 | 方法 | 说明 |
|------|------|------|------|
| **用户** | `/api/v1/user/register` | POST | 用户注册 |
| | `/api/v1/user/login` | POST | 用户登录 |
| | `/api/v1/user/logout` | POST | 用户登出 |
| | `/api/v1/user/info` | GET | 获取用户信息 |
| **知识库** | `/api/v1/kb/create` | POST | 创建知识库 |
| | `/api/v1/kb/list` | GET | 知识库列表 |
| | `/api/v1/kb/update/<kb_id>` | PUT | 更新知识库 |
| | `/api/v1/kb/delete/<kb_id>` | DELETE | 删除知识库 |
| **文档** | `/api/v1/document/upload` | POST | 上传文档 |
| | `/api/v1/document/list` | GET | 文档列表 |
| | `/api/v1/document/delete/<doc_id>` | DELETE | 删除文档 |
| | `/api/v1/document/parse/<doc_id>` | POST | 解析文档 |
| **对话** | `/api/v1/dialog/create` | POST | 创建对话助手 |
| | `/api/v1/dialog/list` | GET | 对话助手列表 |
| | `/api/v1/dialog/chat` | POST | 同步对话 |
| | `/api/v1/dialog/stream` | POST | 流式对话 (SSE) |
| **文件** | `/api/v1/file/upload` | POST | 文件上传 |
| | `/api/v1/file/download/<file_id>` | GET | 文件下载 |
| **Agent** | `/api/v1/canvas/create` | POST | 创建 Agent 画布 |
| | `/api/v1/canvas/run` | POST | 运行 Agent |
| **系统** | `/api/v1/system/health` | GET | 健康检查 |
| | `/api/v1/system/version` | GET | 版本信息 |

### 3.2 SSE 流式接口

#### 3.2.1 流式对话

**接口**: `POST /api/v1/dialog/stream`

**请求**:
```json
{
  "dialog_id": "uuid",
  "question": "什么是 RAG?",
  "conversation_id": "uuid"  // 可选，会话 ID
}
```

**响应 (SSE 流)**:
```
data: {"answer": "RAG", "reference": {}}

data: {"answer": " 是检索增强", "reference": {}}

data: {"answer": "生成技术", "reference": {...}}

data: [DONE]
```

**前端接收**:
```javascript
const eventSource = new EventSource('/api/v1/dialog/stream', {
  headers: { 'Authorization': `Bearer ${token}` }
});

eventSource.onmessage = (event) => {
  if (event.data === '[DONE]') {
    eventSource.close();
    return;
  }
  
  const chunk = JSON.parse(event.data);
  console.log(chunk.answer);
};
```

---

## 4. 与其他层的交互方式

### 4.1 与网关层交互

```
Nginx (443)
  ↓ 反向代理
  ↓ HTTP 请求
Quart API (9380)
  ↓ 路由分发
  ↓ Blueprint 处理
返回 JSON 响应
  ↓
Nginx
  ↓ HTTPS 响应
客户端
```

### 4.2 与业务逻辑层交互

```python
# API 层 (dialog_app.py)
@dialog_app.route('/chat', methods=['POST'])
@token_required
async def chat(user_id, tenant_id):
    req = await request.get_json()
    
    # 调用业务逻辑层服务
    from rag.chat import RAGChat
    from api.db.services.dialog_service import DialogService
    
    # 1. 验证对话是否存在
    dialog = DialogService.get_by_id(req["dialog_id"])
    if not dialog:
        return get_error_data_result(message="Dialog not found")
    
    # 2. 调用 RAG 引擎
    chat = RAGChat(
        dialog_id=dialog.id,
        kb_ids=dialog.kb_ids,
        llm_id=dialog.llm_id
    )
    
    # 3. 生成答案
    answer = await chat.chat(req["question"])
    
    # 4. 保存对话历史
    from api.db.services.conversation_service import ConversationService
    ConversationService.save(
        dialog_id=dialog.id,
        question=req["question"],
        answer=answer["answer"]
    )
    
    return get_data_error_result(data=answer)
```

### 4.3 与数据访问层交互

```python
# API 层调用 Service
from api.db.services.knowledgebase_service import KnowledgebaseService

# Service 内部调用 Model
class KnowledgebaseService:
    @classmethod
    def get_by_id(cls, kb_id):
        from api.db.db_models import Knowledgebase
        return Knowledgebase.query.filter_by(id=kb_id).first()
    
    @classmethod
    def save(cls, **kwargs):
        from api.db.db_models import Knowledgebase
        kb = Knowledgebase(**kwargs)
        db.session.add(kb)
        db.session.commit()
        return kb
```

---

## 5. 关键代码文件路径

### 5.1 核心文件

| 文件 | 路径 | 说明 |
|------|------|------|
| 主服务器 | `api/ragflow_server.py` | 应用入口 |
| 应用工厂 | `api/apps/__init__.py` | Blueprint 注册 |
| 配置文件 | `api/settings.py` | API 配置 |
| 认证模块 | `api/apps/auth/` | 认证授权 |

### 5.2 Blueprint 文件

| Blueprint | 路径 | 行数 |
|-----------|------|------|
| 知识库 | `api/apps/kb_app.py` | ~600 |
| 对话 | `api/apps/dialog_app.py` | ~500 |
| Agent | `api/apps/canvas_app.py` | ~400 |
| 文档 | `api/apps/document_app.py` | ~800 |
| 用户 | `api/apps/user_app.py` | ~400 |
| 系统 | `api/apps/system_app.py` | ~200 |

### 5.3 工具模块

| 模块 | 路径 | 说明 |
|------|------|------|
| 响应工具 | `api/utils/response.py` | 统一响应格式 |
| 参数校验 | `api/validation.py` | 参数验证器 |
| UUID 生成 | `api/utils/uuid.py` | 生成唯一 ID |
| 常量定义 | `api/constants.py` | 全局常量 |

---

## 6. 技术实现细节

### 6.1 Quart 框架特性

#### 6.1.1 异步支持
```python
# 原生异步支持
@app.route('/async_data')
async def async_data():
    result = await async_database_call()
    return jsonify(result)
```

#### 6.1.2 WebSocket 支持
```python
from quart import websocket

@app.websocket('/ws')
async def ws():
    while True:
        data = await websocket.receive()
        await websocket.send(f"Echo: {data}")
```

#### 6.1.3 后台任务
```python
@app.before_serving
async def startup():
    app.background_tasks = set()
    # 启动后台任务
    task = asyncio.create_task(update_progress())
    app.background_tasks.add(task)
```

### 6.2 参数校验

#### 6.2.1 使用 Pydantic
```python
from pydantic import BaseModel, Field

class CreateKBRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    description: str = Field(default="", max_length=500)
    embd_id: str = Field(..., regex=r'^[a-zA-Z0-9-]+$')

@kb_app.route('/create', methods=['POST'])
async def create():
    req = await request.get_json()
    
    # 参数校验
    try:
        validated = CreateKBRequest(**req)
    except ValidationError as e:
        return get_error_data_result(message=str(e))
    
    # 使用验证后的数据
    kb = KnowledgebaseService.save(**validated.dict())
    return get_data_error_result(data=kb.to_dict())
```

### 6.3 错误处理

#### 6.3.1 全局异常处理
```python
from quart import Quart
from common.exceptions import CustomException

app = Quart(__name__)

@app.errorhandler(CustomException)
async def handle_custom_exception(e):
    return get_error_data_result(message=str(e), code=e.code)

@app.errorhandler(Exception)
async def handle_exception(e):
    logging.exception("Unhandled exception")
    return get_error_data_result(message="Internal server error", code=500)
```

#### 6.3.2 统一响应格式
```python
def get_data_error_result(code=0, message="Success", data=None):
    """成功响应"""
    return {
        "code": code,
        "message": message,
        "data": data or {}
    }

def get_error_data_result(message, code=500, data=None):
    """错误响应"""
    return {
        "code": code,
        "message": message,
        "data": data or {}
    }, code
```

### 6.4 性能优化

#### 6.4.1 连接池
```python
# Peewee 数据库连接池
from playhouse.pool import PooledMySQLDatabase, PooledPostgresqlDatabase

db = PooledMySQLDatabase(
    'ragflow',
    max_connections=20,
    stale_timeout=300,
    timeout=0,
    user='ragflow',
    password='infiniflow',
    host='localhost',
    port=3306
)
```

#### 6.4.2 缓存
```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_tenant_config(tenant_id):
    """缓存租户配置"""
    return TenantService.get_config(tenant_id)
```

#### 6.4.3 限流
```python
from quart_rate_limiter import RateLimiter

rate_limiter = RateLimiter(app)

@kb_app.route('/create', methods=['POST'])
@rate_limiter.limit("10/minute")  # 每分钟 10 次
async def create():
    # ...
```

---

## 7. 部署与运维

### 7.1 Docker 部署

#### Dockerfile
```dockerfile
FROM python:3.12-slim

WORKDIR /ragflow

# 安装依赖
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --no-dev

# 复制代码
COPY api/ api/
COPY rag/ rag/
COPY common/ common/

# 暴露端口
EXPOSE 9380

# 启动服务
CMD ["python", "api/ragflow_server.py"]
```

### 7.2 监控指标

#### 7.2.1 健康检查
```python
@system_app.route('/health', methods=['GET'])
async def health():
    """健康检查"""
    try:
        # 检查数据库
        db.session.execute("SELECT 1")
        
        # 检查 Redis
        from rag.utils.redis_conn import REDIS_CONN
        REDIS_CONN.ping()
        
        # 检查 Elasticsearch
        from rag.utils.es_conn import ELASTICSEARCH
        ELASTICSEARCH.ping()
        
        return {"status": "healthy"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}, 503
```

#### 7.2.2 关键指标
- **QPS**: 每秒请求数
- **响应时间**: P50, P95, P99
- **错误率**: 4xx, 5xx 比例
- **并发连接数**: 当前活跃连接
- **内存使用**: 进程内存占用
- **CPU 使用**: CPU 占用率

---

## 8. 总结

### 8.1 层的特点
- ✅ **异步高性能**: Quart 异步框架
- ✅ **模块化设计**: Blueprint 分离关注点
- ✅ **RESTful 规范**: 统一接口风格
- ✅ **流式响应**: 支持 SSE 和 WebSocket
- ✅ **安全可靠**: JWT 认证、RBAC 授权
- ✅ **易于扩展**: 插件化架构

### 8.2 最佳实践
1. **异步编程**: 充分利用 Quart 异步特性
2. **参数校验**: 使用 Pydantic 严格验证
3. **错误处理**: 全局异常处理 + 统一响应
4. **性能优化**: 连接池、缓存、限流
5. **日志监控**: 详细日志 + 健康检查
6. **API 文档**: OpenAPI/Swagger 文档

---

**文档版本**: 1.0  
**最后更新**: 2025-12-18  
**维护者**: RAGFlow 团队
