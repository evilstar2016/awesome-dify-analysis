# 06 - 基础设施层 (Infrastructure Layer)

## 1. 层职责与定位

### 1.1 职责
基础设施层提供底层技术支撑：
- **关系数据库**: MySQL/PostgreSQL 存储结构化数据
- **搜索引擎**: Elasticsearch/OpenSearch/Infinity 向量检索
- **缓存队列**: Redis 缓存和消息队列
- **对象存储**: MinIO (S3兼容) 存储文件
- **容器编排**: Docker + Docker Compose 部署
- **服务发现**: 内部服务互联

### 1.2 定位
- **架构位置**: 最底层，被数据访问层调用
- **技术选型**: 开源成熟技术栈
- **部署方式**: Docker 容器化
- **核心配置**: `docker/docker-compose.yml`

---

## 2. 主要组件/模块

### 2.1 Docker Compose 架构

#### 文件结构
```
docker/
├── docker-compose.yml           # 主配置文件
├── docker-compose-base.yml      # 基础服务（开发环境）
├── docker-compose-macos.yml     # macOS 专用配置
├── docker-compose-CN-oc9.yml    # 中国区 OceanBase 配置
├── service_conf.yaml.template   # 服务配置模板
├── entrypoint.sh                # 容器启动脚本
├── launch_backend_service.sh    # 后端服务启动脚本
├── init.sql                     # MySQL 初始化脚本
├── infinity_conf.toml           # Infinity 配置
└── nginx/                       # Nginx 配置目录
```

### 2.2 核心服务定义

#### 2.2.1 MySQL 服务

**配置**:
```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: ragflow-mysql
    environment:
      MYSQL_ROOT_PASSWORD: infiniflow
      MYSQL_DATABASE: ragflow
      MYSQL_USER: ragflow
      MYSQL_PASSWORD: infiniflow
      TZ: Asia/Shanghai
    ports:
      - "3306:3306"
    volumes:
      - ./ragflow-mysql:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    command: 
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --max_connections=1000
      - --innodb_buffer_pool_size=2G
      - --innodb_log_file_size=256M
    restart: unless-stopped
    networks:
      - ragflow
```

**数据库初始化** (`init.sql`):
```sql
-- 创建数据库
CREATE DATABASE IF NOT EXISTS `ragflow` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE `ragflow`;

-- 创建用户
CREATE USER IF NOT EXISTS 'ragflow'@'%' IDENTIFIED BY 'infiniflow';
GRANT ALL PRIVILEGES ON ragflow.* TO 'ragflow'@'%';
FLUSH PRIVILEGES;

-- 性能优化配置
SET GLOBAL max_connections = 1000;
SET GLOBAL innodb_buffer_pool_size = 2147483648; -- 2GB
```

**性能参数**:
- `max_connections`: 1000 (最大连接数)
- `innodb_buffer_pool_size`: 2GB (缓冲池)
- `innodb_log_file_size`: 256MB (日志文件)
- `character-set-server`: utf8mb4 (字符集)

#### 2.2.2 Elasticsearch 服务

**配置**:
```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.3
    container_name: ragflow-es
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
      - cluster.routing.allocation.disk.threshold_enabled=false
      - bootstrap.memory_lock=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - ./ragflow-es:/usr/share/elasticsearch/data
    restart: unless-stopped
    networks:
      - ragflow
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
```

**或使用 Infinity** (高性能向量数据库):
```yaml
services:
  infinity:
    image: infiniflow/infinity:v0.2.1
    container_name: ragflow-infinity
    ports:
      - "23817:23817"
      - "9090:9090"
    volumes:
      - ./ragflow-infinity:/var/infinity
      - ./infinity_conf.toml:/var/infinity/conf/infinity_conf.toml:ro
    restart: unless-stopped
    networks:
      - ragflow
```

**Infinity 配置** (`infinity_conf.toml`):
```toml
[general]
version = "0.2.1"
timezone = "Asia/Shanghai"

[network]
server_address = "0.0.0.0"
http_port = 23817
grpc_port = 9090

[storage]
data_dir = "/var/infinity/data"
log_dir = "/var/infinity/log"
temp_dir = "/var/infinity/tmp"

[resource]
cpu_limit = 4
memory_limit = "8GB"
```

#### 2.2.3 Redis 服务

**配置**:
```yaml
services:
  redis:
    image: redis:7.2-alpine
    container_name: ragflow-redis
    command: redis-server --requirepass infiniflow --maxmemory 4gb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - ./ragflow-redis:/data
    restart: unless-stopped
    networks:
      - ragflow
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
```

