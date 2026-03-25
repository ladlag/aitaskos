# 微服务架构与服务划分设计

> 基于系统复杂度分析，将 AiTaskOS 平台拆分为清晰的微服务边界，明确每个服务的职责、接口、数据归属和部署策略。

## 1. 微服务划分总览

### 1.1 服务拓扑

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          外部访问层                                      │
│                                                                         │
│    浏览器 ──── Nginx/CDN ──── API Gateway (APISIX)                      │
│                                    │                                    │
│              ┌─────────────────────┼─────────────────────┐              │
│              ▼                     ▼                     ▼              │
│    ┌──────────────────────────────────────────────────────────────┐     │
│    │                    业务微服务集群                              │     │
│    │                                                              │     │
│    │  ┌────────────┐  ┌────────────┐  ┌────────────┐             │     │
│    │  │  员工服务   │  │  任务服务   │  │  项目服务   │             │     │
│    │  │ employee   │  │   task     │  │  project   │             │     │
│    │  │ -service   │  │  -service  │  │  -service  │             │     │
│    │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘             │     │
│    │        │               │               │                    │     │
│    │  ┌────────────┐  ┌────────────┐  ┌────────────┐             │     │
│    │  │  调度服务   │  │  DSL 服务  │  │ 交互服务    │             │     │
│    │  │ scheduler  │  │   dsl     │  │ interaction │             │     │
│    │  │ -service   │  │  -service  │  │  -service  │             │     │
│    │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘             │     │
│    │        │               │               │                    │     │
│    │  ┌────────────┐  ┌────────────┐  ┌────────────┐             │     │
│    │  │Agent 网关   │  │ 知识服务    │  │ 审计考核    │             │     │
│    │  │  agent     │  │ knowledge  │  │  audit     │             │     │
│    │  │ -gateway   │  │  -service  │  │  -service  │             │     │
│    │  └─────┬──────┘  └────────────┘  └────────────┘             │     │
│    │        │                                                    │     │
│    │  ┌────────────┐  ┌────────────┐                             │     │
│    │  │ 文件服务    │  │ 通知服务    │                             │     │
│    │  │   file     │  │ notification│                            │     │
│    │  │  -service  │  │  -service  │                             │     │
│    │  └────────────┘  └────────────┘                             │     │
│    │                                                              │     │
│    └──────────────────────────────────────────────────────────────┘     │
│                                                                         │
│    ┌──────────────────────────────────────────────────────────────┐     │
│    │                    基础设施层                                  │     │
│    │                                                              │     │
│    │  PostgreSQL · Redis · Kafka · Milvus · MinIO · Temporal      │     │
│    │  Keycloak · LiteLLM · Langfuse · Jaeger · Prometheus         │     │
│    │                                                              │     │
│    └──────────────────────────────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 服务清单

| 序号 | 服务名称 | 服务代码 | 核心职责 | 实例数 |
|------|---------|---------|---------|--------|
| 1 | 数字员工服务 | `employee-service` | 数字员工 CRUD、角色管理、Agent 绑定、工作流关联、绩效统计 | 2 |
| 2 | 任务服务 | `task-service` | 任务 CRUD、Task Graph、依赖管理、状态流转、指令队列 | 3 |
| 3 | 项目服务 | `project-service` | 项目/迭代管理、租户管理、用户权限 | 2 |
| 4 | 调度编排服务 | `scheduler-service` | Task→Employee→Agent 匹配、DAG 执行、并发控制、Temporal Worker | 2 |
| 5 | DSL 服务 | `dsl-service` | DSL 读写、版本管理、合并验证、差异计算、同步通知 | 2 |
| 6 | Agent 网关 | `agent-gateway` | 协议适配(A2A/MCP/REST/Dify)、回调管理、流式转发、Skill 注册 | 2 |
| 7 | 知识服务 | `knowledge-service` | 知识库管理、向量化、检索、文档管理 | 2 |
| 8 | 交互服务 | `interaction-service` | 对话管理、SSE 推送、消息路由、WebSocket 管理 | 2 |
| 9 | 审计考核服务 | `audit-service` | 操作审计、执行日志、绩效考核、质量评估、Langfuse 集成 | 1 |
| 10 | 文件服务 | `file-service` | 文件上传/下载、文件解析(Tika/OCR)、制品管理 | 1 |
| 11 | 通知服务 | `notification-service` | 邮件/钉钉/企微通知、告警推送 | 1 |
| 12 | 前端 Web | `web-ui` | Vue3 SPA 静态资源托管 | 2 |

