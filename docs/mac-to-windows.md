# Mac 通过 WireGuard 连接 Windows

本文把本机已有的 Mac 客户端说明、远程访问运行手册、架构和验收记录整理成可复用的脱敏指南。示例地址使用 `10.77.77.0/24`；请替换为不与本地网络冲突、且经批准的 VPN 网段。

## 连接拓扑

```text
Mac 客户端 (10.77.77.3/32)
        │
        │ WireGuard / UDP 51820
        ▼
云端 WireGuard 中继 (10.77.77.1/24)
        │
        │ 私有 VPN 网段
        ▼
Windows 主机 (10.77.77.2/32)
        └── RDP / TCP 3389，仅允许来自 VPN 网段
```

Mac 和 Windows 都主动连接中继。公网只需要 WireGuard 入口；不要把 Windows RDP 或 VNC 端口暴露到公网。

## 前置条件

- 有一台经授权并已配置 WireGuard 的云端中继。
- Windows 版本支持作为远程桌面主机，且管理员已启用 RDP、NLA 和相应防火墙规则。
- Windows 防火墙只允许 VPN 网段访问 RDP。
- 云端安全组只向获批来源开放 WireGuard UDP 端口；SSH 管理端口应限制到管理员来源。

## 1. 安装 Mac 客户端

- 从 [WireGuard 官方安装页](https://www.wireguard.com/install/)安装 macOS 客户端。
- 从 Mac App Store 安装 [Windows App](https://learn.microsoft.com/en-us/windows-app/get-started-connect-devices-desktops-apps)，用于连接远程 Windows PC。

## 2. 在 Mac 上生成密钥

优先用 WireGuard App 创建隧道并生成密钥。若已经安装 WireGuard 命令行工具，也可在 Mac 终端生成：

```bash
umask 077
wg genkey | tee "$HOME/easy-remote-mac-private.key" | wg pubkey > "$HOME/easy-remote-mac-public.key"
```

这条命令只运行一次。私钥文件必须留在 Mac 本机；只把公钥登记到中继端。不要把任何私钥、二维码或完整客户端配置提交到 GitHub。

## 3. 配置 Mac 的 WireGuard 隧道

在 WireGuard App 新建隧道，使用下面的结构，并把尖括号占位符替换为本机生成或管理员提供的值：

```ini
[Interface]
PrivateKey = <MAC_PRIVATE_KEY>
Address = 10.77.77.3/32

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <SERVER_PUBLIC_HOST_OR_IP>:51820
AllowedIPs = 10.77.77.0/24
PersistentKeepalive = 25
```

中继端还需要登记 Mac 的公钥和隧道地址：

```ini
[Peer]
PublicKey = <MAC_PUBLIC_KEY>
AllowedIPs = 10.77.77.3/32
```

`AllowedIPs` 只包含 VPN 网段时，Mac 的普通互联网流量不会被整个导入隧道。只有在明确需要全隧道并理解路由影响时，才调整该设置。

## 4. 启用隧道并检查连通性

启用 Mac 隧道后，在中继端检查 peer 是否出现最新 handshake：

```bash
sudo wg show
```

然后从 Mac 测试 VPN 地址和 RDP 端口：

```bash
ping -c 4 10.77.77.1
ping -c 4 10.77.77.2
nc -vz 10.77.77.2 3389
```

期望结果是能访问中继和 Windows 的 VPN 地址，并且 TCP 3389 只在 VPN 内可达。

## 5. 在 Windows App 里添加 PC

在 Windows App 的 Devices 页面选择“+”并添加 PC：

- PC name：`10.77.77.2`（或管理员提供的 Windows VPN 地址）
- Gateway：No gateway
- User account：填写获批的 Windows 账号
- Friendly name：例如 `Windows via WireGuard`

使用 Windows 登录密码，不是 Windows Hello PIN。连接前确认 Windows 已启用 NLA，且 RDP 入站规则只允许 VPN 网段。

## 排障顺序

| 现象 | 检查 |
| --- | --- |
| Mac 隧道没有 handshake | 隧道是否启用；Endpoint 和服务端公钥是否正确；云安全组是否允许获批来源的 UDP 51820；当前网络是否阻止出站 UDP。 |
| 有 handshake，但 ping 不到 Windows | 检查 Windows WireGuard 服务、peer 公钥、地址和 AllowedIPs；检查 Windows 防火墙是否只对正确 VPN 网段放行。 |
| 能 ping Windows，但 3389 不通 | 检查 Remote Desktop Services、RDP Host 支持、NLA 和 TCP 3389 的防火墙规则。不要通过开放公网 3389 来排障。 |
| Windows App 拒绝登录 | 确认使用获批的 Windows 账号和登录密码，而不是 PIN 或云服务器账号。 |

常用检查命令：

Windows：

```powershell
Get-Service TermService
Get-NetTCPConnection -LocalPort 3389 -State Listen
```

WireGuard 中继：

```bash
sudo systemctl status wg-quick@wg0 --no-pager
sudo wg show
sudo ss -lunp | grep 51820
```

## 输入体验备注

本机还留有 RDP 滚轮方向和 Command/Ctrl 映射的试验脚本。它们没有放入本仓库：AutoHotkey 热键可能影响 Windows 本机输入，不一定只作用于 RDP 窗口。优先使用 Mac 远程桌面客户端的应用级键位设置；如需运行 Windows 侧映射，先确认它被严格限制在远程会话内。

## 本机部署记录（历史）

本机记录的日期为 2026-06-21。当时 Windows peer 尚无 WireGuard handshake，记录推测原因可能是云安全组未开放 UDP 51820，或所在网络阻止出站 UDP。本次只整理文档，没有验证当前网络、服务器或 Windows 配置；部署前应重新检查。

## 不应提交到仓库的内容

- WireGuard、SSH、TLS 或其他私钥和口令。
- 含有私钥的完整隧道配置或二维码。
- 实际公网 IP、内部机器名、个人 Windows 用户名和本机密钥文件路径。
- 云平台密钥、访问令牌、服务器备份或安装包。

本仓库的示例地址、密钥和账号名称均为占位/示例值，不能直接用于真实连接。
