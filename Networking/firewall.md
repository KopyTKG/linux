---
tags:
  - linux
  - networking
  - firewall
  - security
aliases:
  - firewall
  - nftables
  - iptables
  - ufw
  - firewalld
---
A firewall decides which packets are **allowed**, **dropped** or **rejected**. On Linux the filtering itself always happens **in the kernel** (netfilter). The tools you meet — `nft`, `iptables`, `ufw`, `firewalld` — are just different ways to write rules into it.

> [!info] Example setup (a typical desktop)
> | Thing                  | Value                                                   |
> | ---------------------- | ------------------------------------------------------- |
> | Kernel engine          | nftables                                                |
> | Front-end in use       | **ufw** (`ufw.service` active, `ENABLED=yes`)           |
> | `iptables`             | `iptables v1.8.13 (nf_tables)` — the iptables command translated into nftables |
> | nftables.service       | inactive (ufw loads the rules instead)                  |
> | firewalld              | not installed                                           |
> | Default policies (`/etc/default/ufw`) | input **ACCEPT**, output ACCEPT, forward DROP |
> | User rules (`/etc/ufw/user.rules`) | none                                       |
> | Also writing rules     | **Docker** (for container ports)                        |

> [!warning] "Enabled" doesn't mean "filtering"
> In this example, with `DEFAULT_INPUT_POLICY="ACCEPT"` and no rules, ufw is on but lets **everything in**. Anything listening on `0.0.0.0` ([[sockets-ports]]) is reachable from the LAN. To block incoming by default:
> ```bash
> sudo ufw default deny incoming
> sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp   # example exception
> sudo ufw status verbose
> ```

# Layers of tools

```
 ufw  /  firewalld                 ← friendly front-ends (pick ONE)
        │
 iptables (nf_tables backend) / nft ← low-level rule tools
        │
 netfilter in the kernel           ← actually inspects packets
```

| Tool         | What it is                                         | Typical on                   |
| ------------ | -------------------------------------------------- | ---------------------------- |
| `nft`        | Native nftables CLI, modern, one tool for v4+v6    | anything recent              |
| `iptables`   | Classic CLI. Today usually a wrapper on nftables (`iptables -V` shows `nf_tables`) | old scripts, Docker |
| `ufw`        | "Uncomplicated Firewall", simple allow/deny rules  | desktops                     |
| `firewalld`  | Zones (home/public/work), D-Bus API, `firewall-cmd` | workstations, servers       |

> [!warning] One front-end only
> Don't run ufw **and** firewalld (or ufw + your own `nftables.conf`) at the same time — they overwrite each other's rules.

### Detect what's in use
```bash
for s in ufw firewalld nftables iptables; do
  printf "%-10s %s\n" "$s" "$(systemctl is-active "$s" 2>/dev/null)"
done
iptables -V          # shows backend: (nf_tables) or (legacy)
```

# Concepts

### Chains — where in the path a rule applies
```
                 ┌──────────── INPUT ──────► local program
packet in ──► PREROUTING ──┤
                 └─ FORWARD ─► POSTROUTING ──► out (to another host / container)
local program ──► OUTPUT ──► POSTROUTING ──► out
```

| Chain         | Packets that ...                                         |
| ------------- | -------------------------------------------------------- |
| `INPUT`       | arrive **for this machine**                              |
| `OUTPUT`      | this machine sends                                       |
| `FORWARD`     | pass **through** (router, Docker containers)             |
| `PREROUTING`  | just arrived, before routing (used for port forwarding / DNAT) |
| `POSTROUTING` | about to leave (used for NAT / masquerade)               |

### Tables (iptables names)
`filter` (allow/deny), `nat` (address translation), `mangle` (modify packets), `raw`. In nftables you name tables yourself.

### Verdicts / policies

| Verdict   | Effect                                                         |
| --------- | -------------------------------------------------------------- |
| `ACCEPT`  | Let it through                                                 |
| `DROP`    | Silently discard — sender waits until timeout                  |
| `REJECT`  | Discard + tell sender "refused" — fails fast                   |
| policy    | What happens when **no rule matched** (`default deny` = DROP)  |

### Stateful filtering
`ct state established,related accept` — once **you** opened a connection, the replies are allowed back in. That's why "deny incoming" doesn't break browsing.

# Inspecting rules (read-only, but needs root)
Listing rules talks to the kernel's netfilter, which requires root (`Operation not permitted` as a normal user).

```bash
sudo nft list ruleset                 # EVERYTHING, all tables (best overview)
sudo iptables -L -n -v                # filter table, iptables view
sudo iptables -t nat -L -n -v         # NAT rules (Docker port forwards)
sudo ufw status verbose               # ufw rules + defaults
sudo ufw status numbered              # with numbers (to delete)
sudo firewall-cmd --list-all          # firewalld, current zone
```

Without root you can still read the config files:

