---
tags:
  - linux
  - storage
  - filesystem
  - mount
aliases:
  - Mounting
  - mount
  - findmnt
  - Bind mount
---
**Mounting** attaches a filesystem to a directory (the **mount point**). After `mount /dev/sda1 /data`, everything in `/data` comes from `sda1`. Whatever was in `/data` before is hidden (not deleted) until you unmount.

> [!info] Who mounts what
> | When                 | Who                                             | Config                    |
> | -------------------- | ----------------------------------------------- | ------------------------- |
> | Boot                 | systemd turns `/etc/fstab` lines into `.mount` units | [[fstab]], [[units]] |
> | USB stick plugged in | udisks2 (desktop) mounts to `/run/media/$USER/<label>` | [[media]], [[run]] |
> | Manually             | You, with `mount` (root) or `udisksctl` (user)  | [[mnt]]                   |
> | Virtual FS           | systemd at early boot (`/proc`, `/sys`, `/run`, `/dev`) | [[proc]], [[sys]] |

# Looking at mounts

```bash
findmnt                       # tree of everything mounted
findmnt /home                 # one mount point
findmnt --target /home/alice/file.txt   # which mount holds this path
findmnt -t ext4,btrfs,xfs     # only these types
findmnt -D                    # df-like output
findmnt --fstab               # what fstab says (not what is mounted)
findmnt --verify              # check fstab for mistakes
mount | column -t             # classic list
cat /proc/self/mountinfo      # raw kernel view → [[mtab]]
```

```
$ findmnt -t btrfs,vfat,ext4
TARGET    SOURCE                 FSTYPE OPTIONS
/         /dev/nvme0n1p3[/@]     btrfs  rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@
├─/home   /dev/nvme0n1p3[/@home] btrfs  rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@home
├─/boot   /dev/nvme0n1p1         vfat   rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii
└─/data   /dev/sda1              ext4   rw,noatime
```
`[/@home]` = btrfs subvolume → [[btrfs]].

# `mount` and `umount`

```bash
sudo mount /dev/sda1 /data                  # filesystem type auto-detected
sudo mount -t ext4 -o ro,noatime /dev/sda1 /data
sudo mount UUID=9b8a7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d /data
sudo mount /data                            # uses the fstab line for /data
sudo mount -a                               # mount everything in fstab not yet mounted
sudo mount -o remount,rw /                  # change options on a live mount
sudo umount /data                           # by mount point (or by device)
sudo umount -l /data                        # lazy: detach now, clean up when no longer busy
```

> [!warning] "target is busy"
> `umount` fails while any process has a file open or its working directory inside the mount. Find it:
> ```bash
> findmnt /data; fuser -vm /data; lsof +f -- /data
> ```
> Then `cd` out / close the program. Forcing (`-l`, `-f`) on a disk you'll unplug risks losing cached writes — run `sync` first.

# Mount options

| Option              | Meaning                                                              |
| ------------------- | -------------------------------------------------------------------- |
| `defaults`          | `rw,suid,dev,exec,auto,nouser,async`                                 |
| `ro` / `rw`         | Read-only / read-write                                               |
| `noatime`           | Don't update access time on every read (fewer writes, faster)        |
| `relatime`          | Update atime only if older than mtime or 24 h (kernel default)       |
| `nosuid`            | Ignore setuid/setgid bits → [[special-bits]]                         |
| `nodev`             | Ignore device files on this filesystem                               |
| `noexec`            | Don't allow running binaries from it (scripts via `bash x.sh` still work) |
| `nofail`            | Don't fail boot if the device is missing (USB disks, second drives)  |
| `noauto`            | Don't mount with `mount -a` / at boot; only on demand                |
| `user` / `users`    | Allow normal users to mount it (implies `noexec,nosuid,nodev`)       |
| `x-systemd.automount` | Mount on first access instead of at boot                           |
| `x-systemd.device-timeout=10s` | How long boot waits for the device                        |
| `uid=`, `gid=`, `umask=` | Owner/permissions for vfat, exfat, ntfs (no Unix permissions)   |
| `discard`           | Send TRIM continuously → [[block-devices]]                           |
| `subvol=`, `compress=` | btrfs specifics → [[btrfs]]                                       |
| `sync`              | Write immediately (slow; sometimes for USB sticks)                   |

