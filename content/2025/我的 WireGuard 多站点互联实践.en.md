+++
title = "My WireGuard Multi-Site VPN Setup: Connecting Home Networks"
date = 2025-10-13
description = "Building a Hub-and-Spoke WireGuard VPN to connect multiple home networks and mobile devices. A hands-on guide covering network planning, key management, routing configuration, OpenWrt gateway setup, and troubleshooting."
tags = ["wireguard", "vpn", "networking", "homelab", "openwrt"]

[extra.comments]
issue_id = 13

[[extra.faq]]
question = "What are the pros and cons of a Hub-and-Spoke WireGuard architecture?"
answer = "The main advantage is simplicity -- all nodes only need a tunnel to the central server, making configuration and management straightforward. The downside is the central node becomes a single point of failure, and all traffic routes through it, adding latency. For home use, the simplicity usually outweighs the drawbacks."

[[extra.faq]]
question = "What does the AllowedIPs parameter do in WireGuard?"
answer = "AllowedIPs serves two roles: as an outbound routing table (traffic destined for these IPs goes through this tunnel) and as an inbound ACL (packets from this peer must have a source IP within this list, otherwise they are dropped). You need to include both the peer's VPN IP and the LAN subnets it represents."

[[extra.faq]]
question = "Why use an OpenWrt router as a WireGuard gateway?"
answer = "Running WireGuard on the router means every device on the local network can transparently access remote networks without installing a VPN client. This is especially useful for NAS boxes, printers, IoT devices, and other gear that can't run VPN software."

[[extra.faq]]
question = "Does WireGuard automatically reconnect after a network change?"
answer = "WireGuard is stateless by design -- there is no persistent 'connection' to drop. With PersistentKeepalive = 25, the client sends a keepalive packet every 25 seconds to maintain NAT mappings. When the network comes back, the tunnel resumes automatically without manual intervention."

[[extra.faq]]
question = "Why not use Tailscale or ZeroTier instead of self-hosted WireGuard?"
answer = "Tailscale and ZeroTier are great for convenience, but they rely on third-party coordination servers. Self-hosted WireGuard gives you full control over your data path, no device limits, lower latency (direct to your own server), and more flexibility in routing configuration."
+++

I have two homes with network gear in each, plus I frequently need to access my NAS and VMs remotely. I wanted a VPN that could tie everything together. After trying OpenVPN and IPSec, I settled on WireGuard -- the config files are short, performance is excellent, and it runs in kernel space. The overall experience is noticeably better than the alternatives.

This post documents the entire process from planning to deployment, including the pitfalls I hit and practical lessons learned.

<!--more-->

## Requirements and Choosing a Solution

Here's what I needed:

- **Remote access**: Reach my NAS, VMs, and K8s cluster from anywhere
- **Network-to-network**: Both home LANs can talk to each other, with devices on each network not needing to individually run WireGuard
- **Mobile access**: Phone and laptop can connect back home at any time, especially on public WiFi
- **Self-hosted**: No reliance on Tailscale, ZeroTier, or other third-party services -- I want my traffic on my own infrastructure

Here's how the options stacked up:

| Solution | Pros | Cons |
|----------|------|------|
| OpenVPN | Mature, good NAT traversal | Poor performance, complex config, userspace |
| IPSec/IKEv2 | Standard protocol, native OS support | Very complex config, hard to troubleshoot |
| Tailscale | Zero-config, great NAT traversal | Third-party dependency, device limits on free tier |
| **WireGuard** | **Kernel-space, fast, minimal config** | **Needs a server with a public IP** |

I went with WireGuard in a Hub-and-Spoke architecture: one Alibaba Cloud ECS instance with a public IP acts as the relay. All devices connect to this server, which forwards traffic between them. Simple to manage, easy to reason about.

## Network Topology

Here's what my network looks like:

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

Key points:

- **VPN subnet**: `192.168.151.0/24`, dedicated to WireGuard tunnels
- **Two home subnets**: `192.168.2.0/24` and `192.168.4.0/24` -- plan these to avoid conflicts
- **OpenWrt gateways**: Running WireGuard on the router means every device on the LAN gets transparent access to remote networks, no per-device client needed
- **Mobile devices**: Connect directly to the ECS relay, with access to all subnets

## How WireGuard Works

WireGuard's core is public key cryptography. Each node holds a key pair (private + public). Peers exchange public keys and can then establish encrypted tunnels -- no certificate authority (CA) needed like OpenVPN.

Here's how a connection is established:

