# Agent 系统设计

## 1. 设计原则

### 1.1 核心原则：平台与 Agent 完全解耦

```
❗ 关键架构决策:

平台 ≠ Agent 容器
平台 = 数字员工管理 + 任务调度 + 执行监控

所有 Agent（包括需求分析、PRD 生成、开发、测试等）都是外部独立服务。
平台通过标准协议与 Agent 交互，不包含任何业务 Agent 实现。
```

### 1.2 Agent 的来源

| 来源 | 说明 | 示例 |
|------|------|------|
| Dify 创建 | 通过 Dify 平台可视化编排的 AI 工作流 | BA Agent、代码审查 Agent |
| 自行开发 | 团队自主开发，遵循平台标准协议 | 存量系统分析 Agent |
| 第三方服务 | 外部 AI 服务，通过适配器接入 | GPT Agent、Claude Agent |

### 1.3 平台内部能力 vs 外部 Agent

| 项目 | 归属 | 说明 |
|------|------|------|
| 任务路由与分配 | ✅ 平台内部 | 根据能力匹配将任务分配给合适的 Agent |
| 健康检查 | ✅ 平台内部 | 监控外部 Agent 的可用性 |
| 文件预处理 | ✅ 平台内部 | 将用户上传的文件解析为 Agent 可消费的格式 |
| DSL 合并与验证 | ✅ 平台内部 | 将 Agent 输出合并到 DSL，验证数据一致性 |
| 输入/输出转换 | ✅ 平台内部 | 根据 Agent 协议适配输入输出格式 |
| 需求理解/分析 | ❌ 外部 Agent | 由外部 BA Agent 完成 |
| PRD 生成 | ❌ 外部 Agent | 由外部 BA Agent 完成 |
| 流程设计 | ❌ 外部 Agent | 由外部 BA Agent 完成 |
| 代码生成 | ❌ 外部 Agent | 由外部开发 Agent 完成 |
| 测试执行 | ❌ 外部 Agent | 由外部测试 Agent 完成 |

## 2. 数字员工体系

### 2.1 数字员工的定义

数字员工是平台中的一等实体，代表一个可被调度的"虚拟员工"。它不实现任何业务逻辑，而是将 **角色 + Agent + 工作流 + 知识** 组合在一起。

```json
{
  "id": "emp_001",
  "name": "BA 小智",
  "role": "business_analyst",
  "description": "负责需求分析、PRD 生成、流程设计的 BA 数字员工",
  "avatar": "ba_avatar.png",
  "status": "active",
  "agent_bindings": [
    {
      "capability": "requirement_analysis",
      "agent_id": "agent_dify_ba_001",
      "priority": 1
    },
    {
      "capability": "prd_generation",
      "agent_id": "agent_dify_ba_001",
      "priority": 1
    },
    {
      "capability": "flow_design",
      "agent_id": "agent_ext_flow_001",
      "priority": 1
    }
  ],
  "workflow_templates": ["wf_new_requirement", "wf_requirement_review"],
  "knowledge_scope": {
    "global": true,
    "project_ids": ["proj_001"],
    "iteration_ids": ["iter_001"]
  },
  "performance": {
    "tasks_completed": 156,
    "avg_score": 88.5,
    "avg_duration_minutes": 12
  }
}
```

### 2.2 数字员工角色类型

| 角色 | 说明 | 需要的 Agent 能力 |
|------|------|-----------------|
| BA (Business Analyst) | 需求分析、PRD 生成 | requirement_analysis, prd_generation, flow_design, prototype_parsing, requirement_review |
| Developer | 代码开发 | code_generation, code_review, api_design |
| Tester | 测试 | test_generation, test_execution, bug_analysis |
| DevOps | 部署运维 | deployment, monitoring, scaling |
| Data Analyst | 数据分析 | data_extraction, report_generation, etl |
| System Analyst | 系统分析 | legacy_analysis, database_analysis, api_parsing |
| 自定义 | 用户自定义角色 | 用户自行配置能力需求 |

### 2.3 数字员工管理功能

