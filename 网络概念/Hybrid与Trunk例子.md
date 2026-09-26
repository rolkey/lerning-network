下面把前面讨论的 **`port default vlan` 与 `port trunk pvid` 的差别**，以及 **`allow-pass` 与 `pvid` 的概念**，补充进修订版文档中。补充部分用 **“八、补充说明”** 单独成章，便于查阅，也不破坏原有结构。

---

# 同一场景下 Trunk 与 Hybrid 配置完整对比

## 一、场景描述

### 拓扑

```
        PC1(VLAN 10)                          PC2(VLAN 20)
             |                                     |
             |                                     |
        [G0/0/1]                             [G0/0/2]
        ┌──────────────────────────────────────────┐
        │            Switch S1                     │
        │                                          │
        │            [G0/0/24]                     │
        └──────────────────────────────────────────┘
                        |
                        |
                    [G0/0/24]
        ┌──────────────────────────────────────────┐
        │            Switch S2                     │
        │                                          │
        │            [G0/0/1]                      │
        └──────────────────────────────────────────┘
                        |
                        |
                  Server（需要同时处理 VLAN 10 和 VLAN 20）
```

### 需求

1. PC1 属于 **VLAN 10**，PC2 属于 **VLAN 20**。
2. S1 和 S2 之间的级联链路（G0/0/24）需要同时承载 **VLAN 10 和 VLAN 20**。
3. S2 的 G0/0/1 连接一台 **服务器**，服务器需要同时处理 VLAN 10 和 VLAN 20 的流量：
   - VLAN 10：服务器网卡带标签识别（tagged）
   - VLAN 20：服务器网卡不带标签识别（untagged），PVID 设为 VLAN 20

> 这个场景中，**级联链路**两种方式配置完全一样（都用 tagged）；  
> **区别在 S2 连接服务器的接口**。

---

## 二、方式一：使用 Trunk 配置

### 1. S1 配置

```
system-view
sysname S1

# 创建 VLAN
vlan batch 10 20

# 连接 PC1 的接口 —— Access
interface GigabitEthernet0/0/1
 port link-type access
 port default vlan 10
 quit

# 连接 PC2 的接口 —— Access
interface GigabitEthernet0/0/2
 port link-type access
 port default vlan 20
 quit

# 级联接口 —— Trunk
interface GigabitEthernet0/0/24
 port link-type trunk
 port trunk allow-pass vlan 10 20
 quit
```

### 2. S2 配置

```
system-view
sysname S2

vlan batch 10 20

# 级联接口 —— Trunk
interface GigabitEthernet0/0/24
 port link-type trunk
 port trunk allow-pass vlan 10 20
 quit

# 连接服务器的接口 —— Trunk
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20
 port trunk pvid vlan 20
 quit
```

### 3. 效果分析

| 流量方向 | 行为 |
|----------|------|
| 服务器发 VLAN 10 帧（带标签） | 进入 VLAN 10，正常转发 |
| 服务器发 VLAN 20 帧（不带标签） | 打上 PVID 20，进入 VLAN 20 |
| 交换机发 VLAN 10 给服务器 | VLAN 10 ≠ PVID 20 → **带标签**发送 |
| 交换机发 VLAN 20 给服务器 | VLAN 20 = PVID 20 → **去标签**发送 |

### 4. 重要说明

这里 **Trunk 语法本身没有错误**，配置也是合法的，而且在这个具体需求下 **功能上确实能实现**。

但必须注意：

- Trunk 的标签规则是**固定的**：
  - 只有 PVID 对应的 VLAN 去标签；
  - 其他允许通过的 VLAN 一律带标签。
- 因此，Trunk 实现“VLAN 10 tagged、VLAN 20 untagged”是**靠 PVID 规则被动匹配出来的**，不是显式控制。
- 如果需求变成：
  - VLAN 10 也去标签；
  - 或者 VLAN 10 去标签、VLAN 20 带标签；
  - 或者两个 VLAN 都去标签；

  **Trunk 就无法直接做到**。

