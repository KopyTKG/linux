---
tags:
  - linux
  - users
  - security
aliases:
  - Linux groups
  - supplementary groups
  - primary group
  - gpasswd
  - newgrp
---
A **group** is a numbered set of users (a GID). Files have one owning group, and processes carry a list of groups. Groups let you give several users the same access without making files world-readable. Groups are stored in [[group|/etc/group]] (and passwords for them, rarely used, in [[gshadow]]).

# Primary vs supplementary

| Type              | Where it's defined                  | Used for                                         |
| ----------------- | ----------------------------------- | ------------------------------------------------ |
| **Primary**       | 4th field of `/etc/passwd` (GID)    | Group owner of **new files** you create          |
| **Supplementary** | User listed in 4th field of `/etc/group` | Extra access: devices, sockets, sudo, ...  |

```
$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice),7(lp),10(wheel),
29(audio),44(video),999(docker),...
```
- `gid=1000(alice)` = primary group. Most distros create a private group with the same name as the user ("user private groups", `USERGROUPS_ENAB yes` in [[login-defs|login.defs]]).
- `groups=...` = all groups of the process, primary included.

> [!info] Primary members are not listed in `/etc/group`
> `alice:x:1000:` has an empty member list, yet `alice` is in it, because `/etc/passwd` says `gid=1000`. Always check **both** files when asking "who is in group X?" (recipe below).

# Commands

| Command                         | What it does                                             | Root? |
| ------------------------------- | -------------------------------------------------------- | :---: |
| `id`, `id -nG`, `groups`        | Show groups of the current process                       |       |
| `id -nG alice`, `groups alice`  | Show groups from the database (what a new login would get) |     |
| `getent group wheel`            | `wheel:x:10:alice` — group line                           |       |
| `groupadd devs`                 | Create group                                             | ✔     |
| `groupdel devs`                 | Delete group                                             | ✔     |
| `groupmod -n new old`           | Rename                                                   | ✔     |
| `usermod -aG devs alice`        | Add user to group (**`-a`!**, see [[user-management]])   | ✔     |
| `gpasswd -a alice devs`         | Add user to group                                        | ✔     |
| `gpasswd -d alice devs`         | Remove user from group                                   | ✔     |
| `gpasswd -A alice devs`         | Make alice group administrator (can then add members herself) | ✔ |
| `newgrp devs`                   | Start a **new shell** with `devs` as primary group       |       |
| `sg devs -c 'cmd'`              | Run one command with `devs` as group                     |       |
| `chgrp devs file`               | Change group owner of a file                             | owner |

# Why you have to log in again
Group list is copied into a process **when you log in** (by `login`, `sshd`, the display manager calling `initgroups()`), and children inherit it. Adding yourself to a group edits `/etc/group`, but already running processes keep their old list.

```bash
sudo usermod -aG docker alice
id -nG | grep -w docker      # nothing yet: this shell has the old list
id -nG alice | grep -w docker # docker: the database already has it
```

Fixes, from best to quickest:
1. Log out and back in (whole desktop session).
2. `newgrp docker` — new shell with the group, only in that terminal.
3. `su - $USER` — new login shell in that terminal.

> [!tip] Reboot isn't needed, but logout is
> With systemd user services, a plain logout may leave `systemd --user` running with the old groups (if lingering is on). `loginctl terminate-user $USER` restarts everything, see [[login-sessions]].

# Common groups and what they give
Names and numbers **vary by distro**; what a group grants is decided by who owns the device, socket or config that checks it.

| Group                  | Grants                                                                 |
| ---------------------- | ---------------------------------------------------------------------- |
| `wheel` or `sudo` (varies by distro) | Allowed to use `sudo` / `su` (only because [[sudoers]] or PAM says so) |
| `adm`                  | Read logs in `/var/log` (on some distros)                              |
| `systemd-journal`      | Read the full system journal                                           |
| `video`                | Access to `/dev/video*`, sometimes GPU/backlight                       |
| `render`               | GPU render nodes `/dev/dri/renderD*`                                   |
| `audio`                | Direct sound devices `/dev/snd/*` (usually not needed with PipeWire)   |
| `input`                | Read **every** keyboard and mouse `/dev/input/*` (a keylogger can too) |
| `disk`                 | Raw access to block devices `/dev/sd*`, `/dev/nvme*`                   |
| `kvm`                  | Use `/dev/kvm` for virtual machines                                    |
| `lp`                   | Printers                                                               |
| `dialout` / `uucp` (varies by distro) | Serial ports `/dev/ttyS*`, `/dev/ttyUSB*`, `/dev/ttyACM*` |
| `docker`, `libvirt`, `lxd` | Talk to that daemon's socket                                       |
| `users`                | Generic shared group                                                   |
| `nogroup`/`nobody`     | `65534`, unmapped                                                      |

Check what a group grants on your system (`/dev/video*` appears too when a webcam is plugged in):
```bash
find /dev /run -group video 2>/dev/null     # devices/sockets owned by video
```
```
/dev/fb0
/dev/dri/card1
```

> [!danger] Some groups are root in disguise
> - **`docker`** (also `lxd`, `libvirt` in many setups): anyone in it can run `docker run -v /:/host --privileged ...` and edit any file as root. Membership = passwordless root.
> - **`disk`**: can read `/dev/nvme0n1` raw, which includes `/etc/shadow` and everything else.
> - **`input`**: can read every keystroke, including passwords typed in other apps.
>
> If your own account is in `docker`, treat it as root-equivalent.

# Scripting recipes

### Is a user in a group?
```bash
in_group() {  # in_group GROUP [USER]
  id -nG "${2:-$USER}" | tr ' ' '\n' | grep -qx "$1"
}
in_group wheel && echo "can probably sudo"
```

### All members of a group (supplementary + primary)
```bash
members() {
  local gid
  gid=$(getent group "$1" | cut -d: -f3) || return 1
  {
    getent group "$1" | cut -d: -f4 | tr , '\n'          # listed members
    getent passwd | awk -F: -v g="$gid" '$4==g {print $1}' # primary members
  } | grep -v '^$' | sort -u
}
members kvm
```
```
libvirt-qemu
qemu
```

### Do I need to log in again? (session vs database)
```bash
diff <(id -nG | tr ' ' '\n' | sort) <(id -nG "$USER" | tr ' ' '\n' | sort) \
  && echo "session groups match" || echo "re-login to apply group changes"
```

### Empty groups
```bash
awk -F: '$4=="" {print $1}' /etc/group
```

### Groups of a running process
```bash
grep ^Groups: /proc/$pid/status
```
```
Groups:	3 90 948 949 958 980 982 983 984 985 989 995 998 1000
```

# Find stuff

| Question                         | Command                                   |
| -------------------------------- | ----------------------------------------- |
| My groups now                    | `id -nG`                                  |
| My groups after next login       | `id -nG $USER`                            |
| Group line by name / GID         | `getent group wheel`, `getent group 998`  |
| What a group can access          | `find / -xdev -group NAME 2>/dev/null`    |
| Which group a device needs       | `ls -l /dev/dri/card1`, `stat -c %G /dev/dri/card1` |
| Which group sudo trusts          | [[sudoers]] (`%wheel` or `%sudo` lines)   |

# See also
- [[users]] · [[uid-gid]] · [[user-management]] · [[sudo-su]]
- [[group]] · [[gshadow]] · [[sudoers]] · [[login-defs|login.defs]]
- [[permissions]] · [[file-permissions]] · [[special-bits]] — setgid directories for shared folders
- [[dev]] — device files and their groups
- `man 5 group`, `man 8 gpasswd`, `man 1 newgrp`
