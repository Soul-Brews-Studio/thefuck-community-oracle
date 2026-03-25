# /learn dev-environment-files

> Source: https://github.com/josean-dev/dev-environment-files
> Learned: 2026-03-26
> By: Josean Martinez (popular YouTube dev environment tutorials)

## What This Is

A curated macOS dev environment dotfiles repo. Not a framework — a real person's actual config. The value is in seeing how modern CLI tools compose into a cohesive workflow.

## File Structure

```
.zshrc                          # Shell config (the hub)
.tmux.conf                      # Terminal multiplexer
.wezterm.lua                    # WezTerm terminal config
coolnight.itermcolors           # iTerm2 color scheme
.config/
  alacritty/alacritty.toml      # Alacritty terminal config
  alacritty/themes/             # Color themes (including coolnight)
  aerospace/aerospace.toml      # Tiling window manager
  skhd/skhdrc                   # Hotkey daemon for yabai
qmk/                            # Keyboard firmware (Corne)
```

## The Tool Stack

### Terminal Emulators

**WezTerm** (`.wezterm.lua`):
- Custom color scheme (dark blue: `#011423`)
- MesloLGS Nerd Font Mono, size 19
- No tab bar, resize-only decorations
- Clean, minimal config

**Alacritty** (`alacritty.toml`):
- Coolnight custom theme
- 80% opacity with blur (transparent terminal)
- Padding: 10px
- Same Nerd Font
- `option_as_alt = "Both"` — essential for macOS alt-key bindings

### Shell: Zsh (`.zshrc`)

**Prompt**: Powerlevel10k with instant prompt (loads cached prompt before anything else)

**History**:
```zsh
HISTFILE=$HOME/.zhistory
SAVEHIST=1000
HISTSIZE=999
setopt share_history hist_expire_dups_first hist_ignore_dups hist_verify
```
Arrow keys bound to `history-search-backward/forward` — type partial command, press up, get matching history.

**Plugins** (via Homebrew):
- `zsh-autosuggestions` — ghost text suggestions from history
- `zsh-syntax-highlighting` — real-time command coloring

### The CLI Tools Ecosystem (the gold)

This is where the config shines — showing how modern CLI tools integrate:

**fzf** (fuzzy finder):
```zsh
eval "$(fzf --zsh)"                    # Key bindings + completion
export FZF_DEFAULT_COMMAND="fd --hidden --strip-cwd-prefix --exclude .git"
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
export FZF_ALT_C_COMMAND="fd --type=d --hidden --strip-cwd-prefix --exclude .git"
```

Custom FZF theme matching terminal colors:
```zsh
export FZF_DEFAULT_OPTS="--color=fg:#CBE0F0,bg:#011628,hl:#B388FF,..."
```

**fd** powers fzf's file finding:
```zsh
_fzf_compgen_path() { fd --hidden --exclude .git . "$1" }
_fzf_compgen_dir() { fd --type=d --hidden --exclude .git . "$1" }
```

**bat** + **eza** power fzf previews:
```zsh
show_file_or_dir_preview="if [ -d {} ]; then eza --tree --color=always {} | head -200; else bat -n --color=always --line-range :500 {}; fi"
export FZF_CTRL_T_OPTS="--preview '$show_file_or_dir_preview'"
export FZF_ALT_C_OPTS="--preview 'eza --tree --color=always {} | head -200'"
```

Context-aware completions:
```zsh
_fzf_comprun() {
  case "$command" in
    cd)           fzf --preview 'eza --tree ...' ;;
    export|unset) fzf --preview "eval 'echo \${}'" ;;
    ssh)          fzf --preview 'dig {}' ;;
    *)            fzf --preview "$show_file_or_dir_preview" ;;
  esac
}
```

**fzf-git.sh** — Git-aware fzf functions (branches, commits, files, stash, etc.)

**eza** (modern ls):
```zsh
alias ls="eza --icons=always"
```

**bat** (better cat):
```zsh
export BAT_THEME=tokyonight_night
```

**thefuck**:
```zsh
eval $(thefuck --alias)
eval $(thefuck --alias fk)  # Also available as `fk`
```

**zoxide** (smarter cd):
```zsh
eval "$(zoxide init zsh)"
alias cd="z"  # Replace cd entirely
```

