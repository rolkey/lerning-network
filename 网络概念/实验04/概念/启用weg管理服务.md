# 启用华为USG6000V防火墙web-management服务

## 1. 启用web-management服务

1. 在防火墙的命令行界面下，执行以下命令，启用web-management服务。

```
system-view
firewall enable
web-manager enable
```

2. 执行命令display web-manager status，查看web-management服务的状态。

```
<USG6000V> display web-manager status
Web manager status: running
```

3. 在端口上启用服务

```
interface GigabitEthernet0/0/0
service-manage enable
service-manage https permit
```

