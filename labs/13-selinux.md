# Lab 13 — SELinux

22 tasks on **rocky1**, plus 13.22 on **ubuntu2**. Rule for the whole lab: **SELinux stays
enforcing**. Permissive is allowed only as a diagnostic step, and every task ends enforcing.
Do the quiz first.

**Setup (rocky1):**

```bash
sudo dnf install -y nginx policycoreutils-python-utils setroubleshoot-server selinux-policy-doc
sudo systemctl enable --now nginx
sudo firewall-cmd --permanent --add-service=http --add-port=8081/tcp && sudo firewall-cmd --reload
```

---

## 🟢 Warm-up

**13.1** Show the current mode, the mode configured for boot, and the policy name.
*Verify:* `sestatus` shows `enforcing` / `targeted`.

**13.2** Show the context of: `/var/www/html`, `/etc/shadow`, your home directory, the
running nginx processes, your own shell, and port 80.
*Verify:* six contexts; nginx runs as `httpd_t`; port 80 is `http_port_t`.

**13.3** What type *should* `/var/www/html/index.html` have according to the policy, and what
type should `/srv/web/index.html` have? Don't create either file.
*Verify:* `matchpathcon` gives `httpd_sys_content_t` and `var_t` (or `default_t`).

**13.4** List every boolean related to `httpd` that is currently on.
*Verify:* `getsebool -a | grep httpd | grep on$`.

---

## 🔵 Practical

**13.5** Create `index.html` in your home directory with the text `moved`, **move** it into
`/usr/share/nginx/html/` (the Rocky nginx web root), and request it with curl. Find the
denial, explain it, and fix it.
*Verify:* the first curl returns 403 and `ausearch` shows `user_home_t`; after the fix curl
returns `moved`.

**13.6** Repeat 13.5 with `cp` instead of `mv`. Explain why there's no denial this time.
*Verify:* the copied file has `httpd_sys_content_t` immediately.

**13.7** Serve a new site from `/srv/web` on port 80 instead of the default root (change
nginx's `root`). Label the content **persistently**.
*Verify:* curl works; `semanage fcontext -l -C` lists your rule; after
`sudo restorecon -Rv /srv/web` the labels don't change (already correct).

**13.8** Prove `chcon` is temporary: create `/srv/temp/page.html`, label it with `chcon`,
then run `restorecon` on it. What happened?
*Verify:* the type reverted to `var_t`/`default_t`.

**13.9** Make nginx also listen on port `8081`. Make it start successfully.
*Verify:* `sudo ss -tlnp | grep 8081` shows nginx; `semanage port -l -C` shows your change.

**13.10** Make nginx listen on port `8000` too. (Port 8000 already belongs to another type.)
*Verify:* nginx starts; `semanage port -l | grep 8000` shows `http_port_t`.

**13.11** Configure nginx as a reverse proxy for `http://ubuntu2:8000` (the app from lab 11)
on location `/app/`. Find out why it returns 502 and fix it persistently.
*Verify:* `curl -s localhost/app/` prints `ubuntu2`; `getsebool httpd_can_network_connect` → `on`;
still on after reboot.

**13.12** Allow nginx to serve files from users' `~/public_html` directories, using a boolean
and the right file labels.
*Verify:* `curl localhost/~vagrant/` (with an nginx `location` for it) returns your test page.
(Remember the home directory must be traversable — permissions still apply too.)

**13.13** Give nginx a directory `/srv/web/uploads` it may **write** to, while the rest of
`/srv/web` stays read-only to it.
*Verify:* `ls -Zd /srv/web/uploads` shows `httpd_sys_rw_content_t`; the parent still
`httpd_sys_content_t`.

**13.14** Use `audit2why` on a denial from earlier in the lab and summarise in one line what
it tells you.
*Verify:* your summary names the cause (missing type/boolean/port).

**13.15** Install `setroubleshoot-server` (done in setup), reproduce a denial, and read
`sealert`'s explanation of it.
*Verify:* `sealert -a /var/log/audit/audit.log` suggests a command matching your own fix.

**13.16** Make only the `httpd_t` domain permissive, reproduce a denial, show it's logged but
not blocked, then make the domain enforcing again.
*Verify:* `semanage permissive -l` lists `httpd_t`, then doesn't; `getenforce` stayed `Enforcing`.

**13.17** Confine a new user `intern` as SELinux user `user_u`, and show they can't use
`sudo` or `su` even if they're in `wheel`.
*Verify:* `semanage login -l` shows the mapping; `id -Z` as intern shows `user_u`; `sudo -l`
fails.

---

## 🔴 Challenge

**13.18 — The unknown binary.** Write a tiny service: a script in `/usr/local/bin/hello-web`
that runs `python3 -m http.server 8090`, with a systemd unit. Which domain does it run in,
and why is SELinux not stopping it from binding port 8090? Then explain how that would
change if the script were labelled `httpd_exec_t`.
*Verify:* `ps -eZ | grep http.server` shows `unconfined_service_t`.

**13.19 — SSH on a new port.** On rocky1, make sshd listen on port `2222` *in addition to*
22. First try it with only the sshd and firewall changes, and capture what fails and where
it's logged. Then complete it properly. Keep a second session open throughout.
*Verify:* the first restart fails with a bind error and an AVC for `name_bind` on 2222;
after your fix, `ssh -p 2222 vagrant@rocky1` works from ubuntu1 and `getenforce` is
`Enforcing`.

**13.20 — The wrong fix, then the right one.** Make nginx serve a file that carries the
`admin_home_t` type: `sudo cp -a /root/.bashrc /usr/share/nginx/html/leak.txt` (`cp -a`
preserves the label). curl gets 403. Fix it first the **wrong** way — `audit2allow -M` and
`semodule -i` — and read the generated `.te` file. Then remove the module and fix it the
right way.
*Verify:* the `.te` contains `allow httpd_t admin_home_t:file …`; you can say why that's
dangerous; after `semodule -r` and `restorecon`, curl works with no module installed.

**13.21 — Relabel after disabling.** Set `SELINUX=permissive`, reboot, create files under
`/srv/web` while permissive, set enforcing back, and reboot. Do the new files have correct
labels? Now explain what would have happened had you used *disabled* instead, and what fixes it.
*Verify:* files created in permissive mode are labelled correctly; your explanation mentions
`/.autorelabel`.

**13.22 — SELinux on Ubuntu.** On **ubuntu2** (snapshot first!), switch from AppArmor to
SELinux in permissive mode, check for denials from normal operation, then label a web
directory with `semanage fcontext` exactly as on Rocky. Restore the snapshot afterwards.
*Verify:* `sestatus` shows enabled/permissive; `ls -Z` shows SELinux contexts on Ubuntu.

---

## Self-check

- [ ] I never use `setenforce 0` as a fix — only to confirm SELinux is the cause
- [ ] I can read an AVC line and name the fix: label, port, boolean or module
- [ ] I label persistently with `semanage fcontext` + `restorecon`, never `chcon`
- [ ] I know `mv` keeps labels and `cp` doesn't
- [ ] I can add or modify a port type, and set booleans with `-P`
- [ ] I know reverse proxies need `httpd_can_network_connect`
- [ ] I can enable SELinux on Ubuntu and recognise AppArmor

➡ **Solutions:** [solutions/13-selinux.md](../solutions/13-selinux.md)
