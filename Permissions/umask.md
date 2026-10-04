---
tags:
  - linux
  - permissions
aliases:
  - umask
  - Default permissions
---
Why does `touch file` give `644` and `mkdir dir` give `755`? Because of the **umask**: a per-process mask of bits that are **removed** from every new file and directory.

```
$ umask
0022
$ umask -S
u=rwx,g=rx,o=rx
```

# How it works

Programs ask for a mode when they create something. Almost all ask for:
- files: `666` (`rw-rw-rw-`) — no `x`, because data files shouldn't be executable
- directories: `777` (`rwxrwxrwx`)

The kernel then does `final = requested & ~umask`: every bit set in the umask is **cleared**.

```
  requested file   666   rw- rw- rw-
  umask            022   --- -w- -w-
  ─────────────────────────────────
  result           644   rw- r-- r--

  requested dir    777   rwx rwx rwx
  umask            022   --- -w- -w-
  ─────────────────────────────────
  result           755   rwx r-x r-x
```

> [!warning] It's not subtraction
> umask clears bits, it does **not** subtract numbers. With umask `033`: `666 − 033 = 633` would be wrong; the real result is `644`, because `x` was never requested for files.

# Common values

Tested in the scratchpad (`(umask X; touch f; mkdir d)`):

| umask | New file | New dir | Use case                                              |
| ----- | -------- | ------- | ----------------------------------------------------- |
| `022` | `644`    | `755`   | Default: everyone may read                            |
| `002` | `664`    | `775`   | Shared groups: group may write (user private groups) |
| `027` | `640`    | `750`   | Group may read, others nothing                        |
| `077` | `600`    | `700`   | Private: only you (good for secrets, root)            |

> [!tip] User private groups
> Most distros give each user their **own** primary group (`alice:alice`). Then `002` is safe, because "group" is just you. `/etc/login.defs` has `USERGROUPS_ENAB yes` for this scheme → see [[users]].

# Where it is set

umask is **inherited** from parent to child process, like the environment. Whoever sets it first in the chain wins until someone changes it.

| Where                          | What                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| Kernel                         | PID 1 starts with `0000` (`grep Umask /proc/1/status`)        |
| systemd units                  | `UMask=` in the unit, default `0022` (`systemctl show -p UMask user@1000.service` → `UMask=0022`) |
| PAM `pam_umask`                | Sets umask at login, reads `UMASK` from `/etc/login.defs` (here `022`) |
| `/etc/login.defs`              | `UMASK 022`, `HOME_MODE 0700` (mode of new home dirs) → [[etc]] |
| `/etc/profile`, shell rc files | `umask 077` lines override it for shells                     |
| A program itself               | `umask()` syscall, e.g. services that write secrets           |

See current umask of any process:
```bash
grep Umask /proc/self/status     # Umask:  0022
grep Umask /proc/<pid>/status
```
→ [[proc]]

# Changing it

```bash
umask             # show (octal)
umask -S          # show (symbolic)
umask 077         # set for this shell and its children
umask u=rwx,g=,o= # same as 077, symbolic
```

Permanent per user: put `umask 077` in `~/.profile` / `~/.bashrc`, or in fish `~/.config/fish/config.fish` (fish has `umask` as a built-in function too).

> [!note] umask only affects **new** files
> Existing files keep their mode. Use `chmod` for those → [[file-permissions]].
> Programs can also ignore it by calling `chmod()` after creating a file (e.g. `ssh-keygen` always makes private keys `600`). Default [[acl]]s on the parent directory **replace** the umask for files created inside it.

# Scripting recipes

### Create a private file safely
```bash
( umask 077; printf '%s\n' "$secret" > token.txt )   # born as 600, never readable by others
```
Doing `echo > file; chmod 600 file` has a short window where the file is `644`.

### Private temp directory
```bash
tmp=$( umask 077; mktemp -d )
trap 'rm -rf "$tmp"' EXIT
```
(`mktemp` already uses `600`/`700`, the umask is extra safety.) → [[tmp]]

### Compute result for a umask
```bash
um=027
printf 'file %03o  dir %03o\n' $(( 0666 & ~0$um )) $(( 0777 & ~0$um ))
# file 640  dir 750
```

# Find stuff

| Question                         | Command                                   |
| -------------------------------- | ----------------------------------------- |
| My umask                         | `umask`, `umask -S`                       |
| umask of a process               | `grep Umask /proc/<pid>/status`           |
| System default                   | `grep -E '^(UMASK|HOME_MODE)' /etc/login.defs` |
| Set in shell startup files       | `grep -rn umask /etc/profile* ~/.profile ~/.bashrc 2>/dev/null` |
| umask of a systemd service       | `systemctl show -p UMask <unit>`          |

# See also
- [[file-permissions]] — what the numbers mean
- [[acl]] — default ACLs override the umask
- [[users]] — user private groups
- `man 2 umask`, `man 8 pam_umask`, `man 5 login.defs`