所以准确说法是：

> Trunk 在这个场景下“刚好能实现”，但灵活性不如 Hybrid。

---

## 三、方式二：使用 Hybrid 配置

### 1. S1 配置

```
system-view
sysname S1

vlan batch 10 20

# 连接 PC1 —— Hybrid（相当于 Access）
interface GigabitEthernet0/0/1
 port link-type hybrid
 port hybrid pvid vlan 10
 port hybrid untagged vlan 10
 quit

# 连接 PC2 —— Hybrid（相当于 Access）
interface GigabitEthernet0/0/2
 port link-type hybrid
 port hybrid pvid vlan 20
 port hybrid untagged vlan 20
 quit

# 级联接口 —— Hybrid（相当于 Trunk）
interface GigabitEthernet0/0/24
 port link-type hybrid
 port hybrid tagged vlan 10 20
 quit
```

### 2. S2 配置

```
system-view
sysname S2

vlan batch 10 20

# 级联接口 —— Hybrid
interface GigabitEthernet0/0/24
 port link-type hybrid
 port hybrid tagged vlan 10 20
 quit

# 连接服务器 —— Hybrid
interface GigabitEthernet0/0/1
 port link-type hybrid
 port hybrid pvid vlan 20
 port hybrid tagged vlan 10
 port hybrid untagged vlan 20
 quit
```

### 3. 效果分析

| 流量方向 | 行为 |
|----------|------|
| 服务器发 VLAN 10 帧（带标签） | 进入 VLAN 10，正常转发 |
| 服务器发 VLAN 20 帧（不带标签） | 打上 PVID 20，进入 VLAN 20 |
| 交换机发 VLAN 10 给服务器 | 配置为 tagged → **带标签**发送 |
| 交换机发 VLAN 20 给服务器 | 配置为 untagged → **去标签**发送 |

Hybrid 的特点是：**每个 VLAN 的标签行为都可以显式指定**，因此更灵活。

---

## 四、关键对比

### 同样连接服务器接口

**Trunk 配置：**

```
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20
 port trunk pvid vlan 20
```

- VLAN 10 → 带标签（因为 ≠ PVID）
- VLAN 20 → 去标签（因为 = PVID）
- **规则固定，无法单独调整某个 VLAN 的标签行为。**

**Hybrid 配置：**

```
interface GigabitEthernet0/0/1
 port link-type hybrid
 port hybrid pvid vlan 20
 port hybrid tagged vlan 10
 port hybrid untagged vlan 20
```

- VLAN 10 → 带标签（显式配置 tagged）
- VLAN 20 → 去标签（显式配置 untagged）
- **可以自由调整。**

### 如果需求变成：VLAN 10 和 VLAN 20 都去标签

- **Trunk**：做不到（只能一个 PVID 去标签）。
- **Hybrid**：

  ```
  port hybrid untagged vlan 10 20
  ```

  轻松实现。

### 如果需求变成：VLAN 10 去标签、VLAN 20 带标签

- **Trunk**：做不到。
- **Hybrid**：

  ```
  port hybrid pvid vlan 10
  port hybrid untagged vlan 10
  port hybrid tagged vlan 20
  ```

  轻松实现。

---

## 五、完整配置对照表

| 接口 | Trunk 方式 | Hybrid 方式 |
|------|-----------|-------------|
| S1 G0/0/1（PC1） | `port link-type access`<br>`port default vlan 10` | `port link-type hybrid`<br>`port hybrid pvid vlan 10`<br>`port hybrid untagged vlan 10` |
| S1 G0/0/2（PC2） | `port link-type access`<br>`port default vlan 20` | `port link-type hybrid`<br>`port hybrid pvid vlan 20`<br>`port hybrid untagged vlan 20` |
| S1 G0/0/24（级联） | `port link-type trunk`<br>`port trunk allow-pass vlan 10 20` | `port link-type hybrid`<br>`port hybrid tagged vlan 10 20` |
| S2 G0/0/24（级联） | `port link-type trunk`<br>`port trunk allow-pass vlan 10 20` | `port link-type hybrid`<br>`port hybrid tagged vlan 10 20` |
| S2 G0/0/1（服务器） | `port link-type trunk`<br>`port trunk allow-pass vlan 10 20`<br>`port trunk pvid vlan 20` | `port link-type hybrid`<br>`port hybrid pvid vlan 20`<br>`port hybrid tagged vlan 10`<br>`port hybrid untagged vlan 20` |

