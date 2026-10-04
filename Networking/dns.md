---
tags:
  - linux
  - networking
  - dns
aliases:
  - DNS
  - name resolution
  - resolv.conf
  - systemd-resolved
---
**DNS** turns names (`kernel.org`) into IP addresses (`172.105.4.254`). On Linux a lookup doesn't go straight to a DNS server — it goes through a chain: `/etc/nsswitch.conf` decides the order, `/etc/hosts` can answer first, and a local resolver (here **systemd-resolved**) caches answers and asks the real DNS server.

> [!info] DNS on an example machine
> | Thing                  | Value                                                      |
> | ---------------------- | ---------------------------------------------------------- |
> | Resolver               | **systemd-resolved** (stub at `127.0.0.53`)                |
> | `/etc/resolv.conf`     | `nameserver 127.0.0.53` (plain file, mode `foreign`)       |
> | Upstream DNS server    | `192.168.1.1` (router, from DHCP via NetworkManager)         |
> | Fallback servers       | Quad9 `9.9.9.9`, Cloudflare `1.1.1.1`, Google `8.8.8.8`    |
> | Also enabled           | LLMNR + mDNS (`<hostname>.local` resolves on the LAN; hostname here is `myhost`)         |
> | NetworkManager setting | `dns=systemd-resolved` in `/usr/lib/NetworkManager/conf.d/dns.conf` |

# How a lookup flows
```
program (curl, firefox) ──► glibc getaddrinfo()
        │
        ▼
/etc/nsswitch.conf   hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns
        │
        ├─ mymachines  → systemd containers (machinectl)
        ├─ resolve     → systemd-resolved (D-Bus / varlink)  ◄── answers almost everything here
        │                     │
        │                     ├─ /etc/hosts  (resolved reads it too)
        │                     ├─ cache
        │                     ├─ mDNS/LLMNR for *.local and bare LAN names
        │                     └─ DNS server 192.168.1.1  ──► internet
        ├─ files       → /etc/hosts       (only if resolved is not running)
        ├─ myhostname  → own hostname always resolves
        └─ dns         → servers in /etc/resolv.conf directly
```

> [!note] `[!UNAVAIL=return]`
> "If resolved is running, trust its answer (even 'not found') and stop." Only when systemd-resolved is **unavailable** does lookup fall through to `files` and `dns`.

> [!warning] Not every program uses nsswitch
> `dig`, `host`, `nslookup` talk **DNS protocol directly** to the server in `/etc/resolv.conf` (`127.0.0.53`). They **ignore `/etc/hosts`** and nsswitch.
> To test what normal programs see, use `getent hosts <name>` or `resolvectl query <name>`.

# `/etc/resolv.conf`
```
nameserver 127.0.0.53
options edns0 trust-ad
```

| Line         | Meaning                                                   |
| ------------ | --------------------------------------------------------- |
| `nameserver` | DNS server IP (max 3). `127.0.0.53` = systemd-resolved stub |
| `search`     | Domains appended to short names (`search lan` → `nas` tries `nas.lan`) |
| `options`    | `edns0` bigger packets, `timeout:2`, `attempts:3`, ...    |

systemd-resolved can manage this file in different modes. `resolvectl status` shows `resolv.conf mode`:

| Mode       | `/etc/resolv.conf` is ...                                         |
| ---------- | ----------------------------------------------------------------- |
| `stub`     | symlink → `/run/systemd/resolve/stub-resolv.conf` (recommended)   |
| `uplink`   | symlink → `/run/systemd/resolve/resolv.conf` (real servers listed) |
| `static`   | symlink → `/usr/lib/systemd/resolv.conf`                          |
| `foreign`  | a normal file someone else wrote (e.g. edited by hand). Still points to `127.0.0.53`, so it works |

`/run/systemd/resolve/resolv.conf` here contains the real upstream: `nameserver 192.168.1.1`. See [[run]].

# `/etc/hosts`
Static list, checked before DNS. Format: `IP  name [aliases...]`.
```
127.0.0.1  localhost
::1        localhost ip6-localhost ip6-loopback
127.0.1.1  myhost
```
Use it to override a name or name LAN devices:
```
192.168.1.30   nas nas.lan
0.0.0.0      ads.example.com      # block a domain
```
Change takes effect immediately (no restart). → [[etc]]

# systemd-resolved & `resolvectl`
A [[systemd]] service (`systemd-resolved.service`) that caches DNS, handles per-interface DNS servers (VPN uses its own DNS, LAN uses router), mDNS and DNSSEC.

```bash
resolvectl status                  # global + per-link settings
resolvectl dns                     # DNS servers per interface
resolvectl query kernel.org     # resolve, shows which link answered + time
resolvectl statistics              # cache hits/misses
resolvectl flush-caches            # clear cache (after changing DNS records)
resolvectl monitor                 # live: watch every lookup (needs root)
```
```
kernel.org: 172.105.4.254                                -- link: enp3s0
               2600:3c04:e001:324:0:1991:8:25                          -- link: enp3s0
-- Information acquired via protocol DNS in 28.1ms.
```

