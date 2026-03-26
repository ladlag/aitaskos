# API 设计规范

## 1. API 设计原则

| 原则 | 说明 |
|------|------|
| RESTful | 遵循 REST 风格，资源导向 |
| 版本化 | URL 路径包含版本号 `/api/v1/` |
| 统一响应 | 统一的响应格式和错误码 |
| 分页 | 列表接口支持分页 |
| 认证 | 所有接口需要 Bearer Token (Keycloak) |

## 2. 统一响应格式

### 2.1 成功响应

```json
{
  "code": 200,
  "message": "success",
  "data": {},
  "timestamp": "2025-01-01T10:00:00Z",
  "trace_id": "trace_xxx"
}
```

### 2.2 分页响应

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "items": [],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 100,
      "total_pages": 5
    }
  }
}
```

### 2.3 错误响应

```json
{
  "code": 400,
  "message": "参数错误",
  "errors": [
    { "field": "name", "message": "名称不能为空" }
  ],
  "timestamp": "2025-01-01T10:00:00Z",
  "trace_id": "trace_xxx"
}
```

## 3. 核心 API 列表

### 3.1 项目管理 API

```
POST   /api/v1/projects                    创建项目
GET    /api/v1/projects                    项目列表
GET    /api/v1/projects/{id}               项目详情
PUT    /api/v1/projects/{id}               更新项目
DELETE /api/v1/projects/{id}               删除项目

POST   /api/v1/projects/{id}/iterations    创建迭代
GET    /api/v1/projects/{id}/iterations    迭代列表
GET    /api/v1/iterations/{id}             迭代详情
PUT    /api/v1/iterations/{id}             更新迭代
```

### 3.2 DSL 管理 API

```
GET    /api/v1/projects/{projectId}/iterations/{iterationId}/dsl
                                           获取当前 DSL

PUT    /api/v1/dsl/{id}                    更新 DSL（全量）

PATCH  /api/v1/dsl/{id}                    局部更新 DSL
  Body: {
    "path": "pages[0].components[1].columns",
    "operation": "add",
    "value": { "field": "creator", "label": "创建人" }
  }

GET    /api/v1/dsl/{id}/history            DSL 变更历史
GET    /api/v1/dsl/{id}/diff?v1=1&v2=2     版本对比
POST   /api/v1/dsl/{id}/rollback           回滚到指定版本
  Body: { "version": 3 }
```

### 3.3 对话 API

```
POST   /api/v1/conversations               创建对话
GET    /api/v1/conversations                对话列表
GET    /api/v1/conversations/{id}/messages  消息列表

POST   /api/v1/conversations/{id}/messages  发送消息
  Body: {
    "content": "做一个订单管理系统",
    "type": "text",
    "attachments": ["file_id_1"]
  }

GET    /api/v1/conversations/{id}/stream    SSE 流式消息
  Response: SSE Event Stream
  Events:
    - event: stream     data: {"content": "正在分析..."}
    - event: question   data: {"id": "q1", "question": "..."}
    - event: status     data: {"step": "prd_generation", "progress": 60}
    - event: update     data: {"path": "features[0]", "value": {...}}
    - event: done       data: {"message": "完成"}
```

### 3.4 指令队列 API

```
POST   /api/v1/commands                    提交指令
  Body: {
    "project_id": "proj_001",
    "iteration_id": "iter_001",
    "type": "modify",
    "target": "pages[0].components[1]",
    "instruction": "增加一列创建人",
    "priority": "high"
  }

GET    /api/v1/commands?status=pending      查询指令队列
GET    /api/v1/commands/{id}               查询指令状态
DELETE /api/v1/commands/{id}               取消指令
PUT    /api/v1/commands/{id}/priority       调整优先级
```

### 3.5 异步反馈 API

```
GET    /api/v1/questions?status=pending     获取待回答问题
POST   /api/v1/questions/{id}/answer        回答问题
  Body: {
    "answer": "需要两级审批",
    "selected_option": null
  }
POST   /api/v1/questions/{id}/skip          跳过问题
```

### 3.6 数字员工管理 API

```
POST   /api/v1/employees                  创建数字员工
GET    /api/v1/employees                  数字员工列表
GET    /api/v1/employees/{id}             数字员工详情
PUT    /api/v1/employees/{id}             更新数字员工
DELETE /api/v1/employees/{id}             删除数字员工
PUT    /api/v1/employees/{id}/status      启用/禁用/暂停

