# 打印扩展（print-mode）— 用插件实现自定义打印模式

> ⚠️ **状态**：底座实施中（契约已按 2026-07-15 拍板定稿，manifest 可被宿主真实加载）。本目录的演示插件 manifest/spec 已对齐实施拍板，但打印执行底座（P1 参数化 / P2 框式编排）尚未开发，插件当前无法在真实打印流程中被触发执行；契约以宿主仓 `docs/sheet/future/14-printing/kimpo-sheet-print-extension-plan-2026-07-15.md` 设计文档为准，底座落地后本目录随之转正为可运行教程。

Kimpo 的表格设计器（KimpoSheet）内建了完整的打印与导出 PDF 能力：分页、缩放、页眉页脚、权限门控。**打印扩展**允许你的插件在不接触任何渲染细节的前提下，复用这套出图能力实现自定义打印模式——水印、套打、标签 N 联、混排文档等复杂排版。

## 1. 一句话原理

你的插件只做两件事：**声明一个打印模式**（manifest capability），**给出一份排版规格 `PrintJobSpec`**（纯 JSON）。渲染、分页、出 PDF、权限执法全部由平台完成；平台把成品（PDF/页面位图）交回给你（可选）。

```
插件: print-mode 声明 + PrintJobSpec(JSON) ──▶ 平台: 渲染/分页/出图 ──▶ 成品(打印/PDF/回递)
```

spec 的来源分两档：

| 档 | spec_source | 你要写的东西 | 适用 |
|---|---|---|---|
| **A 档 声明式** | `static` | 一份静态 spec 模板 + `{{变量}}` 表达式，**零代码** | 绝大多数场景 |
| **B 档 编程式** | `frontend` | 前端 UMD 导出 `buildSpec(context)` 函数，动态拼 spec | 需要按数据/状态做复杂逻辑时 |

两档产物都是纯 JSON，经平台校验后执行——插件永远不执行渲染。

## 2. 排版能力两层

- **L1 版式覆盖**：在模板已保存的页面设置之上做增量覆盖（纸张/边距/缩放/打印范围/页眉页脚），外加全页图层（水印/底图）。
- **L2 页面编排**：自定义页面模板——按 mm 坐标摆放**内容框**（表格内容片段 / 图片 / 变量文本 / 条码），配迭代规则（标签网格 / 明细行循环 / 多联）。

变量表达式：页码类沿用 `&P &N &D &T &F &U`；数据类用 `{{record.*}}`（单据字段，经权威数据层取值）、`{{user.*}}`、`{{template.*}}`、`{{item.*}}`（迭代行）、`{{params.*}}`（方案参数，见 §4）。**取值口径（2026-07-15 拍板）**：`record.<code>` / `item.<code>` 默认返回**显示值**；追加 `.raw` 返回原始标量，专供比较/三目判断使用，如 `{{record.status.raw == 'void' ? '已作废' : params.watermarkText}}`。受限文法 = 点路径取值 + 字符串字面量 + `==`/`!=` 比较 + 三目，无任意 JS。

## 3. 演示插件与教程索引

演示插件集中在独立仓库 [somsne/kimpo-print-extension](https://github.com/somsne/kimpo-print-extension)（宿主仓内路径 `plugins/kimpo-print-extension/`）：

| 教程 | 类型 | 层级 | 档位 | 演示插件 |
|---|---|---|---|---|
| [watermark.md](watermark.md) | 水印打印（按单据状态动态水印） | L1 | A | `kimpo-print-extension/print-demo-watermark/` |
| [layout-override.md](layout-override.md) | 版式覆盖/局部打印（回执 A5 横向） | L1 | A | `kimpo-print-extension/print-demo-layout-override/` |
| [overlay-print.md](overlay-print.md) | 套打（快递面单/预印刷纸） | L2 | A | `kimpo-print-extension/print-demo-overlay/` |
| [labels-nup.md](labels-nup.md) | 标签 N 联（2×5 网格+条码） | L2 | A | `kimpo-print-extension/print-demo-labels/` |
| [multi-copy.md](multi-copy.md) | 一页多单/多联（存根联+客户联） | L2 | A | `kimpo-print-extension/print-demo-multi-copy/` |
| [composite-doc.md](composite-doc.md) | 混排文档（封面+表格+条款+签章） | L2 | **B** | `kimpo-print-extension/print-demo-composite/` |

建议按表格顺序阅读：前两篇建立 L1 心智，中间三篇覆盖 L2 声明式编排，最后一篇是唯一需要写代码的 B 档。

## 4. 两态调用速览

- **设计态（管理员）**：打印预览对话框 → "打印方案"下拉出现你的模式 → 调参数 → 右侧所见即所得预览 → 可绑定为该模板的默认打印方案。
- **填报态（业务用户）**：打印按钮 / Ctrl+P / 宿主 API 三个入口按绑定方案直打或预览；也可由交互按钮的"打印"动作节点触发（套打场景最自然的入口）。

## 5. 权限红线（所有模式一视同仁）

1. 一切打印统一过 **打印授权位 AND 模板打印策略**，插件模式不可绕过；导出 PDF 另有独立授权位（打印/导出 Excel/导出 PDF 三位互不继承）。
2. **字段可见性与表单会话同口径**：用户无权看到的字段，你的 spec 也取不到——打印不能成为越权导出通道。
3. 单据数据一律由平台经权威数据层（Mirror）取值供给，插件不自行取数。
