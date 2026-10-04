---
tags:
  - linux
  - files
  - etc
  - network
aliases:
  - /etc/hosts
  - hosts file
---
`/etc/hosts` is a **static name → IP table**. It is checked before (or after, see [[nsswitch-conf]]) DNS, so you can name machines on your LAN, test a website on a new server, or block a domain, without touching any DNS server.

> [!info] Quick facts
> - Path: `/etc/hosts` · mode `0644`
> - Changes work **immediately**. No restart. (Browsers may cache old answers for a minute)
> - Only used by programs that go through the normal resolver (`getent`, `ping`, `curl`, browsers). `dig`/`nslookup` ask DNS directly and **ignore it**
> - Manual: `man 5 hosts`

# Format
```
IP-address   canonical-name   [aliases...]
```

Example:
```
127.0.0.1  localhost
::1        localhost ip6-localhost ip6-loopback
ff02::1    ip6-allnodes
ff02::2    ip6-allrouters
127.0.1.1  myhost
```

| Line                     | Why it's there                                                         |
| ------------------------ | ---------------------------------------------------------------------- |
| `127.0.0.1 localhost`    | IPv4 loopback. Never remove                                            |
| `::1 localhost`          | IPv6 loopback                                                          |
| `ff02::1`, `ff02::2`     | IPv6 multicast "all nodes" / "all routers"                             |
| `127.0.1.1 <hostname>`   | Lets the machine resolve its own [[hostname]] without network (Debian convention) |

# Typical uses
```
192.168.1.10   nas nas.home         # name a LAN device
203.0.113.50   example.com          # test a site on a new server before DNS switch
0.0.0.0        ads.example.net      # block a domain (connection fails instantly)
```

> [!tip] Block with `0.0.0.0`, not `127.0.0.1`
> `0.0.0.0` fails right away. `127.0.0.1` tries to connect to your own machine, which is slower and may hit a local web server.

> [!warning] Malware likes this file too
> Lines redirecting bank or update domains to strange IPs are a classic attack. If a site misbehaves on one machine only, check `/etc/hosts` first.

# Scripting recipes

### Resolve like a normal program (respects hosts + DNS order)
```bash
getent hosts localhost myhost
# ::1             localhost
# 127.0.1.1       myhost
getent ahostsv4 example.com     # IPv4 only
```

### Name → IP table from the file
```bash
awk '!/^#/ && NF {ip=$1; for (i=2; i<=NF; i++) print $i, ip}' /etc/hosts
```

### Add an entry only if missing
```bash
line='192.168.1.10 nas'
grep -qxF "$line" /etc/hosts || echo "$line" | sudo tee -a /etc/hosts
```

### Does a name come from hosts or from DNS?
```bash
grep -w nas /etc/hosts && echo "from hosts" || echo "not in hosts, DNS"
```

# See also
- [[nsswitch-conf]] — decides if `files` (hosts) or `dns` is asked first
- [[resolv-conf]] · [[dns]] — the DNS side
- [[hostname]] · [[networking]] · [[etc]]
