---
tags:
  - linux
  - filesystem
  - var
  - logs
  - scripting
aliases:
  - /var
  - variable data
---
`/var` is where the system keeps **variable data**: files that programs change while they run. Logs, caches, package databases, mail queues, container images, and so on.

> [!info] Why `/var` is separate from `/usr`
> The Filesystem Hierarchy Standard (FHS) divides data into:
> - **static** (`/usr`, `/bin`, `/lib`) — changes only when you install or upgrade software. It could even be read-only.
> - **variable** (`/var`) — changes all the time while the system runs.
>
> Because of that, `/var` often gets its **own partition or subvolume**. Then logs that grow out of control fill `/var`, not `/`. Snapshots of `/` also leave out the logs and caches.
> On a btrfs system several parts of `/var` are separate subvolumes:
>
> | Mountpoint   | Subvolume | Why separate                                          |
> | ------------ | --------- | ----------------------------------------------------- |
> | `/var/log`   | `@log`    | Rolling back `/` must not delete logs about the crash |
> | `/var/cache` | `@cache`  | No reason to snapshot gigabytes of re-downloadable data |
> | `/var/tmp`   | `@tmp`    | Temporary junk, never snapshot it                      |
>
> Check it yourself with `findmnt -T /var/log`.

> [!tip] Golden rules for `/var`
> 1. Most of `/var` belongs to **root**. Read with your own user, but writes or deletes need `sudo`. Some folders can't be read at all without it (`/var/lib/docker`, `/var/spool/cups`).
> 2. **Never hand-delete** inside `/var/lib`. It holds the state of programs (package database, docker images, ...). Use the program's own tool (`docker`, `coredumpctl`, your package manager, ...).
> 3. `/var/cache` is safe to clean, because programs can download or rebuild it. Still prefer the program's own clean command over `rm`.
> 4. When the disk is full, `/var` is the first place to look (`du -xsh /var/*`).

# Overview

```
/var
├── cache   → @cache subvolume
├── db
├── empty
├── games
├── lib
├── local
├── lock -> ../run/lock
├── log     → @log subvolume
├── mail -> spool/mail
├── opt
├── run -> ../run
├── spool
└── tmp     → @tmp subvolume
```

| Directory      | What's inside                                                       | Safe to delete?            |
| -------------- | ------------------------------------------------------------------- | -------------------------- |
| `/var/log`     | Logs: systemd journal, `wtmp`, `btmp`, app logs, ...                | Old logs yes, via tools    |
| `/var/cache`   | Data that can be rebuilt or downloaded again (packages, man index)  | Yes, via tools             |
| `/var/lib`     | Persistent **state** of programs (package DB, docker, VMs)          | **No**                     |
| `/var/tmp`     | Temp files that **survive reboot**                                  | Old ones, auto-cleaned     |
| `/var/spool`   | Queues waiting to be processed (print jobs, mail, cron)             | Usually no                 |
| `/var/run`     | Symlink to [[run]] (old location, kept for compatibility)           | —                          |
| `/var/lock`    | Symlink to `/run/lock`                                              | —                          |
| `/var/mail`    | Symlink to `spool/mail`, local user mailboxes                       | —                          |
| `/var/empty`   | Empty dir used as chroot jail (e.g. by `sshd`)                      | —                          |
| `/var/opt`     | Variable data of packages installed in [[opt]]                     | —                          |
| `/var/local`   | Variable data of software in `/usr/local`                           | —                          |
| `/var/games`   | Game scores (group `games`)                                         | —                          |
| `/var/db`      | BSD-style databases (rarely used on Linux)                          | —                          |
| `/var/.updated`| Timestamp from `systemd-update-done`: when `/var` last caught up with `/usr` | —                 |

```bash
du -xsh /var/* 2>/dev/null | sort -rh     # what is big
```
```
4.2G	/var/lib
2.1G	/var/cache
120M	/var/log
```

