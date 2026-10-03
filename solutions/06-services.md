# Solutions 06 — Services, scheduling and logs

---

## 🟢 Warm-up

**6.1**

```bash
systemctl status ssh
```

On Ubuntu 22.10 and later you may find `ssh.service` *inactive* until the first
connection, because `ssh.socket` listens on port 22 and starts the service on demand.
`systemctl status ssh.socket` shows the listener. That is socket activation, not a fault.

**6.2**

```bash
systemctl list-units --failed
systemctl list-units --type=service --state=running --no-legend | wc -l
```

**6.3**

```bash
systemctl get-default
systemctl list-timers --all
```

**6.4**

```bash
journalctl -u ssh --since '1 hour ago'
journalctl -k -b
journalctl -p err -b
```

**6.5**

```bash
systemctl cat cron
systemctl is-enabled systemd-journald      # static
```

`static` means the unit has no `[Install]` section, so there is nothing for `enable` to
hook into — it is started because something else requires it (here, very early boot).

---

## 🔵 Practical

**6.6**

```bash
sudo systemctl enable --now nginx
systemctl is-active nginx; systemctl is-enabled nginx
curl -s localhost | head -3
```

**6.7**

```bash
sudo systemctl edit nginx
```

```ini
[Service]
LimitNOFILE=8192
Restart=on-failure
```

```bash
sudo systemctl restart nginx
systemctl show nginx -p LimitNOFILE -p Restart
grep 'open files' /proc/$(systemctl show -p MainPID --value nginx)/limits
systemctl cat nginx          # vendor file first, then "# /etc/systemd/system/nginx.service.d/override.conf"
```

Limits apply when the process starts, so the restart is required.

**6.8**

```bash
old=$(systemctl show -p MainPID --value nginx)
sudo kill -9 "$old"
sleep 1
systemctl show -p MainPID --value nginx      # a new PID
journalctl -u nginx -n 5
```

`Restart=on-failure` restarts after an unclean signal (KILL, SEGV, ABRT…) or a non-zero
exit, but not after a clean stop. That's why `systemctl stop` does not trigger a restart.

**6.9**

```bash
sudo systemctl mask nginx
sudo systemctl start nginx       # Failed to start nginx.service: Unit nginx.service is masked.
sudo systemctl unmask nginx
sudo systemctl enable --now nginx
```

`mask` links the unit to `/dev/null`. Note that masking a running service doesn't stop
it — it only prevents future starts. Unmasking also leaves it disabled, hence the
`enable --now`.

**6.10**

```bash
sudo useradd -r -s /usr/sbin/nologin -d /srv/webapp webapp
sudo mkdir -p /srv/webapp
echo 'webapp ok' | sudo tee /srv/webapp/index.html
sudo chown -R webapp: /srv/webapp

sudo tee /etc/systemd/system/webapp.service > /dev/null <<'EOF'
[Unit]
Description=Lab web application
After=network-online.target
Wants=network-online.target

[Service]
User=webapp
Group=webapp
WorkingDirectory=/srv/webapp
ExecStart=/usr/bin/python3 -m http.server 8000
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now webapp
curl -s localhost:8000
```

**6.11**

```bash
echo 'PORT=8001' | sudo tee /etc/default/webapp
sudo systemctl edit webapp
```

```ini
[Service]
EnvironmentFile=/etc/default/webapp
ExecStart=
ExecStart=/usr/bin/python3 -m http.server ${PORT}
```

```bash
sudo systemctl restart webapp
ss -tlnp | grep -E ':800[01]'
```

systemd expands `${PORT}` itself; it is not a shell expansion. Remember to clear
`ExecStart=` first in the drop-in.

**6.12**

```bash
sudo systemctl set-property webapp MemoryMax=100M CPUQuota=20%
systemctl show webapp -p MemoryMax -p CPUQuotaPerSecUSec    # 104857600, 200ms
ls /etc/systemd/system.control/webapp.service.d/
```

`CPUQuota=20%` is stored as 200 ms of CPU time per second. cgroup properties like these
apply immediately to the running service — no restart needed.

**6.13**

```bash
sudo systemctl daemon-reload
sudo systemctl start broken
systemctl status broken          # status=217/USER
```

Fix 1 — use a real account (here `nobody`):

```bash
sudo sed -i 's/^User=ghostuser/User=nobody/' /etc/systemd/system/broken.service
sudo systemctl daemon-reload && sudo systemctl start broken
systemctl status broken          # status=203/EXEC
```

Fix 2 — create the program:

```bash
sudo tee /usr/local/bin/hello-lab.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
echo hello
exec sleep 600
EOF
sudo chmod 755 /usr/local/bin/hello-lab.sh
sudo systemctl start broken
systemctl is-active broken
journalctl -u broken -n 3        # "hello" is in the journal: stdout goes there
```

`203/EXEC` also appears when the file exists but isn't executable, or its shebang names an
interpreter that doesn't exist — check all three.

**6.14**

```bash
sudo tee /etc/systemd/system/disk-report.service > /dev/null <<'EOF'
[Unit]
Description=Append disk usage to a report

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo "== $$(date --iso-8601=seconds)" >> /var/log/disk-report.log; df -h / >> /var/log/disk-report.log'
EOF

sudo tee /etc/systemd/system/disk-report.timer > /dev/null <<'EOF'
[Unit]
Description=Disk report every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now disk-report.timer
sudo systemctl start disk-report.service
systemctl list-timers disk-report.timer
tail /var/log/disk-report.log
```

Shell features (redirection, `$(…)`) need an explicit `bash -c`. Two unit-file escapes
matter here: systemd expands `$VAR` itself, so a dollar meant for bash is written `$$`;
and `%` introduces systemd specifiers (`%n`, `%h`…), so a literal percent is `%%`. That
is why the timestamp uses `date --iso-8601` rather than `date +%F`.

**6.15**

```bash
systemd-analyze calendar --iterations=3 'Mon..Fri 08:30'
```

**6.16**

```bash
sudo crontab -u alice -e
```

```text
*/10 8-18 * * 1-5 date '+\%F \%T' >> $HOME/cron.log
```

**6.17**

```bash
echo '15 3 1 * * root /usr/bin/logger lab06-monthly' | sudo tee /etc/cron.d/lab06
```

Files in `/etc/cron.d/` need the user field, must not be group- or world-writable, and on
Debian must have names without dots.

**6.18**

```bash
echo carol | sudo tee -a /etc/cron.deny
sudo -iu carol crontab -l     # You (carol) are not allowed to use this program (crontab)
```

If `/etc/cron.allow` exists, *only* users listed in it may use cron, and `cron.deny` is
ignored.

**6.19**

```bash
echo 'touch /tmp/at-ran' | at now + 2 minutes
echo 'logger never' | at 23:00 tomorrow
atq
atrm <job-number-of-the-second>
atq
```

**6.20**

```bash
ls -d /var/log/journal 2>/dev/null || echo 'not persistent yet'
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\n' | sudo tee /etc/systemd/journald.conf.d/persistent.conf
sudo systemctl restart systemd-journald
sudo reboot
# reconnect
journalctl --list-boots
journalctl -b -1 | head
```

Ubuntu usually ships with `/var/log/journal` already present, so the journal may already
be persistent — the check before the change tells you. `Storage=persistent` makes it
explicit and creates the directory if needed.

**6.21**

```bash
logger -t lab06 -p user.warning 'lab06 checkpoint'
journalctl -t lab06 -p warning
```

**6.22**

```bash
sudo tee /etc/logrotate.d/disk-report > /dev/null <<'EOF'
/var/log/disk-report.log {
    daily
    rotate 5
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root adm
}
EOF
sudo logrotate -d /etc/logrotate.d/disk-report
sudo logrotate -f /etc/logrotate.d/disk-report
ls -l /var/log/disk-report.log*
```

With `delaycompress`, the first rotated file stays as `.1`; it's compressed at the next
rotation. The timer job simply opens the file again each run, so no `postrotate` is needed.

---

## 🔴 Challenge

**6.23**

```bash
sudo mkdir -p /srv/incoming /srv/processed

sudo tee /etc/systemd/system/mover.path > /dev/null <<'EOF'
[Unit]
Description=Watch /srv/incoming

[Path]
DirectoryNotEmpty=/srv/incoming

[Install]
WantedBy=multi-user.target
EOF

sudo tee /etc/systemd/system/mover.service > /dev/null <<'EOF'
[Unit]
Description=Move incoming files

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'mv -n /srv/incoming/* /srv/processed/'
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now mover.path
sudo touch /srv/incoming/a.txt; sleep 1; ls /srv/processed
```

A `.path` unit uses inotify, so it reacts in milliseconds without polling. By default it
starts the service with the same name. `DirectoryNotEmpty=` keeps triggering while files
remain, so a batch arriving mid-move is still picked up.

**6.24**

