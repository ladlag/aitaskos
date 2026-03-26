# 部署与基础设施设计

## 1. 部署架构总览

### 1.1 容器化部署架构

```
┌──────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                        │
│                                                              │
│  ┌──────────── Namespace: aitaskos ─────────────────────┐   │
│  │                                                       │   │
│  │  ┌─────────────────────────────────────────────────┐ │   │
│  │  │            Ingress Controller                    │ │   │
│  │  │          (Nginx Ingress / APISIX)                │ │   │
│  │  └──────────────────┬──────────────────────────────┘ │   │
│  │                     │                                 │   │
│  │  ┌─────────────┬────┴────────┬──────────────┐       │   │
│  │  ▼             ▼             ▼              ▼       │   │
│  │  ┌─────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │   │
│  │  │ Web │  │ API     │  │ Agent   │  │ SSE     │   │   │
│  │  │ UI  │  │ Server  │  │ Gateway │  │ Server  │   │   │
│  │  │ ×2  │  │ ×3      │  │ ×2      │  │ ×2      │   │   │
│  │  └─────┘  └────┬────┘  └────┬────┘  └────┬────┘   │   │
│  │                │            │             │         │   │
│  │  ┌─────────────┴────────────┴─────────────┘         │   │
│  │  │                                                   │   │
│  │  ▼                                                   │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │          Internal Service Mesh                │   │   │
│  │  │                                              │   │   │
│  │  │  ┌────────────┐ ┌───────────┐ ┌───────────┐ │   │   │
│  │  │  │ Temporal   │ │ Dify      │ │ LiteLLM   │ │   │   │
│  │  │  │ Server ×1  │ │ Server ×1 │ │ Proxy ×1  │ │   │   │
│  │  │  └────────────┘ └───────────┘ └───────────┘ │   │   │
│  │  │                                              │   │   │
│  │  │  ┌────────────┐ ┌───────────┐ ┌───────────┐ │   │   │
│  │  │  │ Keycloak   │ │ PaddleOCR │ │ Ollama    │ │   │   │
│  │  │  │ ×1         │ │ ×1        │ │ ×1 (GPU)  │ │   │   │
│  │  │  └────────────┘ └───────────┘ └───────────┘ │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │                                                       │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────── Namespace: data ─────────────────────────┐   │
│  │                                                       │   │
│  │  ┌────────────┐ ┌───────────┐ ┌───────────┐         │   │
│  │  │ PostgreSQL │ │ Redis     │ │ Milvus    │         │   │
│  │  │ ×1 (HA可选)│ │ ×1 (HA)  │ │ ×1        │         │   │
│  │  └────────────┘ └───────────┘ └───────────┘         │   │
│  │                                                       │   │
│  │  ┌────────────┐ ┌───────────┐                        │   │
│  │  │ Kafka      │ │ MinIO     │                        │   │
│  │  │ ×3 (集群)  │ │ ×1        │                        │   │
│  │  └────────────┘ └───────────┘                        │   │
│  │                                                       │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────── Namespace: monitoring ───────────────────┐   │
│  │                                                       │   │
│  │  ┌────────────┐ ┌───────────┐ ┌───────────┐         │   │
│  │  │ Prometheus │ │ Grafana   │ │ Jaeger    │         │   │
│  │  └────────────┘ └───────────┘ └───────────┘         │   │
│  │                                                       │   │
│  │  ┌────────────┐                                      │   │
│  │  │ OpenSearch │                                      │   │
│  │  └────────────┘                                      │   │
│  │                                                       │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 服务清单

| 服务 | 副本数 | 资源需求 | 说明 |
|------|--------|---------|------|
| web-ui | 2 | 0.5C / 512M | 前端静态资源 (Nginx) |
| api-server | 3 | 2C / 4G | Spring Boot 后端 |
| agent-gateway | 2 | 1C / 2G | Agent 网关 |
| sse-server | 2 | 1C / 2G | SSE 推送服务 (WebFlux) |
| temporal-server | 1 | 2C / 4G | 工作流引擎 |
| temporal-worker | 2 | 2C / 4G | 工作流 Worker |
| dify | 1 | 2C / 4G | AI 工作流平台 |
| langfuse | 1 | 1C / 2G | AI Agent 可观测性 |
| litellm | 1 | 1C / 2G | 模型网关 |
| keycloak | 1 | 1C / 2G | 认证服务 |
| paddleocr | 1 | 2C / 4G | OCR 服务 |
| ollama | 1 | 4C / 16G + GPU | 本地模型服务 |
| postgresql | 1 | 2C / 4G | 主数据库 |
| redis | 1 | 1C / 2G | 缓存 |
| milvus | 1 | 2C / 8G | 向量数据库 |
| kafka | 3 | 2C / 4G | 消息队列 |
| minio | 1 | 1C / 2G | 对象存储 |
| prometheus | 1 | 1C / 2G | 监控 |
| grafana | 1 | 0.5C / 1G | 监控仪表盘 |
| jaeger | 1 | 1C / 2G | 链路追踪 |
| opensearch | 1 | 2C / 4G | 日志分析 |

### 1.3 最低资源需求

| 环境 | CPU | 内存 | 存储 | GPU |
|------|-----|------|------|-----|
| 开发环境 | 16C | 32G | 200G SSD | 可选 |
| 测试环境 | 32C | 64G | 500G SSD | 1× |
| 生产环境 | 64C+ | 128G+ | 1T+ SSD | 1×+ |

## 2. Docker Compose 开发环境

```yaml
# docker-compose.yml (开发环境)
version: '3.8'

