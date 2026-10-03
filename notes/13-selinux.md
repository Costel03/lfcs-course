# Chapter 13 — SELinux

Permissions (chapter 04) let a file's owner decide who may use it. That's *discretionary*
access control: if a web server is compromised, the attacker gets everything the server's
user can touch. SELinux adds *mandatory* access control: a policy, set by the administrator
and not changeable by users or processes, that says what each kind of process may do —
even when ordinary permissions would allow more.

The usual "fix" for SELinux is `setenforce 0`. This chapter is about the real fix: reading
the denial and changing a label, a port, or a boolean.

LFCS: *Operations Deployment* — create and enforce MAC using SELinux.

## Objectives

After this chapter you can:

- Check and change SELinux mode, temporarily and persistently.
- Read security contexts on files, processes, users and ports.
- Fix file labels permanently with `semanage fcontext` and `restorecon`.
- Allow a service a non-standard port, and enable behaviour with booleans.
- Find, read and explain AVC denials, and choose the right fix.
- Write a small policy module when nothing else fits — and know when not to.
- Enable SELinux on Ubuntu, and recognise AppArmor, Ubuntu's default.

Run this chapter's examples on **rocky1**, where SELinux is enforcing out of the box.

---

## 1. Modes

```bash
getenforce                 # Enforcing / Permissive / Disabled
sestatus                   # mode, policy, and the mode configured for boot
sudo setenforce 0          # permissive until reboot (1 = enforcing)
```

| Mode | Policy checked | Denials blocked | Denials logged |
|---|---|---|---|
| **Enforcing** | yes | yes | yes |
| **Permissive** | yes | no | yes |
| **Disabled** | no | no | no |

Persistent mode: `/etc/selinux/config`

```text
SELINUX=enforcing
SELINUXTYPE=targeted
```

**Permissive is the diagnostic mode**: everything works, and every action that *would*
have been denied is logged. If a problem disappears in permissive, it's SELinux; turn
enforcing back on and fix the cause.

Going from **disabled** to enabled needs a reboot and a full relabel, because files created
while disabled have no labels: `sudo touch /.autorelabel && sudo reboot`. On current RHEL,
disabling is done on the kernel command line (`selinux=0`), not in the config file.

You can also make one domain permissive instead of the whole system:
`sudo semanage permissive -a httpd_t` — useful for debugging one service.

---

## 2. Security contexts

Every process, file, port and user has a **context** of four fields:

```text
system_u : object_r : httpd_sys_content_t : s0
  user      role          TYPE              level (MLS/MCS)
```

The **targeted** policy is almost entirely **type enforcement**: rules of the form "a process
of type X may do Y to an object of type Z". Process types are called **domains**.

```bash
ls -Z /var/www/html              # file types
ls -dZ /var/www/html
ps -eZ | grep nginx              # process domains: httpd_t
id -Z                            # your own context: unconfined_u:unconfined_r:unconfined_t:s0…
sudo semanage port -l | grep -w http_port_t     # port types
```

The rule that lets nginx serve pages is, in effect:

```text
allow httpd_t httpd_sys_content_t : file { read open getattr };
```

So nginx (`httpd_t`) can read `/var/www/html/index.html` (`httpd_sys_content_t`) but not
`/home/alice/notes.txt` (`user_home_t`) — even if that file is `chmod 777`.

### How a process gets its domain

When systemd starts `/usr/sbin/nginx`, the file's type (`httpd_exec_t`) triggers a
**transition** into `httpd_t`. A binary you put in `/usr/local/bin` has type `bin_t` and runs
as `unconfined_service_t` — SELinux doesn't restrict it, because the policy knows nothing
about it.

Users logged in normally run `unconfined_t`: the targeted policy confines *services*, not
administrators.

---

## 3. File labels

### Where labels come from

1. The policy holds a list of **path patterns → types**: `semanage fcontext -l`.
2. A new file normally **inherits** the type of its parent directory.
3. `restorecon` resets a file to the type the pattern list says it should have.

```bash
sudo semanage fcontext -l | grep '/var/www'
matchpathcon /var/www/html/index.html      # what the policy says it should be
```

### The move trap

```bash
echo hi > ~/index.html                  # created in home: user_home_t
sudo mv ~/index.html /var/www/html/     # mv KEEPS the label: still user_home_t
sudo cp ~/index.html /var/www/html/     # cp creates a new file: httpd_sys_content_t
```

A file moved into the web root keeps its home-directory label, and nginx gets *403
Forbidden* though permissions look perfect. This is the single most common SELinux problem.

```bash
sudo restorecon -v /var/www/html/index.html      # fix it
```

### Changing labels — the right way and the wrong way

| | Command | Survives a relabel? |
|---|---|---|
| ❌ temporary | `chcon -t httpd_sys_content_t /srv/web/index.html` | **no** — `restorecon` or `/.autorelabel` reverts it |
| ✅ persistent | `semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'` then `restorecon -Rv /srv/web` | yes |

