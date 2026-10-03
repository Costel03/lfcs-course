# Solutions 04 — Users, groups and permissions

---

# Part A — Users and groups

## 🟢 Warm-up

**4.1**

```bash
id
getent passwd vagrant
getent group sudo
```

**4.2**

```bash
sudo chage -l vagrant
sudo getent shadow vagrant | cut -d: -f2 | cut -c1-4
```

`$y$` is yescrypt (Ubuntu 22.04 and later, Debian 11 and later), `$6$` is SHA-512 (RHEL 9
and older Ubuntu), `$5$` SHA-256, `$1$` MD5 — the last should never appear on a modern
system. `getent shadow` works because NSS covers the shadow database too.

**4.3**

```bash
sudo groupadd -g 3000 devs
sudo groupadd -r auditors
```

## 🔵 Practical

**4.4**

```bash
sudo useradd -m -s /bin/bash -c "Alice Smith" -G devs alice
sudo useradd -m -s /bin/bash -u 1600 -G devs bob
sudo useradd -m -s /bin/bash carol
for u in alice bob carol; do printf '%s:%s\n' "$u" 'Lab12345!' | sudo chpasswd; done
```

The password is in single quotes for a reason: inside double quotes an interactive bash
may treat `!` as history expansion. `chpasswd` reads `user:password` lines, which is how
you set passwords non-interactively.

**4.5**

```bash
sudo useradd -r -m -d /var/lib/appsvc -s /usr/sbin/nologin appsvc
```

`-r` takes the UID from the system range and skips ageing; `-m` is still needed for the
home directory. Without a password the hash field is `!`, so nobody can log in as it —
admins switch to it with `sudo -u appsvc …`.

**4.6**

```bash
sudo chage -M 60 -W 10 alice
sudo chage -d 0 alice          # or: sudo passwd -e alice
```

**4.7**

```bash
sudo chage -E 2026-12-31 carol
```

**4.8**

```bash
sudo usermod -aG auditors bob
```

**4.9**

```bash
sudo visudo -f /etc/sudoers.d/devs
```

```text
%devs ALL=(root) NOPASSWD: /usr/bin/systemctl restart ssh, /usr/bin/systemctl status ssh
```

```bash
sudo -iu alice
sudo -l
sudo -n systemctl status ssh      # works
sudo -n cat /etc/shadow           # "a password is required" — refused
```

The service is `ssh` on Ubuntu and `sshd` on RedHat. `-n` makes sudo fail rather than
prompt, which is the honest test of a NOPASSWD rule.

**4.10**

```bash
sudo tee /etc/profile.d/lab04.sh > /dev/null <<'EOF'
export EDITOR=vim
export HISTTIMEFORMAT='%F %T '
EOF
sudo -iu carol bash -c 'echo $EDITOR; echo $HISTTIMEFORMAT'
```

`sudo -i` starts a login shell, which reads `/etc/profile`, which sources every `*.sh` in
`/etc/profile.d/`. A plain `sudo -u carol bash -c …` would not — it is non-interactive and
non-login — which is a good way to see the difference for yourself.

**4.11**

```bash
echo 'Welcome to ubuntu1' | sudo tee /etc/skel/README.txt
sudo useradd -m -s /bin/bash dave
sudo cat /home/dave/README.txt
```

`/etc/skel` is copied only when a home directory is **created**, so existing users never
receive later additions.

**4.12**

```bash
sudo tee /etc/security/limits.d/lab04.conf > /dev/null <<'EOF'
@devs   hard   nproc    100
alice   soft   nofile   2048
alice   hard   nofile   4096
EOF

grep -lE '^\s*session.*pam_limits' /etc/pam.d/*
```

Note the regex: a plain `grep -l pam_limits` also matches *commented-out* lines, such as
the one in Debian's `/etc/pam.d/su`, and would mislead you.

Then test through a service the grep listed. Typically `sshd` loads it; on Ubuntu `sudo`
does too:

