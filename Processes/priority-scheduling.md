---
tags:
  - linux
  - processes
  - scheduling
  - performance
aliases:
  - nice
  - renice
  - ionice
  - chrt
  - taskset
  - CPU affinity
---
The kernel **scheduler** decides which runnable thread gets a CPU and for how long. You can influence it per process: **nice** (CPU share), **ionice** (disk share), **real-time policies** (`chrt`) and **CPU affinity** (`taskset`).

# Scheduler basics
- Normal tasks use the fair scheduler: **CFS** (Completely Fair Scheduler), replaced by **EEVDF** in kernel 6.6+. Both share CPU time fairly, weighted by nice value.
- **Real-time** tasks (`SCHED_FIFO`, `SCHED_RR`) always run before normal ones.
- A process only competes when it is runnable (`R`). A sleeping process uses no CPU, so nice only matters under load.

| Policy           | Type       | Priority range | Use                                               |
| ---------------- | ---------- | -------------- | ------------------------------------------------- |
| `SCHED_OTHER`    | normal     | nice −20 … 19  | Everything by default                             |
| `SCHED_BATCH`    | normal     | nice −20 … 19  | CPU-heavy batch jobs, fewer wakeups               |
| `SCHED_IDLE`     | normal     | —              | Run only when the CPU is otherwise idle           |
| `SCHED_FIFO`     | real-time  | 1 … 99         | Audio, control loops. Runs until it yields        |
| `SCHED_RR`       | real-time  | 1 … 99         | Like FIFO with time slices                        |
| `SCHED_DEADLINE` | real-time  | runtime/period | Hard deadlines                                    |

# `nice` and `renice`
Nice value: **−20** (highest priority) to **19** (lowest). Default **0**. Each step ≈ 10% CPU weight difference.

```bash
nice -n 10 tar czf backup.tgz /home/alice     # start with nice 10
nice tar ...                                  # default +10
renice -n 15 -p 4600                          # change running process
renice -n 5 -u alice                          # all of alice's processes
renice -n 5 -g 4600                           # whole process group
ps -o pid,ni,pri,comm -p 4600                 # check
```
```
    PID  NI PRI COMMAND
   4600  10   9 sleep
```

> [!note] Who may do what
> - Normal users can only **raise** nice (lower priority), never go below their current value.
> - Lowering nice (negative values) needs root or `CAP_SYS_NICE` → [[capabilities]], or a limit in `/etc/security/limits.conf` (`nice`) → [[resource-limits]].
> - `PRI` in `ps` is a derived number; different tools show different scales. Look at `NI`.

# `ionice` — disk priority
Works with I/O schedulers that support priorities (e.g. BFQ); others may ignore it.

| Class | Name        | Meaning                                         |
| ----- | ----------- | ----------------------------------------------- |
| 1     | realtime    | Always first (root only)                        |
| 2     | best-effort | Default, levels 0 (high) … 7 (low)              |
| 3     | idle        | Only when nobody else uses the disk             |

```bash
ionice -c 3 rsync -a /data /backup        # idle class
ionice -c 2 -n 7 -p 4600                  # lowest best-effort for running PID
ionice -p $$                              # show
cat /sys/block/nvme0n1/queue/scheduler    # active I/O scheduler → [[sys]]
```

Common combo for background jobs:
```bash
nice -n 19 ionice -c 3 ./nightly-backup.sh
```

# `chrt` — real-time scheduling
```bash
chrt -p $$                        # show policy
chrt -m                           # min/max priority per policy
chrt -b -p 0 4600                 # switch to SCHED_BATCH
chrt -i 0 ./job.sh                # start with SCHED_IDLE
sudo chrt -f 50 ./audio-engine    # SCHED_FIFO priority 50 (needs root / CAP_SYS_NICE)
```
```
pid 4410's current scheduling policy: SCHED_OTHER
pid 4410's current scheduling priority: 0
```

> [!danger] Real-time can freeze the system
> A `SCHED_FIFO` task in an endless loop never gives the CPU back. The kernel keeps a safety margin (`/proc/sys/kernel/sched_rt_runtime_us`, default 950000 of 1000000 µs → 5% always left for normal tasks), but use real-time only for programs designed for it.

# `taskset` — CPU affinity
Pin a process to certain CPUs (cache locality, benchmarks, keeping cores free).
```bash
taskset -pc $$                    # show: current affinity list: 0-15
taskset -c 0-3 make -j4           # start on CPUs 0–3
taskset -pc 4,5 4600              # move running process to CPUs 4 and 5
grep Cpus_allowed_list /proc/4600/status
```
`nproc` respects affinity: `taskset -c 0-3 nproc` → `4`.

# `/proc/<pid>/sched` and friends
```bash
head -20 /proc/$$/sched           # scheduler stats: runtime, switches, policy, prio
cat /proc/$$/schedstat            # time on CPU (ns), time waiting (ns), timeslices
grep ctxt_switches /proc/$$/status
```
Many `nonvoluntary_ctxt_switches` = the process wanted to run but was preempted (CPU contention). Details on these files in [[proc]].

# With systemd
For services, set priorities in the unit instead of wrapping commands → [[services]]:
```ini
[Service]
Nice=10
IOSchedulingClass=idle
CPUSchedulingPolicy=batch
CPUAffinity=0-3
```
Or for a one-off: `systemd-run --user -p Nice=10 ./job.sh`. For CPU **shares** between groups of processes, cgroups (`CPUWeight=`, `CPUQuota=`) are stronger than nice → [[resource-limits#cgroups]].

# Find stuff

| Question                              | Command                                    |
| ------------------------------------- | ------------------------------------------ |
| Nice of a process                     | `ps -o ni= -p PID`                         |
| Niced processes                       | `ps -eo pid,ni,comm \| awk '$2 != 0'`      |
| Real-time processes                   | `ps -eo pid,cls,rtprio,comm \| awk '$2 ~ /FF\|RR/'` |
| CPU affinity                          | `taskset -pc PID`                          |
| I/O class                             | `ionice -p PID`                            |

# See also
- [[processes]] · [[resource-limits]]
- [[proc]] — `/proc/<pid>/sched`, `status`
- [[sys]] — I/O scheduler, CPU frequency
- [[capabilities]] — `CAP_SYS_NICE`
- `man 7 sched`, `man 1 nice`, `man 1 chrt`, `man 1 taskset`, `man 1 ionice`
