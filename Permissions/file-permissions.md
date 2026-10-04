---
tags:
  - linux
  - permissions
  - chmod
aliases:
  - rwx
  - chmod
  - chown
  - File modes
---
Every file and directory has an **owner**, a **group** and a **mode**: 3 × 3 bits saying what the owner, the group and everyone else may do. This is the base layer of [[permissions]].

# Reading `ls -l`

```
-rw-r--r-- 1 root root 2327  9. čec 19.55 /etc/passwd
drwxrwxrwt 22 root root 580 Oct  4 17:52 /tmp
-rwsr-xr-x 1 root root 285880 17. čen 12.46 /usr/bin/sudo
```

```
 -   rw-   r--   r--   1   root   root   2327 ...  /etc/passwd
 │   │     │     │     │   │      │
 │   │     │     │     │   │      └─ group
 │   │     │     │     │   └──────── owner (user)
 │   │     │     │     └──────────── hard link count
 │   │     │     └────────────────── other  (everyone else)
 │   │     └──────────────────────── group
 │   └────────────────────────────── owner
 └────────────────────────────────── file type
```

| Type char | Meaning                    |
| --------- | -------------------------- |
| `-`       | Regular file               |
| `d`       | Directory                  |
| `l`       | Symbolic link              |
| `c`       | Character device → [[dev]] |
| `b`       | Block device → [[dev]]     |
| `p`       | Named pipe (FIFO)          |
| `s`       | Socket                     |

A `+` after the mode (`-rw-r--r--+`) means the file has an [[acl]]. A `.` means an SELinux label → [[mac-selinux-apparmor]].

# What `r`, `w`, `x` mean

> [!important] Same letters, different meaning on directories
> A directory is a list of names → inode numbers. `r`, `w`, `x` act on **that list**, not on the files inside.

| Bit | On a file                       | On a directory                                                      |
| --- | ------------------------------- | ------------------------------------------------------------------- |
| `r` | Read content                    | **List** names (`ls`)                                               |
| `w` | Change content                  | **Create, delete, rename** entries (needs `x` too)                  |
| `x` | Execute as program / script     | **Enter / traverse**: `cd`, and access files inside by name         |

Tested on this machine:

| Directory mode | `ls dir`            | `cat dir/file` (file is readable) |
| -------------- | ------------------- | --------------------------------- |
| `r--` (`600`*) | names listed        | **Permission denied**             |
| `--x` (`100`)  | **Permission denied** | works, if you know the name    |
| `r-x` (`500`)  | works               | works                             |

\* For the owner. `600` = `rw-` but without `x` the `w` is useless.

> [!warning] Deleting depends on the directory, not the file
> `rm` changes the **directory**. A read-only file (`444`) inside a directory you can write to can still be deleted (`rm -f`).
> To protect files in a shared writable dir, use the sticky bit → [[special-bits]].

Scripts need `r` **and** `x` (the interpreter has to read them). Binaries need only `x`.

# Octal notation

Each class is 3 bits: `r=4`, `w=2`, `x=1`. Add them up.

| Octal | Bits  | Typical use                                 |
| ----- | ----- | ------------------------------------------- |
| `7`   | `rwx` | owner of a dir / executable                 |
| `6`   | `rw-` | owner of a normal file                      |
| `5`   | `r-x` | others on dirs and programs                 |
| `4`   | `r--` | read-only                                   |
| `0`   | `---` | nothing                                     |

| Mode   | Symbolic     | Example on this machine                    |
| ------ | ------------ | ------------------------------------------ |
| `644`  | `rw-r--r--`  | `/etc/passwd` → [[passwd]]                 |
| `600`  | `rw-------`  | `/etc/shadow` → [[shadow]]                 |
| `440`  | `r--r-----`  | `/etc/sudoers` → [[sudoers]]               |
| `755`  | `rwxr-xr-x`  | `/usr/bin/*`, most directories             |
| `700`  | `rwx------`  | `/home/alice`, `/run/user/1000` → [[home]]  |
| `750`  | `rwxr-x---`  | `/root`                                    |
| `1777` | `rwxrwxrwt`  | `/tmp`, `/var/tmp`, `/dev/shm` → [[tmp]]   |
| `4755` | `rwsr-xr-x`  | `/usr/bin/sudo`, `/usr/bin/passwd`         |

