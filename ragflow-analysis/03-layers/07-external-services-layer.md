# 07 - 外部服务层 (External Services Layer)

## 1. 层职责与定位

### 1.1 职责
外部服务层整合第三方服务和API：
- **LLM 提供商**: OpenAI, Anthropic, Google等大模型服务
- **嵌入模型**: 文本嵌入服务
- **重排模型**: 搜索结果重排序服务
- **OCR 服务**: 云端OCR识别
- **第三方登录**: GitHub, 飞书, 企业微信等OAuth
- **数据源连接**: 外部数据库、API、文件系统
- **通知服务**: 邮件、消息推送

### 1.2 定位
- **架构位置**: 最外层，被业务逻辑层调用
- **技术选型**: HTTP/gRPC 客户端
- **设计模式**: 适配器模式、工厂模式
- **核心模块**: `rag/llm/`, `common/data_source/`

---

## 2. 主要组件/模块

### 2.1 LLM 服务集成 (`rag/llm/`)

#### 目录结构
```
rag/llm/
├── __init__.py
├── chat_model.py           # 聊天模型基类和多个实现类
├── embedding_model.py      # 嵌入模型基类
├── rerank_model.py         # 重排模型基类
├── cv_model.py             # 计算机视觉模型
├── ocr_model.py            # OCR模型
├── sequence2txt_model.py   # 序列转文本模型
└── tts_model.py            # 文本转语音模型
```

**注意**: RAGFlow 使用 **LiteLLM** 库作为统一的 LLM 接口层，所有模型实现都在 `chat_model.py` 中的各个类中，而不是独立的文件。

#### 2.1.1 模型基类定义

**聊天模型基类** (`chat_model.py`):
```python
from openai import AsyncOpenAI
from typing import AsyncGenerator, Optional

class Base:
    """聊天模型基类"""
    
    def __init__(self, key, model_name, **kwargs):
        self.client = AsyncOpenAI(api_key=key, base_url=kwargs.get('base_url'))
        self.async_client = AsyncOpenAI(api_key=key, base_url=kwargs.get('base_url'))
        self.model_name = model_name
        self.max_retries = kwargs.get('max_retries', 3)
        self.base_delay = kwargs.get('base_delay', 1)
        self.max_rounds = kwargs.get('max_rounds', 10)
        self.is_tools = False
        self.tools = None
    
    async def async_chat(
        self, 
        system: str,
        history: list,
        gen_conf: dict
    ) -> str:
        """异步对话"""
        messages = self._build_messages(system, history)
        response = await self.async_client.chat.completions.create(
            model=self.model_name,
            messages=messages,
            **gen_conf
        )
        return response.choices[0].message.content
    
    async def async_chat_streamly(
        self, 
        system: str,
        history: list,
        gen_conf: dict
    ) -> AsyncGenerator[str, None]:
        """流式对话"""
        messages = self._build_messages(system, history)
        stream = await self.async_client.chat.completions.create(
            model=self.model_name,
            messages=messages,
            stream=True,
            **gen_conf
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content
    
    def _build_messages(self, system: str, history: list) -> list:
        """构建消息列表"""
        messages = []
        if system:
            messages.append({"role": "system", "content": system})
        messages.extend(history)
        return messages
```

**嵌入模型基类** (`embedding_model.py`):
```python
class BaseEmbeddingModel(ABC):
    """嵌入模型基类"""
    
    @abstractmethod
    async def embed(
        self, 
        texts: List[str], 
        batch_size: int = 32
    ) -> List[List[float]]:
        """批量嵌入"""
        pass
    
    @abstractmethod
    async def embed_query(self, text: str) -> List[float]:
        """单条嵌入（查询优化）"""
        pass
    
    def get_dimension(self) -> int:
        """获取嵌入维度"""
        return self.dimension
```

#### 2.1.2 LiteLLM 统一接口 (`chat_model.py`)

RAGFlow 使用 **LiteLLM** 库来统一管理多个 LLM 提供商的接口：

