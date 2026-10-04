---
tags:
  - linux
  - filesystem
  - etc
  - config
  - scripting
aliases:
  - /etc
  - etc directory
---
Etc directory is where the system keeps its **configuration**. Users, mounted disks, hostname, network, services, package manager, boot image — if you want to change *how* the system behaves, the file is almost always somewhere in `/etc`.

> [!info] What `/etc` really is
> - Plain text files on disk. No binaries live here (those are in [[bin]] / [[usr]]).
> - Owned by `root`. Everyone can read most of it, only root can write. Secrets (`shadow`, `sudoers`) are readable only by root.
> - Host specific: two machines with the same packages differ mostly in `/etc`. Back it up.
> - A typical `/etc` has 200+ entries. You don't need to know all of them — learn the ones below.

> [!important] `/etc` overrides `/usr/lib` defaults
> Modern packages (systemd, udev, sysctl, modprobe, tmpfiles, ...) ship their **defaults** in `/usr/lib/...` and read **your overrides** from `/etc/...`.
>
> | Default from package (don't edit) | Your override (edit this) |
> | --------------------------------- | ------------------------- |
> | `/usr/lib/systemd/system/`        | `/etc/systemd/system/`    |
> | `/usr/lib/sysctl.d/`              | `/etc/sysctl.d/`          |
> | `/usr/lib/modprobe.d/`            | `/etc/modprobe.d/`        |
> | `/usr/lib/udev/rules.d/`          | `/etc/udev/rules.d/`      |
> | `/usr/lib/tmpfiles.d/`            | `/etc/tmpfiles.d/`        |
>
> A file in `/etc` with the **same name** replaces the one in `/usr/lib`. A different name is merged in, and files are read in **alphabetical order** — that's why names start with numbers (`50-default.conf`, `99-my.conf`): higher number wins.
> Package updates overwrite `/usr/lib`, never your files in `/etc`.

> [!tip] Golden rules for editing `/etc`
> 1. Make a backup first: `sudo cp file file.bak`.
> 2. Prefer a **drop-in** (`something.d/99-mine.conf`) over editing the main file. Updates won't conflict with it.
> 3. Use the safe editors where they exist: `visudo` for sudoers, `vipw` / `vigr` for passwd and group, `systemctl edit` for units.
> 4. After a change, reload the program: `systemctl daemon-reload`, `sysctl --system`, `udevadm control --reload`, `mount -a`, ...
> 5. Don't **read** users/hosts with `cat /etc/passwd` in scripts — use `getent` (see below). It also sees users from systemd, LDAP, etc.

# Users and groups

### `/etc/passwd`
One user per line, 7 fields separated by `:`. Readable by everyone. Despite the name, it has **no passwords** (`x` = "look in shadow").

```
root:x:0:0::/root:/usr/bin/bash
bin:x:1:1::/:/usr/bin/nologin
nobody:x:65534:65534:Kernel Overflow User:/:/usr/bin/nologin
alice:x:1000:1000:Alice Example:/home/alice:/bin/bash
```

| #   | Field     | Example          | Meaning                                                      |
| --- | --------- | ---------------- | ------------------------------------------------------------ |
| 1   | name      | `alice`           | Login name                                                   |
| 2   | password  | `x`              | Always `x` → real hash is in `/etc/shadow`                   |
| 3   | UID       | `1000`           | User ID. `0` = root, `1`–`999` = system, `1000+` = humans    |
| 4   | GID       | `1000`           | Primary group ID                                             |
| 5   | GECOS     | `Alice Example` | Full name / comment                                          |
| 6   | home      | `/home/alice`     | Home directory → see [[home]], root's is [[root]]            |
| 7   | shell     | `/bin/bash`      | Login shell. `nologin` = can't log in (service accounts)     |

### `/etc/group`
4 fields: `name:password:GID:members`.

```
wheel:x:998:alice
audio:x:995:alice
video:x:983:alice
alice:x:1000:
```

> [!note]
> The member list (field 4) holds only **extra** members. User `alice` is in group `alice` because it's the primary GID in `passwd`, not because it's listed here.
> The admin group allowed to use `sudo` differs between distros: usually `wheel` or `sudo`. Check which one `/etc/sudoers` names.

### `/etc/shadow`
Password hashes. Permissions `-rw------- root root` — only root can read it. 9 fields:

| #   | Field        | Meaning                                                        |
| --- | ------------ | -------------------------------------------------------------- |
| 1   | name         | Login name                                                     |
| 2   | hash         | `$y$...` (yescrypt), `$6$...` (SHA-512), `!` or `*` = locked   |
| 3   | last change  | Days since 1970-01-01 when password was last changed           |
| 4   | min          | Days before password may be changed again                      |
| 5   | max          | Days until password must be changed                            |
| 6   | warn         | Days of warning before expiry                                  |
| 7   | inactive     | Days after expiry until account is disabled                    |
| 8   | expire       | Day the account expires                                        |
| 9   | reserved     | Unused                                                         |

`/etc/gshadow` is the same idea for groups.

> [!warning] Don't edit by hand
> Use `passwd`, `useradd`, `usermod -aG group user`, `chage`, or `vipw` / `vigr` (they lock the file and check syntax). A broken `passwd`/`shadow` = nobody can log in.

### `getent` — the right way to query
`getent` asks NSS (see `nsswitch.conf` below), so it sees every source, not only the file.

```bash
getent passwd alice     # alice:x:1000:1000:Alice Example:/home/alice:/bin/bash
getent group wheel     # wheel:x:998:alice
getent hosts localhost # ::1  localhost
id alice                # uid, gid and all groups
```

# Disks: `/etc/fstab`
"File system table" — what to mount where at boot. 6 fields per line.

```
UUID=ABCD-1234                             /boot      vfat   umask=0077                         0 2
UUID=7d2e4f6a-1b3c-4d5e-8f9a-0b1c2d3e4f5a  swap       swap   defaults                           0 0
UUID=3f6c9e2a-8b1d-4c7e-9a5f-2d4b6e8f0a1c  /          btrfs  subvol=/@,noatime,compress=zstd:1  0 0
UUID=9b8a7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d  /home      btrfs  noatime,compress=zstd:1            0 0
tmpfs                                      /tmp       tmpfs  noatime,mode=1777                  0 0
LABEL=Data                                 /data      ext4   nofail,users,exec                  0 0
```

| #   | Field      | Example                    | Meaning                                                                |
| --- | ---------- | -------------------------- | ---------------------------------------------------------------------- |
| 1   | device     | `UUID=3f6c...`, `LABEL=Data` | What to mount. Use `UUID=` — `/dev/sda1` names can change between boots |
| 2   | mountpoint | `/home`                    | Where. `swap` for swap                                                 |
| 3   | type       | `btrfs`, `vfat`, `xfs`     | Filesystem type                                                        |
| 4   | options    | `noatime,compress=zstd:1`  | Mount options, comma separated                                         |
| 5   | dump       | `0`                        | Old backup flag, always `0`                                            |
| 6   | pass       | `0`, `1`, `2`              | fsck order at boot. `0` = skip, `1` = root, `2` = others               |

| Option     | Meaning                                                     |
| ---------- | ----------------------------------------------------------- |
| `defaults` | `rw,suid,dev,exec,auto,nouser,async`                        |
| `noatime`  | Don't write "last access time" on every read (faster, SSD)  |
| `nofail`   | Don't stop boot if the disk is missing (external drives)    |
| `users`    | Normal users may mount/unmount it                           |
| `subvol=`  | btrfs subvolume to mount                                    |
| `umask=`   | Permissions for filesystems without Unix perms (FAT)        |

```bash
lsblk -f                  # shows UUID and LABEL of every partition
sudo findmnt --verify     # check fstab for errors
sudo mount -a             # mount everything from fstab (test after editing!)
```

> [!warning]
> A wrong line in `fstab` can drop you into emergency mode at boot. Always run `sudo findmnt --verify` and `sudo mount -a` before rebooting. Add `nofail` to non-essential disks.
> What is *currently* mounted is in `/proc/mounts` → see [[proc]]. Manual mount points go to [[mnt]] or `/run/media` → see [[media]].

# Network and name

| File                  | Example                                     | Meaning                                     |
| --------------------- | ------------------------------------------- | ------------------------------------------- |
| `/etc/hostname`       | `myhost`                                    | Machine name. Change with `hostnamectl hostname NAME` |
| `/etc/hosts`          | `127.0.1.1  myhost`                         | Static name → IP table, checked before DNS  |
| `/etc/resolv.conf`    | `nameserver 127.0.0.53`                     | DNS servers                                 |
| `/etc/nsswitch.conf`  | `hosts: mymachines resolve ... files myhostname dns` | Order in which sources are asked   |
| `/etc/NetworkManager/`| connections in `system-connections/`        | Saved Wi‑Fi / Ethernet profiles (root only) |

### `/etc/hosts`
```
127.0.0.1  localhost
::1        localhost ip6-localhost ip6-loopback
127.0.1.1  myhost
```
Format: `IP  name [aliases...]`. Handy for blocking (`0.0.0.0 ads.example.com`) or naming LAN machines (`192.168.1.10 nas`).

### `/etc/resolv.conf`
```
nameserver 127.0.0.53
options edns0 trust-ad
```
`127.0.0.53` = **systemd-resolved** runs a local DNS stub. The real upstream servers: `resolvectl status`.

> [!note]
> On many systems `resolv.conf` is a symlink generated by resolved or NetworkManager. Editing it by hand gets overwritten — configure DNS in NetworkManager or `/etc/systemd/resolved.conf` instead.

### `/etc/nsswitch.conf`
"Name Service Switch" — for each database, the list of sources asked in order.

```
passwd: files systemd
group: files [SUCCESS=merge] systemd
hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns
```
`files` = the `/etc` file (`passwd`, `hosts`), `systemd`/`resolve`/`myhostname` = systemd modules. This is why `getent` finds more than `cat /etc/passwd`.

# System identity

### `/etc/os-release`
Which distro this is. Shell `KEY=value` format, so you can `source` it. The keys are the same everywhere; the values below are an example and differ on every distro.

```
NAME="Debian GNU/Linux"
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
ID=debian
VERSION_ID="12"
VERSION_CODENAME=bookworm
```

```bash
. /etc/os-release; echo "$PRETTY_NAME ($ID $VERSION_ID)"   # Debian GNU/Linux 12 (bookworm) (debian 12)
```
Use `ID` / `ID_LIKE` in scripts when you need to know which distro (or distro family) you are on. Every systemd-based distro has this file.

### Locale and time

| File               | Example                                            | Meaning                                    |
| ------------------ | -------------------------------------------------- | ------------------------------------------ |
| `/etc/locale.gen`  | `de_DE.UTF-8 UTF-8`, `en_US.UTF-8 UTF-8`           | Which locales to build (`sudo locale-gen`) |
| `/etc/locale.conf` | `LANG=en_US.UTF-8`, `LC_TIME=de_DE.UTF-8`          | Which locale is used                       |
| `/etc/localtime`   | symlink → `/usr/share/zoneinfo/Europe/Berlin`      | Time zone                                  |
| `/etc/vconsole.conf` | keymap/font of text console (TTY)                | Console keyboard layout                    |

`LANG` = default for everything, `LC_*` = override one category. Here: English messages, but German date, number and currency format.

```bash
localectl status
timedatectl                                   # time zone, NTP sync
sudo timedatectl set-timezone Europe/Berlin   # changes the /etc/localtime symlink
```

# Permissions: `/etc/sudoers`
Who can run what as root. Mode `-r--r----- root root` — you can't even read it without sudo.

Typical lines:
```
root ALL=(ALL:ALL) ALL
%wheel ALL=(ALL:ALL) ALL          # % = group. Members of wheel can sudo anything
alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```
Format: `who  host=(as_user:as_group)  [NOPASSWD:] commands`.

> [!warning] Always use `visudo`
> `sudo visudo` (or `sudo visudo -f /etc/sudoers.d/mine`) checks syntax before saving. A typo in `sudoers` saved with a normal editor = **no sudo at all**, and you need a live USB to fix it.
> Put your rules into `/etc/sudoers.d/` (drop-in), keep the main file untouched.

# Services: `/etc/systemd`
systemd units (services, timers, mounts, ...) come from two places:

| Path                                | Who writes it    | Priority   |
| ----------------------------------- | ---------------- | ---------- |
| `/usr/lib/systemd/system/` (716 files here) | packages | lowest     |
| `/etc/systemd/system/`              | you / `systemctl`| higher     |
| `/run/systemd/system/`              | runtime, lost on reboot → see [[run]] | highest |
| `~/.config/systemd/user/`, `/etc/systemd/user/` | user services | — |

### Enabled units are just symlinks
`systemctl enable foo` creates a symlink in `<target>.wants/`:

```
/etc/systemd/system/multi-user.target.wants/
├── ananicy-cpp.service  -> /usr/lib/systemd/system/ananicy-cpp.service
├── avahi-daemon.service -> /usr/lib/systemd/system/avahi-daemon.service
├── coolercontrold.service -> /usr/lib/systemd/system/coolercontrold.service
└── cups.path            -> /usr/lib/systemd/system/cups.path
```
"Start this when `multi-user.target` is reached". `disable` removes the symlink. Other examples here: `display-manager.service` (login screen), `timers.target.wants/`, `bluetooth.target.wants/`.

### Drop-ins (override part of a unit)
Don't copy the whole unit. Create `/etc/systemd/system/<unit>.d/override.conf` with only the changed lines:

```bash
sudo systemctl edit docker.service    # opens editor, creates the drop-in for you
```
```ini
# /etc/systemd/system/docker.service.d/override.conf
[Service]
Environment=HTTP_PROXY=http://proxy:3128
```
```bash
systemctl cat docker.service          # shows unit + all drop-ins merged
sudo systemctl daemon-reload          # after editing by hand
```

Other systemd config: `journald.conf` (log size), `logind.conf` (lid close, power button), `resolved.conf`, `system.conf`.

# Kernel and boot

### `/etc/sysctl.d/` — kernel settings at boot
Same settings as `/proc/sys` (see [[proc]]), but **permanent**. Often empty — the distro ships its own tuning as files in `/usr/lib/sysctl.d/`, and files in `/etc/sysctl.d/` with the same name override them.

```bash
echo 'vm.swappiness = 10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system     # apply all files now
```

### `/etc/modprobe.d/` — kernel module options
Options for drivers, or blacklisting them. Example: `blacklist-nouveau.conf` containing `blacklist nouveau`.

```
options snd_hda_intel power_save=1        # pass parameter to module
blacklist nouveau                          # never auto-load this module
```
Changes apply when the module is loaded next. If the module is loaded early from the initramfs, you must rebuild the initramfs (the tool differs per distro) and reboot → see [[boot]].

# Config files after package updates
When an update ships a new default config but you have **modified** the old one, package managers don't overwrite your file. They save the new version next to it with an extra suffix (the name depends on the package manager, e.g. `.new`, `.dist`, or a manager-specific one), or keep your old one as a backup.

> [!tip]
> After big updates, look for such files and merge them. Ignoring them for months = missing new options, or a broken service. Find recently created ones with:
> ```bash
> find /etc -type f -newermt '-1 day' 2>/dev/null
> ```

# Shells and login environment

| File                    | Read by                                | Use it for                                  |
| ----------------------- | -------------------------------------- | ------------------------------------------- |
| `/etc/shells`           | `chsh`, login                          | List of allowed login shells                |
| `/etc/environment`      | PAM, at every login (any shell)        | Simple `KEY=value`, no `$VAR` expansion     |
| `/etc/profile`          | bash/zsh/sh **login** shells           | System PATH etc. Sources `profile.d/*.sh`   |
| `/etc/profile.d/*.sh`   | via `/etc/profile`                     | Package snippets (`gawk.sh`, `freetype2.sh`, ...) |
| `/etc/bash.bashrc`      | interactive bash                       | System bash aliases/prompt                  |
| `/etc/zsh/`             | zsh                                    | `zshenv`, `zprofile`, `zshrc`               |
| `/etc/fish/`            | fish                                   | `config.fish`, `conf.d/` (fish ignores `/etc/profile`!) |
| `/etc/skel/`            | `useradd -m`                           | Template copied into every **new** home     |

Example `/etc/environment`:
```
EDITOR=nvim
LANG=en_US.UTF-8
```
`/etc/skel` typically contains `.bash_logout .bash_profile .bashrc .config .zshrc` — every new user starts with copies of these.

> [!note] Fish users
> Fish does **not** read `/etc/profile` or `profile.d/*.sh` (they're bash syntax). System-wide fish config goes into `/etc/fish/conf.d/*.fish`; per-user into `~/.config/fish/`. `/etc/environment` still works because PAM reads it, not the shell.

```bash
chsh -s /usr/bin/fish   # change your login shell (must be listed in /etc/shells)
```

# Remote access: `/etc/ssh`

| File                          | Meaning                                                    |
| ----------------------------- | ---------------------------------------------------------- |
| `ssh_config`, `ssh_config.d/` | **Client** defaults (`ssh` command). Per user: `~/.ssh/config` |
| `sshd_config`, `sshd_config.d/` | **Server** (`sshd`) config                               |
| `ssh_host_*_key`              | Server identity keys (private ones root only)              |
| `moduli`                      | Diffie-Hellman groups                                      |

```
Include /etc/ssh/sshd_config.d/*.conf
AuthorizedKeysFile	.ssh/authorized_keys
Subsystem	sftp	/usr/lib/ssh/sftp-server
```
Typical hardening drop-in `/etc/ssh/sshd_config.d/10-secure.conf`:
```
PasswordAuthentication no
PermitRootLogin no
```
```bash
sudo sshd -t && sudo systemctl reload sshd   # test config, then apply
```

# Hardware: udev and X11

### `/etc/udev/rules.d/`
Rules that run when a device appears: set permissions, create symlinks, run scripts. Defaults in `/usr/lib/udev/rules.d/`. Example: `99-usb-serial.rules` giving your user access to a USB serial adapter.

```
# give users access to an Arduino and create /dev/arduino
SUBSYSTEM=="tty", ATTRS{idVendor}=="2341", MODE="0666", SYMLINK+="arduino"
```
```bash
udevadm info -a /dev/ttyACM0                          # attributes you can match on
sudo udevadm control --reload && sudo udevadm trigger # apply new rules
```
Devices themselves appear in [[dev]], their attributes in [[sys]].

### `/etc/X11/`
- `xorg.conf.d/` — Xorg settings. Here: `00-keyboard.conf` (keyboard layout, written by `localectl set-x11-keymap`).
- `xinit/` — `xinitrc` for `startx`.
On Wayland (KDE Plasma / GNOME) most of this is unused.

# Scheduled jobs: cron
| Path                   | Meaning                                         |
| ---------------------- | ----------------------------------------------- |
| `/etc/crontab`         | System crontab (has extra *user* column)        |
| `/etc/cron.d/`         | Package cron files                              |
| `/etc/cron.hourly/` … `cron.daily/`, `weekly/`, `monthly/` | Scripts run on that schedule |

Cron line: `min hour day month weekday [user] command`, e.g. `30 2 * * * root /usr/local/bin/backup.sh`.

> [!note]
> Not every system has a cron daemon installed (this one has only `cron.hourly`, no daemon). On systemd systems the modern way is **systemd timers**: `systemctl list-timers` → see [[timers]].

# Other files worth knowing

| File / dir            | What                                                      |
| --------------------- | --------------------------------------------------------- |
| `/etc/machine-id`     | Unique ID of this install (used by systemd, journald)     |
| `/etc/ld.so.conf(.d)` | Extra library search paths → see [[lib]]                  |
| `/etc/ld.so.cache`    | Binary cache built by `ldconfig`                          |
| `/etc/default/`       | Defaults for some tools (`grub`, `useradd`)               |
| `/etc/security/limits.conf` | `ulimit` per user/group (see `/proc/<pid>/limits`)  |
| `/etc/pam.d/`         | How login/sudo authenticate                               |
| `/etc/tmpfiles.d/`    | Create/clean files in [[tmp]], [[run]], [[var]] at boot   |
| `/etc/mime.types`     | File extension → MIME type                                |
| `/etc/issue`          | Text shown above the TTY login prompt                     |
| `/etc/motd`           | "Message of the day" after login                          |
| `/etc/vconsole.conf`  | TTY keymap and font                                       |

# Scripting recipes
All recipes tested in `bash`.

### List human users
```bash
awk -F: '$3>=1000 && $3<60000 {printf "%-10s uid=%-5s home=%-12s shell=%s\n",$1,$3,$6,$7}' /etc/passwd
# alice       uid=1000  home=/home/alice   shell=/bin/bash
```

### Groups a user is listed in
```bash
awk -F: -v u=alice '$4 ~ "(^|,)" u "(,|$)" {print $1}' /etc/group | paste -sd' '
# sys network wheel audio lp storage uucp video users rfkill ...
```
(Or simply `id -nG alice`, which also includes the primary group.)

### Check user exists (works with all NSS sources)
```bash
if getent passwd "$1" >/dev/null; then echo "exists"; else echo "no such user"; fi
```

### Detect distro
```bash
. /etc/os-release
echo "Name:   $PRETTY_NAME"
echo "ID:     $ID"
echo "Family: ${ID_LIKE:-none}"
echo "Version: ${VERSION_ID:-rolling}"
```
Run it in a subshell `( . /etc/os-release; ... )` if you don't want its variables leaking into your script.

### Current time zone
```bash
readlink /etc/localtime | sed 's|.*/zoneinfo/||'   # Europe/Berlin
```

### fstab without comments
```bash
grep -vE '^\s*(#|$)' /etc/fstab | awk '{printf "%-12s %-6s %s\n", $2, $3, $4}'
```

### Enabled systemd units (from the symlinks)
```bash
find /etc/systemd/system -type l -path '*.wants/*' -printf '%f\n' | sort -u
```
Same info, official way: `systemctl list-unit-files --state=enabled`.

### Recently changed configs
```bash
find /etc -type f -newermt '-7 days' 2>/dev/null     # changed in last 7 days
```

### Back up all of `/etc`
```bash
sudo tar czf ~/etc-backup-$(date +%F).tar.gz -C / etc
```

# Find stuff
"Where is the config for X?" — try in this order:

1. **`/etc/<program>.conf`** or **`/etc/<program>/`** — e.g. `/etc/fstab`, `/etc/ssh/`.
2. **Drop-in dir** `/etc/<program>.d/` or `/etc/<program>/<something>.d/`.
3. **Package defaults** in `/usr/lib/<program>/`, `/usr/share/<program>/` → see [[usr]].
4. **Per user** config: `~/.config/<program>/` or `~/.<program>rc` → see [[home]].

```bash
sudo grep -rl 'swappiness' /etc /usr/lib/sysctl.d   # which file sets something
man 5 fstab                                # section 5 = file formats (passwd, fstab, sudoers, ...)
systemctl cat sshd                         # where a unit and its drop-ins live
```

> [!tip]
> `man 5 <filename>` exists for almost every file here: `man 5 passwd`, `man 5 shadow`, `man 5 fstab`, `man 5 sudoers`, `man 5 os-release`, `man 5 sysctl.d`, `man 5 systemd.unit`, `man 5 crontab`.

# See also
- [[usr]] — where packages put their default configs (`/usr/lib`, `/usr/share`)
- [[proc]] — `/proc/sys` = live kernel settings, `/etc/sysctl.d` = permanent ones
- [[boot]] — kernel, initramfs, bootloader
- [[home]] — per-user configs in `~/.config`
- [[run]] — runtime state; `/run/systemd/system` overrides `/etc` until reboot
- [[var]] — where services keep data and logs (configs stay in `/etc`)
- `man 7 hier`, `man 7 file-hierarchy` — official layout description
- FHS spec: <https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch03.html>
