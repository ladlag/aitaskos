# 任务拆分与排期

> 基于详细设计，将系统实现分解为可执行的开发任务，按 Sprint 排期，明确里程碑、验收标准和风险管控。

## 1. 项目总览

### 1.1 整体排期

```
Phase 1: 核心基础 (12 周, Sprint 1-6)
  ├── S1 新需求生成（核心场景）
  ├── S7 局部修改
  ├── S8 需求评审
  └── 基础设施搭建

Phase 2: 场景扩展 (8 周, Sprint 7-10)
  ├── S2 原型驱动需求
  ├── S6 需求迭代
  └── 高级特性

Phase 3: 进阶场景 (8 周, Sprint 11-14)
  ├── S3 HTML/系统反推
  ├── S4 流程图驱动
  └── 性能优化

Phase 4: 企业级 (6 周, Sprint 15-17)
  ├── S5 存量系统升级
  └── 高可用/多租户
```

### 1.2 团队配置建议

| 角色 | 人数 | 技能要求 |
|------|------|---------|
| 后端开发 (Java/Spring Boot) | 3-4 | Spring Boot 3.x, Temporal, Kafka, PostgreSQL |
| 前端开发 (Vue3) | 2 | Vue3, ElementPlus, TipTap, AntV X6, SSE |
| Agent 开发 | 1-2 | Dify, LangChain4j, Prompt Engineering |
| DevOps | 1 | K8s, Docker, CI/CD, 监控 |
| 测试 | 1-2 | 自动化测试, 性能测试, API 测试 |
| 产品/架构 | 1 | 技术方案评审, 验收 |
| **合计** | **9-12** | |

---

## 2. Phase 1: 核心基础（Sprint 1-6, 12 周）

### Sprint 1: 基础设施搭建（第 1-2 周）

| 任务 ID | 任务名称 | 负责角色 | 故事点 | 依赖 |
|---------|---------|---------|--------|------|
| T1.1 | 搭建 Git 仓库 + Maven 多模块结构 | DevOps | 3 | - |
| T1.2 | docker-compose 开发环境（21 服务） | DevOps | 5 | - |
| T1.3 | CI/CD 流水线搭建（GitHub Actions） | DevOps | 5 | T1.1 |
| T1.4 | Keycloak 认证配置（Realm + Client + Roles） | 后端 | 5 | T1.2 |
| T1.5 | APISIX 网关路由配置 | DevOps | 3 | T1.2 |
| T1.6 | PostgreSQL Schema 初始化脚本 | 后端 | 5 | T1.2 |
| T1.7 | Kafka Topic 创建脚本 | DevOps | 2 | T1.2 |
| T1.8 | 前端项目脚手架（Vue3 + ElementPlus + Router + Pinia） | 前端 | 5 | - |
| T1.9 | 统一响应/异常/日志框架 | 后端 | 3 | T1.1 |
| T1.10 | gRPC Proto 定义文件 | 后端 | 3 | T1.1 |

**Sprint 1 合计**: 39 故事点

**验收标准**:
- ✅ 所有开发者可以一键启动完整开发环境
- ✅ CI 流水线可以自动构建、测试、镜像推送
- ✅ 前端可以通过 APISIX 访问后端 API
- ✅ Keycloak 登录流程可用

---

### Sprint 2: 项目与员工核心（第 3-4 周）

| 任务 ID | 任务名称 | 负责角色 | 故事点 | 依赖 |
|---------|---------|---------|--------|------|
| T2.1 | project-service: 项目 CRUD API | 后端 | 5 | T1.6 |
| T2.2 | project-service: 迭代管理 API | 后端 | 3 | T2.1 |
| T2.3 | project-service: 租户/用户管理（Keycloak 同步） | 后端 | 5 | T1.4 |
| T2.4 | employee-service: 数字员工 CRUD API | 后端 | 5 | T1.6 |
| T2.5 | employee-service: Agent 绑定管理 API | 后端 | 5 | T2.4 |
| T2.6 | employee-service: 工作流模板关联 API | 后端 | 3 | T2.4 |
| T2.7 | employee-service: gRPC 服务（供 scheduler 调用） | 后端 | 3 | T1.10 |
| T2.8 | 前端: 项目管理页面 | 前端 | 5 | T2.1 |
| T2.9 | 前端: 数字员工管理页面 | 前端 | 5 | T2.4 |
| T2.10 | 前端: Agent 绑定配置界面 | 前端 | 5 | T2.5 |
| T2.11 | 单元测试: project-service + employee-service | 测试 | 5 | T2.1-T2.6 |

