---
title: Switching between dark and light themes in an instant
date: 2019-11-04 17:07:37
tags: ['theming', 'neovim', 'terminal', 'tmux']
---

## Why
I like to sync up my themes to the lighting in the room. During the day light theme is better and at night darker themes are nicer to look at. I want to switch between these modes in an instant.

## Requirements
I use a lot of terminal applications. So everything in the terminal needs to be able to switch between light and dark mode.

Most important were:

- current terminal emulator
- tmux
- neovim


## Initial colors

### Terminal emulator

I found a tool called [pal](https://github.com/dylanaraps/pal) by dylanaraps. Pal changes colors on currently open terminals. It uses escape codes so the terminal emulator just needs to understand them which most of the modern terminal emulators do. The colors can be supplied as a string or read from a file. I put the script to `~/.scripts/pal`.

```bash
# Set colors from a string
pal "1f221f a0ab9e b6bfb4 67a96c 90a992 79a985 47a961 c7c7c7 575957 a0ab9e b6bfb4 67a96c 90a992 79a985 47a961 c7c7c7"

# Read from a file
pal < file
```

I created 2 files for dark and light themes.

`~/.colors/pal-dark`

```text
#2c323c
#ff0000
#2ecc71
#FF8C00
#3465a4
#cA70D6
#20B2AA
#d3d7cf
#373e4b
#ef2929
#27ae60
#fce94f
#61afef
#DA70D6
#34e2e2
#eeeeec
```

`~/.colors/pal-light`

```text
#f0f0f0
#e45649
#2ecc71
#e5c07b
#3465a4
#a626a4
#4e9f9f
#222222
#d0d0d0
#ca1243
#27ae60
#d5b06b
#4078f2
#DB7093
#0083be
#181a1f
```

From pal documentation I found that these colors can be applied to new terminals using this snippet in `~/.bashrc`.

```bash
{ read -r < ~/.cache/pal/colors; printf %b "$REPLY"; } & disown
```

### Less

Less didn't like the new colors and sometimes the background and foreground would be the same, making text unreadable. I found out that less actually uses environment variables to define those colors. I added custom color exports to my `~/.bashrc`. I haven't noticed any other programs to have this problem.

```bash
# Less
export LESS_TERMCAP_mb=$'\E[6m'          # begin blinking
export LESS_TERMCAP_md=$'\E[37m'         # begin bold
export LESS_TERMCAP_us=$'\E[4;37m'       # begin underline
export LESS_TERMCAP_so=$'\E[1;1;47m\E[1;1;30m'    # begin standout-mode - info box
export LESS_TERMCAP_me=$'\E[0m'          # end mode
export LESS_TERMCAP_ue=$'\E[0m'          # end underline
export LESS_TERMCAP_se=$'\E[0m'          # end standout-mode
```

### Tmux

You could create 2 different themes for tmux, but I just used pal defined colors to avoid that. If you want to use 2 different themes for tmux just follow along.

```
set-option -g status-interval 1
######## THEME ##########
# Basic status bar colors
set -g status-style bg=colour8

setw -g window-status-current-style fg=green
setw -g window-status-style fg=white
set-window-option -g mode-style bg=white,fg=black

# Left side of status bar
set-option -g status-left-length 40
set-option -g status-left ""
set-option -g status-left-style fg=yellow

# Right side of status bar
set-option -g status-right-length 70
set-option -g status-right " %H:%M:%S %d.%m.%y"

set-option -g message-style fg=black,bg=colour12
set-option -g message-command-style fg=black,bg=colour12

# Window status
set-option -g window-status-format " #I:#W "
set-option -g window-status-style fg=default
set-option -g window-status-current-format " #I:#W "
set-option -g window-status-current-style fg=black,bg=colour12

# Window separator
set-option -g window-status-separator ""

# Window status alignment
set-option -g status-justify left

set-option -g pane-border-style fg=colour8
set-option -g pane-active-border-style fg=blue
```

### neovim

To get neovim to respect the current theme I created a script that checks if the current background is light or dark and outputs the theme name. It simply uses pal cache to determine the background color. Put this script somewhere in your path. Mine is at `~/.scripts/pal_theme`.

```bash
#!/usr/bin/env bash
if test -f ~/.cache/pal/colors ; then
    bg_hex=$(grep -o "#[a-fA-F0-9]\{6\}" ~/.cache/pal/colors | head -n1 | tr -d '#')
    r=$(printf "%d" 0x${bg_hex:0:2})
    g=$(printf "%d" 0x${bg_hex:2:2})
    b=$(printf "%d" 0x${bg_hex:4:2})

    # 382 = (255 + 255 + 255) / 2
    if [ $(($r + $g + $b)) -gt 382 ]; then
        echo "light"
    else
        echo "dark"
    fi

    exit $?
else
    exit 1
fi
```

Using this script we can create a wrapper function in `~/.bashrc` to launch neovim.

```bash
neovim_bg_control() {
  if test -f ~/.scripts/pal_theme ; then
    local theme=$(~/.scripts/pal_theme)

    if [ $theme = "light" ]; then
	env VIM_THEME="light" nvim "$@"
    else
      nvim "$@"
    fi
  else
    nvim "$@"
  fi
  return $?
}

alias vim="neovim_bg_control"
```

I experimented with `nvim -c "set background=<theme>"` but it caused the default background to flash before the actual background is set. This is because the `init.vim` is evaluated first and then the given command is executed. Then I found a way to use an environment variable. Because this environment variable is set before launch we can make use of it directly in `init.vim`.

```vim
" Load theme depending on environment variable
if exists("$VIM_THEME") && $VIM_THEME == "light"
  set background=light
else
  set background=dark
endif
```

The theme is now set at startup.


## Switching themes

I use a script called `~/.scripts/switch_theme` to switch between light and dark mode. It uses rofi as a nice way to select the theme but also works from the commandline.

```bash
#!/bin/bash
if [ -x "$(command -v rofi)" ] ; then
        theme=$(echo -ne "dark\nlight" | rofi -p "Switch to theme:" -dmenu -theme default)
else
        theme=$1
fi

current_theme=$(~/.scripts/pal_theme)

if [ $theme = "dark" ]; then
        if [ $current_theme = "light" ]; then
                killall -s 10 nvim
        fi
        $HOME/.scripts/pal < $HOME/.colors/pal-dark

elif [ $theme = "light" ]; then
        if [ $current_theme = "dark" ]; then
                killall -s 10 nvim
        fi
        $HOME/.scripts/pal < $HOME/.colors/pal-light
fi
```

Pal takes care of the terminal emulator theme switching at runtime but what about tmux and neovim?

### tmux

tmux is a server that you can command straight from the commandline. If you are using 2 theme files for example `~/.tmux-dark-status` and `~/.tmux-light-status`. An easy way is to integrate this into the script is:

```bash
if [ $theme = "dark" ]; then
        if [ $current_theme = "light" ]; then
                killall -s 10 nvim
        fi
        tmux source-file ~/.tmux-dark-status
        $HOME/.scripts/pal < $HOME/.colors/pal-dark

elif [ $theme = "light" ]; then
        if [ $current_theme = "dark" ]; then
                killall -s 10 nvim
        fi
        tmux source-file ~/.tmux-light-status
        $HOME/.scripts/pal < $HOME/.colors/pal-light
fi
```

Because one tmux server runs all the sessions, the theme of all your tmux windows will be refreshed.

### neovim

From the `~/.scripts/switch_theme` script you can already guess what we are doing here. Neovim recently introduced a new feature which allows the use of SIGUSR1. We can make an autocommand for this signal to switch themes when the signal is received.

```vim
function! Swap()
    if &background ==? 'dark'
        set background=light
    else
        set background=dark
    endif
endfunction

autocmd Signal * call Swap()
```

## Results

We can easily switch between light and dark themes with one command.

```bash
# Switch to light theme
switch_theme light

# Switch to dark theme
switch_theme dark
```

All of the files modified:

- Install [pal](https://github.com/dylanaraps/pal) to `PATH`.

- Possibly `~/.tmux-dark-status`
- Possibly `~/.tmux-light-status`
- `~/.scripts/switch_theme`
- `~/.scripts/pal_theme`
- Added the following to `~/.bashrc`:

  ```bash
  # Pal to new terminals
  { read -r < ~/.cache/pal/colors; printf %b "$REPLY"; } & disown

  neovim_bg_control() {
    if test -f ~/.scripts/pal_theme ; then
      local theme=$(~/.scripts/pal_theme)

      if [ $theme = "light" ]; then
          env VIM_THEME="light" nvim "$@"
      else
        nvim "$@"
      fi
    else
      nvim "$@"
    fi
    return $?
  }

  alias vim="neovim_bg_control"

  # Less
  export LESS_TERMCAP_mb=$'\E[6m'          # begin blinking
  export LESS_TERMCAP_md=$'\E[37m'         # begin bold
  export LESS_TERMCAP_us=$'\E[4;37m'       # begin underline
  export LESS_TERMCAP_so=$'\E[1;1;47m\E[1;1;30m'    # begin standout-mode - info box
  export LESS_TERMCAP_me=$'\E[0m'          # end mode
  export LESS_TERMCAP_ue=$'\E[0m'          # end underline
  export LESS_TERMCAP_se=$'\E[0m'          # end standout-mode
  ```

- Added the following to `init.vim`
    ```vim
    function! Swap()
        if &background ==? 'dark'
            set background=light
        else
            set background=dark
        endif
    endfunction

    autocmd Signal * call Swap()

    " Load theme depending on environment variable
    if exists("$VIM_THEME") && $VIM_THEME == "light"
      set background=light
    else
      set background=dark
    endif
    ```




Note that all of the scripts are in my `PATH` environment variable for easy access.