## 2. 各服务详细设计

### 2.1 数字员工服务（employee-service）

**职责边界**：
```
employee-service
├── 数字员工 CRUD（创建/查询/修改/删除/启停）
├── 角色模板管理（BA/开发/测试/运维/自定义）
├── Agent 绑定管理（绑定/解绑/优先级调整）
├── 工作流模板关联（关联/取消/配置）
├── 知识范围配置
├── 绩效数据聚合（从 audit-service 拉取）
└── 协作模式配置（chain/fanout/voting/delegation）
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `digital_employees` | ✅ 主表 |
| `employee_agent_bindings` | ✅ 主表 |
| `employee_workflow_templates` | ✅ 主表 |

**对外接口**：
```
REST API:
  POST   /api/v1/employees                 创建数字员工
  GET    /api/v1/employees                 查询列表
  GET    /api/v1/employees/{id}            查询详情
  PUT    /api/v1/employees/{id}            更新
  DELETE /api/v1/employees/{id}            删除
  PUT    /api/v1/employees/{id}/status     启用/禁用
  POST   /api/v1/employees/{id}/bindings   绑定 Agent
  DELETE /api/v1/employees/{id}/bindings/{bindingId}  解绑
  POST   /api/v1/employees/{id}/workflows  关联工作流
  GET    /api/v1/employees/{id}/performance  绩效统计

内部 gRPC:
  EmployeeService.GetById(id) → Employee
  EmployeeService.FindByCapability(capability) → Employee[]
  EmployeeService.GetBindings(employeeId) → AgentBinding[]
```

**依赖服务**：`audit-service`（读取绩效数据）

---

### 2.2 任务服务（task-service）

**职责边界**：
```
task-service
├── 任务 CRUD
├── Task Graph（DAG 任务依赖管理）
├── 任务状态流转
├── 指令队列管理
├── 任务优先级调度
├── 子任务拆分
└── 任务结果存储
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `tasks` | ✅ 主表 |
| `task_dependencies` | ✅ 主表 |
| `command_queue` | ✅ 主表 |

**对外接口**：
```
REST API:
  POST   /api/v1/tasks                    创建任务
  GET    /api/v1/tasks                    查询列表
  GET    /api/v1/tasks/{id}               查询详情
  PUT    /api/v1/tasks/{id}/status        更新状态
  POST   /api/v1/commands                 提交指令
  GET    /api/v1/commands                 查询指令队列
  DELETE /api/v1/commands/{id}            取消指令

Kafka 事件发布:
  topic: task.created       → 新任务创建
  topic: task.status.changed → 任务状态变更
  topic: command.submitted   → 新指令提交

内部 gRPC:
  TaskService.Create(task) → Task
  TaskService.GetById(id) → Task
  TaskService.UpdateStatus(id, status, result)
  TaskService.GetDAG(projectId) → TaskDAG
```

**依赖服务**：`project-service`（项目/迭代校验）、`dsl-service`（DSL 关联）

---

### 2.3 项目服务（project-service）

