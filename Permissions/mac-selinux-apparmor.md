---
tags:
  - linux
  - permissions
  - security
  - lsm
aliases:
  - MAC
  - SELinux
  - AppArmor
  - LSM
  - Linux Security Modules
  - Landlock
---
Normal permissions are **discretionary** (DAC): the owner of a file decides who may use it. **Mandatory Access Control** (MAC) adds a system-wide **policy** that the owner can't change and that limits **even root**. In Linux, MAC and other hardening are plugged into the kernel as **Linux Security Modules (LSMs)**.

> [!info] Order of checks
> For every access the kernel first does the DAC check ([[file-permissions]], [[acl]], [[capabilities]]), then asks every active LSM. **All** must allow it. An LSM can only *deny* more, never allow something DAC denied.

# The modules

| LSM          | Type       | Idea                                                                 |
| ------------ | ---------- | -------------------------------------------------------------------- |
| **SELinux**  | MAC, labels | Every file, process, port has a **label** (`user:role:type:level`). Policy says which process types may touch which file types. Very fine-grained, complex |
| **AppArmor** | MAC, paths | **Profiles** per program, written as path rules (`/etc/nginx/** r,`). Simpler to read and write |
| **Smack**    | MAC, labels | Simple label-based MAC, used in embedded systems                     |
| **TOMOYO**   | MAC, paths | Learns behaviour, path-based                                         |
| **Landlock** | Self-sandbox | An **unprivileged** process restricts **itself** (e.g. "I will only access ~/project from now on") |
| **Yama**     | Hardening  | Restricts `ptrace` (debugging other processes)                       |
| **Lockdown** | Hardening  | Stops even root from modifying the running kernel (e.g. with Secure Boot) |
| **BPF LSM**  | Programmable | Security policies written as eBPF programs                         |
| `capability` | Base       | The [[capabilities]] logic itself, always active                     |

Usually **one** "major" MAC (SELinux **or** AppArmor) is active, plus several small ones. Which one depends on how the system is set up; the kernel often has several compiled in and only enables some.

# Is it active? (this machine)

```bash
cat /sys/kernel/security/lsm
# capability,landlock,lockdown,yama,bpf
```
→ No SELinux or AppArmor active here, but Landlock, Yama, Lockdown and BPF are.

| Check                          | Command                                    | Here                       |
| ------------------------------ | ------------------------------------------ | -------------------------- |
| Active LSMs                    | `cat /sys/kernel/security/lsm`             | `capability,landlock,lockdown,yama,bpf` |
| Compiled-in LSMs               | `zcat /proc/config.gz \| grep CONFIG_SECURITY_` (if available) | SELinux, AppArmor, Smack, TOMOYO, Landlock = `y` |
| Default LSM order              | `CONFIG_LSM=` in kernel config             | `landlock,lockdown,yama,integrity,bpf` |
| Enable at boot                 | kernel parameter `lsm=...` or `security=...` → `/proc/cmdline`, [[boot]] | not set |
| SELinux                        | `getenforce`, `sestatus`, `/sys/fs/selinux/` exists | not present |
| AppArmor                       | `aa-status`, `/sys/kernel/security/apparmor/` exists | not present |
| Lockdown mode                  | `cat /sys/kernel/security/lockdown`        | `[none] integrity confidentiality` |
| Yama ptrace scope              | `cat /proc/sys/kernel/yama/ptrace_scope`   | `1` |

→ see [[sys]], [[proc]]

# SELinux in short

- Modes: **enforcing** (deny + log), **permissive** (only log), **disabled**. `getenforce` / `setenforce 0|1`.
- Labels visible with `-Z`: `ls -Z`, `ps -eZ`, `id -Z`.
  ```
  system_u:object_r:passwd_file_t:s0 /etc/passwd
  ```
  The important part is the **type** (`passwd_file_t`). Rules are "type `httpd_t` may read type `httpd_sys_content_t`".
