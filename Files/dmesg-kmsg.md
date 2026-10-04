---
tags:
  - linux
  - files
  - kernel
  - logs
aliases:
  - dmesg
  - /dev/kmsg
  - kernel ring buffer
  - printk
  - /proc/sys/kernel/printk
  - kernel log
---
The kernel can't write to `/var/log` — at boot there may not even be a filesystem yet. Instead it writes messages into an in-memory **ring buffer**: a fixed-size circular buffer where the oldest messages are overwritten when it's full. Everything you read with `dmesg` comes from there: hardware detection, driver errors, USB plug events, OOM kills, segfaults, filesystem problems.

| Interface            | What it is                                                     |
| -------------------- | -------------------------------------------------------------- |
| `dmesg`              | command that prints the ring buffer                            |
| `/dev/kmsg`          | character device: read = one record per line, write = inject a message (root) |
| `/proc/kmsg`         | older interface, consumes messages (only one reader, used by old syslog daemons) |
| `journalctl -k`      | kernel messages as stored by systemd-journald (persistent, survives reboot if the journal is on disk) |
| `/proc/sys/kernel/printk` | which levels get printed to the **console**               |
| `/proc/sys/kernel/dmesg_restrict` | `1` = only root (CAP_SYSLOG) can read `dmesg`     |

```
crw-r--r-- 1 root root 1, 11 /dev/kmsg
```

> [!warning] `dmesg: read kernel buffer failed: Operation not permitted`
> Many distributions set `kernel.dmesg_restrict = 1` because kernel logs can leak addresses useful to attackers. Then:
> - `sudo dmesg` works, or
> - `journalctl -k` works for users in the `wheel`/`adm`/`systemd-journal` group (depending on distro).

> [!info] Ring buffer is lost on reboot
> The buffer lives in RAM. To read messages from **before** a crash or reboot, use the persistent journal: `journalctl -k -b -1` (previous boot). → [[journal]]

# Reading it

### `dmesg`
```bash
sudo dmesg                   # all, timestamps in seconds since boot
sudo dmesg -T                # human-readable wall clock times (can drift after suspend)
sudo dmesg -H                # pager + colours + relative times
sudo dmesg -w                # follow new messages (like tail -f)
sudo dmesg -l err,warn       # only errors and warnings
sudo dmesg -f kern           # only facility "kern"
sudo dmesg -x                # show facility and level for each line
sudo dmesg -c                # print AND clear the buffer (root)
```

Output looks like:
```
[    0.000000] Linux version 6.x.y ...
[    1.234567] usb 1-2: new high-speed USB device number 3 using xhci_hcd
[   45.678901] EXT4-fs (sda1): mounted filesystem with ordered data mode
```
The number in brackets = seconds since boot (monotonic clock).

### `journalctl -k` (often no root needed)
```bash
journalctl -k                    # kernel messages, this boot
journalctl -k -b -1              # previous boot
journalctl -k -p err             # priority err and worse
journalctl -k -f                 # follow
journalctl -k -o short-monotonic # same [seconds] stamp as dmesg
journalctl -k --since "10 min ago"
```

# `/dev/kmsg` record format
Each `read()` returns one record:

```
priority,sequence,timestamp_us,flags;message text
 SUBSYSTEM=usb                    ← optional continuation lines (start with a space)
 DEVICE=c189:2
```
Example:
```
6,339,5140900,-;NET: Registered PF_INET6 protocol family
```

| Field          | Example   | Meaning                                         |
| -------------- | --------- | ----------------------------------------------- |
| priority       | `6`       | `facility * 8 + level`; kernel facility is 0, so this is just the level |
| sequence       | `339`     | increasing record number (gaps = messages were overwritten) |
| timestamp_us   | `5140900` | microseconds since boot                         |
| flags          | `-`       | `-` normal, `c` continuation of previous record |
| message        | text      | after the `;`                                   |

Writing to `/dev/kmsg` (root) injects a line into the kernel log — handy to mark a moment in the log before reproducing a bug:
```bash
echo "=== starting test 3 ===" | sudo tee /dev/kmsg
```

# Log levels

