---
tags:
  - linux
  - files
  - etc
  - network
  - security
aliases:
  - /etc/ssh/ssh_config
  - /etc/ssh/sshd_config
  - sshd_config
  - ssh_config
---
`/etc/ssh/` configures **OpenSSH** for the whole machine. Two files, easy to mix up:

| File                       | Program | Role                                                        |
| -------------------------- | ------- | ----------------------------------------------------------- |
| `/etc/ssh/ssh_config`      | `ssh`   | **Client**: how *this* machine connects *out* to others     |
| `/etc/ssh/sshd_config`     | `sshd`  | **Server**: who may connect *in* to this machine, and how   |

Your personal client settings and keys live in `~/.ssh/` → [[ssh-user]].

> [!info] Quick facts
> - Both files support `Include`; on many systems they include `ssh_config.d/*.conf` and `sshd_config.d/*.conf`. **Put changes in a drop-in**, so upgrades don't conflict
> - Host keys (server identity): `/etc/ssh/ssh_host_*_key` (private, root only) and `*.pub`. Created on the server's first start; none exist here because `sshd` is disabled
> - `moduli`: Diffie-Hellman groups, leave alone
> - Manual: `man 5 ssh_config`, `man 5 sshd_config`

> [!important] First match wins
> For each option, ssh/sshd uses the **first** value it finds. That's why `Include` sits at the **top** and drop-ins override the main file, and why in a client config specific `Host` blocks must come before `Host *`.

# Client: `ssh_config`
Order of reading: command line `-o` → `~/.ssh/config` → `/etc/ssh/ssh_config` (+ drop-ins). The user file wins over the system file.

```
Host nas
    HostName 192.168.1.10
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

Host *
    ServerAliveInterval 60
```

| Option                  | Meaning                                                  |
| ----------------------- | -------------------------------------------------------- |
| `Host`                  | Pattern this block applies to (`*`, `*.lan`, `nas`)      |
| `HostName`, `User`, `Port` | Real address, login name, port                        |
| `IdentityFile`          | Private key to use                                       |
| `ProxyJump`             | Connect through a jump host                              |
| `ForwardAgent`          | Let the remote use your agent. Only for trusted hosts    |
| `StrictHostKeyChecking` | `ask` (default) / `accept-new` / `no`                    |
| `ServerAliveInterval`   | Keepalive so idle connections don't drop                 |

On recent systemd systems a drop-in (`20-systemd-ssh-proxy.conf`) teaches ssh to connect to local VMs/containers via names like `machine/<name>` and `vsock/<cid>`.

# Server: `sshd_config`
Active lines here (main file + drop-ins):
```
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
UsePAM yes
PrintMotd no
AuthorizedKeysFile .ssh/authorized_keys
AuthorizedKeysCommand /usr/bin/userdbctl ssh-authorized-keys %u
Subsystem sftp /usr/lib/ssh/sftp-server
```

### Options that matter for security
| Option                       | Good value          | Meaning                                       |
| ---------------------------- | ------------------- | --------------------------------------------- |
| `PermitRootLogin`            | `no` / `prohibit-password` | Root login only with key, or never     |
| `PasswordAuthentication`     | `no`                | Keys only. Kills password brute-force         |
| `KbdInteractiveAuthentication` | `no`              | Another password path; disable with the above |
| `PubkeyAuthentication`       | `yes`               | Key login                                     |
| `AllowUsers` / `AllowGroups` | `alice` / `sshusers`| Only these may log in                         |
| `Port`                       | `22`                | Changing it only reduces log noise            |
| `MaxAuthTries`               | `3`                 | Tries per connection                          |
| `UsePAM`                     | `yes`               | Use [[pam-d]] `/etc/pam.d/sshd`               |
| `X11Forwarding`              | `no`                | Unless you need remote GUI apps               |

> [!danger] Don't lock yourself out of a remote server
> 1. Test syntax: `sudo sshd -t`
> 2. Reload, don't restart: `sudo systemctl reload sshd` (existing sessions stay)
> 3. Keep your current session open and test a **new** login in a second terminal before you disconnect.
> Disable `PasswordAuthentication` only **after** key login works.

# Scripting recipes

### Is the server running / enabled?
```bash
systemctl is-active sshd; systemctl is-enabled sshd     # e.g. inactive, disabled on a desktop
ss -tlnp | grep ':22 '
```
(The unit is `ssh.service` on Debian/Ubuntu.)

### Effective client settings for a host (after all files and matching)
```bash
ssh -G example.com | grep -E '^(user|port|hostname|identityfile) '
```

### Effective server settings
```bash
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|port) '
```

### Active lines from the main files and drop-ins
```bash
grep -hvE '^\s*(#|$)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf
```

### Fingerprint of the server's host key (compare with what clients see)
```bash
for k in /etc/ssh/ssh_host_*_key.pub; do ssh-keygen -lf "$k"; done
```

### Failed login attempts
```bash
journalctl -u sshd --since today | grep -i 'failed\|invalid'
```

# See also
- [[ssh-user]] — `~/.ssh/config`, keys, `authorized_keys`, `known_hosts`
- [[pam-d]] · [[sudoers]] · [[wtmp-btmp-lastlog]] — login records
- [[networking]] · [[sockets-ports]] · [[firewall]] · [[systemd]] · [[etc]]