**Sprint 2 合计**: 49 故事点

**验收标准**:
- ✅ 可以创建项目、迭代
- ✅ 可以创建数字员工、绑定 Agent、关联工作流
- ✅ RBAC 权限控制生效
- ✅ 单元测试覆盖率 >70%

---

### Sprint 3: Agent 网关与 DSL（第 5-6 周）

| 任务 ID | 任务名称 | 负责角色 | 故事点 | 依赖 |
|---------|---------|---------|--------|------|
| T3.1 | agent-gateway: Agent 注册管理 API | 后端 | 5 | T1.6 |
| T3.2 | agent-gateway: Skill 注册（OpenClaw 兼容） | 后端 | 8 | T3.1 |
| T3.3 | agent-gateway: Dify API 适配器 | 后端 | 5 | T3.1 |
| T3.4 | agent-gateway: HTTP REST 适配器 | 后端 | 3 | T3.1 |
| T3.5 | agent-gateway: 回调接收与处理 | 后端 | 5 | T3.1 |
| T3.6 | agent-gateway: 健康检查服务 | 后端 | 3 | T3.1 |
| T3.7 | agent-gateway: Circuit Breaker (Resilience4j) | 后端 | 5 | T3.3 |
| T3.8 | dsl-service: DSL 文档 CRUD | 后端 | 5 | T1.6 |
| T3.9 | dsl-service: JSON Patch 引擎（RFC 6902） | 后端 | 8 | T3.8 |
| T3.10 | dsl-service: 版本管理（快照/回滚/diff） | 后端 | 5 | T3.8 |
| T3.11 | dsl-service: 冲突检测与乐观锁 | 后端 | 5 | T3.9 |
| T3.12 | 前端: Agent 管理界面 | 前端 | 5 | T3.1 |
| T3.13 | 集成测试: Agent 调用链路 | 测试 | 5 | T3.3-T3.5 |

**Sprint 3 合计**: 67 故事点

**验收标准**:
- ✅ 可以注册 Dify Agent 并成功调用
- ✅ 可以注册 OpenClaw 兼容的 Skill
- ✅ DSL 文档的增量 Patch 正确
- ✅ 并发 Patch 冲突正确检测
- ✅ Agent 调用失败时 Circuit Breaker 生效

---

### Sprint 4: 工作流与调度（第 7-8 周）

