# 查询体系（代号 艾丽丝/Alice）原理与对接指南 —— 面向插件开发者

> 读者：要开发查询/取数/写数类 Kimpo 插件、或要在模板动作里消费查询能力的**人类工程师**。
> 读完你会明白：Alice 是什么、一次查询从设计到落值的完整旅程、空值在这条链上的确定行为，以及怎么用 SDK 十分钟接上。
> 配套：给 AI 编程助手的结构化版本在同目录 `for-llm-agents.md`；空值语义的业务讲解见 `../fields/empty-values-for-human-developers.md`。

---

## 1. Alice 是什么：一个翻译官的故事

想象你的插件要"从采购订单里把没入库的行查出来，填进当前入库单"。系统里有三个角色：

- **你的插件**：只会说"业务话"——它把查询意图写成一份**表达式模型**（JSON：查哪些表、什么条件、填到哪），交出去，拿回一批行，加工后经表单数据权威层 Mirror（代号 艾莉亚/Aria）写回。**它从头到尾不写、不见、不碰一行 SQL。**
- **翻译官（艾丽丝/Alice，宿主查询体系）**：把表达式模型**编译**成目标数据库的 SQL、只读执行、把行集还给你。方言、软删过滤、权限边界、本报表上下文、空值语义——全是它的事。
- **账本（艾莉亚/Aria Mirror）**：查询结果要落到表单上时，唯一的写入通道。

一句话记住：**插件说业务话，Alice 译数据库话，Aria 记账。** 这个分工是红线：插件直连数据库或自拼 SQL 属违规（见开发规范 R1）。

## 2. 一次查询的完整旅程

```
设计态                          填报态运行
┌──────────────┐   保存    ┌─────────────────────────────────────────┐
│ 表达式编辑器    │ ───────▶ │ 动作触发 → 插件拿到模型                      │
│ (四段采集)     │  模型JSON │   │ SubmitExpression(模型, 上下文)  SDK 反向通道│
│ 来源/关联/     │          │   ▼                                      │
│ 筛选/输出      │          │ Alice：校验→绑定→编译→执行                   │
└──────────────┘          │   │  ①本报表上下文物化(Mirror快照→临时表)      │
      │ compile-preview    │   │  ②空参数条件裁剪(F7，被裁清单可见)         │
      ▼ 同源dry-run        │   ▼                                      │
   真实运行SQL预览          │ 行集(o_*/m_* 列) 回插件                     │
                          │   ▼                                      │
                          │ 插件内存加工：填充策略/左驱行匹配/清空/排序      │
                          │   ▼                                      │
                          │ 经 Aria Mirror 写回表单（grant 白名单内）     │
                          └─────────────────────────────────────────┘
```

关键性质：

- **preview 与运行同源**：`compile-preview` 端点跑的就是运行态编译器，预览出的 SQL 即真实执行的 SQL——设计态排障首选。
- **两条通道并存**：文本 KimpoSQL（单表、能力受限）与表达式模型（多表 JOIN/分组/开窗，全能力）。新功能一律走表达式模型通道。
- **本报表上下文（is_context）**：条件里引用"本报表某字段"时，Alice 把当前填报会话的 Mirror 快照（含未落库的脏值）物化成临时表参与 JOIN，天然把查询"限定在当前单据语境"。

## 3. 表达式模型：你要产出的东西

顶层 `ExpressionModel`（细节以 `model.go`/`types.ts` 为准）：

| 段 | 内容 | 要点 |
|---|---|---|
| `source` | tables[]（别名/template_id/data_table_id/is_context/role）+ joins[] | 一切来源都是模板的数据表；`is_context:true` = 本报表上下文表；主明细 record_id 关联由编辑器自动注入 |
| `where` / `where_expr` | 结构化条件树（and/or/negate + 谓词）或整段 raw 表达式，**二选一互斥** | 操作符：`eq/neq/lt/lte/gt/gte/like/in/not_in/is_null/is_not_null` |
| `target` | 目标表 + outputs[]（字段/表达式/填充策略）+ match_keys[] + clear + post_sort | 匹配键驱动"按行列匹配"（插件内存左驱 join） |
| `order_by/group_by/distinct/limit` | 查询侧属性 | **全部下推 SQL**，不要在插件内存里自己排序去重 |

值的原子是 **ValueRef**（8 类判别联合）：`source_field`（来源表列）、`system_var`（系统变量，运行期宿主解析成值）、`const`、`func`（白名单函数）、`raw`（逃生舱）、`window`、`case`、`form_field`（后端支持但当前编辑器不产出——本报表字段一律走 is_context 的 `source_field`，别自己走 form_field 路）。

函数白名单（canonical）：`coalesce`（别名 IFNULL/ISNULL，「空值默认」）、`concat`、`contains_text`、`starts_with`、`ends_with`、`date_add_day`、`date_diff`（别名 DATEDIFF/日期间隔，单位 d/m）、`round`、`now_local`、`now_utc`、`today`。未注册函数直接编译报错。

## 4. 空值在 Alice 里的确定行为（对接时最常踩的区）

完整契约见 `docs/architecture/field-null-semantics-contract-2026-07-07.md`（宿主仓）；这里给插件开发者视角的速查。

