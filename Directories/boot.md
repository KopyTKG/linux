---
tags:
  - linux
  - filesystem
  - boot
  - kernel
  - scripting
aliases:
  - /boot
  - boot partition
  - ESP
---
`/boot` holds everything the machine needs **before** the root filesystem is mounted: the kernel, the initramfs, CPU microcode and (on UEFI systems) often the bootloader itself.

> [!info] Three common layouts
> | Layout                           | `/boot` is ...                         | ESP mounted at |
> | -------------------------------- | -------------------------------------- | -------------- |
> | ESP **is** `/boot`               | the EFI System Partition, FAT32        | `/boot`        |
> | `/boot` on root FS               | a normal folder on `/`                 | `/boot/efi`    |
> | `/boot` + separate ESP           | own ext4 partition or folder on `/`    | `/efi` or `/boot/efi` |
>
> The **EFI System Partition (ESP)** is always FAT32 (`vfat`), because UEFI firmware can only read FAT. Check yours:
> ```bash
> findmnt /boot; findmnt /boot/efi; findmnt /efi
> ```
> ```
> TARGET SOURCE         FSTYPE OPTIONS
> /boot  /dev/nvme0n1p1 vfat   rw,relatime,fmask=0077,dmask=0077,...
> ```
> `fmask=0077,dmask=0077` means only root can read it — a normal user gets `Permission denied` on `ls /boot`. FAT has no Linux permissions, symlinks or owners, so don't put anything there that relies on them.

> [!warning] Be careful in here
> A broken `/boot` = a machine that doesn't boot. Never delete files by hand. Let the package manager and the initramfs tool manage them, and keep a second kernel or a fallback entry as an emergency exit.

# Boot process overview
What happens between the power button and the login screen. Example timings from `systemd-analyze`:

```
Startup finished in 8.089s (firmware) + 5.409s (loader) + 2.871s (kernel) + 4.362s (initrd) + 11.420s (userspace) = 32.153s
```

| # | Stage          | Who                         | What it does                                                                 | Time    |
| - | -------------- | --------------------------- | ---------------------------------------------------------------------------- | ------- |
| 1 | **Firmware**   | UEFI (or legacy BIOS)       | POST, init hardware, read NVRAM boot entries, load an `.efi` file from the ESP | `8.1s`  |
| 2 | **Loader**     | GRUB, systemd-boot, Limine, ... | Show menu, load kernel + microcode + initramfs into RAM, pass the cmdline | `5.4s`  |
| 3 | **Kernel**     | `vmlinuz`                   | Decompress, init CPU/memory/drivers built into the kernel                    | `2.9s`  |
| 4 | **initrd**     | initramfs                   | Load modules for disk/FS, find `root=...`, mount it, `switch_root`           | `4.4s`  |
| 5 | **Userspace**  | init = `systemd` (PID 1)    | Start services, reach the default target, show login                         | `11.4s` |

```bash
systemd-analyze            # summary above
systemd-analyze blame      # slowest units
systemd-analyze critical-chain
```
More on stage 5 in [[targets-boot]].

> [!note] Legacy BIOS
> Old BIOS machines have no ESP. The firmware runs code from the first sector of the disk (MBR), which loads the bootloader (usually GRUB) from `/boot`. The rest of the chain is the same.

# Kernel image — `vmlinuz`
The kernel is one compressed file, usually `vmlinuz-<version>` or `vmlinuz-<flavour>`. "vm" = virtual memory, "z" = compressed.

```bash
uname -r              # running kernel: 6.12.8
ls /lib/modules/      # one folder per installed kernel
```
```
6.6.70
6.12.8
```

> [!tip] Readable copy in `/lib/modules`
> Many distros also keep the kernel at `/lib/modules/<version>/vmlinuz`, which any user can read, while `/boot` might be root-only:
> ```
> $ file -L /lib/modules/$(uname -r)/vmlinuz
> Linux kernel x86 boot executable, bzImage, version 6.12.8 ...
> ```
> Kernel modules (drivers as `.ko` files) live next to it → see [[lib]].

Keep **two kernels** installed (e.g. latest + LTS). If an update breaks booting, pick the other one in the boot menu.

# initramfs
`initramfs-<version>.img` (also called `initrd.img-<version>` or `initrd`) is a small compressed filesystem (cpio archive) loaded into RAM together with the kernel. It is a "mini Linux" whose only job is to **find and mount the real root filesystem**: load the storage/filesystem/encryption drivers, wait for the disk, ask for a LUKS password if needed, mount `/`, then hand over to the real init.

