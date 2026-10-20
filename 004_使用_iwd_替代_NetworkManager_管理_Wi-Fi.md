# 使用 iwd 替代 NetworkManager 管理 Wi-Fi

> [!NOTE]
> By WuDiXianXin and DeepSeek

> [!IMPORTANT]
> `iwd` 是轻量的 Wi-Fi 守护进程，适合替代 NetworkManager 的无线管理功能。  
> 它不处理有线网络、VPN、移动宽带、连接共享等；如需要这些功能，请保留 NetworkManager，或配合 `systemd-networkd` 等工具使用。  
> 如果只是遇到 `rfkill` 软阻塞，可先尝试 `rfkill list`、`sudo rfkill unblock wifi`，不一定需要更换网络管理器。

在 Arch Linux 上，有时更新系统并重启后，NetworkManager 会出现软阻塞（soft blocked）或无线管理异常。使用 `iwd` 管理 Wi-Fi 可以避免部分问题，且更加轻量。

## 1. 安装 iwd

```bash
sudo pacman -S iwd
```

如果你使用 AUR helper，也可以：

```bash
paru -S iwd
```

## 2. 停止并禁用 NetworkManager

```bash
sudo systemctl disable --now NetworkManager
```

如果系统里还有独立的 `wpa_supplicant` 服务，并且确认不需要它，可一并禁用：

```bash
sudo systemctl disable --now wpa_supplicant
```

## 3. 卸载 NetworkManager（可选，谨慎操作）

```bash
sudo pacman -Rns networkmanager
```

或：

```bash
paru -Rns networkmanager
```

> ⚠️ **注意**：如果桌面环境、VPN 客户端、网络托盘程序等依赖 NetworkManager，卸载后可能失效。卸载前请确认不会破坏系统功能。

## 4. 配置 iwd 和 DNS

`iwd` 本身不直接处理 DNS。推荐配合 `systemd-resolved` 实现自动 DNS 管理，避免手动写死 `/etc/resolv.conf`。

### 4.1 修改 iwd 配置文件

编辑 `/etc/iwd/main.conf`，没有该文件则创建：

```bash
sudo mkdir -p /etc/iwd
sudoedit /etc/iwd/main.conf
```

写入：

```ini
[General]
EnableNetworkConfiguration=true

[Network]
NameResolvingService=systemd
```

保存退出。

### 4.2 启用并启动 systemd-resolved

```bash
sudo systemctl enable --now systemd-resolved
```

### 4.3 链接 `/etc/resolv.conf`

先查看当前文件状态：

```bash
ls -l /etc/resolv.conf
```

然后链接到 `systemd-resolved` 管理的动态文件：

```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

> 说明：链接到 `stub-resolv.conf` 后，本地程序通常通过 `127.0.0.53` 查询 DNS。若某些特殊程序不兼容，可改用 `/run/systemd/resolve/resolv.conf`，但一般推荐 `stub-resolv.conf`。

### 4.4 启用并启动 iwd

```bash
sudo systemctl enable --now iwd
```

如果 iwd 之前已经启动过，建议重启一次：

```bash
sudo systemctl restart iwd
```

## 5. 验证配置

### 检查 DNS 状态

```bash
resolvectl status
```

正常输出应包含网络接口及其分配的 DNS 服务器。

### 测试域名解析

```bash
resolvectl query archlinux.org
```

也可以用：

```bash
ping archlinux.org
```

但 `ping` 可能受 ICMP 限制，建议以 `resolvectl query` 为准。

## 6. 连接 Wi-Fi

使用 `iwctl`：

```bash
iwctl
```

进入交互界面后：

```text
[iwd]# device list
[iwd]# station <无线设备名> scan
[iwd]# station <无线设备名> get-networks
[iwd]# station <无线设备名> connect <SSID>
```

例如设备名可能是 `wlan0`、`wlan1`、`wlp2s0` 等，请以 `device list` 输出为准。

输入密码后即可连接。退出 `iwctl` 用 `exit` 或 `Ctrl+D`。

查看连接状态：

```bash
iwctl station <无线设备名> show
```

## 7. 可能遇到的问题及排查

### 问题 1：连接后无法解析域名

- **排查**：

```bash
systemctl status systemd-resolved --no-pager
resolvectl status
```

- **解决**：确保 `systemd-resolved` 已启用，然后重启相关服务：

```bash
sudo systemctl restart systemd-resolved
sudo systemctl restart iwd
```

如果每次开机都出现 DNS 未就绪，可优先保证 `systemd-resolved` 在 iwd 之前启用，必要时再考虑调整服务依赖。

### 问题 2：服务冲突

- **现象**：iwd 无法正常工作或网络不稳定。
- **排查**：确保同一个无线接口没有被 NetworkManager、`dhcpcd`、`wpa_supplicant`、`systemd-networkd` 等同时接管。
- **解决**：根据实际情况禁用冲突服务，例如：

```bash
sudo systemctl disable --now NetworkManager dhcpcd
```

> 注意：如果你使用 `systemd-networkd` 管理有线网络，不要直接全局禁用；只需避免它和 iwd 同时管理同一个无线接口。

### 问题 3：手动写入的 `/etc/resolv.conf` 被覆盖

链接到 `stub-resolv.conf` 后，系统不再读取固定 DNS 条目。若之前手动写入了 `nameserver`，这些设置会失效。

解决方式是信任 `systemd-resolved` 自动获取的 DNS，不再手动配置固定 DNS。

### 问题 4：无线被 soft blocked

```bash
rfkill list
sudo rfkill unblock wifi
```

然后重启 iwd：

```bash
sudo systemctl restart iwd
```

## 8. 回退到 NetworkManager

如果后续需要恢复 NetworkManager：

```bash
sudo systemctl disable --now iwd
sudo systemctl enable --now NetworkManager
```

如果已经卸载 NetworkManager，需要重新安装：

```bash
sudo pacman -S networkmanager
sudo systemctl enable --now NetworkManager
```
