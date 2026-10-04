---
tags:
  - linux
  - networking
  - routing
aliases:
  - IP addresses
  - routing
  - ip route
  - CIDR
---
An **IP address** identifies your machine on a network. A **route** tells the kernel which interface and which next hop (gateway) to use for a destination. With an address and a default route you can reach the internet.

> [!info] Example machine
> | Thing              | Value                                      |
> | ------------------ | ------------------------------------------ |
> | IPv4               | `192.168.1.50/24` on `enp3s0` (DHCP)         |
> | Network            | `192.168.1.0/24` (254 usable addresses)      |
> | Default gateway    | `192.168.1.1` (the router)                   |
> | IPv6               | only link-local `fe80::.../64` — the network gives **no global IPv6** |
> | Docker networks    | `172.17.0.0/16`, `172.18.0.0/16`            |

# IPv4
32 bits written as 4 numbers `0–255`: `192.168.1.50`.

### CIDR notation `/24`
The number after `/` = how many leading bits are the **network** part. The rest is the **host** part.

| CIDR  | Netmask           | Addresses   | Usable hosts | Typical use              |
| ----- | ----------------- | ----------- | ------------ | ------------------------ |
| `/8`  | `255.0.0.0`       | 16 777 216  | 16 777 214   | `127.0.0.0/8` loopback   |
| `/16` | `255.255.0.0`     | 65 536      | 65 534       | Docker networks          |
| `/24` | `255.255.255.0`   | 256         | 254          | Home LAN (**this one**)  |
| `/30` | `255.255.255.252` | 4           | 2            | Point-to-point link      |
| `/32` | `255.255.255.255` | 1           | 1            | A single host            |

`192.168.1.50/24` → network `192.168.1.0`, broadcast `192.168.1.255`, hosts `192.168.1.1`–`192.168.1.254`.

### Special ranges

| Range                                         | Meaning                                   |
| --------------------------------------------- | ----------------------------------------- |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | **Private** (LAN, not on internet). Router does NAT |
| `127.0.0.0/8`                                 | Loopback, this machine                    |
| `169.254.0.0/16`                              | Link-local: DHCP failed, self-assigned    |
| `100.64.0.0/10`                               | Carrier-grade NAT (also Tailscale)        |
| `0.0.0.0`                                     | "Any" / "all interfaces" when listening   |
| `224.0.0.0/4`                                 | Multicast                                 |

> [!tip] Seeing `169.254.x.x`?
> DHCP failed — the machine gave itself an address. Check the cable / Wi-Fi / router.

# IPv6
128 bits in hex, groups of 4, `::` = run of zeros: `fe80::5054:ff:fe12:3456`.

| Prefix      | Meaning                                                 |
| ----------- | ------------------------------------------------------- |
| `::1/128`   | Loopback                                                |
| `fe80::/10` | **Link-local**, always auto-created, only on this cable |
| `fd00::/8`  | Private (unique local)                                  |
| `2000::/3`  | Global (internet). `2600:3c04:...` = kernel.org      |

Show only internet-capable IPv6: `ip -6 addr show scope global` (empty here).

# `ip route` — the routing table
```bash
ip route          # IPv4
ip -6 route       # IPv6
```
```
default via 192.168.1.1 dev enp3s0 proto dhcp src 192.168.1.50 metric 100
192.168.1.0/24 dev enp3s0 proto kernel scope link src 192.168.1.50 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.18.0.0/16 dev br-1a2b3c4d5e6f proto kernel scope link src 172.18.0.1
```

| Part              | Meaning                                                            |
| ----------------- | ------------------------------------------------------------------ |
| `default`         | Everything not matching another line (= `0.0.0.0/0`)               |
| `via 192.168.1.1`   | Send to this **gateway** (router)                                  |
| `dev enp3s0`      | Out of this interface                                              |
| `proto dhcp`      | Who added it: `dhcp`, `kernel` (auto from IP), `static`            |
| `scope link`      | Destination is directly on the cable, no gateway needed            |
| `src 192.168.1.50`  | Source IP to use                                                   |
| `metric 100`      | Priority. **Lower wins** (Ethernet 100 beats Wi-Fi 600)            |
| `linkdown`        | Interface has no carrier, route is unused                          |

**How the kernel picks:** the **longest prefix** that matches wins. `192.168.1.30` matches `/24` (direct). `1.1.1.1` only matches `default` → goes to the router.

Ask the kernel directly:
```bash
ip route get 1.1.1.1
# 1.1.1.1 via 192.168.1.1 dev enp3s0 src 192.168.1.50 uid 1000
```

# ARP / neighbours
On the local network packets are sent to a **MAC**, not an IP. ARP (IPv4) / NDP (IPv6) finds the MAC for an IP. The kernel caches answers:
```bash
ip neigh
```
```
192.168.1.30 dev enp3s0 lladdr 52:54:00:00:00:30 STALE
192.168.1.1 dev enp3s0 lladdr 52:54:00:00:00:01 REACHABLE
```