| Typical file                             | What                                                   |
| ---------------------------------------- | ------------------------------------------------------ |
| `vmlinuz-<version>`                      | Kernel                                                 |
| `initramfs-<version>.img` / `initrd.img-<version>` | Initramfs with the modules this PC needs     |
| `*-fallback.img`                         | Bigger initramfs with **all** modules (some distros)   |
| `amd-ucode.img` / `intel-ucode.img`      | CPU microcode (if not embedded in the initramfs)       |
| `config-<version>`, `System.map-<version>` | Kernel build config and symbol table (some distros)  |

Distros use different tools to build it — `dracut`, `mkinitcpio`, `initramfs-tools`, `booster`. They all do the same thing: collect modules + firmware + a small init into an archive, and rerun automatically on kernel updates.

> [!note] Rebuild after changing config
> Changes to `/etc/modprobe.d/`, `/etc/crypttab` or the initramfs tool's config only take effect in early boot after the initramfs is rebuilt (see your tool's man page).

```bash
journalctl -b -o short-monotonic | grep -m3 -i initrd   # what happened in the initramfs this boot
```

# CPU microcode
CPUs have bugs. Vendors ship fixes as **microcode** that is loaded into the CPU at every boot (it is not persistent).

```bash
grep -m1 microcode /proc/cpuinfo                 # microcode : 0xa201211
journalctl -k -b | grep -i microcode
```
```
microcode: Current revision: 0x0a201211
microcode: Updated early from: 0x0a201210
```
`Updated early` = loaded from the initramfs before anything else → working correctly.

# Bootloader
The bootloader is the program the firmware starts. It loads the kernel and passes it the command line.

| Bootloader       | Typical config                                             | Recognise it by                          |
| ---------------- | ---------------------------------------------------------- | ---------------------------------------- |
| GRUB             | `/boot/grub/grub.cfg` (generated from `/etc/default/grub`) | `/boot/grub/`, `grub-mkconfig` / `update-grub` |
| systemd-boot     | `<ESP>/loader/loader.conf`, `<ESP>/loader/entries/*.conf`  | `<ESP>/EFI/systemd/`, `bootctl`          |
| Limine           | `<ESP>/limine.conf`                                        | `<ESP>/EFI/limine/`                      |
| rEFInd           | `<ESP>/EFI/refind/refind.conf`                             | `<ESP>/EFI/refind/`                      |
| UKI (no loader)  | —                                                          | `<ESP>/EFI/Linux/*.efi`                  |

A **UKI** (Unified Kernel Image) packs kernel + initramfs + cmdline into one signed `.efi` file the firmware can start directly.

### Detect which one you use
```bash
bootctl status 2>/dev/null | grep -A1 'Current Boot Loader'
```
```
Current Boot Loader:
        Product: systemd-boot 257
```
UEFI boot entries stored in firmware NVRAM:
```bash
efibootmgr
```
```
BootCurrent: 0000
BootOrder: 0000,0001,0004,0002,0003,0005
Boot0000* Linux Boot Manager   HD(1,GPT,1a2b3c4d-...)/\EFI\systemd\systemd-bootx64.efi
Boot0001* Windows Boot Manager HD(1,GPT,1a2b3c4d-...)/\EFI\MICROSOFT\BOOT\BOOTMGFW.EFI
```
Linux and Windows can boot from the **same** ESP (`HD(1,GPT,...)` = partition 1). Each OS gets its own folder under `EFI/`.

# Kernel command line ↔ `/proc/cmdline`
The bootloader passes a line of parameters to the kernel. The kernel shows exactly what it received in `/proc/cmdline` → see [[proc]].

```bash
cat /proc/cmdline
```
```
quiet nowatchdog splash rw rootflags=subvol=/@ root=UUID=3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c resume=UUID=7d2e4f6a-1b3c-4d5e-8f9a-0b1c2d3e4f5a
```

| Parameter                  | Meaning                                                         |
| -------------------------- | --------------------------------------------------------------- |
| `quiet`                    | Fewer kernel messages on screen                                 |
| `splash`                   | Show graphical boot splash                                      |
| `nowatchdog`               | Disable watchdog timers                                         |
| `rw` / `ro`                | Mount root read-write / read-only (fsck first, then remount)    |
| `root=UUID=3f6c...`        | Which partition is `/` → `/dev/nvme0n1p3`                       |
| `rootflags=subvol=/@`      | Mount options for root (here: btrfs subvolume)                  |
| `resume=UUID=7d2e...`      | Swap partition to resume hibernation from                       |
| `init=/bin/bash`           | Emergency: start a shell instead of the normal init             |
| `systemd.unit=rescue.target` | Boot into rescue mode → [[targets-boot]]                      |

