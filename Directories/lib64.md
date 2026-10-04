---
tags:
  - linux
  - filesystem
  - lib64
aliases:
  - /lib64
  - multilib
---
`/lib64` exists for **64-bit libraries** on systems that can run both 32-bit and 64-bit programs. Its most important job today is holding the path to the **dynamic loader**, `ld-linux-x86-64.so.2`, which every dynamically linked program on x86-64 asks for by that exact path.

> [!info] On merged systems `/lib64` is usually a symlink
> ```
> $ ls -l /
> lrwxrwxrwx   1 root root    7 lib64 -> usr/lib
> $ ls -l /usr/lib64
> lrwxrwxrwx   1 root root    3 /usr/lib64 -> lib
> ```
> In the first layout the native libraries are 64-bit and live in `/usr/lib`, and `/lib64` and `/usr/lib64` just point there. Other distros put 64-bit libraries in a real `/usr/lib64` (see the table below). Either way, the old paths keep working. See [[lib]] and the usr-merge in [[bin]].

# Why it still has to exist
Every 64-bit ELF binary contains a hardcoded interpreter path:

```
$ file /usr/bin/ls
/usr/bin/ls: ELF 64-bit LSB pie executable, x86-64, ..., interpreter /lib64/ld-linux-x86-64.so.2
```

The kernel opens **exactly** that path and hands control to the loader, which then loads `libc.so.6` and the rest (see [[lib]]). If `/lib64/ld-linux-x86-64.so.2` disappeared, almost **no program could start** — not even `ls` or `bash`.

> [!warning] Never delete `/lib64` or the loader
> Removing or breaking `ld-linux-x86-64.so.2` makes the system unbootable to userspace. Recovery needs a live USB. Statically linked tools (e.g. `busybox`) are the only things that still run.

# Layout differences between distros
Which folder holds which architecture depends on the distro. There are three common layouts:

| Layout             | 64-bit libs                 | 32-bit libs                 | `/lib64`                     |
| ------------------ | --------------------------- | --------------------------- | ---------------------------- |
| "lib + lib32"      | `/usr/lib`                  | `/usr/lib32`                | symlink to `usr/lib`         |
| "lib64 + lib"      | `/usr/lib64`                | `/usr/lib`                  | symlink to `usr/lib64`       |
| Multiarch triplets | `/usr/lib/x86_64-linux-gnu` | `/usr/lib/i386-linux-gnu`   | only holds the loader link   |

The rest of this note uses the first layout as the example.

> [!tip] Scripts: never hardcode a lib folder
> Ask the system instead: `ldconfig -p | grep libssl`, `pkg-config --variable=libdir zlib`, or `gcc -print-file-name=libc.so.6`.

# 32-bit libraries
Steam, Wine and old games need 32-bit libraries. Depending on the layout they go to `/usr/lib32`, `/usr/lib` or `/usr/lib/i386-linux-gnu`. In the first layout it is `/usr/lib32`.

```
$ file -L /usr/lib32/libc.so.6
/usr/lib32/libc.so.6: ELF 32-bit LSB shared object, Intel i386 ...
```
32-bit programs use their own loader: `/usr/lib/ld-linux.so.2`.

# Find stuff

| Question                                | Command                                         |
| --------------------------------------- | ----------------------------------------------- |
| Which loader does a binary want?        | `file /usr/bin/ls` or `readelf -l /usr/bin/ls \| grep interpreter` |
| Real loader file                        | `readlink -f /lib64/ld-linux-x86-64.so.2`       |
| Is a binary 32 or 64 bit?               | `file -L <binary>`                              |
| All 32-bit libraries the loader knows   | `ldconfig -p \| grep -v x86-64`                 |
| Loader version / help                   | `/lib64/ld-linux-x86-64.so.2 --help`            |
| Which x86-64 level CPU supports         | `/lib64/ld-linux-x86-64.so.2 --help \| grep supported` |

> [!note] x86-64 microarchitecture levels
> x86-64 has levels: baseline (`v1`), `v2` (SSE4.2), `v3` (AVX2), `v4` (AVX-512). Some distros build packages for `v2` or `v3` for speed. The loader `--help` output shows which levels your CPU supports.

# Scripting recipes

### Run a binary through the loader directly
Works even if the file has no `+x` bit (e.g. on a `noexec` mount):
```bash
/lib64/ld-linux-x86-64.so.2 /usr/bin/ls -l
```

### Count 32-bit vs 64-bit ELF files in a folder
```bash
for f in /usr/bin/*; do
  file -bL "$f" 2>/dev/null | grep -o '^ELF [0-9]*-bit'
done | sort | uniq -c
```

### Check that the loader link is healthy
```bash
if [[ -x $(readlink -f /lib64/ld-linux-x86-64.so.2) ]]; then
  echo "loader OK"
else
  echo "loader missing!" >&2
fi
```

# See also
- [[lib]] — shared libraries, `ldd`, `ldconfig`
- [[bin]] — usr-merge and how programs are started
- [[usr]] — `/usr/lib`, `/usr/lib32`
- `man 8 ld.so`
