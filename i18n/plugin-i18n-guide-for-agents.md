# Kimpo 插件多语言（i18n）适配规范 —— Agent 版

> 本文件面向在 Kimpo 插件仓库中执行开发任务的 AI agent。规则为强约束（MUST / MUST NOT），
> 配方为确定性步骤。方案权威出处：主仓 `docs/architecture/plugin-i18n-two-level-fallback-plan-2026-07-03.md`。
> 人类可读的讲解版见同目录 `plugin-i18n-guide-for-developers.md`。

## 0. 心智模型（一段话）

插件文案权威源 = 插件仓库内语言文件，随构建打包，插件优先；宿主中央库只兜两件事：管理员为该插件上传的补充语言、冻结公共词表。查找链固定四级：`插件包[当前locale] → HostI18n.t(key) → 插件包[默认locale] → key 原样`。语言值唯一来源是宿主（前端注入 HostI18n / 后端请求透传 locale），插件侧禁止任何形式的语言探测。

## 1. 硬规则

### MUST

- R1. 任何**用户可见**文本（模板文本、placeholder、aria-label、空状态提示、按钮、tooltip、错误消息、动作执行结果）MUST 通过 i18n 链取得：前端 `i18n.t(key, params?)`，后端 `tr.T(locale, key, params)`。
- R2. 新增 key 时 MUST 同步写入该插件**所有** locale 文件（`frontend/locales/*.json` 或 `cmd/plugin/locales/*.json`），保持各文件 key 集合完全一致。
- R3. 后端用户可见错误 MUST 走 `biz.Fail(errorCode, messageKey, fallback)` 三参形态（`direction/biz`）；fallback = `tr.T(loc, messageKey, params)` 的结果（miss 时为空串，由宿主中央库兜底，勿用 key 顶文案位）。
- R4. 前端初始化 MUST 把宿主注入的 HostI18n（editor surface = `props.hostBridge?.i18n`；action-config surface = `input.hostBridge['i18n']`）传给 `createPluginI18n({ messages, defaultLocale, host, namespace })`；Vue 模板响应式用 ref 薄包 `i18n.onChange`，组件卸载时 `i18n.dispose()`。
- R5. manifest MUST 含 `i18n` 声明（`locales`、`default_locale`；`namespace`/`path` 用缺省可省）；`name`/`description` MUST 为 `{"zh_CN": ..., "en_US": ...}` 结构。
- R6. key 命名 MUST 为扁平点分：`<模块>.<位置>.<语义>`，全小写下划线单词（例 `panel.upload.size_limit`）；插值用 `{name}` 占位。

### MUST NOT

- N1. MUST NOT 探测语言：禁止读 localStorage / `navigator.language` / cookie，禁止写死 locale。唯一来源 = HostI18n（前端）/ `req.GetTriggerContext().GetLocale()`（后端）。
- N2. MUST NOT 依赖公共词表（§3）之外的任何宿主 key。业务文案哪怕语义通用（如"导出"）也必须自带。
- N3. MUST NOT 为漏翻做"就近顶替"（如英文缺就手动填中文值到 en-US.json）。缺翻译就留空缺，让第 ④ 级显示 key 原样暴露问题；除非任务本身就是补翻译。
- N4. MUST NOT 把插件文案写入宿主侧文件（`apps/web-next/src/locales/`、`services/api/internal/modules/i18n/seed/`、`i18n-packs/`）。发现既有寄养文本，不迁移，报告用户（回迁是独立任务 P5）。
- N5. MUST NOT 在 Logger 上报的日志里使用 i18n（日志面向开发者，直接写中文，不翻译）。
- N6. MUST NOT 混淆两种 locale 写法：manifest `name`/`description` 键 = 下划线 `zh_CN`；语言文件名与 `i18n.locales` 数组 = 连字符 `zh-CN`。

## 2. 契约速查

```
前端查找链（createPluginI18n 内置，勿自行实现）:
  ① messages[locale][key] → ② host.t(hostKey)（common.* 原样 / err.plugin.* 前缀 backend. /
    自身 key 前缀 <namespace>.）→ ③ messages[defaultLocale][key] → ④ 返回 key 字符串

前端 API（@kimpo/plugin-sdk，零依赖）:
  createPluginI18n({ messages, defaultLocale, host?, namespace? })
    → { t(key, params?), getLocale(), onChange(cb)→unsub, setLocale(l)/*仅无宿主时生效*/, dispose() }
  HostI18n（宿主注入；editor surface = props.hostBridge?.i18n，action-config = input.hostBridge['i18n']）:
    { locale /*Ref 结构*/, t(key, params?) /*缺 key 返回 ''*/, resolve(localizedText), onLocaleChange(cb)→unsub }
  COMMON_VOCABULARY / ERR_PLUGIN_PREFIX：冻结公共词表的机器可读清单

后端 API（Go SDK）:
  //go:embed locales/*.json → kimpoi18n.MustNew(fs, "locales", "<default_locale>")
    // import kimpoi18n "github.com/kimposoft/kimpo-plugin-sdk/i18n"
  tr.T(locale, key, params)                    // 本地两级：locale → default → ""（空串=miss）
  req.GetTriggerContext().GetLocale()          // 宿主透传的用户 locale（TriggerContextMsg.locale）
  biz.Fail(errorCode, messageKey, fallback)    // ExecuteResponse.message_key，宿主按请求 locale 中央库兜底重译

文件落点:
  manifest.json                  i18n 声明 + name/description 多语言
  frontend/locales/<locale>.json 前端文案（扁平 JSON）
  cmd/plugin/locales/<locale>.json 后端文案（扁平 JSON）
  契约文档: plugins/kimpo-plugin-sdk/docs/02-i18n-contract.md（公共词表冻结清单权威落点）
```