| 任务 ID | 任务名称 | 负责角色 | 故事点 | 依赖 |
|---------|---------|---------|--------|------|
| T4.1 | scheduler-service: Temporal Worker 基础设施 | 后端 | 5 | T1.2 |
| T4.2 | scheduler-service: 任务调度算法（三级匹配） | 后端 | 8 | T2.7, T3.1 |
| T4.3 | scheduler-service: NewRequirementWorkflow 实现 | 后端 | 13 | T4.1, T3.3 |
| T4.4 | scheduler-service: 指令队列消费与执行 | 后端 | 8 | T4.1 |
| T4.5 | task-service: 任务 CRUD + 状态管理 | 后端 | 5 | T1.6 |
| T4.6 | task-service: 指令队列管理 API | 后端 | 5 | T1.6 |
| T4.7 | task-service: Kafka 事件发布（task.created 等） | 后端 | 3 | T1.7 |
| T4.15 | task-service: 条目管理 CRUD + 分类筛选 | 后端 | 5 | T4.5 |
| T4.16 | task-service: 条目关联与追溯链 API | 后端 | 8 | T4.15 |
| T4.17 | task-service: 条目交付物管理 | 后端 | 5 | T4.15 |
| T4.18 | task-service: 条目审计轨迹 | 后端 | 3 | T4.15 |
| T4.19 | task-service: Agent 输出自动条目提取 | 后端 | 8 | T4.15, T4.3 |
| T4.8 | interaction-service: 对话管理 | 后端 | 5 | T1.6 |
| T4.9 | interaction-service: SSE 推送实现 | 后端 | 8 | T4.8 |
| T4.10 | interaction-service: 消息路由（Kafka → SSE） | 后端 | 5 | T4.9, T1.7 |
| T4.11 | 前端: 对话交互界面（流式输出） | 前端 | 8 | T4.9 |
| T4.12 | 前端: 指令队列面板 | 前端 | 5 | T4.6 |
| T4.13 | 前端: 执行过程面板（步骤进度） | 前端 | 5 | T4.9 |
| T4.14 | 集成测试: 端到端 S1 流程 | 测试 | 8 | T4.3 |
| T4.20 | 前端: 条目管理面板（列表+树形+分类筛选） | 前端 | 8 | T4.15 |
| T4.21 | 前端: 条目追溯链可视化 | 前端 | 5 | T4.16 |
| T4.22 | 前端: 条目交付物查看 | 前端 | 3 | T4.17 |

**Sprint 4 合计**: 120 故事点（可拆分为 2-3 个子 Sprint）

**验收标准**:
- ✅ S1 新需求生成端到端流程可用
- ✅ 用户输入 → Agent 执行 → 结果返回 → DSL 更新 → SSE 推送
- ✅ 指令队列优先级排序正确
- ✅ 执行过程实时可见
- ✅ Agent 输出自动提取为结构化条目
- ✅ 条目支持分类筛选、关联、交付物查看
- ✅ 条目操作完整审计轨迹

---

### Sprint 5: 核心交互完善（第 9-10 周）

| 任务 ID | 任务名称 | 负责角色 | 故事点 | 依赖 |
|---------|---------|---------|--------|------|
| T5.1 | scheduler-service: CommandExecutionWorkflow（S7） | 后端 | 8 | T4.1 |
| T5.2 | 异步反馈: AI 提问 → 用户回答 → 继续执行 | 后端 | 8 | T4.3 |
| T5.3 | file-service: 文件上传 + MinIO 存储 | 后端 | 5 | T1.2 |
| T5.4 | file-service: 文件解析（Tika + PaddleOCR） | 后端 | 8 | T5.3 |
| T5.5 | 前端: 产物区 — PRD 编辑器（TipTap） | 前端 | 13 | T3.8 |
| T5.6 | 前端: 产物区 — 流程图编辑器（AntV X6） | 前端 | 13 | T3.8 |
| T5.7 | 前端: 结构区 — 需求树/页面树/流程树 | 前端 | 8 | T3.8 |
| T5.8 | 前端: 异步提问卡片 UI | 前端 | 5 | T5.2 |
| T5.9 | 集成测试: S7 局部修改流程 | 测试 | 5 | T5.1 |
| T5.10 | 集成测试: 文件上传解析流程 | 测试 | 3 | T5.4 |

**Sprint 5 合计**: 76 故事点

**验收标准**:
- ✅ S7 局部修改流程可用（选中→修改→更新→同步）
- ✅ AI 可以主动提问，用户可以回答
- ✅ 文件上传并解析为文本
- ✅ PRD 编辑器可以展示和编辑 DSL 内容
- ✅ 流程图可以展示 DSL flows

---

### Sprint 6: Phase 1 收尾（第 11-12 周）

