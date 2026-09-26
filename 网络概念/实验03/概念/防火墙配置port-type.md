在 USG6000V1 防火墙上配置 VLAN 和接口链路类型，有一个**非常关键的前提**：必须先把接口从三层模式切换为二层模式。防火墙接口默认是三层接口（有 IP 地址），只有变成二层接口后，才能像交换机那样配置 `link-type` 和 VLAN。

### 🔧 前提：将接口切换为二层模式

在系统视图下进入接口，执行 `portswitch` 命令：
```
[USG6000V1] interface GigabitEthernet 1/0/0
[USG6000V1-GigabitEthernet1/0/0] portswitch
```
执行后，该接口就变成了二层交换接口，原来的 IP 地址配置会被清除。

### 📋 配置 Link-type 和 VLAN

切换为二层模式后，配置命令就和华为交换机完全一致了：

**配置 Access 接口（连接终端）**
```
[USG6000V1-GigabitEthernet1/0/0] port link-type access
[USG6000V1-GigabitEthernet1/0/0] port default vlan 100
```
这样该接口就属于 VLAN 100，连接的终端无需识别 VLAN 标签。

**配置 Trunk 接口（连接交换机）**
```
[USG6000V1-GigabitEthernet1/0/0] port link-type trunk
[USG6000V1-GigabitEthernet1/0/0] port trunk allow-pass vlan 100 200
```
如果需要放行所有 VLAN，可以使用 `port trunk allow-pass vlan all`；如果该 Trunk 接口的默认 VLAN 需要修改，可以额外配置 `port trunk pvid vlan <id>`。

### 💡 验证配置

配置完成后，可以用 `display this` 查看接口当前配置，确认 `portswitch`、`port link-type` 和 VLAN 相关命令已生效。

需要注意：配置 VLAN 后，如果还需要该 VLAN 内的终端能够互相通信或访问外网，通常需要创建对应的 `VLANIF` 接口并配置 IP 地址，或者将该 VLAN 的流量通过安全策略进行放行。