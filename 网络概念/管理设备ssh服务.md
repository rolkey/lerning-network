在华为 S5700 交换机（V200R001C00 版本）上，SSH 远程登录服务在华为 VRP 系统中通常被称为 **STelnet** 服务。以下是检查 SSH 服务状态以及开启并完整配置 SSH 服务的标准操作步骤。

### 一、 检查 SSH 服务是否已开启

在交换机的任意视图（用户视图 `<HUAWEI>` 或系统视图 `[HUAWEI]`）下，执行以下命令查看 SSH 服务器状态：

```text
<HUAWEI> display ssh server status
```
**查看回显信息：**
- 如果回显中包含 `STelnet server status: Enable`，说明 SSH 服务**已经开启**。
- 如果回显为 `Disable`，则说明 SSH 服务**未开启**。

此外，你还可以检查是否已经配置了 SSH 用户：
```text
<HUAWEI> display ssh user-information
```
如果有用户列表输出，说明之前已经配置过 SSH 用户；如果为空，则需要重新配置。

---

### 二、 开启并完整配置 SSH 服务

如果检查发现 SSH 服务未开启，或者你需要重新配置，请按照以下完整步骤操作（仅开启服务是不够的，还需配置密钥、用户和 VTY 线路）：

#### 1. 进入系统视图并开启 STelnet (SSH) 服务
```text
<HUAWEI> system-view
[HUAWEI] stelnet server enable
```
*(注：V200R001 版本中，开启 SSH 服务端功能的命令为 `stelnet server enable`)* [[35]]

#### 2. 生成本地密钥对（SSH 加密必需）
SSH 协议需要密钥对来进行加密通信，必须生成 RSA 或 DSA 密钥对：
```text
[HUAWEI] rsa local-key-pair create
```
*(系统会提示输入密钥长度，建议输入 `1024` 或 `2048`。长度越大越安全，但生成时间稍长)* [[28]]

#### 3. 配置 SSH 用户及其认证方式
创建一个 SSH 用户（例如用户名为 `admin`），并指定其认证方式为密码认证，服务类型为 STelnet：
```text
[HUAWEI] ssh user admin authentication-type password
[HUAWEI] ssh user admin service-type stelnet
```
*(如果需要简化配置，也可以执行 `ssh authentication-type default password` 将所有 SSH 用户默认设为密码认证)* [[28]]

#### 4. 配置 AAA 本地用户认证
进入 AAA 视图，为刚才创建的 SSH 用户设置密码、权限级别和服务类型：
```text
[HUAWEI] aaa
[HUAWEI-aaa] local-user admin password cipher YourPassword@123
[HUAWEI-aaa] local-user admin privilege level 15
[HUAWEI-aaa] local-user admin service-type ssh
[HUAWEI-aaa] quit
```
*(请将 `YourPassword@123` 替换为你实际需要的强密码)* [[30]]

#### 5. 配置 VTY 用户界面（允许 SSH 协议接入）
配置虚拟终端线路（VTY），允许 SSH 协议接入，并调用 AAA 认证：
```text
[HUAWEI] user-interface vty 0 4
[HUAWEI-ui-vty0-4] authentication-mode aaa
[HUAWEI-ui-vty0-4] protocol inbound ssh
[HUAWEI-ui-vty0-4] user privilege level 15
[HUAWEI-ui-vty0-4] quit
```
*(注意：V200R001 版本部分设备 VTY 默认协议可能是 Telnet，必须显式配置 `protocol inbound ssh` 或 `protocol inbound all` 才能允许 SSH 连接)* [[32]]

---

### 三、 验证配置结果

配置完成后，建议再次检查以确保配置生效：

1. 检查 SSH 服务状态：
   ```text
   [HUAWEI] display ssh server status
   ```
   *(确认 STelnet server status 为 Enable)* [[65]]

2. 检查 SSH 用户信息：
   ```text
   [HUAWEI] display ssh user-information
   ```
   *(确认能看到你配置的 admin 用户，且 State 为正常)* [[67]]

3. **保存配置**（非常重要，防止重启后丢失）：
   ```text
   [HUAWEI] return
   <HUAWEI> save
   ```

完成以上步骤后，你就可以使用 Xshell、SecureCRT 或 PuTTY 等客户端工具，选择 **SSH** 协议，输入交换机的管理 IP、用户名 `admin` 和对应密码进行远程登录了。