**职责边界**：
```
project-service
├── 项目 CRUD
├── 迭代管理
├── 租户管理
├── 用户管理（与 Keycloak 同步）
├── 权限控制（RBAC）
└── 项目成员管理
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `tenants` | ✅ 主表 |
| `projects` | ✅ 主表 |
| `iterations` | ✅ 主表 |
| `users` | ✅ 主表 |
| `project_members` | ✅ 主表 |

**对外接口**：
```
REST API:
  POST   /api/v1/projects                 创建项目
  GET    /api/v1/projects                 查询列表
  GET    /api/v1/projects/{id}            查询详情
  PUT    /api/v1/projects/{id}            更新
  POST   /api/v1/projects/{id}/iterations 创建迭代
  GET    /api/v1/projects/{id}/iterations 查询迭代列表

内部 gRPC:
  ProjectService.GetById(id) → Project
  ProjectService.ValidateAccess(userId, projectId) → bool
```

**依赖服务**：Keycloak（用户认证同步）

---

### 2.4 调度编排服务（scheduler-service）

**职责边界**：
```
scheduler-service
├── Task → 数字员工 → Agent 三级匹配
├── DAG 执行引擎（拓扑排序、依赖解析）
├── Temporal Workflow Worker（宿主）
├── 并发控制（分布式锁）
├── 超时/重试/降级策略
├── Agent 协作编排（chain/fanout/voting/delegation）
└── 指令队列消费与执行
```

**数据归属**：无独立主表，通过 gRPC 调用其他服务

**对外接口**：
```
Kafka 事件消费:
  topic: task.created       → 触发调度
  topic: command.submitted  → 触发指令执行
  topic: agent.callback     → 接收 Agent 回调结果

Kafka 事件发布:
  topic: task.status.changed → 更新任务状态
  topic: agent.dispatch      → Agent 调度事件
  topic: dsl.updated         → DSL 更新通知

内部 gRPC:
  SchedulerService.ScheduleTask(taskId)
  SchedulerService.ExecuteCommand(commandId)
  SchedulerService.CancelTask(taskId)
```

**依赖服务**：`employee-service`（匹配数字员工）、`agent-gateway`（调度 Agent）、`task-service`（任务状态）、`dsl-service`（DSL 操作）、Temporal Server

---

### 2.5 DSL 服务（dsl-service）

**职责边界**：
```
dsl-service
├── DSL 文档 CRUD
├── DSL 版本管理（快照、回滚）
├── DSL Patch 合并
├── DSL 一致性验证
├── DSL 差异计算（Diff）
├── DSL 导出（PRD Word/PDF、BPMN）
└── DSL 变更通知（事件发布）
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `dsl_documents` | ✅ 主表 |
| `dsl_versions` | ✅ 主表 |

**对外接口**：
```
REST API:
  GET    /api/v1/dsl/{projectId}/{iterationId}       获取当前 DSL
  PUT    /api/v1/dsl/{id}                            全量更新
  PATCH  /api/v1/dsl/{id}                            增量 Patch
  GET    /api/v1/dsl/{id}/history                    版本历史
  GET    /api/v1/dsl/{id}/diff                       版本对比
  POST   /api/v1/dsl/{id}/rollback                   回滚
  POST   /api/v1/dsl/{id}/export                     导出

MCP Server Tools（供外部 Agent 调用）:
  read_dsl       → Agent 读取 DSL
  patch_dsl      → Agent 提交修改
  validate_dsl   → Agent 验证一致性

Kafka 事件发布:
  topic: dsl.updated  → DSL 变更通知

内部 gRPC:
  DslService.GetCurrent(projectId, iterationId) → DSL
  DslService.ApplyPatch(id, patch) → DSL
  DslService.Validate(id) → ValidationResult
```

**依赖服务**：`file-service`（导出文件存储）

---

### 2.6 Agent 网关（agent-gateway）

