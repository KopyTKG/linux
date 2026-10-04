---
tags:
  - linux
  - filesystem
  - run
  - scripting
aliases:
  - /run
  - /var/run
---
`/run` is where running programs keep **runtime state**: PID files, sockets, lock files and other small files that only make sense while the system is up. It lives in RAM and is wiped on every boot.

> [!info] What `/run` really is
> `/run` is a `tmpfs`, a filesystem that lives in **RAM** (and swap), not on disk.
> ```
> tmpfs on /run type tmpfs (rw,nosuid,nodev,size=6550832k,mode=755,...)
> tmpfs on /run/user/1000 type tmpfs (rw,nosuid,nodev,size=3275412k,mode=700,uid=1000,gid=1000,...)
> ```
> - **Cleared on every boot**, so anything in it is temporary by design.
> - It is fast, and by default it can grow to about 20% of RAM (`df -h /run`).
> - It is mounted very early in boot, before `/var` or `/home` exist, which is why daemons can rely on it.
> - Files that need to exist on every boot are re-created by `systemd-tmpfiles` from `/usr/lib/tmpfiles.d/` and `/etc/tmpfiles.d/` → see [[etc]].

> [!tip] `/var/run` and `/var/lock` are symlinks
> `/var/run -> ../run` and `/var/lock -> ../run/lock`. Old scripts and docs use `/var/run/...`, but it is the same place. → [[var]]

# What lives here

| Path                          | Example                                   | What it is                                                  |
| ----------------------------- | ----------------------------------------- | ----------------------------------------------------------- |
| `/run/*.pid`                  | `/run/docker.pid` → `1443`                | PID file: the PID of a running daemon                       |
| `/run/*.sock`                 | `/run/docker.sock`                        | Unix socket the daemon listens on                           |
| `/run/<daemon>/`              | `/run/NetworkManager/`, `/run/cups/`      | Per-daemon runtime folder (sockets, state, PIDs)            |
| `/run/systemd/`               | `sessions/`, `users/`, `units/`, `journal/` | systemd runtime state                                     |
| `/run/dbus/system_bus_socket` |                                           | The **system** D-Bus                                        |
| `/run/udev/`                  | `data/`                                   | udev device database → [[dev]]                             |
| `/run/user/<UID>/`            | `/run/user/1000/`                         | Your private runtime dir (`$XDG_RUNTIME_DIR`)               |
| `/run/lock/`                  | `asound.state.lock`, `lvm/`               | Lock files (used to stop two programs touching one resource) |
| `/run/media/<user>/`          | `/run/media/alice/USB_STICK`               | Where udisks auto-mounts USB drives → compare [[mnt]], [[media]] |
| `/run/mount/`, `/run/credentials/` |                                      | systemd mount helpers and per-service secrets               |
| `/run/utmp`                   |                                           | Who is logged in right now (`who`, `w` read it)             |
| `/run/initramfs/`             |                                           | Leftovers from the early-boot initramfs → [[boot]]          |

# PID files
A PID file holds one line: the PID of a daemon. Scripts and init systems use it to find, check or kill that daemon.

```bash
cat /run/docker.pid                     # 1443
kill -0 "$(< /run/docker.pid)" && echo "docker is running"
```

> [!warning] PID files can lie
> If the daemon crashed, the file stays and the PID can later be **reused by another process**. Always check that the process behind the PID is the right one:
> `[[ $(< /proc/$pid/comm) == dockerd ]]` → see [[proc]].
> With systemd you rarely need PID files: `systemctl show -P MainPID docker` is more reliable.

# Sockets
A Unix socket is a file that two programs use to talk to each other. `ls -l` shows it with type `s`:

```
srw-rw-rw- 1 root root 0  /run/dbus/system_bus_socket
srw-rw-rw- 1 alice alice 0  /run/user/1000/bus
srw-rw-rw- 1 alice alice 0  /run/user/1000/pipewire-0
```

| Socket                              | Who listens       | Who uses it                                  |
| ----------------------------------- | ----------------- | -------------------------------------------- |
| `/run/dbus/system_bus_socket`       | `dbus-daemon`/`dbus-broker` (system) | `systemctl`, NetworkManager, polkit |
| `/run/docker.sock`                  | `dockerd`         | `docker` CLI                                 |
| `/run/systemd/journal/socket`       | `systemd-journald`| every service that logs                      |
| `/run/systemd/notify`               | systemd (PID 1)   | services saying "I'm ready"                  |
| `/run/systemd/io.systemd.*`         | systemd           | Varlink APIs (`hostnamectl`, `userdbctl`...) |
| `/run/user/1000/bus`                | your session D-Bus| desktop apps, notifications                  |
| `/run/user/1000/wayland-0`          | compositor (KWin) | every Wayland window                         |
| `/run/user/1000/pipewire-0`         | PipeWire          | audio and screen sharing                     |
| `/run/user/1000/pulse/native`       | pipewire-pulse    | apps using the PulseAudio API                |

