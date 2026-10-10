# 域安全问题

## 问题

```text
[FW1-policy-security]di this
2026-10-10 07:21:19.000 
#
security-policy
 rule name cloud_ping_fw
  source-zone untrust
  destination-zone local
  source-address 192.168.1.2 mask 255.255.255.255
  destination-address 192.168.1.1 mask 255.255.255.255
  service icmp
  action permit
 rule name trust_to_local_ping
  source-zone trust
  destination-zone local
  service icmp
  action permit
 rule name trust_to_untrust
  source-zone trust
  destination-zone untrust
  action permit
 rule name local_to_untrust_ping
  source-zone local
  destination-zone untrust
  service icmp
  action permit
#
return
[FW1-policy-security]q
[FW1]display zone
2026-10-10 07:21:45.780 
local
 priority is 100
 interface of the zone is (0):
#
trust
 priority is 85
 interface of the zone is (2):
    GigabitEthernet0/0/0
    GigabitEthernet1/0/1
#
untrust
 priority is 5
 interface of the zone is (1):
    GigabitEthernet1/0/0
#
dmz
 priority is 50
 interface of the zone is (0):
#

从策略来说，测试时，把local_to_untrust_ping删除掉后，GigabitEthernet1/0/0就ping不通另外一端，这是什么原因，GigabitEthernet1/0/0不是local呀
```

## 答案

你的疑惑很合理：`GigabitEthernet1/0/0` 确实不是 local 区域，它属于 **untrust** 区域。但关键在于，**从防火墙自身发起的 ping，源区域是 local，不是出接口所在的区域**。

### 为什么删除 `local_to_untrust_ping` 会导致 ping 不通

当你在防火墙 FW1 上执行 `ping 100.1.1.1` 时，这个 ICMP 请求的路径是：

```
源区域 local  →  目的区域 untrust  →  从 GigabitEthernet1/0/0 发出
```

防火墙的安全策略是**基于区域**的，不是基于接口的。它判断的是：

- **源区域**：谁发起的流量？→ 防火墙自己，属于 **local**
- **目的区域**：流量要去哪里？→ 去往 untrust 区域的 100.1.1.1

所以这条流量匹配的策略应该是 **`local → untrust`**，而不是 `untrust → local`。

你删除的 `local_to_untrust_ping` 正好是：

```
rule name local_to_untrust_ping
  source-zone local
  destination-zone untrust
  service icmp
  action permit
```

删掉它之后，`local → untrust` 方向的 ICMP 就没有任何 permit 策略了，默认动作是 **deny**，所以 ping 不通。

### 为什么 `GigabitEthernet1/0/0` 不是 local，却受 local 策略控制

这里要区分两个概念：

| 概念 | 含义 |
|------|------|
| **接口所属区域** | GigabitEthernet1/0/0 属于 untrust，这是接口的“归属” |
| **流量的源区域** | 防火墙自己发起的包，源区域是 local，与出接口无关 |

`GigabitEthernet1/0/0` 是 **untrust 区域的接口**，但它是这条流量的**出接口**，不是流量的**源**。流量的源是防火墙本身，所以源区域是 local。

### 对照你现有的策略

你现有的策略里：

- `cloud_ping_fw`：untrust → local，允许 192.168.1.2 ping 192.168.1.1（外部 ping 防火墙）
- `trust_to_local_ping`：trust → local，允许 trust 区域 ping 防火墙
- `local_to_untrust_ping`：local → untrust，允许防火墙 ping 外部 ← **你删掉的这条**
- `trust_to_untrust`：trust → untrust，允许 trust 区域访问 untrust

注意 `trust_to_untrust` 只允许 **trust 区域** 的流量去 untrust，它**不覆盖 local 发起的流量**。所以防火墙自己 ping 外部，必须靠 `local_to_untrust_ping` 放行。

### 总结

- `GigabitEthernet1/0/0` 是 untrust 接口，但防火墙自己发起的 ping，**源区域是 local**，目的区域是 untrust。
- 删除 `local_to_untrust_ping` 后，local → untrust 的 ICMP 没有 permit 策略，默认 deny，所以 ping 不通。
- 这不是因为 `GigabitEthernet1/0/0` 是 local，而是因为**流量的源是防火墙本身，属于 local 区域**。

如果你希望防火墙能 ping 通外部，需要保留 `local_to_untrust_ping`，或者把这条策略的 `service` 放宽（比如 `any`）以覆盖更多自发起流量。