---
tags:
  - linux
  - filesystem
  - dev
  - devices
  - scripting
aliases:
  - /dev
  - device files
  - devtmpfs
---
`/dev` is where devices show up as files. Disks, terminals, keyboards, GPUs, sound cards and a few "magic" files like `/dev/null` all live here. "Everything is a file" in practice: you `open`, `read` and `write` a device the same way as a normal file.

> [!info] What `/dev` really is
> Like [[proc]], `/dev` is **not on disk**. It is a `devtmpfs` (RAM filesystem) that the kernel fills automatically, and **udev** (`systemd-udevd`) then adjusts it: sets permissions, creates friendly symlinks like `/dev/disk/by-uuid/...`.
> ```
> TARGET SOURCE   FSTYPE   OPTIONS
> /dev   devtmpfs devtmpfs rw,nosuid,size=16311884k,mode=755
> ```
> Plug in a USB stick → kernel creates `/dev/sdb` → udev adds symlinks and rules → desktop shows it. Unplug → it disappears.
>
> Mounted inside `/dev`:
> | Mount            | Type        | What                                      |
> | ---------------- | ----------- | ----------------------------------------- |
> | `/dev/pts`       | `devpts`    | Pseudo terminals (terminal windows, SSH)  |
> | `/dev/shm`       | `tmpfs`     | Shared memory between processes           |
> | `/dev/mqueue`    | `mqueue`    | POSIX message queues                      |
> | `/dev/hugepages` | `hugetlbfs` | Huge memory pages (`pagesize=2M`)         |

> [!danger] Writing to devices is dangerous
> `dd of=/dev/nvme0n1`, `cat x > /dev/sda`, `mkfs /dev/...` overwrite the disk **instantly, with no confirmation and no undo**. Always triple-check device names with `lsblk` before writing. Names like `sda`/`sdb` can **swap between boots** — use `/dev/disk/by-id/...` in scripts.

# Block vs character devices
`ls -l` shows the type in the first letter and **major, minor** numbers instead of the file size.

```
brw-rw---- 1 root disk 259, 1  /dev/nvme0n1
brw-rw---- 1 root disk 259, 3  /dev/nvme0n1p1
crw-rw-rw- 1 root root   1, 3  /dev/null
crw-rw---- 1 root tty    5, 0  /dev/tty
```

| Letter | Type          | How it works                                                  | Examples                          |
| ------ | ------------- | ------------------------------------------------------------- | --------------------------------- |
| `b`    | **Block**     | Random access in fixed blocks, goes through page cache        | disks, partitions, `zram0`, loop  |
| `c`    | **Character** | Stream of bytes, no seeking (mostly)                          | terminals, `null`, input, sound   |
| `l`    | Symlink       | Points to another device                                      | `/dev/stdin`, `/dev/disk/by-*/`   |

| Number    | Meaning                                                    | Example                      |
| --------- | ---------------------------------------------------------- | ---------------------------- |
| **major** | Which **driver** handles it                                | `259` = `blkext` (NVMe parts), `8` = SCSI/SATA disk, `1` = `mem` |
| **minor** | Which **instance** of that driver                          | `nvme0n1p1` = `259:3`        |

The file name doesn't matter to the kernel — only type + major + minor. Driver list: `cat /proc/devices`.

```
Character devices:        Block devices:
  1 mem                     8 sd
  4 tty                   253 zram
  5 /dev/tty              259 blkext
 13 input
240 nvme
```

> [!note] Groups give access
> `brw-rw---- root disk` → only root and the `disk` group can read raw disks. Same idea: `video` (GPU, webcam), `audio`, `input`, `storage`, `uucp` (serial ports). Your groups: `id`.

# Disks and partitions
Example disks (`lsblk -o NAME,SIZE,FSTYPE,PARTTYPENAME,MOUNTPOINTS`):

```
sda           931,5G                                      ← SATA HDD
└─sda1        931,5G ext4   Linux filesystem              /data
zram0           8G   swap                                 [SWAP]
nvme0n1       931,5G                                      ← NVMe system disk
├─nvme0n1p1       1G vfat   EFI System                    /boot
├─nvme0n1p2       8G swap   Linux swap                    [SWAP]
├─nvme0n1p3     200G btrfs  Linux filesystem              / /var/log ...
└─nvme0n1p4   722,5G btrfs  Linux filesystem              /home
```

### Naming