| Level | Name      | `dmesg -l` | Example                               |
| ----- | --------- | ---------- | ------------------------------------- |
| 0     | EMERG     | `emerg`    | system is unusable                    |
| 1     | ALERT     | `alert`    | action needed immediately             |
| 2     | CRIT      | `crit`     | critical hardware/software failure    |
| 3     | ERR       | `err`      | driver errors                         |
| 4     | WARNING   | `warn`     | something odd but working             |
| 5     | NOTICE    | `notice`   | normal but significant                |
| 6     | INFO      | `info`     | informational (most boot messages)    |
| 7     | DEBUG     | `debug`    | debugging (usually hidden)            |

**Lower number = more severe.** Same scale as syslog and `journalctl -p`.

# `/proc/sys/kernel/printk`
Controls which messages are also printed **to the console** (the text screen). It does **not** change what's stored in the buffer.

```
$ cat /proc/sys/kernel/printk
3	3	3	3
```

| # | Name                     | Meaning                                                   |
| - | ------------------------ | --------------------------------------------------------- |
| 1 | console_loglevel         | messages with level **lower** than this go to the console |
| 2 | default_message_loglevel | level used when a message doesn't specify one             |
| 3 | minimum_console_loglevel | lowest value console_loglevel may be set to               |
| 4 | default_console_loglevel | console_loglevel at boot                                  |

`3 3 3 3` means only levels 0–2 (emerg/alert/crit) appear on screen — typical with the `quiet` boot parameter. A value like `7 4 1 7` (common without `quiet`) shows everything except debug.

```bash
echo 8 | sudo tee /proc/sys/kernel/printk     # show everything on console (until reboot)
sudo dmesg -n 1                               # only panic messages on console
```
Boot parameters `quiet` (→ level 4) and `loglevel=N` set it at boot → [[boot]].

Other related knobs in `/proc/sys/kernel/`:

| File                 | Meaning                                                       |
| -------------------- | ------------------------------------------------------------- |
| `dmesg_restrict`     | `1` = non-root can't read the buffer                          |
| `printk_devkmsg`     | `on` / `off` / `ratelimit` — may user space write to `/dev/kmsg` |
| `printk_ratelimit`   | seconds between bursts of rate-limited messages               |

Buffer size is set at build time or with the `log_buf_len=` boot parameter (e.g. `log_buf_len=4M`).

# What to look for

| Message contains                    | Means                                           |
| ----------------------------------- | ----------------------------------------------- |
| `Out of memory: Killed process`     | OOM killer ended a process → [[proc]] `oom_score` |
| `segfault at ... ip ... sp ...`     | a program crashed                               |
| `I/O error, dev sda, sector`        | disk problem — check SMART                      |
| `EXT4-fs error` / `BTRFS error`     | filesystem corruption                           |
| `usb 1-2: new ... USB device`       | something was plugged in                        |
| `link is not ready` / `Link is Up`  | network cable/Wi-Fi state → [[networking]]      |
| `Call Trace:` / `BUG:` / `Oops`     | kernel bug; kernel gets tainted → [[proc-self-highlights]] |
| `firmware: failed to load`          | missing firmware file                           |
| `audit:` / `apparmor=` / `avc:`     | security module denied something                |

# Scripting recipes

### Errors and warnings from this boot (no root)
```bash
journalctl -k -b -p warning --no-pager -q
```

### Count USB plug events this boot
```bash
journalctl -k -b --no-pager -q | grep -ciE 'usb .* new'
```

### Was anything OOM-killed?
```bash
journalctl -k -b --no-pager -q | grep -E 'Out of memory|oom-kill' || echo "no OOM kills"
```

### Watch kernel messages while plugging in a device
```bash
sudo dmesg -w          # or: journalctl -kf
# plug in USB stick → see "sd 2:0:0:0: [sdc] Attached SCSI removable disk"
```

### Parse `/dev/kmsg` style records (root)
```bash
sudo timeout 1 cat /dev/kmsg | awk -F';' '{split($1,h,","); printf "lvl=%s t=%.3fs %s\n", h[1]%8, h[3]/1e6, $2}' | tail
```

# See also
- [[journal]] — persistent logs, including kernel messages
- [[dev]] — `/dev/kmsg` is a character device
- [[dev-special-files]] — other special devices
- [[proc]] — `/proc/sys/kernel`
- [[proc-self-highlights]] — `tainted`, SysRq
- [[boot]] — `quiet`, `loglevel=` boot parameters
- [[files]] — all file notes
- `man dmesg`, <https://docs.kernel.org/core-api/printk-basics.html>, <https://www.kernel.org/doc/Documentation/ABI/testing/dev-kmsg>
