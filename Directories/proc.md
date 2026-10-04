---
tags:
  - linux
  - filesystem
  - proc
  - scripting
aliases:
  - procfs
  - /proc
---
Proc directory is where OS stores process info. Part of that info is OS level info such as `cpuinfo` or `meminfo` and much more.

> [!info] What `/proc` really is
> `/proc` is **not on disk**. It is a virtual filesystem (`procfs`) the kernel generates on the fly every time you read a file. That is why:
> - `ls -l` shows almost every file with size `0`, but `cat` still prints content.
> - Values change between two reads. Reading a file = asking the kernel a question *right now*.
> - Most files are read-only. Writable ones (mostly under `/proc/sys`) change kernel behaviour **immediately**.
>
> Check it yourself: `mount | grep proc` → `proc on /proc type proc`.

> [!tip] Golden rules for scripting with `/proc`
> 1. Read a file **once** into a variable, then parse the variable. The file can change between two reads.
> 2. Processes can die while you read them. Always handle "No such file or directory" (`2>/dev/null`, `|| continue`).
> 3. Units differ per file: **pages**, **clock ticks**, **kB**, **bytes**. Check the unit before doing math.
>    - page size: `getconf PAGESIZE` (usually `4096`)
>    - clock ticks per second: `getconf CLK_TCK` (usually `100`)
> 4. Many values are **counters since boot**. To get a *rate* (CPU %, network speed) read twice, sleep, subtract.
> 5. Some files are `\0`-separated, not newline-separated (`cmdline`, `environ`). Use `tr '\0' '\n'`.
> 6. Other users' processes are partly hidden unless you are root (`environ`, `fd`, `io`, ...).

# `/proc/<pid>`
This directory stores not only OS level info but it goes as far as to listing all processes as directories and their info.

Every running process has a directory named after its PID. Threads live in `/proc/<pid>/task/<tid>/` with the same layout.

| Shortcut              | Meaning                                                                  |
| --------------------- | ------------------------------------------------------------------------ |
| `/proc/self`          | Symlink to the process that is reading it (e.g. `cat /proc/self/status` shows `cat` itself) |
| `/proc/$$`            | Your current shell (`$$` = shell PID in bash/zsh, `$fish_pid` in fish)   |
| `/proc/[0-9]*`        | Glob that matches all processes                                          |
| `/proc/thread-self`   | Like `self`, but for the current thread                                  |

### Files overview

| File            | What it is                                              | Format                    |
| --------------- | ------------------------------------------------------- | ------------------------- |
| `stat`          | Key stats, one line, space separated                    | numbers (see below)       |
| `statm`         | Memory usage                                            | pages                     |
| `status`        | Human readable mix of `stat` + `statm` + more           | `Key:\tvalue`             |
| `comm`          | Process name (max 15 chars)                             | text                      |
| `cmdline`       | Full command with arguments                             | `\0` separated            |
| `environ`       | Environment variables at start                          | `\0` separated            |
| `exe`           | Symlink to the executable                               | symlink                   |
| `cwd`           | Symlink to current working directory                    | symlink                   |
| `root`          | Symlink to process root (differs in containers/chroot)  | symlink                   |
| `fd/`           | One symlink per open file descriptor                    | symlinks                  |
| `fdinfo/`       | Position + flags of each fd                             | `key: value`              |
| `maps`          | Memory mappings (libraries, heap, stack)                | table                     |
| `smaps_rollup`  | Summed memory (RSS, PSS, Swap)                          | `Key: value kB`           |
| `io`            | Bytes read/written                                      | `key: value`              |
| `limits`        | Resource limits (`ulimit`)                              | table                     |
| `cgroup`        | Which cgroup / systemd unit it belongs to               | `id::path`                |
| `ns/`           | Namespaces (containers)                                 | symlinks                  |
| `oom_score`     | How likely OOM killer kills it (higher = sooner)        | number                    |
| `oom_score_adj` | Writable bias for OOM killer (`-1000` … `1000`)         | number                    |
| `wchan`         | Kernel function the process is sleeping in              | text                      |
| `task/`         | Threads                                                 | directories               |

