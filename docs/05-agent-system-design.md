# Agent 系统设计

## 1. Agent 体系总览

### 1.1 Agent 分类

```
Agent 体系
├── 内置 Agent（平台自带，LangChain4j 实现）
│   ├── 需求理解 Agent (RequirementUnderstandingAgent)
│   ├── 需求澄清 Agent (RequirementClarificationAgent)
│   ├── 需求拆解 Agent (RequirementDecompositionAgent)
│   ├── PRD 生成 Agent (PrdGenerationAgent)
│   ├── 流程设计 Agent (FlowDesignAgent)
│   ├── 原型解析 Agent (PrototypeParsingAgent)
│   ├── 存量系统分析 Agent (LegacySystemAnalysisAgent)
│   ├── 需求评审 Agent (RequirementReviewAgent)
│   └── 同步 Agent (SynchronizationAgent)
├── Dify Agent（通过 Dify 平台编排）
│   └── 自定义工作流 Agent
└── 外部 Agent（通过标准协议接入）
    ├── 开发 Agent
    ├── 测试 Agent
    └── 部署 Agent
```

### 1.2 Agent 能力声明规范

每个 Agent 必须声明以下信息：

```json
{
  "agent_code": "requirement_understanding",
  "name": "需求理解 Agent",
  "version": "1.0.0",
  "description": "分析用户输入，提取核心需求，生成需求摘要和功能列表",
  "capabilities": [
    "requirement_analysis",
    "text_extraction",
    "document_parsing"
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

## 2. 内置 Agent 详细设计

### 2.1 需求理解 Agent

```
Agent: RequirementUnderstandingAgent
职责: 分析用户输入，提取核心需求信息

输入:
  - 用户文本输入（一句话 / 长文本 / 2万字）
  - 文档文件（PDF / Word / Excel / 图片）
  - 历史对话上下文

处理流程:
  1. 输入预处理
     ├── 文本清洗
     ├── 文档解析 (Apache Tika)
     └── 图片 OCR (PaddleOCR)
  2. 需求分析 (LLM)
     ├── 提取业务目标
     ├── 识别功能点
     ├── 识别角色
     └── 识别约束条件
  3. 结构化输出
     ├── 需求摘要
     ├── 功能列表
     ├── 角色列表
     └── 非功能需求

输出:
  {
    "summary": "构建一个订单管理系统...",
    "business_goals": ["提升订单处理效率", "..."],
    "features": [
      { "name": "订单创建", "priority": "P0" },
      { "name": "订单审批", "priority": "P1" }
    ],
    "roles": ["管理员", "操作员", "审批人"],
    "constraints": ["支持并发100用户", "响应时间<2秒"],
    "unclear_points": ["审批流程层级不明确", "..."]
  }

使用的 Skill:
  - TextExtractionSkill
  - DocumentParsingSkill
  - RequirementAnalysisSkill
```

### 2.2 需求澄清 Agent

```
Agent: RequirementClarificationAgent
职责: 自动发现需求中的不明确点，生成澄清问题

输入:
  - 需求理解 Agent 的输出
  - 当前 DSL 状态
  - 项目知识上下文

处理流程:
  1. 分析不明确点
     ├── 业务流程模糊
     ├── 数据字段缺失
     ├── 权限规则不清
     └── 异常流程未覆盖
  2. 生成问题
     ├── 生成问题文本
     ├── 提供选项建议
     └── 标注问题优先级
  3. 管理问题状态
     ├── pending → answered
     ├── 答案合并到上下文
     └── 触发后续 Agent

输出:
  {
    "questions": [
      {
        "id": "q_001",
        "question": "订单审批流程需要几级审批？",
        "priority": "high",
        "options": ["单级审批", "两级审批", "多级审批（自定义）"],
        "context": "检测到需求中提到了审批功能但未说明层级",
        "impact": ["flows.order_approve", "features.approval"]
      }
    ]
  }

