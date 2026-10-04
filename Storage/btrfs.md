---
tags:
  - linux
  - storage
  - filesystem
  - btrfs
aliases:
  - btrfs
  - Subvolumes
  - Snapshots
---
**btrfs** ("B-tree FS", say "butter FS") is a **copy-on-write** filesystem with built-in subvolumes, snapshots, compression, checksums and multi-device support. It's part of the mainline kernel and a common default for desktop installs.

> [!info] Copy-on-write in one sentence
> btrfs never overwrites data in place: a change is written to a new place, then the pointers switch over. That makes **snapshots almost free** (they just keep the old pointers) and a crash can't leave half-written blocks, but it fragments files that are rewritten in place a lot (VM images, databases).

# Features

| Feature             | What it gives you                                                       |
| ------------------- | ----------------------------------------------------------------------- |
| Subvolumes          | Independent directory trees inside one filesystem, mountable separately |
| Snapshots           | Instant point-in-time copy of a subvolume (read-only or writable)       |
| Checksums           | Every data and metadata block is checksummed; silent corruption is detected |
| Compression         | Transparent `zstd`, `lzo`, `zlib` per mount or per file                 |
| Multi-device / RAID | One filesystem across several disks (RAID1/10 solid; RAID5/6 not recommended) |
| Send/receive        | Stream differences between snapshots to another disk/machine (backups)  |
| Online resize       | Grow and shrink while mounted                                           |
| Dynamic inodes      | Never runs out of inodes                                                |

# Subvolumes
A subvolume looks like a directory but behaves like its own filesystem root: it can be mounted on its own, snapshotted on its own, and excluded from snapshots of its parent.

Common flat layout (top level = subvolume id 5, names starting with `@` are a convention):

| Subvolume     | Mounted at    | Why separate                                     |
| ------------- | ------------- | ------------------------------------------------ |
| `@`           | `/`           | System; snapshot before updates                  |
| `@home`       | `/home`       | User data; roll back system without losing it    |
| `@log`        | `/var/log`    | Keep logs when rolling back `/`                  |
| `@cache`      | `/var/cache`  | Don't waste snapshot space on caches             |
| `@tmp`        | `/var/tmp`    | Same                                             |
| `@snapshots`  | `/.snapshots` | Where snapshots are stored                       |

fstab lines differ only in `subvol=`:
```
UUID=3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c  /      btrfs  subvol=/@,noatime,compress=zstd:3      0 0
UUID=3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c  /home  btrfs  subvol=/@home,noatime,compress=zstd:3  0 0
```

```bash
findmnt -t btrfs -o TARGET,SOURCE        # [/@home] after the device = subvolume
sudo btrfs subvolume list /              # all subvolumes with IDs
sudo btrfs subvolume show /home          # details of one
btrfs subvolume get-default /            # what mounts without subvol= (may need root)
```

```
$ sudo btrfs subvolume list /
ID 256 gen 48211 top level 5 path @
ID 257 gen 48211 top level 5 path @home
ID 258 gen 48205 top level 5 path @log
ID 259 gen 48210 top level 5 path @cache
ID 260 gen 48190 top level 5 path @tmp
```

# Snapshots

```bash
sudo btrfs subvolume snapshot -r / /.snapshots/root-2026-10-04     # read-only snapshot
sudo btrfs subvolume snapshot /home /home-test                     # writable copy
sudo btrfs subvolume delete /.snapshots/root-2026-10-04
```

> [!warning] Snapshots are not backups
> A snapshot lives on the **same disk**. Disk dies → snapshots die. Use `btrfs send | btrfs receive` (or any backup tool) to another device for real backups.

> [!note] Snapshots hold space
> Deleting a file that a snapshot still contains frees nothing. If the disk fills up mysteriously, list and delete old snapshots first.

# Space: why `df` lies a little
btrfs allocates space in **chunks** (data, metadata, system) and each can have a different RAID profile and compression. `df` can only estimate.

```bash
sudo btrfs filesystem usage /       # the real picture
btrfs filesystem df /               # per chunk type
sudo btrfs filesystem show          # devices and sizes
sudo compsize /home                 # real size after compression (if installed)
```

