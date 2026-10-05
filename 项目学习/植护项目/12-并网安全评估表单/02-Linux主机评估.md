# Linux 主机安全评估

> 对应原表 sheet：`二、安全评估(linux-x.x.x.x)`（70 行 × 20 列）
> 作业项 B1–B26，共 26 项；填表数量 = 本项目 Linux 主机台数（待确认，见"待填项"）

Linux 主机执行病毒查杀与安全加固。数据库与中间件**禁止 root 权限部署**。

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

## 二、作业项

### 1 病毒查杀

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B1 | 使用主机杀毒或杀毒 U 盘软件完成病毒查杀与日志导出 | 下载地址：Ⅲ 区 `https://3.1.1.1`，Ⅰ/Ⅱ 区 `https://1.1.1.1`；完成病毒查杀（最近一次扫描不超过六个月），开启软件携带的 U 盘管控功能；查看 `cat /opt/LinuxKPC/ini/apsc_client_ui.ini` 与 `date -d @时间戳 "+%Y-%m-%d"` 核对最近扫描时间；记录病毒库更新时间 `cat /opt/LinuxKPC/ini/version`；记录发现与清除病毒数量 | 标准 1 |

### 2 设备管理

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B2 | 管理远程工具 | 安装 SSH，用 SSH 代替 telnet、FTP，对传输数据加密。配置值见下表 | 标准 1 |
| B3 | 访问控制 | 默认远程 22 端口改为 **22222**；配置本机访问控制列表，限制远程 SSH 管理 IP，**仅允许堡垒机、态势感知等访问 SSH** | 标准 1 |

B2 的 sshd_config 配置值：

| 配置项 | 值 | 说明 |
| --- | --- | --- |
| `Protocol` | 2 | 使用 ssh2 版本 |
| `X11Forwarding` | yes | 允许窗口图形传输使用 ssh 加密 |
| `IgnoreRhosts` | yes | 完全禁止 SSHD 使用 .rhosts 文件 |
| `RhostsAuthentication` | no | 不设置使用基于 rhosts 的安全验证 |
| `RhostsRSAAuthentication` | no | 不设置使用 RSA 算法的基于 rhosts 的安全验证 |
| `PermitRootLogin` | no | 禁止 root 远程登录 |
| `PermitEmptyPasswords` | no | 禁止空密码 |
| `Banner` | /etc/issue | 指定登录提示文件 |

原表给出的实施步骤：先 `cp -p sshd_config sshd_config.tmp` 备份，再用 `awk` 生成修改了安全配置的临时文件，最后替换原始配置文件。查看 SSH 服务状态：`ps -elf | grep ssh`。

B3 的实施步骤：

```
vi /etc/hosts.allow
ssh:X.X.X.X:allow      # 逐条放行允许访问的 IP
ssh:X.X.X.X:allow

vi /etc/hosts.deny
all:all

pkill -HUP inetd      # 重启进程
```

原表补充说明：更改 `/etc/inetd.conf` 文件后，重启 inetd 的命令是 `pkill -HUP inetd` 或 `/etc/init.d/inetsvc start`。

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
| B11 | root 用户环境变量的安全性 | `echo $PATH | egrep '(^|:)(\.|:|$)'` 检查是否包含父目录；`find \`echo $PATH | tr ':' ' '\` -type d \( -perm -002 -o -perm -020 \) -ls` 检查是否包含组目录权限为 777 的目录 | 标准 1 |

B4 需锁定的默认账号：`daemon bin sys adm uucp lp snapp nobody invscout nuucp lpd imnadm ipsec ldap`

B6 的 login.defs 参数值：

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `PASS_MAX_DAYS` | 180 | 密码最长使用天数（可选） |
| `PASS_MIN_DAYS` | 1 | 密码最短使用天数 |
| `PASS_WARN_AGE` | 28 | 密码到期提前提醒天数 |
| `PASS_MIN_LEN` | 8 | 密码最小长度 |

B6 的 3 级系统登录失败锁定配置（`/etc/pam.d/system-auth`）：

```
auth required pam_env.so
auth required pam_tally.so onerr=fail deny=5 unlock_time=1800
auth sufficient pam_unix.so nullok try_first_pass
auth requisite pam_succeed_if.so uid >= 500 quiet
auth required pam_deny.so
```

B6 还要求询问管理员是否存在类似 `root/root`、`test/test`、`root/root12345678` 的简单用户密码配置。

B9 禁止 FTP 登录的账户：`root daemon bin sys adm lp uucp nuucp listen nobody hpdb useradm`