> [!tip] Where to change it
> | Bootloader   | Edit                                                     | Then                       |
> | ------------ | -------------------------------------------------------- | -------------------------- |
> | GRUB         | `GRUB_CMDLINE_LINUX` in `/etc/default/grub`              | regenerate `grub.cfg` (`grub-mkconfig -o /boot/grub/grub.cfg`, or `update-grub`) |
> | systemd-boot | `options` line in `<ESP>/loader/entries/*.conf`          | nothing                    |
> | Limine       | `cmdline:` line in `limine.conf` (or the distro's generator config) | nothing / regenerate |
> | UKI          | `/etc/kernel/cmdline`                                    | rebuild the UKI            |
>
> For a one-time test, edit the entry in the boot menu instead (GRUB: `e`, systemd-boot: `e`, Limine: `e`).

Full list of parameters: `man 7 bootparam`, <https://docs.kernel.org/admin-guide/kernel-parameters.html>.

# Scripting recipes
All tested in `bash` without root.

### Is the system UEFI or legacy BIOS?
```bash
[[ -d /sys/firmware/efi ]] && echo "UEFI ($(< /sys/firmware/efi/fw_platform_size)-bit)" || echo "Legacy BIOS"
```

### Where is the ESP / is `/boot` separate?
```bash
for m in /boot /boot/efi /efi; do findmnt -no TARGET,SOURCE,FSTYPE "$m"; done
lsblk -o NAME,SIZE,FSTYPE,PARTTYPENAME,MOUNTPOINTS | grep -i 'EFI System'
```

### Installed kernels vs running kernel
```bash
echo "running:   $(uname -r)"
for d in /lib/modules/*/; do
  d=${d%/}; echo "installed: ${d##*/}"
done
```

### Reboot needed after kernel update?
If the running kernel's module folder is gone, or a newer one exists, you are running an old kernel.
```bash
running=$(uname -r)
newest=$(ls -v /lib/modules | tail -1)
if [[ ! -d /lib/modules/$running ]]; then
  echo "Modules of running kernel removed - reboot needed"
elif [[ $newest != "$running" ]]; then
  echo "Newer kernel installed ($newest) - reboot to use it"
else
  echo "OK"
fi
```
With several kernel flavours installed (e.g. LTS + latest) "newest" is only a hint.

### Read one parameter from the cmdline
```bash
param() { for w in $(< /proc/cmdline); do [[ $w == "$1="* ]] && echo "${w#*=}"; done; }
param root        # UUID=3f6c9e2a-...
param rootflags   # subvol=/@
```

### Resolve `root=UUID=...` to a device
```bash
uuid=$(param root); uuid=${uuid#UUID=}
readlink -f /dev/disk/by-uuid/$uuid     # /dev/nvme0n1p3
```

### Slowest boot steps
```bash
systemd-analyze blame | head -5
```

### Kernel log from the previous boot (after a crash)
```bash
journalctl -k -b -1 --no-pager | tail -50
```

# Find stuff

| I want...                         | Look at                                                        |
| --------------------------------- | -------------------------------------------------------------- |
| Kernel version running            | `uname -r`, `/proc/version`                                    |
| Installed kernels                 | `ls /lib/modules`                                              |
| Kernel files (actual boot copy)   | `ls /boot` (maybe `sudo`)                                      |
| Boot parameters used              | `/proc/cmdline`                                                |
| UEFI boot entries                 | `efibootmgr`                                                   |
| Bootloader in use                 | `bootctl status`, folders in `<ESP>/EFI/`                      |
| Boot timing                       | `systemd-analyze`                                              |
| Boot log                          | `journalctl -b` (this boot), `journalctl -b -1` (previous)     |
| UEFI variables                    | `/sys/firmware/efi/efivars/` → [[sys]]                         |

# See also
- [[proc]] — `/proc/cmdline`, `/proc/version`
- [[sys]] — `/sys/firmware/efi`
- [[dev]] — `/dev/disk/by-uuid`, how `root=UUID=` is found
- [[etc]] — `fstab`, `crypttab`, `modprobe.d`
- [[lib]] / [[usr]] — kernel modules in `/lib/modules`
- [[targets-boot]] — what systemd does after the kernel hands over
- `man 7 bootup`, `man 7 bootparam`, `man 7 boot`
- Kernel docs, boot parameters: <https://docs.kernel.org/admin-guide/kernel-parameters.html>
