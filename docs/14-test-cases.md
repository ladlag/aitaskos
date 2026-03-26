# 测试用例

> 基于 8 大业务场景（S1-S8）和 11 个微服务，覆盖功能测试、集成测试、性能测试、安全测试和容错测试。

## 1. 测试策略总览

### 1.1 测试金字塔

```
                    ┌───────────┐
                    │  E2E 测试  │  ~20 个（关键场景）
                    │  (Cypress) │
                  ┌─┴───────────┴─┐
                  │  集成测试       │  ~100 个（跨服务）
                  │(Testcontainers)│
                ┌─┴───────────────┴─┐
                │    API 测试        │  ~200 个（接口级）
                │   (REST Assured)  │
              ┌─┴───────────────────┴─┐
              │      单元测试          │  ~500 个（方法级）
              │     (JUnit 5)         │
              └───────────────────────┘
```

### 1.2 测试工具链

| 层级 | 工具 | 用途 |
|------|------|------|
| 单元测试 | JUnit 5 + Mockito | 业务逻辑 |
| API 测试 | REST Assured + WireMock | 接口契约 |
| 集成测试 | Testcontainers + Spring Boot Test | 跨服务 |
| E2E 测试 | Cypress | 全链路 |
| 性能测试 | Apache JMeter + Gatling | 负载/压力 |
| 安全测试 | OWASP ZAP + SonarQube | 漏洞扫描 |

### 1.3 测试覆盖目标

| 指标 | Phase 1 目标 | Phase 2 目标 |
|------|-------------|-------------|
| 单元测试覆盖率 | >70% | >80% |
| API 测试覆盖率 | >90% (核心 API) | >95% |
| 集成测试 | 核心场景 100% | 全场景 100% |
| E2E 测试 | S1+S7+S8 | S1-S8 |

---

## 2. 功能测试用例 — S1 新需求生成

### TC-S1-001: 一句话需求生成

```
测试 ID:    TC-S1-001
场景:      S1 新需求生成
标题:      一句话需求触发完整需求分析流程
优先级:    P0
前置条件:
  - 用户已登录
  - 项目已创建
  - BA 数字员工已配置并绑定需求分析 Agent

步骤:
  1. 进入项目对话页面
  2. 输入: "做一个订单管理系统"
  3. 点击发送

预期结果:
  - 过程区显示执行步骤:
    Step 1: 输入解析 ✅
    Step 2: 需求理解 🔄 (流式输出)
    Step 3: 需求澄清 → 可能出现 AI 提问
    Step 4: 需求拆解 🔄
    Step 5: PRD 生成 🔄
    Step 6: 流程设计 🔄
    Step 7: DSL 同步 ✅
    Step 8: 评审 ✅
  - 对话区显示流式 AI 输出
  - 产物区自动展示生成的 PRD 文档
  - 结构区显示需求树结构
  - DSL 文档被创建/更新

验证点:
  ✅ SSE 事件正确推送
  ✅ DSL 文档包含 features[] + pages[] + flows[]
  ✅ 审计日志记录完整
  ✅ Agent 执行记录包含 Token 用量
```

### TC-S1-002: 长文本需求输入

```
测试 ID:    TC-S1-002
标题:      2 万字长文本需求处理
优先级:    P0
前置条件:  同 TC-S1-001

步骤:
  1. 粘贴 2 万字需求描述
  2. 点击发送

预期结果:
  - 系统不超时 (5min 内完成)
  - 需求被正确理解和拆解
  - 功能模块数量 ≥5

验证点:
  ✅ 大文本不被截断
  ✅ Agent 接收完整内容
  ✅ 执行时间 <300s
```

### TC-S1-003: 文件上传需求

```
测试 ID:    TC-S1-003
标题:      PDF/Word 文件上传触发需求分析
优先级:    P0

步骤:
  1. 上传 PDF 需求文档 (10 页, ~2MB)
  2. 附加说明: "按这个文档生成需求"

预期结果:
  - 文件被 Tika 解析为文本
  - 文本内容发送给需求分析 Agent
  - 生成完整 DSL

验证点:
  ✅ PDF 文本提取正确
  ✅ 中文内容无乱码
  ✅ 文件存储在 MinIO
```

### TC-S1-004: AI 主动提问（异步反馈）