services:
  # ============ 应用服务 ============
  web-ui:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - api-server

  api-server:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      DB_HOST: postgresql
      REDIS_HOST: redis
      KAFKA_BROKERS: kafka:9092
      MILVUS_HOST: milvus
      TEMPORAL_HOST: temporal
      KEYCLOAK_URL: http://keycloak:8080
      MINIO_ENDPOINT: http://minio:9000
    depends_on:
      - postgresql
      - redis
      - kafka
      - milvus
      - temporal
      - keycloak
      - minio

  agent-gateway:
    build: ./agent-gateway
    ports:
      - "8081:8081"
    environment:
      LITELLM_URL: http://litellm:4000
      DIFY_URL: http://dify:3000

  sse-server:
    build: ./sse-server
    ports:
      - "8082:8082"
    environment:
      REDIS_HOST: redis
      KAFKA_BROKERS: kafka:9092

  temporal-worker:
    build: ./temporal-worker
    environment:
      TEMPORAL_HOST: temporal
      API_SERVER_URL: http://api-server:8080

  # ============ 基础设施 ============
  postgresql:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: aitaskos
      POSTGRES_USER: aitaskos
      POSTGRES_PASSWORD: aitaskos_dev
    volumes:
      - pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7.2-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  milvus:
    image: milvusdb/milvus:v2.4-latest
    ports:
      - "19530:19530"
    volumes:
      - milvus_data:/var/lib/milvus

  kafka:
    image: bitnami/kafka:3.7
    ports:
      - "9092:9092"
    environment:
      KAFKA_CFG_NODE_ID: 0
      KAFKA_CFG_PROCESS_ROLES: controller,broker
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka:9093
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
    volumes:
      - kafka_data:/bitnami/kafka

  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

  # ============ AI 服务 ============
  temporal:
    image: temporalio/auto-setup:latest
    ports:
      - "7233:7233"
    environment:
      DB: postgresql
      POSTGRES_SEEDS: postgresql
      POSTGRES_USER: aitaskos
      POSTGRES_PWD: aitaskos_dev
    depends_on:
      - postgresql

  temporal-ui:
    image: temporalio/ui:latest
    ports:
      - "8088:8080"
    environment:
      TEMPORAL_ADDRESS: temporal:7233

  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    ports:
      - "8180:8080"
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    command: start-dev

  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    ports:
      - "4000:4000"
    volumes:
      - ./config/litellm_config.yaml:/app/config.yaml
    command: --config /app/config.yaml

  dify:
    image: langgenius/dify-api:latest
    ports:
      - "5001:5001"

  langfuse:
    image: langfuse/langfuse:latest
    ports:
      - "3002:3000"
    environment:
      DATABASE_URL: postgresql://aitaskos:aitaskos_dev@postgresql:5432/langfuse
      NEXTAUTH_URL: http://localhost:3002
      NEXTAUTH_SECRET: langfuse_dev_secret
      SALT: langfuse_dev_salt
    depends_on:
      - postgresql

  # ============ 监控 ============
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"

