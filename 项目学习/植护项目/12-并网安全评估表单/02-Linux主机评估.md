# Linux 主机安全评估

> 对应原表 sheet：`二、安全评估(linux-x.x.x.x)`（70 行 × 20 列）
> 作业项 B1–B26，共 26 项；填表数量 = 本项目 Linux 主机台数（待确认，见"待填项"）

Linux 主机执行病毒查杀与安全加固。数据库与中间件**禁止 root 权限部署**。

本文分三部分：**第一节**基本信息、**第二节**保留原表作业项与逐项操作方法、**第三节**起为漏洞清单与参数差异。

> ## 逐项操作方法的来源
>
> 本表 Linux 操作方法**有完整现场实施文档可依**：
>
> - [AGCAVC1(198.120.0.206)_附件1/](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/README.md>) 提供了 **B1–B26 全部作业项**的 Linux 现场实施文档；
> - **注意 B26 不一致**：该目录的 B26 是"调度发令系统主机的特有加固"，与本表 B26"卸载 samba、删除默认路由"**不是同一项**，见第八节与第九节。
>
> ## ⚠️ 发行版差异（本表最大的落地风险）
>
> 原表命令**混用了 CentOS 7 与凝思 Linx（RHEL 6 系）两套体系**，直接照抄会失败：
>
> | 项目 | CentOS 7 系 | 凝思 Linx / RHEL 6 系 |
> | --- | --- | --- |
> | 服务管理 | `systemctl`、`systemctl \| grep active` | `chkconfig --list`、`service xxx start/stop` |
> | 防火墙 | `firewalld`（`iptables` 底层） | `iptables` |
> | 时间同步 | `chronyd`、`chronyc` | `ntpd`、`ntpdate` |
> | 日志 | `rsyslog`（`/etc/rsyslog.conf`） | `syslog`（`/etc/syslog.conf`） |
> | 路由持久化 | `nmcli` / `/etc/sysconfig/network-scripts/` | `/etc/sysconfig/network-scripts/` |
> | 启动脚本 | 无 `startsrc/stopsrc` | `startsrc -s`、`stopsrc -s` |
> | SSH 服务名 | `sshd` | `sshd` |

> 下文每项均标注**两套命令**，填表前**必须先确认实际发行版**（`cat /etc/redhat-release` 或 `cat /etc/os-release`），再取对应分支。**不要在同一台主机上混用两套命令。**

## 一、基本信息（待填）

| 字段 | 值 |
| --- | --- |
| 作业开始时间 | |
| 作业任务 | Linux 主机病毒查杀与安全加固（原表预填） |
| 作业负责人 | |
| 作业人员 | |
| Linux 主机名 | |
| 主机地址（所有网卡 IP） | |
| 设备或系统名称 | |
| Linux 版本 | |
| 数据库版本（禁止 root 权限部署） | |
| 中间件版本（禁止 root 权限部署应用） | |

> **"禁止 root 权限部署"对 Linux 的含义**：数据库与中间件服务**不得以 `root` 运行**，须创建专用服务账号（如 `oracle`、`tomcat`、`weblogic`）并 `chown` 相应目录、`systemd` 单元中设 `User=`。这是**硬性要求**，须在填表前实际验证服务运行账号。

**填表前必须确认的两项**：

```bash
cat /etc/redhat-release 2>/dev/null || cat /etc/os-release   # 发行版与版本
uname -r                                                    # 内核版本
```

## 二、作业项（原表内容）

### 1 病毒查杀

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B1 | 使用主机杀毒或杀毒 U 盘软件完成病毒查杀与日志导出 | 下载地址：Ⅲ 区 `https://3.1.1.1`，Ⅰ/Ⅱ 区 `https://1.1.1.1`；完成病毒查杀（最近一次扫描不超过六个月），开启软件携带的 U 盘管控功能；查看 `cat /opt/LinuxKPC/ini/apsc_client_ui.ini` 与 `date -d @时间戳 "+%Y-%m-%d"` 核对最近扫描时间；记录病毒库更新时间 `cat /opt/LinuxKPC/ini/version`；记录发现与清除病毒数量 | 标准 1 |

### 2 设备管理

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B2 | 管理远程工具 | 安装 SSH，用 SSH 代替 telnet、FTP，对传输数据加密。配置值见下表 | 标准 1 |
| B3 | 访问控制 | 默认远程 22 端口改为 **22222**；配置本机访问控制列表，限制远程 SSH 管理 IP，**仅允许堡垒机、态势感知等访问 SSH** | 标准 1 |

### 3 用户账号与口令安全

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B4 | 限制系统无用的默认帐号登录 | 删除 `#userdel username`；锁定：`/etc/shadow` 中用户名后加 NP，或 `/etc/passwd` 中 shell 域设为 `/bin/noshell`、`/bin/nologin`，或 `#passwd -l username`。需锁定用户清单见下 | 标准 1 |
| B5 | root 用户远程登录限制 | 需先提供或新建一个可登录的普通账号，并在堡垒机设置登录切换。方法一：`/etc/ssh/sshd_config` 将 `PermitRootLogin yes` 改为 `no`，重启 sshd；方法二：`/etc/securetty` 配置 `CONSOLE = /dev/tty01` | 标准 1 |
| B6 | 口令策略 | 启用密码复杂性要求。`more /etc/login.defs` 检查 `PASS_MAX_DAYS`/`PASS_MIN_LEN`/`PASS_MIN_DAYS`/`PASS_WARN_AGE`；`/etc/pam.d/system-auth` 配置登录失败锁定；`awk -F: '($2 == "") { print $1 }' /etc/shadow` 检查空口令账户 | 标准 1、3 |
| B7 | 启用登录失败锁定 | 连续认证失败 **5 次**锁定该账户 **30 分钟**。配置 `/etc/pam.d/login`：`auth required pam_tally2.so deny=5 unlock_time=1800` | 标准 3 |
| B8 | 控制用户登录会话 | 登录超时控制用户登录会话时间为 **300 秒**。3 级系统执行 `vi /etc/profile` 设置 `TMOUT=300` | 标准 1 |
| B9 | FTP 用户帐号控制 | 拒绝系统默认的系统账号使用 ftp 服务。`cat /etc/ftpd/ftpusers` 或 `/etc/vsftpd/ftpusers` 查看；先 `cp -p` 备份，再逐行添加禁止 FTP 登录的账户 | 标准 1 |
| B10 | 检查是否存在除 root 之外 UID 为 0 的用户 | `awk -F: '($3 == 0) { print $1 }' /etc/passwd`。返回值除 root 外还有条目则低于安全要求 | 标准 1 |
| B11 | root 用户环境变量的安全性 | `echo $PATH \| egrep '(^\|:)(\.\|:\|$)'` 检查是否包含父目录；`find \`echo $PATH \| tr ':' ' '\` -type d \( -perm -002 -o -perm -020 \) -ls` 检查是否包含组目录权限为 777 的目录 | 标准 1 |

### 4 日志与审计

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B12 | syslog 登录事件记录 | 捕获 authpriv 消息，记录安全方面日志消息（如网络设备启动、usermod、change 等）。`more /etc/syslog.conf` 查看 authpriv 值 | 标准 1 |
| B13 | 指定日志服务器 | 配置专门的日志服务器 `loghost`（例如态势感知），4 条规则见下 | 标准 1 |
| B14 | 日志系统配置文件保护 | `chmod 400 /etc/syslog.conf`，使管理员只读 | 标准 1 |
| B15 | 开机启动 syslog 服务 | syslog 服务能在启动服务器时开启。`/etc/syslog.conf` 配置 `*.err;kern.debug;daemon.notice; /var/adm/messages`；`ps -eaf \| grep syslog` 查看；`startsrc -s syslogd` 启动，`stopsrc -s syslogd` 停止 | 标准 3 |
| B16 | 日志记录满足 6 个月 | 日志记录满足 6 个月 | 标准 3 |

### 5 服务优化

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B17 | 禁用不必要的系统服务 | `cp /etc/inet/inetd.conf /etc/inet/inetd.conf.backup` 后编辑注释不需要的服务；禁用清单见下。`systemctl \| grep active` 或 `chkconfig --list` 查看开放的服务 | 标准 1 |
| B18 | 防止 icmp 重定向报文的攻击 | 备份后执行 `sysctl -w net.ipv4.conf.all.accept_redirects="0"`，并将 `/proc/sys/net/ipv4/conf/all/accept_redirects` 的值改为 0 | 标准 4 |
| B19 | SNMP、SNMP TRAP 服务接受团体名称设置 | `/etc/snmp/snmpd.conf` 修改默认团体名，**团体名改为 `GXhmrjsw-q3`**；`get-community-name` 将默认 public 修改为复杂团体名（读团体名，即读的密码） | 标准 4 |
| B20 | 系统时间同步 | 系统时间与 NTP 服务器同步，`ntpdate 3.1.1.4` | 标准 4 |

### 6 安全防护

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B21 | Umask 权限 | 控制用户缺省访问权限 UMASK 为 **022**。`vi /etc/profile` 设置 | 标准 1 |
| B22 | 敏感文件安全保护 | `chown root /etc/passwd /etc/shadow /etc/group`；`chmod 644 /etc/passwd /etc/group`；`chmod 400 /etc/shadow` | 标准 1 |
| B23 | Root 操作历史 | 减少 root 输入命令的历史记录长度，保留最新执行的 **5 条**命令。`vi /etc/profile` 修改 `HISTSIZE=5` | 标准 1 |
| B24 | 查看端口进程并结束 | 加固后 `netstat -anp \| grep 端口号`；添加防火墙出入站策略禁用高危端口；进程仍运行则 `kill -9 进程号` 强制结束，重启后再次检查。端口清单见下 | 标准 1 |
| B25 | 禁止多余 USB 接口的使用 | 除鼠标键盘等必要外设接入 USB 外，其余 USB 口用封条封死（原则上保留一个数据拷贝接口） | 标准 5 |
| B26 | 卸载 samba 服务、删除默认路由 | 卸载 samba；删除默认路由 0.0.0.0；配置指向 `10.67.x.0` 的路由，下一条是默认网关；禁止存在默认路由，必须全为明细路由，路由掩码应 **≥24 位**。路由记得保存，不然重启会丢失 | 标准 1 |

## 三、逐项操作方法（B1–B26）

### B1 病毒查杀与日志导出

**问题**：使用主机杀毒或杀毒 U 盘软件完成病毒查杀与日志导出。

#### 🔍 风险原理

Linux 主机同样面临病毒威胁，且**运维人员习惯用 U 盘拷贝文件**，是移动介质投毒的主要路径。病毒库超期未更新则查杀能力失效；日志不导出则无法证明查杀过程。

#### 配置/加固方法

1. 下载安装**安天防病毒软件**：Ⅲ 区 `https://3.1.1.1`，Ⅰ/Ⅱ 区 `https://1.1.1.1`；
2. 开启软件自带的 **U 盘介质管控**功能；
3. 执行全盘扫描，记录**发现病毒数量**与**清除病毒数量**；
4. **导出扫描日志**；
5. 记录最近扫描时间与病毒库更新时间，确认**均不超过六个月**。

```bash
# 查看客户端配置中的最近扫描时间戳
cat /opt/LinuxKPC/ini/apsc_client_ui.ini

# 将时间戳转为可读日期
date -d @<时间戳> "+%Y-%m-%d"

# 查看病毒库版本/更新时间
cat /opt/LinuxKPC/ini/version
```

> `/opt/LinuxKPC/` 为安天 Linux 客户端安装路径，随版本可能不同，以实际安装路径为准。

#### 特殊情况处理

