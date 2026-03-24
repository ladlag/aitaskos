# 数据模型设计

## 1. 核心 DSL 结构

### 1.1 DSL 总体结构

```json
{
  "version": "1.0.0",
  "project": {
    "id": "proj_xxx",
    "name": "订单管理系统",
    "description": "...",
    "iteration_id": "iter_xxx"
  },
  "features": [],
  "pages": [],
  "flows": [],
  "entities": [],
  "rules": [],
  "metadata": {
    "created_at": "2025-01-01T00:00:00Z",
    "updated_at": "2025-01-01T00:00:00Z",
    "created_by": "user_xxx",
    "dsl_hash": "sha256_xxx"
  }
}
```

### 1.2 功能模块（features）

```json
{
  "features": [
    {
      "id": "feat_001",
      "name": "订单管理",
      "description": "订单全生命周期管理",
      "priority": "P0",
      "status": "draft",
      "children": [
        {
          "id": "feat_001_01",
          "name": "订单创建",
          "description": "支持手动和自动创建订单",
          "acceptance_criteria": [
            "用户可以填写订单表单并提交",
            "系统自动生成订单编号"
          ],
          "related_pages": ["page_order_create"],
          "related_flows": ["flow_order_create"],
          "related_entities": ["entity_order"]
        }
      ]
    }
  ]
}
```

### 1.3 页面结构（pages）

```json
{
  "pages": [
    {
      "id": "page_order_list",
      "name": "订单列表",
      "path": "/orders",
      "layout": "full-width",
      "components": [
        {
          "id": "comp_001",
          "type": "filter",
          "label": "筛选区",
          "fields": [
            {
              "id": "field_001",
              "name": "order_no",
              "label": "订单编号",
              "type": "input",
              "rules": { "maxLength": 32 }
            },
            {
              "id": "field_002",
              "name": "status",
              "label": "状态",
              "type": "select",
              "options": [
                { "value": "pending", "label": "待处理" },
                { "value": "approved", "label": "已审批" }
              ]
            },
            {
              "id": "field_003",
              "name": "date_range",
              "label": "日期范围",
              "type": "date-range"
            }
          ]
        },
        {
          "id": "comp_002",
          "type": "table",
          "label": "数据列表",
          "columns": [
            { "field": "order_no", "label": "订单编号", "sortable": true },
            { "field": "customer", "label": "客户名称" },
            { "field": "amount", "label": "金额", "format": "currency" },
            { "field": "status", "label": "状态", "type": "tag" },
            { "field": "created_at", "label": "创建时间", "format": "datetime" }
          ],
          "actions": [
            { "label": "查看", "type": "link", "target": "page_order_detail" },
            { "label": "编辑", "type": "button", "permission": "order:edit" },
            { "label": "删除", "type": "button", "permission": "order:delete", "confirm": true }
          ],
          "pagination": true
        },
        {
          "id": "comp_003",
          "type": "toolbar",
          "actions": [
            { "label": "新建订单", "type": "primary", "target": "page_order_create" },
            { "label": "导出", "type": "default" },
            { "label": "批量操作", "type": "dropdown" }
          ]
        }
      ],
      "permissions": ["order:view"]
    }
  ]
}
```

### 1.4 流程结构（flows）

```json
{
  "flows": [
    {
      "id": "flow_order_approve",
      "name": "订单审批流程",
      "type": "approval",
      "trigger": "order.submit",
      "nodes": [
        {
          "id": "node_001",
          "type": "start",
          "name": "提交订单",
          "next": "node_002"
        },
        {
          "id": "node_002",
          "type": "task",
          "name": "部门审批",
          "assignee": "role:dept_manager",
          "next": "node_003"
        },
        {
          "id": "node_003",
          "type": "gateway",
          "name": "审批结果",
          "conditions": [
            { "expression": "approved == true", "next": "node_004" },
            { "expression": "approved == false", "next": "node_005" }
          ]
        },
        {
          "id": "node_004",
          "type": "task",
          "name": "执行订单",
          "next": "node_006"
        },
        {
          "id": "node_005",
          "type": "task",
          "name": "退回修改",
          "next": "node_001"
        },
        {
          "id": "node_006",
          "type": "end",
          "name": "流程结束"
        }
      ]
    }
  ]
}
```

