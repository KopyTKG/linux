---
tags:
  - linux
  - filesystem
  - usr
  - scripting
aliases:
  - /usr
  - Unix System Resources
---
`/usr` holds **the operating system itself**: all installed programs, their libraries, documentation, icons, fonts and headers. On a modern distro almost everything the package manager installs ends up here. It is meant to be **read-only, shareable and the same on every machine** with the same packages — machine-specific stuff goes to [[etc]] (config) and [[var]] (changing data).

> [!info] The name
> `usr` originally meant **user** — in early Unix, home directories lived here. When the disk filled up, system files spilled over from `/bin` into `/usr/bin` and homes moved to [[home]]. The backronym "Unix System Resources" came later.

> [!info] usr-merge: `/usr` *is* the system
> ```
> $ ls -l /
> lrwxrwxrwx   1 root root    7 bin -> usr/bin
> lrwxrwxrwx   1 root root    7 lib -> usr/lib
> lrwxrwxrwx   1 root root    7 lib64 -> usr/lib
> lrwxrwxrwx   1 root root    7 sbin -> usr/bin
> drwxr-xr-x   1 root root   80 usr
> ```
> On merged systems the top-level program folders are just links into `/usr`. Details in [[bin]].

> [!warning] Don't edit files in `/usr` by hand
> Everything in `/usr` (except `/usr/local`) belongs to the package manager. Your changes get **overwritten on the next update** and can break checksums. Put overrides in [[etc]], custom tools in `/usr/local` or `~/.local`.

# Layout

```
$ ls /usr
bin  include  lib  lib32  lib64  local  sbin  share  src
```

| Folder          | What                                              | Size here | Note       |
| --------------- | ------------------------------------------------- | --------- | ---------- |
| `/usr/bin`      | All commands (4259)                               | 2.3 GB    | [[bin]]    |
| `/usr/sbin`     | Admin commands (here a symlink to `bin`)          |           | [[sbin]]   |
| `/usr/lib`      | Libraries, kernel modules, systemd units          | 8.0 GB    | [[lib]]    |
| `/usr/lib64`    | 64-bit libraries (here a symlink to `lib`)        |           | [[lib64]]  |
| `/usr/lib32`    | 32-bit libraries (on some layouts)                |           | [[lib64]]  |
| `/usr/include`  | C/C++ header files (1389 entries)                 | 528 MB    |            |
| `/usr/share`    | Architecture-independent data: docs, icons, fonts | 6.6 GB    |            |
| `/usr/local`    | Your own system-wide software, not from packages  |           |            |
| `/usr/src`      | Kernel headers, DKMS driver sources               |           |            |

> [!tip] `/usr` is on the root filesystem here
> `findmnt /usr` returns nothing — it is not a separate mount, it lives on `/dev/nvme0n1p3` (btrfs subvolume `@`) together with `/`.

# `/usr/bin`, `/usr/lib`
Covered in their own notes: [[bin]], [[sbin]], [[lib]], [[lib64]].

# `/usr/share` — data that is the same on every CPU
"Share" = could be shared between x86 and ARM machines, because it is text, images, etc., not compiled code.

| Folder                             | What                                                        |
| ---------------------------------- | ----------------------------------------------------------- |
| `man/`                             | Manual pages, by section (`man1` … `man8`) and language     |
| `doc/`                             | README, changelogs, examples from packages                  |
| `info/`                            | GNU info manuals (`info coreutils`)                         |
| `applications/`                    | `.desktop` files = app launcher entries (249 here)          |
| `icons/`                           | Icon themes (`hicolor`, `breeze`, `Adwaita`)                |
| `pixmaps/`                         | Old-style app icons                                         |
| `fonts/`                           | System fonts (724 MB here)                                  |
| `themes/`                          | GTK themes                                                  |
| `wallpapers/`                      | Desktop backgrounds (KDE)                                   |
| `wayland-sessions/`, `xsessions/`  | Sessions shown in the login screen (`plasma.desktop`)       |
| `locale/`                          | Translations (`.mo` files)                                  |
| `zoneinfo/`                        | Time zones — `/etc/localtime` links here                    |
| `mime/`                            | File type database (what opens `.pdf`)                      |
| `terminfo/`                        | Terminal capabilities (`$TERM`)                             |
| `bash-completion/`, `fish/`, `zsh/`| Tab completion scripts                                      |
| `dbus-1/`, `polkit-1/`             | D-Bus services, permission rules                            |
| `X11/xkb/`                         | Keyboard layouts                                            |

### `man/` — manual pages
Pages are gzipped `groff` files: `/usr/share/man/man1/ls.1.gz`.

| Section | Contents                                 | Example             |
| ------- | ---------------------------------------- | ------------------- |
| 1       | User commands                            | `man 1 ls`          |
| 2       | System calls (kernel API)                | `man 2 open`        |
| 3       | C library functions                      | `man 3 printf`      |
| 4       | Device files                             | `man 4 null`        |
| 5       | File formats and config files            | `man 5 proc`, `man 5 fstab` |
| 6       | Games                                    |                     |
| 7       | Overviews, conventions                   | `man 7 hier`        |
| 8       | Admin commands                           | `man 8 mount`       |

> [!tip] `printf` exists in section 1 (shell command) and 3 (C function). `man printf` shows the first; `man 3 printf` the C one. `man -f printf` lists all.

