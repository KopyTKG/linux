---
tags:
  - linux
  - storage
  - disks
  - partitions
aliases:
  - Partitions
  - GPT
  - MBR
  - Partition table
---
A **partition table** at the start of a disk splits it into independent regions (partitions). Each partition becomes its own block device (`nvme0n1p1`, `sda2`, ...) and can hold a different filesystem → [[filesystems]].

# MBR vs GPT

|                         | MBR (msdos, "DOS")                   | GPT (GUID Partition Table)                     |
| ----------------------- | ------------------------------------ | ---------------------------------------------- |
| Introduced              | 1983                                 | ~2000, part of UEFI                            |
| Max disk size           | 2 TiB (with 512 B sectors)           | 8 ZiB (practically unlimited)                  |
| Max partitions          | 4 primary (or 3 + extended with logicals) | 128 by default                            |
| Where stored            | First sector only                    | Start of disk **and** backup copy at the end   |
| Integrity               | None                                 | CRC32 checksums                                |
| Partition type          | 1-byte code (`83` Linux, `82` swap, `ef` EFI) | 16-byte GUID                          |
| Partition names         | No                                   | Yes (`PARTLABEL`)                              |
| Boot firmware           | BIOS (legacy)                        | UEFI (BIOS possible with a BIOS boot partition) |

Use **GPT** for everything new. MBR only for old BIOS-only machines or small devices that need it.

```bash
lsblk -o NAME,PTTYPE -d     # gpt / dos
```

# Common GPT partition types

| Type                         | GUID (shortened)          | fdisk alias | Used for                          |
| ---------------------------- | ------------------------- | ----------- | --------------------------------- |
| EFI System Partition (ESP)   | `C12A7328-F81F-...`       | `uefi`      | Bootloaders, maybe kernels → [[boot]] |
| Linux filesystem             | `0FC63DAF-8483-...`       | `linux`     | Generic Linux data                |
| Linux root (x86-64)          | `4F68BCE3-E8CD-...`       | —           | Auto-discovered root (systemd)    |
| Linux home                   | `933AC7E1-2EB4-...`       | `home`      | Auto-discovered `/home`           |
| Linux swap                   | `0657FD6D-A4AB-...`       | `swap`      | Swap → [[swap]]                   |
| Linux LVM                    | `E6D6D379-F507-...`       | `lvm`       | LVM physical volume               |
| Linux RAID                   | `A19D880F-05FC-...`       | `raid`      | mdadm member                      |
| Microsoft basic data         | `EBD0A0A2-B9E5-...`       | —           | NTFS / exFAT / FAT data           |
| BIOS boot                    | `21686148-6449-...`       | —           | GRUB on GPT + legacy BIOS         |

The type is only a **label**. The kernel does not care; tools like systemd's partition auto-discovery and installers do.

# The ESP
The **EFI System Partition** is a FAT32 (`vfat`) partition the UEFI firmware reads to find bootloaders. Typical size 300 MiB – 1 GiB (larger if kernels live there). Mounted at `/boot`, `/boot/efi` or `/efi` depending on setup → [[boot]].

# Alignment
Partitions should start on a multiple of **1 MiB** (2048 sectors of 512 B). Modern tools do this automatically. Misaligned partitions on SSDs and 4K-sector disks cause extra read-modify-write and slower I/O.

```bash
cat /sys/block/nvme0n1/nvme0n1p1/start      # 2048 = aligned at 1 MiB
```

# Identifiers: UUID vs PARTUUID vs LABEL vs PARTLABEL

| Identifier  | Belongs to        | Set by                       | Changes when ...              | Example                                |
| ----------- | ----------------- | ---------------------------- | ----------------------------- | -------------------------------------- |
| `UUID`      | **Filesystem**    | `mkfs` (random)              | You reformat                  | `3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c` |
| `PARTUUID`  | **Partition** (GPT entry) | Partitioning tool    | You recreate the partition    | `1a2b3c4d-5e6f-4a1b-8c2d-3e4f5a6b7c8d` |
| `LABEL`     | Filesystem        | You (`mkfs -L`, `e2label`)   | You rename it                 | `Data`                                 |
| `PARTLABEL` | Partition (GPT)   | You (`sgdisk -c`, `parted name`) | You rename it             | `root`                                 |

