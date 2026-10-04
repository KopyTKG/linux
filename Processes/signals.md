---
tags:
  - linux
  - processes
  - signals
  - scripting
aliases:
  - signal
  - SIGTERM
  - SIGKILL
  - SIGHUP
  - kill -l
---
A **signal** is a small asynchronous notification the kernel delivers to a process: "the user pressed Ctrl+C", "your child exited", "please terminate". It carries only a number. The process can **handle** it (run a function), **ignore** it, or let the **default action** happen.

> [!info] Where signals come from
> - **Keyboard** in a terminal: Ctrl+C, Ctrl+Z, Ctrl+\\ → [[job-control]]
> - **Other processes**: `kill`, `pkill`, `systemctl stop` → [[sending-signals]]
> - **The kernel**: bad memory access (`SIGSEGV`), writing to a closed pipe (`SIGPIPE`), child exit (`SIGCHLD`), terminal resize (`SIGWINCH`), out of memory (`SIGKILL` from the OOM killer)

# Default actions

| Action   | What happens                                              |
| -------- | --------------------------------------------------------- |
| **Term** | Process ends                                              |
| **Core** | Process ends and writes a core dump (if enabled)          |
| **Stop** | Process is paused (state `T`)                             |
| **Cont** | Paused process continues                                  |
| **Ign**  | Nothing happens                                           |

# Common signals
Numbers below are for x86/ARM Linux. Use **names** in scripts — numbers differ on some architectures.

| #  | Name       | Default | Catchable | Typical source / use                                                     |
| -- | ---------- | ------- | --------- | ------------------------------------------------------------------------ |
| 1  | `SIGHUP`   | Term    | yes       | Terminal closed. Daemons use it as **"reload config"**                   |
| 2  | `SIGINT`   | Term    | yes       | **Ctrl+C**                                                               |
| 3  | `SIGQUIT`  | Core    | yes       | **Ctrl+\\** — quit with core dump                                        |
| 4  | `SIGILL`   | Core    | yes       | Illegal CPU instruction                                                  |
| 6  | `SIGABRT`  | Core    | yes       | `abort()`, failed `assert()`                                             |
| 9  | `SIGKILL`  | Term    | **no**    | Force kill. Cannot be caught, blocked or ignored                         |
| 11 | `SIGSEGV`  | Core    | yes       | Invalid memory access ("segmentation fault")                             |
| 13 | `SIGPIPE`  | Term    | yes       | Write to a pipe with no reader (`yes \| head -1`)                        |
| 14 | `SIGALRM`  | Term    | yes       | Timer from `alarm()`                                                     |
| 15 | `SIGTERM`  | Term    | yes       | **Polite "please exit"**. Default of `kill`, `systemctl stop`            |
| 10 | `SIGUSR1`  | Term    | yes       | User-defined (e.g. `dd` prints progress, nginx reopens logs)             |
| 12 | `SIGUSR2`  | Term    | yes       | User-defined                                                             |
| 17 | `SIGCHLD`  | Ign     | yes       | A child exited or stopped → parent should `wait()`                       |
| 18 | `SIGCONT`  | Cont    | yes*      | Continue a stopped process (`fg`, `bg`)                                  |
| 19 | `SIGSTOP`  | Stop    | **no**    | Force pause                                                              |
| 20 | `SIGTSTP`  | Stop    | yes       | **Ctrl+Z** — polite pause                                                |
| 21 | `SIGTTIN`  | Stop    | yes       | Background job tried to read the terminal                                |
| 22 | `SIGTTOU`  | Stop    | yes       | Background job tried to write the terminal (with `stty tostop`)          |
| 28 | `SIGWINCH` | Ign     | yes       | Terminal window resized (editors redraw)                                 |

\* `SIGCONT` always continues the process; a handler can additionally run.