## 3. 公共词表（第②级唯一许可依赖，冻结清单）

`common.confirm` `common.cancel` `common.save` `common.delete` `common.edit` `common.search` `common.loading` `common.empty` `common.success` `common.failed` 以及 `err.plugin.*` 全族。SDK 文档中的冻结清单为准；本清单外一律视为不存在。

## 4. 任务配方

### 配方 A：给插件新增一段用户可见文本

1. 定 key（R6 命名）；确认非公共词表语义（是 → 直接用 `common.*`，不进语言包）。
2. 写入该端**全部** locale 文件（前端改 `frontend/locales/*.json`，后端改 `cmd/plugin/locales/*.json`）。
3. 代码处调用 `i18n.t('...')` / `tr.T(loc, "...", nil)`。
4. 自检（§5）。

### 配方 B：新增/改造一个前端界面（组件、面板、对话框）

1. 组件内所有文案走 `i18n.t()`——逐项检查：标签、按钮、placeholder、aria-label、空状态、tooltip、确认弹窗。
2. 通用按钮用公共词表（`common.confirm` 等）。
3. 若组件展示宿主下发的多语言结构字段（`{zh_CN:..., en_US:...}`），用 `host.i18n.resolve(value)`，不要自己按 locale 取字段。
4. 新增 key 按配方 A 步骤 2。

### 配方 C：新增后端动作 / 错误路径

1. 用户可见成功消息、错误消息全部入 `cmd/plugin/locales/*.json`。
2. 错误返回一律 `biz.Fail(code, messageKey, tr.T(req.GetTriggerContext().GetLocale(), messageKey, params))`。
3. 纯内部错误（不出现在 UI，只进日志）不翻译（N5）。

### 配方 D：为全新插件初始化 i18n

1. manifest 加 `i18n` 块 + `name`/`description` 双语（R5，注意 N6 键名格式）。
2. 建 `frontend/locales/zh-CN.json` + `en-US.json`（无前端则跳过）；建 `cmd/plugin/locales/` 同理。
3. 前端入口 `activate(ctx)` 中 `createPluginI18n` 接线（R4）；后端 `MustNew` 接线。
4. 从脚手架带出的硬编码文案全部迁入语言文件。

### 配方 E：为插件补一门语言（如 ja-JP）

1. 复制 `zh-CN.json` 为 `<locale>.json`，逐 key 翻译，不得删 key、不得留中文值（留空缺不如不建文件——半成品文件会挡住第③级默认语言兜底，宁可整文件完整）。
2. manifest `i18n.locales` 数组追加该 locale。
3. 前端 `createPluginI18n` 的 `messages` 映射同步追加 import。

## 5. 自检命令（改动后必跑）

```bash
# 1) 硬编码中文扫描（前端模板/脚本；人工排除注释与 console/Logger 行）
grep -rnE '[一-鿿]' frontend/src --include='*.vue' --include='*.ts' | grep -v '^\s*//' | grep -viE 'console|logger|logf'

# 2) 后端用户可见硬编码扫描（重点 Err/错误返回行）
grep -rnE '[一-鿿]' cmd/plugin --include='*.go' | grep -viE '^\s*//|logf|logger'

# 3) locale 文件 key 集合一致性（两两比对，diff 输出必须为空）
for d in frontend/locales cmd/plugin/locales; do
  [ -d "$d" ] && diff <(jq -r 'keys[]' $d/zh-CN.json | sort) <(jq -r 'keys[]' $d/en-US.json | sort)
done

# 4) manifest 声明存在性
jq '.i18n, .name.zh_CN, .name.en_US' manifest.json
```

真机验收（有条件时）：宿主切语言不刷新页面，插件 UI 实时切换；redeploy 后新文案立即生效（插件优先链的特征，若不生效先查构建缓存而非 i18n）。

## 6. 反模式对照

| ❌ 禁止 | ✅ 正确 |
|---|---|
| `<span>请选择</span>` | `<span>{{ i18n.t('select.placeholder') }}</span>` |
| `localStorage.getItem('lang')` | `hostBridge.i18n` 的 locale/onChange |
| `if locale == "zh-CN" { msg = "失败" } else { msg = "failed" }` | `tr.T(loc, "export.failed", nil)` |
| `return errors.New("解析设置失败")`（用户可见） | `return biz.Fail("BAD_SETTINGS", "settings.parse_failed", tr.T(loc, "settings.parse_failed", nil)), nil` |
| en-US.json 缺 key 时填中文顶上 | 补真翻译，或留缺让 key 原样暴露 |
| 把插件文案加进 `apps/web-next/src/locales/` | 写进插件自己的 `frontend/locales/` |
| 自实现四级查找链 | 用 SDK `createPluginI18n` / `kimpoi18n.MustNew` |
