+++
title = "My Dotfiles Project: Automating Modern Dev Environment Setup"
date = "2025-10-15"
description = "Building a dotfiles project with GNU Stow and Makefile for automated dev environment setup: modular design, one-command deployment, and symlink management with full code walkthrough."
tags = ["dotfiles", "automation", "macos", "stow", "makefile", "dev-environment"]

[extra.comments]
issue_id = 12

[[extra.faq]]
question = "What's the difference between a dotfiles project and just copying config files?"
answer = "Copying scatters configs across your home directory, making it hard to sync changes back to a repo. A dotfiles project uses GNU Stow to create symlinks — your config files live in a git repo, and any edit is automatically version-controlled. You can roll back, diff, and sync to other machines instantly."

[[extra.faq]]
question = "What is GNU Stow and why use it for dotfiles?"
answer = "GNU Stow is a symlink manager originally designed for managing software installations in /usr/local. For dotfiles, it maps a directory tree in your repo to corresponding locations in your home directory, automatically creating symlinks. No manual ln -s commands, no mapping files — the directory structure IS the configuration."

[[extra.faq]]
question = "How long does it take to set up a new machine with this project?"
answer = "About 10-15 minutes from git clone to a fully configured environment. Most of that time is Homebrew installing packages. The symlink deployment itself takes only a few seconds."

[[extra.faq]]
question = "Can I use this alongside chezmoi or yadm?"
answer = "You can, but there's no real benefit. chezmoi and yadm are also dotfile managers that handle symlinks or templates — they overlap with GNU Stow. Pick one approach and stick with it. If you already have a Stow-based setup, switching to chezmoi won't give you meaningful gains."

[[extra.faq]]
question = "How do I handle sensitive files like SSH private keys in a dotfiles repo?"
answer = "Never commit private keys in plaintext. Use git-crypt to encrypt sensitive files, or use .gitignore to exclude private keys and only commit SSH config. This project uses git-crypt for files that need encryption."
+++

The worst part of setting up a new machine isn't installing software — Homebrew handles that in one command. The painful part is all the config files scattered across your system. `.gitconfig` in your home directory, Neovim config in `.config/nvim/`, SSH config in `.ssh/`, Zsh split across multiple files. Manually moving everything takes half a day, and you will forget something.

This post walks through how I use GNU Stow + Makefile to manage all my configs, so a new machine gets everything in one command.

<!--more-->

---

## The Core Idea

The whole project boils down to three steps:

{% mermaid() %}
graph LR
    A["git clone<br/>Pull the repo"] --> B["make setup"]
    B --> C["install<br/>Install software"]
    B --> D["deploy<br/>Deploy configs"]
    C --> C1["Homebrew"]
    C --> C2["Brewfile packages"]
    D --> D1["Git config"]
    D --> D2["Zsh config"]
    D --> D3["Neovim config"]
    D --> D4["SSH config"]
    D --> D5["NVM config"]

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

**Clone the repo, install dependencies, symlink configs into place.** That's it.

```bash
git clone https://github.com/fullstackjam/dotfiles.git
cd dotfiles
make setup
```

About 10-15 minutes later, the new machine's dev environment matches the old one exactly.

---

## Project Structure

The project is split into independent modules, one directory per config group:

```
dotfiles/
├── Makefile              # Entry point — all install and deploy commands
├── Brewfile              # Homebrew package manifest
├── scripts/              # Install scripts (Homebrew, Brewfile, etc.)
│   ├── 01-homebrew.sh
│   └── 02-brewfile.sh
├── git/                  # Git config → symlinked to ~/.gitconfig
├── zsh/                  # Zsh config → symlinked to ~/.zshrc etc.
├── nvim/.config/nvim/    # Neovim config → symlinked to ~/.config/nvim/
├── nvm/                  # NVM config → symlinked to ~/.nvmrc etc.
├── ssh/.ssh/             # SSH config → symlinked to ~/.ssh/config
├── .gitignore
└── README.md
```

The key insight: **each directory's internal structure mirrors its target path in the home directory**. For example, `nvim/.config/nvim/init.vim` gets symlinked to `~/.config/nvim/init.vim`. No mapping rules needed — the directory structure IS the mapping.

Why this design works:

- **Modules are independent.** Want to update just your Git config? `make stow-git`. Nothing else is touched.
- **Failures are contained.** If one module's symlinks break, everything else keeps working.
- **Debugging is obvious.** Directory names tell you exactly what each config group does.

---

## GNU Stow: Symlinks Done Right

Most people manage dotfiles by either copying files around or writing a pile of `ln -s` commands. Copying breaks version control sync. Manual `ln -s` becomes unmaintainable past a handful of files.

