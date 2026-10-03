# Chapter 04 — Users, groups and permissions

Two halves of one question: *who* is this process acting as, and *what* is it allowed to
do? The first half covers accounts, passwords, sudo, login environments, resource limits
and directory-backed users from LDAP. The second covers every permission mechanism and
how to tell which one is refusing you.

LFCS weight: **Users and Groups, 10%** — every competency in that domain is in this chapter.

## Objectives

After this chapter you can:

- Create, modify, lock and delete users and groups, including system accounts.
- Read `/etc/passwd`, `/etc/shadow` and `/etc/group` field by field.
- Set password ageing and account expiry policy.
- Grant precise privileges with `sudo`, safely.
- Configure personal and system-wide environment profiles, and know which file runs when.
- Set per-user and per-group resource limits.
- Make a machine resolve and authenticate users from an LDAP directory.
- Explain what `r`, `w` and `x` mean on a directory, and compute umask results.
- Use SUID, SGID, sticky, ACLs, attributes and capabilities — and debug any "permission denied".

---

# Part 1 — Users and groups

## 1. How Linux sees a user

To the kernel a user is just a number, the **UID**. Names exist for humans and are looked
up through the Name Service Switch (`/etc/nsswitch.conf`), which can consult local files,
LDAP, or other sources.

```bash
id                  # uid=1000(vagrant) gid=1000(vagrant) groups=1000(vagrant),27(sudo)
id alice
getent passwd alice # look up through NSS — works for local AND LDAP users
getent group sudo
```

Always prefer `getent` to `grep /etc/passwd`: `grep` only sees local files, so it silently
misses directory users.

| UID range | Meaning |
|---|---|
| 0 | root — the only UID the kernel treats specially |
| 1–999 | system accounts for services (`www-data`, `sshd`, `systemd-*`) |
| 1000+ | regular human users |
| 65534 | `nobody` — the "no privileges" user |

The boundaries come from `UID_MIN` / `SYS_UID_MAX` in `/etc/login.defs`.

---

## 2. The account files

### /etc/passwd — world-readable

```text
alice:x:1001:1001:Alice Smith,,,:/home/alice:/bin/bash
  │   │  │    │       │              │           └ login shell
  │   │  │    │       │              └ home directory
  │   │  │    │       └ comment (GECOS)
  │   │  │    └ primary group ID
  │   │  └ UID
  │   └ "x": the password hash is in /etc/shadow
  └ username
```

### /etc/shadow — root only

```text
alice:$y$j9T$…:20000:0:90:7:14:20500:
  │      │       │   │  │  │  │    └ account expiry (days since 1970-01-01)
  │      │       │   │  │  │  └ inactive days allowed after password expiry
  │      │       │   │  │  └ warning days before expiry
  │      │       │   │  └ maximum days between changes
  │      │       │   └ minimum days between changes
  │      │       └ last change (days since 1970-01-01)
  │      └ hash: $y$ yescrypt, $6$ SHA-512; "!" or "*" means locked / no password
  └ username
```

### /etc/group and /etc/gshadow

```text
devs:x:1005:alice,bob
```

Group name, password placeholder, GID, and the **supplementary** members. A user's
**primary** group is the GID in `/etc/passwd` — it is usually *not* listed here.

Never edit these files with a normal editor while the system is running. Use the tools
below, or `vipw` / `vigr`, which lock the file and check syntax.

---

## 3. Managing users

```bash
sudo useradd -m -s /bin/bash -c "Alice Smith" alice    # -m creates the home directory
sudo passwd alice
sudo useradd -m -G devs,docker -u 1500 bob             # supplementary groups, fixed UID
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/app app   # system account for a service
```

On Debian/Ubuntu `adduser` is an interactive wrapper with sensible defaults; on RedHat
`adduser` is just a link to `useradd`. In scripts and in the exam, use `useradd` — it
behaves the same everywhere. Note that without `-m` and `-s`, Ubuntu's `useradd` creates
no home directory and gives the user `/bin/sh`.

Defaults come from `/etc/default/useradd` and `/etc/login.defs`; the new home directory is
a copy of **`/etc/skel`**, so anything placed there is given to every future user.

```bash
sudo usermod -aG devs alice        # ADD to a supplementary group — -a is essential
sudo usermod -s /bin/zsh alice     # change shell
sudo usermod -l alice2 alice       # rename login
sudo usermod -d /data/alice -m alice   # move home directory
sudo usermod -L alice              # lock (prefixes the hash with !)
sudo usermod -U alice              # unlock
sudo usermod -e 2026-12-31 alice   # account expires on a date
sudo userdel -r alice              # delete, with home directory and mail spool
```

