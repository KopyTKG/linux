---
tags:
  - linux
  - users
  - security
aliases:
  - Users
  - Linux users
  - user accounts
---
A **user** on Linux is, for the kernel, nothing more than a **number** (the UID). Every process runs "as" some UID and every file is owned by some UID. Names like `alice` or `root` exist only for humans: programs translate them through [[passwd|/etc/passwd]] (or another user database via NSS).

> [!info] The kernel doesn't know names
> - `ls -l` shows `alice`, but the inode stores `1000`. `ls -n` shows the raw number.
> - Delete a user and their files stay owned by `1000`, which `ls` then prints as a bare number.
> - Copy files to another machine where `1000` is someone else, and that someone else owns them.
> - Containers map UIDs too: UID `0` inside can be UID `100000` outside (`/proc/<pid>/uid_map`).

# Kinds of users

| Kind             | UID (typical)   | Example on this machine                        | Purpose                                   |
| ---------------- | --------------- | ---------------------------------------------- | ----------------------------------------- |
| root             | `0`             | `root:x:0:0::/root:/usr/bin/bash`              | Superuser, bypasses most permission checks |
| System users     | `1`–`999`       | `dbus`, `polkitd`, 37 in total                 | Run daemons with least privilege          |
| Regular users    | `1000`–`60000`  | `alice:x:1000:1000:...:/home/alice:/bin/fish`    | Humans                                    |
| nobody           | `65534`         | `nobody:x:65534:65534:Kernel Overflow User:/:/usr/bin/nologin` | Unmapped / unprivileged placeholder |

Exact ranges come from `/etc/login.defs` → see [[uid-gid]].

> [!tip] System users can't log in
> On this machine 35 of 38 accounts have the shell `/usr/bin/nologin`. They exist only so a service (D-Bus, polkit, ...) can own files and processes without being root.

# What makes up a user

| Piece                   | Where                          | Note                       |
| ----------------------- | ------------------------------ | -------------------------- |
| Name, UID, GID, home, shell | `/etc/passwd`              | [[passwd]]                 |
| Password hash, aging    | `/etc/shadow` (root only)      | [[shadow]]                 |
| Group memberships       | `/etc/group`                   | [[group]], [[groups]]      |
| Home directory          | `/home/<name>`, `/root`        | [[home]], [[root]]         |
| Default files for new homes | `/etc/skel`                | [[skel]]                   |
| How login is checked    | `/etc/pam.d/`                  | [[pam]]                    |
| Who may become root     | `/etc/sudoers`                 | [[sudoers]], [[sudo-su]]   |
| Live sessions           | `loginctl`, `/run/user/<uid>`  | [[login-sessions]]         |

# Notes in this section
- [[uid-gid]] — numbers, ranges, real vs effective IDs, name lookup
- [[user-management]] — `useradd`, `usermod`, `userdel`, `passwd`, `chage`, locking
- [[groups]] — primary vs supplementary groups, common groups and what they grant
- [[sudo-su]] — becoming another user: `su`, `sudo`, `pkexec`, `doas`
- [[login-sessions]] — who is logged in, `loginctl`, ttys, `XDG_RUNTIME_DIR`
- [[pam]] — how authentication is plugged together

# Scripting recipes

### List regular (human) users
```bash
min=$(awk '/^UID_MIN/{print $2}' /etc/login.defs)
max=$(awk '/^UID_MAX/{print $2}' /etc/login.defs)
awk -F: -v min="$min" -v max="$max" '$3>=min && $3<=max {print $1, $3, $6, $7}' /etc/passwd
```
```
alice 1000 /home/alice /bin/fish
```
> [!note] `/etc/passwd` is not always the full list
> With LDAP, SSSD or systemd-homed, users come from elsewhere. `getent passwd` asks every source configured in `/etc/nsswitch.conf`, so prefer it in scripts: `getent passwd | awk -F: ...`.

### Accounts that can log in (real shell)
```bash
awk -F: '$7 !~ /(nologin|false)$/ {print $1, $7}' /etc/passwd
```

### Am I root?
```bash
if [[ $EUID -ne 0 ]]; then
  echo "run me with sudo" >&2
  exit 1
fi
```

### Processes per user
```bash
ps -eo user= | sort | uniq -c | sort -rn
```
```
    405 root
    133 alice
      2 dbus
```

# Find stuff

| Question                       | Command                                  |
| ------------------------------ | ---------------------------------------- |
| Who am I?                      | `whoami`, `id`                           |
| Info about a user              | `getent passwd alice`, `id alice`          |
| Which groups am I in?          | `id -nG`, `groups`                       |
| Who is logged in?              | `who`, `w`, `loginctl list-sessions`     |
| Who logged in before?          | `last`                                   |
| Who owns a file / process?     | `stat -c %U file`, `ps -o user= -p PID`  |
| Files owned by deleted users   | `find / -xdev \( -nouser -o -nogroup \) 2>/dev/null` |

# See also
- [[uid-gid]] · [[user-management]] · [[groups]] · [[sudo-su]] · [[login-sessions]] · [[pam]]
- [[permissions]] — what users and groups are *allowed* to do with files
- [[passwd]] · [[shadow]] · [[group]] · [[gshadow]] · [[sudoers]] · [[etc]]
- `man 5 passwd`, `man 7 credentials`
