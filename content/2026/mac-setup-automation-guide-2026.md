+++
title = "I've set up 20+ Macs. Here's every way to automate it (so you don't waste a weekend)."
date = 2026-02-14
description = "A developer's honest guide to Mac setup automation — from Brewfiles to full environment snapshots. Every approach I've tried, what worked, what didn't."
tags = ["mac", "setup", "automation", "developer-tools", "productivity"]

[extra.comments]
issue_id = 13

[[extra.faq]]
question = "What's the fastest way to set up a new Mac for development?"
answer = "Use Homebrew for package management plus a setup tool like OpenBoot for everything else. A typical developer setup takes 15-30 minutes with automation vs. 2-4 hours manually. OpenBoot's developer preset installs 29 CLI tools, 14 GUI apps, and 5 npm packages in one command."

[[extra.faq]]
question = "Should I use a Brewfile or a full automation tool?"
answer = "A Brewfile only handles package installation — about 30% of a typical setup. It won't configure your shell, set macOS preferences, set up your git identity, or manage dotfiles. If all you need is packages, a Brewfile is fine. If you want the full setup automated, use OpenBoot, nix-darwin, or a comprehensive shell script."

[[extra.faq]]
question = "What's the difference between chezmoi and a Brewfile?"
answer = "They solve different problems. A Brewfile installs packages. chezmoi manages dotfiles (configuration files like .zshrc, .gitconfig). Neither one covers the full setup on its own. You'd typically need both, plus something for macOS preferences and shell configuration."

[[extra.faq]]
question = "Is nix-darwin worth the learning curve?"
answer = "nix-darwin gives you the most reproducible setup possible — the same config produces the exact same system every time, with built-in rollback. But the learning curve is steep. You need to learn the Nix language, which takes 1-2 weeks. If you're already using Nix on Linux, it's a natural choice. If you're not, consider whether that investment is worth it for your use case."

[[extra.faq]]
question = "Can I share my Mac setup with my team?"
answer = "Yes. OpenBoot lets you snapshot your current environment and share it as a URL. Anyone can replicate your setup with one command. This replaces the traditional Developer Environment Setup Google Doc that's always out of date."

[[extra.faq]]
question = "What CLI tools do most developers install on a new Mac in 2026?"
answer = "The most common CLI tools are: ripgrep (fast grep), fd (fast find), bat (cat with syntax highlighting), fzf (fuzzy finder), eza (modern ls), zoxide (smarter cd), lazygit (git TUI), gh (GitHub CLI), jq (JSON processor), and delta (better git diff). These replace slower built-in Unix tools with faster, more developer-friendly alternatives."

[[extra.faq]]
question = "What macOS preferences should I change for development?"
answer = "The most common defaults write changes are: show file extensions in Finder, show path bar in Finder, speed up Dock auto-hide delay, set fast keyboard repeat rate, save screenshots as PNG, and disable press-and-hold for keys. OpenBoot applies 23 whitelisted developer-friendly preferences automatically."

[[extra.faq]]
question = "Is it safe to run curl | bash install scripts?"
answer = "OpenBoot's install script is open source — you can read it before running. The --dry-run flag lets you preview every change before anything is installed. No SSH keys, API tokens, or .env files are ever captured or transmitted."
+++

