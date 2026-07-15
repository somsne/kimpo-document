# 打印扩展教程：混排文档（B 档 buildSpec）

> ⚠️ 底座实施中：manifest 已按 2026-07-15 契约拍板定稿、插件可被宿主真实加载；但打印执行底座（P2）尚未开发，本插件当前无法在真实打印流程中被触发执行。契约以宿主仓 docs/sheet/future/14-printing/ 设计文档为准。

本篇讲**混排文档**打印模式：把封面、派蒙表格段、固定条款、签章尾页拼成**一份完整 PDF**。它是六个演示插件里**唯一带前端代码的**——用 **B 档（`spec_source: "frontend"`）**，由一个 `buildSpec` 函数按单据状态动态拼 spec。参考实现：`kimpo-print-extension/print-demo-composite/`。

---

## 一、打印效果

以"合同整份文档"为例，导出的 PDF 从头到尾是：

1. **封面页**：上方一幅封面底图（image 框），中部一行 26pt 居中大标题——取值 `{{template.name}}`（如"采购合同"），下面一行"合同编号：xxx　签订日期：xxx"（取自单据字段）；
2. **表格段页**：派蒙渲染的合同明细表格——就是用户在表单里看到的那张单，占满版心，随内容长短占一页或多页；
3. **条款页**：固定条款正文（第一条~第四条），纯静态文字，每份合同一字不差；
4. **签章尾页**：右下方一枚印章图（image 框），底部右对齐落款"经办人：{{user.name}}　日期：&D"。

并且是**动态的**：单据没有明细行（如空白预览）时自动跳过表格段页；单据状态不是"已签署"时不出签章尾页，改为整份 PDF 打上斜向"未签署"水印。

## 二、适用场景

- 合同 / 协议：封面 + 业务数据表 + 法务固定条款 + 签章页，一次导出整份；
- 报告 / 报价书：封面 + 若干数据段 + 免责声明尾页；
- 任何"表单数据只是文档的一部分，前后还要拼固定版面"的场景。

**什么时候必须用 B 档？** 当 spec 的**结构本身**要随上下文变（页要不要出、框要不要放、出口选哪种），静态 JSON 模板 + 变量表达式（A 档）就表达不了了——变量只能改"框里的字"，改不了"有哪些框"。反过来，如果你的版面固定、只是文字随单据变（如水印、套打、多联），**优先用 A 档**：零代码即是最安全的默认。

## 三、插件包结构

```
print-demo-composite/
├── manifest.json          # 插件声明 + print-mode 能力（B 档）
└── frontend/
    └── index.js           # UMD：导出 buildCompositeSpec(context) 函数
```

与 A 档相比，`specs/*.json` 换成了 `frontend/index.js`——spec 不再是静态文件，而是函数的返回值。

## 四、manifest 逐段讲解

```json
{
  "entry": { "frontend": "frontend/index.js" },
  "frontend_global_name": "KimpoPrintDemoComposite",
  "capabilities": {
    "print_modes": [
      {
        "id": "composite-contract",
        "name": { "zh_CN": "合同整份文档", "en_US": "Full Contract Document" },
        "modes": ["design-preview", "form-preview", "form-print", "action"],
        "spec_source": "frontend",
        "entry": "buildCompositeSpec"
      }
    ],
    "frontend": {
      "global_name": "KimpoPrintDemoComposite",
      "bundle": "frontend/index.js"
    }
  }
}
```

`print_modes[]` 通用字段（`id` / `name` / `modes`）同 A 档，见多联篇 §四；**2026-07-15 实施修订**：`print_modes` 落在 `capabilities` 对象下（不是数组形 `capabilities: [{type:"print-mode"}]`），字段全 snake_case。B 档比 A 档多两处：

- `spec_source: "frontend"`：**B 档**——spec 由插件前端代码动态构建，而不是读包内静态文件；
- `entry: "buildCompositeSpec"`（`print_modes[0].entry`）：前端 UMD 包导出的 **buildSpec 函数名**。派蒙在用户选中本方案时，按 `meta.printModes[]` 条目携带的 `scriptUrl`（`/api/v1/plugins/{plugin_id}/assets/{entry_js}`，走既有插件资产端点）+ `globalName` 注入脚本，从 `window[globalName]` 取到模块后以 `buildCompositeSpec(context)` 调用它，拿返回的 PrintJobSpec 去执行。A 档的 `spec` 字段在 B 档不出现，两者互斥。

因此顶层还要声明**前端入口与全局名**——`entry.frontend` 指向脚本相对路径、`frontend_global_name`（以及 `capabilities.frontend.global_name`/`bundle`，与 `entry.frontend` 重复声明供不同消费点读取，参考 `kimpo-records-view` 等真实插件的写法）供宿主按 `scriptUrl`/`globalName` 注入脚本并从 `window` 取到 UMD 导出对象。

