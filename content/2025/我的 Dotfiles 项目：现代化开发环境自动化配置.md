+++
title = "我的 Dotfiles 项目：现代化开发环境自动化配置"
date = "2025-10-15"
description = "从零搭建一个基于 GNU Stow + Makefile 的 dotfiles 项目：模块化设计、一键部署、符号链接管理。附完整代码和项目结构。"
tags = ["dotfiles", "automation", "macos", "stow", "makefile", "开发环境"]

[extra.comments]
issue_id = 12

[[extra.faq]]
question = "Dotfiles 项目和直接 copy 配置文件有什么区别？"
answer = "直接 copy 的问题是配置散落在 home 目录各处，修改后很难同步回仓库。Dotfiles 项目用 GNU Stow 创建符号链接，配置文件实际存在 git 仓库里，修改即版本控制，随时可以回滚和同步到其他机器。"

[[extra.faq]]
question = "GNU Stow 是什么？为什么用它管理 dotfiles？"
answer = "GNU Stow 是一个符号链接管理工具，最初用于管理 /usr/local 下的软件安装。用在 dotfiles 场景，它能把仓库里按目录组织的配置文件自动链接到 home 目录对应位置，保持仓库结构清晰，同时 home 目录也不乱。"

[[extra.faq]]
question = "新机器上部署 dotfiles 需要多长时间？"
answer = "git clone 加上 make setup，整个过程大概 10-15 分钟。其中大部分时间花在 Homebrew 安装软件包上，配置文件的符号链接部署只需要几秒钟。"

[[extra.faq]]
question = "Dotfiles 项目能和 chezmoi 或 yadm 这类工具配合用吗？"
answer = "可以，但没必要。chezmoi 和 yadm 本质上也是管 dotfiles 的符号链接或模板，和 GNU Stow 功能重叠。如果你已经有了基于 Stow 的方案，切换到 chezmoi 并不会带来明显收益。选一个用好就行。"

[[extra.faq]]
question = "如何在 dotfiles 里安全存储 SSH 私钥等敏感文件？"
answer = "不要把私钥明文提交到 git。可以用 git-crypt 对敏感文件加密，或者用 .gitignore 排除私钥文件，只提交 SSH config。本项目用了 git-crypt 来处理需要加密的配置。"
+++

每次换电脑或重装系统，最痛苦的环节不是装软件——Homebrew 一行命令的事——而是那些散落在系统各个角落的配置文件。`.gitconfig` 在 home 目录，Neovim 配置在 `.config/nvim/`，SSH config 在 `.ssh/`，Zsh 配置还分了好几个文件。手动搬一遍至少半天，而且一定会漏掉什么。

这篇文章记录我怎么用 GNU Stow + Makefile 把这些配置管起来，做到新机器上一条命令全部就位。

<!--more-->

---

## 核心思路

整个项目的设计其实就三步：

{% mermaid() %}
graph LR
    A["git clone<br/>拉取仓库"] --> B["make setup"]
    B --> C["install<br/>安装软件"]
    B --> D["deploy<br/>部署配置"]
    C --> C1["Homebrew"]
    C --> C2["Brewfile 软件包"]
    D --> D1["Git 配置"]
    D --> D2["Zsh 配置"]
    D --> D3["Neovim 配置"]
    D --> D4["SSH 配置"]
    D --> D5["NVM 配置"]

    style A fill:#EFF6FF,stroke:#2563EB,color:#1E40AF
    style B fill:#DBEAFE,stroke:#2563EB,color:#1E40AF
    style C fill:#ECFDF5,stroke:#059669,color:#065F46
    style D fill:#ECFDF5,stroke:#059669,color:#065F46
    style C1 fill:#fff,stroke:#94A3B8,color:#64748B
    style C2 fill:#fff,stroke:#94A3B8,color:#64748B
    style D1 fill:#fff,stroke:#94A3B8,color:#64748B
    style D2 fill:#fff,stroke:#94A3B8,color:#64748B
    style D3 fill:#fff,stroke:#94A3B8,color:#64748B
    style D4 fill:#fff,stroke:#94A3B8,color:#64748B
    style D5 fill:#fff,stroke:#94A3B8,color:#64748B
{% end %}

