# 水印打印 · 按单据状态动态水印（A 档纯声明式）

> ⚠️ 底座实施中：manifest/spec 已按 2026-07-15 契约拍板定稿、插件可被宿主真实加载；但打印执行底座（P1）尚未开发，本插件当前无法在真实打印流程中被触发执行。契约以宿主仓 docs/sheet/future/14-printing/ 设计文档为准。

参考实现：[`kimpo-print-extension/print-demo-watermark/`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-watermark/)

---

## 一、打印效果

打印出来的每一页，都是**原样的单据内容**（版式、纸张、边距全部沿用模板保存的打印设置），只是在整页内容之上斜向叠了一层浅灰色大字水印：

- 单据状态是**已作废**（`status.raw == 'void'`）→ 每页斜印「**已作废**」；
- 其他状态（草稿等）→ 每页斜印「**草稿**」（文字可由设计者在参数里改）。

水印逆时针倾斜 30°、48pt 大字、15% 不透明度——盖在内容上但不遮内容，复印/拍照后依然可辨。设计态预览对话框里选中本方案后，右侧预览**所见即所得**地显示水印效果。

## 二、适用场景

- **防误用**：单据还没走完审批就被打印出来传阅，纸面必须一眼看出"这是草稿，不作数"；
- **作废标识**：已作废单据偶尔仍需打印留档（对账、审计），必须显著标记避免被当有效凭证；
- **状态跟着数据走**：同一个打印方案，水印文字由单据的真实状态字段决定，不需要设计者为每种状态各配一个方案。

任何"整页叠一层文字、其余排版不动"的需求（如公司名底纹、「机密」标记）都可以照本例改 `layers` 做出来。

## 三、插件包结构

水印打印是 **L1 / A 档（纯声明式）** 能力：插件包里只有两个 JSON 文件，零前端代码、零后端代码。

```
print-demo-watermark/
├── manifest.json          # 插件清单：声明一个 print-mode 能力
└── specs/
    └── watermark.json     # 静态 PrintJobSpec 模板（打什么、怎么叠水印）
```

## 四、manifest 逐段讲解

完整文件见 [`kimpo-print-extension/print-demo-watermark/manifest.json`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-watermark/manifest.json)。顶层是标准插件清单字段（`plugin_id` / `name` / `version` / `category` / `description` / `author`），打印模式插件**不新增插件类别**——任何类别的插件都可以在 `capabilities.print_modes[]` 里携带打印模式能力，宿主按能力（而非类别）分发。

> **2026-07-15 实施修订**：宿主 `Manifest.Capabilities` 是对象（不是数组），print-mode 能力因此落在 `capabilities.print_modes` 数组字段下，不再是旧稿里 `capabilities: [{ type: "print-mode", ... }]` 的数组形（宿主解析不了那种形状）。字段全部 snake_case，且不再有 `type` 字段——`print_modes` 这个字段名本身就是类型标记。

重点看 `capabilities.print_modes[0]` 声明：

```jsonc
{
  "capabilities": {
    "print_modes": [
      {
        "id": "status-watermark",
        "name": { "zh_CN": "单据状态水印", "en_US": "Record Status Watermark" },
        "modes": ["design-preview", "form-preview", "form-print"],
        "spec_source": "static",
        "spec": "specs/watermark.json",
        "params": [
          { "key": "watermarkText", "type": "string", "default": "草稿",
            "name": { "zh_CN": "草稿水印文字", "en_US": "Draft watermark text" } }
        ]
      }
    ]
  }
}
```

| 字段 | 本例取值 | 说明 |
|---|---|---|
| `id` | `"status-watermark"` | 模式 ID，**实例内唯一**；模板绑定方案时存的引用就是 `pluginId + modeId + params` |
| `name` | 中英双语对象 | 显示在打印预览对话框"打印方案"下拉里的名字 |
| `modes` | `["design-preview", "form-preview", "form-print"]` | 适用场景：设计态预览、填报态预览、填报态直打。水印不需要动作节点触发，所以没声明 `action` |
| `spec_source` | `"static"` | **A 档**：spec 是包内静态 JSON 模板 + 变量表达式，零代码。逻辑复杂时才用 B 档（`"frontend"` + `entry` 指向前端 UMD 导出的 `buildSpec` 函数名） |
| `spec` | `"specs/watermark.json"` | A 档静态 spec 模板在包内的相对路径；宿主注册插件时（dev 扫描 / `.kpp` 安装）读取该文件**内联进能力声明**，会话打开时随 `meta.printModes[]` 一并下发，不单独开静态托管白名单 |
| `params` | 一个 `watermarkText` 参数 | 设计者可配参数，会渲染进预览对话框的设置面板（见下） |

`params` 数组里每项声明一个设计者可改的参数：

```json
{ "key": "watermarkText", "type": "string", "default": "草稿",
  "name": { "zh_CN": "草稿水印文字", "en_US": "Draft watermark text" } }
```

设计者在预览对话框里把它改成「样稿」「DRAFT」等并随方案绑定保存。**参数注入语法（2026-07-15 拍板）**：spec 模板内用 `{{params.<key>}}` 引用（也可作为三目表达式的分支值，如本例 `params.watermarkText`），求值与 `{{record.*}}` 等同一套受限表达式管线；参数值来源 = 模板绑定方案时保存的 params 快照，未配置时取 manifest 声明的 `default`。

## 五、spec 逐段讲解

