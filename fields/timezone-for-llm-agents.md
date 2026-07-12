# 日期时间与时区语义（auto_timezone）— Kimpo 字段规范 · 大模型上下文版

> AUDIENCE: LLM / AI coding agent generating or reviewing Kimpo plugin code, template automation, or SQL against Kimpo app databases.
> USAGE: 将本文整体注入上下文（system prompt / RAG）。生成任何"读写 date/datetime 字段、展示时间、写含日期条件的查询"的代码前，先满足 SELF-CHECK。
> HUMAN VERSION: 同目录 `timezone-for-human-developers.md`。
> LANGUAGE: 代码注释与错误信息用中文；标识符按 SDK 原样。

## FACTS（事实，编号可引用）

- **F1** 每个 date/datetime 字段有布尔属性 **auto_timezone**（数据表管理器"自动匹配时区"，仅 date/datetime 可勾，默认 false）。它声明的是**该列的存储语义**，不是显示偏好。
- **F2** `auto_timezone=false`（墙钟/naive，默认）：字面即语义。DB 列、业务层、屏幕三层**字面全等**；任何用户、任何时区看到同一串数字；全链零换算。
- **F3** `auto_timezone=true`（绝对时刻）：DB 列存 **UTC 墙钟字面值**；DB 之外的一切层（API 响应、Mirror(Aria)、编辑器、公式、前端、插件可见值）一律是**当前会话用户的本地墙钟**。示例：上海用户（UTC+8）录入 `21:00` → DB `13:00:00` → 上海用户读回 `21:00`，纽约用户（7 月 UTC−4）读回 `09:00`。
- **F4** UTC↔本地换算**只发生在宿主的数据库读写边界**（唯一换算点），按会话用户的时区偏好（IANA 名，如 `Asia/Shanghai`）执行。插件、编辑器、前端**均无换算责任，也无换算权利**。
- **F5** 插件经 Mirror 读到的日期值 = 会话用户本地墙钟（与用户格子所见一致）；写入 Mirror 也必须给本地墙钟。插件视角下 auto_timezone 字段与 naive 字段**处理方式完全相同**。
- **F6** 业务日期值的规范文本形态 = 空格分隔串：datetime → `yyyy-MM-dd HH:mm:ss`，date → `yyyy-MM-dd`。RFC3339/ISO 形态（含 `T`/`Z`/偏移）是内部泄漏，不得产出；若读到，按"取墙钟面、忽略 Z 与偏移"解析。
- **F7** 系统审计列（created_at / updated_at 等）恒为绝对时刻：DB 存 UTC，展示层按用户时区偏好换算。与 auto_timezone 机制平行，无字段开关。
- **F8** DB 一律使用**无时区列类型**（MySQL DATETIME/DATE 等），禁用连接时区换算型（TIMESTAMP/timestamptz/datetimeoffset）。字面比较/排序/分组三家数据库行为一致。
- **F9** 匿名会话 / 用户未设时区偏好 → 换算降级为 no-op（naive 直通），不报错。因此"读到的值恰好等于 DB 字面值"在匿名场景是预期行为。
- **F10** 翻转 auto_timezone 开关**不迁移存量数据**：既有字面值的解释语义整体改变。生成建模建议时：字段创建时定死，勿建议对有数据的字段翻转。
- **F11** 纯日期档位（年/年月/年月日）勾 auto_timezone 会产生跨日漂移，语义上不应勾；平台 UI 按 date/datetime 大类放开勾选，档位合理性由设计者负责——生成模板配置时只对含时刻分量的字段建议开启。
- **F12 已知遗留**（引用前先核实是否已修复）：经 `Host.Query()` 取数返回的 auto_timezone 字段**仍是 UTC 字面值**（查询出口换算未实施）。将取数结果展示给用户或回写墙钟字段前，须意识到此差异。

## VALUE-FORMAT（值形态规范）

```
写入（Mirror WriteField/WriteFields 的 TypedValue.raw / text）:
  datetime: "2026-07-11 21:00:00"   ← 用户此刻应看到的本地墙钟，空格分隔全精度
  date:     "2026-07-11"
禁止产出: "2026-07-11T21:00:00Z" / "2026-07-11T21:00:00+08:00" / Unix 时间戳当日期串
读取时兼容: 空格串（正常）；若遇 RFC3339 → 取墙钟面忽略 Z/偏移；数值型 Excel 日期序列号可能出现于编辑器内部值，插件不应依赖
```

## DECISION TABLE（时区决策表）

