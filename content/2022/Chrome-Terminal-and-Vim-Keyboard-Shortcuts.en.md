+++
title = "Chrome, Terminal, and Vim Keyboard Shortcuts: A Practical Guide"
date = 2022-04-24
description = "The keyboard shortcuts I actually use daily as a developer — covering Chrome browser, shell terminal, and Vim editor, organized by real-world use cases with practical tips."
tags = ["productivity", "shortcuts", "vim", "terminal", "chrome", "developer-tools"]

[extra.comments]
issue_id = 2

[[extra.faq]]
question = "What are the most important keyboard shortcuts for developers?"
answer = "In Chrome: Ctrl+T (new tab), Ctrl+W (close tab), Ctrl+Shift+T (restore closed tab). In terminal: Ctrl+A/E (jump to start/end of line), Ctrl+R (reverse search history). In Vim: ciw (change word), dd (delete line), u (undo). These cover about 80% of daily use."

[[extra.faq]]
question = "How do I get started with Vim without losing my mind?"
answer = "Start with three things: press i to enter insert mode, press Esc to go back to normal mode, type :wq to save and quit. Build from there slowly — add o for new line below, dd for delete line, u for undo. Learn 2-3 new commands per week, not everything at once."

[[extra.faq]]
question = "Why does shell use different shortcuts for cut and paste?"
answer = "Shell uses Emacs-style readline shortcuts. Ctrl+K cuts from cursor to end of line, Ctrl+U cuts from cursor to beginning, and Ctrl+Y pastes. These work in both Bash and Zsh and are much faster than arrow keys for editing long commands."

[[extra.faq]]
question = "Can I browse Chrome without using a mouse?"
answer = "Yes, install the Vimium C extension. Press F to see labels on every clickable element, type the letters to click. Use / to search the page, J/K to scroll, and H/L to go back/forward. You can browse the entire web keyboard-only."

[[extra.faq]]
question = "Do these shortcuts work on Mac?"
answer = "Chrome shortcuts use Cmd instead of Ctrl on Mac. Shell readline shortcuts (Ctrl+A/E/K/U) stay the same since they are readline bindings, not OS shortcuts. Vim shortcuts are identical across all platforms."
+++

Keyboard shortcuts are one of those things where you think "why bother memorizing this?" while learning them, and "how did I ever live without this?" once they're muscle memory.

This post covers the shortcuts I actually use every day across three tools: Chrome browser, shell terminal, and Vim editor. This isn't an exhaustive dump of every shortcut from the official docs — it's the ones I reach for in real work. Shortcuts with strikethrough are ones that exist but I rarely use in practice.

<!--more-->

---

## Chrome Browser

Chrome is where developers spend a huge chunk of their time. Reading docs, debugging in DevTools, reviewing PRs — it all happens here. Learning a few key shortcuts lets you fly between tabs without reaching for the mouse.

### Tabs and Windows

This is the highest-frequency group. Restoring accidentally closed tabs (Ctrl+Shift+T) is a lifesaver.

| Action | Shortcut |
| :--- | :--- |
| New window | **Ctrl + N** |
| New incognito window | **Ctrl + Shift + N** |
| New tab | **Ctrl + T** |
| Restore last closed tab | **Ctrl + Shift + T** |
| Switch to next tab | **Ctrl + Tab** |
| Jump to the rightmost tab | **Ctrl + 9** |
| Go back one page | **Alt + ←** |
| Go forward one page | **Alt + →** |
| Close current tab | **Ctrl + W** |

> **Pro tip:** Ctrl+1 through Ctrl+8 jump directly to the tab at that position. When you have a dozen tabs open, this is way faster than Ctrl+Tab cycling.

### Feature Shortcuts

| Action | Shortcut |
| :--- | :--- |
| Toggle bookmarks bar | **Ctrl + Shift + B** |
| Open history | **Ctrl + H** |
| Open downloads | **Ctrl + J** |
| Find on page | **Ctrl + F** |
| Next search match | **Ctrl + G** |
| Open DevTools | **F12** or **Ctrl + Shift + I** |
| Open DevTools console | **Ctrl + Shift + J** |

### Mouse Combos

Your mouse has combo moves too, especially the middle click (if your mouse has one).

| Action | Method |
| :--- | :--- |
| Open link in background tab | **Ctrl + Click** |
| Open link in new window | **Shift + Click** |
| Zoom in | **Ctrl + Scroll up** |
| Zoom out | **Ctrl + Scroll down** |
| Reset zoom | **Ctrl + 0** |

