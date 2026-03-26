# 详细设计与迭代优化方案

> 基于概要设计（01-11 号文档），对各微服务内部逻辑、关键算法、容错机制、性能优化和安全设计进行详细展开，确保开发团队可以直接进入编码实现。

## 1. 各微服务详细设计

### 1.1 员工服务（employee-service）

#### 1.1.1 内部模块划分

```
employee-service/
├── controller/          # REST API 入口
│   ├── EmployeeController        # 数字员工 CRUD
│   ├── AgentBindingController    # Agent 绑定管理
│   └── WorkflowTemplateController # 工作流模板关联
├── service/             # 业务逻辑
│   ├── EmployeeService           # 员工生命周期管理
│   ├── AgentBindingService       # Agent 绑定优先级与能力匹配
│   ├── WorkflowTemplateService   # 工作流模板 CRUD
│   └── PerformanceService        # 绩效统计聚合
├── repository/          # 数据访问
│   ├── DigitalEmployeeRepository
│   ├── EmployeeAgentBindingRepository
│   └── EmployeeWorkflowTemplateRepository
├── domain/              # 领域模型
│   ├── DigitalEmployee
│   ├── EmployeeAgentBinding
│   └── EmployeeWorkflowTemplate
├── event/               # 事件发布
│   └── EmployeeEventPublisher    # Kafka 事件
└── grpc/                # gRPC 服务端
    └── EmployeeGrpcService       # 供 scheduler-service 调用
```

#### 1.1.2 核心类设计

```java
// 领域模型
public class DigitalEmployee {
    Long id;
    Long tenantId;
    String name;
    String role;           // business_analyst / developer / tester / devops / custom
    String description;
    String avatarUrl;
    String status;         // active / inactive / suspended
    JsonNode knowledgeScope;
    JsonNode config;       // 超时、重试等工作参数
    JsonNode performance;  // 绩效快照缓存
}

// 服务层 - Agent 绑定匹配
public class AgentBindingService {
    /**
     * 根据能力需求找到最优 Agent
     * 1. 筛选 employee 的所有 active 绑定
     * 2. 匹配 capability
     * 3. 按 priority ASC 排序
     * 4. 检查 Agent 健康状态（通过 agent-gateway gRPC）
     * 5. 返回第一个健康的 Agent
     */
    public Agent findBestAgent(Long employeeId, String capability) {
        List<EmployeeAgentBinding> bindings = bindingRepo
            .findByEmployeeIdAndCapabilityAndStatus(employeeId, capability, "active");
        bindings.sort(Comparator.comparingInt(b -> b.getPriority()));
        for (EmployeeAgentBinding binding : bindings) {
            AgentHealthResponse health = agentGatewayClient.checkHealth(binding.getAgentId());
            if (health.isHealthy()) {
                return agentGatewayClient.getAgent(binding.getAgentId());
            }
        }
        throw new NoAvailableAgentException(employeeId, capability);
    }
}
```

#### 1.1.3 API 契约示例

```
POST /api/v1/employees
Request:
{
  "name": "BA 小智",
  "role": "business_analyst",
  "description": "负责需求分析、PRD 生成的 BA 数字员工",
  "config": {
    "timeout_seconds": 300,
    "max_retries": 3
  }
}
Response:
{
  "code": 200,
  "data": {
    "id": 1001,
    "name": "BA 小智",
    "role": "business_analyst",
    "status": "active",
    "created_at": "2025-07-01T10:00:00Z"
  }
}

POST /api/v1/employees/{id}/agents
Request:
{
  "agent_id": 501,
  "capability": "requirement_analysis",
  "priority": 1
}
```

#### 1.1.4 数据库索引设计

```sql
-- 高频查询: 按租户+角色查找活跃员工
CREATE INDEX idx_employee_tenant_role ON digital_employees(tenant_id, role) WHERE status = 'active';

-- 高频查询: 按员工+能力查找绑定
CREATE INDEX idx_binding_emp_cap ON employee_agent_bindings(employee_id, capability) WHERE status = 'active';

-- 绩效查询: 按时间范围
CREATE INDEX idx_employee_created ON digital_employees(created_at DESC);
```

#### 1.1.5 缓存策略

```
Redis Key 设计:
  employee:{id}              → 员工详情 JSON, TTL=30min
  employee:{id}:bindings     → Agent 绑定列表, TTL=5min
  employee:{id}:performance  → 绩效统计, TTL=10min
  employee:role:{role}:list  → 角色员工列表, TTL=5min

失效策略:
  - 员工更新 → 删除 employee:{id} + employee:role:*:list
  - 绑定变更 → 删除 employee:{id}:bindings
  - 任务完成 → 删除 employee:{id}:performance
```

---

### 1.2 任务服务（task-service）

#### 1.2.1 内部模块划分

```
task-service/
├── controller/
│   ├── TaskController            # 任务 CRUD
│   ├── CommandQueueController    # 指令队列管理
│   └── TaskGraphController       # Task Graph DAG 管理
├── service/
│   ├── TaskService               # 任务生命周期
│   ├── CommandQueueService       # 指令队列优先级排序
│   ├── TaskGraphService          # DAG 拓扑排序与执行
│   └── TaskEventService          # 事件发布
├── repository/
│   ├── TaskRepository
│   ├── CommandQueueRepository
│   └── TaskDependencyRepository
├── domain/
│   ├── Task
│   ├── CommandQueue
│   └── TaskDependency
├── queue/                # 指令队列核心
│   ├── PriorityCommandQueue      # 优先级队列实现
│   └── CommandInterruptHandler   # 中断处理
└── kafka/
    ├── TaskEventProducer
    └── TaskStatusConsumer
```

#### 1.2.2 指令队列优先级算法