```bash
sudo -iu alice bash -c 'ulimit -Sn; ulimit -Hn'     # 2048 / 4096
ssh alice@localhost 'ulimit -Sn; ulimit -Hn'        # same, via sshd's PAM stack
```

## 🔴 Challenge

**4.13**

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/dave_key
sudo install -d -m 700 -o dave -g dave /home/dave/.ssh
sudo install -m 600 -o dave -g dave ~/.ssh/dave_key.pub /home/dave/.ssh/authorized_keys
ssh -i ~/.ssh/dave_key -o StrictHostKeyChecking=accept-new dave@localhost true && echo key-ok

sudo usermod -L dave
ssh -i ~/.ssh/dave_key dave@localhost true && echo 'still works after -L'

sudo usermod -e 1 dave            # expire the account (1 = 2 Jan 1970)
ssh -i ~/.ssh/dave_key dave@localhost true || echo 'refused'
```

`-L` only invalidates the password hash. With `UsePAM yes` (the default), sshd doesn't
look at the hash for key logins, so the key keeps working. An *expired account* is
rejected in PAM's **account** phase, which runs for every login method. In an incident,
"I locked the account" without also expiring it is a hole.

Undo for the next tasks: `sudo usermod -U -e '' dave`.

The `install` command is worth adopting: it creates the directory or file with owner,
group and mode in one step, so there's no window where `authorized_keys` has the wrong
permissions — and sshd ignores an `authorized_keys` that is group- or world-writable.

**4.14**

```bash
sudo useradd -m -s /bin/bash olduser
sudo -u olduser touch /home/olduser/keepme
sudo usermod -l newname -d /home/newname -m olduser
sudo groupmod -n newname olduser
id newname; ls /home/newname
```

`usermod -l` renames only the login. The primary group still has the old name until you
rename it too — easy to miss, and visible in `ls -l` forever after. The user must not be
logged in while you do this.

**4.15**

As carol:

```bash
sudo less /var/log/syslog
# inside less, type:   !sh
# you are now root
```

`less` can run shell commands with `!`. Because less is running as root, so is the shell.
The same applies to `vim`, `man`, `more`, `find -exec`, `awk`, `git` and many others —
the GTFOBins project catalogues them.

Safe replacements, best first:

```bash
sudo rm /etc/sudoers.d/carol
sudo usermod -aG adm carol        # on Ubuntu, /var/log/syslog is root:adm 640
```

No sudo at all: carol reads the file with her own `less` as herself. If it must be sudo,
the `NOEXEC:` tag stops the program executing anything else:

```text
carol ALL=(root) NOEXEC: /usr/bin/less /var/log/syslog
```

(If your box has no `/var/log/syslog`, rsyslog isn't installed — use
`/var/log/auth.log` or `sudo apt install rsyslog`.)

**4.16**

On ubuntu1:

```bash
sudo apt install -y sssd-ldap libnss-sss libpam-sss ldap-utils
ldapsearch -x -H ldap://ubuntu2 -b dc=lab,dc=local '(uid=ldapuser1)'   # the directory answers?

sudo tee /etc/sssd/sssd.conf > /dev/null <<'EOF'
[sssd]
services = nss, pam
domains = lab

[domain/lab]
id_provider   = ldap
auth_provider = ldap
ldap_uri         = ldap://ubuntu2
ldap_search_base = dc=lab,dc=local
cache_credentials = true
ldap_auth_disable_tls_never_use_in_production = true
EOF
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl enable --now sssd
sudo systemctl restart sssd
sudo pam-auth-update --enable mkhomedir