**职责边界**：
```
agent-gateway
├── 协议适配层
│   ├── A2A Protocol 适配器（Agent Card 发现、Task Model 状态）
│   ├── MCP Server（平台工具暴露：DSL/知识库/任务上下文/文件）
│   ├── Dify API 适配器
│   ├── HTTP REST 适配器
│   ├── gRPC 适配器
│   └── WebSocket 适配器
├── Agent 注册与发现
│   ├── Agent CRUD（注册/配置/启停）
│   ├── Skill 注册（兼容 OpenClaw/AgentSkills 标准）
│   ├── 健康检查（心跳/主动探测）
│   └── 能力发现（A2A Agent Card / Skill 清单）
├── 回调管理器
│   ├── 异步结果接收
│   ├── 状态更新转发
│   └── 事件发布
├── 流式转发器
│   ├── Agent SSE → 平台 SSE
│   ├── 日志流聚合
│   └── 进度推送
├── 可观测集成
│   ├── Langfuse Trace 上报
│   └── OpenTelemetry 遥测
└── 安全管理器
    ├── Token 验证
    ├── 调用审计
    └── 限流控制
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `agents` | ✅ 主表 |
| `agent_executions` | ✅ 主表 |
| `agent_evaluation_records` | ✅ 主表 |

**对外接口**：
```
REST API:
  POST   /api/v1/agents                  注册 Agent
  GET    /api/v1/agents                  查询列表
  GET    /api/v1/agents/{id}             查询详情
  PUT    /api/v1/agents/{id}             更新配置
  PUT    /api/v1/agents/{id}/status      启用/禁用
  GET    /api/v1/agents/{id}/health      健康检查
  GET    /api/v1/agents/{id}/skills      查询 Skill 列表
  POST   /api/v1/agents/{id}/skills      注册 Skill

A2A Protocol 端点:
  GET    /.well-known/agent.json         Agent Card 发现
  POST   /api/v1/a2a/tasks              A2A Task 提交
  GET    /api/v1/a2a/tasks/{id}         A2A Task 查询
  GET    /api/v1/a2a/tasks/{id}/stream  A2A SSE 推送

MCP Server 端点:
  GET    /api/v1/mcp/tools              MCP 工具清单
  POST   /api/v1/mcp/tools/{name}       MCP 工具调用

Agent 回调端点（Agent → 平台）:
  POST   /api/v1/agent/callback         Agent 执行回调

Kafka 事件:
  消费: agent.dispatch     → 接收调度请求
  发布: agent.callback     → 转发 Agent 结果
  发布: agent.health       → 健康状态变更

内部 gRPC:
  AgentGateway.Dispatch(agentId, request) → ExecutionId
  AgentGateway.GetAgent(id) → Agent
  AgentGateway.FindByCapability(capability) → Agent[]
  AgentGateway.GetSkills(agentId) → Skill[]
```

**依赖服务**：`dsl-service`（MCP 工具调用）、`knowledge-service`（MCP 知识检索）、`file-service`（MCP 文件解析）、Langfuse（Trace 上报）

---

### 2.7 知识服务（knowledge-service）

**职责边界**：
```
knowledge-service
├── 知识库管理（CRUD）
├── 文档向量化（Embedding）
├── 向量检索（Milvus）
├── 三层知识体系（通用/项目/迭代）
├── 知识自动沉淀
└── MCP 工具：search_knowledge、get_document
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `knowledge_bases` | ✅ 主表 |
| `knowledge_documents` | ✅ 主表 |
| Milvus: `knowledge_vectors` | ✅ 主表 |

**对外接口**：
```
REST API:
  POST   /api/v1/knowledge               上传知识文档
  GET    /api/v1/knowledge               查询列表
  POST   /api/v1/knowledge/search        检索
  DELETE /api/v1/knowledge/{id}          删除

内部 gRPC:
  KnowledgeService.Search(query, scope) → SearchResult[]
  KnowledgeService.GetDocument(id) → Document
```

**依赖服务**：Milvus（向量存储）、`file-service`（文件解析）、LiteLLM（Embedding 模型）

---

### 2.8 交互服务（interaction-service）