**克隆仓库 → 安装依赖 → 符号链接部署配置**。就这么简单。

```bash
git clone https://github.com/fullstackjam/dotfiles.git
cd dotfiles
make setup
```

大概 10-15 分钟，新机器的开发环境就和旧机器一模一样。

---

## 项目结构

项目按功能拆成独立模块，每个目录对应一组配置：

```
dotfiles/
├── Makefile              # 入口，定义所有安装和部署命令
├── Brewfile              # Homebrew 软件包清单
├── scripts/              # 安装脚本（Homebrew、Brewfile 等）
│   ├── 01-homebrew.sh
│   └── 02-brewfile.sh
├── git/                  # Git 配置 → 链接到 ~/.gitconfig
├── zsh/                  # Zsh 配置 → 链接到 ~/.zshrc 等
├── nvim/.config/nvim/    # Neovim 配置 → 链接到 ~/.config/nvim/
├── nvm/                  # NVM 配置 → 链接到 ~/.nvmrc 等
├── ssh/.ssh/             # SSH 配置 → 链接到 ~/.ssh/config
├── .gitignore
└── README.md
```

这个结构的关键在于：**每个目录的内部结构和它在 home 目录下的目标路径完全对应**。比如 `nvim/.config/nvim/init.vim`，Stow 会把它链接到 `~/.config/nvim/init.vim`。你不需要写任何映射规则，目录结构本身就是规则。

这么设计有几个好处：

- **模块独立**。想单独更新 Git 配置？`make stow-git`。不影响其他任何东西。
- **出错可控**。某个模块的链接炸了，其他配置照常工作。
- **调试直观**。看目录名就知道这组配置是干嘛的，不用翻脚本。

---

## GNU Stow：符号链接的正确打开方式

很多人管理 dotfiles 的方法是直接 copy，或者写一堆 `ln -s`。前者的问题是改了配置没法同步回仓库，后者的问题是链接一多就很难维护。

GNU Stow 把这件事做对了。它的逻辑很简单：把一个目录"映射"到目标目录，自动创建对应的符号链接。

```bash
# 把 git/ 目录下的文件链接到 $HOME
stow --target="$HOME" --restow git
```

执行后，`git/.gitconfig` 就变成了 `~/.gitconfig` 的符号链接。`--restow` 参数的意思是先清理旧链接再重建，保证幂等——跑多少遍结果都一样。

批量部署所有模块：

```bash
for dir in */; do
    if [ -d "$dir" ] && [ "$dir" != "scripts/" ]; then
        stow --target="$HOME" --restow "$dir"
    fi
done
```

跳过 `scripts/` 是因为那个目录存的是安装脚本，不是配置文件，不需要链接到 home 目录。

为什么不直接用 `ln -s`？因为 Stow 帮你处理了所有麻烦事：目标路径自动推导、冲突检测、清理旧链接。手写 `ln -s` 管五六个文件还行，管几十个就是噩梦。

---

## Makefile：依赖关系和执行顺序

用 Makefile 来编排安装流程，天然支持依赖管理。Homebrew 得先装好才能跑 `brew install`，这种顺序关系用 Makefile 表达最自然：

```makefile
# 完整安装：先装软件，再部署配置
setup: install deploy

# 安装阶段：先装 Homebrew，再装软件包
install: homebrew brewfile

homebrew:
	@scripts/01-homebrew.sh

brewfile: homebrew
	@scripts/02-brewfile.sh

# 部署阶段：把所有配置文件链接到 home 目录
deploy: stow-git stow-nvm stow-ssh stow-nvim stow-zsh

stow-git:
	@stow --target="$(HOME)" --restow git

stow-nvm:
	@stow --target="$(HOME)" --restow nvm

stow-ssh:
	@stow --target="$(HOME)" --restow ssh

stow-nvim:
	@stow --target="$(HOME)" --restow nvim

stow-zsh:
	@stow --target="$(HOME)" --restow zsh
```

`make setup` 一条命令走完所有流程。也可以只跑某一步：