- 扫描会占用 I/O 与 CPU，生产主机须在业务低峰执行；
- 杀毒软件与中间件若有兼容性冲突（误杀 Java 进程），**先加白名单**，不得关闭防护；
- 内网主机无法直连下载源时，须从 Ⅲ 区或 Ⅰ/Ⅱ 区对应地址获取，**分区地址不要填错**。

#### 📌 加固后的验证

```bash
cat /opt/LinuxKPC/ini/version
date -d @<时间戳> "+%Y-%m-%d"
```

期望结果：病毒库更新时间 ≤ 6 个月，最近扫描时间 ≤ 6 个月，日志已导出，发现/清除病毒数量已记录。

---

### B2 管理远程工具：SSH 替代 telnet/FTP

**问题**：安装 SSH，用 SSH 代替 telnet、FTP，对传输数据加密。

#### 🔍 风险原理

telnet 与 FTP **全程明文**，口令与传输内容在网路上可被嗅探。电力调度网跨安全区传输，泄露的可能是远动点表、组态文件乃至管理口令。

#### 配置/加固方法

```bash
# 查看是否已安装并运行
rpm -qa | grep openssh        # CentOS
ps -eaf | grep sshd
```

`/etc/ssh/sshd_config` 关键配置值：

| 配置项 | 值 | 说明 |
| --- | --- | --- |
| `Protocol` | 2 | 使用 ssh2 版本 |
| `X11Forwarding` | yes | 允许窗口图形传输使用 ssh 加密 |
| `IgnoreRhosts` | yes | 完全禁止 SSHD 使用 .rhosts 文件 |
| `RhostsAuthentication` | no | 不设置使用基于 rhosts 的安全验证 |
| `RhostsRSAAuthentication` | no | 不设置使用 RSA 算法的基于 rhosts 的安全验证 |
| `PermitRootLogin` | no | 禁止 root 远程登录（与 B5 一致） |
| `PermitEmptyPasswords` | no | 禁止空密码 |
| `Banner` | /etc/issue | 指定登录提示文件 |

原表给出的实施步骤（备份 → 生成临时文件 → 替换）：

```bash
cd /etc/ssh
cp -p sshd_config sshd_config.tmp

# 生成修改了安全配置的临时文件后替换原文件
mv sshd_config.tmp sshd_config

# 重启 sshd（两套体系任选其一，勿混用）
# CentOS 7 系：
systemctl restart sshd
# 凝思 Linx / RHEL 6 系：
service sshd restart
# 或：
/etc/init.d/sshd restart
```

查看 SSH 服务状态：`ps -elf | grep ssh`

#### 特殊情况处理

- **`Protocol`、`RhostsAuthentication`、`RhostsRSAAuthentication` 在 OpenSSH 7.x 起已废弃**，写入会触发 "Unsupported option" 告警但不影响启动；新版主机可忽略这三项；
- **`PermitRootLogin no` 后 root 无法远程登录**，须先确认已创建可登录普通账号（与 B5 强关联），否则改完配置会把自己锁在外面；
- 改 `sshd_config` 前**务必另开会话保持连接**，语法错误会导致 sshd 无法重启；
- `Banner /etc/issue` 会向登录前用户展示提示信息，内容不应泄露厂商/型号等敏感信息；
- 确认 **telnet、ftp 服务已停止**（见 B17），仅装 SSH 不停旧服务等于两个入口都在。

#### 📌 加固后的验证

```bash
sshd -t                      # 语法检查（无输出即正确）
grep -E "^PermitRootLogin|^PermitEmptyPasswords|^Protocol" /etc/ssh/sshd_config
ss -tlnp | grep sshd         # 确认监听端口
```

期望结果：语法检查通过，禁止 root 与空密码已生效，SSH 正常监听（端口见 B3）。

---

### B3 访问控制：改默认端口 + 管理源 IP 白名单

**问题**：默认远程 22 端口改为 **22222**；配置本机访问控制列表，限制远程 SSH 管理 IP，**仅允许堡垒机、态势感知等访问 SSH**。

#### 🔍 风险原理

两个动作解决两个不同问题：

- **改端口（22 → 22222）**：削弱自动化扫描与暴力破解的效率——扫描器优先打 22；
- **IP 白名单**：真正的访问控制——即使口令被猜到，非授权来源也无法建立连接。

> **端口不是安全措施，只是延缓。白名单才是。本项目硬绑定为"仅允许堡垒机、态势感知"。**

#### 配置/加固方法

**1. 改端口**

`/etc/ssh/sshd_config`：

```
Port 22222
```

```bash
# 语法检查后重启（勿用 reload，端口变更需完整重启）
sshd -t && systemctl restart sshd      # CentOS 7 系
sshd -t && service sshd restart        # 凝思 Linx / RHEL 6 系
```

**2. 配置访问控制列表**

```bash
vi /etc/hosts.allow
# 逐条放行允许访问的 IP（原文占位：X.X.X.X）
ssh:<堡垒机IP>:allow
ssh:<态势感知IP>:allow

vi /etc/hosts.deny
# 兜底拒绝
all:all
```

**3. 使配置生效**

```bash
pkill -HUP inetd              # 重启 inetd 进程
```

原表补充说明：更改 `/etc/inetd.conf` 文件后，重启 inetd 的命令是 `pkill -HUP inetd` 或 `/etc/init.d/inetsvc start`。

> 注意：`hosts.allow`/`hosts.deny` 依赖 **TCP Wrappers（inetd）**。若系统已不再使用 inetd，需确认 `libwrap` 支持；现代发行版部分场景改用 `sshd_config` 的 `AllowUsers`/`DenyUsers` 与防火墙实现。

#### 特殊情况处理

- **改端口后原 22 端口须确认已关闭**（`ss -tlnp | grep -w 22` 无输出），否则等于没改；
- **`hosts.deny` 写 `all:all` 是兜底**，若写成 `sshd:all` 只挡 ssh；顺序上 `hosts.allow` 优先于 `hosts.deny`；
- **白名单 IP 须用实际管理 IP**，误填会导致运维失去入口——改前先在当前会话确认可用，并在变更窗口内操作；
- 堡垒机与态势感知若 IP 会变，须用**网段**而非单 IP 授权，并纳入变更评审。

#### 📌 加固后的验证

```bash
ss -tlnp | grep sshd                                    # 应为 22222，无 22
grep -v "^#" /etc/hosts.allow /etc/hosts.deny           # 核对放行与兜底规则

# 从堡垒机测试（应通）
ssh -p 22222 <用户>@<主机IP>
# 从非授权主机测试（应不通）
```

期望结果：22 端口不再监听，22222 仅堡垒机/态势感知可达。

---

### B4 限制系统无用的默认帐号登录

**问题**：删除或锁定系统默认无用账号。

#### 🔍 风险原理

系统自带大量默认服务账号（`daemon`、`bin`、`lp` 等），虽通常不可登录，但部分账号在老旧系统上存在可登录或弱口令历史。它们是攻击者寻找入口时的固定目标。

#### 配置/加固方法

```bash
# 查看账号状态（末尾为 nologin/false 的已锁定）
cat /etc/passwd | grep -E "(daemon|bin|sys|adm|uucp|lp|snapp|nobody|invscout|nuucp|lpd|imnadm|ipsec|ldap)"
```

需锁定清单：`daemon bin sys adm uucp lp snapp nobody invscout nuucp lpd imnadm ipsec ldap`

三种锁定方式：

```bash
# 方式 1：/etc/shadow 中用户名后加 NP（最直接）
vi /etc/shadow

# 方式 2：shell 域设为不可登录
usermod -s /bin/nologin <用户名>
usermod -s /bin/noshell <用户名>

# 方式 3：锁定口令
passwd -l <用户名>
usermod -L <用户名>
```

> 原表中的 `#userdel username`、`#passwd -l username` 前带 `#` 是**注释掉的示例**，实际执行时不带 `#`。

#### 特殊情况处理

- **`nobody`、`ldap`、`ipsec` 是服务运行账号，锁定其 shell 不影响服务**，但**不要 `userdel` 删除**，否则服务启动失败；
- `lpd` 若启用了打印服务则不能锁；
- 锁定前**确认该账号无业务依赖**，删除类操作不可逆；
- 保留一个可登录的普通运维账号（与 B5 配套），否则锁定默认账号后可能无人能登录。

#### 📌 加固后的验证

```bash
# 确认全部为 nologin/false 或已锁定
awk -F: '($3 < 500) { print $1":"$7 }' /etc/passwd
```

期望结果：除 root 外，系统默认账号的 shell 均为 `/sbin/nologin`、`/bin/false` 或 `/bin/noshell`，无法交互登录。

---

### B5 root 用户远程登录限制

**问题**：先提供或新建一个可登录的普通账号，并在堡垒机设置登录切换；`PermitRootLogin no` 或 `/etc/securetty` 配置。

#### 🔍 风险原理

root 权限过大，一旦会话被劫持，攻击者立即拥有系统最高权限，且不留"越权"痕迹。禁止 root 远程登录、强制走普通账号 + `su`，是**权限最小化与可审计**的基本要求。

#### 配置/加固方法

**前置：创建普通账号并配置 sudo**（否则禁 root 后无法提权）

```bash
useradd -m opsuser
passwd opsuser
usermod -aG wheel opsuser          # CentOS 7 系；RHEL 6 为 wheel，同样
# 确认 sudo 可用
visudo                              # 建议用 visudo 而非直接编辑 /etc/sudoers
```

**方法一：禁止 root 远程 SSH 登录**

```bash
vi /etc/ssh/sshd_config
# PermitRootLogin yes  →  no

sshd -t && systemctl restart sshd     # CentOS 7 系
sshd -t && service sshd restart       # 凝思 Linx / RHEL 6 系
```

**方法二：限制 root 只能从本地控制台登录**

```
vi /etc/securetty
# 加入允许的终端设备（原文示例）
CONSOLE = /dev/tty01
```

> `/etc/securetty` 中未列出的终端 root 无法登录，telnet 终端也不允许。

**方法三：配置 sudo 免密切换（堡垒机侧配合）**

```
visudo
# opsuser ALL=(ALL) NOPASSWD: ALL
```

> 免密 sudo 便于运维，但会削弱提权审计。**本项目建议开启 sudo 日志**（默认记入 `/var/log/secure`），或在堡垒机上做命令级控制。

#### 特殊情况处理

- **禁用 root 远程登录前必须确认普通账号可用**，否则会锁死远程运维入口；
- 部分老旧中间件（如早期 WebLogic、Oracle 安装脚本）依赖 root 运行，**禁止 root 登录不影响本地 root 使用**，只影响远程；
- `/etc/securetty` 的 `CONSOLE` 行须与实际终端设备名一致（`tty1`/`tty01` 依系统而定），配错等于 root 无法登录控制台；
- 堡垒机侧须同步配置"登录切换"（普通账号登录后提权），否则运维习惯会不适应。

#### 📌 加固后的验证

```bash
grep -E "^PermitRootLogin" /etc/ssh/sshd_config
# 尝试 root 远程登录（应失败）
ssh root@<主机IP> -p 22222
# 尝试普通账号登录后提权（应成功）
ssh opsuser@<主机IP> -p 22222
```

期望结果：root 远程登录被拒绝；普通账号可登录并通过 sudo/su 提权。

---

### B6 口令策略

**问题**：启用密码复杂性要求；检查 `login.defs` 参数；PAM 配置登录失败锁定；检查空口令账户。

#### 🔍 风险原理

Linux 的口令策略分散在两处：`/etc/login.defs`（**默认策略**，仅对新设口令生效）与 **PAM**（强制执行）。**只改 login.defs 不改 PAM，空口令和弱口令仍可登录**。

#### 配置/加固方法

**1. 检查默认口令策略**

