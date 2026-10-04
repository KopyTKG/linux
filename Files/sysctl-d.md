---
tags:
  - linux
  - files
  - etc
  - kernel
aliases:
  - /etc/sysctl.d
  - sysctl.d
  - /etc/sysctl.conf
  - sysctl
---
`/etc/sysctl.d/` holds **kernel settings applied at every boot**. Each setting is a file under `/proc/sys`; this folder makes changes to them permanent. Writing to `/proc/sys` directly only lasts until reboot (see [[proc]]).

> [!info] Quick facts
> - Your files: `/etc/sysctl.d/*.conf` (often empty)
> - Vendor defaults: `/usr/lib/sysctl.d/*.conf` (6 files here)
> - Also read: `/run/sysctl.d/`, `/usr/local/lib/sysctl.d/`, and the old single file `/etc/sysctl.conf` (gone on many systems)
> - Applied at boot by `systemd-sysctl.service`, or now with `sudo sysctl --system`
> - Manual: `man 5 sysctl.d`, `man 8 sysctl`

# Name ↔ path
Dots in the name are slashes in `/proc/sys`:

| sysctl name                    | File                                       |
| ------------------------------ | ------------------------------------------ |
| `vm.swappiness`                | `/proc/sys/vm/swappiness`                  |
| `net.ipv4.ip_forward`          | `/proc/sys/net/ipv4/ip_forward`            |
| `fs.inotify.max_user_watches`  | `/proc/sys/fs/inotify/max_user_watches`    |
| `kernel.pid_max`               | `/proc/sys/kernel/pid_max`                 |

# File format
```
# /etc/sysctl.d/99-local.conf
vm.swappiness = 10
net.ipv4.ip_forward = 1
-kernel.unknown_key = 1     # leading "-": ignore if the key doesn't exist
```

# Order and overriding
1. All files from all folders are sorted **by file name** (not folder).
2. A file in `/etc/sysctl.d/` **replaces** a file with the **same name** in `/usr/lib/sysctl.d/`.
3. When two files set the same key, the **last one read wins** → name yours `99-something.conf`.
4. To disable a vendor file completely: create an empty file with the same name in `/etc/sysctl.d/`, or symlink it to `/dev/null`.

> [!important] Something else may change values after boot
> Example: a vendor file in `/usr/lib/sysctl.d/` sets `vm.swappiness = 100`, yet the live value is **150**, because a udev rule for zram swap sets it when the zram device appears. Lesson: if a value doesn't match your file, search udev rules, tmpfiles and services too:
> ```bash
> grep -rs swappiness /etc /usr/lib/udev /usr/lib/tmpfiles.d 2>/dev/null
> ```

# Settings worth knowing

| Key                               | Example | Meaning                                                    |
| --------------------------------- | ------- | ---------------------------------------------------------- |
| `vm.swappiness`                   | `10`–`200` | How eagerly to swap. Low = keep apps in RAM             |
| `vm.dirty_ratio`, `vm.dirty_bytes`| `268435456` | How much unwritten data may wait in RAM               |
| `vm.max_map_count`                | `1048576` | Memory maps per process (some games / Elasticsearch need more) |
| `fs.inotify.max_user_watches`     | `524288` | Files one user can watch (IDEs, Syncthing, Dropbox)       |
| `fs.file-max`                     | `2097152` | Max open files system-wide                               |
| `kernel.pid_max`                  | `4194304` | Highest PID                                              |
| `kernel.sysrq`                    | `1`, `16` | Which Magic SysRq key combos are allowed                 |
| `kernel.kptr_restrict`            | `2`     | Hide kernel addresses (security)                           |
| `kernel.dmesg_restrict`           | `1`     | Only root can read `dmesg` → [[dmesg-kmsg]]                |
| `net.ipv4.ip_forward`             | `1`     | Act as router (needed for VMs/containers with NAT)         |
| `net.ipv4.tcp_keepalive_time`     | `120`   | Seconds before TCP keepalive probes                        |
| `net.ipv4.icmp_echo_ignore_all`   | `1`     | Don't answer ping                                          |

# Scripting recipes

### Read one value
```bash
sysctl -n vm.swappiness       # 150
cat /proc/sys/vm/swappiness   # same
```

### Set until reboot
```bash
sudo sysctl -w vm.swappiness=10
```

### Make it permanent
```bash
echo 'vm.swappiness = 10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system           # re-apply all files now
```

### Show all files merged in the order they apply
```bash
systemd-analyze cat-config sysctl.d
```

### Values in config files that differ from the live kernel
```bash
systemd-analyze cat-config sysctl.d 2>/dev/null |
awk -F' *= *' '/^[a-z]/ {print $1, $2}' |
while read -r k v; do
  [[ $k == *'*'* ]] && continue                     # skip glob keys like net.ipv4.conf.*.rp_filter
  live=$(sysctl -n "$k" 2>/dev/null | tr -s '\t' ' ')
  [[ -z $live || $live == "$v" ]] || echo "$k file=$v live=$live"
done
# net.ipv4.conf.default.rp_filter file=2 live=1
# vm.swappiness file=100 live=150
```

### Search keys by name
```bash
sysctl -a 2>/dev/null | grep inotify
```

# See also
- [[proc]] — `/proc/sys`, where the values live
- [[proc-self-highlights]] · [[dmesg-kmsg]]
- [[etc]] · [[usr]] · [[systemd]] · [[networking]]