### 1.5 数据实体（entities）

```json
{
  "entities": [
    {
      "id": "entity_order",
      "name": "Order",
      "label": "订单",
      "fields": [
        {
          "name": "id",
          "type": "bigint",
          "primary_key": true,
          "auto_increment": true
        },
        {
          "name": "order_no",
          "type": "varchar(32)",
          "unique": true,
          "nullable": false,
          "comment": "订单编号"
        },
        {
          "name": "customer_id",
          "type": "bigint",
          "nullable": false,
          "reference": "entity_customer.id",
          "comment": "客户ID"
        },
        {
          "name": "amount",
          "type": "decimal(12,2)",
          "nullable": false,
          "comment": "订单金额"
        },
        {
          "name": "status",
          "type": "varchar(20)",
          "nullable": false,
          "enum": ["pending", "approved", "rejected", "completed"],
          "comment": "订单状态"
        },
        {
          "name": "created_at",
          "type": "timestamp",
          "default": "CURRENT_TIMESTAMP"
        },
        {
          "name": "updated_at",
          "type": "timestamp",
          "default": "CURRENT_TIMESTAMP"
        }
      ],
      "indexes": [
        { "name": "idx_order_no", "fields": ["order_no"], "unique": true },
        { "name": "idx_customer", "fields": ["customer_id"] },
        { "name": "idx_status", "fields": ["status"] }
      ],
      "relations": [
        {
          "type": "many-to-one",
          "target": "entity_customer",
          "field": "customer_id"
        },
        {
          "type": "one-to-many",
          "target": "entity_order_item",
          "mapped_by": "order_id"
        }
      ]
    }
  ]
}
```

### 1.6 业务规则（rules）

```json
{
  "rules": [
    {
      "id": "rule_001",
      "name": "订单金额上限",
      "type": "validation",
      "target": "entity_order",
      "condition": "amount > 100000",
      "action": "require_approval",
      "message": "订单金额超过10万需要总经理审批"
    },
    {
      "id": "rule_002",
      "name": "订单编号规则",
      "type": "generation",
      "target": "entity_order.order_no",
      "pattern": "ORD-{YYYYMMDD}-{SEQ:6}",
      "description": "订单编号格式：ORD-日期-6位序号"
    }
  ]
}
```

## 2. 数据库模型（PostgreSQL）

### 2.1 核心业务表

