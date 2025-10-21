+++
title = "我的WireGuard多站点互联实践"
date = 2025-10-13

[extra.comments]
issue_id = 11
+++

家里有两个地方都有网络设备，加上经常需要在外面访问家里的 NAS，一直想搭个 VPN 把这些网络连起来。试过几种方案后，最终选择了 WireGuard，主要是因为它性能好，而且相对来说比较好搞。

### 一、我的网络拓扑

我的需求其实很简单：
- 在外面时能访问家里的设备，虚拟机或者 K8s
- 两个家庭网络能互通，在家里时每个设备上不需要单独启动 WireGuard 的客户端就能访问另一家里的设备
- 手机和笔记本随时能连回家，特别是用公共 WiFi 的时候
- 不想用第三方 VPN 服务，数据还是自己掌控比较放心

最后选择了 Hub-and-Spoke 的架构，就是用一台有公网 IP 的云服务器做中转。所有设备都连到这台服务器，然后由它来转发流量。虽然看起来单点故障的风险大一些，但对于家庭使用来说够用，也好管理。

**网络拓扑架构：**
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

上面这个拓扑图基本就是我现在的网络结构。虽然所有流量都要经过云服务器，但是架构也并不复杂。

### 二、WireGuard基本原理

WireGuard 的核心就是公钥加密，每个设备都有一对密钥。你把自己的公钥给别人，别人的公钥配到自己这里，这样就能相互认证了。跟其他 VPN 方案比起来，不用搞什么证书授权中心那一套麻烦事。

**连接建立过程：**
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

WireGuard 用的加密算法比较新，连接建立很快，基本上就是 1-2 秒的事。

### 三、实际搭建过程

#### 1. 安装WireGuard
这一步比较简单，Ubuntu 上直接 `apt install wireguard`，OpenWrt 路由器上也有现成的包，手机和电脑就下官方 App 就行。

#### 2. 密钥对生成
在 ECS 上执行以下命令为每个节点生成密钥对：
```bash
# 设置适当权限并生成密钥对（适合批量生成多对密钥）
umask 077
wg genkey > priv_key && wg pubkey < priv_key > pub_key && echo "priv_key" && cat priv_key && echo "pub_key" && cat pub_key && rm -f priv_key pub_key
```

需要为每个节点（服务器、路由器、手机等）都生成独立的密钥对，然后交叉配置彼此的公钥。

比较重要的一个配置是 `AllowedIPs`，这个参数有两个作用：一是告诉系统哪些 IP 要走 VPN 隧道，二是限制对端能使用哪些 IP。配置的时候需要注意，比如我给家庭网络 A 设置了 `192.168.151.2/32, 192.168.2.0/24`，意思是这个节点可以用 .2 这个 VPN IP，同时也代表整个 192.168.2.0 网段。

#### 3. 中心节点配置（ECS服务器）
在云服务器上创建 `/etc/wireguard/wg0.conf`，示例如下：
```ini
[Interface]
Address = 192.168.151.1/24
ListenPort = 24356
PrivateKey = <服务器私钥>
# 启用IP转发
PostUp = sysctl -w net.ipv4.ip_forward=1
# 配置iptables规则允许转发和NAT
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer] # 家庭网络A的路由器
PublicKey = <路由器A的公钥>
AllowedIPs = 192.168.151.2/32, 192.168.2.0/24

[Peer] # 家庭网络B的路由器
PublicKey = <路由器B的公钥>
AllowedIPs = 192.168.151.3/32, 192.168.4.0/24

[Peer] # 笔记本电脑
PublicKey = <笔记本的公钥>
AllowedIPs = 192.168.151.4/32
```

启动服务并开放端口：
```bash
# 启用服务
systemctl enable --now wg-quick@wg0

# 开放防火墙端口
ufw allow 24356/udp

# 验证连接状态
wg show
```

#### 4. 客户端配置详解

**OpenWrt 路由器配置：**
```ini
[Interface]
PrivateKey = <路由器私钥>
Address = 192.168.151.2/24

[Peer]
PublicKey = <服务器公钥>
Endpoint = <服务器公网IP>:24356
AllowedIPs = 192.168.151.0/24, 192.168.4.0/24
PersistentKeepalive = 25
```

**移动设备配置：**
```ini
[Interface]
PrivateKey = <设备私钥>
Address = 192.168.151.4/24
DNS = 192.168.151.1

[Peer]
PublicKey = <服务器公钥>
Endpoint = <服务器IP>:24356
AllowedIPs = 192.168.151.0/24, 192.168.2.0/24, 192.168.4.0/24
PersistentKeepalive = 25
```

OpenWrt 路由器在 Web 界面配置 WireGuard 接口和防火墙规则即可。

#### 5. 连接验证
```bash
# 测试连通性
ping 192.168.151.1  # 服务器
ping 192.168.2.100  # 家庭网络 A
ping 192.168.4.1   # 家庭网络 B

# 查看状态
wg show
```

### 四、注意事项与排错

**稳定性**：配置 `PersistentKeepalive = 25` 保持连接。

**安全性**：
- 端口改为非默认值（如 24356）
- 密钥文件权限设为 600
- AllowedIPs 避免使用 0.0.0.0/0

**常见问题**：
- **连接失败**：检查密钥匹配，确认防火墙放行 UDP 流量
- **路由不通**：用 `wg show` 检查状态，确认 AllowedIPs 设置正确

### 五、使用体验

用了几个月下来，WireGuard 的稳定性很不错，手机从 5G 切 WiFi 基本不会断线。配置好之后基本免维护，偶尔服务器重启也会自动重连。

OpenWrt 配置稍微麻烦一些，但设置好一次就不用动了。设备多了可以考虑用 wg-easy 这种 Web 界面管理。

### 六、相关资源

- [wg-easy](https://github.com/wg-easy/wg-easy)：Web 管理界面
- [WireGuard 官方文档](https://www.wireguard.com/quickstart/)
- [OpenWrt WireGuard 配置指南](https://openwrt.org/docs/guide-user/services/vpn/wireguard/start)
