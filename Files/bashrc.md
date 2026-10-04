---
tags:
  - linux
  - files
  - shell
  - bash
aliases:
  - .bashrc
  - ~/.bashrc
  - bash.bashrc
  - /etc/bash.bashrc
  - /etc/bashrc
---
`.bashrc` is the script bash runs every time it starts an **interactive, non-login** shell — that is, almost every terminal window you open. It is where aliases, prompt (`PS1`), shell options and functions live.

> [!info] Two levels
> | Level       | Path                                                                      | Who it affects |
> | ----------- | ------------------------------------------------------------------------- | -------------- |
> | System-wide | `/etc/bash.bashrc` (Debian, Ubuntu, Arch) or `/etc/bashrc` (Fedora, RHEL) | Every user     |
> | Per-user    | `~/.bashrc`                                                               | Only you       |
>
> The system-wide path **differs per distribution**, and on some it is not read by bash directly but sourced from `/etc/profile` or from `~/.bashrc`. Check yours with `ls /etc/bash.bashrc /etc/bashrc 2>/dev/null`.

# Login vs interactive shells
This is the part everyone gets wrong. A shell is described by **two independent** yes/no properties:

| Property        | Meaning                                                     | How to check (bash)                    |
| --------------- | ----------------------------------------------------------- | -------------------------------------- |
| **Login**       | First shell after you authenticate (tty login, `ssh host`, `su -`, `bash -l`) | `shopt -q login_shell && echo login` |
| **Interactive** | Reads commands from you, has a prompt                       | `[[ $- == *i* ]] && echo interactive`  |

| Situation                               | Login | Interactive |
| --------------------------------------- | ----- | ----------- |
| Text console login (`Ctrl+Alt+F3`)      | yes   | yes         |
| `ssh host`                              | yes   | yes         |
| `ssh host 'ls'`                         | no    | no          |
| New terminal window in a desktop        | no*   | yes         |
| `bash` typed in a terminal              | no    | yes         |
| `bash script.sh`, `#!/bin/bash` scripts | no    | no          |
| `su -` / `sudo -i`                      | yes   | yes         |

\* Some terminal emulators (and macOS Terminal) start login shells by default.

> [!warning] Scripts do NOT read `.bashrc`
> Aliases and functions from `.bashrc` are not available in scripts. That is intentional: a script must behave the same for every user. Put what scripts need inside the script, or in a file you `source` explicitly.

# Load order

### bash
| Shell type                  | Files read, in order                                                                                   |
| --------------------------- | ------------------------------------------------------------------------------------------------------ |
| Login (interactive or not)  | `/etc/profile` → **first one found** of `~/.bash_profile`, `~/.bash_login`, `~/.profile`              |
| Interactive non-login       | system bashrc (if the build enables it) → `~/.bashrc`                                                  |
| Non-interactive (scripts)   | only the file named in `$BASH_ENV`, if set                                                              |
| On logout of a login shell  | `~/.bash_logout`                                                                                        |

Because a login shell does **not** read `~/.bashrc` on its own, almost every distro's default `~/.bash_profile` contains:
```bash
[[ -f ~/.bashrc ]] && . ~/.bashrc
```

### zsh
zsh reads all applicable files, in this order (`/etc/zsh/` on some distros, `/etc/` on others):

| File        | Login | Interactive | Script |
| ----------- | ----- | ----------- | ------ |
| `zshenv`    | yes   | yes         | yes    |
| `zprofile`  | yes   |             |        |
| `zshrc`     |       | yes         |        |
| `zlogin`    | yes   |             |        |
| `zlogout`   | on logout |         |        |

Each is read first from the system directory, then `~/.zsh*` (or `$ZDOTDIR`).

### fish
fish has no login/rc split by file. It reads everything every time and you test inside: see [[fish-config]].

| Order | Path                                    |
| ----- | --------------------------------------- |
| 1     | `$__fish_data_dir/config.fish`          |
| 2     | vendor `conf.d/*.fish` (from packages)  |
| 3     | `/etc/fish/conf.d/*.fish`               |
| 4     | `~/.config/fish/conf.d/*.fish`          |
| 5     | `/etc/fish/config.fish`                 |
| 6     | `~/.config/fish/config.fish`            |

Inside: `status is-login`, `status is-interactive`.

# A typical `~/.bashrc`
A freshly created user usually gets a copy of the one in [[skel]]:

```bash
# If not running interactively, don't do anything
[[ $- != *i* ]] && return

alias ls='ls --color=auto'
alias grep='grep --color=auto'
PS1='[\u@\h \W]\$ '
```

The first line matters: it makes `scp`, `rsync` and `ssh host cmd` safe, because anything printing text in `.bashrc` would break those protocols.

### What belongs where

| Put it in              | What                                                              |
| ---------------------- | ----------------------------------------------------------------- |
| `~/.bashrc`            | aliases, functions, `PS1`, `shopt`, completion, `HISTSIZE`        |
| `~/.profile` / `~/.bash_profile` | `export PATH=...`, `export EDITOR=...`, things for the whole session → [[profile]] |
| A script               | anything a script needs                                           |

### Useful settings
```bash
HISTSIZE=10000              # lines kept in memory
HISTFILESIZE=20000          # lines kept in ~/.bash_history
HISTCONTROL=ignoreboth      # skip duplicates and lines starting with space
shopt -s histappend         # append, don't overwrite, history file
shopt -s globstar           # ** matches recursively
shopt -s autocd             # type a dir name to cd into it
set -o vi                   # vi keybindings (default is emacs)
```

### PS1 escapes
| Escape | Meaning                   |
| ------ | ------------------------- |
| `\u`   | user                      |
| `\h`   | short hostname            |
| `\w`   | full working dir (`~` for home) |
| `\W`   | last part of working dir  |
| `\$`   | `#` for root, `$` otherwise |
| `\t`   | time `HH:MM:SS`           |
| `\[ \]`| wrap colour codes so line editing doesn't break |

# Scripting recipes

### Which kind of shell am I?
```bash
case $- in *i*) echo interactive;; *) echo non-interactive;; esac
shopt -q login_shell && echo login || echo "not login"
```

### Reload after editing
```bash
source ~/.bashrc     # or: . ~/.bashrc
```

### Debug what gets loaded
```bash
bash -x -i -c exit 2>&1 | grep -E '^\++ (\.|source) '   # shows sourced files
PS4='+ ${BASH_SOURCE}:${LINENO}: ' bash -xli -c exit 2>&1 | less
```

### Start bash without any config (to test whether your rc breaks something)
```bash
bash --norc --noprofile
env -i bash --norc --noprofile    # also with empty environment
```

### Keep a machine-local file out of a shared dotfile repo
```bash
[[ -r ~/.bashrc.local ]] && . ~/.bashrc.local
```

> [!tip] Order of definitions
> `.bashrc` runs top to bottom. An alias defined twice uses the **last** definition. Use `type ll` or `alias ll` to see what is active right now.

# See also
- [[profile]] — login-time files and `PATH`
- [[fish-config]] — the fish equivalent
- [[skel]] — where a new user's `.bashrc` comes from
- [[home]] — dotfiles in general
- [[etc]] — system-wide configuration
- [[files]] — all file notes
- `man bash` → section **INVOCATION**