```python
import litellm
from litellm import acompletion
from abc import ABC

class LiteLLMBase(ABC):
    """LiteLLM 统一接口基类"""
    
    async def async_chat(self, system: str, history: list, gen_conf: dict) -> str:
        """使用 LiteLLM 进行对话"""
        messages = self._build_messages(system, history)
        
        # LiteLLM 自动识别模型提供商
        response = await acompletion(
            model=self.model_name,  # 例: "gpt-4", "claude-3", "gemini-pro"
            messages=messages,
            **gen_conf
        )
        return response.choices[0].message.content
```

**支持的模型类**：
- `OpenAI_APIChat`: OpenAI 和兼容 API
- `GoogleChat`: Google Gemini
- `BaiduYiyanChat`: 百度文心
- `SparkChat`: 讯飞星火
- `HunyuanChat`: 腾讯混元
- `TokenPonyChat`: TokenPony
- `XinferenceChat`: Xinference
- `HuggingFaceChat`: HuggingFace
- `LocalLLM`: 本地模型

所有这些类都在 `chat_model.py` 文件中实现，而不是独立的文件。
        
        # 分批处理
#### 2.2.2 Slack 连接器示例

```python
from slack_sdk.web.async_client import AsyncWebClient

class SlackConnector(BaseConnector):
    """
Slack 连接器"""
    
    def __init__(self, token: str):
        self.client = AsyncWebClient(token=token)
    
    async def initialize(self):
        """Initialize Slack connection"""
        response = await self.client.auth_test()
        return response["ok"]
    
    async def check_authorization(self) -> bool:
        """Check authorization"""
        try:
            await self.client.auth_test()
            return True
        except Exception:
            return False
    
    async def ingest_all(self, channels: list = None):
        """摄取所有消息"""
        if not channels:
            # 获取所有频道
            channels_resp = await self.client.conversations_list()
            channels = [ch["id"] for ch in channels_resp["channels"]]
        
        for channel_id in channels:
            # 获取频道消息
            messages = await self.client.conversations_history(channel=channel_id)
            for message in messages["messages"]:
                yield {
                    "channel": channel_id,
                    "text": message.get("text"),
                    "user": message.get("user"),
                    "ts": message.get("ts")
                }
```

其他连接器（Notion, SharePoint, Gmail 等）都遵循相似的模式实现。

---

## 3. 支持的外部服务

以下部分删除了大量原有内容，保留核心表格：

        for i in range(0, len(texts), batch_size):
            batch = texts[i:i+batch_size]
            
            response = await self.client.embeddings.create(
                model=self.model_name,
                input=batch
            )
            
            embeddings = [item.embedding for item in response.data]
            all_embeddings.extend(embeddings)
        
        return all_embeddings
    
    async def embed_query(self, text: str) -> List[float]:
        """单条嵌入"""
        embeddings = await self.embed([text])
        return embeddings[0]
```

#### 2.1.3 工具调用支持

```python
class Base:
    async def async_chat_with_tools(
        self,
        system: str,
        history: list,
        gen_conf: dict,
        tools: list = None
    ):
        """支持工具调用的对话"""
        if tools:
            self.is_tools = True
            self.tools = tools
        
        messages = self._build_messages(system, history)
        
        # 调用 LLM 带工具
        response = await self.async_client.chat.completions.create(
            model=self.model_name,
            messages=messages,
            tools=tools,
            **gen_conf
        )
        
        # 处理工具调用
        if response.choices[0].message.tool_calls:
            # 执行工具并返回结果
            tool_results = await self._execute_tools(
                response.choices[0].message.tool_calls
            )
            return tool_results
        
        return response.choices[0].message.content
