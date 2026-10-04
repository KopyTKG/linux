---
tags:
  - linux
  - processes
  - limits
  - scripting
aliases:
  - ulimit
  - prlimit
  - OOM killer
  - oom_score_adj
  - rlimit
---
The kernel limits what a single process may use: open files, stack size, number of processes, CPU time, ... These are **rlimits** (`ulimit`). For groups of processes (a whole service, a user) there are **cgroups**. When RAM runs out anyway, the **OOM killer** picks a victim.

# Soft vs hard limits

| Limit    | Who can change it                                                  |
| -------- | ------------------------------------------------------------------ |
| **Soft** | The process itself, anywhere up to the hard limit. This is the one enforced. |
| **Hard** | Ceiling for soft. Normal users can only **lower** it; raising needs root (`CAP_SYS_RESOURCE`). |

Limits are **inherited** by children, so setting them in a shell affects everything started from it.

# `ulimit` (shell builtin)
```bash
ulimit -a          # all soft limits
ulimit -Ha         # all hard limits
ulimit -n          # open files (soft)
ulimit -Hn         # open files (hard)
ulimit -n 4096     # set soft (and hard, if not specified otherwise)
ulimit -Sn 4096    # set only soft
ulimit -c unlimited   # allow core dumps in this shell
```

| Flag | Resource              | Typical default        | Note                                  |
| ---- | --------------------- | ---------------------- | ------------------------------------- |
| `-n` | Open files (`NOFILE`) | 1024 soft / 524288 hard | "Too many open files" (`EMFILE`)     |
| `-u` | Processes (`NPROC`)   | ~ size of RAM based    | Per **user**, not per process — fork bomb protection |
| `-s` | Stack size, KiB       | 8192                   | Deep recursion → `SIGSEGV`            |
| `-c` | Core file size        | 0 or unlimited         | 0 = no core dumps                     |
| `-v` | Virtual memory, KiB   | unlimited              |                                       |
| `-t` | CPU time, seconds     | unlimited              | `SIGXCPU`, then `SIGKILL`             |
| `-f` | File size             | unlimited              | `SIGXFSZ` on exceed                   |
| `-l` | Locked memory, KiB    | 8192                   | `mlock()`                             |
| `-e` | Max nice priority     | 0                      | → [[priority-scheduling]]             |
| `-r` | Max real-time prio    | 0                      | → [[priority-scheduling]]             |

> [!tip] Limit a single command, not your whole shell
> ```bash
> ( ulimit -v 1000000; ./memory-hungry )    # subshell: limit applies only inside
> ```

# `/proc/<pid>/limits`
Shows the limits of **any** process → [[proc]].
```bash
cat /proc/$(pgrep -o nginx)/limits
```
```
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max stack size            8388608              unlimited            bytes
Max processes             127696               127696               processes
Max open files            1024                 524288               files
```

# `prlimit` — change a running process
```bash
prlimit --pid 4600                          # show all
prlimit --pid 4600 --nofile                 # one resource
prlimit --pid 4600 --nofile=4096:8192       # set soft:hard (own process, within hard)
prlimit --nofile=4096 --as=2000000000 ./app # start a command with limits
```
```
RESOURCE DESCRIPTION              SOFT HARD UNITS
NOFILE   max number of open files 4096 8192 files
```

# Where defaults come from

| Source                                    | Applies to                                 |
| ----------------------------------------- | ------------------------------------------ |
| Kernel defaults                           | PID 1, inherited by everything             |
| `/etc/security/limits.conf`, `limits.d/`  | Login sessions via PAM `pam_limits` → [[pam]] |
| `/etc/systemd/system.conf` `DefaultLimit*=` | All services                             |
| Unit file `LimitNOFILE=` etc.             | One service → [[services]]                 |
| `ulimit` in [[bashrc]] / profile          | Shells started afterwards                  |

`limits.conf` format:
```
#<domain>  <type>  <item>    <value>
alice      soft    nofile    8192
@audio     -       rtprio    95
*          hard    core      0
```

> [!warning] `limits.conf` doesn't affect services
> It is applied by PAM at **login**. systemd services don't log in, so set `LimitNOFILE=` in the unit instead.

