# 解决安装问题

## 错误信息

```
---------------------------
WinPcap 4.1.3 Setup
---------------------------
A newer version of WinPcap (internal version 5.0.995.510) is already installed on this machine.
The installation will be aborted.
---------------------------
确定   
```

## 解决方法

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