```java
public class PriorityCommandQueue {
    // 优先级权重: high=100, normal=50, low=10
    // 同优先级按提交时间 FIFO
    private static final Map<String, Integer> PRIORITY_WEIGHTS = Map.of(
        "high", 100, "normal", 50, "low", 10
    );

    /**
     * 入队逻辑
     * 1. 计算优先级分数 = weight * 1000 + (MAX_TIMESTAMP - submit_time)
     * 2. 写入 Redis Sorted Set: command_queue:{project_id}, score=优先级分数
     * 3. 发布 Kafka 事件: command.submitted
     * 4. 如果是 high 且当前有 normal/low 执行中 → 发送中断信号
     */
    public void enqueue(Command command) {
        int weight = PRIORITY_WEIGHTS.get(command.getPriority());
        double score = weight * 1_000_000_000L + (Long.MAX_VALUE - System.currentTimeMillis());
        redisTemplate.opsForZSet().add(
            "command_queue:" + command.getProjectId(),
            command.getId().toString(), score
        );
        kafkaTemplate.send("command.submitted", command);

        if ("high".equals(command.getPriority())) {
            interruptHandler.checkAndInterrupt(command.getProjectId());
        }
    }

    /**
     * 出队逻辑
     * 1. ZPOPMAX 获取最高优先级指令
     * 2. 设置分布式锁: command_lock:{command_id}, TTL=600s
     * 3. 返回 Command 对象
     */
    public Command dequeue(Long projectId) {
        Set<ZSetOperations.TypedTuple<String>> result = redisTemplate
            .opsForZSet()
            .reverseRangeWithScores("command_queue:" + projectId, 0, 0);
        // ... 获取并加锁
    }
}
```

#### 1.2.3 数据库索引

```sql
-- 任务查询: 按项目+状态
CREATE INDEX idx_task_project_status ON tasks(project_id, status);
CREATE INDEX idx_task_employee ON tasks(employee_id) WHERE status IN ('pending','running');

-- 指令队列: 按状态+优先级
CREATE INDEX idx_cmd_status_priority ON command_queue(status, priority DESC, created_at ASC);

-- 任务依赖: 快速查找前驱/后继
CREATE INDEX idx_dep_task ON task_dependencies(task_id);
CREATE INDEX idx_dep_depends ON task_dependencies(depends_on);
```

---

### 1.3 项目服务（project-service）

#### 1.3.1 内部模块划分

```
project-service/
├── controller/
│   ├── ProjectController         # 项目 CRUD
│   ├── IterationController       # 迭代管理
│   ├── TenantController          # 租户管理
│   └── UserController            # 用户管理（同步 Keycloak）
├── service/
│   ├── ProjectService
│   ├── IterationService
│   ├── TenantService
│   ├── UserSyncService           # 与 Keycloak 同步用户数据
│   └── ProjectMemberService      # 项目成员关系
├── repository/
├── domain/
└── integration/
    └── KeycloakClient            # Keycloak Admin API 集成
```

#### 1.3.2 租户隔离策略

```java
// 基于 Spring Security + Hibernate Filter 实现行级租户隔离
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = Long.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
@Entity
public class Project {
    @Column(name = "tenant_id", nullable = false)
    private Long tenantId;
}

// 请求拦截器自动设置租户上下文
@Component
public class TenantInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, ...) {
        Long tenantId = extractTenantFromToken(request);
        TenantContext.set(tenantId);
        entityManager.unwrap(Session.class)
            .enableFilter("tenantFilter")
            .setParameter("tenantId", tenantId);
        return true;
    }
}
```

---

### 1.4 调度服务（scheduler-service）

#### 1.4.1 内部模块划分

```
scheduler-service/
├── worker/               # Temporal Worker
│   ├── NewRequirementWorkflowImpl
│   ├── CommandExecutionWorkflowImpl
│   ├── PrototypeDrivenWorkflowImpl
│   ├── LegacyUpgradeWorkflowImpl
│   └── RequirementIterationWorkflowImpl
├── activity/             # Temporal Activity
│   ├── AgentDispatchActivity     # 调用 agent-gateway 分配任务
│   ├── DslOperationActivity      # 调用 dsl-service 读写 DSL
│   ├── NotificationActivity      # 调用 notification-service
│   └── EvaluationActivity        # 调用 audit-service 评估
├── scheduler/            # 调度算法
│   ├── TaskScheduler             # Task → Employee 匹配
│   ├── AgentMatcher              # Employee → Agent 匹配
│   ├── LoadBalancer              # 负载均衡
│   └── DAGExecutor               # DAG 拓扑执行
├── collaboration/        # 多 Agent 协作
│   ├── ChainCollaborator
│   ├── FanoutCollaborator
│   ├── VotingCollaborator
│   └── DelegationCollaborator
└── kafka/
    ├── TaskCreatedConsumer        # 消费 task.created → 触发调度
    └── AgentCallbackConsumer     # 消费 agent.callback → 更新状态
```

#### 1.4.2 任务调度算法

```java
public class TaskScheduler {
    /**
     * 任务调度算法 — 三级匹配
     *
     * Level 1: 能力匹配
     *   task.taskType → 查找拥有该 capability 的数字员工
     *
     * Level 2: 负载均衡
     *   按 employee 当前执行中任务数排序（最少优先）
     *
     * Level 3: 绩效排序
     *   同等负载下，按历史绩效评分降序（质量优先）
     *
     * 算法伪代码:
     */
    public DigitalEmployee schedule(Task task) {
        // 1. 能力匹配
        List<DigitalEmployee> candidates = employeeClient
            .findByCapability(task.getTaskType());

        if (candidates.isEmpty()) {
            throw new NoMatchingEmployeeException(task.getTaskType());
        }

        // 2. 过滤: 仅 active 状态且未达并发上限
        candidates = candidates.stream()
            .filter(e -> "active".equals(e.getStatus()))
            .filter(e -> getRunningTaskCount(e.getId()) < e.getMaxConcurrency())
            .collect(toList());

        // 3. 排序: 负载升序 → 绩效降序
        candidates.sort(
            Comparator.comparingInt((DigitalEmployee e) -> getRunningTaskCount(e.getId()))
                .thenComparing((DigitalEmployee e) -> e.getPerformanceScore(),
                    Comparator.reverseOrder())
        );

        return candidates.get(0);
    }

    private int getRunningTaskCount(Long employeeId) {
        // 从 Redis 缓存读取: employee:{id}:running_count
        return redisTemplate.opsForValue()
            .get("employee:" + employeeId + ":running_count");
    }
}
```

