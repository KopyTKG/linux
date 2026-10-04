---
tags:
  - linux
  - files
  - etc
  - shell
aliases:
  - /etc/environment
  - environment file
  - environment.d
---
`/etc/environment` sets **environment variables for every login**, whatever shell the user runs. It is read by PAM (`pam_env`), not by a shell, so it works the same for bash, fish, zsh, graphical logins and SSH.

> [!info] Quick facts
> - Path: `/etc/environment` · mode `0644`
> - Format: plain `KEY=value` lines. **Not a shell script**
> - Read at login. Change it, then log out and in again
> - Typical content: empty, or a few lines like `EDITOR=nano`
> - Manual: `man 8 pam_env`, `man 5 environment.d`

# What works and what doesn't

```
EDITOR=nvim                      ✔ works
JAVA_HOME="/usr/lib/jvm/default" ✔ quotes are stripped
PATH=$PATH:/opt/foo/bin          ✘ no $variable expansion
export EDITOR=nvim               ✘ no "export" (it's not a shell)
alias ll='ls -l'                 ✘ no aliases or commands
```

> [!warning] Don't put `PATH` here carelessly
> Because `$PATH` isn't expanded, `PATH=...` here **replaces** the whole `PATH`. Write the full value or set `PATH` in [[profile]] instead.

# All the places variables come from
From earliest to latest; later ones win.

| Where                                   | Scope                        | Syntax                    |
| --------------------------------------- | ---------------------------- | ------------------------- |
| `/etc/environment`                      | All users, all logins (PAM)  | `KEY=value`               |
| `/etc/security/pam_env.conf`            | All users (PAM), advanced    | `KEY DEFAULT=... OVERRIDE=...`, can use `${HOME}` |
| `/usr/lib/environment.d/*.conf`, `/etc/environment.d/*.conf` | systemd user services + graphical session | `KEY=value`, `$VAR` **is** expanded |
| `~/.config/environment.d/*.conf`        | Your systemd user session    | same                      |
| [[profile]] `/etc/profile`, `/etc/profile.d/*.sh` | Login shells (sh/bash/zsh) | Shell script, `export` |
| `~/.profile`, `~/.bash_profile`          | Your login shells            | Shell script              |
| [[bashrc]] `~/.bashrc`                   | Interactive bash             | Shell script              |
| [[fish-config]] `~/.config/fish/config.fish` | fish                     | `set -gx KEY value`       |
| Command line: `KEY=value cmd`            | One command                  |                           |

> [!tip] Which one should I use?
> - A variable **every** program should see (GUI apps too): `~/.config/environment.d/` (yours) or `/etc/environment` (everyone).
> - A variable only for your terminal: your shell's config file.
> - Needs `$HOME` or `$PATH` expansion: `environment.d` or a shell profile, not `/etc/environment`.

# Scripting recipes

### Variables of the current process / another process
```bash
env | sort
tr '\0' '\n' < /proc/$pid/environ      # what a running program got (see [[proc]])
```

### What the systemd user session sees (GUI apps inherit this)
```bash
systemctl --user show-environment
```

### Read /etc/environment safely into a script (no code execution)
```bash
while IFS='=' read -r k v; do
  [[ $k =~ ^[A-Za-z_][A-Za-z0-9_]*$ ]] || continue
  v=${v%\"}; v=${v#\"}
  export "$k=$v"
done < /etc/environment
```

### Run a program with a clean environment
```bash
env -i HOME="$HOME" PATH=/usr/bin:/bin bash -c 'env'
```

# See also
- [[profile]] · [[bashrc]] · [[fish-config]] — shell-level variables
- [[pam-d]] — `pam_env.so` reads this file
- [[locale-conf]] — `LANG` and `LC_*` come from there
- [[proc-self-highlights]] · [[etc]] · [[home]]
