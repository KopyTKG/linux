---
tags:
  - linux
  - storage
  - disk-usage
  - scripting
aliases:
  - Disk usage
  - df
  - du
  - Disk full
---
Two tools answer "how full is my disk", and they answer **different questions**:

| Tool | Asks                                      | Source                         | Speed   |
| ---- | ----------------------------------------- | ------------------------------ | ------- |
| `df` | How much does the **filesystem** say is used/free? | Superblock counters (instant) | Instant |
| `du` | How much do these **files** add up to?    | Walks the directory tree       | Slow on big trees |

When they disagree, that difference *is* the clue: deleted-but-open files, files hidden under a mount point, reserved blocks, snapshots, or compression.

# `df`

```bash
df -h                          # human sizes
df -hT                         # + filesystem type
df -h /home                    # only the filesystem holding /home
df -x tmpfs -x devtmpfs -h     # hide RAM filesystems
df -i                          # INODES instead of bytes
df --output=source,fstype,size,used,avail,pcent,target -h
```

```
$ df -hT -x tmpfs -x devtmpfs -x efivarfs
Filesystem     Type   Size  Used Avail Use% Mounted on
/dev/nvme0n1p3 btrfs  923G  412G  508G  45% /
/dev/nvme0n1p3 btrfs  923G  412G  508G  45% /home
/dev/nvme0n1p1 vfat  1022M  180M  843M  18% /boot
/dev/sda1      ext4   916G  610G  260G  71% /data
```

> [!note] `Size ≠ Used + Avail` on ext4
> ext4 reserves **5 %** for root by default (so the system can still log and recover when users fill the disk). On a 916 G data disk that's ~46 G you don't see in `Avail`.
> Check: `sudo tune2fs -l /dev/sda1 | grep 'Reserved block count'`. For pure data disks it can be lowered with `tune2fs -m 1` (harmless, but think before doing it on `/`).

# `du`

```bash
du -sh /var/log                        # one total
du -sh * .[!.]* 2>/dev/null            # each entry here, incl. dotfiles
du -h --max-depth=1 /var | sort -h     # one level deep, sorted
du -xh --max-depth=1 / 2>/dev/null | sort -h   # -x = stay on this filesystem
du -sh --apparent-size big.img         # logical size instead of blocks used
du -ah ~ | sort -rh | head -20         # biggest files and dirs in home
```

| Flag                | Meaning                                                    |
| ------------------- | ---------------------------------------------------------- |
| `-s`                | Summary only                                               |
| `-h`                | Human-readable                                             |
| `-x`                | Don't cross into other filesystems (skip `/proc`, other disks) |
| `-d N` / `--max-depth=N` | Depth                                                 |
| `-a`                | Files too, not only directories                            |
| `--apparent-size`   | File size as `ls` shows it, not disk blocks                |
| `-c`                | Grand total line                                           |
| `--exclude=PATTERN` | Skip                                                       |

> [!tip] `ls -l` vs `du`
> A **sparse file** (VM image, `truncate -s 10G x`) shows 10G in `ls -l` but maybe 0 in `du`. A 1-byte file shows `1` in `ls` but 4.0K in `du` (one block). On btrfs with compression `du` shows **uncompressed** sizes; use `compsize` for real usage → [[btrfs]].

If installed, **`ncdu`** gives an interactive browsable `du`: `ncdu -x /`.

# Disk full but `du` doesn't find it

### 1. Deleted files still open
A deleted file's space is only freed when **the last process closes it**. Classic case: a log file deleted while the service keeps writing.

```bash
sudo lsof +L1                      # open files with link count 0 (deleted)
```

