---
tags:
  - linux
  - files
  - shell
  - fish
aliases:
  - config.fish
  - ~/.config/fish
  - fish_variables
  - conf.d
---
fish keeps all its configuration under `~/.config/fish/` (that is `$__fish_config_dir`, following the XDG layout → [[home]]). Unlike bash, it does **not** split config by login/interactive shell into different files — every fish process reads the same files and you test the shell type inside them.

```
~/.config/fish/
├── config.fish        # main config, runs at every start
├── conf.d/            # *.fish snippets, run before config.fish (alphabetical)
├── functions/         # one function per file, loaded lazily on first use
├── completions/       # tab completions, loaded lazily
└── fish_variables     # universal variables (managed by fish — don't edit by hand)
```

> [!info] fish ignores POSIX profile files
> `/etc/profile`, `~/.profile` and `~/.bashrc` are not read. A variable set there reaches fish only if a parent process exported it. Load order for bash/zsh is in [[bashrc]] and [[profile]].

# Load order

| # | Path                                                         | Variable              |
| - | ------------------------------------------------------------ | --------------------- |
| 1 | `/usr/share/fish/config.fish` (built-in defaults)            | `$__fish_data_dir`    |
| 2 | vendor `conf.d` dirs — snippets shipped by packages, e.g. `/usr/share/fish/vendor_conf.d/` | `$__fish_vendor_confdirs` |
| 3 | `/etc/fish/conf.d/*.fish`                                    | `$__fish_sysconf_dir` |
| 4 | `~/.config/fish/conf.d/*.fish`                               | `$__fish_config_dir`  |
| 5 | `/etc/fish/config.fish` (all users)                          |                       |
| 6 | `~/.config/fish/config.fish` (you)                           |                       |

Steps 2–4 are really one step: the snippets from all three `conf.d` locations are merged and run sorted by **file name**. If the same file name exists in several dirs, only the **first** one (user > system > vendor) runs. That's how you override a packaged snippet: create an empty file with the same name in `~/.config/fish/conf.d/`.

```fish
echo $__fish_config_dir $__fish_sysconf_dir $__fish_data_dir
printf '%s\n' $__fish_vendor_confdirs
```

# `config.fish`
```fish
# runs for every fish: interactive, login, and scripts (fish script.fish)
fish_add_path ~/.local/bin

set -gx EDITOR nvim
set -gx VISUAL nvim

if status is-interactive
    # prompts, abbreviations, key bindings, tools that print output
    abbr -a gs git status
    abbr -a ll ls -lah
end

if status is-login
    # once per login session
end
```

| Check                    | True when                         |
| ------------------------ | --------------------------------- |
| `status is-interactive`  | there is a prompt                 |
| `status is-login`        | first shell after login (`fish -l`) |
| `status is-command-substitution` | inside `( )`              |

> [!warning] Quoted `~` is not expanded
> ```fish
> set PATH "~/.local/bin" $PATH   # WRONG: adds a literal directory named "~"
> set PATH ~/.local/bin $PATH     # expands to /home/you/.local/bin
> fish_add_path ~/.local/bin      # best: no duplicates, persistent by default
> ```
> Same rule as bash: `~` only expands **unquoted**. Check with `printf '%s\n' $PATH`.

# Variables and scopes
fish variables are **lists**. `$PATH` is a list, not a `:`-separated string (fish converts when exporting).

| Flag | Scope       | Lifetime                                                           |
| ---- | ----------- | ------------------------------------------------------------------ |
| `-l` | local       | current block / function                                           |
| `-f` | function    | whole function                                                     |
| `-g` | global      | this fish process                                                  |
| `-U` | universal   | **all** fish sessions of this user, now and after reboot (stored in `fish_variables`) |
| `-x` | export      | combine with the above: passed to child processes                  |

```fish
set -gx EDITOR nvim          # like bash: export EDITOR=nvim
set -e EDITOR                # unset
set -S EDITOR                # show scope, export status, value
set -U                       # list universal vars
```

> [!tip] Universal variables vs `config.fish`
> `set -U` writes to `~/.config/fish/fish_variables` once and is instantly visible in every open fish. Handy, but invisible when you read `config.fish` later. For anything you want in a dotfiles repo, prefer `set -g` in `config.fish`.

# Functions
One file per function in `functions/`, file name = function name. fish loads it on first call, so many functions cost nothing at startup.

```fish
# ~/.config/fish/functions/mkcd.fish
function mkcd --description 'mkdir and cd'
    mkdir -p $argv[1]; and cd $argv[1]
end
```

```fish
funced mkcd          # edit interactively
funcsave mkcd        # write to ~/.config/fish/functions/mkcd.fish
functions --details fish_prompt   # which file defines it
type ls              # alias/function/builtin/file?
```

`alias` exists, but in fish it just creates a function. **Abbreviations** (`abbr`) are usually better for interactive use: they expand when you press space, so history shows the real command.

# Bash → fish cheat sheet

| bash                           | fish                                    |
| ------------------------------ | --------------------------------------- |
| `export X=1`                   | `set -gx X 1`                           |
| `X=1 cmd`                      | `X=1 cmd` (same) or `env X=1 cmd`       |
| `$(cmd)`                       | `(cmd)` or `$(cmd)`                     |
| `cmd1 && cmd2`                 | `cmd1; and cmd2` (or `&&` in fish ≥ 3)  |
| `if [ -f x ]; then ...; fi`    | `if test -f x; ...; end`                |
| `for i in *; do ...; done`     | `for i in *; ...; end`                  |
| `$?`                           | `$status`                               |
| `$$`                           | `$fish_pid`                             |
| `$1`, `$@`                     | `$argv[1]`, `$argv`                     |
| `source ~/.bashrc`             | `source ~/.config/fish/config.fish`     |
| `$((1 + 2))`                   | `math 1 + 2`                            |
| `${var#prefix}`                | `string replace -r '^prefix' '' $var`   |
| `cmd <<< "$s"`                 | `echo $s \| cmd`                        |

Bash scripts still run fine from fish: `bash script.sh` or a `#!/usr/bin/env bash` shebang.

# Scripting recipes

### Debug startup
```fish
fish --no-config                    # start without any config
fish --profile-startup /tmp/p.txt -c exit; sort -nk2 /tmp/p.txt | tail
fish --debug=config -c exit 2>&1 | grep '^config:'   # where fish looks for config
```

### Show PATH one per line
```fish
printf '%s\n' $PATH
```

### Run a bash-only snippet (e.g. a tool's `eval "$(tool init bash)"`)
Most tools have a fish mode: `tool init fish | source`. If not:
```fish
bash -c 'source ./env.sh; env' | string match -r '^MYVAR=.*' | string split -m1 = | read -l --line k v; set -gx $k $v
```

### Reload config
```fish
source ~/.config/fish/config.fish
exec fish          # cleaner: replace the shell with a fresh one
```

# See also
- [[bashrc]] — load order of bash/zsh/fish side by side
- [[profile]] — `PATH` concepts
- [[home]] — XDG directories
- [[skel]] — default dotfiles for new users
- [[files]] — all file notes
- `man fish-doc`, `help` inside fish (opens the docs)