---

## 六、验证命令

配置完成后，可以用以下命令验证：

```
# 查看接口类型和 VLAN 配置
display port vlan

# 查看 VLAN 成员
display vlan

# 查看接口简要信息
display interface brief

# 测试连通性
ping -a 10.1.1.1 10.1.1.2
```

---

## 七、总结

1. **级联链路**：Trunk 和 Hybrid 配置效果完全一样（都用 tagged）。
2. **连接终端/服务器**：Hybrid 更灵活，可以自由控制每个 VLAN 的标签行为。
3. **Trunk 是 Hybrid 的特例**：  
   Trunk ≈ PVID untagged + 其他允许 VLAN tagged。
4. **本场景中 Trunk 能实现需求，但不是因为它灵活，而是因为需求刚好匹配 Trunk 的固定规则。**
5. **实际工作中**：
   - 需求简单、标准交换机级联：用 Trunk；
   - 需要精细控制标签（IP 电话、服务器多 VLAN、单臂路由等）：用 Hybrid。

---

## 八、补充说明：几个最容易混的命令

### 1. `port trunk allow-pass vlan 10 20` 是什么

```
port trunk allow-pass vlan 10 20
```

意思是：**这个 Trunk 口允许 VLAN 10 和 VLAN 20 的流量通过**。

它管的是“**哪些 VLAN 能过**”，是一个**过滤/放行**的概念，不是“打标签”。

| 方向 | 行为 |
|------|------|
| 收到带标签帧 | 如果 VLAN 在 allow-pass 列表里，就放行；否则丢弃 |
| 发送帧 | 如果 VLAN 在 allow-pass 列表里，就按 Trunk 规则发；否则不发 |

它**不管标签去不去**。标签行为由 PVID 和 Trunk 规则决定。

> 注意：在华为/华三设备上，通常建议把 PVID 对应的 VLAN 也加入 allow-pass，否则可能出现该 VLAN 流量无法正常转发的问题。

---

### 2. `port trunk pvid vlan 20` 是什么

```
port trunk pvid vlan 20
```

意思是：**这个 Trunk 口的 PVID（Port VLAN ID）设为 VLAN 20**。

PVID 是**端口默认 VLAN / native VLAN**，作用有两个：

**接收方向：**

- 收到**不带标签**的帧 → 打上 PVID，归入 PVID 对应的 VLAN。
- 收到**带标签**的帧 → 保留原标签，按标签 VLAN 处理，PVID 不影响。

**发送方向：**

- 发出的帧，如果 VLAN = PVID → **去标签**发送。
- 发出的帧，如果 VLAN ≠ PVID → **带标签**发送。

Trunk 口默认 PVID = VLAN 1。  
一个 Trunk 口**只能有一个 PVID**，所以 Trunk 口只能让**一个 VLAN** 去标签，其他都带标签。  
这就是 Trunk 不灵活的根本原因。

---

### 3. 两句合起来看

配置：

```
port link-type trunk
port trunk allow-pass vlan 10 20
port trunk pvid vlan 20
```

可以这样理解：

| 配置项 | 概念 | 管什么 |
|--------|------|--------|
| `allow-pass vlan 10 20` | 允许列表 | VLAN 10、20 能通过这个口 |
| `pvid vlan 20` | 默认 VLAN | 不带标签进来算 VLAN 20；VLAN 20 出去去标签 |

实际转发效果：

**接收方向：**

| 服务器发来的帧 | 处理 |
|----------------|------|
| 带 VLAN 10 标签 | 放行，进入 VLAN 10 |
| 不带标签 | 打上 PVID 20，进入 VLAN 20 |