### 4 日志与审计

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B12 | syslog 登录事件记录 | 捕获 authpriv 消息，记录安全方面日志消息（如网络设备启动、usermod、change 等）。`more /etc/syslog.conf` 查看 authpriv 值 | 标准 1 |
| B13 | 指定日志服务器 | 配置专门的日志服务器 `loghost`（例如态势感知），4 条规则见下 | 标准 1 |
| B14 | 日志系统配置文件保护 | `chmod 400 /etc/syslog.conf`，使管理员只读 | 标准 1 |
| B15 | 开机启动 syslog 服务 | syslog 服务能在启动服务器时开启。`/etc/syslog.conf` 配置 `*.err;kern.debug;daemon.notice; /var/adm/messages`；`ps -eaf | grep syslog` 查看；`startsrc -s syslogd` 启动，`stopsrc -s syslogd` 停止 | 标准 3 |
| B16 | 日志记录满足 6 个月 | 日志记录满足 6 个月 | 标准 3 |

B13 的 syslog 转发规则（原表在"作业结点"与"加固方法"两列重复列出同一组）：

```
kern.warning;*.err;authpriv.none	@loghost
*.info;mail.none;authpriv.none;cron.none	@loghost
*.emerg	@loghost
local7.*	@loghost
```

### 5 服务优化

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B17 | 禁用不必要的系统服务 | `cp /etc/inet/inetd.conf /etc/inet/inetd.conf.backup` 后编辑注释不需要的服务；禁用清单见下。`systemctl | grep active` 或 `chkconfig --list` 查看开放的服务 | 标准 1 |
| B18 | 防止 icmp 重定向报文的攻击 | 备份后执行 `sysctl -w net.ipv4.conf.all.accept_redirects="0"`，并将 `/proc/sys/net/ipv4/conf/all/accept_redirects` 的值改为 0 | 标准 4 |
| B19 | SNMP、SNMP TRAP 服务接受团体名称设置 | `/etc/snmp/snmpd.conf` 修改默认团体名，**团体名改为 `GXhmrjsw-q3`**；`get-community-name` 将默认 public 修改为复杂团体名（读团体名，即读的密码） | 标准 4 |
| B20 | 系统时间同步 | 系统时间与 NTP 服务器同步，`ntpdate 3.1.1.4` | 标准 4 |

B17 禁用服务清单：`telnet sendmail klogin rlogin kshell ntalk tftp imap pop3`

### 6 安全防护

| 编号 | 作业内容 | 标准要求与加固方法 | 依据 |
| --- | --- | --- | --- |
| B21 | Umask 权限 | 控制用户缺省访问权限 UMASK 为 **022**。`vi /etc/profile` 设置 | 标准 1 |
| B22 | 敏感文件安全保护 | `chown root /etc/passwd /etc/shadow /etc/group`；`chmod 644 /etc/passwd /etc/group`；`chmod 400 /etc/shadow` | 标准 1 |
| B23 | Root 操作历史 | 减少 root 输入命令的历史记录长度，保留最新执行的 **5 条**命令。`vi /etc/profile` 修改 `HISTSIZE=5` | 标准 1 |
| B24 | 查看端口进程并结束 | 加固后 `netstat -anp | grep 端口号`；添加防火墙出入站策略禁用高危端口；进程仍运行则 `kill -9 进程号` 强制结束，重启后再次检查。端口清单见下 | 标准 1 |
| B25 | 禁止多余 USB 接口的使用 | 除鼠标键盘等必要外设接入 USB 外，其余 USB 口用封条封死（原则上保留一个数据拷贝接口） | 标准 5 |
| B26 | 卸载 samba 服务、删除默认路由 | 卸载 samba；删除默认路由 0.0.0.0；配置指向 `10.67.x.0` 的路由，下一条是默认网关；禁止存在默认路由，必须全为明细路由，路由掩码应 **≥24 位**。路由记得保存，不然重启会丢失 | 标准 1 |

B24 端口清单（原文）：

| 类别 | 端口 |
| --- | --- |
| 必须禁用 | 135、137、138、139、445、3389、`5323333` |
| 原则上禁用 | 23（telnet）、25（SMTP）、20、21（ftp）、513（login）、25、110（e-mail）、1900（UPnP） |

原表在 B24 把 23333 误写为 `5323333`，见文末不一致说明。

B26 的 samba 卸载命令：

| 发行版 | 查看 | 卸载 |
| --- | --- | --- |
| CentOS | `rpm -qa | grep samba` | `rpm -e --nodeps samba` |
| Ubuntu / Debian | `dpkg -l | grep samba`、`dpkg -l | grep smbfs`、`dpkg -l | grep smb` | `apt-get remove samba` |

