---
tags:
  - linux
  - processes
  - signals
  - scripting
  - bash
aliases:
  - trap
  - cleanup on exit
  - signal handler
---
`trap` is the shell builtin for **handling signals** in scripts: run some code when the script gets `SIGINT`, `SIGTERM`, or simply when it exits. The main use is **cleanup** — removing temp files, killing background jobs, releasing locks — no matter how the script ends.

# Syntax

```bash
trap 'commands' SIGNAL...     # set a handler
trap '' SIGNAL...             # ignore the signal
trap - SIGNAL...              # reset to default
trap -p                       # show current traps
trap -l                       # list signal names (like kill -l)
```
Signal names are written without `SIG`: `INT`, `TERM`, `HUP`.

# Special pseudo-signals (bash)

| Name     | Fires when                                                  |
| -------- | ----------------------------------------------------------- |
| `EXIT`   | The shell exits — for **any** reason (end, `exit`, error with `set -e`, most signals) |
| `ERR`    | A command returns non-zero (same rules as `set -e`)         |
| `DEBUG`  | Before every simple command                                 |
| `RETURN` | A function or `source`d file returns                        |

> [!tip] Trap `EXIT`, not every signal
> Put cleanup in `EXIT`. Then add `INT`/`TERM` traps that just call `exit` — that turns the signal into a normal exit, which runs the `EXIT` trap once.

# The standard cleanup pattern

```bash
#!/usr/bin/env bash
set -euo pipefail

tmp=$(mktemp -d)                       # → [[tmp]]
cleanup() {
  rm -rf -- "$tmp"
  kill $(jobs -p) 2>/dev/null || true  # stop our background jobs
}
trap cleanup EXIT
trap 'exit 130' INT                    # 128 + 2
trap 'exit 143' TERM                   # 128 + 15

curl -so "$tmp/data.json" https://example.com/data.json
process "$tmp/data.json"
```
Whether the script finishes, fails (`set -e`), or gets Ctrl+C, `$tmp` is removed. Exit codes stay meaningful → [[process-lifecycle#Exit codes]].

Test:
```bash
bash -c 'tmp=$(mktemp); trap "rm -f \"$tmp\"; echo cleaned" EXIT; trap "exit 130" INT; kill -INT $$; echo never'
echo $?
```
```
cleaned
130
```

# Ignoring signals
```bash
trap '' INT            # Ctrl+C does nothing during the critical section
critical_update
trap - INT             # back to default
```
Ignored signals are **inherited** by child processes (`SigIgn` in `/proc/<pid>/status` → [[signals]]); handlers are not (exec resets them to default).

> [!warning] `SIGKILL` and `SIGSTOP` can't be trapped
> `trap '...' KILL` is silently useless. If something is killed with `-9`, no cleanup runs — another reason to send `TERM` first → [[sending-signals#Graceful shutdown]].

# `ERR` and `DEBUG`

```bash
set -E                                          # make ERR trap work inside functions
trap 'echo "error on line $LINENO: $BASH_COMMAND" >&2' ERR
false
```
```
error on line 3: false
```

```bash
trap 'echo "+ $BASH_COMMAND" >&2' DEBUG        # poor man's set -x
```

# Reload on `SIGHUP`
Daemon-style script that rereads config without restarting:
```bash
#!/usr/bin/env bash
load_config() { source ./app.conf; echo "config loaded: interval=$interval"; }
trap load_config HUP
load_config
while true; do
  do_work
  sleep "$interval" & wait $!        # see note below
done
```
Then from another shell: `kill -HUP <pid>`.

> [!note] Traps run only between commands
> Bash runs a trap **after the current foreground command finishes**. During `sleep 300` a `SIGTERM` waits up to 5 minutes. Trick: run the long command in background and `wait` for it — `wait` is interrupted by trapped signals immediately:
> ```bash
> sleep 300 & wait $!
> ```

# Subshells and pipelines
- `( ... )`, `$( ... )` and every part of a pipeline run in **subshells**. They inherit ignored signals, but **traps are reset** (except in bash with `set -E`/`set -T` for `ERR`/`DEBUG`/`RETURN`).
- Ctrl+C sends `SIGINT` to the whole foreground process group — every part of the pipeline gets it.
- An `EXIT` trap set inside a subshell runs when the subshell ends.

```bash
trap 'echo main exit' EXIT
( trap 'echo sub exit' EXIT; true )
```
```
sub exit
main exit
```

# Fish
Fish uses event handlers instead of `trap` (a `trap` wrapper exists, but functions are the native way).

```fish
function on_int --on-signal INT
    echo "got Ctrl+C"
end

function cleanup --on-event fish_exit
    rm -rf $tmpdir
end
```
`functions --handlers` lists active handlers → [[fish-config]].

# Find stuff

| Question                           | How                                        |
| ---------------------------------- | ------------------------------------------ |
| What traps are set                 | `trap -p`                                  |
| What does a process catch/ignore   | `SigCgt`/`SigIgn` → [[signals]]            |
| Signal names                       | `trap -l`, `kill -l`                       |

# See also
- [[signals]] · [[sending-signals]] · [[job-control]]
- [[tmp]] — `mktemp`, temp file cleanup
- [[bashrc]] — traps in interactive shells
- `help trap` (bash), `man bash` → "SIGNALS"
