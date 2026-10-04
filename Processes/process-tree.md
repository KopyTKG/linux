---
tags:
  - linux
  - processes
  - scripting
aliases:
  - ps
  - pstree
  - pgrep
  - top
  - process group
  - session
---
Every process has a parent, so all processes form a **tree** rooted at PID 1. This note is about looking at that tree: `ps`, `pstree`, `pgrep`, `top`, plus process groups and sessions, which decide who receives Ctrl+C.

# `ps`
`ps` reads `/proc` → see [[proc]]. It accepts **two option styles**, which is why examples online look so different:

| Style         | Example      | Note                                    |
| ------------- | ------------ | --------------------------------------- |
| UNIX / POSIX  | `ps -ef`     | Options with a dash                     |
| BSD           | `ps aux`     | Options **without** a dash              |

Both show all processes; columns differ.

### `ps aux`
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0  22340 13120 ?        Ss   09:12   0:02 /sbin/init
alice       3120  4.1  2.6 3815204 870412 ?      Sl   09:20   3:15 /usr/lib/firefox/firefox
alice       4410  0.0  0.0   7804  4864 pts/1    Ss   10:02   0:00 bash
```

| Column    | Meaning                                                         |
| --------- | --------------------------------------------------------------- |
| `USER`    | Effective user                                                  |
| `%CPU`    | CPU time used / time alive (average, not "right now")           |
| `%MEM`    | RSS as % of RAM                                                 |
| `VSZ`     | Virtual memory, KiB                                             |
| `RSS`     | Real RAM, KiB                                                   |
| `TTY`     | Controlling terminal, `?` = none (daemons)                      |
| `STAT`    | State + flags → [[process-lifecycle#Process states]]            |
| `START`   | Start time                                                      |
| `TIME`    | Total CPU time used                                             |
| `COMMAND` | Command line; `[name]` = kernel thread                          |

### `ps -ef`
```
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 09:12 ?        00:00:02 /sbin/init
alice       4410    4402  0 10:02 pts/1    00:00:00 bash
```
Shows `PPID`, which `aux` doesn't.

### Custom columns — best for scripts
`-o` picks columns; `=` after a name removes the header.

```bash
ps -o pid,ppid,pgid,sid,tty,stat,ni,%cpu,%mem,rss,etime,cmd -p $$
ps -eo pid,user,rss,comm --sort=-rss | head -6     # top 5 by RAM
ps -eo pid,etimes,comm --sort=-etimes | head        # longest running
ps -o ppid= -p 4410                                 # just the parent PID, no header
ps -eo stat= | cut -c1 | sort | uniq -c             # count processes per state
```

| Useful `-o` field | Meaning                                    |
| ----------------- | ------------------------------------------ |
| `pid,ppid`        | Process and parent                         |
| `pgid,sid`        | Process group, session                     |
| `tid,nlwp`        | Thread ID, thread count (with `-L`)        |
| `user,uid`        | Owner                                      |
| `comm`            | Short name (15 chars)                      |
| `args` / `cmd`    | Full command line                          |
| `stat`            | State + flags                              |
| `ni,pri`          | Nice, priority → [[priority-scheduling]]   |
| `rss,vsz`         | Memory in KiB                              |
| `etime`,`etimes`  | Elapsed time (`[[dd-]hh:]mm:ss` / seconds) |
| `lstart`          | Exact start date                           |
| `wchan`           | Kernel function it sleeps in               |

Selecting: `-e` all, `-p PID`, `-u alice`, `-C nginx` (by name), `--ppid PID` (children of).

# `pstree`
```bash
pstree -p          # with PIDs
pstree -p alice    # only alice's processes
pstree -s -p $$    # path from PID 1 down to my shell
ps -ef --forest    # same idea with ps
```
```
systemd(1)───systemd(1012)───sshd(1150)───bash(4410)───pstree(4520)
```

# `pgrep`, `pidof`

```bash
pgrep nginx          # PIDs whose name contains "nginx"
pgrep -x nginx       # exact name match
pgrep -f 'python3 app.py'   # match full command line
pgrep -u alice bash  # by user
pgrep -l ssh         # with names
pgrep -o nginx       # oldest (usually the master process)
pgrep -n nginx       # newest
pidof nginx          # exact program name, space separated
```

> [!warning] `pgrep -f` matches itself in scripts
> A script containing `pgrep -f myjob` may find its own `bash -c ... myjob ...` command line. Use `pgrep -x`, or a pattern like `'[m]yjob'`.

# `top` / `htop`
```
top - 10:15:01 up  1:03,  2 users,  load average: 0.52, 0.61, 0.70
Tasks: 312 total,   1 running, 311 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.1 us,  1.0 sy,  0.0 ni, 95.6 id,  0.2 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :  31987.4 total,  18011.2 free,   6120.5 used,   8231.0 buff/cache

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   3120 alice     20   0 3815204 870412 210944 S   4.0   2.7   3:15.22 firefox
```

| Field             | Meaning                                                           |
| ----------------- | ----------------------------------------------------------------- |
| `load average`    | From `/proc/loadavg` → [[proc]] (`/proc/loadavg`)                   |
| `us sy ni id wa`  | user, system, niced, idle, I/O wait — from `/proc/stat`           |
| `hi si st`        | hardware IRQ, software IRQ, stolen (VMs)                          |
| `PR` / `NI`       | Priority / nice                                                   |
| `VIRT RES SHR`    | Virtual, resident, shared memory (KiB)                            |
| `%CPU`            | Since last refresh (unlike `ps`)                                  |

Keys in `top`: `P` sort by CPU, `M` by memory, `k` kill, `r` renice, `1` per-CPU, `H` threads, `c` full command, `q` quit.
Non-interactive: `top -b -n 1 | head -15`.

# Process groups and sessions
Processes are grouped in two levels, used by the terminal and [[job-control]]:

```
session (SID 4410) ── controlling terminal pts/1
 ├── process group 4410: bash                        ← session leader
 ├── process group 4600: sleep 100 &                 ← background job
 └── process group 4610: grep foo file | sort        ← foreground job (both processes)
