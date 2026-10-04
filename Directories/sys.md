---
tags:
  - linux
  - filesystem
  - sys
  - hardware
  - scripting
aliases:
  - sysfs
  - /sys
---
`/sys` is where the kernel shows its view of **hardware, drivers and kernel objects**: every device, bus, driver, module and cgroup appears as a folder, and each of its properties is a small file.

> [!info] What `/sys` really is
> `/sys` is a virtual filesystem (`sysfs`). Like [[proc]] it is not on disk; the kernel generates it when you read it.
> - **One value per file.** No tables to parse: `cat /sys/class/net/enp3s0/speed` → `1000`.
> - Folders mirror the kernel's internal **device tree**. The same device shows up in several places through symlinks.
> - Some files are writable (as root) and change the hardware or driver **live**, and the change is **lost on reboot**.

> [!question] `/proc` vs `/sys`
> | | [[proc]] | `/sys` |
> |---|---|---|
> | Main topic | **Processes** (+ old system info) | **Devices, drivers, kernel objects** |
> | Format | Mixed: tables, `key: value`, `\0` lists | One value per file |
> | Age | Old, grown messy | Newer (2.6 kernel), strict rules |
> | Settings | `/proc/sys/*` (`sysctl`) | Device/driver attributes, cgroups |
>
> Rule of thumb: *"what is running?"* → `/proc`. *"what hardware do I have and how is it configured?"* → `/sys`.

> [!tip] Golden rules for scripting with `/sys`
> 1. Read with `$(< file)` (fast, no `cat` process). Values usually have no units in the file, so check what unit it is (often milli-units: `56500` = 56.5 °C).
> 2. Device numbers like `hwmon3`, `card1` and `cooling_device7` **can change between boots**. Find the device by its `name`, not by its number.
> 3. Some files are not readable (`Permission denied`) or return an error (`Invalid argument`) when the device is off. Use `2>/dev/null`.
> 4. Paths under `/sys/devices` are real; everything in `/sys/class` and `/sys/block` is a **symlink** into it. Use `readlink -f` to see where a device physically sits.
> 5. `sudo echo x > /sys/...` doesn't work, use `echo x | sudo tee /sys/...`. To make it permanent use a udev rule, `tmpfiles.d` or a module option in [[etc]].

# Top level

| Folder          | What's inside                                                         |
| --------------- | --------------------------------------------------------------------- |
| `/sys/devices`  | **The real tree** of all devices, ordered by how they are connected   |
| `/sys/class`    | Devices grouped by **type** (net, block, hwmon...). Symlinks into `devices` |
| `/sys/block`    | Block devices (disks). Symlinks into `devices`                        |
| `/sys/bus`      | Devices grouped by **bus** (pci, usb, i2c...) + their drivers         |
| `/sys/dev`      | Devices by `major:minor` number (`char/`, `block/`) → [[dev]]         |
| `/sys/module`   | Loaded kernel modules and their parameters                            |
| `/sys/firmware` | ACPI tables, EFI variables, DMI                                       |
| `/sys/fs`       | Filesystem info: **cgroup**, btrfs, ext4, fuse, bpf                   |
| `/sys/kernel`   | Kernel internals: `mm/`, `debug/`, `security/`, `tracing/`            |
| `/sys/power`    | Suspend/hibernate (`state`, `disk`, `mem_sleep`)                      |
| `/sys/hypervisor` | Info when running inside a VM                                       |

# `/sys/devices` — the device tree
Each device sits under the bus that connects it. For example, the Ethernet card:

```bash
readlink -f /sys/class/net/enp3s0
# /sys/devices/pci0000:00/0000:00:1c.0/0000:03:00.0/net/enp3s0
```
Read it from left to right: PCI root → PCIe root port `00:1c.0` → the card at `03:00.0` → its network interface `enp3s0`.
`lspci` shows the same `03:00.0` address. The name `enp3s0` itself means "**e**thernet, **p**ci bus **3**, **s**lot **0**".