#### 1.4.3 DAG 执行引擎

```java
public class DAGExecutor {
    /**
     * 基于拓扑排序执行 Task Graph
     *
     * 1. 构建邻接表
     * 2. 计算入度
     * 3. BFS 执行:
     *    - 入度=0 的节点 → 并行提交给 Temporal
     *    - 节点完成 → 更新后继节点入度
     *    - 后继入度降为 0 → 加入就绪队列
     * 4. 检测环路（节点数 ≠ 已执行数 → 有环）
     */
    public void execute(TaskGraph graph) {
        Map<Long, Integer> inDegree = calculateInDegree(graph);
        Queue<Long> readyQueue = new LinkedList<>();

        // 初始就绪节点
        inDegree.forEach((taskId, degree) -> {
            if (degree == 0) readyQueue.offer(taskId);
        });

        int executedCount = 0;
        while (!readyQueue.isEmpty()) {
            // 并行启动所有就绪任务
            List<Long> batch = new ArrayList<>();
            while (!readyQueue.isEmpty()) {
                batch.add(readyQueue.poll());
            }

            // 并行提交 Temporal Workflow
            List<CompletableFuture<Void>> futures = batch.stream()
                .map(taskId -> CompletableFuture.runAsync(() ->
                    temporalClient.startWorkflow(taskId)))
                .collect(toList());

            // 等待本批完成
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
            executedCount += batch.size();

            // 更新后继入度
            for (Long taskId : batch) {
                for (Long successor : graph.getSuccessors(taskId)) {
                    int newDegree = inDegree.merge(successor, -1, Integer::sum);
                    if (newDegree == 0) readyQueue.offer(successor);
                }
            }
        }

        if (executedCount != graph.getNodeCount()) {
            throw new CyclicDependencyException("Task Graph contains cycles");
        }
    }
}
```

---

### 1.5 DSL 服务（dsl-service）

#### 1.5.1 内部模块划分

```
dsl-service/
├── controller/
│   ├── DslDocumentController     # DSL 文档 CRUD
│   ├── DslVersionController      # 版本管理
│   └── DslExportController       # 导出（PRD/Markdown/PDF）
├── service/
│   ├── DslDocumentService        # 文档 CRUD + JSONB 操作
│   ├── DslPatchService           # JSON Patch 应用与冲突检测
│   ├── DslVersionService         # 版本快照、回滚、diff
│   ├── DslValidationService      # DSL 结构验证
│   ├── DslExportService          # 导出为 PRD/Markdown
│   └── DslSyncService            # PRD ⇄ 原型 ⇄ 流程图同步
├── patch/                # DSL Patch 核心
│   ├── JsonPatchEngine           # RFC 6902 JSON Patch
│   ├── ConflictDetector          # 冲突检测
│   ├── MergeResolver             # 自动合并/手动冲突
│   └── DslDiffCalculator         # 版本 Diff
├── mcp/                  # MCP Server 工具
│   ├── DslReadTool               # read_dsl
│   ├── DslPatchTool              # patch_dsl
│   ├── DslValidateTool           # validate_dsl
│   └── DslExportTool             # export_dsl
└── repository/
    ├── DslDocumentRepository
    └── DslVersionRepository
```

#### 1.5.2 DSL Patch 算法（详细设计）

```java
/**
 * DSL Patch 算法 — 基于 RFC 6902 JSON Patch
 *
 * 核心流程:
 * 1. 接收 Patch 操作列表 (op: add/remove/replace/move/copy/test)
 * 2. 加载当前 DSL 版本 + 乐观锁 version 号
 * 3. 校验 Patch 合法性（路径存在、类型匹配）
 * 4. 冲突检测（并发修改同一路径）
 * 5. 应用 Patch → 生成新 DSL
 * 6. 验证新 DSL 结构完整性
 * 7. 保存 + 递增版本号
 * 8. 发布 dsl.changed 事件
 */
public class DslPatchService {

    @Transactional
    public DslDocument applyPatch(Long dslId, List<JsonPatchOperation> operations, int expectedVersion) {
        // 1. 加载并检查乐观锁
        DslDocument doc = dslRepo.findByIdForUpdate(dslId)
            .orElseThrow(() -> new DslNotFoundException(dslId));

        if (doc.getVersion() != expectedVersion) {
            throw new DslVersionConflictException(dslId, expectedVersion, doc.getVersion());
        }

        // 2. 冲突检测 — 检查近 5 秒内是否有其他 Patch 修改了相同路径
        List<String> conflictPaths = conflictDetector.detectConflicts(
            dslId, operations, Duration.ofSeconds(5));
        if (!conflictPaths.isEmpty()) {
            throw new DslPatchConflictException(conflictPaths);
        }

        // 3. 应用 JSON Patch
        JsonNode currentDsl = doc.getContent();
        JsonNode patchedDsl = jsonPatchEngine.apply(currentDsl, operations);

        // 4. 结构验证
        ValidationResult validation = dslValidationService.validate(patchedDsl);
        if (!validation.isValid()) {
            throw new DslValidationException(validation.getErrors());
        }

        // 5. 保存新版本
        doc.setContent(patchedDsl);
        doc.setVersion(doc.getVersion() + 1);
        doc.setDslHash(DigestUtils.sha256Hex(patchedDsl.toString()));
        dslRepo.save(doc);

        // 6. 保存版本快照
        dslVersionService.createSnapshot(doc, operations);

        // 7. 发布事件
        kafkaTemplate.send("dsl.changed", new DslChangedEvent(dslId, operations, doc.getVersion()));

        return doc;
    }
}

/**
 * 冲突检测器
 * 检测两个 Patch 是否修改了相同的 DSL 路径（或父子路径）
 */
public class ConflictDetector {

    public List<String> detectConflicts(Long dslId, List<JsonPatchOperation> newOps, Duration window) {
        // 从 Redis 获取近 N 秒内的已应用 Patch 路径
        Set<String> recentPaths = getRecentPatchPaths(dslId, window);

        List<String> conflicts = new ArrayList<>();
        for (JsonPatchOperation op : newOps) {
            String path = op.getPath();
            // 检查完全匹配或父子关系
            for (String recentPath : recentPaths) {
                if (path.startsWith(recentPath) || recentPath.startsWith(path)) {
                    conflicts.add(path + " conflicts with recent: " + recentPath);
                }
            }
        }
        return conflicts;
    }

    private Set<String> getRecentPatchPaths(Long dslId, Duration window) {
        // Redis key: dsl_patch_paths:{dslId}, 使用 Sorted Set 按时间排序
        double minScore = System.currentTimeMillis() - window.toMillis();
        return redisTemplate.opsForZSet()
            .rangeByScore("dsl_patch_paths:" + dslId, minScore, Double.MAX_VALUE);
    }
}
```

