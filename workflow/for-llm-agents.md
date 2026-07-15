# Workflow Engine (codename Katheryne) — Structured Context for LLM Agents

> Feed this entire document to your AI coding assistant when working with or integrating against the Kimpo workflow engine. It is the machine-oriented companion of [for-human-developers.md](for-human-developers.md) (principles) and [for-end-users.md](for-end-users.md) (usage).
> Everything below is contract-level fact, verified against the shipped implementation.

---

## FACTS

- F1. The workflow engine is a **Go sidecar plugin** (`plugin_id=com.kimposoft.workflow`, `category=workflow`), NOT a host module. It owns all workflow semantics (graph traversal, task lifecycle, participant resolution). The host owns generic infrastructure only.
- F2. The plugin exposes exactly **one gRPC method**: `WorkflowPluginService.Call(op, app_id, actor_id, payload) → {ok, payload | error_code, message_key, message_fallback}`. All capabilities are `op` strings + JSON payloads. Business errors travel in the response body; the gRPC layer always returns OK (transport errors only).
- F3. All workflow data lives in **10 tables in the per-app database** (not the instance DB). The plugin has NO database handle; every read/write goes through the host **WorkflowStore** gRPC facade (table/column whitelist + atomic change-sets + optimistic locking). Only `workflow`-category plugins may call it.
- F4. Published workflow versions are **immutable rows**. The engine caches parsed definitions per `(workflowKey, versionNo)` forever. In-flight instances are pinned to the version current at submit time (`wf_instance.base_version`; `effective_version` changes only via explicit `instance.sync_version`).
- F5. Between two wait-states, **all writes commit in ONE atomic change-set** (task state changes + new tasks + candidate rows + action log + instance updates). Side effects (events, scheduler jobs, flow_status transition) run only after the transaction succeeds.
- F6. `wf_instance` and `wf_task` updates always carry an optimistic-lock expectation (`version_no` CAS). Conflicts surface as retryable `CONFLICT`.
- F7. `wf_action_log` is **append-only, enforced by the host data plane** (UPDATE/DELETE rejected at the WorkflowStore layer). Idempotency uses `UNIQUE(task_id, idempotency_key)` on this table.
- F8. The record's main physical table carries a projection column `flow_status VARCHAR(32) NULL` (`NULL | processing | approved | rejected`). Its **only writer** is the host `RecordFlowStatusService` (atomic CAS), and the workflow plugin is that service's only caller. Mirror (Aria) and all other write channels blacklist this column. Records with `flow_status='processing'` cannot be deleted.
- F9. The engine has **no polling threads**. All timing (timeout alerts, escalation, delay nodes, action retries) is delegated to the host scheduler; the host calls back via `op="system.job_fire"` with at-least-once + backoff semantics. Job keys are prefixed: `task_timeout_<taskID>`, `task_escalate_<taskID>`, `delay_<instanceID>_<nodeID>`, `action_retry_<taskID>_<n>`.
- F10. The engine **publishes events, never sends notifications directly**. Events go through the host outbox bus (at-least-once, event_id idempotent, dead-letter). Notification revocation on task cancel is done by the notification module subscribing to `workflow.task.cancelled` — zero coupling.
- F11. Auto-passed tasks (dedup/self-policy/empty-fallback/HP3) are **real task rows** (state=done, action log `auto_pass`), not skips. Progress graphs and repeat-dedup chains rely on this.
- F12. Back/reject-back operations **never mutate historical rows**: the current layer's siblings are cancelled and a NEW task row is created at the target node (`sub_state=from_backward`, participants = original handlers, node_snapshot copied). `prev_task_id` points to the human task that triggered the advance (cc does not count as a step) — this fixed semantic is load-bearing for back-target resolution.
- F13. Withdraw window (strict): allowed only while no task except the start-marker is `done` and none is `claimed`. Withdraw sets instance `withdrawn` and flow_status back to NULL.
- F14. Handle nodes reuse `kind=1` (same state machine as approval); the "three faces" (approval/handle/cc) differ by node type in the definition, not by task kind. Handle rejects `action=reject`.
- F15. Subprocess instances **never touch flow_status** (record status is owned exclusively by the root instance). Cycle prevention: ancestor workflowKey chain in instance variable `subprocess_chain`, depth cap 10.
- F16. Parallel guard: back operations may not cross parallel branches (graph is pre-colored with branch labels; violations → `workflow.back.cross_branch`). `back_start` is allowed (targets pre-split). `parallel_split` out-edges are exempt from branch-group condition/default validation (they all fire).
- F17. Publish-validation is a **dual mirror**: `frontend/src/designer/model/validate.ts` and sidecar `internal/definition/validate.go` implement the same 15 rules. ANY change to one MUST be mirrored in the other.
- F18. Tenant note (0.1): host facades normalize tenant `instance_id <= 0` to `1` (single-tenant default). Plugin-side calls pass 0.

