# 02 - 网关层 (Gateway Layer)

## 1. 层职责与定位

### 1.1 职责
网关层作为系统的统一入口，负责：
- **反向代理**: 将客户端请求转发到后端服务
- **负载均衡**: 在多个后端实例间分发请求
- **SSL/TLS 终止**: 处理 HTTPS 加密连接
- **静态资源服务**: 直接提供前端静态文件
- **请求路由**: 根据 URL 路径路由到不同服务
- **压缩与缓存**: 启用 Gzip 压缩和浏览器缓存
- **安全防护**: 防止 DDoS、限流、请求过滤

### 1.2 定位
- **架构位置**: 前端和后端之间的桥梁
- **技术选型**: Nginx
- **部署方式**: Docker 容器 / 独立服务器
- **访问端口**:
  - HTTP: 80
  - HTTPS: 443

---

## 2. 主要组件/模块

### 2.1 Nginx 配置结构

```
docker/nginx/
├── nginx.conf          # 主配置文件
├── ragflow.conf        # RAGFlow 应用配置
└── proxy.conf          # 代理通用配置
```

### 2.2 核心配置文件

#### 2.2.1 主配置文件 (`nginx.conf`)

**路径**: `docker/nginx/nginx.conf`

**关键配置**:
```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    # 性能优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss
               application/rss+xml font/truetype font/opentype 
               application/vnd.ms-fontobject image/svg+xml;
    
    # 引入站点配置
    include /etc/nginx/conf.d/*.conf;
}
```

#### 2.2.2 应用配置 (`ragflow.conf`)

**路径**: `docker/nginx/ragflow.conf`

**关键配置**:
```nginx
# HTTP 服务器 (默认配置)
server {
    listen 80;
    server_name _;
    root /ragflow/web/dist;

    # Gzip 压缩
    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 9;
    gzip_types text/plain application/javascript application/x-javascript text/css 
               application/xml text/javascript application/x-httpd-php 
               image/jpeg image/gif image/png;
    gzip_vary on;
    gzip_disable "MSIE [1-6]\.";

    # Admin API 路由
    location ~ ^/api/v1/admin {
        proxy_pass http://localhost:9381;
        include proxy.conf;
    }

    # 主 API 路由
    location ~ ^/(v1|api) {
        proxy_pass http://localhost:9380;
        include proxy.conf;
    }

    # 静态文件服务（前端）
    location / {
        index index.html;
        try_files $uri $uri/ /index.html;
        
        # 缓存策略
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
        
        location = /index.html {
            expires -1;
            add_header Cache-Control "no-cache, no-store, must-revalidate";
        }
    }
    
    # API 代理
    location /api/ {
        include /etc/nginx/proxy.conf;
        proxy_pass http://ragflow_api;
        
        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    # Admin API 代理
    location /admin/ {
        include /etc/nginx/proxy.conf;
        proxy_pass http://admin_api;
    }
    
    # SSE 流式响应（特殊配置）
    location /api/v1/dialog/stream {
        include /etc/nginx/proxy.conf;
        proxy_pass http://ragflow_api;
        
        # SSE 必须关闭缓冲
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding on;
        
        # 超时设置（流式响应可能很长）
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
    
    # 健康检查
    location /health {
        access_log off;
        proxy_pass http://ragflow_api/api/health;
    }
    
    # 错误页面
    error_page 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

#### 2.2.3 代理通用配置 (`proxy.conf`)

**路径**: `docker/nginx/proxy.conf`

```nginx
# 通用代理配置
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port $server_port;

# 超时设置
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;

# 缓冲设置
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 4k;
proxy_busy_buffers_size 8k;