**发送方向：**

| 交换机要发的帧 | 处理 |
|----------------|------|
| VLAN 10 | VLAN 10 ≠ PVID 20 → 带标签发送 |
| VLAN 20 | VLAN 20 = PVID 20 → 去标签发送 |

---

### 4. `port default vlan 20` 与 `port trunk pvid vlan 20` 的差别

这两个命令看起来都跟“VLAN 20”有关，但**完全不是一回事**。

| 命令 | 用在什么口 | 含义 | 该口能过几个 VLAN |
|------|-----------|------|------------------|
| `port default vlan 20` | **Access 口** | 这个口**只属于 VLAN 20** | 只能过 VLAN 20 |
| `port trunk pvid vlan 20` | **Trunk 口** | 这个口的**默认 VLAN 是 20** | 可以过多个 VLAN，VLAN 20 只是其中一个 |

一句话：

> `port default vlan 20` 是 Access 口的“唯一 VLAN”；  
> `port trunk pvid vlan 20` 是 Trunk 口的“默认 VLAN / native VLAN”。

#### （1）`port default vlan 20` 详解

只能用在 **Access 口**：

```
interface GigabitEthernet0/0/1
 port link-type access
 port default vlan 20
```

含义：这个口**只属于 VLAN 20**。

- 收到不带标签的帧 → 归入 VLAN 20
- 发出 VLAN 20 的帧 → 去标签发送
- 其他 VLAN 的帧 → 根本不让过

等价于 Hybrid 口：

```
port link-type hybrid
port hybrid pvid vlan 20
port hybrid untagged vlan 20
```

#### （2）`port trunk pvid vlan 20` 详解

只能用在 **Trunk 口**：

```
interface GigabitEthernet0/0/24
 port link-type trunk
 port trunk allow-pass vlan 10 20
 port trunk pvid vlan 20
```

含义：这个口的 **PVID = 20**，但它**不是只属于 VLAN 20**，还能过别的 VLAN（由 `allow-pass` 决定）。

- 收到不带标签 → 打上 PVID 20，归入 VLAN 20
- 发出 VLAN 20 → 去标签
- 发出 VLAN 10 → 带标签

#### （3）核心差别对比

| 对比项 | `port default vlan 20` | `port trunk pvid vlan 20` |
|--------|------------------------|---------------------------|
| 接口类型 | Access | Trunk |
| 这个口属于几个 VLAN | 只属于 VLAN 20 | 可以属于多个 VLAN |
| VLAN 20 的角色 | 唯一 VLAN | 默认 VLAN / native VLAN |
| 能否同时过 VLAN 10 | 不能 | 能（需 allow-pass） |
| 收到不带标签帧 | 归入 VLAN 20 | 归入 VLAN 20 |
| 发出 VLAN 20 帧 | 去标签 | 去标签 |
| 发出 VLAN 10 帧 | 不允许 | 带标签发送 |
| 本质 | 端口归属 | 端口默认值 |

#### （4）最容易混的一点

两者在“**收到不带标签帧**”和“**发出 VLAN 20 帧**”这两个行为上，**看起来是一样的**：

- 收到不带标签 → 都归入 VLAN 20
- 发出 VLAN 20 → 都去标签

但差别在于：

> **Access 口只服务 VLAN 20；Trunk 口还能同时服务 VLAN 10、VLAN 30……**

`port default vlan 20` 是“这个口就是 VLAN 20 的”；  
`port trunk pvid vlan 20` 是“这个口默认按 VLAN 20 处理不带标签的帧，但它还能过别的 VLAN”。

---

### 5. 一句话记忆

```
port default vlan 20
```

→ **Access 口：这个口只属于 VLAN 20。**

```
port trunk pvid vlan 20
```

→ **Trunk 口：这个口的默认 VLAN 是 20，但它还能过其他 VLAN。**

```
port trunk allow-pass vlan 10 20
```

→ **Trunk 口：允许 VLAN 10、20 通过这个口。**

```
port trunk pvid vlan 20
```

