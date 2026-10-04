---
tags:
  - linux
  - users
  - security
aliases:
  - sudo
  - su
  - becoming root
  - pkexec
  - doas
---
Linux has several ways to run something **as another user** (almost always root). They differ in *whose* password you type, *what environment* you get, and *how fine-grained* the permission is.

| Tool       | Password asked     | Configured in                      | Typical use                          |
| ---------- | ------------------ | ---------------------------------- | ------------------------------------ |
| `su`       | **target** user's  | PAM (`/etc/pam.d/su`)              | Become another user completely       |
| `sudo`     | **your own**       | [[sudoers|/etc/sudoers]] + PAM     | Run admin commands, logged, per-command rules |
| `pkexec`   | your own (via polkit agent) | polkit rules              | GUI apps asking for admin rights     |
| `doas`     | your own           | `/etc/doas.conf`                   | Small, simple sudo alternative       |
| `run0`     | your own (via polkit) | polkit                          | systemd's sudo alternative (newer systems) |

All of them are **setuid root** programs (or ask a root service to act for them). That is the only way a normal process can gain root:
```
-rwsr-xr-x 1 root root 285880 /usr/bin/sudo
-rwsr-xr-x 1 root root  55456 /usr/bin/su
-rwsr-xr-x 1 root root  26720 /usr/bin/pkexec
```
See [[special-bits]].

# `su` — switch user

| Command            | Result                                                                  |
| ------------------ | ----------------------------------------------------------------------- |
| `su`               | Become root, **keep** your environment and current directory            |
| `su -` / `su -l`   | Become root with a **login shell**: root's env, `PATH`, home dir, profile |
| `su - alice`       | Become alice (login shell)                                              |
| `su -c 'cmd'`      | Run one command as root                                                 |
| `su -s /bin/bash svc` | Use a shell even if the account has `nologin`                        |

> [!warning] Use `su -`, not plain `su`
> Plain `su` keeps your `PATH`, `HOME` and other variables. Root then runs programs and reads config from **your** home, which is both confusing and a security risk. `su -` gives a clean root environment.

`su` needs the **root password**. Many systems lock root's password (`!` in [[shadow]]), so `su` fails there and `sudo` is the only way in.

# `sudo` — run as root with your own password

| Command                | Meaning                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| `sudo cmd`             | Run `cmd` as root                                                 |
| `sudo -u alice cmd`    | Run as another user                                               |
| `sudo -g video cmd`    | Run with a different group                                        |
| `sudo -i`              | Root **login** shell (like `su -`)                                |
| `sudo -s`              | Root shell, keeping your environment (like plain `su`)            |
| `sudo -l`              | **List what you may run** (asks password)                         |
| `sudo -v`              | Refresh the password timestamp                                    |
| `sudo -k`              | Forget the timestamp: next `sudo` asks again                      |
| `sudo -n cmd`          | Non-interactive: fail instead of prompting (good for scripts)     |
| `sudo -E cmd`          | Keep your environment (if allowed by policy)                      |
| `sudoedit /etc/file`   | Edit a root file **with your editor running as you** (safer than `sudo vim`) |

```
$ sudo -n true
sudo: a password is required
```

### How sudo decides
1. Reads [[sudoers]] (`/etc/sudoers` + `/etc/sudoers.d/*`). Typical rule: `%wheel ALL=(ALL:ALL) ALL` = members of `wheel` may run anything as anyone. The admin group is `wheel` or `sudo` (varies by distro), see [[groups]].
2. Asks PAM to check **your** password (`/etc/pam.d/sudo`, see [[pam]]).
3. Remembers success for a few minutes (`timestamp_timeout`, default 5) per terminal.
4. Logs the command to the journal / auth log.

> [!danger] Always edit sudoers with `visudo`
> A syntax error in `/etc/sudoers` breaks `sudo` for everyone, and if root has no password you're locked out. `sudo visudo` (or `sudo visudo -f /etc/sudoers.d/myrule`) checks syntax before saving.

### The redirect gotcha
```bash
sudo echo 10 > /proc/sys/vm/swappiness   # FAILS: the > is done by YOUR shell
echo 10 | sudo tee /proc/sys/vm/swappiness >/dev/null   # works
sudo sh -c 'echo 10 > /proc/sys/vm/swappiness'          # works
```

# `pkexec` and polkit
**polkit** is a daemon that answers "may this user do this action?" for programs that talk over D-Bus (mounting disks, changing network, rebooting, installing software). A GUI shows a password dialog through a **polkit agent** in your desktop.

| Piece                              | Location                                   |
| ---------------------------------- | ------------------------------------------ |
| Actions (what can be asked)        | `/usr/share/polkit-1/actions/*.policy` (450 actions here) |
| Rules (who may do what, JavaScript)| `/etc/polkit-1/rules.d/`, `/usr/share/polkit-1/rules.d/` |
| List actions                       | `pkaction`                                 |
| Run a program as root              | `pkexec cmd`                               |

`pkexec` is the polkit equivalent of `sudo` for a single program.

# `doas` and `run0`
- **doas** (from OpenBSD): one tiny config file, e.g. `permit persist :wheel`. Smaller attack surface than sudo. Not installed on every system.
- **run0** (systemd 256+): doesn't use setuid at all; asks systemd to start the command in a fresh transient service. Authenticates through polkit.

# Scripting recipes

### Require root, re-run with sudo if needed
```bash
if [[ $EUID -ne 0 ]]; then
  exec sudo -- "$0" "$@"
fi
```

### Check if I can sudo without a password prompt
```bash
if sudo -n true 2>/dev/null; then
  echo "sudo ready (cached or NOPASSWD)"
else
  echo "sudo will ask for a password"
fi
```

### Who started this sudo?
Inside a command run with `sudo`, these variables say who you really are:
```bash
echo "$SUDO_USER $SUDO_UID $SUDO_GID $SUDO_COMMAND"
# use the real user's home, not /root:
real_home=$(getent passwd "${SUDO_USER:-$USER}" | cut -d: -f6)
```

### Drop privileges back to the invoking user
```bash
sudo -u "$SUDO_USER" -- some-command   # from inside a root script
```

### Run a command as a service account that has `nologin`
```bash
sudo -u myapp -- /usr/bin/myapp --check
sudo -u myapp -s                       # shell as myapp (uses YOUR shell)
```

# Find stuff

| Question                                 | Command / file                       |
| ---------------------------------------- | ------------------------------------ |
| What may I run with sudo?                | `sudo -l`                            |
| Who may sudo?                            | `getent group wheel` / `sudo`, [[sudoers]] |
| sudo version                             | `sudo -V` (`Sudo version 1.9.17p2` here) |
| sudo log                                 | `journalctl _COMM=sudo`              |
| Failed su / sudo attempts                | `journalctl -p warning _COMM=sudo _COMM=su` |
| polkit actions                           | `pkaction`, `pkaction -v -a ACTION`  |
| setuid programs on the system            | `find / -xdev -perm -4000 2>/dev/null` |

# See also
- [[users]] · [[groups]] · [[pam]] · [[login-sessions]]
- [[sudoers]] · [[shadow]] · [[special-bits]] · [[capabilities]]
- [[journal]] — where sudo logs go
- `man 8 sudo`, `man 5 sudoers`, `man 1 su`, `man 1 pkexec`, `man 8 polkit`
