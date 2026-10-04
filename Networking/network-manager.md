---
tags:
  - linux
  - networking
  - networkmanager
aliases:
  - NetworkManager
  - nmcli
  - systemd-networkd
  - iwd
---
The kernel can do networking on its own, but *something* has to bring interfaces up, run DHCP, connect to Wi-Fi, set routes and hand DNS servers to the resolver. That's the **network manager** daemon. Linux systems use one of a few common ones — find out which one yours runs before editing any config, because they don't share config files.

> [!info] Example machine
> | Daemon              | Status                      | Role                                     |
> | ------------------- | --------------------------- | ---------------------------------------- |
> | `NetworkManager`    | **active, enabled**         | Manages everything                       |
> | `wpa_supplicant`    | active (started by NM)      | Wi-Fi authentication backend for NM      |
> | `systemd-resolved`  | active                      | DNS; NM hands it the servers → [[dns]]   |
> | `systemd-networkd`  | inactive                    | not used                                 |
> | `iwd`               | inactive                    | not used (`iwctl` exists but isn't running) |

# Which manager is running?
```bash
for s in NetworkManager systemd-networkd iwd wpa_supplicant connman; do
  printf "%-18s %s\n" "$s" "$(systemctl is-active "$s" 2>/dev/null)"
done
```
```
NetworkManager     active
systemd-networkd   inactive
iwd                inactive
wpa_supplicant     active
connman            inactive
```

> [!warning] Run only one manager
> Two managers fighting over the same interface = random disconnects, duplicate DHCP, wrong DNS. If you switch, disable the old one first: `sudo systemctl disable --now <old>` → [[services]].

# The options

| Manager              | CLI           | Best for                          | Config                                    |
| -------------------- | ------------- | --------------------------------- | ----------------------------------------- |
| **NetworkManager**   | `nmcli`, `nmtui` | Desktops, laptops, Wi-Fi, VPN, GUI applets | `/etc/NetworkManager/`          |
| **systemd-networkd** | `networkctl`  | Servers, containers, static setups | `/etc/systemd/network/*.network`         |
| **iwd**              | `iwctl`       | Wi-Fi only (can also do DHCP); can replace wpa_supplicant as NM backend | `/etc/iwd/main.conf`, `/var/lib/iwd/` |
| **wpa_supplicant**   | `wpa_cli`     | Wi-Fi auth only, usually driven by NM | `/etc/wpa_supplicant/`               |
| **ConnMan**          | `connmanctl`  | Embedded / small devices          | `/var/lib/connman/`                       |
| none (manual)        | `ip`          | Rescue, testing                   | nothing is saved                          |

DHCP client: NetworkManager and systemd-networkd have one built in. Standalone clients (`dhcpcd`, `dhclient`) exist for manual setups.

# NetworkManager

### Concepts
| Term           | Meaning                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| **Device**     | A network interface NM sees (`enp3s0`, `wlan0`) → [[interfaces]]         |
| **Connection** | A saved *profile*: settings (DHCP or static, DNS, Wi-Fi password...) that can be applied to a device |
| **Active connection** | A profile currently applied to a device                           |

One device can have many profiles (e.g. "Home Wi-Fi", "Work Wi-Fi"), only one active at a time.

### State
```bash
nmcli general
nmcli device
```
```
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN     METERED
connected  full          enabled  enabled  missing  enabled  no (guessed)

DEVICE           TYPE      STATE                   CONNECTION
enp3s0           ethernet  connected               Wired connection 1
br-1a2b3c4d5e6f  bridge    connected (externally)  br-1a2b3c4d5e6f
lo               loopback  connected (externally)  lo
docker0          bridge    connected (externally)  docker0
wlan0            wifi      disconnected            --
```

| Device state              | Meaning                                                  |
| ------------------------- | -------------------------------------------------------- |
| `connected`               | NM manages it and a profile is active                    |
| `connected (externally)`  | Someone else (Docker, `ip`) configured it; NM just watches |
| `disconnected`            | Managed, nothing active                                  |
| `unavailable`             | No cable / radio off                                     |
| `unmanaged`               | NM ignores it                                            |

`CONNECTIVITY full` = NM checked a test URL and reached the internet. `limited` = local network only, `portal` = captive portal (hotel Wi-Fi login page).

### Connection profiles
```bash
nmcli connection show
nmcli -f NAME,TYPE,DEVICE,FILENAME connection show
nmcli connection show "Wired connection 1"     # every setting
```
```
NAME                TYPE      DEVICE   FILENAME
Wired connection 1  ethernet  enp3s0   /run/NetworkManager/system-connections/Wired connection 1.nmconnection
```

Key settings of `Wired connection 1` here:
```
connection.interface-name:   enp3s0
connection.autoconnect:      yes
ipv4.method:                 auto      ← DHCP
ipv4.dns:                    --        ← use DNS from DHCP
ipv6.method:                 auto      ← SLAAC / DHCPv6
```

> [!note] "Wired connection 1" lives in `/run`
> It's an **auto-generated** default profile kept only in memory ([[run]]). NM recreates it each boot. Once you modify it, NM saves it to `/etc/NetworkManager/system-connections/`.

### Where NetworkManager keeps things

| Path                                           | What                                                      |
| ---------------------------------------------- | --------------------------------------------------------- |
| `/etc/NetworkManager/NetworkManager.conf`      | Main config                                               |
| `/etc/NetworkManager/conf.d/*.conf`            | Your config snippets                                      |
| `/usr/lib/NetworkManager/conf.d/*.conf`        | Defaults shipped with the software (here: `dns=systemd-resolved`) |
| `/etc/NetworkManager/system-connections/`      | **Saved profiles** (`.nmconnection`, INI format). Root-only (`drwx------`) because they contain Wi-Fi passwords |
| `/run/NetworkManager/system-connections/`      | In-memory profiles                                        |
| `/etc/NetworkManager/dispatcher.d/`            | Scripts run on events (up/down) — see recipes             |
| `/var/lib/NetworkManager/`                     | State: DHCP leases, last used                             |

Follows the usual rule: `/etc` overrides `/usr/lib` → [[etc]].

### Common commands

| Task                                    | Command                                                         |
| --------------------------------------- | --------------------------------------------------------------- |
| List Wi-Fi networks                     | `nmcli device wifi list`                                        |
| Connect to Wi-Fi                        | `nmcli device wifi connect "SSID" --ask`                        |
| Bring profile up / down                 | `nmcli connection up "Wired connection 1"` / `down`             |
| Disconnect a device                     | `nmcli device disconnect wlan0`                                 |
| Wi-Fi on/off                            | `nmcli radio wifi off` / `on`                                   |
| Reload config files                     | `sudo nmcli connection reload`                                  |
| Text UI                                 | `nmtui`                                                         |
| Show DNS / IP of a device               | `nmcli -f IP4 device show enp3s0`                               |
| Logs                                    | `journalctl -u NetworkManager` → [[journal]]                    |

### Static IP
```bash
nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.60/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "1.1.1.1 9.9.9.9"
nmcli connection up "Wired connection 1"       # apply
```
Back to DHCP: `nmcli connection modify "Wired connection 1" ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""`.

> [!warning] Over SSH
> Changing the IP of the interface you're connected through drops your session. Do it from a local console.

### Override DNS but keep DHCP IP
```bash
nmcli connection modify "Wired connection 1" ipv4.ignore-auto-dns yes ipv4.dns "1.1.1.1"
nmcli connection up "Wired connection 1"
```

### Ignore an interface
```ini
# /etc/NetworkManager/conf.d/unmanaged.conf
[keyfile]
unmanaged-devices=interface-name:docker0;interface-name:veth*
```

# systemd-networkd (alternative)
Config files in `/etc/systemd/network/`, matched by interface name:
```ini
# /etc/systemd/network/20-wired.network
[Match]
Name=enp3s0

[Network]
DHCP=yes
# or static:
# Address=192.168.1.60/24
# Gateway=192.168.1.1
# DNS=1.1.1.1
```
```bash
networkctl                       # list links + who manages them
networkctl status enp3s0
sudo networkctl reload           # apply changed files
```
Pairs with systemd-resolved for DNS and `iwd` for Wi-Fi.

# iwd (alternative Wi-Fi)
```bash
iwctl station wlan0 scan
iwctl station wlan0 get-networks
iwctl station wlan0 connect "SSID"
```
Can be NetworkManager's Wi-Fi backend instead of wpa_supplicant:
```ini
# /etc/NetworkManager/conf.d/wifi_backend.conf
[device]
wifi.backend=iwd
```

# Scripting recipes

### Which devices are connected
```bash
nmcli -t -f DEVICE,STATE device | awk -F: '$2=="connected"{print $1}'
# enp3s0
```
`-t` = terse (`:` separated), `-f` = choose fields, `-g` = values only. Use them in scripts, never parse the pretty table.

### Do we have internet (according to NM)?
```bash
[[ $(nmcli -g CONNECTIVITY general) == full ]] && echo "online"
```

### DNS server from DHCP
```bash
nmcli -g IP4.DNS device show enp3s0      # 192.168.1.1
```

### Wait for network in a script
```bash
nm-online -q -t 30 && echo "network ready"
```
For [[systemd]] units: `After=network-online.target` + `Wants=network-online.target` → [[units]].

### Dispatcher script (run something when a link goes up)
```bash
#!/usr/bin/env bash
# /etc/NetworkManager/dispatcher.d/50-log-up   (owner root, chmod 755)
iface=$1 action=$2
if [[ $action == up ]]; then
  logger "network: $iface is up ($CONNECTION_ID)"
fi
```
Arguments: `$1` interface, `$2` action (`up`, `down`, `dhcp4-change`, `connectivity-change`, ...). Env vars like `CONNECTION_ID`, `IP4_ADDRESS_0`. Needs `NetworkManager-dispatcher.service`.

# Find stuff

| I want ...                     | Command / path                                  |
| ------------------------------ | ----------------------------------------------- |
| Which manager runs             | `systemctl is-active NetworkManager systemd-networkd iwd` |
| All saved profiles             | `nmcli connection show`                         |
| Profile file of a connection   | `nmcli -f NAME,FILENAME connection show`        |
| Wi-Fi password of a saved network | `sudo nmcli -s -g 802-11-wireless-security.psk connection show "SSID"` (root, shows secret) |
| NM main config                 | `/etc/NetworkManager/NetworkManager.conf`, `NetworkManager --print-config` |
| networkd config                | `/etc/systemd/network/`                         |
| Logs                           | `journalctl -u NetworkManager -b`               |

# See also
- [[networking]] — overview
- [[interfaces]] · [[ip-routing]] · [[dns]]
- [[systemd]] · [[services]] · [[units]] — all of these managers are systemd services
- [[etc]] · [[run]]
