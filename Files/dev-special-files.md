---
tags:
  - linux
  - files
  - dev
  - scripting
aliases:
  - /dev/null
  - /dev/zero
  - /dev/urandom
  - /dev/random
  - /dev/full
  - /dev/tty
  - /dev/stdin
  - /dev/stdout
  - /dev/stderr
  - /dev/tcp
---
A handful of files in [[dev]] are not hardware at all. They are tiny drivers inside the kernel that behave like files and are extremely useful in scripts. All of them are **character devices** (the `c` in `ls -l`), owned by the "memory" driver with major number `1`.

```
crw-rw-rw- 1 root root 1, 3  /dev/null
crw-rw-rw- 1 root root 1, 5  /dev/zero
crw-rw-rw- 1 root root 1, 7  /dev/full
crw-rw-rw- 1 root root 1, 8  /dev/random
crw-rw-rw- 1 root root 1, 9  /dev/urandom
crw-rw-rw- 1 root tty  5, 0  /dev/tty
lrwxrwxrwx 1 root root       /dev/fd     -> /proc/self/fd
lrwxrwxrwx 1 root root       /dev/stdin  -> /proc/self/fd/0
lrwxrwxrwx 1 root root       /dev/stdout -> /proc/self/fd/1
lrwxrwxrwx 1 root root       /dev/stderr -> /proc/self/fd/2
```

| File           | Read gives            | Write does                       | Typical use                         |
| -------------- | --------------------- | -------------------------------- | ----------------------------------- |
| `/dev/null`    | nothing (EOF at once) | discards everything, succeeds    | silence output, empty input         |
| `/dev/zero`    | endless `\0` bytes    | discards                         | create files of a given size, wipe  |
| `/dev/full`    | endless `\0` bytes    | **always fails** with `ENOSPC`   | test error handling for "disk full" |
| `/dev/urandom` | endless random bytes  | mixes data into the pool         | passwords, random names, test data  |
| `/dev/random`  | random bytes          | mixes data into the pool         | same as urandom on modern kernels   |
| `/dev/tty`     | your terminal         | your terminal                    | talk to the user even when stdio is redirected |
| `/dev/stdin` / `stdout` / `stderr` | fd 0 / 1 / 2 | fd 0 / 1 / 2              | give a "file name" to programs that need one |

> [!warning] Mind the major/minor numbers
> If `/dev/null` is ever replaced by a **regular file** (e.g. someone ran `rm /dev/null` as root and then something wrote to it), everything that writes to it fills the disk. Check: `[ -c /dev/null ] && echo ok`. Fix as root: `rm /dev/null && mknod -m 666 /dev/null c 1 3`.

# `/dev/null`
```bash
cmd > /dev/null            # hide normal output
cmd 2> /dev/null           # hide errors
cmd &> /dev/null           # hide both (bash); POSIX: cmd > /dev/null 2>&1
cmd < /dev/null            # run with empty input (stops it from waiting for keyboard)
: > file.log               # truncate a file (alternative: cp /dev/null file.log)
```

> [!tip] Order of redirections matters
> `cmd > /dev/null 2>&1` → both hidden.
> `cmd 2>&1 > /dev/null` → only stdout hidden; stderr goes where stdout **was** (the terminal). Redirections are processed left to right.

# `/dev/zero`
```bash
head -c 10M /dev/zero > zeros.bin              # 10 MiB file of zeros
dd if=/dev/zero of=swapfile bs=1M count=1024   # 1 GiB file (e.g. swapfile)
truncate -s 10G sparse.img                     # 10 GiB "sparse" file — instant, uses no space
```

# `/dev/full`
Every write fails with `No space left on device`. Perfect for checking that a script notices write errors:

```bash
$ echo hi > /dev/full
bash: echo: write error: No space left on device
$ echo $?
1
```