| 任务 ID | 任务名称 | 负责角色 | 故事点 | 依赖 |
|---------|---------|---------|--------|------|
| T6.1 | scheduler-service: ReviewWorkflow（S8） | 后端 | 5 | T4.1 |
| T6.2 | audit-service: 审计日志记录 | 后端 | 5 | T1.6 |
| T6.3 | audit-service: 绩效统计 API | 后端 | 5 | T6.2 |
| T6.4 | notification-service: 邮件通知 | 后端 | 3 | T1.7 |
| T6.5 | knowledge-service: 知识库 CRUD | 后端 | 5 | T1.2 |
| T6.6 | knowledge-service: 向量检索（Milvus） | 后端 | 8 | T6.5 |
| T6.7 | 前端: 审计日志查看 | 前端 | 3 | T6.2 |
| T6.8 | 前端: 知识库管理界面 | 前端 | 5 | T6.5 |
| T6.9 | 前端: 数字员工绩效面板 | 前端 | 5 | T6.3 |
| T6.10 | Langfuse 集成: Agent 调用 Trace | 后端 | 5 | T3.3 |
| T6.11 | 全量集成测试 | 测试 | 8 | All |
| T6.12 | 性能测试: 基准线 | 测试 | 5 | All |
| T6.13 | 安全扫描 + 修复 | DevOps | 3 | All |
| T6.14 | Phase 1 文档更新 | 全员 | 3 | All |

**Sprint 6 合计**: 68 故事点

**Phase 1 验收标准**:
- ✅ S1 新需求生成端到端可用
- ✅ S7 局部修改端到端可用
- ✅ S8 需求评审端到端可用
- ✅ 数字员工管理全功能可用
- ✅ Agent 注册/调用/回调链路通畅
- ✅ DSL 读写/版本/冲突处理正常
- ✅ SSE 实时推送正常
- ✅ 知识库基本可用
- ✅ 单元测试覆盖率 >70%, 集成测试通过
- ✅ P99 延迟 <3s

---

## 3. Phase 2: 场景扩展（Sprint 7-10, 8 周）

### Sprint 7-8: S2 原型驱动 + S6 需求迭代（第 13-16 周）

| 任务 ID | 任务名称 | 负责角色 | 故事点 | 依赖 |
|---------|---------|---------|--------|------|
| T7.1 | scheduler-service: PrototypeDrivenWorkflow（S2） | 后端 | 8 | T4.1 |
| T7.2 | scheduler-service: RequirementIterationWorkflow（S6） | 后端 | 8 | T4.1 |
| T7.3 | file-service: HTML/原型解析增强 | 后端 | 5 | T5.4 |
| T7.4 | dsl-service: DSL Diff 计算（版本间差异） | 后端 | 5 | T3.10 |
| T7.5 | dsl-service: DSL 导出（PRD Markdown/Word） | 后端 | 5 | T3.8 |
| T7.6 | agent-gateway: A2A 协议适配器 | 后端 | 8 | T3.1 |
| T7.7 | agent-gateway: MCP Server 实现 | 后端 | 8 | T3.1 |
| T7.8 | 前端: 原型上传与预览 | 前端 | 5 | T7.3 |
| T7.9 | 前端: DSL Diff 查看器 | 前端 | 5 | T7.4 |
| T7.10 | 前端: PRD 导出功能 | 前端 | 3 | T7.5 |
| T7.11 | Agent 协作: Chain 模式实现 | 后端 | 5 | T4.1 |
| T7.12 | Agent 协作: Fanout 模式实现 | 后端 | 5 | T4.1 |
| T7.13 | 集成测试: S2 + S6 流程 | 测试 | 8 | T7.1, T7.2 |
| T7.14 | 性能优化: Redis 缓存层实现 | 后端 | 5 | T4.5 |

**Sprint 7-8 合计**: 85 故事点

### Sprint 9-10: 高级特性（第 17-20 周）

| 任务 ID | 任务名称 | 负责角色 | 故事点 | 依赖 |
|---------|---------|---------|--------|------|
| T8.1 | Agent 协作: Voting 模式实现 | 后端 | 5 | T7.11 |
| T8.2 | Agent 协作: Delegation 模式实现 | 后端 | 8 | T7.11 |
| T8.3 | audit-service: 自动化评估管线 | 后端 | 8 | T6.2 |
| T8.4 | audit-service: LLM-as-Judge 评估 | 后端 | 5 | T8.3 |
| T8.5 | notification-service: 钉钉/企微集成 | 后端 | 5 | T6.4 |
| T8.6 | 前端: 评估报告展示 | 前端 | 5 | T8.3 |
| T8.7 | 前端: 暗黑模式 + 主题切换 | 前端 | 5 | - |
| T8.8 | 前端: 快捷键支持 | 前端 | 3 | - |
| T8.9 | 数据库优化: 分区表 + 索引调优 | 后端 | 5 | - |
| T8.10 | 监控告警: Prometheus + Grafana 仪表盘 | DevOps | 5 | - |
| T8.11 | 全量回归测试 | 测试 | 5 | All |
| T8.12 | 性能测试: 50 并发用户 | 测试 | 5 | All |

