# Lab 04 — Users, groups and permissions

41 tasks on **ubuntu1**; the LDAP tasks also use **ubuntu2**. You need `sudo`.
Do the quiz first.

To act as another user: `sudo -iu alice` (leave with `exit`).

**Cleanup at the very end:**

```bash
for u in alice bob carol dave newname; do sudo userdel -r "$u" 2>/dev/null; done
sudo userdel appsvc 2>/dev/null
sudo groupdel devs; sudo groupdel auditors
sudo rm -f /etc/sudoers.d/devs /etc/sudoers.d/carol /etc/profile.d/lab04.sh \
           /etc/security/limits.d/lab04.conf /etc/skel/README
sudo rm -rf /srv/project /srv/logs ~/lab04
```

---

# Part A — Users and groups

## 🟢 Warm-up

**4.1** Show your own UID, primary group and supplementary groups. Then show your
`passwd` entry and the members of the `sudo` group **through NSS**, not by reading the
files.
*Verify:* you used `id` and `getent`, not `grep`.

**4.2** Show the password-ageing policy of the `vagrant` user, and identify which hashing
algorithm its password uses.
*Verify:* you can name the algorithm from the prefix of the hash (`$y$`, `$6$`, …).

**4.3** Create a group `devs` with GID `3000`, and a **system** group `auditors`.
*Verify:* `getent group devs auditors` — `devs` has GID 3000, `auditors` has a GID below 1000.

## 🔵 Practical

**4.4** Create these users, each with a home directory, `/bin/bash` as shell, and the
password `Lab12345!`:

| User | Requirements |
|---|---|
| `alice` | comment "Alice Smith", supplementary group `devs` |
| `bob` | UID `1600`, supplementary group `devs` |
| `carol` | no supplementary groups |

*Verify:* `id alice bob carol`; `getent passwd alice` shows the comment; `bob` is UID 1600.

**4.5** Create a system account `appsvc` for a service: no login shell, home directory
`/var/lib/appsvc` (created), no password.
*Verify:* `getent passwd appsvc` shows a UID below 1000 and `/usr/sbin/nologin`;
`sudo -u appsvc -s` would be the only way in.

**4.6** Set alice's password policy: maximum 60 days, warning 10 days before, and she must
change her password at her next login.
*Verify:* `sudo chage -l alice` shows all three.

**4.7** Make carol's account expire at the end of 2026.
*Verify:* `sudo chage -l carol` shows `Account expires : Dec 31, 2026`.

**4.8** Add bob to `auditors` without removing him from `devs`.
*Verify:* `id bob` lists both groups.

**4.9** Allow members of `devs` to run exactly two commands as root, without a password:
`systemctl restart ssh` and `systemctl status ssh`. Nothing else.
*Verify:* as alice, `sudo -l` lists only those two; `sudo -n systemctl status ssh` works;
`sudo -n cat /etc/shadow` is refused.

**4.10** Make every user's **login** shell get `EDITOR=vim` and timestamps in `history`,
using a drop-in file rather than editing `/etc/profile`.
*Verify:* `sudo -iu carol bash -c 'echo $EDITOR; echo $HISTTIMEFORMAT'` prints both values.

**4.11** Make every **future** user get a file `~/README.txt` containing `Welcome to ubuntu1`.
Create `dave` to prove it.
*Verify:* `sudo cat /home/dave/README.txt`; carol (created earlier) does not have one.

**4.12** Set resource limits: members of `devs` may run at most 100 processes (hard);
alice may open 2048 files (soft) and at most 4096 (hard). Then find a PAM service on this
machine that applies `pam_limits` and use it to prove the limits are in effect.
*Verify:* alice's session reports soft `nofile` 2048 and hard 4096.

## 🔴 Challenge

**4.13 — Lock it properly.** Give `dave` an SSH key login from your vagrant account
(`ssh-keygen`, then install the public key into dave's `~/.ssh/authorized_keys` with the
right ownership and modes). Prove key login works. Lock dave with `usermod -L` only and
show the key **still works**. Then disable the account so that keys fail too.
*Verify:* `ssh -i <key> dave@localhost true` succeeds after `-L`, fails after your second step.

**4.14 — Rename a user.** Create `olduser` with a home directory and a file in it. Rename
the account to `newname`, keep the same UID, and move the home directory to
`/home/newname`.
*Verify:* `id newname` shows the old UID; `/home/newname/` contains the file;
`/home/olduser` no longer exists.

**4.15 — The sudo escape.** A colleague added this rule so carol can read the syslog:

```text
carol ALL=(root) /usr/bin/less /var/log/syslog
```

