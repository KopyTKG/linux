---
tags:
  - linux
  - files
  - etc
  - users
aliases:
  - /etc/login.defs
  - login.defs
---
`/etc/login.defs` holds the **defaults for the shadow-utils tools**: `useradd`, `userdel`, `passwd`, `chage`, `login`, `su`. It decides which UID a new user gets, how passwords are hashed, the default `umask`, and more.

> [!info] Quick facts
> - Path: `/etc/login.defs` · mode `0644`
> - Format: `KEY  value`, one per line, `#` comments
> - Read **only when a tool runs**. Changing it doesn't touch existing users
> - Manual: `man 5 login.defs`

> [!note] PAM overrides part of it
> On PAM systems (nearly all), things like password length rules and login retries are mostly controlled by [[pam-d]]. Values here that PAM also handles may be ignored. Keys about UIDs, hashing and new homes still matter.

# Important keys (common defaults)

### Account ID ranges
| Key                         | Value                  | Meaning                                       |
| --------------------------- | ---------------------- | --------------------------------------------- |
| `UID_MIN` / `UID_MAX`       | `1000` / `60000`       | Range for normal users (`useradd`)            |
| `SYS_UID_MIN` / `SYS_UID_MAX` | `500` / `999`        | Range for `useradd --system`                  |
| `GID_MIN` / `GID_MAX`       | `1000` / `60000`       | Same for groups                               |
| `SUB_UID_MIN` / `SUB_UID_COUNT` | `100000` / `65536` | Subordinate UIDs per user (rootless containers, `/etc/subuid`) |

### Passwords
| Key                 | Value      | Meaning                                                  |
| ------------------- | ---------- | -------------------------------------------------------- |
| `ENCRYPT_METHOD`    | `YESCRYPT` | Hash algorithm for new passwords → `$y$` in [[shadow]]   |
| `PASS_MAX_DAYS`     | *(unset)*  | Default `max` for new users (`99999` if unset)           |
| `PASS_MIN_DAYS`     | *(unset)*  | Default `min`                                            |
| `PASS_WARN_AGE`     | *(unset)*  | Default `warn`                                           |
| `PASS_MIN_LEN`      | `5`        | Min length (often ignored, PAM decides)                  |

### Login and homes
| Key               | Value                                     | Meaning                                       |
| ----------------- | ----------------------------------------- | --------------------------------------------- |
| `UMASK`           | `022`                                     | Default permission mask (files `644`, dirs `755`) |
| `HOME_MODE`       | `0700`                                    | Mode of new home dirs (others can't look in)  |
| `USERGROUPS_ENAB` | `yes`                                     | Each new user gets an own group with same name |
| `ENV_PATH`        | `PATH=/usr/local/sbin:/usr/local/bin:/usr/bin` | `PATH` set by `login` for users           |
| `ENV_SUPATH`      | same                                      | `PATH` for root                               |
| `FAIL_DELAY`      | `3`                                       | Seconds to wait after a failed login          |
| `LOGIN_RETRIES`   | `5`                                       | Tries before `login` gives up                 |
| `LOGIN_TIMEOUT`   | `60`                                      | Seconds to type the password                  |
| `MAIL_DIR`        | `/var/spool/mail`                         | Mailbox folder created for users              |
| `HUSHLOGIN_FILE`  | `.hushlogin`                              | If `~/.hushlogin` exists, no [[issue-motd]] message at login |

> [!tip] Where useradd's other defaults live
> `useradd -D` shows the rest (default shell, skeleton dir). They are stored in `/etc/default/useradd`. Files copied into a new home come from [[skel]].

# Scripting recipes

### Read one value
```bash
awk '$1=="UID_MIN" {print $2}' /etc/login.defs      # 1000
```

### All active settings (no comments)
```bash
grep -vE '^\s*(#|$)' /etc/login.defs
```

### List human users using the real ranges, not hardcoded 1000
```bash
min=$(awk '$1=="UID_MIN"{print $2}' /etc/login.defs)
max=$(awk '$1=="UID_MAX"{print $2}' /etc/login.defs)
awk -F: -v a="$min" -v b="$max" '$3>=a && $3<=b {print $1}' /etc/passwd
```

# See also
- [[passwd]] · [[shadow]] · [[group]] — the files these defaults apply to
- [[pam-d]] — what actually checks logins
- [[skel]] · [[profile]] · [[etc]]