getent passwd ldapuser1
id ldapuser1
su - ldapuser1 -c pwd           # asks for Ldap123!
grep ldapuser1 /etc/passwd || echo 'not a local user'
```

If `getent` is empty, work in order: does `ldapsearch` work (network and directory)? Is
sssd running — `systemctl status sssd`, `journalctl -u sssd`? A mode other than `600` on
`sssd.conf` makes it refuse to start. Does `/etc/nsswitch.conf` list `sss` for `passwd`
and `group`? The `libnss-sss` package adds it; `authselect` does on RedHat.

Without `ldap_auth_disable_tls_never_use_in_production`, `getent` works but **logins fail**
— SSSD refuses to send passwords over an unencrypted connection. In production, give the
directory a certificate and use `ldaps://` with `ldap_tls_cacert`.

---

# Part B — Permissions

## 🟢 Warm-up

**4.17**

```bash
touch notes.txt
chmod 640 notes.txt
chmod g-r notes.txt          # or: chmod u=rw,go= notes.txt
```

**4.18**

```bash
umask; umask -S
```

**4.19**

```bash
find / -xdev -perm -4000 -type f 2>/dev/null
find / -xdev -perm -4000 -type f 2>/dev/null | wc -l
```

`-perm -4000` means "at least these bits set". `-xdev` stops `find` from wandering into
`/proc` and other mounts. Expect a dozen or two on a minimal install.

**4.20**

```bash
stat -c '%a %U %G' /etc/shadow
```

On RHEL-family systems `/etc/shadow` is mode `0` — nobody, not even its owner, has
permission bits. Root reads it anyway, because root bypasses ordinary mode checks. On
Debian/Ubuntu it is `640 root:shadow`, so tools in the `shadow` group can read it.

**4.21**

```bash
id alice
groups alice
```

**4.22**

```bash
namei -l /etc/ssh/sshd_config
```

---

## 🔵 Practical

**4.23**

```bash
echo 'echo hello' > hello.sh
./hello.sh           # Permission denied
chmod u+x hello.sh
```

New files are never executable — the base mode is `666` before the umask. Without a
shebang the script still runs because your interactive bash falls back to running it
itself; add `#!/usr/bin/env bash` anyway, since other callers will not.

**4.24**

```bash
umask 027
touch newfile; mkdir newdir
stat -c '%a %n' newfile newdir     # 640 newfile / 750 newdir
```

`666` with `027` cleared → `640`. `777` with `027` cleared → `750`.

**4.25**

```bash
sudo mkdir /srv/project
sudo chgrp devs /srv/project
sudo chmod 2770 /srv/project
```

The leading `2` is SGID. Files created inside inherit the directory's group instead of
the creator's primary group.

**4.26**

alice's umask is `022`, so her file is `rw-r--r--` — group can read, not write. Two fixes,
and only one satisfies "without changing anything after each file is created" for
*every* user:

```bash
# Robust: a default ACL — applies whatever umask the creator has
sudo setfacl -d -m g:devs:rwX /srv/project
sudo setfacl -R -m g:devs:rwX /srv/project      # fix existing files too
```

The alternative — setting `umask 002` for each team member — works only for people whose
shell or service you control, and breaks the moment someone uses a tool that sets its
own umask. When a directory has a default ACL, the creator's umask is not applied to it.

**4.27**

```bash
sudo chmod +t /srv/project
ls -ld /srv/project       # drwxrws--T+ …
```

Capital `T` because others have no execute — that is expected here, not an error.

**4.28**

```bash
sudo setfacl -m u:carol:r /srv/project/plan.txt
```

But carol still cannot reach it — she needs `x` on `/srv/project`, which is `2770`:

```bash
sudo setfacl -m u:carol:x /srv/project
```

This is the lesson of the task. Granting access to a file is useless without access to
every directory above it. `x` alone on the directory lets carol open a file whose name
she knows, without being able to list the others.

**4.29** The `+` after the mode: `-rw-rw----+`. It means "there are entries not shown
here — run `getfacl`".

**4.30**

```bash
sudo mkdir /srv/logs
sudo setfacl -R -m g:auditors:rX /srv/logs     # what exists now
sudo setfacl -d -m g:auditors:rX /srv/logs     # what will exist
```

`-R` covers the present, `-d` covers the future. Forgetting either is the usual mistake.