| 场景 | 决策 |
|---|---|
| 插件读到日期值，要不要转时区？ | **不要**（F4/F5）。它已是会话用户本地墙钟，直接用。 |
| 插件写日期值，要不要先转 UTC？ | **不要**（F4/F5）。给本地墙钟；转了就是双转事故（用户选 21:00 显示成 04:00 的历史故障根因）。 |
| 需要"当前时间"填给用户 | 用会话用户语境的墙钟；MUST NOT 拿服务器本地时钟冒充用户墙钟。 |
| 手写 SQL 给 UTC 语义列（auto 字段/系统列）造数 | 用 `UTC_TIMESTAMP()`；MUST NOT 用 `NOW()`（服务器时区墙钟）。 |
| SQL 里比较 auto 字段与 naive 字段 | **禁止**直接互比（UTC vs 用户墙钟，无业务意义）。 |
| 对 auto 字段 `GROUP BY DATE(col)` | 允许但注意：分的是 **UTC 日**，与用户当地日在时区偏移小时段内相差一天；需按用户日分组时由调用方换算好边界值传入。 |
| 取数结果里的 auto 字段（F12 遗留期） | 视为 UTC 字面；勿直接当本地墙钟展示/回写。 |
| 查库值与页面显示差 = 用户时区偏移 | 预期行为（F3/F7），不是 bug；勿生成"修复"代码。 |
| naive 字段查库值 ≠ 页面显示 | 真 bug，应上报，而非在插件里打补丁换算。 |

## MUST

- M1 处理任何业务日期值时，一律按"会话用户本地墙钟"语义编码（F3/F5），与 naive 字段同构处理。
- M2 写入用空格分隔规范串（F6）；date 档只给日期部分。
- M3 解析外来日期串时容忍 RFC3339 形态但**只取墙钟面**（忽略 Z/偏移）。
- M4 涉及"按日/按周"相对条件且用户时区敏感时，由调用方算好**字面边界值**注入查询条件，不依赖 SQL 端 `NOW()`/`CURDATE()`。
- M5 给用户展示系统审计时间（created_at 等）时按用户时区偏好格式化（F7）。

## MUST NOT

- N1 MUST NOT 在插件/前端/模板自动化中执行任何 UTC↔本地换算（唯一例外：消费 F12 遗留期的取数结果时的显式适配，须注释标明"临时适配 F12 遗留，出口换算实施后删除"）。
- N2 MUST NOT 产出含 `T`/`Z`/时区偏移的日期串写入业务字段。
- N3 MUST NOT 用服务器本地时间冒充用户墙钟；MUST NOT 用 `NOW()` 给 UTC 语义列造数。
- N4 MUST NOT 在 SQL 中让 auto_timezone 字段与 naive 字段直接比较。
- N5 MUST NOT 建议对已有数据的字段翻转 auto_timezone 开关（F10）；MUST NOT 对纯日期档位建议开启（F11）。
- N6 MUST NOT 把"查库字面值 ≠ 页面显示值"当 bug 修（F3/F7 预期）；也 MUST NOT 反向"修复"——绝对时刻字段查库若与页面**相等**才是写入换算失效的征兆。

## TYPICAL SEQUENCE（动作插件写一个 auto_timezone 字段）

```go
// 目标：把"会议开始"设为用户视角的 2026-07-11 21:00（上海用户）
// 正确：直接给本地墙钟，宿主在 DB 边界折算为 13:00 UTC 落库
tv, _ := json.Marshal(map[string]any{"raw": "2026-07-11 21:00:00", "text": "2026-07-11 21:00:00"})
_, err := mirror.WriteField(ctx, sessionID, tableID, "main:0", fieldID, tv, writeGrant)
// 错误示范（双转事故）：先把 21:00 自行转成 "2026-07-11 13:00:00" 再写 —— 用户会看到 05:00
```

## SELF-CHECK（生成代码前逐条确认）

```
□ 代码里没有任何 time.LoadLocation / 时区偏移加减作用于业务日期值        (N1)
□ 写入串是空格分隔规范形态，无 T/Z                                     (M2/N2)
□ 未用服务器本地时间冒充用户墙钟；造数 SQL 用 UTC_TIMESTAMP()            (N3)
□ SQL 无 auto×naive 互比；按日分组已确认 UTC 日语义可接受或已换边界      (N4/M4)
□ 若消费取数结果中的 auto 字段：已按 F12 遗留处理并留标记注释            (F12)
□ 未把"查库≠显示"当 bug 修                                            (N6)
```
