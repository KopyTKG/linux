---
tags:
  - linux
  - filesystem
  - home
  - xdg
aliases:
  - /home
  - home directory
  - "~"
---
`/home` holds one directory per normal user. Your home (`~`, `$HOME`) is the only big place on the system you own and can write to without `sudo`. Every per-user setting, cache and file lives here.

> [!info] Facts about `/home`
> - Your home path comes from `/etc/passwd`: `alice:x:1000:1000:...:/home/alice:/bin/bash` (6th field = home, 7th = shell). See [[etc]].
> - `root` does **not** live here; its home is [[root]].
> - `/home` is often its **own partition or disk**, e.g. `/dev/nvme0n1p4`, while `/` is on `/dev/nvme0n1p3`. Reinstalling the OS doesn't touch your files.
> - `~` = `$HOME` = `/home/alice`. `~user` = home of another user.

# Permissions

| Path         | Mode         | Meaning                                         |
| ------------ | ------------ | ----------------------------------------------- |
| `/home`      | `drwxr-xr-x` | Anyone can list users, only root creates dirs   |
| `/home/alice` | `drwx------` (`700`) | Only you can enter. Other users see nothing |

```bash
stat -c '%a %A %U %n' ~      # 700 drwx------ alice /home/alice
```

> [!warning] Some programs refuse to work if permissions are too open
> `ssh` ignores `~/.ssh` unless it's `700` and keys are `600`. Same for `~/.gnupg`.
> Fix: `chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_*`

# Visible folders — `xdg-user-dirs`
`Desktop`, `Downloads`, `Documents`, ... are defined in `~/.config/user-dirs.dirs`, so they can be renamed or translated:

```
XDG_DESKTOP_DIR="$HOME/Desktop"
XDG_DOWNLOAD_DIR="$HOME/Downloads"
XDG_DOCUMENTS_DIR="$HOME/Documents"
XDG_MUSIC_DIR="$HOME/Music"
XDG_PICTURES_DIR="$HOME/Pictures"
XDG_VIDEOS_DIR="$HOME/Videos"
XDG_TEMPLATES_DIR="$HOME/Templates"
XDG_PUBLICSHARE_DIR="$HOME/Public"
```

```bash
xdg-user-dir DOWNLOAD     # /home/alice/Downloads  — use this in scripts, not a hardcoded path
```

# Hidden folders — XDG Base Directories
Files starting with `.` are hidden (`ls -A` shows them). Modern programs follow the **XDG Base Directory** spec, which says *what kind* of data goes *where*:

| Variable          | Default            | What goes there                                   | Safe to delete? |
| ----------------- | ------------------ | ------------------------------------------------- | --------------- |
| `XDG_CONFIG_HOME` | `~/.config`        | Settings (like a per-user [[etc]])                | No — you lose settings |
| `XDG_DATA_HOME`   | `~/.local/share`   | App data: saves, databases, fonts, `.desktop` files, Trash | No |
| `XDG_STATE_HOME`  | `~/.local/state`   | History, logs, "last opened" — nice to keep, not critical | Mostly |
| `XDG_CACHE_HOME`  | `~/.cache`         | Anything that can be regenerated (like a per-user [[var]]/cache) | **Yes** |
| —                 | `~/.local/bin`     | Your own scripts/binaries (like a per-user [[bin]]) | — |
| `XDG_RUNTIME_DIR` | `/run/user/1000`   | Sockets, locks; tmpfs, wiped on logout → [[run]] | — |

> [!tip] The `~/.local` tree mirrors `/usr`
> `~/.local/bin` ≈ `/usr/bin`, `~/.local/share` ≈ `/usr/share`, `~/.local/lib` ≈ `/usr/lib`.
> Put your scripts into `~/.local/bin` and make sure it's in `$PATH`:
> - fish: `fish_add_path ~/.local/bin`
> - bash: `export PATH="$HOME/.local/bin:$PATH"` in `~/.bashrc`

> [!note] Variables are usually unset
> `XDG_*_HOME` are mostly **not** set; programs fall back to the defaults. In scripts always use `${XDG_CONFIG_HOME:-$HOME/.config}`.

### Example sizes
```
8.2G  ~/.cache          ← cache, can be cleaned
5.1G  ~/.local          ← mostly ~/.local/share (apps, games)
3.4G  ~/.var            ← Flatpak app data (~/.var/app/<id>/)
1.2G  ~/.config
```