| Name pattern     | Device                                  | Partition                | Example                |
| ---------------- | --------------------------------------- | ------------------------ | ---------------------- |
| `sdX`            | SATA / SCSI / **USB** disks (a, b, c…)  | `sdX1`, `sdX2`           | `sdb1`                 |
| `nvmeXnY`        | NVMe controller X, namespace Y          | `nvmeXnYpZ` (note `p`)   | `nvme0n1p3`            |
| `mmcblkX`        | SD card / eMMC                          | `mmcblkXpY`              | `mmcblk0p1`            |
| `vdX`            | Virtual disk in a VM (virtio)           | `vdX1`                   | `vda1`                 |
| `loopN`          | File mounted as a disk (ISO, snap)      | `loopNpM`                | `loop0`                |
| `zramN`          | Compressed RAM disk (used as swap here) | —                        | `zram0`                |
| `dm-N`, `mapper/`| Device mapper: LUKS, LVM                | —                        | `/dev/mapper/cryptroot`|
| `srN`            | CD/DVD                                  | —                        | `sr0`                  |
| `nvmeX`          | NVMe **controller** (char device, not a disk) | —                  | `nvme0`                |

### Stable names: `/dev/disk/by-*`
`sda` vs `sdb` depends on detection order and **can change**. udev creates stable symlinks:

| Folder          | Based on                                        | Example                                                     |
| --------------- | ----------------------------------------------- | ----------------------------------------------------------- |
| `by-uuid/`      | Filesystem UUID (changes on `mkfs`)             | `3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c -> ../../nvme0n1p3`   |
| `by-id/`        | Model + serial number of the hardware           | `ata-ST1000DM010-2EP102_S1234567890-part1 -> ../../sda1`     |
| `by-label/`     | Filesystem label you set                        | `Data`                                                      |
| `by-partuuid/`  | GPT partition UUID (survives `mkfs`)            |                                                             |
| `by-partlabel/` | GPT partition name                              |                                                             |
| `by-path/`      | Physical port/slot (PCI, USB)                   | `pci-0000:01:00.0-nvme-1`                                   |

`root=UUID=...` in `/proc/cmdline` and `UUID=...` lines in `/etc/fstab` use these → see [[boot]], [[etc]].

```bash
readlink -f /dev/disk/by-uuid/3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c   # /dev/nvme0n1p3
```

# Special "magic" devices
All are character devices of driver `1 mem`.

| Device         | maj:min | Read gives                         | Write does                         | Use                                  |
| -------------- | ------- | ---------------------------------- | ---------------------------------- | ------------------------------------ |
| `/dev/null`    | `1:3`   | EOF (nothing)                      | Discards everything                | Hide output: `cmd > /dev/null 2>&1`  |
| `/dev/zero`    | `1:5`   | Endless `\0` bytes                 | Discards                           | Make empty files: `dd if=/dev/zero`  |
| `/dev/full`    | `1:7`   | Endless `\0` bytes                 | Fails with `No space left on device` | Test disk-full error handling      |
| `/dev/random`  | `1:8`   | Cryptographic random bytes         | Adds entropy                       | Keys, passwords                      |
| `/dev/urandom` | `1:9`   | Same as `random` (since kernel 5.6)| Adds entropy                       | Everything random                    |
| `/dev/mem`     | `1:1`   | Physical RAM (root + restricted)   | —                                  | Don't                                |
| `/dev/kmsg`    | `1:11`  | Kernel log (`dmesg` reads it)      | Write a message into kernel log    | Logging from early boot              |

> [!tip] `random` vs `urandom`
> On modern kernels both are the same CSPRNG and never run "out of entropy" after boot. Just use `/dev/urandom`.

# Terminals: `tty`, `pts`, `console`

| Device          | What                                                                 |
| --------------- | -------------------------------------------------------------------- |
| `/dev/tty1`…`tty6` | Virtual consoles (`Ctrl+Alt+F1`…`F6`)                              |
| `/dev/tty0`     | The **currently visible** virtual console                            |
| `/dev/tty`      | **Your own** controlling terminal, whatever it is                     |
| `/dev/pts/N`    | Pseudo terminals: every terminal window, tmux pane, SSH session       |
| `/dev/ptmx`     | "Master" — opening it creates a new `/dev/pts/N`                      |
| `/dev/console`  | Where the kernel prints system messages                               |
| `/dev/ttyS0`    | Serial port (COM1)                                                    |
| `/dev/ttyUSB0`, `ttyACM0` | USB serial adapters, Arduino                                |

```bash
tty                   # which terminal am I on? → /dev/pts/0
ls /dev/pts           # 0 1 2 ptmx → three open terminals
who                   # users and their terminals
echo hi > /dev/pts/1  # write into another terminal window (yours only)
```

`/dev/tty` is handy in scripts: reading from it asks the **user**, even when stdin is a pipe.
```bash
read -r answer < /dev/tty
```

# `/dev/fd`, `stdin`, `stdout`, `stderr`
Symlinks into [[proc]]:

