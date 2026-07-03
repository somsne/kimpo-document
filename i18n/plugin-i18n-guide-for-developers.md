# Kimpo 插件多语言（i18n）开发指南 —— 人类开发者版

> 适用对象：Kimpo 插件开发者（前端 UMD + 后端 Go sidecar）。
> 方案权威出处：主仓 `docs/architecture/plugin-i18n-two-level-fallback-plan-2026-07-03.md`。
> 本文以该方案**已实现**为前提，讲"你开发插件时该怎么做"。

---

## 1. 一分钟理解整套方案

核心原则只有两条：

1. **插件的文案由插件自己维护**——语言文件放在你的插件仓库里，随插件一起构建、发布，不放宿主 kimpo。
2. **语言跟随宿主自动切换**——用户在 kimpo 顶部切换语言，你的插件界面和后端返回的消息会实时跟着变，你不需要（也不允许）自己去探测用户语言。

文案查找是一条**两级回退链**（插件优先）：

```
用户看到一段文字，它是这样被找到的：

① 你的插件语言包 [当前语言]        ← 绝大多数命中在这里
② 宿主中央库 HostI18n.t(key)      ← 只兜两类：a) 管理员给你的插件上传补充的语言
                                              b) 公共词表（common.* / err.plugin.*）
③ 你的插件语言包 [默认语言]        ← 文案兜底
④ key 原样显示                    ← 你漏翻了，界面上会直接看到 key（这是特性，不是 bug）
```

为什么插件优先？因为你的语言包和你的组件代码是**同一次构建**打包的，永远同版本；宿主中央库里那份只是安装时的快照，开发期必然陈旧。插件优先意味着你 redeploy 之后新文案**立即生效**，不需要重装插件。

---

## 2. 你需要做的三件事（Quick Start）

### 第一件：在 manifest 里声明 i18n

```jsonc
// manifest.json
{
  "name":        { "zh_CN": "文档插件", "en_US": "Document Plugin" },   // 注意：manifest 里用下划线 zh_CN
  "description": { "zh_CN": "……",      "en_US": "..." },
  "i18n": {
    "namespace": "com.kimposoft.document",   // 缺省 = plugin_id，一般不用写
    "locales": ["zh-CN", "en-US"],           // 你实际提供的语言
    "path": "frontend/locales",              // 缺省值，一般不用写
    "default_locale": "zh-CN"                // 查找链第 ③ 级用
  }
}
```

⚠️ 一个历史遗留的不一致，记住就好：**manifest 的 name/description 多语言键用下划线**（`zh_CN`/`en_US`），**语言文件名和 i18n.locales 用连字符 BCP-47**（`zh-CN.json`/`en-US.json`）。不要混。

### 第二件：写前端语言文件 + 用 t() 取文案

```
your-plugin/
├─ manifest.json
├─ frontend/
│  ├─ locales/
│  │  ├─ zh-CN.json      ← 扁平点分 key
│  │  └─ en-US.json      ← key 集合必须与 zh-CN.json 完全一致
│  └─ src/...
└─ cmd/plugin/
   └─ locales/            ← 后端文案（见第三件）
      ├─ zh-CN.json
      └─ en-US.json
```

语言文件是扁平 JSON，key 用点分层级、按 `模块.位置.语义` 命名：

```json
// frontend/locales/zh-CN.json
{
  "panel.title": "文档设置",
  "panel.upload.hint": "拖拽文件到此处，或点击选择",
  "panel.upload.size_limit": "文件不能超过 {max} MB",
  "action.export.title": "导出文档"
}
```

组件里通过 SDK 的 i18n helper 取文案，**不要**自己拼接、不要硬编码。宿主注入的 `HostI18n`
按加载链取：editor surface 在 `props.hostBridge?.i18n`，action-config surface 在
`input.hostBridge['i18n']`：

```ts
import { createPluginI18n, type HostI18n } from '@kimpo/plugin-sdk'
import zhCN from '../locales/zh-CN.json'
import enUS from '../locales/en-US.json'

// surface 挂载时初始化一次（host 不传也能跑——降级为纯本地 ①③④ 链，正式代码必须传）
const i18n = createPluginI18n({
  messages: { 'zh-CN': zhCN, 'en-US': enUS },
  defaultLocale: 'zh-CN',
  host: props.hostBridge?.i18n as HostI18n | undefined,
  namespace: 'com.kimposoft.document' // = manifest.i18n.namespace，管理员补 locale 场景要用
})

// 组件中：
i18n.t('panel.title')                          // → "文档设置"
i18n.t('panel.upload.size_limit', { max: 10 }) // → "文件不能超过 10 MB"
i18n.getLocale()                               // 当前语言（'zh-CN'）
```

`createPluginI18n` 内部就是第 1 节那条四级链，并且已经订阅了宿主的 `onLocaleChange`——
**用户在宿主切语言，i18n.t 的返回立即切换**。SDK helper 零依赖（不含 Vue），Vue 组件要让
模板跟着重渲染，用一个 ref 薄包一层即可（组件卸载时记得 `i18n.dispose()`）：

```ts
const localeRef = ref(i18n.getLocale())
i18n.onChange(l => { localeRef.value = l })   // 模板里引用 localeRef 即获得响应式重渲染
```

