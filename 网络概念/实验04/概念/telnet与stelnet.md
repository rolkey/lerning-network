**`stelnet server enable` 就是打开华为交换机的 SSH 服务端功能**（华为把 SSH 服务端称为 **STelnet**，即 Secure Telnet），让设备能够接受来自其他终端的 SSH 登录请求。

### 具体含义

- **STelnet** = Secure Telnet = 基于 SSH 的安全远程登录
- 执行这条命令后，交换机会监听 **22 端口**，接受 SSH 客户端连接
- 不执行的话，即使 VTY 和 AAA 都配好了，SSH 也连不上

### 和 Telnet 的对应关系

| 功能 | Telnet（明文） | SSH（加密） |
|---|---|---|
| 开启服务端 | `telnet server enable` | `stelnet server enable` |
| 登录命令 | `telnet 192.168.1.1` | `stelnet 192.168.1.1` |
| 默认端口 | 23 | 22 |
| 安全性 | 明文传输，不安全 | 加密传输，安全 |
| VTY 放行协议 | `protocol inbound telnet` | `protocol inbound ssh` |

### 完整配置 SSH 登录还需要几步

光开服务端还不够，SSH 比 Telnet 多一个**密钥对**的要求：

```
# 1. 开启 STelnet 服务端
stelnet server enable

# 2. 生成 RSA 密钥对（SSH 加密必需）
rsa local-key-pair create

# 3. VTY 放行 SSH 协议
user-interface vty 0 4
 authentication-mode aaa
 protocol inbound ssh
 quit

# 4. 配置 AAA 用户，服务类型要包含 ssh
aaa
 local-user admin password cipher 你的密码
 local-user admin privilege level 15
 local-user admin service-type ssh
 quit

# 5. 配置用户的 SSH 登录方式
ssh user admin authentication-type password
ssh user admin service-type stelnet
```

### 关键区别

- **Telnet**：只需开服务 + 配 VTY + 配 AAA 用户即可
- **SSH**：额外需要**生成密钥对**、**指定用户 SSH 认证方式**（`ssh user ... authentication-type`）

### 安全建议

现在华为新版本设备**默认可能只开 SSH、不开 Telnet**，甚至部分版本 Telnet 服务端默认就是关闭的。生产环境强烈建议：

```
undo telnet server enable   # 关掉不安全的 Telnet
stelnet server enable       # 只留 SSH
```

**一句话：`stelnet server enable` = 打开 SSH 服务端的开关，对应 Telnet 的 `telnet server enable`，但 SSH 还需要额外配密钥和用户认证方式。**