- FAT filesystems have short UUIDs like `ABCD-1234`; NTFS like `0123456789ABCDEF`.
- **fstab** usually uses `UUID=` → [[fstab]]. Kernel `root=` can use `UUID=` (needs initramfs) or `PARTUUID=` (works without).
- MBR has no real PARTUUID; tools fake one from the disk signature: `1a2b3c4d-01`.

```bash
lsblk -o NAME,UUID,PARTUUID,LABEL,PARTLABEL
ls -l /dev/disk/by-uuid/ /dev/disk/by-partuuid/
```

# Reading partition tables (safe)
These only **read**. Most need root because they open the raw device.

```bash
lsblk -o NAME,SIZE,TYPE,PARTTYPENAME,FSTYPE     # no root needed
cat /proc/partitions                             # no root, sizes in 1 KiB blocks
sudo fdisk -l /dev/nvme0n1                       # human-readable table
sudo sfdisk -d /dev/nvme0n1                      # dump; can be saved as a backup
sudo parted /dev/nvme0n1 unit MiB print          # alternative view
sudo gdisk -l /dev/nvme0n1                       # GPT-focused tool
```

`fdisk -l` output (example):
```
Disk /dev/nvme0n1: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: Samsung SSD 980 1TB
Units: sectors of 1 * 512 = 512 bytes
Disklabel type: gpt

Device            Start        End    Sectors   Size Type
/dev/nvme0n1p1     2048    2099199    2097152     1G EFI System
/dev/nvme0n1p2  2099200   18876415   16777216     8G Linux swap
/dev/nvme0n1p3 18876416 1953523711 1934647296 922.5G Linux filesystem
```

> [!tip] Back up the partition table
> `sudo sfdisk -d /dev/nvme0n1 > nvme0n1-table.txt` — small text file, restorable with `sfdisk /dev/nvme0n1 < nvme0n1-table.txt`.

# Changing partitions

> [!danger] These commands rewrite the partition table
> One typo in the device name can wipe the wrong disk. Before running any of them:
> 1. Run `lsblk -o NAME,SIZE,MODEL,MOUNTPOINTS` and **check the device by size and model**.
> 2. Make sure nothing on that disk is mounted.
> 3. Have a backup.
> ```bash
> sudo fdisk /dev/sdX                 # interactive; nothing is written until 'w'
> sudo cfdisk /dev/sdX                # menu-based
> sudo parted /dev/sdX mklabel gpt    # NEW EMPTY TABLE = all partitions gone
> sudo sgdisk -n 1:0:+1G -t 1:ef00 /dev/sdX   # scripted: create ESP
> sudo sgdisk --zap-all /dev/sdX      # destroy GPT and MBR
> sudo wipefs -a /dev/sdX             # remove all filesystem/partition signatures
> sudo partprobe /dev/sdX             # tell the kernel to re-read the table
> ```

# Scripting recipes

### Partition table type and partition count per disk
```bash
lsblk -dno NAME,PTTYPE | while read -r disk pt; do
  n=$(lsblk -rno TYPE "/dev/$disk" | grep -c '^part$')
  echo "$disk ${pt:-none} $n partitions"
done
```

### Partition start/size in MiB from `/sys`
```bash
for p in /sys/block/nvme0n1/nvme0n1p*; do
  echo "${p##*/} start=$(( $(< "$p/start") / 2048 ))MiB size=$(( $(< "$p/size") / 2048 ))MiB"
done
```

### Find a partition by label or UUID
```bash
readlink -f /dev/disk/by-label/Data
findfs UUID=3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c   # may need root
```

# Find stuff

| Looking for              | Where                                         |
| ------------------------ | --------------------------------------------- |
| GPT or MBR?              | `lsblk -dno NAME,PTTYPE`                      |
| Partition type           | `lsblk -o NAME,PARTTYPENAME`                  |
| UUIDs / labels           | `lsblk -f`, `/dev/disk/by-*`                  |
| Start sector             | `/sys/block/<disk>/<part>/start`              |
| Which partition is ESP   | `lsblk -o NAME,PARTTYPENAME \| grep -i efi`   |

# See also
- [[storage]] · [[block-devices]] · [[filesystems]] · [[mounting]]
- [[boot]] — ESP and bootloaders
- [[fstab]] — mount by UUID
- [[dev]] — `/dev/disk/by-uuid`