#### 1.5.3 DSL 版本 Diff 算法

```java
/**
 * 计算两个 DSL 版本间的差异
 * 返回: 新增/删除/修改的路径列表
 */
public class DslDiffCalculator {

    public DslDiff diff(JsonNode oldDsl, JsonNode newDsl) {
        DslDiff result = new DslDiff();
        diffRecursive("", oldDsl, newDsl, result);
        return result;
    }

    private void diffRecursive(String path, JsonNode old, JsonNode current, DslDiff result) {
        if (old == null && current != null) {
            result.addAddition(path, current);
            return;
        }
        if (old != null && current == null) {
            result.addDeletion(path, old);
            return;
        }
        if (old.isObject() && current.isObject()) {
            // 遍历所有字段
            Set<String> allKeys = new HashSet<>();
            old.fieldNames().forEachRemaining(allKeys::add);
            current.fieldNames().forEachRemaining(allKeys::add);

            for (String key : allKeys) {
                diffRecursive(path + "/" + key, old.get(key), current.get(key), result);
            }
        } else if (old.isArray() && current.isArray()) {
            // 数组比较 — 基于 id 字段匹配（DSL 元素都有 id）
            Map<String, JsonNode> oldMap = arrayToMap(old);
            Map<String, JsonNode> newMap = arrayToMap(current);

            for (String id : oldMap.keySet()) {
                if (!newMap.containsKey(id)) {
                    result.addDeletion(path + "[id=" + id + "]", oldMap.get(id));
                } else {
                    diffRecursive(path + "[id=" + id + "]", oldMap.get(id), newMap.get(id), result);
                }
            }
            for (String id : newMap.keySet()) {
                if (!oldMap.containsKey(id)) {
                    result.addAddition(path + "[id=" + id + "]", newMap.get(id));
                }
            }
        } else if (!old.equals(current)) {
            result.addModification(path, old, current);
        }
    }
}
```

---

### 1.6 Agent 网关（agent-gateway）

#### 1.6.1 内部模块划分

```
agent-gateway/
├── controller/
│   ├── AgentController           # Agent 注册/管理
│   ├── SkillController           # Skill CRUD + SKILL.md 导入
│   ├── A2AController             # A2A 协议端点
│   ├── MCPController             # MCP 工具/资源端点
│   └── CallbackController        # Agent 回调接收
├── service/
│   ├── AgentRegistryService      # Agent 注册与发现
│   ├── SkillRegistryService      # Skill 注册（OpenClaw 兼容）
│   ├── AgentDispatchService      # Agent 调用分发
│   ├── CallbackService           # 回调处理与转发
│   └── HealthCheckService        # Agent 健康检查
├── protocol/             # 协议适配器
│   ├── A2AProtocolAdapter        # A2A Task Model 适配
│   ├── MCPServerAdapter          # MCP Server 实现
│   ├── DifyApiAdapter            # Dify Workflow API 适配
│   ├── HttpRestAdapter           # 标准 HTTP REST 适配
│   └── GrpcAdapter               # gRPC 适配
├── skill/                # Skill 管理
│   ├── SkillMdParser             # SKILL.md YAML+Markdown 解析器
│   ├── SkillValidator            # OpenClaw 命名规范验证
│   └── SkillMapper               # A2A Agent Card → Skill 映射
├── observability/        # 可观测性
│   ├── LangfuseTracer            # Langfuse Trace 上报
│   └── OtelSpanDecorator         # OpenTelemetry Span 装饰
└── resilience/           # 韧性
    ├── AgentCircuitBreaker       # 每个 Agent 独立熔断器
    └── RetryPolicyManager        # 重试策略管理
```

#### 1.6.2 Agent 调用分发流程