Security hardening pattern for data/temporary mounts: `nosuid,nodev,noexec` (e.g. on `/tmp` → [[tmp]]).

# Bind mounts
A **bind mount** makes a directory visible at a second place. No new filesystem, same files.

```bash
sudo mount --bind /data/projects /home/alice/projects
sudo mount --bind -o remount,ro /data/projects /home/alice/projects   # read-only view
sudo mount --rbind /dev /mnt/chroot/dev     # recursive (include sub-mounts)
```
fstab: `/data/projects  /home/alice/projects  none  bind  0 0`

Used for: chroots, containers, exposing one directory to a service, read-only views.

# Propagation (shared / private)
`/proc/self/mountinfo` shows `shared:N` / `master:N`. Shared mounts propagate new sub-mounts to their peers (systemd makes `/` shared by default — that's how mounts appear inside containers/namespaces). Rarely needed outside containers: `mount --make-private /mnt`.

# systemd mount units
Every mount is also a unit: `/home` = `home.mount`, `/data/backup` = `data-backup.mount`.

```bash
systemctl list-units -t mount
systemctl status home.mount
systemd-escape -p --suffix=mount /data/backup   # data-backup.mount
```
fstab lines become units at boot via `systemd-fstab-generator`. You can also write `.mount` / `.automount` files directly → [[units]].

# User mounting with `udisksctl`
No root needed for removable/desktop disks (polkit decides):

```bash
udisksctl status
udisksctl mount -b /dev/sdb1        # → Mounted /dev/sdb1 at /run/media/alice/STICK
udisksctl unmount -b /dev/sdb1
udisksctl power-off -b /dev/sdb     # safe to unplug
udisksctl loop-setup -f image.iso   # mount an ISO without root
```

# Loop devices: mounting a file
```bash
sudo mount -o loop,ro image.iso /mnt
sudo losetup -fP --show disk.img    # → /dev/loop0 (+ loop0p1... for partitions)
losetup -l                          # list
sudo losetup -d /dev/loop0          # detach
```

# Scripting recipes

### Is a directory a mount point?
```bash
mountpoint -q /data && echo "mounted" || echo "not mounted"
```

### Fail early if a backup disk is missing
```bash
dst=/data/backup
findmnt -rno SOURCE --target "$dst" | grep -q '^/dev/sda1' || { echo "backup disk not mounted"; exit 1; }
```

### Mount options of a path as an array-friendly list
```bash
findmnt -no OPTIONS --target /home | tr ',' '\n'
```

### Read-only check
```bash
[[ ",$(findmnt -no OPTIONS --target /)," == *,ro,* ]] && echo "/ is read-only"
```

### Wait for a mount (e.g. network share) up to 30 s
```bash
for i in {1..30}; do mountpoint -q /mnt/share && break; sleep 1; done
```

# Find stuff

| Looking for                       | Where                                         |
| --------------------------------- | --------------------------------------------- |
| Everything mounted                | `findmnt`, `/proc/self/mountinfo`             |
| What mounts at boot               | `/etc/fstab`, `systemctl list-units -t mount` |
| USB sticks                        | `/run/media/$USER/`                           |
| Which process blocks unmount      | `fuser -vm /path`, `lsof +f -- /path`         |
| Mount errors                      | `journalctl -b -u data.mount`, `dmesg`        |

# See also
- [[storage]] · [[filesystems]] · [[btrfs]] · [[disk-usage]]
- [[fstab]] · [[mtab]] · [[units]] · [[systemd]]
- [[mnt]] · [[media]] · [[run]] · [[tmp]]
- `man 8 mount`, `man 8 findmnt`, `man 5 systemd.mount`