```
/dev/fd     -> /proc/self/fd
/dev/stdin  -> /proc/self/fd/0
/dev/stdout -> /proc/self/fd/1
/dev/stderr -> /proc/self/fd/2
```

Useful for programs that want a **file name** but you have a stream:
```bash
some-tool --output /dev/stdout | grep foo
diff <(ls dir1) <(ls dir2)        # bash turns <(...) into /dev/fd/63
echo "error" > /dev/stderr
```

> [!note] Bash-only fake devices
> `/dev/tcp/<host>/<port>` and `/dev/udp/...` **don't exist** on disk — bash fakes them:
> ```bash
> exec 3<>/dev/tcp/127.0.0.1/631 && echo "port 631 open"
> ```

# Hardware devices

### `/dev/input` — keyboard, mouse, gamepads
`eventN` files, one per input device. Readable by root / `input` group.

| File                | Example                                                   |
| ------------------- | --------------------------------------------------------- |
| `event12`           | Keychron K8 keyboard                                      |
| `event10`           | Razer BlackShark V3 X (headset buttons)                   |
| `mice`, `mouseN`    | Legacy combined mouse interface                           |
| `by-id/`            | `usb-Keychron_K8_Keychron_K8-event-kbd -> ../event12`     |

Names come from `/sys/class/input/eventN/device/name` → [[sys]]. Debug with `sudo evtest` or `libinput debug-events`.

### `/dev/dri` — GPU
| File          | What                                                       | Who uses it                 |
| ------------- | ---------------------------------------------------------- | --------------------------- |
| `card1`       | Full GPU access (display output, modesetting)              | Compositor (KWin, Wayland)  |
| `renderD128`  | Render-only (no display), safe to share                    | Games, Vulkan, video decode |
| `by-path/`    | `pci-0000:0a:00.0-card -> ../card1`                        |                             |

GPU vendor: `cat /sys/class/drm/card1/device/vendor` → `0x1002` = AMD (`0x10de` NVIDIA, `0x8086` Intel).

### `/dev/snd` — sound (ALSA)
| File            | Meaning                                         |
| --------------- | ----------------------------------------------- |
| `controlCN`     | Mixer of card N                                 |
| `pcmCNDMp`      | Card N, device M, **p**layback                  |
| `pcmCNDMc`      | Card N, device M, **c**apture (microphone)      |
| `seq`, `timer`  | MIDI sequencer, timer                           |

Card numbers ↔ names: `cat /proc/asound/cards`
```
 0 [X      ]: USB-Audio - BlackShark V3 X
 1 [HDMI   ]: HDA-Intel - HDA ATI HDMI
 2 [Generic]: HDA-Intel - HD-Audio Generic
```
Normally PipeWire owns these; apps talk to PipeWire, not to `/dev/snd` directly.

### Others worth knowing
| Device                  | What                                            |
| ----------------------- | ----------------------------------------------- |
| `/dev/video0`           | Webcam (V4L2)                                   |
| `/dev/hidrawN`          | Raw USB HID (RGB controllers, macro keyboards)  |
| `/dev/loop-control`     | Allocates loop devices (`losetup`)              |
| `/dev/kvm`              | Hardware virtualisation for VMs                 |
| `/dev/fuse`             | Userspace filesystems (sshfs, AppImage)         |
| `/dev/net/tun`          | VPN tunnels (WireGuard, OpenVPN)                |
| `/dev/rfkill`           | Wi-Fi / Bluetooth kill switch                   |
| `/dev/mapper/`          | LUKS / LVM volumes                              |

# `/dev/shm`
A `tmpfs` (RAM) mounted for shared memory. Programs use it for fast inter-process communication (browsers, PulseAudio, Python `multiprocessing`). Anyone can create files there, they vanish on reboot. Good for **fast temp files** in scripts:
```bash
f=$(mktemp -p /dev/shm demo.XXXX); echo hi > "$f"; cat "$f"; rm "$f"
```
Compare with [[tmp]] (may also be tmpfs) and [[run]].

# udev
`systemd-udevd` reacts to kernel events (device added/removed) and applies **rules**: permissions, symlinks, names, run scripts.