# `/dev/urandom` and `/dev/random`
Both return cryptographically secure random bytes from the kernel CSPRNG. Since Linux 5.6, `/dev/random` only blocks until the pool is initialised once at boot, then behaves like `urandom`. **Use `/dev/urandom`** (or `getrandom()`) — there's no reason to prefer `random` anymore.

```bash
tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 20; echo   # random password
head -c 16 /dev/urandom | od -An -tx1                  # 16 random bytes in hex
od -An -tu4 -N4 /dev/urandom                           # random 32-bit number
head -c 32 /dev/urandom | base64                       # random token
cat /proc/sys/kernel/random/uuid                       # random UUID → [[proc-self-highlights]]
```

> [!note] `$RANDOM` is not random enough for secrets
> bash's `$RANDOM` is a 15-bit pseudo-random number (0–32767). Fine for jitter in a sleep, not for passwords or tokens.

# `/dev/tty`
Your **controlling terminal**, no matter where stdin/stdout point. Lets a script ask the user something even inside a pipe:

```bash
read -r -p "Continue? [y/N] " answer < /dev/tty
echo "progress..." > /dev/tty      # visible even when stdout goes to a file
```
If there is no terminal (cron job, systemd service), opening `/dev/tty` fails — check with `[ -t 0 ]` or `tty -s`.

Related: `/dev/tty1..63` are the text consoles, `/dev/pts/N` are terminal windows and SSH sessions, `/dev/console` is the system console. → [[dev]]

# `/dev/stdin`, `/dev/stdout`, `/dev/stderr`, `/dev/fd/N`
Symlinks into [[proc]] (`/proc/self/fd/`). Useful for programs that insist on a file name:

```bash
echo "data" | some-tool --input /dev/stdin
some-tool --log /dev/stderr
diff <(sort a.txt) <(sort b.txt)   # bash turns <( ) into /dev/fd/63 etc.
echo "error: oops" >&2             # shorter than > /dev/stderr
```

# `/dev/tcp` and `/dev/udp` (bash only)
These **don't exist** on disk — `ls /dev/tcp` fails. bash intercepts redirections to `/dev/tcp/HOST/PORT` and opens a network socket instead. Works in bash (and ksh), **not** in sh/dash/fish/zsh.

```bash
# is a port open? (no nc/nmap needed)
port_open() { timeout 1 bash -c "</dev/tcp/$1/$2" 2>/dev/null; }
port_open 127.0.0.1 22 && echo "ssh is listening"

# simple HTTP request
exec 3<>/dev/tcp/example.com/80
printf 'HEAD / HTTP/1.0\r\nHost: example.com\r\n\r\n' >&3
head -1 <&3          # HTTP/1.0 200 OK
exec 3>&-            # close
```
More networking tools: [[networking]].

# Scripting recipes

### Fail if output can't be written
```bash
set -e
printf 'report\n' > "$out" || { echo "cannot write $out" >&2; exit 1; }
# test: out=/dev/full ./script.sh
```

### Is stdin/stdout a terminal?
```bash
[ -t 0 ] && echo "stdin is a terminal"
[ -t 1 ] && echo "stdout is a terminal" || echo "stdout is redirected"
```
Use it to turn colours off when output goes to a file or pipe.

### Random temporary name
```bash
name="tmp-$(tr -dc 'a-z0-9' < /dev/urandom | head -c 8)"
# usually better: mktemp → [[tmp]]
```

### Wait for a TCP service to come up
```bash
until timeout 1 bash -c '</dev/tcp/127.0.0.1/5432' 2>/dev/null; do sleep 1; done
echo "database is up"
```

# See also
- [[dev]] — all device files, block vs character, major/minor
- [[proc]] — `/proc/self/fd`
- [[proc-self-highlights]] — random UUIDs and other fun `/proc` files
- [[dmesg-kmsg]] — `/dev/kmsg`
- [[tmp]] — `mktemp`
- [[files]] — all file notes
- `man 4 null`, `man 4 random`, `man bash` → **REDIRECTION**