```bash
more /etc/login.defs
```

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `PASS_MAX_DAYS` | 180 | 密码最长使用天数（可选） |
| `PASS_MIN_DAYS` | 1 | 密码最短使用天数 |
| `PASS_WARN_AGE` | 28 | 密码到期提前提醒天数 |
| `PASS_MIN_LEN` | 8 | 密码最小长度 |

```bash
# 修改
sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   180/' /etc/login.defs
sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/'   /etc/login.defs
sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE   28/'  /etc/login.defs
sed -i 's/^PASS_MIN_LEN.*/PASS_MIN_LEN     8/'   /etc/login.defs
```

**2. PAM 强制口令复杂度与失败锁定**

`/etc/pam.d/system-auth`（CentOS 7 系典型栈）：

```
auth required pam_env.so
auth required pam_tally.so onerr=fail deny=5 unlock_time=1800
auth sufficient pam_unix.so nullok try_first_pass
auth requisite pam_succeed_if.so uid >= 500 quiet
auth required pam_deny.so
```

> **`nullok` 的取舍**：上例保留 `nullok` 允许空口令通过 PAM，安全性不足；配合下方"空口令账户检查"清零后可接受。**若允许登录的账号均无空口令，可去掉 `nullok` 更严格**。凝思 Linx / RHEL 6 系常见配置：

```
auth required pam_cracklib.so try_first_pass retry=3 minlen=8 dcredit=-1 ucredit=-1
auth required pam_tally2.so deny=5 unlock_time=1800
```

`/etc/pam.d/login`（B7 要求的 5 次 / 30 分钟）：

```
auth required pam_tally2.so deny=5 unlock_time=1800
```

**3. 检查空口令账户**

```bash
awk -F: '($2 == "") { print $1 }' /etc/shadow
```

> **返回为空才是合格**。有输出则必须 `passwd <用户名>` 设口令或 `usermod -L <用户名>` 锁定。

**4. 排查简单口令**

原表要求向管理员询问是否存在类似 `root/root`、`test/test`、`root/root12345678` 的简单用户密码配置——**须逐项排查并当场修改**。

#### 特殊情况处理

- **`/etc/login.defs` 不影响已存在的口令**，改完必须 `passwd <用户名>` 重新设置才生效；
- **PAM 修改有锁死风险**：语法错误或 `deny` 值过小会导致正常账号无法登录，改前备份 `/etc/pam.d/system-auth`，并保留当前会话；
- `pam_tally` 与 `pam_tally2` 参数格式不同，**勿混用**；
- **服务账号（如 `oracle`、`mysql`）通常设 `*` 或 `!!` 口令**，属正常，不算空口令；`awk` 检查结果中若出现此类账号应理解为已锁定；
- 应用自身管理的口令（数据库内部账号）不受 `login.defs` 约束，须在应用侧配置。

#### 📌 加固后的验证

```bash
grep -E "^PASS_(MAX|MIN|WARN)" /etc/login.defs
awk -F: '($2 == "") { print $1 }' /etc/shadow      # 应无输出
grep -E "pam_(tally|cracklib)" /etc/pam.d/system-auth
```

期望结果：参数符合要求，空口令账户为空，PAM 栈含复杂度与失败锁定模块。

---

### B7 启用登录失败锁定

**问题**：连续认证失败 **5 次**锁定该账户 **30 分钟**。配置 `/etc/pam.d/login`：`auth required pam_tally2.so deny=5 unlock_time=1800`。

#### 🔍 风险原理

无失败锁定时，攻击者可无限次尝试口令。30 分钟锁定使每次爆破只能试 5 次，暴力破解在时间上不可行。

> ⚠️ **与 Windows A4 不一致**：Windows A4 要求锁定 **5 分钟**，Linux B7 要求锁定 **30 分钟**（`unlock_time=1800`）。**各按本 sheet 执行，不要互相套用。**

#### 配置/加固方法

```
vi /etc/pam.d/login
auth       required   pam_tally2.so deny=5 unlock_time=1800
```

CentOS 7 系也可在 `/etc/pam.d/system-auth` 中配置（同 B6 的栈）。

```bash
vi /etc/pam.d/system-auth
# 在 auth 栈中 pam_unix 之前加入
auth       required   pam_tally2.so deny=5 unlock_time=1800
```

```bash
# 语法检查：直接尝试一次错误登录，若被计数则生效
ssh -p 22222 <用户名>@<主机IP>     # 故意输错 5 次
```

#### 特殊情况处理

- **计数文件位置**：`/var/log/faillog`（CentOS 7）/ `/var/run/faillog`（RHEL 6 系）。手工清零：
  ```bash
  faillog -u <用户名> -r      # 重置计数
  ```
- **锁定时间过长会锁死运维**：忘记口令后须等 30 分钟或 root 手工解锁；
- **服务账号被误锁会中断服务**：计划任务用固定账号调用脚本时，若脚本内认证配置有误会反复触发锁定；
- `pam_tally2.so` 需已安装（`rpm -q pam_tally2`），缺失会导致 PAM 加载失败。

#### 📌 加固后的验证

```bash
# 连续 5 次错误登录后应被锁定
faillog -u <用户名>        # 查看失败计数
```

期望结果：第 6 次尝试即被拒绝，30 分钟后自动解锁。

---

### B8 控制用户登录会话

**问题**：登录超时控制用户登录会话时间为 **300 秒**。3 级系统执行 `vi /etc/profile` 设置 `TMOUT=300`。

#### 🔍 风险原理

`TMOUT` 是 shell 级的空闲登出：无操作超过设定秒数后自动退出，**终端回到登录界面**。它与 Windows A22 的屏保超时是同类防护，防止"人走机未锁"。

#### 配置/加固方法

```bash
vi /etc/profile
# 全局设置（单位：秒）
TMOUT=300
```

或写入 `/etc/profile.d/timeout.sh` 便于管理：

```bash
echo 'TMOUT=300' > /etc/profile.d/timeout.sh
chmod 644 /etc/profile.d/timeout.sh
```

**注意**：`TMOUT` 需写在 `export` 之后才对该 shell 生效，且**对 root 无效**（root 不会因空闲自动退出）：

```bash
export TMOUT=300
```

仅对单个用户生效则写在其 `~/.bashrc` 中。

#### 特殊情况处理

- **长任务会被中断**：超过 300 秒无交互输入的脚本、编译、大文件传输会收到 SIGHUP 而终止。应将此类任务改为 `nohup`、`screen`、`tmux` 或后台执行；
- **root 不受 `TMOUT` 约束**，这是 Linux 设计，root 的会话管理靠 A/B9 的人工规范；
- 建议配合 `HISTFILESIZE` 一起设置（见 B23）。

#### 📌 加固后的验证

```bash
grep -n "^TMOUT" /etc/profile
# 登录后静置超过 5 分钟，确认已自动登出
```

期望结果：普通用户空闲 300 秒后自动退出登录。

---

### B9 FTP 用户帐号控制

**问题**：拒绝系统默认的系统账号使用 ftp 服务。

#### 🔍 风险原理

FTP 传输明文口令（B2 已要求用 SSH 替代）。若 FTP 未关闭，系统账号可能被用于 FTP 匿名或弱口令登录；某些老旧 FTP（wu-ftpd）的 `ftpusers` 机制正是防止系统账号被用于 FTP。

#### 配置/加固方法

```bash
# 查看当前禁止 FTP 登录的账户
cat /etc/ftpd/ftpusers 2>/dev/null || cat /etc/vsftpd/ftpusers 2>/dev/null
```

禁止 FTP 登录的账户清单：`root daemon bin sys adm lp uucp nuucp listen nobody hpdb useradm`

```bash
# 先备份
cp -p /etc/vsftpd/ftpusers /etc/vsftpd/ftpusers.bak
# 再逐行添加禁止 FTP 登录的账户
echo "root"   >> /etc/vsftpd/ftpusers
echo "daemon" >> /etc/vsftpd/ftpusers
```

#### 特殊情况处理

- **`ftpusers` 只对 wu-ftpd/部分 vsftpd 版本生效**，现代 vsftpd 需在 `userlist_enable`/`userlist_file` 中配置；确认 FTP 实现后再改文件；
- **`nobody` 出现在该清单中是正确的**（禁止该通用账号 FTP 登录），不是笔误；
- **最佳做法是直接停用 FTP 服务**（见 B17），从源头消除明文传输；本条作为纵深防御保留；
- 若业务确需 FTP，须强制 SFTP（基于 SSH）而非明文 FTP。

#### 📌 加固后的验证

```bash
grep -cE "^(root|daemon|bin)" /etc/vsftpd/ftpusers
ftp -n <主机IP>     # 应无法用系统账号登录
```

期望结果：清单内账号均无法用于 FTP 登录。

---

### B10 检查除 root 外 UID 为 0 的用户

**问题**：`awk -F: '($3 == 0) { print $1 }' /etc/passwd`，返回值除 root 外还有条目则低于安全要求。

#### 🔍 风险原理

UID 0 即 root 权限。**任何 UID 0 的账号都等价于 root**——它的登录审计、权限边界全部形同虚设，是典型的权限后门。合法系统通常只应有 root 一个 UID 0 账号。

#### 配置/加固方法

```bash
awk -F: '($3 == 0) { print $1 }' /etc/passwd
```

**返回应仅 `root` 一行。** 若有其他条目：

```bash
# 1. 确认该账号用途（可能是历史遗留或恶意植入）
# 2. 降低其 UID，或禁用其登录
usermod -s /bin/nologin <可疑账号>
passwd -l <可疑账号>
# 3. 若确认无用途，备份后删除
cp -p /etc/passwd /etc/passwd.bak
userdel <可疑账号>
```

> 数据库备份文件、`/etc/passwd-`（含 `-` 后缀的备份）本身也有 UID 0 条目，属正常备份，不算异常账号。

#### 特殊情况处理

- **`useradd -o -u 0` 可创建 UID 0 账号**，是提权后门的手法之一，须重点排查；
- 部分老系统有 `sync`、`shutdown`、`halt` 等系统账号历史上是 UID 0，现代已改为非 0，不必强行改；
- 误删 UID 0 账号（如误删 oracle）会导致数据库无法启动，删除前务必确认服务依赖。

#### 📌 加固后的验证

```bash
awk -F: '($3 == 0) { print $1 }' /etc/passwd
```

期望结果：输出仅 `root`。

---

### B11 root 用户环境变量的安全性

**问题**：检查 `PATH` 是否包含父目录/当前目录；检查 `PATH` 中是否存在组目录权限为 777 的目录。

#### 🔍 风险原理

`PATH` 里若含 `.`（当前目录）或空元素，用户切换目录后执行 `ls` 就可能执行到同名的恶意程序（PATH 劫持）。若 `PATH` 中某目录**其他用户可写**，任何用户都能往里放程序等待 root 执行。

#### 配置/加固方法

**1. 检查 PATH 是否含父目录/当前目录**

```bash
echo $PATH | egrep '(^|:)(\.|:|$)'
```

> **期望无输出**（root 的 PATH 正常形如 `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`）。

检查全部用户：

```bash
for u in $(cut -d: -f1 /etc/passwd); do
  p=$(su - "$u" -c 'echo $PATH' 2>/dev/null)
  echo "$p" | egrep -q '(^|:)(\.|:|$)' && echo "风险用户: $u"
done
```

**2. 检查 PATH 中是否有 777 目录**

```bash
find $(echo $PATH | tr ':' ' ') -type d \( -perm -002 -o -perm -020 \) -ls
```

> **期望无输出**。`-perm -002` = 其他用户可写；`-perm -020` = 组可写。

**3. 修正**