### `applications/` — app launchers
Each app in your menu is one `.desktop` file:

```ini
[Desktop Entry]
Type=Application
TryExec=alacritty
Exec=alacritty
Icon=Alacritty
Terminal=false
Categories=System;TerminalEmulator;
Name=Alacritty
GenericName=Terminal
```

| Location                                 | Priority                    |
| ---------------------------------------- | --------------------------- |
| `~/.local/share/applications/`           | **wins** — yours (Steam games end up here) |
| `/usr/local/share/applications/`         | admin                       |
| `/usr/share/applications/`               | packages                    |

To change a launcher, copy it to `~/.local/share/applications/` and edit the copy.

### `icons/` and `fonts/`
- Icons are searched by **name** (`Icon=Alacritty`) in the current theme, then `hicolor` (the fallback), in size folders like `48x48/apps/`, `scalable/apps/`.
- Fonts: system in `/usr/share/fonts/`, yours in `~/.local/share/fonts/`. After adding fonts run `fc-cache -f`. `fc-list` shows all (985 here).

# `/usr/include` — headers
C/C++ `#include <stdio.h>` is looked up here.

| Header                 | Comes from          | Purpose                         |
| ---------------------- | ------------------- | ------------------------------- |
| `stdio.h`, `unistd.h`  | C library (glibc)   | C standard library              |
| `linux/input.h`        | Kernel API headers  | Kernel ↔ userspace structures   |
| `zlib.h`               | zlib                | Compression library             |

Some distros install headers together with the library, others split them into separate development packages (often named `-dev` or `-devel`).

```bash
echo | gcc -E -Wp,-v - 2>&1 | grep '^ /'   # folders gcc searches for headers
```

# `/usr/local` — your system-wide software
Same layout as `/usr` (`bin`, `lib`, `share`, `include`, `etc`, `src`...), but **the package manager never touches it**. Used by `make install`, `pip install` as root, manual installs.

> [!tip] `/usr/local/bin` wins over `/usr/bin`
> It comes earlier in `$PATH` (see [[bin]]), so a script with the same name as a system command replaces it for everyone. Handy for wrappers, confusing when you forget about it — check with `type -a <cmd>`.

# `/usr/src` — sources
| Entry                        | Purpose                                         |
| ---------------------------- | ----------------------------------------------- |
| `linux-<version>/`, `linux-headers-<version>/` | Kernel headers to build out-of-tree modules (name differs per distro) |
| `<driver>-<version>/`        | Driver source built by DKMS, e.g. `xone-0.5.8/` (Xbox controller) |

`/lib/modules/$(uname -r)/build` always leads to the headers of the running kernel — a symlink into `/usr/src` on many distros, a real folder on others. Build scripts use that path so they don't care where the distro put them.

**DKMS** recompiles these drivers automatically for each new kernel and installs the result into `/usr/lib/modules/<kernel>/` (see [[lib]]).

# Find stuff

| Question                                 | Command                                         |
| ---------------------------------------- | ----------------------------------------------- |
| Path of a man page                       | `man -w ls` → `/usr/share/man/man1/ls.1.gz`     |
| Search man pages                         | `man -k keyword` / `apropos keyword`            |
| Docs of a package                        | `ls /usr/share/doc/<pkg>`                       |
| Launcher of an app                       | `grep -l '^Exec=firefox' /usr/share/applications/*.desktop` |
| Is font X installed?                     | `fc-list \| grep -i noto`                       |
| Biggest folders                          | `du -sh /usr/share/* \| sort -rh \| head`       |
| All files of any kind by name            | `locate stdio.h` (after `sudo updatedb`), or `find /usr -name stdio.h` |

# Scripting recipes
Tested in `bash`.

### List all app launchers (name → command)
```bash
for f in /usr/share/applications/*.desktop; do
  name=$(grep -m1 '^Name=' "$f" | cut -d= -f2-)
  cmd=$(grep -m1 '^Exec=' "$f" | cut -d= -f2-)
  printf '%-30s %s\n' "$name" "$cmd"
done
```

### Everything installed by hand into `/usr/local`
```bash
find /usr/local -type f -newer /etc/hostname -printf '%TY-%Tm-%Td %p\n' | sort
```
Lists files changed after `/etc/hostname` (written at install time), newest last.

### Biggest things in `/usr/share`
```bash
du -sh /usr/share/* 2>/dev/null | sort -rh | head
```

### Install a custom font for your user
```bash
mkdir -p ~/.local/share/fonts
cp MyFont.ttf ~/.local/share/fonts/
fc-cache -f
fc-list | grep -i myfont
```

### Create your own app launcher
```bash
cat > ~/.local/share/applications/myscript.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=My Script
Exec=/home/alice/.local/bin/myscript
Icon=utilities-terminal
Terminal=true
EOF
```

# See also
- [[bin]], [[sbin]] — commands and `$PATH`
- [[lib]], [[lib64]] — libraries, modules, loader
- [[etc]] — where you override what `/usr` ships
- [[var]] — data that changes (logs, caches, package database)
- [[opt]] — third-party apps that don't follow the `/usr` layout
- [[boot]] — kernel image matching `/usr/lib/modules`
- `man 7 hier`, `man 7 file-hierarchy`