| Path                       | What                                       |
| -------------------------- | ------------------------------------------ |
| `/usr/lib/udev/rules.d/`   | Rules shipped by packages (don't edit)     |
| `/etc/udev/rules.d/`       | **Your** rules (override same file name)   |
| `/run/udev/`               | Runtime database                           |

```bash
udevadm info -n /dev/nvme0n1                 # all properties of a device
udevadm info -q property -n /dev/nvme0n1p3   # just KEY=value
udevadm monitor --udev --property            # watch events live (plug in USB)
udevadm info -a -n /dev/sda | less           # attributes to match in rules
sudo udevadm control --reload && sudo udevadm trigger   # apply new rules
```
```
ID_MODEL=Samsung SSD 980 1TB
ID_PATH=pci-0000:01:00.0-nvme-1
ID_FS_TYPE=btrfs
ID_FS_UUID=3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c
```

Example rule — give a USB device to your user (`/etc/udev/rules.d/99-mydevice.rules`):
```
SUBSYSTEM=="usb", ATTRS{idVendor}=="1234", ATTRS{idProduct}=="5678", MODE="0660", GROUP="users"
```

# Scripting recipes
All tested in `bash` as a normal user.

### List disks (no partitions) with model
```bash
lsblk -dno NAME,SIZE,ROTA,TRAN,MODEL
```
```
sda     931,5G 1 sata   ST1000DM010-2EP102           ← ROTA=1 = spinning HDD
nvme0n1 931,5G 0 nvme   Samsung SSD 980 1TB
```
For scripts use JSON: `lsblk -J -o NAME,SIZE,MOUNTPOINTS | jq`.

### Block devices from `/sys` with major:minor and size
```bash
for d in /sys/class/block/*; do
  n=${d##*/}
  printf "%-12s %-7s %sG\n" "$n" "$(< "$d/dev")" "$(( $(< "$d/size") * 512 / 1024**3 ))"
done
```
(`size` is always in 512-byte sectors.)

### Major/minor and type of a device file
```bash
stat -c '%n type=%F major=%Hr minor=%Lr' /dev/null /dev/nvme0n1p1
```

### Device for a UUID / UUID for a device
```bash
readlink -f /dev/disk/by-uuid/3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c
lsblk -no UUID /dev/nvme0n1p3
findmnt -no SOURCE /          # which device is / ? → /dev/nvme0n1p3[/@] ([/@] = btrfs subvolume)
```

### Is a device / file a block device?
```bash
[[ -b /dev/sda ]] && echo "block device"
[[ -c /dev/null ]] && echo "char device"
```

### Am I running in a terminal?
```bash
[[ -t 1 ]] && echo "stdout is a terminal (colors OK)" || echo "piped/redirected"
```

### Random password / hex token
```bash
tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 20; echo
head -c 16 /dev/urandom | od -An -tx1 | tr -d ' \n'; echo
```

### Create a 100 MiB file of zeros (for tests, swap file, image)
```bash
dd if=/dev/zero of=test.img bs=1M count=100 status=progress
```

### Name of every input device
```bash
for e in /dev/input/event*; do
  n=${e##*/}
  printf "%-8s %s\n" "$n" "$(< /sys/class/input/$n/device/name)"
done
```

### Wait for a USB disk to appear
```bash
until [[ -e /dev/disk/by-label/Data ]]; do sleep 1; done
echo "disk connected"
```

# Find stuff

| I want...                              | Look at                                          |
| -------------------------------------- | ------------------------------------------------ |
| All disks and partitions               | `lsblk`, `lsblk -f` (with FS + UUID)             |
| Which device is mounted where          | `findmnt`, `/proc/mounts`                        |
| Device by UUID / label / serial        | `/dev/disk/by-uuid/`, `by-label/`, `by-id/`      |
| Device details (model, serial, path)   | `udevadm info -n /dev/X`                         |
| Driver for a major number              | `/proc/devices`                                  |
| More info about any device             | `/sys/class/<type>/<name>/` → [[sys]]            |
| My terminal                            | `tty`                                            |
| GPU node                               | `/dev/dri/`, `/sys/class/drm/`                   |
| Sound cards                            | `/proc/asound/cards`, `aplay -l`                 |
| Input devices                          | `/dev/input/by-id/`, `libinput list-devices`     |
| USB devices                            | `lsusb`, `/dev/bus/usb/`                         |
| PCI devices                            | `lspci`                                          |
| Why device has these permissions       | `/usr/lib/udev/rules.d/`, `/etc/udev/rules.d/`   |
| Kernel messages about devices          | `journalctl -k`, `dmesg` (reads `/dev/kmsg`)     |

# See also
- [[sys]] — `/sys`, the detailed info behind every device
- [[proc]] — `/proc/devices`, `/proc/self/fd`, `/proc/partitions`
- [[boot]] — how `root=UUID=` finds the disk
- [[etc]] — `/etc/fstab`, `/etc/udev/rules.d/`
- [[run]], [[tmp]] — other RAM filesystems
- `man 4 null`, `man 4 random`, `man 4 tty`, `man 7 udev`, `man 8 lsblk`
- Kernel device list: <https://docs.kernel.org/admin-guide/devices.html>
