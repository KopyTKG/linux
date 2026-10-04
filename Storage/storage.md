---
tags:
  - linux
  - storage
  - disks
  - filesystem
aliases:
  - Storage
  - Disks and filesystems
---
Storage on Linux is a stack of layers. A physical disk becomes a **block device**, a **partition table** splits it, optional layers like encryption or LVM sit in the middle, a **filesystem** organises the bytes into files, and finally that filesystem is **mounted** somewhere in the single directory tree that starts at `/`.

> [!info] Everything is one tree
> Linux has no drive letters. Every filesystem — SSD partition, USB stick, network share, RAM disk — is attached ("mounted") to a directory inside one tree. `/home` can live on a different disk than `/`, and you only notice it with `findmnt` or `df`.

# The layers

```
 physical disk         /dev/nvme0n1                (Samsung SSD 980 1TB)
   └ partition table   GPT
       ├ partition     /dev/nvme0n1p1  → vfat  → mounted at /boot
       ├ partition     /dev/nvme0n1p2  → swap  → [SWAP]
       └ partition     /dev/nvme0n1p3
           └ (optional) LUKS  → /dev/mapper/cryptroot
               └ (optional) LVM → /dev/vg0/root
                   └ filesystem btrfs/ext4/xfs → mounted at /
```

| Layer               | What it does                                    | Note                       |
| ------------------- | ----------------------------------------------- | -------------------------- |
| Block device        | Raw disk the kernel exposes in `/dev`           | [[block-devices]]          |
| Partition table     | Splits a disk into regions (GPT or MBR)         | [[partitions]]             |
| Encryption / LVM / RAID | Optional virtual block devices (device mapper) | [[lvm-raid-luks]]       |
| Filesystem          | Files, directories, permissions, inodes         | [[filesystems]], [[btrfs]] |
| Mount               | Attaches a filesystem to a directory            | [[mounting]]               |
| Swap                | Disk/RAM area used as overflow for memory       | [[swap]]                   |
| Usage & health      | Free space, big files, I/O speed, SMART         | [[disk-usage]], [[disk-health-io]] |

# Which command answers what

| Question                                     | Command                                             |
| -------------------------------------------- | --------------------------------------------------- |
| What disks and partitions do I have?         | `lsblk`, `lsblk -f`                                 |
| What filesystem / UUID is on a partition?    | `lsblk -f`, `blkid` (full info needs root)          |
| What is mounted where, with which options?   | `findmnt`, `findmnt /home`                          |
| How full are my filesystems?                 | `df -h`                                             |
| What is using the space?                     | `du -sh *`, `du -xh --max-depth=1 / \| sort -h`     |
| Am I out of inodes?                          | `df -i`                                             |
| Partition table layout?                      | `sudo fdisk -l /dev/nvme0n1`, `sudo sfdisk -d /dev/nvme0n1` |
| Is it an SSD or HDD?                         | `lsblk -d -o NAME,ROTA` (`0` = SSD)                 |
| Is swap active?                              | `swapon --show`, `cat /proc/swaps`                  |
| Is the disk healthy?                         | `sudo smartctl -a /dev/sda`, `sudo nvme smart-log /dev/nvme0` |
| How busy is the disk right now?              | `/proc/diskstats`, `iostat -x 1`                    |
| Encrypted / LVM / RAID devices?              | `lsblk`, `dmsetup ls`, `lvs`, `cat /proc/mdstat`    |

> [!danger] Commands that destroy data
> `mkfs.*`, `fdisk`/`parted`/`sgdisk` in write mode, `wipefs`, `dd of=/dev/...`, `btrfs balance`, `cryptsetup luksFormat`, resizing tools.
> **Always run `lsblk` first and double-check the device name.** `/dev/sda` today can be `/dev/sdb` after a reboot or after plugging in a USB stick. Prefer `/dev/disk/by-id/...` paths in scripts.

# Where the kernel shows storage

| Path                    | What                                                    |
| ----------------------- | ------------------------------------------------------- |
| `/dev/sd*`, `/dev/nvme*`| Block device files → [[dev]]                            |
| `/dev/disk/by-*`        | Stable symlinks: `by-uuid`, `by-id`, `by-label`, `by-partuuid`, `by-path` |
| `/sys/block/<dev>/`     | Size, model, queue settings, statistics → [[sys]]       |
| `/proc/partitions`      | All block devices with size in 1 KiB blocks → [[proc]]  |
| `/proc/diskstats`       | I/O counters per device                                 |
| `/proc/self/mountinfo`  | What is mounted where (source for `findmnt`) → [[mtab]] |
| `/proc/swaps`           | Active swap                                             |
| `/etc/fstab`            | What to mount at boot → [[fstab]]                       |
| `/etc/crypttab`         | Which encrypted devices to unlock at boot → [[crypttab]]|

# See also
- [[block-devices]] · [[partitions]] · [[filesystems]] · [[mounting]] · [[disk-usage]] · [[btrfs]] · [[lvm-raid-luks]] · [[swap]] · [[disk-health-io]]
- [[dev]], [[sys]], [[proc]], [[mnt]], [[media]], [[boot]]
- `man 8 lsblk`, `man 8 mount`, `man 5 fstab`