```
测试 ID:    TC-S1-004
标题:      需求澄清阶段 AI 提问并等待用户回答
优先级:    P0

步骤:
  1. 输入模糊需求: "做一个系统"
  2. 等待 AI 提问
  3. 回答问题
  4. 等待继续执行

预期结果:
  - AI 生成 2-5 个澄清问题
  - 问题以卡片形式展示（可选选项 + 自由输入）
  - 工作流暂停等待回答 (Temporal Signal)
  - 用户回答后工作流继续

验证点:
  ✅ SSE event: question 正确推送
  ✅ 工作流正确暂停和恢复
  ✅ 多轮提问支持 (最多 3 轮)
  ✅ 超时未回答 → 使用默认值继续
```

### TC-S1-005: 并行生成 PRD + 流程图

```
测试 ID:    TC-S1-005
标题:      PRD 生成和流程设计并行执行
优先级:    P1

步骤:
  1. 完成需求拆解后观察执行过程

预期结果:
  - PRD 生成 Agent 和流程设计 Agent 同时启动
  - 两个 Agent 的输出分别更新 DSL 不同路径
  - 最终 DSL 同步合并

验证点:
  ✅ 并行执行无数据冲突
  ✅ 合并后 DSL 完整
  ✅ 两个 Agent 的 Trace 独立
```

---

## 3. 功能测试用例 — S7 局部修改

### TC-S7-001: 选中文本修改

```
测试 ID:    TC-S7-001
场景:      S7 局部修改
标题:      选中 PRD 段落触发 AI 修改
优先级:    P0

步骤:
  1. 在 PRD 编辑器中选中一段描述
  2. 点击浮动工具栏 "AI 修改"
  3. 输入指令: "增加一列创建人"
  4. 确认执行

预期结果:
  - 指令进入指令队列 (状态: pending)
  - Agent 接收: 选中内容 + 修改指令 + 上下文 DSL
  - 仅修改相关 DSL 路径（增量 Patch）
  - PRD 编辑器局部刷新
  - 其他部分不受影响

验证点:
  ✅ DSL Patch 操作正确 (仅 add 操作)
  ✅ 版本号递增
  ✅ 其他 DSL 路径未变
  ✅ SSE event: dsl_update 推送
```

### TC-S7-002: 指令队列优先级

```
测试 ID:    TC-S7-002
标题:      高优先级指令插队执行
优先级:    P0

步骤:
  1. 提交 normal 优先级指令 A
  2. 提交 normal 优先级指令 B
  3. 提交 high 优先级指令 C

预期结果:
  - 执行顺序: A → C (中断 B) → B
  - 或: A → C → B (A 已完成时)
  - 指令队列面板实时显示状态

验证点:
  ✅ high 优先级正确插队
  ✅ 被中断的指令可恢复
  ✅ 队列状态 SSE 推送
```

### TC-S7-003: 并发修改冲突

```
测试 ID:    TC-S7-003
标题:      两个用户同时修改同一 DSL 区域
优先级:    P0

步骤:
  1. 用户 A 提交指令修改 features[0].name
  2. 用户 B 同时提交指令修改 features[0].description

预期结果:
  - 先到达的指令成功执行
  - 后到达的指令检测到版本冲突
  - 自动重试（最多 3 次）
  - 如果路径不同 → 可以成功合并
  - 如果路径冲突 → 通知用户手动解决

验证点:
  ✅ 乐观锁正确拦截冲突
  ✅ 自动重试成功（路径不同时）
  ✅ 冲突通知正确推送
```

---

## 4. 功能测试用例 — S8 需求评审

### TC-S8-001: 自动评审流程

```
测试 ID:    TC-S8-001
场景:      S8 需求评审
标题:      提交 DSL 触发自动评审
优先级:    P0

步骤:
  1. S1 生成完成后，点击 "提交评审"

预期结果:
  - 评审 Agent 收到完整 DSL
  - 返回评审意见:
    - 功能完整性评分
    - 一致性检查结果
    - 改进建议列表
  - 评审结果展示在产物区

验证点:
  ✅ 评审 Agent 正确接收 DSL
  ✅ 评估记录写入 agent_evaluation_records
  ✅ 评分 0-100 范围合法
```

---

## 5. 功能测试用例 — S2/S6 场景

### TC-S2-001: 原型图上传生成需求

