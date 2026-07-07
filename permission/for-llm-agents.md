# 权限框架（代号 卡侬/Canon）— Kimpo 业务对象赋权规范 · 大模型上下文版

> AUDIENCE: LLM / AI coding agent generating or reviewing Kimpo plugin code, or integrating objects into the permission framework.
> USAGE: 将本文整体注入上下文（system prompt / RAG）。生成任何"涉及权限判断/权限界面/受管数据访问"的代码前，先满足 SELF-CHECK。
> HUMAN VERSION: 同目录 `for-human-developers.md`。
> LANGUAGE: 代码注释与错误信息用中文；标识符按现有 API 原样。

## FACTS（事实，编号可引用）

- **F1** Kimpo 平台权限只有一套 = 权限框架（代号 卡侬/Canon）。插件 **MUST NOT** 自建权限表、角色表或权限判断逻辑。
- **F2** 授权模型：`grant = grantee(主体) × object(客体) × payload(载荷)`。存应用库，随应用备份/分发（分发包剥离 user/org_unit/role 主体的授权，仅保留 app_role 主体授权——用户与部门是实例私有数据）。
- **F3** 四类主体 `grantee_type`：`user`（用户本人）/ `org_unit`（部门，`include_children=true` 时含整棵子树成员）/ `role`（**实例级**角色树上的角色，跨应用复用）/ `app_role`（**应用内**角色，定义随应用走、用户分配关系存实例侧）。四类在裁决中完全平等。
- **F4** 三层客体 `object_type`（逐层收敛，上层不过下层不评估）：`app`（可访问，object_id=appId）→ `menu`（可查看，object_id=应用树节点 node_key）→ `template`（操作矩阵+数据范围，object_id=templateId）。
- **F5** 客体载荷契约（**易错点**）：`app` 客体只用 payload 的 **create 位**承载"允许访问"；`menu` 客体只用 **view 位**承载"允许查看"；`template` 客体用完整 7 操作位 + 三档位。给 app 客体勾 view、给 menu 勾 create 均无效。
- **F6** 模板操作矩阵 7 位（bool）：`create`(新建单据) `view`(查看历史) `modify`(改历史) `delete` `import` `export` `print`。
- **F7** 数据范围（行级）挂在 view/modify/delete 各一份：`all` > `dept_tree`(我的部门子树) > `dept`(我的部门) > `self`(归属人=我) > `""`(未设)。宽严序如左。
- **F8** 行归属按主表可变系统列 `owner_id`(归属人,初始=创建人,可交接转移) / `owner_org_id`(归属部门,初始=创建人当时主部门,快照语义:调岗不迁移)。**不按 created_by** ——交接后受让人可见、原创建人不可见是设计意图。明细/扩展表行跟随主记录可见性，不单独授权。
- **F9** 合并算法：展开用户身份集合（本人 ∪ 实例角色 ∪ 本应用应用角色 ∪ 所属部门 ∪ 祖先部门中 include_children 命中的）→ 取全部命中授权 → 操作位按 OR、数据范围取最宽档。**无 deny 语义**，收权=删授权/缩载荷。
- **F10** 白名单默认：客体无任何命中授权 ⇒ 不可见/不可操作。特权短路（不查授权表直接全权）：`super_admin`、`app_designer`（实例角色码）、该应用 `app_admin`。
- **F11** 后端裁决是唯一权威；前端隐藏/置灰仅体验优化。越权访问单条记录返回 **404**（不泄露存在性），无操作权返回 **403** + `err.canon.no_permission`。
- **F12** 执法点全部在宿主：记录 CRUD 端点操作闸门、列表查询 SQL 层行过滤、应用树端点按可见菜单集过滤、填报会话建立预检（formMode=create 需 create 位；edit/view 需 view 位）、会话内导入/导出端点各查 import/export 位。**插件进程内不存在任何执法点**。
- **F13** 缓存：有效权限按 (实例,应用,用户) 进程内缓存 TTL 60s；授权项增删改立即失效；角色绑定/部门成员变更靠 TTL（最迟 60s 生效）。
- **F14** `write_grant`（动作级写授权，见 aria 文档 R2）与卡侬用户级权限是**两层独立闸门**，叠加生效：前者管"这个动作能写哪张表"，后者管"这个用户能碰哪些数据"。互不替代。

## PLUGIN CONTRACT（插件视角：你已被管控，无需实现）

- **P1** 动作插件：动作能被触发 ⇒ 触发用户已过操作闸门。动作内经 `host.Mirror()/Query()/Record()` 的读写，宿主已在数据面完成行过滤与写校验。插件代码里 **MUST NOT** 出现任何权限判断。
- **P2** Mirror 会话数据 = 该用户可见宇宙内的数据（会话建立时已裁决），直接使用。
- **P3** `host.Record()` 写记录时越权行为表现为"记录不存在"——正常处理 not-found 错误即覆盖越权场景，不要特判。
- **P4** 前端 Surface 可选消费（体验优化专用）：
  ```
  GET /api/v1/apps/{appId}/effective-permission   （同源 cookie 会话）
  → { "is_privileged": bool,        // true ⇒ 全权，MUST 短路，不逐项比对
      "app_access": bool,
      "visible_menu_keys": ["<node_key>", ...],
      "templates": { "<templateId>": {
          "create":bool,"view":bool,"modify":bool,"delete":bool,
          "import":bool,"export":bool,"print":bool,
          "view_scope":"all|dept_tree|dept|self|",
          "modify_scope":"...","delete_scope":"..." } } }
  ```
  未加载完成时默认放行（避免摘要接口异常锁死界面）；**MUST NOT** 据此做安全判断或长期缓存（>页面会话）。

