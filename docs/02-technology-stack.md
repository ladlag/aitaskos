# 技术选型方案

> 所有组件均选用开源、可免费商用的技术，确保无许可证风险。

## 1. 技术选型总览

| 层级 | 技术 | 版本 | 许可证 | 用途 |
|------|------|------|--------|------|
| **前端框架** | Vue 3 | 3.4+ | MIT | 前端 SPA 框架 |
| **前端 UI** | Element Plus | 2.x | MIT | UI 组件库 |
| **富文本编辑** | TipTap | 2.x | MIT | PRD 编辑器 |
| **流程图** | AntV X6 | 2.x | MIT | 流程图、架构图渲染与编辑 |
| **图表** | Apache ECharts | 5.x | Apache 2.0 | 数据可视化 |
| **状态管理** | Pinia | 2.x | MIT | 前端状态管理 |
| **后端框架** | Spring Boot | 3.2+ | Apache 2.0 | 后端主框架 |
| **API 网关** | Apache APISIX | 3.x | Apache 2.0 | API 路由、限流、认证 |
| **Agent 框架** | LangChain4j | 0.35+ | Apache 2.0 | 自建 Agent 可选框架（独立于平台） |
| **Agent 可观测** | Langfuse | 3.x | MIT | AI Agent 推理追踪、成本统计、质量评估 |
| **遥测标准** | OpenTelemetry | 最新 | Apache 2.0 | 统一遥测数据采集标准 |
| **工作流引擎** | Temporal | 1.x | MIT | 长流程编排 |
| **AI 工作流** | Dify | 0.8+ | Apache 2.0 | 低代码 AI 工作流 |
| **模型网关** | LiteLLM | 1.x | MIT | 多模型统一接入 |
| **本地模型** | Ollama | 0.3+ | MIT | 本地大模型服务 |
| **主数据库** | PostgreSQL | 16+ | PostgreSQL License | 结构化数据存储 |
| **缓存** | Redis | 7.x | BSD-3 | 缓存、会话、分布式锁 |
| **向量数据库** | Milvus | 2.4+ | Apache 2.0 | 向量检索（知识库） |
| **消息队列** | Apache Kafka | 3.x | Apache 2.0 | 事件驱动、异步消息 |
| **对象存储** | MinIO | RELEASE | AGPL-3.0 | 文件、附件存储 |
| **文件解析** | Apache Tika | 2.x | Apache 2.0 | PDF/Word/Excel 解析 |
| **OCR** | PaddleOCR | 2.x | Apache 2.0 | 图片文字识别 |
| **浏览器自动化** | Playwright | 1.x | Apache 2.0 | HTML 页面抓取与解析 |
| **容器化** | Docker | 24+ | Apache 2.0 | 容器化部署 |
| **容器编排** | Kubernetes | 1.28+ | Apache 2.0 | 集群编排管理 |
| **监控** | Prometheus | 2.x | Apache 2.0 | 指标采集 |
| **可视化监控** | Grafana | 10.x | AGPL-3.0 | 监控仪表盘 |
| **链路追踪** | Jaeger | 1.x | Apache 2.0 | 分布式链路追踪 |
| **日志** | OpenSearch | 2.x | Apache 2.0 | 日志收集分析（纯开源方案） |
| **认证** | Keycloak | 24+ | Apache 2.0 | SSO / OAuth2 / OIDC |

## 2. 各层详细选型说明

### 2.1 前端层

```
Vue 3 (核心框架)
├── Element Plus (UI 组件)
├── TipTap (富文本编辑器 → PRD编辑)
├── AntV X6 (流程图/拓扑图编辑)
├── Apache ECharts (数据图表)
├── Pinia (状态管理)
├── Vue Router (路由)
├── Axios (HTTP 请求)
└── EventSource API (SSE 实时推送)
```

**选型理由**：
- Vue 3 生态成熟，团队学习成本低，Composition API 适合复杂交互
- Element Plus 提供完善的企业级组件（表格、表单、树形控件等）
- TipTap 基于 ProseMirror，支持协同编辑、结构化内容、自定义扩展
- AntV X6 支持复杂图编辑，BPMN 节点、连线、布局算法等

### 2.2 后端层

```
Spring Boot 3.2+ (JDK 17+)
├── Spring WebFlux (响应式/SSE 推送)
├── Spring Security + Keycloak (认证授权)
├── Spring Data JPA (ORM)
├── Flyway (数据库迁移)
├── MapStruct (对象映射)
└── SpringDoc OpenAPI (API 文档)
```

**选型理由**：
- Spring Boot 3.x 生态完善，企业级验证充分
- WebFlux 支持 SSE/流式推送，满足实时反馈需求
- JDK 17+ 提供 Records、Sealed Classes 等现代语言特性

### 2.3 Agent 层

```
Agent 接入层（所有 Agent 均为外部独立服务）
├── Dify Agent (可视化 AI 工作流编排)
│   ├── BA 需求分析 Agent
│   ├── PRD 生成 Agent
│   ├── 流程设计 Agent
│   └── 自定义工作流 Agent
├── 自建 Agent (标准协议接入)
│   └── 任何遵循平台标准协议的 Agent
└── 第三方 Agent (适配器接入)
    └── 外部 AI 服务

协议支持层
├── A2A Protocol (Agent-to-Agent, Google/Linux Foundation 开放标准)
│   ├── Agent Card 注册与发现
│   ├── Task Model 标准状态流转
│   └── SSE 推送与多模态消息
├── MCP Protocol (Model Context Protocol, Anthropic/Linux Foundation 开放标准)
│   ├── 平台 MCP Server（暴露 DSL/知识库/文件工具给 Agent）
│   └── 标准化工具发现与调用
└── 自定义协议（向后兼容）
    ├── HTTP REST 回调
    ├── Dify API 适配
    └── gRPC 适配

平台内部能力（非 Agent，平台基础服务）
├── 文件预处理 (Apache Tika + PaddleOCR)
├── DSL 合并验证
├── 输入/输出格式转换
└── 任务路由与健康检查
```

