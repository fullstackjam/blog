+++
title = "我的 Dotfiles 项目：现代化开发环境自动化配置"
date = "2025-10-15"
updated = "2026-06-08"
description = "基于 GNU Stow 的纯配置文件管理方案：模块化设计、一条命令部署，搭配 OpenBoot 实现新机器全自动化配置。"
tags = ["dotfiles", "automation", "macos", "stow", "开发环境", "AI工具"]

[extra.comments]
issue_id = 14
[[extra.faq]]
question = "Dotfiles 项目和直接 copy 配置文件有什么区别？"
answer = "直接 copy 的问题是配置散落在 home 目录各处，修改后很难同步回仓库。Dotfiles 项目用 GNU Stow 创建符号链接，配置文件实际存在 git 仓库里，修改即版本控制，随时可以回滚和同步到其他机器。"

[[extra.faq]]
question = "GNU Stow 是什么？为什么用它管理 dotfiles？"
answer = "GNU Stow 是一个符号链接管理工具，最初用于管理 /usr/local 下的软件安装。用在 dotfiles 场景，它能把仓库里按目录组织的配置文件自动链接到 home 目录对应位置，保持仓库结构清晰，同时 home 目录也不乱。"

[[extra.faq]]
question = "为什么软件安装不放在 dotfiles 里？"
answer = "配置文件和软件安装是两个不同的关注点。配置文件变更频繁、需要精细管理；软件安装是一次性的、声明式的。拆开之后 dotfiles 仓库保持纯净，软件安装交给 OpenBoot 这类专门工具，各司其职。"

[[extra.faq]]
question = "新机器上部署需要多长时间？"
answer = "用 OpenBoot 一行命令全自动：装 Homebrew、装软件包、clone dotfiles、部署符号链接、装 Oh-My-Zsh 和插件。整个过程大概 10-15 分钟，大部分时间花在 Homebrew 安装软件包上。"

[[extra.faq]]
question = "--no-folding 参数是什么意思？"
answer = "默认情况下 Stow 会用目录级别的符号链接（folding）——把整个目录链接过去。加了 --no-folding 之后，Stow 只创建文件级别的符号链接，目标目录保持为真实目录。这样其他程序往同一目录写文件时不会出问题，比如 ~/.claude/ 下的 sessions 目录就不应该是符号链接。"
+++

每次换电脑或重装系统，最痛苦的环节不是装软件——Homebrew 一行命令的事——而是那些散落在系统各个角落的配置文件。`.gitconfig` 在 home 目录，SSH config 在 `.ssh/`，Zsh 配置、终端配置、AI 编码工具的配置……手动搬一遍至少半天，而且一定会漏掉什么。