**Sprint 9-10 合计**: 64 故事点

**Phase 2 验收标准**:
- ✅ S2 原型驱动需求可用
- ✅ S6 需求迭代可用
- ✅ A2A / MCP 协议可用
- ✅ 4 种 Agent 协作模式可用
- ✅ 自动化评估管线可用
- ✅ 缓存命中率 >80%

---

## 4. Phase 3: 进阶场景（Sprint 11-14, 8 周）

### Sprint 11-12（第 21-24 周）

| 任务 ID | 任务名称 | 故事点 |
|---------|---------|--------|
| T9.1 | scheduler-service: FlowDrivenWorkflow（S4） | 8 |
| T9.2 | file-service: HTML 页面反推（S3） | 8 |
| T9.3 | scheduler-service: PrototypeDrivenWorkflow 增强（S3） | 5 |
| T9.4 | dsl-service: DSL 大文档分片优化 | 5 |
| T9.5 | 前端: 流程图驱动需求入口 | 5 |
| T9.6 | 前端: HTML 上传与反推 | 5 |
| T9.7 | 集成测试: S3 + S4 | 5 |
| T9.8 | 性能优化: 数据库读写分离 | 5 |

### Sprint 13-14（第 25-28 周）

| 任务 ID | 任务名称 | 故事点 |
|---------|---------|--------|
| T10.1 | 灰度发布: APISIX 流量切分 | 5 |
| T10.2 | 压测: 100 并发用户 | 5 |
| T10.3 | 安全加固: API Key 加密 + mTLS | 5 |
| T10.4 | 日志系统: OpenSearch + Filebeat | 5 |
| T10.5 | 前端: 性能优化（懒加载/虚拟滚动） | 5 |
| T10.6 | 文档: 用户手册 + API 文档 | 5 |
| T10.7 | 全量回归 + 安全扫描 | 5 |

---

## 5. Phase 4: 企业级（Sprint 15-17, 6 周）

### Sprint 15-17（第 29-34 周）

| 任务 ID | 任务名称 | 故事点 |
|---------|---------|--------|
| T11.1 | scheduler-service: LegacyUpgradeWorkflow（S5） | 13 |
| T11.2 | 存量系统分析: 代码/DB/接口 并行分析 | 8 |
| T11.3 | 多租户 Row-Level Security | 8 |
| T11.4 | 高可用: 多可用区部署 | 8 |
| T11.5 | 灾难恢复: 自动备份 + 恢复演练 | 5 |
| T11.6 | 成本仪表盘: Token/调用费用统计 | 5 |
| T11.7 | 最终验收测试 | 8 |
| T11.8 | 生产部署 + 上线 | 5 |

---

## 6. 里程碑定义

| 里程碑 | 时间 | 关键交付 | 质量门 |
|--------|------|---------|--------|
| M1: 基础设施就绪 | 第 2 周 | 开发环境、CI/CD、脚手架 | 环境可用 |
| M2: 核心模型完成 | 第 4 周 | 项目/员工/Agent CRUD | API 测试通过 |
| M3: 核心链路打通 | 第 8 周 | S1 端到端可用 | 集成测试通过 |
| M4: Phase 1 交付 | 第 12 周 | S1+S7+S8 可用 | 全量测试+性能基线 |
| M5: Phase 2 交付 | 第 20 周 | S2+S6+协议标准化 | 回归测试通过 |
| M6: Phase 3 交付 | 第 28 周 | S3+S4+性能优化 | 压测通过 |
| M7: 全量交付 | 第 34 周 | S5+高可用+上线 | 验收测试+安全扫描 |