| File                        | Front-end  | Content                               |
| --------------------------- | ---------- | ------------------------------------- |
| `/etc/default/ufw`          | ufw        | Default policies, IPv6 on/off         |
| `/etc/ufw/ufw.conf`         | ufw        | `ENABLED=yes`, log level              |
| `/etc/ufw/user.rules`       | ufw        | Your `ufw allow/deny` rules (generated) |
| `/etc/ufw/before.rules`     | ufw        | Rules applied before yours (ICMP, DHCP...) |
| `/etc/ufw/applications.d/`  | ufw        | App profiles (`ufw app list`)         |
| `/etc/nftables.conf`        | nftables   | Ruleset loaded by `nftables.service`  |
| `/etc/firewalld/`           | firewalld  | Zones, services                       |

# ufw cheat sheet

| Task                                 | Command                                        |
| ------------------------------------ | ---------------------------------------------- |
| Enable / disable                     | `sudo ufw enable` / `disable`                  |
| Default deny incoming                | `sudo ufw default deny incoming`               |
| Allow port                           | `sudo ufw allow 22/tcp`                        |
| Allow from LAN only                  | `sudo ufw allow from 192.168.1.0/24 to any port 8080` |
| Allow app profile                    | `sudo ufw allow "OpenSSH"`                     |
| Deny                                 | `sudo ufw deny 5432`                           |
| Delete rule                          | `sudo ufw status numbered` → `sudo ufw delete 3` |
| Rate-limit (anti brute force)        | `sudo ufw limit 22/tcp`                        |
| Logging                              | `sudo ufw logging on` → `journalctl -k | grep UFW` |

# nftables example
A minimal "deny incoming, allow SSH" ruleset:
```nft
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    ct state established,related accept
    iif lo accept
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept
    tcp dport 22 accept
  }
  chain forward { type filter hook forward priority 0; policy drop; }
  chain output  { type filter hook output priority 0; policy accept; }
}
```
`inet` = IPv4 + IPv6 together. Test with `sudo nft -c -f file` (check only), load with `sudo nft -f file`.

# Docker and the firewall

> [!danger] Docker bypasses ufw
> When a container publishes a port (`-p 5432:5432`), Docker adds its own NAT + `FORWARD` rules. Traffic to a container goes through **FORWARD**, not INPUT, so `ufw deny 5432` does **not** block it.
> Example: a compose container `myapp-db-1` published as `5432:5432` listens on `0.0.0.0:5432` → PostgreSQL is reachable from the LAN regardless of ufw.
> Fixes:
> - Publish only on localhost: `-p 127.0.0.1:5432:5432` (or `"127.0.0.1:5432:5432"` in compose) — **simplest**.
> - Or add rules to the `DOCKER-USER` chain, which Docker leaves alone.

# Scripting recipes

### Firewall status summary (no root)
```bash
#!/usr/bin/env bash
for s in ufw firewalld nftables; do
  [[ $(systemctl is-active "$s" 2>/dev/null) == active ]] && echo "active front-end: $s"
done
if [[ -r /etc/default/ufw ]]; then
  grep -E '^DEFAULT_(INPUT|OUTPUT|FORWARD)_POLICY' /etc/default/ufw
  n=$(grep -c '^### tuple' /etc/ufw/user.rules 2>/dev/null)
  echo "ufw user rules: ${n:-0}"
fi
```
```
active front-end: ufw
DEFAULT_INPUT_POLICY="ACCEPT"
DEFAULT_OUTPUT_POLICY="ACCEPT"
DEFAULT_FORWARD_POLICY="DROP"
ufw user rules: 0
```

### What's exposed to the network
Listeners on all interfaces = candidates the firewall must handle:
```bash
ss -tulnH | awk '$5 ~ /^(0\.0\.0\.0|\[::\]|\*):/ {print $1, $5}' | sort -u
```

### Test from outside
From **another** machine: `nc -zv 192.168.1.50 5432` → [[tools]]. Testing from the same PC goes through `lo` and doesn't prove anything about the firewall.

### Watch dropped packets
```bash
journalctl -k -f | grep -E 'UFW (BLOCK|AUDIT)'      # needs ufw logging on
```

# Find stuff

| I want ...                   | Command / path                               |
| ---------------------------- | -------------------------------------------- |
| Which firewall is active     | `systemctl is-active ufw firewalld nftables` |
| All kernel rules             | `sudo nft list ruleset`                      |
| ufw rules                    | `sudo ufw status verbose`                    |
| ufw config                   | `/etc/default/ufw`, `/etc/ufw/`              |
| Docker port forwards         | `sudo iptables -t nat -L DOCKER -n`, `docker ps` |
| Firewall log messages        | `journalctl -k | grep -i ufw` → [[journal]]  |
| Is forwarding enabled        | `cat /proc/sys/net/ipv4/ip_forward` → [[proc]] |

# See also
- [[networking]] — overview
- [[sockets-ports]] — what is listening
- [[ip-routing]] — forwarding, NAT
- [[services]] — enabling/disabling firewall services
- [[etc]] — firewall config files