```
测试 ID:    TC-S2-001
场景:      S2 原型驱动需求
标题:      上传 HTML 原型文件生成 PRD
优先级:    P1

步骤:
  1. 上传 HTML 原型文件
  2. 系统解析页面结构
  3. Agent 将页面元素映射为 DSL pages

预期结果:
  - 页面组件被正确识别 (表格/表单/按钮等)
  - DSL pages[] 自动填充
  - 用户可以确认/修改后继续生成 PRD

验证点:
  ✅ HTML DOM 解析正确
  ✅ 组件类型识别准确率 >80%
```

### TC-S6-001: 需求迭代（基于已有 DSL）

```
测试 ID:    TC-S6-001
场景:      S6 需求迭代
标题:      在已有需求基础上添加新功能
优先级:    P1

步骤:
  1. 打开已有项目（含 DSL）
  2. 输入: "增加退款功能"

预期结果:
  - Agent 接收: 现有 DSL + 新需求
  - 差异分析: 识别需要新增/修改的部分
  - 增量更新 DSL (不覆盖已有内容)
  - DSL Diff 展示变更

验证点:
  ✅ 现有 features 保留
  ✅ 新增 feature 正确追加
  ✅ 关联的 pages/flows 同步更新
  ✅ 版本 Diff 清晰可见
```

---

## 6. 功能测试用例 — S3/S4/S5 场景

### TC-S3-001: HTML 页面反推需求

```
测试 ID:    TC-S3-001
场景:      S3 HTML/系统反推需求
优先级:    P2

步骤:
  1. 输入已有系统 URL 或上传 HTML 页面
  2. 系统解析页面结构

预期结果:
  - 页面结构被解析为 DSL pages
  - 推断出业务流程和数据实体
```

### TC-S4-001: 流程图驱动需求

```
测试 ID:    TC-S4-001
场景:      S4 流程图驱动需求
优先级:    P2

步骤:
  1. 在流程编辑器绘制业务流程
  2. 触发 "根据流程生成需求"

预期结果:
  - 流程节点映射为 DSL flows
  - Agent 根据流程推断功能和页面
```

### TC-S5-001: 存量系统升级分析

```
测试 ID:    TC-S5-001
场景:      S5 存量系统升级需求
优先级:    P2

步骤:
  1. 提供存量系统信息（代码仓库/数据库/接口文档）
  2. 多个 Agent 并行分析

预期结果:
  - 代码分析 Agent: 输出技术栈、模块结构
  - 数据库分析 Agent: 输出 ER 图、数据字典
  - 接口分析 Agent: 输出 API 清单
  - 汇总生成系统画像和升级方案
```

---

## 6.5 功能测试用例 — 自动审查优化机制

### TC-AR-001: 自动审查开关关闭时不触发审查

```
测试 ID:    TC-AR-001
标题:      自动审查开关关闭时正常执行无额外审查
优先级:    P0

前置条件:
  - 数字员工已配置，auto_review_enabled = false

步骤:
  1. 创建任务触发工作流
  2. Agent 返回输出（质量评分 60，低于默认阈值 80）

预期结果:
  - 工作流正常完成，不触发自动审查
  - Agent 只执行 1 次
  - 无 auto_review SSE 事件推送
  - agent_executions 记录 iteration = 1

验证点:
  ✅ 开关关闭时零额外开销
  ✅ 行为与原有流程完全一致
```

### TC-AR-002: 首轮输出达标直接通过

```
测试 ID:    TC-AR-002
标题:      自动审查开启，首轮输出评分达标，不迭代
优先级:    P0

前置条件:
  - auto_review_enabled = true
  - quality_threshold = 80
  - max_iterations = 3

步骤:
  1. 创建任务触发工作流
  2. Agent 返回高质量输出（评估管线评分 85）

预期结果:
  - 评估管线执行 1 次
  - 评分 85 ≥ 阈值 80 → 直接通过
  - Agent 只执行 1 次
  - SSE 推送 auto_review 事件：iteration=1, status="passed"

验证点:
  ✅ 达标输出不触发不必要的迭代
  ✅ 审查状态正确推送
```

### TC-AR-003: 首轮不达标，第二轮达标