### `<pid>/stat`
`stat` holds **key** statistics of given process. It is one line with 52 space separated fields. Fields are numbered from **1**.

```
11882 (zsh) S 11469 11882 11882 0 -1 4194304 363 79 0 0 0 0 0 0 16 -4 1 0 233907 12349440 1097 ...
```

| #   | Field         | Example    | Meaning                                                            |
| --- | ------------- | ---------- | ------------------------------------------------------------------ |
| 1   | `pid`         | `11882`    | Process ID                                                         |
| 2   | `comm`        | `(zsh)`    | Name in parentheses                                                |
| 3   | `state`       | `S`        | State, see table below                                             |
| 4   | `ppid`        | `11469`    | Parent PID                                                         |
| 5   | `pgrp`        | `11882`    | Process group                                                      |
| 6   | `session`     | `11882`    | Session ID                                                         |
| 7   | `tty_nr`      | `0`        | Controlling terminal (`0` = none)                                  |
| 10  | `minflt`      | `363`      | Minor page faults                                                  |
| 12  | `majflt`      | `0`        | Major page faults (had to read from disk)                          |
| 14  | `utime`       | `0`        | Time in **user** mode, in clock ticks                              |
| 15  | `stime`       | `0`        | Time in **kernel** mode, in clock ticks                            |
| 18  | `priority`    | `16`       | Scheduling priority                                                |
| 19  | `nice`        | `-4`       | Nice value (`-20` … `19`)                                          |
| 20  | `num_threads` | `1`        | Number of threads                                                  |
| 22  | `starttime`   | `233907`   | Start time **after boot**, in clock ticks                          |
| 23  | `vsize`       | `12349440` | Virtual memory size, in **bytes**                                  |
| 24  | `rss`         | `1097`     | Resident set size (real RAM), in **pages**                         |
| 39  | `processor`   | `6`        | CPU it last ran on                                                 |

**Process states**

| State | Meaning                                                    |
| ----- | ---------------------------------------------------------- |
| `R`   | Running or runnable                                        |
| `S`   | Sleeping (waiting for event, interruptible)                |
| `D`   | Disk sleep (uninterruptible, usually I/O). Can't be killed |
| `Z`   | Zombie: exited, parent hasn't collected it yet             |
| `T`   | Stopped (`Ctrl+Z`, `SIGSTOP`)                              |
| `t`   | Stopped by debugger                                        |
| `I`   | Idle kernel thread                                         |
| `X`   | Dead (you should never see it)                             |

> [!warning] Don't parse `stat` with plain `awk '{print $3}'`
> `comm` can contain **spaces and parentheses**. Kernel thread `mt76-tx phy0` gives `(mt76-tx phy0)`, which shifts every field after it by one.
> Split on the **last** `)` instead:
> ```bash
> stat=$(< /proc/$pid/stat)
> name=${stat#*(}; name=${name%)*}   # text between first ( and last )
> rest=${stat##*) }                   # everything after last ")"
> read -r state ppid pgrp _ <<< "$rest"
> # in $rest field N of stat is word (N-2): state=$1, ppid=$2, utime=$12, starttime=$20
> ```

### `<pid>/statm`
`statm` is exactly what it tells it is. It's *status memory* with fields. All values are in **pages** (multiply by `getconf PAGESIZE` to get bytes).

| size  | resident | shared | text | lib | data | dt  |
| ----- | -------- | ------ | ---- | --- | ---- | --- |
| 3015  | 1234     | 1074   | 240  | 0   | 218  | 0   |

| Field      | Meaning                                         |
| ---------- | ----------------------------------------------- |
| `size`     | Total virtual memory (same as `VmSize`)         |
| `resident` | Pages in RAM (same as `VmRSS`)                  |
| `shared`   | Resident pages backed by a file (libraries ...) |
| `text`     | Program code                                    |
| `lib`      | Unused, always `0`                              |
| `data`     | Data + stack                                    |
| `dt`       | Unused, always `0`                              |

