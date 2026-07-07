# 查询体系（代号 艾丽丝/Alice）—— 面向 AI 编程助手的结构化上下文

> 用法：把本文整体喂给你的 AI 编程助手。它是 `for-human-developers.md` 的机读版：FACTS / INTERFACE / NULL-SEMANTICS / MUST-MUST NOT / 典型序列 / 自检清单。
> 术语：Alice=宿主查询体系（表达式编译+执行）；Aria=表单数据权威层 Mirror；模型=ExpressionModel JSON。

## FACTS（可依赖的事实）

- F1 角色分工：插件提交 ExpressionModel → 宿主 Alice 编译成 SQL 并只读执行 → 行集回插件 → 插件内存加工 → 经 Aria Mirror 写回。插件全程零 SQL、零数据库连接。
- F2 两条通道：文本 KimpoSQL（单表 SELECT，无 JOIN/GROUP BY，能力受限）与表达式模型通道（多表 JOIN/分组/开窗/上下文物化，全能力）。新开发一律模型通道。
- F3 来源表三维标记：source_type 恒 'template'（view/other 未开发即拒）；is_context bool（true=本报表上下文，运行期物化 Mirror 快照为 ctx_<alias> 临时表，按 sheet_session 隔离，含未落库脏值）；role main/detail（驱动编辑器注入 record_id 自动左关联；后端只消费不校验）。
- F4 条件模型：where（ConditionGroup 布尔树 {logic:and|or, negate, items[]}，叶子 Predicate{left,op,right}）与 where_expr（raw 整段）互斥。CompareOp = eq|neq|lt|lte|gt|gte|like|in|not_in|is_null|is_not_null。
- F5 ValueRef 8 类：source_field / system_var / const / func / raw / window / case / form_field。编辑器不产 form_field——本报表字段=is_context 表的 source_field（别名 bt*）。
- F6 函数白名单（canonical，别名→canonical 归一）：coalesce(IFNULL/ISNULL)、concat、contains_text、starts_with、ends_with、date_add_day、date_diff(DATEDIFF/日期间隔; unit 'd'|'m')、round、now_local、now_utc、today。白名单双源：funcs.DefaultRegistry 与 bind.functionSignatures 必须同步登记。
- F7 查询侧属性 DISTINCT/LIMIT/ORDER BY/GROUP BY 全部下推 SQL；插件内存只执行 fill_strategy/separator/clear/post_sort。
- F8 SELECT 列名契约：o_<目标字段ID>=输出列；m_<seq>=匹配键列（ColumnMeta.form_field_id 仅此类注入）；_w<i>=开窗内层；__ord_row_no=隐式稳定排序键。
- F9 按行列匹配=插件内存左驱 left join：本报表明细行为左表；0 命中保留原行、1 命中回填、N 命中复制左行扩行；来源多出的键不产新行。匹配键两来源合并：WHERE 自动提取(RefFormField 判据，编辑器路径不触发)+Target.MatchKeys 手动（编辑器唯一实际入口）。
- F10 grant：读 querygrant / 写 writegrant，call-scoped 一次性。普通写回=字段级白名单；仅 match_keys 场景签发表级 FieldID:"*" 通配。
- F11 compile-preview 与运行态同源编译；预览 SQL=真实运行 SQL；预览响应含被 F7 裁剪的"未生效条件"清单。

## NULL-SEMANTICS（空值确定行为，契约=docs/architecture/field-null-semantics-contract-2026-07-07.md）

- N1 存储诚实：未填=NULL。TypedValue{raw:null,text:""}。禁止任何环节写哨兵（""/0/1900-01-01）。文本写入口归一：全空白→null（系统中不存在空字符串值）；boolean 无 null（未勾=false）。
- N2 表达式域=SQL 语义：空传播（空+1=空；date_diff 入参空→空）；聚合跳空（SUM 全空=0 由编译器包 COALESCE；AVG/MIN/MAX 全空=null；COUNT(字段) 不计空）；比较不命中空。显式兜底=coalesce。唯一内置例外：concat 编译为跳空形态（CONCAT_WS）。
- N3 F7 固定族规则（无用户开关；判定=绑定期空值传播分析，谓词恒 UNKNOWN → 从布尔树中性删除：AND 删子、OR 删子、组空则组亡、根空则无 WHERE）：
  - 为空则不限（条件视同没写）：> >= < <=（含 date_diff 区间形态）、like/contains/starts/ends、neq、in/not_in 空参数列表。
  - 为空则不取（条件保留、恒不命中）：eq。
  - 参数定义：system_var + is_context 主表（单行快照）字段引用 + const 折叠链。is_context 扩展表引用=行数据，永不裁剪。可判定真/假的条件（门闸 参数='x'、守卫 参数 is_not_null）非 UNKNOWN，永不裁剪。
  - 取不到参数值（无会话/读取失败）→ 不裁剪，回退严格 SQL（fail-open to strict）。
