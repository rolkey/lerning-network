这份 H3C（Comware）与华为（VRP）的命令对比表框架很好，但目前涉及的场景较少。以下为你补充并完善涵盖**模式切换、VLAN/端口配置、路由协议、ACL与NAT、安全与运维管理**等日常核心场景的详细对比表：

### 华三 (H3C) vs 华为 (Huawei) 常用命令对照表

| 分类 / 功能 | 华三 (H3C Comware) | 华为 (Huawei VRP) | 说明 / 备注 |
| --- | --- | --- | --- |
| **视图与基础模式** |  |  |  |
| 进入系统视图 | `system-view` | `system-view` | 均为全局配置视图 |
| 物理口转三层接口 | `port link-mode route` | `undo portswitch` | 关键差异：H3C 用模式切换，华为用取消切交换模式 |
| 物理口转二层接口 | `port link-mode bridge` | `portswitch` | H3C 默认为 bridge 模式 |
| 进入 VLAN 三层接口 | `interface Vlan-interface 10` | `interface Vlanif 10` | **关键词差异：`Vlan-interface` vs `Vlanif**` |
| **VLAN 与链路聚合** |  |  |  |
| 配置端口 Access VLAN | `port access vlan 10` | `port default vlan 10` | 华为使用 `default vlan` |
| 配置 Trunk 允许 VLAN | `port trunk permit vlan 10 20` | `port trunk allow-pass vlan 10 20` | 华为使用 `allow-pass` |
| 创建链路聚合接口 | `interface Bridge-Aggregation 1` | `interface Eth-Trunk 1` | H3C 称 Bridge-Aggregation，华为称 Eth-Trunk |
| 将接口加入聚合组 | `port link-aggregation group 1` | `eth-trunk 1` | 华为直接在接口下调用 `eth-trunk` |
| **路由协议** |  |  |  |
| 查看 IP 路由表 | `display ip routing-table` | `display ip routing-table` | 基础查看命令相同 |
| 配置静态路由 | `ip route-static 0.0.0.0 0 192.168.1.1` | `ip route-static 0.0.0.0 0.0.0.0 192.168.1.1` | H3C 支持简写掩码长度 `0` |
| 宣告 OSPF 规则 | `network 192.168.1.0 0.0.0.255` | `network 192.168.1.0 0.0.0.255` | 均在 Area 视图下配置反掩码 |
| **ACL 与 NAT (安全/防火墙)** |  |  |  |
| 创建基本 ACL | `acl basic 2000` 或 `acl number 2000` | `acl 2000` 或 `acl number 2000` | H3C 明确区分 basic / advanced |
| 创建高级 ACL | `acl advanced 3000` | `acl 3000` | H3C 新版 Comware v7 推荐写明类型 |
| NAT 动态地址转换应用 | `nat outbound 2000` | `nat outbound 2000` | 均在出口接口配置 |
| 规则匹配默认顺序 | auto（自动/深度优先） | config（配置顺序/自上而下） | H3C 支持 auto/config，华为默认为 config |
| **运维、监控与抓包** |  |  |  |
| 保存配置 | `save` | `save` | 提示确认时输入 `Y` |
| 查看接口状态/摘要 | `display ip interface brief` | `display ip interface brief` | 相同 |
| 查看当前接口配置 | `display current-configuration interface` | `display current-configuration interface` | 相同 |
| 清除接口统计计数 | `reset counters interface` | `reset counters interface` | 相同 |
| 开启日志/告警输出 | `terminal monitor` | `terminal monitor` | 运维调试常用 |

---

> **核心记忆避坑点：**
> 1. **VLAN 接口名称：** H3C 务必全拼或缩写为 `Vlan-interface`（如 `int vlan 10`），华为固定为 `Vlanif`（如 `int vlanif 10`）。
> 2. **端口模式切换：** 交换机物理口做三层 IP 配置时，H3C 须执行 `port link-mode route`；华为须执行 `undo portswitch`。
> 3. **Trunk 放行：** H3C 用 `permit`，华为用 `allow-pass`。
> 
>

---

以下是针对华三防火墙（SecPath 系列，基于 Comware V7）与华为防火墙（USG 系列，基于 VRP）在安全区域（Security Zone）**与**安全策略（Security Policy）核心命令上的对比表：

### 华三 SecPath vs 华为 USG 安全区域与策略命令对照

| 功能模块 | 华三 SecPath (Comware V7) | 华为 USG (VRP) | 说明 / 核心差异 |
| --- | --- | --- | --- |
| **安全区域管理** |  |  |  |
| 进入/创建安全区域 | `security-zone name Trust` | `firewall zone trust` | 华为自带 Trust、Untrust、DMZ 等默认区域，H3C 也支持自定义及默认区域 |
| 将接口加入安全区域 | 在区域视图下：<br>`import interface GigabitEthernet 1/0/1` | 在区域视图下：<br>`add interface GigabitEthernet 1/0/1` | 华三使用 `import interface`，华为使用 `add interface` |
| **安全策略 (Security Policy)** |  |  |  |
| 进入安全策略视图 | `security-policy ip` | `security-policy` | H3C 需指定协议类型（如 `ip`），华为直接进入 |
| 创建/进入安全策略规则 | `rule name permit_web` | `rule name permit_web` | 均支持自定义规则名称 |
| 配置动作 (允许/拒绝) | `action permit` / `action drop` | `action permit` / `action deny` | 华三拒绝动作为 `drop`，华为为 `deny` |
| 配置源/目的安全区域 | `source-zone trust`<br>`destination-zone untrust` | `source-zone trust`<br>`destination-zone untrust` | 关键字完全一致 |
| 配置源/目的 IP 地址 | `source-ip 192.168.1.0 24`<br>`destination-ip 10.0.0.0 24` | `source-address 192.168.1.0 24`<br>`destination-address 10.0.0.0 24` | 华三使用 `source-ip` / `destination-ip`，华为使用 `source-address` / `destination-address` |
| **对象与状态化巡检** |  |  |  |
| 创建 IP 地址对象组 | `object-group ip address web_servers` | `ip address-set web_servers type object` | 对象组命名与结构定义方式略有不同 |
| 查看安全策略命中统计 | `display security-policy hit-count` | `display security-policy hit-count` | 用于排查策略是否生效及流量匹配情况 |