```
数字员工管理
├── 创建数字员工
│   ├── 选择角色模板（BA/开发/测试/...）
│   ├── 绑定外部 Agent（可绑定多个）
│   ├── 关联工作流模板
│   └── 配置知识库范围
├── 配置与调整
│   ├── 修改 Agent 绑定
│   ├── 调整工作流
│   ├── 更新知识范围
│   └── 设置工作参数（超时、重试等）
├── 启停管理
│   ├── 启用/禁用
│   ├── 暂停/恢复
│   └── 批量管理
└── 考核与统计
    ├── 任务完成数量
    ├── 平均完成时间
    ├── 质量评分
    └── 错误率
```

## 3. 外部 Agent 接入规范

### 3.1 Agent 能力声明

每个外部 Agent 注册到平台时，必须声明以下信息：

```json
{
  "agent_code": "dify_ba_requirement",
  "name": "BA 需求分析 Agent",
  "version": "1.0.0",
  "description": "分析用户输入，提取核心需求，生成需求摘要和功能列表",
  "provider": "dify",
  "endpoint_url": "https://dify.example.com/v1/workflows/run",
  "capabilities": [
    "requirement_analysis",
    "requirement_clarification",
    "requirement_decomposition"
  ],
  "input_schema": {
    "type": "object",
    "properties": {
      "raw_input": { "type": "string", "description": "用户原始输入" },
      "documents": {
        "type": "array",
        "items": { "type": "string" },
        "description": "文档文件路径列表"
      }
    },
    "required": ["raw_input"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "summary": { "type": "string" },
      "features": { "type": "array" },
      "stakeholders": { "type": "array" },
      "constraints": { "type": "array" }
    }
  },
  "supported_modes": ["sync", "async"],
  "idempotent": true,
  "timeout_seconds": 300,
  "retry_config": {
    "max_retries": 3,
    "backoff_ms": 1000
  }
}
```

### 3.2 标准能力类型

平台预定义以下能力类型，外部 Agent 注册时选择对应的能力：

| 能力分类 | 能力代码 | 说明 |
|---------|---------|------|
| **需求类** | requirement_analysis | 需求理解与分析 |
| | requirement_clarification | 需求澄清（生成问题） |
| | requirement_decomposition | 需求拆解 |
| | requirement_review | 需求评审 |
| **产出类** | prd_generation | PRD 文档生成 |
| | flow_design | 流程图设计 |
| | prototype_parsing | 原型/HTML 解析 |
| | data_model_design | 数据模型设计 |
| **系统类** | legacy_analysis | 存量系统分析 |
| | code_analysis | 代码分析 |
| | database_analysis | 数据库分析 |
| | api_parsing | 接口解析 |
| **开发类** | code_generation | 代码生成 |
| | code_review | 代码审查 |
| | api_design | API 设计 |
| **测试类** | test_generation | 测试用例生成 |
| | test_execution | 测试执行 |
| | bug_analysis | 缺陷分析 |
| **运维类** | deployment | 部署 |
| | monitoring | 监控 |
| | scaling | 扩容 |
| **通用类** | text_extraction | 文本提取 |
| | document_parsing | 文档解析 |
| | dsl_synchronization | DSL 同步 |
| | impact_analysis | 影响分析 |

### 3.3 Skill 协议标准化（兼容 OpenClaw/AgentSkills）

平台的 Skill（能力/技能）定义采用标准化协议，**兼容 OpenClaw 的 Skill 格式和 AgentSkills 开放标准**，确保社区 Skill 可以直接接入平台使用。

#### 3.3.1 Skill 定义标准

每个 Skill 遵循 AgentSkills 规范的 YAML+Markdown 格式（`SKILL.md`），平台同时支持 JSON 等价表示：

**SKILL.md 格式（OpenClaw 兼容）**：

```yaml
---
name: requirement-analysis
description: >-
  分析用户输入，提取核心需求，生成需求摘要和功能列表。
  当用户提交新的需求描述、需求文档、或说"分析这个需求"时触发。
version: 1.0.0
author: aitaskos-community
license: Apache-2.0
tags:
  - requirement
  - analysis
  - ba
compatibility: "需要 LLM 模型支持"
metadata:
  aitaskos:
    capability: requirement_analysis
    category: requirement
    input_modes:
      - text
      - file
    output_modes:
      - application/json
    estimated_duration_seconds: 120
    idempotent: true
---
# 需求分析 Skill

## 职责
分析用户提供的原始需求描述，提取核心需求，生成结构化的需求摘要。

## 输入
- `raw_input`: 用户原始需求描述（必需）
- `documents`: 关联文档列表（可选）

## 输出
- `summary`: 需求摘要
- `features`: 功能列表
- `stakeholders`: 干系人
- `constraints`: 约束条件

## 执行说明
1. 首先理解用户的核心诉求
2. 提取关键业务实体和流程
3. 识别功能需求和非功能需求
4. 输出结构化 JSON
```