**4.31**

```bash
touch frozen.txt
sudo chattr +i frozen.txt
sudo rm frozen.txt          # Operation not permitted
echo x | sudo tee frozen.txt    # also fails
sudo chattr -i frozen.txt && rm frozen.txt
```

**4.32**

```bash
touch audit.log
sudo chattr +a audit.log
echo a >> audit.log        # works
echo b > audit.log         # Operation not permitted — even under sudo
```

**4.33**

```bash
find /etc /usr -xdev -type f -perm -0002 2>/dev/null
find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null
```

`! -perm -1000` excludes directories with sticky set. A world-writable directory
*without* sticky lets any user delete or replace any file in it — a real finding if one
turns up.

**4.34**

```bash
chmod -R a+rX tree/
```

Capital `X` adds execute only to directories, and to files that already have execute for
someone. Lowercase `x` would make every text file executable.

---

## 🔴 Challenge

**4.35**

```bash
namei -l /srv/app/conf/db.yml
```

shows `drwx------ root root app` — nobody but root can traverse `/srv/app`. Smallest fix:

```bash
sudo setfacl -m u:alice:x /srv/app
```

`x` without `r` lets alice pass through to a path she already knows, but not list the
directory. carol is unaffected. Making `/srv/app` `711` would have worked too but would
have given traversal to everyone, which the task ruled out.

**4.36**

```bash
setfacl -m u:bob:rw file
chmod 640 file
getfacl file
```

```text
user:bob:rw-     #effective:r--
mask::r--
```

On a file with ACLs, the group bits of `chmod` *are* the mask. `640` set the mask to
`r--`, capping bob. Restore:

```bash
setfacl -m m::rw file      # set the mask directly
# or
chmod g+w file             # same effect, through chmod
```

Watch for this after anyone runs a "fix permissions" script over a directory with ACLs.

**4.37**

```bash
# admin terminal
sudo usermod -aG devs carol
# carol's existing terminal
id                 # devs is missing
newgrp devs        # new shell with devs as primary group
id                 # devs present
touch /srv/project/carol.txt
```

Group membership is read once, at login, into the process's credentials. Changing
`/etc/group` does not alter running processes. `newgrp` starts a fresh shell with fresh
credentials; logging out and in again does the same thing more completely.

Note `-a`: `usermod -G devs carol` without it *replaces* all of carol's supplementary
groups with just `devs`.

**4.38**

```bash
printf '#!/bin/bash\necho ran\n' | sudo tee /mnt/noexec/script.sh >/dev/null
sudo chmod +x /mnt/noexec/script.sh
/mnt/noexec/script.sh          # Permission denied
bash /mnt/noexec/script.sh     # ran
sudo umount /mnt/noexec
```

`noexec` stops the kernel from `execve()`-ing files on that mount. `bash script.sh`
never execs the script — it execs `/bin/bash`, which lives elsewhere, and bash then
*reads* the script as data. So `noexec` blocks accidental or careless execution, not a
determined user. It is still worth setting on `/tmp` and removable media: it stops a lot
of drive-by malware that assumes it can drop and run a binary.

**4.39**

```bash
printf '#!/bin/bash\nid -u\n' | sudo tee /usr/local/bin/whoami.sh >/dev/null
sudo chmod 4755 /usr/local/bin/whoami.sh
sudo -iu alice /usr/local/bin/whoami.sh     # alice's UID
sudo rm /usr/local/bin/whoami.sh
```

Linux ignores SUID on interpreted scripts. Between the kernel checking the SUID bit and
the interpreter opening the script by name, an attacker could swap the file, so the bit
was made meaningless for scripts. When something genuinely needs to run as another user,
grant it with a narrow `sudo` rule (chapter 04).

**4.40**

```bash
python3 -m http.server 80        # PermissionError: [Errno 13]

cp "$(readlink -f "$(command -v python3)")" ~/lab04/py3
sudo setcap cap_net_bind_service=+ep ~/lab04/py3
getcap ~/lab04/py3
~/lab04/py3 -m http.server 80 &
curl -s localhost:80 | head -5
kill %1
```