```sql
-- ============================================================
-- 用户与租户
-- ============================================================

CREATE TABLE tenants (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(50) UNIQUE NOT NULL,
    status          VARCHAR(20) DEFAULT 'active',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT REFERENCES tenants(id),
    username        VARCHAR(50) UNIQUE NOT NULL,
    email           VARCHAR(100),
    display_name    VARCHAR(100),
    avatar_url      VARCHAR(500),
    external_id     VARCHAR(200),           -- Keycloak 用户 ID
    status          VARCHAR(20) DEFAULT 'active',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- 项目与迭代
-- ============================================================

CREATE TABLE projects (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT REFERENCES tenants(id),
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) DEFAULT 'active',
    created_by      BIGINT REFERENCES users(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE iterations (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT REFERENCES projects(id),
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    version         VARCHAR(50),
    status          VARCHAR(20) DEFAULT 'planning',   -- planning/active/completed
    started_at      TIMESTAMP,
    ended_at        TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- DSL 核心数据
-- ============================================================

CREATE TABLE dsl_documents (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT REFERENCES projects(id),
    iteration_id    BIGINT REFERENCES iterations(id),
    version         INTEGER DEFAULT 1,
    content         JSONB NOT NULL,                    -- 完整 DSL JSON
    dsl_hash        VARCHAR(64),                       -- 内容哈希
    status          VARCHAR(20) DEFAULT 'draft',       -- draft/reviewing/approved
    created_by      BIGINT REFERENCES users(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_dsl_project ON dsl_documents(project_id);
CREATE INDEX idx_dsl_iteration ON dsl_documents(iteration_id);
CREATE INDEX idx_dsl_content ON dsl_documents USING gin(content);   -- JSONB 索引

-- DSL 变更历史
CREATE TABLE dsl_change_history (
    id              BIGSERIAL PRIMARY KEY,
    dsl_document_id BIGINT REFERENCES dsl_documents(id),
    version         INTEGER NOT NULL,
    change_type     VARCHAR(20) NOT NULL,              -- add/modify/delete
    change_path     VARCHAR(500),                      -- DSL JSON 路径
    old_value       JSONB,
    new_value       JSONB,
    changed_by      VARCHAR(50),                       -- user/agent
    agent_id        VARCHAR(100),
    command_id      VARCHAR(100),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- 任务管理
-- ============================================================

CREATE TABLE tasks (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT REFERENCES projects(id),
    iteration_id    BIGINT REFERENCES iterations(id),
    employee_id     BIGINT REFERENCES digital_employees(id),  -- 分配给的数字员工
    parent_task_id  BIGINT REFERENCES tasks(id),       -- 父任务（Task Graph）
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    task_type       VARCHAR(50) NOT NULL,               -- requirement_analysis/prd_generation/flow_design/...
    status          VARCHAR(20) DEFAULT 'pending',      -- pending/running/completed/failed/cancelled
    priority        VARCHAR(10) DEFAULT 'normal',       -- high/normal/low
    execution_mode  VARCHAR(10) DEFAULT 'ASYNC',        -- SYNC/ASYNC
    employee_selector JSONB,                            -- 数字员工选择策略 {type: AUTO/MANUAL, employee_id}
    input_data      JSONB,                              -- 任务输入
    output_data     JSONB,                              -- 任务输出
    quality_score   INTEGER CHECK (quality_score >= 0 AND quality_score <= 100),  -- 用户质量评分 0-100
    created_by      BIGINT REFERENCES users(id),
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_task_project ON tasks(project_id);
CREATE INDEX idx_task_status ON tasks(status);
CREATE INDEX idx_task_type ON tasks(task_type);

-- 任务依赖关系
CREATE TABLE task_dependencies (
    id              BIGSERIAL PRIMARY KEY,
    task_id         BIGINT REFERENCES tasks(id),
    depends_on      BIGINT REFERENCES tasks(id),
    dependency_type VARCHAR(20) DEFAULT 'finish_to_start'  -- finish_to_start/start_to_start
);

-- ============================================================
-- 指令队列
-- ============================================================

CREATE TABLE command_queue (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT REFERENCES projects(id),
    iteration_id    BIGINT REFERENCES iterations(id),
    dsl_document_id BIGINT REFERENCES dsl_documents(id),
    command_type    VARCHAR(20) NOT NULL,               -- modify/add/delete/recompute/clarify
    target_path     VARCHAR(500),                       -- DSL 路径
    instruction     TEXT NOT NULL,                      -- 用户指令
    priority        VARCHAR(10) DEFAULT 'normal',       -- high/normal/low
    status          VARCHAR(20) DEFAULT 'pending',      -- pending/running/done/failed/cancelled
    result          JSONB,                              -- 执行结果
    error_message   TEXT,
    created_by      BIGINT REFERENCES users(id),
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_cmd_status ON command_queue(status);
CREATE INDEX idx_cmd_priority ON command_queue(priority);

-- ============================================================
-- 数字员工管理
-- ============================================================

CREATE TABLE digital_employees (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT REFERENCES tenants(id),
    name            VARCHAR(200) NOT NULL,
    role            VARCHAR(50) NOT NULL,               -- business_analyst/developer/tester/devops/custom
    description     TEXT,
    avatar_url      VARCHAR(500),
    status          VARCHAR(20) DEFAULT 'active',       -- active/inactive/suspended
    knowledge_scope JSONB,                              -- 关联的知识库范围
    config          JSONB,                              -- 工作参数（超时、重试等）
    performance     JSONB,                              -- 绩效统计快照
    created_by      BIGINT REFERENCES users(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_emp_tenant ON digital_employees(tenant_id);
CREATE INDEX idx_emp_role ON digital_employees(role);
CREATE INDEX idx_emp_status ON digital_employees(status);

-- 数字员工 ↔ Agent 绑定关系
CREATE TABLE employee_agent_bindings (
    id              BIGSERIAL PRIMARY KEY,
    employee_id     BIGINT REFERENCES digital_employees(id),
    agent_id        BIGINT REFERENCES agents(id),
    capability      VARCHAR(100) NOT NULL,              -- 该绑定覆盖的能力
    priority        INTEGER DEFAULT 1,                   -- 优先级（数值越小优先级越高，1为最高）
    status          VARCHAR(20) DEFAULT 'active',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_binding_emp ON employee_agent_bindings(employee_id);
CREATE INDEX idx_binding_agent ON employee_agent_bindings(agent_id);

-- 数字员工 ↔ 工作流模板关联
CREATE TABLE employee_workflow_templates (
    id              BIGSERIAL PRIMARY KEY,
    employee_id     BIGINT REFERENCES digital_employees(id),
    workflow_code   VARCHAR(100) NOT NULL,              -- 工作流模板代码
    config          JSONB,                              -- 工作流参数覆盖
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- Agent 管理（全部为外部 Agent）
-- ============================================================

CREATE TABLE agents (
    id              BIGSERIAL PRIMARY KEY,
    agent_code      VARCHAR(100) UNIQUE NOT NULL,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    provider        VARCHAR(50) NOT NULL,               -- dify/self_built/third_party
    endpoint_url    VARCHAR(500) NOT NULL,              -- Agent 服务地址
    capabilities    JSONB NOT NULL,                     -- 能力声明
    input_schema    JSONB,                              -- 输入格式声明
    output_schema   JSONB,                              -- 输出格式声明
    config          JSONB,                              -- 配置参数
    status          VARCHAR(20) DEFAULT 'active',       -- active/inactive/error
    health_status   VARCHAR(20) DEFAULT 'unknown',      -- healthy/unhealthy/unknown
    last_health_check TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Agent 执行记录
CREATE TABLE agent_executions (
    id              BIGSERIAL PRIMARY KEY,
    task_id         BIGINT REFERENCES tasks(id),
    agent_id        BIGINT REFERENCES agents(id),
    request_payload JSONB,
    response_payload JSONB,
    status          VARCHAR(20) DEFAULT 'running',      -- running/success/failed/timeout
    progress        INTEGER DEFAULT 0,                   -- 0-100
    logs            TEXT,
    artifacts       JSONB,                               -- 输出制品
    error_message   TEXT,
    started_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    finished_at     TIMESTAMP,
    duration_ms     BIGINT
);

CREATE INDEX idx_exec_task ON agent_executions(task_id);
CREATE INDEX idx_exec_agent ON agent_executions(agent_id);
CREATE INDEX idx_exec_status ON agent_executions(status);

-- ============================================================
-- 异步反馈（AI 提问）
-- ============================================================

CREATE TABLE clarification_questions (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT REFERENCES projects(id),
    iteration_id    BIGINT REFERENCES iterations(id),
    task_id         BIGINT REFERENCES tasks(id),
    agent_id        BIGINT REFERENCES agents(id),
    question        TEXT NOT NULL,
    context         TEXT,                                -- 问题上下文
    options         JSONB,                               -- 选项
    answer          TEXT,
    answered_by     BIGINT REFERENCES users(id),
    status          VARCHAR(20) DEFAULT 'pending',       -- pending/answered/skipped/expired
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    answered_at     TIMESTAMP
);

CREATE INDEX idx_question_status ON clarification_questions(status);

-- ============================================================
-- 对话历史
-- ============================================================

CREATE TABLE conversations (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT REFERENCES projects(id),
    iteration_id    BIGINT REFERENCES iterations(id),
    user_id         BIGINT REFERENCES users(id),
    title           VARCHAR(200),
    status          VARCHAR(20) DEFAULT 'active',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE messages (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT REFERENCES conversations(id),
    role            VARCHAR(20) NOT NULL,                -- user/assistant/system
    content         TEXT NOT NULL,
    message_type    VARCHAR(20) DEFAULT 'text',          -- text/file/question/command
    metadata        JSONB,                               -- 关联信息（DSL路径、命令ID等）
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_msg_conversation ON messages(conversation_id);

-- ============================================================
-- 知识库
-- ============================================================

CREATE TABLE knowledge_documents (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT REFERENCES projects(id),     -- NULL = 通用知识
    iteration_id    BIGINT REFERENCES iterations(id),    -- NULL = 项目级知识
    title           VARCHAR(200) NOT NULL,
    content         TEXT,
    doc_type        VARCHAR(50),                         -- template/standard/requirement/meeting_notes
    knowledge_level VARCHAR(20) NOT NULL,                -- global/project/iteration
    file_path       VARCHAR(500),                        -- MinIO 文件路径
    vector_id       VARCHAR(200),                        -- Milvus 向量 ID
    status          VARCHAR(20) DEFAULT 'active',
    created_by      BIGINT REFERENCES users(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_knowledge_level ON knowledge_documents(knowledge_level);
CREATE INDEX idx_knowledge_project ON knowledge_documents(project_id);

-- ============================================================
-- 审计日志
-- ============================================================

CREATE TABLE audit_logs (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       BIGINT REFERENCES tenants(id),
    user_id         BIGINT,
    agent_id        BIGINT,
    action          VARCHAR(100) NOT NULL,               -- 操作类型
    resource_type   VARCHAR(50),                         -- 资源类型
    resource_id     VARCHAR(100),                        -- 资源ID
    detail          JSONB,                               -- 详细信息
    ip_address      VARCHAR(50),
    trace_id        VARCHAR(100),                        -- 链路追踪ID
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_action ON audit_logs(action);
CREATE INDEX idx_audit_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_time ON audit_logs(created_at);

-- ============================================================
-- 文件上传
-- ============================================================

CREATE TABLE uploaded_files (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT REFERENCES projects(id),
    original_name   VARCHAR(500) NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,               -- MinIO 路径
    file_type       VARCHAR(50),                         -- pdf/docx/xlsx/png/html
    file_size       BIGINT,
    mime_type       VARCHAR(100),
    parsed_content  TEXT,                                 -- 解析后的文本内容
    status          VARCHAR(20) DEFAULT 'uploaded',       -- uploaded/parsing/parsed/error
    uploaded_by     BIGINT REFERENCES users(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2.2 实体关系图

```
┌──────────┐     ┌──────────┐     ┌──────────────┐
│ tenants  │────▶│  users   │────▶│  projects    │
└──────────┘     └──────────┘     └──────┬───────┘
                      │                  │
                      ▼                  │
              ┌──────────────┐           │
              │digital_      │           │
              │employees     │           │
              └──────┬───────┘           │
                     │                   │
         ┌───────────┤         ┌─────────┼──────────────┐
         ▼           ▼         ▼         ▼              ▼
  ┌────────────┐ ┌────────┐ ┌──────┐ ┌──────────┐ ┌──────────┐
  │emp_agent_  │ │emp_wf_ │ │tasks │ │iterations│ │uploaded_ │
  │bindings    │ │templates│ │      │ │          │ │files     │
  └──────┬─────┘ └────────┘ └──┬───┘ └────┬─────┘ └──────────┘
         │                     │          │
         ▼                     │    ┌─────┼──────────┐
  ┌──────────┐                 │    ▼     ▼          ▼
  │  agents  │                 │ ┌──────┐ ┌──────┐ ┌──────────┐
  │(外部全部) │                 │ │dsl_  │ │conv_ │ │knowledge │
  └──────────┘                 │ │docs  │ │ersations│ │_docs   │
         ▲                     │ └──┬───┘ └──┬───┘ └──────────┘
         │                     │    │        │
         │          ┌──────────┘    ▼        ▼
         │          ▼          ┌────────┐ ┌──────┐
         │   ┌──────────────┐  │dsl_    │ │msgs  │
         └───│agent_exec    │  │changes │ └──────┘
             └──────────────┘  └────────┘

  ┌────────────────┐  ┌──────────────┐  ┌──────────────┐
  │clarif_questions│  │command_queue │  │ audit_logs   │
  └────────────────┘  └──────────────┘  └──────────────┘
