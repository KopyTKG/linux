---
tags:
  - linux
  - files
  - index
aliases:
  - Important files
  - Files index
---
One note per file worth knowing. Directory notes like [[etc]] or [[var]] give the overview of a folder. These notes go deeper: exact format, field meanings, security traps and scripting recipes.

> [!tip] How to use this
> - Search the full path in Obsidian (`Ctrl+O`): every note has its real path as an alias, so typing `/etc/shadow` finds [[shadow]].
> - Files marked 🔒 are readable by root only. Their notes describe the format with fake examples.

# Users & auth

| Note           | Path                              | Purpose                                                   |
| -------------- | --------------------------------- | --------------------------------------------------------- |
| [[passwd]]     | `/etc/passwd`                     | User accounts: name, UID, home, shell                     |
| [[shadow]] 🔒  | `/etc/shadow`                     | Password hashes and password aging                        |
| [[group]]      | `/etc/group`                      | Groups and their extra members                            |
| [[gshadow]] 🔒 | `/etc/gshadow`                    | Group passwords and group admins                          |
| [[sudoers]] 🔒 | `/etc/sudoers`, `/etc/sudoers.d/` | Who may run what as root                                  |
| [[login-defs]] | `/etc/login.defs`                 | UID ranges, hash algorithm, umask for new users           |
| [[pam-d]]      | `/etc/pam.d/`                     | How every login and password check works                  |
| [[shells]]     | `/etc/shells`                     | Allowed login shells                                      |

# Boot & disks

| Note           | Path                              | Purpose                                                   |
| -------------- | --------------------------------- | --------------------------------------------------------- |
| [[fstab]]      | `/etc/fstab`                      | What gets mounted where at boot                           |
| [[crypttab]] 🔒| `/etc/crypttab`                   | Encrypted volumes to unlock at boot                       |
| [[mtab]]       | `/etc/mtab`, `/proc/self/mounts`, `mountinfo` | What is mounted right now                     |

# Network

| Note              | Path                                   | Purpose                                         |
| ----------------- | -------------------------------------- | ----------------------------------------------- |
| [[hosts]]         | `/etc/hosts`                           | Static hostname → IP table                      |
| [[resolv-conf]]   | `/etc/resolv.conf`                     | DNS servers the C library asks                  |
| [[nsswitch-conf]] | `/etc/nsswitch.conf`                   | Order of lookup sources (files, DNS, systemd)   |
| [[ssh-config]]    | `/etc/ssh/ssh_config`, `sshd_config`   | System SSH client and server settings           |
| [[ssh-user]]      | `~/.ssh/`                              | Your keys, `config`, `authorized_keys`, `known_hosts` |

# System identity

| Note              | Path                              | Purpose                                              |
| ----------------- | --------------------------------- | ---------------------------------------------------- |
| [[os-release]]    | `/etc/os-release`                 | Which distro and version (for scripts)               |
| [[hostname]]      | `/etc/hostname`                   | Machine name                                         |
| [[machine-id]]    | `/etc/machine-id`                 | Unique ID of this installation                       |
| [[locale-conf]]   | `/etc/locale.conf`                | Language and number/date formats                     |
| [[vconsole-conf]] | `/etc/vconsole.conf`              | Text console keyboard layout and font                |
| [[localtime]]     | `/etc/localtime` → `/usr/share/zoneinfo/...` | Time zone                                 |
| [[issue-motd]]    | `/etc/issue`, `/etc/motd`         | Messages before and after login                      |

# Libraries & system defaults

| Note             | Path                                   | Purpose                                        |
| ---------------- | -------------------------------------- | ---------------------------------------------- |
| [[ld-so-conf]]   | `/etc/ld.so.conf`, `ld.so.conf.d/`, `ld.so.cache` | Where the dynamic linker finds `.so` libraries |
| [[environment]]  | `/etc/environment`, `environment.d/`   | Variables for every login, any shell           |
| [[sysctl-d]]     | `/etc/sysctl.d/`, `/usr/lib/sysctl.d/` | Kernel settings applied at boot                |
| [[crontab]]      | `/etc/crontab`, `/etc/cron.d/`, `crontab -e` | Scheduled jobs (classic cron)            |

# Shell & user

| Note             | Path                                       | Purpose                                     |
| ---------------- | ------------------------------------------ | ------------------------------------------- |
| [[profile]]      | `/etc/profile`, `/etc/profile.d/`, `~/.profile` | Login shell setup (PATH, variables)    |
| [[bashrc]]       | `~/.bashrc`, `/etc/bash.bashrc`            | Interactive bash: aliases, prompt           |
| [[fish-config]]  | `~/.config/fish/`                          | fish config, functions, `fish_variables`    |
| [[skel]]         | `/etc/skel/`                               | Default files copied into new homes         |

# Logs

| Note                    | Path                                         | Purpose                                 |
| ----------------------- | -------------------------------------------- | --------------------------------------- |
| [[wtmp-btmp-lastlog]]   | `/var/log/wtmp`, `/var/log/btmp`, `/run/utmp`, `lastlog` | Who logged in, failed logins, who is on now |
| [[dmesg-kmsg]]          | `dmesg`, `/dev/kmsg`                         | Kernel messages (ring buffer)           |

# Kernel & devices

| Note                       | Path                                       | Purpose                                     |
| -------------------------- | ------------------------------------------ | ------------------------------------------- |
| [[dev-special-files]]      | `/dev/null`, `zero`, `urandom`, `full`, `tty` | Special device files used in scripts     |
| [[proc-self-highlights]]   | `/proc/self/*`, `/proc/sys/kernel/random/*` | Handy single files in `/proc`              |

# See also
- [[README]] — directory overview and "Where do I find ...?" table
- [[etc]] · [[var]] · [[dev]] · [[proc]] · [[home]]
- [[networking]] · [[systemd]]