```bash
# 修改 root 的 PATH
vi /etc/profile
# 或 /root/.bashrc
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

把 777 目录改为 755：

```bash
chmod 755 <目录>
```

#### 特殊情况处理

- **`/tmp` 通常是 1777（sticky）**，本身可写是设计如此，**但绝不应出现在 root 的 PATH 中**；
- 应用自带工具链常把自身 `bin` 加进 PATH（如 `/opt/tomcat/bin`），若该目录属主是应用账号，则 root 执行时可能运行被篡改的脚本——应用目录建议 root 属主；
- 修改全局 `/etc/profile` 影响所有用户，须评估；仅修 root 则改 `/root/.bashrc`。

#### 📌 加固后的验证

```bash
echo $PATH | egrep '(^|:)(\.|:|$)'                                  # 应无输出
find $(echo $PATH | tr ':' ' ') -type d \( -perm -002 -o -perm -020 \) -ls   # 应无输出
```

期望结果：PATH 不含 `.`、空元素与组/其他可写目录。

---

### B12 syslog 登录事件记录

**问题**：捕获 authpriv 消息，记录安全方面日志消息。

#### 🔍 风险原理

**登录、口令变更、账号增删是溯源的关键事件**。默认 syslog 若不记录 `authpriv.*`，`su`、`sshd`、`useradd`、`passwd` 等动作不会留下日志，攻击者拿到 root 后可无痕操作。

#### 配置/加固方法

```bash
# 原表：查看 /etc/syslog.conf
more /etc/syslog.conf
grep -n "authpriv" /etc/syslog.conf
```

**凝思 Linx / RHEL 6 系**（`/etc/syslog.conf`）：

```
# 记录认证与授权相关（安全日志）
authpriv.*      /var/log/secure
auth.*          /var/log/secure

# 记录系统启动、网络设备启动等
local7.*        /var/log/boot.log
```

> **`/var/log/secure` 是 SUSE 系（RHEL 6/CentOS 6）特有的安全日志文件**，记录 su、sudo、sshd、passwd、useradd 等事件，是本项的核心目标。

**CentOS 7 系**（`/etc/rsyslog.conf`，注意路径与格式不同）：

```
authpriv.*                      /var/log/secure
*.*;authpriv.none;mail.none     -/var/log/secure
```

查看实际日志：

```bash
tail -f /var/log/secure
# 确认文件权限
ls -l /var/log/secure      # 应为 600 root:root
```

#### 特殊情况处理

- **CentOS 7 用 rsyslog、配置文件是 `/etc/rsyslog.conf`，不是 `/etc/syslog.conf`**——原表只给了 `/etc/syslog.conf`，**CentOS 7 主机须改路径**；
- `sshd` 的日志走 `authpriv`（成功）与 `auth`（失败），两者都要记，否则看不到失败的暴力破解尝试；
- rsyslog 改动后重启：`systemctl restart rsyslog`；
- `/var/log/secure` 权限须为 600，普通用户不得读取（否则可看到账号信息）。

#### 📌 加固后的验证

```bash
grep -n "authpriv" /etc/rsyslog.conf /etc/syslog.conf 2>/dev/null
# 执行一次 su，观察日志
su - opsuser
tail -20 /var/log/secure
```

期望结果：登录/提权动作在安全日志中可见，文件权限 600。

---

### B13 指定日志服务器

**问题**：配置专门的日志服务器 `loghost`（例如态势感知），4 条规则。

#### 🔍 风险原理

**本机日志可被攻击者一并删除**。日志外送是唯一的抗抵赖手段：本机 `/var/log` 被清空不影响平台上已留存的记录，也是电力监控系统集中审计的合规要求。

> **本项目绑定**：Linux 主机须 **SNMP + SYSLOG 接入安管平台**（见第六节），即 SNMP 团体名 `GXhmrjsw-q3` + syslog 外送。

#### 配置/加固方法

**1. 定义 loghost**

`/etc/hosts`：

```
<日志服务器IP>    loghost
```

**2. 配置转发规则**

`/etc/syslog.conf`（或 `/etc/rsyslog.conf`）：

```
kern.warning;*.err;authpriv.none	@loghost
*.info;mail.none;authpriv.none;cron.none	@loghost
*.emerg	@loghost
local7.*	@loghost
```

> 传统 syslog 的 `@` 表示 **UDP**（不可靠但轻量）。需要可靠传输用 `@@`（TCP）。**传输协议须与日志服务器要求一致**，参见 Windows A12 的同类说明。

**CentOS 7 系（rsyslog）**：

```
# /etc/rsyslog.conf
*.*     @@<日志服务器IP>:514

# 或使用 imlogsd 模块模板
module(load="imudp")
input(type="imudp" port="514")
```

**3. 确保本地日志仍保留**

外送**不能取代**本地日志（B14/B16 仍要求本地配置）。

```bash
# 重启日志服务
systemctl restart rsyslog        # CentOS 7 系
service syslog restart           # 凝思 Linx / RHEL 6 系
```

#### 特殊情况处理

- **网络不通时日志会丢失**，外送配置本身不产生本地留存，须保证 B16 的本地留存；
- `@`（UDP）不保证送达，**关键日志建议 `@@`（TCP）**，但 TCP 断链时可能阻塞日志写入，需评估；
- **不要用 `local7.*`（设施号 7）的转发吞掉本机 `local7` 记录**，B15 要求 `local7.*` 写 `/var/adm/messages`；
- 日志量大的主机应限制级别（如仅 warning 以上），避免日志服务器被淹没；
- 防火墙/网络策略须放通到日志服务器的 UDP/TCP 514。

#### 📌 加固后的验证

```bash
# 日志服务器端确认收到
# 本机触发一条日志
logger "loghost forwarding test"
# 或触发认证事件
tail -f /var/log/secure
```

期望结果：日志服务器平台实时收到该主机日志。

---

### B14 日志系统配置文件保护

**问题**：`chmod 400 /etc/syslog.conf`，使管理员只读。

#### 🔍 风险原理

日志配置决定"什么被记录"。攻击者拿到权限后第一件事往往是**改日志配置关掉记录、清理日志**。把配置文件设为只读（root 也只能读）能有效阻止此类篡改。

#### 配置/加固方法

```bash
# 先备份（400 下无法直接覆盖，故先改权限再改再收回）
cp -p /etc/syslog.conf /etc/syslog.conf.bak
chmod 400 /etc/syslog.conf
```

CentOS 7 系：

```bash
cp -p /etc/rsyslog.conf /etc/rsyslog.conf.bak
chmod 400 /etc/rsyslog.conf
```

```bash
ls -l /etc/syslog.conf
# 应为 -r-------- 1 root root
```

需要修改时：

```bash
chmod 600 /etc/syslog.conf
vi /etc/syslog.conf
# 修改完成后
chmod 400 /etc/syslog.conf
```

#### 特殊情况处理

- **`chmod 400` 后日志服务仍可正常读取**（读取只需 r 位），但**日志服务重载配置时某些工具会临时重写文件**，出现权限报错；重载前后确认服务状态；
- 备份文件 `syslog.conf.bak` 同样含配置信息，权限应一致；
- **日志文件本身**也建议加固权限（`/var/log/secure` 为 600，见 B12），配置文件只读只是第一步；
- 部分系统升级（yum update）会覆盖配置文件并重置权限，升级后须复查。

#### 📌 加固后的验证

```bash
ls -l /etc/syslog.conf /etc/rsyslog.conf 2>/dev/null
# 尝试写入应失败
echo "test" >> /etc/syslog.conf
```

期望结果：文件权限 400，写入被拒绝，日志服务正常读取。

---

### B15 开机启动 syslog 服务

**问题**：syslog 服务能在启动服务器时开启；配置 `*.err;kern.debug;daemon.notice; /var/adm/messages`。

#### 🔍 风险原理

**日志服务未随系统启动 = 重启后审计中断**。攻击者只需重启主机即可让日志服务停止采集，之后的操作不再留痕。因此必须开机自启且状态受监控。

#### 配置/加固方法

**1. 配置本地系统日志**

`/etc/syslog.conf`（原表要求）：

```
*.err;kern.debug;daemon.notice     /var/adm/messages
```

> **`/var/adm/messages` 是 SUSE/RHEL 6 系的路径**；CentOS 7 系用 `/var/log/messages`。填表时按实际发行版调整。

**2. 确认服务已启动**

```bash
ps -eaf | grep syslog
# CentOS 7 系
systemctl status rsyslog
systemctl is-enabled rsyslog      # 应为 enabled（开机自启）
# 凝思 Linx / RHEL 6 系
ps -eaf | grep syslogd
startsrc -s syslogd              # 启动
stopsrc -s syslogd               # 停止
chkconfig --list syslog          # 确认自启状态
```

**3. 设置开机自启**

```bash
systemctl enable rsyslog          # CentOS 7 系
chkconfig syslog on               # 凝思 Linx / RHEL 6 系
```

#### 特殊情况处理

- **`startsrc`/`stopsrc` 是 SUSE 系脚本命令**，CentOS 上不存在，须用 `systemctl`；
- **`/var/adm/messages` 与 `/var/log/messages` 不要同时配错路径**，会导致日志写到不存在的目录而丢失；
- 确认磁盘空间（`/var` 分区），日志写满会导致服务异常；
- 重启验证一次是唯一可靠办法（B16 的目的即此）。

#### 📌 加固后的验证

```bash
systemctl is-enabled rsyslog     # enabled
systemctl status rsyslog         # active (running)
ls -l /var/log/messages /var/adm/messages 2>/dev/null
# 重启后再次确认
```

期望结果：服务开机自启且运行中，本地日志文件正常生成。

---

### B16 日志记录满足 6 个月

**问题**：日志记录满足 6 个月。

#### 🔍 风险原理

6 个月是电力监控系统**安全审计的最低留存期**（与 A10 的"6 个月"同源）。保留期不足则事件回溯窗口不够，取证与合规审计无法通过。

> ⚠️ **本项原表无任何具体配置**，只有一句"日志记录满足 6 个月"。这是原表最不完整的一条。

#### 配置/加固方法

本机日志难以靠 `logrotate` 留存 6 个月（单个文件会很大），**须结合集中留存**：

**1. 本机 logrotate**

```
# /etc/logrotate.d/rsyslog
/var/log/messages /var/log/secure {
    weekly                 # 每周轮转
    rotate 26              # 保留 26 周 ≈ 6 个月
    compress               # 压缩归档
    delaycompress
    missingok
    notifempty
    create 0600 root root
    sharedscripts
    postrotate
        systemctl kill -HUP rsyslog 2>/dev/null || true
    endscript
}
```

> `rotate 26`（周）≈ 182 天。**具体周数按现场日志产生量与磁盘容量核算后调整**。

**2. 集中留存（本项目主路径）**

配合 B13 将日志外送到安管平台/态势感知，由平台按 6 个月策略留存。**这是唯一能真正满足 6 个月且抗篡改的方式。**

```bash
# 检查日志量与磁盘
du -sh /var/log
df -h /var
```

#### 特殊情况处理

- **`rotate 26` 会占用可观的磁盘空间**，须先评估日志产生速率；主机日志量大时应缩短本机周期、以集中留存为主；
- **压缩后的历史日志仍可检索**（`zcat`/`zgrep`），但检索能力弱于平台；
- **`rotate` 与 `maxsize` 冲突时以 `maxsize` 优先**，须明确使用哪一个；
- 本项无明确配置值时，**须在填表时记录实际采用的策略与周数**，并向电网公司确认是否认可。

#### 📌 加固后的验证

```bash
ls -l /var/log/messages.* /var/log/secure.* 2>/dev/null   # 确认历史轮转文件存在
grep -n "rotate" /etc/logrotate.d/rsyslog                 # 确认保留周数
# 平台侧确认历史日志可查询范围 ≥ 6 个月
```

期望结果：日志留存策略生效，历史日志可回溯 ≥ 6 个月。

---

### B17 禁用不必要的系统服务

**问题**：备份后编辑注释不需要的服务；`systemctl | grep active` 或 `chkconfig --list` 查看开放的服务。

#### 🔍 风险原理

每个运行的服务都是攻击面。尤其**明文协议服务**（telnet、rlogin、rsh、ftp）与**不必要的老旧服务**（tftp、sendmail、pop3、ntalk）是典型的后门入口。

#### 配置/加固方法

禁用清单：`telnet sendmail klogin rlogin kshell ntalk tftp imap pop3`

**1. inetd 管理的服务**

```bash
cp /etc/inet/inetd.conf /etc/inet/inetd.conf.backup
vi /etc/inet/inetd.conf
# 将不需要的服务行注释掉
# telnet stream tcp nowait root /usr/sbin/in.telnetd
# rlogin stream tcp nowait root /usr/sbin/rlogind
# tftp dgram udp wait root /usr/sbin/in.tftpd
# sendmail stream tcp nowait root /usr/sbin/sendmail -i
```

```bash
pkill -HUP inetd        # 重启 inetd 生效
```

> `/etc/inet/inetd.conf` 是 **SUSE/RHEL 6 系**的路径；CentOS 7 系改用 systemd 单元。

**2. systemd 管理的服务（CentOS 7 系）**

```bash
# 查看当前运行中的服务
systemctl list-units --type=service --state=running
systemctl | grep active