使用的 Skill:
  - RequirementGapAnalysisSkill
  - QuestionGenerationSkill
```

### 2.3 需求拆解 Agent

```
Agent: RequirementDecompositionAgent
职责: 将整体需求拆解为功能模块和功能点

输入:
  - 需求摘要
  - 澄清后的答案
  - 参考模板（从知识库检索）

处理流程:
  1. 功能分组
     ├── 按业务域拆分
     └── 确定模块边界
  2. 功能点拆解
     ├── 每个模块拆分功能点
     ├── 定义功能点优先级
     └── 标注功能点依赖
  3. 生成 DSL features 节点

输出:
  DSL features 结构（见数据模型文档）

使用的 Skill:
  - FeatureDecompositionSkill
  - PriorityAssignmentSkill
```

### 2.4 PRD 生成 Agent

```
Agent: PrdGenerationAgent
职责: 生成结构化 PRD 文档

输入:
  - DSL features 结构
  - 澄清答案
  - PRD 模板（从知识库检索）
  - 项目上下文

处理流程:
  1. PRD 结构生成
     ├── 需求概述
     ├── 功能需求（按模块）
     ├── 页面需求
     ├── 数据需求
     ├── 非功能需求
     └── 验收标准
  2. 内容填充 (LLM)
     ├── 逐章节流式生成
     └── 引用 DSL 数据
  3. DSL 同步
     └── 更新 pages / entities / rules

输出:
  - PRD Markdown 文档
  - 更新后的 DSL（pages、entities、rules）

使用的 Skill:
  - PrdTemplateSkill
  - ContentGenerationSkill
  - DslSynchronizationSkill
```

### 2.5 流程设计 Agent

```
Agent: FlowDesignAgent
职责: 设计业务流程

输入:
  - DSL features 结构
  - 需求描述
  - 业务规则

处理流程:
  1. 识别流程类型
     ├── 主业务流程
     ├── 审批流程
     └── 异常处理流程
  2. 生成流程节点
     ├── 开始/结束节点
     ├── 任务节点
     ├── 判断网关
     └── 并行网关
  3. 生成 DSL flows 结构
  4. 验证流程完整性

输出:
  DSL flows 结构（见数据模型文档）

使用的 Skill:
  - FlowGenerationSkill
  - FlowValidationSkill
```

### 2.6 原型解析 Agent

```
Agent: PrototypeParsingAgent
职责: 解析已有原型/HTML/截图，反推需求

输入:
  - HTML 文件
  - 截图 (PNG/JPG)
  - Figma/Axure 导出文件

处理流程:
  1. 页面识别
     ├── HTML → DOM 解析 (Playwright)
     ├── 图片 → UI 识别 (Vision Model)
     └── 提取页面结构
  2. 组件识别
     ├── 表格、表单、按钮
     ├── 导航、菜单
     └── 弹窗、抽屉
  3. 字段提取
     ├── 输入字段
     ├── 显示字段
     └── 验证规则
  4. 生成 DSL pages 结构

输出:
  DSL pages 结构

使用的 Skill:
  - HtmlParsingSkill
  - UiRecognitionSkill
  - ComponentIdentificationSkill
```

### 2.7 存量系统分析 Agent

```
Agent: LegacySystemAnalysisAgent
职责: 分析现有系统代码/数据库，生成系统画像

输入:
  - 代码仓库地址
  - 数据库连接信息
  - 接口文档

处理流程:
  1. 代码分析
     ├── 识别技术栈
     ├── 分析项目结构
     ├── 提取 API 接口
     └── 识别业务模块
  2. 数据库分析
     ├── 提取表结构
     ├── 分析表关系
     └── 识别业务实体
  3. 接口分析
     ├── 解析 API 文档
     ├── 提取请求/响应结构
     └── 识别接口依赖
  4. 生成系统画像
     ├── 系统能力清单
     ├── 数据模型
     ├── 模块关系图
     └── 升级建议

