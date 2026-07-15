# 标签 N 联（网格循环打印）· 开发教程

> ⚠️ 底座实施中：manifest/spec 已按 2026-07-15 契约拍板定稿、插件可被宿主真实加载；但打印执行底座（P1/P2）尚未开发，本插件当前无法在真实打印流程中被触发执行。契约以宿主仓 docs/sheet/future/14-printing/ 设计文档为准。

参考实现：[`kimpo-print-extension/print-demo-labels/`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-labels/)（A4 纸 2×5 商品标签）。

---

## 一、打印效果

一张 A4 纸被划成 **2 列 × 5 行 = 10 个标签格**，格与格之间留 2mm 间隙（对应市售背胶标签纸的模切缝）。每个标签格里：

- 左侧一枚二维码，内容是该商品的 SKU 编码，仓库扫码枪直接识别；
- 右侧上下两行：**品名**（大字）和"规格 / 单位"（小字）；
- 格子下半部一行 SKU 明码文本（扫码失败时人眼兜底）+ 一行打印日期小字。

数据来自当前单据的**明细表**：明细里有几行商品，就打几个标签。8 行商品 → 打 8 格、后 2 格留空；23 行商品 → 自动排满第一页 10 格后**续第二、三页**，共 3 页。

## 二、适用场景

一切"一条数据一张贴纸"的批量小票式输出：

- 入库单打商品价签/货架标签（本例）；
- 资产盘点打资产标签；
- 样品送检打样品条码贴；
- 邮寄名单打地址贴纸。

共同特征：版面 = 固定网格 + 每格同构、只换数据。A 档（静态 spec + grid repeat）零代码即可覆盖。

## 三、插件包结构

```
print-demo-labels/
├── manifest.json           # 插件声明 + print-mode 能力
└── specs/
    └── labels-2x5.json     # 2×5 标签网格的 PrintJobSpec 模板（A 档）
```

同套打篇一样，A 档插件没有任何代码文件；标签场景连底图资产都不需要。

## 四、manifest 逐段讲解

```json
"capabilities": {
  "print_modes": [
    {
      "id": "labels-2x5",
      "name": { "zh_CN": "A4 标签 2×5", "en_US": "A4 Labels 2x5" },
      "modes": ["design-preview", "form-preview", "form-print"],
      "spec_source": "static",
      "spec": "specs/labels-2x5.json"
    }
  ]
}
```

> **2026-07-15 实施修订**：`print_modes` 是 `capabilities` 对象下的数组字段（宿主 `Manifest.Capabilities` 本身是对象，不能承载旧稿 `capabilities: [{type:"print-mode"}]` 那种数组形声明），字段全 snake_case。

| 字段 | 说明 |
|---|---|
| `id` | 实例内唯一。想同时提供 2×5 / 3×8 两种规格？在同一插件里声明**两个** `print_modes` 条目，各配一份 spec，下拉里出现两个方案 |
| `modes` | 本例开了设计态预览 + 填报态预览 + 填报态直打；没开 `action`（标签一般从打印入口走，需要按钮触发可自行加上） |
| `spec_source` / `spec` | A 档：指向包内静态 spec 模板 |

`params`（可选）本例未配；如果想让设计者在设置面板里调"起始格偏移"（半张用过的标签纸从第 N 格开始打）这类参数，在这里声明即可。

## 五、spec 逐段讲解