Ports below 1024 traditionally need root. `cap_net_bind_service` is exactly that
privilege and nothing more. Setting it on a *copy* matters — setting it on the system
`python3` would let every Python script on the box bind privileged ports.

In production you would usually give the capability to the service through its systemd
unit (`AmbientCapabilities=CAP_NET_BIND_SERVICE`) rather than on a binary. Chapter 06
shows how.

**4.41**

```bash
{
  echo "== SUID/SGID =="
  find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null
  echo "== Capabilities =="
  getcap -r / 2>/dev/null
  echo "== World-writable files outside tmp =="
  find / -xdev -type f -perm -0002 ! -path '/tmp/*' ! -path '/var/tmp/*' 2>/dev/null
  echo "== Immutable files under /etc =="
  lsattr -R /etc 2>/dev/null | awk '$1 ~ /i/ {print $2}'
} > ~/lab04/audit-$(hostname)-$(date +%F).txt
```

The value of this task is that it is repeatable. Run it today, keep the output, run it
again next month and `diff` the two. A new SUID binary nobody can explain is exactly what
an intrusion looks like.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | `750` | 4+2+1, 4+0+1, 0 |
| 2 | `rw-r-----` | 6 = rw, 4 = r, 0 = nothing |
| 3 | **No** | The owner category matches first and grants nothing; categories never combine |
| 4 | **B, C** | `x` allows reaching known names; listing (A, D) needs `r` |
| 5 | file `640`, dir `750` | `666`/`777` with bits `027` cleared |
| 6 | **No** | The file base mode is `666`; a mask can only clear bits |
| 7 | `X` adds execute only to directories and already-executable files | `x` makes every file executable |
| 8 | SUID: none on Linux · SGID: new files inherit the group · Sticky: only owners can delete | |
| 9 | SUID is set but owner-execute is not — almost certainly a mistake | Lowercase `s` means both |
| 10 | As the normal user | Linux ignores SUID on interpreted scripts |
| 11 | Read only — `#effective:r--` | `chmod`'s group digit sets the ACL mask, which caps bob |
| 12 | The file has ACL entries beyond owner/group/other | Run `getfacl` |
| 13 | Immutable attribute — `lsattr /etc/important.conf` | *Operation not permitted* rather than *Permission denied* is the clue |
| 14 | `noexec` on the mount (`findmnt -T .`); SELinux/AppArmor denial (`ausearch -m avc`) | Also: a bad shebang interpreter path gives a different error |
| 15 | SGID on the directory **and** a default ACL `g:devs:rwX` (or `umask 002` for all members) | SGID fixes ownership; the ACL/umask fixes write access |
| 16 | `/etc/shadow`; only root (on Debian also the `shadow` group) | `/etc/passwd` is world-readable, so hashes moved out of it |
| 17 | Only `docker` (plus her primary group) | `-G` without `-a` replaces the supplementary list |
| 18 | A UID from the system range (below 1000) and no password ageing | For service accounts |
| 19 | **Yes**, with `UsePAM yes`; expire the account: `usermod -e 1` or `chage -E 0` | `-L` only touches the password hash |
| 20 | sudo ignores files in `sudoers.d` whose names contain a `.` | Name it `web` |
| 21 | `:!sh` inside vim gives a root shell | Use `sudoedit` / `sudo -e` |
| 22 | `/etc/environment` — `KEY=value` only, no `export`, no variable expansion | Read by `pam_env`, not by a shell |
| 23 | systemd services don't pass through PAM, so `limits.conf` doesn't apply | `LimitNOFILE=65535` in the unit (`systemctl edit nginx`) |
| 24 | Is sssd running with a `600` config? Does `/etc/nsswitch.conf` list `sss` for passwd? | `ldapsearch` working proves the directory is fine |

