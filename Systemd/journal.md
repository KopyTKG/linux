---
tags:
  - linux
  - systemd
  - logs
  - journald
aliases:
  - journalctl
  - journald
  - systemd logs
---
**journald** (`systemd-journald.service`) collects every log message on the system — kernel, services, user apps, stdout of your scripts — into one indexed binary database. You read it with `journalctl`.

> [!info] Why binary?
> Every entry is stored with metadata fields (unit, PID, UID, boot ID, priority …). That's why you can filter by service, boot or time instantly — no `grep` through text files. Classic text logs in `/var/log/*.log` still exist for some programs. → [[var]]

# Where it is stored
| Path                                  | When                                          |
| ------------------------------------- | --------------------------------------------- |
| `/var/log/journal/<machine-id>/`      | **Persistent** — survives reboot (used here)  |
| `/run/log/journal/<machine-id>/`      | **Volatile** — RAM only, lost on reboot → [[run]] |

`Storage=auto` (default) = persistent **if** `/var/log/journal/` exists. Machine ID is in `/etc/machine-id`. → [[etc]]

```bash
$ journalctl --disk-usage
Archived and active journals take up 43.5M in the file system.
```

Config: `/etc/systemd/journald.conf` (all defaults are commented out) or drop-ins in `/etc/systemd/journald.conf.d/*.conf`.

| Setting            | Meaning                                         |
| ------------------ | ----------------------------------------------- |
| `Storage=`         | `persistent`, `volatile`, `auto`, `none`        |
| `SystemMaxUse=`    | Max disk space (default 10 % of fs, max 4 G)    |
| `MaxRetentionSec=` | Delete entries older than this                  |
| `Compress=`        | Compress large entries (default yes)            |
| `ForwardToSyslog=` | Also send to a syslog daemon                    |

# Priorities
Same 8 levels as classic syslog. Filtering with `-p X` shows **X and everything more important**.

| Num | Name      | `-p` value | Meaning                     | Color in journalctl |
| --- | --------- | ---------- | --------------------------- | ------------------- |
| 0   | emerg     | `emerg`    | System unusable             | bold red            |
| 1   | alert     | `alert`    | Act immediately             | bold red            |
| 2   | crit      | `crit`     | Critical                    | bold red            |
| 3   | err       | `err`      | Error                       | red                 |
| 4   | warning   | `warning`  | Warning                     | yellow              |
| 5   | notice    | `notice`   | Normal but significant      | bold                |
| 6   | info      | `info`     | Informational               | normal              |
| 7   | debug     | `debug`    | Debug                       | grey                |

Range: `-p warning..err`.

# journalctl recipes
### By unit
```bash
journalctl -u NetworkManager             # one service, all time
journalctl -u nginx -u php-fpm           # several units merged
journalctl --user -u myapp               # a user unit → [[user-units]]
journalctl _PID=915                      # by PID
journalctl /usr/bin/sshd                 # by executable
```

### By boot
```bash
journalctl -b          # this boot
journalctl -b -1       # previous boot (great after a crash/freeze)
journalctl --list-boots
```
```
 -2 2b7e9c1d0f3a4e5b8c6d7a9f1e2b3c4d Fri 2026-10-02 09:37:15 UTC Sat 2026-10-03 00:44:32 UTC
 -1 8c1f2e3d4a5b4c6d9e7f0a1b2c3d4e5f Sat 2026-10-03 13:40:33 UTC Sat 2026-10-03 22:25:38 UTC
  0 4f1d2c3b5a6e4b7c9d8e0f1a2b3c4d5e Sun 2026-10-04 16:57:23 UTC Sun 2026-10-04 17:48:00 UTC
```

### By priority
```bash
journalctl -b -p err            # errors and worse, this boot
journalctl -b -p warning -u docker
```
Real output (`journalctl -b -p err`):
```
Oct 04 16:57:23 myhost kernel: virt/tdx: TDX not supported by the host platform
Oct 04 16:57:31 myhost wpa_supplicant[1092]: wlan0: nl80211: kernel reports: multicast RX registrations are not supported
```
> [!note] Errors ≠ problems
> Many `err` lines are harmless noise (this machine has ~100 per boot). Look for errors that **match the time** something broke.

### By time
```bash
journalctl --since "1 hour ago"
journalctl --since today
journalctl --since "2026-10-04 16:00" --until "2026-10-04 17:00"
journalctl --since -10m             # last 10 minutes
```

### Follow / last lines
```bash
journalctl -f                  # like tail -f, whole system
journalctl -u docker -f        # follow one service
journalctl -n 50               # last 50 lines
journalctl -r                  # newest first
journalctl -e                  # open pager at the end
```