{{ mermaid(body="
sequenceDiagram
    participant S as Server<br/>(Private: S_priv, Public: S_pub)
    participant C as Client<br/>(Private: C_priv, Public: C_pub)

    Note over S,C: Configuration phase: exchange public keys
    S->>S: Add C_pub to config file
    C->>C: Add S_pub to config file

    Note over S,C: Handshake process
    C->>S: 1. Send handshake initiation<br/>(encrypted with S_pub, signed with C_priv)
    S->>S: 2. Decrypt with S_priv, verify with C_pub
    S->>C: 3. Send handshake response<br/>(encrypted with C_pub, signed with S_priv)
    C->>C: 4. Decrypt with C_priv, verify with S_pub

    Note over S,C: Both sides derive session keys
    Note over S,C: 5. Handshake complete, shared secret established

    Note over S,C: Data transfer
    S->>C: Send encrypted data
    C->>S: Send encrypted data
") }}

Some technical details worth noting:

- **Noise protocol framework**: WireGuard uses the Noise_IKpsk2 handshake pattern, built on Curve25519, ChaCha20, Poly1305, and BLAKE2s
- **1-RTT handshake**: Connection setup takes a single round trip -- much faster than OpenVPN's TLS handshake
- **Stateless design**: There's no "connection" concept, just cryptographic routing. Survives network switches without re-handshaking
- **Key rotation**: Session keys rotate automatically every 2 minutes with zero manual intervention

## Step-by-Step Setup

### 1. Install WireGuard

Installation is straightforward on every platform:

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install wireguard

# OpenWrt (via opkg)
opkg update && opkg install wireguard-tools kmod-wireguard luci-proto-wireguard
```

For phones and laptops, grab the official WireGuard app from the App Store or Google Play.

### 2. Generate Key Pairs

Generate a unique key pair for each node:

```bash
# Set secure file permissions
umask 077

# Generate and display a key pair (one-liner, no files left behind)
wg genkey | tee /dev/stderr | wg pubkey
# First line of output is the private key, second is the public key
```

Or do it step by step for clarity:

```bash
# Generate private key
wg genkey > server_private.key

# Derive public key from private key
wg pubkey < server_private.key > server_public.key

# View the keys (guard the private key!)
cat server_private.key
cat server_public.key
```

You need key pairs for: the ECS server, both OpenWrt routers, MacBook, and iPhone. Then cross-configure the public keys -- each node's config includes the public keys of every peer it needs to talk to.

### 3. Understanding AllowedIPs

This is the most error-prone part of WireGuard configuration, so it deserves its own section. `AllowedIPs` serves two roles simultaneously:

- **Outbound routing**: Traffic destined for IPs in this list gets sent to this peer
- **Inbound filtering**: Packets received from this peer must have a source IP within this list, or they're dropped

For example, on the ECS server, setting `AllowedIPs = 192.168.151.2/32, 192.168.2.0/24` for Home A's router means:

1. Traffic bound for `192.168.151.2` or `192.168.2.0/24` goes through this tunnel
2. Packets arriving through this tunnel must originate from `192.168.151.2` or the `192.168.2.0/24` subnet

### 4. Central Node Configuration (ECS Server)

This is the core of the architecture -- all traffic flows through here.

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 192.168.151.1/24
ListenPort = 24356
PrivateKey = <server_private_key>

# Enable IP forwarding (allow Linux to route between subnets)
PostUp = sysctl -w net.ipv4.ip_forward=1

# iptables rules: allow forwarding on the WireGuard interface + NAT
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer] # Home Network A - OpenWrt router
PublicKey = <router_a_public_key>
AllowedIPs = 192.168.151.2/32, 192.168.2.0/24

[Peer] # Home Network B - OpenWrt router
PublicKey = <router_b_public_key>
AllowedIPs = 192.168.151.3/32, 192.168.4.0/24

[Peer] # MacBook
PublicKey = <macbook_public_key>
AllowedIPs = 192.168.151.4/32

[Peer] # iPhone
PublicKey = <iphone_public_key>
AllowedIPs = 192.168.151.5/32
```

Start the service and configure the firewall:

```bash
# Enable WireGuard and start on boot
sudo systemctl enable --now wg-quick@wg0

# Open the UDP port
sudo ufw allow 24356/udp

# Check tunnel status
sudo wg show
```

The `wg show` output displays each peer's latest handshake time and transfer stats. If a peer's `latest handshake` is empty, the connection hasn't been established yet -- time to troubleshoot.

### 5. OpenWrt Router Configuration

The router is the linchpin of this setup -- as a gateway, it lets every device on the LAN transparently access remote networks.

WireGuard config file (you can also configure this through the LuCI web interface):

```ini
[Interface]
PrivateKey = <router_private_key>
Address = 192.168.151.2/24

[Peer]
PublicKey = <server_public_key>
Endpoint = 203.0.113.1:24356
AllowedIPs = 192.168.151.0/24, 192.168.4.0/24
PersistentKeepalive = 25
```

Key configuration points:

- **AllowedIPs** includes the VPN subnet and the *other* home's subnet, but *not* this router's own LAN subnet (otherwise you create a routing loop)
- **PersistentKeepalive = 25**: Routers are typically behind NAT, so they need periodic keepalive packets to maintain the mapping. 25 seconds is a good interval
- **Endpoint**: Only the server needs a fixed Endpoint (public IP). The router doesn't need one -- WireGuard automatically remembers the peer's address after the first handshake

In OpenWrt's firewall settings, you also need to:

1. Add the WireGuard interface to a firewall zone (e.g., create a `vpn` zone)
2. Allow forwarding from `lan` to `vpn`
3. Allow forwarding from `vpn` to `lan` (if you want remote access to local devices)

### 6. Mobile Device Configuration

Phone and laptop configs are relatively simple:

```ini
[Interface]
PrivateKey = <device_private_key>
Address = 192.168.151.4/24
DNS = 192.168.151.1

[Peer]
PublicKey = <server_public_key>
Endpoint = 203.0.113.1:24356
AllowedIPs = 192.168.151.0/24, 192.168.2.0/24, 192.168.4.0/24
PersistentKeepalive = 25
```

Here `AllowedIPs` includes all subnets you need to reach. Don't set it to `0.0.0.0/0` (unless you want *all* traffic routed through the VPN) -- that would funnel everything through the server, adding latency and wasting bandwidth.

`DNS = 192.168.151.1` is optional. Use it if you're running a DNS server (like Pi-hole) on the ECS instance; otherwise, remove it.

### 7. Verify Connectivity

Once all nodes are configured, test each path:

```bash
# Check all peer status on ECS
sudo wg show

# Ping various nodes from a mobile device
ping 192.168.151.1   # ECS server
ping 192.168.151.2   # Router A's VPN IP
ping 192.168.2.1     # Router A's LAN IP
ping 192.168.4.1     # Router B's LAN IP

# Cross-subnet test (from a device on Home A, reach Home B)
ping 192.168.4.100   # A device on Home Network B

# Verify routing table
ip route | grep wg0
```

## Tips and Troubleshooting

### Security Hardening

- **Use a non-standard port**: The default 51820 is too easy to scan; I use 24356
- **Lock down key permissions**: Private key files should be chmod 600, readable only by root
- **Precise AllowedIPs**: Don't use `0.0.0.0/0`; specify only the subnets you need
- **Firewall**: Only open the WireGuard UDP port in the ECS security group, nothing else

### Common Issues

**Tunnel won't come up:**
1. Double-check that public keys are correct on both sides (the most common mistake is swapping public and private keys)
2. Verify both the ECS security group *and* the system firewall allow the UDP port
3. Run `tcpdump -i eth0 udp port 24356` on the server to see if packets are arriving

**Can ping VPN IPs but can't reach the remote LAN:**
1. Check if IP forwarding is enabled: `cat /proc/sys/net/ipv4/ip_forward` should return `1`
2. Verify iptables FORWARD rules are active
3. Confirm AllowedIPs includes the remote LAN subnet

**OpenWrt LAN devices can't reach remote networks:**
1. Check if OpenWrt firewall allows forwarding from `lan` to the `vpn` zone
2. Verify the routing table has routes to the remote subnets
3. Run `traceroute` from the router to see if traffic goes through the WireGuard interface

### Persistence and Auto-Start

Make sure WireGuard survives reboots:

```bash
# On ECS
sudo systemctl enable wg-quick@wg0

# Make IP forwarding persistent across reboots
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard.conf
```

Interfaces configured through LuCI on OpenWrt auto-start by default -- no extra steps needed.

## Real-World Experience

After running this setup for several months, here are my takeaways:

- **Rock-solid stability**: Switching between 5G and WiFi on my phone is seamless. WireGuard's stateless design really shines here
- **Acceptable latency**: Relaying through the cloud server adds roughly 10-20ms between the two home networks -- perfectly fine for everyday use
- **Zero maintenance**: Once configured, it just works. Server reboots recover automatically
- **The OpenWrt gateway approach is a game-changer**: NAS, printers, security cameras, and other devices that can't run VPN software are all accessible remotely. This is the most practical part of the entire setup

If you have a lot of peers to manage, hand-editing config files gets tedious. Check out [wg-easy](https://github.com/wg-easy/wg-easy) -- it provides a clean web UI for managing WireGuard.

## Resources

- [WireGuard Official Docs](https://www.wireguard.com/quickstart/) -- Installation and basic configuration
- [WireGuard Whitepaper](https://www.wireguard.com/papers/wireguard.pdf) -- Deep dive into the protocol design
- [OpenWrt WireGuard Guide](https://openwrt.org/docs/guide-user/services/vpn/wireguard/start) -- Detailed router-side configuration
- [wg-easy](https://github.com/wg-easy/wg-easy) -- Web management UI for multi-peer setups
