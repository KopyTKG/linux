---
tags:
  - linux
  - networking
  - index
aliases:
  - Networking
  - network
---
Networking on Linux is a stack of layers. Each layer has its own tool, its own config and its own files in [[proc]], [[sys]] and [[etc]]. When something is "broken", go **bottom-up** through the layers until you find the one that fails.

> [!info] Example machine at a glance
> | Thing             | Value                                                   |
> | ----------------- | ------------------------------------------------------- |
> | Network manager   | **NetworkManager** (`nmcli`) → [[network-manager]]      |
> | DNS               | **systemd-resolved** at `127.0.0.53` → [[dns]]          |
> | Firewall          | **ufw** (on top of nftables) → [[firewall]]             |
> | Main interface    | `enp3s0`, wired, 1000 Mb/s → [[interfaces]]            |
> | IP address        | `192.168.1.50/24` from DHCP → [[ip-routing]]              |
> | Default gateway   | `192.168.1.1` (router, also DNS server)                   |
> | Wi-Fi             | `wlan0`, present but not connected                      |
> | Extra interfaces  | `docker0`, `br-*`, `veth*` — created by Docker          |

# The layers

| # | Layer         | Question it answers                         | Tool                         | Note                    |
| - | ------------- | ------------------------------------------- | ---------------------------- | ----------------------- |
| 1 | Link          | Is the cable / Wi-Fi connected?             | `ip link`, `ethtool`, `nmcli device` | [[interfaces]]  |
| 2 | Address (IP)  | Do I have an IP address?                    | `ip addr`                    | [[interfaces]], [[ip-routing]] |
| 3 | Routing       | Do I know where to send packets?            | `ip route`, `ping`, `tracepath` | [[ip-routing]]       |
| 4 | Name (DNS)    | Can I turn `kernel.org` into an IP?      | `resolvectl`, `getent`, `dig` | [[dns]]                |
| 5 | Ports/sockets | Is the program listening / connected?       | `ss`                         | [[sockets-ports]]       |
| 6 | Filter        | Is a firewall dropping it?                  | `ufw`, `nft`                 | [[firewall]]            |
| 7 | Application   | Does the service answer correctly?          | `curl`, `ssh`, `nc`          | [[tools]]               |

Who configures all of this: [[network-manager]] (NetworkManager here) brings links up, gets IPs via DHCP, sets routes and hands DNS servers to systemd-resolved. Both are [[systemd]] services.

# Troubleshooting checklist (bottom-up)
```bash
ip -br link                       # 1. interface UP and LOWER_UP?
ip -br addr                       # 2. has an IP (not only fe80::)?
ip route show default             # 3. default gateway exists?
ping -c3 192.168.1.1                # 3. gateway reachable? (some routers ignore ping)
ping -c3 1.1.1.1                  # 3. internet reachable by IP?
resolvectl query kernel.org    # 4. DNS works?
curl -I https://kernel.org     # 7. HTTP works?
```

> [!tip] Reading the result
> - Ping by IP works, by name doesn't → **DNS** problem → [[dns]].
> - Nothing past the gateway works → router / ISP, or wrong default route.
> - Your own service is unreachable from another PC → check it listens on `0.0.0.0` not `127.0.0.1` ([[sockets-ports]]), then the [[firewall]].

# Where networking lives in the filesystem

| Path                                   | What                                          |
| -------------------------------------- | --------------------------------------------- |
| `/sys/class/net/<iface>/`              | Live interface info: state, MAC, speed, counters → [[sys]] |
| `/proc/net/dev`                        | Traffic counters per interface → [[proc]]    |
| `/proc/net/tcp`, `udp`, `unix`         | All sockets (what `ss` reads)                 |
| `/proc/sys/net/`                       | Kernel network settings (`ip_forward`, ...)   |
| `/etc/resolv.conf`                     | DNS servers for programs → [[dns]]            |
| `/etc/hosts`                           | Static name → IP list                         |
| `/etc/nsswitch.conf`                   | Order of name lookups                         |
| `/etc/NetworkManager/`                 | NetworkManager config + saved connections     |
| `/etc/ufw/`, `/etc/default/ufw`        | Firewall config → [[firewall]]                |
| `/etc/services`                        | Port number ↔ name list                       |
| `/run/systemd/resolve/`                | systemd-resolved runtime files → [[run]]      |

# Notes in this topic
- [[interfaces]] — network cards, names, MAC, MTU, state, speed
- [[ip-routing]] — IPv4/IPv6, CIDR, routes, gateway, ARP
- [[dns]] — name resolution, systemd-resolved, `/etc/hosts`
- [[network-manager]] — NetworkManager / `nmcli`, profiles, Wi-Fi
- [[sockets-ports]] — TCP vs UDP, ports, `ss`, who listens
- [[firewall]] — ufw, nftables, iptables
- [[tools]] — cheat sheet: `curl`, `ssh`, `rsync`, `nc`, `dig`, ...

# See also
- [[systemd]] — NetworkManager, resolved and Docker are all units
- [[proc]] · [[sys]] · [[etc]] · [[run]]
- `man 8 ip`, `man 8 ss`, `man 5 resolv.conf`, `man 5 nsswitch.conf`
- Kernel networking docs: <https://docs.kernel.org/networking/>
