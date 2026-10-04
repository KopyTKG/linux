---
tags:
  - linux
  - files
  - proc
  - scripting
aliases:
  - /proc/sys/kernel/random/uuid
  - boot_id
  - /proc/self/exe
  - file-nr
  - entropy_avail
  - sysrq-trigger
  - tainted
---
A collection of small, surprising and useful files in [[proc]]. Each one answers a question in a single `cat` — no extra tools needed, which makes them great for scripts and minimal systems (containers, rescue shells).

| File                                | Gives you                                   | Example output          |
| ----------------------------------- | ------------------------------------------- | ----------------------- |
| `/proc/sys/kernel/random/uuid`      | a **new random UUID** on every read         | `4f1d2c3b-5a6e-4b7c-9d8e-0f1a2b3c4d5e` |
| `/proc/sys/kernel/random/boot_id`   | a UUID that stays the same **until reboot** | `8e7d6c5b-...`          |
| `/proc/sys/kernel/osrelease`        | kernel version (`uname -r`)                 | `6.x.y-...`             |
| `/proc/sys/kernel/ostype`           | `Linux`                                     | `Linux`                 |
| `/proc/sys/kernel/hostname`         | hostname (writable as root)                 | `myhost`                |
| `/proc/sys/fs/file-nr`              | open file handles: used, 0, max             | `13902	0	2097152`      |
| `/proc/sys/kernel/random/entropy_avail` | entropy estimate (bits)                 | `256`                   |
| `/proc/sys/kernel/threads-max`      | max threads system-wide                     | `255393`                |
| `/proc/sys/kernel/pid_max`          | highest PID before wrap-around              | `4194304`               |
| `/proc/sys/fs/inotify/max_user_watches` | max files one user can watch            | `524288`                |
| `/proc/sys/kernel/tainted`          | why the kernel is "tainted" (bit mask)      | `0` = clean             |
| `/proc/sys/kernel/randomize_va_space` | ASLR: 0 off, 1 partial, 2 full            | `2`                     |
| `/proc/sys/net/ipv4/ip_forward`     | does this machine route packets?            | `0` or `1`              |
| `/proc/sys/kernel/core_pattern`     | where core dumps go (`\|` = pipe to program) | `\|/usr/lib/systemd/systemd-coredump ...` |
| `/proc/sys/kernel/sysrq`            | which Magic SysRq functions are allowed     | `16`                    |
| `/proc/self/exe`                    | the binary of whoever reads it              | `/usr/bin/readlink`     |

# Random UUIDs: `uuid` vs `boot_id`
```bash
cat /proc/sys/kernel/random/uuid    # different every time
cat /proc/sys/kernel/random/uuid
cat /proc/sys/kernel/random/boot_id # same every time... until reboot
```
- `uuid` — free UUID generator, no `uuidgen` needed.
- `boot_id` — tells you whether the machine **rebooted** since you last looked. systemd's journal uses it to separate boots (`journalctl --list-boots`). → [[journal]]

```bash
# script: detect reboot since last run
state=~/.cache/last_boot_id
now=$(< /proc/sys/kernel/random/boot_id)
[[ -f $state && $(< "$state") != "$now" ]] && echo "machine rebooted since last run"
echo "$now" > "$state"
```

> [!note] `boot_id` vs `/etc/machine-id`
> `/etc/machine-id` identifies the **installation** (stays the same across reboots). `boot_id` identifies **this boot**. Treat `machine-id` as semi-private — it can be used to track a machine.

# `/proc/self` tricks
`/proc/self` is a symlink to the PID of **whoever is looking**, so every program sees its own directory.

```bash
readlink /proc/self/exe      # /usr/bin/readlink  — readlink sees itself
cat /proc/self/comm          # cat
readlink /proc/$$/exe        # the shell running your script
```

Practical uses:
```bash
# which shell is actually running this script? (ignores the shebang guess)
readlink /proc/$$/exe

# a script that finds its own interpreter + arguments
tr '\0' ' ' < /proc/$$/cmdline; echo

# my own open files
ls -l /proc/self/fd

# my own limits / cgroup
cat /proc/self/limits
cat /proc/self/cgroup
```

> [!tip] Recover a deleted but still running binary
> ```bash
> ls -l /proc/<pid>/exe        # → /usr/bin/foo (deleted)
> cp /proc/<pid>/exe ./foo.recovered
> ```
> Same trick works for deleted files a process still has open: `cp /proc/<pid>/fd/<n> ./recovered`.

# `/proc/sys/fs/file-nr`
```
13902	0	2097152
```
| allocated handles | allocated but unused (always 0 on modern kernels) | maximum (`fs.file-max`) |