```

## 3. 指令队列数据结构

### 3.1 指令消息格式（Redis Streams / Kafka）

```json
{
  "id": "cmd_20250101_001",
  "type": "modify",
  "target": "pages[0].components[1].columns",
  "instruction": "在订单列表中增加一列'创建人'",
  "priority": "high",
  "context": {
    "project_id": "proj_001",
    "iteration_id": "iter_001",
    "dsl_document_id": "dsl_001",
    "user_id": "user_001",
    "conversation_id": "conv_001"
  },
  "status": "pending",
  "created_at": "2025-01-01T10:30:00Z"
}
```

## 4. Agent 通信协议

### 4.1 请求协议（平台 → Agent）

```json
{
  "request_id": "req_uuid",
  "task_id": "task_123",
  "context": {
    "project": {
      "id": "proj_001",
      "name": "订单管理系统"
    },
    "iteration": {
      "id": "iter_001",
      "name": "V1.0"
    },
    "dsl": {},
    "knowledge": []
  },
  "instruction": "根据用户需求生成PRD",
  "input": {
    "type": "prd_generation",
    "data": {
      "requirement": "做一个订单管理系统",
      "clarifications": []
    }
  },
  "callback_url": "http://platform/api/v1/agent/callback"
}
```

### 4.2 同步响应

```json
{
  "request_id": "req_uuid",
  "status": "SUCCESS",
  "output": {
    "type": "dsl_patch",
    "data": {}
  },
  "artifacts": [
    {
      "type": "prd",
      "format": "markdown",
      "content": "..."
    }
  ]
}
```

### 4.3 异步回调

```json
{
  "request_id": "req_uuid",
  "task_id": "task_123",
  "status": "RUNNING",
  "progress": 60,
  "stage": "generating_prd",
  "logs": "正在生成功能模块...",
  "partial_output": {
    "type": "dsl_patch",
    "data": {}
  },
  "questions": [
    {
      "id": "q_001",
      "question": "是否需要审批流程？",
      "options": ["是", "否"],
      "context": "检测到订单涉及金额操作"
    }
  ]
}
```

## 5. 知识向量模型（Milvus）

### 5.1 Collection 设计

```json
{
  "collection_name": "knowledge_vectors",
  "fields": [
    { "name": "id", "type": "VARCHAR", "max_length": 100, "is_primary": true },
    { "name": "project_id", "type": "VARCHAR", "max_length": 50 },
    { "name": "iteration_id", "type": "VARCHAR", "max_length": 50 },
    { "name": "knowledge_level", "type": "VARCHAR", "max_length": 20 },
    { "name": "doc_type", "type": "VARCHAR", "max_length": 50 },
    { "name": "content", "type": "VARCHAR", "max_length": 10000 },
    { "name": "embedding", "type": "FLOAT_VECTOR", "dim": 768 }
  ],
  "index": {
    "field_name": "embedding",
    "index_type": "IVF_FLAT",
    "metric_type": "COSINE",
    "params": { "nlist": 1024 }
  }
}
```

### 5.2 检索优先级策略

```
检索时按以下优先级排序结果:

1. 迭代知识 (knowledge_level = 'iteration' AND iteration_id = current)
   → 权重 × 1.5

2. 项目知识 (knowledge_level = 'project' AND project_id = current)
   → 权重 × 1.2

3. 通用知识 (knowledge_level = 'global')
   → 权重 × 1.0

最终分数 = cosine_similarity × level_weight
```

## 6. 缓存策略（Redis）

```
Key 设计规范: {service}:{resource}:{id}

session:{user_id}                          → 用户会话
project:{project_id}:dsl                   → DSL 文档缓存
project:{project_id}:iteration:{id}:cmd    → 指令队列 (Redis Streams)
sse:{user_id}:channel                      → SSE 连接管理
agent:{agent_id}:health                    → Agent 健康状态
lock:dsl:{dsl_id}                          → DSL 修改分布式锁
rate:api:{user_id}                         → API 限流计数
```
