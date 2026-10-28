## 问题

```
[FW1]nat-policy 
[FW1-policy-nat]di this
2026-10-28 04:46:25.220 
#
nat-policy
 rule name snat
  source-zone trust
  destination-zone untrust
  action source-nat address-group 1

华为的这个配置中，source-nat这个表示 什么？难道有destination-nat这种对照的概念？
```

---

是的，你的推测完全正确。华为防火墙的 NAT 策略中确实存在 `destination-nat` 这个概念，它与 `source-nat` 是并列的两种地址转换动作。

### `source-nat` 的含义
在你提供的配置中，`action source-nat address-group 1` 的意思是：对于匹配该规则（从 trust 区域去往 untrust 区域）的流量，防火墙会对其**源 IP 地址**进行转换，将其替换为地址组 `1` 中定义的公网 IP 地址。这通常用于让内网私网用户能够访问互联网。

### 对应的 `destination-nat` 概念
华为防火墙确实支持 `destination-nat`。它的作用是转换报文的**目的 IP 地址**，通常用于将公网用户访问某个公网 IP 的流量，重定向到内网真实的服务器私网 IP 上（类似于服务器映射或端口映射的场景）。

### 两者在配置上的区别
在华为的 NAT 策略视图下，这两个动作的配置语法和用途有明显差异：

*   **源 NAT (`source-nat`)**
    *   **核心作用**：隐藏内部源 IP，解决私网访问公网的问题。
    *   **典型动作参数**：`easy-ip`（使用出接口地址）、`address-group`（使用地址池）、`static-mapping`（静态映射）。
    *   **在你配置中的体现**：`action source-nat address-group 1`。

*   **目的 NAT (`destination-nat`)**
    *   **核心作用**：发布内部服务，解决公网访问私网服务器的问题。
    *   **典型动作参数**：`address`（转换后的 IP）、`address-group`（地址池），并且通常可以附带 `port` 参数来指定端口转换。它也支持 `static` 关键字进行静态的一对一或端口映射。
    *   **配置示例**：`action destination-nat address-group 2` 或 `action destination-nat static address-to-address ...`。

---
**顺便一提**：虽然 `destination-nat` 存在，但在华为防火墙的日常配置中，针对“公网访问内网服务器”这一最常见的目的 NAT 场景，很多时候会直接使用更专门的 **NAT Server** 命令来配置，它的逻辑更直观，专门用于服务器发布。

---
---

## 举例说明destination-nat的使用方法

在华为防火墙中，`destination-nat` 主要出现在 **NAT 策略**里，用于将流量的目的 IP 地址进行转换。虽然它功能上能实现类似“服务器映射”的效果，但在实际配置中，针对“公网访问内网服务器”这个最经典的场景，华为通常推荐使用更简洁的 **NAT Server** 命令。

不过，既然你问的是 `destination-nat` 在策略里的用法，下面就用一个具体的例子来说明。

### 场景与目标
假设你有一台内网 Web 服务器（私网 IP `10.2.0.7`，端口 `80`），现在希望通过防火墙的公网 IP `1.1.10.10` 的 `8080` 端口对外提供服务。

使用 `destination-nat` 策略的实现思路是：在 NAT 策略中匹配“目的地址为 `1.1.10.10`、目的端口为 `8080`”的流量，并执行动作，将其目的地址转换为内网服务器的真实地址。

### 配置示例（CLI）
根据华为官方文档中的类似示例，配置逻辑大致如下：

**1. 配置目的 NAT 地址池**
首先，创建一个包含内网服务器地址的目的 NAT 地址池。
```text
[FW] destination-nat address-group dnat_pool 1
[FW-dnat-address-group-dnat_pool] section 0 10.2.0.7 10.2.0.7
[FW-dnat-address-group-dnat_pool] quit
```

**2. 配置 NAT 策略**
在 `nat-policy` 中创建规则，匹配公网访问流量，并调用上一步的地址池进行目的地址转换。
```text
[FW] nat-policy
[FW-policy-nat] rule name dnat_web
[FW-policy-nat-rule-dnat_web] source-zone untrust
[FW-policy-nat-rule-dnat_web] destination-address 1.1.10.10 32
[FW-policy-nat-rule-dnat_web] service protocol tcp destination-port 8080
[FW-policy-nat-rule-dnat_web] action destination-nat static address-to-port address-group dnat_pool 80 to 80
[FW-policy-nat-rule-dnat_web] quit
```
> **注意**：上面的 `action destination-nat static address-to-port ...` 是华为文档中一种将公网端口映射到私网地址及端口的写法。不同防火墙版本的命令语法（例如是否必须带 `static` 关键字）可能存在差异，请以设备实际命令帮助为准。