```bash
make setup       # 完整安装（软件 + 配置）
make install     # 只装软件
make deploy      # 只部署配置文件

# 单独操作某个模块
make homebrew    # 安装 Homebrew
make brewfile    # 安装 Brewfile 里的软件包
make stow-git    # 只部署 Git 配置
make stow-nvim   # 只部署 Neovim 配置

make help        # 查看所有可用命令
```

Makefile 比 shell 脚本好的地方在于：依赖关系写在规则里，Make 自动决定执行顺序和跳过已完成的步骤。不需要自己写一堆 `if` 判断。

---

## 包含了哪些配置

### Git

```ini
[user]
    name = fullstackjam
    email = openbootdotdev@gmail.com

[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --graph --decorate

[color]
    ui = auto

[merge]
    tool = vimdiff
```

常用别名、颜色显示、合并工具。这些配置不装不影响干活，但装了之后每天能省不少击键。

### SSH

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes
    UseKeychain yes
```

针对 GitHub 的连接优化：自动加载密钥、使用 macOS Keychain 记住密码。不用每次 push 都输密码。

### Zsh

基于 Oh-My-Zsh，配了常用别名和插件。主要是：

- 一组 Git 快捷别名（`gst`、`gco`、`gp` 之类）
- 语法高亮和自动补全插件
- 自定义 prompt 主题

### Neovim

基础但够用的配置：行号显示、合理的缩进（4 空格）、增量搜索、高亮当前行。不追求 IDE 级别的功能，日常改配置文件和快速编辑足够了。

### Brewfile

50+ 个常用软件包，包括 CLI 工具（ripgrep、fd、bat、fzf、lazygit）和 GUI 应用（VS Code、Warp 等）。所有包都写在一个 `Brewfile` 里，用 `brew bundle` 一次性安装。

---

## 安装脚本的细节

脚本里做了两件事情来提升体验：**彩色状态输出**和**错误处理**。

```bash
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

print_status() {
    echo -e "${GREEN}=== $1 ===${NC}"
}

print_warning() {
    echo -e "${YELLOW}Warning: $1${NC}"
}

print_error() {
    echo -e "${RED}Error: $1${NC}"
}
```

装了 50 多个包，没有状态输出的话根本不知道进度到哪了。颜色区分正常信息、警告和错误，出问题一眼就能看出来。

还有一个状态检查命令，装完之后可以验证：

```bash
make status
# === Dotfiles Setup Status ===
# Homebrew: Installed
# git-crypt: Installed
# SSH setup: Linked
# Git config: Linked
# Zsh config: Linked
# Neovim config: Linked
```

如果某个组件显示有问题，可以单独重新部署：`make stow-git` 之类的。

搞砸了也不慌，`make clean` 清理所有符号链接，恢复到初始状态。

---

## 怎么定制成你自己的

如果你想 fork 这个项目改成自己的，主要改这几个地方：

1. **Git 身份**：编辑 `git/.gitconfig`，把用户名和邮箱改成你的
2. **软件包**：编辑 `Brewfile`，删掉不需要的、加上你要的
3. **SSH**：编辑 `ssh/.ssh/config`，改成你的密钥路径
4. **Zsh**：编辑 `zsh/.zshrc`，调整别名和插件
5. **新增模块**：建一个新目录（比如 `tmux/`），按 Stow 的结构放好配置文件，然后在 Makefile 里加一条 `stow-tmux` 规则

Stow 的设计使得新增模块非常简单——建目录、放文件、加规则，三步搞定。不需要改任何全局配置。

---

## 总结

这个项目的核心价值就一句话：**把散落在系统各处的配置文件收拢到一个 git 仓库里，用符号链接保持同步**。

具体来说：

- **GNU Stow** 负责符号链接的创建和管理，不用手写 `ln -s`
- **Makefile** 负责安装顺序和依赖关系，一条命令走完全流程
- **模块化设计** 让每组配置独立，可以单独操作

换机器的时候，clone + make setup，10 分钟搞定。所有配置都在 git 里，随时可以回滚。

如果你还在手动搬配置文件，认真考虑搞一个自己的 dotfiles 项目。前期投入两三个小时，之后每次换机器或重装系统都能省半天。

项目地址：[github.com/fullstackjam/dotfiles](https://github.com/fullstackjam/dotfiles)
