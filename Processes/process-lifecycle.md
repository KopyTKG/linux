---
tags:
  - linux
  - processes
  - scripting
aliases:
  - fork exec
  - zombie
  - orphan
  - exit code
---
Every process on Linux (except PID 1) is created by another process. The life of a process is four system calls: **fork → exec → exit → wait**. Knowing them explains zombies, orphans, exit codes and why `cd` must be a shell builtin.

# The four steps

```
 parent (bash, PID 2301)
   │
   │ fork()                       → exact copy, new PID 2345
   ├────────────► child (bash, 2345)
   │                 │ exec("/usr/bin/ls")   → same PID, memory replaced by ls
   │                 │ ...runs...
   │                 │ exit(0)               → becomes zombie until parent collects
   │ wait() ◄────────┘                       → parent gets exit code, zombie removed
   ▼
```

| Step     | System call            | What happens                                                                 |
| -------- | ---------------------- | ---------------------------------------------------------------------------- |
| **fork** | `fork()` / `clone()`   | Kernel copies the parent: memory (copy-on-write), open fds, env, cwd. New PID. |
| **exec** | `execve()`             | Replaces the program in the current process. PID, fds and cwd stay; memory is new. |
| **exit** | `exit()` / `_exit()`   | Process ends, frees memory and fds, keeps only its exit status.              |
| **wait** | `wait()` / `waitpid()` | Parent reads the exit status; only now does the process fully disappear.     |

> [!tip] Why `cd` is a builtin
> `cd` changes the working directory of the **current** process. An external `/usr/bin/cd` would run in a forked child, change the child's directory, and exit — the shell would stay where it was. Same for `export`, `source`, `exit`.

> [!tip] `exec` in shell scripts
> The shell builtin `exec cmd` skips the fork and replaces the shell with `cmd`. Wrapper scripts use it so the real program keeps the script's PID (important for [[services]] and signal delivery):
> ```bash
> #!/usr/bin/env bash
> export APP_ENV=prod
> exec /usr/bin/myapp "$@"
> ```

# PID 1
The first user-space process, started by the kernel after boot. On most systems it is `systemd` → see [[systemd]]. PID 1 is special:
- It **adopts orphans** (see below) and must reap them.
- It ignores signals it has no handler for, so even `kill -9 1` does nothing.
- If PID 1 dies, the kernel panics.

PID `2` is `kthreadd`, the parent of all [[processes#Kernel threads|kernel threads]].

# Orphans
If a parent exits before its child, the child becomes an **orphan** and is re-parented to PID 1 (or to the nearest "subreaper", e.g. `systemd --user`). It keeps running normally.

```bash
bash -c 'sleep 60 & echo "child $!"'     # bash exits immediately
ps -o pid,ppid,comm -p "$(pgrep -n sleep)"
```
```
    PID    PPID COMMAND
   4410    1012 sleep          # 1012 = systemd --user (subreaper), or 1
```

# Zombies
A process that has exited but whose parent has **not called `wait()` yet** is a **zombie**: state `Z`, shown as `<defunct>`. It uses no memory or CPU, only a slot in the process table and its PID.

Create one for testing — a child exits, but its parent (replaced by `sleep` via `exec`) never waits:
```bash
bash -c 'sleep 0.2 & exec sleep 30' &
sleep 1
ps -o pid,ppid,stat,comm --ppid $!
```
```
    PID    PPID STAT COMMAND
   4502    4500 Z    sleep <defunct>
```

> [!warning] You cannot kill a zombie
> It is already dead. `kill -9` does nothing. Fix the **parent**: make it reap (some programs reap on `SIGCHLD`), or kill the parent — the zombie is then adopted by PID 1, which reaps it immediately.
> A few zombies are harmless. Thousands mean a buggy parent and can exhaust PIDs (`/proc/sys/kernel/pid_max`).

```bash
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'      # list zombies with their parents
```

# Process states
Short version (full table in [[proc]] (`<pid>/stat` section)):

| State | Meaning                                             |
| ----- | --------------------------------------------------- |
| `R`   | Running / runnable                                  |
| `S`   | Sleeping, waiting for an event                      |
| `D`   | Uninterruptible sleep (usually disk or NFS I/O)     |
| `T`   | Stopped (Ctrl+Z, `SIGSTOP`)                         |
| `Z`   | Zombie                                              |
| `I`   | Idle kernel thread                                  |

Extra letters in `ps` `STAT`: `s` session leader, `+` foreground process group, `l` multi-threaded, `<` high priority, `N` low priority (niced).

# Exit codes
`exit(n)` passes a number `0–255` to the parent. In the shell it is in `$?` (bash/zsh) or `$status` (fish).

| Code      | Meaning                                                        |
| --------- | -------------------------------------------------------------- |
| `0`       | Success                                                        |
| `1`       | General error                                                  |
| `2`       | Misuse of shell builtin / bad arguments (convention)           |
| `126`     | Command found but **not executable** (permissions)             |
| `127`     | Command **not found**                                          |
| `128 + N` | Killed by signal `N`                                           |
| `130`     | `128 + 2` → `SIGINT` (Ctrl+C)                                  |
| `137`     | `128 + 9` → `SIGKILL` (often the OOM killer, or `kill -9`)     |
| `143`     | `128 + 15` → `SIGTERM`                                         |

```bash
sleep 30 & pid=$!
kill -TERM "$pid"; wait "$pid"; echo $?     # 143
nonexistentcmd; echo $?                     # 127
touch f; ./f; echo $?                       # 126 (no x bit)
```

```bash
# decode an exit code
code=137
(( code > 128 )) && echo "killed by SIG$(kill -l $((code - 128)))"   # killed by SIGKILL
```

> [!note] Pipelines
> `$?` after `a | b | c` is the exit code of `c` only. Use `set -o pipefail` (fail if any part fails) or `"${PIPESTATUS[@]}"` (bash: all codes).

# See also
- [[processes]] — overview
- [[signals]] — what `128 + N` refers to
- [[process-tree]] — seeing parents and children
- [[proc]] — `/proc/<pid>/stat` field `state`
- `man 2 fork`, `man 2 execve`, `man 2 wait`
