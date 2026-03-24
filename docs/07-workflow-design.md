# 工作流与编排设计

## 1. 编排引擎架构

### 1.1 Temporal 工作流引擎

```
┌──────────────────────────────────────────────────┐
│                Temporal Server                    │
│                                                  │
│  ┌────────────┐  ┌────────────┐                 │
│  │ Workflow    │  │  Activity   │                 │
│  │ Definition  │  │ Definition  │                 │
│  └──────┬─────┘  └──────┬─────┘                 │
│         │               │                        │
│  ┌──────▼───────────────▼─────┐                 │
│  │     Workflow Engine         │                 │
│  │  ·状态管理  ·重试  ·超时    │                 │
│  │  ·信号处理  ·查询  ·版本控制│                 │
│  └─────────────────────────────┘                 │
└──────────────────────────────────────────────────┘
```

### 1.2 选择 Temporal 的理由

| 特性 | 说明 |
|------|------|
| 持久化执行 | 工作流状态自动持久化，进程重启后可恢复 |
| 信号机制 | 支持外部信号（用户回答问题、指令中断） |
| 活动重试 | Activity 失败自动重试，支持自定义策略 |
| 版本控制 | 工作流定义可安全升级，不影响运行中的实例 |
| 可观测 | 内置 Web UI 查看工作流执行状态 |

## 2. 核心工作流定义

### 2.1 新需求生成工作流（S1）

```
Workflow: NewRequirementWorkflow

触发: 用户提交新需求

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  [开始] ──▶ [输入解析] ──▶ [需求理解] ──▶ [需求澄清]        │
│                                              │              │
│                                    ┌─────────┤              │
│                                    ▼         │              │
│                             [等待用户回答]    │(无问题)      │
│                                    │         │              │
│                                    ▼         ▼              │
│                              [合并答案] ──▶ [需求拆解]      │
│                                              │              │
│                                    ┌─────────┴────────┐     │
│                                    ▼                  ▼     │
│                              [PRD 生成]         [流程设计]   │
│                                    │                  │     │
│                                    └────────┬─────────┘     │
│                                             ▼               │
│                                       [DSL 同步]            │
│                                             │               │
│                                             ▼               │
│                                       [需求评审]            │
│                                             │               │
│                                    ┌────────┤               │
│                                    ▼        │               │
│                              (有问题)  (通过) │               │
│                              回到澄清   ▼    │               │
│                                       [完成]  │               │
│                                             │               │
│  ◀══════ 指令队列信号 ══════════════════════╝               │
│  (任何阶段都可以接收用户指令信号，中断当前流程)                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Temporal Workflow 伪代码**：

```java
@WorkflowInterface
public interface NewRequirementWorkflow {

    @WorkflowMethod
    WorkflowResult execute(RequirementInput input);

    @SignalMethod
    void onUserAnswer(UserAnswer answer);

    @SignalMethod
    void onCommand(UserCommand command);

    @SignalMethod
    void onCancel();

    @QueryMethod
    WorkflowStatus getStatus();
}

@WorkflowImpl
public class NewRequirementWorkflowImpl implements NewRequirementWorkflow {

    @Override
    public WorkflowResult execute(RequirementInput input) {
        // 1. 输入解析
        ParsedInput parsed = activities.parseInput(input);

        // 2. 需求理解
        UnderstandingResult understanding = activities.understandRequirement(parsed);

        // 3. 需求澄清（可能多轮）
        ClarificationResult clarification = activities.clarifyRequirement(understanding);

        while (clarification.hasQuestions()) {
            // 发送问题给前端
            activities.sendQuestions(clarification.getQuestions());

            // 等待用户回答（Temporal Signal）
            Workflow.await(() -> this.userAnswers != null);

            // 合并答案，继续澄清
            clarification = activities.clarifyWithAnswers(clarification, this.userAnswers);
            this.userAnswers = null;
        }

        // 4. 需求拆解
        DecompositionResult decomposition = activities.decomposeRequirement(clarification);

        // 5. 并行执行: PRD 生成 + 流程设计
        Promise<PrdResult> prdPromise = Async.function(
            activities::generatePrd, decomposition);
        Promise<FlowResult> flowPromise = Async.function(
            activities::designFlow, decomposition);

        PrdResult prd = prdPromise.get();
        FlowResult flow = flowPromise.get();

        // 6. DSL 同步
        DslResult dsl = activities.synchronizeDsl(prd, flow);

        // 7. 需求评审
        ReviewResult review = activities.reviewRequirement(dsl);

        return new WorkflowResult(dsl, review);
    }
}
```

### 2.2 局部修改工作流（S7）

```
Workflow: CommandExecutionWorkflow

