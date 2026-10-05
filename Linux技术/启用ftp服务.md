## 环境
- 服务端：Debian 12 (bookworm)，IP 10.8.5.151，主机名 rolkeyDebian01
- 客户端：Ubuntu，用于测试
- vsftpd 版本：3.0.3

## 一、安装 vsftpd
```bash
apt update
apt install vsftpd -y
systemctl enable vsftpd
systemctl start vsftpd
```

## 二、创建 FTP 专用用户
```bash
# 创建用户，家目录 /srv/ftp/user1，shell 为 nologin（禁止 SSH 登录，仅用于 FTP）
useradd -m -d /srv/ftp/user1 -s /usr/sbin/nologin user1
passwd user1

# 设置目录权限
chown -R user1:user1 /srv/ftp/user1
chmod 750 /srv/ftp/user1
```

## 三、修改 /etc/vsftpd.conf
在文件末尾追加以下内容（或取消对应行的注释）：

```ini
# 允许本地用户登录
local_enable=YES

# 允许上传/写入
write_enable=YES

# 将用户限制在家目录
chroot_local_user=YES

# 允许被 chroot 的目录可写（必须，否则报 500 OOPS）
allow_writeable_chroot=YES

# 默认 umask
local_umask=022

# PAM 服务名
pam_service_name=vsftpd

# 详细日志（可选，排错时用）
#log_ftp_protocol=YES
```

## 四、关键：修改 /etc/pam.d/vsftpd
**问题根源**：默认配置末尾有 `pam_shells.so`，它会拒绝 shell 为 nologin 的用户，
即使 nologin 已加入 /etc/shells 也无效。

编辑 `/etc/pam.d/vsftpd`，将最后一行注释掉：

```
#auth    required        pam_shells.so
```

修改后文件内容应为：

```
# Standard behaviour for ftpd(8).
auth    required        pam_listfile.so item=user sense=deny file=/etc/ftpusers onerr=succeed

# Note: vsftpd handles anonymous logins on its own. Do not enable pam_ftp.so.

# Standard pam includes
@include common-account
@include common-session
@include common-auth
#auth   required        pam_shells.so
```

## 五、重启服务（必须完全重启）
```bash
systemctl stop vsftpd
systemctl start vsftpd
systemctl status vsftpd
```

**注意**：必须用 stop + start，不要只用 restart。PAM 会话需要完全重新初始化。

## 六、检查用户不在拒绝列表
```bash
grep user1 /etc/ftpusers
# 应无输出
```

## 七、客户端测试
```bash
ftp 10.8.5.151
# Name: user1
# Password: <密码>
# 应返回 230 Login successful
```

## 八、日志查看
Debian 12 使用 systemd-journald，auth.log 可能不存在。

```bash
# vsftpd 日志
tail -f /var/log/vsftpd.log

# systemd 日志
journalctl -u vsftpd -f
```

## 九、防火墙（如启用）
```bash
# 主动模式
ufw allow 21/tcp

# 被动模式（如启用）
# 需在 vsftpd.conf 中加：
# pasv_enable=YES
# pasv_min_port=40000
# pasv_max_port=50000
ufw allow 40000:50000/tcp
```

## 十、常见错误排查
| 现象 | 原因 | 解决 |
|---|---|---|
| 530 Login incorrect | pam_shells.so 拒绝 nologin | 注释 pam_shells.so 行 |
| 500 OOPS: refusing to run with writable root | 家目录可写但未允许 | 加 allow_writeable_chroot=YES |
| 登录后无法上传 | write_enable 未开 | 设 write_enable=YES |
| 卡在"读取目录列表" | 被动模式端口未放行 | 配置 pasv 端口并放行防火墙 |
```

## 三、几个值得记住的要点

1. **`pam_shells.so` 对 nologin 有硬编码拒绝逻辑**，不看你有没有把 nologin 加进 `/etc/shells`。
2. **改完 PAM 后要 stop + start**，`restart` 不一定能让 PAM 会话重新初始化。
3. **Debian 12 没有 `/var/log/auth.log`**，用 `journalctl` 或 `/var/log/vsftpd.log`。
4. **`allow_writeable_chroot=YES` 是必须的**，否则 chroot 到家目录且可写时会报 500 OOPS。
