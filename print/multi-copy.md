# 打印扩展教程：一页多单 / 多联（copies）

> ⚠️ 底座实施中：manifest/spec 已按 2026-07-15 契约拍板定稿、插件可被宿主真实加载；但打印执行底座（P2）尚未开发，本插件当前无法在真实打印流程中被触发执行。契约以宿主仓 docs/sheet/future/14-printing/ 设计文档为准。

本篇讲**一页多联**打印模式：在同一张纸上把同一份单据打两联（或 N 联），各联内容同源、仅联名等框内文字不同。参考实现：`kimpo-print-extension/print-demo-multi-copy/`（**A 档，零前端代码**）。

---

## 一、打印效果

以"领料单一页两联"为例，A4 纵向一页从上到下依次是：

1. **上半页 = 存根联**：一行联名标题（"领料单 · 存根联 单号：xxx 打印：日期 时间"），下面是派蒙渲染出的完整领料单表格；
2. **中缝 = 裁切提示线**：一行 `✂ - - - 沿此线裁开 - - -` 的虚线文字，指示裁刀位置；
3. **下半页 = 客户联**：与存根联**完全相同的表格内容**，联名标题换成"领料单 · 客户联"。

两联的表格是**同一个内容片段**（同源），单号、打印时间等变量在两联上取值一致；唯一的差异是各联自己的联名 text 框。打出来撕开，一联留存、一联随货。

## 二、适用场景

- 领料单 / 出库单 / 收据：存根联留仓库，客户联随货交接；
- 小票两联、押金单两联：一联入账、一联给客户；
- 三联单（如"存根 / 客户 / 财务"）：同样的手法，把 frame 组从两组扩到三组、`repeat.rows` 改 3 即可；
- 任何"同一单据、多份同源版面、仅少量文字有别"的票据类打印。

与它相邻但**不是**本模式的场景：每格绑定**不同**明细行的标签网格（那是 `repeat.kind: "grid"`，见标签 N 联篇）；打多张**不同**单据（那是 P3 批量，本期不做）。

## 三、插件包结构

```
print-demo-multi-copy/
├── manifest.json            # 插件声明 + print-mode 能力（A 档）
└── specs/
    └── two-copies.json      # 静态 PrintJobSpec 模板（本模式的全部"逻辑"）
```

A 档就这两个文件：**没有任何代码**。排版、变量、联数全部由一份纯 JSON 声明，派蒙负责执行。

## 四、manifest 逐段讲解

```json
{
  "plugin_id": "com.kimpo.demo.print-multi-copy",
  "name": { "zh_CN": "打印演示：一页多联", "en_US": "Print Demo: Multi-Copy" },
  "version": "0.1.0",
  "category": "action",
  "description": { "zh_CN": "……", "en_US": "……" },
  "author": { "name": "Kimpo Team" },
  "capabilities": {
    "print_modes": [
      {
        "id": "two-copies",
        "name": { "zh_CN": "领料单一页两联", "en_US": "Two-Copy Requisition" },
        "modes": ["design-preview", "form-preview", "form-print", "action"],
        "spec_source": "static",
        "spec": "specs/two-copies.json"
      }
    ]
  }
}
```

> **2026-07-15 实施修订**：`author` 是对象 `{ name, email?, website? }`，不是裸字符串（宿主 `AuthorInfo` 结构如此，早期草稿的字符串写法会导致 manifest 解码失败）；print-mode 能力挂在 `capabilities.print_modes[]` 数组下，不是旧稿的 `capabilities: [{type:"print-mode"}]` 数组形（宿主 `Manifest.Capabilities` 本身是对象，承载不了那种形状）。

