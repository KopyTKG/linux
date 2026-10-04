---
tags:
  - linux
  - files
  - etc
  - libraries
aliases:
  - /etc/ld.so.conf
  - /etc/ld.so.conf.d
  - ld.so.conf
  - /etc/ld.so.cache
  - ldconfig
---
`/etc/ld.so.conf` tells the **dynamic linker** (`ld.so`) which folders to search for shared libraries (`.so` files) besides the defaults. When you start a program, `ld.so` loads every library it needs before `main()` runs. If one is not found, the program won't start.

> [!info] Quick facts
> - `/etc/ld.so.conf`: main file, usually only `include` lines
> - `/etc/ld.so.conf.d/*.conf`: one folder per line, added by packages or by you
> - `/etc/ld.so.cache`: **binary** index built by `ldconfig` from those folders. `ld.so` reads the cache, not the conf files
> - The linker itself: `/lib64/ld-linux-x86-64.so.2` (x86-64)
> - Manual: `man 8 ld.so`, `man 8 ldconfig`

# Example
```
# /etc/ld.so.conf
include /etc/ld.so.conf.d/*.conf
include /usr/lib/ld.so.conf.d/*.conf
```
```
# /etc/ld.so.conf.d/x86_64-linux-gnu.conf  (multiarch layout on some distros)
/usr/local/lib/x86_64-linux-gnu
/usr/lib/x86_64-linux-gnu
```
`ldconfig -p` → `2000+ libs found in cache '/etc/ld.so.cache'` on a typical desktop.

> [!important] Run `sudo ldconfig` after changing anything
> Adding a folder to `ld.so.conf.d` or copying a `.so` into `/usr/local/lib` does nothing until the cache is rebuilt. Package managers run `ldconfig` for you, but manual installs (`make install`) often don't.

# How a library is found
For each needed library, `ld.so` searches in this order and stops at the first hit:

| # | Where                                       | Set by                                      |
| - | ------------------------------------------- | ------------------------------------------- |
| 1 | `DT_RPATH` in the binary (old)              | Linker flag when the program was built      |
| 2 | `LD_LIBRARY_PATH` env variable              | You, per process (ignored for setuid programs) |
| 3 | `DT_RUNPATH` in the binary                  | Linker flag (`-Wl,-rpath`)                  |
| 4 | `/etc/ld.so.cache`                          | `ldconfig` + `ld.so.conf`                   |
| 5 | Default dirs: `/lib`, `/usr/lib` (`/lib64`, `/usr/lib64` on 64-bit multilib) | Built into `ld.so` |

`LD_PRELOAD=/path/lib.so` loads a library **before** all others, so its functions override the real ones. Useful for debugging, also abused by malware (`/etc/ld.so.preload` does the same system-wide; it should normally not exist).

> [!warning] Don't put `LD_LIBRARY_PATH` in your profile
> Setting it globally can make unrelated programs load the wrong library version and crash. Use it for one command: `LD_LIBRARY_PATH=/opt/app/lib /opt/app/bin/app`, or add a `.conf` file to `ld.so.conf.d`.

# Scripting recipes

### Which libraries does a program need, and where do they come from?
```bash
ldd /bin/ls
#   linux-vdso.so.1 (0x00007f4fe1f6e000)
#   libcap.so.2 => /usr/lib/libcap.so.2
#   libc.so.6 => /usr/lib/libc.so.6
#   /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2
```
> [!danger] `ldd` may run the program
> Don't run `ldd` on untrusted binaries. Safer: `objdump -p file | grep NEEDED` or `readelf -d file | grep NEEDED`.

### Missing libraries
```bash
ldd /path/to/program | grep 'not found'
```

### Find a library in the cache
```bash
ldconfig -p | grep libssl
```

### Folders ldconfig scans
```bash
grep -rh '^/' /etc/ld.so.conf /etc/ld.so.conf.d/ 2>/dev/null
```

### Add a folder permanently
```bash
echo /opt/myapp/lib | sudo tee /etc/ld.so.conf.d/myapp.conf
sudo ldconfig
```

### Watch the linker search (debug)
```bash
LD_DEBUG=libs /bin/true 2>&1 | head -20
```

### Libraries a running process has loaded
```bash
awk '$6 ~ /\.so/ {print $6}' /proc/$pid/maps | sort -u
```

# See also
- [[lib]] · [[lib64]] · [[usr]] — where libraries live
- [[proc]] — `/proc/<pid>/maps`
- [[environment]] · [[etc]]
