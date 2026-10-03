# Solutions 13 — SELinux

All on rocky1 unless stated.

---

## 🟢 Warm-up

**13.1**

```bash
sestatus
```

`Current mode` is now; `Mode from config file` is what the next boot uses.

**13.2**

```bash
ls -dZ /var/www/html /etc/shadow ~
ps -eZ | grep nginx
id -Z
sudo semanage port -l | grep -w 80
```

**13.3**

```bash
matchpathcon /var/www/html/index.html /srv/web/index.html
```

`/srv` has no special rule, so content there gets `var_t` — which nginx may not read.

**13.4**

```bash
getsebool -a | grep httpd | grep 'on$'
```

---

## 🔵 Practical

**13.5**

```bash
echo moved > ~/index.html
sudo mv ~/index.html /usr/share/nginx/html/index.html
curl -s -o /dev/null -w '%{http_code}\n' localhost/     # 403
sudo ausearch -m AVC -ts recent -c nginx
#  denied { read } … name="index.html" … tcontext=unconfined_u:object_r:user_home_t:s0
sudo restorecon -v /usr/share/nginx/html/index.html
curl -s localhost/                                       # moved
```

`mv` within a filesystem renames; the inode — and its label — stay as they were.

**13.6**

```bash
echo copied > ~/index2.html
sudo cp ~/index2.html /usr/share/nginx/html/
ls -Z /usr/share/nginx/html/index2.html          # httpd_sys_content_t
```

`cp` creates a new file, which inherits the type of the directory it's created in. (`cp -a`
or `--preserve=context` would copy the old label — see 13.20.)

**13.7**

```bash
sudo mkdir -p /srv/web && echo 'srv web' | sudo tee /srv/web/index.html
sudo sed -i 's#root\s\+/usr/share/nginx/html;#root /srv/web;#' /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx
curl -s localhost/                                       # 403 — var_t
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'
sudo restorecon -Rv /srv/web
curl -s localhost/                                       # srv web
sudo semanage fcontext -l -C
```

**13.8**

```bash
sudo mkdir -p /srv/temp && echo t | sudo tee /srv/temp/page.html
sudo chcon -t httpd_sys_content_t /srv/temp/page.html
ls -Z /srv/temp/page.html            # httpd_sys_content_t
sudo restorecon -v /srv/temp/page.html
ls -Z /srv/temp/page.html            # var_t again
```

`chcon` changed the file but not the policy's idea of what it should be. Any relabel — a
`restorecon`, a `/.autorelabel`, a package's post-install script — puts it back.

**13.9**

```bash
sudo sed -i '/listen\s\+80;/a \        listen       8081;' /etc/nginx/nginx.conf
sudo nginx -t
sudo systemctl restart nginx            # fails
sudo ausearch -m AVC -ts recent -c nginx | grep name_bind
sudo semanage port -a -t http_port_t -p tcp 8081
sudo systemctl restart nginx
sudo ss -tlnp | grep 8081
```

`nginx -t` passes — it checks syntax, not permission to bind. The restart is what fails.

**13.10**

```bash
sudo semanage port -l | grep -w 8000        # soundd_port_t
sudo semanage port -a -t http_port_t -p tcp 8000   # ValueError: Port tcp/8000 already defined
sudo semanage port -m -t http_port_t -p tcp 8000
```

Then add `listen 8000;` and restart nginx. `-m` modifies the type of a port the base policy
already assigns.

**13.11**

Add inside the `server { }` block in `/etc/nginx/nginx.conf`:

```nginx
location /app/ {
    proxy_pass http://192.168.56.12:8000/;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -s localhost/app/                       # 502
sudo ausearch -m AVC -ts recent -c nginx | audit2why
#  … Was caused by: The boolean httpd_can_network_connect was set incorrectly.
sudo setsebool -P httpd_can_network_connect on
curl -s localhost/app/                       # ubuntu2
```

By default the policy assumes a web server only *answers* connections. Proxying means
opening outgoing ones — `name_connect` — which the boolean allows.

**13.12**

