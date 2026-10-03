# Chapter 06 — Services, scheduling and logs

Almost everything a server does runs as a service. This chapter teaches you to manage
services, write your own, constrain them, schedule work with timers, cron and `at`, and
find out afterwards what happened.

LFCS: *Essential Commands* — create, configure and troubleshoot services; *Operations
Deployment* — manage and schedule jobs, troubleshoot services.

## Objectives

After this chapter you can:

- Start, stop, enable, mask and inspect any unit, and explain `enable` versus `start`.
- Read a unit file and know which copy systemd is actually using.
- Override a vendor unit safely with a drop-in.
- Write a service unit from scratch, with restart policy, user, environment and limits.
- Diagnose a service that won't start from its status, exit code and journal.
- Schedule work with systemd timers, cron and `at`, and know which to choose.
- Query the journal by unit, boot, time and priority, and make it persistent.
- Configure log rotation and test it.

---

## 1. systemd in one page

systemd is PID 1. It starts and supervises everything else, described as **units**:

| Unit type | Describes | Example |
|---|---|---|
| `.service` | a process to run and supervise | `ssh.service` |
| `.socket` | a listening socket; starts a service on first connection | `ssh.socket` |
| `.timer` | a schedule that starts a service | `logrotate.timer` |
| `.target` | a group of units — a "state" the system can reach | `multi-user.target` |
| `.mount` / `.automount` | a filesystem mount (generated from `/etc/fstab`) | `home.mount` |
| `.path` | watch a path; start a service when it changes | |

### Where unit files live, and which wins

| Directory | Owner | Priority |
|---|---|---|
| `/etc/systemd/system/` | **you** | highest |
| `/run/systemd/system/` | runtime (transient units) | middle |
| `/usr/lib/systemd/system/` (`/lib/…` on older Debian) | packages | lowest |

A file in `/etc` with the same name **replaces** the package's file entirely. A
**drop-in** directory — `/etc/systemd/system/nginx.service.d/*.conf` — *adds to* it, which
is almost always what you want.

---

## 2. systemctl

```bash
systemctl status nginx            # state, PID, memory, recent log lines
systemctl start | stop | restart nginx
systemctl reload nginx            # re-read config without stopping (if supported)
systemctl enable nginx            # start at boot
systemctl enable --now nginx      # start at boot AND now
systemctl disable --now nginx
systemctl is-active nginx         # scriptable: active / inactive / failed
systemctl is-enabled nginx        # enabled / disabled / masked / static
systemctl mask nginx              # make it impossible to start, even by dependency
systemctl unmask nginx
systemctl list-units --type=service --state=running
systemctl list-units --failed
systemctl list-unit-files --type=service
systemctl cat nginx               # the unit file(s) systemd is ACTUALLY using
systemctl show nginx -p User -p Restart -p LimitNOFILE
systemctl list-dependencies multi-user.target
systemctl daemon-reload           # re-read unit files after editing them
```

### start versus enable — the exam distinction

| | Running now | Running after reboot |
|---|---|---|
| `start` | ✅ | ❌ |
| `enable` | ❌ | ✅ |
| `enable --now` | ✅ | ✅ |

"Ensure the service is running" in a task almost always means **both**. Checking it:

```bash
systemctl is-active nginx && systemctl is-enabled nginx
```

### Reading status

```text
● nginx.service - A high performance web server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/nginx.service.d
             └─override.conf
     Active: active (running) since Fri 2026-10-03 10:12:01 UTC; 3min ago
   Main PID: 2210 (nginx)
      Tasks: 3 (limit: 2275)
     Memory: 4.1M
        CPU: 32ms
     CGroup: /system.slice/nginx.service
             ├─2210 "nginx: master process"
             └─2211 "nginx: worker process"
```

`Loaded:` tells you which file, whether it's enabled, and whether drop-ins apply.
`Active:` gives the state and, on failure, the result (`exit-code`, `signal`, `timeout`,
`oom-kill`, `start-limit-hit`).

---

## 3. Changing a vendor unit — drop-ins

```bash
sudo systemctl edit nginx
```

opens an editor on `/etc/systemd/system/nginx.service.d/override.conf` and reloads
systemd when you save. Add only what you change:

```ini
[Service]
LimitNOFILE=65536
Restart=on-failure
Environment=APP_ENV=production
```