GNU Stow solves this properly. Its logic is simple: take a directory and "project" it onto a target directory, creating matching symlinks automatically.

```bash
# Symlink everything in git/ to $HOME
stow --target="$HOME" --restow git
```

After running this, `git/.gitconfig` becomes a symlink at `~/.gitconfig`. The `--restow` flag means "remove old symlinks, then recreate" — it's idempotent, so running it multiple times is safe.

To deploy all modules at once:

```bash
for dir in */; do
    if [ -d "$dir" ] && [ "$dir" != "scripts/" ]; then
        stow --target="$HOME" --restow "$dir"
    fi
done
```

We skip `scripts/` because it contains install scripts, not config files — nothing there belongs in the home directory.

Why not just use `ln -s`? Because Stow handles all the tedious parts: automatic target path resolution, conflict detection, and cleanup of stale symlinks. Manual `ln -s` is fine for 5 files. For 30+, it's a nightmare.

---

## Makefile: Dependency Management

A Makefile is a natural fit for orchestrating the setup — it has built-in dependency management. Homebrew must be installed before `brew install` can run. Makefile expresses this cleanly:

```makefile
# Full setup: install software, then deploy configs
setup: install deploy

# Install phase: Homebrew first, then packages
install: homebrew brewfile

homebrew:
	@scripts/01-homebrew.sh

brewfile: homebrew
	@scripts/02-brewfile.sh

# Deploy phase: symlink all config files to home directory
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

`make setup` runs the entire flow. You can also target individual steps:

```bash
make setup       # Full setup (software + configs)
make install     # Install software only
make deploy      # Deploy config files only

# Target individual modules
make homebrew    # Install Homebrew
make brewfile    # Install packages from Brewfile
make stow-git    # Deploy Git config only
make stow-nvim   # Deploy Neovim config only

make help        # List all available commands
```

Makefile beats a shell script here because dependencies are declared in the rules — Make figures out execution order and skips completed steps automatically. No need to write `if` checks yourself.

---

## What's Included

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

Common aliases, color output, merge tool. None of this is critical, but it saves a lot of keystrokes every day.

### SSH

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes
    UseKeychain yes
```

Optimized for GitHub: auto-loads keys and uses the macOS Keychain so you never type your passphrase on every push.

### Zsh

Built on Oh-My-Zsh with:

- Git shortcut aliases (`gst`, `gco`, `gp`, etc.)
- Syntax highlighting and autosuggestions plugins
- Custom prompt theme

### Neovim

A minimal but functional config: line numbers, sensible indentation (4 spaces), incremental search, cursorline highlight. Not trying to be an IDE — just enough for editing config files and quick changes.

### Brewfile

50+ packages including CLI tools (ripgrep, fd, bat, fzf, lazygit) and GUI apps (VS Code, Warp, etc.). Everything in one `Brewfile`, installed with `brew bundle` in a single pass.

---

## Install Script Details

The scripts do two things to improve the experience: **colored status output** and **error handling**.

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

When you're installing 50+ packages, you need to know where you are. Color-coded output makes it obvious at a glance whether things are fine, concerning, or broken.

There's also a status check command for post-install verification:

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

If something shows as missing, redeploy that specific module: `make stow-git`, for example.

Made a mess? `make clean` removes all symlinks and resets to a clean state.

---

## Making It Your Own

If you want to fork this project and customize it, here's what to change:

1. **Git identity**: Edit `git/.gitconfig` — update the name and email
2. **Packages**: Edit `Brewfile` — remove what you don't need, add what you do
3. **SSH**: Edit `ssh/.ssh/config` — update key paths
4. **Zsh**: Edit `zsh/.zshrc` — adjust aliases and plugins
5. **New modules**: Create a new directory (e.g., `tmux/`), lay out config files matching the Stow structure, then add a `stow-tmux` rule to the Makefile

Stow's design makes adding new modules dead simple — create directory, add files, add rule. Three steps. No global configuration changes needed.

---

## Wrapping Up

The core value of this project fits in one sentence: **gather config files scattered across your system into a single git repo, and keep them in sync with symlinks.**

The stack:

- **GNU Stow** handles symlink creation and management — no manual `ln -s`
- **Makefile** handles install order and dependencies — one command for the full flow
- **Modular design** keeps each config group independent and individually operable

When you switch machines: clone, make setup, 10 minutes, done. Everything's in git, so you can always roll back.

If you're still manually copying config files between machines, seriously consider building your own dotfiles project. A couple hours of upfront investment saves half a day every time you set up or reinstall.

Project: [github.com/fullstackjam/dotfiles](https://github.com/fullstackjam/dotfiles)
