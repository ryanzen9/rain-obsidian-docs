---
title: "MacOS Shell 个人配置"
date: 2026-04-20
author: ""
tags: []
categories: []
draft: true
---

## Brew

**[HomeBrew](https://brew.sh/zh-cn/)**：macOS（或 Linux）缺失的软件包的管理器。

**安装**：使用命令 `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`。

## Starship

**[Starship](https://starship.rs/zh-CN/guide/)**：轻量、迅速、客制化的高颜值终端。

**安装**：使用命令 `brew install starship`

在 `~/.zshrc` 的最后，添加以下内容：

```bash
eval "$(starship init zsh)"
```

**配置**：配置文件 `~/.config/starship.toml`。[参考](https://starship.rs/config/)

使用预设：**[Catppuccin Powerline Preset](https://starship.rs/presets/catppuccin-powerline)**

```bash
starship preset catppuccin-powerline -o ~/.config/starship.toml
```

> font: Maple Mono NF CN # 中英文等宽 符合 Nerd Font

## Ghostty

**Ghostty Config File**：

```text
font-family = "Maple Mono NF CN"
font-thicken = true
font-size = 12
adjust-cell-height = 6

theme = "Kanagawa Wave"

background-opacity = 0.95
macos-titlebar-style = transparent
window-padding-x = 14
window-padding-y = 10
window-save-state = never
window-width = 80
window-height = 24
window-theme = auto

cursor-style = bar
cursor-style-blink = true

mouse-hide-while-typing = true

quick-terminal-position = top
quick-terminal-screen = mouse
quick-terminal-autohide = true
quick-terminal-animation-duration = 0.15

confirm-close-surface = false

clipboard-paste-protection = true
clipboard-paste-bracketed-safe = true

shell-integration = detect
shell-integration-features = cursor,sudo,no-title,ssh-env,ssh-terminfo,path

# Tabs
keybind = cmd+t=new_tab
keybind = cmd+shift+left=previous_tab
keybind = cmd+shift+right=next_tab
keybind = cmd+w=close_surface

# Splits
keybind = cmd+d=new_split:right
keybind = cmd+shift+d=new_split:down
keybind = cmd+alt+left=goto_split:left
keybind = cmd+alt+right=goto_split:right
keybind = cmd+alt+up=goto_split:top
keybind = cmd+alt+down=goto_split:bottom

# Font size
keybind = cmd+plus=increase_font_size:1
keybind = cmd+minus=decrease_font_size:1
keybind = cmd+zero=reset_font_size

# Quick terminal global hotkey
# keybind = global:ctrl+grave_accent=toggle_quick_terminal
keybind = global:cmd+down=toggle_quick_terminal

# Splits management
keybind = cmd+shift+e=equalize_splits
keybind = cmd+shift+f=toggle_split_zoom

# Reload config
keybind = cmd+shift+comma=reload_config

# --- Performance ---
scrollback-limit = 25000000
```

## Oh-My-ZSH

安装 **omz** 以及对应插件：

```bash
# Oh-My-Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Zsh 插件
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

修改 `.zshrc` 文件配置 **omz**

```bash
# .zshrc

# oh-my-zsh
export ZSH="$HOME/.oh-my-zsh"

plugins=(git zsh-autosuggestions zsh-syntax-highlighting)

source $ZSH/oh-my-zsh.sh
```

执行 `source ~/.zshrc`

## Yazi

**[Yazi](https://yazi-rs.github.io/)**：基于异步 I/O，用 Rust 编写的超快终端文件管理器。支持多种图片格式预览，文件预览等。

**安装**：命令 `brew install yazi`。

安装相关插件 `brew install ffmpeg-full sevenzip jq poppler fd ripgrep fzf zoxide resvg imagemagick-full`

相关命令

```text
启动: yazi
移动: h j k l
打开: Enter
选择: Space
复制: y
剪切: x
粘贴: p
重命名: r
删除: d / D
隐藏文件: .
帮助: F1 或 ~
退出: q
```
