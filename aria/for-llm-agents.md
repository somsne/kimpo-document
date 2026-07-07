# ARIA (艾莉亚) — Kimpo 填报态业务数据层规范 · 大模型上下文版

> AUDIENCE: LLM / AI coding agent generating or reviewing Kimpo plugin code.
> USAGE: 将本文整体注入上下文（system prompt / RAG）。生成任何"读写填报表单业务数据"的插件代码前，先满足 SELF-CHECK。
> HUMAN VERSION: 同目录 `for-human-developers.md`。
> LANGUAGE: 代码注释与错误信息用中文；标识符按 SDK 原样。

## FACTS（事实，编号可引用）

- **F1** 填报态（form-runtime）下，业务字段数据的唯一权威 = 宿主进程内的 **Mirror（艾莉亚(Aria)，formmirror）**。保存落库、动作取值、插件读取，均以 Mirror 为准。
- **F2** 插件读写业务数据的**唯一通道** = SDK `plugin.Host.Mirror()`（gRPC，宿主网关校验 write-grant）。不存在合法的第二通道。
- **F3** editor（表格编辑器 kimpo-sheet）不是权威：它是渲染视图 + 公式计算内核。它对 Mirror 无否决权；其用户编辑与公式结果同样经宿主有序通道汇入 Mirror。
- **F4** 每个填报会话（sessionID）在宿主侧有独立的有序写队列 + 结算屏障：影响公式的写完成后，依赖它的后续业务读写被挡到公式结算（settle）才放行。跨会话互不影响。
- **F5** 寻址：`(sessionID, dataTableID, rowKey, fieldID)`。主表 rowKey 恒为 `"main:0"`；明细/扩展表 rowKey 为 `"row:N"`（稳定身份键，删行/重排不漂移，绝不能当物理行号用）。
- **F6** 值线格式恒为 JSON `{"raw": <业务标量>, "text": "<显示文本>"}`（TypedValue）。写入时两者都填；nil 值 text 给空串。
- **F7** `write_grant` 是宿主按**单次动作调用**签发的写授权令牌（call-scoped），随触发上下文 `query_context` 注入，只覆盖动作配置声明的目标表/字段。
- **F8** 公式派生字段（computed）对插件只读：宿主在写入口按"列级 computed 登记"拒写（`computed_readonly`）。computed 的值由 editor 重算后经结算通道回写 Mirror。
- **F9** 写成功但"渲染通知失败"= **权威已写入、会随保存落库**，仅界面未即时刷新。业务流程按成功处理。
- **F10** 跨模板/数据表查询走 `Host.Query()`：插件提交表达式模型或 KimpoSQL 文本，宿主编译执行返回行集；插件内存加工。插件禁止裸 SQL。

## INTERFACE（SDK `plugin.MirrorClient`，来自 kimpo-plugin-sdk/plugin/host.go）

获取：
```go
host := plugin.HostFrom(ctx)   // 可能为 nil，判空
mirror := host.Mirror()
sessionID, _ := qc["sheet_session_id"].(string)  // qc = 动作触发上下文 query_context
writeGrant, _ := qc["write_grant"].(string)
```

读（会话内免 grant）：
```go
ReadField(ctx, sessionID, dataTableID, rowKey, fieldID string, waitSettled bool) (found bool, typedValue []byte, err error)
ReadFields(ctx, sessionID string, fields []ReadFieldRef, waitSettled bool) ([]ReadFieldValue, error)
ReadTable(ctx, sessionID, dataTableID string) (rows []byte, err error)        // 全行快照,显示序;rows=JSON []{row_key,row_no,display_order,record_id,cells{fieldID:{raw,text}}}
ListDetailRows(ctx, sessionID, dataTableID string) (rowKeys []string, err error)
```
- `waitSettled=true` ⇒ 等公式结算后返回权威值。需要"计算后的正确值"时必须传 true。

写（必带 writeGrant）：
```go
WriteField(ctx, sessionID, dataTableID, rowKey, fieldID string, typedValue []byte, writeGrant string) (seq int64, err error)
WriteFields(ctx, sessionID string, fields []WriteFieldValue, writeGrant string) ([]WriteFieldResult, error)
// WriteFieldValue{DataTableID, RowKey, FieldID, TypedValue []byte}
// WriteFieldResult{DataTableID, RowKey, FieldID, AppliedRevision, Err}
// 单字段级失败经 results[i].Err 透出（MUST 逐条检查，R7）。语义按 AppliedRevision 区分:
//   Err!="" && AppliedRevision==0 ⇒ 权威未写(如 computed 拒写)——该字段值没进 Mirror;
//   Err!="" && AppliedRevision>0  ⇒ 权威已写、仅渲染通知失败——按成功处理(F9)。
// 单字段 WriteField 的"权威已写仅渲染失败"= *plugin.RenderNotifyError(errors.As 判定,F9)。
AppendDetailRow(ctx, sessionID, dataTableID, writeGrant string) (rowKey string, err error)
RemoveDetailRow(ctx, sessionID, dataTableID, rowKey, writeGrant string) error
ReorderDetailRows(ctx, sessionID, dataTableID string, orderedRowKeys []string, writeGrant string) error
```

编辑器类插件专用（普通动作插件 MUST NOT 调用）：
```go
SyncFields / ReportCalcState / BuildPayload / AllocateDetailRow / MarkRowDeleted
```

## GUARANTEES（宿主内建，插件零实现）

