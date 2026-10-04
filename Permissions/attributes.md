---
tags:
  - linux
  - permissions
  - filesystem
aliases:
  - chattr
  - lsattr
  - File attributes
  - immutable
---
**File attributes** are flags stored by the filesystem, separate from `rwx`. They change how the file *behaves*, and some of them stop **even root** from modifying a file. Read with `lsattr`, set with `chattr`.

> [!info] Not the same as extended attributes
> - **attributes** (`chattr`/`lsattr`) — a fixed set of on/off flags like `i` and `a`.
> - **extended attributes / xattrs** (`getfattr`/`setfattr`) — arbitrary `name=value` pairs. ACLs ([[acl]]), file capabilities ([[capabilities]]) and SELinux labels ([[mac-selinux-apparmor]]) are stored as xattrs.

# Reading

```bash
lsattr file          # flags of a file
lsattr -d dir        # flags of the directory itself, not its content
lsattr -l file       # long names
```

Real output on this machine:
```
---------------------- /home/alice/notes/README.md               ← no flags (normal)
---------------C------ /var/log/journal                           ← C = No_COW
/var/log/journal/.../system.journal No_COW
```
The journal directory has `C` set, so journal files (which are rewritten in place a lot) skip btrfs copy-on-write → see [[journal]], [[var]].

# The important flags

| Flag | Long name           | Effect                                                               | Who can set |
| ---- | ------------------- | -------------------------------------------------------------------- | ----------- |
| `i`  | Immutable           | Can't modify, delete, rename, link, or change metadata. **Not even root** until removed | root (`CAP_LINUX_IMMUTABLE`) |
| `a`  | Append only         | Can only be opened for appending. No overwrite, no truncate, no delete | root (`CAP_LINUX_IMMUTABLE`) |
| `C`  | No copy-on-write    | Btrfs: write in place. Good for databases, VM images, journals. Set on **empty** files or on the directory before files are created | owner |
| `c`  | Compressed          | Filesystem compresses the file (btrfs)                               | owner |
| `m`  | No compression      | Btrfs: never compress this file, even with `compress=` mount option | owner |
| `A`  | No atime updates    | Don't update access time on read                                     | owner |
| `d`  | No dump             | Skipped by the `dump` backup tool                                    | owner |
| `s`  | Secure deletion     | Request zeroing on delete (ignored by most filesystems)              | owner |
| `u`  | Undeletable         | Request keeping data for undelete (ignored by most filesystems)      | owner |
| `e`  | Extents             | ext4 uses extents for this file (set automatically, informational)   | —     |
| `S`  | Synchronous updates | Changes written to disk immediately                                  | owner |

# Setting

```bash
sudo chattr +i /etc/resolv.conf   # make immutable
sudo chattr -i /etc/resolv.conf   # remove again
sudo chattr +a /var/log/app.log   # append-only log
chattr +C dir                     # new files in dir won't use CoW (btrfs)
chattr -R +A dir                  # recursive
chattr =A file                    # set exactly these flags
```

What happens with `+i`:
```
$ sudo chattr +i test
$ sudo rm test
rm: cannot remove 'test': Operation not permitted
$ echo hi | sudo tee test
tee: test: Operation not permitted
```

> [!warning] "Permission denied" vs "Operation not permitted"
> - `Permission denied` (`EACCES`) → normal [[file-permissions]] say no.
> - `Operation not permitted` (`EPERM`) on a file you *should* be able to change → check `lsattr` for `i` or `a`. Classic case: a tool can't update `/etc/resolv.conf` because someone made it immutable.

# Filesystem support

Not every filesystem supports every flag. Results in the scratchpad (on `tmpfs`) as a normal user:
```
chattr +C c → Operation not supported     (tmpfs has no CoW)
chattr +i c → Operation not permitted      (needs root)
```

| Filesystem | Supports                                               |
| ---------- | ------------------------------------------------------ |
| ext4       | `i a A d e s S u j ...` (most)                         |
| btrfs      | `i a A c C d S` (no `s`, `u`)                          |
| xfs        | `i a A d S` (via `chattr`, plus its own `xfs_io`)      |
| tmpfs      | very few (`i`, `a` in recent kernels)                  |
| FAT/exFAT, NTFS | none via `chattr`                                 |

# Scripting recipes

### Find immutable or append-only files
```bash
lsattr -R /etc 2>/dev/null | awk '$1 ~ /^[-a-zA-Z]+$/ && $1 ~ /[ia]/ {print $1, $2}'
```

### Check before editing a file
```bash
if lsattr "$f" 2>/dev/null | cut -d' ' -f1 | grep -q i; then
  echo "$f is immutable, run: sudo chattr -i $f" >&2
  exit 1
fi
```

### Prepare a no-CoW directory for VM images / databases (btrfs)
```bash
mkdir -p ~/vms
chattr +C ~/vms        # must be done while it's empty
lsattr -d ~/vms
```

# Find stuff

| Question                         | Command                                           |
| -------------------------------- | ------------------------------------------------- |
| Flags of a file / dir            | `lsattr file`, `lsattr -d dir`                    |
| Long flag names                  | `lsattr -l file`                                  |
| All flagged files in a tree      | `lsattr -R dir 2>/dev/null \| awk '$1 ~ /^-*[a-zA-Z][-a-zA-Z]*$/'` |
| Extended attributes (xattrs)     | `getfattr -d -m - file`                           |
| Filesystem type of a path        | `findmnt -no FSTYPE -T path`                      |

# See also
- [[file-permissions]] — normal permissions
- [[capabilities]] — `CAP_LINUX_IMMUTABLE` controls `i` and `a`
- [[acl]] — stored as xattrs
- [[var]], [[journal]] — `C` on journal files
- `man 1 chattr`, `man 1 lsattr`, `man 7 xattr`
