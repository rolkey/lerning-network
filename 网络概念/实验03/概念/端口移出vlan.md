把端口移出 VLAN 的具体命令，取决于你使用的**交换机品牌**以及该端口的**当前模式**（Access/Trunk/Hybrid）。操作前建议先用 `display port vlan`（华为）或 `show interfaces switchport`（思科/锐捷）确认一下端口现状。

### 🧩 华为交换机
*   **Access 端口**：执行 `undo port default vlan` 即可完全移除 VLAN 绑定；若只想恢复默认，可改用 `port default vlan 1` 。
*   **Trunk 端口**：使用 `port trunk allow-pass vlan remove <vlan-id>` 精准移除指定的 VLAN，其他 VLAN 不受影响 。
*   **Hybrid 端口**：需区分标签情况。移除不带标签的用 `port hybrid untagged vlan remove <vlan-id>`，移除带标签的则用 `port hybrid tagged vlan remove <vlan-id>` 。

### 🧩 思科 / 锐捷交换机
这两者的语法类似：
*   **Access 端口**：使用 `no switchport access vlan` 可将接口移出当前 VLAN 。
*   **Trunk 端口**：使用 `switchport trunk allowed vlan remove <vlan-id>` 将指定 VLAN 从允许列表中移除 。

### 💡 关键提醒
执行完移除操作后，别忘了在特权模式下执行 **`save`** 命令，否则交换机重启后配置会丢失 。

如果你能告诉我交换机的具体品牌和端口模式，我可以提供更精确的命令。