volumes:
  pg_data:
  redis_data:
  milvus_data:
  kafka_data:
  minio_data:
  grafana_data:
```

## 3. Kubernetes 生产部署

### 3.1 Helm Chart 结构

```
helm/aitaskos/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── api-server/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── configmap.yaml
│   ├── agent-gateway/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── sse-server/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── temporal-worker/
│   │   ├── deployment.yaml
│   │   └── configmap.yaml
│   ├── web-ui/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── ingress.yaml
└── charts/                    # 子 Chart（依赖）
    ├── postgresql/
    ├── redis/
    ├── kafka/
    └── milvus/
```

### 3.2 HPA 自动扩缩容

```yaml
# api-server HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## 4. 网络与安全

### 4.1 网络策略

```
外部流量 → Ingress (HTTPS) → Web UI / API Server
                                  │
                    内部网络（ClusterIP）
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         Agent GW    Temporal    Kafka
              │          │          │
              ▼          ▼          ▼
          LLM APIs   PostgreSQL  Redis
```

### 4.2 安全措施

| 层级 | 措施 |
|------|------|
| 网络 | HTTPS / TLS 加密、网络策略隔离 |
| 认证 | Keycloak SSO (OAuth2 / OIDC) |
| 授权 | RBAC 角色权限控制 |
| API | JWT Token 验证、API 限流 |
| 数据 | PostgreSQL 连接加密、敏感数据脱敏 |
| 审计 | 全操作审计日志 |
| 密钥 | Kubernetes Secrets / HashiCorp Vault |

## 5. 监控与告警

### 5.1 监控指标

| 类别 | 指标 | 告警阈值 |
|------|------|---------|
| 系统 | CPU 使用率 | > 80% 持续 5min |
| 系统 | 内存使用率 | > 85% 持续 5min |
| 系统 | 磁盘使用率 | > 90% |
| API | 请求延迟 P99 | > 3s |
| API | 错误率 | > 5% |
| Agent | 执行成功率 | < 95% |
| Agent | 平均执行时间 | > 60s |
| Kafka | 消费延迟 | > 10000 条 |
| DB | 连接池使用率 | > 80% |
| DB | 慢查询数 | > 10/min |

### 5.2 Grafana 仪表盘

```
仪表盘列表:
├── 系统总览
│   ├── 服务健康状态
│   ├── 请求量 / 延迟 / 错误率
│   └── 资源使用趋势
├── Agent 监控
│   ├── Agent 可用性
│   ├── 执行成功率
│   ├── 平均执行时间
│   └── 队列积压
├── 任务监控
│   ├── 任务数量趋势
│   ├── 各状态占比
│   └── Workflow 执行时间
├── 数据层监控
│   ├── PostgreSQL 性能
│   ├── Redis 命中率
│   ├── Kafka 吞吐量
│   └── Milvus 查询延迟
└── 业务监控
    ├── 活跃用户数
    ├── PRD 生成量
    └── 知识库使用量
```

## 6. 备份与恢复

### 6.1 备份策略

| 数据 | 备份方式 | 频率 | 保留 |
|------|---------|------|------|
| PostgreSQL | pg_dump 全量 + WAL 增量 | 全量每天、增量实时 | 30 天 |
| Redis | RDB + AOF | RDB 每小时 | 7 天 |
| MinIO | 跨区复制 | 实时 | 永久 |
| Milvus | 快照备份 | 每天 | 30 天 |
| Kafka | Topic 备份 | 每天 | 7 天 |

### 6.2 灾难恢复

```
RTO (恢复时间目标): < 4 小时
RPO (恢复点目标):   < 1 小时

恢复流程:
1. 恢复基础设施（K8s 集群）
2. 恢复数据层（PostgreSQL → Redis → Milvus → MinIO）
3. 启动应用服务
4. 验证数据完整性
5. 恢复对外服务
```
