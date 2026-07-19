# Kimpo 插件数据读写规则（Agent 版）

> 本文件是编写、审查和修改 Kimpo 插件时的强制规则。Workflow 不在本规范范围内。

## 1. 先判定数据所有权

Agent 在新增任何持久化数据前，必须先归类：

| 数据 | 所有者 | 正确入口 |
| --- | --- | --- |
| 应用、数据表、字段、记录、明细行、附件引用 | 宿主 | Record / Mirror / Asset Host API |
| 一条记录或明细行上的少量固定标量插件参数 | 插件语义、宿主存储 | Managed Plugin Column |
| 插件配置、设计文档、会话缓存、可重建缓存 | 插件 | `permissions.storage` 对应的插件本地存储 |
| 多行关系、任务、重试、审计、历史、消息 | 暂无通用插件存储规范 | 停止实现，先提交独立存储 ADR |
| Secret、Token、密码、私钥 | 安全域 | 禁止放入内联列或普通插件存储 |

`permissions.storage` 不等于业务数据权限。Editor 可以保存自身设计文档和缓存，但不得建立第二套业务表、字段或记录权威。

## 2. Managed Column 强制契约

插件只允许在 Manifest 声明逻辑列：

```json
{
  "data_storage": {
    "managed_columns": [{
      "column_id": "risk-score-v1",
      "key": "risk_score",
      "scope": "record",
      "type": "decimal",
      "precision": 8,
      "scale": 2,
      "nullable": true,
      "visibility": "host_readable",
      "queryable": true,
      "sortable": true,
      "index": "btree",
      "schema_version": 1
    }]
  }
}
```

MUST：

- 使用稳定 `column_id`；业务名称变化只改 `key`，不得重建列身份。
- `scope` 只能是 `record` 或 `detail_row`。
- 值只能是 boolean、int、bigint、decimal、date、datetime、varchar、stable_id 标量。
- 通过 Host SDK 按 `column_id` 或 `key` 二选一访问。
- 所有写入携带宿主签发的一次性 grant、期望版本和幂等键。
- 明细数据使用宿主稳定 row key / record ID，不使用显示行号或 Sheet 坐标。

MUST NOT：

- 不得连接应用数据库、执行 SQL/DDL、读取 `information_schema`。
- 不得在 Manifest、配置、请求或日志中保存/传递 `px_*` 物理列名。
- 不得用内联列保存 JSON 正文、数组、Blob、附件、日志、历史或无限增长数据。
- 不得把 `plugin_instance_id` 当作数据所有者；长期所有权是 `plugin_id + column_id`。
- 不得向普通 Editor 字段目录、默认导出、默认打印或 REST 同步暴露 `plugin_private`。

## 3. 可见性

- `plugin_private`：只有声明插件和宿主治理链可读写。
- `host_readable`：宿主可按明确业务用途投影；泛型动作和 P1 插件扩展只能使用这一类。
- `public_field`：必须提升为宿主公共字段。当前正常记录 writer 未开放，绑定入口会失败关闭；不得假设可用。

## 4. P1 通用动作投影

Form、UI/Print 等通用动作不能直接读取其他插件的列。动作参数声明宿主投影：

```json
{
  "managed_projection": [{
    "owner_plugin_id": "com.example.risk",
    "logical_key": "risk_score",
    "alias": "riskScore"
  }]
}
```

规则：

- `column_id` / `logical_key` 必须二选一；必须显式给出 `owner_plugin_id`。
- `alias` 必须匹配 `^[A-Za-z][A-Za-z0-9_]{0,63}$`，同一动作内唯一。
- 宿主只解析当前记录或当前明细稳定行上的 active `host_readable` binding。
- 编辑中的值由宿主以“数据库基线 + Mirror overlay”合并；插件只收到 `query_context.managed_projection = {alias: value}`。
- 插件使用 `plugin.ManagedProjectionFromActionContext(req.Context)` 读取防御性副本。
- 任一 owner、selector、scope、状态或目标不匹配时必须整次失败，不允许静默降级到物理名或猜测值。

Form：`ui.fill_form.prefill[].source_managed_alias` 可以引用投影 alias；目标仍必须是宿主公共字段。弹窗关闭后的 writeback 不复用已撤销的读取 grant。

Print：`ui.print` 将 alias 值传入 KimpoSheet；B 档打印插件从 `context.record.managed.<alias>` 读取。A 档静态 spec 不得假装具有动态投影。

REST：目标契约是宿主先形成逐行 alias map，REST 插件只执行 `source_alias -> external_field` 映射；行数不一致、alias 缺失、重复目标或物理列名必须失败关闭。当前仅映射器与测试已实现，宿主 Data 批量调度仍未接线；Agent 不得把它描述为端到端可用。

## 5. 记录动作与事务

- create/update 可在一次 Host Record 请求中同时提交公共字段与 `managed_values`。
- `managed_values` 必须声明列所有者；当前泛型动作仅允许 active `host_readable`。
- 公共字段与托管列必须同事务成功或回滚。
- update 必须同时校验业务 `version_no` 和 `extension_revision`。
- delete/restore 只改变同一宿主记录的生命周期，不搬迁托管值。
- detail sort 只改变显示顺序，不按行号复制或移动托管值。
- extract 默认不复制私有列；只有明确、授权、同语义的宿主策略才可复制。

## 6. 插件生命周期

- install：只登记声明，不对未知业务表执行 DDL。
- bind：管理员显式选择 app/table；宿主预检容量、类型和 scope 后建列并回读。
- activate：所有 binding 必须 ready；否则阻止启动。
- disable：停止新写，保留列与数据。
- upgrade：宿主比较 `column_id` 和 `schema_version`；不兼容变化无迁移 worker 时必须拒绝升级。
- rollback：先做兼容检查，再停止实例或替换文件。
- uninstall：默认 retire，保留映射和数据；删除插件记录失败必须补偿 retire。
- purge：当前未开放完整企业链路。不得自行 DROP；必须另有审批、引用扫描、备份、审计和 worker。
- reinstall：相同 `plugin_id + column_id` 接管原数据，不创建新所有权。

## 7. 提交前门禁

至少执行：

```bash
cd services/api
go test ./internal/actionquery/actiongateway ./internal/bootstrap ./internal/plugins/hostservices ./internal/modules/managedcolumns ./internal/modules/templates ./internal/formmirror -count=1

cd plugins/<go-plugin>
GOWORK=off go test ./...
```

同时检查：Manifest 无物理名；普通字段目录/导出/打印/REST 无 `plugin_private`；无 `SELECT *` 扩散；无数据库直连；失败路径保持旧值和旧 binding 可用。