### 第三件：后端文案 + message_key 错误契约

宿主调用你的动作时，`TriggerContextMsg.locale` 带了用户当前语言（HTTP 边界协商后 proto 透传）。用 SDK 的 Go helper：

```go
import (
    "embed"

    "github.com/kimposoft/kimpo-plugin-sdk/direction/biz"
    "github.com/kimposoft/kimpo-plugin-sdk/gen/bizpb"
    kimpoi18n "github.com/kimposoft/kimpo-plugin-sdk/i18n"
)

//go:embed locales/*.json
var localeFS embed.FS

// 初始化一次（包级或 NewService 里）
var tr = kimpoi18n.MustNew(localeFS, "locales", "zh-CN") // 目录、默认语言

func (s *Service) Execute(ctx context.Context, req *bizpb.ExecuteRequest) (*bizpb.ExecuteResponse, error) {
    loc := req.GetTriggerContext().GetLocale() // 宿主透传的用户语言，如 "en-US"；空=老宿主，T 自动回落默认语言

    // 正常提示文案：
    msg := tr.T(loc, "export.done", map[string]any{"count": n})

    // 错误必须走 message_key 契约（两级链的后端形态）：
    if err != nil {
        // 本地有翻译 → fallback 给译文；本地 miss（T 返回空串）→ 宿主拿 key 按请求 locale 去中央库译
        return biz.Fail("EXPORT_FAILED", "export.failed", tr.T(loc, "export.failed", nil)), nil
        //              ↑错误码            ↑message_key       ↑fallback 文本
    }
    // ...
}
```

规则：**用户可见的错误一律走 `biz.Fail`（带 message_key）**；`tr.T` 双 miss 返回空串时不要用 key 顶文案位，宿主会兜底。日志（Logger 上报给宿主的）不用翻译，统一写中文或英文都行，那是给开发者看的。

---

## 3. 公共词表：你可以"白嫖"什么

为了不让每个插件都翻译一遍"确定/取消"，第 ② 级回退允许你直接使用一份**由 SDK 冻结的公共词表**——这些 key 不要放进你自己的语言包，直接 `i18n.t('common.confirm')` 用：

| 类别 | key 示例 | 说明 |
|---|---|---|
| 通用词 | `common.confirm` `common.cancel` `common.save` `common.delete` `common.edit` `common.search` `common.loading` `common.empty` `common.success` `common.failed` | 确定/取消/保存/删除/编辑/搜索/加载中/暂无数据/操作成功/操作失败 |
| 框架错误 | `err.plugin.*` | 插件不存在、实例不存在等框架级错误 |

**红线：清单外的任何 key，你都不许指望宿主有。** 宿主只对清单内的 key 承诺不改名。你的业务文案（哪怕看起来很通用，比如"导出"）一律自带。判断标准很简单：不在上面清单里 → 写进自己的语言包。

违反的后果：宿主哪天重构了它自己的某个 key，你的界面静默漏翻，而且没人会想到查你的插件。

---

## 4. 常见坑（每条都有人踩过）

1. **不要自己探测语言。** 不许读 localStorage、不许看 `navigator.language`、不许写死 `zh-CN`。语言只有一个来源：宿主注入的 `HostI18n`（前端）/ 请求透传的 locale（后端）。历史上 kimpo-sheet 靠扫 localStorage 猜语言，导致同页切换不生效，已按本方案改造，不要参考旧代码。
2. **界面出现 key 原样（如 `panel.title`）不是框架 bug，是你漏翻了。** 四级链故意不做"随便找个语言顶上"，就是为了让漏翻在开发期一眼可见。补齐对应语言文件即可。
3. **两份语言文件的 key 集合必须完全一致。** zh-CN 有、en-US 没有的 key，在英文环境会走到第 ③ 级显示中文——功能上能跑，但这是欠账，提交前对齐。
4. **redeploy 即生效。** 改文案不需要重装插件、不需要重启 api。如果你改了文案没生效，先怀疑构建缓存（`pnpm install --force` 那个坑），不要怀疑 i18n 链。
5. **manifest 下划线、文件名连字符**（见第 2 节第一件事）。
6. **aria-label、placeholder、空状态提示也是文案。** 最容易漏的三个地方，它们同样要走 `t()`。
7. **管理员可以给你的插件补语言。** 你只发了中英，某实例管理员可以上传日文包（落在你的 namespace），通过第 ② 级生效。所以不要在代码里假设"语言只可能是我 locales 目录里的那几种"——任何 locale 字符串都可能进来，helper 已处理，你别自己 switch 语言。

---

## 5. 提交前自查清单

- [ ] `manifest.json` 有 `i18n` 声明，name/description 有 `zh_CN` + `en_US`；
- [ ] `frontend/locales/` 与 `cmd/plugin/locales/` 各语言文件 key 集合一致；
- [ ] grep 不到硬编码用户可见文案（中文正则 `[一-鿿]` 扫 .vue/.ts 模板段，排除注释与日志）；
- [ ] 用户可见错误全部走 `biz.Fail(errorCode, message_key, fallback)`；
- [ ] 公共词表之外没有依赖任何宿主 key；
- [ ] 真机验证：宿主切语言（不刷新页面），插件界面实时切换；redeploy 后新文案立即生效。
