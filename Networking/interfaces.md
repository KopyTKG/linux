---
tags:
  - linux
  - networking
  - interfaces
aliases:
  - network interfaces
  - NIC
  - ip link
---
An **interface** is anything the kernel can send packets through: a real network card, Wi-Fi, the loopback `lo`, or a virtual device (bridges, VPN tunnels, Docker `veth` pairs). Every interface has a name, a MAC address, an MTU and a state. IP addresses are attached **to** interfaces → [[ip-routing]].

> [!info] Interfaces on an example machine
> | Name              | Kind                     | State   | MTU   | Speed     | IPv4             |
> | ----------------- | ------------------------ | ------- | ----- | --------- | ---------------- |
> | `lo`              | Loopback                 | unknown | 65536 | –         | `127.0.0.1/8`    |
> | `enp3s0`          | Wired Ethernet (real)    | up      | 1500  | 1000 Mb/s | `192.168.1.50/24`  |
> | `wlan0`           | Wi-Fi (real)             | down    | 1500  | –         | –                |
> | `docker0`         | Docker default bridge    | down    | 1500  | –         | `172.17.0.1/16`  |
> | `br-1a2b3c4d5e6f` | Docker compose network   | up      | 1500  | –         | `172.18.0.1/16`  |
> | `veth1234567`     | Cable to a container     | up      | 1500  | –         | –                |
>
> Hardware (`lspci`): `enp3s0` = **Intel I219-V** (driver `e1000e`), `wlan0` = **Intel AX210** (driver `iwlwifi`).

# Interface names

Modern Linux uses **predictable names** (systemd/udev), based on where the hardware sits, so the name doesn't change after reboot.

| Prefix  | Meaning                     | Example → decoded                                  |
| ------- | --------------------------- | -------------------------------------------------- |
| `en`    | Ethernet                    | `enp3s0` = Ethernet, PCI **bus 7**, **slot 0**     |
| `wl`    | Wireless LAN                | `wlp6s0`; `wlan0` here = old style name (from iwd/driver) |
| `ww`    | Mobile broadband (WWAN)     |                                                    |
| `lo`    | Loopback (`127.0.0.1`)      | always there, traffic never leaves the PC          |
| `eth0`  | Old kernel naming           | still used in VMs and containers                   |
| `br-*`, `docker0` | Bridges           | virtual switch                                     |
| `veth*` | Virtual Ethernet pair       | one end in container, other end on bridge          |
| `wg*`, `tun*`, `tap*` | VPN tunnels   |                                                    |

`altname enx04421aed7a0b` in `ip addr` output is an alternative name based on the MAC.

# `ip link` — the link layer
```bash
ip link              # all interfaces, full
ip -br link          # brief, one line each
ip -br -c link       # with colors
```
```
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp3s0           UP             52:54:00:12:34:56 <BROADCAST,MULTICAST,UP,LOWER_UP>
wlan0            DOWN           7a:12:34:56:78:9a <NO-CARRIER,BROADCAST,MULTICAST,UP>
```

| Flag          | Meaning                                                           |
| ------------- | ----------------------------------------------------------------- |
| `UP`          | Interface is **enabled** by admin (`ip link set X up`)            |
| `LOWER_UP`    | Physical link exists (cable plugged / Wi-Fi associated)           |
| `NO-CARRIER`  | Enabled but no link (cable out, Wi-Fi not connected)              |
| `BROADCAST`   | Can broadcast (normal Ethernet)                                   |
| `MULTICAST`   | Supports multicast                                                |
| `LOOPBACK`    | It's `lo`                                                         |

> [!tip] `UP` vs `LOWER_UP`
> `wlan0` above is `UP` but `NO-CARRIER`: turned on, not connected to any network.
> You need **both** `UP` and `LOWER_UP` for traffic.

# `ip addr` — addresses on interfaces
```bash
ip -br addr
ip addr show enp3s0
```
```
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.50/24 brd 192.168.1.255 scope global dynamic noprefixroute enp3s0
       valid_lft 1482sec preferred_lft 1482sec
    inet6 fe80::5054:ff:fe12:3456/64 scope link noprefixroute
```

| Part                 | Meaning                                                         |
| -------------------- | --------------------------------------------------------------- |
| `2:`                 | Interface index (same as `ifindex` in [[sys]])                  |
| `mtu 1500`           | Max packet size in bytes                                        |
| `qdisc mq`           | Queueing discipline (traffic scheduler)                         |
| `link/ether`         | MAC address                                                     |
| `inet 192.168.1.50/24` | IPv4 address + prefix → [[ip-routing]]                          |
| `brd`                | Broadcast address                                               |
| `scope global`       | Usable everywhere. `link` = only on this cable (`fe80::`)       |
| `dynamic`            | From DHCP, has a lease                                          |
| `valid_lft 1482sec`  | DHCP lease time left. NetworkManager renews it before it ends   |
| `inet6 fe80::...`    | IPv6 link-local, always auto-created, not routable              |

# MAC address
48-bit hardware address, `52:54:00:12:34:56`. First 3 bytes = vendor (OUI).
Only matters **inside your local network** — routers don't forward it.

> [!note] Random MACs
> `wlan0` has `7a:12:...`. Second hex digit `a` (binary `...10`) means **locally administered** = randomised by NetworkManager for privacy while scanning Wi-Fi.