**`usermod -G` without `-a` replaces** every supplementary group with the list you give.
`usermod -G devs alice` removes alice from `sudo` — on a machine where that was her only
way to administer it, you have just locked her out.

### Locking properly

`passwd -l` / `usermod -L` disable *password* login only. A user with an SSH key can still
log in. To disable an account completely:

```bash
sudo usermod -L -e 1 alice                # lock password AND expire the account
sudo usermod -s /usr/sbin/nologin alice   # optionally also remove the shell
```

An expired account is refused by PAM's account check for every authentication method,
keys included.

### Groups

```bash
sudo groupadd devs
sudo groupadd -g 2000 -r monitoring    # system group with a fixed GID
sudo groupmod -n developers devs       # rename
sudo gpasswd -a bob devs               # add a member
sudo gpasswd -d bob devs               # remove a member
sudo groupdel devs
```

---

## 4. Password ageing

```bash
sudo chage -l alice                    # show policy
sudo chage -M 90 -m 1 -W 7 alice       # max 90 days, min 1, warn 7 days before
sudo chage -d 0 alice                  # force a password change at next login
sudo chage -E 2026-12-31 alice         # account expiry
sudo chage -I 14 alice                 # 14 inactive days allowed after expiry
```

System-wide defaults for **new** accounts live in `/etc/login.defs`:

```text
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_WARN_AGE   7
```

Changing `login.defs` does not affect existing users — apply `chage` to them.

---

## 5. sudo

`sudo` runs a command as another user (root by default) if a rule allows it, and logs it.

```bash
sudo -l                  # what am I allowed to run?
sudo -u postgres psql    # as a specific user
sudo -i                  # root login shell
sudo -e /etc/hosts       # edit a file safely as root, with your own editor
```

Rules live in `/etc/sudoers` and, preferably, in files under **`/etc/sudoers.d/`**.
Always edit with **`visudo`** — it refuses to save a file with a syntax error, and a broken
sudoers file can leave nobody able to use sudo.

```bash
sudo visudo -f /etc/sudoers.d/devs
sudo visudo -c           # check every sudoers file
```

Rule syntax: `who  where = (as_whom)  what`

```text
alice   ALL=(ALL:ALL) ALL                                   # full root, password required
%devs   ALL=(root) /usr/bin/systemctl restart nginx         # one exact command
%ops    ALL=(ALL) NOPASSWD: /usr/bin/journalctl, /usr/bin/systemctl status *
bob     ALL=(postgres) NOPASSWD: /usr/bin/psql              # only as postgres
Defaults:alice  timestamp_timeout=5
```

`%` means a group. On Ubuntu members of `sudo` get full rights; on RedHat, `wheel`.

Rules to live by:

- **Always full paths** in command lists — otherwise a user could run their own script
  named `systemctl`.
- **Never grant editors, pagers or shells** in a narrow rule: `vim`, `less` and `man` can
  all spawn a shell, so `alice ALL=(root) /usr/bin/vim /etc/hosts` is full root. Use
  `sudoedit` (`sudo -e`) instead, which edits a copy as the user and installs it as root.
- Files in `/etc/sudoers.d/` are **ignored if their name contains a `.` or ends in `~`** —
  `devs.conf` is silently skipped. Name it `devs`.
- File mode must be `0440`; `visudo` sets this for you.

### su versus sudo

| | `su - alice` | `sudo -iu alice` |
|---|---|---|
| Password asked for | **alice's** | **yours** |
| Logged per command | no | yes |
| Granular rules | no | yes |

`su -` (with the dash) gives a full login environment; `su` without it keeps your
environment, which causes confusing PATH problems. Always use the dash.

---

## 6. Environment profiles

Which files run depends on the **kind of shell**:

| Shell kind | Example | Reads |
|---|---|---|
| **Login** | SSH session, `su -`, `sudo -i`, console login | `/etc/profile` → `/etc/profile.d/*.sh` → first of `~/.bash_profile`, `~/.bash_login`, `~/.profile` |
| **Interactive non-login** | a new terminal tab, typing `bash` | `/etc/bash.bashrc` (Debian) or `/etc/bashrc` (RedHat) → `~/.bashrc` |
| **Non-interactive** | a script, `ssh host cmd`, cron | nothing — unless `$BASH_ENV` is set |