**JSON 等价表示（平台 API 注册格式）**：

```json
{
  "name": "requirement-analysis",
  "description": "分析用户输入，提取核心需求，生成需求摘要和功能列表。当用户提交新的需求描述、需求文档、或说\"分析这个需求\"时触发。",
  "version": "1.0.0",
  "author": "aitaskos-community",
  "license": "Apache-2.0",
  "tags": ["requirement", "analysis", "ba"],
  "compatibility": "需要 LLM 模型支持",
  "metadata": {
    "aitaskos": {
      "capability": "requirement_analysis",
      "category": "requirement",
      "input_modes": ["text", "file"],
      "output_modes": ["application/json"],
      "estimated_duration_seconds": 120,
      "idempotent": true
    }
  },
  "input_schema": {
    "type": "object",
    "properties": {
      "raw_input": { "type": "string", "description": "用户原始输入" },
      "documents": { "type": "array", "items": { "type": "string" } }
    },
    "required": ["raw_input"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "summary": { "type": "string" },
      "features": { "type": "array" },
      "stakeholders": { "type": "array" },
      "constraints": { "type": "array" }
    }
  }
}
```

#### 3.3.2 字段规范（对齐 AgentSkills 标准）

| 字段 | 类型 | 必填 | 规范 | 说明 |
|------|------|------|------|------|
| `name` | string | ✅ | 小写字母+连字符，1-64 字符，`^[a-z0-9]+(-[a-z0-9]+)*$` | Skill 唯一标识 |
| `description` | string | ✅ | 1-1024 字符，包含触发关键词 | 描述 Skill 功能和触发条件 |
| `version` | string | 否 | SemVer 格式 | 版本号 |
| `author` | string | 否 | — | 作者 |
| `license` | string | 否 | SPDX 格式 | 许可证 |
| `tags` | string[] | 否 | — | 分类标签 |
| `compatibility` | string | 否 | ≤500 字符 | 运行环境要求 |
| `metadata` | object | 否 | 自由扩展 | 平台专属扩展字段 |
| `input_schema` | object | 否 | JSON Schema | 输入格式定义（平台扩展） |
| `output_schema` | object | 否 | JSON Schema | 输出格式定义（平台扩展） |

> **兼容性保证**：任何符合 OpenClaw SKILL.md 标准的 Skill 文件，都可以直接导入到 AiTaskOS 平台使用。平台通过 YAML frontmatter 解析器自动提取元数据。

#### 3.3.3 Skill 注册流程

```
Skill 注册方式:

方式 1: SKILL.md 文件导入（OpenClaw 兼容）
  上传 SKILL.md → 平台解析 YAML frontmatter → 注册为 Skill
  └── 支持从 ClawHub 批量导入社区 Skill

方式 2: JSON API 注册
  POST /api/v1/agents/{id}/skills → 使用 JSON 等价格式注册

方式 3: A2A Agent Card 自动发现
  读取 Agent Card 中的 skills 数组 → 自动注册
  └── 平台将 A2A skills 字段映射为标准 Skill 格式

方式 4: 目录扫描（自建 Agent）
  Agent 提供 /skills 端点 → 平台主动拉取 Skill 清单
```

#### 3.3.4 Skill 与 Agent 的关系

```
Agent 与 Skill 的关系:

一个 Agent 可以拥有多个 Skill:
  Agent "BA 全能助手"
  ├── Skill: requirement-analysis    (需求分析)
  ├── Skill: prd-generation          (PRD 生成)
  └── Skill: flow-design             (流程设计)

一个 Skill 可以被多个 Agent 实现:
  Skill "requirement-analysis"
  ├── Agent: dify_ba_v1    (Dify 实现)
  ├── Agent: custom_ba_v2  (自建实现)
  └── Agent: third_party_ba (第三方实现)

平台调度时:
  Task (task_type=requirement_analysis)
  → 匹配数字员工 (Employee) 
  → 查找 Agent Binding (capability=requirement_analysis)
  → 找到 Agent → 验证 Agent 拥有对应 Skill
  → 调度执行
```

