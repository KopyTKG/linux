---
tags:
  - linux
  - files
  - etc
  - identity
  - network
aliases:
  - /etc/hostname
  - hostname file
---
`/etc/hostname` holds the **name of this machine**, one word on one line. It is read at boot and copied into the kernel, where every program sees it (prompt, logs, network announcements).

> [!info] Quick facts
> - Path: `/etc/hostname` · mode `0644`
> - Content here: `myhost`
> - Live value in the kernel: `/proc/sys/kernel/hostname` (what `hostname` and `uname -n` print)
> - Change it with `sudo hostnamectl hostname <name>` (writes the file **and** sets the kernel value)
> - Manual: `man 5 hostname`, `man 1 hostnamectl`

# Rules for a good hostname
- Lowercase letters `a-z`, digits, `-`. No spaces, no `_`, no dots (a dot makes it a domain name).
- Max 64 characters; 63 is the DNS limit per label.
- Must not start or end with `-`.

# Three kinds of hostname (systemd)

| Kind       | Stored in                         | Example         | Meaning                                       |
| ---------- | --------------------------------- | --------------- | --------------------------------------------- |
| static     | `/etc/hostname`                   | `myhost`       | The configured one                            |
| transient  | kernel (`/proc/sys/kernel/hostname`) | `myhost`    | Current value, may come from DHCP if no static one |
| pretty     | `/etc/machine-info` (`PRETTY_HOSTNAME=`) | `Kopy's Desktop` | Free text for UIs                       |

```bash
hostnamectl
#  Static hostname: myhost
#        Icon name: computer-desktop
#          Chassis: desktop
```

> [!tip] Also update `/etc/hosts`
> If [[hosts]] has a line like `127.0.1.1 oldname`, change it too. Otherwise `sudo` may wait several seconds on every call trying to resolve the old name.

> [!note] Writing the file alone isn't enough
> `echo newname | sudo tee /etc/hostname` only takes effect after reboot. `hostnamectl` changes both at once. Old way without systemd: `sudo hostname newname` (kernel only, lost at reboot).

# Scripting recipes

### Current hostname (four ways, same answer)
```bash
hostname
uname -n
cat /proc/sys/kernel/hostname
echo "$HOSTNAME"     # bash variable, set when the shell started
```

### Configured vs running (differ = changed without reboot or set by DHCP)
```bash
echo "file=$(< /etc/hostname) kernel=$(< /proc/sys/kernel/hostname)"
```

### Full name with domain
```bash
hostname -f 2>/dev/null || hostname
```

### Short name in a prompt or log prefix
```bash
echo "[$(hostname -s)] backup done"
```

# See also
- [[hosts]] — make the name resolve locally
- [[nsswitch-conf]] — `myhostname` source
- [[os-release]] · [[machine-id]] — other identity files
- [[proc]] · [[networking]] · [[etc]]