Statistics here: `Cache Hits: 723`, `Cache Misses: 783` — about half the lookups never left the PC.

| Config                                    | Purpose                                        |
| ----------------------------------------- | ---------------------------------------------- |
| `/etc/systemd/resolved.conf`              | Main config (all commented out = defaults)     |
| `/etc/systemd/resolved.conf.d/*.conf`     | Your overrides (doesn't exist yet)             |
| `/run/systemd/resolve/`                   | Runtime: generated resolv.conf files, sockets  |

Example — use Quad9 with DNS-over-TLS for everything:
```ini
# /etc/systemd/resolved.conf.d/dns.conf
[Resolve]
DNS=9.9.9.9#dns.quad9.net
DNSOverTLS=yes
Domains=~.
```
then `sudo systemctl restart systemd-resolved`.

# Record types

| Type    | Holds                         | Query                              |
| ------- | ----------------------------- | ---------------------------------- |
| `A`     | IPv4 address                  | `dig +short kernel.org A`       |
| `AAAA`  | IPv6 address                  | `host -t AAAA kernel.org`       |
| `CNAME` | Alias to another name         | `dig +short www.github.com CNAME`  |
| `MX`    | Mail server                   | `dig +short kernel.org MX` → `10 smtp1.kernel.org.` |
| `TXT`   | Text (SPF, verification)      | `dig +short kernel.org TXT`     |
| `NS`    | Authoritative name servers    | `dig +short kernel.org NS`      |
| `PTR`   | Reverse: IP → name            | `dig -x 1.1.1.1 +short`            |

# Lookup tools

| Tool                     | Uses nsswitch + `/etc/hosts`? | Good for                          |
| ------------------------ | ----------------------------- | --------------------------------- |
| `getent hosts <name>`    | yes                           | "What will my programs get?"      |
| `getent ahosts <name>`   | yes                           | All addresses, v4 + v6            |
| `resolvectl query`       | resolved only                 | Which link answered, timing       |
| `dig`                    | no                            | Full DNS detail, any record type  |
| `host`, `nslookup`       | no                            | Quick one-liners                  |
| `drill`                  | no                            | `dig` alternative (not installed) |

```bash
dig kernel.org                    # full answer
dig +short kernel.org             # IP only
dig @1.1.1.1 kernel.org           # ask a specific server
dig +trace kernel.org             # follow from root servers
```

# mDNS / `.local`
Avahi and systemd-resolved both do mDNS here. `myhost.local` resolves from other PCs on the LAN without any DNS server. Printers and Chromecasts use this.

# Troubleshooting

| Symptom                                    | Check                                                         |
| ------------------------------------------ | ------------------------------------------------------------- |
| `ping 1.1.1.1` works, `ping kernel.org` doesn't | DNS. `resolvectl status`, is there a DNS server on `enp3s0`? |
| `dig` works, browser doesn't               | Browser uses DNS-over-HTTPS, or `/etc/hosts` override         |
| `getent` and `dig` disagree                | `/etc/hosts` entry or mDNS answer                             |
| Old IP after changing a record             | `resolvectl flush-caches`                                     |
| "Temporary failure in name resolution"     | resolved down: `systemctl status systemd-resolved` → [[services]] |

# Scripting recipes

### Resolve a name like programs do
```bash
resolve() { getent ahosts "$1" | awk '{print $1}' | sort -u; }
resolve kernel.org
# 172.105.4.254
# 2600:3c04:e001:324:0:1991:8:25
```

### Check a name resolves (exit code)
```bash
if getent hosts kernel.org > /dev/null; then echo "DNS ok"; else echo "DNS broken"; fi
```

### Which DNS server is really used
```bash
resolvectl dns enp3s0                                  # Link 2 (enp3s0): 192.168.1.1
grep ^nameserver /run/systemd/resolve/resolv.conf      # same, from file
```

### Is a name overridden in /etc/hosts?
```bash
grep -w "$(< /etc/hostname)" /etc/hosts    # own hostname listed?
```

### Measure lookup time
```bash
dig kernel.org | awk '/Query time/ {print $4, $5}'   # 20 msec (first), 0 msec (cached)
```

# Find stuff

| I want ...                    | Where / command                                  |
| ----------------------------- | ------------------------------------------------ |
| Lookup order                  | `/etc/nsswitch.conf` → `hosts:` line             |
| Static names                  | `/etc/hosts`                                     |
| Hostname                      | `/etc/hostname`, `hostnamectl`                   |
| Current DNS servers           | `resolvectl dns`                                 |
| resolved config               | `/etc/systemd/resolved.conf`, `resolvectl status` |
| NetworkManager's DNS          | `nmcli -g IP4.DNS device show enp3s0`            |
| resolved logs                 | `journalctl -u systemd-resolved` → [[journal]]   |

# See also
- [[networking]] — overview
- [[network-manager]] — where DNS servers come from (DHCP)
- [[etc]] — `/etc/hosts`, `/etc/nsswitch.conf`, `/etc/resolv.conf`
- [[run]] — `/run/systemd/resolve`
- [[systemd]] · [[services]]
