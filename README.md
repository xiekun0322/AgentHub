# AgentHub 技术设计文档 v1.0（可落地版）

> **文档定位**：将 README 愿景收敛为可进入开发的完整设计基线，融合主流程、Agent 调用链、底层实现与安全边界。
> **阅读路径**：Part 1 跑通流程 → Part 2 看每个 Agent 如何接线 → Part 3 看每个功能落到哪个类/表/伪代码 → Part 4 安全约束 → Part 5 落地附录。
> **架构原则**：模块化单体、状态机锁流程、AOP 锁权限、审批网关锁高危、一切可审计、一切可急停。

---

## 目录

- [Part 0：文档约定与核心决策](#part-0文档约定与核心决策)
- [Part 1：端到端主流程](#part-1端到端主流程)
- [Part 2：每个 Agent 的架构与调用链](#part-2每个-agent-的架构与调用链)
- [Part 3：底层功能实现](#part-3底层功能实现)
- [Part 4：安全设计专章](#part-4安全设计专章补强)
- [Part 5：落地附录](#part-5落地附录)

---

# Part 0：文档约定与核心决策

## 0.1 术语表

| 术语 | 含义 |
|---|---|
| Agent | 绑定人类 owner 的角色化智能体（PM/Arch/Dev/Algo/Audit/QA/Ops） |
| 数字镜像 | Agent 权限严格 ≤ 其 owner 的人类权限 |
| Skill | 通过 `@AgentSkill` 注册的原子能力 |
| Artifact | Agent 结构化产物（计划/设计/代码/报告） |
| ApprovalTicket | 审批单，HIGH/CRITICAL 技能执行前必须 |
| PAUSED | 治理态，冻结执行，不参与业务越级 |
| KillSwitch | 桌面端本地物理级急停能力 |
| Capability Manifest | 技能声明的能力清单（FILE/NET/PROCESS...） |
| Checkpoint | Agent 循环中每步 tool_call 后的可恢复快照 |

## 0.2 核心决策（ADR 摘要）

| 编号 | 决策 | 理由 |
|---|---|---|
| D1 | 模块化单体，不拆微服务 | 复杂度控制，与 README 守则一致 |
| D2 | 状态机唯一入口，业务代码禁止直改状态 | 杜绝越级乱跳 |
| D3 | 双层权限校验（SkillExecutor + AOP） | 双保险，防绕过管道 |
| D4 | 高危动作默认拒绝，超时默认拒绝 | 失败关闭 |
| D5 | Agent 循环设有上限 + Checkpoint | 防死循环 + 支持恢复 |
| D6 | 自进化技能先生成候选，人工上架 | 防越权生成工具 |
| D7 | Agent 链默认串行（PM→Arch→Dev→Algo→Audit→QA→Ops） | 与 README 顺序一致 |
| D8 | 审计只增不删，日志三层留存 | 合规 + 内存控制 |
| D9 | 外部内容进入 Context 前必须 sanitize + 验签 | 防注入 |
| D10 | 急停三级作用域，物理级 KillSwitch 独立于后端 | 断网也可停 |

## 0.3 开源边界（README “白盒底线”落地）

| 类别 | 是否开源 | 说明 |
|---|---|---|
| 状态机引擎、权限网关、审批网关、审计、三次法则检测器、Skill 注册管道 | **必须开源** | 核心治理能力，白盒可审查 |
| LLM Provider 适配器中厂商特定实现 | 可闭源 | 接口开源，实现可替换 |
| 桌面宠物皮肤、商业任务模板库 | 可闭源 | UI/内容资产 |
| 翻译层、沙箱 | 接口开源，实现可插拔 | 用户可替换实现 |
| 用户侧防呆交互 | 可闭源 | “黑盒防呆给用户” |

Phase 5 开源时按此清单剥离 `agenthub-core` 模块。

---

# Part 1：端到端主流程

## 1.0 系统启动流程
Spring Boot 启动
├─ 1. 加载配置（llm/sandbox/approval/evolution/security）
├─ 2. SkillRegistry 扫描 @AgentSkill，注册 SkillDescriptor（含 Capability Manifest）
├─ 3. AgentRegistry 加载角色模板并绑定 tenant/owner，校验一人一 Agent 唯一性
├─ 4. PermissionCache 预热
├─ 5. StateMachineRegistry 注册状态机定义
├─ 6. ApprovalScheduler + TaskWatchdog 启动
├─ 7. RuleOfThreeDetector 启动日志消费
├─ 8. WsBroker 启动，注册频道
└─ 9. /healthz 汇总各组件健康

text

- **失败策略**：核心组件（StateMachineRegistry / SkillRegistry）失败 → **fail-fast**，不进入可服务状态。
- **启动自检**：校验所有 Skill 的 Capability Manifest 完整、所有状态机 Transition 的 Guard 可执行。

## 1.1 主流程总时序
User API TaskAppSvc StateMachine ApprovalGW PM Arch Dev Audit QA Ops WS
|--新建->| | | | | | | | | | |
| |--create------>| | | | | | | | | |
| | |--init(PENDING)->| | | | | | | | |
| | |--dispatch(PM)-------------->|--exec-->| | | | | | |
| | | | | |--拆解->| | | | | |
| |<--approval req-| | |<--ticket--| | | | | |--ws->|
|--同意-->|--approve----->| | | | | | | | | |
| | |--transition(PENDING->CODING)->| | | | | | | |
| | |--dispatch(Arch)------------------------>| | | | | | |
| | | | | | |--设计->| | | | |
| | |--dispatch(Dev)-------------------------------->| | | | | |
| | | | | | | |--编码->| | | |
| | |--transition(CODING->REVIEWING)->| | | | | | | |
| | |--dispatch(Audit)---------------------------->| | |--审计->| | |
| | |--dispatch(QA)（Audit PASS 后）--------------------------->| |--测试->| |
| | |--transition(REVIEWING->DONE/CODING)->| | | | | | | |
| | |--dispatch(Ops)（需审批）-------------------------------------------->| |
|<--ws---| | | | | | | | | | |

text

## 1.2 分步流程

### Step 1：需求输入

- **触发**：用户通过 **No-Code UI**（表单/下拉/拖拽）提交，产出结构化 `TaskForm`。
- **入参 Schema**：
```json
{
  "title": "拍卖系统-高并发防重",
  "template": "AUCTION_HIGH_CONCURRENCY",
  "requirements": {"functional": ["出价防重"], "nonFunctional": {"qps": 5000, "p99Ms": 200}},
  "riskLevel": "MEDIUM",
  "ownerUserId": "u-1001",
  "accountableUserId": "u-1001"
}
调用链：

text
TaskController.create -> TaskApplicationService.create
  -> TaskFormSchemaValidator.validate
  -> TaskRepository.save(Task{status=PENDING, revision=0, accountableUserId})
  -> StateMachineService.init(taskId)
  -> AuditAspect.record(CREATE_TASK)
  -> WsPublisher.publish(task.updated)
  -> DispatchService.dispatchAsync(taskId, Role.PM)
落库：task、task_transition、audit_log。

异常：模板不存在 400；owner 无权限 403；accountable 非 tenant 成员 400。

Step 2：PM 拆解任务
调用链：

text
DispatchService.dispatch
 -> AgentRuntime.execute(task, Role.PM)
    -> ContextBuilder.build(task, PM)
    -> LlmProvider.chat(PmTemplates.DECOMPOSE, tools=[splitTask, estimateRisk])
    -> SkillExecutor.invoke("splitTask", args)
    -> ArtifactRepository.save(PlanArtifact)
    -> ApprovalService.request(TASK_PLAN_APPROVAL, taskId, plan)
       -> 创建 ApprovalTicket(PENDING, expiresAt=+30min)
       -> StateMachineService.pauseForApproval(taskId)   // 不改业务态，仅挂调度
    -> WsPublisher.publish(approval.requested)
落库：artifact、approval_ticket、invocation_log、audit_log、agent_checkpoint。

异常：LLM 超时重试 2 次 → FAILED(PM_TIMEOUT)；Schema 校验失败重拆 ≤3 → FAILED(PLAN_INVALID)。

Step 3：计划审批
text
ApprovalController.approve -> ApprovalApplicationService.approve
 -> 幂等校验（idempotency_key UNIQUE）
 -> 审批人校验（ApprovalPolicy；禁 requester==approver）
 -> ticket: PENDING -> APPROVED
 -> [同事务] StateMachineService.transition(taskId, PLAN_APPROVED)  // PENDING->CODING
 -> [事务后 Outbox] DispatchService.dispatchAll(children, Role.ARCH)
 -> WsPublisher.publish(approval.resolved / task.updated)
异常：超时 → ticket EXPIRED + 任务 FAILED(APPROVAL_TIMEOUT) + 生成 IncidentTicket 给 accountable；重复批准幂等返回 200。

一致性：审批通过 + 状态流转在同一本地事务，Outbox 异步派发。

Step 4：Arch 出设计
text
AgentRuntime.execute(task, Role.ARCH)
 -> ContextBuilder: 注入任务、上游 PlanArtifact、架构守则、ARCH 白名单工具
 -> LlmProvider.chat(ArchTemplates.DESIGN, tools=[writeDoc, drawDiagram, defineInterface])
 -> ArchitectureRuleChecker.review(output)   // 守则硬校验（禁微服务等）
 -> Artifact.save(DesignArtifact)
 -> StateMachineService.transition(task, DESIGN_READY)
 -> DispatchService.dispatch(task, Role.DEV)
异常：违反守则 → 打回，revision+1。

Step 5：Dev 编码
text
AgentRuntime.execute(task, Role.DEV)
 -> 循环 maxRounds=20:
     LlmProvider.chat(DevTemplates.CODE, tools=[readFile, writeFile, runCompile])
     -> 每个 tool_call: SkillExecutor.invoke
        -> Capability 硬校验 -> RBAC -> 风险判定
        -> 沙箱执行 -> 审计 -> InvocationLog
        -> [副作用技能] 写 SideEffectRecord 供恢复去重
     -> AgentCheckpoint.save(sessionId, round, historyHash, sideEffectKeys)
 -> Artifact.save(CodeArtifact)
 -> StateMachineService.transition(CODING -> REVIEWING)
 -> DispatchService.dispatch(task, Role.AUDIT)   // 串行：Audit 后再 QA
异常：编译失败回灌修复 ≤3 → FAILED(COMPILE_ERROR)；触发 SHELL_EXEC → 挂起审批。

Agent 链修正：默认 Audit → QA 串行（与 README 一致）；并行仅模板可选，且 QA 在只读快照上跑并绑定 commit hash。

Step 6：Algo 分支（可选）
requiresAlgorithm=true 时触发，产出 AlgoArtifact，性能不达标迭代 ≤N → FAILED(PERF_UNMET)。

Step 7：CodeAudit
text
AgentRuntime.execute(task, Role.AUDIT)
 -> tools=[staticScan, dependencyCheck, secretScan, ruleCheck]（沙箱内）
 -> AuditArtifact{issues[], severity, pass}
 -> if pass: 标记 AUDIT_PASSED，Dispatch(QA)
    else:    transition(REVIEWING -> CODING)，revision+1
高危（密钥泄露/注入/越权）BLOCK；中低危可配警告。

Step 8：QA
text
AgentRuntime.execute(task, Role.QA)
 -> tools=[runUnitTest, runIntegrationTest, runLoadTest]（沙箱）
 -> TestReportArtifact
 -> if passed: 等待 Ops / transition DONE 候选
    else:      transition(REVIEWING -> CODING)
               WsPublisher.publish(agent.status.changed: QA=ERROR)
               WsPublisher.publish(human.action.requested: [REWRITE, STOP])
状态语义修正（ADR C2）：子任务 FAILED + 主任务 REVIEWING→CODING 打回，主任务不直接 FAILED。

Step 9：Ops 部署
deploy 为 HIGH 风险 → 审批通过后受控执行 → 健康检查通过 DONE，失败回滚 FAILED(DEPLOY_ERROR)。

Step 10：闭环与打回
打回统一 REVIEWING -> CODING，revision+1，生成重写工单。

达 maxRevision=5 → FAILED(MAX_REVISION_EXCEEDED)，生成 IncidentTicket。

每次打回记录 reason + 失败 Artifact 引用，供下一轮 Context 注入。

Part 2：每个 Agent 的架构与调用链
2.0 统一运行时契约
java
public interface AgentRuntime {
    AgentResult execute(TaskContext ctx, AgentRole role);
}
固定六段式：

text
1. ContextBuilder.build(ctx, role)          // 结构化上下文 + untrusted 通道隔离
2. ToolResolver.resolve(role, agent)         // 白名单 + 权限过滤
3. Loop (maxRounds):
     LlmProvider.chat -> tool_calls -> SkillExecutor.invoke -> 回灌
     每步 AgentCheckpoint.save
4. ArtifactBuilder.build(result)
5. StateMachineService.transition(...)       // 不直接改库
6. AuditAspect + Metrics + WsPublisher
强约束：

Agent 禁止直接调 Repository 改状态；

Agent 只能调 ToolResolver 返回的工具；

每次 tool_call 必过 SkillExecutor 管道；

循环上限可配；超限即失败并 escalate 给 accountable；

长任务可被 RevocableTask 实时中止（权限吊销/急停）。

2.1 各角色一览
Agent	职责	工具白名单	权限	输入	输出	失败处理
PM	需求拆解、风险估算	splitTask, estimateRisk, writeDoc	task:split, doc:write	TaskForm + 模板	PlanArtifact	schema 失败重试≤3；超时重试≤2
Arch	模块/接口/数据模型/选型	writeDoc, drawDiagram, defineInterface	doc:write, design:write	Plan 子任务 + 架构守则	DesignArtifact	守则违反打回
Dev	编码、自编译、自修复	readFile, writeFile, runCompile	code:read, code:write, code:compile	Design + 规范	CodeArtifact	编译失败修复≤3
Algo	算法实现、复杂度、基准	writeFile, runBenchmark	code:write, bench:run	算法契约 + 性能约束	AlgoArtifact	未达标迭代≤N
Audit	静态扫描、依赖、密钥	staticScan, dependencyCheck, secretScan, ruleCheck	code:read, audit:run	CodeArtifact	AuditArtifact	高危 BLOCK
QA	单测/集成/压测	runUnitTest, runIntegrationTest, runLoadTest	code:test	CodeArtifact + 测试范围	TestReportArtifact	失败打回
Ops	部署、健康、回滚	deploy, healthCheck, rollback	deploy:run（≤owner）	通过 Audit+QA 产物	DeployArtifact	失败自动 rollback
状态映射：IDLE / CODING / REVIEWING / WAITING_APPROVAL / ERROR / CRASHED / PAUSED 与桌面动画一一对应（见 3.10.3）。

2.2 横向组件
组件	作用	接入点
Translator	双向双轨（术语↔大白话）	Agent 输出前/后、WS 推送前、人类回复解析
ApprovalGateway	审批闸门	SkillExecutor 风险判定 / 状态机计划审批
EvolutionService	三次法则检测与沉淀	InvocationLog 消费
AuditAspect	统一审计	状态变更/技能执行/审批/清理
WsPublisher	状态广播	状态变更/审批/消息/抛按钮
Watchdog	状态驻留超时	各状态 maxDwellTime
ContentSanitizer	防注入	ContextBuilder 输入前
ArtifactSigner	产物签名验签	Artifact 落库/消费
Part 3：底层功能实现
3.1 状态机引擎
3.1.1 模型
java
public enum TaskStatus { PENDING, CODING, REVIEWING, DONE, FAILED, PAUSED }
public enum TaskEvent {
  INIT, PLAN_APPROVED, PLAN_REJECTED, DESIGN_READY,
  CODE_SUBMITTED, AUDIT_PASSED, AUDIT_REJECTED,
  TEST_PASSED, TEST_FAILED, DEPLOY_SUCCESS, DEPLOY_FAILED,
  EMERGENCY_STOP, APPROVAL_GRANTED, APPROVAL_REJECTED,
  TIMEOUT, MAX_REVISION_EXCEEDED
}
3.1.2 流转表
From	To	Event	Guard	Action
PENDING	CODING	PLAN_APPROVED	审批通过 + 拆解合法	建执行上下文
PENDING	FAILED	PLAN_REJECTED / TIMEOUT	-	记原因
CODING	REVIEWING	CODE_SUBMITTED	Artifact 提交 + 编译通过	锁版本
REVIEWING	DONE	TEST_PASSED	QA 通过 + Audit 通过	归档
REVIEWING	CODING	AUDIT_REJECTED / TEST_FAILED	-	revision+1，生成重写工单
任意业务态	PAUSED	EMERGENCY_STOP	三级作用域校验	记 pausedFrom
PAUSED	pausedFrom	APPROVAL_GRANTED	审批通过	从 Checkpoint 恢复
PAUSED	FAILED	APPROVAL_REJECTED / TIMEOUT	-	告警 + Incident
任意	FAILED	MAX_REVISION_EXCEEDED	-	Incident
3.1.3 核心方法
java
@Transactional
public TransitionResult transition(String taskId, TaskEvent event,
                                   Operator op, String traceId, String idempotencyKey) {
    if (idempotencyStore.exists(idempotencyKey)) return idempotencyStore.get(idempotencyKey);
    Task task = taskRepo.lockById(taskId);
    Transition def = registry.find(task.status(), event)
        .orElseThrow(() -> new IllegalTransitionException(task.status(), event));
    if (!def.guard().test(task)) throw new GuardFailedException(def.guard().name());
    TaskStatus prev = task.status();
    task.setStatus(def.to());
    if (def.to() == CODING) task.setRevision(task.getRevision() + 1);
    if (def.to() == PAUSED) task.setPausedFrom(prev);
    taskRepo.save(task);
    transitionRepo.save(new TaskTransition(taskId, prev, def.to(), op, traceId));
    audit.record(...);
    outbox.enqueue(new TaskStatusChanged(taskId, prev, def.to(), traceId));  // 事务后投递
    idempotencyStore.put(idempotencyKey, result);
    return result;
}
3.1.4 关键实现点
并发：行锁 + 乐观锁版本号兜底。

PAUSED 恢复：通过 pausedFrom + Checkpoint 恢复（见 3.5）。

非法流转：409 ILLEGAL_TRANSITION。

可回放：task_transition + traceId 重建。

3.2 技能注册与执行管道
3.2.1 注解与 Capability Manifest
java
@Retention(RUNTIME) @Target(METHOD)
public @interface AgentSkill {
    String name();
    RiskLevel risk() default LOW;
    String[] requiredPerms() default {};
    Capability[] capabilities();       // 必填：FILE_WRITE/NET_ACCESS/PROCESS_EXEC...
    boolean sideEffect() default false; // 恢复去重关键
    Class<?> inputSchema();
    Class<?> outputSchema();
    SandboxPolicy sandbox() default SandboxPolicy.NONE;
}
3.2.2 执行管道
java
public SkillResult invoke(ToolCall call, AgentContext ctx) {
    SkillDescriptor d = registry.get(call.name());
    schemaValidator.validate(d.inputSchema(), call.arguments());

    // 能力清单硬校验（防反射绕过）
    CapabilityProbe.assertWithin(call, d.capabilities());

    // 双层权限
    permissionEngine.assertAllowed(ctx.agent(), d.requiredPerms());

    // 风险
    if (riskPolicy.requiresApproval(d.risk(), d.requiredPerms())) {
        ApprovalTicket t = approvalService.request(SKILL_EXEC, ctx, d, call);
        return SkillResult.pendingApproval(t.id());
    }

    // 沙箱执行 + 实际能力再校验
    SandboxResult sr = sandboxExecutor.execute(d.sandboxPolicy(), () ->
        reflectionInvoker.invoke(d.bean(), d.method(), call.arguments()));

    // 副作用记录（用于 Checkpoint 恢复去重）
    if (d.sideEffect()) sideEffectRepo.record(ctx.sessionId(), call.id(), sr.summary());

    audit.record(SKILL_INVOKED, ctx, d.name(), sr.summary());
    invocationLogRepo.save(InvocationLog.of(ctx.sessionId(), ctx.taskId(), ctx.agentId(), d.name(), call));
    schemaValidator.validate(d.outputSchema(), sr.output());
    return SkillResult.success(sr.output());
}
任何步骤失败 → SkillResult.failure，后续不执行。

3.3 数字镜像 RBAC
java
public Set<String> effectivePerms(Agent agent) {
    return intersect(userPermCache.get(agent.getOwnerUserId()),
                     agentPermCache.get(agent.getId()));
}
public void assertAllowed(Agent agent, String... required) {
    Set<String> eff = effectivePerms(agent);
    for (String p : required) if (!eff.contains(p))
        throw new ForbiddenException(agent.getId(), p);
}
缓存失效：UserPermChanged / AgentGrantChanged 事件驱动；TTL 5 分钟兜底。

实时中止：长时技能（>5s）注册 RevocableTask，订阅权限变更事件立即中断。

AOP 切面作为 SkillExecutor 之外的第二道保险。

3.4 审批网关
3.4.1 审批人规则（ApprovalPolicy）
按 风险等级 + 操作类型 解析审批人集合；

HIGH/CRITICAL 满足四眼原则：至少 2 名不同审批人（含 accountable），或 1 人 + 冷静期；

禁止 requester == approver；

解析失败 → 升级租户管理员 + 告警；

超时默认拒绝。

3.4.2 创建与回调
java
public ApprovalTicket request(ApprovalType type, AgentContext ctx, Object payload) {
    ApprovalTicket t = new ApprovalTicket();
    t.setType(type); t.setTaskId(ctx.taskId()); t.setRequester(ctx.agentId());
    t.setRiskLevel(riskPolicy.of(type));
    t.setPayloadJson(serializer.toJson(payload));
    t.setPlainText(translator.toPlain(payload));               // 双轨：大白话解释
    t.setStatus(PENDING);
    t.setExpiresAt(now.plus(approvalTimeout));
    t.setIdempotencyKey(type + ":" + ctx.taskId() + ":" + payloadHash);
    ticketRepo.saveIfAbsent(t);
    ws.publish(approval.requested, t);
    return t;
}
批准：事务内 APPROVED + 状态机流转；事务后 Outbox 回调（恢复执行/继续技能）。

超时：ApprovalScheduler 每 30s 扫描 → EXPIRED → 任务 FAILED + IncidentTicket。

幂等：idempotency_key UNIQUE。

3.5 Agent 循环与 Checkpoint（恢复一致性）
java
for (int round = 0; round < maxRounds; round++) {
    LlmResponse resp = llm.chat(buildMessages(ctx, history), tools);
    if (resp.hasNoToolCall()) break;
    for (ToolCall call : resp.toolCalls()) {
        SkillResult r = skillExecutor.invoke(call, agentCtx);
        history.add(toolResultMessage(call.id(), r));
        checkpoint.save(sessionId, round, historyHash, sideEffects);
        if (r.requiresApproval()) { stateMachine.pause(taskId); return PAUSED; }
    }
}
恢复：从最近 Checkpoint 继续；已完成 tool_call 结果按 toolCallId 幂等复用；

副作用去重：sideEffect=true 的技能先查 SideEffectRecord，存在则跳过实际执行，直接复用结果；

超轮次：escalate 给 accountable，FAILED(AGENT_LOOP_EXCEEDED)。

3.6 沙箱
java
public record SandboxPolicy(boolean network, int cpuMillis, int memoryMb,
                            int timeoutSec, boolean readOnlyRoot,
                            String mountWorkdir, List<String> denySyscalls) {
  static final SandboxPolicy NONE = ...;       // 纯内存技能
  static final SandboxPolicy WORKSPACE = ...;  // 编译/测试
  static final SandboxPolicy SHELL = ...;      // 高危，无网只读
}

public SandboxResult execute(SandboxPolicy p, Callable<Object> task) {
    if (p == NONE) return runInProcess(task);
    Container c = docker.create().image(BASE)
        .network(p.network() ? "bridge" : "none")
        .readOnlyRoot(p.readOnlyRoot())
        .mount(p.mountWorkdir(), "/workspace")
        .cpuLimit(p.cpuMillis()).memoryLimit(p.memoryMb())
        .tag("tenantId=" + tenantId).build();  // 租户隔离
    try (c) { return c.run(task, Duration.ofSeconds(p.timeoutSec())); }
    catch (TimeoutException e) { throw new SkillExecutionException("SANDBOX_TIMEOUT"); }
}
能力清单硬校验：容器内 syscall/文件/网络行为必须在声明的 Capabilities 内，越界中止；

权限吊销中断：订阅事件 → kill 容器；

超时/逃逸 → 立即销毁 → FAILED + 告警。

3.7 双向双轨翻译
3.7.1 Agent → 人类（术语 → 大白话）
术语原文与翻译同时落库；

翻译失败回退原文 + 标记 translationFailed；

高危解释模板强制包含：动作 / 影响范围 / 是否可逆 / 建议；

低温度 + 模板约束 + 长度上限。

3.7.2 人类 → Agent（回复 → 结构化意图）
java
public interface IntentParser {
    HumanIntent parse(String rawText, IntentContext ctx); // 受约束 LLM + 枚举
}
public enum IntentType { APPROVE, REJECT, REWRITE, MODIFY, QUESTION, OTHER }
解析失败回退“自由文本 + 人工确认”；

高风险意图（如对部署任务 REJECT）二次确认；

落 warroom_message{contentType=HUMAN_INTENT, plainText=原文}，双份保存。

3.8 War Room 与 WebSocket
消息模型：contentType ∈ {TERM, PLAIN, HUMAN_INTENT}，sequence 频道内单调递增；

租户隔离：频道 key = tenantId:channelId，订阅强校验；

断线重连：客户端带 lastSequence，服务端补发增量；

翻译失败回退原文并标记。

3.9 自进化：三次法则
3.9.1 日志与归一化
sql
CREATE TABLE invocation_log (
  id TEXT PRIMARY KEY, session_id TEXT, task_id TEXT, agent_id TEXT,
  skill_name TEXT, args_hash TEXT, seq INTEGER, created_at TIMESTAMP
);
argsHash = sha256(normalize(args))（去空、排序 key、时间戳/ID 归一化）。

3.9.2 检测算法
窗口：同 session 最近 500 条 / 最近 30 分钟；

滑窗 n-gram（n∈[2,5]）；

次数 ≥ 3 且不在 24h 冷却期；

生成 SkillCandidate(DRAFT) + 发送 ABSTRACT_SKILL；

异步生成候选，不阻塞业务（ADR C3）。

3.9.3 候选审查与上架
权限扩张检查：requiredPerms ⊆ union(sourceSkills.requiredPerms) ∩ ownerPerms；

高危检测：含 PROCESS_EXEC → 降级为“每次需审批”；

复杂度超阈值 → 人工重点审查；

人类批准 → SkillRegistry.register(status=ACTIVE) → 审计 → 归档清理（见 3.9.4）。

3.9.4 上下文清理（避免与审计冲突）
三层留存：

audit_log（永不删）；

invocation_summary（按会话聚合，保留 N 天）；

invocation_log_detail（可删，清理前归档冷存储）；

清理动作本身记审计。

3.10 审计与可观测
3.10.1 审计约束
audit_log 只 insert，禁止 update/delete（DB 层触发器/权限约束）；

所有审计带 traceId，可回放；

关键动作（状态变更、权限判定、技能执行、审批、自进化、清理）全覆盖。

3.10.2 指标
task_transition_duration、approval_pending_count、approval_pass_rate、permission_denied_total、sandbox_timeout_total、skill_candidate_adoption_rate、llm_token_total、agent_loop_rounds、incident_open_count。

3.10.3 桌面宠物状态映射
Agent 状态	动画
IDLE	站立
CODING	敲键盘
REVIEWING	看报告
WAITING_APPROVAL	举手 / 抛按钮
ERROR	冒烟
CRASHED	流泪
PAUSED	抱头蹲防
Part 4：安全设计专章（补强）
4.1 物理级急停与本地 KillSwitch
背景：README 要求“物理级急停”，纯 HTTP 依赖网络，最需要停的场景可能恰是后端/网络异常。

设计：

桌面端（Tauri/JavaFX）内置 KillSwitch：

本地热键（如 Ctrl+Shift+F12）+ 常驻按钮；

直达后端 /api/emergency-stop 专用端点（独立线程池、无业务锁、优先响应，10ms 级返回）；

若 Agent 跑在桌面进程，直接 SIGTERM 本地进程，无需后端往返；

急停事件双写：本地日志 + 服务端审计，断网也留痕。

后端急停端点：独立线程池、无业务锁、仅校验 emergency:stop 专用权限。

前端降级提示：WS 断连时进入“急停不可用”红色告警，引导使用本地 KillSwitch。

4.2 急停三级作用域
作用域	触发	二次确认
TASK	按钮单击	无
AGENT	长按 2s	有（确认停该 Agent 全部任务）
TENANT	管理员专用	双人确认 + WS 全租户广播
4.3 抛按钮协议（human.action.requested）
json
{
  "type": "human.action.requested",
  "requestId": "har-001",
  "taskId": "t-001",
  "agentId": "agent-qa",
  "actions": [
    {"code": "REWRITE", "label": "打回重写", "api": "POST /api/tasks/t-001/rewrite", "risk": "LOW"},
    {"code": "STOP",    "label": "急停",     "api": "POST /api/tasks/t-001/pause",   "risk": "HIGH"}
  ],
  "expiresAt": "...",
  "plainText": "测试发现并发死锁，建议打回重写。"
}
前端维护“抛按钮队列”，同一时刻只展示最高优先级，其余进收件箱；

点击后以 requestId + idempotencyKey 回传，防重复触发。

4.4 No-Code UI（TaskFormSchema + WorkflowDefinition）
动态表单：TaskFormSchema（JSON Schema）驱动；

拖拽节点：可视化编辑子任务 DAG → 序列化为 WorkflowDefinition，由状态机校验合法性（只允许已注册的 state/event 组合）；

决策点：审批按钮由 human.action.requested 渲染，用户无需理解底层状态；

唯一出口：所有 UI 操作只产出事件，不直接写状态。

4.5 一人一 Agent 约束
(tenantId, ownerUserId) 在租户内对应唯一活跃 Agent；

允许按 role 建多个 Agent（如 PM-Agent 与 Dev-Agent），但同一 role 唯一；

Agent memory 按 owner 隔离，跨任务复用需显式“记忆共享”授权；

审计记录 agentId + ownerUserId 双字段。

4.6 兜底机制（IncidentTicket）
每个 Task 有 accountableUserId（默认创建者，可转交）；

FAILED / 审批超时 / 急停后生成 IncidentTicket 指派 accountable；

未处理逐级升级：owner → 团队管理员 → 租户管理员；

Agent 侧：循环超限/反复打回主动 escalate 给 owner，不静默失败；

转交链入审计。

4.7 注入防护（ContentSanitizer + Artifact 签名）
所有进入 Context 的外部内容（War Room 消息、翻译输出、Artifact）先过 ContentSanitizer：剥离指令性语句，标记 untrusted 区块；

Context 使用角色化分隔符（system/user/tool/untrusted），untrusted 只作数据不作指令；

Artifact 落库时 ArtifactSigner 用产出 Agent 的密钥 HMAC 签名，消费前验签，防伪造。

4.8 多租户隔离
WS 频道 key = tenantId:channelId，订阅强校验；

技能库两层：system skills（全局只读）+ tenant skills（租户私有）；

沙箱 volume = /sandbox/{tenantId}/{taskId}，容器带 tenant 标签；

Repository 层 TenantGuard：无 tenant 上下文直接拒绝。

4.9 审批规则强化
见 3.4.1：四眼原则、禁 requester==approver、解析失败升级。

4.10 权限吊销实时性
长时技能注册 RevocableTask；

UserPermChanged 事件 → 立即中断 + 沙箱 kill；

权限校验不仅准入前，长时技能执行中周期校验。

4.11 状态驻留超时（Watchdog）
状态	maxDwellTime	超时动作
PENDING	24h	告警 + Incident
CODING	4h	告警 + 升级 accountable
REVIEWING	2h	告警
PAUSED	24h	告警 + 升级
审批单	30m	EXPIRED + FAILED
人工显式延长动作记审计。

4.12 记忆隔离
task-scoped memory：任务结束封存（只读归档）；

agent-persistent memory：只存经验/偏好，禁存业务数据；

写入经 MemoryWritePolicy 过滤敏感字段。

4.13 事务与 Outbox
审批通过 + 状态流转同本地事务；

跨模块异步（WS/派发/回调）走 Outbox，事务提交后投递；

流转失败 → 审批单回滚 PENDING 并附原因。

4.14 审计不可删
audit_log 只 insert；DB 触发器/权限兜底；

三层留存（3.9.4）；

清理动作自审计。

Part 5：落地附录
5.1 数据模型（核心表 DDL）
sql
CREATE TABLE task (
  id TEXT PRIMARY KEY, tenant_id TEXT, title TEXT,
  status TEXT, revision INT DEFAULT 0,
  parent_id TEXT, owner_agent_id TEXT, current_agent_id TEXT,
  owner_user_id TEXT, accountable_user_id TEXT,
  risk_level TEXT, paused_from TEXT,
  created_at TIMESTAMP, updated_at TIMESTAMP
);

CREATE TABLE task_transition (
  id TEXT PRIMARY KEY, task_id TEXT, from_status TEXT, to_status TEXT,
  event TEXT, operator_type TEXT, operator_id TEXT,
  reason TEXT, trace_id TEXT, created_at TIMESTAMP
);

CREATE TABLE approval_ticket (
  id TEXT PRIMARY KEY, type TEXT, task_id TEXT, requester_agent_id TEXT,
  risk_level TEXT, payload_json TEXT, plain_text TEXT,
  status TEXT, approvers TEXT, operator_id TEXT,
  expires_at TIMESTAMP, idempotency_key TEXT UNIQUE, created_at TIMESTAMP
);

CREATE TABLE skill (
  id TEXT PRIMARY KEY, name TEXT UNIQUE, risk TEXT,
  required_perms TEXT, capabilities TEXT, side_effect BOOLEAN,
  sandbox_policy TEXT, status TEXT, version INT
);

CREATE TABLE skill_candidate (
  id TEXT PRIMARY KEY, pattern_hash TEXT, source_task_ids TEXT,
  draft_code TEXT, required_perms TEXT, capabilities TEXT,
  risk TEXT, status TEXT, review_note TEXT, created_at TIMESTAMP
);

CREATE TABLE agent_checkpoint (
  id TEXT PRIMARY KEY, session_id TEXT, round INT,
  history_hash TEXT, side_effect_keys TEXT, created_at TIMESTAMP
);

CREATE TABLE side_effect_record (
  id TEXT PRIMARY KEY, session_id TEXT, tool_call_id TEXT,
  result_json TEXT, created_at TIMESTAMP
);

CREATE TABLE incident_ticket (
  id TEXT PRIMARY KEY, task_id TEXT, accountable_user_id TEXT,
  reason TEXT, status TEXT, escalated_to TEXT, created_at TIMESTAMP
);

CREATE TABLE audit_log (
  id TEXT PRIMARY KEY, trace_id TEXT, actor_type TEXT, actor_id TEXT,
  action TEXT, target TEXT, result TEXT, detail_json TEXT, created_at TIMESTAMP
);
CREATE INDEX idx_audit_trace ON audit_log(trace_id);
-- 禁止 UPDATE/DELETE：由 DB 权限或触发器约束

CREATE TABLE invocation_log (
  id TEXT PRIMARY KEY, session_id TEXT, task_id TEXT, agent_id TEXT,
  skill_name TEXT, args_hash TEXT, seq INTEGER, created_at TIMESTAMP
);
CREATE INDEX idx_inv_session_seq ON invocation_log(session_id, seq);
5.2 API 草案
方法	路径	说明
POST	/api/tasks	创建任务（No-Code UI 提交）
GET	/api/tasks/{id}	详情
POST	/api/tasks/{id}/approve-plan	同意拆解
POST	/api/tasks/{id}/pause	任务级急停
POST	/api/tasks/{id}/rewrite	打回重写
POST	/api/agents/{id}/pause	Agent 级急停（长按确认）
POST	/api/tenant/emergency-stop	租户级（管理员 + 双人确认）
POST	/api/emergency-stop	物理级急停专用端点（独立线程池）
GET	/api/approvals	审批列表
POST	/api/approvals/{id}/approve	批准（幂等）
POST	/api/approvals/{id}/reject	拒绝
GET	/api/incidents	兜底工单
GET	/api/skills	技能库
GET	/api/skills/candidates	候选
POST	/api/skills/candidates/{id}/approve	上架候选
GET	/api/warroom/{channelId}/messages	消息历史
5.3 WebSocket 事件
task.updated / agent.status.changed / approval.requested / approval.resolved / warroom.message.created / skill.candidate.created / human.action.requested / incident.created / system.emergency.stop

均带 traceId、tenantId、timestamp、sequence。

5.4 关键配置
yaml
agenthub:
  llm: { provider: openai-compatible, timeout: 60s, max-retries: 2 }
  agent: { max-rounds: 20, max-revision: 5 }
  approval:
    timeout: 30m
    default-on-timeout: REJECT
    require-four-eyes: [HIGH, CRITICAL]
    forbid-self-approval: true
  sandbox: { image: agenthub/sandbox:latest, default-timeout: 60s }
  evolution: { window-size: 500, min-occurrence: 3, cooldown: 24h }
  security:
    digital-twin-strict: true
    sanitize-external-content: true
    artifact-signature: true
  watchdog:
    dwell: { PENDING: 24h, CODING: 4h, REVIEWING: 2h, PAUSED: 24h }
  workflow:
    default-serial-audit-qa: true
5.5 错误码
错误码	HTTP	场景
ILLEGAL_TRANSITION	409	非法流转
PERMISSION_DENIED	403	RBAC 拒绝
APPROVAL_REQUIRED	202	高危挂起
APPROVAL_TIMEOUT	410	审批超时
SANDBOX_TIMEOUT	504	沙箱超时
SKILL_SCHEMA_INVALID	400	Schema 不符
CAPABILITY_VIOLATION	403	超出能力清单
MAX_REVISION_EXCEEDED	409	打回超限
AGENT_LOOP_EXCEEDED	500	循环超限
REVOKED_DURING_EXEC	409	执行中权限被吊销
5.6 测试清单
层	用例
状态机	非法流转、并发、幂等、PAUSED 恢复、revision 递增、驻留超时
权限	owner 收缩后失效、requiredPerms 不足、缓存失效、吊销实时中止
审批	超时拒绝、四眼、禁止自批、重复批准幂等、急停恢复
沙箱	超时、无网、只读、资源限制、能力越界、逃逸
注入	sanitize 生效、untrusted 隔离、Artifact 伪造被拒
多租户	WS 越权订阅、技能库隔离、沙箱 volume 隔离
恢复	Checkpoint 续跑、副作用去重
自进化	三次触发、权限扩张否决、高危降级、冷却期、异步不阻塞
事务	审批+流转一致、Outbox 投递、流转失败回滚
兜底	Incident 生成、升级链
双向翻译	术语→大白话、人类回复→意图、解析失败回退
物理急停	断网下本地 KillSwitch、专用端点优先级
5.7 实施路线（含安全项优先级）
阶段	交付	必做安全项
Phase 1 单体	状态机 + 技能管道 + PM 闭环	A1 物理急停、A4 一人一 Agent、A5 兜底、B5 Checkpoint、B8 Watchdog、B10 事务
Phase 2 权限	@RequirePerm + 数字镜像	A2 抛按钮协议、A8 Capability Manifest、B1 注入防护、B3 四眼、B6 吊销实时
Phase 3 通讯	双轨 + War Room	A3 No-Code UI、A6 反向翻译、B2 租户隔离、B4 三级作用域、B9 记忆隔离
Phase 4 桌面	悬浮窗 + 状态映射	KillSwitch 桌面端集成
Phase 5 开源	治理中间件剥离	A9 审计分层、A10 开源边界
5.8 ADR（原文件歧义决议）
编号	歧义	决议
C1	Step 4 只提 QA 未提 Audit	默认 Audit 强制，模板可关（记录决策）
C2	QA FAILED 是子任务还是主任务	子任务 FAILED，主任务 REVIEWING→CODING 打回
C3	自进化是否阻塞任务	异步生成候选，不阻塞业务
C4	SHELL_EXEC 谁挂起	仅发起者任务挂起，其他 Agent 继续
C5	Agent 链 >> 是否强串行	默认串行；WorkflowDefinition 显式声明依赖
5.9 变更记录
版本	变更
v1.0	首次成文；融合主流程、Agent 链、底层实现、安全补强（A1-A10、B1-B10、C1-C5）；确定 Agent 链串行、审批-流转同事务、Checkpoint 恢复、双向双轨、物理急停等关键决策
落地顺序（最终建议）：

task + task_transition + StateMachineService（含 Outbox、Watchdog、幂等）；

@AgentSkill + Capability Manifest + SkillRegistry + SkillExecutor 管道；

PermissionEngine + ApprovalGateway（四眼、禁自批、超时拒绝）；

AgentCheckpoint + 副作用去重 + ContentSanitizer + ArtifactSigner；

PM 闭环跑通，再依次接入 Arch/Dev/Algo/Audit/QA/Ops 与沙箱；

War Room 双向双轨 + 三次法则（异步）；

桌面端 KillSwitch + 状态映射；

Phase 5 按开源边界剥离 agenthub-core。