# 禁用
systemctl stop telnet
systemctl disable telnet
systemctl mask telnet          # 彻底禁止启动（mask 最彻底）
```

**3. xinetd（CentOS 6/7 常见）**

```bash
chkconfig --list | grep -E "on"
chkconfig telnet off
chkconfig rsh off
chkconfig rlogin off
```

#### 特殊情况处理

- **`sendmail` 关闭前确认该主机不承担邮件服务**（有些运维通知依赖本机 mail）；
- **`mask` 后要恢复须先 `unmask`**，误用会阻止服务启动；
- **`disable` 只取消自启不停止当前进程**，须同时 `stop`；
- **inetd/xinetd 不存在时服务由 systemd 管**，用错命令会报"command not found"，此时用 systemd；
- 禁用服务前先 `netstat -tlnp` 记录基线，禁用后逐一确认端口已消失。

#### 📌 加固后的验证

```bash
systemctl list-units --type=service --state=running   # CentOS 7 系
chkconfig --list | grep on                           # 凝思 Linx / RHEL 6 系
netstat -tlnp                                        # 确认无 23/25/69/110/143 监听
```

期望结果：清单内服务均未运行且未自启，无相关端口监听。

---

### B18 防止 ICMP 重定向报文攻击

**问题**：备份后执行 `sysctl -w net.ipv4.conf.all.accept_redirects="0"`，并将 `/proc/sys/net/ipv4/conf/all/accept_redirects` 的值改为 0。

#### 🔍 风险原理

ICMP 重定向报文可**动态改变本机路由表**。攻击者在同网段发送伪造的重定向报文，即可把流量导向自己的主机，实施中间人窃听。接受重定向等于允许路由表被外部改变。

> **与 Windows A20 对应**：A20 的 `EnableICMPRedirect=0` 与本项是同类防护，两端一致。

#### 配置/加固方法

```bash
# 1. 临时生效（立即）
sysctl -w net.ipv4.conf.all.accept_redirects=0
sysctl -w net.ipv4.conf.default.accept_redirects=0

# 2. 永久生效（写入配置文件）
echo "net.ipv4.conf.all.accept_redirects = 0"     >> /etc/sysctl.conf
echo "net.ipv4.conf.default.accept_redirects = 0" >> /etc/sysctl.conf

# 3. 加载
sysctl -p

# 4. 确认（/proc 与 sysctl 两条路径一致）
sysctl net.ipv4.conf.all.accept_redirects
cat /proc/sys/net/ipv4/conf/all/accept_redirects     # 应为 0
```

> 原表要求同时改 `sysctl -w`（运行时）与 `/proc/sys/...`（内核接口）。实际上 `/proc/sys/net/ipv4/conf/all/accept_redirects` 就是 `sysctl net.ipv4.conf.all.accept_redirects` 的底层，**改 `sysctl -w` 即写 `/proc`**，无需分别操作；**关键是写入 `/etc/sysctl.conf` 使其重启后仍生效**。

**补充**：同时建议关闭发送重定向与源路由：

```bash
echo "net.ipv4.conf.all.send_redirects = 0"      >> /etc/sysctl.conf
echo "net.ipv4.conf.all.accept_source_route = 0" >> /etc/sysctl.conf
```

#### 特殊情况处理

- **在多网卡/多路由的服务器上关闭重定向通常无副作用**；但在依赖 ICMP 重定向优化路由的罕见环境中关闭后需手工配置静态路由；
- **改 `sysctl.conf` 前先备份**，参数写错可能导致网络异常（如 `net.ipv4.ip_forward` 被误改）；
- 用 sysctl 关闭的策略**重启后失效**，必须写配置文件；
- Docker/容器宿主机上需额外处理容器网桥的 `accept_redirects`，见 B18 补充说明。

#### 📌 加固后的验证

```bash
sysctl net.ipv4.conf.all.accept_redirects
sysctl net.ipv4.conf.default.accept_redirects
grep accept_redirects /etc/sysctl.conf
```

期望结果：运行时与配置均为 0，重启后依然为 0。

---

### B19 SNMP、SNMP TRAP 团体名称设置

**问题**：`/etc/snmp/snmpd.conf` 修改默认团体名，**团体名改为 `GXhmrjsw-q3`**；`get-community-name` 将默认 `public` 修改为复杂团体名。

#### 🔍 风险原理

SNMP v1/v2c 的团体名是**明文密码**，默认 `public` 全球已知。攻击者用它可读取全网系统信息、拓扑、接口状态；v2c 写权限更可直接改配置。

#### 配置/加固方法

**1. 修改只读团体名**

```
# /etc/snmp/snmpd.conf
# 语法：rocommunity <团体名> <来源IP/网段>
# 删除或注释掉所有 public 行，改为：
rocommunity GXhmrjsw-q3 <态势感知IP>
```

**2. 写团体名（若平台需要写）**

```
rwcommunity GXhmrjsw-q3 <堡垒机IP>
```

**3. SNMP TRAP 目标**

```
# trap 目标（注意：陷阱目标不需要 rocommunity）
trapsink <态势感知IP>
trapcommunity GXhmrjsw-q3
```

**4. 重启服务**

```bash
systemctl restart snmpd         # CentOS 7 系
service snmpd restart           # 凝思 Linx / RHEL 6 系
snmpwalk -v2c -c public localhost          # 用默认团体名——应失败
snmpwalk -v2c -c GXhmrjsw-q3 <本机IP>      # 用新团体名——应成功
```

#### 特殊情况处理

- **本项目硬绑定**：团体名必须是 **`GXhmrjsw-q3`**，与 Windows A12、防火墙 C12、交换机 D15 一致。**不要使用华为设备的 `public`/私用团体名默认值**，见 [../09-文件间参数差异对照.md](../09-文件间参数差异对照.md)；
- **禁用 SNMP 是更彻底的方案**：若安管平台可用其他方式采集（如 syslog），可直接 `systemctl disable --now snmpd`，此时 B19 填"未启用"；
- **`snmpd.conf` 中可能有多处 `rocommunity`**，必须全部替换，遗留一行 `public` 即留后门；
- 改团体名后**平台侧须同步更新**，否则监控中断；
- `trapcommunity` 与 `rocommunity` 可用不同值，但本项目统一。

#### 📌 加固后的验证

```bash
grep -nE "rocommunity|rwcommunity|trapcommunity|public" /etc/snmp/snmpd.conf
snmpwalk -v2c -c public localhost .1.3.6.1.2.1.1.1.0    # 应失败
snmpwalk -v2c -c GXhmrjsw-q3 localhost .1.3.6.1.2.1.1.1.0  # 应成功
```

期望结果：无 `public` 残留；仅授权 IP 可用正确团体名访问。

---

### B20 系统时间同步

**问题**：系统时间与 NTP 服务器同步，`ntpdate 3.1.1.4`。

#### 🔍 风险原理

时间不准会同时破坏三件事：日志溯源的时间基准（A10/B12/B16）、证书与加密隧道校验（E4）、跨设备事件关联分析。**时间戳错乱的日志在审计中基本不可用。**

#### 配置/加固方法

**方式一：`ntpdate`（一次性同步，RHEL 6 系常用）**

```bash
ntpdate 3.1.1.4              # 原文指定的 NTP 服务器
hwclock -w                    # 同步到硬件时钟（虚拟机可跳过）
```

> **`ntpdate` 是"一次性对齐"，会跳变时间**，对运行中的数据库、中间件可能导致异常。**生产环境建议改用常驻服务**（方式二）。

**方式二：`chronyd`（CentOS 7 系推荐）**

```bash
# /etc/chrony.conf
server 3.1.1.4 iburst

