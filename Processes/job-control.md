---
tags:
  - linux
  - processes
  - shell
  - scripting
aliases:
  - jobs
  - fg
  - bg
  - nohup
  - disown
  - background process
---
**Job control** is how an interactive shell runs several commands from one terminal: one in the **foreground** (gets your keyboard), others in the **background** or **stopped**. A "job" is one process group — usually one pipeline → see [[process-tree#Process groups and sessions]].

# Cheat sheet

| Action                               | How                     | Signal sent           |
| ------------------------------------ | ----------------------- | --------------------- |
| Start in background                  | `cmd &`                 | —                     |
| List jobs                            | `jobs -l`               | —                     |
| Stop (pause) foreground job          | **Ctrl+Z**              | `SIGTSTP`             |
| Interrupt foreground job             | **Ctrl+C**              | `SIGINT`              |
| Quit with core dump                  | **Ctrl+\\**             | `SIGQUIT`             |
| Continue stopped job in background   | `bg %1`                 | `SIGCONT`             |
| Bring job to foreground              | `fg %1`                 | `SIGCONT` if stopped  |
| Kill a job                           | `kill %1`               | `SIGTERM`             |
| Wait for background jobs             | `wait` / `wait %1`      | —                     |
| End-of-input (not a signal)          | **Ctrl+D**              | — (closes stdin)      |

Keyboard keys are configurable per terminal: `stty -a | grep -o 'intr = [^;]*\|susp = [^;]*\|quit = [^;]*'`.

# Job specs

| Spec   | Means                                |
| ------ | ------------------------------------ |
| `%1`   | Job number 1                         |
| `%+` / `%%` | Current job (last started/stopped) |
| `%-`   | Previous job                         |
| `%vim` | Job whose command starts with `vim`  |
| `%?log`| Job whose command contains `log`     |

```console
$ sleep 100 &
[1] 4600
$ vim notes.txt          # press Ctrl+Z
[2]+  Stopped                 vim notes.txt
$ jobs -l
[1]-  4600 Running                 sleep 100 &
[2]+  4612 Stopped                 vim notes.txt
$ fg %2                  # back into vim
```

> [!note] Background jobs and the terminal
> - A background job that **reads** from the terminal gets `SIGTTIN` and stops.
> - A background job that **writes** prints over your prompt (unless `stty tostop` is set, then it gets `SIGTTOU`).
> - Redirect output of background jobs: `cmd > out.log 2>&1 &`.

# `$!` and `$$`

| Variable | bash/zsh       | fish            | Meaning                         |
| -------- | -------------- | --------------- | ------------------------------- |
| `$$`     | `$$`           | `$fish_pid`     | PID of the current shell        |
| `$!`     | `$!`           | `$last_pid`     | PID of the last background job  |
| `$?`     | `$?`           | `$status`       | Exit code of the last command   |
| `$PPID`  | `$PPID`        | —               | Parent of the shell             |

> [!warning] `$$` in subshells
> `$$` stays the PID of the **main** shell even inside `( ... )` or `$( ... )`. Use `$BASHPID` (bash) for the real PID of a subshell.

# `nohup`, `disown`, `setsid` — surviving logout
When a terminal closes, the kernel sends `SIGHUP` to the session; the shell forwards it to its jobs and they usually die. Three ways to avoid that:

| Tool                  | What it does                                                       | Use when                              |
| --------------------- | ------------------------------------------------------------------ | ------------------------------------- |
| `nohup cmd &`         | Starts `cmd` ignoring `SIGHUP`; output to `nohup.out` if a tty     | Starting something you know is long   |
| `disown %1`           | Removes job from the shell's job table, so shell won't forward HUP | You already started it with `&`       |
| `setsid cmd`          | Runs `cmd` in a **new session** with no controlling terminal       | Fully detach (daemon-like)            |
| `systemd-run --user cmd` | Runs as a transient systemd unit                                 | Want logs and management → [[user-units]] |
| `tmux` / `screen`     | Keeps a whole terminal alive                                       | Interactive long work                 |

```bash
nohup ./backup.sh > backup.log 2>&1 &
./long-task.sh & disown
setsid -f ./server.sh > server.log 2>&1 < /dev/null
```

# Background jobs in scripts
Job control (`fg`, `bg`, Ctrl+Z) is **off** in scripts, but `&`, `$!` and `wait` work.

### Run things in parallel, wait for all, collect exit codes
```bash
#!/usr/bin/env bash
pids=()
for host in web1 web2 web3; do
  ping -c1 -W1 "$host" > /dev/null 2>&1 &
  pids+=("$!")
done
fail=0
for p in "${pids[@]}"; do
  wait "$p" || fail=$((fail + 1))      # wait PID returns that job's exit code
done
echo "$fail failed"
```

### Limit parallel jobs
```bash
max=4
for f in *.log; do
  gzip "$f" &
  while (( $(jobs -rp | wc -l) >= max )); do wait -n; done   # bash 4.3+
done
wait
```

### Kill background jobs when the script ends
```bash
trap 'kill $(jobs -p) 2>/dev/null' EXIT
./worker.sh &
./worker.sh &
wait
```
More in [[trap]].

# Fish differences

| bash                   | fish                                   |
| ---------------------- | -------------------------------------- |
| `cmd &`, `jobs`, `fg`, `bg` | Same                              |
| `$!`                   | `$last_pid`                            |
| `disown %1`            | `disown %1` (also takes a PID)         |
| `wait`                 | `wait` (also `wait PID` / `wait name`) |
| `%1` job spec          | `%1` works in `fg`/`bg`/`kill`         |

# See also
- [[process-tree]] — process groups and sessions behind job control
- [[signals]] — `SIGTSTP`, `SIGCONT`, `SIGHUP`, `SIGTTIN`
- [[trap]] — cleanup when a script is interrupted
- [[bashrc]] · [[fish-config]]
- `man bash` → "JOB CONTROL", `man 1 setsid`, `man 1 nohup`