→ **Trunk 口：默认 VLAN 是 20；不带标签进来算 VLAN 20，VLAN 20 出去去标签。**

---

## 九、三种接口类型速查表

| 对比项 | Access | Trunk | Hybrid |
|--------|--------|-------|--------|
| 能过的 VLAN | 只 1 个 | 多个（allow-pass） | 多个（tagged/untagged） |
| 收不带标签帧 | 归入 default VLAN | 归入 PVID | 归入 PVID |
| 发 VLAN 帧 | 去标签 | 只有 PVID 去标签，其他带标签 | 按 tagged/untagged 显式配置 |
| 能否多 VLAN 去标签 | 只能 1 个 | 不能（只能 PVID） | 能 |
| 典型用途 | 连 PC/打印机 | 交换机级联 | 服务器、IP 电话、单臂路由 |

---

这样整份文档就同时包含了：

- 原场景的 Trunk / Hybrid 对比；
- `allow-pass` 与 `pvid` 的概念；
- `port default vlan` 与 `port trunk pvid` 的差别；
- 三种接口类型的速查表。

下面把 **“Trunk 口收发帧处理流程图”** 补进文档，作为 **第十部分**。同时补一张 **Hybrid 口对照流程图**，方便和 Trunk 对比。

---

## 十、Trunk 口收发帧处理流程图

### 1. 总体规则

Trunk 口处理帧，只看两件事：

1. **这个 VLAN 是否在 `allow-pass` 列表里？**
2. **这个 VLAN 是否等于 PVID？**

规则总结：

| 方向 | 判断 | 处理 |
|------|------|------|
| 接收 | 不带标签 | 打上 PVID，归入 PVID 对应 VLAN |
| 接收 | 带标签，VLAN 在 allow-pass | 保留标签，归入该 VLAN |
| 接收 | 带标签，VLAN 不在 allow-pass | 丢弃 |
| 发送 | VLAN = PVID | 去标签发送 |
| 发送 | VLAN ≠ PVID，且在 allow-pass | 带标签发送 |
| 发送 | VLAN 不在 allow-pass | 不发送 |

---

### 2. 接收方向流程图

```
收到一个帧
    │
    ▼
是否带 VLAN 标签？
    │
    ├── 不带标签 ──────────────────────────────┐
    │                                          │
    │                                          ▼
    │                              打上 PVID（如 VLAN 20）
    │                                          │
    │                                          ▼
    │                              归入 PVID 对应的 VLAN
    │                                          │
    │                                          ▼
    │                              检查该 VLAN 是否在 allow-pass
    │                                          │
    │                          ┌───────────────┴───────────────┐
    │                          │                               │
    │                          ▼                               ▼
    │                     在 allow-pass                    不在 allow-pass
    │                          │                               │
    │                          ▼                               ▼
    │                       正常转发                          丢弃
    │
    └── 带标签 ────────────────────────────────┐
                                               │
                                               ▼
                                    读取标签中的 VLAN ID
                                               │
                                               ▼
                                    检查该 VLAN 是否在 allow-pass
                                               │
                               ┌───────────────┴───────────────┐
                               │                               │
                               ▼                               ▼
                          在 allow-pass                    不在 allow-pass
                               │                               │
                               ▼                               ▼
                        保留标签，归入该 VLAN                丢弃
                               │
                               ▼
                            正常转发
```

---

### 3. 发送方向流程图

```
交换机要发出一个 VLAN X 的帧
    │
    ▼
VLAN X 是否在 allow-pass？
    │
    ├── 不在 ────────────────────────────────► 不发送
    │
    └── 在 ──────────────────────────────────┐
                                             │
                                             ▼
                                  VLAN X 是否等于 PVID？
                                             │
                          ┌──────────────────┴──────────────────┐
                          │                                     │
                          ▼                                     ▼
                    等于 PVID                              不等于 PVID
                          │                                     │
                          ▼                                     ▼
                    去标签发送                            带标签发送
                          │                                     │
                          ▼                                     ▼
                  服务器收到不带标签帧                  服务器收到带标签帧
```

