---
tags:
  - linux
  - files
  - ssh
  - security
aliases:
  - ~/.ssh
  - .ssh
  - authorized_keys
  - known_hosts
  - ssh config
---
`~/.ssh/` holds your personal SSH setup: your keys, the servers you trust, who may log in **as you**, and shortcuts for hosts you connect to. The system-wide equivalents live in `/etc/ssh/` → see [[etc]].

| File                     | Purpose                                          | Must be private? |
| ------------------------ | ------------------------------------------------ | ---------------- |
| `id_ed25519`             | your **private** key                             | **yes, 600**     |
| `id_ed25519.pub`         | your public key — safe to share                  | no (644)         |
| `authorized_keys`        | public keys allowed to log in **to this account** | 600 or 644, never group/world-writable |
| `known_hosts`            | fingerprints of servers you have connected to    | 600 recommended  |
| `config`                 | per-host shortcuts and options                   | 600 recommended, must not be writable by others |
| `agent/`, sockets        | ssh-agent sockets (some setups)                  | 700              |

> [!danger] Permissions are not optional
> `sshd` refuses to use `authorized_keys` and `ssh` refuses to use a private key if the permissions are too open. You'll see
> `WARNING: UNPROTECTED PRIVATE KEY FILE!` or a silent fallback to password login.
> ```bash
> chmod 700 ~/.ssh
> chmod 600 ~/.ssh/id_* ~/.ssh/config ~/.ssh/authorized_keys
> chmod 644 ~/.ssh/*.pub
> ```
> Your **home directory** also must not be group/world-writable (`StrictModes` in `sshd_config`).

> [!warning] Never paste the private key anywhere
> Only `*.pub` files are meant to be shared. A file containing `-----BEGIN OPENSSH PRIVATE KEY-----` is the secret.

# Key types

| Type          | File name       | Notes                                                    |
| ------------- | --------------- | -------------------------------------------------------- |
| `ssh-ed25519` | `id_ed25519`    | **Use this.** Small, fast, secure                        |
| `ecdsa-sha2-nistp256` | `id_ecdsa` | Fine; common on servers' host keys                      |
| `ssh-rsa`     | `id_rsa`        | Old default. OK at ≥ 3072 bits; SHA-1 signatures are disabled in modern OpenSSH |
| `sk-ssh-ed25519@openssh.com` | `id_ed25519_sk` | Hardware key (YubiKey, FIDO2)              |
| `ssh-dss`     | `id_dsa`        | Obsolete, disabled                                       |

```bash
ssh-keygen -t ed25519 -C "laptop-2026"       # create key (asks for passphrase)
ssh-keygen -lf ~/.ssh/id_ed25519.pub         # fingerprint: 256 SHA256:... comment (ED25519)
ssh-keygen -y -f ~/.ssh/id_ed25519           # regenerate .pub from private key
ssh-keygen -p -f ~/.ssh/id_ed25519           # change passphrase
ssh-copy-id user@host                        # append your .pub to the server's authorized_keys
```

# `authorized_keys`
One public key per line. Lines starting with `#` are comments.

```
[options] keytype base64-key [comment]
```

| Part     | Example                                    |
| -------- | ------------------------------------------ |
| options  | `from="192.168.1.0/24",no-port-forwarding` (optional, comma separated, no spaces) |
| keytype  | `ssh-ed25519`                              |
| key      | `AAAAC3NzaC1lZDI1NTE5AAAA...` (base64)     |
| comment  | `alice@laptop` (free text, ignored)         |

Useful options:

| Option                      | Effect                                       |
| --------------------------- | -------------------------------------------- |
| `from="pattern"`            | only accept from these IPs/hosts             |
| `command="/path/script"`    | always run this command, ignore what client asked (`$SSH_ORIGINAL_COMMAND` holds the request) |
| `restrict`                  | disable forwarding, PTY, etc. — then allow selectively |
| `no-pty`, `no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding` | self-explanatory |
| `expiry-time="20271231"`    | key stops working after that date            |

