+++
title = "Linux 基础知识：系统启动过程、文件系统与分区"
date = 2022-04-17
description = "深入理解 Linux 系统启动全过程：从 BIOS/UEFI 上电自检到 GRUB 引导加载，从内核初始化到 systemd 服务管理，以及文件系统和分区的基本概念。"
tags = ["Linux", "boot-process", "systemd", "filesystem", "GRUB"]

[extra.comments]
issue_id = 1

[[extra.faq]]
question = "Linux 启动过程有哪些阶段？"
answer = "Linux 启动过程分为六个阶段：上电 → BIOS/UEFI（POST 自检）→ Boot Loader（如 GRUB）→ 内核加载与解压 → initramfs 挂载根文件系统 → init/systemd 启动用户空间服务。"

[[extra.faq]]
question = "MBR 和 GPT 分区方案有什么区别？"
answer = "MBR（主引导记录）使用 512 字节的引导扇区，最多支持 4 个主分区和 2TB 磁盘。GPT（GUID 分区表）是 UEFI 标准的一部分，支持最多 128 个分区和 9.4ZB 磁盘，还有 CRC32 校验保护。"

[[extra.faq]]
question = "initramfs 的作用是什么？"
answer = "initramfs 是一个临时的内存文件系统，内核启动时会将它解压到内存中。它包含挂载真正根文件系统所需的驱动程序和工具（如存储驱动、文件系统驱动），完成挂载后会通过 pivot_root 切换到真实根文件系统并释放内存。"

[[extra.faq]]
question = "systemd 相比传统 SysVinit 有什么优势？"
answer = "systemd 通过并行启动服务大幅缩短了开机时间。它用声明式的 unit 配置文件取代了复杂的 shell 脚本，提供了统一的 systemctl 命令管理服务，并通过 journald 集中管理日志。"

[[extra.faq]]
question = "Linux 支持哪些常见的文件系统？"
answer = "Linux 支持多种文件系统：传统磁盘文件系统如 ext4、XFS、Btrfs；闪存文件系统如 UBIFS、JFFS2；特殊文件系统如 procfs（进程信息）、sysfs（设备信息）、tmpfs（内存临时文件系统）等。"
+++

这篇文章带你走一遍 Linux 系统从按下电源键到出现登录界面的完整过程。我们还会聊聊文件系统和分区这些基础概念。

<!--more-->

## Linux 启动过程

Linux 的启动过程（boot process）是从按下电源按钮到用户界面完全可用的整个初始化流程。这个过程经过多个精心设计的阶段，每个阶段负责不同的任务。

<img src="/images/linux-boot-process-overview.svg" alt="Linux 启动过程总览：从上电到用户登录的六个阶段" style="width:100%;max-width:900px;" />

简单来说，启动过程可以分为六个阶段：

1. **上电** - 硬件接通电源
2. **BIOS/UEFI** - 固件初始化和硬件自检
3. **Boot Loader** - 引导加载程序（如 GRUB）
4. **内核加载** - 解压并初始化 Linux 内核
5. **initramfs** - 临时文件系统，挂载真正的根文件系统
6. **init/systemd** - 启动用户空间服务

下面我们逐一展开。

## BIOS/UEFI：第一步

启动 x86 架构的 Linux 系统，第一步是 **BIOS**（Basic Input/Output System，基本输入输出系统）或更现代的 **UEFI**（Unified Extensible Firmware Interface，统一可扩展固件接口）。

<img src="/images/linux-bios-post.svg" alt="BIOS 初始化和 POST 自检过程" style="width:100%;max-width:900px;" />

### POST 自检做了什么

当你按下电源键，CPU 开始执行固件中的代码。固件首先进行 **POST**（Power-On Self-Test，上电自检），主要完成以下工作：

1. **CPU 测试** - 验证处理器寄存器和基本指令是否正常
2. **内存测试** - 检测 RAM 大小，进行读写完整性校验
3. **设备枚举** - 检测存储设备、显卡、USB、网卡等硬件
4. **查找启动设备** - 按照 CMOS 中设置的启动顺序，找到可引导的设备

