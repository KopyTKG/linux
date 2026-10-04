---
tags:
  - linux
  - filesystem
  - opt
aliases:
  - /opt
  - optional software
---
`/opt` ("optional") holds **self-contained third-party applications**. Each app gets one folder with *everything* inside — binary, libraries, resources — instead of being spread over `/usr/bin`, `/usr/lib`, `/usr/share`.

> [!info] Why apps use `/opt`
> Normal packages split files by type: binaries → [[bin]], libraries → [[lib]], data → `/usr/share` → see [[usr]].
> Big apps that **bundle their own libraries** (Electron apps, browsers, proprietary software, AppImage-like bundles) don't fit that model, so they go to `/opt/<app>/` as one block. Easy to remove: delete one folder.

# Example
```
/opt/
├── containerd
├── firefox             280M
├── google/             380M   (chrome)
├── onlyoffice          1.3G
└── zoom                520M
```

Each folder is one app. They usually come from vendor packages or tarballs that don't follow the normal `/usr` split. Some folders are not apps at all, e.g. `containerd` is created at runtime by containerd/Docker for its own data.

Inside one app — everything bundled:
```
/opt/firefox/
├── firefox            ← real binary
├── application.ini
├── browser/
├── defaults/
├── fonts/
├── libxul.so ...      ← its own libraries
```

# How they end up in `$PATH`
`/opt/*` is **not** in `$PATH`. Packages add a link or wrapper to `/usr/bin`:

```bash
/usr/bin/zoom             -> /opt/zoom/ZoomLauncher                  # symlink
/usr/bin/onlyoffice       -> /opt/onlyoffice/desktopeditors/DesktopEditors  # symlink
```
```bash
# /usr/bin/google-chrome — wrapper script
#!/bin/bash
exec /opt/google/chrome/google-chrome "$@"
```

> [!tip] Installing something into `/opt` yourself
> 1. Extract: `sudo tar xf app.tar.gz -C /opt` → `/opt/app/`
> 2. Link into PATH: `sudo ln -s /opt/app/app /usr/local/bin/app`
> 3. Menu entry: `~/.local/share/applications/app.desktop` → see [[home]]
>
> Use `/usr/local/bin` for manual links, **not** `/usr/bin` — `/usr/bin` belongs to your package manager → see [[usr]].

> [!note] Related places
> - `/etc/opt/<app>/` — config for `/opt` apps
> - `/var/opt/<app>/` — variable data for `/opt` apps
> - `/usr/local/` — your own manually compiled software (split by type, like `/usr`)

# Scripting recipes

### Size of each app
```bash
du -sh /opt/* 2>/dev/null | sort -rh
```

### Biggest files inside each app
```bash
for d in /opt/*/; do
  printf '%s  ' "$d"
  find "$d" -type f -printf '%s %p\n' 2>/dev/null | sort -rn | awk 'NR==1 {printf "%.0f MiB %s\n", $1/1048576, $2} END {if (!NR) print "(nothing readable)"}'
done
```

### Which commands point into `/opt`
```bash
find /usr/bin /usr/local/bin -maxdepth 1 -type l -lname '/opt/*' -printf '%p -> %l\n'
grep -l '/opt/' /usr/bin/* 2>/dev/null     # wrapper scripts (slow-ish)
```

### Where does a running app live
```bash
readlink /proc/$(pgrep -o firefox)/exe     # /opt/firefox/firefox  → see [[proc]]
```

# Find stuff
| Looking for                       | Command                                  |
| --------------------------------- | ---------------------------------------- |
| Real path of a command            | `readlink -f "$(command -v zoom)"`       |
| All executables of an app         | `find /opt/firefox -maxdepth 1 -type f -executable` |
| App folders not changed in 1 year | `find /opt -mindepth 1 -maxdepth 1 -type d -mtime +365` |
| App's `.desktop` launcher         | `grep -l /opt /usr/share/applications/*.desktop` |

# See also
- [[usr]] — normal package layout, `/usr/local`
- [[bin]] — where commands live
- [[lib]] — shared libraries (which `/opt` apps bundle instead)
- [[srv]], [[var]]
