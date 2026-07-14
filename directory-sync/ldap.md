# LDAP / AD 通讯录同步 · 手把手教程

> 把企业自建的 LDAP 目录（含 Windows Active Directory、OpenLDAP 等）里的部门和员工同步进 Kimpo。
> 建议先读[总览](./README.md)了解「认领 / 权威模式 / 工号 / 密钥安全」这几个概念。

LDAP 比钉钉/飞书稍技术一点——它不是"点几下网页"，而是要填一些目录服务的连接参数。别怕，本文把每个字段都讲清楚，并给出 **Windows AD / OpenLDAP 的常见取值**，最后还有一个**免注册的公共练手服务器**让你先跑通。

---

## 0. 先理解：LDAP 是什么

LDAP 是很多企业存"组织架构 + 账号"的标准目录服务：

- **Active Directory（AD）**：Windows 域控用的，最常见。
- **OpenLDAP** 等：Linux 世界的开源目录。

里面的数据是一棵树，每个节点用 **DN（Distinguished Name，可以理解为"全路径名"）** 定位，例如：
- 一个员工：`uid=zhangsan,ou=People,dc=corp,dc=example,dc=com`
- 一个部门：`ou=研发部,dc=corp,dc=example,dc=com`

要读它，你需要：**服务器地址**、**一个能只读查询的账号**、**从哪个节点开始查（Base DN）**、以及**怎么区分"部门"和"人"（过滤器）**。

---

## A. 在你的目录侧准备

找你的**域管理员 / 运维**要这些信息（他们一看就懂）：

| 你要问到的 | 说明 | AD 常见例 | OpenLDAP 常见例 |
|---|---|---|---|
| 服务器地址 + 端口 | 目录服务器 | `dc01.corp.example.com` : `389`（或 LDAPS `636`） | 同左 |
| 只读绑定账号 DN | 一个用于查询的服务账号（**建议只读**） | `CN=svc-kimpo,OU=Service Accounts,DC=corp,DC=example,DC=com` | `cn=readonly,dc=example,dc=com` |
| 该账号的密码 | ✅ 敏感，走环境变量 | —— | —— |
| Base DN | 从哪开始查（通常是域根或某个 OU） | `DC=corp,DC=example,DC=com` | `dc=example,dc=com` |
| 是否加密 | 生产强烈建议 LDAPS(636) 或 StartTLS | —— | —— |

> 🔒 安全建议：给 Kimpo 用一个**专用的只读服务账号**，不要用域管理员账号。

---

## B. 在 Kimpo 控制台配置

### B0. 密钥交给运维设环境变量

把绑定账号的**密码**交给部署同事，设成环境变量，例如：

```
LDAP_BIND_PASSWORD = <只读账号的密码>
```