停止服务：`service smbd stop` 或 `/etc/rc.d/init.d/smbd stop` 或 `/etc/init.d/smbd stop`。

B26 的默认路由处理：

```
route -n                                  # 查看路由，确认默认路由对应网卡（原文示例 eth0）
vi /etc/sysconfig/network-scripts/ifcfg-eth0
# 在 GATEWAY=xxxx 行前加 # 注释该行，wq 保存
service network restart

# 配置指向 10.67.x.0 的明细路由
# 方法 1：/etc/rc.local
route add -net 10.67.x.0/24 dev eth0
route add -net 10.67.x.0/24 gw 10.67.1.1

# 方法 2：/etc/sysconfig/network 末尾
GATEWAY=gw-ip    # 或 GATEWAY=gw-dev

# 方法 3：/etc/sysconfig/static-routes
any net 10.67.x.0/24 gw 10.67.1.1
```

Ubuntu / Debian 的路由保存路径：`/etc/sysconfig` 下的 `static-routes`，文件不存在时可同时创建。

## 三、漏洞处置清单（14 条）

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

## 四、必装软件

| 项 | 要求 |
| --- | --- |
| 主机加固软件 | G01 或信达或优炫 |
| 防病毒软件 | 安天防病毒软件 |
| 堡垒机 | 是否接入堡垒机 |
| 安管平台 | SNMP、SYSLOG 接入安管平台 |

Linux 表单比 Windows 表单多一项"SNMP、SYSLOG 接入安管平台"，即 Linux 主机除 SNMP 外还须接入 syslog，与 B13 的 `loghost` 转发配置配套。

## 五、与 Windows 表单的参数差异

| 项 | Windows | Linux |
| --- | --- | --- |
| 默认路由掩码 | ≥16 位（255.255.0.0） | **≥24 位**，且须显式配置 `10.67.x.0` 网段路由 |
| 远程管理协议 | SSH、HTTPS | SSH，端口 22 改 **22222** |
| 远程管理访问控制 | 限制管理 IP | `/etc/hosts.allow` + `/etc/hosts.deny`（`all:all`） |
| root 远程登录 | 重命名 Administrator | `PermitRootLogin no` 或 `/etc/securetty` |
| 登录失败锁定 | 5 次 / 锁定 5 分钟 | 5 次 / 锁定 30 分钟（`unlock_time=1800`） |
| 会话超时 | 屏幕保护 + 强制注销 | `TMOUT=300` |
| 日志转发 | SNMP 接入态势感知 | SNMP + SYSLOG 接入安管平台 |
| USB | 封条封死 | 封条封死 |
| SNMP 团体名 | `GXhmrjsw-q3` | `GXhmrjsw-q3`（一致） |

## 六、本项目相关

| 项 | 说明 |
| --- | --- |
| 防病毒系统 Linux 授权 | 清单序号 11，1 套含 10 个 Linux 服务器防护授权 + 3 年升级许可 |
| 主机加固系统 | 清单序号 10，1 项 |
| 主机台账 | 待确认。防病毒授权 10 个 Linux 授权不等于 10 台主机，需向运维索取实际主机台账 |

## 七、原表不一致处

| 位置 | 不一致内容 | 处理建议 |
| --- | --- | --- |
| B24 | 端口列表写成 `5323333` | 正确为 `23333`，与其他 sheet 保持一致 |
| B3 / B26 | B3 说"路由掩码应 ≥24 位"写在 B26，B26 又重复"路由掩码应 ≥24 位" | 原文如此，按 B26 执行 |
| sheet 版本列 | 表头"版本"列的 A1 内容里写的是"Ⅲ 区下载地址 https://3.1.1.1 / Ⅰ/Ⅱ 区 https://1.1.1.1" | 内容属 B1 病毒查杀，非版本信息，属模板填写错位 |
| B13 | "作业结点"与"加固方法"两列内容完全重复 | 无实质影响 |
| 异常记录示例 | B1 填"无法安装杀毒软件"，B22 行空白 | 属示例填写，非本项目记录 |

## 相关文档

- [README.md](README.md)：表单通用规则、依据标准、跨 sheet 通用参数
- [01-Windows主机评估.md](01-Windows主机评估.md)：Windows 主机加固，与本表参数对照填写
- [../07-二次安防系统.md](../07-二次安防系统.md)：主机加固、防病毒的清单要求
- [../08-设备选型与到货签收.md](../08-设备选型与到货签收.md)：设备到货型号与数量
- [../../../系统加固/服务器加固.md](../../../系统加固/服务器加固.md)：主机加固相关资料
- [../../../系统加固/加固信息索引.md](../../../系统加固/加固信息索引.md)：加固信息索引