BIOS 固件存储在主板上的 ROM/Flash 芯片中。系统时间、日期和启动顺序等配置保存在 **CMOS** 中，由主板上的纽扣电池供电，即使关机也不会丢失。

### BIOS vs UEFI

传统 BIOS 已经用了几十年，但有很多限制（只能寻址 1MB 内存、16 位实模式运行）。现代系统基本都用 **UEFI** 了，它的优势包括：

- 支持 GPT 分区表，突破 2TB 磁盘限制
- 支持 Secure Boot（安全启动），防止未签名的恶意软件
- 自带 FAT32 文件系统驱动，可以直接读取 EFI 分区
- 图形化的设置界面，支持鼠标操作
- 更快的启动速度

## 主引导记录（MBR）与引导加载

POST 完成后，控制权从固件转交给 **引导加载程序**（Boot Loader）。根据系统是传统 BIOS 还是 UEFI，引导加载的方式有所不同。

<img src="/images/linux-mbr-bootloader.svg" alt="MBR 与 GPT/EFI 分区方案对比" style="width:100%;max-width:900px;" />

### MBR（传统方式）

MBR 位于磁盘的第一个扇区（512 字节），结构如下：

- **引导代码**：446 字节，包含第一阶段的引导程序
- **分区表**：64 字节，记录最多 4 个主分区信息
- **引导签名**：2 字节（`0x55AA`），标识这是一个可引导的磁盘

MBR 的局限性：最大只支持 **2TB** 磁盘，最多 **4** 个主分区（可以通过扩展分区绕过这个限制，但方案比较 hacky）。

### GPT（现代方式）

GPT（GUID Partition Table）是 UEFI 标准的一部分：

- 支持最大 **9.4 ZB** 的磁盘（基本上没有限制）
- 最多 **128** 个分区
- 使用 **CRC32 校验和**，数据更安全
- 磁盘末尾有备份 GPT 头，防止单点故障
- 保留了一个 Protective MBR 用于向后兼容

## 引导加载程序的工作过程

Linux 最常用的引导加载程序是 **GRUB**（GRand Unified Bootloader）。它的工作分为两个阶段：

<img src="/images/linux-bootloader-action.svg" alt="BIOS 和 UEFI 两种引导路径的对比" style="width:100%;max-width:900px;" />

### BIOS/MBR 路径

1. **Stage 1** - MBR 中 446 字节的引导代码被加载到内存，它的唯一任务是找到并加载 Stage 2
2. **Stage 1.5** - 位于 MBR 和第一个分区之间的空隙中，包含文件系统驱动，让 Stage 1 能读取 `/boot` 分区
3. **Stage 2** - GRUB 的主体，位于 `/boot/grub/` 目录。显示引导菜单，让你选择要启动的操作系统或内核版本

### UEFI/GPT 路径

1. **UEFI 固件** 直接读取 Boot Manager 配置，知道从哪里找 EFI 应用
2. **EFI 系统分区**（ESP）上存放着 `.efi` 格式的引导文件，比如 `/EFI/ubuntu/grubx64.efi`
3. **GRUB EFI 版本** 被加载，显示同样的引导菜单

不管走哪条路径，最终 GRUB 都会把 **内核镜像**（vmlinuz）和 **initramfs** 加载到内存中。

### 其他引导加载程序

除了 GRUB，还有一些其他选择：

- **ISOLINUX** / **SYSLINUX** - 用于光盘和 USB 启动盘
- **U-Boot** - 嵌入式设备常用（树莓派、路由器等）
- **systemd-boot** - 一个更简单的 UEFI 引导管理器
- **rEFInd** - 一个图形化的 UEFI 引导管理器

## 内核加载与初始化

引导加载程序将内核和 initramfs 都加载到内存后，控制权正式移交给 Linux 内核。

<img src="/images/linux-kernel-loading.svg" alt="Linux 内核加载和初始化过程" style="width:100%;max-width:900px;" />