#### 3.3.5 Skill 数据模型

```sql
-- Skill 注册表（兼容 OpenClaw/AgentSkills 标准）
CREATE TABLE agent_skills (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        BIGINT REFERENCES agents(id),
    name            VARCHAR(64) NOT NULL,               -- OpenClaw 标准: 小写+连字符
    description     VARCHAR(1024) NOT NULL,             -- OpenClaw 标准: 1-1024 字符
    version         VARCHAR(20),                        -- SemVer
    author          VARCHAR(200),
    license         VARCHAR(50),                        -- SPDX 格式
    tags            JSONB,                              -- 标签数组
    compatibility   VARCHAR(500),
    metadata        JSONB,                              -- 扩展元数据（含 aitaskos 专属字段）
    input_schema    JSONB,                              -- JSON Schema
    output_schema   JSONB,                              -- JSON Schema
    skill_source    VARCHAR(20) DEFAULT 'manual',       -- manual/openclaw/a2a/scan
    source_url      VARCHAR(500),                       -- 原始 SKILL.md 地址
    status          VARCHAR(20) DEFAULT 'active',       -- active/inactive
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(agent_id, name)
);

CREATE INDEX idx_skill_agent ON agent_skills(agent_id);
CREATE INDEX idx_skill_name ON agent_skills(name);
CREATE INDEX idx_skill_tags ON agent_skills USING GIN(tags);
```

#### 3.3.6 与 A2A Agent Card 的映射

A2A Agent Card 的 `skills` 字段可以直接映射到平台 Skill 模型：

```
A2A Agent Card skills          →    平台 agent_skills 表
─────────────────────────────────────────────────────────
skills[].id                    →    name
skills[].name                  →    description (前 64 字符)
skills[].description           →    description
skills[].tags                  →    tags
skills[].examples              →    metadata.examples
```

OpenClaw SKILL.md 字段映射：

```
OpenClaw SKILL.md              →    平台 agent_skills 表
─────────────────────────────────────────────────────────
name                           →    name
description                    →    description
version                        →    version
author                         →    author
license                        →    license
tags                           →    tags
compatibility                  →    compatibility
metadata                       →    metadata
(Markdown body)                →    metadata.instructions
```

### 3.4 BA Agent 参考实现（Dify 示例）

以下是使用 Dify 创建 BA Agent 的参考设计，这些 Agent 独立于平台运行：

```
Dify 工作流示例: BA 需求分析 Agent
├── 输入节点: 接收平台标准请求
├── LLM 节点: 调用 DeepSeek/Qwen 分析需求
├── 条件判断: 是否需要澄清
│   ├── 是 → 生成问题列表 → 回调平台
│   └── 否 → 继续
├── LLM 节点: 功能拆解
├── LLM 节点: 生成结构化输出
└── 输出节点: 返回标准响应

Dify 工作流示例: BA PRD 生成 Agent
├── 输入节点: 接收 DSL features + 模板
├── LLM 节点: 逐章节生成 PRD
├── 流式输出: 通过 SSE 逐段返回
├── LLM 节点: 生成页面结构 (DSL pages)
├── LLM 节点: 生成数据模型 (DSL entities)
└── 输出节点: 返回完整 DSL 补丁
```

> **注意**：以上仅为参考实现。用户可以用任何方式创建 Agent，只要遵循平台标准协议即可。

## 4. Agent Gateway 设计

### 4.1 架构

