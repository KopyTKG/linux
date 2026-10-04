---
tags:
  - linux
  - filesystem
  - root
aliases:
  - /root
  - root home
---
`/root` is the **home directory of the `root` user** (UID 0). It's not "the root of the filesystem" — that's `/`.

| Name        | Means                                   |
| ----------- | --------------------------------------- |
| `/`         | Root **directory** — top of the tree    |
| `/root`     | Home of the root **user**               |
| `root`      | The superuser account (UID `0`)         |

```bash
getent passwd root      # root:x:0:0::/root:/usr/bin/bash
```

# Why it's not in `/home`
> [!info] Separate on purpose
> - `/home` is often a **separate disk or partition** (e.g. `/dev/nvme0n1p4`). If it fails to mount, root must still be able to log in and fix things. `/root` lives with the system.
> - Keeps the admin's files away from normal users.
> - On some btrfs setups `/root` is its own btrfs subvolume `@root` on the system disk:
>   `/root  /dev/nvme0n1p3[/@root]  btrfs`
>   so system snapshots of `@` don't roll back root's files and vice versa.

# Permissions
```
drwxr-x---  root root  /root     (750)
```
Normal users can't even list it:
```bash
ls /root      # ls: cannot open directory '/root': Permission denied
sudo ls -A /root
```

# What's inside
Same layout as a normal [[home]], just for root:

| Path                    | What                                         |
| ----------------------- | -------------------------------------------- |
| `/root/.bashrc`         | root's shell config (root's shell here: `bash`) |
| `/root/.bash_history`   | Commands run in a root shell                 |
| `/root/.ssh/`           | Keys for root SSH login (if allowed)         |
| `/root/.config/`, `.cache/` | Created by tools run as root             |

> [!warning] `sudo` vs `sudo -i` and `$HOME`
> - `sudo cmd` → runs as root, but **keeps some of your environment**. Files a tool writes to `$HOME` can end up in **your** home owned by root (breaks things later).
> - `sudo -i` / `sudo -H` → real root login shell, `$HOME=/root`.
> Check: `sudo sh -c 'echo $HOME'`
> Fix files accidentally owned by root in your home: `find ~ -user root` → `sudo chown -R $USER: <path>`

# Scripting recipes

### Require root in a script
```bash
if [[ $EUID -ne 0 ]]; then
  echo "Run as root: sudo $0" >&2
  exit 1
fi
```

### Re-run self with sudo
```bash
[[ $EUID -eq 0 ]] || exec sudo "$0" "$@"
```

### Get home of any user (don't hardcode `/home/$USER`)
```bash
getent passwd root | cut -d: -f6     # /root
getent passwd "$USER" | cut -d: -f6  # /home/alice
```

# Find stuff
| Looking for                          | Command                                  |
| ------------------------------------ | ---------------------------------------- |
| What root ran recently               | `sudo cat /root/.bash_history`, better: `journalctl _COMM=sudo` |
| Files root left in your home         | `find ~ -user root 2>/dev/null`          |
| Who can become root                  | `getent group wheel` + `/etc/sudoers` → [[etc]] |

# See also
- [[home]] — normal users' homes
- [[etc]] — `/etc/passwd`, `/etc/sudoers`
- [[sbin]] — admin commands