-- Agent 绑定管理
POST   /api/v1/employees/{id}/agents      绑定 Agent
DELETE /api/v1/employees/{id}/agents/{agentId}  解绑 Agent
PUT    /api/v1/employees/{id}/agents/{agentId}  更新绑定配置

-- 工作流关联
POST   /api/v1/employees/{id}/workflows   关联工作流模板
DELETE /api/v1/employees/{id}/workflows/{code}  取消关联

-- 自动审查优化配置
PUT    /api/v1/employees/{id}/workflows/{code}/auto-review  配置自动审查
GET    /api/v1/employees/{id}/workflows/{code}/auto-review  获取自动审查配置
GET    /api/v1/tasks/{taskId}/review-iterations              查看审查迭代历史

-- 考核与统计
GET    /api/v1/employees/{id}/performance  绩效统计
GET    /api/v1/employees/{id}/tasks        任务历史
```

### 3.7 任务管理 API

```
POST   /api/v1/tasks                       创建任务
GET    /api/v1/tasks                       任务列表
GET    /api/v1/tasks/{id}                  任务详情
GET    /api/v1/tasks/{id}/executions       执行记录
PUT    /api/v1/tasks/{id}/cancel           取消任务
PUT    /api/v1/tasks/{id}/retry            重试任务

-- 任务条目管理
POST   /api/v1/tasks/{id}/items                    创建条目
GET    /api/v1/tasks/{id}/items                    条目列表（支持分类筛选）
GET    /api/v1/tasks/{id}/items/tree               条目树形结构
GET    /api/v1/items/{itemId}                      条目详情
PUT    /api/v1/items/{itemId}                      更新条目
PUT    /api/v1/items/{itemId}/status               变更条目状态
DELETE /api/v1/items/{itemId}                      删除条目

-- 条目关联
POST   /api/v1/items/{itemId}/relations            创建条目关联
GET    /api/v1/items/{itemId}/relations            查看条目关联
DELETE /api/v1/items/{itemId}/relations/{relationId} 删除条目关联
GET    /api/v1/items/{itemId}/trace                追溯链（上下游条目全链路）

-- 条目交付物
POST   /api/v1/items/{itemId}/deliverables         关联交付物
GET    /api/v1/items/{itemId}/deliverables         查看交付物列表
GET    /api/v1/deliverables/{deliverableId}        交付物详情
PUT    /api/v1/deliverables/{deliverableId}        更新交付物
DELETE /api/v1/deliverables/{deliverableId}        删除交付物

-- 条目审计
GET    /api/v1/items/{itemId}/audit-trail          条目审计轨迹
GET    /api/v1/tasks/{id}/items/summary            条目分类统计摘要
```

### 3.8 Agent 管理 API

```
POST   /api/v1/agents                     注册外部 Agent
GET    /api/v1/agents                     Agent 列表
GET    /api/v1/agents/{id}                Agent 详情
PUT    /api/v1/agents/{id}                更新 Agent 配置
PUT    /api/v1/agents/{id}/status         启用/禁用 Agent
GET    /api/v1/agents/{id}/health         Agent 健康检查

-- Agent 回调接口（外部 Agent → 平台）
POST   /api/v1/agent/callback             Agent 执行回调
  Body: {
    "request_id": "req_xxx",
    "task_id": "task_123",
    "status": "RUNNING",
    "progress": 60,
    "logs": "正在生成...",
    "partial_output": {},
    "questions": []
  }

-- Skill 管理（兼容 OpenClaw/AgentSkills 标准）
GET    /api/v1/agents/{id}/skills          Agent 的 Skill 列表
POST   /api/v1/agents/{id}/skills          注册 Skill（JSON 格式）
POST   /api/v1/agents/{id}/skills/import   导入 SKILL.md（OpenClaw 格式）
PUT    /api/v1/agents/{id}/skills/{skillId} 更新 Skill
DELETE /api/v1/agents/{id}/skills/{skillId} 删除 Skill
GET    /api/v1/skills                      全平台 Skill 搜索
  Query: ?tags=requirement&name=requirement-analysis

-- A2A 协议端点
GET    /.well-known/agent.json             平台 Agent Card（A2A 标准发现端点）
POST   /api/v1/a2a/tasks                  A2A 标准任务提交
GET    /api/v1/a2a/tasks/{id}             A2A 任务状态查询
GET    /api/v1/a2a/tasks/{id}/stream      A2A SSE 推送

