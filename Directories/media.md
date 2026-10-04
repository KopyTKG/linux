---
tags:
  - linux
  - filesystem
  - media
  - mount
aliases:
  - /media
  - /run/media
  - removable media
---
`/media` is where **removable media** (USB sticks, SD cards, CDs, external disks) get mounted automatically when you plug them in.

> [!info] `/media` often does not exist
> ```
> ls: cannot access '/media': No such file or directory
> ```
> **udisks2** (the automount service desktops use) mounts to **`/run/media/$USER/<label>`** on most modern setups. Some systems, and older ones, use `/media/$USER/<label>` or `/media/<label>` instead. Same idea, different path; it's a build-time choice of udisks2.
> `/run` is a tmpfs, so mountpoint directories vanish on reboot → see [[run]].

# How automounting works
1. You plug in a USB stick → kernel creates `/dev/sdb1` → [[dev]]
2. `udev` notices it and tells **udisks2** (`udisksd` daemon)
3. Your file manager (Dolphin, Nautilus, ...) asks udisks2 to mount it
4. udisks2 creates `/run/media/alice/<LABEL>` and mounts there, **owned by you**, no sudo needed

Without a desktop environment nothing automounts — use `udisksctl` or `udiskie` (small automount daemon).

| Path                           | Used by                                   |
| ------------------------------ | ----------------------------------------- |
| `/run/media/$USER/<label>`     | udisks2, default on most modern desktops  |
| `/media/$USER/<label>`         | udisks2 built with the `/media` option    |
| `/media/<label>`               | Older systems (udisks 1, hal)             |
| `/mnt/...`                     | **You**, manually → [[mnt]]               |

# Commands
```bash
lsblk -f                            # find the device, e.g. sdb1, LABEL=USB
udisksctl mount -b /dev/sdb1        # Mounted /dev/sdb1 at /run/media/alice/USB
udisksctl unmount -b /dev/sdb1
udisksctl power-off -b /dev/sdb     # spin down, safe to unplug
```

> [!warning] Always unmount before unplugging
> Linux caches writes in RAM. Pulling the stick right after copying can lose data. `udisksctl unmount` (or "Eject" in the file manager) flushes everything first. `sync` forces flush manually.

> [!tip] Label your drives
> Mountpoint name = filesystem **label**. Without a label you get a UUID like `/run/media/alice/3A1F-9C2B`.
> Set label: `sudo fatlabel /dev/sdb1 USB` (FAT), `sudo e2label /dev/sdb1 Backup` (ext4), `sudo exfatlabel /dev/sdb1 USB`.

# Scripting recipes

### Where is my USB stick mounted?
```bash
findmnt -rno TARGET,SOURCE | grep "^/run/media/$USER"
ls "/run/media/$USER" 2>/dev/null || echo "Nothing mounted"
```

### Mount by label in a script
```bash
dev=$(blkid -L USB) || { echo "USB not plugged in"; exit 1; }
mp=$(findmnt -rno TARGET "$dev") || {
  udisksctl mount -b "$dev" >/dev/null
  mp=$(findmnt -rno TARGET "$dev")
}
echo "USB at $mp"
```

### Backup to external disk when it's present
```bash
dest=/run/media/$USER/Backup
mountpoint -q "$dest" || { echo "Backup disk not mounted"; exit 1; }
rsync -a --delete ~/Documents/ "$dest/Documents/"
```

### Watch plug / unplug events live
```bash
udevadm monitor --udev --subsystem-match=block
```

# Find stuff
| Looking for                  | Command                                      |
| ---------------------------- | -------------------------------------------- |
| Mounted removable drives     | `ls /run/media/$USER`                        |
| Is a device removable        | `lsblk -o NAME,RM,HOTPLUG,TRAN` (`RM=1`, `TRAN=usb`) |
| Device by label / UUID       | `ls -l /dev/disk/by-label/` → [[dev]]        |
| Mount history / errors       | `journalctl -u udisks2 -b`                   |

# See also
- [[mnt]] — manual mounts, `findmnt`, `/etc/fstab`
- [[run]] — `/run/media`
- [[dev]] — `/dev/sdX`, `/dev/disk/by-*`
- `man 1 udisksctl`
