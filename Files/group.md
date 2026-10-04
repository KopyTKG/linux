---
tags:
  - linux
  - files
  - etc
  - users
aliases:
  - /etc/group
  - group file
---
`/etc/group` lists **groups** and which users are **extra members** of them. Groups decide who can open a device, read a log, use `sudo`, talk to Docker, and so on.

> [!info] Quick facts
> - Path: `/etc/group` · owner `root:root` · mode `0644`
> - Typical desktop system: 50–80 groups
> - Edit with `groupadd`, `groupdel`, `gpasswd`, `usermod -aG`, or `sudo vigr`
> - Manual: `man 5 group`

# Format
Four fields separated by `:`.

```
wheel:x:998:alice
```

| #   | Field    | Example | Meaning                                                      |
| --- | -------- | ------- | ------------------------------------------------------------ |
| 1   | name     | `wheel` | Group name                                                   |
| 2   | password | `x`     | Hash lives in [[gshadow]]. Group passwords are almost unused |
| 3   | GID      | `998`   | Group ID                                                     |
| 4   | members  | `alice`  | Comma separated list of **supplementary** members            |

### Example lines
```
root:x:0:root
sys:x:3:bin
wheel:x:998:alice
audio:x:995:alice,bob
video:x:985:alice
input:x:994:
systemd-journal:x:990:
docker:x:960:alice
alice:x:1000:
bob:x:1001:
```

> [!important] Primary group is NOT in the members list
> `alice:x:1000:` has an empty member list, yet `alice` is in it. Your **primary** group comes from field 4 of [[passwd]]. The members field only lists **supplementary** groups.
> So never answer "is X in group Y" by grepping this file. Use `id -nG X`.

# Groups worth knowing

| Group             | Gives access to                                                    |
| ----------------- | ------------------------------------------------------------------ |
| `wheel` / `sudo`  | Run commands as root via [[sudoers]] (name depends on distro)      |
| `audio`           | Sound devices in `/dev/snd` (mostly not needed with PipeWire)      |
| `video`           | GPU / framebuffer devices, backlight                               |
| `input`           | Raw keyboard/mouse events in `/dev/input` (keyloggers love it)     |
| `disk`            | Raw disks. **Effectively root**                                    |
| `docker`          | Docker socket. **Effectively root**                                |
| `systemd-journal` | Read the full system journal without sudo                          |
| `adm`             | Read logs in [[var]] `/var/log` (Debian family)                    |
| `uucp` / `dialout`| Serial ports `/dev/ttyS*`, `/dev/ttyUSB*`                          |
| `kvm`             | `/dev/kvm` for virtual machines                                    |

> [!warning] Some groups equal root
> Members of `docker`, `disk`, `lxd` or `libvirt` can become root without a password (e.g. `docker run -v /:/host ...`). Treat adding someone to these like giving them sudo.

> [!tip] Group changes need a new login
> After `sudo usermod -aG docker alice` the running session still has the old groups. Log out and in, or start a shell with `newgrp docker`.
> **Always use `-aG`.** `-G` alone **replaces** all supplementary groups.

# Scripting recipes

### Members of a group (supplementary only)
```bash
getent group wheel | cut -d: -f4 | tr , '\n'
```

### All groups of a user (incl. primary)
```bash
id -nG alice
# alice wheel audio video docker
```

### Is the current user in a group?
```bash
if id -nG | grep -qw docker; then echo "yes"; fi
```

### Every user whose primary group is X
```bash
gid=$(getent group users | cut -d: -f3)
awk -F: -v g="$gid" '$4==g {print $1}' /etc/passwd
```

### Groups with no supplementary members
```bash
awk -F: '$4==""' /etc/group
```

### Who can read this file via group?
```bash
stat -c '%G %A' /var/log/journal
```

# See also
- [[gshadow]] — group passwords and group admins
- [[passwd]] — primary group of each user
- [[sudoers]] — what `wheel` actually allows
- [[dev-special-files]] · [[dev]] — device permissions per group
- [[etc]]