```bash
sudo tee /etc/systemd/system/prepare.service > /dev/null <<'EOF'
[Unit]
Description=Prepare webapp content

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'echo "prepared $$(date)" > /srv/webapp/index.html'
EOF

sudo systemctl edit webapp
```

```ini
[Unit]
Requires=prepare.service
After=prepare.service
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart webapp && systemctl is-active prepare webapp

# break it
sudo sed -i 's#/srv/webapp/index.html#/nonexistent/x#' /etc/systemd/system/prepare.service
sudo systemctl daemon-reload
sudo systemctl stop prepare
sudo systemctl restart webapp      # A dependency job for webapp.service failed.
```

You need **both** lines: `Requires=` makes `prepare` start and makes its failure fatal;
`After=` makes `webapp` wait for it. With `Requires=` alone they'd start in parallel and
`webapp` could be running before `prepare` failed. `RemainAfterExit=yes` keeps the
oneshot "active" after it finishes, which is what makes "both are active" true. Restore
the path afterwards.

**6.25**

```bash
sudo tee /etc/systemd/system/lab06-flap.service > /dev/null <<'EOF'
[Service]
ExecStart=/bin/false
Restart=always
RestartSec=200ms
EOF
sudo systemctl daemon-reload
sudo systemctl start lab06-flap
sleep 3; systemctl status lab06-flap      # Failed with result 'start-limit-hit'
sudo systemctl reset-failed lab06-flap
sudo systemctl start lab06-flap           # tries again
```

The default rate limit is 5 starts within 10 seconds (`StartLimitBurst=`,
`StartLimitIntervalSec=` in the `[Unit]` section). It stops a broken service from
restarting in a tight loop forever. After fixing the real problem, `reset-failed` clears
the counter.

**6.26**

```bash
for i in 1 2 3; do ssh -o StrictHostKeyChecking=accept-new nosuchuser@localhost true; done
journalctl -u ssh --since '15 min ago' | grep 'Invalid user' \
  | awk '{for (i = 1; i <= NF; i++) if ($i == "from") print $(i+1)}' | sort | uniq -c
```

Every attempt for a non-existent user logs `Invalid user nosuchuser from <ip> port <n>`,
whatever authentication method was tried. Locating the field *after* `from` is more robust
than counting fields, because the message shape varies between OpenSSH versions.

**6.27**

```bash
systemd-analyze security webapp | tail -1
sudo systemctl edit webapp
```

```ini
[Service]
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictSUIDSGID=yes
```

```bash
sudo systemctl restart webapp
systemd-analyze security webapp | tail -1
curl -s localhost:8001
```

`ProtectSystem=strict` makes the whole filesystem read-only for the service; this app only
reads `/srv/webapp`, so it still works. An app that writes data needs
`ReadWritePaths=` for those directories.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | Active now **and** enabled at boot; `systemctl enable --now nginx` | Exams check both |
| 2 | `/etc/systemd/system/app.service` | `/etc` overrides `/usr/lib` |
| 3 | `edit` creates a drop-in with only your changes; `--full` copies the whole unit | Use drop-ins so package updates to the unit still apply |
| 4 | An empty `ExecStart=` line first, to clear the inherited one | Only oneshot services may have several |
| 5 | `systemctl daemon-reload` | systemd caches unit files |
| 6 | Orders startup if both are starting; it doesn't start postgresql or require it | Add `Wants=` or `Requires=` |
| 7 | `Wants=`: your service still starts; `Requires=`: it doesn't (and stops if the dependency stops) | |
| 8 | `>>` isn't interpreted — no shell; it's passed to the app as an argument | Use `bash -c '…'` or log to stdout |
| 9 | Wrong/relative path; not executable; bad shebang interpreter | |
| 10 | The `User=` account doesn't exist | |
| 11 | `MemoryMax=512M` (unit or `set-property`); the cgroup OOM-kills the process | Result shows `oom-kill` |
| 12 | `disable` removes boot links (still startable); `mask` makes it unstartable | |
| 13 | `backup.timer` | The service has no `[Install]` section |
| 14 | Runs a missed schedule at the next opportunity after boot | |
| 15 | `45 6 * * 1 /usr/local/bin/report.sh` | |
| 16 | `%` is a newline in crontab; escape it as `\%` | |
| 17 | The user to run as | `/etc/crontab` too |
| 18 | The journal is volatile (in `/run`); set `Storage=persistent` or create `/var/log/journal` | |
| 19 | `journalctl -u ssh -b -p err` | |
| 20 | `copytruncate`; or `create` plus a `postrotate` that makes the app reopen its log (reload/HUP) | |
