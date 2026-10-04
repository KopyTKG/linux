---
tags:
  - linux
  - permissions
  - security
aliases:
  - setuid
  - setgid
  - sticky bit
  - SUID
  - SGID
---
Besides `rwx` there are three **special bits**. They form the extra leading octal digit (`4755`, `2775`, `1777`) and show up as `s`, `s` and `t` in `ls -l`.

| Bit      | Octal | Shown as                  | On a file                          | On a directory                                   |
| -------- | ----- | ------------------------- | ---------------------------------- | ------------------------------------------------ |
| setuid   | `4`   | `s` in owner `x` slot     | Runs with the **owner's** UID      | Ignored on Linux                                 |
| setgid   | `2`   | `s` in group `x` slot     | Runs with the **file's group**     | New entries **inherit the directory's group**    |
| sticky   | `1`   | `t` in other `x` slot     | Ignored on Linux                   | Only the **owner** may delete/rename entries     |

> [!note] Capital `S` / `T`
> Lowercase `s`/`t` = special bit **and** `x` are set. Capital `S`/`T` = special bit set but `x` is **not** — usually a mistake.
> `-rwSr--r--` → setuid on a non-executable file, does nothing.

# setuid — run as the file's owner

When an executable with setuid is started, the process's **effective UID** becomes the file's owner. Owner `root` → the program runs as root, no matter who started it.

That is how a normal user can change their password: `/etc/shadow` is `600 root` ([[shadow]]), but `passwd` is setuid root.

```
-rwsr-xr-x 1 root root  88984 /usr/bin/passwd
-rwsr-xr-x 1 root root 285880 /usr/bin/sudo
```

```bash
grep -E '^(Uid|Gid)' /proc/self/status
# Uid:  real  effective  saved  filesystem
```
In a setuid program `real` = you, `effective` = owner. The program can drop privileges by setting effective back to real → see [[capabilities]].

setuid binaries on this machine (`find /usr/bin /usr/lib -xdev -perm -4000 -type f`):

| Binary                | Why it needs root                          |
| --------------------- | ------------------------------------------ |
| `sudo`, `su`, `pkexec`| Switch user → [[users]]                    |
| `passwd`, `chage`, `chsh`, `chfn`, `gpasswd`, `newgrp` | Edit [[passwd]], [[shadow]], [[group]] |
| `unix_chkpwd`         | Let screen lockers check your password     |
| `mount`, `umount`, `fusermount3` | Mount filesystems allowed by `user` option in fstab |
| `dbus-daemon-launch-helper` | Start system D-Bus services          |

> [!warning] setuid is dangerous
> Every setuid-root binary is a possible root exploit: a bug in it runs **as root**.
> - The kernel **ignores setuid on scripts** (`#!/bin/bash`) because of race conditions. Only real binaries work.
> - Filesystems mounted with `nosuid` ignore the bit (common for `/tmp`, `/home`, USB drives).
> - Modern systems replace setuid with **file capabilities** where possible. Example: `ping` here is plain `755` with no capability; the kernel allows unprivileged ICMP sockets via `/proc/sys/net/ipv4/ping_group_range` (`0 2147483647` = all groups).
> - A setuid file is cleared of the bit when a non-root user writes to it or `chown`s it.

# setgid

### On a file: run as the file's group
```
-rwxr-sr-x 1 root tty 22688 /usr/bin/wall
```
`wall` runs with group `tty`, so it can write to every terminal that allows messages, but it gets **no** root power. Terminals are owned `user:tty`; `mesg y` sets them to `620` (group may write), `mesg n` to `600` (here: `crw------- alice tty /dev/pts/0`). Group-only privilege is a safer pattern than setuid root.

setgid binaries here: `wall`, `write` (group `tty`), `utempter` (group `utmp`, writes login records), `plocate` (group `locate`, reads the file-name database).

### On a directory: group inheritance
New files and subdirectories get the **directory's group**, not the creator's primary group. Subdirectories also inherit the setgid bit. This is the standard way to make a **shared team folder**.

Tested in the scratchpad:
```bash
mkdir grp
chgrp audio grp          # a group I'm member of
chmod 2775 grp
touch grp/x; mkdir grp/sub
stat -c '%A %U %G %n' grp grp/x grp/sub
```
```
drwxrwsr-x alice audio grp
-rw-r--r-- alice audio grp/x       ← group inherited
drwxr-sr-x alice audio grp/sub     ← group AND setgid inherited
```
Combine with umask `002` or a default [[acl]] so group members can also **write** the new files.

# Sticky bit — "only delete your own"

On a world-writable directory everyone can delete everyone's files ([[file-permissions]]: delete = write on the dir). The sticky bit adds a rule: an entry can only be removed or renamed by **its owner**, the **directory owner**, or root.

```
drwxrwxrwt 22 root root 580 /tmp
drwxrwxrwt  1 root root 1248 /var/tmp
drwxrwxrwt  2 root root 440 /dev/shm
```

> [!tip] Why `/tmp` is `1777`
> `777` → anyone can create files. `1` → nobody can delete or replace someone else's file. Together with `fs.protected_symlinks=1` this blocks classic `/tmp` symlink attacks → see [[tmp]].

# Setting the bits

```bash
chmod u+s file      # setuid       (4xxx)
chmod g+s dir       # setgid       (2xxx)
chmod +t dir        # sticky       (1xxx)
chmod 2775 dir      # setgid + rwxrwxr-x
chmod 1777 dir      # sticky + rwxrwxrwx
chmod u-s,g-s file  # remove
```

> [!note] Octal `chmod` on directories
> GNU `chmod 755 dir` keeps an existing setgid bit on directories. To clear it you must say so: `chmod g-s dir` or `chmod 00755 dir`.

# Scripting recipes

### List all setuid / setgid files
```bash
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -printf '%M %u:%g %p\n' 2>/dev/null
```

### Baseline and diff (detect new setuid binaries)
```bash
find / -xdev -perm -4000 -type f 2>/dev/null | sort > suid-baseline.txt
# later:
find / -xdev -perm -4000 -type f 2>/dev/null | sort | diff suid-baseline.txt - && echo "no changes"
```

### Which mounts ignore setuid (`nosuid`)
```bash
findmnt -rn -o TARGET,OPTIONS | awk '$2 ~ /nosuid/ {print $1}'
```

### Check if a directory is sticky
```bash
[[ -k /tmp ]] && echo "/tmp is sticky"
[[ -g dir ]]  && echo "dir is setgid"
[[ -u /usr/bin/sudo ]] && echo "sudo is setuid"
```

# Find stuff

| Question                         | Command                                             |
| -------------------------------- | --------------------------------------------------- |
| setuid files                     | `find / -xdev -perm -4000 -type f 2>/dev/null`      |
| setgid files                     | `find / -xdev -perm -2000 -type f 2>/dev/null`      |
| Sticky dirs                      | `find / -xdev -type d -perm -1000 2>/dev/null`      |
| Real vs effective UID of process | `grep -E '^(Uid|Gid)' /proc/<pid>/status`           |
| Mount options (`nosuid`)         | `findmnt -o TARGET,OPTIONS`                         |

# See also
- [[file-permissions]] — the normal `rwx` bits
- [[capabilities]] — the safer replacement for setuid root
- [[tmp]] — sticky `/tmp` in practice
- [[users]] — `sudo` and `su`
- `man 2 execve` (section "set-user-ID"), `man 7 credentials`
