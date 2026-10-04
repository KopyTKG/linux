---
tags:
  - linux
  - storage
  - lvm
  - raid
  - luks
  - encryption
aliases:
  - LVM
  - RAID
  - LUKS
  - mdadm
  - Device mapper
  - dm-crypt
---
Between a partition and a filesystem you can stack **virtual block devices**: encryption (LUKS), flexible volumes (LVM) and redundancy (RAID). Each layer takes block devices in and presents a new block device out, so the filesystem on top doesn't know or care.

```
/dev/nvme0n1p3 ──► LUKS (dm-crypt) ──► /dev/mapper/cryptroot ──► LVM VG "vg0" ──► /dev/vg0/root ──► ext4 → /
                                                                              └─► /dev/vg0/home ──► ext4 → /home
```

```bash
lsblk                     # shows the whole stack as a tree (TYPE crypt, lvm, raid1)
```
```
nvme0n1           259:0  0 931,5G 0 disk
├─nvme0n1p1       259:1  0     1G 0 part  /boot
└─nvme0n1p2       259:2  0 930,5G 0 part
  └─cryptroot     254:0  0 930,5G 0 crypt
    ├─vg0-root    254:1  0   100G 0 lvm   /
    ├─vg0-swap    254:2  0     8G 0 lvm   [SWAP]
    └─vg0-home    254:3  0 822,5G 0 lvm   /home
```

# Device mapper
The kernel framework behind LUKS and LVM (and many other things). Every mapped device is `/dev/dm-N` with a friendly name in `/dev/mapper/`.

```bash
ls -l /dev/mapper/
sudo dmsetup ls                 # names and major:minor
sudo dmsetup ls --tree          # stack
sudo dmsetup table              # mapping tables (sector ranges, targets: crypt, linear, ...)
ls /sys/block/dm-0/slaves/      # what dm-0 is built on
cat /sys/block/dm-0/dm/name     # its name (no root)
```

# LUKS (disk encryption)
**dm-crypt** encrypts a block device; **LUKS** is the on-disk header format that stores the encrypted master key in up to 32 **keyslots** (passphrases, key files, TPM, FIDO2).

| Term            | Meaning                                                           |
| --------------- | ----------------------------------------------------------------- |
| LUKS2 / LUKS1   | Header versions. LUKS2 is current (Argon2 key derivation)         |
| Keyslot         | One way to unlock (a passphrase, a key file, TPM...)              |
| Master key      | Real encryption key, stored encrypted in each keyslot             |
| Mapping         | Unlocked device in `/dev/mapper/<name>`                           |
| `/etc/crypttab` | Which devices to unlock at boot → [[crypttab]]                    |

Read-only inspection:
```bash
lsblk -f                                   # FSTYPE crypto_LUKS on the raw partition
sudo cryptsetup isLuks /dev/nvme0n1p2 && echo LUKS
sudo cryptsetup luksDump /dev/nvme0n1p2    # version, cipher, keyslots (no secrets)
sudo cryptsetup status cryptroot           # cipher, key size, underlying device
```

Daily use:
```bash
sudo cryptsetup open /dev/sdb1 backup      # → /dev/mapper/backup (asks passphrase)
sudo mount /dev/mapper/backup /mnt
sudo umount /mnt && sudo cryptsetup close backup
```

> [!danger] Destructive LUKS operations
> `luksFormat` **destroys all data** on the device. Losing the header (or all passphrases) = data is gone forever, nobody can recover it.
> Double-check the device with `lsblk -o NAME,SIZE,MODEL,FSTYPE` first.
> ```bash
> sudo cryptsetup luksFormat /dev/sdX1                 # NEW encrypted container, wipes data
> sudo cryptsetup luksAddKey /dev/sdX1                 # add a passphrase
> sudo cryptsetup luksKillSlot /dev/sdX1 1             # remove keyslot 1 (don't remove the last one!)
> sudo cryptsetup luksHeaderBackup /dev/sdX1 --header-backup-file hdr.img   # DO this one, keep it safe
> ```

# LVM (Logical Volume Manager)
LVM pools disks together and cuts them into resizable volumes, independent of partition boundaries.

| Layer | Name               | Example           | List   | Details        |
| ----- | ------------------ | ----------------- | ------ | -------------- |
| PV    | Physical Volume    | `/dev/mapper/cryptroot`, `/dev/sda1` | `pvs` | `pvdisplay` |
| VG    | Volume Group (pool)| `vg0`             | `vgs`  | `vgdisplay`    |
| LV    | Logical Volume     | `/dev/vg0/root`   | `lvs`  | `lvdisplay`    |

```
$ sudo lvs
  LV   VG  Attr       LSize   Pool Origin Data%
  home vg0 -wi-ao---- 822.50g
  root vg0 -wi-ao---- 100.00g
  swap vg0 -wi-ao----   8.00g
```