`1234 pages × 4096 = 5 054 464 B ≈ 4.8 MiB` in RAM.

### `<pid>/status`
Same info as `stat` + `statm` but readable, `Key:<TAB>value`. **Best file for scripts** when you need one or two values, because you grep by name and not by position.

```
Name:	zsh
State:	S (sleeping)
Pid:	11882
PPid:	11469
Uid:	1000	1000	1000	1000
VmPeak:	   12060 kB
VmSize:	   12060 kB
VmRSS:	    4936 kB
VmSwap:	       0 kB
Threads:	1
voluntary_ctxt_switches:	8
nonvoluntary_ctxt_switches:	3
```

| Key                | Meaning                                                       |
| ------------------ | ------------------------------------------------------------- |
| `Name`             | Same as `comm`                                                |
| `State`            | Letter + word                                                 |
| `PPid`             | Parent PID                                                    |
| `Uid` / `Gid`      | Real, effective, saved, filesystem IDs                        |
| `VmPeak`           | Max virtual memory ever used                                  |
| `VmSize`           | Current virtual memory                                        |
| `VmHWM`            | Max RAM ever used ("high water mark")                         |
| `VmRSS`            | Current RAM used (`RssAnon` + `RssFile` + `RssShmem`)         |
| `VmSwap`           | Swapped out memory                                            |
| `Threads`          | Thread count                                                  |
| `Cap*`             | Capabilities bitmask (decode with `capsh --decode=<hex>`)     |
| `Cpus_allowed_list`| CPUs it may run on                                            |

```bash
awk '/^VmRSS:/ {print $2}' /proc/$pid/status    # RAM in kB
```

### `<pid>/cmdline` & `<pid>/environ`
Arguments and environment are separated by `\0` (null byte), so plain `cat` glues them together.

```bash
tr '\0' ' '  < /proc/$pid/cmdline; echo   # command on one line
tr '\0' '\n' < /proc/$pid/environ          # one variable per line
```

> [!note]
> - `cmdline` is **empty** for kernel threads (names like `kworker/...`). Use `comm` for them.
> - `environ` shows the environment **at process start**. Later `export` inside the process is not visible.

### `<pid>/exe`, `cwd`, `root`
Symlinks. Use `readlink`.

```bash
readlink /proc/$pid/exe   # /usr/bin/zsh
readlink /proc/$pid/cwd   # /home/alice
```

> [!tip] Recover a deleted binary
> If the file was deleted but the process still runs, `exe` shows `/usr/bin/foo (deleted)` and `cp /proc/$pid/exe ./foo` still copies it.

### `<pid>/fd/`
Every open file descriptor is a symlink. `0` = stdin, `1` = stdout, `2` = stderr.

```
0 -> socket:[160115]
1 -> /tmp/output.log
2 -> /tmp/output.log
10 -> /dev/null
```

Targets can be files, `socket:[inode]`, `pipe:[inode]`, `anon_inode:[eventfd]`, ...
`socket:[inode]` matches the `inode` column in `/proc/net/tcp`, that is how `ss -p` / `lsof` find which process owns a port.

```bash
ls /proc/$pid/fd | wc -l   # number of open files
```

### `<pid>/maps`
Memory map: what is loaded where.

```
55c536077000-55c536087000 r--p 00000000 00:21 1255071   /usr/bin/zsh
```

| Column      | Meaning                                                         |
| ----------- | --------------------------------------------------------------- |
| address     | Start–end virtual address                                       |
| perms       | `r` read, `w` write, `x` execute, `p` private / `s` shared      |
| offset      | Offset in the file                                              |
| dev         | Device `major:minor`                                            |
| inode       | File inode (`0` = anonymous)                                    |
| path        | File, or `[heap]`, `[stack]`, `[vdso]`, empty = anonymous       |

