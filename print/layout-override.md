# 版式覆盖 / 局部打印 · 打印回执（A 档纯声明式）

> ⚠️ 底座实施中：manifest/spec 已按 2026-07-15 契约拍板定稿、插件可被宿主真实加载；但打印执行底座（P1）尚未开发，本插件当前无法在真实打印流程中被触发执行。契约以宿主仓 docs/sheet/future/14-printing/ 设计文档为准。

参考实现：[`kimpo-print-extension/print-demo-layout-override/`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-layout-override/)

---

## 一、打印效果

一张 **A5 横向**的小纸：不管模板保存的打印设置是 A4 竖向还是别的什么，选中本方案后**静默切换**为 A5 横向 + 窄边距，且**只打模板里指定的"回执"叶子分区**——单据主体、明细表、审批记录等其他分区一概不出现。回执分区的所有列被缩放到一页宽，不会横向裂页。

页脚右下角固定落一行 9pt 小字：

```
打印人：张三 2026-07-15
```

打印人取当前操作者姓名，日期按其时区渲染。整个过程用户只是点了下"打印"，纸张切换、区域裁剪、页脚落款全部自动完成。

## 二、适用场景

- **业务回执**：收款回执、领用回执、签收联——单据本体是一大张 A4 表单，但交给对方的只是其中一小块"回执区"，用半张纸（A5）打更合适；
- **局部打印**：只打审批意见区、只打条码区等"从大模板里抠一块打"的需求；
- **落款追责**：回执类纸面必须能追溯"谁在什么时候打的"，页脚自动落打印人+日期，不依赖用户手写。

任何"版式与模板保存值不同 + 只打某个分区"的需求，都可以照本例改 `target` 和 `pageSetupOverride` 做出来。

## 三、插件包结构

版式覆盖是 **L1 / A 档（纯声明式）** 能力——本质上是把填报态"临时调整打印属性不落库"的既有机制正式化为契约入口。插件包只有两个 JSON 文件：

```
print-demo-layout-override/
├── manifest.json          # 插件清单：声明一个 print-mode 能力
└── specs/
    └── receipt-a5.json    # 静态 PrintJobSpec 模板（打哪个叶子、版式怎么覆盖）
```

## 四、manifest 逐段讲解

完整文件见 [`kimpo-print-extension/print-demo-layout-override/manifest.json`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-layout-override/manifest.json)。顶层是标准插件清单字段（`plugin_id` / `name` / `version` / `category` / `description` / `author`）；打印模式是 `capabilities.print_modes[]` 里的一项能力声明（2026-07-15 实施修订：宿主 `Manifest.Capabilities` 是对象，print-mode 落在 `print_modes` 数组字段，不是旧稿 `capabilities: [{type:"print-mode"}]` 的数组形；字段全 snake_case），**任何类别插件均可携带**，宿主按能力分发、随会话下发给派蒙（kimposheet，editor 插件）。

`print_modes[0]` 各字段：

| 字段 | 本例取值 | 说明 |
|---|---|---|
| `id` | `"receipt-a5"` | 模式 ID，实例内唯一；模板绑定方案存的引用 = `pluginId + modeId + params` |
| `name` | 中英双语对象 | "打印方案"下拉里的显示名 |
| `modes` | `["design-preview", "form-preview", "form-print", "action"]` | 四个场景全声明。**特别声明了 `action`**：回执最自然的触发方式是交互按钮上的"打印"动作节点（"点击 → 用回执方案打印当前单据"），动作上下文还可带行级目标 |
| `spec_source` | `"static"` | A 档：静态 spec 模板，零前端代码。版式覆盖是固定规则，无需 B 档 `buildSpec` hook |
| `spec` | `"specs/receipt-a5.json"` | 包内静态 spec 模板路径 |

本例没有声明 `params`——回执的纸张、边距、目标分区都是方案本身的固定语义，没有需要设计者调的旋钮。需要设计者可配时（例如让页脚文字可改），照水印教程的 `params` 写法加即可。

## 五、spec 逐段讲解

完整文件见 [`kimpo-print-extension/print-demo-layout-override/specs/receipt-a5.json`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-layout-override/specs/receipt-a5.json)：

```json
{
  "version": 1,
  "target": { "scope": "leaf", "leafId": null },
  "pageSetupOverride": {
    "paper": "A5",
    "orientation": "landscape",
    "marginsPreset": "narrow",
    "fitAllColumnsOnePage": true,
    "headerFooter": {
      "footer": {
        "right": { "kind": "text", "text": "打印人：{{user.name}} &D", "fontSizePt": 9 }
      }
    }
  },
  "output": { "kind": "print" }
}
```

逐字段说明：