输出:
  {
    "system_portrait": {
      "tech_stack": {},
      "modules": [],
      "api_list": [],
      "data_model": {},
      "issues": [],
      "upgrade_suggestions": []
    }
  }

使用的 Skill:
  - CodeAnalysisSkill
  - DatabaseAnalysisSkill
  - ApiParsingSkill
  - SystemPortraitSkill
```

### 2.8 需求评审 Agent

```
Agent: RequirementReviewAgent
职责: 检查需求质量，发现缺失和冲突

输入:
  - 完整 DSL
  - PRD 文档
  - 评审规则（从知识库检索）

处理流程:
  1. 完整性检查
     ├── 功能点是否都有页面
     ├── 页面是否都有字段
     ├── 流程是否有异常分支
     └── 实体是否有完整字段
  2. 一致性检查
     ├── PRD 与 DSL 是否一致
     ├── 流程与功能是否对应
     └── 实体与页面字段是否匹配
  3. 规范性检查
     ├── 命名规范
     ├── 字段类型合理性
     └── 权限完整性
  4. 生成评审报告

输出:
  {
    "review_report": {
      "score": 85,
      "issues": [
        {
          "severity": "high",
          "type": "missing",
          "message": "订单删除功能缺少权限控制",
          "suggestion": "建议增加 order:delete 权限",
          "target": "pages.order_list.actions.delete"
        }
      ],
      "suggestions": []
    }
  }

使用的 Skill:
  - CompletenessCheckSkill
  - ConsistencyCheckSkill
  - StandardCheckSkill
```

### 2.9 同步 Agent

```
Agent: SynchronizationAgent
职责: 保持 PRD、原型、流程图三者同步

输入:
  - DSL 变更事件
  - 变更路径
  - 变更内容

处理流程:
  1. 分析变更影响范围
     ├── 变更了 features → 影响 PRD、pages
     ├── 变更了 pages → 影响 PRD
     ├── 变更了 flows → 影响 PRD
     └── 变更了 entities → 影响 pages、PRD
  2. 计算需要同步的目标
  3. 逐个更新受影响的 DSL 节点
  4. 通知前端更新展示

输出:
  - 更新后的 DSL
  - 变更通知事件

使用的 Skill:
  - ImpactAnalysisSkill
  - DslSynchronizationSkill
```

## 3. Skill 能力平台

### 3.1 Skill 分类与清单

| 分类 | Skill 名称 | 说明 | 幂等 |
|------|------------|------|------|
| **PRD 类** | RequirementAnalysisSkill | 需求分析 | ✅ |
| | FeatureDecompositionSkill | 功能拆解 | ✅ |
| | PrdTemplateSkill | PRD 模板渲染 | ✅ |
| | ContentGenerationSkill | 内容生成 | ❌ |
| | RequirementGapAnalysisSkill | 需求缺口分析 | ✅ |
| | QuestionGenerationSkill | 问题生成 | ❌ |
| | PriorityAssignmentSkill | 优先级分配 | ✅ |
| **原型类** | HtmlParsingSkill | HTML 解析 | ✅ |
| | UiRecognitionSkill | UI 识别 | ✅ |
| | ComponentIdentificationSkill | 组件识别 | ✅ |
| **流程类** | FlowGenerationSkill | 流程生成 | ❌ |
| | FlowValidationSkill | 流程验证 | ✅ |
| | BpmnParsingSkill | BPMN 解析 | ✅ |
| **代码类** | CodeAnalysisSkill | 代码分析 | ✅ |
| | ApiParsingSkill | 接口解析 | ✅ |
| | DatabaseAnalysisSkill | 数据库分析 | ✅ |
| | SystemPortraitSkill | 系统画像 | ✅ |
| **通用类** | TextExtractionSkill | 文本提取 | ✅ |
| | DocumentParsingSkill | 文档解析 | ✅ |
| | DslSynchronizationSkill | DSL 同步 | ✅ |
| | ImpactAnalysisSkill | 影响分析 | ✅ |
| | CompletenessCheckSkill | 完整性检查 | ✅ |
| | ConsistencyCheckSkill | 一致性检查 | ✅ |
| | StandardCheckSkill | 规范性检查 | ✅ |

### 3.2 Skill 接口规范

```java
public interface Skill<I, O> {

