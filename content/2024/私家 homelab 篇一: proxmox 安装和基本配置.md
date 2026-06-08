+++
title = "私家 Homelab 篇一：Proxmox VE 安装和基本配置"
date = 2024-03-04
description = "从零开始搭建 Proxmox VE 虚拟化平台：ISO 镜像制作、系统安装、存储优化、国内源配置、Cloud-Init 模板创建，一篇搞定 Homelab 的虚拟化底座。"
tags = ["homelab", "proxmox", "虚拟化", "linux", "self-hosted"]

[extra.comments]
issue_id = 4

[[extra.faq]]
question = "Proxmox VE 和 ESXi 比，家用选哪个？"
answer = "PVE 免费开源、基于 Debian、支持 KVM 和 LXC 容器，社区活跃。ESXi 免费版功能阉割严重，而且 Broadcom 收购 VMware 后政策更不友好。家用 Homelab 场景下 PVE 是目前最佳选择。"

[[extra.faq]]
question = "Proxmox 安装对硬件有什么要求？"
answer = "最低要求：支持 VT-x/VT-d 的 64 位 CPU、至少 2GB 内存（建议 8GB+）、至少 32GB 存储。网卡建议用 Intel 芯片，兼容性最好。家用 Homelab 推荐至少 32GB 内存和 NVMe SSD。"

[[extra.faq]]
question = "为什么要删掉 local-lvm 只用一个 local 分区？"
answer = "PVE 默认把硬盘分成 root 和 data 两个逻辑卷，data 用的 lvm-thin 格式。单机使用时这种分法浪费空间且管理复杂，不如合并成一个大分区，用目录存储（dir）统一管理 ISO、备份、磁盘镜像。"

[[extra.faq]]
question = "Cloud-Init 模板有什么好处？"
answer = "创建一次模板，以后每次需要新虚拟机只需要克隆 + 改参数，几分钟就能拿到一台配好网络、SSH、基础软件的虚拟机。不需要每次都从头装系统，效率提升巨大。"

[[extra.faq]]
question = "国内网络装 Proxmox 会不会很慢？"
answer = "默认源在国外，确实很慢。但换成中科大或清华镜像源后速度会好很多。建议装完系统后第一件事就是换源，然后再做后续配置。"
+++

折腾 Homelab 有一段时间了，今天有空把基于 Proxmox VE 虚拟化平台的安装和基础配置过程完整记录一下。这是"私家 Homelab"系列的第一篇，先把虚拟化底座搭好。

<!--more-->

## 为什么选 Proxmox VE

选虚拟化平台之前，我也纠结过 ESXi、Unraid、TrueNAS Scale。最后选了 Proxmox VE（以下简称 PVE），理由很简单：

- **免费开源**：基于 Debian，没有功能阉割，企业订阅只是多了技术支持
- **KVM + LXC 双引擎**：重量级服务跑虚拟机（KVM），轻量级服务跑容器（LXC），灵活度拉满
- **Web UI 好用**：浏览器里就能管理所有虚拟机和容器，不需要装额外客户端
- **社区活跃**：论坛、Wiki 质量都很高，遇到问题基本都能搜到

特别是 Broadcom 收购 VMware 之后，ESXi 免费版的前景更不明朗了。PVE 是目前 Homelab 圈子里的主流选择。

## 安装 PVE 系统

### 准备工作

