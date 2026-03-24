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

### 3.3 BA Agent 参考实现（Dify 示例）

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
│  │ ·Dify API  │  │ ·Agent注册  │            │
│  │ ·HTTP REST │  │ ·能力匹配   │            │
│  │ ·gRPC      │  │ ·负载均衡   │            │
│  │ ·WebSocket │  │            │            │
│  └────────────┘  └────────────┘            │
│                                             │
│  ┌────────────┐  ┌────────────┐            │
│  │ 回调管理器  │  │ 流式转发器  │            │
│  │            │  │            │            │
│  │ ·结果接收   │  │ ·SSE 转发  │            │
│  │ ·状态更新   │  │ ·日志流    │            │
│  │ ·事件发布   │  │ ·进度推送  │            │
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
| Dify Agent | Dify API (HTTP) | 转换为 Dify Workflow API 格式 |
| 自建 Agent | HTTP REST | 标准 JSON 协议 |
| 自建 Agent | gRPC | Protocol Buffers 协议 |
| 第三方 Agent | HTTP REST | 标准 JSON 协议 + 自定义适配器 |

### 4.3 回调机制

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

### 4.4 输入预处理（平台内部能力）

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

### 4.5 输出后处理（平台内部能力）

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
