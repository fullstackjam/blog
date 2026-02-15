+++
title = "配了 20 多台 Mac 的开发环境后，这是我试过的每一种自动化方案"
date = 2026-02-15
description = "一个开发者的 Mac 环境自动化实战总结 — 从 Brewfile 到 nix-darwin，5 种方案的真实体验和踩坑记录。附 2026 年必装工具清单和国内镜像源配置。"
tags = ["mac", "开发环境", "自动化", "效率工具", "homebrew"]

[extra.comments]
issue_id = 14

[[extra.faq]]
question = "新 Mac 配开发环境最快的方法是什么？"
answer = "用 Homebrew 装包 + OpenBoot 之类的自动化工具处理剩余配置，整个过程 15-30 分钟。手动配的话至少半天，而且接下来一个月都在补漏。"

[[extra.faq]]
question = "Brewfile 够用吗？还是需要更完整的自动化？"
answer = "Brewfile 只管安装软件包，大概覆盖 30% 的配置工作。Shell 配置、macOS 偏好设置、Git 身份、dotfiles 都不管。如果你只需要装包，Brewfile 足够。如果想一步到位，需要 OpenBoot、nix-darwin 或者自己写脚本。"

[[extra.faq]]
question = "国内网络慢，Homebrew 装包怎么办？"
answer = "把 Homebrew 源换成清华或中科大镜像，速度可以从几 KB/s 提升到几 MB/s。npm 用淘宝镜像，pip 用清华源。镜像源配置是国内开发者配环境的第一步。"

[[extra.faq]]
question = "nix-darwin 值得学吗？"
answer = "nix-darwin 能实现最高程度的系统可复现性，同一份配置在任何 Mac 上都能生成完全一致的环境，还能回滚。但学习曲线很陡，Nix 语言至少要花 1-2 周才能上手。如果你在 Linux 上已经用 Nix，值得。否则要掂量投入产出比。"

[[extra.faq]]
question = "团队怎么统一开发环境？"
answer = "用 OpenBoot 可以把环境配置导出为一个链接，新人一条命令就能复制整个团队的开发环境。这比维护一份永远过时的飞书文档靠谱得多。"
+++

> 本文也有[英文版](/2026/mac-setup-automation-guide-2026/)，写得更详细，有完整的命令和预设配置。

配新 Mac 这件事，我已经干过二十多次了。自己的、公司的、借的、还有凌晨两点因为把 Python 环境搞炸了怒而重装的。

每次都一样的剧情：拆箱，兴奋十二分钟，打开终端，一片荒漠。没有 Git，没有 Node，没有 Docker。Shell 是光秃秃的 zsh，连你叫什么都不知道。Finder 默认藏着文件扩展名，Dock 动画慢得让人想砸电脑。

然后你开始手动装。两小时过去，装了一半。一周后发现忘了 jq。两周后发现 git 邮箱没配，提交记录全是 "unknown"。

下面是我踩过的所有坑。

---

## 先说一个事实：你的环境没你以为的那么简单

我数过，我的开发环境有 83 个东西：包管理器、CLI 工具（30+）、GUI 应用（15+）、编程语言和运行时、Shell 配置、dotfiles、Git 身份、macOS 偏好设置。加起来轻松 **60-100 项**。手动配大半天，而且接下来一个月你都在补漏。

---

## 方案一：Brewfile

最简单的方案。Homebrew 自带，零学习成本。

```bash
# 从当前 Mac 导出
brew bundle dump --file=~/Brewfile

# 在新 Mac 上恢复
brew bundle --file=~/Brewfile
```

导出来长这样：

```ruby
brew "ripgrep"
brew "fd"
brew "bat"
brew "fzf"
brew "lazygit"
cask "visual-studio-code"
cask "warp"
cask "raycast"
```

优点是简单直接，一个文件扔 GitHub 就行。

问题是它只管装包。Shell 配置、macOS 偏好、Git 身份，一个都不管，大概只搞定了三成。而且 80 行的 Brewfile 维护起来很痛苦 — 三个月后你根本不记得某个包是干嘛装的。

---

## 方案二：Shell 脚本

Brewfile 不够用，你就开始写 bash — 把 Homebrew 安装、`brew install`、Oh-My-Zsh、`defaults write`、`git config` 全塞一个脚本里。管的事情多了，但脚本不幂等（跑两遍就报错），网络一抖就全挂（国内家常便饭），而且 200 行的 bash 维护起来想死。