```bash
awk '$6 ~ /\.so/ {print $6}' /proc/$pid/maps | sort -u   # loaded shared libraries
```

For real memory accounting use `smaps_rollup` — it has `Pss` (proportional set size: shared pages divided between processes), which is more honest than RSS.

### `<pid>/io`
```
rchar: 226045        # bytes read via read() syscalls (includes cache)
wchar: 11313         # bytes written via write()
syscr: 130           # number of read syscalls
syscw: 36            # number of write syscalls
read_bytes: 176128   # bytes actually fetched from disk
write_bytes: 0       # bytes actually sent to disk
cancelled_write_bytes: 0
```

### `<pid>/limits`
Same as `ulimit -a`, but for **any** process.

```
Limit                     Soft Limit           Hard Limit           Units
Max open files            1024                 524288               files
Max stack size            8388608              unlimited            bytes
```

### `<pid>/cgroup`
```
0::/user.slice/user-1000.slice/user@1000.service/app.slice/app-Alacritty@....service
```
Tells you which **systemd unit** / container started the process. Detail in `/sys/fs/cgroup` → see [[sys]].

# System wide files

### `/proc/cpuinfo`
One block per **logical** CPU (thread), separated by an empty line.

| Key          | Example                               | Meaning                       |
| ------------ | ------------------------------------- | ----------------------------- |
| `processor`  | `0`                                   | Logical CPU number            |
| `model name` | `AMD Ryzen 7 5800X 8-Core Processor`  | CPU model                     |
| `cpu MHz`    | `4754.899`                            | Current frequency             |
| `cache size` | `512 KB`                              | L2 cache per core             |
| `siblings`   | `16`                                  | Threads per socket            |
| `cpu cores`  | `8`                                   | Physical cores per socket     |
| `flags`      | `... avx2 ...`                        | CPU features                  |

```bash
grep -c ^processor /proc/cpuinfo                       # thread count (or: nproc)
grep -m1 'model name' /proc/cpuinfo | cut -d: -f2-     # model
grep -qw avx2 /proc/cpuinfo && echo "AVX2 supported"   # feature check
```

### `/proc/meminfo`
`Key: value kB`.

```
MemTotal:       32754148 kB
MemFree:         7917564 kB
MemAvailable:   25365616 kB
Buffers:            6200 kB
Cached:         17425360 kB
SwapTotal:      ...
SwapFree:       ...
```

> [!important] `MemFree` vs `MemAvailable`
> `MemFree` = RAM that is completely unused. Linux uses "free" RAM as disk cache (`Cached`), so `MemFree` looks scary low.
> `MemAvailable` = how much can be given to new programs (free + reclaimable cache). **Use `MemAvailable`** to compute "used memory":
> `used = MemTotal - MemAvailable`

### `/proc/stat`
CPU time counters since boot, in clock ticks (`getconf CLK_TCK`, usually `100` = 1/100 s).

```
cpu  208200 6492 54114 5363967 13073 8318 6071 0 0 0
cpu0 33195 799 7453 193431 802 631 961 0 0 0
ctxt 86655661
btime 1791125839
processes 11923
procs_running 2
procs_blocked 0
```

`cpu` line columns:

| user   | nice | system | idle    | iowait | irq  | softirq | steal | guest | guest_nice |
| ------ | ---- | ------ | ------- | ------ | ---- | ------- | ----- | ----- | ---------- |
| 208200 | 6492 | 54114  | 5363967 | 13073  | 8318 | 6071    | 0     | 0     | 0          |

| Line            | Meaning                                         |
| --------------- | ----------------------------------------------- |
| `cpu`           | Sum of all CPUs                                 |
| `cpuN`          | Single logical CPU                              |
| `ctxt`          | Context switches since boot                     |
| `btime`         | Boot time as Unix timestamp (`date -d @1791125839`) |
| `processes`     | Forks since boot                                |
| `procs_running` | Runnable right now                              |
| `procs_blocked` | Waiting for I/O right now                       |

