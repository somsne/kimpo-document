# 流程引擎（代号 凯瑟琳/Katheryne）架构原理 —— 面向开发者

> 读者：想理解 Kimpo 流程引擎内部机理、或要与它集成（程序化发起流程、贡献取人能力、订阅流程事件）的**人类工程师**。
> 读完你会明白：引擎为什么长在插件里而不是宿主里、一张单据从提交到定局在系统里经历了什么、状态机与操作框架如何保证并发安全与可审计、以及你的代码可以从哪几个口子接进来。
> 配套：使用视角的白话版在 [for-end-users.md](for-end-users.md)；给 AI 编程助手的结构化契约版在 [for-llm-agents.md](for-llm-agents.md)。

---

## 1. 总体架构：引擎是一个"什么都不亲自干"的插件

凯瑟琳是一个 **category=workflow 的 Go sidecar 插件**，不是宿主模块。它与宿主之间的角色切割非常干净：

```
┌─ Kimpo 宿主（框架层）────────────────────────────────────────┐
│ 通用基建: 取人查询 / 定时任务 / 站内通知 / 事件总线(outbox)     │
│ 单据集成: flow_status 系统列 + 提交编排 + 编辑锁定三层合成       │
│ 数据面:   WorkflowStore(十表白名单 CRUD+原子变更集)             │
│ REST 面:  workflowproxy(/api/v1/apps/{app}/workflow/*)        │
│           workflowtasks(/api/v1/workflow/tasks|badges, 跨库聚合)│
└──────┬───────────────────────────────▲──────────────────────┘
       │ 正向: WorkflowPluginService.Call(op, payload)          │ 反向(SDK Host):
       │ (定义 CRUD/发起/办理/回调 全走这一个 gRPC 方法)          │ WorkflowStore/Directory/
       ▼                                                        │ Scheduler/Notification/
┌─ 凯瑟琳 sidecar（流程语义唯一持有者）─────────────────────────┐│ EventBus/RecordFlowStatus
│ 定义域(草稿/发布/版本) · 状态机(Advance) · 操作框架(ExecuteAction)││
│ 取人解析(十规则→去重→兜底) · 多人模式 · 并行/子流程/延时/动作    │┘
└──────────────────────────────────────────────────────────────┘
```

**分工铁律**（理解一切设计决策的钥匙）：

- **引擎不亲自干任何"脏活"**：不读表单结构、不解析组织架构、不执行 SQL、不发通知、不起定时线程——全部经 SDK 反向通道委托宿主。引擎唯一持有的是**流程语义**（图怎么走、任务怎么算完、谁该审）。
- **宿主不理解流程**：宿主看不懂节点/连线/会签，它只认识三样东西——插件的 gRPC 契约、`app_workflow_binding` 协议表（"这个模板绑了哪条流程"）、单据上的 `flow_status` 投影列。
- **数据只有一个入口**：引擎的十张表全部经宿主 WorkflowStore 数据面读写（插件没有数据库句柄），表/列白名单 + 原子变更集 + 乐观锁都由这一层强制。

### 1.1 单方法 gRPC 信封

插件对宿主只暴露**一个** gRPC 方法：

```
WorkflowPluginService.Call(op string, app_id, actor_id, payload bytes)
  → { ok, payload } | { ok:false, error_code, message_key, message_fallback }
```

一切能力（定义 CRUD、发起、办理、任务查询、定时回调）都是一个 `op` 字符串 + JSON payload。好处：后续加功能 proto 零改动；业务错误全走响应体（gRPC 层恒 OK），宿主代理只做通用的错误码 → HTTP 状态映射，永远不需要理解新错误的含义。

### 1.2 宿主侧两个 REST 代理

- **workflowproxy**：`/api/v1/apps/{appID}/workflow/*` → 哑管道直通 sidecar（设计器的定义存取走它）。惰性连接：HTTP 路由启动时占位，每次请求时解析 sidecar 地址（sidecar 是异步拉起、热重载会换端口的）。
- **workflowtasks**：任务中心聚合面。因为流程数据**分库存放在各应用库**（见 §2），"我的全部待办"需要宿主遍历应用库逐库调 sidecar 再合并——单库失败降级跳过不拖垮整页。