To **replace** a list-type setting such as `ExecStart=`, clear it first with an empty
assignment — otherwise you add a second one, and a non-oneshot service refuses to start:

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/nginx -g 'daemon off;' -c /etc/nginx/custom.conf
```

`systemctl edit --full nginx` copies the whole unit into `/etc` instead — use it rarely,
because you then stop receiving the package's fixes to the unit.

`systemctl revert nginx` removes your overrides and returns to the vendor version.

---

## 4. Writing a service

`/etc/systemd/system/webapp.service`:

```ini
[Unit]
Description=Example web application
Documentation=man:python3(1)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=webapp
Group=webapp
WorkingDirectory=/srv/webapp
EnvironmentFile=-/etc/default/webapp
ExecStart=/usr/bin/python3 -m http.server 8000
Restart=on-failure
RestartSec=5
LimitNOFILE=4096
MemoryMax=256M

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webapp
```

### The settings that matter

| Setting | Meaning |
|---|---|
| `After=` / `Before=` | **ordering only** — if both start, start this one after X |
| `Wants=` | weak dependency — start X too; carry on if X fails |
| `Requires=` | strong dependency — if X fails or stops, stop this too |
| `Type=simple` / `exec` | `ExecStart` is the main process and stays in the foreground |
| `Type=forking` | the program daemonises itself; needs `PIDFile=` |
| `Type=oneshot` | runs to completion; often with `RemainAfterExit=yes` |
| `Type=notify` | the program tells systemd when it is ready |
| `ExecStart=` | **absolute path**, no shell features |
| `ExecStartPre=` / `ExecStartPost=` / `ExecReload=` / `ExecStop=` | hooks |
| `Restart=` | `no`, `on-failure`, `always`, `on-abnormal` |
| `User=` / `Group=` | run unprivileged — always, unless it truly needs root |
| `Environment=` / `EnvironmentFile=` | variables; `-` before the path means "ignore if missing" |
| `WantedBy=multi-user.target` | what `enable` hooks it into |

**`After=` without `Wants=` or `Requires=` does not start anything.** It only orders units
that are starting anyway. Most "my service starts before the network" bugs are this.

### No shell in ExecStart

```ini
ExecStart=/usr/bin/myapp > /var/log/myapp.log 2>&1      # WRONG: ">" is passed as an argument
ExecStart=/bin/bash -c '/usr/bin/myapp >> /var/log/myapp.log 2>&1'   # works, if you must
```

Better: let the program write to stdout and let the journal collect it (section 8).

### Constraining a service

These are how you answer "this service must not use more than…":

| Setting | Limits |
|---|---|
| `LimitNOFILE=`, `LimitNPROC=`, `LimitCORE=` | per-process rlimits (what `limits.conf` does for logins) |
| `MemoryMax=` | memory of the whole cgroup — exceeded means OOM kill |
| `MemoryHigh=` | soft memory limit — throttles instead of killing |
| `CPUQuota=50%` | at most half of one CPU |
| `TasksMax=` | processes and threads |
| `IOWeight=` | relative disk priority |

Change them on a running unit without editing files:

```bash
sudo systemctl set-property webapp MemoryMax=128M CPUQuota=25%
```

(`set-property` writes a persistent drop-in under `/etc/systemd/system.control/`; add
`--runtime` for a temporary change.)

### Light hardening

```ini
[Service]
NoNewPrivileges=yes          # can never gain privileges, e.g. through SUID binaries
PrivateTmp=yes               # its own /tmp
ProtectSystem=strict         # /usr, /etc read-only to it
ReadWritePaths=/srv/webapp/data
ProtectHome=yes
AmbientCapabilities=CAP_NET_BIND_SERVICE   # bind ports < 1024 without root
```

`systemd-analyze security webapp` scores a unit and suggests more.

---

## 5. When a service won't start

```bash
systemctl status webapp
journalctl -xeu webapp            # -x explanations, -e jump to end, -u this unit
systemd-analyze verify /etc/systemd/system/webapp.service
```

The exit status in `status` usually names the problem directly:

| Status | Meaning | Typical cause |
|---|---|---|
| `203/EXEC` | couldn't execute `ExecStart` | wrong path, not executable, bad shebang, relative path |
| `217/USER` | couldn't switch to `User=` | user doesn't exist |
| `200/CHDIR` | couldn't enter `WorkingDirectory=` | missing directory or no permission |
| `226/NAMESPACE` | sandboxing setup failed | a `ReadWritePaths=` that doesn't exist |
| `1`, `2`, … | the program itself exited with an error | read its log lines |
| `start-limit-hit` | restarted too often too quickly | fix the cause, then `systemctl reset-failed` |

Working order: **status → journal → run `ExecStart` by hand as the service user** — e.g.
`sudo -u webapp /usr/bin/python3 -m http.server 8000` — which shows the real error in your
terminal.

Edited the unit but nothing changed? You forgot `systemctl daemon-reload`. `status` warns
you: *"The unit file … changed on disk. Run 'systemctl daemon-reload'."*

---

## 6. Targets and boot

Targets group units into system states:

| Target | Equivalent | Meaning |
|---|---|---|
| `multi-user.target` | runlevel 3 | normal server: networking, no GUI |
| `graphical.target` | runlevel 5 | with a desktop |
| `rescue.target` | runlevel 1 | single user, local filesystems, few services |
| `emergency.target` | — | root shell, root filesystem read-only, nothing else |

```bash
systemctl get-default
sudo systemctl set-default multi-user.target
sudo systemctl isolate rescue.target       # switch now — drops your SSH session!
```

Chapter 12 covers booting into these from GRUB to recover a broken machine.

---

## 7. Scheduling

### systemd timers

A timer starts a service of the same name on a schedule. Two files:

`/etc/systemd/system/backup.service`

```ini
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

