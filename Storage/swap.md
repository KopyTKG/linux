---
tags:
  - linux
  - storage
  - memory
  - swap
aliases:
  - Swap
  - zram
  - Swap file
  - swappiness
---
**Swap** is space the kernel can move rarely used memory pages to, freeing RAM for things that need it now. It is not "emergency RAM" only: a healthy system with plenty of RAM still swaps out idle pages so more RAM is available for file cache.

> [!info] What swap is good for
> - Moving out memory that was used once at startup and never touched again.
> - Surviving short memory spikes instead of the OOM killer ending a process → [[resource-limits]].
> - **Hibernation** (suspend to disk) — needs disk swap at least as big as used RAM.

# Three kinds of swap

| Kind              | Lives on                      | Speed       | Hibernation | Notes                                    |
| ----------------- | ----------------------------- | ----------- | ----------- | ---------------------------------------- |
| Swap partition    | Its own partition (`type swap`) | Disk speed | Yes         | Classic; fixed size → [[partitions]]     |
| Swap file         | A file on a normal filesystem | Disk speed  | Yes (needs `resume_offset`) | Easy to resize. btrfs needs a no-CoW file |
| **zram**          | Compressed block device **in RAM** | RAM speed | **No**   | Compresses ~2–4×; great default on desktops |
| zswap             | Compressed RAM **cache in front of** disk swap | — | Yes | Kernel feature, not a device |

Multiple swaps can be active; the kernel uses higher **priority** first. Typical combo: zram (priority 100) + disk swap (priority -2) for overflow and hibernation.

# Looking at swap

```bash
swapon --show                 # active swap devices
cat /proc/swaps               # same, raw (sizes in KiB)
free -h                       # RAM + swap totals
grep -E 'Swap(Total|Free|Cached)' /proc/meminfo     # → [[proc]]
```

```
$ swapon --show
NAME           TYPE       SIZE USED PRIO
/dev/zram0     partition   16G 128M  100
/dev/nvme0n1p2 partition    8G   0B   -2
```

`/proc/swaps`:
```
Filename          Type       Size      Used    Priority
/dev/zram0        partition  16777212  131072  100
/dev/nvme0n1p2    partition  8388604   0       -2
```

Per process: `VmSwap` in `/proc/<pid>/status`.

# swappiness
`/proc/sys/vm/swappiness` (0–200, kernel default **60**) sets how eagerly the kernel swaps **anonymous memory** compared to dropping **file cache**.

| Value    | Behaviour                                                  |
| -------- | ---------------------------------------------------------- |
| `0`–`10` | Avoid swapping, drop cache first (servers with lots of RAM) |
| `60`     | Default balance                                            |
| `100`+   | Prefer swapping; sensible with fast **zram** (often 150–180) |

```bash
cat /proc/sys/vm/swappiness
sysctl vm.swappiness
echo 10 | sudo tee /proc/sys/vm/swappiness                  # live, lost on reboot
echo 'vm.swappiness = 10' | sudo tee /etc/sysctl.d/99-swap.conf   # permanent → [[sysctl-d]]
```

# zram
```bash
cat /sys/block/zram0/comp_algorithm     # [zstd] = active algorithm
cat /sys/block/zram0/disksize           # configured size in bytes
cat /sys/block/zram0/mm_stat            # orig_data_size compr_data_size mem_used_total ...
zramctl                                 # nice summary
```

```
$ zramctl
NAME       ALGORITHM DISKSIZE   DATA  COMPR TOTAL STREAMS MOUNTPOINT
/dev/zram0 zstd           16G   512M   128M  134M      16 [SWAP]
```
Here 512 MiB of swapped pages use only 134 MiB of real RAM.

Usually set up by a systemd generator (`zram-generator`, config in `/etc/systemd/zram-generator.conf`) or a distro service.

# Creating swap

> [!danger] `mkswap` erases the target
> `mkswap` on a partition destroys whatever filesystem was there. Check the device with `lsblk -f` first.
> ```bash
> # swap partition
> sudo mkswap -L swap /dev/sdX2
> sudo swapon /dev/sdX2
> # fstab: UUID=7d2e4f6a-1b3c-4d5e-8f9a-0b1c2d3e4f5a  none  swap  defaults  0 0
>
> # swap file (ext4/xfs)
> sudo fallocate -l 8G /swapfile        # (btrfs: sudo btrfs filesystem mkswapfile --size 8G /swap/swapfile)
> sudo chmod 600 /swapfile
> sudo mkswap /swapfile
> sudo swapon /swapfile
> # fstab: /swapfile  none  swap  defaults  0 0
>
> sudo swapoff /swapfile                # turn off (pages move back to RAM — needs enough free RAM!)
> ```

> [!warning] Swap file permissions
> A swap file contains raw memory pages — passwords, keys, anything. It **must** be `600 root:root`. `swapon` warns if it isn't.

# Hibernation needs
1. Disk swap ≥ used RAM (+ a bit).
2. Kernel parameter `resume=UUID=<swap UUID>` (and `resume_offset=` for a swap file) → [[boot]].
3. The initramfs must include the resume hook (distro-specific tool).
4. `systemctl hibernate`.

```bash
cat /sys/power/state           # "disk" listed = hibernation possible
cat /sys/power/resume          # major:minor of resume device (0:0 = not set)
grep -o 'resume=[^ ]*' /proc/cmdline
```

# Scripting recipes

### Swap usage summary
```bash
awk 'NR>1 {s+=$3; u+=$4} END {printf "swap %d MiB, used %d MiB (%.1f%%)\n", s/1024, u/1024, s ? u*100/s : 0}' /proc/swaps
```

### Top processes by swap usage
```bash
for d in /proc/[0-9]*; do
  awk -v pid="${d#/proc/}" '/^Name:/{n=$2} /^VmSwap:/{s=$2} END{if (s>0) print s, pid, n}' "$d/status" 2>/dev/null
done | sort -rn | head -10 | awk '{printf "%8.1f MiB  %-7s %s\n", $1/1024, $2, $3}'
```

### zram compression ratio
```bash
read -r orig compr used _ < /sys/block/zram0/mm_stat
(( compr > 0 )) && echo "ratio $(( orig / compr )):1, RAM used $(( used / 1024**2 )) MiB"
```

### Warn if no swap at all
```bash
[[ $(awk 'NR>1' /proc/swaps | wc -l) -eq 0 ]] && echo "No swap configured"
```

# Find stuff

| Looking for                 | Where                                          |
| --------------------------- | ---------------------------------------------- |
| Active swap                 | `swapon --show`, `/proc/swaps`                 |
| Swap at boot                | `/etc/fstab` swap lines, `systemctl list-units -t swap` |
| swappiness                  | `/proc/sys/vm/swappiness`, `/etc/sysctl.d/`    |
| zram                        | `zramctl`, `/sys/block/zram0/`                 |
| Hibernation resume device   | `/proc/cmdline`, `/sys/power/resume`           |
| Who is swapped out          | `VmSwap` in `/proc/<pid>/status`               |

# See also
- [[storage]] · [[partitions]] · [[block-devices]] · [[btrfs]]
- [[proc]] — `/proc/swaps`, `/proc/meminfo`, `/proc/sys/vm`
- [[resource-limits]] — OOM killer
- [[fstab]] · [[sysctl-d]] · [[boot]]
- `man 8 swapon`, `man 8 mkswap`, kernel docs: `admin-guide/blockdev/zram`
