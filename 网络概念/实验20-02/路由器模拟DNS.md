## 一句话结论

AR2200（V200R003C00）**没有独立的"DNS 服务器软件"**，所谓"模拟 DNS 服务器"= 开启 **DNS Proxy/Relay**（让设备监听 UDP 53、替客户端应答/转发）+ 用 **`ip host` 配静态"A 记录"** + 用 **DHCP 把 DNS 地址指向路由器自己**。客户端眼中，这台 AR 就是一台 DNS 服务器。

> 顺带澄清：网上有些教程写 `dns server enable`，那是其他厂商的写法；华为 VRP（含 V200R003C00）对应命令是 `dns proxy enable` / `dns relay enable`。

---

## 完整配置（可直接照抄）

假设：内网口 `GE0/0/0 = 192.168.1.1/24`，要把 `www.lab.com → 10.1.1.10`，未知域名转发到上游 `114.114.114.114`。

```text
system-view
sysname AR-DNS
#
dhcp enable
#
interface GigabitEthernet0/0/0
 ip address 192.168.1.1 255.255.255.0
 dhcp select interface
 dhcp server dns-list 192.168.1.1        // 关键：把"我自己"作为 DNS 下发给 PC
 quit
#
ip host www.lab.com 10.1.1.10            // 本地静态解析，相当于 A 记录
ip host ftp.lab.com 10.1.1.20
#
dns proxy enable                         // 关键：开启 DNS 代理，监听 53 端口
dns resolve                              // 允许本机做动态解析/向上游查询
dns server 114.114.114.114               // 上游 DNS（运营商或公网 DNS）
#
ip route-static 0.0.0.0 0.0.0.0 <下一跳>   // 保证路由器能到上游 DNS
```

如果习惯用全局地址池，把接口下的 DHCP 换成：

```text
ip pool lan
 network 192.168.1.0 mask 255.255.255.0
 gateway-list 192.168.1.1
 dns-list 192.168.1.1                    // 同样指向路由器自己
 quit
interface GigabitEthernet0/0/0
 undo dhcp select interface
 dhcp select global
```

静态表项命中时，Proxy 直接回答案给客户端，不再往上转发；未命中的域名才由 `dns resolve` + `dns server` 转发到上游。 静态表项一个主机名只能对应一个 IPv4 地址，重复配置以最后一次为准。

---

## 验证

在路由器上：

```text
display ip host                          // 静态"记录"是否生效
display dns dynamic-host                 // 转发生成的缓存
display current-configuration | include dns
ping www.lab.com                         // 设备自身解析是否正常
```

在 PC 上（DHCP 获取或手工把 DNS 设为 `192.168.1.1`）：

```text
nslookup www.lab.com 192.168.1.1
ping www.lab.com
```

---

## 几种变体玩法

- **纯离线实验、不需要上公网**：只留 `ip host` + `dns proxy enable`，不配 `dns server`。此时只有表内域名能解析，其余会超时。
- **想让任意域名都解析到固定地址**：V200R003 时代支持 `dns spoofing <ip>`（典型用于拨号/实验），所有未命中查询统一回指定 IP。
- **只做转发不做缓存**：部分版本可用 `dns relay enable` 代替 `dns proxy enable`，二者**不要同时开**；老版本以 `dns proxy enable` 为主，用 `dns proxy enable ?` 确认支持情况。
- **eNSP 里更简单的方式**：如果拓扑允许放一台 Server 设备，直接用 Server 自带的 DNS 服务配多条主机记录，AR 只负责 DHCP 下发 `dns-list <Server IP>`；用 AR 模拟更适合练"代理/转发/静态表"本身的逻辑。

---

## 容易踩的坑

1. **客户端 DNS 没改**：只开了代理但 PC 仍指向 8.8.8.8，请求根本不经过 AR——DHCP 的 `dns-list` 必须下发 `192.168.1.1`。
2. **缺到上游的路由/NAT**：`dns proxy enable` 开着、`dns server` 也配了，但路由器出不了网，表现为"表名能解析、外网域名全超时"。
3. **记录类型有限**：`ip host` 只有"域名→单个 IPv4"，没有 CNAME/MX/泛域名/区域文件；要真正的权威 DNS（SOA、NS、多记录、递归缓存），还是得用服务器。
4. **UDP 53 被拦**：接口 ACL、防火墙策略或安全功能可能丢弃 DNS 报文，排查时可用 `display current-configuration | include dns` 复核配置。
5. **版本细节**：V200R003C00 属较老版本，个别加固/绑定类命令（如 `dns server bind local-ip` / `dns proxy-client ipv4 source-ip`）可能不存在；这些只是可选的安全绑定项，不影响上面的主体配置。

如果你告诉我拓扑（真机还是 eNSP、内网段、需要解析哪些域名、是否有上游 DNS），我可以直接给你一份按你接口编号排好的最终配置。