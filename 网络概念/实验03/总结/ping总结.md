# ping总结

## 操作过程

- 配置安全策略，允许ICMP协议通过区域

```bash
[AC6605]security-policy
[AC6605-security-policy]rule 2000
[AC6605-security-policy-rule-2000]action permit
[AC6605-security-policy-rule-2000]service icmp
[AC6605-security-policy-rule-2000]source-zone trust
[AC6605-security-policy-rule-2000]destination-zone untrust
[AC6605-security-policy-rule-2000]quit
[AC6605-security-policy]quit
```
**注意：** action permit与service icmp不分先后顺序

- 把入和出的端口加入区域

```bash
[AC6605]security-zone trust
[AC6605-security-zone-trust]add interface GigabitEthernet 1/0/0
[AC6605]security-zone untrust
[AC6605-security-zone-untrust]add interface GigabitEthernet 1/0/1
```

- 配置端口 service-manage ping permit

```bash
[fw1]interface GigabitEthernet 1/0/0
[fw1-GigabitEthernet1/0/0]service-manage ping permit
```

## 注意细节

- 通过防火墙/核心交换机ping外网时(多级设备访问)，要注意配置回程路由，否则会产生ping不通的问题

```bash
# 在防火墙上配置回程路由，注意回程路由是到交换机的连接接口
system-view
ip route-static 192.168.21.0 255.255.255.0 192.168.10.2
ip route-static 192.168.22.0 255.255.255.0 192.168.10.2
```

- 检查回程路由

```bash
display ip routing-table 192.168.21.3
```

**注意**：如果回程不存在，要补路由
