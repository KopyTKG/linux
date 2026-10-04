---
tags:
  - linux
  - systemd
  - units
aliases:
  - unit files
  - systemd units
  - drop-ins
---
A **unit** is anything systemd manages: a service, a socket, a timer, a mount, a device… Each unit is described by a small INI-style text file, and its **type is the file extension**.

> [!info] Unit = file name
> `NetworkManager.service`, `fstrim.timer`, `boot.mount` — the name **is** the identity. Commands accept the name with or without `.service`: `systemctl status NetworkManager` works.

# Unit types
| Type          | What it represents                                          | Example on this machine                     |
| ------------- | ----------------------------------------------------------- | ------------------------------------------- |
| `.service`    | A program/daemon or one-shot command                        | `NetworkManager.service`, `docker.service`  |
| `.socket`     | A socket; starts the service when someone connects          | `dbus.socket`, `systemd-rfkill.socket`      |
| `.timer`      | Schedules another unit (usually a `.service`)               | `fstrim.timer`, `logrotate.timer`           |
| `.target`     | A group of units / sync point (like old runlevels)          | `multi-user.target`, `graphical.target`     |
| `.mount`      | A mounted filesystem (generated from `/etc/fstab`)          | `-.mount` (= `/`), `boot.mount`             |
| `.automount`  | Mount on first access                                       | `proc-sys-fs-binfmt_misc.automount`         |
| `.swap`       | Swap device/file                                            | zram swap                                   |
| `.path`       | Watches a file/dir, starts a unit on change                 | `cups.path` (starts CUPS when a print job appears) |
| `.slice`      | cgroup group for resource limits                            | `system.slice`, `user.slice`, `user-1000.slice` |
| `.scope`      | Externally started processes (not created by a unit file)  | `init.scope`, `session-2.scope`             |
| `.device`     | A device udev reports (auto-generated, no file)             | `dev-nvme0n1p3.device`                      |

> [!tip] Escaped names
> Paths become unit names: `/` → `-`, and a real `-` → `\x2d`.
> `/dev/disk/by-uuid/4A02…` → `dev-disk-by\x2duuid-4A02….device`.
> Convert with `systemd-escape -p /home/alice/data` → `home-alice-data`.

# Where unit files live
Searched **in this order — the first match wins** (`systemd-analyze unit-paths`):

| Path                              | Who writes it                          | Priority     |
| --------------------------------- | -------------------------------------- | ------------ |
| `/etc/systemd/system/`            | **You** (admin) → [[etc]]              | highest      |
| `/run/systemd/system/`            | Runtime, generators (gone on reboot) → [[run]] | middle |
| `/usr/local/lib/systemd/system/`  | Locally installed software             |              |
| `/usr/lib/systemd/system/`        | **Packages** (package manager) → [[usr]] | lowest       |

User units: `~/.config/systemd/user/` (yours) → `/usr/lib/systemd/user/` (packages). See [[user-units]].

> [!important] Never edit files in `/usr/lib/systemd/system/`
> A package update will overwrite them. Either copy the whole file to `/etc/systemd/system/` (full override) or, better, add a **drop-in** that changes only the lines you need.

# Drop-ins (overrides)
A directory named `<unit>.d/` with `*.conf` files is merged on top of the main unit file.

```
/usr/lib/systemd/system/foo.service          ← package file
/etc/systemd/system/foo.service.d/override.conf  ← your changes
```

```bash
sudo systemctl edit foo.service          # creates override.conf for you + reloads
sudo systemctl edit --full foo.service   # copy whole file to /etc to edit
systemctl cat foo.service                # show final file + all drop-ins
sudo systemctl revert foo.service        # delete your overrides
systemd-delta                            # list everything overridden on the system
```

Real example — `systemctl cat systemd-timesyncd` starts with the path of each piece:
```
# /usr/lib/systemd/system/systemd-timesyncd.service
[Unit]
Description=Network Time Synchronization
...
```
If there were drop-ins, each would follow with its own `# /etc/systemd/system/...d/override.conf` header.

> [!warning] Lists need a reset
> Settings like `ExecStart=` can appear multiple times. In a drop-in, assign it **empty first** to clear the original, otherwise you add a second command:
> ```ini
> [Service]
> ExecStart=
> ExecStart=/usr/bin/foo --new-flag
> ```

# Anatomy of a unit file
```ini
[Unit]                         # generic: description + dependencies
Description=Network Time Synchronization
Documentation=man:systemd-timesyncd.service(8)
After=systemd-resolved.service
Before=time-set.target sysinit.target shutdown.target
Wants=time-set.target
Conflicts=shutdown.target
ConditionVirtualization=!container

[Service]                      # type-specific section: [Service], [Timer], [Socket], [Mount], [Path]...
ExecStart=/usr/lib/systemd/systemd-timesyncd

[Install]                      # only used by `systemctl enable`
WantedBy=sysinit.target
```