### Level Up: Vimium C Extension

If you're a Vim user, I strongly recommend the [Vimium C](https://github.com/gdh1995/vimium-c) browser extension. It adds Vim-style navigation to Chrome:

- **F** — shows labels on every clickable element; type the letters to click
- **/** — search within the page
- **J / K** — scroll down / up
- **H / L** — go back / forward
- **x** — close current tab
- **X** — restore closed tab

Once you install it, you can basically browse the web without touching your mouse.

---

## Shell Terminal

When editing commands in the terminal, most people use arrow keys to move one character at a time. That's painfully slow. Your shell has a built-in set of Emacs-style shortcuts (readline keybindings) that make editing long commands dramatically faster.

### Cursor Movement

This is the most valuable group to memorize. Ctrl+A to jump to the beginning and Ctrl+E to jump to the end — I use these literally every day.

| Shortcut | Action |
| :--- | :--- |
| **Ctrl + A** | Jump to beginning of line |
| **Ctrl + E** | Jump to end of line |
| ~~Ctrl + F~~ | ~~Move forward one character (same as right arrow, not worth memorizing)~~ |
| ~~Ctrl + B~~ | ~~Move backward one character (same as left arrow, not worth memorizing)~~ |
| **Alt + F** | Jump forward one word |
| **Alt + B** | Jump backward one word |
| **Ctrl + L** | Clear screen, cursor to top-left (same as the `clear` command) |

> **Real-world scenario:** You've typed a long command and realize the beginning is wrong. Ctrl+A jumps you to the start instantly — 10x faster than holding the left arrow key.

### Text Editing

| Shortcut | Action |
| :--- | :--- |
| ~~Ctrl + D~~ | ~~Delete character at cursor~~ |
| ~~Ctrl + T~~ | ~~Swap character at cursor with the one before it~~ |
| ~~Alt + T~~ | ~~Swap word at cursor with the one before it~~ |
| ~~Alt + L~~ | ~~Lowercase from cursor to end of word~~ |
| ~~Alt + U~~ | ~~Uppercase from cursor to end of word~~ |

I've struck these through because I rarely use them in practice. That said, Alt+U is occasionally handy for uppercasing variable names.

### Cut and Paste

Shell has its own clipboard system called the "kill ring." You cut with Ctrl+K/U and paste with Ctrl+Y — it's separate from your system clipboard.

| Shortcut | Action |
| :--- | :--- |
| **Ctrl + K** | Cut from cursor to end of line |
| **Ctrl + U** | Cut from cursor to beginning of line |
| ~~Alt + D~~ | ~~Cut from cursor to end of current word~~ |
| ~~Alt + Backspace~~ | ~~Cut from cursor to beginning of current word~~ |
| **Ctrl + Y** | Paste the most recently cut text |

> **Real-world scenario:** You've typed a command but aren't ready to run it yet. Ctrl+U cuts the whole line, you do something else, then Ctrl+Y pastes it back when you're ready. No need to retype anything.

### Command History

| Shortcut | Action |
| :--- | :--- |
| **Ctrl + R** | Reverse search through command history (incredibly useful) |
| **Ctrl + P** | Previous command (same as up arrow) |
| **Ctrl + N** | Next command (same as down arrow) |
| **!!** | Run the last command again |
| **sudo !!** | Re-run the last command with sudo |

> **Ctrl+R is the single most valuable terminal shortcut.** Press it, start typing any part of a previous command, and the shell fuzzy-searches your history. Ran a complex docker command an hour ago? Ctrl+R then type `docker` to find it instantly.

---

## Vim Editor

Vim has a steep learning curve, but once you're past the initial hump, your editing speed takes a quantum leap. I'm not going to try to cover everything here — just the core operations that matter most.

### Mode Switching

Vim's most counter-intuitive concept is modes. The key insight: **normal mode is the default state; insert mode is temporary.** You dip into insert mode to type, then come back to normal mode to navigate and manipulate.

| Shortcut | Action | When to use |
| :--- | :--- | :--- |
| **i** | Insert before cursor | Most common way to start typing |
| **I** | Insert at beginning of line | Adding something to the line start |
| **a** | Insert after cursor | Appending after current position |
| **A** | Insert at end of line | Adding to the end of a line |
| **o** | Open new line below and insert | Most common — adding a new line |
| **O** | Open new line above and insert | Inserting above current line |
| **s** | Delete character and insert | Replacing a single character |
| **S** | Delete entire line and insert | Rewriting a whole line |
| **Esc** | Return to normal mode | Anytime, anywhere |

### Cursor Movement

| Shortcut | Action |
| :--- | :--- |
| **h / j / k / l** | Left / Down / Up / Right |
| **w** | Jump to next word start |
| **b** | Jump to previous word start |
| **e** | Jump to end of current word |
| **0** | Jump to beginning of line |
| **$** | Jump to end of line |
| **gg** | Jump to beginning of file |
| **G** | Jump to end of file |
| **Ctrl + D** | Scroll down half a page |
| **Ctrl + U** | Scroll up half a page |
| **{** / **}** | Jump to previous / next blank line (paragraph navigation) |

> **Beginner tip:** Don't force yourself to use hjkl right away. Arrow keys work fine. Get comfortable with other operations first, and hjkl will come naturally over time.

### Delete Operations

Vim's delete operations follow a `verb + scope` grammar that's incredibly powerful once you internalize the pattern.

| Shortcut | Action |
| :--- | :--- |
| **x** | Delete character at cursor |
| **dd** | Delete entire line |
| **dw** | Delete from cursor to next word start |
| **daw** | Delete entire word (including surrounding spaces) |
| **diw** | Delete entire word (keeping surrounding spaces) |
| **D** | Delete from cursor to end of line (same as d$) |
| **dG** | Delete from current line to end of file |
| **dgg** | Delete from current line to beginning of file |

### Change Operations

The `c` commands are the upgraded version of `d` — they delete and then drop you into insert mode, saving a keystroke.

| Shortcut | Action |
| :--- | :--- |
| **cw** | Change to end of word |
| **ciw** | Change entire word |
| **ci"** | Change inside quotes |
| **ci(** | Change inside parentheses |
| **cc** | Change entire line |
| **C** | Change from cursor to end of line |

> **ciw is my most-used Vim command.** Cursor can be anywhere in the word — ciw selects and replaces the whole thing. Renaming variables, changing function arguments — it's absurdly convenient.

### Copy and Paste

| Shortcut | Action |
| :--- | :--- |
| **yy** | Yank (copy) entire line |
| **yw** | Yank to end of word |
| **p** | Paste after cursor |
| **P** | Paste before cursor |

### Undo and Redo

| Shortcut | Action |
| :--- | :--- |
| **u** | Undo |
| **Ctrl + R** | Redo |

### Search and Replace

| Shortcut | Action |
| :--- | :--- |
| **/keyword** | Search forward |
| **?keyword** | Search backward |
| **n** | Jump to next match |
| **N** | Jump to previous match |
| **\*** | Search for word under cursor |