## DATA MODEL (app database)

```
app_workflow          (id AI, app_id, workflow_key UNIQUE-per-app, name, category, state, draft_json JSON, ...)
app_workflow_version  (id AI, workflow_id, version_no UNIQUE-per-workflow, schema_version, definition JSON,
                       status, published_by, published_at)          -- immutable after insert
app_workflow_binding  (id AI, workflow_id, object_type='template', object_ref=templateId, role, state)
wf_instance           (id ULID, app_id, template_id, record_id, workflow_id, base_version, effective_version,
                       status INT, start_user, start_at, end_at, final_user, final_at, final_opinion, version_no)
wf_task               (id ULID, instance_id, prev_task_id, node_id, node_path, kind INT, state INT, sub_state INT,
                       candidates JSON, claimed_by, deal_user, deal_at, node_snapshot JSON, summary,
                       alert_job_id, group_key, sub_instance_id, created_at, version_no)
wf_task_candidate     (task_id, user_id)  PK(user_id, task_id)      -- inbox inverted index
wf_action_log         (id AI, instance_id, task_id, action, effect, operator, operated_at, opinion, note,
                       payload JSON, idempotency_key)                -- APPEND-ONLY; UNIQUE(task_id, idempotency_key)
wf_delegation         (id AI, app_id, owner_user, delegate_to, scope 'all'|'workflow:<key>', start_at, end_at, state)
wf_comment            (id AI, instance_id, task_id, user_id, content, created_at)
wf_variable           (id AI, instance_id, var_key, var_value JSON, set_by, updated_at)
```

Reserved `wf_variable` keys (do not write): `subprocess_chain`, `join_arrivals_*`, `task_draft_<taskId>`, `voided`.

## STATE ENUMS

```
instance.status : 1 processing | 2 approved | 3 terminated | 4 withdrawn
task.kind      : 1 approval/handle | 2 cc | 3 system marker (start / join-arrival / subprocess-wait / delay-wait)
task.state     : 0 pending | 1 claimed | 2 done | 3 cancelled
task.sub_state : 0 normal | 1 from_backward | 2 resubmit
flow_status    : NULL | 'processing' | 'approved' | 'rejected'   (record projection column)
```

## OP INTERFACE (plugin `Call` ops)

Definition domain:
```
definition.list          {}                                  → [WorkflowMeta]            (top-level array)
definition.load          {workflowKey}                       → {meta, definition, draft…}
definition.save_draft    {workflowKey, definition}           → {ok}
definition.publish       {workflowKey, definition}           → {ok:true, versionNo} | {ok:false, errors:[{nodeId?,edgeId?,code,message,level}]}
                                                               NOTE: validation failure is op-level SUCCESS with business payload
definition.load_version  {workflowKey, versionNo}            → DefinitionDoc             (no wrapper)
```

Runtime domain:
```
instance.start        {workflowKey, templateId, recordId}    → {instanceId, baseVersion}
                       errors: OP_INVALID workflow.start.no_published_version | CONFLICT workflow.start.already_processing
instance.progress     {instanceId}                           → {instance, tasks[]}
instance.sync_version {instanceId, targetVersion}            → anchor-check per active task (node exists & same type in target)
instance.variables    {instanceId}                           → non-reserved variables
task.execute          {taskId, action, idempotencyKey, opinion?, note?, params?}
                      → {ok, taskState, instanceStatus, idempotentReplay}
task.acl_for_record   {recordId, userId}                     → {hasActiveTask, taskId, fieldAcl{editable,hidden}}  (F9 layer-2)
task.batch_approve    {taskIds[], idempotencyKey}            → {succeeded[], excluded[], failed[]}  (auto-claims pending)
task.query / task.badge / task.detail                        (task-center; userId = envelope ActorID, never from payload)
delegation.upsert     {id?, delegateTo, scope, startAt, endAt} (owner = ActorID)
delegation.list       {}
system.job_fire       {job_key, payload}                     (host scheduler callback; dispatch by key prefix)
system.ping           {}                                     → {pong, plugin_version}
```

`task.execute` actions:
```
claim | unclaim | forward | reject | back_prev | back_start | back_to(params.targetNodeId)
withdraw | void | transfer(params.toUserId) | urge | save_draft(params.draft)
add_sign_before | add_sign_after | add_sign_parallel (params.userIds) | remove_sign(params.targetTaskId)
admin_terminate | admin_transfer | cc_read
```
Idempotency key required for all except claim/unclaim (CAS-natural). withdraw/void anchor on the instance's start-marker taskId.

## REST INTERFACE (host proxies, session-cookie auth)

