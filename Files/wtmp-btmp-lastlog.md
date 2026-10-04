---
tags:
  - linux
  - files
  - logs
  - security
aliases:
  - /var/log/wtmp
  - /var/log/btmp
  - /run/utmp
  - utmp
  - lastlog
  - last
---
Linux keeps **login records** in a few small **binary** files. You can't `cat` them — use `last`, `lastb`, `who` and `lastlog`/`lastlog2`, or `utmpdump` to turn them into text.

| File                 | What it records                               | Read with       | Survives reboot |
| -------------------- | --------------------------------------------- | --------------- | --------------- |
| `/run/utmp`          | who is logged in **right now**                | `who`, `w`, `users` | no ([[run]] is tmpfs) |
| `/var/log/wtmp`      | history of logins, logouts, reboots, shutdowns | `last`         | yes             |
| `/var/log/btmp`      | **failed** login attempts                     | `lastb` (root)  | yes             |
| `/var/log/lastlog`   | last login time per UID (sparse file)         | `lastlog`       | yes             |
| `/var/lib/lastlog/lastlog2.db` | newer SQLite replacement for `lastlog` | `lastlog2`    | yes             |

Which of `lastlog` / `lastlog2` exists depends on the distribution and its version. The old `lastlog` file is a sparse file indexed by UID, so with large UIDs `ls -l` can show a huge size that doesn't really use disk.

> [!info] Why binary?
> Each record is a fixed-size C `struct utmp` (384 bytes on x86-64): type, PID, terminal, user, host, timestamp. Fixed size lets programs seek and update records quickly. Count records with:
> ```bash
> echo $(( $(stat -c %s /var/log/wtmp) / 384 ))
> ```

> [!warning] 2038 problem and the move to `wtmpdb`
> `struct utmp` stores time as a 32-bit value, which overflows in January 2038. Some distributions are replacing `wtmp`/`lastlog` with SQLite databases (`wtmpdb`, `lastlog2`). The commands look the same; the files move to `/var/lib/`. If `/var/log/wtmp` is missing on your system, look for `/var/lib/wtmpdb/`.

# Record types
`utmpdump` shows the type number in the first column:

| Type | Name            | Meaning                         |
| ---- | --------------- | ------------------------------- |
| 1    | `RUN_LVL`       | runlevel change / shutdown      |
| 2    | `BOOT_TIME`     | system boot (`reboot` in `last`) |
| 5    | `INIT_PROCESS`  | process started by init         |
| 6    | `LOGIN_PROCESS` | getty waiting for login         |
| 7    | `USER_PROCESS`  | a user logged in                |
| 8    | `DEAD_PROCESS`  | a session ended                 |

```bash
utmpdump /run/utmp
```
```
[2] [00000] [~~  ] [reboot  ] [~           ] [6.x.y-generic       ] [0.0.0.0        ] [2026-10-04T14:57:27,206171+00:00]
[8] [07252] [ts/1] [        ] [pts/1       ] [                    ] [0.0.0.0        ] [2026-10-04T15:09:20,559694+00:00]
```
`[type] [pid] [id] [user] [line/tty] [host or kernel] [ip] [time]`

# `last` — history from wtmp
```bash
last                       # everything, newest first
last -n 10                 # last 10 entries
last alice                  # one user
last reboot                # boot history (pseudo-user "reboot")
last -x shutdown           # include shutdowns and runlevel changes
last -F                    # full date+time for login and logout
last -i                    # show IP instead of hostname
last -s yesterday -t today # time range
```

Example:
```
reboot   system boot  6.x.y-generic  Sun Oct  4 16:57   still running
reboot   system boot  6.x.y-generic  Sat Oct  3 13:40 - 22:25  (08:45)
shutdown system down  6.x.y-generic  Sat Oct  3 22:25 - 16:57  (18:31)

wtmp begins Sat Jun 20 17:38:00 2026
```

| Column          | Meaning                                          |
| --------------- | ------------------------------------------------ |
| user            | login name, or `reboot` / `shutdown`             |
| tty             | `tty2` (console), `pts/0` (terminal/SSH), `system boot` |
| host            | remote host for SSH, kernel version for reboots  |
| login – logout  | `still logged in`, `still running`, `crash`, `down` |
| (duration)      | `hh:mm` or `days+hh:mm`                          |

> [!note] Desktop sessions may be missing
> Graphical logins only appear in `wtmp`/`utmp` if the display manager writes them. Many modern desktops don't, so `who` can be empty even though you are logged in. `loginctl list-sessions` (systemd-logind) is more reliable → [[systemd]].

# `lastb` — failed logins
Needs root because `btmp` contains things people typed as usernames — sometimes passwords typed into the wrong field.

```bash
sudo lastb -n 20
sudo lastb | awk '{print $3}' | sort | uniq -c | sort -rn | head   # top source IPs
```
On a server with SSH open to the internet `btmp` grows fast; logrotate keeps it in check.

# `who`, `w`, `users` — right now (utmp)
```bash
who            # user, tty, login time, (remote host)
who -b         # last boot time
w              # who + what they are running + load average
users          # just names
```

# `lastlog` / `lastlog2`
```bash
lastlog2                 # or: lastlog
lastlog2 -u root
```
```
Username         Port     From          Latest
alice             tty2                   Sun Oct  4 16:57:37 +0200 2026
root             tty2                   Sun Jul 19 11:56:30 +0200 2026
```
Good for "has this account ever been used?".

# Scripting recipes

### Number of boots in wtmp
```bash
last reboot | grep -c '^reboot'
```

### Uptime of each previous boot
```bash
last -x reboot -F | awk '/^reboot/ && /\(/ {print $5, $6, $7, $8, $NF}'
```

### Was there an unclean shutdown?
```bash
last -x | grep -E 'crash' && echo "unclean shutdown found"
```
`crash` means a boot record was followed by another boot without a shutdown in between (power loss, hard reset, kernel panic). Look at the previous boot's log: `journalctl -b -1 -e` → [[journal]].

### Export wtmp to text, filter, convert back
```bash
utmpdump /var/log/wtmp > wtmp.txt         # text, editable
utmpdump -r < wtmp.txt > wtmp.new         # back to binary (root to install)
```

### Alert when someone logs in (script for a timer)
```bash
current=$(who | awk '{print $1}' | sort -u)
[[ -n $current ]] && echo "logged in: $current"
```

> [!tip] systemd journal has the same info
> ```bash
> journalctl _COMM=sshd | grep -E 'Accepted|Failed'
> journalctl -u systemd-logind | grep 'New session'
> ```
> → [[journal]]

# See also
- [[var]] — `/var/log`
- [[run]] — `/run/utmp`
- [[journal]] — text-based login events
- [[ssh-user]] — SSH keys and access
- [[passwd]], [[shadow]] — the accounts these records refer to
- [[files]] — all file notes
- `man 5 utmp`, `man last`, `man utmpdump`