- `version: 1` —— 契约版本号，供派蒙侧 JSON Schema 校验。
- `target.scope: "leaf"` —— 局部打印的核心：打印目标从默认的 `activeContainer`（整个容器）收窄为**单个叶子分区**。四种取值：`activeContainer`（当前容器，默认）/ `leaf`（指定叶子）/ `selection`（选定区域）/ `printArea`（模板设定的打印区域）。
- `target.leafId: null` —— 要打的叶子分区 ID。**`null`（2026-07-15 拍板语义）指"当前活动叶子"**——演示插件因此无需配真实叶子 ID 即可跑通；真实业务插件若要固定打某个具名分区（而不是"当前选中哪个打哪个"），把这里换成叶子结构清单里的真实 leafId 即可（清单可从 PrintContext 的模板结构信息里拿到）。
- `pageSetupOverride` —— 与派蒙 SheetPageSetup **同构的增量覆盖**：只写你要改的字段，**没写（null）的字段一律沿用模板保存值**——所以本方案不会影响模板原有的打印区域、居中等其他设置：
  - `paper: "A5"` + `orientation: "landscape"` —— 回执用半张 A4 的横向小纸，静默切换、不落库、不弄脏模板设置；
  - `marginsPreset: "narrow"` —— 窄边距预设，小纸面尽量留给内容；
  - `fitAllColumnsOnePage: true` —— 回执区所有列缩到一页宽，A5 纸窄，防止横向裂页；
  - `headerFooter` —— 与派蒙页眉页脚结构同构（header/footer 各含 left/center/right 三段，每段 `kind` 为 `text` 或 `image` 二选一）。本例只覆盖**页脚右段**：`text` 里混用两套变量——`{{user.name}}` 取当前打印人姓名（受限表达式，`user.*` 命名空间），`&D` 是页眉页脚内置日期变量；`fontSizePt: 9` 小字号落款。
- **没有 `layers`、没有 `pageTemplate`** —— 本方案纯 L1：不叠图层、不做框式编排，就是"换版式 + 抠区域"。
- `output.kind: "print"` —— 回执直接唤起浏览器打印。要存档时可改 `"pdf"`（下载，`fileName` 支持变量）或 `"pdfBlob"`（成品回递插件）。

> 💡 变量表达式只有两套合法写法：`&P/&N/&D/&T/&F/&U`（页码/总页数/日期/时间/模板名/制表人）与 `{{...}}` 受限表达式（命名空间 `record.*` / `user.*` / `template.*` / `item.*` / `params.*`）。`record.<code>`/`item.<code>` 默认显示值，`.raw` 取原始标量供比较。表达式在派蒙侧求值，A 档插件不执行任何代码。

## 六、设计态调用

1. 设计者在派蒙工具栏点**打印** → 打开打印预览对话框；
2. 设置区顶部**「打印方案」下拉**选中「打印回执（A5 横向）」；
3. 派蒙读取包内静态 spec，按方案渲染：**右侧预览立即变成 A5 横向小页**，只显示回执分区，页脚带落款（设计态无真实单据，`user.name` 等以当前设计者/样例数据渲染）；
4. 本模式没有 `params`，设置区无额外参数项；
5. 确认效果后，把方案**绑定为该模板的默认打印方案**（可与其他方案并存，如"整单打印"+"打印回执"绑两个），随页面设置保存——绑定只存方案引用，不内联 spec。

## 七、填报态调用

复用既有**三入口**，不新增入口：

1. **打印按钮** / **Ctrl+P** / 宿主或插件经 SDK 调 **`openPrintPreview()`**；
2. 派蒙查模板绑定方案：
   - 只绑了回执方案且策略为直打 → quickPrint 路径：读 spec → 用真实单据上下文求值（打印人 = 当前登录用户，日期按其时区）→ 渲染 A5 回执 → 直接唤起浏览器打印，**全程不弹版式设置**；
   - 绑了多方案 → 轻量方案选择或带方案切换的预览对话框；
3. **动作入口（本方案声明了 `action`）**：设计者可在模板上放一个"打印回执"交互按钮，绑"打印"动作节点并选中本方案——业务用户点按钮即出回执，这是回执类打印最顺手的触发方式。

## 八、权限与边界

- **权限统一执法，插件模式不可绕过**：一切打印统一过**卡侬（赋权层）Print 位 AND 模板打印策略三项**（RuntimeAllowPrint / RuntimeAllowPreview / RuntimeAllowAdjust）。注意：`pageSetupOverride` 的静默版式切换同样受"可调整打印属性"策略约束——策略不放行时方案不能改版式；PDF 出口另过 ExportPdf 位（三位独立）。
- **字段可见性与表单会话同口径**：spec 表达式能取到的 `record.*` 字段值都已经过卡侬裁决，用户表单里不可见的字段打印链路同样拿不到——防止局部打印变成越权导出通道。
- **业务数据一律经 Mirror**：字段值由宿主从表单数据权威层 Mirror（代号艾莉亚）取值下发；插件不接触 editor 内部状态，`leafId` 也只是结构 ID，不涉及任何画布几何。
- **时区**：`&D` 日期与一切日期变量按当前用户时区换算，遵守时区单一换算点约束。
- `pageSetupOverride` 是**会话内临时覆盖**，不写回模板的打印设置；一期不提供跨单据/跨模板取数。