```bash
mkdir -p ~/public_html && echo 'home page' > ~/public_html/index.html
chmod 711 ~ && chmod 755 ~/public_html
sudo restorecon -Rv ~/public_html            # → httpd_user_content_t (from the policy's pattern)
sudo setsebool -P httpd_enable_homedirs on
```

nginx (inside `server { }`):

```nginx
location ~ ^/~([^/]+)(/.*)?$ {
    alias /home/$1/public_html$2;
    index index.html;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -s localhost/~vagrant/
```

Three layers had to agree: Unix permissions (nginx must traverse `/home/vagrant`), the file
type (`httpd_user_content_t`), and the boolean that lets `httpd_t` read that type.

**13.13**

```bash
sudo mkdir -p /srv/web/uploads
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/srv/web/uploads(/.*)?'
sudo restorecon -Rv /srv/web
ls -Zd /srv/web /srv/web/uploads
```

The more specific rule wins for paths under `uploads`. Unix permissions still have to allow
the nginx user to write there.

**13.14**

```bash
sudo ausearch -m AVC -ts today | audit2why | less
```

Typical summary: "nginx denied read on `user_home_t` — missing type enforcement rule; the
file needs relabelling", or "denied `name_connect` — boolean `httpd_can_network_connect`".

**13.15**

```bash
sudo sealert -a /var/log/audit/audit.log | less
```

sealert lists each distinct problem with *If you want to … then you must …* suggestions and
a confidence score. Treat the suggestions as hints: it sometimes offers `audit2allow` where
a `restorecon` is right.

**13.16**

```bash
sudo semanage permissive -a httpd_t
sudo semanage permissive -l
# reproduce, e.g. a mislabeled file in the web root: the request now succeeds…
sudo ausearch -m AVC -ts recent -c nginx      # …and the AVC shows permissive=1
sudo semanage permissive -d httpd_t
getenforce                                    # Enforcing throughout
```

**13.17**

```bash
sudo useradd -m -G wheel intern
printf '%s:%s\n' intern 'Intern123!' | sudo chpasswd
sudo semanage login -a -s user_u intern
sudo semanage login -l | grep intern
ssh intern@localhost 'id -Z; sudo -l'          # user_u…; sudo fails
```

The mapping applies at login, so test with a real login (SSH or console), not `sudo -u`.
Unix says intern is in `wheel`; SELinux says `user_u` may not transition to root's domain —
mandatory beats discretionary.

---

## 🔴 Challenge

**13.18**

```bash
sudo tee /usr/local/bin/hello-web > /dev/null <<'EOF'
#!/usr/bin/env bash
exec python3 -m http.server 8090
EOF
sudo chmod 755 /usr/local/bin/hello-web
sudo tee /etc/systemd/system/hello-web.service > /dev/null <<'EOF'
[Service]
ExecStart=/usr/local/bin/hello-web
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl start hello-web
ps -eZ | grep http.server          # unconfined_service_t
ls -Z /usr/local/bin/hello-web     # bin_t
```

systemd starts it from a `bin_t` file, for which the policy defines no transition into a
confined domain, so it runs as `unconfined_service_t` — SELinux allows it almost everything.
If you labelled the script `httpd_exec_t`, it would transition into `httpd_t` and be
confined like nginx: port 8090 isn't `http_port_t`, so the bind would be denied. That's how
custom software gets confined — by reusing or writing a domain for it.

**13.19**

```bash
printf 'Port 22\nPort 2222\n' | sudo tee /etc/ssh/sshd_config.d/40-ports.conf
sudo firewall-cmd --permanent --add-port=2222/tcp && sudo firewall-cmd --reload
sudo sshd -t && sudo systemctl restart sshd
sudo journalctl -u sshd -n 5        # error: Bind to port 2222 … failed: Permission denied
sudo ausearch -m AVC -ts recent -c sshd | grep name_bind
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo systemctl restart sshd
sudo ss -tlnp | grep sshd           # 22 and 2222
```