这个"前端 UMD 导出 hook 函数"的模式与动作插件前端完全同构，写过动作插件前端的开发者可以直接套用工程骨架。

## 五、buildSpec 逐段讲解

完整代码见 `kimpo-print-extension/print-demo-composite/frontend/index.js`（约 120 行，含注释）。

### 5.1 先弄清 B 档与 A 档的边界

- **A 档**：静态 spec 模板 + `{{...}}` 变量，零代码，表达式在**派蒙侧**用受限求值器求值，插件不执行任何代码；
- **B 档**：插件前端导出 `buildSpec(context: PrintContext): PrintJobSpec`，代码运行在**插件前端沙箱**里，能做分支、循环、拼装；
- **共同的铁律**：B 档的产物**仍然是一份纯 JSON 的 PrintJobSpec**，交回派蒙做 JSON Schema 校验后由派蒙的 paginator / composer 执行。插件从头到尾**不碰渲染**——没有 canvas、没有分页器、没有 PDF 库，只是"拼了一份声明"。渲染器对插件零暴露，派蒙内部随时可以重构而不破坏你的插件。

### 5.2 buildSpec 收到什么：PrintContext 五类信息

`context` 是纯 JSON，按契约 §3 分五类（字段命名以宿主契约文档为准）：

1. **单据业务数据**（权威 = Mirror/艾莉亚）：主表字段 `fields[key] = { display, raw }`（显示值/原始值双口径）、明细表行 `detailRows[tableKey]`、单据元信息（recordId、状态、创建人/时间）。设计态没有真实单据，派蒙按字段类型下发**样例数据**保证预览可看；
2. **模板与结构**：templateId、模板名、版本号、字段结构清单、分区结构、已保存的页面设置——做参数面板字段选择器、或按结构决定版面时用；
3. **用户与组织**：当前用户 ID/姓名/部门/角色、语言、**时区**（日期渲染必须按它换算）、应用/实例名；
4. **打印任务上下文**：触发场景（四态之一）与动作触发时的动作上下文（含行级目标）、权限解算结果（canPrint / canPreview / canOverride / canExportPdf，**只供展示，执法在派蒙**）、渲染后回传的分页摘要；
5. **产物与资产**：渲染成品 PrintArtifact（见 §七）；底图/印章等图片走宿主资产存储用 `assetId` 引用，或走插件自带静态资产（`frontend/` 目录，经既有插件资产端点）用 `url` 直链引用，二者二选一（2026-07-15 拍板）。

红线：拿不到 editor 内部状态（canvas/图层/选区几何）；字段可见性与表单会话同口径；一期不给跨单据/跨模板取数。

### 5.3 代码走读

**UMD 头**：浏览器下把工厂产物挂到全局 `KimpoPrintDemoComposite`（宿主前端加载器由此取到 `entry` 声明的函数），CommonJS 下走 `module.exports` 方便本地单测。纯 JS，无任何框架依赖。

**四个 frame 工厂函数**——每"页"一组 frame，id 前缀区分（`cover-` / `sheet-` / `terms-` / `seal-`），`rect` 是页内 mm 坐标（A4 纵向、15mm 边距 → 版心 180×267）：

- `coverFrames()`：封面底图（`image` 框，`url: "/api/v1/plugins/com.kimpo.demo.print-composite/assets/frontend/cover.png"`，`fit: "contain"`）+ 大标题 `text` 框 `{{template.name}}` + 合同编号/日期行（`{{record.contract_no}}` / `{{record.sign_date}}`）。注意：**变量表达式留在返回的 JSON 里、由派蒙求值**，代码不需要（也不应该）自己去拼字段值进字符串——这样字段可见性裁决始终在派蒙/宿主一侧；
- `sheetFrames()`：一个占满版心的 `sheetFragment` 框（`scope: "activeContainer"`，`fit: "width"`）——合同明细就是表单本身，派蒙渲染成内容条带摆进来；
- `termsFrames()`：条款标题 + 条款正文两个 `text` 框，正文是内联的静态字符串常量（无变量，也就没有任何越权取数面）；
- `sealFrames()`：印章 `image` 框（`url: "/api/v1/plugins/com.kimpo.demo.print-composite/assets/frontend/seal.png"`，摆右下）+ 落款 `text` 框 `"经办人：{{user.name}}　日期：&D"` 右对齐。图片走插件包 `frontend/` 目录自带的 PNG，经宿主既有资产端点 `url` 直链引用（也可换成 `assetId` 走宿主资产存储，二者二选一，见套打篇 §五）。

**`buildCompositeSpec(context)` 本体**，三个动态点：

