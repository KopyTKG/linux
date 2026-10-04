---
tags:
  - linux
  - filesystem
  - lib
  - scripting
aliases:
  - /lib
  - libraries
  - shared libraries
---
`/lib` holds **shared libraries** — code that many programs use, loaded at runtime instead of being copied into every binary. It is the Linux version of `.dll` files on Windows. It also holds kernel modules, firmware and lots of system data (systemd units, udev rules).

> [!info] On merged systems `/lib` is a symlink
> ```
> $ ls -l /
> lrwxrwxrwx   1 root root    7 lib -> usr/lib
> lrwxrwxrwx   1 root root    7 lib64 -> usr/lib
> ```
> So the real directory is `/usr/lib` (8 GB, ~5800 entries here). Unmerged systems still have a real `/lib` with the libraries needed for boot. See [[bin]] for the usr-merge and [[lib64]] for the 64-bit story.

# Shared library basics

### Naming and the symlink chain
```
$ ls -l /usr/lib/libz.so*
libz.so   -> libz.so.1                    # dev link: used by the compiler (-lz)
libz.so.1 -> libz.so.1.3.1.zlib-ng        # soname: used by programs at runtime
libz.so.1.3.1.zlib-ng                     # real file
```

| Name                | Example                   | Who uses it                                           |
| ------------------- | ------------------------- | ----------------------------------------------------- |
| real name           | `libz.so.1.3.1.zlib-ng`   | the actual file                                       |
| **soname**          | `libz.so.1`               | programs record this; bumped only on breaking changes |
| linker name         | `libz.so`                 | `gcc ... -lz` at build time                           |

> [!tip] Why it's done this way
> A bug-fix update replaces `libz.so.1.3.1` with `libz.so.1.3.2` and moves the `libz.so.1` link. Every program picks up the fix **without recompiling**. A breaking change ships `libz.so.2` next to `.1`, so old programs keep working.

### File types

| Extension | Meaning                                                          |
| --------- | ---------------------------------------------------------------- |
| `.so`     | Shared object — loaded at runtime (5134 here)                    |
| `.a`      | Static archive — copied into binary at build time (176 here)     |
| `.so` that is text | Linker script, e.g. `/usr/lib/libc.so` contains `GROUP ( /usr/lib/libc.so.6 ... )` |
| `.pc`     | `pkgconfig/*.pc` — tells compilers where headers/libs are        |

### How a program finds its libraries
1. Kernel reads the ELF header, sees `interpreter /lib64/ld-linux-x86-64.so.2` (the **dynamic loader**, see [[lib64]]).
2. Loader reads the list of needed sonames (`libc.so.6`, `libcap.so.2`...).
3. It searches, in order: `LD_LIBRARY_PATH` → `/etc/ld.so.cache` → default folders (`/lib`, `/usr/lib`, or their `lib64` variants).
4. Maps them into memory — visible in `/proc/<pid>/maps` (see [[proc]]).

```
$ ldd /usr/bin/ls
	linux-vdso.so.1 (0x00007f2691c5b000)                     # injected by the kernel, no file
	libcap.so.2 => /usr/lib/libcap.so.2 (0x00007f2691be1000)
	libc.so.6 => /usr/lib/libc.so.6 (0x00007f2691800000)
	/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2
```

> [!warning] Don't run `ldd` on untrusted binaries
> `ldd` may execute the program's loader. Use `objdump -p file | grep NEEDED` or `readelf -d file` for unknown files.

> [!note] `ld.so.cache`
> Searching folders every launch would be slow, so `ldconfig` builds `/etc/ld.so.cache` (3921 libs here). Extra folders are listed in `/etc/ld.so.conf.d/*.conf`. The package manager runs `ldconfig` for you; after `make install` into `/usr/local/lib` run `sudo ldconfig` yourself.

# Important subfolders of `/usr/lib`