> [!important] `SIGTERM` first, `SIGKILL` last
> `SIGTERM` lets the program save data, remove lock files and stop children. `SIGKILL` cuts it off instantly — no cleanup, temp files stay, child processes may be orphaned. Always try `TERM`, wait a few seconds, then `KILL` → [[sending-signals#Graceful shutdown]].

> [!note] What even `SIGKILL` can't do
> - A process in state `D` (uninterruptible I/O) dies only when the I/O returns.
> - A zombie (`Z`) is already dead → [[process-lifecycle#Zombies]].
> - PID 1 ignores signals it has no handler for.

# Listing signals
```bash
kill -l              # all names and numbers
kill -l 15           # TERM
kill -l TERM         # 15
kill -l $((143-128)) # decode exit code 143 → TERM
```

# Real-time signals
`SIGRTMIN` … `SIGRTMAX` (usually 34–64). Unlike normal signals they **queue** (sending 5 delivers 5; normal signals collapse into one if pending) and are delivered in order. Used by programs internally (threads libraries, systemd). `kill -l` lists them as `RTMIN+1`, etc.

# Pending, blocked, ignored, caught
For every process the kernel keeps bitmasks, visible in `/proc/<pid>/status` → [[proc]]:

| Field    | Meaning                                                   |
| -------- | --------------------------------------------------------- |
| `SigQ`   | Queued signals / limit                                    |
| `SigPnd` | Pending for this thread                                   |
| `ShdPnd` | Pending for the whole process                             |
| `SigBlk` | **Blocked** (delayed until unblocked)                     |
| `SigIgn` | **Ignored**                                               |
| `SigCgt` | **Caught** (process has a handler)                        |

Each is a 64-bit hex number; bit `N-1` set = signal `N`. Example for a bash that ignores INT and QUIT and handles USR1 and TERM:
```
SigIgn:	0000000000000006
SigCgt:	0000000000014200
```
`0x6` = bits 1 and 2 → signals 2 (INT) and 3 (QUIT).

### Decode a signal mask
```bash
decode() {
  local mask=$((16#$1)) i
  for i in {1..64}; do
    (( mask & (1 << (i - 1)) )) && printf '%s ' "$(kill -l "$i")"
  done
  echo
}
# test process: ignores INT, handles USR1 and TERM
bash -c 'trap "" INT; trap "echo x" USR1 TERM; sleep 30' &
pid=$!; sleep 0.3
for f in SigBlk SigIgn SigCgt; do
  printf '%s: ' "$f"
  decode "$(awk -v k="$f:" '$1==k {print $2}' /proc/$pid/status)"
done
kill -KILL "$pid"
```
```
SigBlk:
SigIgn: INT QUIT
SigCgt: USR1 TERM CHLD
```
(`QUIT` is ignored because non-interactive bash started in background ignores it; `CHLD` is caught by bash itself.)
Handy to answer "will this process react to `SIGHUP`, or just die?" — if `HUP` is in `SigCgt`, it has a handler (often reload).

# Signals and exit codes
A process killed by signal `N` has exit status `128 + N` in the shell → [[process-lifecycle#Exit codes]].

| Exit code | Signal                         |
| --------- | ------------------------------ |
| `130`     | `SIGINT` (Ctrl+C)              |
| `137`     | `SIGKILL` (OOM killer, `-9`)   |
| `139`     | `SIGSEGV` (crash)              |
| `141`     | `SIGPIPE`                      |
| `143`     | `SIGTERM`                      |

# Find stuff

| Question                              | Command                                          |
| ------------------------------------- | ------------------------------------------------ |
| Name ↔ number                         | `kill -l 9`, `kill -l KILL`                      |
| What does process X handle            | decode `SigCgt` (recipe above)                   |
| Was something OOM-killed              | `journalctl -k \| grep -i 'killed process'` → [[journal]] |
| Did a program crash (core dumps)      | `coredumpctl list` (systemd)                     |
| Signal docs                           | `man 7 signal`                                   |

# See also
- [[sending-signals]] — `kill`, `pkill`, `timeout`
- [[trap]] — handling signals in shell scripts
- [[job-control]] — keyboard signals
- [[proc]] — `/proc/<pid>/status`
- `man 7 signal`
