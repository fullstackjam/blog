+++
title = "My Dotfiles Project: Automating Modern Dev Environment Setup"
date = "2025-10-15"
updated = "2026-06-08"
description = "A pure config management approach with GNU Stow: modular design, one-command deployment, paired with OpenBoot for full new-machine automation."
tags = ["dotfiles", "automation", "macos", "stow", "dev-environment", "AI-tools"]

[extra.comments]
issue_id = 14

[[extra.faq]]
question = "What's the difference between a dotfiles project and just copying config files?"
answer = "Copying scatters configs across your home directory, making it hard to sync changes back to a repo. A dotfiles project uses GNU Stow to create symlinks — your config files live in a git repo, and any edit is automatically version-controlled. You can roll back, diff, and sync to other machines instantly."

[[extra.faq]]
question = "What is GNU Stow and why use it for dotfiles?"
answer = "GNU Stow is a symlink manager originally designed for managing software installations in /usr/local. For dotfiles, it maps a directory tree in your repo to corresponding locations in your home directory, automatically creating symlinks. No manual ln -s commands, no mapping files — the directory structure IS the configuration."

[[extra.faq]]
question = "Why not handle software installation in the dotfiles repo?"
answer = "Config files and software installation are separate concerns. Configs change frequently and need fine-grained management; software installation is one-time and declarative. Splitting them keeps the dotfiles repo pure, while a dedicated tool like OpenBoot handles the installation side."

[[extra.faq]]
question = "How long does it take to set up a new machine?"
answer = "With OpenBoot, one command does everything automatically: install Homebrew and packages, clone dotfiles, deploy symlinks, install Oh-My-Zsh and plugins. The whole process takes about 10-15 minutes, mostly spent on Homebrew installing packages."

[[extra.faq]]
question = "What does the --no-folding flag do?"
answer = "By default, Stow creates directory-level symlinks (folding) — linking an entire directory instead of individual files. With --no-folding, Stow only creates file-level symlinks, keeping target directories as real directories. This prevents issues when other programs need to write to the same directory, like Claude Code creating sessions/ under ~/.claude/."
+++

The worst part of setting up a new machine isn't installing software — Homebrew handles that in one command. The painful part is all the config files scattered across your system. `.gitconfig` in your home directory, SSH config in `.ssh/`, Zsh config, terminal config, AI coding tool configs... Manually moving everything takes half a day, and you will forget something.