# Dotfiles (old style)
Older programs put a file/dir directly into `~` instead of following XDG:

| Path                      | What                                       |
| ------------------------- | ------------------------------------------ |
| `~/.bashrc`               | bash: runs for every interactive shell     |
| `~/.bash_profile`         | bash: runs for login shells                |
| `~/.bash_history`         | bash history                               |
| `~/.config/fish/config.fish` | fish config (fish follows XDG)          |
| `~/.profile`              | Generic login shell config                 |
| `~/.gitconfig`            | git (can also be `~/.config/git/config`)   |
| `~/.ssh/`                 | SSH keys, `config`, `known_hosts`          |
| `~/.gnupg/`               | GPG keys                                   |
| `~/.mozilla/`, `~/.librewolf/` | Browser profiles                      |
| `~/.var/app/`             | Flatpak apps (sandboxed home per app)      |
| `~/.local/share/Trash/`   | Trash bin of file managers                 |

> [!danger] Never share or publish
> `~/.ssh`, `~/.gnupg`, browser profiles, and any `*token*` / `.env` / `*.json` credential file. If you version your dotfiles in git, add these to `.gitignore`.

New users get a copy of `/etc/skel/` as their initial home → see [[etc]].

# Scripting recipes

### Get standard paths safely
```bash
config=${XDG_CONFIG_HOME:-$HOME/.config}
data=${XDG_DATA_HOME:-$HOME/.local/share}
cache=${XDG_CACHE_HOME:-$HOME/.cache}
state=${XDG_STATE_HOME:-$HOME/.local/state}
downloads=$(xdg-user-dir DOWNLOAD)

mkdir -p "$config/myscript"   # store your script's config here
```

### What takes space in home
```bash
du -sh ~/.[!.]* ~/* 2>/dev/null | sort -rh | head -15   # hidden + visible, biggest first
du -h -d1 ~/.cache 2>/dev/null | sort -rh | head        # one level deep
```
Interactive alternative: `ncdu ~` or `gdu ~`.

### Files bigger than 1 GiB
```bash
find ~ -xdev -type f -size +1G -printf '%s %p\n' 2>/dev/null \
  | sort -rn | awk '{printf "%.1f GiB  %s\n", $1/2^30, $2}'
```
`-xdev` = don't cross into other mounted filesystems.

### Cache files not touched for 90 days
```bash
find ~/.cache -type f -atime +90 | wc -l      # count
find ~/.cache -type f -atime +90 -delete      # delete (close apps first)
```
> [!note] `-atime` vs `noatime`
> This system mounts with `noatime`, so access time is rarely updated. `-mtime +90` (modified time) is more reliable here.

### Backup config of one app
```bash
tar czf ~/fish-config-$(date +%F).tar.gz -C ~/.config fish
```

# Find stuff

| I'm looking for...                     | Look in                                         |
| -------------------------------------- | ----------------------------------------------- |
| Settings of an app                     | `~/.config/<app>/`, then `~/.<app>`             |
| Saves / local DB / downloaded content  | `~/.local/share/<app>/`                         |
| Flatpak app's data                     | `~/.var/app/<app-id>/`                          |
| Shell history                          | `~/.local/share/fish/fish_history`, `~/.bash_history` |
| My own app launchers (`.desktop`)      | `~/.local/share/applications/`                  |
| User fonts                             | `~/.local/share/fonts/`                         |
| User systemd services                  | `~/.config/systemd/user/`                       |
| Autostart apps                         | `~/.config/autostart/`                          |
| Deleted files                          | `~/.local/share/Trash/files/`                   |
| Which file an app writes               | `strace -f -e trace=openat <app> 2>&1 \| grep $HOME` |

```bash
grep -rl "some_setting" ~/.config 2>/dev/null   # which config contains a setting
find ~/.config -mmin -10                        # configs changed in last 10 min
```

# See also
- [[root]] — home of the root user
- [[etc]] — system-wide config (`~/.config` overrides it per user)
- [[tmp]] — temporary files
- [[run]] — `/run/user/1000` (`XDG_RUNTIME_DIR`)
- XDG spec: <https://specifications.freedesktop.org/basedir-spec/latest/>
- `man 7 file-hierarchy`, `man 5 user-dirs.dirs`
