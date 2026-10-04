---
tags:
  - linux
  - systemd
  - init
aliases:
  - systemd overview
  - PID 1
  - init system
---
systemd is the **first program the kernel starts** (PID 1) and the manager of everything that runs after it. It starts services, mounts filesystems, collects logs, handles logins, keeps time in sync and shuts the machine down cleanly.

> [!info] What PID 1 means
> The kernel boots, mounts the root filesystem and runs exactly one program: `/sbin/init`. On this machine that is a symlink to systemd:
> ```
>     PID COMMAND   COMMAND
>       1 systemd   /usr/lib/systemd/systemd --switched-root --system --deserialize=60 splash
> ```
> Every other process is a child (or grand-child …) of PID 1. If PID 1 dies, the kernel panics. See [[proc]] → `/proc/1/`.

This machine runs **systemd 262** (`systemctl --version`).

# What systemd does
| Job                          | How                                                                  | Note                       |
| ---------------------------- | -------------------------------------------------------------------- | -------------------------- |
| Start/stop programs          | **units** (`.service`, `.socket`, `.timer`, …)                       | [[units]], [[services]]    |
| Boot in the right order      | **targets** + dependencies                                           | [[targets-boot]]           |
| Run things on a schedule     | **timers** (replacement for cron)                                    | [[timers]]                 |
| Collect logs                 | **journald** → `journalctl`                                          | [[journal]]                |
| Limit/measure resources      | **cgroups** (each service gets its own) → `/sys/fs/cgroup`           | [[sys]]                    |
| Per-user services            | `systemctl --user`                                                   | [[user-units]]             |
| Mount filesystems            | turns `/etc/fstab` lines into `.mount` units                         | [[etc]]                    |
| Logins, sessions, power keys | **logind**                                                           |                            |
| Temporary files cleanup      | **systemd-tmpfiles**                                                 | [[tmp]], [[run]]           |

# Components
systemd is a family of small daemons, all living in `/usr/lib/systemd/` (see [[usr]]). State on this machine (`systemctl is-active` / `is-enabled`):

| Component            | Job                                              | Here                                   |
| -------------------- | ------------------------------------------------ | -------------------------------------- |
| `systemd` (PID 1)    | Service manager                                  | running                                |
| `systemd-journald`   | Log collector                                    | **active** (static)                    |
| `systemd-logind`     | User sessions, seats, suspend, power button      | **active** (static)                    |
| `systemd-udevd`      | Device events, creates `/dev` symlinks           | **active** (static) → [[dev]]          |
| `systemd-resolved`   | DNS resolver / cache                             | **active** (enabled) → [[dns]]         |
| `systemd-timesyncd`  | NTP time sync                                    | **active** (enabled)                   |
| `systemd-oomd`       | Kills memory hogs before the kernel OOM killer   | **active** (disabled, started on demand) |
| `systemd-networkd`   | Network configuration                            | inactive — **NetworkManager** is used instead → [[network-manager]] |
| `systemd-tmpfiles`   | Creates/cleans temp dirs (not a daemon, a timer) | `systemd-tmpfiles-clean.timer`         |

> [!note] `static` vs `enabled`
> `static` = the unit has no `[Install]` section, so you can't enable/disable it. Something else pulls it in (another unit, a socket, D-Bus). See [[units]].

# The three commands you need
| Command        | For                                                    |
| -------------- | ------------------------------------------------------ |
| `systemctl`    | Control units: start, stop, enable, status → [[systemctl-cheatsheet]] |
| `journalctl`   | Read logs → [[journal]]                                |
| `systemd-analyze` | Boot time, verify unit files, test calendar syntax → [[targets-boot]], [[timers]] |

Others: `loginctl` (sessions), `resolvectl` (DNS), `timedatectl` (time), `hostnamectl` (hostname), `systemd-run` (run a command as a temporary unit), `systemd-cgtop` (top for cgroups).

# Quick look at this machine
```bash
systemctl --version | head -1                 # systemd 262
systemctl get-default                         # graphical.target
systemctl list-units --failed                 # 0 loaded units listed. (good)
systemctl list-units --type=service --state=running | head
systemd-analyze                               # boot took 32.153s
```

Unit files on this system (`systemctl list-unit-files | …`):

| Type        | Count |
| ----------- | ----- |
| `service`   | 431   |
| `socket`    | 126   |
| `target`    | 82    |
| `timer`     | 28    |
| `mount`     | 21    |
| `slice`     | 9     |
| `path`      | 7     |
| `swap`      | 2     |
| `scope`     | 2     |
| `automount` | 1     |

# Notes in this folder
1. [[units]] — unit types, file locations, drop-ins, dependencies, how `enable` works
2. [[services]] — writing `.service` files, `Type=`, restart, hardening, full example
3. [[timers]] — scheduled jobs, `OnCalendar` syntax, cron comparison
4. [[journal]] — `journalctl` recipes, priorities, storage
5. [[targets-boot]] — boot process, targets, `systemd-analyze`, rescue mode
6. [[user-units]] — `systemctl --user`, lingering
7. [[systemctl-cheatsheet]] — all commands on one page, scripting helpers

# Find stuff
| Looking for                         | Command / path                                  |
| ----------------------------------- | ----------------------------------------------- |
| Which unit runs a process           | `ps -o unit= -p <pid>` or `systemctl status <pid>` |
| Unit file of a service              | `systemctl cat <unit>` (shows path on 1st line) |
| Package defaults                    | `/usr/lib/systemd/system/` → [[usr]]            |
| Your changes                        | `/etc/systemd/system/` → [[etc]]                |
| Runtime units, sockets              | `/run/systemd/` → [[run]]                       |
| Logs                                | `/var/log/journal/` → [[var]], [[journal]]      |
| cgroup of a service                 | `/sys/fs/cgroup/system.slice/<unit>/` → [[sys]] |
| Config of systemd itself            | `/etc/systemd/system.conf`, `journald.conf`, `logind.conf` |
| Everything you changed vs defaults  | `systemd-delta`                                 |

# See also
- [[networking]] — NetworkManager and resolved are both systemd units
- [[boot]] — what happens before systemd starts
- [[proc]] — `/proc/1` is systemd
- `man systemd`, `man systemd.unit`, `man systemd.index` (list of all systemd man pages)
- <https://systemd.io>
- `man systemctl`, `man journalctl`