CPU % = `(Δtotal − Δidle) / Δtotal` where `idle = idle + iowait` → recipe below.

### `/proc/loadavg`
```
0.77 0.72 0.78 3/1958 11896
```
| 1 min | 5 min | 15 min | running/total threads | last PID |
| ----- | ----- | ------ | --------------------- | -------- |
| 0.77  | 0.72  | 0.78   | 3/1958                | 11896    |

Load ≈ how many threads want CPU (or wait on disk). Compare it with CPU count: load `8` on 16 threads is fine, on 4 threads it is overload.

### `/proc/uptime`
```
2339.09 53639.65
```
1. Seconds since boot.
2. Sum of idle time of **all** CPUs (can be bigger than uptime).

### `/proc/pressure/` (PSI)
Better than load average: "% of time tasks were **stalled** waiting for resource".
Files: `cpu`, `memory`, `io`.

```
some avg10=0.00 avg60=0.00 avg300=0.00 total=6535664
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```
- `some` = at least one task was waiting.
- `full` = all tasks were waiting (nothing useful happened).
- `avg10/60/300` = % over last 10 s / 60 s / 300 s.

### `/proc/version` & `/proc/cmdline`
```bash
cat /proc/version   # kernel version + compiler (same as uname -a, roughly)
cat /proc/cmdline   # parameters the bootloader gave the kernel
```
```
quiet nowatchdog splash rw rootflags=subvol=/@ root=UUID=3f6c... resume=UUID=7d2e...
```
Bootloader config lives in [[boot]].

### `/proc/mounts`
Symlink to `/proc/self/mounts`. Same format as `/etc/fstab`:

| device           | mountpoint | type    | options                     | dump | pass |
| ---------------- | ---------- | ------- | --------------------------- | ---- | ---- |
| `/dev/nvme0n1p3` | `/`        | `btrfs` | `rw,noatime,compress=zstd:1` | 0    | 0    |

```bash
awk '$2=="/" {print $3}' /proc/mounts    # filesystem type of /
```

### Other useful ones

| File                 | What                                                        |
| -------------------- | ----------------------------------------------------------- |
| `/proc/filesystems`  | Filesystems the kernel supports (`nodev` = virtual)         |
| `/proc/partitions`   | Block devices + sizes in 1 KiB blocks                       |
| `/proc/diskstats`    | I/O counters per disk (what `iostat` reads)                 |
| `/proc/swaps`        | Active swap devices                                         |
| `/proc/modules`      | Loaded kernel modules (`lsmod` reads this)                  |
| `/proc/interrupts`   | IRQ count per CPU per device                                |
| `/proc/devices`      | Major numbers of char/block drivers → see [[dev]]           |
| `/proc/kallsyms`     | Kernel symbols (addresses hidden for non-root)              |
| `/proc/config.gz`    | Kernel build config (if enabled): `zcat /proc/config.gz`    |

# `/proc/net`
Network state of **your network namespace** (symlink to `/proc/self/net`).

### `/proc/net/dev`
Counters per interface since boot.

```
 face |bytes    packets errs drop ...|bytes    packets errs drop ...
    lo: 497356420 6399403    0    0 ... 497356420 6399403 ...
enp3s0: 3135155058 2174149   0  346 ... 69555257  849354 ...
```
After `iface:` → fields 1–8 = **receive**, 9–16 = **transmit**. Column 1 = RX bytes, column 9 = TX bytes.

### `/proc/net/tcp` & `tcp6`
Every TCP socket. Addresses are **hex**, IPv4 in **little-endian**.

```
  sl  local_address rem_address   st ... uid  timeout inode
   0: 00000000:699C 00000000:0000 0A ... 1000        0 47846
   1: 0100007F:2ED3 00000000:0000 0A ...    0        0 29230
```

| Column          | Example         | Decoded                          |
| --------------- | --------------- | -------------------------------- |
| `local_address` | `0100007F:2ED3` | `127.0.0.1:11987`                |
| `st`            | `0A`            | state (`0A` = LISTEN)            |
| `inode`         | `29230`         | matches `socket:[29230]` in `fd/` |

