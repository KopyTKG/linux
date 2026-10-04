---
tags:
  - linux
  - permissions
  - security
  - capabilities
aliases:
  - Capabilities
  - getcap
  - setcap
  - CAP_SYS_ADMIN
---
Traditional Unix has two kinds of process: **root** (UID 0, can do everything) and **everyone else**. Linux splits root's power into about 40 separate **capabilities**. A process can hold only the ones it needs — e.g. "may bind port 80" without "may read every file".

This kernel knows `cat /proc/sys/kernel/cap_last_cap` + 1 = **41** capabilities (numbered 0–40).

> [!info] Root = all capabilities
> A root process is powerful *because* it holds all capabilities, not because UID 0 is checked everywhere. Take capabilities away from a root process and it fails like a normal user. That's how systemd sandboxes services that run as root.

# Common capabilities

| Capability              | Bit | Allows                                                             |
| ----------------------- | --- | ------------------------------------------------------------------ |
| `CAP_CHOWN`             | 0   | Change file owner                                                  |
| `CAP_DAC_OVERRIDE`      | 1   | Ignore read/write/execute permissions → [[file-permissions]]        |
| `CAP_DAC_READ_SEARCH`   | 2   | Ignore read permission on files and dirs (backups, `btop`)         |
| `CAP_FOWNER`            | 3   | Act as owner of any file (`chmod`, set ACLs)                       |
| `CAP_KILL`              | 5   | Send signals to any process                                        |
| `CAP_SETUID` / `CAP_SETGID` | 7 / 6 | Change UID / GID (`su`, `sudo`, login)                     |
| `CAP_LINUX_IMMUTABLE`   | 9   | Set `i` and `a` attributes → [[attributes]]                         |
| `CAP_NET_BIND_SERVICE`  | 10  | Bind ports below 1024                                              |
| `CAP_NET_ADMIN`         | 12  | Configure interfaces, routes, firewall → [[networking]]             |
| `CAP_NET_RAW`           | 13  | Raw sockets (`ping` the old way, packet sniffing)                  |
| `CAP_SYS_MODULE`        | 16  | Load kernel modules                                                |
| `CAP_SYS_PTRACE`        | 19  | Debug / trace any process                                          |
| `CAP_SYS_ADMIN`         | 21  | "The new root": mount, many admin ops. Avoid giving it out          |
| `CAP_SYS_BOOT`          | 22  | Reboot                                                             |
| `CAP_SYS_NICE`          | 23  | Raise priority, real-time scheduling (`kwin_wayland` here)         |
| `CAP_SYS_TIME`          | 25  | Set the system clock (`systemd-timesyncd` here)                    |
| `CAP_PERFMON`, `CAP_BPF` | 38, 39 | Performance monitoring, BPF programs                          |

Full list: `man 7 capabilities`.

# Capability sets of a process

Every process (thread, really) has five sets. Shown as hex bitmasks in `/proc/<pid>/status` → [[proc]]:

```
$ grep Cap /proc/self/status          # a normal shell
CapInh:	0000000800000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	000001ffffffffff
CapAmb:	0000000000000000

$ grep Cap /proc/1/status             # PID 1 (systemd, root)
CapPrm:	000001ffffffffff
CapEff:	000001ffffffffff
```

| Set          | Meaning                                                                       |
| ------------ | ----------------------------------------------------------------------------- |
| `CapEff` effective   | What the kernel actually checks **right now**                         |
| `CapPrm` permitted   | What the process *may* switch on in effective                         |
| `CapInh` inheritable | What may be passed to a program across `execve()` (with file caps)    |
| `CapBnd` bounding    | Upper limit; once dropped, can never come back in this process tree   |
| `CapAmb` ambient     | Kept across `execve()` of normal programs (no file caps needed)       |

`000001ffffffffff` = bits 0–40 all set = every capability.

### Decoding the hex

```bash
capsh --decode=0000000000002500
# 0x0000000000002500=cap_setpcap,cap_net_bind_service,cap_net_raw
```
Without `capsh`, list bit numbers in pure bash:
```bash
hex=$(awk '/^CapEff/ {print $2}' /proc/$pid/status)
v=$((16#$hex))
for i in $(seq 0 "$(cat /proc/sys/kernel/cap_last_cap)"); do (( v >> i & 1 )) && printf '%d ' "$i"; done; echo
# 8 10 13
```
Then look the numbers up in the table above or `/usr/include/linux/capability.h`.

Real services on this machine:

| Process            | User              | CapEff                                          |
| ------------------ | ----------------- | ----------------------------------------------- |
| `systemd` (PID 1)  | root              | all                                             |
| `systemd-resolved` | `systemd-resolve` | `cap_setpcap,cap_net_bind_service,cap_net_raw`  |
| `systemd-timesyncd`| `systemd-timesync`| `cap_sys_time`                                  |
| `kwin_wayland`     | alice              | `cap_sys_nice` (from file capability)           |
| your shell         | alice              | none                                            |

So `resolved` can listen on DNS port 53 without being root, and `timesyncd` can set the clock and nothing else → [[dns]], [[systemd]].

# File capabilities

Instead of making a binary setuid root ([[special-bits]]), you can attach **only the needed capabilities** to the file. They're stored in the `security.capability` extended attribute.

```bash
getcap /usr/bin/btop
# /usr/bin/btop cap_dac_read_search,cap_perfmon=ep

getcap -r /usr/bin 2>/dev/null     # all binaries with file caps
```

Examples found here:
```
/usr/bin/btop          cap_dac_read_search,cap_perfmon=ep
/usr/bin/kwin_wayland  cap_sys_nice=ep
/usr/bin/newuidmap     cap_setuid=ep
/usr/bin/newgidmap     cap_setgid=ep
```

The suffix says which sets get the caps when the file runs: `e` effective, `p` permitted, `i` inheritable. `=ep` is the usual "just have them".

```bash
sudo setcap cap_net_bind_service=+ep ./myserver   # may bind :80 as normal user
sudo setcap -r ./myserver                          # remove
```

> [!warning] File caps vs setuid
> | | setuid root | file capabilities |
> |---|---|---|
> | Power | Everything root can do | Only listed caps |
> | Visible in `ls -l` | Yes (`s`) | **No** — use `getcap` |
> | Lost when | File is written / chowned | File is written / copied (xattr not copied by plain `cp`) |
> | Ignored on | `nosuid` mounts | `nosuid` mounts |
>
> Security audits must check **both**: `find -perm -4000` and `getcap -r`.

# Capabilities in systemd services

Services drop what they don't need, set in the unit file → [[services]]:
```ini
[Service]
User=myapp
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=yes
```
```bash
systemctl show -p CapabilityBoundingSet,AmbientCapabilities systemd-resolved
# CapabilityBoundingSet=cap_setpcap cap_net_bind_service cap_net_raw
# AmbientCapabilities=cap_setpcap cap_net_bind_service cap_net_raw
```

`NoNewPrivileges=yes` (`NoNewPrivs:` in `/proc/<pid>/status`) makes the kernel ignore setuid bits and file caps for that process and its children.

# Scripting recipes

### Capabilities of every process
```bash
for d in /proc/[0-9]*; do
  eff=$(awk '/^CapEff/ {print $2}' "$d/status" 2>/dev/null) || continue
  [[ $eff == 0000000000000000 || -z $eff ]] && continue
  printf '%-7s %-20s %s\n' "${d#/proc/}" "$(< "$d/comm")" "$(capsh --decode="$eff" | cut -d= -f2)"
done 2>/dev/null
```
(`pscap` from libcap-ng prints the same as a table, if installed.)

### Run a command with fewer capabilities
```bash
sudo capsh --drop=cap_net_raw -- -c 'ping -c1 127.0.0.1'
setpriv --no-new-privs bash                        # setuid/file caps ignored from now on (works as normal user)
sudo setpriv --bounding-set=-all bash              # drop the whole bounding set (needs root)
```

### Check one capability
```bash
has_cap() {   # has_cap <pid> <bit>
  local hex; hex=$(awk '/^CapEff/ {print $2}' "/proc/$1/status")
  (( (16#$hex >> $2) & 1 ))
}
has_cap 1 21 && echo "PID 1 has CAP_SYS_ADMIN"
```

# Find stuff

| Question                              | Command                                     |
| ------------------------------------- | ------------------------------------------- |
| Caps of my shell                      | `capsh --print`, `grep Cap /proc/$$/status` |
| Caps of a process                     | `grep Cap /proc/<pid>/status`, `getpcaps <pid>` |
| Decode a hex mask                     | `capsh --decode=<hex>`                      |
| Binaries with file caps               | `getcap -r / 2>/dev/null`                   |
| Number of caps this kernel knows      | `cat /proc/sys/kernel/cap_last_cap`         |
| Caps of a systemd unit                | `systemctl show -p CapabilityBoundingSet <unit>` |

# See also
- [[permissions]] — where capabilities fit in the access check
- [[special-bits]] — setuid, the older approach
- [[attributes]] — xattrs that store file caps
- [[mac-selinux-apparmor]] — another layer on top
- [[users]] — root and `sudo`
- `man 7 capabilities`, `man 8 getcap`, `man 1 capsh`, `man 1 setpriv`