```

| Concept                | What                                                                    |
| ---------------------- | ----------------------------------------------------------------------- |
| **Process group** (PGID) | One job: a pipeline started by the shell. PGID = PID of its first process. |
| **Session** (SID)      | All groups started from one login/terminal. SID = PID of the leader (the shell). |
| **Controlling terminal** | The tty of the session. Ctrl+C sends `SIGINT` to the **foreground group** only. |
| **Foreground group**   | The one group that may read from the terminal (`+` in `STAT`).        |

```bash
ps -o pid,pgid,sid,tty,stat,comm -t "$(tty | sed 's|/dev/||')"
```
```
    PID    PGID     SID TT       STAT COMMAND
   4410    4410    4410 pts/1    Ss   bash
   4600    4600    4410 pts/1    S    sleep
   4610    4610    4410 pts/1    R+   ps
```

Why it matters:
- Ctrl+C kills the whole pipeline (whole group), not just one process.
- `kill -- -PGID` signals a whole group → [[sending-signals]].
- When the terminal closes, the session gets `SIGHUP` → [[job-control]] (`nohup`, `disown`, `setsid`).
- Daemons have `TTY ?` because they left the session with `setsid`.

# Recipes

### Children of a process (recursive)
```bash
descendants() {
  local c
  for c in $(pgrep -P "$1"); do echo "$c"; descendants "$c"; done
}
descendants $$
```

### Parent chain up to PID 1
```bash
pid=$$
while (( pid > 0 )); do
  printf '%s %s\n' "$pid" "$(ps -o comm= -p "$pid")"
  pid=$(ps -o ppid= -p "$pid" | tr -d ' ')
done
```

### Count processes per user
```bash
ps -eo user= | sort | uniq -c | sort -rn
```

# See also
- [[processes]] · [[process-lifecycle]] · [[job-control]] · [[sending-signals]]
- [[proc]] — where `ps` gets its data
- `man 1 ps`, `man 1 pgrep`, `man 1 top`, `man 7 credentials`
