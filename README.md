# AiTaskOS — AI 数字员工平台

> 构建一个"多租户、可审计的 AI 任务调度与执行编排平台"，内置初级产品经理级 BA 数字员工能力。

## 项目简介

AiTaskOS 是一个 AI 数字员工平台（AI Digital Employee Platform），核心定位为 **AI 任务调度与执行编排中枢**。平台负责任务图管理、数字员工（Agent）标准化接入与调度执行，通过统一协议与回调机制管理所有执行过程与结果。

### 核心能力

- 🧠 **BA 数字员工**：需求理解、需求澄清、需求拆解、PRD 生成、流程设计、原型设计、需求评审、需求优化、存量系统升级设计
- 🔧 **平台能力**：任务管理、Agent 管理、调度编排、执行数据管理、审计与可观测
- 💬 **交互能力**：实时交互、人机协同、局部修改、多轮迭代、知识沉淀

### 数字员工模型

```
数字员工 = Workflow（流程）+ Agent（决策）+ Skill（能力）
         + Knowledge（知识）+ Memory（上下文）+ Queue（指令队列）
```

## 核心技术栈

| 层级 | 技术 | 许可证 |
|------|------|--------|
| 前端 | Vue 3 + Element Plus + TipTap + AntV X6 | MIT |
| 后端 | Spring Boot 3.x + Spring WebFlux | Apache 2.0 |
| Agent 框架 | LangChain4j | Apache 2.0 |
| 工作流 | Temporal | MIT |
| AI 工作流 | Dify | Apache 2.0 |
| 模型网关 | LiteLLM | MIT |
| 数据库 | PostgreSQL | PostgreSQL License |
| 缓存 | Redis 7.2 / Valkey | BSD-3 |
| 向量库 | Milvus | Apache 2.0 |
| 消息队列 | Apache Kafka | Apache 2.0 |
| 文件存储 | MinIO | AGPL-3.0 |
| 认证 | Keycloak | Apache 2.0 |
| AI 模型 | DeepSeek / Qwen（开源可商用） | MIT |

> 所有组件均为开源、可免费商用。

## 系统架构

```
┌────────────────────────────────────────────────────┐
│          用户交互层 (Vue3 + Element Plus)            │
│    对话区 │ 过程区 │ 产物区 │ 结构区                  │
├────────────────────────────────────────────────────┤
│          API 网关层 (Apache APISIX)                 │
├────────────────────────────────────────────────────┤
│       数字员工管理层 (Spring Boot 3.x)               │
├────────────────────────────────────────────────────┤
│      调度编排层 (Temporal + 指令队列)                 │
├────────────────────────────────────────────────────┤
│      Agent 网关层 (标准协议 + 适配器)                 │
├────────────────────────────────────────────────────┤
│    Agent 执行层 (内置Agent / Dify / 外部Agent)       │
├────────────────────────────────────────────────────┤
│   模型与工具层 (LiteLLM / Ollama / PaddleOCR)       │
├────────────────────────────────────────────────────┤
│  数据与知识层 (PostgreSQL / Redis / Milvus / Kafka)  │
└────────────────────────────────────────────────────┘
```

## 设计文档

| 文档 | 说明 |
|------|------|
| [01-系统总览与架构设计](docs/01-system-overview.md) | 系统定位、分层架构、核心机制、系统边界 |
| [02-技术选型方案](docs/02-technology-stack.md) | 各层技术选型、许可证合规矩阵 |
| [03-用户交互层设计](docs/03-user-interaction-design.md) | 四区布局、交互流程、组件设计、前端架构 |
| [04-数据模型设计](docs/04-data-model-design.md) | DSL 结构、数据库模型、通信协议 |
| [05-Agent 系统设计](docs/05-agent-system-design.md) | 9 个内置 Agent、Skill 平台、调度策略、Gateway |
| [06-API 设计规范](docs/06-api-design.md) | RESTful API、SSE 事件、错误码规范 |
| [07-工作流与编排设计](docs/07-workflow-design.md) | Temporal 工作流、指令队列编排、事件驱动、并发控制 |
| [08-部署与基础设施设计](docs/08-deployment-design.md) | Docker Compose、K8s 部署、监控告警、备份恢复 |
| [09-场景实现指南](docs/09-scenario-implementation.md) | 8 大场景实现细节、开发路线图 |

## 需求文档

| 文档 | 说明 |
|------|------|
| [数字员工-需求.docx](数字员工-需求.docx) | 核心需求与技术方案 |
| [数字员工-场景设计.docx](数字员工-场景设计.docx) | 8 大核心场景设计 |
| [数字员工-部分想法.docx](数字员工-部分想法.docx) | 架构升级思路 |

## 业务场景

| 阶段 | 场景 | 说明 |
|------|------|------|
| Phase 1 | S1 新需求生成 | 从 0 到 1 生成完整 PRD + 流程 + 原型 |
| Phase 1 | S7 局部修改 | 指令队列驱动的实时修改 |
| Phase 1 | S8 需求评审 | AI 自动评审需求质量 |
| Phase 2 | S2 原型驱动需求 | 已有 UI → 反推 PRD |
| Phase 2 | S6 需求迭代 | 版本变更差异管理 |
| Phase 3 | S3 HTML 反推需求 | 存量系统 HTML → PRD |
| Phase 3 | S4 流程图驱动需求 | BPMN/drawio → PRD |
| Phase 4 | S5 存量系统升级 | 代码+DB 分析 → 升级方案 |

## 快速开始

### 环境要求

- Docker & Docker Compose
- JDK 17+
- Node.js 18+

### 启动开发环境

```bash
# 启动基础设施
docker compose up -d

# 启动后端
cd backend && ./mvnw spring-boot:run

# 启动前端
cd frontend && npm install && npm run dev
```

## License

[Apache 2.0](LICENSE)