---

## 7. 风险登记簿

| 风险 ID | 风险描述 | 影响 | 概率 | 级别 | 缓解措施 |
|---------|---------|------|------|------|---------|
| R1 | Temporal 学习曲线陡峭 | 工期延长 2-4 周 | 中 | 🟡 | 提前安排 Temporal 培训；准备 Fallback 方案（Spring State Machine） |
| R2 | 外部 Agent 接口不稳定 | S1 流程不可用 | 中 | 🟡 | Mock Agent 用于开发/测试；Circuit Breaker 保护 |
| R3 | DSL 结构过于复杂 | 前端渲染性能差 | 中 | 🟡 | 限制 DSL 最大深度；虚拟滚动；分片加载 |
| R4 | 团队规模不足 | Phase 1 延期 | 高 | 🔴 | 最小 MVP 范围（仅 S1）；外包部分前端 |
| R5 | Dify API 变更 | Agent 调用失败 | 低 | 🟢 | 适配器模式隔离；版本锁定 |
| R6 | Milvus 向量检索精度低 | 知识库效果差 | 低 | 🟢 | 调优 HNSW 参数；测试多种 Embedding 模型 |
| R7 | 并发 DSL 修改冲突 | 数据不一致 | 中 | 🟡 | 乐观锁+自动重试；冲突提示用户 |
| R8 | Kafka 消息丢失 | 事件未处理 | 低 | 🟢 | acks=all；DLQ 处理；监控消费 Lag |

---

## 8. Sprint 容量规划

### 每 Sprint 容量（2 周）

```
假设: 团队 10 人, 每人每 Sprint 可用 8 天 (扣除会议、Review 等)

总容量 = 10 人 × 8 天 × 理想系数 0.7 = 56 人天 ≈ 56-70 故事点

Sprint 故事点上限: 70
高风险 Sprint 建议: 50 以内

各 Sprint 负载:
  Sprint 1:  39 点 ✅ 正常
  Sprint 2:  49 点 ✅ 正常
  Sprint 3:  67 点 ⚠️ 偏高（可拆分）
  Sprint 4:  91 点 🔴 过高 → 拆为 Sprint 4a + 4b
  Sprint 5:  76 点 🔴 过高 → 拆为 Sprint 5a + 5b
  Sprint 6:  68 点 ⚠️ 偏高
```

### Sprint 4 拆分建议

```
Sprint 4a (第 7 周):
  T4.1 + T4.2 + T4.5 + T4.6 + T4.7 + T4.8 = 31 点

Sprint 4b (第 8 周):
  T4.3 + T4.4 + T4.9 + T4.10 + T4.11 + T4.12 + T4.13 + T4.14 = 60 点
  → 仍偏高，T4.3 可拆分为:
    T4.3a: 工作流骨架（无 Agent 调用） = 5 点
    T4.3b: Agent 调用集成 = 8 点
```

---

## 9. 依赖关系图

```
Sprint 1 (基础设施)
    │
    ├──▶ Sprint 2 (项目+员工)
    │        │
    │        ├──▶ Sprint 3 (Agent+DSL)
    │        │        │
    │        │        ├──▶ Sprint 4 (工作流+调度)
    │        │        │        │
    │        │        │        ├──▶ Sprint 5 (交互完善)
    │        │        │        │        │
    │        │        │        │        ├──▶ Sprint 6 (Phase 1 收尾)
    │        │        │        │        │
    │        │        │        │        └──▶ Sprint 7-8 (S2+S6)
    │        │        │        │
    │        │        │        └──▶ Sprint 9-10 (高级特性)
    │        │        │
    │        │        └──▶ Sprint 11-12 (S3+S4)
    │        │
    │        └──▶ Sprint 15-17 (S5+企业级)
    │
    └──▶ Sprint 13-14 (性能+安全)

关键路径: Sprint 1 → 2 → 3 → 4 → 5 → 6 (Phase 1)
```
