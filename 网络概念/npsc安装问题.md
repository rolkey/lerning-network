# npf服务安装与检查


## 🎯 错误排查

```
---------------------------
WinPcap 4.1.3 Setup
---------------------------
A newer version of WinPcap (internal version 5.0.995.510) is already installed on this machine.
The installation will be aborted.
---------------------------
确定
```

- 解决方法

这个报错说明你之前清理的 Npcap 只是表面卸载，它在系统目录里留下的 `wpcap.dll` 和 `Packet.dll` 文件依然存在，并且被标记为“内部版本 5.x”，导致 WinPcap 4.1.3 安装程序误以为系统里已经装了更高版本，直接拒绝安装。

需要手动把这些残留的 DLL 文件重命名或删除，让安装程序“看不见”它们。

### 🛠️ 解决方法

**1. 重命名残留的 DLL 文件**

打开文件资源管理器，分别进入以下两个目录（如果存在对应文件的话）：

*   **`C:\Windows\System32`**
*   **`C:\Windows\SysWOW64`**

在这两个文件夹里，找到以下两个文件，把它们**重命名**（比如在文件名后面加 `.bak` 后缀）：

*   `wpcap.dll` → 改为 `wpcap.dll.bak`
*   `Packet.dll` → 改为 `Packet.dll.bak`

> 如果提示文件正在使用无法重命名，说明还有程序（比如 Wireshark 或 eNSP）在后台运行，先把它们彻底关闭再试。

**2. 重新安装 WinPcap 4.1.3**

完成重命名后，再次运行 `WinPcap_4_1_3.exe`，这次应该就能正常安装了。

### 💡 补充说明

你之前执行 `sc delete npf` 时提示“服务未安装”，是因为 Npcap 的服务名可能不叫 `npf`，或者已经被清除了，但这不影响 DLL 文件残留的判断。安装完 WinPcap 后，记得**重启一次电脑**，再打开 eNSP 的 Cloud 配置界面，看绑定列表里能否出现“Microsoft KM-TEST 环回适配器”。

## 🎯 安装

- 检查是否安装

```bash
C:\Windows\System32>sc query npf

SERVICE_NAME: npf
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 4  RUNNING
                                (STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
```

从命令输出看，**`npf` 服务其实已经安装好了，并且正在运行**。`STATE : 4 RUNNING` 就是明确的证明。

你在 `services.msc` 里找不到它，只是因为图形界面的服务列表默认不显示内核驱动（KERNEL_DRIVER 类型）。它们没有用户态的“服务”界面，所以不会出现在那个列表里，但这完全不影响它的功能。

### ✅ 现在的状态是正常的

WinPcap 的安装和运行状态如下：

*   **驱动服务 `npf`**：已注册、正在运行，启动类型应该是自动。
*   **驱动文件 `npf.sys`**：已在 `C:\Windows\System32\drivers\` 中加载。
*   **用户态 DLL**：`wpcap.dll` 和 `Packet.dll` 已由安装程序写入系统目录。

所以，**底层抓包环境已经就绪，不需要再折腾服务了**。

### 🎯 接下来直接验证 eNSP

现在可以打开 eNSP 测试了：

1.  **以管理员身份运行 eNSP**。
2.  新建一个拓扑，拖入一个 **Cloud**。
3.  双击 Cloud，在“绑定信息”下拉列表里，看是否能找到 **“Microsoft KM-TEST 环回适配器”**。

如果能看到它，说明问题已经解决。如果仍然看不到，那说明问题出在 eNSP 的网卡枚举环节，而不是 WinPcap 底层驱动。此时可以尝试：

*   在 eNSP 菜单栏点击“**工具**” -> “**注册设备**”，重新注册一下。
*   确认环回网卡在 `ncpa.cpl` 里没有被禁用，并且已配置好同网段的 IP 地址。
*   彻底关闭 eNSP（包括右下角托盘图标），再重新打开。