```
$ sudo btrfs filesystem usage /
Overall:
    Device size:                 922.52GiB
    Device allocated:            430.02GiB
    Device unallocated:          492.50GiB
    Used:                        411.84GiB
    Free (estimated):            508.37GiB      (min: 262.12GiB)
Data,single: Size:420.01GiB, Used:404.12GiB (96.22%)
Metadata,DUP: Size:5.00GiB, Used:3.86GiB (77.20%)
System,DUP: Size:8.00MiB, Used:64.00KiB (0.78%)
```

"No space left" while `df` shows free space usually means **metadata** chunks are full and no unallocated space is left → a filtered balance can help (below).

# Compression
Mount option `compress=zstd:3` (level 1–15) or `compress-force=...`. Applies to **newly written** data.

```bash
findmnt -no OPTIONS / | tr ',' '\n' | grep compress
btrfs property get /home/alice/file compression     # per file
btrfs property set ~/vm compression zstd             # per dir (new files)
```

# No-CoW for VM images and databases
Files that are rewritten in place (VM disks, databases, torrents) fragment heavily under CoW. Disable CoW for a **new empty** directory; files created inside inherit it:

```bash
mkdir ~/vms && chattr +C ~/vms        # → [[attributes]]
lsattr -d ~/vms                       # ---------------C------ /home/alice/vms
```
`+C` on an existing non-empty file has no effect on its existing data. No-CoW files also lose checksums and compression.

# Maintenance

| Task      | What it does                                                         | How often         |
| --------- | -------------------------------------------------------------------- | ----------------- |
| **scrub** | Reads all data, verifies checksums, repairs from a mirror if RAID1   | Monthly           |
| **balance** | Rewrites chunks to reclaim/redistribute space or convert RAID profile | Only when needed |
| **defrag** | Rewrites fragmented files                                           | Rarely; **breaks snapshot sharing** (duplicates data) |
| **trim**  | `fstrim` / `discard=async`                                           | Automatic         |

```bash
sudo btrfs scrub status /         # read-only: last scrub result
sudo btrfs device stats /         # read-only: error counters per device
```

> [!danger] Long-running / risky operations
> These rewrite large parts of the filesystem. Run them only when you know why, with a backup, and not on a nearly full disk without reading the docs first:
> ```bash
> sudo btrfs scrub start /                       # safe but heavy I/O for a long time
> sudo btrfs balance start -dusage=50 /          # rewrite data chunks under 50 % full
> sudo btrfs balance start -dconvert=raid1 -mconvert=raid1 /   # change RAID profile
> sudo btrfs filesystem defragment -r /home      # duplicates data shared with snapshots
> sudo btrfs filesystem resize -50G /            # shrink (partition must be shrunk separately!)
> sudo btrfs check --repair /dev/sdX             # LAST resort, on unmounted FS, can make things worse
> ```

# Scripting recipes

### Is a path on btrfs?
```bash
[[ $(stat -f -c %T /home) == btrfs ]] && echo "btrfs"
```

### Which subvolume is a path in?
```bash
findmnt -no SOURCE --target /home/alice | sed -n 's/.*\[\(.*\)\]/\1/p'    # /@home
```

### Timestamped read-only snapshot of /home (run as root)
```bash
#!/usr/bin/env bash
set -euo pipefail
dst=/.snapshots
mkdir -p "$dst"
btrfs subvolume snapshot -r /home "$dst/home-$(date +%F_%H%M)"
# keep the newest 7
ls -1d "$dst"/home-* | head -n -7 | xargs -r -n1 btrfs subvolume delete
```

### Report device errors (for a timer/cron)
```bash
sudo btrfs device stats / | awk '$2 != 0 {print "ERROR:", $0; bad=1} END {exit bad}'
```

# Find stuff

| Looking for                  | Command                                       |
| ---------------------------- | --------------------------------------------- |
| Subvolumes                   | `sudo btrfs subvolume list /`                 |
| Which subvolume is mounted   | `findmnt -t btrfs`                            |
| Real free space              | `sudo btrfs filesystem usage /`               |
| Compression ratio            | `sudo compsize /`                             |
| Errors                       | `sudo btrfs device stats /`, `journalctl -k \| grep -i btrfs` |
| Kernel btrfs info            | `/sys/fs/btrfs/<UUID>/`                       |

# See also
- [[storage]] · [[filesystems]] · [[mounting]] · [[disk-usage]]
- [[attributes]] — `chattr +C`
- [[fstab]] — `subvol=` lines
- [[sys]] — `/sys/fs/btrfs`
- `man 5 btrfs`, `man 8 btrfs-subvolume`, <https://btrfs.readthedocs.io>