Paths: `/dev/vg0/root` = `/dev/mapper/vg0-root` (dashes in names are doubled: `my-vg` → `my--vg`).

Features: grow volumes online, add disks to a pool, thin provisioning, LVM snapshots, caching (SSD in front of HDD).

> [!danger] Changing LVM
> ```bash
> sudo lvextend -r -L +20G /dev/vg0/home     # grow LV and filesystem (-r) — fairly safe
> sudo lvreduce -r -L -20G /dev/vg0/home     # SHRINK — xfs can't shrink; backup first
> sudo lvremove /dev/vg0/test                # deletes the volume and its data
> sudo vgreduce / pvremove / pvcreate        # change the pool
> ```
> Always check free space first: `sudo vgs` (column `VFree`).

# Software RAID (mdadm)
The kernel `md` driver combines disks into `/dev/mdN`.

| Level  | Min disks | Survives          | Usable space     | Use                       |
| ------ | --------- | ----------------- | ---------------- | ------------------------- |
| RAID0  | 2         | **No disk loss**  | 100 %            | Speed only, never for data you care about |
| RAID1  | 2         | n−1 disks         | 1 disk           | Mirroring, simplest       |
| RAID5  | 3         | 1 disk            | n−1              | Space-efficient, slow rebuild |
| RAID6  | 4         | 2 disks           | n−2              | Big arrays                |
| RAID10 | 4         | 1 per mirror      | 50 %             | Speed + redundancy        |

```bash
cat /proc/mdstat                     # status of all arrays (file exists only when md is loaded)
sudo mdadm --detail /dev/md0         # one array in detail
sudo mdadm --examine /dev/sda1       # RAID superblock on a member disk
```

`/proc/mdstat` example:
```
Personalities : [raid1]
md0 : active raid1 sdb1[1] sda1[0]
      976630464 blocks super 1.2 [2/2] [UU]
      bitmap: 0/8 pages [0KB], 65536KB chunk

unused devices: <none>
```
`[UU]` = both disks up. `[U_]` = one disk missing/failed → **degraded**.

> [!warning] RAID is not a backup
> RAID protects against a **dead disk**. It happily mirrors an accidental `rm -rf`, ransomware, or filesystem corruption to all disks.

> [!danger] Creating / changing arrays
> ```bash
> sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdX1 /dev/sdY1   # wipes members
> sudo mdadm /dev/md0 --fail /dev/sdX1 --remove /dev/sdX1
> sudo mdadm /dev/md0 --add /dev/sdZ1                                           # starts rebuild
> ```

btrfs and ZFS can do RAID themselves without mdadm → [[btrfs]].

# Scripting recipes

### Show the full stack under a mount point
```bash
src=$(findmnt -no SOURCE --target /home | sed 's/\[.*//')
lsblk -s "$src"                   # -s = inverse tree: from the device down to the disk
```

### Is anything encrypted?
```bash
lsblk -rno NAME,TYPE | awk '$2=="crypt" {print "encrypted mapping:", $1}'
lsblk -rno NAME,FSTYPE | awk '$2=="crypto_LUKS" {print "LUKS container:", $1}'
```

### Alert on degraded RAID (for a timer)
```bash
[[ -r /proc/mdstat ]] || exit 0
if grep -q '\[[U]*_[U_]*\]' /proc/mdstat; then
  echo "RAID degraded:"; cat /proc/mdstat; exit 1
fi
```

### Names of all device-mapper devices without root
```bash
for d in /sys/block/dm-*; do
  [[ -e $d ]] || continue
  echo "${d##*/} = $(< "$d/dm/name") (on: $(ls "$d/slaves" | tr '\n' ' '))"
done
```

# Find stuff

| Looking for              | Where                                       |
| ------------------------ | ------------------------------------------- |
| Whole stack              | `lsblk`, `lsblk -s /dev/mapper/...`         |
| Mapped devices           | `/dev/mapper/`, `dmsetup ls`, `/sys/block/dm-*` |
| LUKS header info         | `cryptsetup luksDump`                       |
| Unlock at boot           | `/etc/crypttab` → [[crypttab]]              |
| LVM                      | `pvs`, `vgs`, `lvs`, `/etc/lvm/`            |
| RAID                     | `/proc/mdstat`, `mdadm --detail`, `/etc/mdadm.conf` |

# See also
- [[storage]] · [[block-devices]] · [[partitions]] · [[filesystems]] · [[swap]]
- [[crypttab]] · [[fstab]] · [[boot]] — initramfs unlocks root before mounting
- [[dev]] — `/dev/mapper`
- `man 8 cryptsetup`, `man 8 lvm`, `man 8 mdadm`, `man 8 dmsetup`
