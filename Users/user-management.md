---
tags:
  - linux
  - users
  - administration
aliases:
  - useradd
  - usermod
  - userdel
  - passwd command
  - chage
---
The **shadow-utils** commands (`useradd`, `usermod`, `userdel`, `passwd`, `chage`, `groupadd`, ...) are the standard low-level way to manage accounts on every Linux system. They edit [[passwd]], [[shadow]], [[group]] and [[gshadow]] safely (with locking), so you never have to edit those files by hand.

> [!warning] All of these need root
> Run them with `sudo` (see [[sudo-su]]). The examples below are for reading and understanding; nothing here was executed on this machine.

# Which command touches which file

| Command    | `/etc/passwd` | `/etc/shadow` | `/etc/group` | `/etc/gshadow` | home dir | 
| ---------- | :-----------: | :-----------: | :----------: | :------------: | :------: |
| `useradd`  | ✔             | ✔             | ✔ (user's own group) | ✔      | creates (with `-m`) |
| `usermod`  | ✔             | ✔             | ✔ (`-G`)     | ✔              | moves (with `-d -m`) |
| `userdel`  | ✔             | ✔             | ✔            | ✔              | removes (with `-r`) |
| `passwd`   |               | ✔             |              |                |          |
| `chage`    |               | ✔             |              |                |          |
| `chsh`     | ✔ (shell)     |               |              |                |          |
| `chfn`     | ✔ (GECOS)     |               |              |                |          |
| `groupadd`/`groupmod`/`groupdel` |  |       | ✔            | ✔              |          |

Defaults for new accounts come from two places:

| File                    | Holds                                                         |
| ----------------------- | ------------------------------------------------------------- |
| `/etc/login.defs`       | UID/GID ranges, password aging defaults, hash method (`ENCRYPT_METHOD YESCRYPT` here), `UMASK`, `USERGROUPS_ENAB` → [[login-defs|login.defs]] |
| `/etc/default/useradd`  | Default shell, home base, skel dir. Show with `useradd -D`    |

```
$ useradd -D
GROUP=1000
HOME=/home
INACTIVE=-1
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
```

# `useradd` — create a user

| Flag                | Meaning                                                     |
| ------------------- | ----------------------------------------------------------- |
| `-m`                | **Create home** directory and copy [[skel|/etc/skel]] into it. Without it no home is made! |
| `-s /bin/bash`      | Login shell (must be listed in `/etc/shells` to be "valid") |
| `-G wheel,video`    | Supplementary groups                                        |
| `-g users`          | Primary group (default: new group with the same name, if `USERGROUPS_ENAB yes`) |
| `-u 1500`           | Force a UID                                                 |
| `-c "Full Name"`    | Comment / GECOS field                                       |
| `-d /srv/app`       | Home path                                                   |
| `-r`                | **System** account (UID from the system range, no aging)    |
| `-e 2026-12-31`     | Account expiry date                                         |
| `-k /path`          | Use another skel directory                                  |

```bash
sudo useradd -m -s /bin/bash -G video,audio -c "Alice" alice   # human user
sudo passwd alice                                              # set password
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/myapp myapp    # service account
```

> [!note] `adduser`
> Some distros ship an interactive `adduser` wrapper (a Perl/shell script that asks questions and calls `useradd`). Others make `adduser` a plain symlink to `useradd`, and some don't have it at all. `useradd` exists everywhere, so use it in scripts.

### What happens to the new home
`useradd -m` copies everything from `/etc/skel` (here: `.bash_logout`, `.bash_profile`, `.bashrc`, `.config`, `.zshrc`), sets ownership to the new user and applies `HOME_MODE` / `UMASK` from `login.defs`. See [[skel]] and [[home]].

# `usermod` — change a user

| Flag              | Meaning                                                           |
| ----------------- | ----------------------------------------------------------------- |
| `-aG docker`      | **Append** to supplementary group. Always use `-a` with `-G`!     |
| `-G a,b`          | **Replace** all supplementary groups with exactly `a,b`           |
| `-s /bin/zsh`     | Change shell (or use `chsh`)                                      |
| `-l newname`      | Rename login                                                      |
| `-d /new -m`      | Change home and move its contents                                 |
| `-L` / `-U`       | Lock / unlock password (puts / removes `!` in front of the hash)  |
| `-e 1`            | Expire the account (no login at all)                              |
| `-u 1501`         | Change UID (also re-owns files in home; others must be fixed by hand) |

> [!danger] `usermod -G` without `-a`
> `sudo usermod -G docker alice` **removes** `alice` from every other group, including `wheel`. You lose `sudo`. Always write `usermod -aG`.

Group changes only apply to **new logins**, see [[groups]].

# `userdel` — remove a user

| Flag  | Meaning                                                   |
| ----- | --------------------------------------------------------- |
| (none)| Remove from passwd/shadow/group; **keep** home and mail   |
| `-r`  | Also delete home and mail spool                           |
| `-f`  | Force, even if user is logged in                          |

Files outside home are **not** removed; they become owned by a bare UID. Find them afterwards:
```bash
sudo find / -xdev -nouser 2>/dev/null
```

# `passwd` — passwords

| Command                 | Effect                                                 |
| ----------------------- | ------------------------------------------------------ |
| `passwd`                | Change your own password (asks for the old one)        |
| `sudo passwd alice`     | Set someone else's password (no old password needed)   |
| `sudo passwd -l alice`  | Lock: prefix hash with `!` (password login fails, SSH keys still work!) |
| `sudo passwd -u alice`  | Unlock                                                  |
| `sudo passwd -d alice`  | Delete password (empty: may allow login with no password, dangerous) |
| `sudo passwd -e alice`  | Expire: force change on next login                     |
| `passwd -S alice`        | Status: `alice P 2026-06-20 -1 99999 7 -1` (P = usable, L = locked, NP = no password) |

`passwd` itself is a **setuid root** program (`-rwsr-xr-x root /usr/bin/passwd`): that is how a normal user can write to root-only [[shadow]]. See [[special-bits]].

# `chage` — password aging

| Flag              | Field in shadow     | Meaning                                 |
| ----------------- | ------------------- | --------------------------------------- |
| `-l alice`        | —                   | Show aging info (readable)              |
| `-d 0`            | last change         | Force password change on next login     |
| `-M 90`           | max                 | Password valid for 90 days              |
| `-m 1`            | min                 | Can't change again within 1 day         |
| `-W 7`            | warn                | Warn 7 days before expiry               |
| `-I 14`           | inactive            | Lock 14 days after password expired     |
| `-E 2026-12-31`   | expire              | Account expiry date (`-E -1` = never)   |

Field layout and day math are in [[shadow]].

# Locking accounts — options compared

| Method                              | Password login | SSH key login | `su` from root | Undo                     |
| ----------------------------------- | :------------: | :-----------: | :------------: | ------------------------ |
| `passwd -l` / `usermod -L`          | ✘              | ✔             | ✔              | `passwd -u`              |
| `usermod -e 1` (expire account)     | ✘              | ✘             | ✔              | `usermod -e ''`          |
| `usermod -s /usr/sbin/nologin`      | ✘ (no shell)   | ✘ (no shell)  | ✘              | set a real shell back    |
| All three together                  | ✘              | ✘             | ✘              |                          |

> [!info] `nologin` and `false`
> `nologin` prints "This account is currently not available." and exits; `/bin/false` exits silently. Both are used as shells for system accounts, so nobody can get an interactive shell as `dbus` or `polkitd`. On this machine 35 of 38 accounts use `nologin`. The path differs: `/usr/sbin/nologin`, `/sbin/nologin` or `/usr/bin/nologin`, check with `command -v nologin`.

# Scripting recipes

### Does a user exist?
```bash
if getent passwd "$name" >/dev/null; then
  echo "$name exists"
fi
```
`id "$name" &>/dev/null` works too.

### Idempotent "create service user if missing"
```bash
name=myapp
getent passwd "$name" >/dev/null ||
  sudo useradd -r -s "$(command -v nologin)" -d "/var/lib/$name" -M "$name"
```
(`-M` = don't create home; create the state directory separately or let systemd do it with `StateDirectory=`, see [[services]].)

### Users with a valid login shell from `/etc/shells`
```bash
awk -F: 'NR==FNR { if ($0 !~ /^#/ && NF) ok[$0]=1; next }
         ($7 in ok) { print $1, $7 }' /etc/shells /etc/passwd
```

### Users whose home directory is missing
```bash
getent passwd | awk -F: '$3>=1000 && $3<65534 {print $1, $6}' |
  while read -r u h; do [[ -d $h ]] || echo "$u: $h missing"; done
```

> [!tip] Modern alternative: `systemd-sysusers`
> Packages and services can declare users in a small config file (`/usr/lib/sysusers.d/*.conf`, e.g. `u myapp - "My App" /var/lib/myapp`), and `systemd-sysusers` creates them at boot. That's how most system accounts appear on systemd-based distros. See [[systemd]].

# Find stuff

| Question                         | Where / command                    |
| -------------------------------- | ---------------------------------- |
| Defaults for new users           | `useradd -D`, `/etc/default/useradd`, `/etc/login.defs` |
| Valid login shells               | `/etc/shells`                      |
| Template home                    | `/etc/skel`                        |
| Account / password status        | `passwd -S user`, `sudo chage -l user` |
| Declared system users            | `/usr/lib/sysusers.d/`, `/etc/sysusers.d/` |

# See also
- [[users]] · [[uid-gid]] · [[groups]] · [[sudo-su]] · [[pam]]
- [[passwd]] · [[shadow]] · [[group]] · [[gshadow]] · [[login-defs|login.defs]] · [[skel]] · [[home]]
- `man 8 useradd`, `man 8 usermod`, `man 1 passwd`, `man 1 chage`
