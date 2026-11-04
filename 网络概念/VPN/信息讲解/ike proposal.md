* 基础信息

```bash
[FW2]display ike proposal
2026-11-03 08:18:52.190 

Number of IKE Proposals: 2

-------------------------------------------
 IKE Proposal: 1
   Authentication Method      : PRE_SHARED
   Authentication Algorithm   : SHA2-256 
   Encryption Algorithm       : AES-256 
   Diffie-Hellman Group       : MODP-2048 
   SA Duration(Seconds)       : 86400
   Integrity Algorithm        : HMAC-SHA2-256 
   Prf Algorithm              : HMAC-SHA2-256 
-------------------------------------------

-------------------------------------------
 IKE Proposal: Default
   Authentication Method      : PRE_SHARED
   Authentication Algorithm   : SHA2-512 SHA2-384 SHA2-256 
   Encryption Algorithm       : AES-256 AES-192 AES-128 
   Diffie-Hellman Group       : MODP-2048 
   SA Duration(Seconds)       : 86400
   Integrity Algorithm        : HMAC-SHA2-256 
   Prf Algorithm              : HMAC-SHA2-256 
-------------------------------------------
```

---

这段是华为防火墙/路由器上查看 IKE 安全提议的输出。下面把关键单词按「字段 → 全称 → 中文 → 干嘛用」拆开翻译：

字段对照表

英文 全称 / 展开 中文 作用

display ike proposal — 显示 IKE 提议（安全提案） 查看已配置的 IKE 参数套件

IKE Internet Key Exchange 互联网密钥交换 IPSec 里用来协商密钥、建立安全通道的协议

Proposal — 提议 / 安全提案 一组「认证+加密+DH+寿命」参数的组合，两端必须匹配

Number of IKE Proposals — IKE 提议数量 当前共 2 套

Authentication Method — 认证方法 / 身份验证方式 双方怎么证明「我是我」

PRE_SHARED Pre-Shared Key (PSK) 预共享密钥 两边提前配同一个密码，最常见

Authentication Algorithm — 认证算法 此处 SHA2-256＝SHA-2 家族、256 位摘要

Encryption Algorithm — 加密算法 AES-256＝AES 高级加密标准，256 位密钥

Diffie-Hellman Group DH Group DH（迪菲-赫尔曼）组 密钥交换用的数学群，决定强度

MODP-2048 Modular Exponentiation P, 2048-bit 2048 位模幂素数群 即常见 group 14，位数越大越难暴力破

SA Security Association 安全联盟 / 安全关联 协商成功后建立的一套「加密规矩」

SA Duration(Seconds) — SA 有效期（秒） 86400＝24 小时后重新协商密钥

Integrity Algorithm — 完整性算法 防篡改校验

HMAC-SHA2-256 Hash-based MAC + SHA2-256 基于 SHA2-256 的哈希消息认证码 给每个包盖个「没被改过」的章

Prf Algorithm PRF = Pseudo-Random Function 伪随机函数算法 从 DH 结果里派生出实际加密/认证密钥

Default — 默认提议 系统自带、可兜底的一整套算法列表

FW2 Firewall 2 防火墙 2（设备名） 只是这台设备的主机名

算法后缀速记

• SHA2-512 / 384 / 256：SHA-2 哈希，数字＝输出位数，越大越抗碰撞

• AES-256 / 192 / 128：AES 对称加密，数字＝密钥长度

• HMAC-＝带密钥的哈希；PRF＝密钥派生；MODP-＝DH 群的模数位数

连起来读一遍

这台 FW2 上有 2 套 IKE 提议：自定义 1 号用「预共享密钥 + AES-256 加密 + SHA2-256 认证 + 2048 位 DH 组 + HMAC-SHA2-256 完整性/PRF，24 小时重协商」；Default 是系统兜底，认证/加密各允许多种强度按序协商，其余参数同 1 号。

一个容易混的点：Authentication Algorithm 和 Integrity Algorithm 在 IKEv2 里都填了 HMAC-SHA2-256，前者管「身份/握手认证」，后者管「报文完整性」，Prf 管「派生密钥」——三者各司其职，不是重复写错。