**Redis 配置参数**:
- `requirepass`: 密码认证
- `maxmemory`: 最大内存 4GB
- `maxmemory-policy`: LRU 淘汰策略
- `appendonly`: AOF 持久化

#### 2.2.4 MinIO 服务

**配置**:
```yaml
services:
  minio:
    image: minio/minio:latest
    container_name: ragflow-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
      MINIO_REGION_NAME: us-east-1
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - ./ragflow-minio:/data
    restart: unless-stopped
    networks:
      - ragflow
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
```

**MinIO 初始化**:
```bash
# 创建 bucket (在启动后执行)
mc alias set ragflow http://localhost:9000 minioadmin minioadmin
mc mb ragflow/ragflow
mc policy set public ragflow/ragflow
```

#### 2.2.5 应用服务

**RAGFlow API 服务**:
```yaml
services:
  ragflow-api:
    build:
      context: ../
      dockerfile: Dockerfile
    container_name: ragflow-api
    environment:
      - PYTHONUNBUFFERED=1
      - MYSQL_HOST=mysql
      - MYSQL_PORT=3306
      - MYSQL_USER=ragflow
      - MYSQL_PASSWORD=infiniflow
      - MYSQL_DATABASE=ragflow
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=infiniflow
      - ES_HOST=elasticsearch
      - ES_PORT=9200
      - MINIO_ENDPOINT=minio:9000
      - MINIO_ACCESS_KEY=minioadmin
      - MINIO_SECRET_KEY=minioadmin
    ports:
      - "9380:9380"
      - "9381:9381"
    volumes:
      - ../api:/ragflow/api
      - ../rag:/ragflow/rag
      - ../common:/ragflow/common
      - ./ragflow-logs:/ragflow/logs
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy
      minio:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - ragflow
```

**Nginx 网关**:
```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: ragflow-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ragflow.conf:/etc/nginx/conf.d/ragflow.conf:ro
      - ../web/dist:/usr/share/nginx/html:ro
      - ./ragflow-logs/nginx:/var/log/nginx
    depends_on:
      - ragflow-api
    restart: unless-stopped
    networks:
      - ragflow
```

### 2.3 网络配置

```yaml
networks:
  ragflow:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

**服务内部域名**:
- `mysql` → `172.28.0.2:3306`
- `redis` → `172.28.0.3:6379`
- `elasticsearch` → `172.28.0.4:9200`
- `minio` → `172.28.0.5:9000`
- `ragflow-api` → `172.28.0.6:9380`
- `nginx` → `172.28.0.7:80`

### 2.4 数据卷管理

```yaml
volumes:
  ragflow-mysql:
    driver: local
  ragflow-es:
    driver: local
  ragflow-redis:
    driver: local
  ragflow-minio:
    driver: local
  ragflow-logs:
    driver: local
```

**卷路径映射**:
| 容器内路径 | 宿主机路径 | 说明 |
|-----------|-----------|------|
| `/var/lib/mysql` | `./ragflow-mysql` | MySQL 数据 |
| `/usr/share/elasticsearch/data` | `./ragflow-es` | ES 数据 |
| `/data` | `./ragflow-redis` | Redis 数据 |
| `/data` | `./ragflow-minio` | MinIO 数据 |
| `/ragflow/logs` | `./ragflow-logs` | 应用日志 |

---

## 3. 服务端口映射

| 服务 | 内部端口 | 外部端口 | 协议 | 用途 |
|------|---------|---------|------|------|
| Nginx | 80 | 80 | HTTP | 前端入口 |
| Nginx | 443 | 443 | HTTPS | 前端入口（加密） |
| MySQL | 3306 | 3306 | TCP | 数据库 |
| Redis | 6379 | 6379 | TCP | 缓存队列 |
| Elasticsearch | 9200 | 9200 | HTTP | 搜索API |
| Elasticsearch | 9300 | 9300 | TCP | 集群通信 |
| MinIO | 9000 | 9000 | HTTP | 对象存储API |
| MinIO | 9001 | 9001 | HTTP | 管理控制台 |
| RAGFlow API | 9380 | 9380 | HTTP | API 服务 |
| RAGFlow Admin | 9381 | 9381 | HTTP | 管理服务 |
| Infinity | 23817 | 23817 | HTTP | 向量检索 |

---

## 4. 部署与管理

### 4.1 启动服务

#### 完整部署
```bash
cd docker
docker compose up -d
```

#### 仅启动基础服务（开发环境）
```bash
cd docker
docker compose -f docker-compose-base.yml up -d
```

#### 查看服务状态
```bash
docker compose ps
```

### 4.2 服务管理

#### 停止服务
```bash
docker compose stop
```

#### 重启服务
```bash
docker compose restart [service_name]
```

#### 查看日志
```bash
docker compose logs -f [service_name]
```

#### 进入容器
```bash
docker exec -it ragflow-api bash
```

### 4.3 数据备份

#### MySQL 备份
```bash
docker exec ragflow-mysql mysqldump -u ragflow -pinfiniflow ragflow > backup.sql
```

#### Redis 备份
```bash
docker exec ragflow-redis redis-cli --rdb /data/dump.rdb
docker cp ragflow-redis:/data/dump.rdb ./backup/
```

#### Elasticsearch 备份
```bash
curl -XPUT "http://localhost:9200/_snapshot/backup" -H 'Content-Type: application/json' -d '{
  "type": "fs",
  "settings": {
    "location": "/usr/share/elasticsearch/backup"
  }
}'
```

#### MinIO 备份
```bash
mc mirror ragflow/ragflow ./backup/minio
```

### 4.4 监控与健康检查

#### 服务健康检查
```bash
# MySQL
docker exec ragflow-mysql mysqladmin ping -u root -pinfiniflow

