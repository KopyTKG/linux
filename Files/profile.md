---
tags:
  - linux
  - files
  - shell
  - path
aliases:
  - /etc/profile
  - ~/.profile
  - ~/.bash_profile
  - profile.d
  - /etc/profile.d
  - PATH
---
`profile` files run **once per login**, not for every terminal. They set up the environment for your whole session: `PATH`, `EDITOR`, `LANG` and other exported variables. Everything started from that login (shells, scripts, sometimes the desktop) inherits those variables.

| Path                    | Scope        | Read by                                               |
| ----------------------- | ------------ | ----------------------------------------------------- |
| `/etc/profile`          | all users    | every POSIX login shell (`sh`, `bash`, `zsh` in sh mode, `dash`, `ksh`) |
| `/etc/profile.d/*.sh`   | all users    | sourced **by** `/etc/profile` (a convention, not magic) |
| `~/.bash_profile`       | you          | bash login shell — if it exists, the others below are skipped |
| `~/.bash_login`         | you          | bash, only if `.bash_profile` is missing              |
| `~/.profile`            | you          | bash (if both above are missing), `sh`, `dash`, many display managers |
| `/etc/zprofile`, `~/.zprofile` | zsh   | zsh login shell                                       |

Load order details and the login/interactive difference: [[bashrc]].

> [!info] Why `profile.d`?
> Packages can't safely edit `/etc/profile` — your edits and theirs would collide. So `/etc/profile` loops over `/etc/profile.d/*.sh` and each package drops its own small file in there. Same idea as every other `.d` directory in [[etc]].
>
> A typical `/etc/profile` contains:
> ```sh
> if test -d /etc/profile.d/; then
>     for profile in /etc/profile.d/*.sh; do
>         test -r "$profile" && . "$profile"
>     done
>     unset profile
> fi
> ```
> Files run in **alphabetical order**, which is why they are often named `10-foo.sh`, `70-bar.sh`. There are often `.csh` twins for the C shell; ignore them.

> [!warning] fish does not read any of these
> fish is not a POSIX shell, so `/etc/profile` and `~/.profile` are ignored. Variables still reach fish if a parent process (the display manager or systemd) exported them. To set things for fish, use [[fish-config]].

# `PATH`
`PATH` is a `:`-separated list of directories. When you type `ls`, the shell searches them **left to right** and runs the first `ls` it finds.

```bash
echo "$PATH" | tr ':' '\n'      # one directory per line
type -a python3                 # every match, in PATH order
command -v python3              # the one that will run (scripts: use this, not which)
hash -r                         # forget cached locations after installing something
```

Common directories:

| Directory        | Contains                                       |
| ---------------- | ---------------------------------------------- |
| `~/.local/bin`   | your own scripts and per-user installs         |
| `/usr/local/bin` | software installed by hand for all users → [[usr]] |
| `/usr/bin`       | software from the distribution → [[bin]]       |
| `/usr/sbin`      | admin tools (merged into `/usr/bin` on some distros) → [[sbin]] |

> [!tip] Front or back?
> - `PATH="$HOME/.local/bin:$PATH"` → your version **wins** over the system one.
> - `PATH="$PATH:$HOME/.local/bin"` → the system version wins, yours is a fallback.
>
> Putting a world-writable directory (or `.`) at the front is a security hole: anyone could drop a fake `ls` there.

### Adding to PATH without duplicates
```sh
# in ~/.profile (POSIX sh syntax, so every shell can read it)
path_prepend() {
    case ":$PATH:" in
        *":$1:"*) ;;                       # already there
        *) PATH="$1${PATH:+:$PATH}" ;;
    esac
}
path_prepend "$HOME/.local/bin"
export PATH
```
Many distros' `/etc/profile` define a similar `append_path` helper and then `unset` it at the end, so you can't use theirs from your own files.

# A sensible `~/.profile`
```sh
# ~/.profile — read once at login by sh/bash/zsh(sh mode) and many display managers
export EDITOR=nvim
export VISUAL=nvim
export PAGER=less

case ":$PATH:" in *":$HOME/.local/bin:"*) ;; *) PATH="$HOME/.local/bin:$PATH" ;; esac
export PATH

# bash login shells skip ~/.bashrc unless told; .bash_profile usually does this.
# If you only keep ~/.profile, add:
[ -n "$BASH_VERSION" ] && [ -f "$HOME/.bashrc" ] && . "$HOME/.bashrc"
```

> [!important] Keep `~/.profile` POSIX
> No `[[ ]]`, no arrays, no `source` (use `.`). It may be read by `sh`/`dash`, where bash syntax is an error and can **break your graphical login**.

### Changes don't apply until next login
Edits to profile files take effect when you **log out and back in**. To test in the current shell: `. ~/.profile`.

> [!note] Desktop sessions
> Whether a graphical session reads `~/.profile` depends on the display manager. With systemd, `~/.config/environment.d/*.conf` (`KEY=value` lines) is the reliable place for variables that GUI apps must see. Check with `systemctl --user show-environment`. See [[systemd]].

# Scripting recipes

### Show which profile files exist
```bash
for f in /etc/profile /etc/profile.d/*.sh ~/.bash_profile ~/.bash_login ~/.profile ~/.zprofile; do
  [[ -e $f ]] && echo "$f"
done
```

### Which `/etc/profile.d` file sets a variable?
```bash
grep -l 'JAVA_HOME' /etc/profile.d/*.sh
```

### Remove duplicates from PATH
```bash
PATH=$(printf '%s' "$PATH" | tr ':' '\n' | awk '!seen[$0]++' | paste -sd:)
```

### See the environment a process really got
```bash
tr '\0' '\n' < /proc/$pid/environ | grep ^PATH=
```
→ [[proc]]

### Run something with a clean environment
```bash
env -i HOME="$HOME" PATH=/usr/bin:/bin bash -l -c 'echo $PATH'
```

# See also
- [[bashrc]] — interactive shell config and full load-order table
- [[fish-config]] — fish equivalent
- [[skel]] — default `.bash_profile` / `.profile` for new users
- [[etc]], [[usr]], [[bin]], [[home]]
- [[files]] — all file notes
- `man bash` → **INVOCATION**; `man 7 environ`