### Kernel messages
```bash
journalctl -k          # same as dmesg, but with real timestamps
journalctl -k -b -1    # kernel log of previous boot (dmesg can't do this)
```

### Search text
```bash
journalctl -b -g 'usb|disconnect'     # --grep, regex, case-insensitive if all lowercase
```

### Output formats
| `-o` value      | Looks like                                    |
| --------------- | --------------------------------------------- |
| `short`         | Default, syslog style                         |
| `short-iso`     | `2026-10-04T16:57:28+02:00 myhost systemd[1]: …` |
| `short-precise` | Microsecond timestamps                        |
| `cat`           | Message only, no metadata (good for scripts)  |
| `json`          | One JSON object per line                      |
| `json-pretty`   | Readable JSON                                 |
| `verbose`       | All fields of each entry                      |

A JSON entry contains fields like:
```json
{"_SYSTEMD_UNIT":"systemd-timesyncd.service","SYSLOG_IDENTIFIER":"systemd-timesyncd",
 "_BOOT_ID":"67322f86…","_HOSTNAME":"myhost","PRIORITY":"6","MESSAGE":"…", …}
```
Any field can be a filter: `journalctl _SYSTEMD_UNIT=sshd.service PRIORITY=3`. List values of a field: `journalctl -F _SYSTEMD_UNIT`.

> [!tip] Add `--no-pager` in scripts
> Otherwise journalctl opens `less` and your script waits. `-q` hides "-- No entries --" style info lines.

# Disk cleanup (described, needs root)
```bash
sudo journalctl --vacuum-size=200M     # shrink to 200 MB
sudo journalctl --vacuum-time=2weeks   # delete older than 2 weeks
sudo journalctl --rotate               # close current files first, then vacuum
```
Permanent limit: put `SystemMaxUse=200M` under `[Journal]` in `/etc/systemd/journald.conf.d/size.conf`, then `sudo systemctl restart systemd-journald`.

> [!warning] Permissions
> A normal user sees their own logs plus system logs only if in group `systemd-journal`, `adm` or `wheel` (depends on distro). Otherwise use `sudo journalctl`.

# Writing to the journal from scripts
```bash
echo "backup done" | systemd-cat -t mybackup            # tag = mybackup
systemd-cat -t mybackup -p err echo "backup FAILED"     # with priority
logger -t mybackup "hello"                              # classic syslog way, also lands in journal
journalctl -t mybackup                                  # read back
```
In a service, plain `echo` already goes to the journal — prefix lines with `<3>` to mark them as `err` (`echo "<3>disk full"`).

# Scripting recipes
### Count errors this boot
```bash
journalctl -b -p err -q --no-pager | wc -l
```

### Messages of a unit, text only
```bash
journalctl -u systemd-timesyncd -b -o json --no-pager | jq -r '.MESSAGE'
# or without jq:
journalctl -u systemd-timesyncd -b -o cat --no-pager
```

### Noisiest units this boot
```bash
journalctl -b -o json --no-pager | jq -r '._SYSTEMD_UNIT // empty' | sort | uniq -c | sort -rn | head -5
```
Real top here: `user@1000.service` (desktop apps, 1575 lines), `init.scope` (PID 1 itself), `NetworkManager.service`.

### Alert when a unit logs an error (live)
```bash
journalctl -f -p err -o cat -u myapp | while read -r line; do
  notify-send "myapp error" "$line"
done
```

### Did the last shutdown crash?
```bash
journalctl -b -1 -n 20 --no-pager    # last lines of previous boot
```
A clean shutdown ends with systemd stopping units and unmounting filesystems (`Unmounted /tmp.`, `…mount: Deactivated successfully.`). If the log stops in the middle of normal activity, the machine froze or lost power.

# Find stuff
| Looking for                       | Command                                  |
| --------------------------------- | ---------------------------------------- |
| Why a service failed              | `systemctl status <unit>` then `journalctl -u <unit> -b` |
| Previous boot (after freeze)      | `journalctl -b -1 -p warning`            |
| Kernel / hardware / driver errors | `journalctl -k -p err`                   |
| Crashed programs (core dumps)     | `coredumpctl list`                       |
| Login attempts                    | `journalctl -u sshd`, `journalctl _COMM=sudo` |
| Log size                          | `journalctl --disk-usage`                |
| All field names                   | `journalctl -N`                          |
| Storage                           | `/var/log/journal/` → [[var]]            |

# See also
- [[services]] — stdout of services goes here
- [[targets-boot]] — boot problems
- [[systemctl-cheatsheet]]
- [[var]] — classic log files in `/var/log`
- `man journalctl`, `man systemd.journal-fields`, `man journald.conf`