```js
var fields = record.fields || {};                          // §3.1：主表字段 fields[key] = { display, raw }
var itemCount = (detailRows.items || []).length;            // 动态点 1：明细行数
var signed = !!(fields.status && fields.status.raw === "signed"); // 动态点 2：单据状态（判断用 .raw 原始标量口径）
```

> 注意 JS 里访问 `context.record` 要走 **`record.fields[key].raw`**（原始标量，比较用）而不是 `record.fields[key].display`（显示值）——B 档函数运行在插件前端沙箱，取值口径必须与 spec 表达式的 `.raw` 拍板保持一致，不能直接拿 `record.status` 之类扁平字段做字符串比较。

- 明细行数为 0 → 不 concat `sheetFrames()`，整页跳过；
- 未签署 → 不出 `sealFrames()`，同时在 `layers` 里放一个 L1 水印图层 `{ "type": "watermark", "text": "未签署", "angle": -30, "opacity": 0.15 }`，作用于全部页面；
- 动态点 3 在 `output`：触发场景是动作节点（`job.scene === "action"`）时用 `"pdfBlob"` 把成品回递给插件归档，其余场景用 `"pdf"` 直接下载；`fileName` 用变量 `"{{template.name}}_{{record.contract_no}}.pdf"`。

最后返回的对象就是一份标准 PrintJobSpec：`version` / `target` / `layers` / `pageTemplate`（paper/orientation/margins/frames）/ `output`——与 A 档静态模板的字段**一模一样**，只是这次它是算出来的。

## 六、设计态调用

1. 插件安装后 print-mode 随会话下发给派蒙；
2. 设计者在模板设计器点工具栏"打印"，**打印预览对话框**设置区顶部的"打印方案"下拉里选中"合同整份文档"；
3. 派蒙构造**设计态 PrintContext**（业务数据为按字段类型生成的样例数据）调用 `buildCompositeSpec(context)`，对返回的 spec 做 Schema 校验后渲染，**右侧预览所见即所得**——封面、表格段、条款页、（样例状态决定的）水印或签章页逐页可翻；
4. 设计者可把本方案**绑定为该模板的默认打印方案**（可绑多个），随页面设置保存；绑定只存 `pluginId + modeId + params` 引用。

## 七、填报态调用

复用三入口：表单"打印"按钮 / Ctrl+P / 宿主 `openPrintPreview()`；单绑定方案 + 直打策略走 quickPrint，多方案弹选择。**动作入口尤其适合本模式**：审批通过后动作节点自动"按合同方案出 PDF"。

混排文档的出口重点讲两种：

- **`output.kind: "pdf"`（下载）**：派蒙渲染 → pdf-composer 合成 → 浏览器直接下载，文件名由 `fileName` 变量求值而来（如 `采购合同_HT-2026-001.pdf`）。走此出口须过 ExportPdf 位（见 §八）；
- **`output.kind: "pdfBlob"`（回递插件）**：成品不落下载，而是作为 **PrintArtifact**（契约 §6）回递给插件消费：

  ```json
  {
    "kind": "pdf",
    "blob": "<Blob>",
    "meta": { "pageCount": 5, "paper": "A4", "templateName": "采购合同",
              "recordId": "…", "generatedAt": "…" }
  }
  ```

  插件拿到 blob 后可做归档上传、推云打印、与其他文档二次合并——这正是"合同出完自动归档到文件柜"链路的接法。`kind: "pages"` 变体则回递逐页位图（`pages[].dataUrl`），适合做缩略图或图像化流转。

## 八、权限与边界

- **打印权限**：一切模式统一过 **卡侬 Print 位 AND 模板策略三项**（RuntimeAllowPrint / Preview / PaperOverride），buildSpec 写什么都绕不过；
- **PDF 出口独立**：本模式 `output.kind` 是 `pdf` / `pdfBlob`，**必须另过卡侬 ExportPdf 位**——Print、Preview、ExportPdf 三位互相独立，这是铁律。用户有打印权没导出权时，PDF 出口会被派蒙拒绝，插件应准备好这条失败路径；
- **字段可见性与表单会话同口径**：PrintContext 里的字段值经卡侬裁决后才下发，用户不可见的字段 buildSpec 根本拿不到、`{{record.*}}` 也求不出——混排导出不会成为越权通道；
- **权限执法在派蒙**：context 里的 canPrint / canExportPdf 只是**展示用**的解算结果（比如你想在参数面板置灰某选项），真正的闸在派蒙执行侧；
- **代码边界**：buildSpec 运行在插件前端沙箱，产物是纯 JSON 且必过派蒙 Schema 校验——自创字段、非法表达式命名空间（`record` / `user` / `template` / `item` 与 `&P/&N/&D/&T/&F/&U` 之外）都会被校验拒绝；
- **时区**：`&D` 与日期字段变量按当前用户时区换算（时区单一换算点），不要在 buildSpec 里自己 `new Date()` 拼日期字符串。
