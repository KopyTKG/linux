---
tags:
  - linux
  - processes
  - scripting
aliases:
  - process
  - processes
  - PID
---
A **process** is a running instance of a program. The program is a file on disk (`/usr/bin/bash`); the process is what the kernel creates when that file is executed: its own memory, its own open files, its own identity. Run `bash` three times and you get one program but three processes.

> [!info] What the kernel tracks for every process
> | Thing              | Example                     | Where to see it                          |
> | ------------------ | --------------------------- | ---------------------------------------- |
> | PID                | `2345`                      | `echo $$`, `/proc/<pid>/`                |
> | Parent PID (PPID)  | `2301`                      | `ps -o ppid= -p 2345`, `PPid:` in status |
> | Program            | `/usr/bin/bash`             | `readlink /proc/2345/exe`                |
> | Arguments          | `bash -l`                   | `/proc/2345/cmdline`                     |
> | Credentials        | UID/GID `1000` (`alice`)    | `Uid:`/`Gid:` in `/proc/2345/status`     |
> | Memory             | code, heap, stack, libs     | `/proc/2345/maps`, `VmRSS:`              |
> | Open files         | stdin/stdout/stderr, files  | `/proc/2345/fd/`                         |
> | Working directory  | `/home/alice`               | `readlink /proc/2345/cwd`                |
> | Environment        | `PATH=...`                  | `/proc/2345/environ`                     |
> | State              | `S` (sleeping)              | `ps -o stat=`, `State:`                  |
> | Signal handlers    | which signals are caught    | `SigCgt:` in status → [[signals]]        |
> | Limits, priority   | nice `0`, 1024 open files   | `/proc/2345/limits` → [[resource-limits]] |
>
> All of it is visible under `/proc/<pid>/` → see [[proc]].

# Program vs process vs thread

| Term        | What it is                                                         | Shares memory with siblings? |
| ----------- | ------------------------------------------------------------------ | ---------------------------- |
| **Program** | Executable file on disk (`/usr/bin/nginx`)                         | —                            |
| **Process** | Running program with its own address space and PID                 | No (each has its own)        |
| **Thread**  | Second (third, ...) line of execution **inside** one process       | Yes, same memory and fds     |

On Linux, threads and processes are both "tasks" for the kernel. A thread has its own ID (TID); the main thread's TID equals the PID. Threads show up in `/proc/<pid>/task/<tid>/` and with `ps -L`:

```bash
ps -L -o pid,tid,nlwp,comm -p "$(pgrep -o firefox)"
```
```
    PID     TID NLWP COMMAND
   3120    3120   78 firefox
   3120    3127   78 IPC I/O Parent
   3120    3131   78 Timer
```
`NLWP` = number of threads ("light-weight processes").

# Kernel threads
Processes with names in brackets, like `[kworker/0:1]` or `[ksoftirqd/3]`, are **kernel threads**. They run kernel code only, have no program file and an empty `cmdline`, and their parent is PID `2` (`kthreadd`).

# Notes in this folder

| Note                       | Covers                                                               |
| -------------------------- | -------------------------------------------------------------------- |
| [[process-lifecycle]]      | fork, exec, wait, exit, PID 1, orphans, zombies, exit codes          |
| [[process-tree]]           | `ps`, `pstree`, `pgrep`, `top`, sessions and process groups          |
| [[job-control]]            | `&`, `jobs`, `fg`, `bg`, Ctrl+Z, `nohup`, `disown`, `setsid`         |
| [[signals]]                | What signals are, full table, decoding signal masks                  |
| [[sending-signals]]        | `kill`, `pkill`, `killall`, `timeout`, graceful shutdown             |
| [[trap]]                   | Catching signals in scripts, cleanup on exit                         |
| [[priority-scheduling]]    | `nice`, `renice`, `ionice`, `chrt`, `taskset`                        |
| [[resource-limits]]        | `ulimit`, `prlimit`, OOM killer, cgroups                             |

# Find stuff

| Question                                   | Command                                          |
| ------------------------------------------ | ------------------------------------------------ |
| My shell's PID                             | `echo $$` (bash/zsh), `echo $fish_pid` (fish)    |
| PID of the last background job             | `echo $!` (bash/zsh), `echo $last_pid` (fish)    |
| PIDs of a program                          | `pgrep -x nginx`, `pidof nginx`                  |
| Full process list                          | `ps -ef`, `ps aux`                               |
| Tree view                                  | `pstree -p`, `ps -ef --forest`                   |
| Live view                                  | `top`, `htop`                                    |
| Who started it                             | `ps -o user=,ppid= -p PID`                       |
| Which program file                         | `readlink /proc/PID/exe`                         |
| Which systemd unit owns it                 | `cat /proc/PID/cgroup`, `systemctl status PID`   |
| How many processes are there               | `ls -d /proc/[0-9]* \| wc -l`                    |

# See also
- [[proc]] — `/proc/<pid>/` files in detail
- [[systemd]] — PID 1 and services → [[services]]
- [[users]] — UIDs a process runs as, [[capabilities]]
- `man 7 credentials`, `man 2 fork`, `man 1 ps`
