---
tags:
  - linux
  - systemd
  - user
  - services
aliases:
  - systemctl --user
  - user services
  - lingering
---
Besides the system manager (PID 1), systemd starts a **second manager for every logged-in user**: `user@UID.service`. It runs as **you**, not root, and manages your own services and timers. No `sudo` needed.

> [!info] Two managers
> | | System manager | User manager |
> |---|---|---|
> | Process | PID 1, runs as root | `systemd --user`, runs as you |
> | Command | `systemctl …` | `systemctl --user …` |
> | Logs | `journalctl -u name` | `journalctl --user -u name` |
> | Default target | `graphical.target` | `default.target` |
> | Lives in | `system.slice` | `user.slice/user-1000.slice/user@1000.service` |
>
> On a desktop most things you see running are user units: sound server (`pipewire`, `wireplumber`), D-Bus session bus, the apps started from the menu (`app-*.service`).

Real state (`systemctl --user status`):
```
● myhost
    State: running
    Units: 367 loaded (incl. loaded aliases)
   Failed: 0 units
    Since: Sun 2026-10-04 16:57:37 UTC; 55min ago
   CGroup: /user.slice/user-1000.slice/user@1000.service
```

# When does the user manager run?
- **Starts** at your first login (graphical, tty or SSH) — `logind` starts `user@1000.service`.
- **Stops** when your **last** session ends — and kills all your user services with it.

```
$ loginctl list-sessions
SESSION  UID USER  SEAT  LEADER CLASS   TTY  IDLE SINCE
      2 1000 alice seat0 1297   user    tty2 no   -
      3 1000 alice -     1303   manager -    no   -
```
Session `3` with class `manager` is the user manager itself.

# Lingering
**Lingering** = keep your user manager running even when you are not logged in, and start it at **boot**. Needed for user services that should run 24/7 (a home server, a sync daemon).

```bash
loginctl show-user $USER -p Linger      # Linger=no  (here)
sudo loginctl enable-linger $USER        # described: turn on
loginctl disable-linger $USER            # turn off
ls /var/lib/systemd/linger/              # one empty file per lingering user
```

# `XDG_RUNTIME_DIR`
Every user manager gets a private tmpfs: `/run/user/<UID>` (here `/run/user/1000`). It is created at login, deleted when the user manager stops. → [[run]]

| Inside `/run/user/1000/` | What                                 |
| ------------------------ | ------------------------------------ |
| `bus`                    | Session D-Bus socket (`DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus`) |
| `systemd/`               | User manager's private state         |
| `pipewire-0`             | Sound server socket                  |
| `wayland-0`              | Wayland display socket               |

> [!warning] `systemctl --user` over `sudo`/`su`/cron fails
> `Failed to connect to user scope bus` = the shell has no `XDG_RUNTIME_DIR` / `DBUS_SESSION_BUS_ADDRESS`. Log in properly (SSH, tty), or use
> `systemctl --user -M alice@ status` from root.

# Where user units live
| Path                                  | Who                                          |
| ------------------------------------- | -------------------------------------------- |
| `~/.config/systemd/user/`             | **You** — highest priority → [[home]]        |
| `/etc/systemd/user/`                  | Admin, for all users → [[etc]]               |
| `/run/systemd/user/`                  | Runtime                                      |
| `/usr/lib/systemd/user/`              | Packages (128 files here) → [[usr]]          |

Enabled units are symlinks in `~/.config/systemd/user/default.target.wants/` — same mechanism as system units ([[units]]).

Enabled user units here (`systemctl --user list-unit-files --state=enabled`):
```
wireplumber.service      enabled enabled
xdg-user-dirs.service    enabled enabled
pipewire-pulse.socket    enabled enabled
pipewire.socket          enabled enabled
p11-kit-server.socket    enabled enabled
```

# Differences from system units
| In a user unit…                                     | Because                                       |
| --------------------------------------------------- | --------------------------------------------- |
| Use `WantedBy=default.target` (not `multi-user.target`) | User manager has its own targets          |
| No `User=` / `Group=`                               | It already runs as you                        |
| Can't depend on system units (`After=network-online.target` does nothing) | Separate managers |
| `%h` = your home, `%t` = `/run/user/UID`            | Specifiers → [[units]]                        |
| Environment comes from `systemctl --user show-environment` | Not from your `.bashrc`!               |

# Example: personal service + timer
Goal: every 30 minutes, write your disk usage to a file — no root at all.

**`~/.config/systemd/user/diskreport.service`**
```ini
[Unit]
Description=Write disk usage report

[Service]
Type=oneshot
ExecStart=/usr/bin/bash -c 'df -h / > %h/diskreport.txt'
```

**`~/.config/systemd/user/diskreport.timer`**
```ini
[Unit]
Description=Disk usage report every 30 minutes

[Timer]
OnCalendar=*:0/30
Persistent=true

[Install]
WantedBy=timers.target
```

**Install** (described, run yourself — no sudo):
1. `mkdir -p ~/.config/systemd/user` and save both files there.
2. `systemctl --user daemon-reload`
3. `systemctl --user start diskreport.service` — test; then `cat ~/diskreport.txt`.
4. `systemctl --user enable --now diskreport.timer`
5. `systemctl --user list-timers` — check next run.
6. Want it to run while logged out? `sudo loginctl enable-linger $USER`.

A long-running user service looks the same as in [[services]], just with `WantedBy=default.target`.

> [!tip] Test without files
> ```bash
> systemd-run --user --wait -p Type=oneshot /usr/bin/true          # tested here: works
> systemd-run --user --on-active=10min notify-send "Stand up"
> ```

# Scripting recipes
### Is my user service running?
```bash
systemctl --user is-active --quiet pipewire && echo "sound server up"
```

### My failed user units
```bash
systemctl --user list-units --failed --no-legend --plain
```

### Which user units use most memory
```bash
systemctl --user list-units --type=service --state=running --no-legend --plain | awk '{print $1}' |
while read -r u; do
  m=$(systemctl --user show -P MemoryCurrent "$u")
  [[ $m =~ ^[0-9]+$ ]] && echo "$m $u"
done | sort -rn | head -5 | awk '{printf "%8.1f MiB  %s\n", $1/1048576, $2}'
```
Browsers and chat apps started from the menu show up here as `app-….service`.

### Set a variable for all user services
```bash
systemctl --user set-environment MYVAR=1         # current session only
# permanent: ~/.config/environment.d/10-my.conf  →  MYVAR=1
```

# Find stuff
| Looking for                        | Command                                     |
| ---------------------------------- | ------------------------------------------- |
| Running user units                 | `systemctl --user list-units --state=running` |
| User timers                        | `systemctl --user list-timers`              |
| Your unit files                    | `~/.config/systemd/user/`                   |
| User logs                          | `journalctl --user -u <unit>`               |
| Sessions                           | `loginctl list-sessions`, `loginctl session-status` |
| Lingering                          | `loginctl show-user $USER -p Linger`        |
| User manager's environment         | `systemctl --user show-environment`         |
| User cgroup tree                   | `systemd-cgls --user-unit`                  |

# See also
- [[services]], [[timers]] — unit syntax
- [[units]] — locations and `enable`
- [[run]] — `/run/user/UID`
- [[home]] — `~/.config`
- `man systemd.special` (user targets), `man loginctl`, `man environment.d`
