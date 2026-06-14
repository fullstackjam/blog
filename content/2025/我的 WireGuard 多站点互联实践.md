+++
title = "我的 WireGuard 多站点互联实践"
date = 2025-10-13
description = "使用 WireGuard 构建 Hub-and-Spoke 架构，实现多个家庭网络和移动设备的安全互联。详细记录从网络规划、密钥管理到路由配置的完整过程，包含 OpenWrt 网关配置和常见排错方法。"
tags = ["wireguard", "vpn", "networking", "homelab", "openwrt"]

[extra.comments]
issue_id = 11

[[extra.faq]]
question = "WireGuard 的 Hub-and-Spoke 架构有什么优缺点？"
answer = "优点是配置简单、管理集中，所有节点只需要和中心服务器建立隧道。缺点是中心节点成为单点故障，且所有流量都要经过中心节点转发，增加了延迟。对于家庭使用场景，这个架构的简洁性通常足够。"

[[extra.faq]]
question = "WireGuard 的 AllowedIPs 参数有什么作用？"
answer = "AllowedIPs 有两个作用：一是作为路由表，决定哪些目标 IP 的流量走 VPN 隧道；二是作为访问控制，限制对端只能使用列表中的源 IP 发送数据。配置时需要同时包含对端的 VPN IP 和它代表的内网网段。"

[[extra.faq]]
question = "OpenWrt 路由器做 WireGuard 网关有什么好处？"
answer = "在路由器层面配置 WireGuard，整个局域网内的设备都可以透明地访问远端网络，不需要每台设备单独安装 WireGuard 客户端。这对于 NAS、打印机、IoT 设备等无法安装 VPN 客户端的设备特别有用。"

[[extra.faq]]
question = "WireGuard 隧道断线后能自动重连吗？"
answer = "WireGuard 本身是无状态协议，不存在传统意义上的'断线'。配置 PersistentKeepalive = 25 后，客户端每 25 秒发送一次保活包维持 NAT 映射。网络恢复后隧道会自动恢复，通常无需手动干预。"

[[extra.faq]]
question = "为什么不用 Tailscale 或 ZeroTier 这类现成方案？"
answer = "Tailscale 和 ZeroTier 确实更方便开箱即用，但它们依赖第三方服务器做协调。自建 WireGuard 的优势是完全自主可控、没有设备数量限制、延迟更低（直连云服务器而非绕行第三方节点），并且能更灵活地配置路由策略。"
+++

家里有两个地方都有网络设备，加上经常需要在外面访问家里的 NAS 和虚拟机，一直想搭个 VPN 把这些网络连起来。试过 OpenVPN、IPSec 之后，最终选了 WireGuard：配置文件短、性能好、内核态运行，整体体验比其他方案好不少。

这篇文章记录了我从规划到部署的完整过程，包括踩过的坑和一些实际经验。

<!--more-->

## 需求分析和方案选择

先说说我的具体需求：

- **远程访问**：在外面时能访问家里的 NAS、虚拟机和 K8s 集群
- **网络互通**：两个家庭网络能互通，局域网内的设备不需要单独装 WireGuard 客户端
- **移动接入**：手机和笔记本随时能连回家，用公共 WiFi 时也有安全保障
- **自主可控**：不想依赖 Tailscale、ZeroTier 这类第三方服务，数据走自己的链路

方案对比的时候考虑过几种：

| 方案 | 优点 | 缺点 |
|------|------|------|
| OpenVPN | 成熟稳定，穿透性好 | 性能差，配置复杂，用户态运行 |
| IPSec/IKEv2 | 标准协议，系统原生支持 | 配置非常复杂，排错困难 |
| Tailscale | 开箱即用，NAT 穿透 | 依赖第三方，免费版有设备限制 |
| **WireGuard** | **内核态，性能好，配置简洁** | **需要一台有公网 IP 的服务器** |

最终选了 WireGuard + Hub-and-Spoke 架构。用一台阿里云 ECS（有公网 IP）做中转节点，所有设备都连到这台服务器，由它负责转发流量。架构简单，也方便管理。

## 网络拓扑

下面这张图就是我现在的网络结构：

