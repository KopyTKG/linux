---
tags:
  - linux
  - systemd
  - boot
  - targets
aliases:
  - targets
  - boot process
  - runlevels
  - systemd-analyze
---
A **target** is a unit that does nothing by itself — it is a **named group of units** and a **synchronization point** during boot. "Reach `multi-user.target`" = start everything it wants, in the right order.

> [!info] Targets are the new runlevels
> Old SysV init had numbered runlevels (0–6). systemd replaced them with named targets. Recent systemd versions (this machine: 262) **no longer ship** the `runlevelN.target` aliases — use the names.

| Old runlevel | Target              | State                                         |
| ------------ | ------------------- | --------------------------------------------- |
| 0            | `poweroff.target`   | Shut down                                     |
| 1, `s`       | `rescue.target`     | Single user, root shell, basic system mounted |
| —            | `emergency.target`  | Even less: root mounted **read-only**, nothing else |
| 3            | `multi-user.target` | Full system, text login, network, no GUI      |
| 5            | `graphical.target`  | `multi-user` + display manager (GUI login)    |
| 6            | `reboot.target`     | Reboot                                        |

# Default target
```bash
$ systemctl get-default
graphical.target
```
`default.target` is just a symlink. Change it (described):
```bash
sudo systemctl set-default multi-user.target   # boot to text console from now on
sudo systemctl set-default graphical.target    # back to GUI
```
This creates `/etc/systemd/system/default.target` → overrides `/usr/lib/systemd/system/default.target`. See [[units]] for the precedence rules.

One-time (this boot only): add `systemd.unit=multi-user.target` to the kernel command line in the boot menu → see [[boot]], `/proc/cmdline` in [[proc]].

Switch **now** without reboot: `sudo systemctl isolate multi-user.target` (stops the GUI!).

# The boot chain
```
firmware (UEFI/BIOS)
  └─ bootloader                         → [[boot]]
      └─ kernel + initramfs              (initrd has its own small systemd)
          └─ switch to real root, exec /sbin/init = systemd (PID 1)
              └─ default.target → graphical.target
                  └─ multi-user.target
                      └─ basic.target
                          └─ sockets.target, timers.target, paths.target, slices.target
                              └─ sysinit.target
                                  └─ local-fs.target (mounts from /etc/fstab), swap.target,
                                     udev (devices), tmpfiles, journald …
```
All targets on one side are started **in parallel** as far as dependencies allow — that's why systemd boots fast.

### Important targets
| Target                   | Reached when …                                              |
| ------------------------ | ----------------------------------------------------------- |
| `sysinit.target`         | Early setup done: mounts, swap, udev, journald              |
| `basic.target`           | Sockets, timers, paths ready                                |
| `network.target`         | Network **stack** is configured (not necessarily online!)   |
| `network-online.target`  | Network actually **connected** (needs a wait-online service) → [[networking]] |
| `multi-user.target`      | All system services up                                      |
| `graphical.target`       | Display manager started                                     |
| `local-fs.target`        | All local filesystems from fstab mounted                    |
| `remote-fs.target`       | NFS/SMB mounts done                                         |
| `time-sync.target`       | Clock synchronized (for services that need correct time)    |
| `shutdown.target`        | Everything should stop                                      |

# How long did boot take? `systemd-analyze`
```
$ systemd-analyze
Startup finished in 8.089s (firmware) + 5.409s (loader) + 2.871s (kernel) + 4.362s (initrd) + 11.420s (userspace) = 32.153s
graphical.target reached after 11.382s in userspace.
```

| Phase       | Here     | What                                          |
| ----------- | -------- | --------------------------------------------- |
| firmware    | 8.089 s  | UEFI POST (RAM training, devices)             |
| loader      | 5.409 s  | Bootloader incl. menu timeout                 |
| kernel      | 2.871 s  | Kernel init until initramfs                   |
| initrd      | 4.362 s  | initramfs: find & mount root                  |
| userspace   | 11.420 s | systemd starting your services                |

### `systemd-analyze blame` — slowest units
```
1min 46.953s plocate-updatedb.service
      7.395s NetworkManager-wait-online.service
      5.139s dev-disk-by\x2duuid-0123456789ABCDEF.device
      ...
```
> [!warning] `blame` lies a bit
> It lists time per unit, but units start **in parallel**. `plocate-updatedb` took 1 min 47 s and did **not** delay boot at all (it runs in the background from a timer). Use `critical-chain` to see what actually delays you.