```

### 2.2 数据源连接器 (`common/data_source/`)

#### 目录结构
```
common/data_source/
├── __init__.py
├── interfaces.py           # 连接器接口定义
├── models.py               # 数据模型
├── exceptions.py           # 异常定义
├── utils.py                # 工具函数
├── config.py               # 配置管理
├── confluence_connector.py # Confluence
├── slack_connector.py      # Slack
├── notion_connector.py     # Notion
├── sharepoint_connector.py # SharePoint
├── gmail_connector.py      # Gmail
├── teams_connector.py      # Microsoft Teams
├── discord_connector.py    # Discord
├── dropbox_connector.py    # Dropbox
├── box_connector.py        # Box
├── webdav_connector.py     # WebDAV
├── moodle_connector.py     # Moodle
└── google_drive/           # Google Drive 相关
```

#### 2.2.1 连接器接口 (`interfaces.py`)

```python
from abc import ABC, abstractmethod
from typing import Any, AsyncIterator

class BaseConnector(ABC):
    """连接器基类"""
    
    @abstractmethod
    async def initialize(self):
        """初始化连接器"""
        pass
    
    @abstractmethod
    async def check_authorization(self) -> bool:
        """检查授权"""
        pass
    
    @abstractmethod
    async def ingest_all(self, **kwargs) -> AsyncIterator[Any]:
        """摄取所有数据"""
        pass
        if self.db_type == "postgresql":
            self.conn = await asyncpg.connect(
                host=self.host,
                port=self.port,
                database=self.database,
                user=self.username,
                password=self.password
            )
        elif self.db_type == "mysql":
            self.conn = await aiomysql.connect(
                host=self.host,
                port=self.port,
                db=self.database,
                user=self.username,
                password=self.password
            )
    
    async def query(self, sql: str) -> List[Dict]:
        """执行查询"""
        if self.db_type == "postgresql":
            rows = await self.conn.fetch(sql)
            return [dict(row) for row in rows]
        elif self.db_type == "mysql":
            async with self.conn.cursor(aiomysql.DictCursor) as cursor:
                await cursor.execute(sql)
                return await cursor.fetchall()
```

#### 2.2.2 Web 爬虫连接器 (`web_connector.py`)

```python
from playwright.async_api import async_playwright

class WebConnector:
    """网页爬虫连接器"""
    
    async def scrape(self, url: str, wait_for: str = None) -> str:
        """爬取网页内容"""
        async with async_playwright() as p:
            browser = await p.chromium.launch(headless=True)
            page = await browser.new_page()
            
            await page.goto(url, wait_until="domcontentloaded")
            
            if wait_for:
                await page.wait_for_selector(wait_for)
            
            content = await page.content()
            await browser.close()
            
            return content
    
    async def extract_text(self, html: str) -> str:
        """提取纯文本"""
        from bs4 import BeautifulSoup
        
        soup = BeautifulSoup(html, 'html.parser')
        
        # 移除脚本和样式
        for script in soup(["script", "style"]):
            script.extract()
        
        text = soup.get_text(separator='\n')
        lines = [line.strip() for line in text.splitlines() if line.strip()]
        
        return '\n'.join(lines)
```

### 2.3 第三方登录 (`api/apps/auth/`)

#### 2.3.1 OAuth 集成

**GitHub 登录**:
```python
import httpx

