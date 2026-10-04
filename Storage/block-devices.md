---
tags:
  - linux
  - storage
  - disks
  - dev
aliases:
  - Block devices
  - lsblk
---
A **block device** is anything the kernel can read and write in fixed-size blocks at random positions: SSDs, hard disks, USB sticks, SD cards, partitions, RAM disks, encrypted volumes. Each one appears as a file in [[dev]] and as a directory in `/sys/block` → [[sys]].

> [!info] Block vs character device
> `ls -l /dev/nvme0n1` starts with `b` (block). `/dev/tty` or `/dev/null` start with `c` (character, a byte stream without seeking). Only block devices can hold filesystems.

# Naming

| Name pattern           | Device type                               | Partitions               |
| ---------------------- | ----------------------------------------- | ------------------------ |
| `sda`, `sdb`, ...      | SATA/SAS/USB disks (SCSI subsystem)       | `sda1`, `sda2`           |
| `nvme0n1`              | NVMe SSD: controller `0`, namespace `1`   | `nvme0n1p1`, `nvme0n1p2` |
| `mmcblk0`              | SD card / eMMC                            | `mmcblk0p1`              |
| `vda`, `xvda`          | Virtual disks in VMs (virtio / Xen)       | `vda1`                   |
| `sr0`                  | Optical drive                             | —                        |
| `loop0`                | Loop device: a file used as a disk        | `loop0p1`                |
| `zram0`                | Compressed RAM disk (usually swap)        | —                        |
| `dm-0`, `dm-1`         | Device mapper (LUKS, LVM) → [[lvm-raid-luks]] | names in `/dev/mapper/` |
| `md0`, `md127`         | Software RAID (mdadm)                     | `md0p1`                  |

> [!warning] Names are not stable
> `sda`/`sdb` are given in detection order. Plug in a USB stick before boot and your data disk may become `sdc`. NVMe numbers can swap too.
> For anything permanent (fstab, scripts) use **UUID** or `/dev/disk/by-id/...` → [[partitions]].

# Major and minor numbers
Every device file has a `major:minor` pair. The **major** number picks the driver, the **minor** picks the device/partition.

```
$ lsblk -o NAME,MAJ:MIN
sda           8:0
├─sda1        8:1
zram0       253:0
nvme0n1     259:0
├─nvme0n1p1 259:1
```

| Major | Driver                |
| ----- | --------------------- |
| `8`   | SCSI disks (`sd*`)    |
| `259` | Block extended (NVMe partitions and more) |
| `253`/`254` | Device mapper / zram (dynamic) |
| `7`   | Loop devices          |

Full list: `/proc/devices`.

# `lsblk`
The main tool. Reads `/sys` and udev, no root needed.

```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 931,5G  0 disk
└─sda1        8:1    0 931,5G  0 part /data
zram0       253:0    0    16G  0 disk [SWAP]
nvme0n1     259:0    0 931,5G  0 disk
├─nvme0n1p1 259:1    0     1G  0 part /boot
├─nvme0n1p2 259:2    0     8G  0 part [SWAP]
└─nvme0n1p3 259:3    0 922,5G  0 part /home
                                       /
```

`/` and `/home` on the same partition = btrfs subvolumes → [[btrfs]].

| Option                   | Shows                                                  |
| ------------------------ | ------------------------------------------------------ |
| `lsblk -f`               | Filesystem type, label, UUID, free space, mountpoint   |
| `lsblk -d`               | Only whole disks, no partitions                        |
| `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,UUID,MODEL,ROTA` | Custom columns |
| `lsblk -b`               | Sizes in bytes (for scripts)                           |
| `lsblk -J` / `-P` / `-r` | JSON / `KEY="value"` pairs / raw (for scripts)         |
| `lsblk -S`               | SCSI info: transport (sata, usb), vendor               |
| `lsblk -D`               | Discard (TRIM) support                                 |

Useful columns: `NAME SIZE TYPE FSTYPE LABEL UUID PARTUUID MOUNTPOINTS MODEL SERIAL ROTA TRAN RM RO`.