States: `01` ESTABLISHED, `02` SYN_SENT, `06` TIME_WAIT, `08` CLOSE_WAIT, `0A` LISTEN.

Hex → decimal: `echo $((16#2ED3))` → `11987`.
IP `0100007F` read **bytes backwards**: `7F 00 00 01` → `127.0.0.1`.

Also: `udp`, `udp6`, `unix`, `route`, `arp`.

# `/proc/sys` — kernel settings
The only big writable part. Path = setting name with `/` instead of `.`:
`/proc/sys/vm/swappiness` ⇔ `sysctl vm.swappiness`.

```bash
cat /proc/sys/vm/swappiness                 # read (kernel default 60)
cat /proc/sys/kernel/hostname               # hostname
cat /proc/sys/kernel/pid_max                # max PID (4194304)
cat /proc/sys/fs/file-nr                    # open files: used, 0, max
echo 10 | sudo tee /proc/sys/vm/swappiness  # write (resets on reboot!)
```

> [!warning] Writes are live and temporary
> Writing to `/proc/sys` changes the running kernel **instantly**, and the change is **lost on reboot**.
> To make it permanent put `vm.swappiness = 10` into `/etc/sysctl.d/99-custom.conf` → see [[etc]].
> `sudo echo 10 > /proc/sys/...` **doesn't work** — redirect runs as your user. Use `| sudo tee`.

| Folder              | What's inside                                  |
| ------------------- | ---------------------------------------------- |
| `/proc/sys/kernel/` | hostname, pid_max, panic, sysrq, ...           |
| `/proc/sys/vm/`     | memory: swappiness, dirty_ratio, overcommit    |
| `/proc/sys/net/`    | networking: `ipv4/ip_forward`, TCP tuning      |
| `/proc/sys/fs/`     | file limits: `file-max`, `inotify/*`           |

# Scripting recipes
All recipes tested in `bash`. Run them with `bash script.sh` (fish has different syntax, see the end).

### CPU usage %
```bash
#!/usr/bin/env bash
read -r _ u n s i w q sq st _ < /proc/stat
t1=$((u+n+s+i+w+q+sq+st)); i1=$((i+w))
sleep 1
read -r _ u n s i w q sq st _ < /proc/stat
t2=$((u+n+s+i+w+q+sq+st)); i2=$((i+w))
echo "CPU: $(( 100 * ((t2-t1) - (i2-i1)) / (t2-t1) ))%"
```

### Memory usage
```bash
awk '/^MemTotal:/{t=$2} /^MemAvailable:/{a=$2}
     END{printf "Used: %.1f%% (%d MiB of %d MiB)\n", (t-a)*100/t, (t-a)/1024, t/1024}' /proc/meminfo
```

### Mini `ps`
```bash
#!/usr/bin/env bash
printf "%-7s %-7s %-5s %8s  %s\n" PID PPID STATE RSS_KiB NAME
for d in /proc/[0-9]*; do
  stat=$(< "$d/stat") 2>/dev/null || continue     # process may be gone
  name=${stat#*(}; name=${name%)*}
  read -r state ppid _ <<< "${stat##*) }"
  rss=$(awk '/^VmRSS:/{print $2}' "$d/status" 2>/dev/null)
  printf "%-7s %-7s %-5s %8s  %s\n" "${d#/proc/}" "$ppid" "$state" "${rss:-0}" "$name"
done
```

### Top 5 processes by RAM
```bash
for d in /proc/[0-9]*; do
  awk -v pid="${d#/proc/}" '/^Name:/{n=$2} /^VmRSS:/{r=$2} END{if(r) print r, pid, n}' "$d/status" 2>/dev/null
done | sort -rn | head -5 | awk '{printf "%8.1f MiB  %-7s %s\n", $1/1024, $2, $3}'
```