From ubuntu1: `ssh -p 2222 vagrant@rocky1 hostname`. sshd still listened on 22, so the open
session was never at risk — but had you replaced 22 with 2222, the failed restart would have
left sshd listening on nothing.

**13.20**

```bash
sudo cp -a /root/.bashrc /usr/share/nginx/html/leak.txt
ls -Z /usr/share/nginx/html/leak.txt                  # admin_home_t — cp -a kept it
curl -s -o /dev/null -w '%{http_code}\n' localhost/leak.txt     # 403

# the WRONG fix
sudo ausearch -m AVC -ts recent -c nginx | audit2allow -M nginx_leak
cat nginx_leak.te
#   allow httpd_t admin_home_t:file { open read getattr };
sudo semodule -i nginx_leak.pp
curl -s localhost/leak.txt                             # works…

# undo it, and fix it properly
sudo semodule -r nginx_leak
sudo restorecon -v /usr/share/nginx/html/leak.txt      # → httpd_sys_content_t
curl -s localhost/leak.txt
```

The module lets the web server read **every** file of type `admin_home_t` — that is, root's
home directory. If nginx were ever compromised, `/root/.ssh/` would now be readable through
it. The real problem was one mislabelled file; the module fixed it by widening the policy for
everything.

**13.21**

```bash
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config && sudo reboot
sudo mkdir -p /srv/web/new && echo x | sudo tee /srv/web/new/f.html
ls -Z /srv/web/new/f.html             # httpd_sys_content_t — labelling still works in permissive
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config && sudo reboot
```

In permissive mode the policy is fully loaded: files get labels, only denials aren't
enforced. With SELinux **disabled**, nothing labels new files; when you re-enable it they
have no valid context and services fail across the board. The fix is a full relabel at the
next boot: `sudo touch /.autorelabel && sudo reboot` (or `fixfiles -F onboot`).

**13.22** (ubuntu2, after `vagrant snapshot save ubuntu2 pre-selinux`)

```bash
sudo systemctl disable --now apparmor
sudo apt install -y selinux-basics selinux-policy-default auditd policycoreutils-python-utils
sudo selinux-activate
sudo reboot                        # relabels, then reboots again
sestatus                           # enabled, permissive
sudo ausearch -m AVC -ts boot | audit2why | head -40
sudo mkdir -p /srv/web && echo ubuntu | sudo tee /srv/web/index.html
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'
sudo restorecon -Rv /srv/web
ls -Z /srv/web
```

Restore the snapshot afterwards — later labs assume AppArmor on ubuntu2. If the exam asks
for SELinux on an Ubuntu host, this sequence is what it looks like; check whether the host
already has it active (`sestatus`) before changing anything.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | DAC: owners decide access; MAC: a system-wide policy decides, and owners/processes can't override it | |
| 2 | It's allowed, and logged as an AVC with `permissive=1` | |
| 3 | `setenforce 0`; `SELINUX=enforcing` in `/etc/selinux/config` | |
| 4 | The type; for processes it's a domain | |
| 5 | `mv` kept `user_home_t`; `restorecon -v` the file | |
| 6 | `chcon` is undone by any relabel; use `semanage fcontext -a …` then `restorecon -Rv` | |
| 7 | `/srv/web` itself and everything beneath it | `(/.*)?` is optional |
| 8 | `semanage port -a -t http_port_t -p tcp 8081` | |
| 9 | The port has another type; use `-m` to modify it | |
| 10 | `httpd_can_network_connect` | |
| 11 | Makes the boolean change persistent across reboots | |
| 12 | `/var/log/audit/audit.log`; `ausearch -m AVC -ts recent` | |
| 13 | nginx may not connect to a `soundd_port_t` port: `setsebool -P httpd_can_network_connect on` (or relabel the port to `http_port_t` with `semanage port -m`) | |
| 14 | It allows whatever was denied, which may widen access far beyond the real need — fix labels, ports and booleans first | |
| 15 | A `dontaudit` rule; `semodule -DB`, reproduce, then `semodule -B` | |
| 16 | Relabel the filesystem at boot: `touch /.autorelabel`, then reboot | |