# MTU
Maximum Transmission Unit: biggest packet the link carries.

| Value   | Where                                      |
| ------- | ------------------------------------------ |
| `1500`  | Standard Ethernet / Wi-Fi                  |
| `65536` | Loopback                                   |
| `1420`  | Typical WireGuard VPN (overhead)           |
| `9000`  | "Jumbo frames" in data centres             |

Wrong MTU symptom: small things work (ping, DNS), big downloads / HTTPS hang.

# `/sys/class/net/` — interfaces as files
Every interface is a folder in [[sys]]. One value per file, perfect for scripts.

| File                       | Example (`enp3s0`)    | Meaning                                |
| -------------------------- | --------------------- | -------------------------------------- |
| `operstate`                | `up`                  | `up`, `down`, `unknown`, `dormant`     |
| `carrier`                  | `1`                   | Cable/link present (error if iface down) |
| `address`                  | `52:54:00:12:34:56`   | MAC                                    |
| `mtu`                      | `1500`                | MTU                                    |
| `speed`                    | `1000`                | Mb/s (`-1` or error = unknown/down)    |
| `duplex`                   | `full`                |                                        |
| `type`                     | `1`                   | `1` = Ethernet, `772` = loopback       |
| `ifindex`                  | `2`                   | Index number                           |
| `device`                   | symlink               | Exists **only for real hardware**      |
| `wireless/`                | –                     | Exists only for Wi-Fi                  |
| `statistics/rx_bytes`      | `3280328554`          | Bytes received since boot              |
| `statistics/tx_bytes`      |                       | Bytes sent                             |
| `statistics/rx_dropped`    |                       | Dropped packets                        |

Same counters as `/proc/net/dev` → [[proc]].

# `ethtool` — card details
```bash
ethtool enp3s0
```
```
	Speed: 1000Mb/s
	Duplex: Full
	Port: Twisted Pair
	Link detected: yes
```
`ethtool -i enp3s0` shows the driver, `ethtool -S enp3s0` driver statistics.
Wi-Fi details: `iw dev`, `iw dev wlan0 link`.

# Changing interfaces (needs root, temporary)
```bash
sudo ip link set enp3s0 down          # disable
sudo ip link set enp3s0 up            # enable
sudo ip link set enp3s0 mtu 1400      # change MTU
sudo ip addr add 192.168.50.2/24 dev enp3s0   # add second IP
```

> [!warning] Changes made with `ip` are lost
> NetworkManager owns `enp3s0`. Anything you set with `ip` is lost on reboot and may be overwritten when NetworkManager reconnects. Make permanent changes with `nmcli` → [[network-manager]].

# Scripting recipes

### Interface table
```bash
#!/usr/bin/env bash
printf "%-18s %-8s %-6s %-7s %s\n" IFACE STATE MTU SPEED MAC
for n in /sys/class/net/*; do
  i=${n##*/}
  speed=$(cat "$n/speed" 2>/dev/null) || speed=-
  [[ $speed == -1 || -z $speed ]] && speed=-
  printf "%-18s %-8s %-6s %-7s %s\n" "$i" "$(<"$n/operstate")" "$(<"$n/mtu")" "$speed" "$(<"$n/address")"
done
```

### Only physical cards (skip lo, docker, bridges)
```bash
for n in /sys/class/net/*; do [[ -e $n/device ]] && echo "${n##*/}"; done
# enp3s0
# wlan0
```

### Is the cable connected?
```bash
[[ $(< /sys/class/net/enp3s0/carrier) == 1 ]] && echo "cable connected"
```

### Total traffic since boot
```bash
rx=$(< /sys/class/net/enp3s0/statistics/rx_bytes)
tx=$(< /sys/class/net/enp3s0/statistics/tx_bytes)
echo "RX $(( rx/1024/1024 )) MiB  TX $(( tx/1024/1024 )) MiB"
```
Live speed (read twice) → see network speed recipe in [[proc]].

### Machine-readable output
Most `ip` commands take `-j` (JSON):
```bash
ip -j addr show enp3s0 | jq -r '.[0].addr_info[] | select(.family=="inet") | .local'
```

# Find stuff

| I want ...                         | Command                                      |
| ---------------------------------- | -------------------------------------------- |
| All interfaces                     | `ip -br link`, `ls /sys/class/net`           |
| IPs                                | `ip -br addr`                                |
| MAC of a card                      | `cat /sys/class/net/enp3s0/address`          |
| Link speed                         | `ethtool enp3s0`, `cat /sys/class/net/enp3s0/speed` |
| Driver                             | `ethtool -i enp3s0`, `readlink /sys/class/net/enp3s0/device/driver` |
| PCI card model                     | `lspci | grep -i -E 'ethernet|network'`      |
| Wi-Fi info                         | `iw dev`, `nmcli device wifi`                |
| Which manager controls it          | `nmcli device` → [[network-manager]]         |
| Kernel log about link up/down      | `journalctl -k | grep enp3s0` → [[journal]]  |

# See also
- [[networking]] — overview
- [[ip-routing]] — what the addresses mean
- [[network-manager]] — making changes permanent
- [[sys]] — `/sys/class/net`
- [[proc]] — `/proc/net/dev`
- [[dev]] — network cards are **not** in `/dev` (no device file), unlike disks