完整文件见 [`specs/labels-2x5.json`](https://github.com/somsne/kimpo-print-extension/tree/main/print-demo-labels/specs/labels-2x5.json)。

### 5.1 页面与网格几何

```json
"pageTemplate": {
  "paper": "A4", "orientation": "portrait",
  "margins": { "t": 10, "b": 10, "l": 8, "r": 8 },
  ...
  "repeat": {
    "kind": "grid",
    "cols": 2, "rows": 5,
    "gap": { "x": 2, "y": 2 },
    "dataSource": "detailRows.items",
    "frameSetPerItem": ["label-barcode", "label-name", "label-spec", "label-sku-text", "label-footer"]
  }
}
```

- 边距按标签纸实物量取：A4（210×297mm）去掉边距后可用区 194×277mm；
- `kind: "grid"`：N-up 标签网格。可用区被 `cols × rows` 均分：每格宽 = (194 − 2) / 2 = **96mm**，每格高 = (277 − 2×4) / 5 = **53.8mm**——买标签纸时认准和这个格子尺寸匹配的模切规格，或反过来按纸改 margins/gap；
- `gap`：格间隙（mm），对应模切缝，避免内容跨缝被裁。

### 5.2 repeat 迭代语义（本篇核心）

- `dataSource: "detailRows.items"`：数据源指向 PrintContext 里的明细表行集合——`detailRows[tableKey]`，`items` 是本模板明细表的 tableKey，按你的模板实际 key 填。**明细行数据的权威来源是表单数据权威层 Mirror**，经字段可见性裁决后下发。
- **迭代规则**：编辑器拿到 N 行明细，逐行"盖章"——第 1 行盖进第 1 格（左上），第 2 行盖第 2 格（右上），第 3 行盖第二排左格……**先行后列、从左到右、从上到下**填满 `cols × rows` 个格子；
- **自动续页**：明细行数 > 10（一页格数）时自动开新页继续，直到行打完。23 行 = 3 页，最后一页剩 7 格空白。你**不需要**在 spec 里写任何分页逻辑；
- `frameSetPerItem`：声明哪些 frame 属于"每格重复"的一组。列出的 frame 会随每行数据在每个格子里重新排一遍；**它们的 `rect` 坐标以所在格子的左上角为原点**（不是纸的左上角）。本例五个 frame 全在组里，所以格内坐标都落在 96×53.8mm 之内。不在组里的 frame（本例没有）按页坐标只摆一次，适合放页级标题/页码。

### 5.3 格内内容框

```json
{ "id": "label-barcode",
  "rect": { "x": 4, "y": 4, "w": 36, "h": 20 },
  "content": { "type": "barcode", "format": "qrcode", "expr": "{{item.sku}}" } }
```

- **`{{item.*}}` 命名空间是 repeat 专用的**：迭代到哪一行，`item` 就是哪一行的字段集合。`{{item.sku}}`、`{{item.name}}`、`{{item.spec}}`、`{{item.unit}}` 都是明细表字段 key，按你的模板实际字段名替换；
- 品名/规格是普通 `text` 框，表达式里可以混排静态文字：`"规格：{{item.spec}} / 单位：{{item.unit}}"`；
- footer 框用了页眉页脚变量 `&D &T`（打印日期时间，按用户时区渲染）——`&` 系变量和 `{{}}` 表达式可混用；
- `{{template.name}}` 取模板名，`{{record.*}}`（单据级字段，如单号）在格内同样可用——每格都渲染同一个值。`record.<code>` / `item.<code>` 默认取显示值；若要做条件判断（如按品类切标签样式），追加 `.raw` 取原始标量比较，如 `{{item.category.raw == 'fragile' ? '易碎' : ''}}`。

### 5.4 页数保护

repeat 的数据源可能很大（千行明细 = 百余页）。编辑器沿用现有页数保护：**预计页数 > 200 弹确认框、> 1000 直接拒绝**。插件无需自己做限制，但业务上应引导用户先筛选明细再打。

### 5.5 出口

```json
"output": { "kind": "print", "fileName": "{{template.name}}_labels.pdf" }
```

标签直接出纸用 `print`；要留档改 `pdf`（下载）或 `pdfBlob`（回递给插件自行上传归档）。

## 六、设计态调用

1. 模板设计器点工具栏"打印" → 打印预览对话框 → "打印方案"下拉选 **"A4 标签 2×5"**；
2. 设计态没有真实单据，编辑器按明细字段类型生成**样例行**填进网格 → 右侧预览直接看到 2×5 的标签网格排版（所见即所得）：格子尺寸、条码位置、字号是否合适一目了然；
3. 不满意就改 spec 的 margins/gap/格内 rect 再预览，直到和手上的标签纸实物对上；
4. 绑定为模板默认打印方案保存（存方案引用，不内联 spec）。

## 七、填报态调用

填报态**三个既有入口**（打印按钮 / Ctrl+P / 宿主 `openPrintPreview()`）统一按绑定方案走：

- 单方案 + 直打策略：入库员保存入库单后点打印，编辑器取当前单据的明细行喂给 grid repeat，渲染后直接唤起浏览器打印——放好标签纸出贴纸；
- 模板绑了多个方案（比如"整单 A4 报表"和"标签 2×5"并存）：弹轻量方案选择，或进预览对话框切换方案后再打。

打之前在预览里扫一眼页数和末页空格数，再决定放几张标签纸。

## 八、权限与边界

- **统一过卡侬（赋权层）Print 位 AND 模板策略三项**（RuntimeAllowPrint / Preview / PaperOverride），插件模式不可绕过；`pdf` 出口另过 ExportPdf 位，三位独立。
- **明细行字段可见性与表单会话同口径**：当前用户看不到"成本价"列，`{{item.costPrice}}` 就取不到值——标签打印不会成为越权导出通道。行级数据一律经 Mirror 取。
- 一期 repeat 数据源**只限当前单据的明细表**，不支持跨单据聚合打标签（多单合并属 P3 批量场景）。
- 页数保护（>200 confirm / >1000 拒）是编辑器侧硬闸，插件与用户都不可关闭。