# Redis
docker exec ragflow-redis redis-cli ping

# Elasticsearch
curl http://localhost:9200/_cluster/health

# MinIO
curl http://localhost:9000/minio/health/live
```

#### 资源监控
```bash
# 实时监控
docker stats

# 磁盘使用
docker system df
```

---

## 5. 性能优化

### 5.1 MySQL 优化

**配置调优**:
```ini
# my.cnf
[mysqld]
max_connections = 1000
innodb_buffer_pool_size = 2G
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
query_cache_size = 0
query_cache_type = 0
```

### 5.2 Elasticsearch 优化

**JVM 堆内存**:
```yaml
environment:
  - "ES_JAVA_OPTS=-Xms4g -Xmx4g"  # 4GB 堆内存
```

**分片策略**:
```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "30s"
  }
}
```

### 5.3 Redis 优化

**内存策略**:
```bash
redis-server --maxmemory 4gb --maxmemory-policy allkeys-lru
```

### 5.4 MinIO 优化

**多盘并行**:
```bash
minio server /data1 /data2 /data3 /data4
```

---

## 6. 安全加固

### 6.1 网络隔离

```yaml
networks:
  ragflow-frontend:
    driver: bridge
  ragflow-backend:
    driver: bridge
    internal: true  # 仅内部访问
```

### 6.2 密码管理

**使用 Docker Secrets**:
```yaml
services:
  mysql:
    secrets:
      - mysql_root_password
      - mysql_password

secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt
  mysql_password:
    file: ./secrets/mysql_password.txt
```

### 6.3 防火墙规则

```bash
# 只允许必要端口
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny 3306/tcp
ufw deny 9200/tcp
```

---

## 7. 故障排查

### 7.1 常见问题

#### MySQL 连接失败
```bash
# 检查服务状态
docker compose logs mysql

# 检查端口
netstat -tulnp | grep 3306

# 测试连接
docker exec ragflow-api mysql -h mysql -u ragflow -pinfiniflow
```

#### Elasticsearch 内存不足
```bash
# 增加堆内存
ES_JAVA_OPTS=-Xms8g -Xmx8g

# 检查内存使用
curl http://localhost:9200/_nodes/stats/jvm
```

#### Redis 连接超时
```bash
# 检查网络
docker exec ragflow-api ping redis

# 检查 Redis 配置
docker exec ragflow-redis redis-cli config get timeout
```

---

## 8. 总结

### 8.1 层的特点
- ✅ **容器化部署**: Docker + Docker Compose
- ✅ **微服务架构**: 独立服务，易扩展
- ✅ **数据持久化**: 卷挂载保证数据安全
- ✅ **健康检查**: 自动重启故障服务
- ✅ **网络隔离**: 内部网络安全

### 8.2 最佳实践
1. **数据备份**: 定期备份关键数据
2. **资源监控**: 实时监控服务状态
3. **日志管理**: 集中式日志收集
4. **安全加固**: 密码管理、网络隔离
5. **性能调优**: 根据负载调整配置

---

**文档版本**: 1.0  
**最后更新**: 2025-12-18  
**维护者**: RAGFlow 团队