For find-and-replace, use the `:s` command with the syntax `:%s/old/new/flags`:

```vim
:%s/foo/bar       " Replace first foo on each line with bar
:%s/foo/bar/g     " Replace ALL foo with bar globally
:%s/foo/bar/gc    " Replace all, but ask for confirmation each time
:%s/foo/bar/gi    " Replace all, case-insensitive
```

### Useful Command-Line Mode

```vim
:w                " Save
:q                " Quit
:wq               " Save and quit
:q!               " Quit without saving (force)
:w !sudo tee %    " Forgot to open with sudo? This saves the day
:set nu           " Show line numbers
:set nonu         " Hide line numbers
```

### My Vim Advice

1. **Don't try to learn everything at once.** Add 2-3 new commands per week. Use them until they're muscle memory, then learn more.
2. **Master quitting Vim first.** `:q!` to force quit, `:wq` to save and quit. Get those down, then build from there.
3. **ciw, dd, and o cover a huge portion of daily editing.** Prioritize these three.
4. **If you mostly write code, consider the Vim extension in VS Code.** You get Vim's editing efficiency with VS Code's ecosystem — best of both worlds.

---

## Recommended Resources

- [Vimium C](https://github.com/gdh1995/vimium-c) — Vim-style keyboard navigation for Chrome
- [The Linux Command Line](http://billie66.github.io/TLCL/book/chap09.html) — Detailed reference for shell shortcuts
- [Practical Vim](https://pragprog.com/titles/dnvim2/practical-vim-second-edition/) — The best book on Vim, hands down
- `vimtutor` — Vim's built-in interactive tutorial; just type `vimtutor` in your terminal