**职责边界**：
```
interaction-service
├── 对话管理（CRUD）
├── 消息存储与路由
├── SSE 连接管理与推送
├── WebSocket 管理（可选）
├── 异步反馈（AI 提问/用户回答）
└── 实时状态广播
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `conversations` | ✅ 主表 |
| `messages` | ✅ 主表 |
| `clarification_questions` | ✅ 主表 |

**对外接口**：
```
REST API:
  POST   /api/v1/conversations           创建对话
  GET    /api/v1/conversations           查询列表
  POST   /api/v1/conversations/{id}/messages  发送消息
  GET    /api/v1/conversations/{id}/messages  消息历史

SSE 端点:
  GET    /api/v1/sse/connect             SSE 连接
  事件类型: stream/question/status/dsl_update/command_update/error/done

Kafka 事件消费:
  topic: task.status.changed  → 推送任务状态
  topic: dsl.updated         → 推送 DSL 变更
  topic: agent.callback      → 推送 Agent 结果
  topic: question.created    → 推送 AI 提问

内部 gRPC:
  InteractionService.PushEvent(userId, event)
  InteractionService.SendQuestion(question) → QuestionId
```

**依赖服务**：Redis（SSE 连接管理、Pub/Sub）

---

### 2.9 审计考核服务（audit-service）

**职责边界**：
```
audit-service
├── 操作审计日志
├── Agent 执行日志
├── 全链路追踪（OpenTelemetry → Jaeger）
├── Agent 可观测性（Langfuse 集成）
├── 绩效考核统计
├── 自动化质量评估管线
│   ├── 结构化验证
│   ├── LLM-as-Judge 语义评估
│   ├── 一致性检查
│   └── 用户反馈收集
└── 报表与数据导出
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `audit_logs` | ✅ 主表 |
| `agent_evaluation_records` | ✅ 共享（agent-gateway 写入、audit-service 分析） |

**对外接口**：
```
REST API:
  GET    /api/v1/audit/logs              查询审计日志
  GET    /api/v1/audit/trace/{traceId}   链路追踪
  GET    /api/v1/employees/{id}/performance  绩效数据
  GET    /api/v1/reports/quality          质量报告

Kafka 事件消费:
  topic: audit.event          → 审计事件记录
  topic: agent.callback       → Agent 执行结果 → 质量评估

内部 gRPC:
  AuditService.GetPerformance(employeeId) → PerformanceData
  AuditService.RecordAudit(event)
```

**依赖服务**：Langfuse（Agent 推理追踪）、Jaeger（链路追踪）

---

### 2.10 文件服务（file-service）

**职责边界**：
```
file-service
├── 文件上传（MinIO 存储）
├── 文件下载
├── 文件解析（Apache Tika）
│   ├── PDF/Word/Excel → 文本
│   ├── 图片 → OCR（PaddleOCR）
│   └── HTML → DOM 解析（Playwright）
├── 制品管理（Agent 输出制品存储）
└── MCP 工具：parse_file、download_artifact
```

**数据归属**：
| 表名 | 归属 |
|------|------|
| `file_metadata` | ✅ 主表 |
| MinIO 对象 | ✅ 主存储 |

**对外接口**：
```
REST API:
  POST   /api/v1/files/upload            上传文件
  GET    /api/v1/files/{id}              文件信息
  GET    /api/v1/files/{id}/content      解析后内容
  GET    /api/v1/files/{id}/download     下载

内部 gRPC:
  FileService.Parse(fileId) → ParsedContent
  FileService.Upload(bytes, metadata) → FileId
  FileService.Download(fileId) → bytes
```

**依赖服务**：MinIO（对象存储）、PaddleOCR（图片识别）

---

### 2.11 通知服务（notification-service）

**职责边界**：
```
notification-service
├── 邮件通知
├── 钉钉/企业微信通知
├── WebHook 通知
├── 系统内通知（写入 DB + SSE 推送）
└── 告警通知（来自监控系统）
```

**Kafka 事件消费**：
```
topic: notification.send  → 发送通知
```

---

## 3. 服务间通信设计

### 3.1 通信方式矩阵