完整文件见 [`kimpo-print-extension/print-demo-watermark/specs/watermark.json`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-watermark/specs/watermark.json)。spec 就是一份 **PrintJobSpec**——你声明"打什么、怎么排版"，派蒙（kimposheet，editor 插件）用自己的分页/合成管线执行，渲染器对插件零暴露。

```json
{
  "version": 1,
  "target": { "scope": "activeContainer", "leafId": null },
  "layers": [
    { "type": "watermark",
      "text": "{{record.status.raw == 'void' ? '已作废' : params.watermarkText}}",
      "angle": -30, "opacity": 0.15, "font": { "size": 48 } }
  ],
  "output": { "kind": "print" }
}
```

逐字段说明：

- `version: 1` —— 契约版本号，供派蒙侧 JSON Schema 校验。
- `target.scope: "activeContainer"` —— 打**当前容器**（容器内全部分区按布局顺序组成打印流），这是填报态打印单据的主场景默认粒度；水印要盖全单，所以不指定叶子，`leafId` 置 `null`。
- **没有 `pageSetupOverride`** —— 这是本例的关键取舍：水印方案**不动版式**，纸张/方向/边距/缩放全部沿用模板保存的打印设置，设计者原来怎么配打印就怎么出。
- `layers` —— L1 全页图层数组，按 z 序叠加在渲染内容之上：
  - `type: "watermark"` —— 水印图层（另一种是 `background` 预印底图，用于套打）；
  - `text` —— 水印文字，用受限表达式 `{{...}}` 动态取值：`record.status.raw` 来自 **Mirror（表单数据权威层，代号艾莉亚）** 下发的单据数据（`.raw` 取原始标量做比较，与展示用的 `record.status` 显示值口径不同），已作废印「已作废」、否则印**设计者配置的 `params.watermarkText`**（默认「草稿」）。**表达式求值在派蒙侧执行**，A 档插件不运行任何代码——这是安全默认；
  - `angle: -30` —— 逆时针斜 30°，经典水印角度；
  - `opacity: 0.15` —— 15% 不透明度，保证盖章感但不影响读内容；
  - `font.size: 48` —— 48pt 大字，A4 整页视觉上约两拳宽，复印后仍清晰。
- `output.kind: "print"` —— 出口走浏览器打印。改成 `"pdf"` 则下载 PDF（可配 `fileName`，同样支持 `{{template.name}}` 等变量）；`"pdfBlob"` 则把成品 PrintArtifact 回递给插件（归档、上传云打印等）。

> 💡 变量表达式只允许两套：`&P/&N/&D/&T/&F/&U`（页码/总页数/日期/时间/模板名/制表人，沿用页眉页脚变量集）与 `{{...}}` 受限表达式（命名空间 `record.*` / `user.*` / `template.*` / `item.*` / `params.*`）。`record.<code>`/`item.<code>` 默认取显示值，追加 `.raw` 取原始标量供比较用。受限文法 = 点路径取值 + 字符串字面量 + `==`/`!=` 比较 + 三目，没有任意 JS——求值器是受限表达式引擎。

## 六、设计态调用

1. 设计者在派蒙工具栏点**打印** → 打开打印预览对话框；
2. 设置区顶部的**「打印方案」下拉**里，除内建标准方案外，会列出所有已注册的 print-mode——选中「单据状态水印」；
3. 本模式声明的 `params` 渲染进设置区：设计者可把「草稿水印文字」改成自己想要的默认文字；
4. **右侧预览所见即所得**：设计态没有真实单据，宿主会按字段类型生成样例数据填充 `record.*`，水印按样例状态渲染出来；
5. 满意后，设计者可把方案（含参数值）**绑定为该模板的默认打印方案**（一个模板可绑多个），随页面设置一起保存。绑定只存引用（`pluginId + modeId + params`），不内联 spec 全文。

## 七、填报态调用

填报态**复用既有三入口，不新增入口**：

1. 用户点表单上的**打印按钮**，或按 **Ctrl+P**，或宿主/其他插件经 SDK 调 **`openPrintPreview()`**；
2. 派蒙查该模板绑定的打印方案：
   - 只绑了本方案且策略为直打 → 走既有 quickPrint 路径：读静态 spec → 用**真实单据上下文**（Mirror 权威字段值）求值水印表达式 → 渲染 → 直接唤起浏览器打印；
   - 绑了多个方案 → 弹轻量方案选择或带方案切换的预览对话框，用户选「单据状态水印」后同上。

用户全程无感知"插件"存在——打出来的纸上，作废单自动带「已作废」斜印。

## 八、权限与边界

- **权限统一执法，插件不可绕过**：一切打印模式统一过**卡侬（赋权层）Print 位 AND 模板打印策略三项**（RuntimeAllowPrint / RuntimeAllowPreview / RuntimeAllowAdjust）；`output.kind` 为 PDF 出口时另过 ExportPdf 位。执法在派蒙侧，插件在 PrintContext 里只拿到 `canPrint/canPreview/...` 解算结果做展示用。
- **字段可见性与表单会话同口径**：`record.*` 里能取到的字段值都已经过卡侬裁决——用户在表单里看不见的字段，水印表达式也取不到。否则打印就成了越权导出通道。
- **业务数据一律经 Mirror**：`record.status` 由宿主从 Mirror（艾莉亚）权威层取值下发，插件不接触 editor 内部状态（canvas/渲染器/选区一概不暴露）。
- **日期变量按用户时区换算**（如 `&D`），遵守平台时区单一换算点约束。
- 一期不提供跨单据/跨模板取数；批量打印属 P3 单独设计。