```
测试 ID:    TC-AR-003
标题:      自动审查迭代一次后达标
优先级:    P0

前置条件:
  - auto_review_enabled = true
  - quality_threshold = 80
  - max_iterations = 3
  - iteration_strategy = "feedback_loop"

步骤:
  1. 创建任务触发工作流
  2. Agent 首轮输出评分 60（不达标）
  3. 系统生成审查反馈并重新调度 Agent
  4. Agent 第二轮输出评分 90（达标）

预期结果:
  - Agent 执行 2 次
  - 第二轮输入包含 review_feedback 和 previous_output
  - agent_executions 有 2 条记录：
    - iteration=1, review_score=60
    - iteration=2, parent_execution_id=第一条ID
  - SSE 推送 2 个 auto_review 事件

验证点:
  ✅ 审查反馈正确传递给 Agent
  ✅ 迭代链路（parent_execution_id）正确
  ✅ 最终采用第二轮输出
```

### TC-AR-004: 达到最大迭代次数

```
测试 ID:    TC-AR-004
标题:      多轮迭代均不达标，达到最大次数后停止
优先级:    P0

前置条件:
  - auto_review_enabled = true
  - quality_threshold = 80
  - max_iterations = 3
  - notify_on_max_iterations = true

步骤:
  1. Agent 三轮输出评分分别为 50、60、65（均不达标）

预期结果:
  - Agent 执行 3 次后停止（不超过 max_iterations）
  - 采用评分最高的一轮输出（65 分）
  - 向用户发送通知："自动审查优化已达最大迭代次数（3次），
    当前最佳评分 65/80，请人工审查"
  - SSE 推送最终事件：status="max_iterations_reached"

验证点:
  ✅ 严格不超过 max_iterations
  ✅ 用户通知正确发送
  ✅ 采用最佳结果而非最后结果
```

### TC-AR-005: simple_retry 策略

```
测试 ID:    TC-AR-005
标题:      简单重试策略不传递反馈
优先级:    P1

前置条件:
  - auto_review_enabled = true
  - iteration_strategy = "simple_retry"

步骤:
  1. Agent 首轮输出评分 60（不达标）
  2. 系统重新调度 Agent

预期结果:
  - 第二轮输入 = 原始输入（不包含 review_feedback）
  - Agent 使用原始输入重新执行

验证点:
  ✅ simple_retry 不传递反馈
  ✅ 输入与首轮完全相同
```

### TC-AR-006: scope=final 仅审查最终输出

```
测试 ID:    TC-AR-006
标题:      scope=final 时仅对工作流最终输出进行审查
优先级:    P1

前置条件:
  - auto_review_enabled = true
  - scope = "final"

步骤:
  1. 工作流执行多个步骤
  2. 中间步骤（需求理解、拆解）输出评分低

预期结果:
  - 中间步骤不触发自动审查
  - 仅最终输出（DSL 合并后）经过审查

验证点:
  ✅ 中间步骤不被审查
  ✅ 仅最终步骤触发审查迭代
```

### TC-AR-007: 使用外部审查 Agent

```
测试 ID:    TC-AR-007
标题:      配置 review_agent_id 时使用外部 Agent 审查
优先级:    P1

前置条件:
  - auto_review_enabled = true
  - review_agent_id = "agent_reviewer_001"

步骤:
  1. Agent 输出后调度审查 Agent 进行评审

预期结果:
  - 审查由 review_agent_id 指定的外部 Agent 执行（而非内置评估管线）
  - 审查 Agent 返回结构化评分和反馈
  - 反馈传递给原 Agent 进行迭代

验证点:
  ✅ 外部审查 Agent 正确调度
  ✅ 评分格式与内置管线兼容
```

---

## 6.6 功能测试用例 — 任务条目化与追溯

### TC-TI-001: 从 Agent 输出自动提取条目

```
测试 ID:    TC-TI-001
标题:      PRD 生成后自动提取需求条目
优先级:    P0

前置条件:
  - 任务已创建并分配 Agent
  - Agent 输出包含 DSL features 数组

步骤:
  1. Agent 完成 PRD 生成任务
  2. 系统自动从 output_data.features 中提取条目

预期结果:
  - 每个 feature 生成一个 task_item（category=requirement）
  - 条目 source_type = "agent_output"
  - 条目 source_ref 包含 execution_id 和 dsl_path
  - task_item_audit_trail 记录 action="created"

验证点:
  ✅ 条目数量与 features 数量一致
  ✅ 条目标题/描述正确映射
  ✅ 审计轨迹自动生成
```

### TC-TI-002: 条目分类筛选