Every device folder has some common entries:

| Entry        | Meaning                                                      |
| ------------ | ------------------------------------------------------------ |
| `device`     | Symlink to the parent hardware device (from a class entry)   |
| `driver`     | Symlink to the driver bound to it                            |
| `subsystem`  | Symlink to its class or bus                                  |
| `uevent`     | Key=value info udev uses (`DRIVER=`, `MODALIAS=`...)         |
| `vendor`, `device` | PCI/USB IDs: `0x8086` = Intel, `0x15f3` = I225-V NIC   |
| `power/`     | Runtime power management (`control`, `runtime_status`)       |

# `/sys/class` — devices by type
The most useful part for scripts. Some of the classes on a typical system:

| Class                  | Example entries                                  | Use it for                     |
| ---------------------- | ------------------------------------------------ | ------------------------------ |
| `net`                  | `enp3s0`, `wlan0`, `lo`, `docker0`, `br-*`, `veth*` | Network interfaces          |
| `block`                | `nvme0n1`, `sda`, `sda1`, `zram0`                | Disks and partitions           |
| `hwmon`                | `hwmon0` … `hwmon9`                              | **Temperatures, fans, voltages** |
| `thermal`              | `cooling_device0` …, `thermal_zone*` (laptops)   | Thermal zones + fans/throttling |
| `power_supply`         | `BAT0`, `AC` (empty on a desktop)                | Battery and charger            |
| `backlight`            | `intel_backlight`, `amdgpu_bl0` (laptops)        | Screen brightness              |
| `drm`                  | `card1`, `card1-DP-1`, `card1-HDMI-A-1`, `renderD128` | GPU and monitor outputs  |
| `input`                | `input0`, `event3`...                            | Keyboards, mice, controllers   |
| `leds`                 | `input3::capslock`                               | LEDs, including keyboard ones  |
| `dmi/id`               | `sys_vendor`, `board_name`, `bios_version`       | Motherboard / PC model         |
| `bluetooth`, `rfkill`  | `hci0`, `rfkill0`                                | Radios on/off                  |
| `nvme`                 | `nvme0`, `nvme1`                                 | NVMe controllers               |

### `/sys/class/net/<iface>`
| File                  | Example (`enp3s0`)    | Meaning                                    |
| --------------------- | --------------------- | ------------------------------------------ |
| `operstate`           | `up`                  | `up`, `down`, `unknown` (loopback), `dormant` |
| `address`             | `52:54:00:12:34:56`   | MAC address                                |
| `speed`               | `1000`                | Link speed in Mb/s (`-1` or error when down) |
| `mtu`                 | `1500`                | Max packet size                            |
| `carrier`             | `1`                   | Cable plugged in                           |
| `statistics/rx_bytes` | `3240458290`          | Received bytes (same as `/proc/net/dev`)   |
| `statistics/tx_bytes` |                       | Sent bytes                                 |
| `device`              | → PCI device          | Missing for virtual interfaces (`lo`, `docker0`, `veth*`) |
| `wireless/`           | exists for `wlan0`    | Present only on Wi-Fi interfaces           |

### `/sys/class/hwmon/hwmonN`
Hardware monitoring chips. **Always find them by `name`**, the numbers move.

| `hwmonN` | `name`       | Example                             |
| -------- | ------------ | ----------------------------------- |
| hwmon0   | `nvme`       | NVMe SSD                            |
| hwmon3   | `amdgpu`     | GPU (`edge`, `junction`, `mem`)     |
| hwmon4   | `k10temp`    | AMD CPU (`Tctl`, `Tccd1`, `Tccd2`)  |
| hwmon5–6 | `jc42`       | RAM sticks                          |
| hwmon7   | `iwlwifi_1`  | Wi-Fi card                          |
| hwmon9   | `nct6798`    | Motherboard Super-I/O: fans, voltages |