```java
public class AgentDispatchService {
    /**
     * Agent 调用分发 — 统一入口
     *
     * 1. 根据 Agent 的 provider 类型选择适配器
     * 2. 构造标准化请求
     * 3. 通过 Circuit Breaker 调用
     * 4. 上报 Langfuse Trace
     * 5. 异步等待回调 / 同步等待响应
     */
    public AgentExecutionResult dispatch(AgentDispatchRequest request) {
        Agent agent = agentRegistry.getAgent(request.getAgentId());
        ProtocolAdapter adapter = selectAdapter(agent.getProvider());

        // Langfuse Trace 开始
        String traceId = langfuseTracer.startTrace(
            "agent_dispatch",
            Map.of("agent_id", agent.getId(), "capability", request.getCapability())
        );

        try {
            // Circuit Breaker 保护
            AgentExecutionResult result = circuitBreakers
                .get(agent.getId())
                .executeSupplier(() -> adapter.invoke(agent, request));

            langfuseTracer.endTrace(traceId, "success", result.getTokenUsage());
            return result;

        } catch (Exception e) {
            langfuseTracer.endTrace(traceId, "error", null);

            // 降级: 尝试备选 Agent
            Agent fallbackAgent = findFallbackAgent(request);
            if (fallbackAgent != null) {
                return dispatch(request.withAgent(fallbackAgent.getId()));
            }
            throw new AgentDispatchException(agent.getId(), e);
        }
    }

    private ProtocolAdapter selectAdapter(String provider) {
        return switch (provider) {
            case "dify" -> difyApiAdapter;
            case "a2a" -> a2aProtocolAdapter;
            case "self_built" -> httpRestAdapter;
            case "grpc" -> grpcAdapter;
            default -> httpRestAdapter;
        };
    }
}
```

#### 1.6.3 Circuit Breaker 配置

```yaml
# 每个 Agent 独立的熔断器配置
resilience4j:
  circuitbreaker:
    configs:
      agentDefault:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10           # 最近 10 次调用
        failureRateThreshold: 50        # 50% 失败率触发熔断
        waitDurationInOpenState: 30s    # 熔断 30s
        permittedNumberOfCallsInHalfOpenState: 3  # 半开状态试探 3 次
        slowCallRateThreshold: 80       # 80% 慢调用触发熔断
        slowCallDurationThreshold: 60s  # >60s 视为慢调用
    instances:
      # 动态创建: agent_{id} 使用 agentDefault 配置
```

---

### 1.7 知识服务（knowledge-service）

#### 1.7.1 核心流程

```
文档向量化流程:
  上传文档 → Tika 解析 → 文本分块(chunk_size=512, overlap=64)
  → Embedding(nomic-embed-text via LiteLLM)
  → 存入 Milvus(collection: knowledge_vectors)
  → 建立索引(HNSW, M=16, efConstruction=256)

检索流程:
  查询文本 → Embedding → Milvus ANN 检索(top_k=10, ef=128)
  → 按 score 阈值过滤(>0.7) → 返回文档块
```

#### 1.7.2 缓存策略

```
Redis Key:
  knowledge:search:{hash(query)}:{scope}  → 搜索结果缓存, TTL=5min
  knowledge:doc:{id}                      → 文档元数据, TTL=30min

Milvus 索引参数:
  - 索引类型: HNSW
  - M: 16 (每节点最大连接数)
  - efConstruction: 256 (构建时扩展因子)
  - ef: 128 (搜索时扩展因子)
  - metric_type: COSINE
```

---

### 1.8 交互服务（interaction-service）

#### 1.8.1 SSE 推送设计

```java
/**
 * SSE 连接管理
 * - 每个用户维护一个 SSE 连接
 * - 基于 Redis Pub/Sub 实现跨实例推送
 * - 心跳间隔: 30s
 * - 超时断开: 5min 无事件
 */
@RestController
public class SseController {

    @GetMapping(value = "/api/v1/sse/connect", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter connect(@RequestParam Long projectId, @RequestParam Long iterationId) {
        SseEmitter emitter = new SseEmitter(300_000L); // 5min 超时
        String channel = "sse:" + projectId + ":" + iterationId;

        // 注册到 Redis Pub/Sub
        redisSseManager.subscribe(channel, emitter);

        emitter.onCompletion(() -> redisSseManager.unsubscribe(channel, emitter));
        emitter.onTimeout(() -> redisSseManager.unsubscribe(channel, emitter));

        // 发送心跳
        scheduledExecutor.scheduleAtFixedRate(
            () -> emitter.send(SseEmitter.event().name("heartbeat").data("")),
            30, 30, TimeUnit.SECONDS
        );

        return emitter;
    }
}

/**
 * SSE 事件类型
 */
public enum SseEventType {
    STREAM,          // 流式文本输出
    QUESTION,        // AI 提问
    STATUS,          // 步骤状态变更
    DSL_UPDATE,      // DSL 变更通知
    COMMAND_UPDATE,  // 指令队列状态
    ERROR,           // 错误
    DONE             // 完成
}
```

---

### 1.9 审计服务（audit-service）

#### 1.9.1 评估管线详细设计

```
Agent 输出 → 评估管线 → 评估记录

Step 1: 结构化验证 (自动, <1s)
  ├── 输出是否为合法 JSON
  ├── 是否符合 output_schema
  ├── 必填字段是否完整
  └── 评分: 0-100 (通过=100, 每个缺陷 -10)

Step 2: 语义质量评估 (LLM-as-Judge, ~10s)
  ├── 调用评估 LLM (DeepSeek/Qwen)
  ├── Prompt: "请评估以下 {type} 的质量..."
  ├── 维度: 完整性/一致性/可执行性/清晰度
  └── 评分: 0-100

Step 3: 一致性检查 (自动, ~2s)
  ├── PRD 功能 vs DSL features 交叉验证
  ├── 流程节点 vs 页面引用 一致性
  ├── 实体字段 vs 页面表单 一致性
  └── 评分: 0-100

Step 4: 用户反馈 (手动, 异步)
  ├── 用户显式评分 (1-5 星)
  ├── 用户修改次数 (隐式: 修改越多评分越低)
  └── 评分: 0-100

最终评分 = w1*结构化 + w2*语义 + w3*一致性 + w4*用户反馈
默认权重: w1=0.2, w2=0.3, w3=0.3, w4=0.2
```

#### 1.9.2 自动审查优化机制详细设计

