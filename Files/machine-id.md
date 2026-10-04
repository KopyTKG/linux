---
tags:
  - linux
  - files
  - etc
  - identity
aliases:
  - /etc/machine-id
  - machine-id
---
`/etc/machine-id` is a **unique random ID of this installation**: 32 hex characters, created on first boot. systemd and many programs use it as "who is this machine", for example the journal folder name, DHCP client identity and D-Bus.

> [!info] Quick facts
> - Path: `/etc/machine-id` · mode `0444` · 33 bytes (32 hex chars + newline)
> - Also linked as `/var/lib/dbus/machine-id` on many systems
> - Created by `systemd-machine-id-setup` or `systemd-firstboot`
> - Survives reboots and upgrades. Changes only on reinstall
> - Manual: `man 5 machine-id`

```
f3a1c9...e7    ← 32 lowercase hex characters (value not shown, see below)
```

> [!warning] Treat it as semi-private
> It's not a password, but it uniquely identifies your machine forever. Don't paste it publicly. If an app needs a stable ID for itself, derive one: `systemd-id128 machine-id --app-specific=<app-uuid>`. That gives a different ID per app, which can't be traced back.

# Where you meet it

| Place                          | Use                                                      |
| ------------------------------ | -------------------------------------------------------- |
| `/var/log/journal/<machine-id>/` | Journal files of this machine → [[var]]               |
| `/boot/<machine-id>/...`       | Some bootloaders name kernel entries by it → [[boot]]    |
| DHCP / DHCPv6 (`systemd-networkd`) | Stable client identifier, stable IPv6 address        |
| D-Bus                          | Machine identity for the session bus                     |
| `hostnamectl`, `journalctl --list-boots` | Shown / used for filtering                      |

> [!danger] Cloned VMs and disk images
> If you copy a VM or disk image, both copies keep the **same** machine-id. They then get the same DHCP address, mixed journal folders, etc. Before making a template image, empty the file (`sudo truncate -s 0 /etc/machine-id`, don't delete it). A new ID is generated on next boot.

# Related IDs

| ID                | Where                                    | Changes                    |
| ----------------- | ---------------------------------------- | -------------------------- |
| machine-id        | `/etc/machine-id`                        | On reinstall               |
| boot-id           | `/proc/sys/kernel/random/boot_id`        | Every boot                 |
| product UUID      | `/sys/class/dmi/id/product_uuid` (root)  | Never (hardware/firmware)  |

# Scripting recipes

### Check it is valid
```bash
grep -qE '^[0-9a-f]{32}$' /etc/machine-id && echo valid
```

### Boot ID (different each boot)
```bash
cat /proc/sys/kernel/random/boot_id
```

### Journal folder of this machine
```bash
ls -d /var/log/journal/"$(cat /etc/machine-id)"
```

### Stable app-specific ID (safe to send to a server)
```bash
systemd-id128 machine-id --app-specific=4f1c2a8a6c1e4b5c9a7d2e3f4a5b6c7d
```

# See also
- [[os-release]] · [[hostname]] — other identity files
- [[proc-self-highlights]] · [[proc]] — `boot_id`
- [[journal]] · [[var]] · [[systemd]] · [[etc]]
