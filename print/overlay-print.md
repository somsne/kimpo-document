# 套打（预印刷纸精确定位）· 开发教程

> ⚠️ 底座实施中：manifest/spec 已按 2026-07-15 契约拍板定稿、插件可被宿主真实加载；但打印执行底座（P1/P2）尚未开发，本插件当前无法在真实打印流程中被触发执行。契约以宿主仓 docs/sheet/future/14-printing/ 设计文档为准。

参考实现：[`kimpo-print-extension/print-demo-overlay/`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-overlay/)（快递面单套打）。

---

## 一、打印效果

想象一叠从快递公司领来的**预印刷面单**：纸上已经印好了边框、"收件人/寄件人"栏位标签、快递公司 Logo。你的任务只是把**数据**打进对应的空格里。本插件打出来是：

- 收件人姓名、电话、地址，寄件人姓名、电话、地址——各自落在面单对应空格内，位置按**毫米**精确对齐；
- 面单右上角一枚 **24×24mm 二维码**，内容是当前单据 ID，扫码即可回查系统单据；
- 底部一行小字"打印人：张三 — 2026-07-15 14:30"作为打印留痕。

在系统的**打印预览**里，你会看到面单底图和数据叠在一起（所见即所得，方便对位）；真正送到打印机时**只打数据不打底图**——因为底图已经印在纸上了。

## 二、适用场景

一切"纸是现成的、只补数据"的业务：

- 快递/物流面单（本例）；
- 增值税发票、银行进账单、支票；
- 政府/行业的制式表单（预印刷套表）。

共同特征：排版由纸决定，插件只负责"哪个字段打在哪个坐标"。这类需求用 A 档（静态 spec 模板）零代码即可完成。

## 三、插件包结构

```
print-demo-overlay/
├── manifest.json              # 插件声明 + print-mode 能力
└── specs/
    └── express-sheet.json     # 面单套打的 PrintJobSpec 模板（A 档）
```

没有任何后端代码——A 档插件就是"一份声明 + 一份排版模板"。面单底图放在插件包的 `frontend/` 目录下（宿主 assets 端点只 serve `frontend/` 子树），spec 里用 `url` 直链引用（见 §五）；也可以走宿主资产存储上传后用 `assetId` 引用，二者二选一。

## 四、manifest 逐段讲解

```json
"plugin_id": "com.kimpo.demo.print-overlay",
"category": "action",
```

print-mode 是**能力（capability）不是插件类别**：任何类别的插件都可以携带它，宿主按 capabilities 分发。这里选 `action` 是因为套打天然适合被动作按钮触发（见 §七）。

```json
"capabilities": {
  "print_modes": [
    {
      "id": "express-sheet",
      "name": { "zh_CN": "快递面单套打", "en_US": "Express Waybill Overlay" },
      "modes": ["design-preview", "form-preview", "form-print", "action"],
      "spec_source": "static",
      "spec": "specs/express-sheet.json"
    }
  ]
}
```

> **2026-07-15 实施修订**：宿主 `Manifest.Capabilities` 是对象（不是数组），print-mode 落在 `capabilities.print_modes` 数组字段，不是旧稿 `capabilities: [{type:"print-mode"}]` 的数组形；字段全 snake_case，不再有单独的 `type` 字段。

| 字段 | 说明 |
|---|---|
| `id` | 实例内唯一的模式标识；模板绑定方案时存的引用就是 `pluginId + modeId` |
| `name` | 多语言名称，出现在打印预览对话框的"打印方案"下拉里 |
| `modes` | 适用场景：设计态预览 / 填报态预览 / 填报态直打 / 动作节点触发。面单四态全开，尤其 `action`——套打最自然的入口是交互按钮 |
| `spec_source` | `static` = A 档：包内静态 spec 模板 + 变量表达式，零前端代码。逻辑复杂才需要 B 档（`frontend` + `entry` 指向 buildSpec 函数名） |
| `spec` | 包内 spec 模板的相对路径；宿主注册插件时读取该文件**内联进能力声明**，随会话 `meta.printModes[]` 一并下发 |

本例没有配 `params`（可选字段）：面单的坐标是纸决定的死值，没有需要设计者调的参数。

## 五、spec 逐段讲解

完整文件见 [`specs/express-sheet.json`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-overlay/specs/express-sheet.json)。逐段看：

### 5.1 底图图层（套打的灵魂）

```json
"layers": [
  { "type": "background",
    "url": "/api/v1/plugins/com.kimpo.demo.print-overlay/assets/frontend/waybill-bg.png",
    "fit": "page" }
]
```

