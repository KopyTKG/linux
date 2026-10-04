Know how that i want to learn but i dont know where should i get it

# Directories
1. [[bin|/bin]] — user commands (often a symlink to `/usr/bin`)
2. [[boot|/boot]] — kernel, initramfs, bootloader
3. [[dev|/dev]] — device files (disks, terminals, `/dev/null`)
4. [[etc|/etc]] — system-wide configuration
5. [[home|/home]] — users' personal files and dotfiles
6. [[lib|/lib]] — shared libraries and kernel modules (often a symlink to `/usr/lib`)
7. [[lib64|/lib64]] — 64-bit libraries and the dynamic loader (often a symlink into `/usr`)
8. [[media|/media]] — removable media mount points
9. [[mnt|/mnt]] — temporary manual mounts
10. [[opt|/opt]] — self-contained third-party apps
11. [[proc|/proc]] — live process and kernel info (virtual)
12. [[root|/root]] — root user's home
13. [[run|/run]] — runtime data since boot (PIDs, sockets)
14. [[sbin|/sbin]] — admin commands (often a symlink into `/usr` on modern systems)
15. [[srv|/srv]] — data served by this machine
16. [[sys|/sys]] — live hardware and driver info (virtual)
17. [[tmp|/tmp]] — temporary files, wiped on reboot
18. [[usr|/usr]] — installed software, read-only data
19. [[var|/var]] — changing data: logs, caches, databases

# Topics
- [[networking|Networking]] — interfaces, IP and routing, DNS, network manager, ports, firewall, tools
  - [[interfaces]] · [[ip-routing]] · [[dns]] · [[network-manager]] · [[sockets-ports]] · [[firewall]] · [[tools]]
- [[systemd|Systemd]] — PID 1, units, services, timers, logs, boot
  - [[units]] · [[services]] · [[timers]] · [[journal]] · [[targets-boot]] · [[user-units]] · [[systemctl-cheatsheet]]
- [[users|Users]] — UID/GID, managing users, groups, sudo/su, login sessions, PAM
  - [[uid-gid]] · [[user-management]] · [[groups]] · [[sudo-su]] · [[login-sessions]] · [[pam]]
- [[permissions|Permissions]] — rwx, special bits, umask, ACLs, attributes, capabilities, SELinux/AppArmor
  - [[file-permissions]] · [[special-bits]] · [[umask]] · [[acl]] · [[attributes]] · [[capabilities]] · [[mac-selinux-apparmor]]
- [[processes|Processes]] — lifecycle, process tree, job control, signals, trap, priority, limits
  - [[process-lifecycle]] · [[process-tree]] · [[job-control]] · [[signals]] · [[sending-signals]] · [[trap]] · [[priority-scheduling]] · [[resource-limits]]
- [[storage|Disks & filesystems]] — block devices, partitions, filesystems, mounting, disk usage, btrfs, LVM/RAID/LUKS, swap, disk health
  - [[block-devices]] · [[partitions]] · [[filesystems]] · [[mounting]] · [[disk-usage]] · [[btrfs]] · [[lvm-raid-luks]] · [[swap]] · [[disk-health-io]]

# Important files
[[files|All important files]] — one note per file (`passwd`, `shadow`, `fstab`, `sudoers`, `hosts`, `/dev/null`, `.bashrc`, `config.fish`, ...)

# Where do I find ...?

| I'm looking for ...                     | Look in                                          |
| --------------------------------------- | ------------------------------------------------ |
| A program / command                     | [[usr]] `/usr/bin`, `~/.local/bin`, [[opt]]      |
| Where a command comes from              | `which cmd`, `type -a cmd`, `readlink -f $(which cmd)` |
| A shared library (`.so`)                | [[usr]] `/usr/lib`, `ldconfig -p`                |
| System config                           | [[etc]]                                          |
| My app config                           | [[home]] `~/.config`                             |
| Default config shipped by package       | [[usr]] `/usr/lib`, `/usr/share`                 |
| Logs                                    | [[var]] `/var/log`, `journalctl`                 |
| Disks and partitions                    | [[dev]] `/dev/nvme*`, `/dev/disk/by-uuid`, `lsblk` |
| Mounted drives / USB sticks             | [[run]] `/run/media/$USER`, [[mnt]], `findmnt`   |
| Running process info                    | [[proc]] `/proc/<pid>`                           |
| CPU / memory / load                     | [[proc]] `/proc/cpuinfo`, `/proc/meminfo`, `/proc/loadavg` |
| Hardware: temps, battery, network card  | [[sys]] `/sys/class/*`                           |
| Kernel settings (live)                  | [[proc]] `/proc/sys`, [[sys]]                    |
| Kernel settings (permanent)             | [[etc]] `/etc/sysctl.d`, `/etc/modprobe.d`       |
| Kernel + bootloader                     | [[boot]]                                         |
| Kernel modules                          | [[lib]] `/lib/modules/$(uname -r)`               |
| systemd units                           | [[usr]] `/usr/lib/systemd/system`, [[etc]] `/etc/systemd/system` |
| Caches / program state data             | [[var]] `/var/cache`, `/var/lib` |
| Sockets, PID files                      | [[run]]                                          |
| Temporary files                         | [[tmp]], [[var]] `/var/tmp`                      |
| Man pages, icons, fonts, .desktop files | [[usr]] `/usr/share`                             |