| File pattern      | Unit                    | Example                |
| ----------------- | ----------------------- | ---------------------- |
| `temp*_input`     | milli-°C                | `56500` = 56.5 °C      |
| `temp*_label`     | text                    | `Tctl`                 |
| `fan*_input`      | RPM                     | `1100`                 |
| `in*_input`       | milli-volts             | `1200` = 1.2 V         |
| `power*_average`  | micro-watts             | `45000000` = 45 W      |
| `pwm*`            | `0`–`255` fan duty      | writable               |

> [!note] Weird readings are normal
> Motherboard chips like `nct6798` expose inputs that are not connected, which is why you see values like `-63 °C` or `85 °C` on `AUXTIN*`. Trust inputs that have a meaningful label (`CPUTIN`, `SYSTIN`) or the per-device chips (`k10temp`, `amdgpu`, `nvme`).

### `/sys/class/power_supply/BAT0` (laptops)
| File                 | Example       | Meaning                               |
| -------------------- | ------------- | ------------------------------------- |
| `capacity`           | `87`          | Charge %                              |
| `status`             | `Discharging` | `Charging`, `Discharging`, `Full`, `Not charging` |
| `energy_now` / `energy_full` | µWh   | Charge now / at full (some use `charge_*` in µAh) |
| `power_now`          | µW            | Current drain                         |
| `cycle_count`        | `312`         | Battery cycles                        |
| `charge_control_end_threshold` | `80` | Charge limit (if supported, writable) |

### `/sys/class/backlight/<name>` (laptops)
`brightness` (writable) and `max_brightness`. Percent = `brightness * 100 / max_brightness`.

### `/sys/class/drm`
`card1` is the GPU, `card1-DP-1` and the others are its outputs. `cat /sys/class/drm/card1-*/status` gives `connected` / `disconnected` for each port. Each output also has an `edid` file (binary monitor info).

# `/sys/block` — disks
| File                     | Example (`nvme0n1`)   | Meaning                                  |
| ------------------------ | --------------------- | ---------------------------------------- |
| `size`                   | `1953525168`          | Size in **512-byte sectors** (always 512, whatever the disk) |
| `queue/rotational`       | `0`                   | `0` = SSD/NVMe, `1` = spinning HDD       |
| `queue/scheduler`        | `none mq-deadline [kyber] adios bfq` | I/O scheduler, `[]` = active             |
| `queue/logical_block_size` | `512`               | Sector size the disk reports             |
| `device/model`           | `Samsung SSD 980 1TB` | Model                                    |
| `removable`              | `0`                   | `1` = USB stick, card reader             |
| `ro`                     | `0`                   | Read-only                                |
| `stat`                   |                       | I/O counters (like `/proc/diskstats`)    |
| `nvme0n1p1/`, `sda1/`    |                       | Partitions are subfolders                |
| `holders/`, `slaves/`    |                       | LVM / LUKS / RAID layers on top or below |

`zram0` is compressed swap in RAM; it has no `device` link because it is not real hardware.

# `/sys/bus` — devices by bus + drivers
```bash
ls /sys/bus                          # pci usb i2c hid scsi nvme platform ...
ls /sys/bus/pci/devices              # every PCI device (same IDs as lspci)
ls /sys/bus/usb/devices              # every USB device (1-1, 1-1.2, ...)
ls /sys/bus/pci/drivers              # loaded drivers for PCI
cat /sys/bus/usb/devices/*/product   # names of connected USB devices
```
Each driver folder has `bind` / `unbind` files. Writing a device address there attaches or detaches the driver, which is how GPU passthrough to VMs works.

# `/sys/module` — kernel modules
One folder per module (296 here). `lsmod` lists the loaded ones (via [[proc]] `/proc/modules`); `/sys/module` also has modules built into the kernel.