Install it in `/etc/sudoers.d/carol`, then demonstrate how carol gets a root shell from it.
Replace it with a rule that gives her the same ability safely.
*Verify:* with your new rule carol can read the log; the shell escape no longer works.

**4.16 — LDAP accounts.** Build a small directory on **ubuntu2**:

```bash
# on ubuntu2
sudo debconf-set-selections <<'EOF'
slapd slapd/password1 password admin
slapd slapd/password2 password admin
slapd slapd/domain string lab.local
slapd shared/organization string Lab
EOF
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y slapd ldap-utils

PW=$(slappasswd -s 'Ldap123!')
cat > /tmp/lab.ldif <<EOF
dn: ou=people,dc=lab,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=lab,dc=local
objectClass: organizationalUnit
ou: groups

dn: cn=ldapusers,ou=groups,dc=lab,dc=local
objectClass: posixGroup
cn: ldapusers
gidNumber: 5000
memberUid: ldapuser1

dn: uid=ldapuser1,ou=people,dc=lab,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ldapuser1
sn: One
cn: LDAP User One
uidNumber: 5001
gidNumber: 5000
homeDirectory: /home/ldapuser1
loginShell: /bin/bash
userPassword: $PW
EOF
ldapadd -x -D cn=admin,dc=lab,dc=local -w admin -f /tmp/lab.ldif
```

(If `ldapadd` reports *Invalid credentials*, the base DN didn't come out as
`dc=lab,dc=local` — run `sudo dpkg-reconfigure slapd` and answer the questions.)

Now configure **ubuntu1** so that `ldapuser1` exists, can log in with `Ldap123!`, and gets
a home directory created on first login. Do not create the user locally.
*Verify:* on ubuntu1, `getent passwd ldapuser1` shows UID 5001; `id ldapuser1` shows group
`ldapusers`; `su - ldapuser1 -c pwd` prints `/home/ldapuser1`; `grep ldapuser1 /etc/passwd`
finds nothing.

---

# Part B — Permissions

If you skipped Part A, create the users and groups Part B relies on:

```bash
sudo groupadd devs; sudo groupadd auditors
for u in alice bob carol; do sudo useradd -m -s /bin/bash "$u"; printf '%s:%s\n' "$u" 'Lab12345!' | sudo chpasswd; done
sudo usermod -aG devs alice; sudo usermod -aG devs bob
```

```bash
mkdir -p ~/lab04 && cd ~/lab04
```

## 🟢 Warm-up

**4.17** Create `notes.txt` and set its mode to `rw-r-----` using octal, then to
`rw-------` using symbolic notation.
*Verify:* `stat -c '%a' notes.txt` prints `600`.

**4.18** Print your current umask in both octal and symbolic form.
*Verify:* you see something like `0022` and `u=rwx,g=rx,o=rx`.

**4.19** Find every SUID file on the root filesystem. Count them.
*Verify:* the list includes `/usr/bin/passwd`.

**4.20** Show the octal mode, owner and group of `/etc/shadow` in one command.
*Verify:* on Ubuntu, `640 root shadow`. (On Rocky it would be `0 root root`.)

**4.21** Find out which groups `alice` belongs to without logging in as her.
*Verify:* the output includes `devs`.

**4.22** Display the full permission chain from `/` to `/etc/ssh/sshd_config`.
*Verify:* you see one line per path component.

---

## 🔵 Practical

**4.23** Create a script `hello.sh` containing `echo hello`. Run it as `./hello.sh` and
note the error. Fix it with the minimum permission change.
*Verify:* `./hello.sh` prints `hello`.

**4.24** Set your umask to `027` for the current shell, then create one file and one
directory. Predict their modes before checking.
*Verify:* `stat -c '%a %n' newfile newdir` shows `640` and `750`.

**4.25** Create `/srv/project` as a shared directory for `devs`: group `devs`, team can
create files, others have no access, and new files automatically belong to `devs`.
*Verify:* as alice, `touch /srv/project/a`; then `ls -l /srv/project/a` shows group `devs`.

**4.26** Continue from 2.9. As alice, create `/srv/project/plan.txt`. As bob, try to edit
it. If bob cannot, fix it so the team can always edit each other's files *without*
changing anything after each file is created.
*Verify:* bob can `echo edit >> /srv/project/plan.txt` on a file alice made after your fix.

**4.27** Make `/srv/project` a directory where team members cannot delete each other's
files, while still being able to create their own.
*Verify:* as bob, `rm /srv/project/<alice's file>` fails; `rm` of bob's own file succeeds.

**4.28** carol is not in `devs`. Give carol **read-only** access to
`/srv/project/plan.txt` without changing its owner, group, or carol's groups.
*Verify:* as carol, `cat` works and `echo x >> …` fails.