**3. 配套的必要配置**
*   **安全策略**：必须放行 `untrust` 到 `dmz`（或服务器所在区域）的流量，且安全策略里的**目的地址应写转换后的私网 IP**（即 `10.2.0.7`），而不是公网 IP。
*   **黑洞路由**：如果公网 IP `1.1.10.10` 与防火墙接口 IP 不在同一网段，通常需要配置一条指向 `NULL0` 的黑洞路由，防止路由环路。

### 为什么不直接用 `destination-nat` 发布服务器？
在华为防火墙上，发布内部服务器更常见、更推荐的做法是使用 **NAT Server**。对比来看：

*   **NAT Server 更直观**：命令 `nat server` 将“公网地址/端口”与“私网地址/端口”直接绑定，逻辑清晰，一步到位。
*   **`destination-nat` 策略更灵活**：它本质是策略路由式的匹配，适合需要基于源地址、服务类型等复杂条件进行目的地址转换的场景。对于简单的服务器发布，配置 NAT 策略显得有些繁琐。

所以，如果你只是想发布一台服务器，直接用 `nat server protocol tcp global 1.1.10.10 8080 inside 10.2.0.7 80` 这类命令会更简单。`destination-nat` 策略的存在，是为了在更复杂的 NAT 策略编排中提供目的地址转换的能力。

---
---

## 举例nat server

`nat server` 是华为防火墙上发布内部服务器最直接、最常用的命令。它本质上是目的 NAT 的一种固定映射形态，将“公网 IP:端口”与“私网服务器 IP:端口”直接绑定。

### 场景与目标
沿用之前的设定：内网 Web 服务器私网 IP 为 `10.2.0.7`，端口 `80`。现在要通过防火墙的公网 IP `1.1.1.10` 的 `8080` 端口对外发布。公网接口 IP 假设为 `1.1.1.1`，内网服务器网关指向防火墙内网接口 `10.2.0.1`。

### 核心配置命令
在系统视图下，一条命令即可完成映射：
```text
[FW] nat server policy_web protocol tcp global 1.1.1.10 8080 inside 10.2.0.7 80
```
*   **`policy_web`**：这条映射规则的名称，可自定义。
*   **`protocol tcp`**：指定协议类型，Web 服务用 TCP。
*   **`global 1.1.1.10 8080`**：外部用户实际访问的公网地址和端口。
*   **`inside 10.2.0.7 80`**：内网服务器真实的私网地址和实际监听端口。

### 配套的必要步骤
光配 `nat server` 还不够，必须配合以下配置流量才能通：

**1. 安全策略（最容易漏的一步）**
必须放行从 Untrust 到内网服务器所在区域（如 DMZ 或 Trust）的流量。
**关键点**：安全策略里的**目的地址要写转换后的私网 IP**（即 `10.2.0.7`），而不是公网 IP `1.1.1.10`。
```text
[FW] security-policy
[FW-policy-security] rule name allow_web
[FW-policy-security-rule-allow_web] source-zone untrust
[FW-policy-security-rule-allow_web] destination-zone dmz
[FW-policy-security-rule-allow_web] destination-address 10.2.0.7 32
[FW-policy-security-rule-allow_web] service http
[FW-policy-security-rule-allow_web] action permit
```

**2. 黑洞路由（视公网 IP 情况而定）**
如果公网 IP `1.1.1.10` **不是**防火墙接口本身的 IP，且与接口 IP 不在同一网段，通常需要配置一条指向 `NULL0` 的黑洞路由，防止路由环路。
```text
[FW] ip route-static 1.1.1.10 32 NULL0
```

### 与 destination-nat 的关键区别
你之前问的 `destination-nat` 策略和 `nat server` 虽然都能做目的地址转换，但在华为设备上有明确差异：
*   **优先级**：`nat server` 的优先级**高于** `destination-nat`。防火墙收到首包时，会先查 `nat server` 生成的 Server-Map 表，匹配不上才去走 `destination-nat` 策略。
*   **方向性**：`nat server` 默认是双向的（回程流量会自动转换回来），除非加了 `no-reverse` 参数；而 `destination-nat` 仅做单向转换。

所以，对于“发布服务器”这个简单直接的需求，用 `nat server` 一条命令远比写 `nat-policy` 加 `destination-nat` 动作要简洁得多。

---
**补充**：如果公网 IP 比较紧张，想直接用防火墙出接口的公网 IP 来发布，可以把 `global` 后面的地址换成 `current-interface`，例如：`nat server protocol tcp global current-interface 8080 inside 10.2.0.7 80`。