```
测试 ID:    TC-TI-002
标题:      按分类筛选条目列表
优先级:    P0

前置条件:
  - 任务有多个条目: 3个 requirement + 2个 design + 1个 review

步骤:
  1. GET /api/v1/tasks/{id}/items?category=requirement
  2. GET /api/v1/tasks/{id}/items?category=design
  3. GET /api/v1/tasks/{id}/items (无筛选)

预期结果:
  - category=requirement → 返回 3 条
  - category=design → 返回 2 条
  - 无筛选 → 返回 6 条

验证点:
  ✅ 分类筛选正确
  ✅ 返回结果包含分类统计
```

### TC-TI-003: 条目关联与追溯链

```
测试 ID:    TC-TI-003
标题:      创建条目关联并查看追溯链
优先级:    P0

前置条件:
  - requirement 条目 A
  - design 条目 B（由 A 派生）
  - development 条目 C（实现 B）

步骤:
  1. POST 创建关联: A ──derives_from──▶ B
  2. POST 创建关联: B ──implements──▶ C
  3. GET /api/v1/items/{A}/trace
  4. POST 尝试创建循环关联: C ──derives_from──▶ A → 应被拒绝

预期结果:
  - 追溯链: A → B → C（下游方向）
  - 从 C 追溯: C → B → A（上游方向）
  - 关联关系类型正确
  - 步骤 4 返回 400 Bad Request（检测到循环引用）

验证点:
  ✅ 追溯链完整（上下游双向）
  ✅ 防止循环引用
  ✅ 深度限制（最大 10 层）
```

### TC-TI-004: 条目交付物关联

```
测试 ID:    TC-TI-004
标题:      为条目关联交付物并查看
优先级:    P0

前置条件:
  - design 条目存在
  - DSL 片段已生成

步骤:
  1. POST /api/v1/items/{itemId}/deliverables 关联 DSL 片段
  2. POST /api/v1/items/{itemId}/deliverables 关联设计文档
  3. GET /api/v1/items/{itemId}/deliverables

预期结果:
  - 返回 2 个交付物
  - deliverable_type 分别为 dsl_fragment 和 document
  - content_ref 包含正确引用信息

验证点:
  ✅ 交付物正确关联到条目
  ✅ 通过条目追溯可查到所有交付物
  ✅ 交付物版本号正确
```

### TC-TI-005: 条目审计轨迹

```
测试 ID:    TC-TI-005
标题:      条目操作产生完整审计轨迹
优先级:    P0

前置条件:
  - 条目已创建

步骤:
  1. 创建条目 → 记录 action=created
  2. 更新条目标题 → 记录 action=updated
  3. 变更状态 pending→in_progress → 记录 action=status_changed
  4. 添加关联 → 记录 action=relation_added
  5. 添加交付物 → 记录 action=deliverable_added
  6. GET /api/v1/items/{itemId}/audit-trail

预期结果:
  - 返回 5 条审计记录，按时间倒序排列
  - 每条记录包含: action, actor_type, actor_id, old_value, new_value
  - status_changed 记录有 old_value 和 new_value

验证点:
  ✅ 所有操作都有审计记录
  ✅ 变更前后值完整记录
  ✅ actor_type 正确区分 user/agent/system
```

### TC-TI-006: 条目树形结构

```
测试 ID:    TC-TI-006
标题:      条目支持层级结构（父子条目）
优先级:    P1

前置条件:
  - 父条目 A (requirement)
  - 子条目 A.1, A.2 (requirement，parent_item_id = A)

步骤:
  1. GET /api/v1/tasks/{id}/items/tree

预期结果:
  - 返回树形结构：A 包含 children [A.1, A.2]
  - 扁平条目列表中 A.1, A.2 的 parent_item_id = A.id

验证点:
  ✅ 树形结构正确嵌套
  ✅ 子条目与父条目关联正确
```

### TC-TI-007: 条目分类统计摘要

```
测试 ID:    TC-TI-007
标题:      查看任务条目分类统计
优先级:    P1

前置条件:
  - 任务有多种分类的条目

步骤:
  1. GET /api/v1/tasks/{id}/items/summary

预期结果:
  - 返回各分类的条目数量和状态分布:
    {
      "total": 10,
      "by_category": {
        "requirement": {"total": 4, "completed": 2, "pending": 2},
        "design": {"total": 3, "completed": 1, "in_progress": 2},
        "review": {"total": 3, "completed": 3}
      }
    }

验证点:
  ✅ 分类统计准确
  ✅ 状态分布正确
```

