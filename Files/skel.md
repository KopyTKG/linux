---
tags:
  - linux
  - files
  - users
aliases:
  - /etc/skel
  - skeleton directory
---
`/etc/skel` ("skeleton") is the **template for new home directories**. When an administrator creates a user with `useradd -m` (or `adduser` on Debian-based systems), the new home is created and everything inside `/etc/skel` — including hidden dotfiles — is copied into it and handed to the new user.

```
$ ls -la /etc/skel
-rw-r--r-- 1 root root   21  .bash_logout
-rw-r--r-- 1 root root   57  .bash_profile
-rw-r--r-- 1 root root  172  .bashrc
drwxr-xr-x 1 root root       .config/
-rw-r--r-- 1 root root   56  .zshrc
```
The exact contents differ per distribution and per installed packages: shell packages put default `.bashrc`/`.zshrc` here, desktop environments may add `.config/` files.

> [!info] Copied once, never synced
> `/etc/skel` is used **only at the moment a user is created**. Changing it later does not touch existing users. And existing users' files are theirs — they can edit or delete them freely.

> [!tip] Restore a default dotfile
> Broke your `~/.bashrc`? Compare with or copy back the original:
> ```bash
> diff /etc/skel/.bashrc ~/.bashrc
> cp /etc/skel/.bashrc ~/.bashrc
> ```

# Related settings
How `useradd` behaves is configured in two files in [[etc]]:

| File                    | Key                  | Meaning                                         |
| ----------------------- | -------------------- | ----------------------------------------------- |
| `/etc/default/useradd`  | `SKEL=/etc/skel`     | which template directory to copy                |
|                         | `SHELL=/bin/bash`    | default login shell                             |
|                         | `HOME=/home`         | parent of new home dirs                         |
| `/etc/login.defs`       | `CREATE_HOME yes`    | create home even without `-m`                   |
|                         | `HOME_MODE 0700`     | permissions of new home dirs                    |
|                         | `UMASK 022`          | default umask (used for home if `HOME_MODE` unset) |
|                         | `USERGROUPS_ENAB yes`| create a private group per user                 |

```bash
useradd -D                                     # show current defaults
grep -E '^(UMASK|HOME_MODE|CREATE_HOME|USERGROUPS_ENAB)' /etc/login.defs
```

# Using it
```bash
# as root: create a user with home from /etc/skel
useradd -m -s /bin/bash alice

# use a different template
useradd -m -k /srv/skel-students bob
```

Ideas for a custom skeleton (on a machine with many users):
- a standard `.bashrc` with company aliases
- `~/.config/git/config` with a commit template
- an empty `~/bin` or `~/.local/bin`
- a `README.txt` explaining where to store data

> [!warning] Don't put secrets in skel
> Every new user receives a copy — never place keys, tokens or passwords there.

# Scripting recipes

### Which of my dotfiles still match the defaults?
```bash
for f in /etc/skel/.*; do
  [[ -f $f ]] || continue
  n=${f##*/}
  if cmp -s "$f" ~/"$n"; then echo "$n: unchanged"; else echo "$n: modified or missing"; fi
done
```

### List everything a new user would get
```bash
find /etc/skel -mindepth 1 -printf '%M %p\n'
```

### Copy skel into an existing home without overwriting anything
```bash
cp -rn /etc/skel/. ~/      # -n: never overwrite; "/." copies hidden files too
```

# See also
- [[home]] — what lives in a home directory
- [[bashrc]], [[profile]], [[fish-config]] — the dotfiles skel provides
- [[passwd]], [[shadow]] — where the user account itself is stored
- [[etc]] — `/etc/login.defs`, `/etc/default/useradd`
- [[files]] — all file notes
- `man useradd`, `man login.defs`