- Denials are logged as `avc: denied` in the audit log / journal → [[journal]].
- After copying files into a service directory, labels can be wrong → `restorecon -Rv /path`.
- Config: `/etc/selinux/config` → [[etc]].

> [!warning] "Permission denied" but `rwx` looks fine?
> On a system with SELinux/AppArmor that's often the MAC policy. Check the logs for `avc: denied` (SELinux) or `apparmor="DENIED"` before touching `chmod`.

# AppArmor in short

- One **profile** per program, in `/etc/apparmor.d/` → [[etc]]. Programs without a profile are unconfined.
- Profile modes: **enforce** and **complain** (log only).
- `aa-status` lists loaded profiles; `cat /proc/<pid>/attr/current` (or `/proc/<pid>/attr/apparmor/current`) shows a process's profile.
- Denials logged as `apparmor="DENIED"` in the kernel log.

# Yama — ptrace scope

`/proc/sys/kernel/yama/ptrace_scope`:

| Value | Meaning                                                               |
| ----- | --------------------------------------------------------------------- |
| `0`   | Any process may trace any other process with the same UID             |
| `1`   | Only **descendants** may be traced (default on many systems, **here**) |
| `2`   | Only with `CAP_SYS_PTRACE`                                            |
| `3`   | No ptrace at all, can't be changed until reboot                       |

With `1`, `gdb -p <pid>` or `strace -p <pid>` on an unrelated process of yours fails unless you use `sudo`. `strace ./prog` (child) still works.

# Lockdown

Protects the **running kernel** from root: blocks writing `/dev/mem`, loading unsigned modules, hibernation to untrusted images, etc. Usually turned on automatically with Secure Boot.

| Mode              | Effect                                                     |
| ----------------- | ---------------------------------------------------------- |
| `none`            | Off (**here**)                                             |
| `integrity`       | Root can't modify the kernel                               |
| `confidentiality` | Also can't **read** kernel memory (no `/proc/kcore` etc.)  |

# Landlock

Lets **any** program sandbox itself without root. Tools and libraries use it to say "from now on I may only read these directories". No system policy, nothing to configure as admin — it's a building block for apps and sandboxing tools.

# Scripting recipes

### Detect the active MAC in a script
```bash
lsm=$(cat /sys/kernel/security/lsm 2>/dev/null)
case ",$lsm," in
  *,selinux,*)  echo "SELinux active: $(getenforce 2>/dev/null)";;
  *,apparmor,*) echo "AppArmor active";;
  *)            echo "No major MAC active (LSMs: $lsm)";;
esac
```

### Is ptrace of my own processes allowed?
```bash
scope=$(cat /proc/sys/kernel/yama/ptrace_scope 2>/dev/null || echo 0)
(( scope == 0 )) && echo "can attach to any of my processes" || echo "ptrace restricted (scope $scope)"
```

### Search the logs for MAC denials
```bash
journalctl -k -b | grep -E 'avc: +denied|apparmor="DENIED"'
```

# Find stuff

| Question                     | Command                                           |
| ---------------------------- | ------------------------------------------------- |
| Which LSMs run               | `cat /sys/kernel/security/lsm`                    |
| Security label of files      | `ls -Z`                                           |
| Security label of processes  | `ps -eZ`, `cat /proc/<pid>/attr/current`          |
| SELinux status               | `getenforce`, `sestatus`                          |
| AppArmor status              | `aa-status`                                       |
| Denials                      | `journalctl -k \| grep -E 'avc|apparmor'`         |
| Lockdown / ptrace            | `/sys/kernel/security/lockdown`, `/proc/sys/kernel/yama/ptrace_scope` |

# See also
- [[permissions]] — the whole access check
- [[capabilities]] — the `capability` LSM
- [[sys]] — `/sys/kernel/security`
- [[boot]] — `lsm=` kernel parameter
- `man 8 selinux`, `man 7 apparmor`, `man 7 landlock`, Kernel docs: <https://docs.kernel.org/admin-guide/LSM/index.html>