```bash
read -r used _ max < /proc/sys/fs/file-nr
echo "$used of $max file handles ($(( used * 100 / max ))%)"
```
If you see `Too many open files`, it's usually the **per-process** limit (`ulimit -n`, `/proc/<pid>/limits`), not this global one.

# `entropy_avail`
Shows how much entropy the kernel thinks it has. Since Linux 5.18 the random number generator was rewritten and this value is basically always `256` — "fully seeded". Old advice to install `haveged` because entropy is "low" no longer applies. Just read `/dev/urandom` → [[dev-special-files]].

# `tainted`
A bit mask that says whether the kernel is in a state kernel developers won't debug (proprietary driver, unsigned module, previous oops, ...). `0` = clean.

| Bit | Letter | Meaning                                   |
| --- | ------ | ----------------------------------------- |
| 0   | `P`    | proprietary module loaded (e.g. NVIDIA)   |
| 1   | `F`    | module was force-loaded                   |
| 2   | `S`    | kernel running on out-of-spec system / CPU |
| 4   | `M`    | machine check exception happened          |
| 7   | `D`    | kernel died recently (oops / BUG)         |
| 9   | `W`    | kernel issued a warning                   |
| 12  | `O`    | out-of-tree module loaded (e.g. DKMS, VirtualBox, ZFS) |
| 13  | `E`    | unsigned module loaded                    |

```bash
t=$(< /proc/sys/kernel/tainted); flags=PFSRMBUDAWCIOEKXTN; out=""
for i in $(seq 0 17); do (( t >> i & 1 )) && out+="${flags:i:1}(bit $i) "; done
echo "$t => ${out:-not tainted}"
# example: 12292 => S(bit 2) O(bit 12) E(bit 13)
```
Full table: `Documentation/admin-guide/tainted-kernels.rst` in the kernel source, or <https://docs.kernel.org/admin-guide/tainted-kernels.html>.

# Magic SysRq — `/proc/sysrq-trigger`
```
--w------- 1 root root 0 /proc/sysrq-trigger
```
Write-only, root-only. Writing one letter performs an emergency kernel action **immediately**: sync disks, remount read-only, kill all processes, reboot — without asking. The same actions are on the keyboard as `Alt+SysRq+<key>` when `/proc/sys/kernel/sysrq` allows it.

> [!danger] Not for experimenting
> Some letters reboot or crash the machine instantly, losing unsaved work. Read about it, don't test it on a machine you care about.

| Key | Action                                  |
| --- | --------------------------------------- |
| `s` | sync all filesystems                    |
| `u` | remount all filesystems read-only       |
| `e` | send SIGTERM to all processes (except init) |
| `i` | send SIGKILL to all processes (except init) |
| `f` | call the OOM killer once                |
| `b` | reboot immediately (no sync!)           |
| `h` | print help to the kernel log            |

The famous "**R**aising **E**lephants **I**s **S**o **U**tterly **B**oring" sequence (`r e i s u b`, a few seconds apart) is the safest way to reboot a frozen system from the keyboard.

`/proc/sys/kernel/sysrq` value is a bit mask of allowed functions: `0` all disabled, `1` all enabled, other values = some (e.g. `16` = only sync). Output of SysRq commands goes to the kernel log → [[dmesg-kmsg]].

# Scripting recipes

### Quick system fingerprint without any tools
```bash
printf 'kernel:   %s\n' "$(< /proc/sys/kernel/osrelease)"
printf 'host:     %s\n' "$(< /proc/sys/kernel/hostname)"
printf 'boot id:  %s\n' "$(< /proc/sys/kernel/random/boot_id)"
printf 'uptime:   %ss\n' "$(cut -d. -f1 /proc/uptime)"
printf 'cpus:     %s\n' "$(grep -c ^processor /proc/cpuinfo)"
```

### Unique ID for a temp file / job
```bash
id=$(< /proc/sys/kernel/random/uuid)
```

### Am I inside a container?
```bash
[[ -f /.dockerenv ]] && echo docker
grep -qaE 'container=' /proc/1/environ 2>/dev/null && echo "container (systemd-nspawn/podman)"
p1=$(readlink /proc/1/ns/pid 2>/dev/null) && [[ $p1 != $(readlink /proc/self/ns/pid) ]] && echo "different PID namespace than PID 1"
```
(The last check needs permission to read PID 1's namespaces — usually root. Without it `readlink` fails and the check is skipped instead of giving a false alarm.)

# See also
- [[proc]] — the full `/proc` guide
- [[dev-special-files]] — `/dev/urandom` and friends
- [[dmesg-kmsg]] — kernel log, where SysRq output lands
- [[sys]] — hardware side of the kernel
- [[files]] — all file notes
- `man 5 proc`, <https://docs.kernel.org/admin-guide/sysrq.html>