---

> **核心避坑提示：**
> 1. **区域绑定逻辑：** 华为防火墙习惯在具体 Zone 视图下用 `add interface` 把接口拉进来；而华三 SecPath 同样是在区域视图下用 `import interface` 引入，逻辑方向类似，但动词不同。
> 2. **策略地址匹配：** 华三策略中定义 IP 范围时使用 `source-ip`，华为则习惯使用标准的 `source-address`。
> 
>

---

以下是华三 SecPath 防火墙（基于 Comware V7）与华为 USG 防火墙（基于 VRP）在 **NAT 策略**（包括源 NAT/SNAT、目的 NAT/DNAT、NAT Server）上的核心命令对比表：

### 华三 SecPath vs 华为 USG NAT 策略命令对照表

| 功能模块与类型 | 华三 SecPath (Comware V7) | 华为 USG (VRP) | 说明 / 核心差异 |
| --- | --- | --- | --- |
| **源 NAT (SNAT / 动态地址转换)** |  |  |  |
| **传统接口下 SNAT**<br>(基于 ACL + 地址池) | 1. 定义地址池：<br>`nat address-group 1`<br>` address 1.1.1.10 1.1.1.20`<br>2. 接口调用：<br>`interface GigabitEthernet 1/0/1`<br>` nat outbound 2000 address-group 1` | 1. 定义地址组：<br>`nat address-group ag1 0`<br>` mode pat`<br>` section 0 1.1.1.10 1.1.1.20`<br>2. 接口调用：<br>`nat-policy`<br>` rule name snat`<br>` action source-nat address-group ag1` | 华为在新版 USG 中更推荐采用统一的“NAT策略”视图，而传统接口命令两家逻辑类似。 |
| **Easy IP** (借用接口IP) | `nat outbound 2000` | `action source-nat easy-ip` (在 NAT 策略中) 或接口下 `nat outbound` | 均支持使用出接口 IP 进行端口多路复用 (PAT)。 |
| **全球 NAT 策略 (Global NAT Policy)** | `nat policy`<br>` rule name snat_policy`<br>` source-zone trust`<br>` destination-zone untrust`<br>` action source-nat address-group 1` | `nat-policy`<br>` rule name snat_policy`<br>` source-zone trust`<br>` destination-zone untrust`<br>` action source-nat address-group ag1` | 现代防火墙架构下，两家均支持基于安全区域、IP、应用等细粒度匹配的集中式 NAT 策略。 |
| **目的 NAT (DNAT / 端口转发)** |  |  |  |
| **基于 NAT 策略的 DNAT** | `nat policy`<br>` rule name dnat`<br>` destination-zone untrust`<br>` action destination-nat static 2.2.2.2 interface GigabitEthernet 1/0/1` | `nat-policy`<br>` rule name dnat`<br>` destination-zone untrust`<br>` action destination-nat address 2.2.2.2 port 80` | 华为和华三的高级策略中均可定义目的地址转换。 |
| **NAT Server** (内部服务器发布) | **全局/接口下配置：**<br>`nat server protocol tcp global 2.2.2.2 www inside 192.168.1.100 www` | **系统视图下配置：**<br>`nat server 1 zone untrust global 2.2.2.2 inside 192.168.1.100` | 华三通常在接口下配置 `nat server`；华为多在系统视图下配置，并可直接绑定安全区域。 |
| **其他高级 NAT** |  |  |  |
| **NAT Hairpin**<br>(内网访问内网服务器) | `nat hairpin enable`<br>(在连接内网的接口上开启) | `nat hairpin enable`<br>(在域间或接口上开启) | 用于解决内部用户通过公网 IP 访问内网服务器的问题。 |
| **查看 NAT 会话与统计** | `display nat session`<br>`display nat statistics` | `display firewall session table`<br>`display nat session` | 排查 NAT 转换是否成功及连接建立情况。 |

---


> **核心避坑与配置逻辑提示：**
> 1. **NAT Server 差异：** 华三 SecPath 的 `nat server`（如发布内网 Web 服务）通常直接在连接公网的**接口视图**下敲；而华为 USG 系列习惯在**系统视图**下集中配置，并用关键字指定 `zone`。
> 2. **NAT 与安全策略联动：** 在华为 USG 防火墙上做完 DNAT（NAT Server）后，**安全策略**放行的目标 IP 必须是**内网私网真实 IP**（而非转换前的公网 IP）；华三 SecPath 在安全策略匹配上也遵循相同的“先NAT、后安全策略检查”或基于转换后地址的原则，需特别注意放行规则的编写。
> 
>