```java
/**
 * 自动审查优化 Activity 实现
 *
 * 核心逻辑:
 *   1. 接收 Agent 输出
 *   2. 运行评估管线（结构化验证 + 语义评估 + 一致性检查）
 *   3. 综合评分 < 阈值 → 生成审查反馈
 *   4. 审查反馈 + 原始输入 + 上轮输出 → 重新调度 Agent
 *   5. 重复直到达标或达最大次数
 */
public class AutoReviewActivityImpl implements AutoReviewActivity {

    @Override
    public ReviewScore evaluateOutput(
            String capability, Object output, Object originalInput,
            List<String> dimensions) {

        ReviewScore score = new ReviewScore();

        // Step 1: 结构化验证（自动, <1s）
        SchemaValidationResult schemaResult = schemaValidator.validate(
            output, agentRegistry.getOutputSchema(capability));
        score.setSchemaScore(schemaResult.getScore());

        // Step 2: 语义质量评估（LLM-as-Judge, ~10s）
        if (dimensions.contains("completeness") || dimensions.contains("clarity")) {
            SemanticEvalResult semanticResult = llmJudge.evaluate(
                output, originalInput, capability, dimensions);
            score.setSemanticScore(semanticResult.getScore());
            score.setIssues(semanticResult.getIssues());
        }

        // Step 3: 一致性检查（自动, ~2s）
        if (dimensions.contains("consistency")) {
            ConsistencyResult consistencyResult = consistencyChecker.check(
                output, originalInput);
            score.setConsistencyScore(consistencyResult.getScore());
            score.addIssues(consistencyResult.getIssues());
        }

        // 综合评分
        score.calculateOverall();  // 加权计算
        return score;
    }

    @Override
    public ReviewFeedback generateReviewFeedback(
            String capability, Object output, ReviewScore score) {

        // 基于评估结果生成结构化改进建议
        // 可选: 调用 LLM 生成自然语言反馈
        ReviewFeedback feedback = new ReviewFeedback();
        feedback.setOverallScore(score.getOverallScore());

        for (ReviewIssue issue : score.getIssues()) {
            feedback.addSuggestion(new Suggestion(
                issue.getDimension(),
                issue.getDescription(),
                issue.generateSuggestion()  // 自动生成改进建议
            ));
        }

        return feedback;
    }
}
```

**自动审查与手动审查的关系**：

```
自动审查优化 (Auto-Review):
  ├── 时机: Agent 每轮输出后立即执行
  ├── 目的: 自动迭代改进，减少人工干预
  ├── 执行者: 评估管线（自动）或审查 Agent（外部）
  └── 结果: 通过 → 继续 / 不通过 → 自动迭代

手动需求评审 (S8):
  ├── 时机: 工作流完成后由用户触发
  ├── 目的: 人工验收最终产出
  ├── 执行者: 评审 Agent（外部，由用户选择）
  └── 结果: 通过 → 完成 / 不通过 → 用户指令修改

两者互补: 自动审查保证每步输出基本质量 → 手动评审确认最终产出满足业务需求
```

#### 1.9.3 审计日志表分区

```sql
-- 审计日志按月分区（高写入量）
CREATE TABLE audit_logs (
    id          BIGSERIAL,
    tenant_id   BIGINT NOT NULL,
    action      VARCHAR(100) NOT NULL,
    resource    VARCHAR(100) NOT NULL,
    resource_id BIGINT,
    actor_id    BIGINT NOT NULL,
    details     JSONB,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 自动创建月度分区
CREATE TABLE audit_logs_2025_07 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');
CREATE TABLE audit_logs_2025_08 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-08-01') TO ('2025-09-01');

-- 索引
CREATE INDEX idx_audit_tenant_time ON audit_logs(tenant_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_logs(resource, resource_id);
```

---

### 1.10 文件服务（file-service）

#### 1.10.1 文件处理流程

```
上传 → 存储(MinIO) → 异步解析 → 缓存解析结果

支持格式:
  PDF  → Apache Tika → 文本提取
  Word → Apache Tika → 文本+结构提取
  Excel → Apache Tika → 表格数据提取
  图片 → PaddleOCR → 文字识别
  HTML → Jsoup → DOM 解析

文件大小限制:
  单文件: 100MB
  总量/项目: 10GB

MinIO Bucket 设计:
  bucket: aitaskos-{tenant_id}
  path: {project_id}/{type}/{filename}
  type: uploads / artifacts / exports
```

---

### 1.11 通知服务（notification-service）

#### 1.11.1 通知渠道

```
Kafka 消费 → 路由 → 发送

Topic: notification.send
Message:
{
  "type": "task_completed",       // 通知类型
  "channel": ["email", "dingtalk"], // 发送渠道
  "recipients": [101, 102],       // 用户 ID
  "template": "task_completed",    // 模板名
  "variables": {                   // 模板变量
    "task_name": "需求分析",
    "employee_name": "BA 小智",
    "result": "success"
  }
}

渠道适配器:
  - EmailSender: Spring Mail + SMTP
  - DingTalkSender: DingTalk Webhook API
  - WeComSender: 企业微信 API
  - InAppSender: 写入 DB + SSE 推送
```

---

## 2. Agent 协作模式详细设计

### 2.1 串行链模式（Chain）

```
适用场景: 需求理解 → 需求拆解 → PRD 生成

执行流程:
  Agent A (需求理解)
    │ 输出: 需求摘要
    ▼
  Agent B (需求拆解)
    │ 输入: Agent A 输出 + 原始需求
    │ 输出: 功能列表
    ▼
  Agent C (PRD 生成)
    │ 输入: Agent A + B 输出
    │ 输出: PRD DSL

错误处理:
  - 任一节点失败 → 整条链终止
  - 支持从失败节点重试（不重复执行已成功节点）
  - 超时: 单节点 300s, 整条链 900s

数据传递:
  - 每个节点输出存入 TaskContext
  - 后续节点可访问所有前驱节点输出
  - 格式: { "step_1_output": {...}, "step_2_output": {...} }
```

