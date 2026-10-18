### 华三 vs 华为命令对比

为了让你更清楚差异，这里列出几个关键区别：

| 功能 | 华三 (H3C) | 华为 (Huawei) |
| :--- | :--- | :--- |
| **进入 VLAN 三层接口** | `interface Vlan-interface 10` | `interface Vlanif 10` |
| **接口命名关键词** | `Vlan-interface` | `Vlanif` |
| **物理口转三层** | `port link-mode route` | `undo portswitch` |
| **查看路由表** | `display ip routing-table` | `display ip routing-table` |

所以在华三设备上，请记住 **`Vlan-interface`** 这个关键词，不要受华为 `Vlanif` 的影响。