| 场景 | 通信方式 | 说明 |
|------|---------|------|
| 同步查询 | gRPC | 服务间数据查询（低延迟、强类型） |
| 异步事件 | Kafka | 事件驱动（解耦、可靠投递） |
| 用户请求 | HTTP REST | 前端 → API Gateway → 各服务 |
| 实时推送 | SSE | 服务 → 前端（单向流式） |
| Agent 调用 | HTTP/A2A/MCP | agent-gateway → 外部 Agent |

### 3.2 Kafka Topic 完整清单

| Topic | 生产者 | 消费者 | 说明 |
|-------|--------|--------|------|
| `task.created` | task-service | scheduler-service | 新任务触发调度 |
| `task.status.changed` | scheduler-service | interaction-service, audit-service | 任务状态变更推送 |
| `command.submitted` | task-service | scheduler-service | 新指令触发执行 |
| `agent.dispatch` | scheduler-service | agent-gateway | Agent 调度请求 |
| `agent.callback` | agent-gateway | scheduler-service, interaction-service, audit-service | Agent 执行结果 |
| `agent.health` | agent-gateway | audit-service | Agent 健康状态 |
| `dsl.updated` | dsl-service | interaction-service, audit-service | DSL 变更通知 |
| `question.created` | agent-gateway | interaction-service | AI 提问推送 |
| `question.answered` | interaction-service | scheduler-service | 用户回答转发 |
| `knowledge.updated` | knowledge-service | — | 知识库更新 |
| `notification.send` | 各服务 | notification-service | 通知发送 |
| `audit.event` | 各服务 | audit-service | 审计事件 |

### 3.3 服务依赖关系图

```
project-service ◄──── task-service ◄──── scheduler-service
                                              │
employee-service ◄────────────────────────────┤
                                              │
agent-gateway ◄───────────────────────────────┘
      │
      ├──── dsl-service
      ├──── knowledge-service
      └──── file-service

interaction-service ◄──── Kafka 事件 ◄──── 各服务

audit-service ◄──── Kafka 事件 ◄──── 各服务
```

### 3.4 gRPC Proto 定义规范

```protobuf
// 统一的 proto 管理仓库: aitaskos-proto
syntax = "proto3";

// 员工服务
service EmployeeService {
  rpc GetById(GetByIdRequest) returns (Employee);
  rpc FindByCapability(FindByCapabilityRequest) returns (EmployeeList);
  rpc GetBindings(GetBindingsRequest) returns (AgentBindingList);
}

// 任务服务
service TaskService {
  rpc Create(CreateTaskRequest) returns (Task);
  rpc GetById(GetByIdRequest) returns (Task);
  rpc UpdateStatus(UpdateStatusRequest) returns (Task);
  rpc GetDAG(GetDAGRequest) returns (TaskDAG);
}

// DSL 服务
service DslService {
  rpc GetCurrent(GetCurrentDslRequest) returns (DslDocument);
  rpc ApplyPatch(ApplyPatchRequest) returns (DslDocument);
  rpc Validate(ValidateRequest) returns (ValidationResult);
}

// Agent 网关
service AgentGateway {
  rpc Dispatch(DispatchRequest) returns (DispatchResponse);
  rpc GetAgent(GetByIdRequest) returns (Agent);
  rpc FindByCapability(FindByCapabilityRequest) returns (AgentList);
  rpc GetSkills(GetSkillsRequest) returns (SkillList);
}
```

## 4. 数据库拆分策略

### 4.1 数据库归属

采用 **逻辑隔离** 策略：所有服务共享同一个 PostgreSQL 实例，但使用不同的 Schema 进行逻辑隔离。

