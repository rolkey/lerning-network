# vlan间安全策略

下面用你当前的实验背景来举例：

- VLAN 10：`10.1.12.0/24`，网关 `10.1.12.2`（SW1 的 Vlanif10）
- VLAN 20：`10.1.20.0/24`，网关 `10.1.20.1`（SW1 的 Vlanif20）
- SW1 是三层交换机，负责 VLAN 10 和 VLAN 20 之间的路由

三种情况都通过 **高级 ACL 3000 + 流策略** 实现。

---

## 一、只允许特定 IP 互访

### 需求

只允许：

- PC1：`10.1.12.3` 访问 PC2：`10.1.20.4`
- 其他 VLAN 10 和 VLAN 20 之间的流量全部拒绝

### ACL 配置

```text
acl number 3000
 rule 5 permit ip source 10.1.12.3 0 destination 10.1.20.4 0
 rule 10 deny ip source 10.1.12.0 0.0.0.255 destination 10.1.20.0 0.0.0.255
quit
```

说明：

- `rule 5`：只放行 `10.1.12.3` 到 `10.1.20.4` 的 IP 流量
- `rule 10`：拒绝 VLAN 10 其他地址访问 VLAN 20
- 高级 ACL 默认最后隐含 `deny ip`，所以其他流量也会被拒绝

### 流策略配置

```text
traffic classifier c1 operator and
 if-match acl 3000
quit

traffic behavior b1
 permit
quit

traffic policy p1
 classifier c1 behavior b1
quit
```

### 应用到 Vlanif10 入方向

```text
interface Vlanif10
 traffic-policy p1 inbound
quit
```

### 效果

- `10.1.12.3` ping `10.1.20.4`：通
- `10.1.12.3` ping `10.1.20.5`：不通
- `10.1.12.4` ping `10.1.20.4`：不通

### 注意

如果希望 PC2 也能主动访问 PC1，还需要在 `Vlanif20` 入方向放行反向流量，或者把 ACL 写成双向匹配。

---

## 二、只允许特定协议

### 需求

允许 VLAN 10 和 VLAN 20 之间：

- 允许 ICMP（ping）
- 允许 HTTP（TCP 80）
- 拒绝其他所有协议

### ACL 配置

```text
acl number 3000
 rule 5 permit icmp source 10.1.12.0 0.0.0.255 destination 10.1.20.0 0.0.0.255
 rule 10 permit tcp source 10.1.12.0 0.0.0.255 destination 10.1.20.0 0.0.0.255 destination-port eq 80
 rule 20 deny ip source 10.1.12.0 0.0.0.255 destination 10.1.20.0 0.0.0.255
quit
```

说明：

- `rule 5`：允许 ping
- `rule 10`：允许访问 HTTP 80 端口
- `rule 20`：拒绝其他 IP 流量

### 流策略配置

```text
traffic classifier c2 operator and
 if-match acl 3000
quit

traffic behavior b2
 permit
quit

traffic policy p2
 classifier c2 behavior b2
quit
```

### 应用到 Vlanif10 入方向

```text
interface Vlanif10
 traffic-policy p2 inbound
quit
```

### 效果

- VLAN 10 ping VLAN 20：通
- VLAN 10 访问 VLAN 20 的 Web：通
- VLAN 10 访问 VLAN 20 的 FTP、Telnet、共享文件夹：不通

### 注意

如果只允许 ping，不允许其他，把 `rule 10` 删掉即可：

```text
acl number 3000
 rule 5 permit icmp source 10.1.12.0 0.0.0.255 destination 10.1.20.0 0.0.0.255
 rule 10 deny ip source 10.1.12.0 0.0.0.255 destination 10.1.20.0 0.0.0.255
quit
```

---

## 三、单向通讯

### 需求

只允许：

- VLAN 10 主动访问 VLAN 20
- VLAN 20 不能主动访问 VLAN 10

也就是说：

- `10.1.12.3` 可以 ping `10.1.20.4`
- `10.1.20.4` 不能 ping `10.1.12.3`

### 关键点

单向通讯不能只在一个方向配 ACL，因为：

- VLAN 10 访问 VLAN 20 的请求包从 `Vlanif10` 入
- VLAN 20 返回给 VLAN 10 的回应包从 `Vlanif20` 入

如果只在 `Vlanif10` 入方向放行，回程包可能被 `Vlanif20` 的默认规则挡住，导致 ping 不通。

所以要做两件事：

1. 在 `Vlanif10` 入方向允许 VLAN 10 → VLAN 20
2. 在 `Vlanif20` 入方向拒绝 VLAN 20 → VLAN 10 的新建连接，但允许回程流量

华为 ACL 可以用 `established` 匹配已建立的 TCP 连接，但 ICMP 没有类似选项。  
更简单的做法是：