触发: 用户提交修改指令

[接收指令] ──▶ [解析目标] ──▶ [加载 DSL] ──▶ [Agent 执行修改]
                                                    │
                                                    ▼
                                            [更新 DSL]
                                                    │
                                                    ▼
                                            [同步 Agent]
                                            (更新 PRD/原型/流程)
                                                    │
                                                    ▼
                                            [推送更新]
                                            (SSE → 前端)
```

### 2.3 原型驱动工作流（S2）

```
Workflow: PrototypeDrivenWorkflow

触发: 用户上传原型/HTML/截图

[接收原型] ──▶ [原型解析 Agent]
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
      [页面识别] [组件识别] [字段识别]
          │         │         │
          └─────────┼─────────┘
                    ▼
            [生成 DSL pages]
                    │
                    ▼
            [用户确认/修改]
                    │
                    ▼
            [PRD 生成 Agent]
                    │
                    ▼
            [DSL 同步]
```

### 2.4 存量系统升级工作流（S5）

```
Workflow: LegacyUpgradeWorkflow

触发: 用户提供存量系统信息

[接收系统信息] ──▶ [存量系统分析 Agent]
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
          [代码分析] [DB分析] [接口分析]
              │         │         │
              └─────────┼─────────┘
                        ▼
                [系统画像生成]
                        │
                        ▼
                [用户确认画像]
                        │
                        ▼
                [升级需求生成]
                        │
                        ▼
                [影响范围分析]
                        │
                        ▼
                [改造方案输出]
```

### 2.5 需求迭代工作流（S6）

```
Workflow: RequirementIterationWorkflow

触发: 新迭代的需求变更

[加载历史 DSL] ──▶ [接收新需求/变更]
                        │
                        ▼
                [差异分析 Agent]
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
          [新增功能] [修改功能] [删除功能]
              │         │         │
              └─────────┼─────────┘
                        ▼
                [生成变更 DSL]
                        │
                        ▼
                [影响模块分析]
                        │
                        ▼
                [变更 PRD 生成]
                        │
                        ▼
                [评审与确认]
```

## 3. 指令队列编排

### 3.1 指令队列处理流程

```
┌────────────────────────────────────────────────────────┐
│                  指令队列处理器                          │
│                                                        │
│  用户指令 ──▶ [入队]                                    │
│                 │                                      │
│                 ▼                                      │
│          [优先级排序]                                    │
│          high > normal > low                           │
│                 │                                      │
│                 ▼                                      │
│          [是否中断当前?]                                 │
│           │           │                                │
│         是│         否│                                │
│           ▼           ▼                                │
│    [暂停当前任务]  [等待当前完成]                         │
│           │           │                                │
│           └─────┬─────┘                                │
│                 ▼                                      │
│          [获取锁 (DSL)]                                 │
│                 │                                      │
│                 ▼                                      │
│          [加载 DSL]                                     │
│                 │                                      │
│                 ▼                                      │
│          [分配 Agent]                                   │
│                 │                                      │
│                 ▼                                      │
│          [执行指令]                                     │
│                 │                                      │
│          ┌──────┴──────┐                               │
│          ▼             ▼                               │
│       [成功]        [失败]                              │
│          │             │                               │
│          ▼             ▼                               │
│    [更新 DSL]    [错误处理]                              │
│          │             │                               │
│          ▼             │                               │
│    [同步 Agent]        │                               │
│          │             │                               │
│          ▼             ▼                               │
│    [推送结果]    [推送错误]                              │
│          │             │                               │
│          └─────┬───────┘                               │
│                ▼                                       │
│          [释放锁]                                       │
│                │                                       │
│                ▼                                       │
│          [处理下一条]                                    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 3.2 指令优先级与中断