| Folder                         | What                                                 | Example                                  |
| ------------------------------ | ---------------------------------------------------- | ---------------------------------------- |
| `/usr/lib/modules/<kernel>/`   | Kernel modules (drivers), one folder per kernel      | `$(uname -r)` = running kernel           |
| `/usr/lib/firmware/`           | Firmware blobs loaded into devices                   | `*.bin.zst` for GPUs, Wi-Fi              |
| `/usr/lib/systemd/system/`     | Unit files shipped by packages (716 here)            | override in `/etc/systemd/system` → [[etc]] |
| `/usr/lib/udev/rules.d/`       | Default device rules                                 | `10-dm.rules` → [[dev]]                  |
| `/usr/lib/sysctl.d/`           | Default kernel settings                              | `50-default.conf` → [[proc]]             |
| `/usr/lib/tmpfiles.d/`         | Rules for creating files in [[tmp]], [[run]], [[var]]|                                          |
| `/usr/lib/os-release`          | Distro name/version (`/etc/os-release` links here)   |                                          |
| `/usr/lib/python3.14/`         | Python standard library                              | also `site-packages/`                    |
| `/usr/lib/pkgconfig/`          | `.pc` files for compilers                            | `pkg-config --libs zlib`                 |
| `/usr/lib32/`                  | 32-bit libraries on some layouts (see [[lib64]])     | `libc.so.6` is `ELF 32-bit ... i386`     |

> [!important] `/usr/lib` vs `/etc` — vendor vs admin
> Packages put **defaults** in `/usr/lib/...`. You put **overrides** with the same name in `/etc/...`. Your file wins and survives updates. Same pattern for `systemd`, `sysctl.d`, `udev`, `tmpfiles.d`, `modprobe.d`.

# Find stuff

| Question                                   | Command                                       |
| ------------------------------------------ | --------------------------------------------- |
| Libraries a binary needs                   | `ldd /usr/bin/ls`                             |
| Same, safe for untrusted files             | `readelf -d file \| grep NEEDED`              |
| Is library X installed? Where?             | `ldconfig -p \| grep libz`                    |
| Real file behind the soname                | `readlink -f /usr/lib/libz.so.1`              |
| Which running processes use it?            | `grep -l libz.so /proc/*/maps`                |
| 32 or 64 bit?                              | `file -L /usr/lib32/libc.so.6`                |
| Exported functions                         | `nm -D --defined-only /usr/lib/libz.so.1 \| head` |
| Kernel module info                         | `modinfo btrfs`, `lsmod`                      |
| Compiler flags for a library               | `pkg-config --cflags --libs zlib`             |

# Scripting recipes
Tested in `bash`.

### Find binaries with missing libraries
Useful after a partial upgrade or when an app doesn't start.
```bash
for f in /usr/bin/*; do
  ldd "$f" 2>/dev/null | grep -q 'not found' && echo "$f"
done
```
Then see what's missing: `ldd /usr/bin/arpd | grep 'not found'` → `libdb-6.2.so => not found`.

### Which processes still use an old library (restart needed after update)
```bash
lib=libz.so
grep -l "$lib" /proc/[0-9]*/maps 2>/dev/null | cut -d/ -f3 | while read -r p; do
  echo "$p $(cat /proc/$p/comm 2>/dev/null)"
done
```
After an update the old file is deleted but still mapped — look for `(deleted)`:
```bash
grep -l '\.so.*(deleted)' /proc/[0-9]*/maps 2>/dev/null | cut -d/ -f3
```

### Run a program with a custom library folder
```bash
LD_LIBRARY_PATH=$HOME/mylibs ./myapp
LD_DEBUG=libs ./myapp 2>&1 | head      # show how the loader searches
```

### Disk usage of kernel modules per kernel
```bash
du -sh /usr/lib/modules/*
```
Old kernels you removed can leave folders here. A folder with no matching kernel image in [[boot]] is usually a leftover.

# See also
- [[lib64]] — the dynamic loader and 32/64-bit layout
- [[bin]] — the programs that use these libraries
- [[usr]] — `/usr/include` has the matching headers
- [[etc]] — `/etc/ld.so.conf.d`, overrides for `/usr/lib/*.d`
- [[boot]] — kernel image that loads `/usr/lib/modules`
- `man 8 ld.so`, `man 8 ldconfig`, `man 1 ldd`
