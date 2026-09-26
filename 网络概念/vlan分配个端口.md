给 VLAN 配 IP，**不是配在端口上，而是配在 Vlanif 接口上**。无论这个 VLAN 里有多少个端口，都只配一个 Vlanif 接口的 IP。

### 📌 核心逻辑

```
VLAN 10 里有 G0/0/1 和 G0/0/2 两个端口
        ↓
这两个端口都属于 VLAN 10（二层）
        ↓
Vlanif10 配一个 IP（三层）
        ↓
这个 IP 就是整个 VLAN 10 的网关
```

### ✅ 配置步骤

假设 VLAN 10 里有两个端口 `G0/0/1` 和 `G0/0/2`：

**第一步：创建 VLAN 并把两个端口都划进去**

```text
[Huawei]vlan 10
[Huawei-vlan10]quit

[Huawei]interface GigabitEthernet0/0/1
[Huawei-GigabitEthernet0/0/1]port link-type access
[Huawei-GigabitEthernet0/0/1]port default vlan 10
[Huawei-GigabitEthernet0/0/1]quit

[Huawei]interface GigabitEthernet0/0/2
[Huawei-GigabitEthernet0/0/2]port link-type access
[Huawei-GigabitEthernet0/0/2]port default vlan 10
[Huawei-GigabitEthernet0/0/2]quit
```

**第二步：给 Vlanif10 配一个 IP**

```text
[Huawei]interface Vlanif 10
[Huawei-Vlanif10]ip address 192.168.10.1 24
[Huawei-Vlanif10]quit
```

**只配这一个 IP**，VLAN 10 里的两个端口就都能用了。

### 🔍 验证

```text
display ip interface brief
```

应该看到：

```
Interface         IP Address/Mask      Physical   Protocol
Vlanif10          192.168.10.1/24      up         up
```

Vlanif10 是 up 的条件是：**VLAN 10 里至少有一个物理端口是 up 的**。你有两个端口，只要有一个连着设备且 up，Vlanif10 就会 up。

### ⚠️ 常见误区

| 错误想法 | 正确理解 |
|---|---|
| 每个端口都要配一个 IP | 一个 VLAN 只配一个 Vlanif IP |
| 两个端口配两个不同 IP | 同一 VLAN 内所有端口共享同一个网关 IP |
| 端口上直接配 IP | 二层口不能配 IP，必须通过 Vlanif |

### 💡 如果两个端口要属于不同网段

那就得分成两个 VLAN：

```text
VLAN 10 → G0/0/1 → Vlanif10 → 192.168.10.1/24
VLAN 20 → G0/0/2 → Vlanif20 → 192.168.20.1/24
```

每个 VLAN 一个 Vlanif，各自配各自的 IP。