### Processes with most open files
```bash
for d in /proc/[0-9]*; do
  n=$(ls "$d/fd" 2>/dev/null | wc -l)
  (( n > 0 )) && echo "$n ${d#/proc/} $(< "$d/comm")"
done 2>/dev/null | sort -rn | head
```
Run with `sudo` to see other users' processes.

### Who has this file open? (poor man's `lsof`)
```bash
target=$(realpath "$1")
for fd in /proc/[0-9]*/fd/*; do
  if [[ $(readlink "$fd" 2>/dev/null) == "$target" ]]; then
    p=${fd#/proc/}; pid=${p%%/*}
    echo "$pid $(< /proc/$pid/comm)"
  fi
done 2>/dev/null | sort -u
```

### How long is a process running?
```bash
pid=$1
hz=$(getconf CLK_TCK)
stat=$(< /proc/$pid/stat)
start=$(awk '{print $20}' <<< "${stat##*) }")   # field 22 = starttime
up=$(cut -d. -f1 /proc/uptime)
echo "PID $pid running for $(( up - start / hz ))s"
```

### Listening TCP ports
```bash
for f in /proc/net/tcp /proc/net/tcp6; do
  while read -r _ local _ st _; do
    [[ $st == 0A ]] && echo $((16#${local##*:}))
  done < <(tail -n +2 "$f")
done | sort -nu
```

### Network speed
```bash
iface=${1:-enp3s0}
rx() { awk -v i="$iface:" '$1==i {print $2}' /proc/net/dev; }
tx() { awk -v i="$iface:" '$1==i {print $10}' /proc/net/dev; }
r1=$(rx); t1=$(tx); sleep 1; r2=$(rx); t2=$(tx)
echo "RX $(( (r2-r1)/1024 )) KiB/s   TX $(( (t2-t1)/1024 )) KiB/s"
```

### Wait until a process exits
```bash
while [[ -d /proc/$pid ]]; do sleep 1; done
echo "PID $pid finished"
```

### Find processes by name (poor man's `pgrep`)
```bash
for d in /proc/[0-9]*; do
  [[ $(< "$d/comm") == "$1" ]] 2>/dev/null && echo "${d#/proc/}"
done
```

### Fish versions
Fish has no `$((...))`, `<<<` or `${var#...}`. Use `math`, `string` and `read`.

```fish
# memory used
awk '/^MemTotal:/{t=$2} /^MemAvailable:/{a=$2} END{print (t-a)/1024 " MiB"}' /proc/meminfo

# process name from stat (safe for spaces)
set stat (cat /proc/$fish_pid/stat)
echo (string match -r '\((.*)\)' $stat)[2]

# hex port to decimal
math 0x2ED3
```

# Commands that read `/proc` for you
Good to know which tool reads which file — when tool is missing (containers, minimal systems) you read the file yourself.

| Command      | Reads                                      |
| ------------ | ------------------------------------------ |
| `ps`, `top`, `htop` | `/proc/<pid>/stat`, `status`, `cmdline` |
| `free`       | `/proc/meminfo`                            |
| `uptime`     | `/proc/uptime`, `/proc/loadavg`            |
| `lsof`       | `/proc/<pid>/fd`, `/proc/net/*`            |
| `ss`, `netstat` | `/proc/net/tcp`, `udp`, `unix` (ss uses netlink first) |
| `lsmod`      | `/proc/modules`                            |
| `mount`      | `/proc/self/mountinfo`                     |
| `sysctl`     | `/proc/sys/*`                              |
| `nproc`, `lscpu` | `/proc/cpuinfo`, `/sys/devices/system/cpu` |
| `pmap`       | `/proc/<pid>/maps`                         |

# See also
- [[sys]] — `/sys`, the newer virtual FS for devices and drivers (`/proc` = processes, `/sys` = hardware)
- [[dev]] — device files
- `man 5 proc` — full reference (also `man 5 proc_pid_stat`, `man 5 proc_meminfo`)
- Kernel docs: <https://docs.kernel.org/filesystems/proc.html>
