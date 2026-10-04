---
tags:
  - linux
  - files
  - etc
  - security
aliases:
  - /etc/pam.d
  - pam.d
  - PAM
---
`/etc/pam.d/` configures **PAM** (Pluggable Authentication Modules). Every program that logs someone in or checks a password (`login`, `sudo`, `su`, `sshd`, display managers, screen lockers) asks PAM, and PAM follows the file in this folder named after the program.

> [!info] Quick facts
> - Path: `/etc/pam.d/` · one file per service (`sudo`, `login`, `sshd`, `su`, ...)
> - Fallback file: `other` (used when a program has no own file; normally denies everything)
> - Modules (`pam_unix.so` etc.) live in `/usr/lib/security/` or `/lib/x86_64-linux-gnu/security/`
> - Manual: `man 5 pam.d`, `man 8 pam_unix`, `man pam_<name>`

> [!danger] You can lock yourself out
> A mistake in `system-auth`, `common-auth` or `login` can make **all** logins and `sudo` fail. Before editing, open a **second root shell** (`sudo -i`) and keep it open until you tested a new login in another terminal.

# How a PAM file is built
Each line is: `type  control  module  [arguments]`

```
auth       required    pam_faillock.so  preauth
auth       [success=1 default=bad]  pam_unix.so  try_first_pass nullok
account    required    pam_unix.so
password   required    pam_unix.so  try_first_pass nullok shadow
session    required    pam_limits.so
```

### Types (the 4 stacks)
| Type       | Question it answers                                   |
| ---------- | ----------------------------------------------------- |
| `auth`     | Are you who you say? (password, key, fingerprint)     |
| `account`  | Are you allowed now? (expired, time of day, locked)   |
| `password` | How to **change** the password                        |
| `session`  | Setup/cleanup around the session (limits, env, logs)  |

### Controls
| Control       | Meaning                                                              |
| ------------- | -------------------------------------------------------------------- |
| `required`    | Must succeed. On failure keep going (hide which step failed), then fail |
| `requisite`   | Must succeed. On failure stop **immediately**                        |
| `sufficient`  | If it succeeds (and nothing required failed before), done → success  |
| `optional`    | Result only matters if it is the only module                         |
| `include x`   | Paste all lines of the same type from file `x`                       |
| `substack x`  | Like include, but `x`'s result is evaluated as one step              |
| `[success=N default=bad]` | Advanced form: on success skip next N lines              |

A `-` before the type (`-auth`) means: skip silently if the module isn't installed.

# Shared base files
Programs rarely list modules themselves. They `include` a common file:

| Distro family      | Shared files                                                     |
| ------------------ | ---------------------------------------------------------------- |
| Arch, Fedora, RHEL | `system-auth`, `system-login`, `password-auth`                   |
| Debian, Ubuntu     | `common-auth`, `common-account`, `common-password`, `common-session` |

Example `/etc/pam.d/sudo`:
```
#%PAM-1.0
auth      include   system-auth
account   include   system-auth
session   include   system-auth
```

# Modules worth knowing

| Module             | What it does                                                    |
| ------------------ | --------------------------------------------------------------- |
| `pam_unix.so`      | Classic check against [[passwd]] / [[shadow]]                   |
| `pam_faillock.so`  | Lock account after N wrong passwords (`faillock --user alice` to see/reset) |
| `pam_limits.so`    | Apply `/etc/security/limits.conf` (`ulimit` values)             |
| `pam_env.so`       | Set variables from [[environment]] and `/etc/security/pam_env.conf` |
| `pam_systemd.so`   | Register session with `systemd-logind`, create `/run/user/UID`  |
| `pam_time.so`      | Allow login only at certain times (`/etc/security/time.conf`)   |
| `pam_nologin.so`   | Block non-root login while `/etc/nologin` exists                |
| `pam_wheel.so`     | Only `wheel` members may `su`                                   |
| `pam_pwquality.so` | Password strength rules                                         |
| `pam_securetty.so` | Root only on terminals listed in `/etc/securetty`               |
| `pam_permit.so` / `pam_deny.so` | Always succeed / always fail                       |

> [!tip] `/etc/security/`
> Many modules read their settings from `/etc/security/` (`limits.conf`, `faillock.conf`, `time.conf`, `pam_env.conf`, `access.conf`). PAM files say **which** modules run; `/etc/security` says **how** they behave.

# Scripting recipes

### Services that have a PAM file
```bash
ls /etc/pam.d
```

### Which modules does sudo end up using (follows includes one level)
```bash
f=/etc/pam.d/sudo
grep -vE '^\s*(#|$)' "$f"
awk '$2=="include"||$2=="substack" {print $3}' "$f" | sort -u | while read -r inc; do
  echo "--- $inc"; grep -vE '^\s*(#|$)' "/etc/pam.d/$inc"
done
```

### Is faillock enabled? Who is locked?
```bash
grep -l pam_faillock /etc/pam.d/* 2>/dev/null
faillock --user "$USER"
```

### Current limits PAM gave your session
```bash
ulimit -a
```

# See also
- [[shadow]] · [[passwd]] — what `pam_unix` checks against
- [[sudoers]] · [[login-defs]] · [[environment]]
- [[ssh-config]] — `UsePAM yes` in `sshd_config`
- [[etc]] · [[systemd]]