Ubuntu's default `~/.profile` sources `~/.bashrc`, so in practice login shells get both.

Where to put what:

| Need | File |
|---|---|
| Variable for **all users**, every session, set by PAM (not a script!) | `/etc/environment` — `KEY=value` lines only; no `export`, no `$VAR` expansion |
| Script snippet for all users' login shells | `/etc/profile.d/company.sh` |
| Alias or prompt for all users' interactive shells | `/etc/bash.bashrc` (Debian) / `/etc/bashrc` (RedHat) |
| One user's variables | `~/.profile` or `~/.bash_profile` |
| One user's aliases and functions | `~/.bashrc` |
| Defaults for **future** users | `/etc/skel/` |

```bash
echo 'export HISTTIMEFORMAT="%F %T "' | sudo tee /etc/profile.d/history.sh
echo 'JAVA_HOME=/usr/lib/jvm/default' | sudo tee -a /etc/environment
```

Test the result in a **new login shell** — your current one already read the old files:

```bash
sudo -iu alice env | grep JAVA_HOME
bash -lc 'echo "$HISTTIMEFORMAT"'      # -l: behave as a login shell
```

Prefer a new file in `/etc/profile.d/` over editing `/etc/profile`: a package upgrade may
replace `/etc/profile`, but it never touches your drop-in.

---

## 7. Resource limits

Per-user limits are applied by **PAM's `pam_limits` module** at session start, from
`/etc/security/limits.conf` and `/etc/security/limits.d/*.conf`:

```text
#<domain>   <type>   <item>      <value>
alice        soft     nofile      4096
alice        hard     nofile      8192
@devs        hard     nproc       200
*            hard     core        0
@students    -        maxlogins   2
```

| Field | Values |
|---|---|
| domain | a user, `@group`, or `*` for everyone |
| type | `soft` (effective; the user may raise it up to hard), `hard` (ceiling), `-` (both) |
| item | `nofile` (open files), `nproc` (processes), `core`, `as` (address space), `cpu` (minutes), `maxlogins`, `fsize`, `stack` |

Prefer a file in `/etc/security/limits.d/` to editing `limits.conf`.

```bash
ulimit -a            # current shell's soft limits
ulimit -Hn           # hard limit on open files
```

Limits apply to **new sessions** — and only for services whose PAM configuration loads
`pam_limits`. Check which do on your machine:

```bash
grep -l pam_limits /etc/pam.d/*
```

Two consequences people trip over:

- **systemd services never pass through PAM**, so `limits.conf` does nothing for them.
  Use `LimitNOFILE=` and friends in the unit (chapter 06).
- Whether `su` applies limits depends on the distribution's `/etc/pam.d/su` — on Debian
  and Ubuntu the `pam_limits` line there is commented out by default. Test with an SSH
  login, or a service that loads the module, when in doubt.

---

## 8. Users from an LDAP directory

In an organisation, accounts live in a central directory (LDAP, or Active Directory which
speaks LDAP) so one account works on every server. A Linux client needs two things:

1. **Identity** — `getent passwd ldapuser` must return the user (NSS).
2. **Authentication** — logging in must check the password against the directory (PAM).

The standard tool for both is **SSSD**, which also caches credentials so logins still work
if the directory is briefly unreachable.

```text
login / ssh / id ─▶ PAM / NSS ─▶ sssd ─▶ LDAP server
                                  └─▶ local cache
```

### Configure an Ubuntu client

```bash
sudo apt install sssd-ldap libnss-sss libpam-sss ldap-utils
```

Check the directory answers *before* involving SSSD — this separates network and directory
problems from client configuration problems:

```bash
ldapsearch -x -H ldap://ubuntu2 -b dc=lab,dc=local '(uid=ldapuser1)'
```

`/etc/sssd/sssd.conf`:

```ini
[sssd]
services = nss, pam
domains = lab

[domain/lab]
id_provider   = ldap
auth_provider = ldap
ldap_uri         = ldap://ubuntu2
ldap_search_base = dc=lab,dc=local
cache_credentials = true
# Lab only: allow password checks over unencrypted LDAP.
# In production use ldaps:// or StartTLS with ldap_tls_cacert instead.
ldap_auth_disable_tls_never_use_in_production = true
```