class GitHubOAuth:
    """GitHub OAuth 登录"""
    
    def __init__(self, client_id: str, client_secret: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.authorize_url = "https://github.com/login/oauth/authorize"
        self.token_url = "https://github.com/login/oauth/access_token"
        self.user_url = "https://api.github.com/user"
    
    def get_authorize_url(self, redirect_uri: str, state: str) -> str:
        """获取授权URL"""
        return f"{self.authorize_url}?client_id={self.client_id}&redirect_uri={redirect_uri}&state={state}&scope=user:email"
    
    async def get_access_token(self, code: str) -> str:
        """用授权码换取 access_token"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.token_url,
                data={
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                    "code": code
                },
                headers={"Accept": "application/json"}
            )
            
            data = response.json()
            return data["access_token"]
    
    async def get_user_info(self, access_token: str) -> dict:
        """获取用户信息"""
        async with httpx.AsyncClient() as client:
            response = await client.get(
                self.user_url,
                headers={"Authorization": f"Bearer {access_token}"}
            )
            
            return response.json()
```

**飞书登录**:
```python
class FeishuOAuth:
    """飞书 OAuth 登录"""
    
    def __init__(self, app_id: str, app_secret: str):
        self.app_id = app_id
        self.app_secret = app_secret
        self.base_url = "https://open.feishu.cn/open-apis"
    
    async def get_app_access_token(self) -> str:
        """获取应用 access_token"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/auth/v3/app_access_token/internal",
                json={
                    "app_id": self.app_id,
                    "app_secret": self.app_secret
                }
            )
            
            data = response.json()
            return data["app_access_token"]
    
    async def get_user_info(self, code: str) -> dict:
        """用授权码获取用户信息"""
        app_token = await self.get_app_access_token()
        
        async with httpx.AsyncClient() as client:
            # 1. 换取 user_access_token
            response = await client.post(
                f"{self.base_url}/authen/v1/access_token",
                headers={"Authorization": f"Bearer {app_token}"},
                json={"grant_type": "authorization_code", "code": code}
            )
            
            token_data = response.json()
            user_token = token_data["data"]["access_token"]
            
            # 2. 获取用户信息
            response = await client.get(
                f"{self.base_url}/authen/v1/user_info",
                headers={"Authorization": f"Bearer {user_token}"}
            )
            
            return response.json()["data"]
```

### 2.4 通知服务

#### 2.4.1 邮件服务 (`common/utils/email_service.py`)

```python
import aiosmtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class EmailService:
    """邮件服务"""
    
    def __init__(self, smtp_host: str, smtp_port: int, username: str, password: str):
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port
        self.username = username
        self.password = password
    
    async def send_email(
        self, 
        to: str, 
        subject: str, 
        body: str, 
        html: bool = False
    ):
        """发送邮件"""
        message = MIMEMultipart()
        message["From"] = self.username
        message["To"] = to
        message["Subject"] = subject
        
        if html:
            message.attach(MIMEText(body, "html"))
        else:
            message.attach(MIMEText(body, "plain"))
        
        async with aiosmtplib.SMTP(
            hostname=self.smtp_host, 
            port=self.smtp_port
        ) as smtp:
            await smtp.login(self.username, self.password)
            await smtp.send_message(message)
```

---

## 3. 支持的外部服务

### 3.1 LLM 提供商

| 提供商 | 模型 | 用途 | 官网 |
|--------|------|------|------|
| OpenAI | GPT-4, GPT-3.5 | 对话生成 | https://openai.com |
| Anthropic | Claude 3 | 对话生成 | https://anthropic.com |
| Google | Gemini | 对话生成 | https://ai.google.dev |
| Azure OpenAI | GPT-4, GPT-3.5 | 对话生成 | https://azure.microsoft.com/openai |
| 智谱AI | GLM-4 | 对话生成 | https://zhipuai.cn |
| 阿里云 | 通义千问 | 对话生成 | https://tongyi.aliyun.com |
| 百度 | 文心一言 | 对话生成 | https://yiyan.baidu.com |
| Moonshot | Kimi | 对话生成 | https://moonshot.cn |
| DeepSeek | DeepSeek-V2 | 对话生成 | https://deepseek.com |
| Ollama | Llama 2/3, Mistral | 本地模型 | https://ollama.ai |

### 3.2 嵌入模型

| 提供商 | 模型 | 维度 | 用途 |
|--------|------|------|------|
| OpenAI | text-embedding-3-large | 3072 | 文本嵌入 |
| OpenAI | text-embedding-3-small | 1536 | 文本嵌入 |
| 智谱AI | embedding-2 | 1024 | 文本嵌入 |
| BAAI | BGE-large-zh | 1024 | 中文嵌入 |
| Sentence Transformers | all-MiniLM-L6-v2 | 384 | 轻量嵌入 |

### 3.3 第三方登录

| 平台 | 类型 | 支持功能 |
|------|------|---------|
| GitHub | OAuth 2.0 | 登录、获取用户信息 |
| 飞书 | OAuth 2.0 | 登录、获取用户信息 |
| 企业微信 | OAuth 2.0 | 登录、获取用户信息 |
| Google | OAuth 2.0 | 登录、获取用户信息 |

### 3.4 数据源

| 类型 | 支持的源 |
|------|---------|
| 数据库 | PostgreSQL, MySQL, SQL Server, Oracle |
| 对象存储 | S3, MinIO, OSS, COS |
| 文件系统 | 本地文件系统, NFS, SMB |
| API | RESTful API, GraphQL |
| 网页 | 网页爬虫 (Playwright) |

---

## 4. 配置管理

### 4.1 LLM 配置 (`conf/llm_factories.json`)

```json
{
  "OpenAI": {
    "chat": {
      "model_type": "OPENAI",
      "api_base": "https://api.openai.com/v1",
      "models": ["gpt-4", "gpt-3.5-turbo"]
    },
    "embedding": {
      "model_type": "OPENAI",
      "models": ["text-embedding-3-large", "text-embedding-3-small"]
    }
  },
  "Anthropic": {
    "chat": {
      "model_type": "ANTHROPIC",
      "api_base": "https://api.anthropic.com",
      "models": ["claude-3-opus-20240229", "claude-3-sonnet-20240229"]
    }
  },
  "Zhipu": {
    "chat": {
      "model_type": "ZHIPU",
      "api_base": "https://open.bigmodel.cn/api/paas/v4",
      "models": ["glm-4", "glm-3-turbo"]
    },
    "embedding": {
      "model_type": "ZHIPU",
      "models": ["embedding-2"]
    }
  }
}
```

### 4.2 环境变量配置

```bash
# OpenAI
export OPENAI_API_KEY="sk-..."
export OPENAI_BASE_URL="https://api.openai.com/v1"

