+++
title = "我的 Dotfiles 项目：现代化开发环境自动化配置"
date = "2025-10-15"
description = "分享我的 macOS 开发环境自动化配置项目，从手动配置到一键部署的演进历程"
tags = ["dotfiles", "automation", "development", "macos", "devops"]
[extra.comments]
issue_id = 12
+++

## 为什么做这个项目

每次换电脑或者重装系统，最头疼的就是重新配置开发环境。装 Homebrew、配 Git、设置 SSH、调教编辑器...一套下来至少得折腾半天，还经常漏掉什么。

后来实在受不了了，就搞了个 dotfiles 项目，现在新机器上一条命令就能搞定所有配置。

```bash
git clone https://github.com/fullstackjam/dotfiles.git
cd dotfiles
make setup
```

大概 10-15 分钟就完事了。

## 项目结构

整个项目按功能分成了几个模块：

```
dotfiles/
├── Makefile              # 主要入口
├── Brewfile              # Homebrew 包列表
├── scripts/              # 安装脚本
├── git/                  # Git 配置
├── nvm/                  # Node.js 版本管理
├── ssh/.ssh/             # SSH 配置
├── nvim/.config/nvim/    # Neovim 配置
├── zsh/                  # Zsh 配置
├── .gitignore            
└── README.md             
```

这样设计的好处是：
- 可以单独安装某个组件
- 出错了不会影响其他部分
- 调试起来比较方便

## 可用命令

支持一键安装，也支持单独安装某个组件：

```bash
make setup       # 完整安装
make install     # 只装软件
make deploy      # 只部署配置

# 单独安装
make homebrew    # 装 Homebrew
make brewfile    # 装软件包
make stow-git    # 部署 Git 配置
make stow-nvm    # 部署 NVM 配置
make stow-ssh    # 部署 SSH 配置
make stow-nvim   # 部署 Neovim 配置

make help        # 看所有命令
```

## 技术实现

### 用 Stow 管理配置文件

配置文件用 GNU Stow 来管理符号链接，这样 home 目录不会乱，而且容易同步到其他机器。

```bash
# 批量创建符号链接
for dir in */; do
    if [ -d "$dir" ] && [ "$dir" != ".ssh/" ] && [ "$dir" != "scripts/" ]; then
        stow --target="$HOME" --restow "$dir"
    fi
done
```

### Makefile 依赖管理

用 Makefile 来管理安装顺序，确保先装 Homebrew 再装软件包：

```makefile
setup: install deploy

install: homebrew brewfile

homebrew:
    @scripts/01-homebrew.sh

brewfile: homebrew
    @scripts/02-brewfile.sh

deploy: stow-git stow-nvm stow-ssh stow-nvim stow-zsh

stow-git:
    @stow --target="$HOME" --restow git

stow-nvm:
    @stow --target="$HOME" --restow nvm

stow-ssh:
    @stow --target="$HOME" --restow ssh

stow-nvim:
    @stow --target="$HOME" --restow nvim

stow-zsh:
    @stow --target="$HOME" --restow zsh
```

### SSH 密钥加密

SSH 密钥这种敏感文件用 git-crypt 加密，避免泄露：

```bash
# 检查并解锁加密文件
if [ -f "$DOTFILES_DIR/git-crypt-key" ]; then
    if git-crypt status | grep -q "encrypted"; then
        git-crypt unlock "$DOTFILES_DIR/git-crypt-key"
    fi
fi
```

## 包含的工具

主要包含这些工具和配置：

- **Homebrew**：装了 50+ 个常用软件包
- **Git**：配了一些实用的别名和颜色
- **SSH**：针对 GitHub 优化了一下
- **NVM**：Node.js 版本管理
- **Neovim**：基础配置，够用

配置文件方面：
- **Zsh**：oh-my-zsh 主题 + 一些别名
- **Git**：颜色显示、别名、合并工具
- **Neovim**：行号、缩进、搜索这些基础功能

## 安装脚本

脚本里加了一些错误处理和状态提示，出错了能看出来：

```bash
# 彩色输出
print_status() {
    echo -e "${GREEN}=== $1 ===${NC}"
}

print_warning() {
    echo -e "${YELLOW}⚠️  $1${NC}"
}

print_error() {
    echo -e "${RED}❌ $1${NC}"
}
```

还提供了状态检查命令：

```bash
make status
# 输出：
# === Dotfiles Setup Status ===
# Homebrew: ✅ Installed
# git-crypt: ✅ Installed
# SSH setup: ✅ Linked
# Git config: ✅ Linked
# Zsh config: ✅ Linked
```

如果出问题了，可以用 `make clean` 清理掉所有符号链接。

## 自定义配置

想改配置的话：
- **Git**：编辑 `git/.gitconfig`，改姓名邮箱什么的
- **软件包**：编辑 `Brewfile` 添加新包
- **其他配置**：直接改对应目录里的文件
- **新配置**：新建目录，然后在 scripts 里加部署命令

## 总结

这个项目把半天的手动配置工作压缩成一条命令，换机器的时候环境能保持一致。所有配置都在 git 里，随时可以回滚。

如果你也经常折腾开发环境，建议搞个自己的 dotfiles 项目。

项目地址：https://github.com/fullstackjam/dotfiles
