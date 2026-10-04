---
tags:
  - linux
  - users
  - systemd
aliases:
  - sessions
  - loginctl
  - who
  - tty
  - pts
  - XDG_RUNTIME_DIR
---
A **session** is one login: a console login on a virtual terminal, a graphical desktop, an SSH connection. On systemd systems, **systemd-logind** tracks sessions, users and seats, gives each user a runtime directory and decides who may access which devices.

# Terminals: tty vs pts

| Type               | Device             | What it is                                                    |
| ------------------ | ------------------ | ------------------------------------------------------------- |
| Virtual console    | `/dev/tty1`…`tty6` | Text consoles on the real screen, `Ctrl+Alt+F1`…`F6`          |
| Graphical session  | also a `tty`       | Desktop runs on one VT (here `tty2`)                          |
| Pseudo-terminal    | `/dev/pts/N`       | Terminal emulator windows, SSH, `tmux`, `screen`              |
| Serial             | `/dev/ttyS0`, `ttyUSB0` | Real serial lines                                        |
| Current terminal   | `/dev/tty`         | Always "my controlling terminal", whatever it is              |

```bash
tty                  # /dev/pts/3 in a terminal window, "not a tty" in a script/pipe
ls -l /dev/tty2      # crw------- 1 alice tty 4, 2 ... (owned by the logged-in user)
```
More in [[dev]].

# Who is logged in?

| Command      | Shows                                                          |
| ------------ | -------------------------------------------------------------- |
| `who`        | Current logins (from `/run/utmp`)                              |
| `who -b`     | Last boot time: `system boot  2026-10-04 16:57`                |
| `w`          | Logins + idle time + what they're running                      |
| `users`      | Just names                                                     |
| `last`       | Login history (from `/var/log/wtmp`)                           |
| `last reboot`| Boot history                                                   |
| `lastb`      | Failed logins (root, `/var/log/btmp`)                          |
| `lastlog` / `lastlog2` | Last login per user                                  |

```
$ w -h
alice    tty2      16:57   57:49   0.02s  0.02s /usr/bin/startplasma-wayland
$ last -n 2 alice
alice    pts/0        :0               Wed Sep 30 11:13 - 11:17  (00:04)
alice    pts/0        :0               Sun Sep 27 00:31 - 00:31  (00:00)
```
File formats are in [[wtmp-btmp-lastlog]].

> [!note] Not every terminal is a "login"
> Terminal emulator windows inside a desktop often don't register in utmp, so `who` can show only the desktop session or nothing at all. `loginctl` is the reliable source on systemd systems.

# `loginctl` — systemd-logind

Three objects:

| Object      | Meaning                                                           |
| ----------- | ----------------------------------------------------------------- |
| **session** | One login (numbered). Has a class (`user`, `greeter`, `manager`), type (`tty`, `x11`, `wayland`), seat, tty |
| **user**    | A UID with at least one session (or lingering)                    |
| **seat**    | A set of devices (screen, keyboard, mouse). Normal PC: just `seat0` |

```
$ loginctl list-sessions
SESSION  UID USER  SEAT  LEADER CLASS   TTY  IDLE SINCE
      2 1000 alice seat0 1297   user    tty2 no   -
      3 1000 alice -     1303   manager -    no   -

$ loginctl show-session 2 -p Type -p Class -p TTY -p Seat -p Remote
Type=wayland
Class=user
TTY=tty2
Seat=seat0
Remote=no
```
Session `3` with class `manager` is the user's `systemd --user` instance (`user@1000.service`), see [[user-units]].

| Command                               | Does                                        | Root? |
| ------------------------------------- | ------------------------------------------- | :---: |
| `loginctl` / `list-sessions`          | List sessions                               |       |
| `loginctl list-users`                 | List users with sessions                    |       |
| `loginctl user-status [USER]`         | Tree of all processes of a user             |       |
| `loginctl session-status [ID]`        | Details + processes of a session            |       |
| `loginctl show-user USER -p Linger`   | Property for scripts                        |       |
| `loginctl lock-session` / `unlock-session` | Lock screen                            |       |
| `loginctl terminate-session ID`       | Kill one session                            | other users |
| `loginctl terminate-user USER`        | Log a user out completely                   | other users |
| `loginctl enable-linger USER`         | Keep user services running without login    | other users |