**yazi** (terminal file manager):
```zsh
function y() {
    local tmp="$(mktemp -t "yazi-cwd.XXXXXX")" cwd
    yazi "$@" --cwd-file="$tmp"
    if cwd="$(command cat -- "$tmp")" && [ -n "$cwd" ] && [ "$cwd" != "$PWD" ]; then
        builtin cd -- "$cwd"
    fi
    rm -f -- "$tmp"
}
```
This wrapper changes directory to wherever yazi navigated to on exit.

### Tmux (`.tmux.conf`)

**Prefix**: `C-a` (not default `C-b`)

**Splits**: `|` for horizontal, `-` for vertical (intuitive)

**Resize**: vim-style `h/j/k/l` with 5-unit steps

**Vi mode**: `mode-keys vi`, `v` to select, `y` to copy

**Plugins** (via TPM):
- `vim-tmux-navigator` — Seamless vim/tmux pane switching
- `tmux-tokyo-night` — Theme
- `tmux-resurrect` — Persist sessions across restarts
- `tmux-continuum` — Auto-save sessions every 15 min

**Key insight**: `escape-time 10` — removes ESC delay for Neovim inside tmux. Without this, ESC in insert mode has a noticeable lag.

### Window Management

**AeroSpace** (`aerospace.toml`):
- Tiling WM for macOS
- Auto-start at login
- Mouse follows focus
- i3-style workspace management

**skhd** (`skhdrc`) — hotkey daemon for yabai (alternative tiling WM):
- `alt + hjkl` — focus windows
- `hyper + hjkl` — swap windows
- `hyper + m` — maximize/toggle
- `hyper + 1-7` — move window to space
- Vim-style keybindings throughout

### Keyboard: QMK (Corne)

Custom firmware for a split ergonomic keyboard (Corne/crkbd). The config file is a JSON layout — indicates deep investment in typing efficiency.

## Key Patterns

### 1. Tools Compose, Not Replace

The genius of this setup is composition:
- **fd** finds files → feeds **fzf** → preview via **bat** (files) or **eza** (dirs)
- **zoxide** tracks directories → replaces `cd` entirely
- **thefuck** catches errors → fixes them
- Each tool does one thing. Together they create a workflow.

### 2. Consistent Theme

Everything uses the same dark blue color scheme:
- Terminal: `#011423` / `#011628`
- FZF: matching colors
- Bat: tokyonight_night
- Tmux: tokyo-night
- Alacritty: coolnight

### 3. Vim Keybindings Everywhere

- Neovim (editor)
- Tmux (copy mode + pane navigation)
- skhd/aerospace (window management)
- fzf (navigation)

One muscle memory, applied everywhere.

### 4. Nerd Fonts as the Glue

MesloLGS Nerd Font Mono everywhere — enables icons in eza, Powerlevel10k, Neovim, and tmux status bar.

## Tools Summary

| Tool | Replaces | Key Feature |
|------|----------|-------------|
| fzf | manual search | Fuzzy find anything |
| fd | find | Simpler, faster, respects .gitignore |
| bat | cat | Syntax highlighting, line numbers |
| eza | ls | Icons, colors, tree view |
| zoxide | cd | Learns your frequent directories |
| thefuck | re-typing | Auto-correct commands |
| yazi | Finder | Terminal file manager with preview |
| tmux | tabs | Persistent sessions, splits, remote |
| Powerlevel10k | plain prompt | Git status, instant prompt |
| AeroSpace/yabai | manual window mgmt | Tiling WM for macOS |

## Lessons for Dev Environment Setup

1. **Start with the shell** — .zshrc is the hub that wires everything together
2. **Let tools compose** — fd + fzf + bat + eza is more powerful than any single tool
3. **Consistent keybindings** — vim-style everywhere reduces cognitive load
4. **Consistent theming** — one color scheme across all tools creates visual cohesion
5. **Alias aggressively** — `ls="eza --icons=always"`, `cd="z"` — make the better tool the default
6. **Preview everything** — fzf previews make file navigation visual and fast
7. **Persist state** — tmux-resurrect + continuum means you never lose your layout

---

*"The terminal remembers what fingers forget."*
*— Thefuck Community Oracle*
