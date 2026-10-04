---
tags:
  - linux
  - systemd
  - services
  - scripting
aliases:
  - service files
  - .service
  - daemons
---
A `.service` unit tells systemd **how to run a program**: what command, as which user, what to do when it crashes, which resources it may touch. It is the most common unit type (431 of them on this machine). General unit file rules are in [[units]].

> [!info] Why use a service instead of `nohup script &`
> - Starts at boot, in the right order (after network, after a mount …)
> - Restarts automatically when it crashes
> - Logs go to the journal for free (`journalctl -u name`) → [[journal]]
> - Clean stop: systemd kills the **whole cgroup**, no orphaned children
> - Resource accounting and limits (memory, CPU) → [[sys]] `/sys/fs/cgroup`

# The `[Service]` section
| Key                    | Meaning                                                                  |
| ---------------------- | ------------------------------------------------------------------------ |
| `Type=`                | How systemd knows the service is "started" (table below)                 |
| `ExecStart=`           | The command. **Absolute path** required, no shell features (no `|`, `>`, `&&`) |
| `ExecStartPre=` / `ExecStartPost=` | Run before/after main command                              |
| `ExecStop=`            | Custom stop command (default: send `SIGTERM`)                            |
| `ExecReload=`          | What `systemctl reload` does (often `kill -HUP $MAINPID`)                |
| `Restart=`             | When to restart (table below)                                            |
| `RestartSec=`          | Wait before restart (default `100ms`)                                    |
| `User=` / `Group=`     | Run as this user instead of root                                         |
| `WorkingDirectory=`    | `cwd` of the process                                                     |
| `Environment=`         | `Environment="KEY=value" "OTHER=x"`                                      |
| `EnvironmentFile=`     | Load `KEY=value` lines from a file (`-/path` = optional)                 |
| `StandardOutput=`      | `journal` (default), `null`, `file:/path`, `append:/path`                |
| `TimeoutStartSec=`     | Give up starting after this long                                         |
| `TimeoutStopSec=`      | `SIGKILL` if still alive after this long (default 90 s)                  |

> [!tip] Need shell features?
> `ExecStart=` is **not** run by a shell. For pipes/redirects either call a script, or:
> `ExecStart=/usr/bin/bash -c 'mycmd | grep foo > /tmp/out'`

### `Type=`
| Type       | systemd considers it started when …                    | Use for                                       |
| ---------- | ------------------------------------------------------ | --------------------------------------------- |
| `simple`   | Immediately after `fork()` (default)                   | Programs that stay in foreground              |
| `exec`     | After the binary was successfully `exec`ed             | Same as simple, but reports a bad path as failure — **prefer this** |
| `forking`  | Parent process exits (classic daemon that backgrounds itself) | Old daemons; set `PIDFile=`            |
| `oneshot`  | Process **exits** (successfully)                       | Scripts that do a job and finish (backup, setup) |
| `notify`   | Program sends `READY=1` via `sd_notify()`              | systemd-aware daemons (`systemd-timesyncd`, `docker`) |
| `dbus`     | Program takes its `BusName=` on D-Bus                  | D-Bus services                                |
| `idle`     | Like simple, but waits until boot jobs are done        | Console output that shouldn't mix with boot   |

`oneshot` + `RemainAfterExit=yes` = unit stays "active" after the script finished (useful for "setup done" states).

### `Restart=`
| Value          | Restart when exits with … |
| -------------- | ------------------------- |
| `no`           | never (default)           |
| `on-failure`   | non-zero code, signal, timeout, watchdog — **most common** |
| `on-abnormal`  | signal, timeout, watchdog (not non-zero code) |
| `on-abort`     | uncaught signal only      |
| `always`       | always, even clean exit 0 |

> [!warning] Restart loops
> By default a unit that restarts more than **5 times in 10 s** is given up (`StartLimitBurst=`, `StartLimitIntervalSec=` in `[Unit]`). It then shows `failed (Result: start-limit-hit)`. Reset with `systemctl reset-failed name`.

# Reading `status`
```
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/usr/lib/systemd/system/systemd-timesyncd.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-10-04 16:57:28 UTC; 50min ago
   Main PID: 572 (systemd-timesyn)
     Status: "Contacted time server 162.159.200.1:123 (time.cloudflare.com)."
      Tasks: 1 (limit: 38308)
     Memory: 1.1M (peak: 3.6M)
        CPU: 34ms
     CGroup: /system.slice/systemd-timesyncd.service
             └─572 /usr/lib/systemd/systemd-timesyncd
```
Line by line explained in [[systemctl-cheatsheet]].

# Hardening basics
systemd can sandbox a service with one line per restriction. `systemd-timesyncd` on this machine uses many of them (`systemctl cat systemd-timesyncd`):