This post walks through how I use GNU Stow to manage all my configs, paired with [OpenBoot](https://openboot.dev) for software installation, so a new machine gets everything in one command.

<!--more-->

---

## The Core Idea

The project does exactly one thing: **manage config file symlinks**. Software installation is handled by OpenBoot.

{% mermaid() %}
graph LR
    A["curl openboot.dev"] --> B["Install Homebrew<br/>+ packages"]
    A --> C["clone dotfiles<br/>to ~/.dotfiles"]
    C --> D["make install"]
    D --> D1["Git config"]
    D --> D2["SSH config"]
    D --> D3["Zsh config"]
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

Full automated setup in one line:

```bash
curl -fsSL openboot.dev/fullstackjam | bash
```

OpenBoot handles everything: install Homebrew and packages, clone dotfiles to `~/.dotfiles`, deploy configs via Stow, install Oh-My-Zsh and plugins.

To deploy configs only:

```bash
git clone https://github.com/fullstackjam/dotfiles.git ~/.dotfiles
cd ~/.dotfiles
make install
```

---

## Project Structure

The project is split into independent modules, one directory per config group:

```
dotfiles/
├── Makefile                              # Entrypoint — install / uninstall
├── git/.gitconfig                        # Git configuration
├── ssh/.ssh/config                       # SSH client config
├── zsh/.zshrc                            # Zsh configuration
├── claude/.claude/CLAUDE.md              # Claude Code global instructions
├── claude/.claude/settings.json          # Claude Code settings
├── claude/.claude/statusline-command.sh  # Claude Code statusline script
├── ghostty/.config/ghostty/config        # Ghostty terminal configuration
├── opencode/.config/opencode/opencode.json
├── opencode/.config/opencode/oh-my-openagent.json
├── opencode/.config/opencode/tui.json
└── .gitignore
```

The key insight: **each directory's internal structure mirrors its target path in the home directory**. For example, `ghostty/.config/ghostty/config` gets symlinked to `~/.config/ghostty/config`. The directory structure IS the mapping — no rules to configure.

Why this design works:

- **Modules are independent.** Add or remove configs without affecting anything else.
- **Failures are contained.** If one module's symlinks break, everything else keeps working.
- **Debugging is obvious.** Directory names tell you exactly what each config group does.

---

## GNU Stow: Symlinks Done Right

Most people manage dotfiles by either copying files around or writing a pile of `ln -s` commands. Copying breaks version control sync. Manual `ln -s` becomes unmaintainable past a handful of files.

GNU Stow solves this properly. Its logic is simple: take a directory and "project" it onto a target directory, creating matching symlinks automatically.

```bash
# Symlink everything in git/ to $HOME
stow --no-folding --target="$HOME" git
```

After running this:

```
~/.gitconfig                    → ~/.dotfiles/git/.gitconfig
~/.ssh/config                   → ~/.dotfiles/ssh/.ssh/config
~/.zshrc                        → ~/.dotfiles/zsh/.zshrc
```

Note the `--no-folding` flag instead of the default behavior. By default, Stow creates directory-level symlinks (folding) — it would symlink the entire `~/.claude` directory to the repo. But Claude Code creates runtime directories like `sessions/` under `~/.claude/`, and that breaks if the whole directory is a symlink. `--no-folding` forces Stow to only create file-level symlinks, keeping target directories as real directories.

Why not just use `ln -s`? Because Stow handles all the tedious parts: automatic target path resolution and conflict detection. Manual `ln -s` is fine for 5 files. For 30+, it's a nightmare.

---

## Makefile: As Simple As It Gets

Earlier versions had a complex Makefile with install, deploy, status, clean, and per-module targets. After splitting software installation into OpenBoot, the Makefile became minimal:

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

Two commands: `make install` to deploy, `make uninstall` to clean up.

`install` first creates necessary target directories with `mkdir -p`, then stows all packages in one go. The `-v` (verbose) flag shows each symlink being created.

Adding a new module is trivial — just add its name to the `PACKAGES` list.

---

## What's Included

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

Beyond the usual aliases, a few practical settings: `pull.rebase = true` defaults to rebase instead of merge; `rerere` remembers conflict resolutions so identical conflicts resolve automatically next time; `help.autocorrect` fixes mistyped commands.

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

Two key choices: **1Password SSH Agent** for key management — private keys never live on disk as files; GitHub over **port 443** (`ssh.github.com`) so pushes work even on networks that block port 22.

### Zsh

Built on Oh-My-Zsh with a focused plugin set:

```bash
plugins=(git helm kubectl fast-syntax-highlighting zsh-autocomplete)
```

Plus modern CLI tool integrations:

- **fzf**: fuzzy search (`source <(fzf --zsh)`)
- **zoxide**: smart cd (`eval "$(zoxide init zsh)"`)
- A convenience alias: `alias c="claude --dangerously-skip-permissions"`

### Claude Code

Added in 2025 — global config for the AI coding assistant. Three files:

- **CLAUDE.md**: global behavioral instructions defining coding style preferences (simplicity first, surgical changes, goal-driven execution, etc.)
- **settings.json**: model selection, permission mode, enabled plugins (superpowers, codex, etc.)
- **statusline-command.sh**: terminal statusline script in robbyrussell style showing directory, Git branch, model, and context usage

Managing Claude Code config in dotfiles means your AI assistant's behavior preferences come along to new machines automatically.

### Ghostty

Config for the [Ghostty](https://ghostty.org) terminal emulator:

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

Font, transparency, cursor style, macOS integration. Small details like copy-on-select and left Option as Alt that add up to a smoother terminal experience.

### OpenCode

Config for another AI coding tool, including plugin settings (oh-my-openagent, superpowers) and TUI interface configuration.

---

## Making It Your Own

If you want to fork this project and customize it, here's what to change:

1. **Git identity**: Edit `git/.gitconfig` — update name and email
2. **SSH**: Edit `ssh/.ssh/config` — adjust for your key management setup
3. **Zsh**: Edit `zsh/.zshrc` — tweak plugins and aliases
4. **AI tools**: Modify configs under `claude/` or `opencode/`, or remove modules you don't use
5. **New modules**: Create a directory (e.g., `tmux/`), lay out config files matching the Stow structure, add the name to `PACKAGES` in the Makefile, and add the target directory to the `mkdir -p` line if it doesn't exist

---

## Wrapping Up

This project evolved through several iterations and converged on a clean separation:

- **Dotfiles repo** handles only config files — GNU Stow for symlinks, Makefile as the entry point
- **Software installation** is delegated to [OpenBoot](https://openboot.dev) — Homebrew, packages, Oh-My-Zsh in one command

This keeps the dotfiles repo pure — just config files and a minimal Makefile. New machine setup: `curl -fsSL openboot.dev/fullstackjam | bash`, 10 minutes, done.

If you're still manually copying config files between machines, seriously consider building your own dotfiles project. A couple hours of upfront investment saves half a day every time you set up or reinstall.

Project: [github.com/fullstackjam/dotfiles](https://github.com/fullstackjam/dotfiles)
