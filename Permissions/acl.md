---
tags:
  - linux
  - permissions
  - acl
aliases:
  - ACL
  - Access Control Lists
  - getfacl
  - setfacl
---
Classic permissions allow exactly **one** owner and **one** group per file. **POSIX ACLs** (Access Control Lists) add extra entries: "user `alice` may read, group `devs` may write", without changing owner or group.

Tools: `getfacl` (read) and `setfacl` (write). Supported by ext4, btrfs, xfs, tmpfs and most Linux filesystems (not FAT/exFAT).

# The `+` in `ls -l`

```
-rw-r--r--+ 1 alice alice 0 Oct  4 17:53 a
          ^
          └─ this file has extra ACL entries
```

# Reading an ACL

Demo in the scratchpad: give user `nobody` read access to file `a`:
```bash
setfacl -m u:nobody:r a
getfacl a
```
```
# file: a
# owner: alice
# group: alice
user::rw-          ← owner           (= classic owner bits)
user:nobody:r--    ← named user
group::r--         ← owning group
mask::r--          ← upper limit for named users + all groups
other::r--         ← everyone else   (= classic other bits)
```

| Entry           | Meaning                                                 |
| --------------- | ------------------------------------------------------- |
| `user::`        | File owner                                              |
| `user:NAME:`    | Specific user                                           |
| `group::`       | Owning group                                            |
| `group:NAME:`   | Specific group                                          |
| `mask::`        | **Maximum** rights for named users, owning group, named groups |
| `other::`       | Everyone else                                           |

# The mask

The mask is a ceiling. An entry's **effective** rights = entry AND mask. `getfacl` prints `#effective:` when they differ.

```bash
setfacl -m m::--- a
getfacl a | grep -E 'nobody|mask'
```
```
user:nobody:r--	#effective:---
mask::---
```

> [!warning] With an ACL, the group bits in `ls -l` show the **mask**, not the group
> Once a file has an ACL, `chmod g=...` changes the **mask**. So `chmod 600 file` can silently disable all named entries.
> Seen in the demo: after setting the mask to `---` and removing the ACL with `setfacl -b`, the file was left as `-rw----r--` — the group bits kept the mask value.

`setfacl` recalculates the mask automatically when you add entries (union of all group-class entries). Use `-n` to stop that.

# Setting ACLs

```bash
setfacl -m u:alice:rw file        # add / modify user entry
setfacl -m g:devs:rx dir          # group entry
setfacl -m u:alice:rw,g:devs:r f  # several at once
setfacl -x u:alice file           # remove one entry
setfacl -b file                   # remove ALL extended entries
setfacl -R -m g:devs:rwX dir      # recursive, X = x only for dirs
getfacl a | setfacl --set-file=- b   # copy ACL from a to b
```

# Default ACLs (inheritance)

A directory can have a **default ACL**. Every new file or subdirectory created inside gets it as its own ACL. Default ACLs **replace the [[umask]]** for those files.

```bash
mkdir share
setfacl -d -m g:audio:rwX share
touch share/new
getfacl -c share/new        # -c = no header
```
```
user::rw-
group::r-x	#effective:r--
group:audio:rwx	#effective:rw-
mask::rw-
other::r--
```
The new file requested `666` (no `x`), so the mask became `rw-` and group `audio` effectively got `rw-`.

> [!tip] Shared folder recipe
> Combine with setgid ([[special-bits]]) so the group is also inherited:
> ```bash
> chgrp devs /srv/project
> chmod 2770 /srv/project
> setfacl -R -m g:devs:rwX /srv/project      # existing files
> setfacl -R -d -m g:devs:rwX /srv/project   # future files
> ```

# Gotchas

| Situation                     | What happens                                                    |
| ----------------------------- | --------------------------------------------------------------- |
| `cp file dest`                | ACL **not** copied. Use `cp -a` or `cp --preserve=all`          |
| `mv` on same filesystem       | ACL kept (same inode)                                           |
| `tar`                         | Needs `--acls` to store/restore                                 |
| `rsync`                       | Needs `-A`                                                      |
| FAT, exFAT, some network FS   | No ACL support                                                  |
| Root                          | ACLs don't limit root (capabilities bypass DAC) → [[capabilities]] |

# Scripting recipes

### Does a file have an ACL?
```bash
ls -ld "$f" | cut -c11 | grep -q '+' && echo "$f has an ACL"
```
(`stat -c %A` does **not** show the `+`, only `ls` does.)

### List all files with ACLs under a directory
```bash
getfacl -R -s -p /path 2>/dev/null | awk '/^# file:/ {print $3}'
```
`-s` = skip files with only the basic entries, `-p` = keep leading `/`.

### Backup and restore ACLs
```bash
getfacl -R -p /srv/project > acl-backup.txt
setfacl -P --restore=acl-backup.txt   # -P = don't follow symlinks
```

# Find stuff

| Question                       | Command                            |
| ------------------------------ | ---------------------------------- |
| ACL of one file                | `getfacl file`                     |
| Only extended entries          | `getfacl -c -s file`               |
| Files with ACLs                | `getfacl -R -s -p dir 2>/dev/null` |
| Does the FS support ACLs       | `setfacl -m u:$USER:r file` (error = no) |

# See also
- [[file-permissions]] — the base `rwx` model ACLs extend
- [[umask]] — replaced by default ACLs
- [[special-bits]] — setgid directories for shared folders
- [[permissions]] — overview
- `man 5 acl`, `man 1 getfacl`, `man 1 setfacl`