### `systemd-analyze critical-chain` — what boot waited for
```
graphical.target @11.382s
└─multi-user.target @11.382s
  └─docker.service @10.658s +723ms
    └─network-online.target @10.657s
      └─NetworkManager-wait-online.service @3.262s +7.395s
        └─NetworkManager.service @3.082s +178ms
          └─basic.target @3.079s
            └─dbus-broker.service @3.058s +21ms
              └─dbus.socket @3.054s +27us
                └─sysinit.target @3.048s
```
`@` = when it became active, `+` = how long it took to start.

Reading this real example: boot waited **7.4 s** for the network to come online, only because `docker.service` wants `network-online.target`. Without that, graphical.target would be reached ~7 s earlier.

```bash
systemd-analyze critical-chain docker.service   # chain for one unit
systemd-analyze plot > boot.svg                 # timeline picture, open in browser
systemd-analyze dot | dot -Tsvg > deps.svg      # dependency graph (needs graphviz)
```

# Rescue and emergency
When boot fails (bad fstab line, broken root), systemd drops into **emergency mode**. You can also go there on purpose:

| How                                  | Gets you                                         |
| ------------------------------------ | ------------------------------------------------ |
| Kernel parameter `systemd.unit=rescue.target` | Root shell, local filesystems mounted, no network |
| Kernel parameter `systemd.unit=emergency.target` | Root shell, `/` **read-only**, nothing else   |
| Kernel parameter `init=/bin/bash`    | No systemd at all, raw shell (last resort)       |
| `sudo systemctl rescue`              | Switch running system to rescue (drops GUI)      |

Typical repair in emergency mode (described):
1. Read the error: `journalctl -xb -p err`.
2. Make `/` writable: `mount -o remount,rw /`.
3. Fix the cause, e.g. the bad line in `/etc/fstab` → [[etc]] (add `nofail` to non-essential disks so a missing disk never blocks boot again).
4. `systemctl daemon-reload`, then `systemctl default` (continue booting) or `reboot`.

> [!tip] `nofail` saves boots
> An external/secondary disk in `/etc/fstab` without `nofail` → if it's unplugged, boot waits 90 s and then falls into emergency mode.

# Shutdown
`poweroff`, `reboot`, `systemctl suspend`, `systemctl hibernate` — all go through targets (`poweroff.target` …). Shutdown stops units in **reverse** order of `After=`.

# Scripting recipes
### Boot time in seconds only
```bash
systemd-analyze | head -1 | grep -oE '= .*'      # = 32.153s
```

### Units slowing boot more than 2 s
```bash
systemd-analyze blame --no-pager | awk '{
  t=0
  for (i=1; i<NF; i++) { v=$i
    if (v ~ /min$/) t += v*60; else if (v ~ /ms$/) t += v/1000
    else if (v ~ /us$/) t += v/1e6; else if (v ~ /s$/) t += v+0 }
  if (t > 2) printf "%7.1fs  %s\n", t, $NF }'
```
Times can be `1min 46.953s` (two words), `7.395s` or `723ms`, so the script converts every part to seconds. Output here: `107.0s plocate-updatedb.service`, `7.4s NetworkManager-wait-online.service` …

### Is the system fully booted?
```bash
systemctl is-system-running        # running / degraded (some unit failed) / starting / maintenance
```

### Script must wait for boot to finish
```bash
systemctl is-system-running --wait
```

# Find stuff
| Looking for                         | Command                                     |
| ----------------------------------- | ------------------------------------------- |
| Default target                      | `systemctl get-default`                     |
| All targets                         | `systemctl list-units --type=target`        |
| What a target pulls in              | `systemctl list-dependencies graphical.target` |
| Boot time                           | `systemd-analyze`                           |
| Slow units                          | `systemd-analyze blame`, `critical-chain`   |
| Boot errors                         | `journalctl -b -p err` → [[journal]]        |
| Previous boot errors                | `journalctl -b -1 -p err`                   |
| Special target docs                 | `man systemd.special`, `man bootup`         |

# See also
- [[boot]] — firmware, bootloader, kernel, initramfs
- [[units]] — `Wants=`/`After=`
- [[journal]] — boot logs
- [[networking]] — `network-online.target`
- `man bootup`, `man systemd.special`, `man systemd-analyze`