# What logind does at login
1. PAM module `pam_systemd.so` (see [[pam]]) registers the session.
2. Creates `user-1000.slice` and a `session-N.scope` cgroup for the processes ([[sys|/sys/fs/cgroup]]).
3. Starts `user@1000.service` (the per-user systemd).
4. Creates and mounts **`/run/user/1000`** (tmpfs, mode `700`).
5. Gives the active session's user access to devices (sound, GPU, webcam) through ACLs, which is why desktop users don't need the `audio` / `video` groups anymore. See [[groups]] and [[acl]].
6. On the last logout: stops `user@1000.service` and removes `/run/user/1000`, unless **lingering** is on.

# `XDG_RUNTIME_DIR` — `/run/user/<uid>`
Per-user, per-boot directory for sockets and small runtime files. Private (`drwx------ alice alice`), on tmpfs, deleted at logout.

```
$ echo $XDG_RUNTIME_DIR
/run/user/1000
$ ls /run/user/1000
bus  pipewire-0  wayland-0  systemd/  dconf/  at-spi/ ...
```

| Entry           | What                                   |
| --------------- | -------------------------------------- |
| `bus`           | User D-Bus socket                      |
| `wayland-0`     | Wayland compositor socket              |
| `pipewire-0`    | PipeWire audio socket                  |
| `systemd/`      | User systemd manager state             |
| `gnupg/`, `keyring/` | Agents' sockets                   |

Other session variables set at login:
```
XDG_SESSION_ID=2
XDG_SESSION_TYPE=wayland
XDG_SESSION_CLASS=user
XDG_SEAT=seat0
XDG_VTNR=2
```
More in [[run]].

> [!tip] `sudo` / `su` don't give you a new session
> After `sudo -u alice -i` there's no `/run/user/<alice uid>` and no user D-Bus, so `systemctl --user` fails. Use `machinectl shell alice@` (or a real login) to get a full session.

# Lingering
Normally user services die when the user's last session ends. **Linger** keeps `user@UID.service` running from boot to shutdown, without any login: needed for user timers and services on a server.

```bash
loginctl show-user "$USER" -p Linger     # Linger=no on this machine
ls /var/lib/systemd/linger/              # one empty file per lingering user
loginctl enable-linger                   # turn on for yourself
```
See [[user-units]].

# Scripting recipes

### Am I in a graphical session?
```bash
case ${XDG_SESSION_TYPE:-tty} in
  wayland|x11) echo "graphical ($XDG_SESSION_TYPE)";;
  *)           echo "text console or SSH";;
esac
```

### Am I connected over SSH?
```bash
[[ -n $SSH_CONNECTION ]] && echo "remote from ${SSH_CONNECTION%% *}"
```

### Is the script running interactively (attached to a terminal)?
```bash
if [[ -t 0 && -t 1 ]]; then echo "interactive"; else echo "pipe/cron/service"; fi
```

### List sessions as "id user tty type"
```bash
for s in $(loginctl list-sessions --no-legend | awk '{print $1}'); do
  loginctl show-session "$s" -p Id -p Name -p Type -p TTY --value | paste -sd' '
done
```
```
2 alice tty2 wayland
3 alice  unspecified
```
(`--value` prints in systemd's internal property order, not the order you asked for.)

### Idle time of every logged-in user
```bash
w -h | awk '{print $1, $2, "idle:", $4}'
```

### Last boot and uptime
```bash
who -b
uptime -p
```

# Find stuff

| Question                         | Command / path                          |
| -------------------------------- | --------------------------------------- |
| My session ID / type             | `echo $XDG_SESSION_ID $XDG_SESSION_TYPE`|
| All sessions                     | `loginctl list-sessions`                |
| Processes of a user              | `loginctl user-status USER`             |
| Current logins (raw)             | `/run/utmp`                             |
| Login history (raw)              | `/var/log/wtmp`                         |
| logind config (idle, lid switch, KillUserProcesses) | `/etc/systemd/logind.conf` |
| Lingering users                  | `/var/lib/systemd/linger/`              |

# See also
- [[users]] · [[pam]] · [[sudo-su]] · [[groups]]
- [[user-units]] · [[systemd]] · [[run]] · [[dev]]
- [[wtmp-btmp-lastlog]]
- `man 1 loginctl`, `man 8 systemd-logind`, `man 1 w`, `man 1 last`