---

### 4. 本场景代入

Trunk 口配置：

```
port link-type trunk
port trunk allow-pass vlan 10 20
port trunk pvid vlan 20
```

**接收方向：**

| 服务器发来的帧 | 判断 | 处理 |
|----------------|------|------|
| 不带标签 | 打 PVID 20 | 归入 VLAN 20，allow-pass 里有 20 → 正常转发 |
| 带 VLAN 10 标签 | VLAN 10 在 allow-pass | 保留标签，归入 VLAN 10 → 正常转发 |
| 带 VLAN 30 标签 | VLAN 30 不在 allow-pass | 丢弃 |

**发送方向：**

| 交换机要发的帧 | 判断 | 处理 |
|----------------|------|------|
| VLAN 20 | = PVID | 去标签发送 |
| VLAN 10 | ≠ PVID，在 allow-pass | 带标签发送 |
| VLAN 30 | 不在 allow-pass | 不发送 |

---

## 十一、Hybrid 口收发帧处理流程图（对照）

Hybrid 口比 Trunk 多一个维度：**每个 VLAN 可以单独指定 tagged 或 untagged**。

### 1. 总体规则

| 方向 | 判断 | 处理 |
|------|------|------|
| 接收 | 不带标签 | 打上 PVID，归入 PVID 对应 VLAN |
| 接收 | 带标签，VLAN 在 tagged/untagged 列表 | 保留标签，归入该 VLAN |
| 接收 | 带标签，VLAN 不在任何列表 | 丢弃 |
| 发送 | VLAN 在 untagged 列表 | 去标签发送 |
| 发送 | VLAN 在 tagged 列表 | 带标签发送 |
| 发送 | VLAN 不在任何列表 | 不发送 |

### 2. 接收方向流程图

```
收到一个帧
    │
    ▼
是否带 VLAN 标签？
    │
    ├── 不带标签 ──────────────────────────────┐
    │                                          │
    │                                          ▼
    │                              打上 PVID（如 VLAN 20）
    │                                          │
    │                                          ▼
    │                              归入 PVID 对应的 VLAN
    │                                          │
    │                                          ▼
    │                     该 VLAN 是否在 tagged 或 untagged 列表？
    │                                          │
    │                          ┌───────────────┴───────────────┐
    │                          │                               │
    │                          ▼                               ▼
    │                       在列表                          不在列表
    │                          │                               │
    │                          ▼                               ▼
    │                       正常转发                          丢弃
    │
    └── 带标签 ────────────────────────────────┐
                                               │
                                               ▼
                                    读取标签中的 VLAN ID
                                               │
                                               ▼
                            该 VLAN 是否在 tagged 或 untagged 列表？
                                               │
                               ┌───────────────┴───────────────┐
                               │                               │
                               ▼                               ▼
                            在列表                          不在列表
                               │                               │
                               ▼                               ▼
                        保留标签，归入该 VLAN                丢弃
                               │
                               ▼
                            正常转发
```

### 3. 发送方向流程图

```
交换机要发出一个 VLAN X 的帧
    │
    ▼
VLAN X 在哪个列表？
    │
    ├── 在 untagged 列表 ────────────────────► 去标签发送
    │
    ├── 在 tagged 列表 ──────────────────────► 带标签发送
    │
    └── 不在任何列表 ────────────────────────► 不发送
```

### 4. 本场景代入

Hybrid 口配置：

```
port link-type hybrid
port hybrid pvid vlan 20
port hybrid tagged vlan 10
port hybrid untagged vlan 20
```

**接收方向：**

| 服务器发来的帧 | 判断 | 处理 |
|----------------|------|------|
| 不带标签 | 打 PVID 20 | 归入 VLAN 20，untagged 列表里有 20 → 正常转发 |
| 带 VLAN 10 标签 | VLAN 10 在 tagged 列表 | 保留标签，归入 VLAN 10 → 正常转发 |
| 带 VLAN 30 标签 | VLAN 30 不在任何列表 | 丢弃 |