---

## 方案三：Dotfile 管理器

**chezmoi** 能模板化 dotfiles，一个仓库管多台机器。但学习成本不低，我花在学它目录结构上的时间比花在配 dotfiles 上还多。**dotbot** 轻量只管 symlink。**mackup** 专门备份 VS Code、iTerm2 之类的应用设置到 iCloud。

共同的问题：只管配置，不管安装。我一度同时维护 Brewfile + dotbot + macOS 偏好脚本。三个系统管一台电脑，累了。

---

## 方案四：上硬核手段

### nix-darwin

折腾了一个周末。本以为能搞出一套完美可复现的系统，结果收获了一堆报错和对 Nix 日常用户的敬意。

```nix
{ pkgs, ... }: {
  environment.systemPackages = with pkgs; [
    ripgrep fd bat fzf nodejs go
  ];
  homebrew.casks = [ "visual-studio-code" "warp" ];
  system.defaults.dock.autohide = true;
}
```

这配置确实优雅。同一份文件在任何 Mac 上生成一模一样的环境，还能回滚。但 Nix 有自己的语言、自己的包管理器、自己的世界观。我那个周末大部分时间在 Nix 论坛查 shell 为什么加载不出来。

如果你在 Linux 上已经用 Nix，上手会快很多。没用过的话，准备投入一到两周。

