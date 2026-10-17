在 eNSP 中模仿“绑定 MAC 地址才允许访问网络”，最直接有效的方式是配置交换机的**端口安全（Port Security）** 功能。核心思路是：让交换机端口只允许特定的 MAC 地址通信，其他未知设备接入将被丢弃数据包或关闭端口。

下面是一套可以直接在 eNSP 里复现的完整方案。

### 实验拓扑与准备
1.  **拖拽设备**：放置一台二层交换机（如 **S3700**）和两台 PC（PC1、PC2）。
2.  **连线**：PC1 连接交换机 **Ethernet 0/0/1**，PC2 连接 **Ethernet 0/0/2**。
3.  **配置 IP**：给 PC1 和 PC2 配置同一网段的 IP（例如 192.168.1.1/24 和 192.168.1.2/24）。
4.  **获取 MAC**：在 PC 的命令行输入 `ipconfig /all`，记下各自的 MAC 地址（eNSP 中通常以 `5489-98xx-xxxx` 开头）。

### 核心配置步骤（在交换机 CLI 中执行）

- **自动绑定**

这里演示“仅允许 PC1 访问，且绑定在 0/0/1 端口”的配置。关键命令是 `port-security` 配合 `sticky` 或静态绑定。

```text
<Huawei> system-view
[Huawei] sysname SW1

# 1. 进入连接 PC1 的接口（以 Eth0/0/1 为例）
[SW1] interface Ethernet 0/0/1
[SW1-Ethernet0/0/1] port link-type access   # 端口安全通常仅支持 Access 模式
[SW1-Ethernet0/0/1] port-security enable    # 开启端口安全

# 2. 配置 MAC 地址绑定与违规动作
[SW1-Ethernet0/0/1] port-security max-mac-num 1          # 限制该端口最多学习 1 个 MAC
[SW1-Ethernet0/0/1] port-security protect-action restrict # 违规时丢弃报文并告警（推荐）
[SW1-Ethernet0/0/1] port-security mac-address sticky      # 将当前学习到的 MAC 转为 Sticky（粘性）绑定
[SW1-Ethernet0/0/1] quit
```

**说明**：`sticky` 模式非常方便，它会自动把该端口当前连接设备的 MAC 地址“粘”住并保存。配置后，如果此时 PC1 在线，它的 MAC 就被自动绑定了；如果不在线，PC1 第一次接入时也会自动绑定。如果你想手动指定，也可以用 `port-security mac-address sticky xxxx-xxxx-xxxx vlan 1`。

- **手动绑定**：把上一轮配置里的 sticky 那行去掉，换成指定 PC1 的具体 MAC

[SW1] interface Ethernet 0/0/1
[SW1-Ethernet0/0/1] port-security enable
[SW1-Ethernet0/0/1] port-security max-mac-num 1
[SW1-Ethernet0/0/1] port-security protect-action restrict
[SW1-Ethernet0/0/1] port-security mac-address 5489-98xx-xxxx vlan 1   # 换成PC1真实的MAC

### 验证效果
1.  **正常测试**：用 PC1 去 `ping` PC2，应该可以通（因为 PC1 的 MAC 已被端口允许）。
2.  **非法接入测试**：断开 PC1，把 **PC2** 插到交换机的 **Ethernet 0/0/1** 端口上。此时用 PC2 去 `ping` 其他设备（或尝试访问网络），应该会失败。这是因为端口已经绑定了 PC1 的 MAC，PC2 的 MAC 不同，触发了 `restrict` 动作，数据包被丢弃。
3.  **查看绑定表**：在交换机上执行 `display mac-address sticky` 或 `display port-security interface Ethernet 0/0/1`，可以看到绑定的 MAC 地址和安全状态。

### 备选方案：如果你需要严格区分“允许/拒绝”
如果你的需求不是“绑定谁”，而是想明确配置**仅允许特定 MAC 通过**（白名单），可以使用 ACL 来实现。不过对于“模仿允许访问”的实验目的，**端口安全（Sticky MAC）** 是华为交换机上最标准、最贴合“绑定”语义的做法，它能防止用户私自更换电脑或挪动网线位置。

---
**注意**：在 eNSP 中，如果配置后 PC1 无法 ping 通，请检查 PC1 是否确实连接在 Eth0/0/1，且该端口必须是 Access 模式。另外，如果你先配置了 `sticky` 但当时端口是空的，需要让 PC1 先发一个包（如 ping 一下网关或 PC2）触发学习，绑定才会生效。