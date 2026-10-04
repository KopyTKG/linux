---
tags:
  - linux
  - filesystem
  - mnt
  - mount
aliases:
  - /mnt
  - mounting
---
`/mnt` is an empty directory meant for **temporary, manual mounts** by the admin — e.g. a disk you plug in for a quick fix or a backup. Nothing mounts there automatically.

> [!info] What "mounting" means
> Linux has **one** directory tree starting at `/`. A disk/partition isn't a drive letter (`D:`); you attach ("mount") it to some **empty directory**, and its content appears there.
> Usually `/mnt` is empty: `mountpoint -q /mnt` → not a mountpoint.

# `/mnt` vs `/media` vs `/run/media`

| Path                    | Who mounts                         | When                              |
| ----------------------- | ---------------------------------- | --------------------------------- |
| `/mnt`                  | **You**, manually (`sudo mount`)   | Temporary admin work              |
| `/run/media/$USER/<label>` | **udisks2** automatically (file manager click) | USB sticks, extra disks — most modern desktops |
| `/media/<label>`        | udisks2 on some/older systems      | Same as above, other path → [[media]] |
| anywhere (e.g. `/data`) | **`/etc/fstab`** at boot           | Permanent disks                   |

# Mount by hand
```bash
lsblk -f                                   # list disks, filesystems, labels, UUIDs
sudo mkdir -p /mnt/win
sudo mount /dev/sda2 /mnt/win              # filesystem type detected automatically
sudo mount -o ro /dev/sda2 /mnt/win        # read-only (safe for inspecting)
sudo umount /mnt/win                       # detach — umount, not "unmount"
```

Example — a Windows disk that's **not** mounted:
```
NAME          SIZE FSTYPE MOUNTPOINTS
sda         232,9G
├─sda1         16M
├─sda2        232G ntfs
└─sda3        889M ntfs
```
`sudo mount -t ntfs3 /dev/sda2 /mnt` would make it readable in `/mnt`.

Other mount types:
```bash
sudo mount -o loop image.iso /mnt          # ISO / disk image
sudo mount -t tmpfs -o size=1G tmpfs /mnt  # RAM disk
sudo mount --bind /some/dir /mnt           # same dir visible in second place
```

> [!warning] "target is busy"
> `umount` fails if any process uses a file there — or your shell's **cwd** is inside. `cd` out, then find the culprit:
> `fuser -vm /mnt` or `lsof +f -- /mnt`

# Mount with udisks (no sudo)
This is what the file manager does:
```bash
udisksctl mount -b /dev/sda2      # → mounted at /run/media/alice/<label>
udisksctl unmount -b /dev/sda2
udisksctl power-off -b /dev/sda   # safe to unplug
```

# Permanent mounts — `/etc/fstab`
```
# <device>                                 <mountpoint>  <type>  <options>          <dump> <pass>
UUID=3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c  /             btrfs   subvol=/@,noatime  0      0
```
Use `UUID=` (from `lsblk -f`), not `/dev/sdX` — letters can change between boots. Test with `sudo mount -a` before rebooting. Details → [[etc]].

# Scripting recipes

### Is something mounted?
```bash
mountpoint -q /mnt && echo mounted || echo "not mounted"
```

### What device / filesystem is a path on?
```bash
findmnt -T ~                       # mount that contains a path
df -h ~                            # free space of that mount
findmnt -no FSTYPE -T /            # just the type: btrfs
```

### List real disk mounts (skip virtual ones)
```bash
findmnt -t btrfs,ext4,vfat,exfat,ntfs3,xfs -o TARGET,SOURCE,FSTYPE
```
Example (btrfs with subvolumes):
```
/            /dev/nvme0n1p3[/@]      btrfs
├─/var/tmp   /dev/nvme0n1p3[/@tmp]   btrfs
├─/root      /dev/nvme0n1p3[/@root]  btrfs
├─/var/cache /dev/nvme0n1p3[/@cache] btrfs
├─/var/log   /dev/nvme0n1p3[/@log]   btrfs
├─/srv       /dev/nvme0n1p3[/@srv]   btrfs
├─/boot      /dev/nvme0n1p1          vfat
└─/home      /dev/nvme0n1p4          btrfs
```
`[/@tmp]` = btrfs **subvolume** mounted from the same partition.

### Mount, do work, always unmount
```bash
dev=/dev/sda2; mp=$(mktemp -d)
sudo mount -o ro "$dev" "$mp"
trap 'sudo umount "$mp"; rmdir "$mp"' EXIT
ls "$mp"
```

# Find stuff
| Looking for                      | Command                              |
| -------------------------------- | ------------------------------------ |
| All mounts as a tree             | `findmnt`                            |
| Disks, partitions, UUIDs, labels | `lsblk -f`                           |
| Free space per mount             | `df -h -x tmpfs -x devtmpfs`         |
| Raw kernel mount list            | `cat /proc/self/mountinfo` → [[proc]]|
| Mounts defined at boot           | `/etc/fstab`, `systemctl list-units -t mount` |
| Disk device files                | `/dev/sd*`, `/dev/nvme*`, `/dev/disk/by-uuid/` → [[dev]] |

# See also
- [[media]] — automatic mounts of removable media
- [[dev]] — block devices you mount
- [[etc]] — `/etc/fstab`
- [[run]] — `/run/media/$USER`
- `man 8 mount`, `man 8 findmnt`, `man 5 fstab`