```
$ lsblk -d -o NAME,SIZE,MODEL,ROTA,TRAN
NAME      SIZE MODEL                ROTA TRAN
sda     931,5G ST1000DM010             1 sata
nvme0n1 931,5G Samsung SSD 980 1TB     0 nvme
```

# `/sys/block/<dev>/`
One file per value → [[sys]].

| File                         | Meaning                                         |
| ---------------------------- | ----------------------------------------------- |
| `size`                       | Size in **512-byte sectors** (always 512, even on 4K disks) |
| `device/model`               | Model name (SATA); NVMe: `device/model` too     |
| `removable`                  | `1` = removable media                           |
| `ro`                         | `1` = read-only                                 |
| `queue/rotational`           | `1` = spinning HDD, `0` = SSD                   |
| `queue/scheduler`            | I/O scheduler, active in `[...]` → [[disk-health-io]] |
| `queue/logical_block_size`   | Smallest addressable unit (usually 512)         |
| `queue/physical_block_size`  | Real sector size (512 or 4096)                  |
| `queue/discard_max_bytes`    | `0` = no TRIM support                           |
| `queue/read_ahead_kb`        | Read-ahead size                                 |
| `stat`                       | I/O counters (same as `/proc/diskstats`)        |
| `<dev>p1/`, `<dev>1/`        | Partitions as subdirectories                    |
| `holders/`                   | Who sits on top (dm-0 for LUKS/LVM)             |

```bash
echo $(( $(< /sys/block/nvme0n1/size) * 512 / 1024**3 )) GiB
```

# SSD vs HDD, TRIM
SSDs need to be told which blocks are free (**TRIM** / **discard**) so they can erase them in advance and stay fast.

| Way                          | How                                                       |
| ---------------------------- | --------------------------------------------------------- |
| Periodic (recommended)       | `fstrim.timer` (systemd) runs `fstrim -av` weekly → [[timers]] |
| Continuous                   | Mount option `discard` (btrfs uses `discard=async` by default on SSD) |
| Manual                       | `sudo fstrim -v /`                                        |

```bash
lsblk -D                        # DISC-GRAN/DISC-MAX non-zero = TRIM supported
systemctl is-enabled fstrim.timer
```

# Scripting recipes

### SSD or HDD for every disk
```bash
for d in /sys/block/*; do
  [[ -e $d/device ]] || continue                  # skip loop, zram, dm
  [[ $(< "$d/queue/rotational") == 1 ]] && t=HDD || t=SSD
  printf "%-10s %s\n" "${d##*/}" "$t"
done
```

### Disk sizes in GiB
```bash
for d in /sys/block/*; do
  [[ -e $d/device ]] || continue
  printf "%-10s %6d GiB\n" "${d##*/}" $(( $(< "$d/size") * 512 / 1024**3 ))
done
```

### Which disk is a partition on?
```bash
lsblk -no PKNAME /dev/nvme0n1p3    # nvme0n1
```

### Which device holds a directory?
```bash
findmnt -no SOURCE --target /home/alice    # /dev/nvme0n1p3[/@home]
df --output=source /home/alice | tail -1
```

### Script-friendly listing
```bash
lsblk -rno NAME,TYPE,FSTYPE | awk '$2=="part"'
lsblk -J -o NAME,SIZE,MOUNTPOINTS | jq '.blockdevices[].name'   # if jq installed
```

# Find stuff

| Looking for                       | Where                                        |
| --------------------------------- | -------------------------------------------- |
| All block devices                 | `lsblk`, `/proc/partitions`, `/sys/block`    |
| Stable names                      | `ls -l /dev/disk/by-id/`                     |
| Disk model / serial               | `lsblk -d -o NAME,MODEL,SERIAL`              |
| USB or SATA?                      | `lsblk -d -o NAME,TRAN`                      |
| What sits on top of a disk        | `ls /sys/block/nvme0n1/nvme0n1p3/holders/`   |
| udev properties                   | `udevadm info /dev/sda`                      |

# See also
- [[storage]] · [[partitions]] · [[lvm-raid-luks]] · [[disk-health-io]]
- [[dev]] — device files, `/dev/disk/by-*`
- [[sys]] — `/sys/block`, `/sys/class/block`
- [[proc]] — `/proc/partitions`, `/proc/diskstats`
