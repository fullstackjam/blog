+++
title = "私家 homelab 篇一: proxmox 安装和基本配置"
date = 2024-03-04

[extra.comments]
issue_id = 4
+++

折腾 homelab 有一段时间了，今天有空记录下基于 proxmox 虚拟化平台的安装和基础配置。

<!--more-->

## 安装 PVE 系统

[官网](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)下载最新版本 PVE ISO 镜像，直接使用 [Rufus](https://rufus.ie/en/) 制作一个 U 盘启动盘。U 盘插入需要安装的机器，重启进入 BIOS 修改 U 盘为启动项，之后基本就是傻瓜式的一直点下一步就可以了。需要注意的就是安装的时候记得插入网线自动获取 IP，之后需要用到这个 IP，比如我这里是 192.168.2.217。

安装完成后就可以断开显示器的连接，之后在另一台设备访问其 Web UI，我这里是 https://192.168.2.217:8006/，注意这里是 https 而非 http，一般会提示 HTTPS 证书无效，忽略即可。接着输入刚刚安装设置的密码，用户名是 root，登录后就可以看到 PROXMOX Web UI 界面了。

大概是下面这个样子：

![image-20240304133518738](https://fullstackjam-1257718633.cos.ap-nanjing.myqcloud.com/imgs/202403041337159.png)

至此，Proxmox 的安装就算完成。

## 基本配置

Proxmox 功能很强大，配置也很多。这里只介绍一些我自己在用的基本配置。

- 关闭企业订阅源

  Datacenter->pve->Update->Repositories 最后两个 Origin 是 Proxmox 的订阅源 disable 掉。

- 修改存储使其支持所有文件类型

  Datacenter->Storage->local 点击 edit 后勾选上 content 中的所有类型。

- 删除lvm，使用整个硬盘的空间

  ```bash
  df -h
  lvdisplay
  umount /dev/pve/data
  lvremove /dev/pve/data
  lvresize -l +100%FREE /dev/pve/root
  resize2fs /dev/mapper/pve-root
  root@pve:~# df -h
  Filesystem            Size  Used Avail Use% Mounted on
  udev                   32G     0   32G   0% /dev
  tmpfs                 6.3G  1.7M  6.3G   1% /run
  /dev/mapper/pve-root  450G  2.5G  428G   1% /
  tmpfs                  32G   46M   32G   1% /dev/shm
  tmpfs                 5.0M     0  5.0M   0% /run/lock
  efivarfs              256K  117K  135K  47% /sys/firmware/efi/efivars
  /dev/nvme0n1p2       1022M   12M 1011M   2% /boot/efi
  /dev/fuse             128M   16K  128M   1% /etc/pve
  tmpfs                 6.3G     0  6.3G   0% /run/user/0
  ```

- 换国内源

  ```
  mkdir /etc/apt/sources_backup
  cp /etc/apt/sources.list /etc/apt/sources_backup/sources.list.bak
  cp /etc/apt/sources.list.d/ceph.list /etc/apt/sources_backup/ceph.list.bak
  cp /etc/apt/sources.list.d/pve-enterprise.list /etc/apt/sources_backup/pve-enterprise.list.bak
  # sources.list
  sed -i 's|^deb http://ftp.debian.org|deb https://mirrors.ustc.edu.cn|g' /etc/apt/sources.list
  sed -i 's|^deb http://security.debian.org|deb https://mirrors.ustc.edu.cn/debian-security|g' /etc/apt/sources.list
  # ceph.list
  echo "deb https://mirrors.ustc.edu.cn/proxmox/debian/ceph-quincy bookworm no-subscription" > /etc/apt/sources.list.d/ceph.list
  sed -i 's|^deb http://ftp.debian.org|deb https://mirrors.ustc.edu.cn|g' /etc/apt/sources.list
  sed -i 's|^deb http://security.debian.org|deb https://mirrors.ustc.edu.cn/debian-security|g' /etc/apt/sources.list
  echo "deb https://mirrors.ustc.edu.cn/proxmox/debian/ceph-quincy bookworm no-subscription" > /etc/apt/sources.list.d/ceph.list
  apt update -y
  ```

- 安装传感器工具

  ```
  apt update -y; apt install linux-cpupower && modprobe msr && echo msr > /etc/modules-load.d/turbostat-msr.conf && chmod +s /usr/sbin/turbostat && echo 成功！
  (curl -Lf -o /tmp/temp.sh https://raw.githubusercontent.com/a904055262/PVE-manager-status/main/showtempcpufreq.sh || curl -Lf -o /tmp/temp.sh https://mirror.ghproxy.com/https://raw.githubusercontent.com/a904055262/PVE-manager-status/main/showtempcpufreq.sh) && chmod +x /tmp/temp.sh && /tmp/temp.sh remod
  ```

- 安装常用工具

  ```
  apt update -y
  apt install qemu-guest-agent fail2ban -y
  ```

## 模版创建

一般配 Linux 虚拟机，我们当然希望能在虚拟机启动时，就自动配置好 IP 地址、SSH 密钥、文件系统自动扩容，这样能免去很多手工操作。cloudinit 就是一个能帮你自动完成这些功能的工具，AWS、阿里云等各大云服务厂商都支持这种配置方式，好消息是 PVE 也支持。

首先 cloudinit 必须使用特殊的系统镜像，这里以 Ubuntu 的 cloudimg为例。首先[下载](https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img)好镜像到Proxmox 宿主机，并以导入的磁盘为该虚拟机的硬盘，命令如下：

```
# 创建新虚拟机
qm create 9000 --name ubuntu-noble-template --memory 2048 --net0 virtio,bridge=vmbr0

# 将下载好的 img/qcow2 镜像导入为新虚拟机的硬盘,注意在镜像当前目录
qm importdisk 9000 ubuntu-24.04-server-cloudimg-amd64.img local

# 通过 scsi 方式，将导入的硬盘挂载到虚拟机上
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local:9000/vm-9000-disk-0.raw

# qcow2 镜像默认仅 2G 大小，需要手动扩容到 32G，否则虚拟机启动会报错
qm resize 9000 scsi0 50G

# 设置后 cloud-init 部分才不再是灰色
qm set 9000 --ide2 local:cloudinit
```

上面的工作都完成后，还需要做一些后续配置

1. 调整启动顺序，确保 scsi0 设备在启动项中被启用
2. 手动设置 cloud-init 参数，重新生成 cloudinit image，启动虚拟机，并通过 ssh 登入远程终端
   1. cloud image 基本都没有默认密码，并且禁用了 SSH 密码登录。必须通过 cloud-init 参数添加私钥、设置账号、密码、私钥。
   2. `apt install neovim vim git qemu-guest-agent curl wget openssh-server git -y`
3. 检查 qemu-guest-agent，如果未自带，一定要手动安装它！
   1. ubuntu 需要通过 `sudo apt install qemu-guest-agent` 手动安装它。
4. 安装所需的基础环境，如 docker/docker-compose/vim/git/python3。
5. 关闭虚拟机，然后将虚拟机设为模板。

接下来就可以从这个模板虚拟机，克隆各类新虚拟机了~

##  参考

- [Proxmox Virtual Environment 使用指南 - This Cute World](https://thiscute.world/posts/proxmox-virtual-environment-instruction)
