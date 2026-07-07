# kimpo-document — Kimpo 插件开发文档

面向**第三方插件开发者**的公开文档仓：讲清 Kimpo 平台与插件体系的业务原理，给出标准 SDK 对接方式与开发规范，让你不读宿主源码也能快速开发出合规、稳定的 Kimpo 插件。

- 读者：人类工程师 与 AI 编程助手（部分专题提供两个版本）。
- 原则：所有文档**不依赖宿主源码可读**，可独立分发；与宿主内部架构文档分工明确——这里只讲"你该知道什么、该怎么接"。

---

## 1. Kimpo 是什么

Kimpo 是一个**电子表格式的低代码表单/报表平台**：管理员在类 Excel 的设计器里画出模板（字段、公式、控件、动作），业务用户基于模板填报数据，平台负责数据落库、查询、权限与流程编排。

你需要掌握的核心概念：

| 概念 | 说明 |
|---|---|
| **应用（App）** | 租户内的业务单元，持有一棵模板目录树与独立的业务数据库 |
| **模板（Template）** | 一张表单/报表的定义。分**设计态**（管理员画模板：字段绑定、公式、动作、控件）与**填报态**（用户新增/修改/查看数据单据） |
| **数据表（DataTable）** | 模板落库的物理载体：**主表**（一单一行）+ **明细/扩展表**（一单多行，行键 `row:N`）。字段落物理列 |
| **字段（Field）** | 业务数据最小单元，绑定到模板的单元格；分 **input**（用户/插件可写）与 **computed**（公式派生，只读） |
| **动作（Action）** | 模板上编排的自动化流程（如"打开表单时取数"）：触发时机 + 条件 + 动作节点。**动作的执行体由插件贡献** |
| **艾莉亚(Aria) / Mirror** | 填报态业务数据的**唯一权威**：会话级内存镜像，所有读写经宿主统一入口有序进行。插件读写表单数据的唯一通道 → 必读 [aria/](#3-文档目录) |
| **表达式/查询** | 平台统一的取数语言：插件提交表达式模型（或 KimpoSQL 文本），宿主编译成 SQL 执行并返回行集——**插件永不碰裸 SQL** |

## 2. 插件体系

### 2.1 架构：独立进程 sidecar + 双向 gRPC

```
┌──────────── Kimpo 宿主（单体服务）────────────┐
│  模板/数据/权限/动作编排/艾莉亚(Aria) Mirror/查询编译   │
│     ▲ 反向通道(插件→宿主: Host 服务)            │
│     │                    │ 正向通道(宿主→插件)   │
└─────┼────────────────────┼────────────────────┘
      │                    ▼
   ┌──┴─────────────────────────┐   ×N 个插件实例
   │ 你的插件进程（sidecar）        │   每实例独立进程、
   │ plugin.bin + 可选前端 UMD    │   资源限额、崩溃隔离
   └────────────────────────────┘
```

- 宿主以**独立子进程**拉起你的插件二进制（`plugin.bin --instance-id … --config …`），崩溃不拖垮平台；SDK 内建"宿主死亡自杀 watchdog"，不留孤儿进程。
- **正向通道**：宿主调你注册的方向服务（动作执行、编辑器 RPC…）。
- **反向通道**：你经 SDK `Host` 句柄调宿主能力（数据读写、查询、日志…）。

### 2.2 插件类型（按 capabilities 声明）

| 类型 | runtime_kinds / directions | 干什么 | 参考实现 |
|---|---|---|---|
| **动作插件** | `action` / `biz` | 贡献动作执行体（取数、写数、通知…），被模板动作编排调用 | `kimpo-extraction`（取数）、`kimpo-record-create/update/delete`（三写） |
| **编辑器插件** | `editor` | 提供一种表单设计/填报编辑器（渲染 + 公式内核），承担艾莉亚(Aria)编辑器契约 | `kimpo-sheet`（电子表格编辑器） |
| 复合插件 | 多方向组合 | 一个插件同时注册多个方向（editor+data+hooks） | — |

### 2.3 插件包（.kpp）

`.kpp` = Kimpo Plugin Package（zip 归档），经宿主 `POST /api/v1/plugins/install` 安装，装完自动重启 sidecar：

```
my-plugin-1.0.0.kpp
├── manifest.json                  # 插件自描述（必须）
├── plugin.bin                     # 后端二进制（Go，SDK 构建）
├── config.schema.json             # 实例配置 JSON Schema（可选）
└── frontend/dist/plugin.umd.js    # 前端 Surface bundle（可选）
```

`manifest.json` 关键字段（以真实写动作插件为例）：

```jsonc
{
  "plugin_id": "com.example.myplugin",      // 全局唯一，反域名风格
  "name":        { "zh_CN": "…", "en_US": "…" },   // 多语言（规范 R6）
  "description": { "zh_CN": "…", "en_US": "…" },
  "version": "1.0.0",
  "category": "action",
  "entry": { "backend": "plugin.bin" },
  "config_schema": "config.schema.json",
  "process": { "memory_limit_mb": 128, "cpu_limit_pct": 25, "max_concurrent_grpc": 8 },
  "permissions": { "network": [], "storage": false, "exec": false },  // 诚实最小化
  "lifecycle": { "allow_multiple_config_instances": true, "max_config_instances": 8 },
  "capabilities": {
    "runtime_kinds": ["action"],
    "directions": ["biz"],                  // 注册方向必须 ⊆ 此声明
    "backend_services": ["my-plugin"]
  }
}
```

### 2.4 SDK（Go）

模块：`github.com/kimposoft/kimpo-plugin-sdk`。入口范式：

```go
func main() {
    // ParseFlags 解析 sidecar 约定参数（--instance-id/--config/--grpc-addr）
    if err := plugin.Serve(plugin.ParseFlags(),
        data.RegisterSource(myActions),   // 按需注册一个或多个方向
        // editor.Register(myEditor), hooks.Register(myHooks), …
    ); err != nil { log.Fatal(err) }
}
```

运行期经 `host := plugin.HostFrom(ctx)` 拿宿主句柄，十个子客户端：

| 客户端 | 用途 |
|---|---|
| `Mirror()` | **填报态业务数据读写**（艾莉亚(Aria)，必读 aria/ 文档） |
| `Query()` | 反向查询：提交表达式模型/KimpoSQL，宿主编译执行返回行集 |
| `Record()` | 记录级写服务（写动作 insert/update/delete 专用通道） |
| `Logger()` | **分级日志上报**（见规范 R5）；`Log()` 为底层通道 |
| `Template()` | 模板/记录元数据读取（设计文档、报表记录快照等） |
| `Capability()` | 能力总线：注册自身能力 / 调用其他插件声明的能力（插件间**唯一**合法交互方式） |
| `Cache()` / `Asset()` / `Event()` | KV 缓存 / 资产上传下载 / 事件上报 |

动作插件的触发上下文（`query_context`）里由宿主注入：`sheet_session_id`（目标填报会话）、`write_grant`（本次调用的写授权令牌）等——原样取用、原样透传。

## 3. 文档目录

### aria/ — 艾莉亚(Aria)（填报态业务数据层）★ 开发任何读写表单数据的插件前必读

| 文档 | 读者 | 内容 |
|---|---|---|
| [for-human-developers.md](aria/for-human-developers.md) | 人类开发者 | 账本/前台/显示屏类比讲业务原理 → 写入生命周期 → 四项内建保障 → SDK 对接步骤 → 红线与 FAQ |
| [for-llm-agents.md](aria/for-llm-agents.md) | AI 编程助手 | 结构化上下文版：FACTS / INTERFACE / 错误决策表 / MUST-MUST NOT / 典型调用序列 / 自检清单。整体喂给你的 AI 即可 |

### 规划中

反向查询与表达式模型专题 · 前端 Surface 对接专题 · 动作插件完整教程（脚手架到上架） · 编辑器插件契约专题

## 4. 快速开始（动作插件 10 分钟骨架）

1. **脚手架**：用 `new-action-plugin` 脚手架生成骨架（manifest / main.go / Makefile / config.schema.json）。
2. **实现动作**：在注册的动作 handler 里，从 `query_context` 取 `sheet_session_id`/`write_grant`，用 `host.Query()` 取数、`host.Mirror()` 写回（照抄 aria 文档 S1-S3 序列）。
3. **构建打包**：`make build` 产 `plugin.bin`（目标平台注意 GOOS/GOARCH，避免陈旧 x86_64 在 ARM 宿主上走翻译层）→ zip 成 `.kpp`。
4. **安装验证**：`POST /api/v1/plugins/install` 上传 → 平台自动重启 sidecar → 在模板动作编排里挂上你的动作真机验证。日志看宿主控制台"插件日志"（你经 `host.Logger()` 上报的会集中展示）。

## 5. 开发规范

### 5.1 硬性红线（违反即打回）

- **R1 数据唯一通道**：业务数据读写只走 `host.Mirror()` / `host.Record()`，查询只走 `host.Query()`。**禁止**：直连编辑器插件（import editorpb / 调 editor RPC / 经 WS 命令写值）、自拼 SQL 碰 `app_table_*` 物理表、读"界面显示值"当数据源。
- **R2 grant 纪律**：`write_grant` 是宿主按单次动作调用签发的一次性凭证——原样透传，**禁止**缓存、持久化、跨动作复用、伪造。
- **R3 插件间低耦合**：插件之间**只能**经宿主能力总线（`Capability()`）交互，禁止互相直连进程/共享文件/私自约定端口。
- **R4 权限诚实最小化**：manifest `permissions`（network/storage/exec）按实际需要声明，`process` 资源限额如实填写。
- **R5 进程行为**：优雅处理 SIGTERM 退出；不 fork 不受管的子进程；不绕过 SDK 自建对宿主的连接。

### 5.2 工程规范（强烈建议，上架审查参考）

- **日志**：关键路径用 `host.Logger()` 分级上报（trace/debug/info/warn/error），错误必带上下文（sessionID/tableID/动作名）；`security`/`audit` 类恒上报不受级别过滤。实例日志级别由管理员在控制台配置（`config_json` 顶层保留字 `log_level`），你无需自己实现开关。
- **多语言**：manifest 的 `name`/`description` 至少提供 `zh_CN` + `en_US`；所有面向最终用户的文案（错误提示、动作参数面板）须多语言，随插件自带 locale 资源。
- **配置**：实例配置用 `config.schema.json` 声明结构（宿主安装时浅校验）；顶层键 `log_level` 为平台保留字，勿挪作他用。
- **错误语义**：遵守 aria 错误决策表——特别是"权威已写、仅渲染通知失败"必须按成功处理，勿回滚业务流程。
- **性能**：批量写用 `WriteFields` 分批（一批=一个屏障窗口=一帧渲染）；读结算值才传 `waitSettled=true`；追加明细行前先复用空行。
- **测试**：动作逻辑单测覆盖 + `go test -race` 干净；对宿主句柄做 fake（SDK 接口皆可 mock）；发布前真机走通"触发→写回→保存落库"全链。
- **版本兼容**：`dependencies.platform` 如实声明最低平台版本；平台线协议（proto）演进后**须重新构建打包**，旧 `.kpp` 可能收不到新字段（如批量写的单字段错误透出）。

### 5.3 提交前自检清单

```
□ 只出现 host.Mirror()/Record()/Query()，无 editorpb、无裸 SQL      (R1)
□ write_grant 未被缓存/复用                                        (R2)
□ 插件间交互只经 Capability()                                      (R3)
□ manifest 权限与资源限额如实                                       (R4)
□ 关键路径有分级日志；文案多语言                                     (5.2)
□ go test -race 绿；真机全链验证过                                  (5.2)
□ 处理了 computed 拒写 / not-writable / session-not-found / 渲染失败四种错误语义
```

## 6. 参考实现

| 插件 | 看什么 |
|---|---|
| **kimpo-record-extraction**（取数） | 动作插件全范式：`Host.Query()` 反向取数 + `Host.Mirror()` 写回 + 内存加工（左驱 join/填充策略）+ 分级日志 |
| **kimpo-record-create / update / delete**（三写） | 写动作范式：`Host.Record()` 记录写通道 + 表达式来源 + grant 消费 |
| **kimpo-sheet**（电子表格编辑器） | 编辑器插件契约：多方向注册、艾莉亚(Aria)编辑器三契约、前端 Surface |