- `type: "background"`：全页底图图层，放面单的扫描图/设计稿。**它只在预览时可见**，用途是给设计者和填报者对位——数据框是否落在面单空格里，一眼可见。
- **真实打印时的开关**：纸上已经印了底图，所以走真实打印（预印刷纸）时以 `printBackground:false` 语义隐藏底图、只打数据。编辑器段以透明底渲染叠上（网格线固定不打印），这是契约 §4 的套打要点。若你要打在**白纸**上连底图一起打（如内部留档），则打开 printBackground 即可，同一份 spec 两用。
- **图片源二选一（2026-07-15 拍板）**：`assetId`（宿主资产存储，须先上传拿到 `ast_xxx`）或 `url`（直链）。本例把底图直接放进插件包 `frontend/waybill-bg.png`，走宿主既有插件资产端点 `/api/v1/plugins/{plugin_id}/assets/{path}`（该端点只 serve `frontend/` 子树）用 `url` 引用，插件包自带即用、无需额外上传步骤。业务场景需要设计者自行换底图时，`assetId` + 资产存储上传流程仍是更合适的选择（有权限与缓存管理）。spec 是会被存库复用、随方案绑定下发的纯 JSON，两种引用方式都不会把 base64 大图内联进 spec 本身。

### 5.2 页面模板与数据框

```json
"pageTemplate": {
  "paper": "A5", "orientation": "portrait",
  "margins": { "t": 0, "b": 0, "l": 0, "r": 0 },
  "frames": [ ... ]
}
```

- `pageTemplate` 存在时**接管分页布局**（L2 框式编排），页面完全由 frames 摆出来；
- 边距全 0：套打坐标以**纸张物理左上角**为原点量取，和你拿尺子在真实面单上量的毫米数一一对应，不要让边距再偏移一次。

每个数据框一个 frame，`rect` 单位是 **mm**：

```json
{ "id": "recipient-name",
  "rect": { "x": 24, "y": 30, "w": 50, "h": 8 },
  "content": { "type": "text", "expr": "{{record.recipientName}}",
               "style": { "size": 12, "align": "left" } } }
```

- `rect`：拿真实面单量出来的坐标——收件人姓名空格左上角距纸左边 24mm、距顶 30mm，宽 50mm 高 8mm；
- `expr`：`{{record.*}}` 命名空间取当前单据主表字段（权威来源是表单数据权威层 Mirror，按字段可见性裁决后下发）；
- 收件人电话/地址、寄件人三项同理，只是坐标和字号不同——地址框更高（14mm）以容纳换行。

二维码框：

```json
{ "id": "waybill-qrcode",
  "rect": { "x": 118, "y": 6, "w": 24, "h": 24 },
  "content": { "type": "barcode", "format": "qrcode", "expr": "{{record.id}}" } }
```

`barcode` 内容框由编辑器负责生码渲染，插件只声明格式和内容表达式；绑定单据 ID 让扫码回查闭环。

打印留痕框混用了两套变量：

```json
"expr": "打印人：{{user.name}} — &D &T"
```

`{{user.*}}` 取当前用户，`&D`/`&T` 是沿用页眉页脚的日期/时间变量（按**用户时区**渲染）。

### 5.3 出口

```json
"output": { "kind": "print", "fileName": "{{template.name}}_{{record.id}}.pdf" }
```

`print` = 直接唤起浏览器打印（面单场景就是"点了就出纸"）；`fileName` 供切到 `pdf` 下载出口时命名用，支持变量。

## 六、设计态调用

1. 设计者在模板设计器点工具栏"打印"，打开打印预览对话框；
2. 设置区顶部的**"打印方案"下拉**里，选中"快递面单套打"（即本插件注册的 print-mode）；
3. 编辑器读取包内静态 spec，用**样例数据**（设计态没有真实单据，按字段类型自动生成）渲染 → **右侧预览所见即所得**：面单底图 + 叠在上面的样例数据，坐标对不对当场看出；对不上就改 spec 里的 mm 值再看；
4. 对位满意后，把该方案**绑定为此模板的默认打印方案**（可绑多个），随页面设置一起保存——绑定只存方案引用（pluginId + modeId + 参数值），不内联 spec 全文。

## 七、填报态调用

绑定之后，填报态**三个既有入口**（打印按钮 / Ctrl+P / 宿主 `openPrintPreview()`）都会按绑定方案走：单方案 + 直打策略时直接 `spec + 当前单据数据 → 渲染 → 唤起浏览器打印`，用户无感知；多方案则弹轻量选择。

**动作按钮触发（套打最顺手的入口）**：把"打印"配置成动作节点——在模板上放一个交互按钮，动作配成"点击 → 用快递面单方案打印当前单据"。仓库发货员在单据上点一下"打面单"，出纸，完事。动作上下文还可以带**行级目标**：比如列表里每行一个打印按钮，点哪行打哪行对应的单据。

## 八、权限与边界

- **权限执法在编辑器/宿主侧，插件不可绕过**：一切打印模式统一过 **卡侬（赋权层）Print 位 AND 模板策略三项**（RuntimeAllowPrint / Preview / PaperOverride）；PDF 出口另过 ExportPdf 位，三位独立。
- **字段可见性与表单会话同口径**：spec 里 `{{record.recipientPhone}}` 能不能取到值，取决于当前用户在表单会话里能不能看到这个字段——卡侬裁决后不可见的字段，插件拿不到，表达式渲染为空。套打不会成为越权导出通道。
- 一期**不支持跨单据/跨模板取数**：spec 只能引用当前单据上下文；批量打面单属 P3 批量场景，另有受控批量上下文。
- 表达式是**受限求值**（无任意 JS），在编辑器侧执行；A 档插件全程零代码，这是安全默认。