**选型理由**：
- Dify 提供可视化 AI 工作流编排，用户可快速创建和迭代业务 Agent
- 平台不耦合任何业务 Agent，只提供标准协议接入能力
- 兼容 A2A 和 MCP 两大行业标准协议，降低 Agent 接入门槛，实现跨平台互操作
- LangChain4j 可用于自建 Agent（独立于平台部署），但不是平台依赖

### 2.4 AI 模型层

```
模型接入策略
├── LiteLLM (模型网关/路由)
│   ├── DeepSeek V3/R1 (主力模型 - 推理/生成)
│   ├── Qwen 2.5 (备选模型)
│   ├── GLM-4 (备选模型)
│   └── 其他兼容 OpenAI API 的模型
└── Ollama (本地部署)
    ├── DeepSeek Coder (代码分析)
    ├── Qwen2.5 (通用推理)
    └── nomic-embed-text (向量嵌入)
```

**选型理由**：
- LiteLLM 统一所有模型调用接口，支持负载均衡、降级、计费
- DeepSeek 系列模型开源可商用，中文理解能力优秀
- Ollama 支持本地部署，满足数据安全/离线场景

### 2.5 数据层

```
数据存储策略
├── PostgreSQL (结构化数据)
│   ├── 任务数据
│   ├── Agent 数据
│   ├── DSL 数据 (JSONB)
│   ├── 用户数据
│   └── 审计日志
├── Redis (缓存与实时)
│   ├── 会话缓存
│   ├── 指令队列 (Redis Streams)
│   ├── 分布式锁
│   └── SSE 连接管理
├── Milvus (向量检索)
│   ├── 知识文档向量
│   ├── 历史需求向量
│   └── 项目知识向量
├── MinIO (文件存储)
│   ├── 上传文档
│   ├── 生成的 PRD
│   ├── 原型文件
│   └── 附件
└── Kafka (事件总线)
    ├── 任务事件
    ├── Agent 回调
    ├── 异步反馈
    └── 审计事件
```

### 2.6 基础设施层

```
部署与运维
├── Docker (容器化)
├── Kubernetes (编排)
├── Helm (包管理)
├── Prometheus + Grafana (监控)
├── Jaeger (链路追踪)
├── OpenSearch (日志分析)
└── Keycloak (统一认证)
```

## 3. 许可证合规性矩阵

| 组件 | 许可证 | 商业使用 | 修改分发 | 备注 |
|------|--------|----------|----------|------|
| Vue 3 | MIT | ✅ | ✅ | 无限制 |
| Element Plus | MIT | ✅ | ✅ | 无限制 |
| TipTap | MIT | ✅ | ✅ | 无限制 |
| AntV X6 | MIT | ✅ | ✅ | 无限制 |
| Spring Boot | Apache 2.0 | ✅ | ✅ | 无限制 |
| LangChain4j | Apache 2.0 | ✅ | ✅ | 无限制 |
| Temporal | MIT | ✅ | ✅ | 无限制 |
| Dify | Apache 2.0 | ✅ | ✅ | 商标限制 |
| PostgreSQL | PostgreSQL | ✅ | ✅ | 类 MIT |
| Redis | BSD-3 | ✅ | ✅ | 7.4+ 改为 RSALv2/SSPLv1，使用 7.2 或 Valkey |
| Milvus | Apache 2.0 | ✅ | ✅ | 无限制 |
| Kafka | Apache 2.0 | ✅ | ✅ | 无限制 |
| MinIO | AGPL-3.0 | ✅ | ⚠️ | 修改需开源，不修改可商用 |
| Grafana | AGPL-3.0 | ✅ | ⚠️ | 仅用于监控，不修改源码 |
| Keycloak | Apache 2.0 | ✅ | ✅ | 无限制 |
| PaddleOCR | Apache 2.0 | ✅ | ✅ | 无限制 |
| Langfuse | MIT | ✅ | ✅ | 无限制，自部署 |
| OpenTelemetry | Apache 2.0 | ✅ | ✅ | 无限制 |
| DeepSeek | MIT (模型权重) | ✅ | ✅ | 无限制 |

> **注意**：Redis 7.4+ 版本更改了许可证。建议使用 Redis 7.2（BSD-3）或 Valkey（BSD-3，Redis 兼容分支）。

## 4. 开发工具链

| 工具 | 用途 | 许可证 |
|------|------|--------|
| IntelliJ IDEA CE | Java IDE | Apache 2.0 |
| VS Code | 前端开发 | MIT |
| Git | 版本控制 | GPL-2.0 |
| Maven | 构建管理 | Apache 2.0 |
| Vite | 前端构建 | MIT |
| ESLint | 前端代码检查 | MIT |
| Checkstyle | Java 代码检查 | LGPL-2.1 |
| JUnit 5 | 单元测试 | EPL-2.0 |
| Vitest | 前端测试 | MIT |
