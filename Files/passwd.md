---
tags:
  - linux
  - files
  - etc
  - users
aliases:
  - /etc/passwd
  - passwd file
---
`/etc/passwd` is the **user database**: one line per account, readable by everyone. Despite the name it no longer holds passwords. Those moved to [[shadow]], which only root can read.

> [!info] Quick facts
> - Path: `/etc/passwd` · owner `root:root` · mode `0644` (world readable on purpose: `ls -l`, `ps` and `id` need to turn UIDs into names)
> - Created at install time; `useradd`/`userdel`/`usermod` edit it
> - Typical desktop: 30–50 lines, of which only one or two are human users; the rest are system accounts with a `nologin` shell
> - Manual: `man 5 passwd`

# Format
Seven fields separated by `:`.

```
alice:x:1000:1000:Alice Example:/home/alice:/bin/fish
```

| #   | Field    | Example       | Meaning                                                              |
| --- | -------- | ------------- | -------------------------------------------------------------------- |
| 1   | name     | `alice`        | Login name                                                           |
| 2   | password | `x`           | `x` = real hash is in [[shadow]]. Empty = **no password at all**     |
| 3   | UID      | `1000`        | User ID. The kernel only knows this number, not the name             |
| 4   | GID      | `1000`        | Primary group ID (name in [[group]])                                 |
| 5   | GECOS    | `Alice Example` | Comment: full name, room, phone, comma separated. Free text          |
| 6   | home     | `/home/alice`  | Home directory → [[home]]                                            |
| 7   | shell    | `/bin/fish`   | Program started at login. Must be listed in [[shells]] for `chsh`    |

### Example lines

```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/usr/sbin/nologin
daemon:x:2:2:daemon:/usr/sbin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
alice:x:1000:1000:Alice Example:/home/alice:/bin/bash
bob:x:1001:1001:Bob:/home/bob:/bin/zsh
```

# UID ranges
Ranges come from [[login-defs]] in the example.

| UID range       | Who                                                            |
| --------------- | -------------------------------------------------------------- |
| `0`             | `root`. Any account with UID 0 **is** root, whatever its name  |
| `1` – `999`     | System accounts for services (`daemon`, `http`, `systemd-*`)     |
| `1000` – `60000`| Normal (human) users. First user gets `1000`                   |
| `65534`         | `nobody`: owns nothing, used for "no privileges"               |
| `100000`+       | Subordinate IDs for containers (`/etc/subuid`)                 |

> [!warning] Security gotchas
> - A second line with UID `0` is a classic backdoor. Check with the recipe below.
> - An **empty** password field (`alice::1000:...`) means login without password. It should always be `x`.
> - `/usr/bin/nologin` prints "This account is currently not available" and exits. Services use it so nobody can log in as them.

> [!tip] Don't edit by hand
> Use `useradd`, `usermod`, `userdel`, `chsh`, `chfn`. If you must edit, use `sudo vipw`: it locks the file and checks syntax. `sudo pwck` checks the file for errors afterwards.

# `passwd` is not the only source
[[nsswitch-conf]] says `passwd: files systemd`. So users can also come from **systemd** (dynamic users of services, `systemd-homed`). That is why scripts should use `getent passwd` instead of reading the file.

```bash
getent passwd alice         # works for every source
getent passwd 1000         # lookup by UID
```

# Scripting recipes

### List human users
```bash
awk -F: '$3>=1000 && $3<60000 {printf "%-10s uid=%-5s home=%-14s shell=%s\n",$1,$3,$6,$7}' /etc/passwd
# alice       uid=1000  home=/home/alice     shell=/bin/fish
```

### Read fields in bash without awk
```bash
while IFS=: read -r name _ uid gid gecos home shell; do
  echo "$name -> $home"
done < /etc/passwd
```

### Find extra root accounts (should print only `root`)
```bash
awk -F: '$3==0 {print $1}' /etc/passwd
```

### Duplicate UIDs (should print nothing)
```bash
cut -d: -f3 /etc/passwd | sort -n | uniq -d
```

### Check every human user has a valid shell
```bash
while IFS=: read -r u _ uid _ _ _ sh; do
  (( uid >= 1000 && uid < 60000 )) || continue
  grep -qxF "$sh" /etc/shells && echo "$u ok ($sh)" || echo "$u shell NOT in /etc/shells"
done < /etc/passwd
```

### Home directory of a user (safe way)
```bash
home=$(getent passwd "$user" | cut -d: -f6)
```

# See also
- [[shadow]] — the password hashes
- [[group]] · [[gshadow]] — groups
- [[login-defs]] — UID ranges and defaults for `useradd`
- [[shells]] — allowed login shells
- [[skel]] — files copied into a new home
- [[etc]] · [[home]]