---

## 2. 数据模型：十张表，全在应用库

流程定义与运行时数据**全部放应用库**（不在实例库）——应用自包含、与业务单据同库、备份分发一体。代价是任务中心要跨库聚合（§1.2），0.1 阶段可接受。

**定义域三表**：

| 表 | 干什么 | 关键点 |
|---|---|---|
| `app_workflow` | 流程登记 + **草稿**（draft_json） | workflow_key 是对外稳定标识 |
| `app_workflow_version` | **发布版本快照** | 版本号自增；**发布后行不可变**（引擎纪律保证，因此解析缓存永不失效） |
| `app_workflow_binding` | 对象绑定（模板 ↔ 流程） | 协议表：宿主提交编排直查它（这是宿主唯一"知道"的流程表） |

**运行时七表**：

| 表 | 干什么 | 关键点 |
|---|---|---|
| `wf_instance` | 流程实例 | 钉 base_version/effective_version 双版本；`version_no` 乐观锁 |
| `wf_task` | 任务（审批/抄送/系统留痕三种 kind） | node_snapshot 冻结节点配置快照；`version_no` 乐观锁 |
| `wf_task_candidate` | 候选人倒排 | (user_id, task_id) 复合主键，"我的待办"毫秒级查询 |
| `wf_action_log` | 操作流水 | **append-only（数据面强制禁 UPDATE/DELETE）**；幂等键唯一索引在这 |
| `wf_delegation` | 委托代理规则 | 时间窗 + 作用域 |
| `wf_comment` / `wf_variable` | 评论 / 实例变量 | 变量承载动作出参回填、子流程传参、join 计数等 |

ID 体系：实例/任务用 **ULID**（时间有序、无中心发号）；app_id 是 UUID 字符串。

---

## 3. 定义域：草稿 → 发布 → 不可变版本

- **保存草稿**：设计器整图 JSON 存 `app_workflow.draft_json`，无版本概念，随便存。
- **发布**：服务端先做 **R1 规范化**（给无后续节点的末端自动补隐式 end 并连线——与前端同一算法的 Go 镜像，幂等可重放），再跑**发布校验**（15 条规则，与前端 validate.ts 逐条镜像：唯一 start、可达性、无环、分支组兜底必选、审批节点必配参与者/按钮、reject 必配 effect……**双端镜像是纪律：改任何一端校验必须同步另一端**），全过才 INSERT 一行新版本。
- **版本不可变**：发布行永不 UPDATE。运行时按 `(workflowKey, versionNo)` 做进程内缓存，因不可变所以**永不失效**——这是"在途实例钉版本"能零成本实现的根基。
- **并发发布**：版本号 = max+1 抢号，撞 UNIQUE(workflow_id, version_no) 的一方收到可重试的 CONFLICT。

一个值得知道的细节：发布校验失败返回的是 **op 级成功 + 业务级 `{ok:false, errors:[...]}`**（不是错误码）——因为错误信封只有三个字符串字段，装不下逐节点定位的错误数组，而前端需要它们做点击定位。

---

## 4. 运行时状态机：一个推进内核，两条铁律

### 4.1 状态全集

- 实例：`1 processing → 2 approved / 3 terminated / 4 withdrawn`
- 任务：`0 pending → 1 claimed → 2 done / 3 cancelled`；kind：`1 审批(含办理) / 2 抄送 / 3 系统留痕`（办理节点与审批共用 kind=1，语义靠 node type 区分——三面孔复用同一状态机）；sub_state 标记 `from_backward(退回复活) / resubmit(重提重走)`。

### 4.2 Advance：唯一推进内核

所有"流程向前走"收敛到一个函数：**Advance（给定某任务已完成，沿图推进直到停在下一个等待态）**。发起、同意、自动节点完成、定时回调，全都最终调它。推进循环：