```bash
ls /sys/module/usbcore/parameters      # autosuspend nousb ...
cat /sys/module/usbcore/parameters/autosuspend   # 2 (seconds)
modinfo -p usbcore                     # what each parameter means
```
Some parameters are writable live. To make one permanent, add `options usbcore autosuspend=5` to `/etc/modprobe.d/*.conf` → [[etc]]. Module files themselves live in `/usr/lib/modules/$(uname -r)/` → [[lib]].

# `/sys/fs/cgroup` — control groups (cgroup v2)
cgroups limit and measure resources for **groups of processes**. systemd puts every service, user session and app in its own cgroup, so the tree looks like systemd's unit tree:

```
/sys/fs/cgroup/
├── init.scope                          # PID 1
├── system.slice/
│   ├── NetworkManager.service/
│   ├── docker.service/
│   └── ...
└── user.slice/user-1000.slice/user@1000.service/app.slice/app-Alacritty@....service/
```

Which cgroup a process is in: `cat /proc/<pid>/cgroup` → append the path to `/sys/fs/cgroup`.

| File                  | Example        | Meaning                                         |
| --------------------- | -------------- | ----------------------------------------------- |
| `cgroup.procs`        |                | PIDs in this group (writable: move a process)   |
| `cgroup.controllers`  | `cpuset cpu io memory hugetlb pids rdma misc dmem` | Available controllers |
| `memory.current`      | `25686016`     | RAM used now, **bytes** (NetworkManager ≈ 24 MiB) |
| `memory.max`          | `max`          | Hard limit (`MemoryMax=` in systemd)            |
| `memory.peak`         |                | Highest usage seen                              |
| `memory.stat`         | `anon ...`, `file ...` | Breakdown in bytes                      |
| `cpu.stat`            | `usage_usec ...` | CPU time used, microseconds                  |
| `cpu.max`             | `max 100000`   | CPU quota (`CPUQuota=`)                         |
| `pids.current`        |                | Number of tasks                                 |
| `io.stat`             |                | Bytes read/written per disk                     |
| `*.pressure`          | `some avg10=...` | PSI per group, same as `/proc/pressure`       |
| `cgroup.kill`         |                | Write `1` = kill all processes in the group     |

> [!tip] Change limits through systemd
> Don't write limits into `/sys/fs/cgroup` by hand, systemd will overwrite them. Use `systemctl set-property docker.service MemoryMax=2G` or a drop-in file in `/etc/systemd/system/<unit>.d/`. Live view: `systemd-cgtop`.

# `/sys/firmware`
| Path                           | What                                                       |
| ------------------------------ | ---------------------------------------------------------- |
| `/sys/firmware/efi/`           | Exists only when booted in **UEFI** mode                   |
| `/sys/firmware/efi/fw_platform_size` | `64` = 64-bit UEFI                                   |
| `/sys/firmware/efi/efivars/`   | UEFI variables (boot order, Secure Boot...) → `efibootmgr` |
| `/sys/firmware/acpi/tables/`   | Raw ACPI tables (root only)                                |
| `/sys/class/dmi/id/`           | `sys_vendor` = `ASUS`, `board_name` = `ROG STRIX B550-F GAMING WIFI II`, `bios_version` = `3611` |

> [!danger] Don't delete files in `efivars`
> Some badly written firmware gets **bricked** when certain EFI variables are removed. The files are immutable by default for that reason; leave it that way. Use `efibootmgr` for boot entries. → [[boot]]

# CPU frequency: `/sys/devices/system/cpu`
```
/sys/devices/system/cpu/
├── cpu0/ … cpu15/
│   ├── cpufreq/       # frequency scaling
│   ├── topology/      # core_id, thread_siblings_list
│   └── online         # 1/0, take a core offline (not cpu0)
├── online             # 0-15
└── smt/control        # hyper-threading on/off
```