{{ mermaid(body="
flowchart TD
    Internet[Public Internet]

    subgraph Cloud[Alibaba Cloud ECS Relay Server]
        ECS[<strong>ECS Relay Server</strong><br>Public IP: 203.0.113.1<br>VPN IP: 192.168.151.1/24<br>Port: 24356/UDP]
    end

    subgraph HomeA[Home Network A]
        OP1[<strong>OpenWrt Gateway A</strong><br>LAN: 192.168.2.1/24<br>VPN: 192.168.151.2/24]
        OtherA[Other Devices<br>192.168.2.0/24]
    end

    subgraph HomeB[Home Network B]
        OP2[<strong>OpenWrt Gateway B</strong><br>LAN: 192.168.4.1/24<br>VPN: 192.168.151.3/24]
        OtherB[Other Devices<br>192.168.4.0/24]
    end

    subgraph Mobile[Mobile Devices]
        MAC[<strong>MacBook</strong><br>VPN IP: 192.168.151.4/24]
        IPH[<strong>iPhone</strong><br>VPN IP: 192.168.151.5/24]
    end

    Internet -- Internet Connection --> ECS

    ECS -- WireGuard Tunnel<br>PersistentKeepalive: 25 --> OP1
    ECS -- WireGuard Tunnel<br>PersistentKeepalive: 25 --> OP2
    MAC -- WireGuard Tunnel<br>PersistentKeepalive: 25 --> ECS
    IPH -- WireGuard Tunnel<br>PersistentKeepalive: 25 --> ECS

    OP1 -- LAN Network --> OtherA
    OP2 -- LAN Network --> OtherB
") }}

关键点：

- **VPN 网段**：`192.168.151.0/24`，专门给 WireGuard 隧道用
- **两个家庭网段**：分别是 `192.168.2.0/24` 和 `192.168.4.0/24`，规划时注意不要冲突
- **OpenWrt 网关**：路由器上跑 WireGuard，这样整个局域网的设备都能透明访问远端，不需要每台设备单独安装客户端
- **移动设备**：直连 ECS 中转节点，可以访问所有网段

## WireGuard 基本原理

WireGuard 的核心是公钥加密体系，每个节点持有一对密钥（私钥 + 公钥）。双方交换公钥后即可建立加密隧道，不需要像 OpenVPN 那样搞证书颁发机构（CA）。

下面是连接建立的过程：

{{ mermaid(body="
sequenceDiagram
    participant S as 服务器<br/>(私钥S_priv, 公钥S_pub)
    participant C as 客户端<br/>(私钥C_priv, 公钥C_pub)

    Note over S,C: 配置阶段：交换公钥
    S->>S: 配置文件中添加C_pub
    C->>C: 配置文件中添加S_pub

    Note over S,C: 握手建立过程
    C->>S: 1. 发送握手初始化包<br/>(用S_pub加密，用C_priv签名)
    S->>S: 2. 用S_priv解密，用C_pub验证签名
    S->>C: 3. 发送握手响应包<br/>(用C_pub加密，用S_priv签名)
    C->>C: 4. 用C_priv解密，用S_pub验证签名

    Note over S,C: 双方协商会话密钥
    Note over S,C: 5. 握手完成，生成共享密钥

    Note over S,C: 数据传输
    S->>C: 发送加密数据
    C->>S: 发送加密数据
") }}

几个值得注意的技术细节：

- **Noise 协议框架**：WireGuard 用的是 Noise_IKpsk2 握手模式，基于 Curve25519、ChaCha20、Poly1305、BLAKE2s 这些现代密码学原语
- **1-RTT 握手**：只需要一次往返就能建立连接，比 OpenVPN 的 TLS 握手快得多
- **无状态设计**：没有"连接"的概念，只有密码学路由。网络切换后自动恢复，不需要重新握手
- **密钥轮换**：每 2 分钟自动轮换会话密钥，无需手动干预

## 实际搭建过程

### 1. 安装 WireGuard

各平台的安装都很直观：

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install wireguard

# OpenWrt（通过 opkg）
opkg update && opkg install wireguard-tools kmod-wireguard luci-proto-wireguard
```

手机和电脑直接在应用商店搜 WireGuard 官方 App 即可。

### 2. 生成密钥对

为每个节点生成独立的密钥对：

```bash
# 设置安全的文件权限
umask 077

# 生成密钥对并输出（一次性命令，不留文件）
wg genkey | tee /dev/stderr | wg pubkey
# 第一行输出是私钥，第二行是公钥
```

也可以分步操作，更清晰一些：

```bash
# 生成私钥
wg genkey > server_private.key

# 从私钥推导公钥
wg pubkey < server_private.key > server_public.key

# 查看密钥（注意保管私钥！）
cat server_private.key
cat server_public.key
```

需要为 ECS 服务器、两台 OpenWrt 路由器、MacBook、iPhone 各生成一对。然后交叉配置公钥：每个节点的配置里要包含所有它要通信的对端的公钥。

### 3. 理解 AllowedIPs

这是 WireGuard 配置中最容易出错的地方，值得单独说一下。`AllowedIPs` 同时充当两个角色：

- **出站路由**：目标 IP 匹配这个列表的流量会被发送到这个 Peer
- **入站过滤**：从这个 Peer 收到的数据包，源 IP 必须在这个列表内，否则丢弃

举个例子，在 ECS 服务器上给家庭网络 A 的路由器配置 `AllowedIPs = 192.168.151.2/32, 192.168.2.0/24`，意思是：

1. 发往 `192.168.151.2` 和 `192.168.2.0/24` 的流量走这条隧道
2. 从这条隧道进来的数据包，源 IP 必须是 `192.168.151.2` 或 `192.168.2.0/24` 网段

### 4. 中心节点配置（ECS 服务器）

这是整个架构的核心，所有流量都经过这里转发。

创建 `/etc/wireguard/wg0.conf`：

```ini
[Interface]
Address = 192.168.151.1/24
ListenPort = 24356
PrivateKey = <服务器私钥>

# 启用 IP 转发（让 Linux 转发不同网段的流量）
PostUp = sysctl -w net.ipv4.ip_forward=1

# 配置 iptables：允许 WireGuard 接口的转发，并做 NAT
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer] # 家庭网络 A - OpenWrt 路由器
PublicKey = <路由器A的公钥>
AllowedIPs = 192.168.151.2/32, 192.168.2.0/24

[Peer] # 家庭网络 B - OpenWrt 路由器
PublicKey = <路由器B的公钥>
AllowedIPs = 192.168.151.3/32, 192.168.4.0/24

[Peer] # MacBook
PublicKey = <笔记本的公钥>
AllowedIPs = 192.168.151.4/32

[Peer] # iPhone
PublicKey = <iPhone的公钥>
AllowedIPs = 192.168.151.5/32
```

启动服务并配置防火墙：

```bash
# 启用 WireGuard 并设置开机自启
sudo systemctl enable --now wg-quick@wg0

# 开放 UDP 端口
sudo ufw allow 24356/udp

# 验证隧道状态
sudo wg show
```

`wg show` 的输出会显示每个 Peer 的最近握手时间和传输流量。如果某个 Peer 的 `latest handshake` 是空的，说明连接还没建立成功，需要排查。

### 5. OpenWrt 路由器配置

路由器是这个方案的关键，它作为网关，让整个局域网的设备无感知地访问远端网络。

WireGuard 配置文件（也可以在 LuCI Web 界面中配置）：

```ini
[Interface]
PrivateKey = <路由器私钥>
Address = 192.168.151.2/24

[Peer]
PublicKey = <服务器公钥>
Endpoint = 203.0.113.1:24356
AllowedIPs = 192.168.151.0/24, 192.168.4.0/24
PersistentKeepalive = 25
```

几个关键配置点：

- **AllowedIPs** 包含 VPN 网段和对方的家庭网段，但不包含自己的家庭网段（否则会形成路由环路）
- **PersistentKeepalive = 25**：路由器通常在 NAT 后面，需要定期发送保活包维持映射。25 秒是个比较合适的间隔
- **Endpoint**：只有服务器需要固定的 Endpoint（公网 IP），路由器这边不需要配，WireGuard 会自动记住对端地址

在 OpenWrt 的防火墙设置中，还需要：

1. 将 WireGuard 接口加入一个防火墙区域（比如新建一个 `vpn` 区域）
2. 允许 `lan` 到 `vpn` 的转发
3. 允许 `vpn` 到 `lan` 的转发（如果需要远端访问本地设备）

### 6. 移动设备配置

手机和笔记本的配置相对简单：

```ini
[Interface]
PrivateKey = <设备私钥>
Address = 192.168.151.4/24
DNS = 192.168.151.1

[Peer]
PublicKey = <服务器公钥>
Endpoint = 203.0.113.1:24356
AllowedIPs = 192.168.151.0/24, 192.168.2.0/24, 192.168.4.0/24
PersistentKeepalive = 25
```

这里 `AllowedIPs` 包含了所有需要访问的网段。注意不要设成 `0.0.0.0/0`（除非你想让所有流量都走 VPN），那样会把所有流量都转发到服务器，增加延迟也浪费带宽。

`DNS = 192.168.151.1` 是可选的，如果你在 ECS 上跑了 DNS 服务（比如 Pi-hole）可以用这个配置，否则删掉就行。

### 7. 验证连通性

所有节点配好之后，逐一测试：

```bash
# 在 ECS 上查看所有 Peer 状态
sudo wg show

# 从移动设备 ping 各节点
ping 192.168.151.1   # ECS 服务器
ping 192.168.151.2   # 路由器 A 的 VPN IP
ping 192.168.2.1     # 路由器 A 的 LAN IP
ping 192.168.4.1     # 路由器 B 的 LAN IP

# 跨网段测试（从家庭网络 A 的设备访问家庭网络 B）
ping 192.168.4.100   # 家庭网络 B 中的某台设备

# 查看路由表，确认 VPN 路由正确
ip route | grep wg0
```

## 注意事项和排错

### 安全加固

- **改用非标准端口**：默认的 51820 太容易被扫描，我用的 24356
- **密钥文件权限**：私钥文件设为 600，只有 root 可读
- **精确的 AllowedIPs**：不要用 `0.0.0.0/0`，按需配置具体网段
- **防火墙**：ECS 安全组只放行 WireGuard 的 UDP 端口，不要多开

### 常见问题排查

**隧道建不起来：**
1. 确认双方公钥配置正确（最常见的错误就是公钥私钥搞反了）
2. 确认 ECS 安全组和系统防火墙都放行了 UDP 端口
3. 用 `tcpdump -i eth0 udp port 24356` 在服务器上抓包，看看有没有数据到达

**能 ping 通 VPN IP 但访问不了对端内网：**
1. 检查 ECS 上的 IP 转发是否开启：`cat /proc/sys/net/ipv4/ip_forward`，应该是 `1`
2. 检查 iptables FORWARD 规则是否生效
3. 确认 AllowedIPs 包含了对端的内网网段

**OpenWrt 局域网设备无法访问远端：**
1. 检查 OpenWrt 防火墙是否允许 lan 到 vpn 区域的转发
2. 确认路由表中有到远端网段的路由
3. 在路由器上 `traceroute` 看看流量是否经过了 WireGuard 接口

### 持久化和自启动

确保网络重启后 WireGuard 自动恢复：

```bash
# ECS 上
sudo systemctl enable wg-quick@wg0

# 确认 IP 转发在重启后仍然生效
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard.conf
```

OpenWrt 通过 LuCI 配置的接口默认会开机自启，不需要额外操作。

## 使用体验

用了几个月下来，几点感受：

- **稳定性很好**：手机从 5G 切 WiFi、WiFi 切 5G，基本上无感知切换，WireGuard 的无状态设计在这方面确实有优势
- **延迟可接受**：通过云服务器中转，两个家庭网络之间额外增加大约 10-20ms 的延迟，日常使用完全没问题
- **免维护**：配置好之后基本不用管，服务器重启也会自动恢复
- **OpenWrt 网关方案好用**：家里的 NAS、打印机、监控摄像头这些不能装 VPN 的设备都能被远程访问到，这是最实用的部分

如果设备多了，手动管理配置文件会比较麻烦，可以考虑用 [wg-easy](https://github.com/wg-easy/wg-easy) 这个项目，它提供了一个简洁的 Web 界面来管理 WireGuard。

## 相关资源

- [WireGuard 官方文档](https://www.wireguard.com/quickstart/) -- 安装和基础配置
- [WireGuard 白皮书](https://www.wireguard.com/papers/wireguard.pdf) -- 深入了解协议设计
- [OpenWrt WireGuard 配置指南](https://openwrt.org/docs/guide-user/services/vpn/wireguard/start) -- 路由器端详细配置
- [wg-easy](https://github.com/wg-easy/wg-easy) -- Web 管理界面，适合管理多个 Peer