```
┌─────────────────────────────────────────────┐
│              Agent Gateway                   │
│                                             │
│  ┌────────────┐  ┌────────────┐            │
│  │ 协议适配器  │  │ 路由管理器  │            │
│  │            │  │            │            │
│  │ ·A2A 协议  │  │ ·Agent注册  │            │
│  │ ·Dify API  │  │ ·能力匹配   │            │
│  │ ·HTTP REST │  │ ·负载均衡   │            │
│  │ ·gRPC      │  │            │            │
│  │ ·WebSocket │  │            │            │
│  └────────────┘  └────────────┘            │
│                                             │
│  ┌────────────┐  ┌────────────┐            │
│  │ MCP Server │  │ 回调管理器  │            │
│  │(平台工具)   │  │            │            │
│  │            │  │ ·结果接收   │            │
│  │ ·DSL 读写  │  │ ·状态更新   │            │
│  │ ·知识检索  │  │ ·事件发布   │            │
│  │ ·任务上下文│  │            │            │
│  │ ·文件解析  │  └────────────┘            │
│  └────────────┘                            │
│                                             │
│  ┌────────────┐  ┌────────────┐            │
│  │ 流式转发器  │  │ 可观测集成  │            │
│  │            │  │            │            │
│  │ ·SSE 转发  │  │ ·Langfuse  │            │
│  │ ·日志流    │  │  Trace 上报 │            │
│  │ ·进度推送  │  │ ·OTEL 遥测 │            │
│  └────────────┘  └────────────┘            │
│                                             │
│  ┌────────────┐  ┌────────────┐            │
│  │ 安全管理器  │  │ 输入预处理  │            │
│  │            │  │ (平台内部)  │            │
│  │ ·Token验证 │  │            │            │
│  │ ·调用审计  │  │ ·文件解析   │            │
│  │ ·限流控制  │  │ ·格式转换   │            │
│  └────────────┘  │ ·上下文组装 │            │
│                  └────────────┘            │
└─────────────────────────────────────────────┘
```

### 4.2 协议适配

| Agent 来源 | 协议 | 适配方式 |
|-----------|------|---------|
| A2A Agent | A2A Protocol (HTTP + SSE) | 标准 A2A Agent Card 发现 + Task Model |
| Dify Agent | Dify API (HTTP) | 转换为 Dify Workflow API 格式 |
| MCP Agent | MCP Protocol | Agent 通过 MCP 调用平台工具 |
| 自建 Agent | HTTP REST | 标准 JSON 协议 |
| 自建 Agent | gRPC | Protocol Buffers 协议 |
| 第三方 Agent | HTTP REST | 标准 JSON 协议 + 自定义适配器 |

**协议选择策略**：
```
新接入 Agent → 推荐 A2A 协议（行业标准，跨平台互操作）
Dify Agent  → Dify API 适配器（无缝集成）
存量 Agent  → 自定义 HTTP REST（向后兼容）
Agent 工具调用 → MCP 协议（Agent 访问平台 DSL/知识库/文件）
```

### 4.3 MCP Server（平台工具暴露）

平台作为 MCP Server，将内部工具/资源暴露给外部 Agent 调用：

```
AiTaskOS MCP Server
├── DSL 工具
│   ├── read_dsl         ← Agent 读取当前 DSL 结构
│   ├── patch_dsl        ← Agent 提交 DSL 修改补丁
│   └── validate_dsl     ← Agent 验证 DSL 数据一致性
├── 知识库工具
│   ├── search_knowledge ← Agent 向量检索知识库
│   └── get_document     ← Agent 获取文档内容
├── 任务工具
│   ├── get_task_context  ← Agent 获取任务上下文信息
│   ├── submit_question   ← Agent 提交澄清问题
│   └── report_progress   ← Agent 上报执行进度
└── 文件工具
    ├── parse_file        ← Agent 请求平台解析文件
    └── download_artifact ← Agent 下载已生成的制品
```

### 4.4 A2A Agent Card 兼容

外部 Agent 可通过 A2A Agent Card 标准注册到平台。平台自动将 A2A `skills` 字段解析为标准 Skill 模型（兼容 OpenClaw 格式）：