- 顶层字段（`plugin_id` / `name` / `version` / `category` / `description` / `author`）与普通插件完全一致。**任何类别**的插件都可以携带 print-mode 能力，不需要新插件类别——这里挂在 `category: "action"` 下只是因为演示插件没有别的方向。
- `capabilities.print_modes[]`：声明"我贡献一种打印模式"（数组可放多条）。宿主按 **capabilities**（而非 category）收编进能力注册表，随会话下发给派蒙。
- `id`：模式在实例内的唯一标识，模板绑定方案时引用它（存的是 `pluginId + modeId + params` 的引用，不内联 spec 全文）。
- `name`：显示在打印预览对话框"打印方案"下拉里的名字，多语言。
- `modes`：本模式适用的触发场景。四个取值：`design-preview`（设计态预览）、`form-preview`（填报态预览）、`form-print`（填报态直打）、`action`（动作节点触发）。多联单四态都有意义，全声明。
- `spec_source: "static"`：**A 档**——spec 来自包内静态 JSON 模板，插件零代码。与之相对的 `"frontend"` 是 B 档（前端函数动态拼 spec，见混排文档篇）。
- `spec`：A 档专用，指向包内 spec 模板的相对路径；宿主注册插件时读取该文件内联进能力声明，随会话 `meta.printModes[]` 一并下发。

## 五、spec 逐段讲解

完整文件见 `kimpo-print-extension/print-demo-multi-copy/specs/two-copies.json`，逐段拆开：

### 5.1 骨架

```json
{
  "version": 1,
  "target": { "scope": "activeContainer", "leafId": null },
  "pageTemplate": { … },
  "output": { "kind": "print" }
}
```

- `version: 1`：契约版本号，固定写 1。
- `target`：打什么。`"activeContainer"` = 当前激活的表单容器；本模式下真正的内容来源写在各 frame 的 `sheetFragment` 里，这里保持默认即可。
- `output.kind: "print"`：出口是浏览器打印（多联票据通常直打）。也可以换 `"pdf"` 下载留档。
- 一旦声明了 `pageTemplate`，就进入 **L2 框式页面编排**：页面布局由你摆的 frame 接管，不再是派蒙默认的顺序分页。

### 5.2 pageTemplate：纸张与内容框

```json
"pageTemplate": {
  "paper": "A4",
  "orientation": "portrait",
  "margins": { "t": 10, "b": 10, "l": 10, "r": 10 },
  "frames": [ … ]
}
```

纸张 A4 纵向、四边 10mm 边距，得到 190 × 277 mm 的内容区。**frame 的 `rect` 单位是 mm**，坐标相对内容区左上角。

### 5.3 frames：上下两组 + 一条裁切线

五个 frame 分三部分——

**存根联（上半页）**：

```json
{ "id": "stub-label", "rect": { "x": 0, "y": 0, "w": 190, "h": 8 },
  "content": { "type": "text",
    "expr": "领料单 · 存根联　单号：{{record.bill_no}}　打印：&D &T",
    "style": { "size": 10, "align": "center" } } },
{ "id": "stub-body", "rect": { "x": 0, "y": 10, "w": 190, "h": 118 },
  "content": { "type": "sheetFragment", "scope": "activeContainer",
    "leafId": null, "fit": "width" } }
```

- `text` 框：联名文字 + 变量。`{{record.bill_no}}` 从 Mirror（艾莉亚）权威层取"单号"字段值（默认显示值口径；若要做条件判断需追加 `.raw` 取原始标量比较）；`&D` / `&T` 是打印日期/时间（沿用页眉页脚变量集，按**当前用户时区**渲染）。
- `sheetFragment` 框：让派蒙把表单内容渲染成一条**定宽、不分页的内容条带**摆进框里。`fit: "width"` 表示按框宽缩放。**这就是"同源"的关键**：两联的 fragment 声明完全相同，指向同一个内容来源。

**裁切线（中缝）**：

```json
{ "id": "cut-line", "rect": { "x": 0, "y": 132, "w": 190, "h": 6 },
  "content": { "type": "text",
    "expr": "✂ - - - - - - - - - - - - - -  沿此线裁开  - - - - - - - - - - - - - -",
    "style": { "size": 8, "align": "center" } } }
```