systemctl enable chronyd
systemctl restart chronyd
chronyc sources -v
chronyc tracking
```

**方式三：`ntpd` 常驻（RHEL 6 系）**

```
# /etc/ntp.conf
server 3.1.1.4 iburst
service ntpd restart
ntpq -p                       # 查看同步状态
chkconfig ntpd on
```

**确认时区**

```bash
timedatectl                   # CentOS 7 系
cat /etc/timezone             # 应为 Asia/Shanghai
date                          # 确认 UTC+8
```

#### 特殊情况处理

- **Windows A24 未指明 NTP 地址**，本表 B20 给出 `3.1.1.4`，**两端是否同一时间源须向运维确认**；
- **必须同时确认时区为 UTC+8**，否则"同步成功但时间错误"；
- `ntpdate` 在有业务运行的库上有风险（时间跳变），生产环境优先 chronyd/ntpd 常驻；
- **B20 是 E2 日志时间戳、E4 证书校验的前置条件**，应优先完成；
- 虚拟机的时间由宿主机传递时，虚拟机内配置 NTP 仍需确认实际生效。

#### 📌 加固后的验证

```bash
chronyc tracking            # 或 ntpq -p
timedatectl | grep "Time zone"
date
```

期望结果：时间源显示为配置的 NTP 服务器，偏差为 0，时区为 Asia/Shanghai。

---

### B21 Umask 权限

**问题**：控制用户缺省访问权限 UMASK 为 **022**。

#### 🔍 风险原理

`umask` 决定**新创建文件/目录的默认权限**。`0000`（全开放）意味着新建文件任何人可读写；`0077` 过于严格会导致服务账号无法共享文件。`022` 是通用折中：属主可写、其他人只读可执行。

#### 配置/加固方法

```bash
vi /etc/profile
# 全局默认
umask 022
```

或写入独立文件便于管理：

```bash
echo 'umask 022' > /etc/profile.d/umask.sh
chmod 644 /etc/profile.d/umask.sh
```

```bash
umask                          # 应输出 0022
```

**关键服务账号应单独设置**：

```bash
vi ~/.bashrc                   # 应用服务账号的家目录
umask 0027
```

#### 特殊情况处理

- **umask 022 对应创建文件权限 644、目录 755**——若业务要求数据库数据文件更严（如 600），需为该服务账号单独设 `umask 0077`；
- **改全局 umask 可能影响应用创建的文件权限**，数据库/中间件的运行账号应单独设置而非改全局；
- **root 的 umask 通常已是 022**，普通用户登录 shell 未必如此，需逐账号确认；
- 挂载点、SAMBA 共享等场景 umask 另有规则。

#### 📌 加固后的验证

```bash
umask                          # 应为 0022
# 实测
touch /tmp/_t && ls -l /tmp/_t && rm -f /tmp/_t     # 应为 -rw-r--r--
mkdir -p /tmp/_d && ls -ld /tmp/_d && rmdir /tmp/_d # 应为 drwxr-xr-x
```

期望结果：umask 为 0022，新建文件 644、目录 755。

---

### B22 敏感文件安全保护

**问题**：`chown root` + `chmod 644/400` 加固 `/etc/passwd`、`/etc/shadow`、`/etc/group`。

#### 🔍 风险原理

`/etc/shadow` 保存**所有用户的口令哈希**，权限不当时任何本地用户可读取并离线爆破。`/etc/passwd` 与 `/etc/group` 则暴露系统账号结构。

#### 配置/加固方法

```bash
chown root /etc/passwd /etc/shadow /etc/group
chmod 644 /etc/passwd /etc/group      # 属主可写、其他人只读
chmod 400 /etc/shadow                # 仅属主可读
```

```bash
ls -l /etc/passwd /etc/shadow /etc/group
# 应为：
# -rw-r--r-- 1 root root /etc/passwd
# -r-------- 1 root root /etc/shadow
# -rw-r--r-- 1 root root /etc/group
```

> 注意 `/etc/gshadow` 与 `/etc/shadow` 同级敏感，应一并确认权限（通常 400）。

#### 特殊情况处理

- **`chmod 400 /etc/shadow` 后 PAM 仍可读取**（以 root 运行），业务不受影响；
- **某些老旧脚本用非 root 账号改口令会失败**，属预期行为；
- **`/etc/passwd` 必须保持 644 可读**，否则大量系统工具（`getent`、`lslogin`）异常；
- 修改属主/权限前备份：`cp -p /etc/shadow /etc/shadow.bak`；
- 备份文件 `shadow.bak`（含口令哈希）权限应同样收紧。

#### 📌 加固后的验证

```bash
ls -l /etc/passwd /etc/shadow /etc/group
# 用普通用户尝试读取 shadow（应失败）
su - opsuser -c "cat /etc/shadow"
```

期望结果：权限符合要求，普通用户无法读取 `/etc/shadow`。

---

### B23 Root 操作历史

**问题**：减少 root 输入命令的历史记录长度，保留最新执行的 **5 条**命令。`vi /etc/profile` 修改 `HISTSIZE=5`。

#### 🔍 风险原理

历史记录是**取证线索也是泄露源**。保留过多会记录含口令的命令（`mysql -p`、`curl -u user:pass`）明文；保留过少则不利于追溯。`HISTSIZE=5` 是"保留最少可用记录"的取舍。

#### 配置/加固方法

```bash
vi /etc/profile
HISTSIZE=5
HISTFILESIZE=5                # 已保存的历史文件条数
```

```bash
# 确认
echo $HISTSIZE
history | wc -l
```

**全系统范围**：

```bash
grep -r "^HISTSIZE" /etc/profile /etc/profile.d/ /root/.bashrc 2>/dev/null
```

#### 特殊情况处理

- **root 的历史记录实际在 `/root/.bash_history`**，仅改 `/etc/profile` 的 `HISTSIZE` 会**在新登录 shell 中生效**，但历史写入文件时的条数由 `HISTFILESIZE` 控制，**两个都要改**；
- **正在运行的 shell 需 `exec bash` 或重新登录才生效**；
- **`HISTSIZE=5` 极短，实用性有限**。实际运维中更常见的做法是 `HISTSIZE=1000`（保留足够追溯） + `HISTCONTROL=ignorespace`（不记录以空格开头的命令）+ `set +o history`（临时关闭记录）。**本表要求 5 条，按原表执行，但须告知运维追溯能力受限**；
- **含口令的命令即使删了历史也可能已落盘**，更根本的做法是用 `--defaults-file=` 或环境变量传口令而非命令行。

#### 📌 加固后的验证

```bash
grep -n "HISTSIZE\|HISTFILESIZE" /etc/profile
history | wc -l            # 应不超过 5
```

期望结果：历史记录条数受限。

---

### B24 查看端口进程并结束

**问题**：加固后 `netstat -anp | grep 端口号`；添加防火墙出入站策略禁用高危端口；进程仍运行则 `kill -9 进程号` 强制结束，重启后再次检查。

#### 🔍 风险原理

**禁用服务配置不等于进程已停**，也不等于端口已关闭。必须以"端口实际不监听"为验收标准。

#### 配置/加固方法

**1. 端口清单**

| 类别 | 端口 |
| --- | --- |
| 必须禁用 | 135、137、138、139、445、3389、`5323333` |
| 原则上禁用 | 23（telnet）、25（SMTP）、20、21（ftp）、513（login）、25、110（e-mail）、1900（UPnP） |

> ⚠️ **原表将 `23333` 误写为 `5323333`**，正确值为 **`23333`**（与 Windows A26 一致），见第七节。

**2. 排查监听进程**

```bash
netstat -anp | grep 23333
# 或
ss -tlnp | grep 23333
lsof -i :23333
```

**3. 结束进程**

```bash
kill -9 <进程号>
```

**4. 防火墙出入站策略**

```bash
# CentOS 7 系（firewalld）
firewall-cmd --permanent --add-port=23333/tcp        # 允许（本项目远程管理端口）
firewall-cmd --permanent --remove-port=23/tcp
firewall-cmd --permanent --remove-port=25/tcp
firewall-cmd --permanent --remove-port=21/tcp
firewall-cmd --reload
firewall-cmd --list-all

# CentOS 7 系（iptables 底层）
iptables -A INPUT -p tcp --dport 23 -j DROP
iptables -A OUTPUT -p tcp --dport 23 -j DROP
# 允许堡垒机访问管理端口
iptables -A INPUT -p tcp --dport 23333 -s <堡垒机IP> -j ACCEPT
iptables -A INPUT -p tcp --dport 23333 -j DROP
iptables-save > /etc/sysconfig/iptables      # 持久化

# 凝思 Linx / RHEL 6 系（仅 iptables）
service iptables save
```

> **RHEL 6 / CentOS 6 的默认防火墙策略是 `INPUT ACCEPT`（全放行）**，仅写 DROP 规则等于没拦；必须先把 `INPUT` 默认策略改为 `DROP` 再逐条放通。

**5. 重启后复查**

```bash
reboot
netstat -anp | grep -E "23|25|21|23333"    # 应无输出
```

#### 特殊情况处理

- **`kill -9` 直接杀进程会导致服务不自启**，正确做法是 `systemctl stop` 后再 `disable`（见 B17）；
- **禁用 53（DNS）会破坏解析**，除非纯静态 IP 且不用域名；
- **端口被占用可能是业务必需**（如某些中间件监听 8443），先确认归属再处置；
- 防火墙规则变更须在窗口期执行，并保留回滚（`iptables-save` 备份）；
- **容器/虚拟机端口转发**（Docker `-p`、libvirt）需在宿主机层面核查，主机内 `iptables` 未必拦得住。

#### 📌 加固后的验证

```bash
netstat -anp | grep -E ":(23|25|21)\b"     # 应无输出
# 从非授权主机测试
telnet <主机IP> 23                          # 应不通
```

期望结果：清单端口无监听，重启后仍无监听。

---

### B25 物理封堵多余 USB 接口

**问题**：除鼠标键盘等必要外设接入 USB 外，其余 USB 口用封条封死（原则上保留一个数据拷贝接口）。

#### 🔍 风险原理

Linux 主机同样是移动介质投毒目标。封条提供**软件手段无法绕过的物理管控**，与 A23 形成同一治理思路。

#### 配置/加固方法

1. 识别主机 USB 口，区分必要外设口与多余口；
2. 用防拆封条封死多余 USB 口，登记封条编号；
3. **原则上保留 1 个受控数据拷贝接口**（用于导入补丁/配置文件），使用时须审批并经 B1 的 U 盘查杀。

**配合软件侧（可选，增强）**：

```bash
# 禁用 USB 存储设备（需 usb-storage 模块支持）
echo "blacklist usb-storage" >> /etc/modprobe.d/blacklist-usb.conf
modprobe -r usb_storage
```

```bash
lsmod | grep usb_storage       # 应无输出
dmesg | tail -20               # 插入 U 盘应有拒绝记录
```

> `blacklist usb_storage` 会**完全禁用 U 盘存储**，键盘/鼠标（`usbhid`、`usbserial`）不受影响。**启用前确认保留的数据拷贝接口需求**——两者冲突，按现场选择。

#### 特殊情况处理

- **物理封堵与 `blacklist usb_storage` 二选一或叠加**：叠加则软件层面也不可读，封条是最后一道防线；
- 关闭 USB 存储后**安装系统/驱动时无法使用 U 盘**，须提前准备介质或临时解除；
- 部分服务器有虚拟 USB（用于远程挂载 ISO），一并评估；
- 封条须登记编号，破损立即上报。

#### 📌 加固后的验证

```bash
lsmod | grep usb_storage       # 应无输出（若已 blacklist）
# 目视检查封条完好、编号清晰
```

期望结果：多余接口已封堵/禁用，保留接口处于受控状态。

---

### B26 卸载 samba、删除默认路由

**问题**：卸载 samba；删除默认路由 0.0.0.0；配置指向 `10.67.x.0` 的路由，下一条是默认网关；禁止存在默认路由，必须全为明细路由，路由掩码应 **≥24 位**。路由记得保存，不然重启会丢失。

> ⚠️ **参考文档不一致**：[AGCAVC1 序号42.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号42.md>) 的 B26 是"**调度发令系统主机的特有加固**"，与本表 B26"卸载 samba + 删除默认路由"**不是同一项**。该文档的 B26 **不可直接用于本表**，须以本表原文为准。

#### 🔍 风险原理

**Samba** 实现 SMB，历史上是 Windows/Linux 横向移动的主通道（如 EternalBlue），且常与 `net use` 空连接组合攻击。

**默认路由（0.0.0.0/0）** 意味着任何未被明确路由的目的地址都会被转发。安全区之间靠明细路由精确控制可达范围——没有默认路由，跨区访问必须逐条配置，不存在"漏一条就全通"。

#### 配置/加固方法

**1. 卸载 samba**

| 发行版 | 查看 | 卸载 |
| --- | --- | --- |
| CentOS | `rpm -qa \| grep samba` | `rpm -e --nodeps samba` |
| Ubuntu / Debian | `dpkg -l \| grep samba`、`dpkg -l \| grep smbfs`、`dpkg -l \| grep smb` | `apt-get remove samba` |

```bash
# 停止服务
service smbd stop
# 或
/etc/rc.d/init.d/smbd stop
# 或
/etc/init.d/smbd stop