---

## 7. 集成测试用例

### TC-INT-001: 任务创建到 Agent 执行全链路

```
测试 ID:    TC-INT-001
标题:      Task → Employee → Agent 调度链路
优先级:    P0

测试链路:
  task-service 创建任务
    → Kafka: task.created
    → scheduler-service 消费
    → gRPC 调用 employee-service 查找员工
    → gRPC 调用 agent-gateway 获取 Agent
    → agent-gateway 调用外部 Agent
    → Agent 回调 agent-gateway
    → Kafka: agent.callback
    → scheduler-service 更新任务状态
    → Kafka: task.status.changed
    → interaction-service 推送 SSE

验证点:
  ✅ Kafka 消息正确生产/消费
  ✅ gRPC 调用正常
  ✅ 外部 Agent Mock 正确响应
  ✅ SSE 事件推送到前端
  ✅ 全链路耗时 <30s（不含 Agent 处理）
```

### TC-INT-002: DSL 并发写入一致性

```
测试 ID:    TC-INT-002
标题:      10 并发 DSL Patch 一致性验证
优先级:    P0

步骤:
  1. 创建一个 DSL 文档 (version=1)
  2. 10 个线程同时发送不同路径的 Patch
  3. 验证最终 DSL 内容

预期结果:
  - 所有非冲突 Patch 成功应用
  - 冲突 Patch 自动重试成功
  - 最终版本号 = 1 + 成功 Patch 数
  - DSL 内容完整无丢失

验证点:
  ✅ 乐观锁正确工作
  ✅ 无数据丢失
  ✅ 版本号单调递增
```

### TC-INT-003: Agent 熔断降级

```
测试 ID:    TC-INT-003
标题:      Agent 不可用时触发熔断和降级
优先级:    P0

步骤:
  1. 数字员工绑定 Agent A (优先级 1) 和 Agent B (优先级 2)
  2. Mock Agent A 返回 500 错误（连续 5 次）
  3. 提交任务

预期结果:
  - 前 5 次调用 Agent A → 全部失败
  - Circuit Breaker 打开
  - 自动降级到 Agent B
  - 任务执行成功
  - Agent A 30s 后半开状态，试探恢复

验证点:
  ✅ Circuit Breaker 状态: CLOSED → OPEN → HALF_OPEN
  ✅ 降级 Agent B 被正确选中
  ✅ 告警通知发送
```

### TC-INT-004: Kafka DLQ 处理

```
测试 ID:    TC-INT-004
标题:      消息消费失败进入死信队列
优先级:    P1

步骤:
  1. 发送 task.created 消息
  2. Mock scheduler-service 消费抛出异常（连续 3 次）

预期结果:
  - 重试 3 次（1s → 2s → 4s）
  - 进入 DLQ: task.created.dlt
  - 审计日志记录失败
  - 告警通知发送

验证点:
  ✅ 重试间隔正确（指数退避）
  ✅ DLQ 消息完整
  ✅ 主 Topic 消费不阻塞
```

### TC-INT-005: SSE 连接断开重连

```
测试 ID:    TC-INT-005
标题:      SSE 连接断开后自动重连并补发事件
优先级:    P1

步骤:
  1. 建立 SSE 连接
  2. 模拟网络中断 5s
  3. 恢复连接

预期结果:
  - 客户端自动重连
  - 服务端缓存最近 5min 事件
  - 重连后发送 Last-Event-ID 之后的事件
  - 无事件丢失

验证点:
  ✅ 重连成功
  ✅ 事件不重复
  ✅ 事件不丢失
```

---

## 8. 性能测试用例

### TC-PERF-001: API 延迟基准

```
测试 ID:    TC-PERF-001
标题:      核心 API P99 延迟基准测试
工具:      Apache JMeter

测试配置:
  - 并发用户: 50
  - 持续时间: 10min
  - 预热时间: 2min

测试 API 及目标:
  | API | P50 目标 | P99 目标 | TPS 目标 |
  |-----|---------|---------|---------|
  | GET /api/v1/employees | <30ms | <100ms | >200 |
  | GET /api/v1/tasks | <30ms | <100ms | >200 |
  | GET /api/v1/dsl/{id} | <50ms | <200ms | >100 |
  | PATCH /api/v1/dsl/{id} | <100ms | <500ms | >50 |
  | POST /api/v1/commands | <50ms | <200ms | >100 |
  | GET /api/v1/sse/connect | <100ms | <500ms | >100 |
  | POST /api/v1/knowledge/search | <200ms | <1000ms | >50 |
```