```
# Definition (workflowproxy → sidecar, dumb pipe)
GET    /api/v1/apps/{appID}/workflow/workflows
GET    /api/v1/apps/{appID}/workflow/workflows/{key}
PUT    /api/v1/apps/{appID}/workflow/workflows/{key}/draft            body {definition}
POST   /api/v1/apps/{appID}/workflow/workflows/{key}/publish          body {definition}
GET    /api/v1/apps/{appID}/workflow/workflows/{key}/versions/{n}

# Submit chain (host orchestration, NOT a proxy)
GET    /api/v1/apps/{appID}/templates/{tplID}/workflow-binding        → {hasWorkflow, workflowKey}
POST   /api/v1/apps/{appID}/templates/{tplID}/records/{recordID}/submit
       → {submitted:true, instanceId} | {submitted:false, reason:"no_workflow"} | 409 already in workflow
       Steps: check binding → flow_status CAS NULL→processing → instance.start → publish record.submitted
              (failure compensates back to NULL)

# Task center (workflowtasks, cross-app aggregation)
GET    /api/v1/workflow/tasks?tab=inbox|done|cc|initiated&page&size   → {items[], total}
GET    /api/v1/workflow/badges                                        → {inbox, cc}
GET    /api/v1/workflow/tasks/{appId}/{taskId}                        → detail {task, nodeSnapshot, definition, instanceTasks, draft}
POST   /api/v1/workflow/tasks/{appId}/{taskId}/execute                body = task.execute payload
POST   /api/v1/workflow/tasks/{appId}/{taskId}/save-draft             body {draft}
```

Error-code → HTTP mapping (workflowproxy): `OP_UNKNOWN|NOT_FOUND→404, OP_INVALID|SCHEMA_UNSUPPORTED→400, CONFLICT→409, PERMISSION_DENIED→403, INTERNAL→500, transport→502 PLUGIN_UNAVAILABLE`.

## NODE CONFIG SCHEMA (definition JSON, frontend `types.ts` is the frozen authority)

```jsonc
DefinitionDoc: { schemaVersion: 1, nodes: [{id, type, config, …layout}], edges: [{id, from, to, condition?, isDefault?, kind?}] }

node.type ∈ start | approval | handle | cc | terminate | parallel_split | parallel_join
          | action | subprocess | dispatch | manual_route | delay | end

// Human nodes (approval/handle/cc):
config = {
  assignee: {
    rules: [{kind, params}],
      // kind ∈ user | role | direct_manager | dept_manager | initiator | form_field
      //        | manager_chain | prev_handler | self_select | capability
    multiMode: "countersign"|"anyone"|"sequential"|"ratio",   // designer default: anyone
    ratioPercent?: 1..100,
    vetoPolicy: "immediate"|"wait_all",                        // ratio+immediate = mathematical short-circuit
    emptyPolicy: "auto_pass"|"to_admin"|"to_user"|"suspend", emptyToUserId?,
    dedup: { selfPolicy: "auto_pass"|"still_review"|"to_manager",
             repeatPolicy: "consecutive"|"first_only"|"always" }
  },
  buttons?: [{label, action:"forward"|"reject", effect?:"terminate"|"back_prev"|"back_start"}],
      // reject REQUIRES effect (publish validation)
  fieldAcl?: {editable:[], hidden:[], required?:[]},           // required = handle-node mandatory fill
  opinion?: {enabled, required, title},
  summary?: {mode:"none"|"format"|"expr", format?, expr?},
  timeout?: {alertAfterHours?, repeatEveryHours?,
             escalate?: "none"|"auto_pass"|"auto_reject"|"auto_transfer"|"mark",
             escalateAfterHours?, escalateTransferUserId?,
             notifyTargets?: ("current"|"prev"|"initiator")[]},
  autoPassExpr?: string,                                       // HP3
  backClearFields?: string[],                                  // fields to re-fill after back
  resubmitMode?: "direct"|"restart"                            // default direct (jump back to rejecting node)
}

// action:      {capabilityId, paramMapping, retryTimes?, onFailure?: "error_edge"|"manual", outputVars?}
// subprocess:  {workflowId, paramIn?, paramOut?}              // engine also accepts workflowKey/inputVars/outputVars
// dispatch:    {assignee, templateId, paramMapping?}          // engine also needs dataTableId for field seeding
// manual_route:{candidateNodeIds?, candidateRules?}
// delay:       {mode:"duration"|"until", durationHours?, untilExpr?}   // v1 implements duration only
// edge.condition: {expr, …}; edge.isDefault marks fallback; edge.kind=="error" marks action error branch
```

## PARTICIPANT RESOLUTION PIPELINE (fixed order)

```
union(rules) → active-employment filter → dedup(selfPolicy → repeatPolicy)
→ if empty: emptyPolicy fallback → delegation check (per final candidate, no recursion)
```
Auto-pass results still create done task rows (F11). Fallback candidates also pass the delegation check.