**Ansible** — Jeff Geerling 的 [mac-dev-playbook](https://github.com/geerlingguy/mac-dev-playbook) 是最火的例子。幂等性好，重复跑不炸。但用企业级 YAML 给自己电脑装 VS Code，杀鸡用牛刀了。

---

## 方案五：全自动化

在上面几种方案之间反复横跳之后，我自己写了一个 — [OpenBoot](https://openboot.dev)。利益相关：这是我做的，所以下面的内容你自己判断。

```bash
brew install openbootdotdev/tap/openboot
openboot
```

跑起来是一个终端 UI，80 多个工具按分类排列。选一个预设或者自己勾选，回车，等 15 分钟。

- Homebrew 四路并行安装（带重试，不会因为一个包失败就全挂）
- Shell 自动配好 Oh-My-Zsh + 插件
- macOS 偏好自动改（23 项开发者常用设置）
- Git 身份自动设
- Dotfiles 自动 clone 和 link

`developer` 预设：29 个 CLI 工具 + 14 个 GUI 应用 + 5 个 npm 包。

核心功能是 **snapshot**：

```bash
openboot snapshot
```

扫描你当前系统的所有配置，打包成文件，上传后生成链接。别人一条命令复制你的整个环境：

```bash
openboot install yourname/my-setup
```

说实话，它不是万能的 — 没有回滚、不支持 Linux、dotfile 管理比 chezmoi 弱。但覆盖了大部分人 85% 的需求，不用学任何新语言。

---

## 五种方案放一起看

| | Brewfile | Shell 脚本 | chezmoi | nix-darwin | OpenBoot |
|---|---------|-----------|---------|-----------|---------|
| **装包** | ✅ | ✅ | ❌ | ✅ | ✅ |
| **GUI 应用** | ✅ | ✅ | ❌ | ✅ | ✅ |
| **Shell 配置** | ❌ | 手动 | ✅ | ✅ | ✅ |
| **macOS 偏好** | ❌ | 手动 | ❌ | ✅ | ✅ |
| **Git 身份** | ❌ | 手动 | ❌ | ❌ | ✅ |
| **Dotfiles** | ❌ | 手动 | ✅ | ✅ | 基础 |
| **回滚** | ❌ | ❌ | ✅ | ✅ | ❌ |
| **离线可用** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **学习成本** | 5 分钟 | 10 分钟 | 1 小时 | 1-2 周 | 5 分钟 |
| **覆盖率** | ~30% | ~70% | ~20% | ~90% | ~85% |

怎么选看你愿意花多少时间。Brewfile 五分钟上手但只管装包；chezmoi 管 dotfiles 很强但不管安装；nix-darwin 最全但学习成本最高。我做 OpenBoot 就是想在"全"和"简单"之间找个平衡，不过它也有短板 — 没回滚，不支持 Linux。

---

## 国内开发者必做的一步：换镜像源

这是国内配环境最大的坑，英文文章里不会讲。Homebrew 默认从 GitHub 拉取，裸连速度你懂的。**装任何东西之前先把源换了**：

```bash
# Homebrew 换清华源
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git"
export HOMEBREW_API_DOMAIN="https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles/api"
export HOMEBREW_BOTTLE_DOMAIN="https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles"

# npm 换淘宝源
npm config set registry https://registry.npmmirror.com

# pip 换清华源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

不换源的话，一个 `brew install` 能卡你半天。建议把这几行加到 `.zshrc` 里，省得每次都敲。

---

## 2026 年必装工具

### CLI 工具

用了就回不去的那种：

**ripgrep** 和 **fd** — 替代 grep 和 find，快到离谱。我已经忘了 `find` 原生语法了，`fd` 不用记。**fzf** — 任何列表都能变成可搜索的菜单，`Ctrl+R` 终于能用了。**zoxide** — 记住你常去的目录，`z blog` 直达，不用 `cd ../../../`。

**bat** 是带语法高亮的 cat，**eza** 是带 git 状态的 ls，**jq** 处理 JSON 必备，**delta** 让 git diff 变得能看。

**lazygit** 单独说一下 — 我一直坚持用命令行做 git，直到有次要 rebase 30 个 commit，投降了。复杂 git 操作它确实比 CLI 好使。

**gh** 收尾 — GitHub PR 管理现在基本都在终端里做了。

| 工具 | 替代 |
|------|------|
| ripgrep | grep |
| fd | find |
| bat | cat |
| eza | ls |
| zoxide | cd |
| fzf | Ctrl+R |
| lazygit | 复杂 git 操作 |
| delta | git diff |
| jq | — |
| gh | 浏览器里看 PR |

### GUI 应用

**VS Code** 还是主力。**Cursor** 在写新代码时越来越好用，AI 集成比插件强。

终端我在用 **Warp**。**Raycast** 装完就把 Spotlight 忘了，剪贴板历史和扩展太好用了。**Rectangle** 管窗口，免费够用。

**OrbStack** 是我安利最多的 — 从 Docker Desktop 切过来之后风扇安静了。如果你还在用 Docker Desktop，你在受不必要的苦。

**Arc** 浏览器看人喜好，我喜欢垂直标签页和工作空间。

### 2026 年新面孔

- **Ghostty** — Mitchell Hashimoto 做的终端，原生渲染，快得离谱。我在 Warp 和 Ghostty 之间反复横跳。
- **Zed** — 开大文件的时候不用等 VS Code 索引半天。
- **uv** — Python 终于有了不让人崩溃的包管理器。
- **Bun** — 写脚本替代 Node，启动速度差距大到尴尬。
- **Ollama** — 本地跑大模型，不用把代码发给 API。

---

## 团队怎么用

每个公司都有那么一份飞书文档，叫"开发环境配置指南"，47 步，上次更新是 2023 年。第 12 步让你装 Node 18，但项目需要 Node 22。第 23 步说"找张三要 .env 模板"，但张三早离职了。

用 OpenBoot 可以把这份文档变成一行命令：

```markdown
## 开发环境

    openboot install acme/frontend

不放心的话先预览：`openboot install acme/frontend --dry-run`
```

新人跑完去倒杯咖啡，回来就能提代码了。

---

## 常见问题

**最快的配环境方法？** Homebrew + OpenBoot，十五到二十分钟。

**Brewfile 够不够？** 只装包够了，但那只是 30% 的工作。Shell、macOS 偏好、Git 啥都不管。

**nix-darwin 值得吗？** 在 Linux 上已经用 Nix 的人，值得。其他人准备花 1-2 周上手。

**国内网络慢怎么办？** 换镜像源，上面有完整配置。不换的话什么方案都慢。

**团队能用吗？** `openboot snapshot` 导出配置，新人一条命令复制。比飞书文档靠谱。

---

不管你选哪个，别再手动配了。周末应该拿来写代码，不是一个一个敲 `brew install`。

英文版写得更详细，所有预设和命令都列全了，在[这里](/2026/mac-setup-automation-guide-2026/)。

---

**相关资源：**

- [OpenBoot](https://openboot.dev) — 全自动 Mac 环境配置
- [OpenBoot GitHub](https://github.com/openbootdotdev/openboot) — 开源，MIT 协议
- [Homebrew 清华镜像使用说明](https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/) — 国内必备
- [chezmoi](https://chezmoi.io) — Dotfile 管理
- [nix-darwin](https://github.com/LnL7/nix-darwin) — 声明式 macOS 配置
- [awesome-mac](https://github.com/jaywcjlove/awesome-mac) — Mac 应用精选列表