```
取当前节点出边 → 条件边逐条求值(经宿主表达式服务)，全不中走默认线
→ 目标节点按类型分派 NodeExecutor：
   人任务  → 解析参与者 → 按多人模式建任务 → 停(等待态)
   抄送    → 建抄送任务，不停，继续推进
   并行-分 → 全部出边同时递归推进
   并行-合 → 记到达留痕；未齐→停本支线；齐→继续
   动作/延时/子流程 → 登记副作用(定时/子实例)，建系统等待任务 → 停
   终止    → 实例 terminated + 活跃任务全 cancelled + 单据→rejected
   结束    → 若还有其他活跃任务(并行在途)只结束本支线；否则定局四件套
```

### 4.3 两条铁律

**① 等待态之间单事务**。从一个等待态到下一个等待态之间的**全部写**（任务状态变更 + 新任务 + 候选行 + 流水 + 实例变更）先在内存里收集成一个"变更计划"（changePlan），最后**一次**提交给 WorkflowStore 的原子变更集。任何中间失败整体回滚——不存在"任务改了状态但下一站没建出来"的半截状态。事件发布、定时注册、flow_status 迁移这类**副作用统一放在事务成功之后**执行（事件经 outbox 表保证至少一次；定时注册失败仅告警——提醒是尽力而为侧路，不值得回滚业务事务）。

**② 乐观锁贯穿**。`wf_instance`/`wf_task` 的每次 UPDATE 都带 `version_no` 期望值（CAS），冲突方收到可重试的 CONFLICT。并行汇合的"谁最后到齐谁推进"竞态正是靠实例 CAS 裁决：两支线同时到齐时只有一方 CAS 成功并继续推进，另一方重读后发现已推进、只留痕不重复推进。

---

## 5. 操作框架：ExecuteAction 的三件套

所有人工干预（claim/forward/reject/退回族/撤回/转办/加签/催办/暂存/批量/管理员干预）走统一入口 `task.execute {taskId, action, idempotencyKey, opinion?, params?}`，入口处统一做三件事：

1. **身份校验**：claim 要求 ∈ 候选人；forward/reject 要求 = 认领人；通用前置拦截终态任务与非 processing 实例。
2. **幂等**：`wf_action_log` 上有 `UNIQUE(task_id, idempotency_key)`。执行前按幂等键查流水，命中直接返回"幂等重放"；并发双写时靠唯一索引兜底（事务失败 → 回读 → 按重放返回）。前端网络重试、用户双击因此天然安全。
3. **流水**：每个动作在同一事务里追加一条 append-only 流水（操作人/动作/后果/意见）。数据面层面禁止对 `wf_action_log` UPDATE/DELETE——审计链不可篡改是物理保证不是约定。

几个有代表性的动作语义：

- **forward（同意）**：先做多人模式的完成判定（会签数兄弟、比例签算票、依次链式续建），判定"这一站过了"才调 Advance；本任务 done + 流水 + Advance 产物合一个事务。
- **reject**：按钮上配的 effect 分派——terminate 复用终止计划；back_* 走退回算法。
- **退回**：当前层兄弟全 cancelled；目标站**新建**一条 `from_backward` 任务（参与者=原办理人、复制原节点快照），**绝不改历史行**。退回目标靠 `prev_task_id` 链上溯（该字段固定指向"触发本次推进的人任务"，抄送不算一步）。重提默认直达退回点。
- **withdraw（撤回）**：窗口从严——除发起留痕外无 done 且无 claimed 才允许；执行后单据 flow_status 回 NULL。
- **batch_approve**：逐任务独立事务独立幂等键（外层 key 派生），pending 任务自动先 claim 再 forward，单条失败不中断其余。

---

## 6. 取人管线：解析 → 过滤 → 去重 → 兜底 → 委托

参与者解析是纯逻辑包（resolver），管线固定：