### TC-PERF-002: 并发用户压力测试

```
测试 ID:    TC-PERF-002
标题:      100 并发用户系统稳定性
工具:      Gatling

测试场景:
  - 100 用户同时操作
  - 每用户行为: 登录 → 创建项目 → 输入需求 → 查看结果
  - 持续时间: 30min

通过标准:
  - 错误率 <1%
  - P99 延迟 <3s
  - 系统内存无持续增长（无泄漏）
  - CPU 使用率 <80%
```

### TC-PERF-003: 大文件处理性能

```
测试 ID:    TC-PERF-003
标题:      大文件上传与解析性能
优先级:    P1

测试用例:
  | 文件类型 | 文件大小 | 目标耗时 |
  |---------|---------|---------|
  | PDF | 10MB | <30s |
  | PDF | 50MB | <120s |
  | Word | 10MB | <30s |
  | Excel | 20MB | <60s |
  | 图片(OCR) | 5MB | <60s |
```

### TC-PERF-004: DSL 大文档性能

```
测试 ID:    TC-PERF-004
标题:      大型 DSL 文档读写性能
优先级:    P1

测试用例:
  | DSL 大小 | 操作 | 目标耗时 |
  |---------|------|---------|
  | 100KB (小型) | 读取 | <50ms |
  | 100KB | Patch | <100ms |
  | 1MB (中型) | 读取 | <200ms |
  | 1MB | Patch | <500ms |
  | 10MB (大型) | 读取 | <1s |
  | 10MB | Patch | <3s |
```

---

## 9. 安全测试用例

### TC-SEC-001: 认证授权

```
测试 ID:    TC-SEC-001
标题:      认证与授权安全验证

测试项:
  1. 未登录访问 API → 401 Unauthorized
  2. 过期 Token 访问 → 401
  3. 无权限角色访问管理 API → 403 Forbidden
  4. 跨租户数据访问 → 403 (租户 A 访问租户 B 数据)
  5. Token 伪造 → 401 (签名验证失败)

验证点:
  ✅ 所有 API 均需认证（除健康检查）
  ✅ RBAC 矩阵正确执行
  ✅ 租户隔离有效
```

### TC-SEC-002: 输入验证

```
测试 ID:    TC-SEC-002
标题:      输入验证与注入防护

测试项:
  1. SQL 注入: name="'; DROP TABLE users;--" → 400 Bad Request
  2. XSS: description="<script>alert(1)</script>" → 内容被转义
  3. 命令注入: filename="test;rm -rf /" → 被拒绝
  4. 路径遍历: path="../../../etc/passwd" → 403
  5. 超长输入: 10MB 文本字段 → 413 Payload Too Large

验证点:
  ✅ 所有输入经过验证和清洗
  ✅ 无 SQL 注入漏洞
  ✅ 无 XSS 漏洞
```

### TC-SEC-003: API 限流

```
测试 ID:    TC-SEC-003
标题:      API 限流验证

步骤:
  1. 同一用户 1 分钟内发送 1001 个请求

预期结果:
  - 前 1000 个请求正常 (200)
  - 第 1001 个请求 → 429 Too Many Requests
  - Response Header: X-RateLimit-Remaining: 0

验证点:
  ✅ 限流正确触发
  ✅ 限流窗口 60s 后重置
  ✅ 不同用户独立计数
```

### TC-SEC-004: 敏感数据保护

```
测试 ID:    TC-SEC-004
标题:      敏感数据加密与脱敏

测试项:
  1. Agent API Key 存储 → 数据库中为密文
  2. 审计日志中的 API Key → 仅显示前 4 位
  3. API 响应中不包含 password 字段
  4. 日志中不输出 Token/Key

验证点:
  ✅ 数据库查询 API Key → AES-256 加密
  ✅ API 响应无敏感字段泄露
  ✅ 日志脱敏生效
```

---

## 10. 容错测试用例

### TC-FAULT-001: 数据库故障

