---
tags:
  - linux
  - users
  - security
  - authentication
aliases:
  - PAM
  - Pluggable Authentication Modules
  - /etc/pam.d
---
**PAM** (Pluggable Authentication Modules) is the library every login-related program uses to ask "is this really the user, and may they log in?". `login`, `sshd`, `su`, `sudo`, display managers, `passwd`, screen lockers — none of them check passwords themselves. They call PAM, and PAM runs a **stack of modules** configured per program in `/etc/pam.d/`.

> [!info] Why it exists
> Change how *everyone* authenticates (add 2FA, lock after failed attempts, use LDAP, fingerprint) by editing config, without recompiling `sudo` or `sshd`.

# Files

| Path                           | What                                                           |
| ------------------------------ | -------------------------------------------------------------- |
| `/etc/pam.d/<service>`         | Stack for one program: `login`, `sshd`, `su`, `sudo`, `passwd`, ... |
| `/etc/pam.d/` shared stacks    | Included by others. Names vary by distro: `system-auth`, `system-login`, `common-auth`, `common-session`, ... |
| `/etc/pam.d/other`             | Fallback for programs without their own file (should deny)     |
| `/etc/security/`               | Config for individual modules (`limits.conf`, `faillock.conf`, `access.conf`, `pam_env.conf`, `time.conf`) |
| `/lib/security/` or `/usr/lib/security/` or `/usr/lib/x86_64-linux-gnu/security/` | The modules themselves (`pam_*.so`), path varies by distro |

The service name is chosen by the program: `sudo` reads `/etc/pam.d/sudo`, `sshd` reads `/etc/pam.d/sshd`. More in [[pam-d|pam.d]].

# Line format

```
type     control     module            arguments
auth     sufficient  pam_rootok.so
auth     required    pam_unix.so
```

Real `/etc/pam.d/su` on this machine:
```
auth            sufficient      pam_rootok.so
auth            required        pam_unix.so
account         required        pam_unix.so
session         required        pam_unix.so
session         required        pam_limits.so
password        include         system-auth
```
Read it as: "If the caller is already root, let them in. Otherwise check the password against [[shadow]]. Then check the account is valid, and at session start apply resource limits."

### The four types (stacks)

| Type       | Question it answers                                    | Example modules                     |
| ---------- | ------------------------------------------------------ | ----------------------------------- |
| `auth`     | **Who are you?** Prove identity (password, key, fingerprint) | `pam_unix`, `pam_faillock`, `pam_u2f`, `pam_fprintd` |
| `account`  | **Are you allowed now?** Expired? Locked? Right time / host? | `pam_unix`, `pam_access`, `pam_time`, `pam_nologin` |
| `password` | **Change the credential**                              | `pam_unix`, `pam_pwquality`         |
| `session`  | **Set up / tear down** the login                       | `pam_limits`, `pam_env`, `pam_systemd`, `pam_mkhomedir`, `pam_motd` |

A leading `-` (e.g. `-session optional pam_systemd.so`) means "skip quietly if the module file doesn't exist".

### Control flags

| Flag         | If module succeeds               | If module fails                         |
| ------------ | -------------------------------- | --------------------------------------- |
| `required`   | Continue                         | Remember failure, **continue** stack, fail at end |
| `requisite`  | Continue                         | **Fail immediately**                    |
| `sufficient` | **Succeed immediately** (if no earlier `required` failed) | Ignore, continue |
| `optional`   | Ignored unless it's the only module | Ignored                              |
| `include`    | Insert the whole stack of another file here                   ||
| `substack`   | Like include, but `done`/`die` stop only the substack         ||
| `[value=action ...]` | Fine-grained form, e.g. `[success=1 default=ignore]` = on success skip the next 1 line ||

> [!tip] Why `required` continues after failure
> So an attacker can't tell *which* check failed (user doesn't exist vs wrong password) from timing or error messages.

# Common modules

