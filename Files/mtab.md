---
tags:
  - linux
  - files
  - filesystem
  - mount
aliases:
  - /etc/mtab
  - /proc/mounts
  - /proc/self/mounts
  - /proc/self/mountinfo
  - mountinfo
---
`/etc/mtab` ("mounted table") lists **what is mounted right now**. Long ago it was a real file that `mount` and `umount` updated by hand — and it was often wrong after a crash. Today it is just a symlink to the kernel's own list:

```
$ ls -l /etc/mtab
lrwxrwxrwx 1 root root 19 /etc/mtab -> ../proc/self/mounts
```

| File                     | Format                  | Who uses it                                     |
| ------------------------ | ----------------------- | ----------------------------------------------- |
| `/etc/fstab`             | what **should** be mounted at boot | you, `mount -a`, systemd → [[etc]]   |
| `/etc/mtab`              | symlink → `/proc/self/mounts` | old programs, `df`                        |
| `/proc/mounts`           | symlink → `/proc/self/mounts` | same                                      |
| `/proc/self/mounts`      | simple, fstab-like      | quick scripts                                   |
| `/proc/self/mountinfo`   | detailed, one line per mount | `findmnt`, `mount`, systemd, container tools |

> [!info] Why `self`?
> Mounts are per **mount namespace**. A container, a `systemd` service with `PrivateTmp=yes`, or a Flatpak app sees a different set of mounts. `/proc/self/...` always shows the view of the process reading it; `/proc/<pid>/mountinfo` shows another process's view. → [[proc]]

# `/proc/self/mounts`
Same six columns as `/etc/fstab`:

```
/dev/nvme0n1p3 / btrfs rw,noatime,compress=zstd:1,ssd,subvol=/@ 0 0
tmpfs /tmp tmpfs rw,noatime,inode64 0 0
/dev/nvme0n1p1 /boot vfat rw,relatime,fmask=0077,dmask=0077 0 0
```

| # | Field     | Example              |
| - | --------- | -------------------- |
| 1 | source    | `/dev/nvme0n1p3`, `tmpfs`, `proc` |
| 2 | mount point | `/`, `/tmp`        |
| 3 | fs type   | `btrfs`, `ext4`, `tmpfs`, `vfat` |
| 4 | options   | `rw,noatime,...`     |
| 5 | dump      | always `0`           |
| 6 | pass      | always `0`           |

> [!warning] Spaces are escaped
> A mount point with a space shows as `\040` (octal), tab as `\011`, newline `\012`, backslash `\134`. Example: `/run/media/user/My\040Disk`. Decode with `printf '%b' "$path"`. This applies to `mounts` and `mountinfo`.

# `/proc/self/mountinfo`
More information, especially for **bind mounts**, **btrfs subvolumes** and **containers**.

```
40 1 0:33 /@ / rw,noatime shared:1 - btrfs /dev/nvme0n1p3 rw,compress=zstd:1,ssd,subvol=/@
56 40 0:50 / /tmp rw,noatime shared:153 - tmpfs tmpfs rw,inode64
216 40 259:3 / /boot rw,relatime shared:183 - vfat /dev/nvme0n1p1 rw,fmask=0077,dmask=0077
```

| # | Field              | Example (`/` line)  | Meaning                                              |
| - | ------------------ | ------------------- | ---------------------------------------------------- |
| 1 | mount ID           | `40`                | unique ID of this mount                              |
| 2 | parent ID          | `1`                 | mount ID of the parent (builds a tree)               |
| 3 | major:minor        | `0:33`              | device number (`0:N` = virtual or btrfs)             |
| 4 | root               | `/@`                | which directory **inside the filesystem** is mounted (here the btrfs subvolume `@`; for bind mounts, the source dir) |
| 5 | mount point        | `/`                 | where it appears                                     |
| 6 | mount options      | `rw,noatime`        | per-mount options (VFS level)                        |
| 7… | optional fields   | `shared:1`          | propagation: `shared:N`, `master:N`, `unbindable`, or none (private) |
| – | separator          | `-`                 | marks end of optional fields                         |
| 8 | fs type            | `btrfs`             |                                                      |
| 9 | source             | `/dev/nvme0n1p3`    | device or `tmpfs`, `proc`, ...                       |
| 10 | super options     | `rw,compress=zstd:1,...` | filesystem-level options                        |

Because the number of optional fields varies, **find the `-` first**, then count from there.

> [!tip] mounts vs mountinfo
> - Need "is X mounted, which fs type?" → `mounts` is enough.
> - Need to know which subvolume / bind source / propagation → `mountinfo`.
> - Interactive use → just run `findmnt`.

# Useful commands
```bash
findmnt                       # tree of all mounts (reads mountinfo)
findmnt /home                 # one mount point
findmnt -no SOURCE,FSTYPE /   # /dev/nvme0n1p3[/@] btrfs   ([/@] = subvolume/bind root)
findmnt -t ext4,btrfs,xfs     # only real disk filesystems
findmnt --verify              # check /etc/fstab for mistakes
findmnt -D                    # like df
mount | column -t             # old style
```

# Scripting recipes

### Is a directory a mount point?
```bash
mountpoint -q /home && echo "/home is mounted"
# pure /proc version:
is_mounted() { awk -v m="$1" '$2==m {f=1} END{exit !f}' /proc/self/mounts; }
is_mounted /mnt/backup || { echo "backup disk not mounted" >&2; exit 1; }
```
Always do this before writing backups to a mount point — otherwise you fill the root filesystem with data meant for the external disk. → [[mnt]]

### Filesystem type of a path
```bash
awk '$2=="/" {print $3}' /proc/self/mounts      # btrfs
stat -f -c %T /home                              # works for any path, not just mount points
```

### Mount point, fs type and source from mountinfo
```bash
awk '{for(i=7;$i!="-";i++); print $5, $(i+1), $(i+2)}' /proc/self/mountinfo
```
```
/ btrfs /dev/nvme0n1p3
/tmp tmpfs tmpfs
/boot vfat /dev/nvme0n1p1
```

### Read-only mounts (e.g. after filesystem errors)
```bash
awk '$4 ~ /(^|,)ro(,|$)/ {print $2, $3}' /proc/self/mounts
```

### Wait until a USB disk appears
```bash
until findmnt -rno TARGET -S LABEL=Backup >/dev/null; do sleep 2; done
echo "mounted at $(findmnt -rno TARGET -S LABEL=Backup)"
```

### Compare mounts of a container/service with yours
```bash
diff <(awk '{print $5}' /proc/self/mountinfo | sort) \
     <(sudo awk '{print $5}' /proc/<pid>/mountinfo | sort)
```

# See also
- [[etc]] — `/etc/fstab`
- [[proc]] — `/proc/self`
- [[mnt]], [[media]] — where things get mounted
- [[dev]] — the block devices being mounted
- [[run]] — `/run/media/$USER`
- [[files]] — all file notes
- `man 5 proc` (**/proc/pid/mountinfo**), `man findmnt`, `man mount_namespaces`