The persistent way adds a rule to the policy's pattern list, then applies it:

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'
sudo restorecon -Rv /srv/web
ls -Zd /srv/web /srv/web/*
```

`'/srv/web(/.*)?'` is a regular expression: the directory itself, plus everything below it.
Quote it.

```bash
sudo semanage fcontext -l -C               # only your local customisations
sudo semanage fcontext -d '/srv/web(/.*)?' # remove a rule
sudo semanage fcontext -a -e /var/www /srv/web   # "label /srv/web like /var/www" (equivalence)
sudo restorecon -Rnv /srv                  # -n: dry run — show what WOULD change
```

Common types:

| Type | For |
|---|---|
| `httpd_sys_content_t` | web content, read-only |
| `httpd_sys_rw_content_t` | web content the server may write (uploads, caches) |
| `httpd_log_t` | web server logs |
| `public_content_t` | shared read-only content (FTP, Samba, web) |
| `samba_share_t` | Samba shares |
| `user_home_t`, `user_home_dir_t` | home directories |
| `var_log_t` | logs |
| `container_file_t` | files mounted into containers (`:Z` in podman) |

---

## 4. Ports

Services may only bind ports of the right type:

```bash
sudo semanage port -l | grep -E '^(http_port_t|ssh_port_t)'
#  http_port_t   tcp   80, 81, 443, 488, 8008, 8009, 8443, 9000
#  ssh_port_t    tcp   22
sudo semanage port -a -t http_port_t -p tcp 8081      # add
sudo semanage port -m -t http_port_t -p tcp 8000      # modify: port already has another type
sudo semanage port -d -t http_port_t -p tcp 8081      # delete
```

nginx configured to `listen 8081;` without this fails to start with *Permission denied*
on bind — and `ss` shows nothing listening. Same for sshd on port 2222 (`ssh_port_t`).

---

## 5. Booleans

Booleans switch optional parts of the policy on and off, without writing rules:

```bash
getsebool -a | grep httpd
sudo semanage boolean -l | grep httpd_can_network_connect    # with descriptions
sudo setsebool httpd_can_network_connect on       # until reboot
sudo setsebool -P httpd_can_network_connect on    # -P: persistent
```

| Boolean | Allows |
|---|---|
| `httpd_can_network_connect` | web server to open outgoing connections — **needed for nginx as a reverse proxy** |
| `httpd_can_network_connect_db` | web server to connect to database ports |
| `httpd_enable_homedirs` | serve `~/public_html` |
| `httpd_use_nfs` | serve content from NFS mounts |
| `ftpd_full_access` | FTP access to the whole filesystem |
| `samba_enable_home_dirs` | share home directories |
| `use_nfs_home_dirs` | home directories on NFS |

Forgetting `-P` is the boolean version of the runtime-only firewall rule: works until reboot.

---

## 6. Reading denials

Denials are **AVC** (access vector cache) messages, logged by `auditd` to
`/var/log/audit/audit.log`:

```bash
sudo ausearch -m AVC -ts recent              # last 10 minutes
sudo ausearch -m AVC -ts today -c nginx      # by command
sudo grep denied /var/log/audit/audit.log | tail
```

```text
type=AVC msg=audit(1727950000.123:456): avc:  denied  { read } for  pid=2210 comm="nginx"
  name="index.html" dev="dm-0" ino=1311 scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0
```

Read it as a sentence: **`nginx`** (domain **`httpd_t`**) was **denied `read`** on a **file**
named **`index.html`** whose type is **`user_home_t`**. The fix is obvious once you read it:
the file has the wrong label.

### Let the tools explain

```bash
sudo ausearch -m AVC -ts recent | audit2why        # which rule, and whether a boolean would allow it
sudo dnf install setroubleshoot-server
sudo sealert -a /var/log/audit/audit.log           # plain-English diagnosis with suggested commands
journalctl -t setroubleshoot                       # sealert summaries appear here too
```

### Choosing the fix

| Denial says | Fix |
|---|---|
| Wrong file type (`user_home_t`, `default_t`, `var_t`…) | `semanage fcontext` + `restorecon` (or plain `restorecon` if the rule exists) |
| `name_bind` on a port | `semanage port -a` |
| `name_connect` from a web server | boolean `httpd_can_network_connect` |
| `audit2why` names a boolean | `setsebool -P` |
| None of the above, the access is legitimate | a local policy module (section 7) |

### Silent denials

Some denials are hidden by `dontaudit` rules to keep logs readable. If something fails with
no AVC logged, disable them temporarily:

```bash
sudo semodule -DB          # rebuild policy without dontaudit
# reproduce, read ausearch
sudo semodule -B           # restore
```

---

## 7. Custom policy modules — last resort

When an application legitimately needs access the policy doesn't grant, and no label,
port or boolean covers it:

```bash
sudo ausearch -m AVC -ts recent -c myapp | audit2allow -m myapp     # show the rules it would write
sudo ausearch -m AVC -ts recent -c myapp | audit2allow -M myapp     # create myapp.te / myapp.pp
cat myapp.te                                                         # READ IT
sudo semodule -i myapp.pp
sudo semodule -l | grep myapp
sudo semodule -r myapp                                               # remove
```

`audit2allow` blindly allows whatever was denied. If the denial was really a wrong label,
you've just granted a service access to a type it should never touch. Always try labels,
ports and booleans first, and read the `.te` file before installing it.

---

## 8. Confining users

By default Linux users map to `unconfined_u`. Mapping a login to a confined SELinux user
restricts what it can do regardless of Unix permissions:

```bash
sudo semanage login -l
sudo semanage login -a -s user_u alice      # alice: no su, no sudo, no SUID programs
sudo semanage login -a -s guest_u guest     # no network, no execution in home or /tmp
```

| SELinux user | Can |
|---|---|
| `unconfined_u` | anything Unix permissions allow |
| `staff_u` | normal work; `sudo` only to roles granted |
| `user_u` | normal work; no `su`/`sudo` |
| `guest_u` | terminal only, no network, no execution of own files |

---

## 9. SELinux on Ubuntu

Ubuntu ships AppArmor, but SELinux can be enabled — and it's what the LFCS objective names.
Use a disposable VM (**ubuntu2**, snapshot first):

```bash
sudo systemctl disable --now apparmor        # two MAC systems can't both be active
sudo apt install -y selinux-basics selinux-policy-default auditd
sudo selinux-activate                        # adds security=selinux to GRUB, schedules a relabel
sudo reboot                                  # first boot relabels the filesystem, then reboots
sestatus                                     # enabled, mode: permissive
```

Ubuntu activates SELinux in **permissive** mode. Check for denials from normal operation
before enforcing:

```bash
sudo ausearch -m AVC -ts boot | audit2why | less
sudo selinux-config-enforcing                # set enforcing in /etc/selinux/config
sudo reboot
```

Ubuntu's SELinux policy is the upstream reference policy (`default`), less polished than
RHEL's `targeted` — expect more denials and fix them with the same tools. All of
`semanage`, `restorecon`, `getsebool`, `ausearch` work identically.

---

## 10. AppArmor — Ubuntu's default

Recognise it, because on an Ubuntu box it's what's actually running:

| | SELinux | AppArmor |
|---|---|---|
| Model | labels on every object | profiles listing **paths** each program may use |
| Default on | RedHat family | Ubuntu, Debian, SUSE |
| Status | `sestatus` | `sudo aa-status` |
| Policy | `semanage`, booleans, modules | files in `/etc/apparmor.d/` |
| Debug mode | permissive | `sudo aa-complain /etc/apparmor.d/usr.sbin.foo` |
| Enforce | enforcing | `sudo aa-enforce …` |
| Denials | `/var/log/audit/audit.log` | `journalctl -k \| grep apparmor="DENIED"` |

`aa-complain`/`aa-enforce` come from the `apparmor-utils` package. After editing a profile:
`sudo apparmor_parser -r /etc/apparmor.d/<profile>`.

---

## 11. Gotchas

**`setenforce 0` as a fix.** It's a diagnostic switch. Find the denial and fix the cause.

**`mv` keeps labels; `cp` doesn't.** Moved files often need `restorecon`.

**`chcon` is temporary** — the next relabel undoes it. Use `semanage fcontext`.

**Forgetting `-P` on `setsebool`** — reverts at reboot.

**A new port without `semanage port`** — the service fails to bind with *Permission denied*.

**Reverse proxy returns 502** — `httpd_can_network_connect` is off.

**Unquoted fcontext regex** — the shell eats `(/.*)?`.

**Re-enabling a disabled system without a relabel** — unlabelled files, nothing works. Use
`/.autorelabel`.

**`audit2allow -M` without reading the `.te`** — you may be allowing an attack.

---

## 12. Commands introduced

| Command | Purpose |
|---|---|
| `getenforce`, `setenforce`, `sestatus`, `/etc/selinux/config` | modes |
| `ls -Z`, `ps -Z`, `id -Z` | contexts |
| `semanage fcontext`, `restorecon`, `chcon`, `matchpathcon` | file labels |
| `semanage port` | port labels |
| `getsebool`, `setsebool -P`, `semanage boolean -l` | booleans |
| `ausearch -m AVC`, `audit2why`, `sealert` | diagnose denials |
| `audit2allow`, `semodule` | custom policy modules |
| `semanage login`, `semanage permissive` | confine users; per-domain permissive |
| `selinux-activate`, `selinux-config-enforcing` | SELinux on Ubuntu |
| `aa-status`, `aa-complain`, `aa-enforce` | AppArmor |
| `man selinux`, `man semanage-fcontext`, `man httpd_selinux`, `man booleans` | offline references |

`dnf install selinux-policy-doc` provides a man page for every service domain
(`man httpd_selinux`) listing its types, ports and booleans — invaluable with no browser.

---

➡ **Next:** [quiz](../quizzes/13-selinux.md) → [lab](../labs/13-selinux.md)
