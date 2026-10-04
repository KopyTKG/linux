---
tags:
  - linux
  - storage
  - disks
  - performance
aliases:
  - Disk health
  - SMART
  - I/O
  - diskstats
  - iostat
---
Two questions about disks come up all the time: **is it dying?** (SMART health data) and **why is it slow / who is hammering it?** (I/O statistics). The kernel exposes I/O counters in [[proc]] and [[sys]]; health data comes from the drive itself.

# Disk health: SMART
Drives keep self-monitoring data (**S.M.A.R.T.**): reallocated sectors, temperature, wear level, error logs. Reading it needs root because it talks to the device directly.

```bash
sudo smartctl -i /dev/sda          # identity: model, serial, firmware, SMART supported?
sudo smartctl -H /dev/sda          # overall verdict: PASSED / FAILED
sudo smartctl -a /dev/sda          # everything
sudo smartctl -a /dev/nvme0        # NVMe works too
sudo smartctl -t short /dev/sda    # start a 2-minute self-test (safe, runs in background)
sudo smartctl -l selftest /dev/sda # self-test results
```

### SATA attributes that matter

| ID  | Attribute                 | Bad sign                                         |
| --- | ------------------------- | ------------------------------------------------ |
| 5   | Reallocated_Sector_Ct     | > 0 and growing → disk is remapping bad sectors  |
| 187 | Reported_Uncorrect        | > 0 → read errors the drive couldn't fix         |
| 197 | Current_Pending_Sector    | > 0 → sectors waiting to be remapped (unreadable) |
| 198 | Offline_Uncorrectable     | > 0 → bad sectors found during scans             |
| 199 | UDMA_CRC_Error_Count      | Growing → **cable/connection** problem, not the disk |
| 194 | Temperature_Celsius       | HDD > 50 °C for long periods                     |
| 9   | Power_On_Hours            | Age                                              |
| 177/231/233 | Wear_Leveling / SSD_Life_Left / Media_Wearout | SSD wear (vendor-specific) |

### NVMe health
```bash
sudo nvme smart-log /dev/nvme0     # nvme-cli
```
```
critical_warning                        : 0
temperature                             : 38 °C
available_spare                         : 100%
percentage_used                         : 3%
data_units_written                      : 41 234 567 (21.11 TB)
media_errors                            : 0
num_err_log_entries                     : 12
```
Watch `critical_warning` (must be 0), `percentage_used` (wear, 100 % = rated life used), `media_errors` (must be 0).

Without root: temperature via hwmon → [[sys]]:
```bash
for h in /sys/class/hwmon/hwmon*; do
  [[ $(< "$h/name") == nvme ]] && echo "NVMe: $(( $(< "$h/temp1_input") / 1000 )) °C"
done
```

> [!tip] Automate it
> `smartd` (from smartmontools) runs as a service, checks drives periodically and logs or mails warnings. Config: `/etc/smartd.conf`. Check its findings with `journalctl -u smartd` → [[journal]].

# I/O statistics: `/proc/diskstats`
Counters since boot, one line per device and partition. Same numbers per device in `/sys/block/<dev>/stat` (without the first 3 columns).

```
 259       0 nvme0n1 112733 13893 9798509 55465 20060 550 918593 16716 0 27337 72921 1412 0 1019872 448 382 290
```

| Field | Name                   | Meaning                                                |
| ----- | ---------------------- | ------------------------------------------------------ |
| 1–3   | major minor name       | Device                                                 |
| 4     | reads completed        | Number of read requests                                |
| 5     | reads merged           | Adjacent reads merged into one                         |
| 6     | **sectors read**       | × 512 = bytes read                                     |
| 7     | ms reading             | Total time spent on reads                              |
| 8     | writes completed       |                                                        |
| 9     | writes merged          |                                                        |
| 10    | **sectors written**    | × 512 = bytes written                                  |
| 11    | ms writing             |                                                        |
| 12    | I/Os in progress       | Right now (the only non-counter)                       |
| 13    | **ms doing I/O**       | Time device was busy → utilisation %                   |
| 14    | weighted ms doing I/O  | Busy time × queue length                               |
| 15–18 | discards               | completed, merged, sectors, ms (TRIM)                  |
| 19–20 | flushes                | completed, ms                                          |

> [!note] Sectors are always 512 bytes here
> Even on 4K-sector disks the kernel reports in 512-byte units.

Utilisation % = Δ(field 13) / Δ(wall time in ms) × 100. Near 100 % on an HDD = saturated. On NVMe (many parallel queues) 100 % doesn't necessarily mean saturated.

# Tools

| Tool           | Shows                                                    |
| -------------- | -------------------------------------------------------- |
| `iostat -xz 1` | Per-device throughput, IOPS, latency (`r_await`), `%util` (sysstat) |
| `pidstat -d 1` | Read/write KB/s **per process** (sysstat)                |
| `iotop`        | top-like, per process/thread I/O (root)                  |
| `vmstat 1`     | Column `wa` = CPU time waiting for I/O; `bi`/`bo` = blocks in/out |
| `/proc/<pid>/io` | Bytes a process read/wrote → [[proc]]                  |
| `/proc/pressure/io` | PSI: % of time tasks stalled on I/O → [[proc]]      |

