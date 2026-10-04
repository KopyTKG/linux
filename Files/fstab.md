---
tags:
  - linux
  - files
  - etc
  - disks
aliases:
  - /etc/fstab
  - fstab
---
`/etc/fstab` (filesystem table) lists **what gets mounted where at boot**. systemd turns each line into a `.mount` unit (`systemd-fstab-generator`), `mount -a` mounts everything listed, and `mount /games` works with only the mount point because the rest is here.

> [!info] Quick facts
> - Path: `/etc/fstab` · mode `0644`
> - One line per filesystem, 6 columns separated by spaces/tabs
> - Read at boot and by `mount`, `umount`, `findmnt`, `swapon -a`
> - Manual: `man 5 fstab`, `man 8 mount` (all options)

> [!danger] A broken fstab can stop the boot
> If a non-`nofail` line can't mount, boot drops into **emergency mode**. After every edit run
> ```bash
> sudo findmnt --verify      # syntax + do devices exist
> sudo systemctl daemon-reload && sudo mount -a   # actually try it
> ```
> before you reboot.

# The 6 columns

```
UUID=3f6c9e2a-...   /home   btrfs   noatime,compress=zstd:1   0 0
```

| # | Column      | Example             | Meaning                                                     |
| - | ----------- | ------------------- | ----------------------------------------------------------- |
| 1 | device      | `UUID=3f6c9e2a-...` | What to mount (see next table)                              |
| 2 | mount point | `/home`             | Where. `swap` / `none` for swap                             |
| 3 | type        | `btrfs`             | `ext4`, `xfs`, `btrfs`, `vfat`, `ntfs3`, `tmpfs`, `nfs`, `swap`, `auto` |
| 4 | options     | `noatime,compress`  | Comma separated, no spaces. `defaults` if nothing special   |
| 5 | dump        | `0`                 | Old backup tool flag. Always `0`                            |
| 6 | pass        | `0`                 | `fsck` order at boot: `1` = root, `2` = others, `0` = never |

### Ways to name the device

| Form                  | Example                        | Stable?                                    |
| --------------------- | ------------------------------ | ------------------------------------------ |
| `UUID=`               | `UUID=ABCD-1234`               | Yes. Best choice. `blkid` / `lsblk -f` show it |
| `LABEL=`              | `LABEL=Work`                   | Yes, if you keep labels unique             |
| `PARTUUID=`           | `PARTUUID=...`                 | Yes (GPT partition ID, survives reformat)  |
| `/dev/disk/by-id/...` | `/dev/disk/by-id/nvme-...-part1` | Yes                                      |
| `/dev/sdb1`           | `/dev/sdb1`                    | **No**. Names change when disks are added/removed → [[dev]] |
| `tmpfs`, `server:/path` | `tmpfs`                      | Virtual / network                          |

# Example
```
UUID=ABCD-1234       /boot       vfat   umask=0077                              0 2
UUID=7d2e4f6a-...    swap        swap   defaults                                0 0
UUID=3f6c9e2a-...    /           btrfs  subvol=/@,noatime,compress=zstd:1       0 0
UUID=3f6c9e2a-...    /var/log    btrfs  subvol=/@log,noatime,compress=zstd:1    0 0
UUID=9b8a7c6d-...    /home       btrfs  noatime,compress=zstd:1                 0 0
UUID=1c2d3e4f-...    /data       xfs    lazytime,noatime,inode64,logbsize=256k  0 2
tmpfs                /tmp        tmpfs  noatime,mode=1777                       0 0
LABEL=Work           /work       xfs    nofail,users,exec                       0 0
```
Same UUID on several lines = one btrfs filesystem, different **subvolumes** (`subvol=`) mounted in different places.

# Options you will meet

| Option              | Meaning                                                                 |
| ------------------- | ----------------------------------------------------------------------- |
| `defaults`          | `rw,suid,dev,exec,auto,nouser,async`                                    |
| `ro` / `rw`         | Read-only / read-write                                                  |
| `noatime`           | Don't write "last access time" on every read. Faster, less SSD wear     |
| `relatime`          | Default: atime only updated sometimes                                   |
| `lazytime`          | Keep timestamps in RAM, write them later                                |
| `nofail`            | **Boot continues** if the device is missing. Use for USB/extra disks    |
| `noauto`            | Don't mount at boot, only on `mount /point`                             |
| `user` / `users`    | Normal users may mount it (`users`: anyone may unmount too)             |
| `exec` / `noexec`   | Allow / forbid running programs from it                                 |
| `nosuid`, `nodev`   | Ignore setuid bits / device files. Good for removable and `/tmp`        |
| `x-systemd.automount` | Mount on first access instead of at boot                              |
| `x-systemd.device-timeout=5s` | How long to wait for the device                               |
| `_netdev`           | Network filesystem: wait for network                                    |
| `uid=`,`gid=`,`umask=` | Owner/permissions for filesystems without Unix permissions (vfat, ntfs) |
| `subvol=`, `compress=` | btrfs subvolume, transparent compression                            |
| `mode=1777`         | tmpfs permissions (sticky, world writable)                              |

> [!tip] `users` implies `noexec,nosuid,nodev`
> That's why the `/work` line adds `exec` back after `users`. Order matters: later options win.

# Scripting recipes

### fstab without comments, readable
```bash
awk '!/^#/ && NF {printf "%-12s %-6s %s\n", $2, $3, $4}' /etc/fstab
```

### Check every UUID in fstab exists
```bash
awk '!/^#/ && $1 ~ /^UUID=/ {sub("UUID=", "", $1); print $1}' /etc/fstab | sort -u |
while read -r u; do
  [[ -e /dev/disk/by-uuid/$u ]] && echo "$u present" || echo "$u MISSING"
done
```

### Which fstab entries are not mounted right now
```bash
awk '!/^#/ && NF && $2 ~ /^\// {print $2}' /etc/fstab | while read -r m; do
  findmnt -n "$m" >/dev/null && echo "mounted     $m" || echo "NOT mounted $m"
done
```

### Line for a new disk (copy-paste ready)
```bash
dev=/dev/sdb1
printf 'UUID=%s  /mnt/data  %s  defaults,nofail  0 2\n' \
  "$(lsblk -no UUID "$dev")" "$(lsblk -no FSTYPE "$dev")"
```

### See systemd's units made from fstab
```bash
systemctl list-units --type=mount
systemctl cat home.mount
```

# See also
- [[mtab]] — what is mounted **now** (fstab = what **should** be)
- [[crypttab]] — encrypted volumes unlocked before fstab is processed
- [[dev]] · [[dev-special-files]] — device names and `/dev/disk/by-*`
- [[etc]] · [[boot]] · [[systemd]] · [[proc]] (`/proc/mounts`)