```bash
sudo chmod 600 /etc/sssd/sssd.conf      # sssd refuses to start otherwise
sudo systemctl enable --now sssd
sudo pam-auth-update --enable mkhomedir # create home directories on first login
grep -E '^(passwd|group|shadow):' /etc/nsswitch.conf   # should include "sss"
getent passwd ldapuser1
id ldapuser1
su - ldapuser1
```

On RedHat-family systems the last steps are done by `authselect`:
`sudo authselect select sssd with-mkhomedir --force`.

### Troubleshooting order

| Symptom | Check |
|---|---|
| `getent` returns nothing | Does `ldapsearch` work? Is sssd running (`journalctl -u sssd`)? Is `sssd.conf` mode 600? |
| User found, login fails | `auth_provider`, TLS settings, `/var/log/auth.log` (Debian) or `/var/log/secure` (RedHat) |
| Stale data after a directory change | `sudo sss_cache -E` clears the cache |
| No home directory | `mkhomedir` not enabled |

---

# Part 2 — Permissions

Most "permission denied" errors are solved in seconds by someone who knows which of
five mechanisms is refusing. The rest of this chapter teaches all five and how to tell
them apart.

## 9. The basic model

Every file has an **owner**, a **group**, and three sets of permissions: for the owner,
for members of the group, and for everyone else. The kernel checks them in order and
stops at the **first category that matches you** — it does not combine them.

```text
-rwxr-x---  1 alice devs  4096 Oct  3 10:00 deploy.sh
 └┬┘└┬┘└┬┘
  │  │  └ others: nothing
  │  └ group devs: read and execute
  └ owner alice: read, write, execute
```

That "first match" rule has a surprising consequence:

```bash
chmod 007 file    # ------rwx: owner and group have NOTHING, others have everything
```

The owner of that file cannot read it, even though "others" can — the owner matched
the owner category first and got no permissions. (The owner can still `chmod` it back,
because changing the mode depends on ownership, not on the mode.)

### Octal

| Bit | Value | On a file | On a directory |
|---|---|---|---|
| `r` | 4 | read contents | list the names inside |
| `w` | 2 | modify contents | create, delete, rename entries |
| `x` | 1 | execute | enter it, and reach anything inside |

```text
rwx = 4+2+1 = 7      rw- = 6      r-x = 5      r-- = 4      --- = 0
755 = rwxr-xr-x      644 = rw-r--r--      600 = rw-------      750 = rwxr-x---
```

```bash
chmod 640 secret.conf          # octal: exact result
chmod u+x,g-w,o= script.sh     # symbolic: relative changes
chmod -R g+rX shared/          # capital X: execute only on dirs (and already-executable files)
```

`g+rX` is the one to remember. `chmod -R 755` on a tree makes every file executable,
which is wrong. `-R a+rX` gives directories the `x` they need and leaves plain files
alone.

---

## 10. Directories are different

A directory is a file containing a list of names. Its permissions control that list,
not the files themselves.

| You have on the directory | You can |
|---|---|
| `r` only | list names, but not see sizes or open anything |
| `x` only | open a file *if you already know its name* |
| `r` + `x` | list and open — normal read access |
| `w` + `x` | create, delete and rename files inside |

Two consequences that surprise people:

**Deleting a file depends on the directory, not the file.** A read-only file owned by
root can be deleted by anyone with `w` on its directory.

```bash
sudo touch /tmp/demo/rootfile && sudo chmod 444 /tmp/demo/rootfile
rm /tmp/demo/rootfile     # succeeds if you have w+x on /tmp/demo
```

**Every directory on the path needs `x`.** To read `/srv/app/conf/db.yml` you need `x`
on `/srv`, `/srv/app` and `/srv/app/conf`, plus `r` on the file. One missing `x`
anywhere up the tree and the file is unreachable however open its own mode is.

---

## 11. umask

New files do not start at their final mode. The program asks for a base mode — `666`
for files, `777` for directories — and the **umask** removes bits from it.

```text
base file       666   rw-rw-rw-
umask           022   ----w--w-
result          644   rw-r--r--

base dir        777   rwxrwxrwx
umask           022   ----w--w-
result          755   rwxr-xr-x
```

```bash
umask           # show it: 0022
umask 027       # new files 640, new dirs 750
umask -S        # symbolic: u=rwx,g=rx,o=
```

It is a mask, not a subtraction — bits are cleared, never borrowed. And because the
file base is `666`, **no umask ever makes a new file executable.** That is why you
`chmod +x` scripts after writing them.