这篇文章记录我怎么用 GNU Stow 把这些配置管起来，搭配 [OpenBoot](https://openboot.dev) 处理软件安装，做到新机器上一条命令全部就位。

<!--more-->

## 核心思路

项目只做一件事：**管理配置文件的符号链接**。软件安装交给 OpenBoot。

{% mermaid() %}
graph LR
    A["curl openboot.dev"] --> B["安装 Homebrew<br/>+ 软件包"]
    A --> C["clone dotfiles<br/>到 ~/.dotfiles"]
    C --> D["make install"]
    D --> D1["Git 配置"]
    D --> D2["SSH 配置"]
    D --> D3["Zsh 配置"]
    D --> D4["Claude Code"]
    D --> D5["Ghostty"]
    D --> D6["OpenCode"]

    style A fill:#EFF6FF,stroke:#2563EB,color:#1E40AF
    style B fill:#ECFDF5,stroke:#059669,color:#065F46
    style C fill:#DBEAFE,stroke:#2563EB,color:#1E40AF
    style D fill:#DBEAFE,stroke:#2563EB,color:#1E40AF
    style D1 fill:#fff,stroke:#94A3B8,color:#64748B
    style D2 fill:#fff,stroke:#94A3B8,color:#64748B
    style D3 fill:#fff,stroke:#94A3B8,color:#64748B
    style D4 fill:#fff,stroke:#94A3B8,color:#64748B
    style D5 fill:#fff,stroke:#94A3B8,color:#64748B
    style D6 fill:#fff,stroke:#94A3B8,color:#64748B
{% end %}

全自动部署只需要一行：

```bash
curl -fsSL openboot.dev/fullstackjam | bash
```

OpenBoot 会自动完成：安装 Homebrew 和软件包、clone dotfiles 到 `~/.dotfiles`、用 Stow 部署配置、安装 Oh-My-Zsh 和插件。

如果只想单独部署配置文件，三条命令：

```bash
git clone https://github.com/fullstackjam/dotfiles.git ~/.dotfiles
cd ~/.dotfiles
make install
```

## 项目结构

项目按功能拆成独立模块，每个目录对应一组配置：

```
dotfiles/
├── Makefile                              # 入口，install / uninstall
├── git/.gitconfig                        # Git 配置
├── ssh/.ssh/config                       # SSH 客户端配置
├── zsh/.zshrc                            # Zsh 配置
├── claude/.claude/CLAUDE.md              # Claude Code 全局指令
├── claude/.claude/settings.json          # Claude Code 设置
├── claude/.claude/statusline-command.sh  # Claude Code 状态栏脚本
├── ghostty/.config/ghostty/config        # Ghostty 终端配置
├── opencode/.config/opencode/opencode.json
├── opencode/.config/opencode/oh-my-openagent.json
├── opencode/.config/opencode/tui.json
└── .gitignore
```

这个结构的关键在于：**每个目录的内部结构和它在 home 目录下的目标路径完全对应**。比如 `ghostty/.config/ghostty/config`，Stow 会把它链接到 `~/.config/ghostty/config`。目录结构本身就是映射规则。

这么拆分有几个实际好处：加配置、删配置互不影响；某个模块的链接炸了，其他照常工作；看目录名就知道这组配置是干嘛的，调试很直观。

## GNU Stow：符号链接的正确打开方式

很多人管理 dotfiles 的方法是直接 copy，或者写一堆 `ln -s`。前者的问题是改了配置没法同步回仓库，后者的问题是链接一多就很难维护。

GNU Stow 把这件事做对了。它的逻辑很简单：把一个目录"映射"到目标目录，自动创建对应的符号链接。

```bash
# 把 git/ 目录下的文件链接到 $HOME
stow --no-folding --target="$HOME" git
```

执行后效果：

```
~/.gitconfig → ~/.dotfiles/git/.gitconfig
~/.ssh/config → ~/.dotfiles/ssh/.ssh/config
~/.zshrc → ~/.dotfiles/zsh/.zshrc
```

这里用了 `--no-folding` 而不是默认行为。默认情况下 Stow 会创建目录级别的符号链接——比如把整个 `~/.claude` 链接到仓库里。但问题是 Claude Code 会在 `~/.claude/` 下创建 `sessions/` 等运行时目录，如果整个目录是符号链接就会出问题。`--no-folding` 强制 Stow 只创建文件级别的链接，目标目录保持为真实目录。

为什么不直接用 `ln -s`？因为 Stow 帮你处理了所有麻烦事：目标路径自动推导、冲突检测。手写 `ln -s` 管五六个文件还行，管几十个就是噩梦。

## Makefile：简单到极致

早期版本的 Makefile 有 install、deploy、status、clean 一大堆 target。后来把软件安装剥离给 OpenBoot 之后，Makefile 变得极其简单：

```makefile
HOME     ?= $(shell echo $$HOME)
STOW     := stow --no-folding -v --target=$(HOME)
PACKAGES := git ssh zsh claude ghostty opencode

.PHONY: install uninstall

install:
	mkdir -p $(HOME)/.ssh $(HOME)/.claude $(HOME)/.config/ghostty $(HOME)/.config/opencode
	$(STOW) $(PACKAGES)

uninstall:
	stow -D --no-folding -v --target=$(HOME) $(PACKAGES)
```

就两个命令：`make install` 部署，`make uninstall` 清理。

`install` 先用 `mkdir -p` 创建必要的目标目录，然后一次性把所有模块 stow 过去。加了 `-v`（verbose），执行时能看到每个符号链接的创建过程。

新增模块也很简单——在 `PACKAGES` 列表里加个名字就行。

---

## 包含了哪些配置

### Git

```ini
[user]
    name = fullstackjam
    email = fullstackjam@outlook.com

[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --decorate --graph --all
    amend = commit --amend --no-edit
    undo = reset HEAD~1
    wip = commit -am "WIP"

[pull]
    rebase = true

[rerere]
    enabled = true

[help]
    autocorrect = 1

[credential]
    helper = osxkeychain
```

除了常用别名，有几个实用配置：`pull.rebase = true` 默认用 rebase 而不是 merge；`rerere` 记住冲突解决方式，下次遇到同样的冲突自动解决；`help.autocorrect` 打错命令会自动修正。

### SSH

```
Host *
    ServerAliveInterval 30
    ControlMaster auto
    ControlPath ~/.ssh/%r@%h:%p
    IdentityAgent ~/Library/Group\ Containers/2BUA8C4S2C.com.1password/t/agent.sock

Host github.com
    HostName ssh.github.com
    User git
    port 443
```

两个重点：用 **1Password SSH Agent** 管理密钥，不再把私钥文件直接放在磁盘上；GitHub 走 **443 端口**（`ssh.github.com`），在某些网络环境下 22 端口被封时照样能 push。

### Zsh

基于 Oh-My-Zsh，插件精简到只留有用的：

```bash
plugins=(git helm kubectl fast-syntax-highlighting zsh-autocomplete)
```

加了几个现代 CLI 工具的集成：

- **fzf**：模糊搜索（`source <(fzf --zsh)`）
- **zoxide**：智能 cd（`eval "$(zoxide init zsh)"`）
- 一个快捷别名：`alias c="claude --dangerously-skip-permissions"`

### Claude Code

这是 2025 年新增的——AI 编码助手的全局配置。包含三个文件：

- **CLAUDE.md**：全局行为指令，定义了编码风格偏好（简洁优先、外科手术式修改、目标驱动执行等）
- **settings.json**：模型选择、权限模式、启用的插件（superpowers、codex 等）
- **statusline-command.sh**：终端状态栏脚本，仿 robbyrussell 风格显示目录、Git 分支、模型和上下文用量

把 Claude Code 配置纳入 dotfiles 管理，换机器时 AI 助手的行为偏好也能无缝迁移。

### Ghostty

[Ghostty](https://ghostty.org) 终端模拟器的配置：

```ini
font-family = "JetBrainsMono Nerd Font Mono"
font-size = 14
background-opacity = 0.95
background-blur = 20
cursor-style = block
cursor-style-blink = false
copy-on-select = clipboard
macos-option-as-alt = left
```

主要是字体、透明度、光标样式、macOS 适配这些。选中即复制、左 Option 当 Alt 键，都是提升终端效率的小细节。

### OpenCode

另一个 AI 编码工具的配置，包含插件设置（oh-my-openagent、superpowers）和 TUI 界面配置。

---

## 怎么定制成你自己的

如果你想 fork 这个项目改成自己的，主要改这几个地方：

1. **Git 身份**：编辑 `git/.gitconfig`，把用户名和邮箱改成你的
2. **SSH**：编辑 `ssh/.ssh/config`，改成你的密钥管理方式
3. **Zsh**：编辑 `zsh/.zshrc`，调整插件和别名
4. **AI 工具**：根据你用的工具，修改 `claude/` 或 `opencode/` 下的配置，或者删掉不需要的模块
5. **新增模块**：建一个新目录（比如 `tmux/`），按 Stow 的结构放好配置文件，在 Makefile 的 `PACKAGES` 里加上名字，如果目标目录不存在还需要在 `mkdir -p` 那行加上

## 小结

这个项目演进了几个版本，最后收敛成一个很清晰的分工：dotfiles 仓库只管配置文件，用 GNU Stow 创建符号链接，Makefile 做入口；软件安装交给 [OpenBoot](https://openboot.dev)，Homebrew、软件包、Oh-My-Zsh 一条命令搞定。

早期版本里软件安装也塞在 dotfiles 仓库，配置和安装脚本搅在一起。把安装剥给 OpenBoot 之后清爽多了，仓库里就剩配置文件和一个极简的 Makefile。新机器上 `curl -fsSL openboot.dev/fullstackjam | bash`，10 分钟搞定。

如果你还在手动搬配置文件，认真考虑搞一个自己的 dotfiles 项目。前期投入两三个小时，之后每次换机器都能省半天。

项目地址：[github.com/fullstackjam/dotfiles](https://github.com/fullstackjam/dotfiles)