`/etc/systemd/system/backup.timer`

```ini
[Unit]
Description=Run backup nightly

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer      # enable the TIMER, not the service
systemctl list-timers
systemd-analyze calendar 'Mon..Fri 08:00'     # check an expression before using it
sudo systemctl start backup.service           # run it once now, to test
```

| Setting | Meaning |
|---|---|
| `OnCalendar=daily` / `weekly` / `Mon *-*-* 09:00` / `*:0/15` | wall-clock schedule |
| `OnBootSec=10min` | 10 minutes after boot |
| `OnUnitActiveSec=1h` | 1 hour after the service last started |
| `Persistent=true` | if the machine was off at the scheduled time, run at next boot |
| `RandomizedDelaySec=` | spread load across many machines |

Why timers over cron: output goes to the journal automatically, runs are visible in
`list-timers`, missed runs can catch up, and the job gets every unit feature — `User=`,
limits, sandboxing, dependencies.

### cron

```bash
crontab -e            # edit your own
crontab -l            # list
sudo crontab -u alice -e
```

```text
# ┌ minute (0-59)
# │  ┌ hour (0-23)
# │  │  ┌ day of month (1-31)
# │  │  │  ┌ month (1-12)
# │  │  │  │  ┌ day of week (0-7, 0 and 7 = Sunday)
# │  │  │  │  │
  30 2  *  *  *    /usr/local/bin/backup.sh
  */15 * * * *     /usr/local/bin/check.sh          # every 15 minutes
  0 9  *  *  1-5   /usr/local/bin/report.sh         # 09:00 on weekdays
  @reboot          /usr/local/bin/on-boot.sh
```

System-wide cron files have an extra **user** field:

```text
# /etc/crontab and /etc/cron.d/*
30 2 * * *   root   /usr/local/bin/backup.sh
```

Scripts dropped into `/etc/cron.daily/`, `/etc/cron.weekly/` etc. run on that schedule
(through `run-parts` — on Debian, file names there must not contain a dot).

Access control: `/etc/cron.allow` and `/etc/cron.deny`.

**cron's three classic traps:**

1. **Minimal environment.** `PATH` is usually just `/usr/bin:/bin`. Use full paths, or set
   `PATH=` at the top of the crontab.
2. **`%` is special** — it means newline. `date +%F` must be written `date +\%F`.
3. **Output goes to local mail**, which nobody reads. Redirect it:
   `… >> /var/log/backup.log 2>&1`.

### at — run once

```bash
echo '/usr/local/bin/cleanup.sh' | at now + 30 minutes
at 23:00 tomorrow <<< 'systemctl restart webapp'
atq                     # queued jobs
at -c 3                 # show job 3's full script
atrm 3                  # remove it
```

`at` needs the `atd` service running (`sudo apt install at`). A one-off timer without files:
`sudo systemd-run --on-active=30m /usr/local/bin/cleanup.sh`.

### Which scheduler?

| Need | Use |
|---|---|
| Recurring, on a new system | systemd timer |
| Recurring, existing cron culture, or the task says "cron" | cron |
| Once, at a time | `at` or `systemd-run --on-calendar=` |
| Once, after a delay | `systemd-run --on-active=` |

---

## 8. The journal

`systemd-journald` collects stdout/stderr of every service, kernel messages and syslog
messages into one indexed store.

```bash
journalctl -u nginx                 # one unit
journalctl -u nginx -f              # follow
journalctl -u nginx --since '1 hour ago'
journalctl --since '2026-10-03 09:00' --until '2026-10-03 10:00'
journalctl -b                       # this boot
journalctl -b -1                    # previous boot (needs persistence)
journalctl --list-boots
journalctl -p err                   # priority err and worse
journalctl -p warning -b -u ssh
journalctl -k                       # kernel ring buffer (like dmesg)
journalctl _PID=1234
journalctl _UID=1001
journalctl -o verbose -n 1          # every field of the newest entry
journalctl -o json-pretty -n 1
journalctl --disk-usage
sudo journalctl --vacuum-size=200M  # trim
sudo journalctl --vacuum-time=7d
```