**大原则**：Alice 属于**表达式域=数据库世界**——空诚实传播（`空+1=空`），聚合跳过空（`AVG` 不进分母，`SUM` 全空得 0），比较不命中空。要兜底用显式 `coalesce`。唯一内置例外：`concat` 跳空（一参为空不会让整串消失）。

**F7 固定规则（空参数条件自动失效）**：条件里引用的"参数"（系统变量、本报表**主表**字段快照）为空时——

| 操作符族 | 行为 |
|---|---|
| `> >= < <=`、date_diff 区间形态 | 该条件**视同没写**（自动从 WHERE 裁掉） |
| `like/包含/开头/结尾`、`≠`、`in/not_in` 空列表 | 同上 |
| `=` | 条件保留 → 恒不命中（"单号没填当然配不上"） |

- 覆盖机制**不是开关**，是老写法：`或者 参数为空` 强制"不限"；`并且 参数不为空` 强制"没填不取"——守卫/门闸类条件（可判定真假）永远不会被裁。
- 本报表**扩展表**字段引用是逐行数据（匹配键），不是参数，永不参与裁剪；且匹配键全是等值，等值族本就不裁——**双保险，匹配键不可能被 F7 误伤**。
- 被裁条件在 `compile-preview` 结果与执行日志（debug）里有清单——排查"为什么查出这么多行"先看它。

**兼容别名**：表达式里 `x=''`/`x<>''` 被编译期理解为"为空/不为空"（新模型中空字符串不存在，无歧义）。

**匹配键红线**：按行列匹配的内存 join 中，**匹配键任一分量为空的行不参与匹配**（源行按未命中处理、目标行不入索引）。"没填也算一档"的业务请给字段配默认值，不要指望空键互配。

## 5. 动手：SDK 对接（取数类动作范式）

```go
host := plugin.HostFrom(ctx)

// ① 从触发上下文原样取用（宿主注入，勿缓存勿伪造）
sessionID := queryContext["sheet_session_id"]
grant     := queryContext["write_grant"]

// ② 提交表达式模型（设计态编辑器落库的 behavior JSON 原样透传）
rows, cols, err := host.Query().SubmitExpression(ctx, modelJSON, triggerCtx, nil)

// ③ 内存加工：按 o_<目标字段ID> 读输出列、m_<seq> 读匹配键列
//    填充策略/分隔符/清空/写回后排序 —— 插件职责
//    DISTINCT/LIMIT/排序 —— 已在 SQL 里做完，别重复做

// ④ 经 Aria 写回（批量用 WriteFields 分批；错误语义遵守 aria 契约）
host.Mirror().WriteFields(ctx, sessionID, tableID, items)
```

设计态：插件的动作配置 surface 直接挂共享包 `@kimpo/expression-editor`（薄壳），四段采集零 UI 复制；校验按钮调 `compile-preview`。

## 6. 红线与坑（每条都有人踩过）

1. **永不碰裸 SQL / 物理表**：不 import 数据库驱动、不拼 `app_table_*`。查询只走 `host.Query()`。
2. **函数白名单双源**：新增函数要同时登记 `funcs.DefaultRegistry` 与 bind 包 `functionSignatures`，漏一边=设计态过、绑定期炸。
3. **DISTINCT/LIMIT/查询排序是 SQL 侧的**；插件内存只做 fill/separator/clear/post_sort。
4. **主明细 record_id 自动关联是编辑器注入的**，后端只消费不校验——别手工构造模型时传错别名。
5. **本报表字段=is_context 的 source_field**，别走 form_field（无 UI 支撑）。
6. **写权限白名单**：普通写回按字段级 grant；仅"按行列匹配"场景签发表级 `*` 通配。grant 一次性，禁缓存复用。
7. **临时表 text 列 VARCHAR(512)**：超长文本作匹配键会被截断——别用长文本字段当键。
8. **空值语义按 §4 办**：不要在插件里自作主张把空转 0/""——那是把哨兵从插件侧带回来，审查会打回。

## 7. FAQ

**Q：我想"参数没填就查全部"，要写什么？**
什么都不用写。区间/模糊/排除类条件天然如此（F7）；等值条件想要这个行为，加一句 `或者 该参数为空`（老写法 `或者 参数=''` 原样可用）。

**Q：为什么我的条件在预览 SQL 里消失了？**
它引用的参数当时为空，被 F7 裁掉了（preview 有"未生效条件"清单）。想强制生效：给参数配默认值/必填，或写守卫 `并且 参数不为空`（会变成"没填→查不到"而不是"没填→不限"）。

**Q：查询结果里某数值列是 null，我该写回 0 吗？**
写回 null（`TypedValue{raw:nil}`=真实清空）。"没查到"和"查到 0"是两个业务事实，账本要如实。需要按 0 参与计算的是**表达式里**的事（`coalesce(x,0)`），不是写回侧的事。

**Q：聚合 SUM 全空为什么得 0 而 AVG 得空？**
契约表三：SUM 的"什么都没有加"自然是 0；AVG 的"没有样本"没有均值，显示空白。两者都跳过空值（空不进分母）。
