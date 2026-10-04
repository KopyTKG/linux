---
tags:
  - linux
  - permissions
  - security
aliases:
  - Permissions
  - Access control
  - Linux permissions
---
Every time a process opens a file, lists a directory, sends a signal or binds a port, the kernel asks one question: **is this process allowed to do this?** This note explains how the kernel answers that question and links to the detail notes.

> [!info] Permissions belong to processes, not to people
> The kernel never sees "the user alice". It sees a **process** with a set of credentials:
> - **UID** and **GID** (real, effective, saved, filesystem)
> - **supplementary groups**
> - **capabilities**
> - optionally a **security label** (SELinux / AppArmor / ...)
>
> When you log in, your shell gets your UID and groups, and every program you start inherits them. Check yours with `id`, or look at `Uid:`, `Gid:`, `Groups:` and `Cap*` in `/proc/self/status` → see [[proc]].

```
$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice),7(lp),10(wheel),29(audio),44(video),999(docker),...
```

# How the kernel decides (files)

The classic Unix check runs in this order, and **stops at the first class that matches**:

```
        process wants to read/write/execute a file
                          │
            is effective UID 0 (root)  ──yes──▶ allowed*  (via capabilities)
                          │ no
          does UID == file owner?  ──yes──▶ use OWNER bits  (rwx------)
                          │ no
   is file group in process groups? ──yes──▶ use GROUP bits (---rwx---)
                          │ no
                          ▼
                    use OTHER bits (------rwx)
            ▼
   then ACLs / MAC (SELinux, AppArmor, Landlock) may still deny
```

\* Root is not magic. It is allowed because a root process holds **capabilities** like `CAP_DAC_OVERRIDE` (ignore rwx) and `CAP_FOWNER` (act as owner). Remove those and root is denied too → see [[capabilities]].

> [!warning] Only ONE class is used
> If you are the owner, **only** the owner bits count, even if "other" has more rights.
> A file with mode `---r--r--` (`044`) can be read by everyone **except its owner**.
> ```bash
> touch x; chmod 044 x; cat x   # Permission denied — you are the owner
> ```

### The layers

| Layer                         | What it controls                                  | Note                       |
| ----------------------------- | ------------------------------------------------- | -------------------------- |
| Owner / group / other `rwx`   | Basic file access                                 | [[file-permissions]]       |
| setuid / setgid / sticky      | Run as owner, group inheritance, safe shared dirs | [[special-bits]]           |
| umask                         | Default permissions of new files                  | [[umask]]                  |
| ACLs                          | Extra users/groups per file                       | [[acl]]                    |
| File attributes               | Immutable, append-only, no copy-on-write          | [[attributes]]             |
| Capabilities                  | Root's power split into ~41 pieces                | [[capabilities]]           |
| MAC (SELinux, AppArmor, ...)  | System-wide policy that even root must follow     | [[mac-selinux-apparmor]]   |

Order of evaluation for an `open()`: **DAC** (rwx + ACL, overridable by capabilities) first, then **LSM/MAC** hooks. Both must say yes.

> [!tip] DAC vs MAC
> - **DAC** (Discretionary Access Control) — the *owner* decides who gets access (`chmod`, `setfacl`).
> - **MAC** (Mandatory Access Control) — the *system policy* decides, and the owner can't change it.

### Where identities come from

| Thing                  | Stored in                     | Note        |
| ---------------------- | ----------------------------- | ----------- |
| Users, UIDs, home, shell | `/etc/passwd`               | [[passwd]]  |
| Password hashes        | `/etc/shadow`                 | [[shadow]]  |
| Groups and members     | `/etc/group`                  | [[group]]   |
| Who may use `sudo`     | `/etc/sudoers`                | [[sudoers]] |
| Overview of users      | —                             | [[users]]   |

> [!note] Group membership is read at login
> Adding yourself to a group (`usermod -aG`) does **not** affect running processes. Their group list was copied at login. Log out and back in (or start a new login shell with `su - $USER` / `newgrp group`).

# Not only files

The same UID/capability idea controls other things:

| Action                         | Rule                                                              |
| ------------------------------ | ----------------------------------------------------------------- |
| `kill` a process               | Same real/effective UID as target, or `CAP_KILL`                  |
| Read `/proc/<pid>/environ`, `fd/` | Same UID (and allowed by ptrace rules), or root                |
| Attach debugger (`ptrace`)     | Same UID + `/proc/sys/kernel/yama/ptrace_scope` (here: `1` = only descendants) |
| Bind port < 1024               | `CAP_NET_BIND_SERVICE` (limit in `/proc/sys/net/ipv4/ip_unprivileged_port_start`, here `1024`) |
| Device access (`/dev/*`)       | Normal file permissions on the device node → see [[dev]]          |

```
crw-rw-rw- 1 root root   1, 3  /dev/null      # everyone can read/write
brw-rw---- 1 root disk 259, 1  /dev/nvme0n1   # only root and group "disk"
```

### Kernel hardening switches
Live in `/proc/sys/fs/` (all `1` on this machine):

| File                    | When `1`                                                                      |
| ----------------------- | ----------------------------------------------------------------------------- |
| `protected_symlinks`    | In sticky world-writable dirs (`/tmp`), symlinks are followed only if owner matches |
| `protected_hardlinks`   | You can't hardlink files you don't own / can't read+write                     |
| `protected_regular`     | Restricts `O_CREAT` opening of other users' files in sticky dirs              |

# Scripting recipes

### Can I read / write / execute this?
```bash
f=/etc/shadow
[[ -r $f ]] && echo readable   || echo "not readable"
[[ -w $f ]] && echo writable   || echo "not writable"
[[ -x $f ]] && echo executable || echo "not executable"
```
These tests ask the kernel (`access()`), so they include ACLs and root.

### Am I root?
```bash
if (( EUID != 0 )); then
  echo "Run me with sudo" >&2
  exit 1
fi
```

### Am I in group X?
```bash
in_group() { id -nG | tr ' ' '\n' | grep -qx "$1"; }
in_group wheel && echo "yes, in wheel"
```

### Explain why a path is not accessible
Every directory on the way needs `x`. `namei -l` shows each step:
```bash
namei -l /home/alice/notes/README.md
```
```
f: /home/alice/notes/README.md
dr-xr-xr-x root root /
drwxr-xr-x root root home
drwx------ alice alice alice        ← other users stop here
drwxr-xr-x alice alice notes
-rw-r--r-- alice alice README.md
```

# Find stuff

| Question                              | Command                                         |
| ------------------------------------- | ----------------------------------------------- |
| Who am I, which groups                | `id`, `groups`                                  |
| Permissions of a file                 | `ls -l`, `stat -c '%A %a %U %G' file`           |
| Permissions along a path              | `namei -l /path`                                |
| Credentials of a running process      | `grep -E 'Uid|Gid|Groups|Cap' /proc/<pid>/status` |
| setuid binaries                       | `find / -xdev -perm -4000 -type f 2>/dev/null`  |
| Files with capabilities               | `getcap -r /usr 2>/dev/null`                    |
| Files with ACLs                       | `getfacl -R -s /path 2>/dev/null`               |
| Active security modules               | `cat /sys/kernel/security/lsm`                  |

# See also
- [[file-permissions]] · [[special-bits]] · [[umask]] · [[acl]] · [[attributes]] · [[capabilities]] · [[mac-selinux-apparmor]]
- [[users]] — users, groups, sudo
- [[proc]] — `/proc/<pid>/status` shows credentials
- `man 7 credentials`, `man 7 capabilities`, `man 2 access`, `man 7 path_resolution`