```
优先级规则:
├── high   → 立即执行，可中断 normal/low 任务
├── normal → 按顺序执行
└── low    → 空闲时执行

中断策略:
├── 当前任务为幂等 Skill → 直接中断，稍后重试
├── 当前任务为非幂等 Skill → 等待当前步骤完成后中断
└── 中断后保存上下文 → 恢复时从中断点继续
```

## 4. 事件驱动架构

### 4.1 Kafka Topic 设计

| Topic | 说明 | 生产者 | 消费者 |
|-------|------|--------|--------|
| `task.created` | 任务创建事件 | 任务管理服务 | 调度器 |
| `task.status.changed` | 任务状态变更 | 调度器 | 任务管理服务、审计服务 |
| `agent.execution.started` | Agent 执行开始 | Agent Gateway | 监控服务 |
| `agent.execution.completed` | Agent 执行完成 | Agent Gateway | 调度器、任务管理 |
| `agent.callback` | Agent 异步回调 | Agent | Agent Gateway |
| `dsl.changed` | DSL 变更事件 | DSL 管理服务 | 同步 Agent、审计服务 |
| `command.submitted` | 指令提交事件 | 指令队列服务 | 调度器 |
| `question.created` | 问题创建事件 | Agent | SSE 推送服务 |
| `question.answered` | 问题回答事件 | 用户服务 | Agent Gateway |
| `knowledge.updated` | 知识更新事件 | 知识管理 | 向量索引服务 |
| `audit.event` | 审计事件 | 全部服务 | 审计日志服务 |

### 4.2 事件消息格式

```json
{
  "event_id": "evt_uuid",
  "event_type": "task.status.changed",
  "source": "orchestration-service",
  "timestamp": "2025-01-01T10:30:00Z",
  "trace_id": "trace_xxx",
  "data": {
    "task_id": "task_123",
    "old_status": "running",
    "new_status": "completed",
    "metadata": {}
  }
}
```

## 5. 并发控制

### 5.1 DSL 修改的乐观锁

```
修改 DSL 时:
1. 读取 DSL 及 version
2. 执行修改
3. 更新 DSL，WHERE version = 读取时的版本
4. 如果更新失败（版本冲突）→ 重新读取并合并

适用场景: 用户手动编辑 DSL
```

### 5.2 指令队列的分布式锁

```
执行指令时:
1. 获取 Redis 分布式锁: lock:dsl:{dsl_id}
2. 锁的超时时间 = Agent 最大超时 + 缓冲
3. 执行完成后释放锁
4. 获取锁失败 → 指令继续排队

适用场景: 指令队列自动执行
```

## 6. 容错与恢复

### 6.1 Temporal 重试策略

```java
RetryOptions retryOptions = RetryOptions.newBuilder()
    .setInitialInterval(Duration.ofSeconds(1))
    .setMaximumInterval(Duration.ofMinutes(1))
    .setBackoffCoefficient(2.0)
    .setMaximumAttempts(3)
    .setDoNotRetry(
        IllegalArgumentException.class.getName(),    // 参数错误不重试
        DslValidationException.class.getName()       // 验证错误不重试
    )
    .build();
```

### 6.2 Agent 不可用降级

```
降级策略:
├── 首选 Agent 不可用 → 匹配备选 Agent
├── 所有 Agent 不可用 → 任务进入等待队列
├── 等待超过阈值 → 通知管理员
└── 支持手动指定 Agent 重试
```