- 在 `Vlanif10` 入方向放行 VLAN 10 → VLAN 20
- 在 `Vlanif20` 入方向放行 VLAN 20 → VLAN 10 的**回程流量**
- 但回程流量和主动访问流量很难完全区分

所以实际实验中，常用方法是：

### 方法一：用 ACL 拒绝 VLAN 20 主动访问，但允许已建立的会话

对于 TCP，可以用：

```text
acl number 3001
 rule 5 permit tcp source 10.1.20.0 0.0.0.255 destination 10.1.12.0 0.0.0.255 established
 rule 10 deny ip source 10.1.20.0 0.0.0.255 destination 10.1.12.0 0.0.0.255
quit
```

然后应用到 `Vlanif20` 入方向：

```text
interface Vlanif20
 traffic-policy p3 inbound
quit
```

其中 `established` 表示只允许已经建立连接的 TCP 回程包。

### 方法二：实验里更简单的单向控制

如果只是演示“单向 ping 不通”，可以这样：

#### Vlanif10 入方向：允许 VLAN 10 → VLAN 20

```text
acl number 3000
 rule 5 permit ip source 10.1.12.0 0.0.0.255 destination 10.1.20.0 0.0.0.255
quit

traffic classifier c3 operator and
 if-match acl 3000
quit

traffic behavior b3
 permit
quit

traffic policy p3
 classifier c3 behavior b3
quit

interface Vlanif10
 traffic-policy p3 inbound
quit
```

#### Vlanif20 入方向：拒绝 VLAN 20 → VLAN 10

```text
acl number 3001
 rule 5 deny ip source 10.1.20.0 0.0.0.255 destination 10.1.12.0 0.0.0.255
quit

traffic classifier c4 operator and
 if-match acl 3001
quit

traffic behavior b4
 deny
quit

traffic policy p4
 classifier c4 behavior b4
quit

interface Vlanif20
 traffic-policy p4 inbound
quit
```

### 效果

- `10.1.12.3` ping `10.1.20.4`：可能通，也可能因为回程被拒绝而不通
- `10.1.20.4` ping `10.1.12.3`：不通

### 更准确的单向控制建议

如果要做严格的单向 TCP 访问，比如：

- VLAN 10 可以访问 VLAN 20 的 Web
- VLAN 20 不能访问 VLAN 10 的 Web

推荐：

1. `Vlanif10` 入方向：放行 VLAN 10 → VLAN 20 的 TCP 80
2. `Vlanif20` 入方向：放行 `established` 回程，拒绝其他

```text
acl number 3001
 rule 5 permit tcp source 10.1.20.0 0.0.0.255 destination 10.1.12.0 0.0.0.255 established
 rule 10 deny ip source 10.1.20.0 0.0.0.255 destination 10.1.12.0 0.0.0.255
quit
```

这样：

- VLAN 10 主动访问 VLAN 20 的 Web：通
- VLAN 20 主动访问 VLAN 10 的 Web：不通

---

## 四、三种情况对比

| 情况 | ACL 关键规则 | 应用位置 | 效果 |
|---|---|---|---|
| 只允许特定 IP 互访 | `permit ip source 10.1.12.3 destination 10.1.20.4` | Vlanif10 入 | 只有指定 IP 能通 |
| 只允许特定协议 | `permit icmp`、`permit tcp ... eq 80` | Vlanif10 入 | 只有 ping 和 HTTP 能通 |
| 单向通讯 | 正向放行 + 反向 `established` 或 deny | Vlanif10 入 + Vlanif20 入 | 一个方向能主动访问，另一个方向不能 |

---

## 五、通用配置模板

### 1. 创建 ACL

```text
acl number 3000
 rule 5 permit ...
 rule 10 deny ...
quit
```

### 2. 创建流分类

```text
traffic classifier c1 operator and
 if-match acl 3000
quit
```

### 3. 创建流行为

```text
traffic behavior b1
 permit
quit
```

### 4. 创建流策略

```text
traffic policy p1
 classifier c1 behavior b1
quit
```

### 5. 应用到接口

```text
interface Vlanif10
 traffic-policy p1 inbound
quit
```

### 6. 查看

```text
display acl 3000
display traffic-policy applied-record
display current-configuration interface Vlanif10
```

---

## 六、总结

- **只允许特定 IP 互访**：ACL 里精确匹配源和目的 IP，其他拒绝。
- **只允许特定协议**：ACL 里匹配协议和端口，比如 `icmp`、`tcp eq 80`，其他拒绝。
- **单向通讯**：正向放行，反向用 `established` 或直接拒绝，并注意回程流量。

如果你告诉我你具体想限制哪两个 IP、哪种协议、哪个方向，我可以直接帮你写出对应的 ACL 和流策略配置。