# `/var/log`
Logs. There are two kinds of logs on a modern systemd system:
1. **systemd journal** — binary files in `/var/log/journal/`, read with `journalctl`.
2. **Plain text files** written by programs themselves (`cups/`, `libvirt/`, your package manager's log, ...).

| File / Dir           | What                                                         | Read with                 |
| -------------------- | ------------------------------------------------------------ | ------------------------- |
| `journal/<machine-id>/` | systemd journal (all services, kernel, boot)              | `journalctl`              |
| `wtmp`               | Login history (binary)                                       | `last`                    |
| `btmp`               | **Failed** logins (binary, root only)                        | `sudo lastb`              |
| `lastlog`            | Last login per user                                          | `lastlog`                 |
| `boot.log`           | Boot messages from plymouth (`.1`…`.7` = older boots)        | `less`                    |
| `audit/`             | Linux audit framework (SELinux/security events)              | `ausearch`                |
| `cups/`              | Printer service                                              | `less`                    |
| `snapper.log`        | btrfs snapshot tool                                          | `less`                    |
| `old/`               | Rotated logs                                                 | —                         |

> [!note] Where is `/var/log/syslog` or `messages`?
> Those files are written by a **syslog daemon** (`rsyslog`, `syslog-ng`). Many systems install one next to journald, so the same messages exist twice: binary in the journal, plain text in `/var/log/syslog` or `/var/log/messages`.
> Systems without a syslog daemon (like this one) keep everything **only** in the journal. `journalctl` works on both, so prefer it in scripts.

### systemd journal
```
/var/log/journal/0123456789abcdef0123456789abcdef/
├── system.journal              # current, system-wide
├── system@...-....journal      # rotated archives
└── user-1000.journal           # your user's services
```
The folder name is your **machine ID** (`cat /etc/machine-id`).
The files are binary (indexed, compressed), so `cat` and `grep` don't work on them. Use `journalctl`.

```bash
journalctl --disk-usage                  # 43.5M here
journalctl -b                            # this boot
journalctl -b -1                         # previous boot
journalctl --list-boots                  # all boots stored
journalctl -b -p err                     # errors only (emerg..err)
journalctl -u NetworkManager             # one service
journalctl --user -u pipewire            # one user service
journalctl -k                            # kernel only (like dmesg)
journalctl -f                            # follow live (like tail -f)
journalctl --since "1 hour ago"
journalctl --since 2026-10-03 --until "2026-10-03 22:00"
journalctl _PID=1234                     # by PID
journalctl /usr/bin/sshd                 # by executable
journalctl -o json-pretty -n 1           # all fields of one entry
journalctl -o cat -u foo                 # message text only, good for grep
```

**Priorities** (`-p`):

| Number | Name      | Use                                      |
| ------ | --------- | ---------------------------------------- |
| 0      | `emerg`   | System unusable                          |
| 1      | `alert`   | Act immediately                          |
| 2      | `crit`    | Critical                                 |
| 3      | `err`     | Error                                    |
| 4      | `warning` | Warning                                  |
| 5      | `notice`  | Normal but significant                   |
| 6      | `info`    | Informational                            |
| 7      | `debug`   | Debug                                    |

`-p err` means "err and **more** severe" (0–3). Use a range for exact levels: `-p warning..warning`.

> [!tip] Clean the journal
> ```bash
> sudo journalctl --vacuum-size=500M   # keep max 500 MB
> sudo journalctl --vacuum-time=2weeks # delete older than 2 weeks
> ```
> To set a permanent limit put `SystemMaxUse=500M` in `/etc/systemd/journald.conf` → see [[etc]].

### Plain-text logs and `logrotate`
Programs that write their own text logs would grow them forever. `logrotate` (usually run daily by a systemd timer or cron) renames `app.log` → `app.log.1` → `app.log.2.gz` ... and deletes the oldest. Rules live in `/etc/logrotate.conf` and `/etc/logrotate.d/` → see [[etc]].

```bash
ls /etc/logrotate.d/                          # bootlog cups libvirtd snapper ...
find /var/log -type f -mmin -1440 2>/dev/null # log files changed in the last 24 h
tail -f /var/log/cups/error_log               # follow a text log live
```

> [!tip] "Something broke after an update"
> 1. `journalctl -b -1 -p warning` — warnings from the **previous** boot.
> 2. Check your package manager's log in `/var/log/` to see what was upgraded that day.
> 3. Old package versions often still sit in `/var/cache/<package manager>/` — your downgrade path.

# `/var/cache`
Data a program can recreate. Deleting it costs time or bandwidth, not data.

| Dir                  | What                                                   |
| -------------------- | ------------------------------------------------------ |
| `<package manager>/` | Downloaded packages (usually the biggest thing here)   |
| `man/`               | Index for `man -k` / `apropos`                         |
| `fontconfig/`        | Font cache (`fc-cache`)                                |
| `ldconfig/`          | Shared library cache (`ldconfig`) → see [[lib]]         |
| `cups/`, `libvirt/`  | Service caches                                         |

### Package cache
Every package manager downloads packages into its own folder under `/var/cache` before installing them. Some never delete old versions by themselves, so the cache can grow to many gigabytes (10 GB or more is not unusual).

```bash
du -sh /var/cache/* 2>/dev/null | sort -rh | head -5   # which cache is big
```

> [!warning] Don't empty it blindly
> Old package versions in the cache are your **downgrade path** if an upgrade breaks something. Use your package manager's own clean command, which can keep the last 1–2 versions, instead of `rm -rf`.

# `/var/lib`
**State** of programs: the data a program needs to keep working after reboot. **Do not delete things by hand here.** It is the most important part of `/var` to back up.

| Dir                       | What                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `<package manager>/`      | **Installed packages database** — never edit by hand          |
| `systemd/`                | systemd state: `coredump/`, `timers/`, `linger/`, `random-seed` |
| `docker/`, `containerd/`  | Container images, volumes, layers (root only)                |
| `libvirt/`                | Virtual machine disks and definitions                        |
| `NetworkManager/`         | DHCP leases, saved state                                     |
| `bluetooth/`              | Paired devices                                               |
| `dkms/`                   | Out-of-tree kernel modules being built (nvidia, ...)         |
| `plocate/`                | Database for `locate`                                        |
| `machines/`, `portables/` | `systemd-nspawn` containers                                  |
| `AccountsService/`        | User avatars and login settings                              |

> [!danger] Lock files in `/var/lib`
> Package managers and other tools create a **lock file** while they run so two copies can't corrupt the database. If the tool crashes, the lock may stay behind and the next run refuses to start.
> Delete a stale lock **only** after checking that no copy of the program is running (`pgrep -a <name>`).

### `systemd/coredump`
When a program crashes, systemd saves its memory dump here:
```
core.myapp.1000.4f1d2c3b....8012.1791126686000000.zst
     name uid  boot ID      PID  timestamp
```
```bash
coredumpctl list          # list crashes
coredumpctl info <PID>    # details + backtrace
```

# `/var/tmp` vs `/tmp`

|                      | [[tmp]]                         | `/var/tmp`                      |
| -------------------- | ------------------------------- | ------------------------------- |
| Survives reboot      | **No** (here it is `tmpfs` = RAM) | **Yes** (on disk)              |
| Auto-cleaned after   | 10 days                         | 30 days                         |
| Use for              | Small, short-lived files        | Bigger files or ones needed after reboot |

Cleaning rules come from `systemd-tmpfiles` (`/usr/lib/tmpfiles.d/tmp.conf`):
```
q /tmp 1777 root root 10d
q /var/tmp 1777 root root 30d
```

> [!note] `systemd-private-*` folders
> You will see folders like `systemd-private-6732...-bluetooth.service-9haqP3` in both temp dirs. Services with `PrivateTmp=yes` get their **own** private `/tmp`, so they can't see each other's temp files. systemd deletes the folder when the service stops.

# `/var/spool`
Queues: data waiting to be processed.

| Dir                 | What                                       |
| ------------------- | ------------------------------------------ |
| `spool/cups/`       | Print jobs waiting for the printer         |
| `spool/mail/`       | Local mailboxes (`/var/mail` points here)  |
| `spool/cron/`       | User crontabs (when a cron daemon is installed) |
| `spool/plymouth/`   | Boot splash data                           |

# `/var/run` & `/var/lock`
Old locations, now **symlinks**:
```
/var/run  -> ../run
/var/lock -> ../run/lock
```
Runtime data (PID files, sockets) moved to [[run]], which is a `tmpfs` that is emptied on every boot. The symlinks stay so old programs and scripts don't break. Write `/run` in new scripts.

# Scripting recipes
All tested in `bash`, no root needed.

### Recent logins and reboots
```bash
last -n 5            # from /var/log/wtmp
last reboot | head -3
```
```
reboot   system boot  6.12.8          Sun Oct  4 16:57   still running
reboot   system boot  6.12.8          Sat Oct  3 13:40 - 22:25  (08:45)
```

### Boots stored in the journal
```bash
journalctl --list-boots --no-pager | tail -3
```
```
 -1 8c1f2e3d4a5b4c6d9e7f0a1b2c3d4e5f Sat 2026-10-03 13:40:33 CEST Sat 2026-10-03 22:25:38 CEST
  0 4f1d2c3b5a6e4b7c9d8e0f1a2b3c4d5e Sun 2026-10-04 16:57:23 CEST Sun 2026-10-04 17:51:27 CEST
```

### Log files changed recently
```bash
find /var/log -type f -mmin -60 -printf '%TT  %p\n' 2>/dev/null | sort   # last hour
```

### Biggest caches
```bash
du -sh /var/cache/* 2>/dev/null | sort -rh | head
```

### Errors from this boot, grouped by program
```bash
journalctl -b -p err -o json --output-fields=SYSLOG_IDENTIFIER --no-pager -q |
  grep -o '"SYSLOG_IDENTIFIER":"[^"]*"' | cut -d'"' -f4 | sort | uniq -c | sort -rn
```

### Disk usage of `/var`
```bash
du -xsh /var/* 2>/dev/null | sort -rh              # top level
du -xh --max-depth=2 /var 2>/dev/null | sort -rh | head -20   # one level deeper
sudo du -xsh /var/lib/* | sort -rh | head          # needs root for docker etc.
```
`-x` = stay on one filesystem. Without it `du` would also count mounted things inside, like docker overlays.

### Free up space checklist
```bash
journalctl --disk-usage
du -sh /var/cache/* | sort -rh        # caches (clean with the owning tool)
coredumpctl list                      # old crash dumps (/var/lib/systemd/coredump)
docker system df                      # docker usage (if installed)
```

# Find stuff

| I want to find...                         | Look in                                         |
| ----------------------------------------- | ----------------------------------------------- |
| Why a service failed                      | `journalctl -u <service> -b`                    |
| What happened at last boot / crash        | `journalctl -b -1 -p warning`                   |
| What was updated and when                 | your package manager's log in `/var/log/`       |
| Downloaded packages                       | `/var/cache/<package manager>/`                 |
| Installed packages database               | `/var/lib/<package manager>/`                   |
| Who logged in                             | `/var/log/wtmp` (`last`)                        |
| Failed login attempts                     | `/var/log/btmp` (`sudo lastb`)                  |
| Crash dumps                               | `/var/lib/systemd/coredump/` (`coredumpctl`)    |
| Docker images and volumes                 | `/var/lib/docker/`                              |
| VM disk images                            | `/var/lib/libvirt/images/`                      |
| Print queue                               | `/var/spool/cups/` (`lpq`)                      |
| What is eating disk space                 | `du -xsh /var/* \| sort -rh`                    |

# See also
- [[run]] — runtime data, the new home of `/var/run` and `/var/lock`
- [[tmp]] — the other temp directory
- [[etc]] — configuration (`journald.conf`, `logrotate.d`, `tmpfiles.d`)
- [[usr]] — the static counterpart of `/var`
- [[proc]] — live process info (the journal stores what `/proc` showed at the time)
- `man 7 hier`, `man 7 file-hierarchy` — official layout description
- `man journalctl`, `man logrotate`, `man tmpfiles.d`
- FHS spec: <https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05.html>