```json
{
  "name": "BA 需求分析 Agent",
  "description": "分析用户输入，提取核心需求，生成需求摘要和功能列表",
  "url": "https://agent.example.com",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "stateTransitionHistory": true
  },
  "authentication": {
    "schemes": ["oauth2", "apiKey"]
  },
  "defaultInputModes": ["text", "file"],
  "defaultOutputModes": ["text", "application/json"],
  "skills": [
    {
      "id": "requirement-analysis",
      "name": "需求分析",
      "description": "分析用户输入，提取核心需求。当用户提交新的需求描述或说\"分析这个需求\"时触发。",
      "tags": ["requirement", "analysis", "ba"],
      "examples": ["分析这份需求文档", "做一个订单管理系统"]
    },
    {
      "id": "prd-generation",
      "name": "PRD 生成",
      "description": "根据需求分析结果生成完整的产品需求文档。当需要生成 PRD 或产品文档时触发。",
      "tags": ["prd", "generation", "document"],
      "examples": ["生成 PRD 文档", "输出产品需求规格"]
    }
  ]
}
```

> A2A `skills` 中的每个条目会自动注册为平台 `agent_skills` 表中的记录，`id` 映射为 `name`（需满足 OpenClaw 命名规范：小写+连字符）。

### 4.5 回调机制

```
异步回调流程:

Platform ──请求──▶ External Agent
                    │
                    ├── 进度回调 (progress: 30%)
                    │     ▼
                    │   Platform 更新状态 → SSE 推送前端
                    │
                    ├── 提问回调 (question)
                    │     ▼
                    │   Platform 创建问题 → SSE 推送前端
                    │     ▼
                    │   用户回答 → Platform 转发答案 → Agent
                    │
                    ├── 进度回调 (progress: 80%)
                    │     ▼
                    │   Platform 更新状态 → SSE 推送前端
                    │
                    └── 完成回调 (status: SUCCESS)
                          ▼
                        Platform 保存结果 → 更新 DSL → SSE 推送前端
```

### 4.6 输入预处理（平台内部能力）

平台内部负责将用户输入预处理为 Agent 可消费的标准格式，这不是业务逻辑，而是平台基础能力：

```
文件预处理流程:
├── PDF → Apache Tika → 文本内容
├── Word → Apache Tika → 文本内容
├── Excel → Apache Tika → 结构化数据
├── 图片 → PaddleOCR → 文字识别结果
├── HTML → Playwright → DOM 结构
└── 所有结果 → 标准输入格式 → 发送给 Agent
```

### 4.7 输出后处理（平台内部能力）

平台接收 Agent 输出后，负责：

```
输出后处理:
├── 验证输出格式符合 Agent 声明的 output_schema
├── 将输出合并到 DSL（如果输出包含 DSL patch）
├── 检查 DSL 一致性
├── 触发关联更新（通知其他 Agent 同步）
├── 存储制品（artifacts）到 MinIO
└── 推送结果到前端（SSE）
```

## 5. Agent 调度策略

### 5.1 Task → 数字员工 → Agent 匹配流程

```
匹配逻辑:
1. 检查 task.employee_selector
   ├── MANUAL → 使用指定的数字员工
   └── AUTO → 进入自动匹配
2. 自动匹配数字员工:
   ├── 筛选: 数字员工拥有 task.task_type 对应的 Agent 能力
   ├── 过滤: 数字员工.status == 'active'
   ├── 排序: 按负载排序（执行中任务最少优先）
   └── 选择: 取第一个
3. 通过数字员工找到 Agent:
   ├── 查找数字员工的 agent_bindings
   ├── 匹配 capability == task.task_type
   ├── 检查 Agent 健康状态
   └── 发送请求到 Agent
```

### 5.2 DAG 执行引擎

```
场景 S1 (新需求生成) 的 Task DAG:

平台调度器将任务分配给外部 Agent：

    [输入预处理(平台)]
         │
         ▼
    [需求理解(外部Agent)]
         │
         ▼
    [需求澄清(外部Agent)] ──── (等待用户回答)
         │
         ▼
    [需求拆解(外部Agent)]
         │
    ┌────┴────┐
    ▼         ▼
[PRD生成]  [流程设计]    ← 均为外部 Agent
    │         │
    └────┬────┘
         ▼
    [DSL合并验证(平台)]
         │
         ▼
    [需求评审(外部Agent)]
```

### 5.3 执行控制

```
控制策略:
├── 超时控制: 每个 Agent 执行有超时时间，超时自动取消
├── 重试机制: 失败后按配置重试（指数退避）
├── 并发控制: 同一 DSL 的修改操作需要加分布式锁
├── 优先级: 指令队列中高优先级指令可中断低优先级任务
├── 幂等保证: 幂等 Agent 重试安全，非幂等 Agent 需要检查状态
└── 降级策略: 首选 Agent 不可用时，自动切换到数字员工绑定的备选 Agent
```