```
测试 ID:    TC-FAULT-001
标题:      PostgreSQL 主库不可用时的系统行为

步骤:
  1. 停止 PostgreSQL 主库
  2. 观察各服务行为

预期结果:
  - 读请求降级到只读副本 (如有)
  - 写请求返回 503 Service Unavailable
  - 健康检查端点报告 DOWN
  - 告警通知发送
  - 数据库恢复后 → 系统自动恢复

验证点:
  ✅ 连接池正确处理断连
  ✅ 无数据损坏
  ✅ 恢复后无需人工干预
```

### TC-FAULT-002: Kafka 不可用

```
测试 ID:    TC-FAULT-002
标题:      Kafka 集群不可用时的系统行为

步骤:
  1. 停止 Kafka 服务
  2. 提交任务

预期结果:
  - 事件发布失败被捕获
  - 事件暂存本地队列（内存/文件）
  - Kafka 恢复后自动重发
  - 不影响同步 API 响应

验证点:
  ✅ 生产者异常正确处理
  ✅ 本地缓存不丢失
  ✅ 恢复后消息正确发送
```

### TC-FAULT-003: Agent 响应超时

```
测试 ID:    TC-FAULT-003
标题:      Agent 执行超时处理

步骤:
  1. Mock Agent 延迟 600s 响应（超过 300s 超时）

预期结果:
  - 300s 后任务标记超时
  - 通知用户
  - 尝试备选 Agent（如有）
  - Temporal Activity 正确处理超时

验证点:
  ✅ 超时正确触发
  ✅ 资源正确释放
  ✅ 工作流状态正确更新
```

### TC-FAULT-004: Redis 不可用

```
测试 ID:    TC-FAULT-004
标题:      Redis 不可用时的降级行为

步骤:
  1. 停止 Redis
  2. 正常操作系统

预期结果:
  - 缓存 miss → 直接查数据库
  - 分布式锁降级 → 数据库锁
  - SSE 跨实例推送降级 → 仅本实例推送
  - 性能下降但功能可用

验证点:
  ✅ 系统不崩溃
  ✅ 功能正常（性能降级）
  ✅ Redis 恢复后缓存自动重建
```

### TC-FAULT-005: 网络分区

```
测试 ID:    TC-FAULT-005
标题:      微服务间网络分区处理

步骤:
  1. 断开 scheduler-service 与 agent-gateway 网络

预期结果:
  - gRPC 调用超时 (5s)
  - 重试 3 次
  - 标记 Agent 不可达
  - 任务进入等待队列
  - 网络恢复后重试成功

验证点:
  ✅ 超时正确
  ✅ 重试策略正确
  ✅ 不阻塞其他任务
```

---

## 11. 验收测试矩阵

### 各场景验收标准

| 场景 | Phase | 验收标准 | 测试用例 |
|------|-------|---------|---------|
| S1 新需求生成 | 1 | 输入→分析→澄清→拆解→PRD→流程→评审 全流程可用 | TC-S1-001~005 |
| S7 局部修改 | 1 | 选中→指令→Agent执行→DSL增量更新→实时同步 | TC-S7-001~003 |
| S8 需求评审 | 1 | 提交→Agent评审→评分→改进建议 | TC-S8-001 |
| S2 原型驱动 | 2 | 上传原型→解析→DSL pages→确认→PRD | TC-S2-001 |
| S6 需求迭代 | 2 | 现有DSL→新需求→差异→增量更新 | TC-S6-001 |
| S3 HTML 反推 | 3 | URL/HTML→解析→页面结构→推断需求 | TC-S3-001 |
| S4 流程驱动 | 3 | 绘制流程→推断功能→生成需求 | TC-S4-001 |
| S5 存量升级 | 4 | 系统信息→并行分析→画像→升级方案 | TC-S5-001 |

### 非功能验收标准

| 指标 | 目标值 | 测试用例 |
|------|--------|---------|
| API P99 延迟 | <500ms | TC-PERF-001 |
| 并发用户 | 100 | TC-PERF-002 |
| 系统错误率 | <0.5% | TC-PERF-002 |
| Agent 成功率 | >95% | TC-INT-003 |
| 安全漏洞 | 0 个高危 | TC-SEC-001~004 |
| 系统可用性 | >99.5% | TC-FAULT-001~005 |
| 数据一致性 | 100% | TC-INT-002 |
