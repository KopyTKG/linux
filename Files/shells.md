---
tags:
  - linux
  - files
  - etc
  - shell
  - users
aliases:
  - /etc/shells
---
`/etc/shells` lists the **valid login shells**, one full path per line. `chsh` only lets normal users pick a shell from this list, and some services (FTP servers, `pam_shells`) refuse users whose shell isn't on it.

> [!info] Quick facts
> - Path: `/etc/shells` · mode `0644`
> - Package scripts add their shell when you install bash, zsh, fish...
> - Manual: `man 5 shells`, `man 1 chsh`

# Example
```
/bin/sh
/bin/bash
/bin/rbash
/usr/bin/sh
/usr/bin/bash
/usr/bin/rbash
/usr/bin/systemd-home-fallback-shell
/usr/bin/git-shell
/usr/bin/fish
/bin/fish
/bin/zsh
/usr/bin/zsh
```
Each shell appears twice (`/bin` and `/usr/bin`) because `/bin` is a symlink to `/usr/bin` → [[bin]]. Both spellings must be listed, since [[passwd]] can use either.

# Notable entries

| Shell                        | What it is                                                            |
| ---------------------------- | --------------------------------------------------------------------- |
| `sh`                         | POSIX shell. Usually a link to `bash` or `dash`. Use for portable scripts |
| `bash`                       | Default on most distros                                               |
| `rbash`                      | Restricted bash: no `cd`, no changing `PATH`, no `/` in commands      |
| `zsh`, `fish`                | Interactive shells with nicer features                                |
| `git-shell`                  | Allows only git push/pull over SSH. For git-only accounts             |
| `nologin` / `false`          | **Not listed** on purpose: accounts with them can't log in            |

> [!warning] fish is not POSIX
> Scripts with `#!/bin/sh` or `#!/usr/bin/env bash` are unaffected by your login shell. But snippets pasted into a fish prompt that use `$(...)`, `export X=y`, `&&` chains in old fish, or `${var}` may fail. Run them with `bash -c '...'`.

# Changing your shell
```bash
chsh -l                       # list (reads /etc/shells), util-linux version
chsh -s /usr/bin/fish         # change your own (asks your password)
sudo chsh -s /bin/bash alice  # change someone else's
```
The new shell applies at next login. Check with `getent passwd "$USER" | cut -d: -f7`.

> [!tip] Adding a shell installed by hand
> A shell built from source or in `/usr/local/bin` isn't listed, so `chsh` refuses it. Add the path: `echo /usr/local/bin/nu | sudo tee -a /etc/shells`.

# Scripting recipes

### Valid shells only (skip comments)
```bash
grep '^/' /etc/shells
```

### Listed shells that are not installed
```bash
grep '^/' /etc/shells | while read -r s; do [[ -x $s ]] || echo "missing: $s"; done
```

### My login shell vs the shell I'm running now
```bash
getent passwd "$USER" | cut -d: -f7     # login shell
ps -p $$ -o comm=                       # current shell
```

### Users whose shell is not valid
```bash
awk -F: 'NR==FNR {ok[$0]=1; next} $3>=1000 && $3<60000 && !($7 in ok) {print $1, $7}' /etc/shells /etc/passwd
```

# See also
- [[passwd]] — field 7 is the login shell
- [[bashrc]] · [[profile]] · [[fish-config]] — what each shell reads at start
- [[bin]] · [[etc]]
