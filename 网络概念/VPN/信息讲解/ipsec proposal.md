* 原始信息

```bash
[FW2]display ipsec proposal
2026-11-03 08:11:58.380 

Number of proposals: 1

IPSec proposal name: 1                                
 Encapsulation mode: Tunnel                                
 Transform         : esp-new
 ESP protocol      : Authentication SHA2-HMAC-256 
                     Encryption AES-256
```

---

这段输出是华为防火墙（FW2）上查看 **IPSec 提议（proposal）** 的结果，下面逐项解读。

## 整体信息

```
Number of proposals: 1
```
当前设备上共配置了 **1 个** IPSec 提议。

---

## 提议详细内容

```
IPSec proposal name: 1
```
提议名称为 `1`（这是管理员自定义的命名，实际部署中建议用有意义的名称，如 `proposal-sha256-aes256`，便于维护）。

```
Encapsulation mode: Tunnel
```
**封装模式为隧道模式（Tunnel）**。这是站点到站点（Site-to-Site）VPN 的标准模式：原始 IP 报文会被整体封装进新的 IP 包头中，隐藏内部地址，适合网关之间的 VPN 连接。另一种模式是传输模式（Transport），多用于端到端主机通信。

```
Transform : esp-new
```
**安全协议使用 ESP（封装安全载荷）**，`esp-new` 表示采用新版 ESP 实现。ESP 同时提供**加密**和**认证**，是 IPSec VPN 最常用的协议（相比 AH 协议，ESP 支持加密且能穿越 NAT）。

```
ESP protocol : Authentication SHA2-HMAC-256
               Encryption AES-256
```
ESP 的两个核心算法：

| 功能 | 算法 | 说明 |
|------|------|------|
| 认证（Authentication） | SHA2-HMAC-256 | 使用 HMAC-SHA2-256 做完整性校验和报文认证，安全性高于 SHA1-HMAC |
| 加密（Encryption） | AES-256 | 使用 256 位密钥的 AES 对称加密，属于高强度加密算法 |

---

## 小结与注意事项

1. **该提议的安全强度较高**：AES-256 + SHA2-HMAC-256 属于当前推荐的组合，符合等保/合规场景对强算法的要求。
2. **对端必须匹配**：IPSec 协商（IKE 第二阶段）要求通信双方提议中的**封装模式、协议、认证算法、加密算法完全一致**，否则 SA 建立失败。请确认对端设备的 IPSec 提议配置与此一致。
3. **仅显示提议参数**：此命令只展示提议本身的算法配置，不显示 IKE 参数（如 IKE 版本、DH 组、预共享密钥等），也不显示 SA 是否已建立。若需确认隧道状态，可进一步查看 `display ipsec sa` 和 `display ike sa`。

如果你需要，我可以帮你对照对端配置做一次匹配性检查，或说明如何查看 IPSec SA 建立状态。