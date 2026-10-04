---
tags:
  - linux
  - files
  - etc
  - users
  - security
aliases:
  - /etc/gshadow
---
`/etc/gshadow` is to [[group]] what [[shadow]] is to [[passwd]]: the secret part. It holds **group passwords** and **group administrators**.

> [!info] Quick facts
> - Path: `/etc/gshadow` · owner `root:root` · mode `0600`
> - Readable only by root
> - In practice almost every group has no password here. You will rarely touch it
> - Edit with `gpasswd` or `sudo vigr -s`
> - Manual: `man 5 gshadow`

# Format
Four fields separated by `:`. Fake example:

```
project:!::alice,bob
devs:$y$j9T$...:carol:alice,bob
```

| #   | Field    | Example       | Meaning                                                            |
| --- | -------- | ------------- | ------------------------------------------------------------------ |
| 1   | name     | `devs`        | Group name, same as in [[group]]                                   |
| 2   | password | `$y$...` / `!`| Hash (same formats as [[shadow]]). `!` or `*` = no password        |
| 3   | admins   | `carol`       | Users allowed to add/remove members with `gpasswd` (no root needed)|
| 4   | members  | `alice,bob`   | Members. Should match field 4 of [[group]]                         |

# What a group password does
With a password set, a user who is **not** a member can run `newgrp devs`, type the group password and get that group for the shell. Everyone who knows the password shares it, so this is considered bad practice. Use membership instead.

# Group admins
Delegate membership management without giving sudo:

```bash
sudo gpasswd -A carol devs     # carol becomes admin of devs
gpasswd -a alice devs          # carol (no sudo) adds alice
gpasswd -d alice devs          # carol removes alice
```

# Scripting recipes
Root needed.

```bash
# groups that have a real password set (should usually be empty)
sudo awk -F: '$2 ~ /^\$/ {print $1}' /etc/gshadow

# groups with admins
sudo awk -F: '$3 != "" {print $1, "admins:", $3}' /etc/gshadow

# check group and gshadow agree
sudo grpck -r
```

# See also
- [[group]] · [[shadow]] · [[passwd]] · [[etc]]
