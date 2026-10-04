---
tags:
  - linux
  - users
  - security
aliases:
  - UID
  - GID
  - user ID
  - group ID
  - effective UID
---
A **UID** (user ID) and a **GID** (group ID) are plain integers. The kernel checks them on every file access, signal and privileged operation. Names are only a lookup on top. See [[users]] for the bigger picture.

# Ranges
Ranges are a **convention**, set in `/etc/login.defs` and used by `useradd` when it picks a free number. The kernel itself only treats `0` specially.

```
UID_MIN			 1000
UID_MAX			60000
SYS_UID_MIN		  500
SYS_UID_MAX		  999
GID_MIN			 1000
GID_MAX			60000
```

| Range            | Meaning                                                               |
| ---------------- | --------------------------------------------------------------------- |
| `0`              | root. Full privileges (really: all capabilities, see [[permissions]]) |
| `1`–`99`         | Static system accounts, often fixed by the distro (`bin`, `daemon`)   |
| `100`/`500`–`999`| Dynamic system accounts, created for services (`SYS_UID_MIN`–`MAX`)   |
| `1000`–`60000`   | Regular users (`UID_MIN`–`UID_MAX`). First human is usually `1000`    |
| `61184`–`65519`  | systemd `DynamicUser=` services (allocated at runtime)                |
| `65534`          | `nobody` / `nogroup`: "unknown user", e.g. over NFS or in user namespaces |
| `100000`+        | Subordinate IDs for rootless containers (`/etc/subuid`, `/etc/subgid`) |

> [!note] Ranges differ a little between distros
> Some use `SYS_UID_MIN 100`, some `500`; always read `/etc/login.defs` instead of hardcoding. More in [[login-defs|login.defs]].

# Real, effective, saved and filesystem IDs
Every process actually has **four** UIDs (and four GIDs):

| ID             | Meaning                                                                    |
| -------------- | -------------------------------------------------------------------------- |
| real (RUID)    | Who started the process                                                    |
| effective (EUID)| Who the process **acts as** for permission checks                         |
| saved (SUID)   | Stored copy so a program can drop privileges and later take them back      |
| filesystem (FSUID) | Used for file access (follows EUID almost always)                      |

```bash
grep -E '^(Uid|Gid):' /proc/self/status
```
```
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
```
Order: real, effective, saved, filesystem.

For a normal program all four are the same. They differ for **setuid** programs: when you run `passwd`, real UID stays `1000` but effective becomes `0`, so it can write to [[shadow]]. See [[special-bits]].

> [!tip] `$UID` vs `$EUID` in bash
> `$UID` = real, `$EUID` = effective. Check `$EUID` when testing for root, because that is what the kernel checks.

# Commands

| Command              | Output on this machine                                  |
| -------------------- | ------------------------------------------------------- |
| `whoami`             | `alice` (name of the *effective* user)                   |
| `id`                 | `uid=1000(alice) gid=1000(alice) groups=1000(alice),10(wheel),...` |
| `id -u` / `id -g`    | `1000` / `1000`                                         |
| `id -un`             | `alice`                                                  |
| `id -G` / `id -nG`   | All group IDs / names                                   |
| `id root`            | Info about another user                                 |
| `ls -n`              | Like `ls -l` but numeric owner and group                |
| `stat -c '%u %U' f`  | Owner UID and name of a file                            |

# Name ⇄ number lookup (NSS)
Programs never read `/etc/passwd` directly in a well behaved system. They call `getpwnam()` / `getpwuid()`, and glibc asks the sources listed in `/etc/nsswitch.conf`:

```
passwd: files systemd
group:  files [SUCCESS=merge] systemd
```

`files` = `/etc/passwd`, `/etc/group`; `systemd` = dynamic users and systemd-homed; others could be `ldap`, `sss`, `winbind`.

`getent` does the same lookup from the shell:
```bash
getent passwd 1000        # by number
getent passwd alice        # by name
getent group 10           # wheel:x:10:alice
```

# Scripting recipes

### UID / name conversion
```bash
uid_to_name() { getent passwd "$1" | cut -d: -f1; }
name_to_uid() { id -u "$1"; }
uid_to_name 1000      # alice
name_to_uid alice      # 1000
```

### All four IDs of a process
```bash
read -r _ r e s f < <(grep ^Uid: /proc/$pid/status)
echo "real=$r eff=$e saved=$s fs=$f"
```

### Find setuid processes running right now (real ≠ effective)
```bash
for d in /proc/[0-9]*; do
  read -r _ r e _ < <(grep ^Uid: "$d/status" 2>/dev/null) || continue
  [[ $r != "$e" ]] && echo "${d#/proc/} $(< "$d/comm") real=$r eff=$e"
done
```

### Find the next free UID in the regular range
```bash
min=$(awk '/^UID_MIN/{print $2}' /etc/login.defs)
getent passwd | awk -F: -v m="$min" '$3>=m && $3<65534 {print $3}' | sort -n |
  awk -v m="$min" 'BEGIN{n=m} $1==n{n++} END{print n}'
```

### Files whose owner no longer exists
```bash
find /home -xdev -nouser 2>/dev/null
```

# Find stuff

| Question                     | Where / command                         |
| ---------------------------- | --------------------------------------- |
| UID ranges                   | `/etc/login.defs`                       |
| Subordinate IDs (containers) | `/etc/subuid`, `/etc/subgid`            |
| UID map of a namespace       | `/proc/<pid>/uid_map`                   |
| Process UIDs                 | `/proc/<pid>/status` (`Uid:` line)      |
| Lookup order                 | `/etc/nsswitch.conf`                    |

# See also
- [[users]] · [[groups]] · [[user-management]]
- [[special-bits]] — setuid/setgid
- [[capabilities]] — root split into pieces
- [[passwd]] · [[group]] · [[login-defs|login.defs]] · [[proc]]
- `man 7 credentials`, `man 5 login.defs`, `man 5 nsswitch.conf`