**4.29** Show that carol's access exists using `ls -l` alone — what tells you to look
further?
*Verify:* you can point to the `+`.

**4.30** Create `/srv/logs` and give group `auditors` read access to every file that
will *ever* be created in it, as well as the ones already there.
*Verify:* a file created afterwards shows `group:auditors:r--` in `getfacl`.

**4.31** Make `~/lab04/frozen.txt` impossible to modify or delete, even by root. Prove it,
then undo it.
*Verify:* `sudo rm frozen.txt` fails with *Operation not permitted*; after undoing, it succeeds.

**4.32** Create `audit.log` that can only be appended to — no truncation, no editing.
*Verify:* `echo a >> audit.log` works; `echo b > audit.log` fails, as root too.

**4.33** Using `find`, list every world-writable **file** under `/etc` and `/usr` (if any),
and every world-writable **directory** on the system that does *not* have the sticky bit.
*Verify:* `/tmp` does not appear in the second list.

**4.34** Recursively give a directory tree read access for everyone: directories must be
enterable, but no file may gain execute permission unless it already had it.
*Verify:* a plain `.txt` file in the tree is not executable afterwards.

---

## 🔴 Challenge

**4.35 — The unreachable file.** Run this setup:

```bash
sudo mkdir -p /srv/app/conf
echo "secret" | sudo tee /srv/app/conf/db.yml >/dev/null
sudo chmod 644 /srv/app/conf/db.yml
sudo chmod 700 /srv/app
```

As alice, `cat /srv/app/conf/db.yml` fails though the file is world-readable. Diagnose
using one command, then grant alice access with the smallest possible change — no
ownership changes, no making `/srv/app` world-accessible.
*Verify:* alice can read `db.yml` but `ls /srv/app` as carol still fails.

**4.36 — The mask trap.** Give bob `rw` via ACL on a file whose group bits are `r--`.
Then run `chmod 640` on it. What is bob's effective permission now? Restore bob's write
access *without* removing and re-adding his ACL entry.
*Verify:* `getfacl` shows no `#effective` restriction on bob.

**4.37 — Group not taking effect.** Add carol to `devs` while she has an open session
(`sudo -iu carol` in another terminal first). Show that her existing session cannot
write to `/srv/project`, then get it working in that session without logging out.
*Verify:* `id` in carol's session lists `devs` and she can create a file.

**4.38 — noexec.** Create a small filesystem mounted `noexec`, put an executable script
on it, and show that `./script.sh` fails despite `+x`. Then show the script still runs
another way, and explain why `noexec` is not a security boundary for scripts.

```bash
dd if=/dev/zero of=/tmp/small.img bs=1M count=20
mkfs.ext4 -q /tmp/small.img
sudo mkdir -p /mnt/noexec
sudo mount -o loop,noexec /tmp/small.img /mnt/noexec
```

*Verify:* `./script.sh` → Permission denied; `bash script.sh` → runs.

**4.39 — SUID scripts.** Create a root-owned script with the SUID bit that prints `id -u`.
Run it as alice. Explain the output.
*Verify:* the output is alice's UID, not `0`.

**4.40 — Bind a low port without root.** Write a tiny server with
`python3 -m http.server 80` as a normal user and watch it fail. Make it work by copying
the Python binary and giving the copy a single capability — never SUID, never sudo.
*Verify:* `curl -s localhost:80` returns a directory listing; `getcap` shows one capability.

**4.41 — Write the audit.** Write a one-page report (a text file) for ubuntu1 listing:
every SUID/SGID binary, every file with capabilities, every world-writable file outside
`/tmp` and `/var/tmp`, and every file with the immutable attribute under `/etc`. Use one
command per category.
*Verify:* each section of your report was produced by a single command you could paste
into a cron job.

---

## Self-check

- [ ] I can create users, groups and system accounts with the right options first time
- [ ] I never use `usermod -G` without `-a`
- [ ] I can write a narrow sudo rule and know which commands must never be granted
- [ ] I know which profile file runs for login, interactive and non-interactive shells
- [ ] I can set limits and prove they apply
- [ ] I can point a client at LDAP with SSSD and debug it in order
- [ ] I can say what `r`, `w`, `x` each mean on a directory
- [ ] I can compute umask results in my head
- [ ] I can set up a shared team directory with SGID and the right umask or default ACL
- [ ] I understand the ACL mask and why `chmod` changes it
- [ ] I know the difference between *Permission denied* and *Operation not permitted*
- [ ] I can find the blocking component of any path with `namei -l`

➡ **Solutions:** [solutions/04-users-permissions.md](../solutions/04-users-permissions.md)