| File (in `cpuN/cpufreq/`)      | Example                       | Meaning                          |
| ------------------------------ | ----------------------------- | -------------------------------- |
| `scaling_cur_freq`             | `4616082`                     | Current frequency in **kHz**     |
| `cpuinfo_max_freq`             | `4954565`                     | Hardware max, kHz                |
| `scaling_min_freq` / `scaling_max_freq` |                      | Limits the governor uses (writable) |
| `scaling_driver`               | `amd-pstate-epp`              | Frequency driver                 |
| `scaling_governor`             | `performance`                 | Policy (writable)                |
| `scaling_available_governors`  | `performance powersave`       | Choices                          |
| `energy_performance_preference`| `performance`, `balance_power`... | Hint for EPP drivers         |
| `boost`                        | `1`                           | Turbo boost on/off               |

# Scripting recipes
All tested in `bash`.

### All temperatures
```bash
for h in /sys/class/hwmon/hwmon*; do
  name=$(< "$h/name")
  for t in "$h"/temp*_input; do
    [[ -r $t ]] || continue
    label=$(cat "${t%_input}_label" 2>/dev/null || basename "${t%_input}")
    awk -v n="$name" -v l="$label" '{printf "%-12s %-10s %5.1f °C\n", n, l, $1/1000}' "$t"
  done
done 2>/dev/null
```

### CPU temperature only (find the chip by name, not number)
```bash
for h in /sys/class/hwmon/hwmon*; do
  [[ $(< "$h/name") == k10temp || $(< "$h/name") == coretemp ]] || continue
  echo "CPU: $(( $(< "$h/temp1_input") / 1000 )) °C"   # k10temp temp1 = Tctl
done
```
`k10temp` = AMD, `coretemp` = Intel.

### Battery %
```bash
shopt -s nullglob
bats=(/sys/class/power_supply/BAT*)
if (( ${#bats[@]} == 0 )); then
  echo "No battery (desktop)"
else
  for b in "${bats[@]}"; do echo "${b##*/}: $(< "$b/capacity")% ($(< "$b/status"))"; done
fi
```

### Network cards: state, MAC, speed
```bash
for n in /sys/class/net/*; do
  [[ -e $n/device ]] || continue          # skip virtual (lo, docker0, veth...)
  printf "%-8s %-5s %-17s %s Mb/s\n" "${n##*/}" "$(< "$n/operstate")" \
    "$(< "$n/address")" "$(cat "$n/speed" 2>/dev/null || echo '?')"
done
```
```
enp3s0   up    52:54:00:12:34:56 1000 Mb/s
wlan0    down  52:54:00:ab:cd:ef ? Mb/s
```

### Disks: SSD or HDD, size, model
```bash
for b in /sys/block/*; do
  [[ -e $b/device ]] || continue          # skip zram, loop, dm-*
  kind=SSD; [[ $(< "$b/queue/rotational") == 1 ]] && kind=HDD
  printf "%-8s %-4s %6d GiB  %s\n" "${b##*/}" "$kind" \
    "$(( $(< "$b/size") * 512 / 1024**3 ))" "$(cat "$b/device/model" 2>/dev/null | xargs)"
done
```
```
nvme0n1  SSD     931 GiB  Samsung SSD 980 1TB
sda      HDD     931 GiB  ST1000DM010-2EP102
```

### CPU frequencies
```bash
for c in /sys/devices/system/cpu/cpu[0-9]*/cpufreq/scaling_cur_freq; do
  cpu=${c#/sys/devices/system/cpu/}; cpu=${cpu%%/*}
  echo "$cpu $(( $(< "$c") / 1000 )) MHz"
done | sort -V
```