1. 从[官网](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)下载最新版 PVE ISO 镜像
2. 用 [Rufus](https://rufus.ie/en/)（Windows）或 [balenaEtcher](https://etcher.balena.io/)（macOS/Linux）制作 USB 启动盘
3. 确保目标机器的 BIOS 已开启 VT-x / VT-d 虚拟化支持

### 安装步骤

U 盘插入目标机器，重启进入 BIOS，修改启动顺序为 U 盘优先。之后基本就是一路"下一步"：

1. **选择安装盘**：选你要装系统的硬盘，建议用 SSD/NVMe
2. **设置地区和时区**：选 Asia/Shanghai
3. **设置密码和邮箱**：root 密码一定要记牢，邮箱随便填
4. **网络配置**：这一步很关键 -- 建议先插好网线，让它自动获取 IP。记下这个 IP 地址（比如我的是 `192.168.2.217`），之后访问 Web UI 要用

安装完成后就可以拔掉显示器了。从此以后，所有操作都通过 Web UI 或 SSH 完成。

### 访问 Web UI

在同局域网的另一台设备上打开浏览器，访问：

```
https://192.168.2.217:8006/
```

注意几点：
- 是 **https** 不是 http
- 浏览器会提示 HTTPS 证书无效，正常现象，忽略即可
- 用户名是 `root`，密码是安装时设置的

登录后就能看到 PVE 的管理界面了：

<img src="/images/proxmox-dashboard-overview.svg" alt="Proxmox VE Dashboard 概览：左侧树形导航、资源监控面板、系统信息" style="width:100%;max-width:900px;" />

左侧是树形导航，列出了数据中心下所有的节点、虚拟机、容器、存储；右侧是当前选中对象的详情面板，包括 CPU、内存、存储的使用情况和系统信息。

## 基本配置

PVE 装好后，先做几项基础配置，省得后面踩坑。

### 1. 关闭企业订阅源

PVE 默认启用了企业订阅源，没订阅的话 `apt update` 会报错。

**操作路径**：Datacenter -> pve -> Updates -> Repositories

找到 Origin 为 `Proxmox` 的企业源（通常是最后两个），点 `Disable` 关掉。然后点 `Add`，添加 `No-Subscription` 免费源。

### 2. 换国内镜像源

默认源在国外，速度感人。换成中科大镜像：

```bash
# 备份原始源
mkdir -p /etc/apt/sources_backup
cp /etc/apt/sources.list /etc/apt/sources_backup/sources.list.bak
cp /etc/apt/sources.list.d/ceph.list /etc/apt/sources_backup/ceph.list.bak
cp /etc/apt/sources.list.d/pve-enterprise.list /etc/apt/sources_backup/pve-enterprise.list.bak

# 替换 Debian 源为中科大镜像
sed -i 's|^deb http://ftp.debian.org|deb https://mirrors.ustc.edu.cn|g' /etc/apt/sources.list
sed -i 's|^deb http://security.debian.org|deb https://mirrors.ustc.edu.cn/debian-security|g' /etc/apt/sources.list

# 替换 Ceph 源
echo "deb https://mirrors.ustc.edu.cn/proxmox/debian/ceph-quincy bookworm no-subscription" > /etc/apt/sources.list.d/ceph.list

# 更新软件包索引
apt update -y
```

### 3. 修改存储配置

PVE 默认的 `local` 存储只允许存放 ISO 和备份文件。要让它支持虚拟机磁盘等所有类型：

**操作路径**：Datacenter -> Storage -> local -> Edit

把 Content 下的所有类型都勾上（Disk image、ISO image、Container template、Snippets、VZDump backup file 等）。

### 4. 合并磁盘分区（单盘用户推荐）

PVE 安装时默认把硬盘分成两个逻辑卷：`pve-root`（系统分区）和 `pve-data`（数据分区，lvm-thin 格式）。单机使用时，这种分法反而浪费空间。不如合并成一个大分区：

```bash
# 查看当前磁盘使用情况
df -h

# 查看逻辑卷信息
lvdisplay

# 卸载并删除 data 卷
umount /dev/pve/data
lvremove /dev/pve/data

# 把释放的空间全部给 root 卷
lvresize -l +100%FREE /dev/pve/root

# 扩展文件系统
resize2fs /dev/mapper/pve-root
```

操作完成后验证一下：

```bash
df -h
# 输出类似：
# /dev/mapper/pve-root  450G  2.5G  428G   1% /
```

这样整个硬盘的空间就都归 root 分区使用了。

> **注意**：删除 `pve-data` 后，记得去 Web UI 的 Datacenter -> Storage 里把 `local-lvm` 删掉，只保留 `local`。

### 5. 安装常用工具

```bash
apt update -y

# CPU 频率和温度监控
apt install linux-cpupower -y
modprobe msr
echo msr > /etc/modules-load.d/turbostat-msr.conf
chmod +s /usr/sbin/turbostat

# QEMU Guest Agent（让宿主机能正确管理虚拟机）
apt install qemu-guest-agent -y

# fail2ban（防暴力破解 SSH）
apt install fail2ban -y
```

如果你还想在 PVE 的 Summary 页面直接看到 CPU 温度和频率，可以装社区的增强脚本：

```bash
(curl -Lf -o /tmp/temp.sh https://raw.githubusercontent.com/a904055262/PVE-manager-status/main/showtempcpufreq.sh || curl -Lf -o /tmp/temp.sh https://mirror.ghproxy.com/https://raw.githubusercontent.com/a904055262/PVE-manager-status/main/showtempcpufreq.sh) && chmod +x /tmp/temp.sh && /tmp/temp.sh remod
```

## Cloud-Init 模板创建

每次需要新虚拟机都从 ISO 装一遍太蠢了。PVE 支持 Cloud-Init，和 AWS、阿里云那些云平台一样，可以做到克隆模板后自动配置网络、SSH 密钥、用户账号。

### 什么是 Cloud-Init

Cloud-Init 是云环境下的标准初始化工具。虚拟机首次启动时，它会自动完成：

- 设置主机名和网络配置（IP、DNS、网关）
- 注入 SSH 公钥
- 创建用户和设置密码
- 扩展文件系统到整个磁盘

PVE 原生支持 Cloud-Init，但需要使用专门的 Cloud Image（不是普通的 ISO 安装镜像）。

### 创建 Ubuntu Cloud-Init 模板

以 Ubuntu 24.04 LTS 为例：

```bash
# 1. 下载 Ubuntu Cloud Image
wget https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img

# 2. 创建新虚拟机（ID 9000，留给模板用）
qm create 9000 --name ubuntu-noble-template --memory 2048 --net0 virtio,bridge=vmbr0

# 3. 将下载的镜像导入为虚拟机硬盘
qm importdisk 9000 ubuntu-24.04-server-cloudimg-amd64.img local

# 4. 将导入的硬盘通过 SCSI 挂载到虚拟机
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local:9000/vm-9000-disk-0.raw

# 5. Cloud Image 默认只有 2GB，扩容到 50GB
qm resize 9000 scsi0 50G

# 6. 添加 Cloud-Init 光驱
qm set 9000 --ide2 local:cloudinit
```

### 配置并转为模板

命令执行完后，还需要在 Web UI 里完成几步：

1. **调整启动顺序**：Hardware -> Boot Order，确保 `scsi0` 在启动项中且已启用
2. **配置 Cloud-Init 参数**：Cloud-Init 面板中设置：
   - **User**：你的用户名
   - **Password**：登录密码
   - **SSH public key**：你的 SSH 公钥（Cloud Image 默认禁用密码登录，必须设置）
   - **IP Config**：DHCP 或静态 IP
3. **点击 "Regenerate Image"**：让配置生效
4. **启动虚拟机测试**：SSH 登录，确认一切正常
5. **安装基础软件**：

```bash
sudo apt update -y
sudo apt install neovim vim git qemu-guest-agent curl wget openssh-server -y
```

6. **确认 QEMU Guest Agent 已安装并运行**：

```bash
sudo systemctl status qemu-guest-agent
# 如果没有自带，手动安装：
sudo apt install qemu-guest-agent -y
```

7. 根据需要安装 Docker、Python 等基础环境
8. **关闭虚拟机**，然后右键 -> **Convert to Template**

### 从模板克隆新虚拟机

模板创建完成后，以后新建虚拟机只需要：

1. 右键模板 -> Clone
2. 选择 Full Clone（完整克隆）
3. 修改 Cloud-Init 参数（IP、主机名等）
4. 启动，几分钟就能拿到一台可用的虚拟机

## 参考

- [Proxmox Virtual Environment 使用指南 - This Cute World](https://thiscute.world/posts/proxmox-virtual-environment-instruction)
- [Proxmox VE 官方文档](https://pve.proxmox.com/wiki/Main_Page)
- [Cloud-Init 官方文档](https://cloudinit.readthedocs.io/)