**发送方向：**

| 交换机要发的帧 | 判断 | 处理 |
|----------------|------|------|
| VLAN 20 | 在 untagged 列表 | 去标签发送 |
| VLAN 10 | 在 tagged 列表 | 带标签发送 |
| VLAN 30 | 不在任何列表 | 不发送 |

---

## 十二、Trunk 与 Hybrid 收发规则对照总表

| 对比项 | Trunk | Hybrid |
|--------|-------|--------|
| 收不带标签帧 | 打 PVID，归入 PVID VLAN | 打 PVID，归入 PVID VLAN |
| 收带标签帧 | 查 allow-pass | 查 tagged/untagged 列表 |
| 发 VLAN = PVID | 去标签 | 看该 VLAN 在哪个列表 |
| 发 VLAN ≠ PVID | 带标签 | 看该 VLAN 在哪个列表 |
| 能否多个 VLAN 去标签 | 不能 | 能 |
| 能否显式指定某 VLAN 带标签 | 不能（被动） | 能（显式） |
| 灵活度 | 低 | 高 |
| 典型用途 | 交换机级联 | 服务器、IP 电话、单臂路由 |

---

## 十三、一句话总记忆

```
Trunk：只有 PVID 去标签，其他都带标签。
Hybrid：每个 VLAN 单独指定 tagged / untagged。
Access：只属于一个 VLAN，收不带标签、发不带标签。
```

---

现在整份文档的结构是：

1. 场景描述
2. 方式一：Trunk 配置
3. 方式二：Hybrid 配置
4. 关键对比
5. 完整配置对照表
6. 验证命令
7. 总结
8. 补充说明：`allow-pass`、`pvid`、`port default vlan` 与 `port trunk pvid` 的差别
9. 三种接口类型速查表
10. Trunk 口收发帧处理流程图
11. Hybrid 口收发帧处理流程图
12. Trunk 与 Hybrid 收发规则对照总表
13. 一句话总记忆

---

## 十四、Access / Trunk / Hybrid 三种口互转等价配置表

这一部分用于实验时**直接替换**：同一个接口，用不同 link-type 实现**相同转发效果**时，命令怎么写。

### 1. Access 效果 → 用 Hybrid 实现

Access 口：

```text
interface GigabitEthernet0/0/1
 port link-type access
 port default vlan 20
```

等价 Hybrid 口：

```text
interface GigabitEthernet0/0/1
 port link-type hybrid
 port hybrid pvid vlan 20
 port hybrid untagged vlan 20
```

效果：

| 行为 | 结果 |
|------|------|
| 收不带标签帧 | 归入 VLAN 20 |
| 发 VLAN 20 帧 | 去标签 |
| 其他 VLAN | 不让过 |

> Access = Hybrid 的“只配一个 untagged VLAN，且 PVID 等于它”。

---

### 2. Trunk 效果 → 用 Hybrid 实现

Trunk 口：

```text
interface GigabitEthernet0/0/24
 port link-type trunk
 port trunk allow-pass vlan 10 20
 port trunk pvid vlan 20
```

等价 Hybrid 口：

```text
interface GigabitEthernet0/0/24
 port link-type hybrid
 port hybrid pvid vlan 20
 port hybrid tagged vlan 10
 port hybrid untagged vlan 20
```

效果：

| 行为 | 结果 |
|------|------|
| 收不带标签帧 | 归入 VLAN 20 |
| 收 VLAN 10 带标签帧 | 归入 VLAN 10 |
| 发 VLAN 20 帧 | 去标签 |
| 发 VLAN 10 帧 | 带标签 |

> Trunk = Hybrid 的“PVID 那个 VLAN 配 untagged，其他允许 VLAN 配 tagged”。

---

### 3. Hybrid 效果 → 用 Trunk 实现（仅当需求刚好匹配时）

Hybrid 口：

```text
interface GigabitEthernet0/0/1
 port link-type hybrid
 port hybrid pvid vlan 20
 port hybrid tagged vlan 10
 port hybrid untagged vlan 20
```

