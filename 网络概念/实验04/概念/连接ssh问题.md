# 连接不上

## 注册问题

- 现象

```
新的错误

PS C:\Users\Rolkey.DESKTOP-TGICN02\Desktop> ssh sshuser@hwLsw
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
SHA256:xbykpMN5vsCARzI4DLz0BFWCtY6IyK2dM42cLIykuYA.
Please contact your system administrator.
Add correct host key in C:\\Users\\Rolkey.DESKTOP-TGICN02/.ssh/known_hosts to get rid of this message.
Offending RSA key in C:\\Users\\Rolkey.DESKTOP-TGICN02/.ssh/known_hosts:35
Host key for 192.168.254.1 has changed and you have requested strict checking.
Host key verification failed.
```

- 解决

```bash
ssh-keygen -R 192.168.254.1
```

## host配置文件

```
# 交换机配置文件
Host hwLsw
    HostName 192.168.254.1
    User sshuser
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers 3des-cbc,aes128-cbc,aes192-cbc,aes256-cbc,aes128-ctr,aes192-ctr,aes256-ctr>
    MACs hmac-sha1,hmac-sha1-96,hmac-md5,hmac-md5-96

# 防火墙配置文件
Host hwFw
    HostName 192.168.0.1
    User sshuser
    KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers 3des-cbc,aes128-cbc,aes192-cbc,aes256-cbc,aes128-ctr,aes192-ctr
    MACs hmac-sha1,hmac-sha1-96,hmac-md5,hmac-md5-96
```

## 注册问题2

- 现象

```bash
> ssh hwLsw
ssh_dispatch_run_fatal: Connection to 192.168.254.1 port 22: Invalid key length
```

- 解决，按提示覆盖一下即可，长度要大于1024

```bash
rsa local-key-pair create
```