### `[Unit]` keys
| Key               | Meaning                                                                    |
| ----------------- | -------------------------------------------------------------------------- |
| `Description=`    | Human name shown in `status` and logs                                      |
| `Documentation=`  | `man:` or `https://` links                                                 |
| `Wants=`          | Start these too; **don't care** if they fail (weak)                        |
| `Requires=`       | Start these too; **if they fail/stop, I stop** (strong)                    |
| `BindsTo=`        | Like `Requires=`, also stop when the other disappears (devices)            |
| `After=`          | **Order only**: start me after these                                       |
| `Before=`         | **Order only**: start me before these                                      |
| `Conflicts=`      | Starting me stops those                                                    |
| `Condition…=`     | Skip silently if false (`ConditionPathExists=`, `ConditionVirtualization=`) |
| `Assert…=`        | Like Condition, but fail loudly                                            |

> [!important] `Wants`/`Requires` ≠ `After`
> Dependencies and ordering are **separate**. `Wants=network-online.target` alone starts both **in parallel**. You almost always want the pair:
> ```ini
> Wants=network-online.target
> After=network-online.target
> ```

### `[Install]` keys
| Key            | Meaning                                                           |
| -------------- | ----------------------------------------------------------------- |
| `WantedBy=`    | `enable` creates a symlink in `<target>.wants/`                   |
| `RequiredBy=`  | Same, in `<target>.requires/`                                     |
| `Alias=`       | Extra name (e.g. `display-manager.service`)                       |
| `Also=`        | Enable these units together with me                              |

# How `enable` works
`enable` does **not** start anything. It only creates symlinks according to `[Install]`:

```bash
$ readlink /etc/systemd/system/multi-user.target.wants/docker.service
/usr/lib/systemd/system/docker.service
$ readlink /etc/systemd/system/display-manager.service
/usr/lib/systemd/system/<your-login-manager>.service   # Alias=display-manager.service (gdm, sddm, …)
```

When booting reaches `multi-user.target`, systemd starts everything in `multi-user.target.wants/`. Real content here:
```
avahi-daemon.service  cups.path  cups.service  docker.service
NetworkManager.service  remote-fs.target  ufw.service  ...
```

| Command                    | Effect                                                        |
| -------------------------- | ------------------------------------------------------------- |
| `enable foo`               | Create symlinks (start on next boot)                          |
| `enable --now foo`         | Symlinks **and** start now                                    |
| `disable foo`              | Remove symlinks                                               |
| `mask foo`                 | Link to `/dev/null` — cannot be started at all, even manually |
| `unmask foo`               | Undo mask                                                     |
| `start foo`                | Start now, not at boot                                        |

`is-enabled` states: `enabled`, `disabled`, `static` (no `[Install]`), `masked`, `alias`, `indirect`, `generated` (made by a generator from fstab etc.).

# Templates (`@` units)
`foo@.service` is a template; `foo@bar.service` is an instance and `%i` inside the file becomes `bar`.
Examples: `getty@tty1.service` (login prompt on tty1), `user@1000.service` (user manager of UID 1000), `systemd-fsck@….service`.

| Specifier | Value                        |
| --------- | ---------------------------- |
| `%i`      | Instance name (`tty2`)       |
| `%n`      | Full unit name               |
| `%h`      | User's home directory        |
| `%u`      | User name                    |
| `%t`      | Runtime dir (`/run` or `/run/user/UID`) |

# Scripting recipes
### Where is a unit file and is it overridden?
```bash
u=NetworkManager
systemctl show -P FragmentPath "$u"     # /usr/lib/systemd/system/NetworkManager.service
systemctl show -P DropInPaths  "$u"     # empty = no overrides
```

### List all units you enabled by hand (differ from vendor preset)
```bash
systemctl list-unit-files --state=enabled --no-legend --plain |
  awk '$3 == "disabled" {print $1}'
```
Columns are `UNIT STATE PRESET`; preset `disabled` but state `enabled` = you (or an installer) turned it on.

### What does a target pull in?
```bash
systemctl list-dependencies multi-user.target          # tree
systemctl list-dependencies --reverse docker.service   # who wants docker
```

### Check a unit file for mistakes before installing
```bash
systemd-analyze verify ./hello.service && echo "verify ok"
```

# Find stuff
| Looking for                   | Command                                              |
| ----------------------------- | ---------------------------------------------------- |
| All unit files + state        | `systemctl list-unit-files`                          |
| Loaded units (in memory)      | `systemctl list-units --all`                         |
| One type                      | `systemctl list-units --type=timer`                  |
| File path of a unit           | `systemctl show -P FragmentPath <unit>`              |
| All overrides on system       | `systemd-delta`                                      |
| Search paths                  | `systemd-analyze unit-paths`                         |

# See also
- [[services]] — the `[Service]` section in detail
- [[targets-boot]] — targets and the boot chain
- [[systemctl-cheatsheet]]
- [[etc]], [[usr]], [[run]] — where the files live
- `man systemd.unit`, `man systemd.special`, `man systemd.syntax`