### Switch governor on all cores (temporary)
```bash
echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### Memory of a systemd service (from its cgroup)
```bash
unit=${1:-NetworkManager.service}
cg=/sys/fs/cgroup$(systemctl show -P ControlGroup "$unit")
echo "$unit: $(( $(< "$cg/memory.current") / 1024**2 )) MiB, $(wc -l < "$cg/cgroup.procs") procs"
```
```
NetworkManager.service: 24 MiB, 1 procs
```

### Which cgroup am I in?
```bash
cg=/sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)
echo "$cg"
grep -E '^(anon|file) ' "$cg/memory.stat"
```

### UEFI or BIOS?
```bash
[[ -d /sys/firmware/efi ]] && echo "UEFI boot" || echo "BIOS (legacy) boot"
```

### Which monitors are connected?
```bash
for o in /sys/class/drm/card*-*; do
  [[ $(< "$o/status") == connected ]] && echo "${o##*/card?-}"
done
```

### Which driver does a device use?
```bash
basename "$(readlink -f /sys/class/net/enp3s0/device/driver)"   # igc
```

# Find stuff

| I want...                           | Look at                                                |
| ----------------------------------- | ------------------------------------------------------ |
| Temperatures / fans                 | `/sys/class/hwmon/*/temp*_input`, `fan*_input`         |
| Battery, AC adapter                 | `/sys/class/power_supply/`                             |
| Screen brightness                   | `/sys/class/backlight/*/brightness`                    |
| Network card speed / MAC / up?      | `/sys/class/net/<iface>/{speed,address,operstate}`     |
| Disk size, SSD/HDD, model           | `/sys/block/<disk>/{size,queue/rotational,device/model}` |
| CPU frequency / governor            | `/sys/devices/system/cpu/cpu*/cpufreq/`                |
| PC / motherboard / BIOS model       | `/sys/class/dmi/id/`                                   |
| Which driver a device uses          | `readlink <device>/driver`                             |
| Module options                      | `/sys/module/<mod>/parameters/`                        |
| Memory/CPU of a service or app      | `/sys/fs/cgroup/<path from /proc/pid/cgroup>/`         |
| Connected monitors                  | `/sys/class/drm/card*-*/status`                        |
| USB devices                         | `/sys/bus/usb/devices/*/product`                       |
| UEFI variables                      | `/sys/firmware/efi/efivars/`                           |
| Huge pages / THP setting            | `/sys/kernel/mm/transparent_hugepage/enabled` → `[always] madvise never` |
| Find any file by name               | `find /sys -name 'temp*_label' 2>/dev/null` (avoid `-L`, the links loop) |

# Commands that read `/sys` for you

| Command                   | Reads                                         |
| ------------------------- | --------------------------------------------- |
| `sensors`                 | `/sys/class/hwmon`                            |
| `lsblk`                   | `/sys/block`, `/sys/dev/block`                |
| `lspci`, `lsusb`          | `/sys/bus/pci`, `/sys/bus/usb`                |
| `ip link`                 | netlink (same data as `/sys/class/net`)       |
| `cpupower`, `lscpu`       | `/sys/devices/system/cpu`                     |
| `brightnessctl`           | `/sys/class/backlight`, `/sys/class/leds`     |
| `upower`                  | `/sys/class/power_supply`                     |
| `systemd-cgtop`           | `/sys/fs/cgroup`                              |
| `efibootmgr`              | `/sys/firmware/efi/efivars`                   |
| `udevadm info /sys/...`   | `uevent` files → [[dev]]                      |

# See also
- [[proc]] — processes; `/proc/sys` holds kernel settings (`sysctl`), `/sys` holds device settings
- [[dev]] — device **files** you read/write data through; `/sys` describes those devices
- [[etc]] — make `/sys` changes permanent (`udev/rules.d`, `modprobe.d`, `tmpfiles.d`)
- [[boot]] — EFI system partition, bootloader
- [[run]] — `/run/udev` holds the udev database built from `/sys`
- `man 5 sysfs`, `udevadm info -a /sys/class/net/enp3s0` (walks the tree upward with all attributes)
- Kernel docs: <https://docs.kernel.org/admin-guide/sysfs-rules.html>, <https://docs.kernel.org/admin-guide/cgroup-v2.html>