记住变量名 `LDAP_BIND_PASSWORD`，界面只填名字、不填密码本身。详见[总览 §5](./README.md#5-一条安全铁律密钥不落库不填明文)。

### B1. 新建目录源

**控制台 → 外部目录** → **【新建目录源】**，**目录类型选「LDAP」**，然后会多出一组 LDAP 专属字段：

| 表单项 | 填什么 | 说明 |
|---|---|---|
| **名称** | 如"总部AD" | —— |
| **凭据引用** | `LDAP_BIND_PASSWORD` | 绑定密码的环境变量名 |
| **主机地址** | `dc01.corp.example.com` | 目录服务器域名/IP |
| **端口** | `389`（LDAPS 用 `636`） | —— |
| **使用 LDAPS** | 生产建议开 | 走 636 加密端口时开 |
| **使用 StartTLS** | 视目录支持 | 389 端口上升级为加密时开 |
| **Bind DN** | `CN=svc-kimpo,...,DC=corp,DC=example,DC=com` | 只读账号的全路径名 |
| **Base DN** | `DC=corp,DC=example,DC=com` | 从哪开始查 |
| **用户过滤器** | 见下表 | 怎么识别"人" |
| **部门过滤器** | `(objectClass=organizationalUnit)` | 怎么识别"部门" |
| **登录名属性** | AD:`sAMAccountName` / OpenLDAP:`uid` | 作为登录标识 |
| **姓名属性** | AD:`displayName` / OpenLDAP:`cn` | 显示名 |
| **邮箱属性** | `mail` | —— |
| **手机属性** | `mobile` 或 `telephoneNumber` | —— |
| **权威模式** | 「仅补全本地空字段」（推荐） | —— |
| **匹配策略** | 「自动绑定已有对象」（开启认领） | —— |
| **登录来源码** | 默认 `ldap` | —— |

**过滤器常见取值**：

| 目录 | 用户过滤器 | 部门过滤器 |
|---|---|---|
| Active Directory | `(&(objectClass=user)(objectCategory=person))` | `(objectClass=organizationalUnit)` |
| OpenLDAP | `(objectClass=inetOrgPerson)` | `(objectClass=organizationalUnit)` |

保存。

### B2~B5. 测试连接 → 预览 → 认领 → 应用

和其它来源一样：

1. **测试连接** → "连接成功：部门 X · 用户 Y"。连不上先看[常见问题](#常见问题)。
2. **同步预览** → 生成差异（不写数据）。
3. **差异审阅** → 需要合并的手动认领。
4. **应用已选** → 真正写入。

### B6. 关于工号

- Kimpo 会读取标准的 **`employeeNumber`** 属性作为真实工号（AD/LDAP 里存工号的标准字段）。
- 如果某人没有 `employeeNumber`，Kimpo 自动生成系统工号 `U000123`。
- LDAP 的 `uid` / `sAMAccountName` 只作登录身份，**不**当工号。

---

## C. 想先练手？用公共测试服务器（免注册）

如果你还没拿到公司目录的参数，可以先用一个免注册的公共测试 LDAP 服务器把整条流程跑通、熟悉界面。**这些参数经过实测可用**：

| 表单项 | 值 |
|---|---|
| 凭据引用 | 一个你设成值 `password` 的环境变量名（如 `LDAP_DEMO_PW`） |
| 主机地址 | `ldap.forumsys.com` |
| 端口 | `389` |
| 使用 LDAPS / StartTLS | 都关 |
| Bind DN | `cn=read-only-admin,dc=example,dc=com` |
| Base DN | `dc=example,dc=com` |
| 用户过滤器 | `(objectClass=inetOrgPerson)` |
| 部门过滤器 | `(objectClass=organizationalUnit)` |
| 登录名属性 | `uid` |
| 姓名属性 | `cn` |
| 邮箱属性 | `mail` |

跑完你会看到：测试连接显示 **0 部门 · 14 用户**（这个测试库是扁平的、没有部门），应用后进来 14 位科学家（牛顿、爱因斯坦…）。因为他们都没有 `employeeNumber`，所以每人拿到一个系统工号 `U0000xx`——正好演示了"无工号自动生成、技术 uid 不当工号"。

> ⚠️ 这是公共测试库，**仅供练手**，别拿它当真实数据源。

---

## 常见问题

| 现象 | 原因 & 解决 |
|---|---|
| 连不上 / 超时 | 主机、端口不对，或网络/防火墙不通。让运维确认 Kimpo 服务器能访问目录服务器 |
| `ldap bind` 失败 | Bind DN 写错，或密码环境变量没设对。核对 DN 全路径、变量名与"凭据引用"一致 |
| 部门数为 0 | 你的目录里部门可能不是 `organizationalUnit`（有些用组/别的类）。让运维告诉你部门的 objectClass，改"部门过滤器" |
| 用户数为 0 | 用户过滤器不匹配。AD 用 `(&(objectClass=user)(objectCategory=person))`，OpenLDAP 用 `(objectClass=inetOrgPerson)` |
| 姓名/邮箱空 | 属性名映射不对。让运维确认你的目录里姓名/邮箱存在哪个属性，改对应"…属性"字段 |

---

## 参考资料

- Microsoft · Active Directory 架构与属性（含 `employeeNumber`）：https://learn.microsoft.com/windows/win32/adschema/attributes
- OpenLDAP 官方文档：https://www.openldap.org/doc/
- 公共测试 LDAP（练手）：https://www.forumsys.com/2022/05/10/online-ldap-test-server/

---

← 返回 [外部目录同步总览](./README.md)