- **G1 写前校验 fail-closed**：computed 拒写 / view 会话拒写 / 失效会话拒写，均在宿主入口拦截，Mirror 不脏。
- **G2 结算屏障**：写触发公式 ⇒ 后续业务操作等结算。多步动作（写→读→再写）天然读到已结算值。
- **G3 渲染自动同步**：宿主写后自动通知 editor 更新单元格并推送浏览器。
- **G4 防回声**：插件写入值不被 editor 投影旧值覆盖（值锚定）；用户后续手改可正常覆盖插件值。

## ERROR DECISION TABLE（写路径错误 → 动作）

| err 含义（匹配子串） | 处置 |
|---|---|
| `computed` （公式派生字段拒写；批量时在 `results[i].Err` 且 AppliedRevision==0） | 放弃写该字段；改写其输入字段 |
| `not writable`（view 会话） | 放弃全部写，动作按只读场景结束 |
| `session not found`（会话失效/回收） | 终止动作，禁止重试 |
| 渲染通知失败（单字段 = `*plugin.RenderNotifyError`，errors.As 判定；批量 = `results[i].Err`!="" 且 AppliedRevision>0） | **按写成功处理**，继续流程，可记 warn（F9） |
| grant 校验失败 | 检查动作配置声明的目标表/字段与实际写目标是否一致 |

## RULES（硬约束）

- **R1 MUST** 一切业务数据读写经 `host.Mirror()`；**MUST NOT** import editorpb、调 editor RPC、经 WS 命令写业务值。
- **R2 MUST NOT** 对 `app_table_*` 物理表执行任何自拼 SQL；查询一律 `host.Query()`。
- **R3 MUST** `writeGrant` 从当次触发上下文取用并原样透传；**MUST NOT** 缓存、持久化或跨动作复用。
- **R4 MUST** 需要权威值的读传 `waitSettled=true`；**MUST NOT** 以浏览器/editor 展示值为数据源。
- **R5 MUST** typedValue 用 F6 线格式；数值 raw 放数值型，text 放渲染串。
- **R6 SHOULD** 批量写用 `WriteFields`（一批=一个屏障窗口=一帧渲染）；**SHOULD NOT** 循环单调 `WriteField` 写大量字段。
- **R7 MUST** 逐条检查 `WriteFields` 返回的 `results[i].Err`：`Err!=""` 且 `AppliedRevision==0` ⇒ 该字段权威未写（如 computed 拒写），须按决策表处置；`AppliedRevision>0` ⇒ 仅渲染通知失败，按成功处理（F9）。仍 SHOULD 只写动作配置声明的输入字段（grant 本就只覆盖它们）。
- **R8 MUST NOT** 因渲染通知失败回滚业务流程（F9）。
- **R9 SHOULD** 追加明细行前复用既有空行（`ReadTable` 后检测全空行），避免凭空增行。
- **R10 MUST**（仅编辑器类插件）实现来源过滤：宿主内部通道写入的 input 值不回投；开会话声明 computed 字段清单。

## CANONICAL SEQUENCES（典型调用序列）

S1 · 写主表单字段（最小闭环）：
```go
host := plugin.HostFrom(ctx); mirror := host.Mirror()
sessionID, _ := qc["sheet_session_id"].(string)
grant, _ := qc["write_grant"].(string)
tv, _ := json.Marshal(map[string]any{"raw": v, "text": fmt.Sprint(v)})
if _, err := mirror.WriteField(ctx, sessionID, tableID, "main:0", fieldID, tv, grant); err != nil {
    // 按 ERROR DECISION TABLE 分支
}
```

S2 · 读结算值 → 计算 → 批量写明细：
```go
_, base, _ := mirror.ReadField(ctx, sessionID, mainTbl, "main:0", srcField, true) // waitSettled=true
rowKeys, _ := mirror.ListDetailRows(ctx, sessionID, detailTbl)
var fields []plugin.WriteFieldValue
for _, rk := range rowKeys {
    fields = append(fields, plugin.WriteFieldValue{DataTableID: detailTbl, RowKey: rk, FieldID: dstField, TypedValue: derive(base)})
}
results, err := mirror.WriteFields(ctx, sessionID, fields, grant)
if err != nil {
    // 整批级错误（decode/grant/not-writable/session）按 ERROR DECISION TABLE 分支
}
for _, r := range results { // R7: 单字段级失败逐条检查
    if r.Err != "" && r.AppliedRevision == 0 {
        // 该字段权威未写（如 computed 拒写）——按决策表处置
    }
}
```

S3 · 明细行不足则追加再写：
```go
rowKeys, _ := mirror.ListDetailRows(ctx, sessionID, detailTbl)
for len(rowKeys) < need {
    rk, err := mirror.AppendDetailRow(ctx, sessionID, detailTbl, grant)
    if err != nil { break }
    rowKeys = append(rowKeys, rk)
}
// 然后 WriteFields 逐行写值
```

## SELF-CHECK（生成代码前逐项确认）

1. 所有业务数据交互是否只出现 `host.Mirror()` 与 `host.Query()`？（R1/R2）
2. 每个写调用是否带了来自触发上下文的 grant？（R3）
3. 需要计算后值的读是否 `waitSettled=true`？（R4）
4. typedValue 是否 `{"raw","text"}` 双字段？（R5）
5. 是否处理了 computed 拒写 / not-writable / session-not-found / notify-失败四种语义？批量写是否逐条检查了 `results[i].Err`？（决策表/R7）
6. 大批量写是否用了 `WriteFields` 分批？（R6）
7. 主表 rowKey 是否写死 `"main:0"`、明细是否用 `ListDetailRows/AppendDetailRow` 返回的 rowKey？（F5）