```bash
cat /proc/pressure/io
# some avg10=0.00 avg60=0.05 avg300=0.06 total=19324442
# full avg10=0.00 avg60=0.04 avg300=0.05 total=17855587
```
`full` > a few % sustained = the system is really waiting on disk.

# I/O schedulers
Each device has a request queue with a scheduler that orders requests.

```bash
cat /sys/block/nvme0n1/queue/scheduler    # none mq-deadline kyber [bfq]   ([x] = active)
cat /sys/block/sda/queue/scheduler
```

| Scheduler     | Good for                                                 |
| ------------- | -------------------------------------------------------- |
| `none`        | Fast NVMe: let the device handle ordering (lowest overhead) |
| `mq-deadline` | General purpose, SSD/HDD servers                         |
| `bfq`         | Desktops with HDDs: fair, responsive under load          |
| `kyber`       | Fast devices, latency targets                            |

Change live (lost on reboot): `echo mq-deadline | sudo tee /sys/block/sda/queue/scheduler`. Make permanent with a udev rule:
```
# /etc/udev/rules.d/60-ioscheduler.rules
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
ACTION=="add|change", KERNEL=="nvme[0-9]*n[0-9]*", ATTR{queue/scheduler}="none"
```

Per-process I/O priority: `ionice` → [[priority-scheduling]].

# Kernel error messages
Disk trouble usually shows in the kernel log first:
```bash
journalctl -k -p warning | grep -Ei 'ata|nvme|I/O error|blk_update|reset'
sudo dmesg -T | grep -Ei 'I/O error|medium error|link reset'    # → [[dmesg-kmsg]]
```
`I/O error, dev sda, sector ...` repeated = take a backup **now**.

# Scripting recipes

### Read/write speed of a disk (iostat-like)
```bash
#!/usr/bin/env bash
dev=${1:-nvme0n1}
snap() { awk -v d="$dev" '$3==d {print $6, $10, $13}' /proc/diskstats; }
read -r r1 w1 b1 < <(snap); sleep 1; read -r r2 w2 b2 < <(snap)
printf "%s  read %6d KiB/s  write %6d KiB/s  busy %3d%%\n" \
  "$dev" $(( (r2-r1)/2 )) $(( (w2-w1)/2 )) $(( (b2-b1)/10 ))
```
(sectors × 512 / 1024 = sectors / 2 KiB; busy ms per 1000 ms / 10 = %).

### Continuous monitor for all disks
```bash
declare -A r w                      # previous counters per device
while :; do
  while read -r _ _ n _ _ rs _ _ _ ws _; do
    [[ -e /sys/block/$n/device ]] || continue
    printf "%-9s R %6d KiB/s  W %6d KiB/s\n" "$n" $(( (rs-${r[$n]:-$rs})/2 )) $(( (ws-${w[$n]:-$ws})/2 ))
    r[$n]=$rs; w[$n]=$ws
  done < /proc/diskstats
  sleep 1; echo
done
```

### Total data written since boot per disk
```bash
awk '{ if (system("test -e /sys/block/" $3 "/device") == 0)
         printf "%-9s %8.1f GiB read  %8.1f GiB written\n", $3, $6*512/2^30, $10*512/2^30 }' /proc/diskstats
```

### Top processes by bytes written (needs permission to read their io)
```bash
for d in /proc/[0-9]*; do
  wb=$(awk '/^write_bytes:/ {print $2}' "$d/io" 2>/dev/null) || continue
  (( wb > 0 )) && echo "$wb ${d#/proc/} $(< "$d/comm")"
done 2>/dev/null | sort -rn | head | awk '{printf "%8.1f MiB  %-7s %s\n", $1/2^20, $2, $3}'
```

### SMART verdict for all disks (root)
```bash
for d in /dev/sd? /dev/nvme?; do
  [[ -e $d ]] && printf "%-12s %s\n" "$d" "$(smartctl -H "$d" | grep -Eo 'PASSED|FAILED|OK' | head -1)"
done
```

# Find stuff

| Looking for                | Where                                             |
| -------------------------- | ------------------------------------------------- |
| Disk health                | `smartctl -a`, `nvme smart-log`                   |
| Temperatures               | `/sys/class/hwmon/*/temp*_input` → [[sys]]        |
| Raw I/O counters           | `/proc/diskstats`, `/sys/block/<dev>/stat`        |
| Who does I/O               | `iotop`, `pidstat -d`, `/proc/<pid>/io`           |
| Is the system I/O-bound    | `/proc/pressure/io`, `vmstat` `wa` column         |
| Scheduler                  | `/sys/block/<dev>/queue/scheduler`                |
| Disk errors                | `journalctl -k`, `dmesg`                          |

# See also
- [[storage]] · [[block-devices]] · [[disk-usage]]
- [[proc]] — `/proc/diskstats`, `/proc/<pid>/io`, `/proc/pressure/io`
- [[sys]] — `/sys/block`, hwmon
- [[priority-scheduling]] — `ionice`
- [[journal]] · [[dmesg-kmsg]] — kernel errors
- `man 8 smartctl`, `man 1 iostat`, kernel docs `admin-guide/iostats`