> [!warning] Socket permissions = access
> Whoever can write to `/run/docker.sock` controls Docker, which is effectively **root** on the machine. Treat socket permissions as security settings.

# `/run/user/<UID>`
Your **private** runtime directory, created by `systemd-logind` when you log in and **deleted when your last session ends**. Mode `700`, owned by you, its own small tmpfs.

```bash
echo $XDG_RUNTIME_DIR    # /run/user/1000
echo $WAYLAND_DISPLAY    # wayland-0  → socket is $XDG_RUNTIME_DIR/wayland-0
id -u                    # 1000
```

| Entry                   | What                                              |
| ----------------------- | ------------------------------------------------- |
| `bus`                   | Session D-Bus socket (`$DBUS_SESSION_BUS_ADDRESS`) |
| `wayland-0`             | Wayland display socket                            |
| `pipewire-0`, `pulse/`  | Audio                                             |
| `systemd/`              | Your `systemctl --user` manager                   |
| `gnupg/`                | gpg-agent sockets                                 |
| `doc/`                  | Flatpak document portal (FUSE mount)              |
| `nvim.*`, `*.sock`      | Per-app sockets (Neovim, Alacritty...)            |

> [!tip] Use it in your own scripts
> Need a temp socket, lock or FIFO that must vanish at logout and nobody else can read? Put it in `$XDG_RUNTIME_DIR`, not in [[tmp]].

# `/run/lock`
Lock files mark "this resource is busy". The modern way is not to check if a file exists but to hold a **kernel lock** on it with `flock`. The lock is released automatically when the process dies, so it can't go stale.

```bash
exec 9> /run/lock/myscript.lock 2>/dev/null || exec 9> "$XDG_RUNTIME_DIR/myscript.lock"
flock -n 9 || { echo "already running"; exit 1; }
# ... only one copy of the script runs this part ...
```
`/run/lock` is writable for everyone only on some distros; the fallback to `$XDG_RUNTIME_DIR` covers that.

# Scripting recipes

### Is a daemon alive (from its PID file)?
```bash
pidfile=/run/docker.pid
if [[ -r $pidfile ]] && pid=$(< "$pidfile") && [[ -d /proc/$pid ]]; then
  echo "running as PID $pid ($(< /proc/$pid/comm))"
else
  echo "not running"
fi
```

### List all sockets in `/run`
```bash
find /run -maxdepth 3 -type s 2>/dev/null
```

### List all PID files and check them
```bash
find /run -maxdepth 2 -name '*.pid' 2>/dev/null | while read -r f; do
  pid=$(< "$f")
  if [[ -d /proc/$pid ]]; then state="alive ($(< /proc/$pid/comm))"; else state=STALE; fi
  echo "$f  $pid  $state"
done
```

### Who is logged in (from systemd state)
```bash
ls /run/systemd/users      # UIDs with a session: 1000
ls /run/systemd/sessions   # session IDs (loginctl show-session <id>)
```

### Run one copy of a script only
See the `flock` example above.

### Talk to Docker without the CLI
```bash
curl -s --unix-socket /run/docker.sock http://localhost/version
```

# Find stuff

| I want...                              | Look at                                   |
| -------------------------------------- | ----------------------------------------- |
| PID of a daemon                        | `/run/<name>.pid`, `/run/<name>/*.pid`, or `systemctl show -P MainPID <unit>` |
| Socket of a daemon                     | `/run/<name>.sock`, `/run/<name>/`        |
| My Wayland/audio/D-Bus sockets         | `$XDG_RUNTIME_DIR` = `/run/user/$(id -u)` |
| Where my USB stick got mounted         | `/run/media/$USER/`                       |
| Who holds a lock                       | `fuser /run/lock/<file>` or `lsof`        |
| Which tmpfiles rule made a file        | `systemd-tmpfiles --cat-config \| grep <path>` |
| Size of the RAM disks                  | `df -h /run /run/user/$(id -u)`           |

# See also
- [[var]] — persistent state (`/var/lib`, `/var/log`); `/var/run` is just a link here
- [[tmp]] — temporary files, but shared by all users
- [[proc]] — check that a PID from a PID file really exists
- [[dev]] — `/dev/shm` is another tmpfs, for shared memory
- [[etc]] — `/etc/tmpfiles.d/` creates files in `/run` on boot
- `man 5 tmpfiles.d`, `man 8 pam_systemd` (`$XDG_RUNTIME_DIR`), `man 1 flock`
- FHS: <https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch03s15.html>