# 确认已卸载
rpm -qa | grep samba        # 应无输出
netstat -tlnp | grep -E ":(139|445)\b"   # 应无输出
```

**2. 查看并删除默认路由**

```bash
route -n                                  # 查看路由，确认默认路由对应网卡（原文示例 eth0）
# 或
ip route show
```

删除默认路由：

```bash
route del default
# 或（CentOS 7 系）
ip route del default via <网关IP> dev eth0
```

**3. 配置明细路由（掩码 ≥24 位）**

> ⚠️ **先确认发行版再选方法**。`/etc/sysconfig/*` 属于 **RHEL / CentOS / 凝思 Linx** 家族；**Ubuntu / Debian 没有 `/etc/sysconfig` 目录**，照抄 RHEL 的路径不会生效，必须用下面的 netplan 或 `/etc/network/interfaces`。

**RHEL / CentOS / 凝思 Linx 系**（`/etc/sysconfig` 家族）：

```bash
# 方法 1：命令行临时添加（重启即失效，仅用于验证）
route add -net 10.67.x.0/24 dev eth0
route add -net 10.67.x.0/24 gw 10.67.1.1

# 方法 2：/etc/sysconfig/static-routes（推荐持久化，不与 GATEWAY 冲突）
echo "any net 10.67.x.0/24 gw 10.67.1.1" >> /etc/sysconfig/static-routes

# 方法 3：/etc/sysconfig/network 末尾设 GATEWAY
# ⚠️ 会产生默认路由，与"禁止默认路由"冲突，仅在无其他要求时使用
GATEWAY=gw-ip
```

**Ubuntu / Debian 系（无 /etc/sysconfig，必须用 netplan 或 interfaces）**：

```bash
# 方法 A：netplan（Ubuntu 18.04+ 首选）——编辑 /etc/netplan/01-eth0.yaml
#   network:
#     version: 2
#     ethernets:
#       eth0:
#         addresses:
#           - 10.67.x.10/24
#         routes:                          # 明细路由
#           - to: 10.67.x.0/24
#             via: 10.67.1.1
#         # 注意：不配置 gateway4，即不产生默认路由
netplan try      # 校验语法，失败自动回滚
netplan apply    # 生效；失败用 netplan revert 回滚
```

```bash
# 方法 B：/etc/network/interfaces（ifupdown 管理的系统）
#   auto eth0
#   iface eth0 inet static
#       address 10.67.x.10/24
#       up route add -net 10.67.x.0/24 gw 10.67.1.1 dev eth0
#       # 不写 gateway 关键字即不产生默认路由
ifdown eth0 && ifup eth0    # 生效；远程操作建议先开一个保持连接
```

**NetworkManager 通用（RHEL 8+/CentOS 7+/Ubuntu 桌面版均可）**：

```bash
nmcli connection show eth0
nmcli connection modify eth0 +ipv4.routes "10.67.x.0/24 10.67.1.1"
nmcli connection modify eth0 ipv4.never-default yes   # 不使用默认路由
nmcli connection up eth0
```

> 💡 `ipv4.never-default yes` 是"禁止默认路由"在 NetworkManager 下的正确表达，比注释掉 `GATEWAY=` 更可靠；netplan 则是**完全不写 `gateway4`**。

**4. 保存配置（持久化）**

路由持久化**已由上一步的配置文件负责**，改完配置文件即已持久化，重启自动加载，无需额外操作：

| 发行版 | 持久化文件 | 生效方式 |
| --- | --- | --- |
| RHEL / CentOS / 凝思 Linx | `/etc/sysconfig/static-routes` | `systemctl restart network` |
| Ubuntu 18.04+（netplan） | `/etc/netplan/*.yaml` | `netplan apply`（先用 `netplan try` 校验） |
| Ubuntu / Debian（ifupdown） | `/etc/network/interfaces` | `ifdown eth0 && ifup eth0` |
| NetworkManager | `nmcli connection` | `nmcli connection up eth0` |

> ⚠️ **`/etc/rc.local` 仅是 RHEL 家族的遗留兜底方式，且已被 systemd 标记为 deprecated**：RHEL 7 须 `chmod +x /etc/rc.local` 并 `systemctl enable rc-local`，Ubuntu 默认不执行该文件。**加固文档不应把 rc.local 作为路由持久化的推荐手段**，此处仅在无上述配置文件能力时参考。
>
> ⚠️ **远程操作注意**：删除默认路由、`ifdown/ifup` 断网卡、netplan 改错配置都会立即断开会话。**必须先开一个 SSH 会话保持连接，或在带外/console 上操作**，并优先用 `netplan try` 这类会自动回滚的方式。

#### 特殊情况处理

- **删除默认路由会中断本机对外访问**（DNS、NTP、补丁源、态势感知上报），**须先确认明细路由齐备再删，删后立即验证**；
- **`route add` 是临时的**，重启即失效——**原表特别强调"路由记得保存，不然重启会丢失"**；
- **`GATEWAY=` 只能有一条**，配了 `GATEWAY=` 即等于有默认网关，与"禁止默认路由"冲突。**要精确控制可达范围就用 `static-routes` 或 nmcli routes，不要用 `GATEWAY=`**；
- **删除 `GATEWAY=` 行**（原文做法，**仅适用于 RHEL / CentOS / 凝思 Linx**）：
  ```bash
  vi /etc/sysconfig/network-scripts/ifcfg-eth0
  # 在 GATEWAY=xxxx 行前加 # 注释该行，wq 保存
  # Ubuntu/Debian 无此文件：netplan 删掉 gateway4，ifupdown 删掉 gateway 关键字
  systemctl restart network        # CentOS 7 用 service network restart
  ```
- **与 Windows A28 的差异**：Linux 要求掩码 **≥24 位**并显式配置 `10.67.x.0` 明细路由；Windows 为 **≥16 位**。**填表时不要互相套用**；
- 删除默认路由后若 DHCP 下发网关，会重新出现默认路由——须改为静态配置。

#### 📌 加固后的验证

```bash
# 1. samba 已卸载（按发行版二选一）
rpm -qa | grep -Ei "samba|smbfs"          # RHEL/CentOS/凝思：应无输出
dpkg -l | grep -Ei "samba|smbfs|smbd"     # Ubuntu/Debian：应无输出
# 端口无监听（通用，两种发行版均可用；ss 优先，netstat 兼容老系统）
ss -lntup | grep -E ":(139|445)\b" && echo "仍存在监听，需处理" || echo "139/445 无监听：OK"
netstat -tlnp 2>/dev/null | grep -E ":(139|445)\b"   # 老系统兜底

# 2. 无默认路由
ip route show | grep -E "^default" && echo "仍存在默认路由，需处理" || echo "无默认路由：OK"

# 3. 明细路由掩码 ≥24 位
ip route show | grep -E "^10\.67\..*/(2[4-9]|3[0-2]) "   # 应存在且掩码 ≥/24

