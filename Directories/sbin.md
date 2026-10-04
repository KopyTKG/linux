---
tags:
  - linux
  - filesystem
  - sbin
  - scripting
aliases:
  - /sbin
  - system binaries
---
`/sbin` means **system binaries**: commands meant for the administrator (root), such as `fdisk`, `mkfs`, `ip`, `modprobe`, `useradd`, `reboot`. Normal users can often run them, but most need root to actually do anything.

> [!info] On merged systems `/sbin` is just a symlink
> ```
> $ ls -l / | grep sbin
> lrwxrwxrwx   1 root root    7 sbin -> usr/bin
> $ ls -l /usr/sbin
> lrwxrwxrwx   1 root root    3 /usr/sbin -> bin
> ```
> The usr-merge (see [[bin]]) is common across modern distros, but it comes in two flavours:
> - `/sbin → usr/sbin`: there is still a real `/usr/sbin` for admin commands.
> - `/sbin → usr/bin` and `/usr/sbin → bin` (fully merged): **one** folder for all commands.
>
> Some distros still have a separate, real `/sbin`. Scripts should never hardcode `/sbin/...` — use `command -v`.

> [!note] "sbin" is not "secure" or "static"
> The `s` stands for *system*. Very old Unixes did keep statically linked binaries there so the system could be repaired with a broken `/lib`, which is where the "static" myth comes from.

# Typical admin commands
All of these resolve to `/usr/bin` on a fully merged system; on other systems they may be in `/usr/sbin` or `/sbin`:

| Command          | Real file                | Purpose                                     | Related note |
| ---------------- | ------------------------ | ------------------------------------------- | ------------ |
| `fdisk`, `parted`| `/usr/bin/fdisk`         | Partition disks                             | [[dev]]      |
| `mkfs.ext4`, `mkfs.btrfs` | `/usr/bin/mkfs.btrfs` | Create filesystems                      | [[dev]]      |
| `mount`, `umount`| `/usr/bin/mount`         | Attach filesystems                          | [[mnt]]      |
| `ip`             | `/usr/bin/ip`            | Network interfaces, routes                  | [[sys]]      |
| `modprobe`       | `/usr/bin/kmod`          | Load kernel modules                         | [[lib]]      |
| `useradd`, `usermod` | `/usr/bin/useradd`   | Manage users                                | [[etc]], [[home]] |
| `visudo`         | `/usr/bin/visudo`        | Safely edit `/etc/sudoers`                  | [[etc]]      |
| `reboot`, `shutdown` | `/usr/bin/systemctl` | Power control (multi-call symlinks)         | [[run]]      |
| `sysctl`         | `/usr/bin/sysctl`        | Kernel settings                             | [[proc]]     |
| `depmod`         | `/usr/bin/kmod`          | Rebuild kernel module dependency list       | [[boot]]     |

> [!tip] Multi-call binaries
> `modprobe`, `lsmod`, `insmod`, `rmmod` are all symlinks to `kmod`; `reboot`, `poweroff`, `halt`, `shutdown` all point to `systemctl`. The program looks at the name it was called with (`argv[0]`) and acts accordingly.
> ```bash
> readlink -f "$(command -v modprobe)"   # /usr/bin/kmod
> ```

# Why a user sometimes gets `command not found` for admin tools
On distros **without** the full merge, `$PATH` for normal users may not include `/usr/sbin`. Typing `fdisk` as a user then gives `command not found`, while `sudo fdisk` works because sudo uses `secure_path` from `/etc/sudoers`. Fix by running with `sudo` or calling the full path.

# Find stuff

| Question                                | Command                                        |
| --------------------------------------- | ---------------------------------------------- |
| Where is the admin command?             | `command -v fdisk`                             |
| What does the symlink really run?       | `readlink -f "$(command -v reboot)"`           |
| Admin man pages (section 8)             | `ls /usr/share/man/man8`, `man 8 mount`        |
| Search admin commands                   | `man -k . -s 8 \| grep -i user`                |
| What does sudo use as `PATH`?           | `sudo grep secure_path /etc/sudoers`           |

# Scripting recipes

### Portable path to an admin tool
```bash
fdisk_bin=$(command -v fdisk || echo /usr/sbin/fdisk)
"$fdisk_bin" --version
```

### Require root at the top of a script
```bash
if (( EUID != 0 )); then
  echo "Run as root: sudo $0" >&2
  exit 1
fi
```

### Re-run the script with sudo automatically
```bash
[[ $EUID -eq 0 ]] || exec sudo "$0" "$@"
```

### List all section-8 (admin) commands you have
```bash
ls /usr/share/man/man8 | sed 's/\.8.*//' | sort -u | column
```

# See also
- [[bin]] — normal user commands and how `$PATH` works
- [[usr]] — where everything really lives
- [[etc]] — config for most admin tools
- `man 8 intro`, `man 7 hier`