Example — a backup-only key:
```
restrict,command="/usr/local/bin/backup-receive" ssh-ed25519 AAAAC3Nz... backup@nas
```

# `known_hosts`
Every time you connect, `ssh` checks the server's **host key** against this file. A mismatch means the server was reinstalled — or someone is in the middle.

```
hostname[,ip] keytype base64-key
|1|salt|hash keytype base64-key            # hashed form (HashKnownHosts yes)
[host.example]:2222 ssh-ed25519 AAAA...    # non-standard port
```

| Situation                                             | What to do                         |
| ----------------------------------------------------- | ---------------------------------- |
| `The authenticity of host ... can't be established`   | First connection. Verify the fingerprint out-of-band, then `yes` |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`    | **Stop.** If you know the server was reinstalled: `ssh-keygen -R host` |

```bash
ssh-keygen -F github.com          # is this host in known_hosts?
ssh-keygen -R oldserver           # remove its entries (keeps known_hosts.old backup)
ssh-keygen -H                     # hash all hostnames (privacy if the file leaks)
```

# `config`
Blocks start with `Host <pattern>`. For each option the **first** value found wins, so put specific hosts at the top and `Host *` at the bottom.

```
Include ~/.ssh/config.d/*

Host nas
    HostName 192.168.1.20
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_nas
    IdentitiesOnly yes

Host work-*
    User jdoe
    ProxyJump bastion.example.com

Host *
    ServerAliveInterval 30
    AddKeysToAgent yes
```
Now `ssh nas` = `ssh -p 2222 -i ~/.ssh/id_ed25519_nas admin@192.168.1.20`. It also works for `scp`, `rsync`, `git`.

| Option             | Meaning                                                 |
| ------------------ | ------------------------------------------------------- |
| `HostName`         | real address                                            |
| `User`, `Port`     | defaults for that host                                  |
| `IdentityFile`     | which private key                                       |
| `IdentitiesOnly yes` | only try that key (avoids "Too many authentication failures") |
| `ProxyJump`        | hop through a bastion host                              |
| `LocalForward 8080 localhost:80` | port forwarding                           |
| `ForwardAgent yes` | let the remote use your agent — only for hosts you trust |
| `Include`          | split config into several files. Put it at the **top**: inside a `Host` block it only applies to that block |

```bash
ssh -G nas | grep -E '^(hostname|user|port|identityfile) '   # final config ssh will use
```

# Scripting recipes

### Audit permissions
```bash
for f in ~/.ssh ~/.ssh/config ~/.ssh/authorized_keys ~/.ssh/known_hosts; do
  [ -e "$f" ] && stat -c '%a %n' "$f"
done
# private keys should all be 600:
find ~/.ssh -type f ! -name '*.pub' -exec grep -l 'PRIVATE KEY' {} + 2>/dev/null |
  xargs -r stat -c '%a %n'
```

### Fingerprints of all public keys
```bash
for f in ~/.ssh/*.pub; do [ -f "$f" ] && ssh-keygen -lf "$f"; done
```

### Key types allowed to log in to this account
```bash
awk '!/^#/ && NF {for(i=1;i<=NF;i++) if($i ~ /^(ssh-|ecdsa-|sk-)/){print $i; break}}' \
    ~/.ssh/authorized_keys | sort | uniq -c
```

### Non-interactive check from scripts
```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 host true && echo "key login works"
```
`BatchMode=yes` makes ssh fail instead of asking for a password — essential in cron jobs and [[timers]].

### Who is logged in over SSH right now
```bash
who | grep -v ':0'          # remote sessions show an IP in ()
```
→ [[wtmp-btmp-lastlog]]

# See also
- [[etc]] — `/etc/ssh/sshd_config` (server) and `/etc/ssh/ssh_config` (client defaults)
- [[home]] — dotfiles in general
- [[networking]] — ports, firewall
- [[wtmp-btmp-lastlog]] — login history
- [[files]] — all file notes
- `man ssh_config`, `man sshd` → **AUTHORIZED_KEYS FILE FORMAT**, `man ssh-keygen`