等价 Trunk 口：

```text
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20
 port trunk pvid vlan 20
```

效果：

| 行为 | 结果 |
|------|------|
| 收不带标签帧 | 归入 VLAN 20 |
| 收 VLAN 10 带标签帧 | 归入 VLAN 10 |
| 发 VLAN 20 帧 | 去标签 |
| 发 VLAN 10 帧 | 带标签 |

> 只有当 Hybrid 的规则恰好是“一个 VLAN untagged，其余 VLAN tagged”时，才能等价转换成 Trunk。

---

### 4. Hybrid 效果 → 用 Trunk 实现不了的情况

以下 Hybrid 配置，Trunk **无法等价实现**：

**情况一：两个 VLAN 都去标签**

```text
port link-type hybrid
port hybrid pvid vlan 20
port hybrid untagged vlan 10 20
```

Trunk 做不到，因为 Trunk 只能让 PVID 一个 VLAN 去标签。

**情况二：VLAN 10 去标签、VLAN 20 带标签**

```text
port link-type hybrid
port hybrid pvid vlan 10
port hybrid untagged vlan 10
port hybrid tagged vlan 20
```

Trunk 也做不到，因为 Trunk 的规则是“PVID 去标签，其他带标签”，无法反过来。

**情况三：三个以上 VLAN，需要多个去标签**

```text
port link-type hybrid
port hybrid pvid vlan 20
port hybrid untagged vlan 20 30
port hybrid tagged vlan 10 40
```

Trunk 同样做不到。

---

### 5. 三种口互转总表

| 目标效果 | Access | Trunk | Hybrid |
|----------|--------|-------|--------|
| 只属于一个 VLAN，收发都不带标签 | `port link-type access`<br>`port default vlan 20` | 不适用 | `port link-type hybrid`<br>`port hybrid pvid vlan 20`<br>`port hybrid untagged vlan 20` |
| 多 VLAN 通过，只有一个 VLAN 去标签 | 不适用 | `port link-type trunk`<br>`port trunk allow-pass vlan 10 20`<br>`port trunk pvid vlan 20` | `port link-type hybrid`<br>`port hybrid pvid vlan 20`<br>`port hybrid tagged vlan 10`<br>`port hybrid untagged vlan 20` |
| 多 VLAN 通过，多个 VLAN 去标签 | 不适用 | 做不到 | `port link-type hybrid`<br>`port hybrid pvid vlan 20`<br>`port hybrid untagged vlan 10 20` |
| 多 VLAN 通过，去标签/带标签自由指定 | 不适用 | 做不到 | `port link-type hybrid`<br>`port hybrid pvid vlan X`<br>`port hybrid untagged vlan ...`<br>`port hybrid tagged vlan ...` |

---

### 6. 一句话记忆

```text
Access = Hybrid 只配一个 untagged VLAN
Trunk  = Hybrid 的 PVID 配 untagged，其他允许 VLAN 配 tagged
Hybrid = 每个 VLAN 单独指定 tagged / untagged，最灵活
```

---

## 十五、整份文档结构目录（更新版）

1. 场景描述
2. 方式一：Trunk 配置
3. 方式二：Hybrid 配置
4. 关键对比
5. 完整配置对照表
6. 验证命令
7. 总结
8. 补充说明：`allow-pass`、`pvid`、`port default vlan` 与 `port trunk pvid` 的差别
9. 三种接口类型速查表
10. Trunk 口收发帧处理流程图
11. Hybrid 口收发帧处理流程图
12. Trunk 与 Hybrid 收发规则对照总表
13. 一句话总记忆
14. Access / Trunk / Hybrid 三种口互转等价配置表
15. 整份文档结构目录（更新版）

---

现在这份文档已经比较完整了：

- 有场景、有配置、有对比；
- 有 `allow-pass` / `pvid` / `default vlan` 的概念澄清；
- 有 Trunk / Hybrid 收发流程图；
- 有三种接口类型速查表；
- 有互转等价配置表。