-- MCP 协议端点（平台 MCP Server）
GET    /api/v1/mcp/capabilities            MCP 能力发现
POST   /api/v1/mcp/tools/{toolName}        MCP 工具调用
GET    /api/v1/mcp/resources               MCP 资源列表
GET    /api/v1/mcp/resources/{resourceId}   MCP 资源读取
```

### 3.9 知识库 API

```
POST   /api/v1/knowledge                  上传知识文档
GET    /api/v1/knowledge                  知识列表
GET    /api/v1/knowledge/{id}             知识详情
DELETE /api/v1/knowledge/{id}             删除知识

POST   /api/v1/knowledge/search           知识检索
  Body: {
    "query": "订单审批流程",
    "project_id": "proj_001",
    "iteration_id": "iter_001",
    "top_k": 5
  }
```

### 3.10 文件上传 API

```
POST   /api/v1/files/upload               上传文件
  Content-Type: multipart/form-data
  Field: file (支持 pdf/doc/docx/xls/xlsx/txt/jpg/png/html)

GET    /api/v1/files/{id}                 文件信息
GET    /api/v1/files/{id}/content         获取解析后的内容
GET    /api/v1/files/{id}/download        下载原文件
```

### 3.11 导出 API

```
POST   /api/v1/export/prd                 导出 PRD
  Body: {
    "dsl_id": "dsl_001",
    "format": "docx",           -- docx/pdf/markdown
    "template": "default"
  }

POST   /api/v1/export/flow                导出流程图
  Body: {
    "dsl_id": "dsl_001",
    "flow_id": "flow_001",
    "format": "bpmn"            -- bpmn/svg/png
  }
```

### 3.12 审计 API

```
GET    /api/v1/audit/logs                  审计日志
  Params:
    - action: 操作类型
    - resource_type: 资源类型
    - start_time: 开始时间
    - end_time: 结束时间
    - page, page_size

GET    /api/v1/audit/trace/{traceId}       链路追踪
```

## 4. SSE 事件规范

### 4.1 事件类型

| 事件类型 | 说明 | 数据格式 |
|---------|------|---------|
| `stream` | AI 流式文字输出 | `{ "content": "..." }` |
| `question` | AI 提问 | `{ "id": "q1", "question": "...", "options": [...] }` |
| `status` | 执行状态变更 | `{ "step": "...", "progress": 60, "status": "running" }` |
| `dsl_update` | DSL 更新 | `{ "path": "...", "operation": "add", "value": {...} }` |
| `command_update` | 指令状态变更 | `{ "id": "cmd1", "status": "done" }` |
| `error` | 错误通知 | `{ "code": "...", "message": "..." }` |
| `auto_review` | 自动审查状态 | `{ "capability": "...", "iteration": 2, "max_iterations": 3, "score": 65, "threshold": 80, "status": "iterating" }` |
| `done` | 流程完成 | `{ "message": "处理完成" }` |

### 4.2 SSE 连接

```
GET /api/v1/sse/connect?token={jwt_token}&project_id={id}

Response Headers:
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive

Event Format:
  event: stream
  data: {"content": "正在分析需求..."}

  event: question
  data: {"id": "q1", "question": "是否需要审批?", "options": ["是", "否"]}

  event: dsl_update
  data: {"path": "features[0]", "operation": "add", "value": {...}}
```

## 5. 错误码规范

| 错误码 | HTTP 状态码 | 说明 |
|--------|-----------|------|
| 200 | 200 | 成功 |
| 400 | 400 | 参数错误 |
| 401 | 401 | 未认证 |
| 403 | 403 | 无权限 |
| 404 | 404 | 资源不存在 |
| 409 | 409 | 资源冲突（如 DSL 版本冲突） |
| 429 | 429 | 请求过于频繁 |
| 500 | 500 | 服务器内部错误 |
| 503 | 503 | 服务不可用（Agent 不可用） |
| 10001 | 400 | DSL 格式错误 |
| 10002 | 400 | 指令目标路径不存在 |
| 10003 | 409 | DSL 版本冲突，请刷新后重试 |
| 10004 | 503 | 无可用 Agent |
| 10005 | 408 | Agent 执行超时 |