Priorities, highest first: `emerg 0`, `alert 1`, `crit 2`, `err 3`, `warning 4`,
`notice 5`, `info 6`, `debug 7`. `-p err` means "err and anything more severe".

### Persistence

By default the journal is persistent only if `/var/log/journal` exists — otherwise it
lives in `/run` and is lost at reboot, and `journalctl -b -1` says there is no previous
boot. To make it persistent:

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\nSystemMaxUse=500M\n' | \
  sudo tee /etc/systemd/journald.conf.d/persistent.conf
sudo systemctl restart systemd-journald
journalctl --list-boots
```

### Writing to the log

```bash
logger "Backup started"
logger -t backup -p user.err "Backup failed: disk full"
journalctl -t backup
```

`logger` from a script is how you make cron jobs and scripts leave a searchable trail.

### rsyslog and /var/log

Many systems also run `rsyslog`, which receives messages and writes classic text files:

| File | Debian/Ubuntu | RedHat |
|---|---|---|
| General | `/var/log/syslog` | `/var/log/messages` |
| Authentication | `/var/log/auth.log` | `/var/log/secure` |
| Kernel | `/var/log/kern.log` | in `messages` |

Its rules (`/etc/rsyslog.d/*.conf`) are `facility.priority  destination`:

```text
local0.*        /var/log/myapp.log
*.err           /var/log/errors.log
auth,authpriv.* /var/log/auth.log
```

Forward everything to a central server: `*.* @@logserver:514` (`@@` TCP, `@` UDP).

---

## 9. Log rotation

`logrotate` runs daily (a systemd timer or cron job) and rotates according to
`/etc/logrotate.conf` and `/etc/logrotate.d/*`:

```text
/var/log/webapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 webapp adm
    sharedscripts
    postrotate
        systemctl reload webapp >/dev/null 2>&1 || true
    endscript
}
```

| Directive | Meaning |
|---|---|
| `daily` / `weekly` / `monthly` / `size 100M` | when to rotate |
| `rotate 14` | keep 14 old files |
| `compress`, `delaycompress` | gzip old logs, but leave the newest old one uncompressed |
| `missingok`, `notifempty` | no error if missing; skip if empty |
| `create MODE USER GROUP` | create a fresh log after moving the old one |
| `copytruncate` | copy then truncate in place — for programs that can't reopen their log |
| `postrotate … endscript` | e.g. tell the program to reopen its files |

```bash
sudo logrotate -d /etc/logrotate.d/webapp     # debug: show what WOULD happen
sudo logrotate -f /etc/logrotate.d/webapp     # force a rotation now
```

The program must stop writing to the old file after rotation — via `postrotate` reload,
or `copytruncate`. Otherwise it keeps writing to the renamed (or deleted) file — the
deleted-but-open disk leak from chapter 05.

---

## 10. Gotchas

**Forgetting `daemon-reload`** after editing a unit — the old version keeps running.

**Enabling the service instead of the timer.** `systemctl enable backup.service` does
nothing useful when the service has no `[Install]` section; enable `backup.timer`.

**`After=network.target` doesn't wait for the network.** Use
`After=network-online.target` **and** `Wants=network-online.target`.

**A masked unit can't start** — not even manually. `systemctl status` says `masked`;
`unmask` it.

**Relative paths and shell syntax in `ExecStart=`** fail with `203/EXEC` or with arguments
like `>` passed literally.

**cron's `%`, `PATH` and silent output** — see section 7.

**Logs vanish at reboot** without a persistent journal.

---

## 11. Commands introduced

| Command | Purpose |
|---|---|
| `systemctl` | manage units |
| `systemctl edit`, `revert`, `cat`, `show`, `set-property` | inspect and change units |
| `systemd-analyze verify`, `calendar`, `security`, `blame` | check units, schedules, boot |
| `systemd-run` | transient services and one-off timers |
| `journalctl` | query the journal |
| `logger` | write to the log |
| `crontab`, `at`, `atq`, `atrm` | schedule jobs |
| `logrotate -d / -f` | test and force rotation |
| `man systemd.service`, `systemd.unit`, `systemd.timer`, `systemd.exec`, `systemd.resource-control`, `crontab(5)` | offline references for every setting |

---

➡ **Next:** [quiz](../quizzes/06-services.md) → [lab](../labs/06-services.md)