### 2.2 并行扇出模式（Fanout）

```
适用场景: PRD 生成 + 流程设计 并行执行

执行流程:
  输入数据
    ├──▶ Agent A (PRD 生成)     ─── 输出 A
    └──▶ Agent B (流程设计)     ─── 输出 B
              │
              ▼
         合并 (Merge)
              │
              ▼
         DSL 同步

合并策略:
  - 默认: 将所有输出合并到 DSL 不同路径
    - Agent A → dsl.features + dsl.pages
    - Agent B → dsl.flows
  - 冲突: 如果修改了相同路径 → 按优先级选择

错误处理:
  - 部分失败: 成功的部分保留, 失败的标记并通知用户
  - 全部失败: 整个任务标记失败
  - 超时: 单分支 300s, 等待所有分支完成
```

### 2.3 投票共识模式（Voting）

```
适用场景: 多 Agent 评审, 选择最佳结果

执行流程:
  输入数据
    ├──▶ Agent A (评审) ─── 评分 + 意见 A
    ├──▶ Agent B (评审) ─── 评分 + 意见 B
    └──▶ Agent C (评审) ─── 评分 + 意见 C
              │
              ▼
         投票规则
              │
              ▼
         最终决策

投票规则:
  - 多数通过: >50% 通过 → 采纳
  - 加权投票: 按 Agent 历史准确率加权
  - 一票否决: 任一评审不通过 → 需要人工干预

超时: 单个评审 180s, 全部评审 300s
```

### 2.4 委托嵌套模式（Delegation）

```
适用场景: Agent 在执行中发现需要子任务

执行流程:
  Agent A (需求分析)
    │ 发现需要代码分析
    │ 委托请求 → 平台调度器
    ▼
  平台调度 → Agent B (代码分析)
    │ 输出: 代码分析结果
    ▼
  回传给 Agent A
    │ 继续执行
    ▼
  最终输出

限制:
  - 最大嵌套深度: 3 层
  - 委托超时: 子任务 180s
  - 循环检测: 禁止 A→B→A 循环委托
```

---

## 3. 容错与韧性设计

### 3.1 服务级容错矩阵

| 故障类型 | 检测方式 | 处理策略 | 恢复时间 |
|---------|---------|---------|---------|
| Agent 调用超时 | Circuit Breaker | 降级到备选 Agent | <5s |
| Agent 服务不可用 | 健康检查失败 | 熔断 + 通知 | 30s 后重试 |
| Kafka 消费失败 | 消费异常 | 重试 3 次 → DLQ | <30s |
| 数据库连接池耗尽 | HikariCP 监控 | 降级为只读 + 告警 | 需人工 |
| Redis 不可用 | 连接异常 | 降级（跳过缓存） | 自动恢复 |
| DSL 版本冲突 | 乐观锁异常 | 自动重试 3 次 | <3s |
| 文件解析失败 | 异常捕获 | 返回原始文本 | 即时 |
| SSE 连接断开 | 心跳超时 | 客户端自动重连 | <5s |

### 3.2 Kafka 消费失败处理

```java
@KafkaListener(topics = "task.created")
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000),
    dltTopicSuffix = ".dlt",
    autoCreateTopics = "true"
)
public void handleTaskCreated(TaskCreatedEvent event) {
    taskScheduler.schedule(event.getTaskId());
}

// DLQ 处理: 记录日志 + 发送告警
@DltHandler
public void handleDlt(TaskCreatedEvent event) {
    log.error("Task scheduling failed after 3 retries: {}", event.getTaskId());
    auditService.recordFailure("task_scheduling", event.getTaskId());
    notificationService.alertOps("Task scheduling DLQ", event);
}
```

### 3.3 Saga 模式 — 任务创建跨服务事务

```
任务创建 Saga:

  Step 1: task-service      → 创建任务记录 (PENDING)
  Step 2: employee-service   → 分配数字员工
  Step 3: scheduler-service  → 启动 Temporal Workflow
  Step 4: agent-gateway      → 调度 Agent 执行

  补偿操作（逆序）:
  Step 4 失败 → Step 3: 取消 Workflow
  Step 3 失败 → Step 2: 释放员工
  Step 2 失败 → Step 1: 标记任务 CANCELLED

  实现方式: Temporal Saga (内置补偿支持)
```

```java
@WorkflowImpl
public class TaskCreationSagaImpl implements TaskCreationSaga {

    private final Saga.Options sagaOptions = new Saga.Options.Builder()
        .setParallelCompensation(false)  // 顺序补偿
        .build();

    @Override
    public TaskResult createTask(TaskRequest request) {
        Saga saga = new Saga(sagaOptions);
        try {
            // Step 1: 创建任务
            Long taskId = activities.createTask(request);
            saga.addCompensation(activities::cancelTask, taskId);

            // Step 2: 分配员工
            Long employeeId = activities.assignEmployee(taskId, request.getCapability());
            saga.addCompensation(activities::releaseEmployee, taskId, employeeId);

            // Step 3: 启动工作流
            String workflowId = activities.startWorkflow(taskId, employeeId);
            saga.addCompensation(activities::cancelWorkflow, workflowId);

            // Step 4: 调度 Agent
            activities.dispatchToAgent(taskId, employeeId);

            return new TaskResult(taskId, "SUCCESS");

        } catch (Exception e) {
            saga.compensate();
            throw e;
        }
    }
}
```

---

## 4. 性能优化设计

### 4.1 缓存体系总览