## 6. 考核与统计

### 6.1 数字员工绩效指标

| 指标 | 说明 | 计算方式 |
|------|------|---------|
| 任务完成率 | 成功完成的任务占比 | completed / (completed + failed) |
| 平均耗时 | 任务平均执行时间 | sum(duration) / count |
| 质量评分 | 用户对输出质量的评分 | 用户反馈加权平均 |
| 错误率 | 执行失败的比例 | failed / total |
| 可用率 | Agent 在线可用时间占比 | uptime / total_time |

### 6.2 考核数据用途

```
考核数据 → 自动优化调度策略
├── 高绩效数字员工 → 优先分配任务
├── 低绩效数字员工 → 自动降权或告警
├── Agent 频繁失败 → 建议更换 Agent
└── 响应慢的 Agent → 调整超时策略
```

## 7. Agent 可观测性

### 7.1 双层可观测架构

```
全链路观测:

用户请求 → API Server → 调度器 → Agent Gateway → 外部 Agent
    │           │          │           │             │
    └───────────┴──────────┴───────────┴─────────────┘
                         │
              ┌──────────┴──────────┐
              │  OpenTelemetry SDK  │
              └──────────┬──────────┘
                    │         │
                    ▼         ▼
              ┌─────────┐ ┌──────────┐
              │ Jaeger  │ │ Langfuse │
              │(基础设施)│ │(Agent/LLM)│
              └─────────┘ └──────────┘
                    │         │
                    └────┬────┘
                         ▼
                   ┌──────────┐
                   │ Grafana  │
                   │(统一仪表盘)│
                   └──────────┘
```

### 7.2 Langfuse 集成

Agent Gateway 在每次 Agent 调用时，向 Langfuse 上报 Trace 数据：

| 采集维度 | 说明 |
|---------|------|
| 推理链路 | Agent 每一步的输入/输出/决策过程 |
| Token 用量 | 按 Agent/任务/数字员工统计 Token 消耗 |
| 调用成本 | 按模型和 Token 数计算费用 |
| 延迟分布 | 各步骤耗时分析 |
| 质量评分 | 自动评估 + 用户反馈 |

### 7.3 自动化评估管线

```
Agent 输出 → 自动评估管线
├── 结构化验证
│   ├── 输出是否符合 output_schema
│   ├── DSL patch 是否有效
│   └── 字段完整性检查
├── 语义质量评估
│   ├── LLM-as-Judge（用另一个 LLM 评估输出质量）
│   ├── 与参考答案的相似度
│   └── 幻觉检测
├── 一致性检查
│   ├── PRD 与 DSL 一致性
│   ├── 流程与功能对应性
│   └── 跨 Agent 输出一致性
└── 用户反馈
    ├── 显式评分（用户打分）
    └── 隐式信号（用户修改次数、重试次数）
```

## 8. 多 Agent 协作模式

### 8.1 协作模式定义

数字员工绑定多个 Agent 时，可配置 Agent 间的协作模式：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **chain** | 串行链：Agent A → Agent B → Agent C | 需求理解 → 需求拆解 → PRD 生成 |
| **fanout** | 并行扇出：同时调度多个 Agent | PRD 生成 + 流程设计 并行 |
| **voting** | 投票共识：多个 Agent 独立执行，选最佳结果 | 多个评审 Agent 独立评审 |
| **delegation** | 委托嵌套：Agent A 委托子任务给 Agent B | 需求分析中需要代码分析 |

### 8.2 协作配置示例

```json
{
  "employee_id": "emp_001",
  "workflow_code": "wf_new_requirement",
  "collaboration_mode": "chain",
  "collaboration_config": {
    "steps": [
      {
        "capability": "requirement_analysis",
        "agent_id": "agent_ba_001"
      },
      {
        "capability": "prd_generation",
        "mode": "fanout",
        "agents": ["agent_prd_001", "agent_prd_002"],
        "merge_strategy": "best_score"
      }
    ]
  }
}
```