    /**
     * Skill 元数据
     */
    SkillMetadata metadata();

    /**
     * 执行 Skill
     */
    SkillResult<O> execute(SkillContext context, I input);

    /**
     * 是否幂等
     */
    boolean isIdempotent();
}

public record SkillMetadata(
    String code,
    String name,
    String description,
    String category,
    Class<?> inputType,
    Class<?> outputType
) {}

public record SkillResult<T>(
    boolean success,
    T data,
    String errorMessage,
    Map<String, Object> metrics
) {}

public record SkillContext(
    String projectId,
    String iterationId,
    Map<String, Object> parameters,
    KnowledgeRetriever knowledgeRetriever
) {}
```

## 4. Agent 调度策略

### 4.1 Task → Agent 匹配规则

```
匹配逻辑:
1. 检查 task.agent_selector
   ├── MANUAL → 使用指定的 agent_id
   └── AUTO → 进入自动匹配
2. 自动匹配:
   ├── 筛选: task.task_type ∈ agent.capabilities
   ├── 过滤: agent.status == 'active' AND agent.health_status == 'healthy'
   ├── 排序: 按负载排序（执行中任务最少优先）
   └── 选择: 取第一个
```

### 4.2 DAG 执行引擎

```
场景 S1 (新需求生成) 的 Task DAG:

    [需求理解]
        │
        ▼
    [需求澄清] ──────── (等待用户回答)
        │
        ▼
    [需求拆解]
        │
    ┌───┴───┐
    ▼       ▼
[PRD生成] [流程设计]
    │       │
    └───┬───┘
        ▼
    [DSL 同步]
        │
        ▼
    [需求评审]
```

### 4.3 执行控制

```
控制策略:
├── 超时控制: 每个 Agent 执行有超时时间，超时自动取消
├── 重试机制: 失败后按配置重试（指数退避）
├── 并发控制: 同一 DSL 的修改操作需要加分布式锁
├── 优先级: 指令队列中高优先级指令可中断低优先级任务
└── 幂等保证: 幂等 Skill 重试安全，非幂等 Skill 需要检查状态
```

## 5. Agent Gateway 设计

### 5.1 架构

```
┌─────────────────────────────────────────────┐
│              Agent Gateway                   │
│                                             │
│  ┌────────────┐  ┌────────────┐            │
│  │ 协议适配器  │  │ 路由管理器  │            │
│  │            │  │            │            │
│  │ ·Internal  │  │ ·Agent注册  │            │
│  │ ·Dify API  │  │ ·能力匹配   │            │
│  │ ·HTTP REST │  │ ·负载均衡   │            │
│  │ ·gRPC      │  │            │            │
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
│  ┌────────────┐                            │
│  │ 安全管理器  │                            │
│  │            │                            │
│  │ ·Token验证 │                            │
│  │ ·调用审计  │                            │
│  │ ·限流控制  │                            │
│  └────────────┘                            │
└─────────────────────────────────────────────┘
```

### 5.2 协议适配

| Agent 类型 | 协议 | 适配方式 |
|-----------|------|---------|
| 内置 Agent | Java 方法调用 | 直接调用，无网络开销 |
| Dify Agent | Dify API (HTTP) | 转换为 Dify Workflow API 格式 |
| 外部 Agent | HTTP REST | 标准 JSON 协议 |
| 外部 Agent | gRPC | Protocol Buffers 协议 |

### 5.3 回调机制

```
异步回调流程:

Platform ──请求──▶ Agent
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
