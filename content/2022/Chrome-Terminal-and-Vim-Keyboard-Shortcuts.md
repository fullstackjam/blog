+++
title = "Chrome, Shell and Vim Keyboard Shortcuts"
date = 2022-04-24
description = "Essential keyboard shortcuts for Chrome browser, Shell terminal, and Vim editor to boost your productivity"

[extra.comments]
issue_id = 2
+++
In this post, you will learn：

- Practical Chrome keyboard Shortcuts.
- Practical Shell Keyboard Shortcuts keys.
- Parctical Vim Skill.

<!--more-->

## Chrome keyboard shortcuts

### Tab and window shortcuts

| Action                                                       | Shortcut              |
| ------------------------------------------------------------ | :-------------------- |
| Open a new window                                            | **Ctrl + n**          |
| Open a new window in Incognito mode                          | **Ctrl + Shift + n**  |
| Open a new tab, and jump to it                               | **Ctrl + t**          |
| Reopen previously closed tabs in the order they were closed  | **Ctrl + Shift + t**  |
| Jump to the next open tab                                    | **Ctrl + Tab**        |
| Jump to the rightmost tab                                    | **Ctrl + 9**          |
| Open the previous page from your browsing history in the current tab | **Alt + Left arrow**  |
| Open the next page from your browsing history in the current tab | **Alt + Right arrow** |
| Close the current tab                                        | **Ctrl + w**          |

### Google Chrome feature shortcuts

| **Action**                                     | **Shortcut**           |
| ---------------------------------------------- | ---------------------- |
| Show or hide the Bookmarks bar                 | **Ctrl + Shift + b**   |
| Open the History page in a new tab             | **Ctrl + h**           |
| Open the Downloads page in a new tab           | **Ctrl + j**           |
| Open the Find Bar to search the current page   | **Ctrl + f** or **F3** |
| Jump to the next match to your Find Bar search | **Ctrl + g**           |

### Mouse shortcuts

| **Action**                          | **Shortcut**                           |
| ----------------------------------- | -------------------------------------- |
| Open a link in new background tab   | **Ctrl +** Click a link                |
| Open a link in a new window         | **Shift +** Click a link               |
| Make everything on the page bigger  | **Ctrl +** Scroll your mousewheel up   |
| Make everything on the page smaller | **Ctrl +** Scroll your mousewheel down |

- [gdh1995/vimium-c: A keyboard shortcut browser extension for keyboard-based navigation and tab operations with an advanced omnibar (github.com)](https://github.com/gdh1995/vimium-c)

## Shell

### Cursor Movement Commands

| Key        | Action                                                       |
| :--------- | :----------------------------------------------------------- |
| Ctrl-a     | Move cursor to the beginning of the line.                    |
| Ctrl-e     | Move cursor to the end of the line.                          |
| ~~Ctrl-f~~ | ~~Move cursor forward one character;same as the right arrow key.~~ |
| ~~Ctrl-b~~ | ~~Move cursor backward one character;same as the left arrow key.~~ |
| Alt-f      | Move cursor forward one word.                                |
| Alt-b      | Move cursor backward one word.                               |
| Ctrl-l     | Clear the screen and move the cursor to the top left corner. The clear command does the same thing. |

### Text Editing Commands

| Key        | Action                                                       |
| :--------- | :----------------------------------------------------------- |
| ~~Ctrl-d~~ | ~~Delete the character at the cursor location~~              |
| ~~Ctrl-t~~ | ~~Transpose(exchange)the character at the cursor location with the one preceding it.~~ |
| ~~Alt-t~~  | ~~Transpose the word at the cursor location with the one preceding it.~~ |
| ~~Alt-l~~  | ~~Convert the characters from the cursor location to the end of the word to lowercase.~~ |
| ~~Alt-u~~  | ~~Convert the characters from the cursor location to the end of the word to uppercase.~~ |

### Cut And Paste Commands

| Key               | Action                                                       |
| :---------------- | :----------------------------------------------------------- |
| Ctrl-k            | Kill text from the cursor location to the end of line.       |
| Ctrl-u            | Kill text from the cursor location to the beginning of the line. |
| ~~Alt-d~~         | ~~Kill text from the cursor location to the end of the current word.~~ |
| ~~Alt-Backspace~~ | ~~Kill text from the cursor location to the beginning of the word. If the cursor is at the beginning of a word, kill the previous word.~~ |
| Ctrl-y            | Yank text from the kill-ring and insert it at the cursor location. |

- [TLCL (billie66.github.io)](http://billie66.github.io/TLCL/book/chap09.html)

## Vim

```
cw
x w b e
i I a A o O s S
0 $
gg G
d dd dw daw D dG dgg 
p yy 
u ctrl + r
```

```
:w !sudo tee %
:set nu
:%s/zempty/handsome/i
:%s/zempty/handsome
:%s/zempty/handsome/g
:%s/zempty/handsome/gc
```

- [Vim实用技巧.pdf (aliyuncs.com)](https://library-cdq.oss-cn-beijing.aliyuncs.com/technology/Vim实用技巧.pdf)