```
┌─────────────────────────────────────────────────┐
│              缓存分层架构                         │
│                                                 │
│  L1: JVM 本地缓存 (Caffeine)                     │
│  ├── 用途: 高频只读数据（配置、角色模板）          │
│  ├── 容量: 1000 条                               │
│  ├── TTL: 5min                                   │
│  └── 策略: LRU 淘汰                              │
│                                                 │
│  L2: Redis 分布式缓存                             │
│  ├── 用途: 会话、员工信息、搜索结果               │
│  ├── TTL: 5min-30min (按数据类型)                │
│  └── 策略: 写穿透 + 异步失效                      │
│                                                 │
│  L3: 数据库 (PostgreSQL)                         │
│  └── 用途: 持久化存储、事务保证                    │
│                                                 │
│  查询路径: L1 → L2 → L3                          │
│  写入路径: L3 → 失效 L2 → 失效 L1                │
└─────────────────────────────────────────────────┘
```

### 4.2 数据库连接池配置

```yaml
# HikariCP 配置（每个微服务）
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # 最大连接数
      minimum-idle: 5              # 最小空闲连接
      idle-timeout: 300000         # 空闲超时 5min
      max-lifetime: 1800000        # 最大生命周期 30min
      connection-timeout: 30000    # 获取连接超时 30s
      leak-detection-threshold: 60000  # 泄露检测 60s
```

### 4.3 关键查询性能目标

| 查询场景 | 目标延迟 (P99) | 优化策略 |
|---------|---------------|---------|
| 员工列表查询 | <50ms | 索引 + Redis 缓存 |
| 任务状态查询 | <30ms | Redis 实时状态 |
| DSL 文档读取 | <100ms | Redis 缓存 + JSONB 索引 |
| 知识库检索 | <500ms | Milvus HNSW + 结果缓存 |
| 审计日志查询 | <200ms | 表分区 + 时间索引 |
| SSE 事件推送 | <50ms | Redis Pub/Sub |

---

## 5. 安全详细设计

### 5.1 RBAC 权限矩阵

| API 资源 | 管理员 | 项目经理 | 开发者 | 只读用户 |
|---------|--------|---------|--------|---------|
| 数字员工管理 | ✅ CRUD | ✅ CR | ❌ | ❌ |
| 任务管理 | ✅ CRUD | ✅ CRUD | ✅ R | ✅ R |
| DSL 读写 | ✅ CRUD | ✅ CRUD | ✅ CRU | ✅ R |
| Agent 注册 | ✅ CRUD | ✅ CR | ❌ | ❌ |
| 审计日志 | ✅ R | ✅ R | ❌ | ❌ |
| 指令提交 | ✅ | ✅ | ✅ | ❌ |
| 知识库管理 | ✅ CRUD | ✅ CRUD | ✅ CR | ✅ R |
| 系统设置 | ✅ | ❌ | ❌ | ❌ |

### 5.2 JWT Token 生命周期

```
Access Token:
  - 签发: Keycloak
  - 有效期: 15min
  - 算法: RS256
  - 载荷: user_id, tenant_id, roles[], permissions[]

Refresh Token:
  - 有效期: 7d
  - 存储: HttpOnly Cookie
  - 续期: 每次刷新生成新的 Refresh Token (Rotation)

服务间通信:
  - 方式: mTLS (Mutual TLS)
  - 证书管理: cert-manager (K8s)
  - 有效期: 90d 自动轮转
```

### 5.3 敏感数据加密

```
加密策略:
  - 算法: AES-256-GCM
  - 密钥管理: HashiCorp Vault / K8s Secret
  - 加密字段:
    - Agent API Key
    - Agent Endpoint URL
    - 用户个人信息 (PII)

审计日志脱敏规则:
  - API Key: 仅显示前 4 位 + "****"
  - 邮箱: u***@example.com
  - IP: 保留前 2 段 (192.168.***)
```

### 5.4 API 限流配置

```yaml
# APISIX 限流配置
apisix:
  plugins:
    limit-count:
      - route: /api/v1/*
        count: 1000          # 每分钟 1000 请求
        time_window: 60
        key: consumer_name   # 按用户限流
      - route: /api/v1/dsl/*/patch
        count: 100           # DSL Patch 每分钟 100 次
        time_window: 60
      - route: /api/v1/files/upload
        count: 50            # 文件上传每分钟 50 次
        time_window: 60
    limit-req:
      - route: /api/v1/sse/*
        rate: 10             # SSE 连接每秒 10 个
        burst: 20
```

---

## 6. 迭代优化路线图

### 6.1 Phase 1 → Phase 2 优化点

| 优化项 | 当前状态 | 目标 | 优先级 |
|--------|---------|------|--------|
| 缓存命中率 | 无缓存 | >85% | P0 |
| API 延迟 P99 | 未知 | <500ms | P0 |
| Agent 成功率 | 未知 | >90% | P0 |
| 单元测试覆盖 | 0% | >70% | P1 |
| 数据库慢查询 | 未知 | <10/日 | P1 |
| 日志标准化 | 无 | 结构化日志 | P1 |

### 6.2 Phase 2 → Phase 3 优化点

| 优化项 | 目标 | 说明 |
|--------|------|------|
| 多租户数据隔离 | Schema 隔离 → Row-Level Security | 更细粒度隔离 |
| DSL 大文档优化 | 分片存储 | >10MB 的 DSL 自动分片 |
| Agent 调用成本 | 成本仪表盘 | Langfuse Token 统计 → 成本告警 |
| 水平扩展验证 | 100 并发用户 | 压测 + HPA 调优 |
| 灰度发布 | 金丝雀部署 | APISIX 流量切分 |

### 6.3 Phase 3 → Phase 4 优化点

| 优化项 | 目标 | 说明 |
|--------|------|------|
| 高可用 | 99.9% SLA | 多可用区部署 |
| 数据备份 | RPO <1h | 增量备份 + 异地复制 |
| 成本优化 | 降低 30% | 预留实例 + Spot 混合 |
| 全球化 | 多地域部署 | CDN + 就近接入 |
| 智能调度 | ML 调度模型 | 基于历史数据预测最优 Agent |
