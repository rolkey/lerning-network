想让 VLAN 10 和 VLAN 20 进行“部分通讯”，核心思路是：**先让两个 VLAN 能互通（通过三层路由），再用 ACL 精确控制哪些流量可以过、哪些不能过。**

结合你目前的 SW1 配置，具体分三步走。

### 🔗 第一步：确保 VLAN 10 和 VLAN 20 能互通

你的 SW1 已经有 VLANIF 10 的地址 `10.1.12.2`，现在需要补上 VLAN 20 的网关，让两个网段具备三层互通的基础。

```text
system-view
vlan batch 20

interface Vlanif20
 ip address 10.1.20.1 255.255.255.0
quit
```

同时，**SW1 和 SW2 之间的 Trunk 口**必须放行 VLAN 20（你之前已经加上了），并且接 PC2 或 VLAN 20 设备的接口要划入 VLAN 20。

### 🛡️ 第二步：用高级 ACL 定义“部分通讯”的规则

“部分通讯”的含义由你来定。比如常见的限制场景：

*   **允许 ping，禁止其他**：VLAN 10 可以 ping 通 VLAN 20，但无法访问其 HTTP 或 FTP 服务。
*   **只允许访问特定服务器**：VLAN 10 可以访问 VLAN 20 里的一台服务器，但不能访问其他设备。

假设需求是：**允许 VLAN 10 访问 VLAN 20 的 10.1.20.100 这台服务器的 80 端口（HTTP），其他流量全部拒绝。**

创建高级 ACL 3000：

```text
acl number 3000
 rule 5 permit tcp source 10.1.12.0 0.0.0.255 destination 10.1.20.100 0 destination-port eq 80
 rule 10 deny ip source 10.1.12.0 0.0.0.255 destination 10.1.20.0 0.0.0.255
quit
```

**配置要点**：
*   规则 `5` 是精确匹配，先放行特定流量。
*   规则 `10` 是宽泛拒绝，堵住其他所有去往 VLAN 20 的流量。
*   最后隐含的 `deny` 会挡住不匹配任何规则的流量，所以规则顺序至关重要。

### ⚙️ 第三步：通过流策略将 ACL 应用到 VLAN 10 的入口

ACL 定义好了，但不会自动生效。最精准的办法是在 **VLAN 10 的流量进入 SW1 的三层接口时**进行过滤。

```text
traffic classifier vlan10-in operator and
 if-match acl 3000
quit

traffic behavior vlan10-in
 permit
quit

traffic policy vlan10-in
 classifier vlan10-in behavior vlan10-in
quit

interface Vlanif10
 traffic-policy vlan10-in inbound
quit
```

**为什么绑定到 `Vlanif10` 的 `inbound`？**
因为 VLAN 10 的 PC 去访问 VLAN 20，流量会先进入 SW1 的 `Vlanif10` 接口，然后才发生三层转发。在这里拦截，效率最高，也最清晰。

### 🔍 验证与排查

配置完成后，在 SW1 上检查：

```text
display acl 3000
display traffic-policy applied-record
```

然后在 PC 上测试：
*   访问 `10.1.20.100` 的 Web 服务应该成功。
*   直接 ping `10.1.20.100` 或其他 VLAN 20 内的地址，应该失败（如果 ACL 只放行了 TCP 80）。

### 💡 最后确认一点

你想要的“**部分通讯**”，具体是指哪种情况？
1.  **只允许特定 IP 互访**（比如只允许 PC1 访问 PC2，其他不行）？
2.  **只允许特定协议**（比如只允许 ping，不允许文件共享）？
3.  **单向通讯**（比如 VLAN 10 能访问 VLAN 20，但 VLAN 20 不能反向访问）？

如果是第 3 种，还需要在 VLAN 20 的入口方向（`Vlanif20` 的 `inbound`）再配置一个反向的 ACL 来拒绝，否则回程流量可能被放行。