# The OOM killer
When memory (RAM + swap) is exhausted, the kernel's **OOM (out of memory) killer** sends `SIGKILL` to one process to free memory. The victim is chosen by **`oom_score`** (higher = more likely), mostly based on memory use.

```bash
cat /proc/4600/oom_score        # 0 … 1000+, computed by kernel
cat /proc/4600/oom_score_adj    # -1000 … 1000, your bias (default 0)
```

| `oom_score_adj` | Effect                                        |
| --------------- | --------------------------------------------- |
| `-1000`         | Never kill this process                       |
| `< 0`           | Less likely (needs root / `CAP_SYS_RESOURCE`) |
| `0`             | Default                                       |
| `> 0`           | More likely — any user may raise it for own processes |
| `1000`          | Kill this first                               |

```bash
echo 500 > /proc/$$/oom_score_adj     # make my shell (and its children) a preferred victim
```

### Top OOM candidates
```bash
for d in /proc/[0-9]*; do
  printf '%s %s %s\n' "$(cat "$d/oom_score" 2>/dev/null)" "${d#/proc/}" "$(cat "$d/comm" 2>/dev/null)"
done | sort -rn | head -5
```

### Was something OOM-killed?
```bash
journalctl -k | grep -iE 'out of memory|oom-kill|killed process'      # → [[journal]]
```
```
kernel: Out of memory: Killed process 5120 (java) total-vm:9123456kB, anon-rss:7340032kB, ...
```
The killed process exits with code `137` (`128 + 9`) → [[process-lifecycle#Exit codes]].

> [!note] Userspace OOM daemons
> Many systems also run `systemd-oomd` or `earlyoom`, which act **before** the kernel by watching memory pressure (`/proc/pressure/memory` → [[proc]]) and killing whole cgroups.

# cgroups
rlimits are per process. **Control groups** (cgroup v2) limit and measure whole **groups** of processes — every systemd service, user session and container is one. Files live in `/sys/fs/cgroup/` → [[sys]].

```bash
cat /proc/$$/cgroup                       # which cgroup am I in
cg=/sys/fs/cgroup$(cut -d: -f3 /proc/$$/cgroup)
cat "$cg/memory.current" "$cg/pids.current"
systemd-cgtop                             # live per-cgroup usage
```

Set limits through systemd → [[units]], [[services]]:
```bash
systemd-run --user -p MemoryMax=500M -p CPUQuota=50% ./job.sh
systemctl set-property nginx.service MemoryMax=1G      # persistent (root)
```

| Property      | Effect                                         |
| ------------- | ---------------------------------------------- |
| `MemoryMax=`  | Hard RAM limit; above it the group is OOM-killed |
| `MemoryHigh=` | Soft limit; throttles and reclaims             |
| `CPUQuota=`   | Max CPU time (`200%` = two CPUs)               |
| `CPUWeight=`  | Relative share (default 100)                   |
| `TasksMax=`   | Max processes/threads                          |
| `IOWeight=`   | Relative disk share                            |

# Find stuff

| Question                              | Command                                          |
| ------------------------------------- | ------------------------------------------------ |
| My limits                             | `ulimit -a`, `ulimit -Ha`                        |
| Limits of another process             | `cat /proc/PID/limits`, `prlimit --pid PID`      |
| How many files does PID have open     | `ls /proc/PID/fd \| wc -l`                       |
| System-wide open files                | `cat /proc/sys/fs/file-nr` → [[proc]]            |
| Max PIDs                              | `cat /proc/sys/kernel/pid_max`                   |
| OOM kills                             | `journalctl -k \| grep -i oom`                   |
| Memory of a service                   | `systemctl status UNIT` (Memory: line)           |

# See also
- [[processes]] · [[priority-scheduling]] · [[signals]]
- [[proc]] — `/proc/<pid>/limits`, `oom_score`, `/proc/pressure`
- [[sys]] — `/sys/fs/cgroup`
- [[units]] · [[services]] — resource control in systemd
- `man 2 getrlimit`, `man 1 prlimit`, `man 5 limits.conf`, `man 7 cgroups`
