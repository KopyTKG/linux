---
tags:
  - linux
  - processes
  - signals
  - scripting
aliases:
  - kill
  - pkill
  - killall
  - timeout
---
Signals are sent with `kill` (by PID), `pkill` / `killall` (by name), `timeout` (after a time limit) or `systemctl kill` (to a whole service). Despite the name, `kill` just **sends a signal** — `SIGTERM` by default. What each signal means is in [[signals]].

> [!warning] Permissions
> You may signal only your **own** processes (same real or effective UID). Signalling other users' processes needs root or `CAP_KILL` → [[capabilities]].

# `kill`

```bash
kill 4600              # SIGTERM (15)
kill -TERM 4600        # same, explicit
kill -s TERM 4600      # same, POSIX form
kill -9 4600           # SIGKILL — last resort
kill -HUP 1150         # ask a daemon to reload config
kill -STOP 4600        # pause
kill -CONT 4600        # resume
kill %1                # job 1 of this shell → [[job-control]]
kill 4600 4601 4602    # several PIDs
```

### `kill -0` — does the process exist?
Signal `0` is not sent; only existence and permission are checked. Perfect for scripts.
```bash
if kill -0 "$pid" 2>/dev/null; then echo "running"; else echo "gone (or not mine)"; fi
```
Alternative without permission issues: `[[ -d /proc/$pid ]]`.

### Process groups: `kill -- -PGID`
A **negative** PID means "the whole process group" → [[process-tree#Process groups and sessions]]. Kills a pipeline or a script with all its children at once.
```bash
pgid=$(ps -o pgid= -p "$pid" | tr -d ' ')
kill -TERM -- "-$pgid"        # -- stops kill from reading -4600 as an option
```

# `pkill` / `pgrep`
Same matching options as [[process-tree|pgrep]]; always test with `pgrep -l` first.

```bash
pgrep -l -x nginx       # check what would match
pkill -x nginx          # SIGTERM to processes named exactly nginx
pkill -HUP -x nginx     # reload
pkill -f 'python3 app.py'   # match full command line
pkill -u alice          # everything of user alice (careful!)
pkill -t pts/3          # everything on terminal pts/3
pkill -o -x nginx       # only the oldest match
```

> [!danger] Name matching is a regex on 15 chars
> Without `-x`, `pkill ssh` also kills `sshd`, `ssh-agent`, ... `comm` is cut to 15 characters, so long names need `-f`. Run `pgrep -l` first.

# `killall`
Kills by **exact** name (with the common Linux `psmisc` version).
```bash
killall nginx
killall -s HUP nginx
killall -i firefox      # ask for each
killall -w myjob        # wait until they're dead
```

# `timeout`
Run a command with a time limit.
```bash
timeout 10 curl https://example.com        # SIGTERM after 10 s
timeout -s KILL 5 ./hang.sh                # SIGKILL after 5 s
timeout -k 3 10 ./job.sh                   # TERM at 10 s, KILL 3 s later if still alive
timeout 1 sleep 5; echo $?                 # 124 = timed out
```

| Exit code | Meaning                                  |
| --------- | ---------------------------------------- |
| `124`     | Timed out (TERM sent)                    |
| `137`     | Killed with `KILL`                       |
| `125`     | `timeout` itself failed                  |
| other     | Command's own exit code                  |

# Graceful shutdown
The standard pattern: `TERM`, wait, then `KILL`.

```bash
stop() {
  local pid=$1 grace=${2:-10} i
  kill -TERM "$pid" 2>/dev/null || return 0          # already gone
  for (( i = 0; i < grace * 10; i++ )); do
    kill -0 "$pid" 2>/dev/null || return 0           # exited in time
    sleep 0.1
  done
  echo "pid $pid ignored SIGTERM, sending SIGKILL" >&2
  kill -KILL "$pid"
}
stop 4600 5
```
`systemctl stop` does exactly this for services (`TimeoutStopSec=`, default 90 s) → [[services]].

# `systemctl kill`
For services, signal the unit instead of hunting PIDs. It reaches **all** processes in the unit's cgroup → [[systemd]], [[units]].
```bash
systemctl kill nginx.service                    # SIGTERM to all processes of the unit
systemctl kill -s HUP --kill-whom=main nginx    # only the main process
systemctl reload nginx                          # preferred: uses ExecReload=
```

# Other ways to send signals

| Tool                       | Example                              |
| -------------------------- | ------------------------------------ |
| `top` / `htop`             | press `k` / `F9`                     |
| `xkill` (X11)              | click a window to kill its client    |
| `loginctl terminate-user`  | end all of a user's sessions → [[users]] |
| Bash builtin               | `kill` in bash also takes `%job`     |

# Find stuff

| Question                            | Command                                  |
| ----------------------------------- | ---------------------------------------- |
| Is PID alive                        | `kill -0 PID`, `[[ -d /proc/PID ]]`      |
| What would `pkill X` hit            | `pgrep -l X` / `pgrep -a X`              |
| Process group of a PID              | `ps -o pgid= -p PID`                     |
| Does it handle `HUP`                | decode `SigCgt` → [[signals#Decode a signal mask]] |

# See also
- [[signals]] — meaning of each signal
- [[trap]] — receiving side in scripts
- [[job-control]] — `kill %1`, Ctrl+C
- [[services]] — how systemd stops services
- `man 1 kill`, `man 1 pkill`, `man 1 timeout`, `man 2 kill`