## MUST / MUST NOT

MUST:
- M1. Route ALL workflow-table access through the host WorkflowStore facade (or read-only SQL for reporting at most).
- M2. Keep publish-validation changes mirrored in BOTH validate.ts and validate.go (F17).
- M3. For programmatic starts: persist the record FIRST, then start (`WorkflowStart.StartForRecord` or the "start workflow" action). Never invert.
- M4. Supply an `idempotencyKey` on every task.execute (except claim/unclaim); reuse the same key when retrying the same logical click.
- M5. Treat `CONFLICT` as retryable (re-read, re-submit with same idempotency key); treat claim-CONFLICT as "someone else owns it".
- M6. When adding a node type: designer stencil/panel + types.ts + dual validation + engine NodeExecutor + docs.
- M7. Register/deregister scheduler jobs only through the existing prefix conventions and the single task-state-change choke point.

MUST NOT:
- N1. Never UPDATE/DELETE `wf_action_log` rows or published `app_workflow_version` rows.
- N2. Never write the record `flow_status` column directly (only RecordFlowStatusService; only the engine calls it).
- N3. Never write reserved `wf_variable` keys (see DATA MODEL).
- N4. Never mutate historical task rows for back/redo flows — always create new `from_backward` rows (F12).
- N5. Never change `prev_task_id` semantics to a hop-chain (it is fixed: the human task that triggered the advance).
- N6. Never let a subprocess instance transition flow_status (F15).
- N7. Never build polling loops inside the plugin — use scheduler jobs + `system.job_fire`.
- N8. Never trust request-body userId for identity in task-center ops — identity comes from the envelope `ActorID`.

## TYPICAL SEQUENCES

Submit → approve → finalize:
```
POST …/records/{r}/submit                        → {submitted, instanceId}
GET  /api/v1/workflow/tasks?tab=inbox            → find taskId
POST …/tasks/{app}/{task}/execute {action:claim, idempotencyKey:k1}
POST …/tasks/{app}/{task}/execute {action:forward, idempotencyKey:k2, opinion}
→ instance approved, flow_status=approved, log: start/claim/forward
```

Reject-back → resubmit:
```
execute {action:reject}   (button effect back_start) → siblings cancelled, new from_backward task at start-owner
initiator edits record → execute {action:forward} on the resubmit task
→ resubmitMode=direct: jumps straight back to the rejecting node (sub_state=resubmit)
```

Parallel:
```
split fires ALL out-edges → N pending branch tasks
finish branch A → instance still processing (join not satisfied, other branches pending)
finish last branch → join satisfied in the same change-set → advance → finalize
back within a branch: OK; back across branches: OP_INVALID workflow.back.cross_branch
```

Programmatic start (from any plugin):
```
host.Template().SaveReportRecord(Mode:"create", …)   // if the record does not exist yet
host.WorkflowStart().StartForRecord(appId, templateId, recordId, actorId)
→ full host orchestration (binding check / CAS guard / compensation) is reused
```

Timeout path:
```
task created → RegisterJob(task_timeout_<id>) [+ task_escalate_<id> if configured]
job fires → system.job_fire → plugin: still active? notify per notifyTargets : CancelJob
escalation fires → auto_pass|auto_reject|auto_transfer|mark per node config
task leaves active state → jobs cancelled at the single choke point
```

## ERROR TABLE (business codes inside `Call` responses)

```
OP_UNKNOWN            unregistered op
OP_INVALID            bad payload / illegal action for state / cross-branch back / unknown effect
SCHEMA_UNSUPPORTED    definition schemaVersion != 1
CONFLICT              optimistic-lock loss | claim race | version-number race | already_processing   (retryable)
TASK_NOT_FOUND / NOT_CANDIDATE / NOT_CLAIMER / TASK_NOT_ACTIVE / OPINION_REQUIRED / FIELDS_REQUIRED
PERMISSION_DENIED     caller plugin category ≠ workflow (host facades)
INTERNAL              normalized unexpected error (details in host log center, category=workflow)
```

## SELF-CHECK BEFORE YOU SHIP

1. Did any code path write wf_* tables or flow_status outside the sanctioned facades? → refactor.
2. Does every user-triggered execute carry a stable idempotencyKey (reused on retry)?
3. If you touched validation: are validate.ts and validate.go still line-for-line equivalent (15 rules + parallel_split exemption)?
4. If you created records then started workflows: is persistence strictly before start?
5. If you added timing behavior: is it a scheduler job with a prefixed key, registered post-commit, cancelled at task terminal states?
6. If you consumed events: is your handler idempotent (at-least-once delivery)?
7. Run the engine test suite: `plugins/kimpo-workflow: make build && GOWORK=off go test ./... -count=1` (5 packages must pass), and `frontend: pnpm test && pnpm exec playwright test` if you touched the designer.
```