```
PostgreSQL Instance
├── schema: project    → project-service 独占
│   ├── tenants
│   ├── projects
│   ├── iterations
│   ├── users
│   └── project_members
├── schema: employee   → employee-service 独占
│   ├── digital_employees
│   ├── employee_agent_bindings
│   └── employee_workflow_templates
├── schema: task       → task-service 独占
│   ├── tasks
│   ├── task_dependencies
│   └── command_queue
├── schema: dsl        → dsl-service 独占
│   ├── dsl_documents
│   └── dsl_versions
├── schema: agent      → agent-gateway 独占
│   ├── agents
│   ├── agent_executions
│   └── agent_evaluation_records
├── schema: knowledge  → knowledge-service 独占
│   ├── knowledge_bases
│   └── knowledge_documents
├── schema: interaction → interaction-service 独占
│   ├── conversations
│   ├── messages
│   └── clarification_questions
├── schema: audit      → audit-service 独占
│   ├── audit_logs
│   └── (agent_evaluation_records 跨 schema 视图)
└── schema: file       → file-service 独占
    └── file_metadata
```

> **演进策略**：初期共享数据库实例 + Schema 隔离；当单个服务负载增大时，可独立拆库而不影响其他服务（只需修改该服务的数据源配置）。

### 4.2 跨服务数据访问规则

```
规则:
├── 每个服务只能直接读写自己 Schema 下的表
├── 跨服务数据通过 gRPC 接口获取（禁止跨 Schema 直接查询）
├── 共享常量数据（如枚举值）通过 Proto 定义统一管理
└── 异步事件数据（Kafka）可携带必要的冗余字段避免反查
```

## 5. API Gateway 路由配置

### 5.1 APISIX 路由表

```yaml
# APISIX 路由配置
routes:
  # 项目管理
  - uri: /api/v1/projects*
    upstream: project-service:8080
  
  # 迭代管理
  - uri: /api/v1/iterations*
    upstream: project-service:8080

  # 数字员工管理
  - uri: /api/v1/employees*
    upstream: employee-service:8080

  # 任务管理
  - uri: /api/v1/tasks*
    upstream: task-service:8080
  
  # 指令队列
  - uri: /api/v1/commands*
    upstream: task-service:8080

  # DSL 管理
  - uri: /api/v1/dsl*
    upstream: dsl-service:8080

  # Agent 管理
  - uri: /api/v1/agents*
    upstream: agent-gateway:8080
  
  # A2A 协议
  - uri: /.well-known/agent.json
    upstream: agent-gateway:8080
  - uri: /api/v1/a2a*
    upstream: agent-gateway:8080
  
  # MCP 协议
  - uri: /api/v1/mcp*
    upstream: agent-gateway:8080
  
  # Agent 回调
  - uri: /api/v1/agent/callback
    upstream: agent-gateway:8080

  # 知识库
  - uri: /api/v1/knowledge*
    upstream: knowledge-service:8080

  # 对话与 SSE
  - uri: /api/v1/conversations*
    upstream: interaction-service:8080
  - uri: /api/v1/sse*
    upstream: interaction-service:8080

  # 文件管理
  - uri: /api/v1/files*
    upstream: file-service:8080

  # 审计与考核
  - uri: /api/v1/audit*
    upstream: audit-service:8080
  - uri: /api/v1/reports*
    upstream: audit-service:8080

  # 前端静态资源
  - uri: /*
    upstream: web-ui:80
```

## 6. 部署与扩缩容策略

### 6.1 资源规格与扩缩容

| 服务 | CPU (request/limit) | 内存 (request/limit) | 最小实例 | 最大实例 | HPA 指标 |
|------|---------------------|---------------------|---------|---------|----------|
| employee-service | 0.5C / 1C | 512M / 1G | 2 | 4 | CPU > 70% |
| task-service | 1C / 2C | 1G / 2G | 2 | 8 | CPU > 70% |
| project-service | 0.5C / 1C | 512M / 1G | 2 | 4 | CPU > 70% |
| scheduler-service | 1C / 2C | 1G / 2G | 2 | 6 | CPU > 70% |
| dsl-service | 0.5C / 1C | 512M / 1G | 2 | 4 | CPU > 70% |
| agent-gateway | 1C / 2C | 1G / 2G | 2 | 8 | 并发连接数 > 1000 |
| knowledge-service | 0.5C / 1C | 512M / 1G | 2 | 4 | CPU > 70% |
| interaction-service | 1C / 2C | 1G / 2G | 2 | 6 | SSE 连接数 > 5000 |
| audit-service | 0.5C / 1C | 512M / 1G | 1 | 3 | CPU > 70% |
| file-service | 1C / 2C | 1G / 2G | 1 | 4 | CPU > 70% |
| notification-service | 0.25C / 0.5C | 256M / 512M | 1 | 2 | 队列积压 > 1000 |
| web-ui | 0.1C / 0.2C | 128M / 256M | 2 | 4 | QPS > 1000 |

