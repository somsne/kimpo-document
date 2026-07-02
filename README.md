# kimpo-document

Kimpo 插件开发文档子工程（子模块）。面向**插件开发者**（人类与 AI 编程助手）的对外文档：讲清平台核心机制的业务原理，并引导用标准 SDK（`kimpo-plugin-sdk`）完成数据交互与对接。

与宿主仓 `docs/` 的分工：宿主 `docs/` 记录内部架构与实现细节（含宿主源码引用）；本仓只面向插件开发视角，**不依赖宿主源码可读**，可独立分发。

## 目录

### polaris/ — 北极星（填报态业务数据层）

填报表单业务数据的唯一权威与读写通道。开发任何要读写表单数据的插件前必读。

| 文档 | 读者 | 内容 |
|---|---|---|
| [for-human-developers.md](polaris/for-human-developers.md) | 人类插件开发者 | 账本/前台/显示屏类比讲业务原理 → 写入生命周期 → 四项内建保障 → SDK 对接步骤（句柄/读/写/明细行/错误处理）→ 红线与 FAQ |
| [for-llm-agents.md](polaris/for-llm-agents.md) | 大模型 / AI 编程助手 | 结构化上下文版：FACTS / INTERFACE / GUARANTEES / 错误决策表 / MUST-MUST NOT 规则 / 典型调用序列 / 生成前自检清单。整体注入 system prompt 或 RAG 即可 |

**怎么选**：你自己读 → 人类版；你让 AI 助手写插件代码 → 把 LLM 版喂给它（两份内容同源，粒度不同）。

## 参考实现

- **kimpo-extraction（取数插件）**：北极星对接的活样板——全部宿主交互仅 `host.Query()` + `host.Mirror()`。

## 规划中

- 反向查询（`Host.Query()` / 表达式模型）专题
- 插件前端 Surface 对接专题
- 日志上报（`host.Logger()`）专题
