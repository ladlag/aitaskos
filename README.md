# AiTaskOS — AI 数字员工平台

> 构建一个"多租户、可审计的 AI 数字员工管理与任务调度平台"。所有业务 Agent 均为外部独立服务，平台只负责管人、派活、收结果、打分。

## 项目简介

AiTaskOS 是一个 AI 数字员工平台（AI Digital Employee Platform），核心定位为 **数字员工管理 + 任务分配中枢**。平台负责数字员工的定义与管理、任务调度编排、执行数据收集、监控审计与绩效考核。

**核心边界**：平台不包含任何业务 Agent 实现。所有 Agent（需求分析、PRD 生成、开发、测试等）均为外部独立服务，可以是 Dify 创建的、第三方的、或自行开发的。

### 核心能力

- 👤 **数字员工管理**：数字员工定义、角色配置、Agent 绑定、工作流关联、绩效考核
- 📋 **任务管理**：任务创建、Task Graph、依赖管理、状态流转、优先级调度
- 🔧 **调度编排**：Task → 数字员工 → Agent 匹配，DAG 执行，并发控制
- 📊 **监控考核**：执行日志、操作审计、全链路追踪、绩效统计
- 💬 **交互能力**：实时交互、人机协同、局部修改、多轮迭代、知识沉淀

### 数字员工模型

```
数字员工 (Digital Employee)
  = 角色定义（Role）        ← 平台定义：BA、开发、测试、运维...
  + Agent 绑定（Agent[]）   ← 外部 Agent：Dify / 第三方 / 自建
  + Workflow（流程）         ← 工作流模板：任务执行的 DAG 编排
  + Knowledge（知识）       ← 关联的知识库
  + Memory（上下文）        ← 项目/迭代上下文
  + Queue（指令队列）       ← 接收的任务指令
```

### 平台与 Agent 的关系

```
Platform (AiTaskOS) = 数字员工管理 + 任务调度 + 执行监控 + 绩效考核
Agent (外部独立)    = 业务逻辑执行（需求分析/开发/测试/...）

Platform ──调度──▶ External Agent ──回调──▶ Platform ──SSE──▶ Frontend
```

## 核心技术栈

| 层级 | 技术 | 许可证 |
|------|------|--------|
| 前端 | Vue 3 + Element Plus + TipTap + AntV X6 | MIT |
| 后端 | Spring Boot 3.x + Spring WebFlux | Apache 2.0 |
| Agent 框架 | LangChain4j | Apache 2.0 |
| AI 工作流 | Dify | Apache 2.0 |
| 工作流 | Temporal | MIT |
| 模型网关 | LiteLLM | MIT |
| 数据库 | PostgreSQL | PostgreSQL License |
| 缓存 | Redis 7.2 / Valkey | BSD-3 |
| 向量库 | Milvus | Apache 2.0 |
| 消息队列 | Apache Kafka | Apache 2.0 |
| 文件存储 | MinIO | AGPL-3.0 |
| 认证 | Keycloak | Apache 2.0 |
| Agent 可观测 | Langfuse (自部署) | MIT |
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
│    数字员工管理层 (Spring Boot 3.x)                  │
│    数字员工 · 任务 · 调度 · 考核                      │
├────────────────────────────────────────────────────┤
│      调度编排层 (Temporal + 指令队列)                 │
├────────────────────────────────────────────────────┤
│      Agent 网关层 (A2A + MCP + 自定义协议)              │
├────────────────────────────────────────────────────┤
│   外部 Agent 执行层 (Dify / 自建 / 第三方)           │
│   全部解耦，通过标准协议接入                           │
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
| [05-Agent 系统设计](docs/05-agent-system-design.md) | 数字员工体系、外部 Agent 接入规范、Gateway、调度策略、考核统计 |
| [06-API 设计规范](docs/06-api-design.md) | RESTful API、SSE 事件、错误码规范 |
| [07-工作流与编排设计](docs/07-workflow-design.md) | Temporal 工作流、指令队列编排、事件驱动、并发控制 |
| [08-部署与基础设施设计](docs/08-deployment-design.md) | Docker Compose、K8s 部署、监控告警、备份恢复 |
| [09-场景实现指南](docs/09-scenario-implementation.md) | 8 大场景实现细节、开发路线图 |
| [10-技术趋势与优化建议](docs/10-technology-trends-and-optimization.md) | MCP/A2A 协议、Langfuse 可观测性、多 Agent 协作、评估管线 |

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