Where it is set: per shell in `~/.bashrc`, system-wide in `/etc/login.defs` (`UMASK`)
or `/etc/profile`, and for services in their systemd unit (`UMask=0027`).

---

## 12. The special bits

A fourth octal digit sits in front: `chmod 4755` means SUID plus `755`.

| Bit | Octal | On a file | On a directory |
|---|---|---|---|
| **SUID** | 4000 | runs as the file's *owner*, not as you | no effect on Linux |
| **SGID** | 2000 | runs as the file's *group* | new files inherit the directory's group |
| **Sticky** | 1000 | no effect on Linux | only a file's owner may delete it |

### SUID

```bash
ls -l /usr/bin/passwd
-rwsr-xr-x. 1 root root 32648 … /usr/bin/passwd
```

The `s` in the owner's execute slot means SUID. You can change your password, which
means writing `/etc/shadow`, which only root may write — so `passwd` runs as root for
you. Every SUID-root binary is a program that hands root to whoever runs it, and a bug
in one is a privilege escalation. Audit them:

```bash
find / -xdev -perm -4000 -type f 2>/dev/null
```

Linux ignores SUID on shell scripts. The race between the kernel opening the script
and the interpreter reading it made SUID scripts unsafe, so the bit is simply not
honoured. Use `sudo` rules instead (chapter 04).

### SGID on a directory — shared folders

This is the one you will use most:

```bash
sudo mkdir /srv/project
sudo chgrp devs /srv/project
sudo chmod 2775 /srv/project
```

Without SGID, a file alice creates belongs to alice's primary group, so bob in `devs`
cannot edit it. With SGID, every new file inside gets group `devs` automatically.
Combine with `umask 002` and the whole team can edit everything.

### Sticky bit

```bash
ls -ld /tmp
drwxrwxrwt. 20 root root 4096 … /tmp
```

`/tmp` is writable by everyone, so without sticky anyone could delete anyone's files.
The `t` restricts deletion to the file's owner, the directory's owner, and root.

### Capital letters mean something is wrong

| Shows | Means |
|---|---|
| `s` | SUID/SGID set, and execute set |
| `S` | SUID/SGID set, but execute **not** set — almost always a mistake |
| `t` | sticky set, and others-execute set |
| `T` | sticky set, others-execute not set |

---

## 13. Ownership

```bash
chown alice file              # owner
chown alice:devs file         # owner and group
chown :devs file              # group only (same as chgrp devs file)
chown -R alice:devs dir/      # recursively
chown --reference=a b         # copy ownership from a to b
```

Only root can give a file away. A normal user can change a file's group only to a group
they belong to. That stops users from escaping disk quotas by giving files to others.

---

## 14. ACLs — when owner/group/other is not enough

You need bob to read one file, but bob is not in the file's group and you do not want to
change the group. Access Control Lists add extra entries:

```bash
setfacl -m u:bob:r report.txt          # give bob read
setfacl -m g:auditors:rx /srv/logs     # give a group read+execute
getfacl report.txt
```

```text
# file: report.txt
# owner: alice
# group: devs
user::rw-
user:bob:r--
group::r--
mask::r--
other::---
```

`ls -l` shows a `+` when ACLs exist: `-rw-r-----+`. Without that `+` you would never
guess bob has access.

### The mask

The `mask` is the maximum any named user, named group or the owning group can get. If
the mask is `r--` and you grant bob `rw-`, bob's **effective** permission is `r--`.
`getfacl` shows `#effective:r--` when they differ.

The trap: **`chmod` on a file with ACLs changes the mask**, not the group bits. A
careless `chmod 640` can silently cut every ACL user down.

### Default ACLs — inherited by new files

```bash
setfacl -d -m g:devs:rwX /srv/project    # new files inherit this entry
setfacl -R -m g:devs:rwX /srv/project    # apply to what already exists
```

You need both: `-d` affects only the future, `-R` affects only the present.

```bash
setfacl -x u:bob report.txt     # remove one entry
setfacl -b report.txt           # remove all ACLs
```

---

## 15. File attributes — beyond root

Attributes live in the filesystem itself and are checked *before* permissions — even
root obeys them.

```bash
sudo chattr +i /etc/resolv.conf    # immutable: no write, delete, rename, link
lsattr /etc/resolv.conf            # ----i----------- /etc/resolv.conf
sudo rm /etc/resolv.conf           # rm: cannot remove: Operation not permitted
sudo chattr -i /etc/resolv.conf    # undo
```

