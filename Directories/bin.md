---
tags:
  - linux
  - filesystem
  - bin
  - scripting
aliases:
  - /bin
  - binaries
---
`/bin` is where the **essential commands** live — programs every user needs, such as `ls`, `cp`, `bash`, `cat`, `grep`. Historically it held only what was needed to boot and repair the system before `/usr` was mounted.

> [!info] On modern distros `/bin` is just a symlink
> Most modern distros use the **usr-merge**: `/bin`, `/sbin`, `/lib` and `/lib64` are symlinks into [[usr]].
> ```
> $ ls -l /
> lrwxrwxrwx   1 root root    7 bin -> usr/bin
> lrwxrwxrwx   1 root root    7 lib -> usr/lib
> lrwxrwxrwx   1 root root    7 lib64 -> usr/lib
> lrwxrwxrwx   1 root root    7 sbin -> usr/bin
> drwxr-xr-x   1 root root   80 usr
> ```
> So `/bin/ls` and `/usr/bin/ls` are **the same file**. On fully merged systems even `/sbin` points to `usr/bin`; other systems keep a separate `/usr/sbin` (see [[sbin]]). Some smaller or older distros are not merged at all, which is one more reason to look commands up with `command -v` instead of hardcoding paths.

> [!note] Why the split existed
> In the 1970s the root disk was small, so the system was split: `/bin` (needed for boot) on the root disk, `/usr/bin` (everything else) on a second disk. Today the initramfs (see [[boot]]) mounts everything before userspace starts, so the split has no purpose. Merging into `/usr` means the whole OS lives in one directory that can be snapshotted, mounted read-only or shared.

# What's inside
A typical desktop `/usr/bin` holds **a few thousand entries**. Most are compiled ELF binaries; several hundred are scripts.

| Kind               | Example                     | How to recognise                                  |
| ------------------ | --------------------------- | ------------------------------------------------- |
| ELF binary         | `/usr/bin/ls`               | `file` says `ELF 64-bit LSB pie executable`       |
| Script             | `/usr/bin/ldd`              | first line is a shebang (`#!/bin/bash`)           |
| Symlink            | `/usr/bin/sh -> bash`       | `ls -l` shows `->`                                |
| Multi-call binary  | `reboot -> systemctl`       | one program behaves differently by `argv[0]`      |
| setuid binary      | `/usr/bin/sudo`             | `ls -l` shows `s` in place of `x` (`-rwsr-xr-x`)  |

```
$ file /usr/bin/ls /usr/bin/ldd
/usr/bin/ls:  ELF 64-bit LSB pie executable, x86-64, dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, stripped
/usr/bin/ldd: Bourne-Again shell script, ASCII text executable
```

> [!warning] setuid binaries run as their **owner**, not as you
> `sudo`, `passwd`, `pkexec`, `su`, `mount.cifs` have the setuid bit and run as `root`. They are the classic target of privilege escalation bugs. Never set `chmod u+s` on your own scripts (the kernel ignores it on scripts anyway).

# How the shell finds a command
When you type `ls`, the shell checks, in order:

1. **Aliases** (`alias ls='ls --color=tty'`)
2. **Functions** and **builtins** (`cd`, `echo`, `read`, `type` are builtins — there is no file for `cd`)
3. Every directory in `$PATH`, **left to right**. First match wins.

```
$ echo $PATH | tr ':' '\n'
/home/alice/.local/share/mise/installs/python/3.13/bin
...
/home/alice/.local/bin
/usr/local/bin
/usr/bin
/bin
...
```

> [!tip] Order matters
> `~/.local/bin` and `/usr/local/bin` come **before** `/usr/bin`, so your own scripts override system ones. That's how version managers (mise, pyenv, nvm) work:
> ```
> $ type -a python3
> python3 is /home/alice/.local/share/mise/installs/python/3.13/bin/python3
> python3 is /home/alice/.local/share/mise/shims/python3
> python3 is /usr/bin/python3
> ```
> After installing a new binary, bash caches lookups — run `hash -r` if the old path is still used.

| Location               | Who puts files there              | Use for                               |
| ---------------------- | --------------------------------- | ------------------------------------- |
| `/usr/bin`             | the package manager only          | never edit by hand                    |
| `/usr/local/bin`       | admin, by hand / `make install`   | system-wide custom tools              |
| `~/.local/bin`         | you                               | personal scripts, no `sudo` needed    |
| `/opt/<app>/bin`       | big third-party apps              | see [[opt]]                           |

# Find stuff

| Question                                   | Command                                  |
| ------------------------------------------ | ---------------------------------------- |
| Where is the command?                      | `which ls` / `command -v ls`             |
| All matches, incl. aliases and builtins    | `type -a ls`                             |
| Real file behind symlinks                  | `readlink -f "$(command -v reboot)"` → `/usr/bin/systemctl` |
| Binary, script or something else?          | `file /usr/bin/ldd`                      |
| Which libraries does it need?              | `ldd /usr/bin/ls` → see [[lib]]          |
| Its manual                                 | `man ls`, `ls --help`, `man -w ls` (path to man page) |
| Search commands by description             | `apropos partition` / `man -k partition` |

# Scripting recipes
All snippets tested in `bash`.

### Check that a command exists
```bash
if command -v rg >/dev/null 2>&1; then
  echo "rg installed"
else
  echo "rg missing" >&2; exit 1
fi
```
Use `command -v`, not `which` — it's a POSIX builtin and also sees functions and aliases.

### Commands that appear in more than one `$PATH` directory
Because of the usr-merge, `/bin` and `/usr/bin` are the same folder, so resolve directories first or every command shows up twice.
```bash
declare -A seen
IFS=: read -ra dirs <<< "$PATH"
for d in "${dirs[@]}"; do
  r=$(realpath -e "$d" 2>/dev/null) || continue
  [[ -n ${seen[$r]} ]] && continue
  seen[$r]=1
  ls -1 "$r"
done | sort | uniq -d
```

### Broken symlinks in `/usr/bin`
```bash
find /usr/bin -maxdepth 1 -xtype l
```
`-xtype l` = "is still a link after following it" = target is missing.

### Count ELF binaries vs scripts
```bash
for f in /usr/bin/*; do
  [[ -f $f && -r $f ]] || continue
  case $(head -c4 "$f") in
    $'\x7fELF') echo elf ;;
    '#!'*)      echo script ;;
    *)          echo other ;;
  esac
done | sort | uniq -c
```

### List setuid binaries (security audit)
```bash
find /usr/bin -maxdepth 1 -perm -4000 -printf '%u %p\n'
```

### Biggest commands
```bash
find /usr/bin -maxdepth 1 -type f -printf '%s %f\n' | sort -rn | head | numfmt --field=1 --to=iec
```

### Install your own script properly
```bash
mkdir -p ~/.local/bin
install -m 755 myscript.sh ~/.local/bin/myscript
command -v myscript   # should print ~/.local/bin/myscript
```

# See also
- [[usr]] — where `/bin` really lives
- [[sbin]] — admin commands (also merged into `/usr/bin`)
- [[lib]] — shared libraries the binaries load
- [[opt]] — third-party apps with their own `bin/`
- [[proc]] — `/proc/<pid>/exe` shows which binary a running process came from
- `man 7 hier` — official filesystem layout, `man 7 file-hierarchy` — systemd's version
- Filesystem Hierarchy Standard: <https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html>