### 内核启动的几个关键步骤

1. **解压** - `vmlinuz` 是压缩过的内核镜像（z 代表 zlib/gzip，现代内核也支持 zstd），它首先将自己解压到内存中
2. **内存管理初始化** - 设置页表、MMU（内存管理单元）、虚拟内存系统
3. **硬件初始化** - 配置 CPU、中断控制器（IRQ）、定时器、PCI 总线、ACPI 电源管理等
4. **加载内建驱动** - 编译进内核的驱动程序在这个阶段初始化
5. **启动 PID 1** - 挂载 initramfs，执行 `/sbin/init`

### 内核的主要子系统

Linux 内核不是一个简单的程序，它是一个完整的操作系统核心，包含多个子系统：

- **进程管理** - 调度器、fork()、线程、信号处理
- **内存管理** - 虚拟内存、页面缓存、交换空间（swap）
- **VFS 和文件系统** - ext4、XFS、Btrfs 等文件系统的统一接口
- **网络栈** - TCP/IP 协议栈、Socket 接口、netfilter 防火墙
- **设备驱动** - 可加载内核模块（`.ko` 文件）
- **系统调用接口** - 用户空间和内核之间的桥梁
- **安全模块** - SELinux、AppArmor 等访问控制框架

## initramfs：临时根文件系统

这是启动过程中一个巧妙的设计。内核刚加载的时候，它还不知道真正的根文件系统在哪里，也不一定有访问存储设备的驱动。initramfs 就是解决这个鸡生蛋、蛋生鸡问题的。

<img src="/images/linux-initramfs.svg" alt="initramfs 加载和挂载过程" style="width:100%;max-width:900px;" />

### initramfs 的工作流程

1. **解压** - 内核将 initramfs（一个 cpio 归档文件，通常用 gzip 或 zstd 压缩）解压到内存中
2. **udev 设备检测** - 通过 udev 守护进程扫描硬件，加载必要的驱动模块，创建 `/dev` 下的设备节点
3. **挂载根文件系统** - 根据内核参数（`root=`）找到根分区，必要时进行 fsck 检查，以只读方式挂载
4. **pivot_root** - 将挂载点切换为真正的根文件系统，释放 initramfs 占用的内存，执行真正的 `/sbin/init`

### initramfs 里有什么

initramfs 实际上是一个微型 Linux 系统：

- `/bin`、`/sbin` - 基本工具（通常是 busybox 的精简版）
- `/lib/modules` - 存储控制器驱动、文件系统驱动
- `/etc` - udev 规则、modprobe 配置
- `/scripts` - 初始化脚本和钩子函数

你可以用 `lsinitramfs`（Debian/Ubuntu）或 `lsinitrd`（RHEL/Fedora）查看你系统的 initramfs 内容。

## init 进程与 systemd

根文件系统挂载完成后，`/sbin/init` 作为 **PID 1** 进程运行。在现代 Linux 发行版中，`/sbin/init` 实际上是指向 `/lib/systemd/systemd` 的符号链接。

<img src="/images/linux-init-services.svg" alt="systemd 服务管理树" style="width:100%;max-width:900px;" />

### 从 SysVinit 到 systemd

传统的 **SysVinit** 用运行级别（runlevel）和 shell 脚本来管理服务。它是**串行**执行的，一个服务启动完才能启动下一个，在多核处理器时代显得很浪费。

**systemd** 是目前主流发行版的标准，它的核心改进包括：

- **并行启动** - 服务之间没有依赖关系的可以同时启动
- **声明式配置** - 用 `.service`、`.timer` 等 unit 文件代替 shell 脚本
- **按需启动** - 通过 socket activation，服务在第一次被访问时才启动
- **统一日志** - journald 集中收集所有服务的日志

### systemd 的 target 概念

systemd 用 **target** 替代了传统的 runlevel：

