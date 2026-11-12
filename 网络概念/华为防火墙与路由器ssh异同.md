# 华为防火墙（FW1）与路由器（AR1）开启 SSH 服务的异同总结

## 一、相同点

| 项目 | 说明 |
|------|------|
| **基础命令** | 两者均使用 `stelnet server enable` 开启 SSH 服务（SSH 服务在华为设备上称 Stelnet） |
| **VTY 认证** | 均需在 `user-interface vty 0 4` 下配置 `authentication-mode aaa`，并 `protocol inbound ssh` |
| **AAA 用户** | 均需在 `aaa` 视图下创建本地用户，配置密码（cipher 加密）及 SSH 服务类型 |
| **用户级别** | 均需为用户配置级别（如 `level 15` / `privilege level 3`），级别影响可执行命令范围 |
| **安全加固** | 均建议关闭不安全的 SSH1.x（防火墙显式 `undo ssh server compatible-ssh1x enable`） |
| **SSH 算法** | 均可通过 `ssh server cipher` / `ssh server hmac` 指定加密与认证算法 |

## 二、不同点

### 1. 命令位置与语法差异

| 项目 | 防火墙 FW1 | 路由器 AR1 |
|------|-----------|-----------|
| 开启命令 | `stelnet server enable` | `stelnet server enable`（语法相同） |
| 用户创建视图 | `manager-user`（AAA 视图下） | `local-user`（AAA 视图下） |
| 服务类型 | `service-type web terminal` / `web ssh`（按角色区分） | `service-type ssh`（直接指定） |
| 用户级别 | `level 15`（manager-user 视图） | `privilege level 3`（local-user 视图） |
| 认证类型 | `ssh authentication-type default password` | 未显式配置，依赖 AAA 默认 |
| 算法配置 | 同时配置 `ssh server cipher/hmac` 和 `ssh client cipher/hmac` | 未配置（使用默认） |

### 2. 用户模型差异（核心区别）

- **防火墙**采用 **manager-user + role 绑定** 模型：
  - 用户通过 `manager-user` 创建，再 `bind manager-user xxx role system-admin` 绑定角色
  - 支持多角色（system-admin、device-admin、audit-admin 等），权限粒度更细
  - 用户可同时具备 `web`、`terminal`、`ssh` 多种服务类型
- **路由器**采用 **local-user + privilege level** 模型：
  - 直接在 `aaa` 视图下 `local-user sshuser service-type ssh`
  - 通过 `privilege level 3` 简单分级（0–15）

### 3. 接口服务绑定差异（关键区别）

这是防火墙与路由器在管理访问控制上最本质的差异，采用"双层控制"模型：

- **防火墙**：非管理接口默认**关闭**所有管理服务权限，必须在报文**入接口**上显式开启对应服务，否则即使安全策略放行也无法登录。
  ```
  interface GigabitEthernet1/0/1
   service-manage ssh permit     ← 必须显式开启，否则 SSH 无法接入
  ```
  - 管理口（如独立带外管理口）默认放开，无需此配置；但若管理口作为业务口加入区域（如 FW1 的 GE0/0/0 加入 trust 区），则同样受此限制
  - 判断依据是**报文进入的接口**：想从哪个接口登录，就必须在哪个接口上开启 `service-manage ssh permit`
  - 当前 FW1 中 GE0/0/0、GE1/0/1 已开启 SSH，但 **GE1/0/0（10.1.23.1）未开启**，从该网段 SSH 登录会失败

- **路由器**：无 `service-manage` 机制，只要 SSH 服务开启、接口 IP 可达、VTY 允许、AAA 用户正确，SSH 即可建立，属于"通到就能管"。

> ⚠️ 结论：防火墙是"**接口允许 + 策略放行**"两道关卡缺一不可；路由器仅需"可达 + VTY + AAA"。

### 4. 管理面与数据面隔离

- **防火墙**有独立的管理面概念：
  - `firewall dataplane to manageplane application-apperceive default-action drop`（数据面到管理面默认丢弃）
  - 结合接口 `service-manage` 实现精细管控
- **路由器**无管理面/数据面隔离概念，接口一旦可达且 VTY 允许即可 SSH 登录

### 5. 安全策略联动

- **防火墙** SSH 访问受 **安全策略（security-policy）** 约束：
  - 需放行 `source-zone → destination-zone local` 的 SSH 流量（当前配置中 `untrust_to_local` 仅放行 icmp，**未放行 SSH**，需额外加规则）
- **路由器**无区域安全策略概念，VTY 配置即可

### 6. Web 管理关联

- **防火墙** SSH 与 Web 管理同属管理面，配置了 `web-manager security version tlsv1.1 tlsv1.2`，且 `user-manage web-authentication security port 8887`
- **路由器** SSH 与 HTTP 管理分离，`local-user admin service-type http` 单独指定

### 7. 其他细节

| 项目 | 防火墙 FW1 | 路由器 AR1 |
|------|-----------|-----------|
| SSH 版本兼容 | 显式 `undo ssh server compatible-ssh1x enable` | 未显式配置 |
| Telnet | `undo telnet server enable` 显式关闭 | 未提及 |
| 登录横幅 | `banner enable` | `header login information "..."` |
| 密码策略 | `password-policy level high` | 无 |

## 三、配置要点对比（最小可用配置）

**防火墙最小 SSH 配置：**
```
stelnet server enable
ssh authentication-type default password
aaa
 manager-user test
  password cipher xxx
  service-type ssh
  level 15
 bind manager-user test role system-admin
user-interface vty 0 4
 authentication-mode aaa
 protocol inbound ssh
# 三道关卡缺一不可：
# ① 入接口开启服务：interface xxx → service-manage ssh permit
# ② 安全策略放行：source-zone → destination-zone local 的 SSH
# ③ VTY + AAA 用户正确
```

**路由器最小 SSH 配置：**
```
stelnet server enable
aaa
 local-user test password cipher xxx
 local-user test privilege level 3
 local-user test service-type ssh
user-interface vty 0 4
 authentication-mode aaa
 protocol inbound ssh
```

## 四、总结

> 两者开启 SSH 的**基础流程一致**（stelnet enable → AAA 用户 → VTY 认证），但管控模型差异显著：
>
> - **防火墙**采用"**接口 service-manage + 安全策略 + manager-user/role**"三层精细化管控，管理面与数据面隔离，任何一环缺失都会导致 SSH 登录失败；
> - **路由器**采用传统的"**local-user + privilege level**"模型，无接口服务绑定与区域策略概念，配置更直接、约束更少，属于"通到就能管"。
>
> ⚠️ 当前 FW1 需注意两点：① `security-policy` 中**未放行 SSH 到 local 的规则**（`untrust_to_local` 仅放行 icmp）；② **GE1/0/0（10.1.23.1）未配置 `service-manage ssh permit`**。若需从 untrust 侧或 10.1.23.0/24 网段 SSH 登录，两处均需补充。