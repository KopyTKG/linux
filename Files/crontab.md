---
tags:
  - linux
  - files
  - scheduling
  - cron
aliases:
  - crontab
  - /etc/crontab
  - cron
  - /etc/cron.d
  - cron.daily
---
**cron** is the classic Unix job scheduler: a daemon wakes up every minute, reads the crontabs and runs whatever is due. Many systems now use **systemd timers** instead (→ [[timers]]), and some don't install a cron daemon at all. Check with:

```bash
command -v crontab || echo "no cron installed"
systemctl list-unit-files | grep -Ei 'cron'   # cron.service / crond.service / cronie.service
```

| Location                     | Who edits it         | Has a user field? |
| ---------------------------- | -------------------- | ----------------- |
| `crontab -e` (per user)      | each user            | no — runs as that user |
| `/etc/crontab`               | root                 | **yes**           |
| `/etc/cron.d/*`              | root, packages       | **yes**           |
| `/etc/cron.hourly/`, `.daily/`, `.weekly/`, `.monthly/` | root, packages | — these are plain **scripts**, run by `run-parts` or anacron |

User crontabs are stored in `/var/spool/cron/` (path differs: `/var/spool/cron/crontabs/<user>` on Debian, `/var/spool/cron/<user>` elsewhere) — never edit those directly, always use `crontab -e`. → [[var]]

> [!info] `cron.hourly` etc. without cron
> A directory like `/etc/cron.hourly/` may exist even when no cron daemon is installed — a package dropped a script there in case one is. Without cron, those scripts simply never run (the package normally ships a systemd timer for the same job).

# Line format

```
┌───────── minute        (0–59)
│ ┌─────── hour          (0–23)
│ │ ┌───── day of month  (1–31)
│ │ │ ┌─── month         (1–12 or jan–dec)
│ │ │ │ ┌─ day of week   (0–7, 0 and 7 = Sunday, or sun–sat)
│ │ │ │ │
* * * * *  command                     ← user crontab
* * * * *  user  command               ← /etc/crontab and /etc/cron.d/*
```

| Syntax     | Meaning               | Example                        |
| ---------- | --------------------- | ------------------------------ |
| `*`        | every value           | `* * * * *` every minute       |
| `,`        | list                  | `0 8,12,18 * * *` three times a day |
| `-`        | range                 | `0 9-17 * * 1-5` hourly, office hours |
| `/`        | step                  | `*/15 * * * *` every 15 minutes |
| `@reboot`  | once at daemon start  | `@reboot ~/bin/startup.sh`     |
| `@hourly` `@daily` `@weekly` `@monthly` `@yearly` | shortcuts | `@daily` = `0 0 * * *` |

> [!warning] Day-of-month OR day-of-week
> If **both** fields are restricted, the job runs when **either** matches.
> `0 0 13 * 5` runs on every 13th **and** every Friday, not only on Friday the 13th.

### Examples
```
# m  h  dom mon dow  command
30   3  *   *   *    /home/alice/bin/backup.sh >> /home/alice/backup.log 2>&1
*/5  *  *   *   *    /usr/local/bin/check-disk.sh
0    9  *   *   1    echo "Monday!" | mail -s hi me@example.com
@reboot              sleep 60 && /home/alice/bin/after-boot.sh
```

# Gotchas

| Problem                        | Why                                                         | Fix |
| ------------------------------ | ----------------------------------------------------------- | --- |
| "works in terminal, not in cron" | cron's environment is minimal: `PATH=/usr/bin:/bin`, no `.bashrc`, `SHELL=/bin/sh` | use full paths, or set `PATH=` at the top of the crontab |
| `%` breaks the command         | `%` means newline in crontab                                | escape: `date +\%F` |
| no output, no error            | output is **mailed** to the user; without a mail system it's lost | redirect: `>> file.log 2>&1` |
| missed job                     | machine was off at that time; cron doesn't catch up         | anacron, or systemd timer with `Persistent=true` |
| last line ignored              | crontab must end with a newline                             | `crontab -e` handles it |

Variables allowed at the top of a crontab:
```
SHELL=/bin/bash
PATH=/home/alice/.local/bin:/usr/local/bin:/usr/bin:/bin
MAILTO=""                # don't send mail
```

# Commands
```bash
crontab -l               # list your jobs
crontab -e               # edit (opens $EDITOR, validates on save)
crontab -r               # remove ALL your jobs — no confirmation!
crontab -l > backup.cron; crontab backup.cron     # backup / restore
sudo crontab -u alice -l # another user's crontab
```

> [!danger] `crontab -r` vs `crontab -e`
> `r` and `e` are neighbours on the keyboard. `-r` deletes your whole crontab instantly. Keep a backup: `crontab -l > ~/crontab.bak`.

# cron vs systemd timers

|                          | cron                          | systemd timer                       |
| ------------------------ | ----------------------------- | ----------------------------------- |
| Config                   | one line                      | two files (`.timer` + `.service`)   |
| Logs                     | mail or your own redirect     | automatic in the journal (`journalctl -u name`) |
| Missed runs              | lost (unless anacron)         | `Persistent=true` catches up        |
| Random delay / spread    | no                            | `RandomizedDelaySec=`               |
| Resource limits, sandbox | no                            | everything systemd units can do     |
| Dependencies             | no                            | `After=network-online.target` etc.  |
| Portability              | every Unix                    | systemd systems only                |

Syntax mapping: cron `30 3 * * *` = timer `OnCalendar=*-*-* 03:30:00`. Check any expression:
```bash
systemd-analyze calendar '*-*-* 03:30:00'
#   Normalized form: *-*-* 03:30:00
#       Next elapse: Mon 2026-10-05 03:30:00 CEST
```
Full timer guide: [[timers]].

# Scripting recipes

### Add a job from a script without opening an editor
```bash
( crontab -l 2>/dev/null; echo '0 * * * * /usr/local/bin/hourly.sh' ) | crontab -
```

### Add only if not already present
```bash
job='0 * * * * /usr/local/bin/hourly.sh'
crontab -l 2>/dev/null | grep -qxF "$job" || ( crontab -l 2>/dev/null; echo "$job" ) | crontab -
```

### Make a script cron-safe
```bash
#!/usr/bin/env bash
set -euo pipefail
export PATH=/usr/local/bin:/usr/bin:/bin
cd "$(dirname "$0")"
exec 9> /tmp/myjob.lock; flock -n 9 || exit 0     # skip if previous run still going
# ... work ...
```

### Simulate cron's environment
```bash
env -i SHELL=/bin/sh PATH=/usr/bin:/bin HOME="$HOME" /bin/sh -c '/path/to/script.sh'
```

# See also
- [[timers]] — the systemd replacement
- [[systemd]] — overview
- [[journal]] — logs of timer jobs
- [[var]] — `/var/spool/cron`
- [[etc]] — `/etc/crontab`, `/etc/cron.d`
- [[files]] — all file notes
- `man 5 crontab`, `man 8 cron`