# 不转发敏感头
proxy_pass_request_headers on;
proxy_hide_header X-Powered-By;
```

---

## 3. 对外提供的接口或服务

### 3.1 HTTP/HTTPS 入口

#### 3.1.1 端口映射
| 外部端口 | 内部端口 | 协议 | 用途 |
|---------|---------|------|------|
| 80 | - | HTTP | 重定向到 HTTPS |
| 443 | - | HTTPS | 主要入口 |

#### 3.1.2 URL 路由规则
| URL 路径 | 转发目标 | 说明 |
|---------|---------|------|
| `/` | 静态文件 | 前端应用 (index.html) |
| `/api/*` | `ragflow-api:9380` | 后端 API |
| `/admin/*` | `ragflow-api:9381` | 管理 API |
| `/api/v1/dialog/stream` | `ragflow-api:9380` | SSE 流式响应 |
| `/health` | `ragflow-api:9380/api/health` | 健康检查 |
| `/*.js, /*.css, ...` | 静态文件 | 前端资源 |

### 3.2 静态资源服务

#### 3.2.1 前端文件
- **根目录**: `/usr/share/nginx/html`
- **内容**: 
  - `index.html` (单页应用入口)
  - JavaScript 文件 (`*.js`)
  - CSS 文件 (`*.css`)
  - 图片、字体等资源

#### 3.2.2 缓存策略
```nginx
# JS/CSS/图片等：长期缓存（1年）
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# HTML：不缓存（确保获取最新版本）
location = /index.html {
    expires -1;
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

---

## 4. 与其他层的交互方式

### 4.1 与前端展示层交互

```
浏览器
  ↓ HTTPS GET /
Nginx (443)
  ↓ 读取静态文件
/usr/share/nginx/html/index.html
  ↓
返回 HTML 给浏览器
  ↓
浏览器加载 JS/CSS
  ↓ HTTPS GET /static/js/main.js
Nginx
  ↓ 读取并返回（带缓存头）
浏览器缓存资源
```

### 4.2 与 API 服务层交互

```
浏览器
  ↓ HTTPS POST /api/v1/kb/create
Nginx (443)
  ↓ 匹配 location /api/
  ↓ 添加代理头
  ↓ HTTP 转发
ragflow-api:9380
  ↓ 处理请求
  ↓ HTTP 响应
Nginx
  ↓ HTTPS 响应
浏览器
```

### 4.3 负载均衡（多实例场景）

```nginx
# upstream 配置
upstream ragflow_api {
    # Round Robin (默认)
    server ragflow-api-1:9380;
    server ragflow-api-2:9380;
    server ragflow-api-3:9380;
    
    # 或使用 IP Hash（同一客户端路由到同一服务器）
    # ip_hash;
    
    # 或使用 Least Connections（最少连接）
    # least_conn;
}
```

**流程**:
```
客户端请求
  ↓
Nginx
  ↓ 负载均衡算法
  ├→ ragflow-api-1 (33%)
  ├→ ragflow-api-2 (33%)
  └→ ragflow-api-3 (34%)
  ↓
返回响应
```

### 4.4 SSE 流式响应特殊处理

```
浏览器
  ↓ EventSource('/api/v1/dialog/stream')
Nginx
  ↓ proxy_buffering off（关键！）
  ↓ chunked_transfer_encoding on
  ↓ 保持长连接
ragflow-api:9380
  ↓ 流式生成数据
  ↓ chunk 1
Nginx ─→ 浏览器（立即转发）
  ↓ chunk 2
Nginx ─→ 浏览器（立即转发）
  ↓ chunk 3
Nginx ─→ 浏览器（立即转发）
  ...
  ↓ done
Nginx ─→ 浏览器（关闭连接）
```

---

## 5. 关键代码文件路径

### 5.1 配置文件
| 文件 | 路径 | 说明 |
|------|------|------|
| 主配置 | `docker/nginx/nginx.conf` | Nginx 全局配置 |
| 应用配置 | `docker/nginx/ragflow.conf` | RAGFlow 路由和代理 |
| 代理配置 | `docker/nginx/proxy.conf` | 通用代理设置 |
| Dockerfile | `docker/Dockerfile` | Nginx 容器构建 |

### 5.2 SSL 证书
| 文件 | 路径 | 说明 |
|------|------|------|
| 证书 | `docker/nginx/ssl/cert.pem` | SSL 证书 |
| 私钥 | `docker/nginx/ssl/key.pem` | SSL 私钥 |

### 5.3 日志文件
| 日志 | 路径 | 说明 |
|------|------|------|
| 访问日志 | `docker/ragflow-logs/nginx/access.log` | HTTP 请求日志 |
| 错误日志 | `docker/ragflow-logs/nginx/error.log` | Nginx 错误日志 |

---

## 6. 技术实现细节

### 6.1 Nginx 核心特性

#### 6.1.1 事件驱动架构
- **模型**: Epoll (Linux) / Kqueue (BSD)
- **优势**: 高并发、低内存占用
- **配置**: `worker_processes auto` (自动检测 CPU 核心数)

#### 6.1.2 异步非阻塞 I/O
- 单个 Worker 进程可处理数千并发连接
- 不为每个连接创建线程，节省资源

### 6.2 性能优化

#### 6.2.1 Gzip 压缩
```nginx
gzip on;
gzip_comp_level 6;          # 压缩级别 (1-9)
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;       # 小于 1KB 不压缩
```

**效果**: 
- 文本文件压缩率 60-80%
- JS/CSS 文件显著减小
- 降低带宽消耗

#### 6.2.2 静态文件缓存
```nginx
# 浏览器缓存（1年）
expires 1y;
add_header Cache-Control "public, immutable";

# Nginx 文件缓存
open_file_cache max=1000 inactive=20s;
open_file_cache_valid 30s;
open_file_cache_min_uses 2;
```

#### 6.2.3 连接优化
```nginx
keepalive_timeout 65;       # 保持连接 65 秒
keepalive_requests 100;     # 每连接最多 100 请求
sendfile on;                # 零拷贝文件传输
tcp_nopush on;              # 优化网络包
tcp_nodelay on;             # 禁用 Nagle 算法
```

### 6.3 安全加固

#### 6.3.1 SSL/TLS 配置
```nginx
# 只启用安全协议
ssl_protocols TLSv1.2 TLSv1.3;

# 安全加密套件
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
ssl_prefer_server_ciphers on;

# HSTS (强制 HTTPS)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

#### 6.3.2 安全头
```nginx
# 防止点击劫持
add_header X-Frame-Options "SAMEORIGIN" always;

# 防止 MIME 类型嗅探
add_header X-Content-Type-Options "nosniff" always;

# XSS 保护
add_header X-XSS-Protection "1; mode=block" always;

# CSP (内容安全策略)
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline';" always;
```

#### 6.3.3 请求限流
```nginx
# 限制请求速率
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

location /api/ {
    limit_req zone=api_limit burst=20 nodelay;
    # ...
}

# 限制连接数
limit_conn_zone $binary_remote_addr zone=addr:10m;

location /api/ {
    limit_conn addr 10;
    # ...
}
```

### 6.4 负载均衡策略

#### 6.4.1 Round Robin (默认)
```nginx
upstream ragflow_api {
    server api1:9380;
    server api2:9380;
    server api3:9380;
}
```
- 平均分配请求
- 简单高效

#### 6.4.2 Least Connections
```nginx
upstream ragflow_api {
    least_conn;
    server api1:9380;
    server api2:9380;
    server api3:9380;
}
```
- 选择连接数最少的服务器
- 适合长连接

#### 6.4.3 IP Hash
```nginx
upstream ragflow_api {
    ip_hash;
    server api1:9380;
    server api2:9380;
    server api3:9380;
}
```
- 同一 IP 总是路由到同一服务器
- 适合有状态会话

#### 6.4.4 权重配置
```nginx
upstream ragflow_api {
    server api1:9380 weight=3;   # 30%
    server api2:9380 weight=2;   # 20%
    server api3:9380 weight=5;   # 50%
}
```

#### 6.4.5 健康检查
```nginx
upstream ragflow_api {
    server api1:9380 max_fails=3 fail_timeout=30s;
    server api2:9380 max_fails=3 fail_timeout=30s;
    # 3 次失败后，30 秒内不再转发请求
}
```

### 6.5 特殊场景处理

#### 6.5.1 WebSocket 代理
```nginx
location /ws/ {
    proxy_pass http://ragflow_api;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400;  # 24 小时
}
```

#### 6.5.2 大文件上传
```nginx
client_max_body_size 100M;
client_body_buffer_size 10M;
client_body_timeout 60s;
```

#### 6.5.3 慢速攻击防护
```nginx
client_body_timeout 10s;
client_header_timeout 10s;
send_timeout 10s;
```

---

## 7. 部署与运维

### 7.1 Docker 部署

#### Dockerfile 配置
```dockerfile
FROM nginx:alpine

# 复制配置文件
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/ragflow.conf /etc/nginx/conf.d/ragflow.conf
COPY nginx/proxy.conf /etc/nginx/proxy.conf

# 复制 SSL 证书
COPY nginx/ssl/ /etc/nginx/ssl/

# 复制前端静态文件
COPY web/dist/ /usr/share/nginx/html/

# 暴露端口
EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

#### Docker Compose 配置
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ragflow.conf:/etc/nginx/conf.d/ragflow.conf:ro
      - ./nginx/proxy.conf:/etc/nginx/proxy.conf:ro
      - ../web/dist:/usr/share/nginx/html:ro
      - ./ragflow-logs/nginx:/var/log/nginx
    depends_on:
      - ragflow-api
    networks:
      - ragflow
    restart: unless-stopped
```

### 7.2 日志管理

#### 7.2.1 日志格式
```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" '
                'rt=$request_time uct="$upstream_connect_time" '
                'uht="$upstream_header_time" urt="$upstream_response_time"';
```

#### 7.2.2 日志轮转
```bash
# logrotate 配置
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 nginx adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

### 7.3 监控指标

#### 7.3.1 Nginx Status 模块
```nginx
location /nginx_status {
    stub_status on;
    access_log off;
    allow 127.0.0.1;
    deny all;
}
```

**输出示例**:
```
Active connections: 123
server accepts handled requests
 12345 12345 123456
Reading: 10 Writing: 20 Waiting: 93
```

#### 7.3.2 关键指标
- **Active connections**: 当前活跃连接数
- **QPS**: 每秒请求数
- **响应时间**: P50, P95, P99
- **错误率**: 4xx, 5xx 响应比例
- **带宽使用**: 入站/出站流量

---

## 8. 故障排查

### 8.1 常见问题

#### 8.1.1 502 Bad Gateway
**原因**:
- 后端服务未启动
- 后端服务响应超时
- Upstream 配置错误

**排查**:
```bash
# 检查后端服务
docker ps | grep ragflow-api

# 检查 Nginx 错误日志
tail -f /var/log/nginx/error.log

# 测试后端连接
curl http://ragflow-api:9380/api/health
```

#### 8.1.2 504 Gateway Timeout
**原因**:
- 后端处理时间过长
- `proxy_read_timeout` 设置过小

**解决**:
```nginx
location /api/ {
    proxy_read_timeout 300s;  # 增加超时时间
}
```

#### 8.1.3 413 Request Entity Too Large
**原因**:
- 上传文件超过限制

**解决**:
```nginx
client_max_body_size 100M;
```

### 8.2 性能调优

#### 8.2.1 连接数优化
```nginx
# 增加 worker 连接数
events {
    worker_connections 4096;
}

# 调整系统限制
# /etc/security/limits.conf
nginx soft nofile 65535
nginx hard nofile 65535
```

#### 8.2.2 缓冲区优化
```nginx
proxy_buffer_size 16k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;
```

---

## 9. 总结

### 9.1 层的特点
- ✅ **高性能**: 事件驱动、异步 I/O
- ✅ **高可用**: 负载均衡、健康检查
- ✅ **安全**: SSL/TLS、安全头、限流
- ✅ **灵活**: 丰富的配置选项
- ✅ **可扩展**: 支持多后端实例

### 9.2 最佳实践
1. **使用 HTTPS**: 强制所有流量使用 HTTPS
2. **启用 Gzip**: 减少带宽消耗
3. **配置缓存**: 合理设置缓存策略
4. **日志监控**: 定期分析日志和指标
5. **安全加固**: 限流、安全头、防火墙
6. **定期更新**: 保持 Nginx 版本最新

---

**文档版本**: 1.0  
**最后更新**: 2025-12-18  
**维护者**: RAGFlow 团队