```
多规则并集(十种规则，经宿主 Directory 服务取数)
→ 在职过滤(离职剔除)
→ 去重: ①selfPolicy(候选=发起人: auto_pass/照常/转主管)
        ②repeatPolicy(∈上一步办理人→auto_pass, 可配全程/不去重)
→ 空兜底: auto_pass / to_admin / to_user / suspend
→ 委托检查: 每个最终候选查有效委托规则(时间窗+作用域)，命中则替换为受托人(不递归)
```

关键设计：**auto_pass 不是跳过**——照常建任务行（state=done、流水 auto_pass），保证进度图完整、去重可链式传导（第二级自动过后，第三级的 repeatPolicy 判定基于"实际办理人集合"依然正确）。十种规则里的 `capability` 是开放口：调用任意插件注册的能力动态算人（入参流程上下文，出参 userIds）——值班表、地区负责人矩阵这类组织之外的取人逻辑从这里接入。

---

## 7. 与单据的集成：三个衔接点

流程引擎与表单体系的耦合被压缩到三个精确的衔接点：

**① 提交链路（宿主编排五步）**：

```
前端 submit → 宿主: 查 binding(无→"没绑流程"正常返回)
→ RecordFlowStatus.Transition(NULL→processing)   ← 原子 CAS 先占状态，天然防双击双发起
→ 调插件 instance.start(建实例+首层任务)
→ 成功: 发布 record.submitted 事件
→ 失败: 补偿 Transition(processing→NULL)
```

**② flow_status 系统列（F8）**：单据主表上的 `flow_status VARCHAR(32) NULL` 投影列（NULL/processing/approved/rejected）。**写入口唯一**：宿主 RecordFlowStatusService 的原子 CAS，凯瑟琳是唯一调用方；表单数据权威层(Mirror/艾莉亚)与其他写通道对该列是黑名单。列表筛选"审批中的单据"就查这列——宿主无需理解流程。`flow_status=processing` 的单据在删除链路被拒删。

**③ 编辑锁定（F9，三层合成）**：填报态打开单据时宿主合成可写集：

```
第一层 record 状态锁: flow_status ∈ {processing,approved,rejected} ⇒ 默认整单只读
第二层 任务放行:      当前用户在该单据上有已认领的活跃任务 ⇒ 经插件 op 取该节点 fieldAcl.editable 白名单
第三层 权限交集:      ∩ 权限框架(卡侬)可写集
```

合成结果经 Mirror 的字段级写策略原语注入——拦截只在 Mirror 层（数据权威层）实施，编辑器前端的置灰只是体验优化。sidecar 不可达时**从严整单只读**（安全默认）。

---

## 8. 定时与事件：引擎没有自己的时钟

**引擎内没有任何轮询线程**。一切"到点做事"委托宿主定时服务：

```
任务创建(事务成功后) → Scheduler.RegisterJob(jobKey="task_timeout_<taskID>", fireAt, interval?)
宿主 Runner 到期 → 调度回调桥 → 反向调插件 Call(op="system.job_fire", {job_key, payload})
插件按 jobKey 前缀分派: task_timeout_(提醒) / task_escalate_(升级) / delay_(延时节点) / action_retry_(动作重试)
任务离开活跃态 → 统一注销相关 job(单点收口在任务状态变更处)
```

失败语义：回调返回 error → 宿主按至少一次语义退避重试，天然覆盖"宿主先起、sidecar 后起"的窗口。

**事件**走宿主 outbox 总线（落库派发、event_id 幂等、至少一次、死信兜底）。引擎发布 `workflow.instance.started/finished/terminated/withdrawn`、`workflow.task.created/cancelled/transferred` 等；**引擎只发事件不直调通知**——例如"任务取消要撤回站内通知"是通知模块自己订阅 `workflow.task.cancelled` 完成的联动，双方零耦合。

---

## 9. 结构节点的实现要点（速览）

