+++
title = "我的WireGuard多站点互联实践"
date = 2025-10-13

[extra.comments]
issue_id = 9
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

**配置示例：**
假设为服务器生成了密钥对：
```
priv_key: ABC123...（服务器私钥）
pub_key: XYZ789...（服务器公钥）
```

服务器配置文件中使用自己的私钥：
```ini
[Interface]
PrivateKey = ABC123...  # 服务器自己的私钥
```

客户端配置文件中引用服务器的公钥：
```ini
[Peer]
PublicKey = XYZ789...   # 服务器的公钥
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

**重要配置说明：**

上述iptables规则的作用：
- **FORWARD规则**：允许WireGuard接口的流量转发，实现不同VPN客户端间通信
- **MASQUERADE规则**：对出口流量进行NAT，让VPN客户端可以访问互联网
- **eth0接口**：需要根据服务器实际网卡名称调整（可能是ens3、ens33等）

```bash
# 检查网卡名称
ip route | grep default

# 启用服务
systemctl enable --now wg-quick@wg0

# 开放防火墙端口（根据云服务商调整）
ufw allow 24356/udp

# 验证连接状态和路由
wg show
ip route show table all | grep wg0
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

**移动设备配置模板：**

**MacBook 配置 (macOS)：**
```ini
[Interface]
PrivateKey = <MacBook私钥>
Address = 192.168.151.4/24
DNS = 192.168.151.1, 1.1.1.1

[Peer]
PublicKey = <服务器公钥>
Endpoint = 203.0.113.1:24356
AllowedIPs = 192.168.151.0/24, 192.168.2.0/24, 192.168.4.0/24
PersistentKeepalive = 25
```

OpenWrt 的配置比较麻烦，需要在 Web 界面里设置。基本流程是添加 WireGuard 接口，填好密钥和 IP，然后配置防火墙规则。对端设置里需要打开路由允许的 IP，这样才能保证在局域网内的设备可以通过 OpenWrt 访问异地局域网内的设备。

#### 5. 连接验证
配置完成后，验证网络连通性：

```bash
# 在移动设备上测试连接
ping 192.168.151.1  # 测试到 ECS 服务器
ping 192.168.2.100  # 测试到家庭网络 A 的 NAS
ping 192.168.4.1  # 测试到家庭网络 B 的设备

# 检查路由表
ip route show table all | grep wg0

# 查看WireGuard状态
wg show
```

### 四、关键技术细节与排错

#### 1. 连接稳定性优化
由于大多数网络环境都存在 NAT 超时机制，建议配置 `PersistentKeepalive = 25`（单位：秒），定期发送保活包维持连接。

#### 2. 一些安全注意事项
端口我没用默认的 51820，改成了 24356，虽然作用不大但至少能避开一些自动扫描。密钥文件权限记得设成 600，这个很重要。另外 AllowedIPs 不要设成 0.0.0.0/0 除非你真的需要所有流量都走 VPN，否则会影响正常上网。

#### 3. 常见问题排查
- **连接失败**：首先检查密钥是否匹配，确认防火墙规则是否放行 UDP 流量
- **路由不通**：使用 `wg show` 命令检查对等体状态，确认各节点的 AllowedIPs 设置正确
- **性能问题**：确保中心节点有足够的网络带宽处理所有流量转发
- **DNS 解析**：如需内网 DNS，在客户端配置 DNS = 192.168.151.1

### 五、方案总结与优化思考

用了几个月下来，感觉 WireGuard 真的很香。最明显的是手机连接很稳定，从 4G 切 WiFi 基本不会断，比之前用过的那些 VPN 靠谱多了。而且配置好了之后基本不用管，偶尔重启一下服务器也会自动重连。

唯一稍微麻烦的就是 OpenWrt 那边的配置，需要在界面里点来点去设置路由和防火墙，不过设置好一次就不用动了。如果以后设备多了，可能考虑搞个 wg-easy 这种 Web 界面来管理，不过目前这个规模手动管理完全够用。

### 六、推荐资源

对于家庭用户，以下资源可能会有帮助：

**简化管理工具：**
- [wg-easy](https://github.com/wg-easy/wg-easy): 提供 Web 界面，方便添加和管理设备
- WireGuard 官方移动端 App: iOS 和 Android 都有，界面简洁易用

**参考文档：**
- [WireGuard 官方快速入门](https://www.wireguard.com/quickstart/)
- [OpenWrt WireGuard 配置](https://openwrt.org/docs/guide-user/services/vpn/wireguard/start)
