---
tags:
  - linux
  - systemd
  - timers
  - scheduling
aliases:
  - systemd timers
  - cron replacement
  - OnCalendar
---
A `.timer` unit **starts another unit at a given time** — usually a `.service` with the same name (`fstrim.timer` → `fstrim.service`). It is systemd's replacement for cron.

> [!info] No cron on this machine
> `crontab` is not installed and `cronie` is inactive. Every scheduled job here (logrotate, fstrim, man-db, tmpfiles cleanup …) is a systemd timer. Many modern distributions ship without cron by default. → [[etc]]

# Timers vs cron
|                         | cron                              | systemd timer                                   |
| ----------------------- | --------------------------------- | ----------------------------------------------- |
| Files                   | 1 line in crontab                 | 2 files (`.timer` + `.service`)                 |
| Logs                    | mail / your own redirect          | `journalctl -u name` automatically              |
| Missed run (PC was off) | lost (unless anacron)             | `Persistent=true` runs it at next boot          |
| Random delay            | no                                | `RandomizedDelaySec=` (spread load)             |
| Run relative to boot    | `@reboot` only                    | `OnBootSec=15min`, `OnUnitActiveSec=1h`         |
| Run manually            | copy the command                  | `systemctl start name.service`                  |
| Resource limits, sandbox| no                                | everything from [[services]]                    |
| Overlapping runs        | possible                          | impossible: a running service isn't started twice |
| Test schedule syntax    | no                                | `systemd-analyze calendar '...'`                |

# Real timers on this machine
`systemctl list-timers`:
```
NEXT                          LEFT     LAST                          PASSED       UNIT                          ACTIVATES
Mon 2026-10-05 00:00:00 UTC  6h       Sun 2026-10-04 16:57:30 UTC  50min ago    shadow.timer                  shadow.service
Mon 2026-10-05 00:08:08 UTC  6h       Sun 2026-10-04 17:46:08 UTC  1min 44s ago logrotate.timer               logrotate.service
Mon 2026-10-05 00:19:36 UTC  6h       Sun 2026-10-04 17:00:13 UTC  47min ago    plocate-updatedb.timer        plocate-updatedb.service
Mon 2026-10-05 00:56:01 UTC  7h       Mon 2026-09-28 00:07:02 UTC  -            fstrim.timer                  fstrim.service
Mon 2026-10-05 17:12:21 UTC  23h      Sun 2026-10-04 17:12:21 UTC  35min ago    systemd-tmpfiles-clean.timer  systemd-tmpfiles-clean.service
```
Desktop apps can also install **user** timers (`systemctl --user list-timers`). See [[user-units]].

A real timer file — `/usr/lib/systemd/system/fstrim.timer`:
```ini
[Unit]
Description=Discard unused filesystem blocks once a week
Documentation=man:fstrim
ConditionVirtualization=!container

[Timer]
OnCalendar=weekly
AccuracySec=1h
Persistent=true
RandomizedDelaySec=100min

[Install]
WantedBy=timers.target
```

# `[Timer]` keys
| Key                     | Meaning                                                        |
| ----------------------- | -------------------------------------------------------------- |
| `OnCalendar=`           | Wall-clock time (syntax below). Can appear multiple times      |
| `OnBootSec=`            | X after boot (`OnBootSec=10min`)                               |
| `OnStartupSec=`         | X after systemd (the manager) started — for user timers = login |
| `OnActiveSec=`          | X after the timer itself was activated                         |
| `OnUnitActiveSec=`      | X after the service **last started** (repeat interval)         |
| `OnUnitInactiveSec=`    | X after the service **last finished**                          |
| `Persistent=true`       | If a run was missed (PC off), run once at next boot            |
| `RandomizedDelaySec=`   | Add random delay up to X                                       |
| `AccuracySec=`          | How precise (default `1min`; set `1s` if you need precision)   |
| `Unit=`                 | Activate a differently named unit                              |
| `WakeSystem=true`       | Wake the PC from suspend                                       |

Time units: `s`, `min`, `h`, `d`, `w`, `M`, `y` — combine: `1h 30min`.

# `OnCalendar` syntax
Full form: `DayOfWeek Year-Month-Day Hour:Minute:Second`. `*` = any, `,` = list, `..` = range, `/N` = every N.

Tested with `systemd-analyze calendar` on this machine:

| Expression         | Normalized form              | Means                              | Next run (from Sun 2026-10-04 17:48) |
| ------------------ | ---------------------------- | ---------------------------------- | ------------------------------------ |
| `daily`            | `*-*-* 00:00:00`             | Every midnight                     | Mon 2026-10-05 00:00                 |
| `weekly`           | `Mon *-*-* 00:00:00`         | Monday midnight                    | Mon 2026-10-05 00:00                 |
| `*:0/15`           | `*-*-* *:00/15:00`           | Every 15 minutes                   | Sun 2026-10-04 18:00                 |
| `Mon..Fri 09:00`   | `Mon..Fri *-*-* 09:00:00`    | Weekdays at 9:00                   | Mon 2026-10-05 09:00                 |
| `*-*-01 03:00:00`  | `*-*-01 03:00:00`            | 1st of every month 3:00            | Sun 2026-11-01 03:00                 |
| `Sat,Sun 10:00`    | `Sat,Sun *-*-* 10:00:00`     | Weekends at 10:00                  | Sat 2026-10-10 10:00                 |

Other shortcuts: `minutely`, `hourly`, `monthly`, `yearly`, `quarterly`, `semiannually`.

```bash
systemd-analyze calendar 'Mon..Fri 09:00'              # check syntax + next run
systemd-analyze calendar --iterations=5 '*:0/15'       # next 5 runs
```

> [!warning] Cron habits that don't work
> - `*/15` (cron) is `*:0/15` in systemd. The `/` goes **after** a start value.
> - Day of week is a **name** (`Mon`), not a number.
> - Timer time zone is the system's local time unless you append a zone: `OnCalendar=*-*-* 09:00 Europe/Berlin`.

# Example: daily backup timer + service
**`/etc/systemd/system/backup-home.service`**
```ini
[Unit]
Description=Backup home to external disk

[Service]
Type=oneshot
ExecStart=/usr/bin/rsync -a --delete /home/alice/Documents/ /mnt/backup/Documents/
```
No `[Install]` section: the timer starts it, it is never enabled itself.

**`/etc/systemd/system/backup-home.timer`**
```ini
[Unit]
Description=Daily home backup

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=30min

[Install]
WantedBy=timers.target
```

**Install** (described, run yourself):
1. Copy both files to `/etc/systemd/system/`.
2. `sudo systemctl daemon-reload`
3. `sudo systemctl start backup-home.service` — test the job once by hand; check `journalctl -u backup-home`.
4. `sudo systemctl enable --now backup-home.timer` — enable the **timer**, not the service.
5. `systemctl list-timers backup-home.timer` — shows next run.

Per-user version (no root): same files in `~/.config/systemd/user/`, `WantedBy=timers.target`, commands with `--user`. See [[user-units]].

> [!tip] One-off "run this later" without files
> ```bash
> systemd-run --user --on-active=30min /usr/bin/notify-send "Tea is ready"
> systemd-run --user --on-calendar='18:00' /usr/local/bin/myjob
> ```

# Scripting recipes
### When did a timer last run and was it OK?
```bash
t=logrotate
systemctl show -P LastTriggerUSec "$t.timer"                  # last trigger time
systemctl show -P Result "$t.service"                         # success / exit-code
journalctl -u "$t.service" -n 5 --no-pager                    # last output
```

### List timer names only
```bash
systemctl list-timers --all --no-legend --plain | awk '{print $(NF-1)}'
```

### Timers that never ran
```bash
for t in $(systemctl list-timers --all --no-legend --plain | awk '{print $(NF-1)}'); do
  [[ -z $(systemctl show -P LastTriggerUSec "$t") ]] && echo "never ran: $t"
done
```
Prints nothing here: every timer on this machine has run at least once. (Don't use the `PASSED` column for this; it shows `-` for some timers that did run, e.g. `fstrim.timer`.)

# Find stuff
| Looking for                     | Command                                    |
| ------------------------------- | ------------------------------------------ |
| All timers + next/last run      | `systemctl list-timers --all`              |
| User timers                     | `systemctl --user list-timers`             |
| What a timer starts             | `systemctl show -P Triggers <name>.timer`  |
| Validate a schedule             | `systemd-analyze calendar '<expr>'`        |
| Timer files                     | `/usr/lib/systemd/system/*.timer`, `/etc/systemd/system/*.timer` |
| Timer docs                      | `man systemd.timer`, `man systemd.time`    |

# See also
- [[services]] — the job a timer starts
- [[journal]] — output of timer jobs
- [[user-units]] — personal timers
- [[systemctl-cheatsheet]]
- `man systemd.timer`, `man systemd.time`
