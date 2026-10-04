---
tags:
  - linux
  - systemd
  - cheatsheet
  - scripting
aliases:
  - systemctl
  - systemctl cheat sheet
---
Every `systemctl` command on one page. Add `--user` to any of them to talk to your user manager instead of the system one ([[user-units]]). Commands that change something need `sudo` for system units.

> [!tip] Unit name shortcuts
> `.service` can be left out: `systemctl status docker` = `docker.service`. Globs work: `systemctl status 'systemd-*'`. A PID works too: `systemctl status 915`.

# Look
| Command                                         | Shows                                          |
| ----------------------------------------------- | ---------------------------------------------- |
| `systemctl status <unit>`                       | State, PID, memory, last log lines (explained below) |
| `systemctl status`                              | Whole system tree                              |
| `systemctl cat <unit>`                          | Unit file + all drop-ins                       |
| `systemctl show <unit>`                         | **All** properties as `Key=value`              |
| `systemctl show -p A,B <unit>`                  | Only these properties                          |
| `systemctl show -P A <unit>`                    | Only the **value** (best for scripts)          |
| `systemctl list-units`                          | Loaded + active units                          |
| `systemctl list-units --all`                    | Including inactive                             |
| `systemctl list-units --type=service --state=running` | Running services                         |
| `systemctl list-units --failed` / `--failed`    | Failed units                                   |
| `systemctl list-unit-files`                     | All installed units + enabled state            |
| `systemctl list-timers --all`                   | Timers → [[timers]]                            |
| `systemctl list-sockets`                        | Socket units and what they activate            |
| `systemctl list-dependencies <unit>`            | Dependency tree                                |
| `systemctl list-dependencies --reverse <unit>`  | Who depends on it                              |
| `systemctl list-jobs`                           | What systemd is doing right now (boot hangs!)  |

# Control (now)
| Command                         | Does                                                     |
| ------------------------------- | -------------------------------------------------------- |
| `start <unit>`                  | Start                                                    |
| `stop <unit>`                   | Stop                                                     |
| `restart <unit>`                | Stop + start                                             |
| `reload <unit>`                 | Re-read config without stopping (if the service supports it) |
| `reload-or-restart <unit>`      | Reload if possible, else restart                         |
| `try-restart <unit>`            | Restart only if already running                          |
| `kill <unit>`                   | Send signal to all processes (`-s SIGHUP`)               |
| `reset-failed [<unit>]`         | Clear "failed" state and restart counter                 |
| `isolate <target>`              | Switch to a target, stop everything else → [[targets-boot]] |

# Boot-time (enable)
| Command                         | Does                                               |
| ------------------------------- | -------------------------------------------------- |
| `enable <unit>`                 | Start at boot (creates symlinks) → [[units]]       |
| `enable --now <unit>`           | Enable + start                                     |
| `disable <unit>`                | Don't start at boot                                |
| `disable --now <unit>`          | Disable + stop                                     |
| `mask <unit>`                   | Make impossible to start (link to `/dev/null`)     |
| `unmask <unit>`                 | Undo                                               |
| `preset <unit>`                 | Reset enabled state to vendor default              |
| `get-default` / `set-default`   | Default boot target                                |

# Edit
| Command                           | Does                                                |
| --------------------------------- | --------------------------------------------------- |
| `edit <unit>`                     | Create/edit drop-in `override.conf` (auto reload)   |
| `edit --full <unit>`              | Copy whole unit to `/etc` and edit                  |
| `revert <unit>`                   | Delete your overrides                               |
| `daemon-reload`                   | Re-read all unit files — **after every manual edit**|
| `set-property <unit> MemoryMax=1G`| Change resource limit live (persistent)             |
| `set-property --runtime …`        | Same, until reboot                                  |

# Power
`poweroff`, `reboot`, `suspend`, `hibernate`, `hybrid-sleep`, `soft-reboot` (restart userspace only, keep kernel), `rescue`, `emergency`.

# Reading `status`
```
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/usr/lib/systemd/system/systemd-timesyncd.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-10-04 16:57:28 UTC; 50min ago
 Invocation: dcfa18ee17e24d0492e574e2e632f40d
       Docs: man:systemd-timesyncd.service(8)
   Main PID: 572 (systemd-timesyn)
     Status: "Contacted time server 162.159.200.1:123 (time.cloudflare.com)."
      Tasks: 1 (limit: 38308)
     Memory: 1.1M (peak: 3.6M)
        CPU: 34ms
     CGroup: /system.slice/systemd-timesyncd.service
             └─572 /usr/lib/systemd/systemd-timesyncd

Oct 04 16:57:28 myhost systemd[1]: Starting Network Time Synchronization...
```