# 4. 持久化配置已生效（核对与实际生效路由一致）
grep -vE "^\s*#|^\s*$" /etc/sysconfig/static-routes 2>/dev/null   # RHEL/CentOS/凝思
grep -A5 "routes:" /etc/netplan/*.yaml 2>/dev/null                # Ubuntu netplan

# 5. 重启后复查（关键，原表强调"路由记得保存，不然重启会丢失"）
reboot && ip route show
```

期望结果：samba 已卸载且 139/445 无监听；**无默认路由**；`10.67.x.0/24` 明细路由掩码 ≥/24 且**重启后仍然存在**；持久化配置文件中的条目与实际生效路由一致。

> ⚠️ 远程主机执行第 5 步前，先确认第 3、4 步均已通过。删除默认路由后若明细路由有遗漏，主机会立即失联且**无法自救**，须在带外/console 或保留的 SSH 会话中操作。

## 四、漏洞处置清单（14 条）

原表 Linux 表内列出调度指令类 7 条 + 信息通报类 7 条。

### 调度指令类（7 条）

| 编号 | 漏洞名称 | Linux 处置方法 |
| --- | --- | --- |
| 调度 20190615-001 | WebLogic 中间件 async 漏洞 | `find / -name wls9_async_response.war`、`find / -name wls-wsat.war`；查到后 `mv wls9_async_response.war wls9_async_response.war.bak`、`mv wls-wsat.war wls-wsat.war.bak`；重启服务。查找不到说明未采用 |
| 调度 20190617-001 | Apache axis 漏洞 | `find / -name server-config.wsdd`；`cp server-config.wsdd server-config.wsdd.bak`；`vim server-config.wsdd` 将 `enableRemoteAdmin` 改为 false；重启服务 |
| 调度 20190618-002 | "帆软"报表组件漏洞 | 存在高危漏洞且暂无补丁，先卸载。`find / -name *FineReport*` 定位后卸载 |
| 调度 20190619-001 | WebLogic 中间件反序列化漏洞 CVE-2019-2729 | 同 async 漏洞处置：禁止访问 `/_async/*`、`/wls-wsat/*`，删除或改名两个 war 文件，重启服务 |
| 调度 20190622-001 | 网站安全狗（Apache 版）V4.0.23957 webshell 绕过漏洞 | `find / -name *safedog*` 定位后处置 |
| 调度 20190626-001 | JBOSS 中间件管理后台 web-console 或 jmx-console 漏洞 | 浏览器访问 `XX.XX.XX.XX:8080/web-console` 或 `/jmx-console` 确认；`find / -name "web-console.war"`、`find / -name "jmx-console.war"`；返回路径为 `/usr/ems/jboss/WEB-INF/` 时直接 `rm /usr/ems/jboss/WEB-INF/web-console.war`、`rm /usr/ems/jboss/WEB-INF/jmx-console.war`；再次访问返回不存在页面即成功 |
| 调度 20190626-002 | IBM Websphere 中间件低版本漏洞 CVE-2019-4279 | `find / -name "versionInfo.sh"`；已知安装目录（`/opt/IBM/WebSphere/AppServer/bin`）可直接 `cat versionInfo.sh` 查看版本号与 Package 日期，低于 20190503 说明存在安全风险 |

### 信息通报类（7 条）

| 编号 | 漏洞名称 | 影响范围与处置 |
| --- | --- | --- |
| 信息通报 36 期 | Linux 内核 TCP SACK | 内核 2.6.29 及之后版本处理 TCP SACK 时存在整数溢出漏洞，攻击者可构造特定 SACK 包远程触发内核模块溢出，实现远程拒绝服务。整改：`sysctl -w net.ipv4.tcp_sack=0` 并在 `/etc/sysctl.conf` 写入 `net.ipv4.tcp_sack=0`；或升级内核 |
| 信息通报 38 期 | Apache Tomcat 中间件 DoS 漏洞 | 受影响 `0.0.M1 < Tomcat < 9.0.14`、`5.0 < Tomcat < 8.5.37`。执行 `version.sh` 或 `find / -name version.sh` 确认版本后更新 |
| 信息通报 39 期 | Firefox 远程代码执行漏洞（低于 67.0.3） | 卸载所有低于 Firefox 67.0.3 和 Firefox ESR 60.7.1 的版本，或升级到 67.0.4 |
| 信息通报 41 期 | Vim 和 Neovim 编辑器任意代码执行漏洞 | 影响 Vim 8.1.1365 与 Neovim 0.3.6 之前所有版本，建议卸载：`yum -y remove vimz` |
| 信息通报 20-1 期 | Apache Tomcat 中间件 AJP 协议漏洞 | AJP Connector 协议设计缺陷，可读取或包含 webapp 目录下任意文件，配合文件上传可远程代码执行。受影响：Apache Tomcat 6、Tomcat 7 < 7.0.100、Tomcat 8 < 8.5.51、Tomcat 9 < 9.0.31 |
| 信息通报 20-10 期 | WebLogic 中间件远程代码执行漏洞 CVE-2020-2798、2801、2828、2867 | 受影响：Oracle WebLogic Server 10.3.6.0.0、12.1.3.0.0、12.2.1.3.0、12.2.1.4.0、14.1.1.0.0 |
| 信息通报 20-9 期 | fastjson java 库执行漏洞（fastjson ≤ 1.2.67） | `find / -name "*fastjson*"` 定位；将低版本 jar 替换为 1.2.58 及以上；`cp fastjson-1.2.58.jar /users/ems/tomcat_osb/webapps/fservice/WEB-INF/lib/`、`rm /users/ems/tomcat_osb/webapps/fservice/WEB-INF/lib/fastjson-1.1.20.jar`；重启中间件服务 |

Linux 内核 TCP SACK 修复版本：

| 发行版 | 修复版本 |
| --- | --- |
| CentOS 6 | 6.32-754.15.3 |
| CentOS 7 | 10.0-957.21.3 |
| Ubuntu 18.04 LTS | 15.0-52.56 |
| Ubuntu 16.04 LTS | 4.0-151.178 |

> **信息通报 36 期（TCP SACK）补充**：这是**内核参数级缓解**，不是完整修复。设置 `net.ipv4.tcp_sack=0` 会**关闭 TCP SACK**，可能影响高延迟链路的性能；**优先升级内核到修复版本**，参数仅作为无法立即升级时的临时缓解。修改后须写入 `/etc/sysctl.conf` 持久化（与 B18 同理）。

## 五、必装软件

| 项 | 要求 | 对应项 |
| --- | --- | --- |
| 主机加固软件 | G01 或信达或优炫 | 见 [序号60.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号60.md>) |
| 防病毒软件 | 安天防病毒软件 | B1 |
| 堡垒机 | 是否接入堡垒机 | B3、B5 |
| **安管平台** | **SNMP、SYSLOG 接入安管平台** | B19（团体名 `GXhmrjsw-q3`）、B13（syslog 外送） |

Linux 表单比 Windows 表单多一项"SNMP、SYSLOG 接入安管平台"，即 Linux 主机除 SNMP 外还须接入 syslog，与 B13 的 `loghost` 转发配置配套。

安天杀毒软件下载地址：Ⅲ 区 `https://3.1.1.1`，Ⅰ/Ⅱ 区 `https://1.1.1.1`。

## 六、本项目相关

| 项 | 说明 |
| --- | --- |
| 防病毒系统 Linux 授权 | 清单序号 11，1 套含 10 个 Linux 服务器防护授权 + 3 年升级许可 |
| 主机加固系统 | 清单序号 10，1 项 |
| SNMP 硬绑定 | 团体名 **`GXhmrjsw-q3`**，与 Windows A12、防火墙 C12、交换机 D15 一致 |
| 远程管理端口 | **22222**（B3），仅堡垒机/态势感知可达 |
| 主机台账 | **待确认**。10 个 Linux 授权不等于 10 台主机，须向运维索取实际主机台账 |
| 发行版待确认 | **填表前必须确认每台主机的实际发行版**（CentOS 7 系 vs 凝思 Linx/RHEL 6 系），原表命令混用两套体系 |

## 七、原表不一致处

| 位置 | 不一致内容 | 处理建议 |
| --- | --- | --- |
| B24 | 端口列表写成 `5323333` | 正确为 **`23333`**，与 Windows A26 一致 |
| B3 / B26 | B3 说"路由掩码应 ≥24 位"写在 B26，B26 又重复 | 原文如此，按 B26 执行 |
| B7 vs A4 | Linux 锁定 30 分钟；Windows 锁定 5 分钟 | 各自按本 sheet 执行，不要互相套用 |
| B26 vs A28 | Linux 掩码 ≥24 位 + 显式 `10.67.x.0` 路由；Windows ≥16 位 | 各自按本 sheet 执行 |
| B20 vs A24 | Linux 用 `ntpdate 3.1.1.4`；Windows 未指明 NTP 地址 | **两端是否同一时间源须向运维确认**，否则跨设备日志时间轴不一致 |
| B12 | 只给 `/etc/syslog.conf` | **CentOS 7 系用 `/etc/rsyslog.conf`**，路径不同须替换 |
| B15 | 只给 `/var/adm/messages` 与 `startsrc/stopsrc` | `/var/adm/messages` 与 `startsrc` 均为 SUSE/RHEL 6 系路径与命令；**CentOS 7 系用 `/var/log/messages` 与 `systemctl`** |
| B2 | 含 `Protocol`、`RhostsAuthentication`、`RhostsRSAAuthentication` | **OpenSSH 7.x 起已废弃**，新版主机可忽略 |
| B6 | PAM 栈含 `nullok` | `nullok` 允许空口令通过 PAM；配合"空口令账户检查"清零后可用，否则应去掉 |
| B16 | 只有"日志记录满足 6 个月"一句，**无任何配置** | 须自行制定并记录策略（logrotate `rotate` 周数 + 集中留存），并向电网公司确认认可 |
| B24 | "路由记得保存，不然重启会丢失" 但 B26 的 `GATEWAY=` 用法与之冲突 | `GATEWAY=` 只能一条且等于默认网关，**要精确控制须用 `static-routes` 或 nmcli routes** |
| B4 | 示例命令带 `#` 前缀（`#userdel username`） | `#` 是注释符，实际执行时去掉 |
| B26 参考文档 | AGCAVC1 序号42.md 的 B26 是"调度发令系统主机的特有加固" | **与本表 B26 不是同一项，不可直接套用** |
| B3 | `hosts.allow`/`hosts.deny` 依赖 TCP Wrappers | 现代发行版部分场景需改用 `sshd_config` 的 `AllowUsers` 或防火墙 |
| sheet 版本列 | 表头"版本"列的 A1 内容里写的是"Ⅲ 区下载地址 https://3.1.1.1 / Ⅰ/Ⅱ 区 https://1.1.1.1" | 内容属 B1 病毒查杀，非版本信息，属模板填写错位 |
| B13 | "作业结点"与"加固方法"两列内容完全重复 | 无实质影响 |
| 异常记录示例 | B1 填"无法安装杀毒软件"，B22 行空白 | 属示例填写，非本项目记录 |

## 八、与 Windows 表单的参数差异

| 项 | Windows | Linux |
| --- | --- | --- |
| 默认路由掩码 | ≥16 位（255.255.0.0） | **≥24 位**，且须显式配置 `10.67.x.0` 网段路由 |
| 远程管理协议 | SSH、HTTPS | SSH，端口 22 改 **22222** |
| 远程管理访问控制 | 限制管理 IP | `/etc/hosts.allow` + `/etc/hosts.deny`（`all:all`） |
| root 远程登录 | 重命名 Administrator | `PermitRootLogin no` 或 `/etc/securetty` |
| 登录失败锁定 | 5 次 / 锁定 5 分钟 | 5 次 / 锁定 30 分钟（`unlock_time=1800`） |
| 会话超时 | 屏幕保护 + 强制注销 | `TMOUT=300`（对 root 无效） |
| 日志转发 | SNMP 接入态势感知 | SNMP + SYSLOG 接入安管平台 |
| USB | 封条封死 | 封条封死（可加 `blacklist usb_storage`） |
| SNMP 团体名 | `GXhmrjsw-q3` | `GXhmrjsw-q3`（一致） |
| 病毒查杀路径 | 控制面板/服务 | `/opt/LinuxKPC/ini/` |
| 日志配置文件 | 事件查看器 | `/etc/syslog.conf`（RHEL 6）/ `/etc/rsyslog.conf`（CentOS 7） |
| 服务管理 | `services.msc` | `systemctl`（CentOS 7）/ `chkconfig`、`startsrc`（RHEL 6） |
| 空口令检查 | `net user` 逐个测试 | `awk -F: '($2==""){print $1}' /etc/shadow` |
| UID 0 检查 | 不适用 | `awk -F: '($3==0){print $1}' /etc/passwd` |
| umask | 不适用 | `umask 022` |

## 九、逐项参考文档索引

以下为 [AGCAVC1(198.120.0.206)_附件1/](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/README.md>) 中的现场实施文档，序号对应原表 Excel 行号，**完整步骤见对应文档**：

| 分组 | 作业项 | 参考文档 |
| --- | --- | --- |
| 1 病毒查杀 | B1 | [序号11.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号11.md>) |
| 2 设备管理 | B2 | [序号16.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号16.md>) |
| | B3 | [序号17.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号17.md>) |
| 3 用户账号与口令安全 | B4 | [序号18.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号18.md>) |
| | B5 | [序号19.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号19.md>) |
| | B6 | [序号20.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号20.md>) |
| | B7 | [序号23.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号23.md>) |
| | B8 | [序号24.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号24.md>) |
| | B9 | [序号25.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号25.md>) |
| | B10 | [序号26.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号26.md>) |
| | B11 | [序号27.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号27.md>) |
| 4 日志与审计 | B12 | [序号28.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号28.md>) |
| | B13 | [序号29.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号29.md>) |
| | B14 | [序号30.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号30.md>) |
| | B15 | [序号31.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号31.md>) |
| | B16 | [序号32.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号32.md>) |
| 5 服务优化 | B17 | [序号33.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号33.md>) |
| | B18 | [序号34.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号34.md>) |
| | B19 | [序号35.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号35.md>) |
| | B20 | [序号36.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号36.md>) |
| 6 安全防护 | B21 | [序号37.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号37.md>) |
| | B22 | [序号38.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号38.md>) |
| | B23 | [序号39.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号39.md>) |
| | B24 | [序号40.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号40.md>) |
| | B25 | [序号41.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号41.md>) |
| | **B26** | ⚠️ [序号42.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号42.md>) 是"**调度发令系统主机的特有加固**"，**与本表 B26 不是同一项**，仅可参考排查思路 |
| 7 漏洞（调度 20190615-001） | — | [序号45.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号45.md>) |
| 漏洞（调度 20190617-001） | — | [序号46.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号46.md>) |
| 漏洞（调度 20190618-002） | — | [序号47.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号47.md>) |
| 漏洞（调度 20190619-001） | — | [序号48.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号48.md>) |
| 漏洞（调度 20190622-001） | — | [序号49.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号49.md>) |
| 漏洞（调度 20190626-001） | — | [序号50.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号50.md>) |
| 漏洞（调度 20190626-002） | — | [序号51.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号51.md>) |
| 漏洞（信息通报 36 期） | — | [序号52.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号52.md>) |
| 漏洞（信息通报 38 期） | — | [序号53.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号53.md>) |
| 漏洞（信息通报 39 期） | — | [序号54.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号54.md>) |
| 漏洞（信息通报 41 期） | — | [序号55.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号55.md>) |
| 漏洞（信息通报 20-1 期） | — | [序号56.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号56.md>) |
| 漏洞（信息通报 20-10 期） | — | [序号57.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号57.md>) |
| 漏洞（信息通报 20-9 期） | — | [序号58.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号58.md>) |
| 8 必装软件 | 主机加固软件 | [序号60.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号60.md>) |
| 必装软件 | 防病毒软件 | [序号61.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号61.md>) |
| 必装软件 | 接入堡垒机 | [序号62.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号62.md>) |
| 必装软件 | SNMP、SYSLOG 接入安管平台 | [序号63.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/序号63.md>) |

> **注意**：上述文档针对 **220kV 双文光伏电站 AGCAVC1（198.120.0.206）**，其主机名、IP、服务状态与本项目不同，**参数与命令可参考，具体取值须按本项目实际核对**。特别是 `AGCAVC1` 与本表 B26 的**作业项定义不同**（见上表）。

## 相关文档

- [README.md](README.md)：表单通用规则、依据标准、跨 sheet 通用参数
- [01-Windows主机评估.md](01-Windows主机评估.md)：Windows 主机加固，与本表参数对照填写
- [../07-二次安防系统.md](../07-二次安防系统.md)：主机加固、防病毒的清单要求
- [../08-设备选型与到货签收.md](../08-设备选型与到货签收.md)：设备到货型号与数量
- [../09-文件间参数差异对照.md](../09-文件间参数差异对照.md)：SNMP 团体名等跨 sheet 矛盾说明
- [../../../系统加固/服务器加固.md](../../../系统加固/服务器加固.md)：主机加固相关资料
- [../../../系统加固/加固信息索引.md](../../../系统加固/加固信息索引.md)：加固信息索引
- [AGCAVC1(198.120.0.206)_附件1/README.md](<../../../系统加固/220kV双文光伏电站/加固报告/AGCAVC1(198.120.0.206)_附件1/README.md>)：B1–B26 现场实施文档索引