- N4 覆盖惯例（用户老写法，天然优先于固定规则）：`或者 <参数> is_null` = 强制不限；`并且 <参数> is_not_null` = 强制没填不取。兼容别名：右值空字符串常量的 eq/neq 编译为 IS NULL/IS NOT NULL。
- N5 匹配键红线：内存 join 匹配键任一分量为空（nil 或全空白串）→ 源行不参与匹配、目标行不入索引。禁止 nil 序列化成 "" 参与 map 键。
- N6 排序空值恒末尾（编译器注入 `expr IS NULL` 前缀键）；分组空值单独一组，展示「(空白)」；写回空=真实清空（raw:nil），未命中行为走 clear 策略。

## INTERFACE（怎么调）

- I1 反向查询（插件→宿主）：`host.Query().SubmitExpression(ctx, modelJSON, triggerContext, paramValues) → (rowsJSON, columnsJSON, err)`。rows=[]map[colName]any。旧 SubmitQuery(kimpoSQL) 仅单表场景。
- I2 触发上下文注入（原样取用禁伪造）：query_context.sheet_session_id / write_grant / system_vars。
- I3 写回：`host.Mirror().WriteField/WriteFields`（批量分批=一批一个屏障窗口）；行操作 AppendDetailRow/RemoveDetailRow/ReorderDetailRows；错误语义遵守 aria 契约（AppliedRevision>0=权威已写仅渲染失败，按成功处理）。
- I4 设计态：动作配置 surface 挂共享包 @kimpo/expression-editor；校验/排障调 POST …/expression/compile-preview（dry-run）与 …/expression/parse-sql（纯语法）。

## MUST

- M1 查询只走 SubmitExpression；写只走 Mirror/Record 通道。
- M2 新增函数同时改 funcs.DefaultRegistry + bind.functionSignatures，并补编译/绑定两层测试。
- M3 写回空值用 raw:nil；读到 null 保持 null 加工，需要数值语义时显式 coalesce 在**表达式里**做。
- M4 匹配键构造遵守 N5；多分量键任一空即整键无效。
- M5 排障顺序：compile-preview 看 SQL 与"未生效条件"清单 → 执行日志（host.Logger() debug）→ 再查插件加工。

## MUST NOT

- X1 不拼 SQL、不猜物理表名（app_table_*）、不 import editorpb。
- X2 不缓存/复用/伪造 grant。
- X3 不在插件内存做 DISTINCT/LIMIT/查询排序（SQL 已做）。
- X4 不把空转成 0/""/1900 再写回；不实现"空键互配"的匹配逻辑。
- X5 不手工构造 form_field 引用（编辑器不产、无 UI 支撑）；本报表字段用 is_context source_field。
- X6 不假设 expression 通道有行级数据域权限（已知缺口，仅软删注入）。

## 典型序列（取数动作）

1. 设计态：编辑器产出模型（source: 显式表 + 按需 bt* 上下文表；where 条件树；target: outputs+match_keys+clear+post_sort）→ compile-preview 校验。
2. 运行态：动作触发 → 插件取 query_context → SubmitExpression → 行集(o_*/m_*) → applyFillStrategy / 左驱匹配 / clear / post_sort → Mirror 写回（grant 白名单内）→ 完。

## 自检清单（提交前）

```
□ 模型只含稳定 ID（template_id/data_table_id/field_id），无物理名无裸 SQL
□ 新函数两处登记 + 单测（编译渲染 + 绑定签名 + 空值行为）
□ 空值：无任何 空→0/"" 的暗转换；写回空=raw:nil
□ 匹配键：空分量行不参与；无 nil→"" 序列化
□ 条件测试覆盖：参数空的四象限（区间双参数）、= 恒不命中、escape/守卫覆盖默认、门闸不被裁
□ preview 的"未生效条件"清单在排障文档/日志中可见
□ DISTINCT/LIMIT/排序未在插件侧重复实现
□ grant 未缓存；错误语义按 aria 决策表处理
```