**Quick answer:** Most developers just need Homebrew + a setup tool like [OpenBoot](https://openboot.dev) — about 15 minutes total. Dotfile nerds should look at chezmoi, and the truly committed can go with nix-darwin for full reproducibility. This guide covers all five approaches with real commands you can run today.

---

I've been a developer for years and I've set up more Macs than I care to admit — personal machines, work laptops, loaners, that one time I rage-wiped my drive at 2am because I broke my Python environment beyond repair. 

You know the feeling. You unbox a new Mac, it smells like premium aluminum and potential, and you're excited for exactly twelve minutes. Then you open Terminal. 

It's a desert. No Git. No Node. No Docker. No VS Code. Your shell is a bare `zsh` prompt that doesn't even know your name. Finder is hiding file extensions like they're state secrets. The Dock has that 45-second animation that makes you want to throw the machine out the window. Your clipboard history? Gone. Your aliases? Vanished.

So you start the ritual. You open the Homebrew homepage. You copy-paste the install command. Then you start typing. Two hours later, you've installed maybe half of what you actually use. A week from now, you'll be in the middle of a high-priority bug fix and realize you forgot `jq`. Two weeks from now, you'll realize you never set up your git email and every commit you've made for the new job says "unknown."

I've tried every level of automation to fix this. I've used the "I'll remember it" method (spoiler: I didn't), I've used massive shell scripts that broke halfway through, and I've spent weekends lost in the rabbit hole of Nix. 

---

## First: the problem is bigger than you think

Before I show you the tools, I want to talk about scope. Most developers undercount their setup. I counted once. My dev environment has 83 individual things across 8 categories. If you think your setup is "just Homebrew and VS Code," you're lying to yourself.

### 1. Package manager

macOS still ships without a package manager. Step zero is always Homebrew. I've done this so many times I can type the URL from memory, which is a sad state of affairs.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

This also grabs the Xcode Command Line Tools. Without this, nothing else works.

### 2. CLI tools (20-50 packages)

These are the tools that live in your muscle memory. You don't realize how much you rely on them until you type `rg` and get `command not found`.

| Category | Tools |
|----------|-------|
| **Search & navigation** | ripgrep, fd, fzf, zoxide, tree |
| **File viewing** | bat, eza, jq, yq |
| **System monitoring** | htop, btop |
| **Git** | gh, lazygit, git-delta, git-lfs |
| **Networking** | curl, wget, httpie |
| **Documentation** | tealdeer (tldr) |

Most of us have at least 30 of these. The nightmare isn't installing them; it's remembering that you need `tealdeer` until the moment you're trying to remember the flags for `tar`.

### 3. GUI applications (10-25 apps)

The stuff that lives in `/Applications`. This is where you spend your day.

| Category | Apps |
|----------|------|
| **Editors** | VS Code, Cursor, Zed, WebStorm |
| **Terminals** | Warp, iTerm2, Ghostty, Kitty |
| **Browsers** | Chrome, Arc, Firefox |
| **Productivity** | Raycast, Rectangle, Maccy, Stats |
| **Dev tools** | OrbStack, Postman, TablePlus |
| **Communication** | Slack, Discord, Notion |

### 4. Programming languages & runtimes

The heavy hitters. Node, Go, Python, Rust, Deno, Bun. Plus the package managers like `pnpm`, `uv`, and `cargo`. Setting these up manually is a recipe for version mismatch hell.

### 5. Shell configuration

This is your home. Oh-My-Zsh, Starship, plugins for syntax highlighting and auto-suggestions. Your `.zshrc` is basically your developer fingerprint.

### 6. Dotfiles

The configuration files: `.gitconfig`, `.zshrc`, `.vimrc`, `.ssh/config`. These make your tools actually behave. Without them, your terminal feels like someone else's house.

### 7. Git identity

The two lines everyone forgets. Every time.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### 8. macOS preferences

The "fixes" for macOS's annoying defaults. I have a list of `defaults write` commands that I run immediately because I can't stand a slow Dock or hidden path bars.

```bash
# Show file extensions in Finder
defaults write NSGlobalDomain AppleShowAllExtensions -bool true

# Show path bar in Finder
defaults write com.apple.finder ShowPathbar -bool true

# Speed up Dock auto-hide delay
defaults write com.apple.dock autohide-delay -float 0

# Fast keyboard repeat
defaults write NSGlobalDomain KeyRepeat -int 2
defaults write NSGlobalDomain InitialKeyRepeat -int 15

# Save screenshots as PNG
defaults write com.apple.screencapture type -string "png"

# Disable press-and-hold for keys (enable key repeat)
defaults write NSGlobalDomain ApplePressAndHoldEnabled -bool false
```

### Total count

When you add it all up, a "standard" setup is easily **60-100 items**. Doing that manually takes a full afternoon, and you'll still be finding missing pieces for the next month.

---

## Level 1: The Brewfile

I used a Brewfile for about a year. It's the simplest automation because Homebrew has it built in. 

### Create a Brewfile from your current Mac

```bash
brew bundle dump --file=~/Brewfile
```

This spits out a list of everything you have:

```ruby
tap "homebrew/bundle"
brew "ripgrep"
brew "fd"
brew "bat"
brew "fzf"
brew "node"
brew "go"
brew "lazygit"
cask "visual-studio-code"
cask "warp"
cask "raycast"
cask "google-chrome"
```

### Restore on a new Mac

```bash
brew bundle --file=~/Brewfile
```

I liked the Brewfile because it has zero dependencies and it's easy to read. You can just throw it in a GitHub repo and you're done. But I eventually moved away from it.

The problem is that it only handles packages. It doesn't touch your shell config, your macOS preferences, or your git identity. You're only automating about 30% of the job. Also, maintaining it is a bit of a chore — you have to manually edit the file every time you want to add something new, and if you have 80 lines, it's a mess. There's no way to categorize things or remember why you installed that random utility three months ago.

---

## Level 2: The shell script

This is the next logical step. You get tired of the Brewfile's limitations and write a bash script to do everything. I had one of these for a while.

```bash
#!/bin/bash
set -euo pipefail

echo "Installing Homebrew..."
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

echo "Installing packages..."
brew install ripgrep fd bat fzf node go docker lazygit gh

echo "Installing apps..."
brew install --cask visual-studio-code warp raycast google-chrome

echo "Setting up Oh-My-Zsh..."
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

echo "Configuring macOS..."
defaults write NSGlobalDomain AppleShowAllExtensions -bool true
defaults write com.apple.dock autohide-delay -float 0

echo "Configuring git..."
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

echo "Done!"
```

It covers more ground, but here's what went wrong when I actually used it: one time I ran mine and a single `brew install` failed because of a network hiccup, and the whole thing just stopped. Or worse, it kept going and I had no idea what actually finished. 

Shell scripts are also fragile because they aren't idempotent. If you run a script twice, it usually yells at you. The Oh-My-Zsh installer will complain, the git config will overwrite everything, and you'll get a bunch of "already installed" warnings from Homebrew. And let's be honest: sitting there staring at 80 packages installing with zero progress bar is just anxiety-inducing. 

When I shared a version of this on Reddit, someone replied with their 200-line shell script that did something similar. Which proved my point — if you need 200 lines of bash to set up a Mac, maybe bash isn't the right tool for the job.

---

## Level 3: Dotfile managers

If you're serious about your configuration, you eventually look into dotfile managers. These are built specifically to sync your configuration files across machines.

### chezmoi

I used chezmoi for about two months. It can template your dotfiles so the same repo works on your work laptop and personal machine, and it even encrypts secrets.

```bash
chezmoi init --apply your-github-username
```

Here's the thing though: I spent more time learning chezmoi's directory structure and commands than I spent actually setting up my dotfiles. If you manage configs across five machines with different OSes, it's worth it. If you just want your `.zshrc` on two Macs, it's a lot of ceremony for a symlink.

### dotbot

This is a lightweight alternative. It uses YAML and lives as a git submodule in your dotfiles repo. It's simple and does one thing well, but it's very limited. 

```yaml
- link:
    ~/.zshrc: zshrc
    ~/.gitconfig: gitconfig
    ~/.vimrc: vimrc
- shell:
    - [git submodule update --init --recursive, Installing submodules]
```

It handles symlinks and shell commands, but it won't install your apps or touch your macOS preferences. You're still left doing the hard work yourself.

### mackup

Mackup is great for application settings. It backs everything up to iCloud or Dropbox.

```bash
mackup backup    # save settings
mackup restore   # restore on new machine
```

It's the only tool that reliably handles settings for things like VS Code and iTerm2 that aren't stored in simple dotfiles. But again, it doesn't install the actual apps. You still need Homebrew for that.

### The gap

Dotfile managers handle configuration files. They don't handle installation. You still need a Brewfile or a script to actually get the tools on your machine. At one point I had a Brewfile, a dotbot config, AND a shell script for macOS preferences. Three systems to maintain for one laptop. That's when I started thinking about whether someone had solved this already.

---

## Level 4: Infrastructure-as-code

This is for the engineering purists who want their Mac to be as reproducible as a production server.

### nix-darwin

I tried nix-darwin for a weekend. I thought I'd emerge with a perfectly reproducible system. Instead I emerged with a mass of error messages and a newfound respect for anyone who actually uses Nix daily.

```nix
{ pkgs, ... }: {
  environment.systemPackages = with pkgs; [
    ripgrep fd bat fzf nodejs go
  ];
  homebrew.casks = [ "visual-studio-code" "warp" ];
  system.defaults.dock.autohide = true;
}
```

Look at that config. It's elegant. The same file produces the exact same system on any Mac, and you can roll back if something breaks. The problem is getting there. Nix has its own language, its own package manager, its own way of thinking about software. I spent most of my weekend on the Nix discourse forums trying to figure out why my shell wasn't loading. If you're already in the Nix ecosystem on Linux, this is the natural choice. If not — prepare to invest a week or two before it pays off.

### Ansible (geerlingguy/mac-dev-playbook)

Ansible is the industry standard for server automation, and people have adapted it for Mac. Jeff Geerling's [mac-dev-playbook](https://github.com/geerlingguy/mac-dev-playbook) is probably the most popular example.

```yaml
- name: Install Homebrew packages
  community.general.homebrew:
    name: "{{ item }}"
  loop:
    - ripgrep
    - fd
    - bat
```

The idempotency is genuinely nice — you can re-run it without breaking things. But there's something absurd about writing enterprise YAML to install VS Code on your personal laptop. And when a playbook fails (it will), the error messages read like server logs, not helpful diagnostics.

### The tradeoff

These tools are rigorous, but they're a sledgehammer for a nail. You'll spend hours configuring things before you even get to your first `npm install`. If you're managing 500 laptops for a corporate IT department, use Ansible. If you're one person, it's probably too much.

---

## Level 5: Full automation

I spent years bouncing between these levels. I wanted something that handled all 8 categories — packages, apps, shell, dotfiles, macOS preferences, git, language runtimes — but without the configuration nightmare of Nix or Ansible.

I'm biased — I built [OpenBoot](https://openboot.dev) because I was tired of wasting my Sundays on this. I wanted a tool that felt like a modern CLI app, not a server config.

### Install

```bash
brew install openbootdotdev/tap/openboot
openboot
```

If you're on a brand new machine without Homebrew, you can just run the install script:

```bash
curl -fsSL openboot.dev/install.sh | bash
```

### What actually happens

When you run it, a terminal UI opens up. It's not a text file; it's an interactive menu with 80+ tools organized by category. 

1. You pick a preset (`minimal`, `developer`, or `full`) or toggle things one by one.
2. You hit Enter.
3. Homebrew starts installing your packages 4x in parallel (with retry logic so it doesn't just die).
4. Your shell gets configured with Oh-My-Zsh and the best plugins.
5. Your macOS preferences get fixed.
6. Your git identity gets set up.
7. Your dotfiles get cloned and linked.

The `developer` preset covers what 90% of us need. It installs 29 CLI tools, 14 GUI apps, and 5 npm packages. On a decent connection, it takes about 15 minutes. 

### Why this beats a shell script

The big difference is the interactivity. You see every tool before it's installed. You can toggle Notion off if you don't use it, or add `kubectl` if you're a DevOps person. No YAML, no editing scripts. 

It also has smart detection. If you already have Node installed, it just skips it. You can run it once a month to make sure your system hasn't drifted, and it won't break anything. 

### Three presets

| Preset | CLI tools | GUI apps | NPM packages | Best for |
|--------|-----------|----------|--------------|----------|
| **minimal** | 19 | 5 | 0 | Minimalists, servers |
| **developer** | 29 | 14 | 5 | Most developers |
| **full** | 48 | 25 | 9 | Polyglot / DevOps |

### Snapshot: capture your current Mac

If you've already spent years perfecting your setup, you don't want to start from scratch. This is the feature that got the most attention when I shared this on Reddit.

```bash
openboot snapshot
```

It scans your system — Homebrew packages, cask apps, npm globals, macOS preferences, shell config, git identity — and bundles it into a config file. You can upload it to [openboot.dev](https://openboot.dev) and get a shareable URL.

What gets captured:

| Category | How |
|----------|-----|
| Homebrew formulae | `brew leaves` (only the stuff you actually installed) |
| Homebrew casks | `brew list --cask` |
| NPM global packages | `npm list -g` |
| macOS preferences | 23 whitelisted developer settings |
| Shell config | Oh-My-Zsh theme, plugins, aliases |
| Git config | name, email, editor, default branch |
| Dev tool versions | Go, Node, Python, Rust, Docker |

I was careful about security: no SSH keys, no API tokens, no `.env` files. Only the public-facing stuff.

### Share your setup

Every config gets a URL like `openboot.dev/fullstackjam/my-setup`. Now, if someone asks "what tools do you use?", you just send them the link. They can replicate your exact environment with one command:

```bash
openboot install yourname/my-setup
```

Automation is better when it's social. You can browse what other developers are using at [openboot.dev/explore](https://openboot.dev/explore).

---

## For teams: replacing the onboarding doc

Every company has that one Google Doc. It's called "Developer Environment Setup," it's 47 steps long, and it was last updated in 2023 by a guy who moved to a different department. Step 12 tells you to install Node 18, but the project won't even build without Node 22. Step 23 says "ask Sarah for the .env template," but Sarah doesn't work here anymore.

I've seen whole weeks wasted on this. With OpenBoot, you can replace that entire doc with one line in your `CONTRIBUTING.md`:

```markdown
## Development Setup

    openboot install acme/frontend

Preview first: `openboot install acme/frontend --dry-run`
```

The new hire runs that, goes to grab coffee, and twenty minutes later they have the exact same environment as the rest of the team. They can actually push code before lunch on their first day.

---

## How they compare

| | Brewfile | Shell script | chezmoi | nix-darwin | OpenBoot |
|---|---------|-------------|---------|-----------|---------|
| **Packages** | Yes | Yes | No | Yes | Yes |
| **GUI apps** | Yes | Yes | No | Yes | Yes |
| **Shell config** | No | DIY | Yes | Yes | Yes |
| **macOS prefs** | No | DIY | No | Yes | Yes |
| **Git identity** | No | DIY | No | No | Yes |
| **Dotfiles** | No | DIY | Yes | Yes | Basic |
| **Interactive UI** | No | No | No | No | Yes |
| **Rollback** | No | No | Yes | Yes | No |
| **Works offline** | Yes | Yes | Yes | Yes | No |
| **Sharing** | File | File | Git repo | Git repo | URL |
| **Team support** | Manual | Manual | No | No | Built-in |
| **Learning curve** | 5 min | 10 min | 1 hour | 1-2 weeks | 5 min |
| **Covers** | ~30% | ~70% | ~20% | ~90% | ~85% |

Honestly, the right tool depends on how much you care. Brewfile if packages are all you need. chezmoi if your dotfiles are your identity. Nix if you want to go full declarative and don't mind the learning investment.

I built OpenBoot because I wanted the coverage of Nix without spending a week learning a new language. It doesn't do everything — no rollback, no Linux, dotfile support is basic compared to chezmoi — but it handles the part that matters for most setups without asking you to write YAML or learn Nix expressions.

---

## The tools most developers install in 2026

I spend a lot of time looking at community configs on [openboot.dev/explore](https://openboot.dev/explore) and reading threads on r/macapps and Hacker News. If you don't have these tools, you're working harder than you need to.

### CLI tools everyone installs

The ones I can't live without: **ripgrep** and **fd**. They replace `grep` and `find` with something that's actually fast. I can never remember `find`'s syntax anyway. **fzf** turns any list into a searchable menu — pipe your command history through it and suddenly Ctrl+R is usable. **zoxide** remembers where you go, so `z blog` takes me to this repo from anywhere.

For looking at things: **bat** (cat with syntax highlighting — you won't go back), **eza** (ls but with git status in the listing), **jq** (essential if you ever touch JSON), and **delta** (makes git diffs readable in the terminal).

**lazygit** deserves its own mention. I resisted it for a long time because I'm stubborn about doing git from the command line. Then I had to do an interactive rebase across 30 commits and I caved. It's genuinely better for complex git operations.

**gh** rounds it out — I do almost all my PR reviews from the terminal now.

| Tool | Replaces |
|------|----------|
| ripgrep | grep |
| fd | find |
| bat | cat |
| eza | ls |
| zoxide | cd |
| fzf | Ctrl+R |
| lazygit | git CLI (for complex stuff) |
| delta | git diff |
| jq | — |
| gh | browser-based PR reviews |

### GUI apps developers actually use

**VS Code** is still the default. I flirt with other editors but keep coming back for the extension ecosystem. **Cursor** is eating into that lead though — I've been using it more and more for anything that involves writing new code.

For terminals, I'm on **Warp** right now. **Raycast** replaced Alfred for me instantly — the clipboard history alone is worth it, and the extensions are wild. **Rectangle** does window management and it's free, which is all I need to say about that.

**OrbStack** is the one I evangelize the most. I switched from Docker Desktop and my fans went quiet for the first time in months. If you're running Docker on a Mac and you haven't tried it, you're suffering unnecessarily.

**Arc** is polarizing but I like the vertical tabs and workspaces.

| App | Category |
|-----|----------|
| VS Code / Cursor | Editor |
| Warp | Terminal |
| Raycast | Launcher |
| Rectangle | Window management |
| OrbStack | Docker & Linux VMs |
| Arc | Browser |

### The 2026 additions

A few newer tools that have earned a permanent spot:

- **Ghostty** — Mitchell Hashimoto's terminal emulator. Native rendering, stupid fast, and it just looks right. I keep going back to Warp for the AI features but Ghostty tempts me every time I open it.
- **Zed** — I use this when I need to open a massive file and don't want to wait for VS Code to index the universe.
- **Ollama** — Local LLMs. Mostly for plane rides and for when I don't want to send code to an API.
- **uv** — Python finally has a package manager that doesn't make me want to quit programming.
- **Bun** — Replaced Node for all my throwaway scripts. The startup time difference is embarrassing.

---

## Quick-start recipes

### The minimalist (19 CLI tools, 5 apps)

For the people who want to keep their machine as lean as possible.

```bash
brew install openbootdotdev/tap/openboot
openboot --preset minimal
```

This gives you the essentials: ripgrep, fd, bat, fzf, eza, zoxide, lazygit, gh, htop, btop, plus Warp, Raycast, Rectangle, Maccy, and Stats.

### The full-stack developer (29 CLI tools, 14 apps, 5 npm packages)

The sweet spot. This is what I use on my primary machine.

```bash
openboot --preset developer
```

Adds: Node, Go, Docker, pnpm, tmux, neovim, VS Code, Chrome, Arc, OrbStack, TablePlus, and global npm packages like TypeScript and Prettier.

### The everything preset (48 CLI tools, 25 apps, 9 npm packages)

For the polyglots and the DevOps engineers.

```bash
openboot --preset full
```

Adds: Python, Rust, Deno, Bun, kubectl, Helm, Terraform, AWS CLI, PostgreSQL, Redis, Ollama, Cursor, Figma, Firefox, and Obsidian.

### From a teammate's setup

```bash
openboot install teammate/their-setup
```

### CI / automation (no interaction)

If you're setting up a machine in a lab or a headless environment:

```bash
OPENBOOT_GIT_NAME="Your Name" \
OPENBOOT_GIT_EMAIL="you@example.com" \
openboot --preset developer --silent
```

### Preview without installing anything

If you're nervous about what a tool might do to your system:

```bash
openboot --preset developer --dry-run
```

---

## After setup: maintenance

Setup isn't a one-time thing. You're constantly adding and removing tools.

### Keep packages updated

```bash
openboot update            # update everything in one go
openboot update --dry-run  # see what's about to change
```

### Clean up drift

We all install things we only use once.

```bash
openboot clean              # compare against your original snapshot
openboot clean --dry-run    # see what you can safely remove
```

### Check system health

```bash
openboot doctor
```

---

Whatever you pick — don't do it manually again. Life's too short to type `brew install ripgrep` from memory for the 21st time. 

If you want to check out what I've been working on, head over to [OpenBoot](https://openboot.dev). It's open source, it's fast, and it might just save you a weekend.

---

## Quick answers to things people ask me

**Fastest way to set up a new Mac?** Homebrew + OpenBoot. Fifteen to twenty minutes on a decent connection.

**Brewfile or full automation?** Brewfile if you only need packages installed. That's like 30% of a real setup though — it won't touch your shell, your git config, or your macOS preferences.

**chezmoi vs Brewfile?** Different jobs. Brewfile installs software. chezmoi syncs config files. You'd need both plus something else to cover everything.

**Is nix-darwin worth it?** If you're already in the Nix ecosystem, absolutely. If not, you're looking at 1-2 weeks before it starts paying off. Most people bounce off it.

**Can I share my setup with my team?** That's the whole point of `openboot snapshot`. One URL, one command. Beats the Google Doc that nobody maintains.

**Is `curl | bash` safe?** Read the script first — it's [on GitHub](https://github.com/openbootdotdev/openboot). Or just use `--dry-run` to see what it would do. No SSH keys, tokens, or .env files are ever touched.

---

**Resources:**

- [OpenBoot](https://openboot.dev) — Full setup automation with TUI and sharing
- [OpenBoot on GitHub](https://github.com/openbootdotdev/openboot) — Open source, MIT licensed
- [Explore community configs](https://openboot.dev/explore) — See what other developers install
- [Homebrew Bundle docs](https://github.com/Homebrew/homebrew-bundle) — Brewfile reference
- [chezmoi](https://chezmoi.io) — Dotfile management
- [nix-darwin](https://github.com/LnL7/nix-darwin) — Declarative macOS config
- [awesome-mac](https://github.com/jaywcjlove/awesome-mac) — Curated Mac app list