Without lsof, straight from [[proc]]:
```bash
for fd in /proc/[0-9]*/fd/*; do
  t=$(readlink "$fd" 2>/dev/null) || continue
  [[ $t == *' (deleted)' ]] || continue
  [[ $t == /memfd:* || $t == /dev/shm/* ]] && continue   # RAM-only, not disk
  p=${fd#/proc/}; pid=${p%%/*}
  size=$(stat -L -c %s "$fd" 2>/dev/null)
  printf "%-7s %-15s %10s  %s\n" "$pid" "$(< /proc/$pid/comm)" "${size:-?}" "$t"
done 2>/dev/null | sort -k3 -rn | head
```
Fix: restart that process (`systemctl restart <service>`), or truncate through the fd: `: > /proc/<pid>/fd/<n>`.

### 2. Files hidden under a mount point
If `/data` had files *before* the disk was mounted there, they still use space on `/` but are invisible. Look underneath with a bind mount of `/`:
```bash
sudo mount --bind / /mnt && sudo du -sh /mnt/data && sudo umount /mnt
```

### 3. Out of inodes
`df -h` says free space, but you can't create files: **"No space left on device"**.
```bash
df -i                                          # IUse% 100% = out of inodes
# which directory has millions of small files:
sudo find / -xdev -type d -print0 2>/dev/null |
  xargs -0 -I{} sh -c 'echo "$(find "{}" -maxdepth 1 | wc -l) {}"' | sort -rn | head
```
Usual suspects: session/cache dirs, mail queues, build artefacts. btrfs and xfs allocate inodes dynamically and practically never hit this.

### 4. btrfs snapshots and metadata
`df` on btrfs is approximate. Snapshots keep old data alive; deleting a file doesn't free space if a snapshot still references it. Use `btrfs filesystem usage /` → [[btrfs]].

# Scripting recipes

### Warn when a filesystem is above a threshold
```bash
#!/usr/bin/env bash
limit=${1:-90}
df -P -x tmpfs -x devtmpfs -x efivarfs | awk -v l="$limit" 'NR>1 {
  use=$5; sub("%","",use)
  if (use+0 >= l) printf "WARNING %s is %s%% full (%s)\n", $6, use, $1
}'
```

### Free space of one path in bytes (for scripts)
```bash
df -B1 --output=avail /home/alice | tail -1
```

### Ensure enough space before copying
```bash
need=$(du -sb ./src | cut -f1)
have=$(df -B1 --output=avail /data | tail -1)
(( have > need )) || { echo "Not enough space on /data"; exit 1; }
```

### 10 biggest files on a filesystem
```bash
sudo find / -xdev -type f -size +100M -printf '%s\t%p\n' 2>/dev/null |
  sort -rn | head -10 | awk -F'\t' '{printf "%7.1f GiB  %s\n", $1/2^30, $2}'
```

### Biggest directories in home, one level
```bash
du -xh --max-depth=1 ~ 2>/dev/null | sort -rh | head -15
```

### Files not accessed for 1 year (candidates for cleanup)
```bash
find ~/Downloads -type f -atime +365 -printf '%k KiB\t%p\n' | sort -rn | head
```
(Only meaningful if the filesystem isn't mounted with `noatime`.)

### What grew today
```bash
sudo find / -xdev -type f -mtime -1 -size +50M -printf '%s\t%p\n' 2>/dev/null | sort -rn | head
```

# Find stuff

| Looking for               | Command                                        |
| ------------------------- | ---------------------------------------------- |
| Fullness of every FS      | `df -hT`                                       |
| Inode usage               | `df -i`                                        |
| Biggest dirs              | `du -xh --max-depth=1 / \| sort -h`, `ncdu -x /` |
| Deleted-but-open files    | `lsof +L1`, `/proc/*/fd` → `(deleted)`         |
| Logs eating space         | `du -sh /var/log/*`, `journalctl --disk-usage` → [[var]], [[journal]] |
| Caches                    | `du -sh ~/.cache /var/cache/*`                 |

# See also
- [[storage]] · [[filesystems]] · [[btrfs]] · [[mounting]]
- [[var]] — logs and caches grow here
- [[tmp]] — temporary files
- [[proc]] — `/proc/<pid>/fd`