| Part              | Meaning                                                                        |
| ----------------- | ------------------------------------------------------------------------------ |
| `●` dot           | green = active, white `○` = inactive, red `×` = failed                         |
| `Loaded:`         | Unit file path; `enabled`/`disabled`/`static`; `preset:` = distro default      |
| `Drop-In:`        | (if any) override files → [[units]]                                           |
| `Active:`         | High-level state + sub-state + since when (table below)                        |
| `Invocation:`     | ID of this particular run (changes on every restart)                           |
| `Main PID:`       | Main process → `/proc/572/` in [[proc]]                                       |
| `Status:`         | Free text the service reports itself (only `Type=notify` services)             |
| `Tasks:`          | Processes + threads; limit = `TasksMax=`                                       |
| `Memory:`         | cgroup memory now and peak → `/sys/fs/cgroup/system.slice/<unit>/memory.current` in [[sys]] |
| `CPU:`            | Total CPU time used                                                            |
| `CGroup:`         | All processes of the service as a tree                                         |
| last lines        | Last 10 journal entries → [[journal]] (`-n 50` for more)                      |

### Active states
| `ActiveState`  | Typical `SubState`             | Meaning                                    |
| -------------- | ------------------------------ | ------------------------------------------ |
| `active`       | `running`                      | Service process is running                 |
| `active`       | `exited`                       | `oneshot` + `RemainAfterExit=yes`, done    |
| `active`       | `waiting`, `listening`, `mounted`, `plugged` | Timer/path waiting, socket listening, mount mounted, device present |
| `inactive`     | `dead`                         | Not running                                |
| `activating`   | `start`, `auto-restart`        | Starting / waiting to restart              |
| `deactivating` | `stop-sigterm`                 | Stopping                                   |
| `failed`       | `failed`                       | Crashed / non-zero exit / start limit hit  |
| `reloading`    |                                | Reloading config                           |

# For scripts: exit codes instead of parsing text
| Command                          | Exit 0 when …                 | Prints                              |
| -------------------------------- | ----------------------------- | ----------------------------------- |
| `systemctl is-active <unit>`     | active                        | `active`, `inactive`, `failed` …    |
| `systemctl is-enabled <unit>`    | enabled (also static, alias…) | `enabled`, `disabled`, `static`, `masked` … |
| `systemctl is-failed <unit>`     | **failed**                    | `failed`, `active` …                |
| `systemctl is-system-running`    | fully booted, nothing failed  | `running`, `degraded`, `starting` … |

Add `--quiet` (`-q`) to print nothing — just use the exit code.

```bash
if systemctl is-active -q docker; then echo "docker up"; fi
systemctl is-enabled -q sshd || echo "sshd won't start at boot"
```

### Output flags for parsing
| Flag            | Effect                                         |
| --------------- | ---------------------------------------------- |
| `--no-legend`   | No header / footer lines                       |
| `--plain`       | No tree drawing, no `●`                        |
| `--no-pager`    | Never open `less`                              |
| `-o json` *(list commands)* | JSON output: `systemctl list-units -o json` |
| `-P <prop>`     | Value of one property (`show`)                 |
| `--value`       | Same as `-P` together with `-p`                |

### Useful properties for `show -P`
| Property          | Example value                                        |
| ----------------- | ---------------------------------------------------- |
| `ActiveState`     | `active`                                             |
| `SubState`        | `running`                                            |
| `MainPID`         | `915`                                                |
| `MemoryCurrent`   | `25464832` (bytes)                                   |
| `NRestarts`       | `0`                                                  |
| `FragmentPath`    | `/usr/lib/systemd/system/NetworkManager.service`     |
| `DropInPaths`     | override files, empty = none                         |
| `ExecMainStartTimestamp` | when the main process started                 |
| `Result`          | `success`, `exit-code`, `signal`, `timeout`          |
| `UnitFileState`   | `enabled`                                            |

# Scripting recipes
### Health check: print failed units, exit 1 if any
```bash
failed=$(systemctl list-units --failed --no-legend --plain | awk '{print $1}')
if [[ -n $failed ]]; then echo "FAILED: $failed"; exit 1; fi
echo "all units OK"
```

### Restart a service only if it's not running
```bash
svc=myapp
systemctl is-active -q "$svc" || sudo systemctl restart "$svc"
```

### Table of selected services
```bash
for s in NetworkManager systemd-resolved systemd-timesyncd sshd; do
  printf "%-22s %-9s %s\n" "$s" "$(systemctl is-active "$s")" "$(systemctl is-enabled "$s" 2>/dev/null)"
done
```
Output here:
```
NetworkManager         active    enabled
systemd-resolved       active    enabled
systemd-timesyncd      active    enabled
sshd                   inactive  disabled
```

### Which unit owns this process?
```bash
ps -o unit= -p 915          # NetworkManager.service
systemctl status 915        # full status of the owning unit
```

### JSON → jq
```bash
systemctl list-units --type=service --state=running -o json | jq -r '.[].unit'
```

# Find stuff
| Question                          | Command                                      |
| --------------------------------- | -------------------------------------------- |
| Is it running?                    | `systemctl is-active <unit>`                 |
| Why did it fail?                  | `systemctl status <unit>`, `journalctl -u <unit> -b` |
| Where is its file?                | `systemctl show -P FragmentPath <unit>`      |
| What did I override?              | `systemd-delta`                              |
| What's hanging at boot/shutdown?  | `systemctl list-jobs`                        |
| Resource use of all units live    | `systemd-cgtop`                              |
| Process tree by unit              | `systemd-cgls`                               |

# See also
- [[systemd]] — overview
- [[units]], [[services]], [[timers]], [[user-units]]
- [[journal]], [[targets-boot]]
- `man systemctl`
