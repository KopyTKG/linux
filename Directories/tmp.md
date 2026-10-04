---
tags:
  - linux
  - filesystem
  - tmp
  - scripting
aliases:
  - /tmp
  - /var/tmp
---
`/tmp` is a place where **any program and any user** can create temporary files. Content is **not guaranteed to survive** — on most modern systems it's wiped on every reboot.

> [!info] `/tmp` lives in RAM here
> ```
> TARGET SOURCE FSTYPE OPTIONS
> /tmp   tmpfs  tmpfs  rw,noatime,inode64,huge=advise
> ```
> `tmpfs` = filesystem in RAM (can swap out). Super fast, **gone on reboot**, and limited in size — default is half of RAM:
> ```
> tmpfs   16G   13M   16G   1% /tmp
> ```
> Writing a 20 GB file into `/tmp` fails with "No space left on device" even though your disk is empty.

# `/tmp` vs `/var/tmp`

|                    | `/tmp`                    | `/var/tmp`                             |
| ------------------ | ------------------------- | -------------------------------------- |
| Storage here       | RAM (`tmpfs`)             | Disk (btrfs subvolume `@tmp`)          |
| Survives reboot    | **No**                    | **Yes**                                |
| Auto-cleaned after | 10 days unused            | 30 days unused                         |
| Use for            | Small, short-lived files  | Big files, or files needed after reboot (e.g. package builds) |

Cleanup ages come from `/usr/lib/tmpfiles.d/tmp.conf`:
```
q /tmp 1777 root root 10d
q /var/tmp 1777 root root 30d
```
`systemd-tmpfiles-clean.timer` runs daily and deletes files older than that. Override in `/etc/tmpfiles.d/tmp.conf` → see [[etc]].

# Permissions — the sticky bit
```
drwxrwxrwt  root root  /tmp      (1777)
```
- `rwxrwxrwx` → everyone can create files.
- `t` at the end = **sticky bit** (`1` in `1777`) → you can delete or rename **only your own** files, even though the directory is world-writable.

```bash
stat -c '%a %A' /tmp      # 1777 drwxrwxrwt
```

> [!warning] Security: never use predictable names
> `/tmp/myscript.log` is dangerous: another user can create that file (or a symlink with that name pointing to your files) **before** your script runs.
> Always use `mktemp`, which creates a file with a random name and `600` permissions.

> [!note] Private `/tmp` for services
> systemd services with `PrivateTmp=yes` get their own `/tmp` — files they write there are not visible in your `/tmp`. They're in `/tmp/systemd-private-*-<service>/`.

Per-user alternative: `$XDG_RUNTIME_DIR` (`/run/user/1000`) — only you can access it, wiped on logout → [[run]].

# Scripting recipes

### Safe temp file / dir with automatic cleanup
```bash
#!/usr/bin/env bash
tmpdir=$(mktemp -d)                      # e.g. /tmp/tmp.Xa9d2kLm
trap 'rm -rf "$tmpdir"' EXIT             # runs on normal exit, error, Ctrl+C

curl -so "$tmpdir/page.html" https://example.com
grep -c '<p>' "$tmpdir/page.html"
# no manual cleanup needed
```

```bash
tmpfile=$(mktemp)                        # single file
tmpfile=$(mktemp --suffix=.json)         # with extension
tmpfile=$(mktemp /tmp/myscript.XXXXXX)   # custom prefix, X = random
big=$(mktemp -p /var/tmp)                # on disk instead of RAM
```

Fish version:
```fish
set tmpdir (mktemp -d)
function cleanup --on-event fish_exit; rm -rf $tmpdir; end
```

### How full is `/tmp`
```bash
df -h --output=used,size,pcent /tmp | tail -1
du -sh /tmp/* 2>/dev/null | sort -rh | head
```

### My files in `/tmp`
```bash
find /tmp -maxdepth 1 -user "$USER"
find /tmp -user "$USER" -mmin +60 -delete   # delete mine older than 1 hour
```

### Lock file so a script runs only once at a time
```bash
exec 9> "${XDG_RUNTIME_DIR:-/tmp}/myscript.lock"
flock -n 9 || { echo "Already running"; exit 1; }
```

# Find stuff
| Looking for                     | Where                                         |
| ------------------------------- | --------------------------------------------- |
| X11 display sockets             | `/tmp/.X11-unix/X0`                           |
| Service private tmp             | `/tmp/systemd-private-*` (root only)          |
| Package / source build dirs     | usually `/tmp` or `/var/tmp` (depends on tool) |
| Who fills `/tmp`                | `sudo du -sh /tmp/* \| sort -rh \| head`      |
| Cleanup rules                   | `systemd-tmpfiles --cat-config \| grep tmp`   |

# See also
- [[run]] — `/run/user/$UID`, per-user runtime dir
- [[var]] — `/var/tmp`, `/var/cache`
- [[proc]] — `/proc/mounts` shows tmpfs mounts
- `man 5 tmpfiles.d`, `man 1 mktemp`