| Attribute | Effect |
|---|---|
| `i` | immutable — nobody can change it, root included |
| `a` | append-only — can be added to, never rewritten or truncated |

`+a` is how you keep a log file that an attacker who gains root cannot quietly edit —
they would need to remove the attribute first, which itself is auditable.

When root gets `Operation not permitted` (not `Permission denied`), check `lsattr`
first. That different wording is the clue.

---

## 16. Capabilities — root, divided up

Instead of all-or-nothing root, Linux splits privilege into capabilities. A program can
be given just the one it needs:

```bash
sudo setcap cap_net_bind_service=+ep ./myserver   # bind port 80 without root
getcap ./myserver                      # ./myserver cap_net_bind_service=ep
getcap -r / 2>/dev/null                # audit the whole system
sudo setcap -r ./myserver              # remove
```

`ping` is the classic example of why this exists. It used to be SUID root just to open
a raw socket. Depending on the distribution it now either holds only `cap_net_raw`, or
needs no privilege at all because the kernel allows unprivileged ICMP sockets for the
groups in `net.ipv4.ping_group_range`. Run `ls -l` and `getcap` on your own lab's
`ping` to see which approach it takes.

---

## 17. Debugging "permission denied"

Work through these in order. Each is a different mechanism:

| # | Check | Command |
|---|---|---|
| 1 | Every directory on the path has `x` for you | `namei -l /full/path/to/file` |
| 2 | The file mode for *your* matching category | `ls -l`, `id` |
| 3 | ACLs, and their mask | `getfacl file` |
| 4 | Immutable or append-only attributes | `lsattr file` |
| 5 | SELinux / AppArmor (chapter 13) | `ls -Z file`, `ausearch -m avc -ts recent` |
| 6 | Mount options — `ro`, `noexec`, `nosuid` | `findmnt -T /path` |

`namei -l` is the tool people don't know and should:

```text
$ namei -l /srv/app/conf/db.yml
f: /srv/app/conf/db.yml
dr-xr-xr-x root root  /
drwxr-xr-x root root  srv
drwx------ app  app   app        ← no x for anyone but app: stops here
drwxr-xr-x app  app   conf
-rw-r--r-- app  app   db.yml
```

The file itself is world-readable. The parent directory is the problem.

A `noexec` mount produces "Permission denied" when you run a script that is clearly
`+x`. `/tmp` is often mounted `noexec` on hardened systems.

---

## 18. Gotchas

**Group membership needs a new login.** `usermod -aG devs alice` does not affect
alice's existing sessions. `id` in her current shell will not show `devs` until she
logs in again, or runs `newgrp devs`.

**`chmod -R 777` is never the answer.** It fixes the symptom and removes every
protection, including the sticky bit's purpose. If you find yourself typing it, go back
to section 9.

**Copying changes ownership; moving does not.** `cp` creates a new file owned by you
with your umask; `mv` within a filesystem renames, keeping everything. `cp -a` preserves
mode, ownership and timestamps when you have the rights to.

**ACLs vanish on some copies.** Not every tool preserves them. `cp -a`, `rsync -A`, and
`tar --acls` do; plain `cp` and many archive formats do not.

---

## 19. Commands introduced

| Command | Purpose |
|---|---|
| `chmod` | change mode (octal or symbolic) |
| `chown`, `chgrp` | change owner, group |
| `umask` | set default-permission mask |
| `setfacl`, `getfacl` | manage ACLs |
| `chattr`, `lsattr` | manage filesystem attributes |
| `getcap`, `setcap` | manage capabilities |
| `namei -l` | show permissions along a whole path |
| `stat` | every metadata field, including octal mode |
| `find -perm` | find files by permission bits |
| `newgrp` | start a shell with a new primary group |

### Users and groups commands

| Command | Purpose |
|---|---|
| `useradd`, `usermod`, `userdel` | manage users |
| `groupadd`, `groupmod`, `groupdel`, `gpasswd` | manage groups |
| `passwd`, `chage` | passwords and ageing |
| `id`, `getent`, `groups` | look up identities through NSS |
| `vipw`, `vigr` | safely edit the account files |
| `visudo`, `sudo -l`, `sudoedit` | sudo rules |
| `ulimit` | per-shell limits |
| `ldapsearch`, `sss_cache`, `pam-auth-update`, `authselect` | LDAP clients |

---

➡ **Next:** [quiz](../quizzes/04-users-permissions.md) → [lab](../labs/04-users-permissions.md)