### 6.2 服务优先级与启动顺序

```
启动顺序（按依赖关系）:

阶段 1: 基础设施（数据存储与消息）
  PostgreSQL → Redis → Kafka → Milvus → MinIO

阶段 2: 基础设施（编排与认证）
  Temporal Server（依赖 PostgreSQL）→ Keycloak

阶段 3: 监控与可观测
  Langfuse → Jaeger → Prometheus → Grafana

阶段 3: 核心服务
  project-service → employee-service → task-service → dsl-service

阶段 4: 网关与调度
  agent-gateway → scheduler-service → interaction-service

阶段 5: 辅助服务
  knowledge-service → file-service → audit-service → notification-service

阶段 6: 接入层
  API Gateway (APISIX) → web-ui
```

## 7. 统一技术规范

### 7.1 服务代码结构（Spring Boot 标准）

```
aitaskos-{service-name}/
├── src/main/java/com/aitaskos/{service}/
│   ├── AitaskosApplication.java              # 启动类
│   ├── config/                               # 配置类
│   │   ├── KafkaConfig.java
│   │   ├── RedisConfig.java
│   │   └── GrpcServerConfig.java
│   ├── controller/                           # REST Controller
│   │   └── XxxController.java
│   ├── grpc/                                 # gRPC Server
│   │   └── XxxGrpcService.java
│   ├── service/                              # 业务逻辑
│   │   ├── XxxService.java
│   │   └── impl/XxxServiceImpl.java
│   ├── repository/                           # 数据访问
│   │   └── XxxRepository.java
│   ├── domain/                               # 领域模型
│   │   ├── entity/
│   │   ├── vo/
│   │   └── event/
│   ├── client/                               # gRPC Client Stub
│   │   └── XxxServiceClient.java
│   └── kafka/                                # Kafka 消费者/生产者
│       ├── XxxEventConsumer.java
│       └── XxxEventProducer.java
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   └── db/migration/                         # Flyway 迁移脚本
├── src/test/
├── Dockerfile
├── pom.xml
└── README.md
```

### 7.2 Maven 多模块结构

```
aitaskos/
├── pom.xml                                  # 父 POM
├── aitaskos-common/                         # 公共模块（DTO、工具类、常量）
├── aitaskos-proto/                          # gRPC Proto 定义
├── aitaskos-employee-service/               # 数字员工服务
├── aitaskos-task-service/                   # 任务服务
├── aitaskos-project-service/                # 项目服务
├── aitaskos-scheduler-service/              # 调度编排服务
├── aitaskos-dsl-service/                    # DSL 服务
├── aitaskos-agent-gateway/                  # Agent 网关
├── aitaskos-knowledge-service/              # 知识服务
├── aitaskos-interaction-service/            # 交互服务
├── aitaskos-audit-service/                  # 审计考核服务
├── aitaskos-file-service/                   # 文件服务
├── aitaskos-notification-service/           # 通知服务
└── aitaskos-web-ui/                         # 前端
```

### 7.3 统一配置规范

```yaml
# 每个服务的 application.yml 规范
server:
  port: 8080

spring:
  application:
    name: aitaskos-{service-name}
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/aitaskos
    schema: {service-schema}
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
  redis:
    host: ${REDIS_HOST:localhost}

grpc:
  server:
    port: 9090

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  tracing:
    sampling:
      probability: 1.0
```