| Setting                        | Effect                                                   |
| ------------------------------ | -------------------------------------------------------- |
| `NoNewPrivileges=yes`          | Can't gain privileges (setuid, sudo won't work inside)   |
| `ProtectSystem=strict`         | Whole filesystem read-only except `ReadWritePaths=`      |
| `ProtectHome=yes`              | `/home`, `/root`, `/run/user` invisible                  |
| `PrivateTmp=yes`               | Own private `/tmp` and `/var/tmp`                        |
| `PrivateDevices=yes`           | Only `/dev/null`, `zero`, `random` …                     |
| `DynamicUser=yes`              | Runs as a throwaway UID created at start                 |
| `CapabilityBoundingSet=`       | Limit root powers (`CAP_SYS_TIME` only, for timesyncd)   |
| `MemoryDenyWriteExecute=yes`   | No writable+executable memory (blocks many exploits)     |
| `MemoryMax=` / `CPUQuota=`     | Resource limits (`MemoryMax=500M`, `CPUQuota=50%`)       |

Score any service: `systemd-analyze security NetworkManager` (0 = locked down, 10 = wide open).

# Full example: run your own script as a service
Goal: a script that writes system stats every minute to a log, starts at boot, restarts if it crashes.

**1. The script** — `/usr/local/bin/statlogger` (see [[usr]] for why `/usr/local`)
```bash
#!/usr/bin/env bash
while true; do
  load=$(cut -d' ' -f1 /proc/loadavg)
  mem=$(awk '/^MemAvailable:/{print $2}' /proc/meminfo)
  echo "load=$load mem_available_kb=$mem"     # stdout goes to the journal
  sleep 60
done
```

**2. The unit** — `/etc/systemd/system/statlogger.service`
```ini
[Unit]
Description=Log load and memory every minute
After=local-fs.target

[Service]
Type=exec
ExecStart=/usr/local/bin/statlogger
Restart=on-failure
RestartSec=5
User=nobody
Environment="LANG=C"

# sandbox
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

**3. Install** (described, run these yourself):
1. `sudo install -m 755 statlogger /usr/local/bin/statlogger` — copy script, make executable.
2. `sudo cp statlogger.service /etc/systemd/system/`
3. `systemd-analyze verify /etc/systemd/system/statlogger.service` — check for typos.
4. `sudo systemctl daemon-reload` — systemd re-reads unit files (needed after **every** edit).
5. `sudo systemctl enable --now statlogger` — enable at boot and start now.
6. `systemctl status statlogger` — should show `active (running)`.
7. `journalctl -u statlogger -f` — watch the output live.

**Remove:** `sudo systemctl disable --now statlogger`, delete both files, `sudo systemctl daemon-reload`.

> [!tip] Don't need root? Make it a user service
> Put the unit in `~/.config/systemd/user/`, drop `User=`, use `WantedBy=default.target` and `systemctl --user …`. See [[user-units]].

> [!tip] Quick test without writing a file
> `systemd-run` turns any command into a temporary unit:
> ```bash
> systemd-run --user --wait -p Type=oneshot /usr/bin/true    # tested here: works
> systemd-run --user --unit=test-sleep sleep 300              # then: systemctl --user status test-sleep
> ```

# Scripting recipes
### Is a service running?
```bash
svc=NetworkManager
if systemctl is-active --quiet "$svc"; then echo "$svc is running"; else echo "$svc is NOT running"; fi
```

### Memory usage of all running services (top 5)
```bash
systemctl list-units --type=service --state=running --no-legend --plain | awk '{print $1}' |
while read -r u; do
  m=$(systemctl show -P MemoryCurrent "$u")
  [[ $m =~ ^[0-9]+$ ]] && echo "$m $u"
done | sort -rn | head -5 | awk '{printf "%8.1f MiB  %s\n", $1/1048576, $2}'
```
Output here: `docker.service 140 MiB`, `containerd.service 90 MiB` … (`user@1000.service` is at the top because it contains your whole desktop session).

### Wait until a service is up (max 30 s)
```bash
timeout 30 bash -c 'until systemctl is-active --quiet docker; do sleep 1; done' && echo up
```

### Main PID, restart count
```bash
systemctl show -P MainPID NetworkManager      # 915
systemctl show -P NRestarts NetworkManager    # 0
```

# Find stuff
| Looking for                     | Command                                          |
| ------------------------------- | ------------------------------------------------ |
| Running services                | `systemctl list-units --type=service --state=running` |
| Failed services                 | `systemctl --failed`                             |
| Service of a PID                | `ps -o unit= -p <pid>`                           |
| Processes of a service          | `systemctl status <unit>` (CGroup tree) or `systemd-cgls -u <unit>` |
| Its logs                        | `journalctl -u <unit>`                           |
| Security score                  | `systemd-analyze security <unit>`                |
| All `[Service]` options         | `man systemd.service`, `man systemd.exec`, `man systemd.resource-control` |

# See also
- [[units]] — dependencies, drop-ins, `[Install]`
- [[timers]] — run a service on a schedule
- [[journal]] — logs of services
- [[user-units]] — services without root
- [[systemctl-cheatsheet]]
- [[networking]] — `network-online.target` and NetworkManager