- **并行网关**：图预处理做**支线分色**（split 出边序号序列标签，嵌套叠加），退回守卫据此判定"跨支线"并拒绝；join 到齐 = 数系统留痕行（每支线到达 join 记一行 kind=3），到齐判定与继续推进同事务，竞态靠实例 CAS。
- **子流程**：同步嵌套——父侧建系统等待任务（sub_instance_id 指向子实例，进度图下钻锚点）；子实例定局时回执推进父实例；**子实例不碰 flow_status**（单据状态归父实例唯一管理）；防环靠发起链变量（祖先 workflowKey 去重 + 深度上限）。
- **动作节点**：经能力总线 `Capability.Invoke` 调用插件能力，出参可回填实例变量（R10）；失败策略=重试(定时驱动)→错误分支边→转人工（生成管理员确认任务，确认后按正常边继续——人工确认即视为动作结果达成）。
- **分发填报**：对每个名单成员经宿主无头建单（SaveReportRecord Mode=create，预填种子字段），任务快照记子单据 id；全部交齐（复用会签完成判定）主流程才继续。

---

## 10. 对外扩展点：你的代码从哪接入

| 想做什么 | 接入方式 |
|---|---|
| **程序化发起流程**（定时发起、外部系统触发、数据满足条件自动发起） | ① 任何插件经 SDK `WorkflowStart.StartForRecord(appId, templateId, recordId, actorId)`——复用宿主完整提交编排（防双击占位/补偿都在）；② 动作编排里用现成的"发起流程"动作（支持"已有单据直发"与"先无头建单再发起"两种模式，**先落库再发起的顺序不可颠倒**） |
| **自定义取人逻辑** | 实现并注册一个能力（入参流程上下文 JSON，出参 `{userIds:[...]}`），设计者在取人规则里选"能力取人"引用它 |
| **监听流程事件做自动化** | ① 动作编排的"流程事件"触发时机（workflow-event）：流程实例结束/任务创建等事件自动触发模板上编排的动作；② 后端插件订阅事件总线的 `workflow.*` 事件 |
| **节点级钩子** | 节点上配置监听器（onEnter/onLeave 发布自定义事件 `workflow.node.<type>`），由上面的事件通道承接——引擎只发事件不内化任何逻辑 |
| **读流程数据做报表** | 运行时数据在应用库 wf_* 表（只读查询无妨）；跨应用聚合参考宿主 workflowtasks 的做法 |

**红线**（违反会破坏引擎不变量）：

1. **绝不直接写 wf_* 表**——绕过白名单/乐观锁/append-only 三重保护，状态机会精神分裂。
2. **绝不直接写 flow_status 列**——唯一写入口是 RecordFlowStatusService（凯瑟琳专用），手写会让单据状态与实例状态脱钩。
3. **发布版本行不可 UPDATE**——运行时缓存永不失效的前提。
4. **程序化发起必须"先落库、再发起"**——发起时刻单据必须已存在，引擎按 recordId 锚定取数。

---

## 11. FAQ

**Q：为什么引擎做成插件而不是宿主模块？**
A：角色分离与故障隔离。流程语义复杂多变（节点类型、干预操作会持续膨胀），装进插件让它可以独立演进、独立热重载、崩溃不拖垮平台；宿主只沉淀稳定的通用基建（取人/定时/通知/事件/数据面）。sidecar 崩溃时提交/办理临时不可用（返回 502），但已落库数据零损失——所有状态在库里，进程无状态。

**Q：并发双击"同意"会不会推进两次？**
A：不会，三重保险：前端同一面板复用幂等键 → 服务端幂等查询短路 → 数据面唯一索引兜底。

**Q：两个人同时抢一个或签任务？**
A：claim 是 CAS：一人成功，另一人收到 CONFLICT 与占用者信息。

**Q：宿主和 sidecar 之间会不会状态不一致？**
A：单据 flow_status 是投影不是权威——权威永远是 wf_instance。定局事务成功后投影迁移失败的窗口（极小）只影响列表筛选显示，有日志告警，管理员可干预修复；反过来的不一致不存在（先占位后发起，失败必补偿）。

**Q：想给流程加一种新节点类型要动哪里？**
A：五处：前端设计器（形状库+属性面板+types）、发布校验（**双端镜像**）、引擎 NodeExecutor 注册表、（若有新配置）节点 config 契约、文档。数据模型通常零改动——node_snapshot 是 JSON。