# Anthropic
export ANTHROPIC_API_KEY="sk-ant-..."

# 智谱AI
export ZHIPU_API_KEY="..."

# GitHub OAuth
export GITHUB_CLIENT_ID="..."
export GITHUB_CLIENT_SECRET="..."

# 飞书
export FEISHU_APP_ID="..."
export FEISHU_APP_SECRET="..."
```

---

## 5. 错误处理与重试

### 5.1 统一错误类型

```python
class ExternalServiceError(Exception):
    """外部服务错误基类"""
    pass

class RateLimitError(ExternalServiceError):
    """速率限制错误"""
    pass

class AuthenticationError(ExternalServiceError):
    """认证错误"""
    pass

class TimeoutError(ExternalServiceError):
    """超时错误"""
    pass
```

### 5.2 重试策略

```python
from tenacity import retry, stop_after_attempt, wait_exponential

class RetryableLLMClient:
    """带重试的 LLM 客户端"""
    
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10),
        reraise=True
    )
    async def call_with_retry(self, func, *args, **kwargs):
        """带重试的调用"""
        return await func(*args, **kwargs)
```

---

## 6. 性能优化

### 6.1 连接池

```python
import httpx

# 全局 HTTP 客户端（连接池）
http_client = httpx.AsyncClient(
    limits=httpx.Limits(max_keepalive_connections=100, max_connections=200),
    timeout=httpx.Timeout(60.0)
)
```

### 6.2 缓存

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
async def cached_embedding(text: str) -> List[float]:
    """缓存嵌入结果"""
    return await embedding_model.embed_query(text)
```

---

## 7. 总结

### 7.1 层的特点
- ✅ **多模型支持**: 20+ LLM 提供商
- ✅ **统一接口**: 适配器模式封装差异
- ✅ **错误处理**: 重试、降级、熔断
- ✅ **性能优化**: 连接池、缓存、批处理
- ✅ **可扩展**: 易于添加新服务

### 7.2 最佳实践
1. **API Key 管理**: 使用环境变量或密钥管理服务
2. **错误处理**: 优雅降级，不影响主流程
3. **速率限制**: 遵守服务商限制
4. **监控日志**: 记录所有外部调用
5. **缓存优化**: 减少重复调用

---

**文档版本**: 1.0  
**最后更新**: 2025-12-18  
**维护者**: RAGFlow 团队