## ADMIN API（授权管理端点，鉴权=super_admin 或该应用 app_admin）

```
GET    /api/v1/apps/{appId}/permission-grants?object_type=&object_id=   # 列授权项(含 grantee_display_name)
POST   /api/v1/apps/{appId}/permission-grants                           # 批量新增: 多主体×单客体×同载荷
       body: { object_type, object_id, grantee_type, grantee_ids: [..],
               include_children: bool, payload: {7位+3档位} }
PATCH  /api/v1/apps/{appId}/permission-grants/{id}                      # 改载荷/include_children
       body: { include_children?, payload?, version_no }               # version_no 乐观锁必带
DELETE /api/v1/apps/{appId}/permission-grants/{id}
```

约束：`app_admin` 内置应用角色不可作为 grantee（特权身份配授权=语义冲突，后端拒绝）；分类节点（node_type=category 的角色）不可作为 grantee，只有 role 节点可以。

前端复用组件（**宿主前端开发**勿重造；插件 Surface 不可 import 宿主组件）：`@/components/common/ObjectGrantsPanel.vue`——传 `appId + objectType + objectId + objectLabel` 即得完整"授权项列表+添加(三页签:部门/角色/用户)+编辑+删除"面板；应用设置权限管理页与模板版本管理页均用它。

## INTEGRATION（把新业务对象纳入卡侬，按优先级）

- **I1 首选零开发**：对象若呈现在应用导航树上（page 类插件的页面节点等）⇒ 它已是 `menu` 客体，宿主树过滤天然覆盖，管理员可直接对它授权。
- **I2 绑定模板语义**：功能围绕某模板展开的（取数/审批/视图类），复用该模板的操作矩阵——用户对模板的权 = 对该功能的权，不新造客体。
- **I3 新客体类型（宿主侧开发，与平台团队协作）**：
  - object_type 为开放字符串枚举（现 app/menu/template），payload 加 bool 操作位不动模型；
  - 接入模式=**小接口+注入**：消费模块自定义最小接口（先例：templates.PermissionGate / apptree.MenuGate / sheetproxy.PermissionGate），宿主装配层（bootstrap）注入卡侬适配器；模块 MUST NOT 直接 import 卡侬包（保持低耦合，接口为 nil 时=未接线放行）；
  - 裁决入口：`ComputeForUser(ctx, instanceID, appID, userID) → EffectivePermission`（内建特权短路+缓存，调用方无需持有会话对象）；行过滤入口：`ResolveRowPredicate(...) → {scope, userID, orgIDs[]}`，翻译为 SQL 谓词：self ⇒ `owner_id=?`；dept/dept_tree ⇒ `owner_org_id IN (?...)`（orgIDs 已由卡侬展开，空集=恒假）；all ⇒ 不加。

## RULES

- **MUST NOT** 自建权限表/角色模型/权限判断（F1）；插件进程内出现"user 有没有权"的 if 即违规。
- **MUST NOT** 绕过行过滤取数（自拼 SQL 碰 `app_table_*`、直连 editor——同时违反 aria R1）。
- **MUST NOT** 把 effective-permission 用于安全判断或跨页面缓存（P4/F13）。
- **MUST NOT** 缓存/复用/伪造 `write_grant`（F14 + aria R2）。
- **MUST** 前端消费 is_privileged=true 时短路全权（P4）。
- **MUST** PATCH 授权时携带最新 version_no（乐观锁）。
- **MUST** 对 app/menu 客体只操作其有效载荷位（F5）：app→create、menu→view。
- **MUST** 处理 404 语义：越权记录=不存在，不重试、不特判、不提示"无权限"（F11——那会泄露存在性）。
- 新增客体/操作位 **MUST** 走 I3 注入模式，禁止在业务模块内直连卡侬存储。

## TYPICAL SEQUENCES

S1 动作插件（什么都不做即正确）：
```go
// 动作被触发 ⇒ 用户已有权。直接干业务：
host := plugin.HostFrom(ctx)
rows, _ := host.Query().Execute(...)          // 宿主套用查询者权限
host.Mirror().WriteFields(..., writeGrant)    // 动作级 grant 原样透传
// ← 全程无任何权限代码。若出现 CheckPermission/isAllowed 之类 ⇒ 设计错误
```

S2 前端 Surface 按权限定制界面：
```ts
const { data } = await fetch(`/api/v1/apps/${appId}/effective-permission`).then(r => r.json());
const perm = data;
const canDelete = perm.is_privileged || (perm.templates[tplId]?.delete ?? false);
// 仅用于隐藏/置灰；后端 403/404 仍是最终裁决
```

S3 管理面板嵌入（给某对象做授权配置 UI；**仅宿主前端开发适用**——插件 Surface 是 UMD 独立构建，
不能 import 宿主组件，Surface 需要授权配置时应跳转宿主既有权限页而非重造）：
```vue
<ObjectGrantsPanel :app-id="appId" object-type="template" :object-id="tplId" :object-label="tplName" />
```

## SELF-CHECK（生成代码后逐项过）

```
□ 代码里没有任何自建的权限/角色存储或判断逻辑                      (F1/RULES)
□ 没有为"越权"写特殊分支——404 当 not-found 处理                    (F11/P3)
□ effective-permission 只用于 UI 显隐，先判 is_privileged 短路      (P4)
□ 若操作 app/menu 客体授权：只碰 create/view 对应位                 (F5)
□ PATCH grant 带了 version_no                                      (RULES)
□ 授权配置 UI 复用 ObjectGrantsPanel 而非重造                       (ADMIN API)
□ 新对象纳管：优先 I1/I2；I3 时用小接口+注入，未 import 卡侬包       (I3)
□ write_grant 原样透传，未与用户级权限混为一谈                       (F14)
```