契约没有专门的"裁切线"内容类型，**用一个 text 框画虚线提示**就够了：剪刀符号 + 连字符横贯页宽，落在两联之间的空档（y=132~138）。要更克制的话，把文字换成纯 `- - -` 即可。

**客户联（下半页）**：`customer-label` + `customer-body`，与存根联**逐字段相同**，只有两处不同——`rect.y` 平移到下半页（141 / 151），联名文字换成"客户联"。这就是"同源内容、框内文字不同"的全部配置法：**内容框复制、文字框改词**。

### 5.4 repeat：声明 copies 语义

```json
"repeat": { "kind": "copies", "rows": 2, "cols": 1, "gap": { "x": 0, "y": 5 } }
```

`repeat.kind: "copies"` 向派蒙声明"这是**同一单据的多联**"：纵向 2 联（`rows: 2, cols: 1`）、联间留 5mm 空档（`gap.y`，即裁切线所在的中缝）。与 `"grid"`（每格绑一行明细）和 `"records"`（明细行循环）不同，copies **不绑定 `dataSource`**——每联的数据都是整张单据本身，派蒙可据此把同源 fragment 只渲染一次、多处复用。联与联之间的版面差异（联名文字）由上面显式摆放的 frame 表达。

## 六、设计态调用

1. 插件安装后，宿主把本 print-mode 收进能力注册表并随会话下发；
2. 设计者打开模板设计器，点工具栏"打印"进入**打印预览对话框**；
3. 设置区顶部的**"打印方案"下拉**里，除内建标准方案外会出现"领料单一页两联"——选中它；
4. 派蒙读取包内 `specs/two-copies.json`，用**样例数据**（设计态没有真实单据，按字段类型生成）填充变量，按 spec 渲染，**右侧预览所见即所得**：上下两联、中缝裁切线直接可见；
5. 满意后，把本方案**绑定为该模板的默认打印方案**（可绑多个），随页面设置一起保存。绑定只存方案引用（`pluginId + modeId + params`），不内联 spec。

## 七、填报态调用

填报态**复用既有三入口，不新增入口**：

1. 表单里点"打印"按钮 / 按 Ctrl+P / 宿主调 `openPrintPreview()`；
2. 派蒙查该模板绑定的方案：**单方案 + 直打策略** → 走 quickPrint 路径，按真实单据数据渲染两联后直接唤起浏览器打印；**多方案** → 弹轻量方案选择（或预览对话框带方案切换）；
3. **动作入口**：把"打印"配成动作节点（如交互按钮"点击 → 用一页两联方案打印当前单据"），出库确认后一键打联最顺手。

变量此时取真实值：`{{record.bill_no}}` 是这张单据的真实单号（经 Mirror 取值），`&D &T` 按操作用户的时区渲染。

## 八、权限与边界

- **打印权限**：一切插件打印模式统一过 **卡侬 Print 位 AND 模板策略三项**（RuntimeAllowPrint / Preview / PaperOverride），插件模式不可绕过、也无法放宽；
- **PDF 出口独立**：若把 `output.kind` 改成 `"pdf"`，还要**另过卡侬 ExportPdf 位**——Print、Preview、ExportPdf 三位互相独立，这是铁律；
- **字段可见性与表单会话同口径**：`{{record.*}}` 能取到的字段值经卡侬裁决后才下发，用户在表单里看不见的字段，打印上同样拿不到——多联打印不会成为越权导出通道；
- **权限执法在派蒙侧**，插件（尤其 A 档纯 JSON）只声明版面，既不参与也不能干预权限判定；
- **性能边界**：copies 联数是常数级，通常不会触发页数保护；但若与大数据源 repeat 组合，沿用现有保护（>200 页确认 / >1000 页拒绝）。