| SysVinit Runlevel | systemd Target | 说明 |
|---|---|---|
| 0 | poweroff.target | 关机 |
| 1 | rescue.target | 单用户/救援模式 |
| 3 | multi-user.target | 多用户文本界面 |
| 5 | graphical.target | 图形桌面环境 |
| 6 | reboot.target | 重启 |

### 常用 systemctl 命令

```bash
# 服务管理
sudo systemctl start httpd      # 启动服务
sudo systemctl stop httpd       # 停止服务
sudo systemctl restart httpd    # 重启服务
sudo systemctl status httpd     # 查看状态

# 开机自启
sudo systemctl enable sshd     # 设置开机自启
sudo systemctl disable sshd    # 取消开机自启

# 运行级别/Target
systemctl get-default           # 查看默认 target
sudo systemctl set-default graphical.target  # 设置默认 target

# 日志
journalctl -u sshd             # 查看某服务的日志
journalctl -b                  # 查看本次启动的日志
journalctl -f                  # 实时跟踪日志
```

## Linux 文件系统基础

### 文件系统类型

Linux 支持种类繁多的文件系统：

**传统磁盘文件系统：**
- **ext4** - 最常用的 Linux 文件系统，稳定可靠，支持最大 1EB 文件系统和 16TB 单文件
- **XFS** - 高性能文件系统，擅长处理大文件，RHEL 的默认选择
- **Btrfs** - "下一代"文件系统，支持快照、子卷、透明压缩等高级功能

**闪存文件系统：**
- **UBIFS** / **JFFS2** / **YAFFS** - 专为 NAND Flash 设计，考虑了磨损均衡

**特殊文件系统：**
- **procfs**（`/proc`）- 内核和进程信息的虚拟文件系统
- **sysfs**（`/sys`）- 设备和驱动信息
- **tmpfs**（`/tmp`）- 基于内存的临时文件系统，重启后丢失
- **devtmpfs**（`/dev`）- 设备节点

### 分区与文件系统的区别

这两个概念经常被混淆，其实很简单：

- **分区**（Partition）是磁盘上物理上连续的一段空间
- **文件系统**（Filesystem）是在分区上组织和存储文件的方式

打个比方：分区就像一个空房间，文件系统就是房间里的书架和收纳系统。

| | Windows | Linux |
|---|---|---|
| 分区表示 | Disk 1 | /dev/sda1 |
| 文件系统 | NTFS / FAT32 | ext4 / XFS / Btrfs |
| 挂载方式 | 盘符（C:, D:） | 挂载点（/home, /boot） |
| 根目录 | C:\ | / |

### 文件系统层级标准（FHS）

Linux 使用 `/` 作为路径分隔符（Windows 用 `\`），没有盘符的概念。所有的磁盘和分区都挂载到单一目录树的某个位置。

几个重要的目录：

| 目录 | 用途 |
|---|---|
| `/` | 根目录，一切的起点 |
| `/boot` | 内核和引导文件 |
| `/home` | 用户主目录 |
| `/etc` | 系统配置文件 |
| `/var` | 可变数据（日志、缓存等） |
| `/tmp` | 临时文件 |
| `/usr` | 用户程序和库 |
| `/dev` | 设备文件 |
| `/proc` | 内核/进程虚拟文件系统 |
| `/sys` | 设备/驱动虚拟文件系统 |

可移动设备（U 盘、光盘等）通常会自动挂载到 `/run/media/用户名/卷标` 或 `/media/` 目录下。比如用户名是 `student`，U 盘卷标是 `FEDORA`，那它就会挂载在 `/run/media/student/FEDORA`。

## 总结

Linux 的启动过程是一个精心编排的接力赛：固件负责硬件初始化，引导加载程序负责找到并加载内核，内核负责初始化操作系统核心，initramfs 负责找到并挂载根文件系统，最终 systemd 接管一切并启动用户空间的服务。

理解这个过程不仅帮你排查启动问题（比如 GRUB rescue、内核 panic、服务启动失败），也能让你更深入地理解 Linux 系统的架构设计。