A 4th leading digit holds the special bits: `4` setuid, `2` setgid, `1` sticky → [[special-bits]].

# `chmod`

```bash
chmod 640 file             # absolute: rw-r-----
chmod u+x script.sh        # add x for owner
chmod go-w file            # remove w from group and other
chmod a+r file             # a = all (u+g+o)
chmod u=rw,g=r,o= file     # set exactly
chmod -R u+rwX dir         # capital X: x only on dirs (and files already executable)
```

| Who | `u` owner · `g` group · `o` other · `a` all |
| --- | -------------------------------------------- |
| Op  | `+` add · `-` remove · `=` set exactly       |
| What | `r` `w` `x` `X` `s` `t`                     |

Real run:
```
chmod 640 f        → -rw-r----- 640
chmod u+x,o+r f    → -rwxr--r-- 744
chmod a-x f        → -rw-r--r-- 644
```

> [!tip] `chmod -R 755` is usually wrong
> It makes every file executable. Use `X`, or `find`:
> ```bash
> find dir -type d -exec chmod 755 {} +
> find dir -type f -exec chmod 644 {} +
> ```

# `chown` / `chgrp`

```bash
chown alice file           # change owner (root only)
chown alice:devs file      # owner and group
chown :devs file           # only group
chgrp devs file            # same as above
chown -R alice:alice dir   # recursive
chown --reference=a b      # copy owner/group from a
```

> [!note] Who may change what
> - Changing the **owner** needs root (`CAP_CHOWN`), otherwise users could give away files to dodge quotas.
> - The owner may change the **group** to any group they are a member of.
> - `chown` by a normal user clears setuid/setgid bits for safety.

# `stat`

```bash
stat -c '%A %a %U %G %n' /etc/passwd /etc/shadow /tmp /usr/bin/sudo
```
```
-rw-r--r-- 644 root root /etc/passwd
-rw------- 600 root root /etc/shadow
drwxrwxrwt 1777 root root /tmp
-rwsr-xr-x 4755 root root /usr/bin/sudo
```

| Format | Meaning               |
| ------ | --------------------- |
| `%A`   | Symbolic mode         |
| `%a`   | Octal mode            |
| `%U` / `%u` | Owner name / UID |
| `%G` / `%g` | Group name / GID |
| `%n`   | File name             |
| `%F`   | File type in words    |

`ls -ln` shows numeric UID/GID — useful when a file is owned by a deleted user (it then shows only a number).

# Scripting recipes

### Get octal mode of a file
```bash
mode=$(stat -c '%a' "$file")
[[ $mode == 600 ]] || echo "warning: $file has mode $mode"
```

### Find world-writable files (not symlinks, not sticky dirs)
```bash
find /home /etc -xdev -type f -perm -o+w 2>/dev/null
```

### Find files not owned by any existing user
```bash
find /home -xdev \( -nouser -o -nogroup \) 2>/dev/null
```

### Fix a private directory (e.g. `~/.ssh`)
```bash
chmod 700 ~/.ssh
find ~/.ssh -type f -name 'id_*' ! -name '*.pub' -exec chmod 600 {} +
```

### Files owned by me that others can read
```bash
find ~ -xdev -user "$USER" -type f -perm -o+r 2>/dev/null | head
```

### `find -perm` cheat sheet
| Expression     | Matches                                 |
| -------------- | --------------------------------------- |
| `-perm 644`    | Mode is **exactly** 644                 |
| `-perm -644`   | **All** these bits are set (at least)   |
| `-perm /022`   | **Any** of these bits is set            |

# Find stuff

| Question                         | Command                         |
| -------------------------------- | ------------------------------- |
| Mode / owner of a file           | `stat -c '%A %a %U %G' file`    |
| Numeric owner                    | `ls -ln file`                   |
| Every step of a path             | `namei -l /path`                |
| Files with a given mode          | `find dir -perm 777`            |
| Who owns most files in a dir     | `find dir -printf '%u\n' \| sort \| uniq -c \| sort -rn` |

# See also
- [[permissions]] — overview and the access check order
- [[special-bits]] — the 4th digit (setuid, setgid, sticky)
- [[umask]] — why new files get `644`
- [[acl]] — more than one owner/group
- `man 1 chmod`, `man 1 chown`, `man 1 stat`, `man 7 inode`