| State        | Meaning                                       |
| ------------ | --------------------------------------------- |
| `REACHABLE`  | Confirmed recently                            |
| `STALE`      | Known, not confirmed lately; re-checked on use |
| `FAILED`     | No answer — host is down or not on this LAN   |
| `PERMANENT`  | Manually added                                |

Also in `/proc/net/arp` → [[proc]].

# DHCP
The router hands out IP, gateway and DNS server automatically. Here NetworkManager runs the DHCP client → [[network-manager]].
```bash
nmcli -f IP4 device show enp3s0
# IP4.ADDRESS[1]: 192.168.1.50/24
# IP4.GATEWAY:    192.168.1.1
# IP4.DNS[1]:     192.168.1.1
```

# Testing reachability
```bash
ping -c 4 192.168.1.1           # ICMP echo, Ctrl+C to stop without -c
ping -6 -c 4 ::1              # IPv6
tracepath -n 1.1.1.1          # every router on the way + path MTU
mtr 1.1.1.1                   # live traceroute (separate package, may need installing)
```

> [!note] No ping reply ≠ host down
> Many routers and servers drop ICMP. In this example the gateway `192.168.1.1` doesn't answer ping, but it's clearly up (`ip neigh` says `REACHABLE`, DNS through it works). Test a real port instead: [[sockets-ports]].

# Forwarding (routing for others)
A Linux box only forwards packets between interfaces if this is `1`:
```bash
cat /proc/sys/net/ipv4/ip_forward
```
Docker turns it on for its containers. See [[proc]] → `/proc/sys`.

# Changing routes (root, temporary)
```bash
sudo ip route add 192.168.100.0/24 via 192.168.1.5   # route a subnet via another router
sudo ip route del 192.168.100.0/24
```
Permanent: `nmcli connection modify "Wired connection 1" +ipv4.routes "192.168.100.0/24 192.168.1.5"` → [[network-manager]].

# Scripting recipes

### Default gateway and interface
```bash
read -r _ _ gw _ dev _ < <(ip route show default)
echo "gateway=$gw dev=$dev"
# gateway=192.168.1.1 dev=enp3s0
```
JSON variant: `ip -j route show default | jq -r '.[0].gateway'`

### My LAN IP (the one used to reach the internet)
```bash
ip route get 1.1.1.1 | awk '{for(i=1;i<=NF;i++) if($i=="src") print $(i+1)}'
```
No packet is sent — the kernel only looks up the route.

### CIDR calculator in pure bash
```bash
cidr=192.168.1.50/24
ip=${cidr%/*}; bits=${cidr#*/}
IFS=. read -r a b c d <<< "$ip"
n=$(( (a<<24)|(b<<16)|(c<<8)|d ))
mask=$(( (0xFFFFFFFF << (32-bits)) & 0xFFFFFFFF ))
net=$(( n & mask )); bc=$(( net | (~mask & 0xFFFFFFFF) ))
to_ip() { echo "$(( $1>>24&255 )).$(( $1>>16&255 )).$(( $1>>8&255 )).$(( $1&255 ))"; }
echo "network $(to_ip $net)/$bits  broadcast $(to_ip $bc)  mask $(to_ip $mask)  hosts $(( (1<<(32-bits)) - 2 ))"
# network 192.168.1.0/24  broadcast 192.168.1.255  mask 255.255.255.0  hosts 254
```

### Wait until network is up (for startup scripts)
```bash
for i in {1..30}; do
  ip route show default | grep -q . && { echo "network up"; break; }
  sleep 1
done
```
In systemd units use `After=network-online.target` + `Wants=network-online.target` instead → [[units]].

### Who is on my LAN (from the ARP cache)
```bash
ip neigh | grep -v FAILED
```
Only shows hosts this PC talked to. Full scan needs `nmap -sn 192.168.1.0/24` (not installed).

# Find stuff

| I want ...                     | Command                                     |
| ------------------------------ | ------------------------------------------- |
| My IP                          | `ip -br addr`, `ip -4 -o addr show scope global` |
| Gateway                        | `ip route show default`                     |
| Which route a packet takes     | `ip route get <ip>`                         |
| Path to a host                 | `tracepath -n <host>`                       |
| MAC of a LAN device            | `ip neigh`                                  |
| DHCP lease info                | `nmcli -f DHCP4 device show enp3s0`         |
| Is forwarding on               | `cat /proc/sys/net/ipv4/ip_forward`         |
| Raw routing table              | `/proc/net/route` (hex, little-endian)      |

# See also
- [[networking]] — overview
- [[interfaces]] — where addresses live
- [[dns]] — names → IPs
- [[network-manager]] — permanent config
- [[proc]] — `/proc/net/route`, `/proc/net/arp`, `/proc/sys/net`