| Module            | Type(s)            | Does                                                          |
| ----------------- | ------------------ | ------------------------------------------------------------- |
| `pam_unix.so`     | all four           | Classic password check against [[passwd]]/[[shadow]]; hashes new passwords |
| `pam_rootok.so`   | auth               | Succeeds if caller's UID is 0 (why root can `su` without a password) |
| `pam_faillock.so` | auth, account      | Locks account after N failed attempts (`/etc/security/faillock.conf`); `faillock --user USER` shows/resets |
| `pam_limits.so`   | session            | Applies `ulimit`s from `/etc/security/limits.conf` and `limits.d/` |
| `pam_env.so`      | auth, session      | Sets environment from `/etc/environment`, `/etc/security/pam_env.conf` |
| `pam_nologin.so`  | auth, account      | If `/etc/nologin` (or `/run/nologin`) exists, only root may log in |
| `pam_shells.so`   | auth               | User's shell must be listed in `/etc/shells`                  |
| `pam_access.so`   | account            | Allow/deny by user and origin (`/etc/security/access.conf`)   |
| `pam_time.so`     | account            | Allow login only at certain times (`/etc/security/time.conf`) |
| `pam_wheel.so`    | auth               | Only `wheel` members may `su` to root (if enabled)            |
| `pam_systemd.so`  | session            | Registers session with logind, creates `/run/user/UID`, see [[login-sessions]] |
| `pam_loginuid.so` | session            | Sets audit login UID (`/proc/self/loginuid`)                  |
| `pam_umask.so`    | session            | Sets umask from `login.defs` / GECOS, see [[umask]]           |
| `pam_motd.so`     | session            | Shows message of the day                                      |
| `pam_mkhomedir.so`| session            | Creates home on first login (LDAP setups)                     |
| `pam_pwquality.so`| password           | Password strength rules                                       |
| `pam_permit.so` / `pam_deny.so` | any  | Always succeed / always fail                                  |

Most-used modules across all stacks here:
```
$ grep -hoE 'pam_[a-z0-9_]+\.so' /etc/pam.d/* | sort | uniq -c | sort -rn | head -5
     24 pam_unix.so
     10 pam_permit.so
      8 pam_rootok.so
      6 pam_nologin.so
      5 pam_env.so
```

# Following a login through PAM
`sudo` on this machine:
```
/etc/pam.d/sudo
  auth     include  system-auth      → faillock preauth → pam_unix (your password) → faillock
  account  include  system-auth      → pam_unix (not expired?) → pam_time
  session  include  system-auth      → pam_limits → pam_unix (log session)
```
Shared stack names (`system-auth` here, `common-auth` on other distros) let all programs share one policy: change it once, and `login`, `su`, `sudo`, the display manager all follow.

> [!danger] Editing PAM can lock you out
> One typo in a shared stack can break **every** login, including `sudo` to fix it.
> - Keep a root shell open in another terminal while you test.
> - Test with a second login (`su - youruser`, new SSH session) before closing it.
> - Never put `pam_deny.so` or a missing module as `required` in a shared stack.

# Scripting recipes

### Which PAM services include a given stack?
```bash
grep -l 'system-auth\|common-auth' /etc/pam.d/* 2>/dev/null
```

### Which modules does a service use (following includes one level)?
```bash
svc=sudo
awk '!/^#/ && NF {print}' "/etc/pam.d/$svc" | while read -r type ctl mod _; do
  if [[ $ctl == include || $ctl == substack ]]; then
    awk -v t="${type#-}" '!/^#/ && $1 ~ t {print "  (via '"$mod"')", $0}' "/etc/pam.d/$mod"
  else
    echo "$type $ctl $mod"
  fi
done
```

### Where are the modules installed?
```bash
find /lib /usr/lib /lib64 /usr/lib64 -name pam_unix.so 2>/dev/null
```
```
/usr/lib/security/pam_unix.so
```

### Is login disabled for non-root users?
```bash
[[ -e /etc/nologin || -e /run/nologin ]] && echo "nologin active"
```

### Current limits applied by pam_limits
```bash
ulimit -a              # this shell
cat /proc/self/limits  # any process, see [[proc]]
```

# Find stuff

| Question                              | Where / command                         |
| ------------------------------------- | --------------------------------------- |
| PAM config of a program               | `/etc/pam.d/<program>`                  |
| Why did a login fail?                 | `journalctl -t sudo -t login -t sshd -p warning` |
| Failed-attempt lock status            | `faillock --user USER`                  |
| Login limits (nproc, nofile)          | `/etc/security/limits.conf`, `limits.d/` |

# See also
- [[users]] · [[sudo-su]] · [[login-sessions]] · [[user-management]]
- [[pam-d|pam.d]] · [[shadow]] · [[passwd]] · [[login-defs|login.defs]]
- [[umask]] · [[etc]]
- `man 8 pam`, `man 5 pam.d`, `man 8 pam_unix`, `man 8 pam_faillock`, `man 5 limits.conf`
