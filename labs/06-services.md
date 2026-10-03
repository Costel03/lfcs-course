# Lab 06 — Services, scheduling and logs

27 tasks on **ubuntu1**. From this chapter on, every task that changes system state has
two verifications: it works **now**, and it would still work **after a reboot**. That is
exactly how the exam is marked. Do the quiz first.

**Setup:**

```bash
sudo apt install -y nginx at
sudo systemctl stop nginx       # tasks below decide how it should run
```

**Cleanup:**

```bash
for u in webapp prepare broken disk-report lab06-flap mover; do
  sudo systemctl disable --now "$u".service "$u".timer "$u".path 2>/dev/null
  sudo rm -f /etc/systemd/system/"$u".{service,timer,path}
done
sudo rm -rf /etc/systemd/system/nginx.service.d /etc/systemd/system.control/webapp.service.d \
            /etc/default/webapp /etc/cron.d/lab06 /etc/logrotate.d/disk-report /srv/webapp /srv/incoming /srv/processed
sudo systemctl daemon-reload; sudo userdel webapp 2>/dev/null
```

---

## 🟢 Warm-up

**6.1** For the `ssh` service, answer from one command: is it running, is it enabled, which
unit file is loaded, and are there drop-ins?
*Verify:* each answer came from the `Loaded:`, `Drop-In:` and `Active:` lines.

**6.2** List any failed units on the system, and count the running services.
*Verify:* the count came from `systemctl list-units`, not from `ps`.

**6.3** Show the default boot target and every timer with its next run time.
*Verify:* `multi-user.target` (or `graphical.target`); `logrotate.timer` is in the list.

**6.4** From the journal show: (a) the `ssh` unit's messages from the last hour, (b) kernel
messages from this boot, (c) only messages of priority `err` or worse from this boot.
*Verify:* three different `journalctl` commands.

**6.5** Print the unit file systemd is actually using for `cron.service`, then show what
`systemctl is-enabled` reports for `systemd-journald.service` and explain the answer.
*Verify:* `static` — and you can say why a unit with no `[Install]` section can't be enabled.

---

## 🔵 Practical

**6.6** Make nginx run now and at every boot. Check both conditions with commands that
print one word each.
*Verify:* `systemctl is-active nginx` → `active`; `systemctl is-enabled nginx` → `enabled`;
`curl -s localhost | head -3` shows HTML.

**6.7** Without touching the vendor unit file, give nginx an open-files limit of 8192 and
make it restart automatically on failure.
*Verify:* `systemctl show nginx -p LimitNOFILE -p Restart`; the master process's
`/proc/<pid>/limits` shows 8192; the vendor file is unchanged (`systemctl cat` shows your drop-in separately).

**6.8** Kill the nginx master process with `SIGKILL`. Show that systemd restarted it.
*Verify:* the main PID changed; `journalctl -u nginx` shows the restart.

**6.9** Make nginx impossible to start, try to start it, then undo it.
*Verify:* `systemctl start nginx` fails with a message about the unit being masked;
afterwards nginx is running and enabled again.

**6.10** Create a system user `webapp` and a service `webapp.service` that runs
`/usr/bin/python3 -m http.server 8000` as that user, from `/srv/webapp` (containing an
`index.html` with the text `webapp ok`). It must restart on failure, start after the network
is online, and start at boot.
*Verify:* `curl -s localhost:8000` prints `webapp ok`; `ps -o user= -p $(systemctl show -p MainPID --value webapp)` prints `webapp`; enabled.

**6.11** Move the port into `/etc/default/webapp` as `PORT=8001` and make the unit read it.
*Verify:* `ss -tlnp | grep 8001` shows python; nothing listens on 8000.

**6.12** Limit `webapp` to 100 MB of memory and 20% of one CPU, persistently, without
editing its unit file by hand.
*Verify:* `systemctl show webapp -p MemoryMax -p CPUQuotaPerSecUSec` reflects both; a
drop-in exists under `/etc/systemd/system.control/`.

**6.13** Install this broken unit and get it running. Fix each problem as you find it, and
note the status code each one produced.

```ini
# /etc/systemd/system/broken.service
[Unit]
Description=Broken on purpose

[Service]
User=ghostuser
ExecStart=/usr/local/bin/hello-lab.sh

[Install]
WantedBy=multi-user.target
```

(`hello-lab.sh` should print `hello` and sleep 600. It doesn't exist yet.)
*Verify:* you recorded `217/USER` and `203/EXEC` (and how you fixed each); finally
`systemctl is-active broken` → `active`.

**6.14** Create `disk-report.service` and `disk-report.timer` that append the output of
`df -h /` with a timestamp to `/var/log/disk-report.log` every 5 minutes.
*Verify:* `systemctl list-timers disk-report.timer` shows the next run; after triggering the
service once by hand, the log has an entry; the timer (not the service) is enabled.

**6.15** Without creating anything, show the next three times the expression
`Mon..Fri 08:30` would fire.
*Verify:* three weekday dates at 08:30.

**6.16** As alice, add a crontab entry that appends the output of `date '+%F %T'` to
`~/cron.log` every 10 minutes, Monday to Friday, between 08:00 and 18:59.
*Verify:* `sudo crontab -l -u alice` shows the line with `%` escaped; within ten minutes of
a weekday working hour, `~alice/cron.log` has a timestamp.

**6.17** Create a system cron file `/etc/cron.d/lab06` that runs `/usr/bin/logger lab06-monthly`
as root at 03:15 on the first day of every month.
*Verify:* the line has six fields before the command (five time fields plus `root`).

**6.18** Stop carol from using cron at all.
*Verify:* as carol, `crontab -l` reports she is not allowed.

**6.19** Schedule a job with `at` that creates `/tmp/at-ran` in two minutes. Schedule a
second one for tomorrow 23:00 and then cancel it.
*Verify:* `/tmp/at-ran` appears; `atq` is empty at the end.

**6.20** Make the journal persistent, reboot, and show messages from the previous boot.
*Verify:* `journalctl --list-boots` lists at least two boots; `journalctl -b -1 | head` works.

**6.21** From the shell, write the message `lab06 checkpoint` to the log with the tag
`lab06` at priority `warning`. Retrieve it using a filter on the tag *and* the priority.
*Verify:* `journalctl -t lab06 -p warning` shows it.

**6.22** Configure log rotation for `/var/log/disk-report.log`: daily, keep 5, compressed,
new file mode `0640` owned by `root:adm`. Test it without rotating, then force a rotation.
*Verify:* `logrotate -d` shows the plan; after `-f`, `disk-report.log.1` exists and a new
empty `disk-report.log` has mode `640`.

---

## 🔴 Challenge

**6.23 — React to files.** Whenever a file appears in `/srv/incoming`, it must be moved to
`/srv/processed` automatically. Use a `.path` unit, no polling loop, no cron.
*Verify:* `touch /srv/incoming/a.txt`; within a second it is in `/srv/processed`.

**6.24 — Dependencies that mean it.** Create a oneshot service `prepare.service` that
writes `/srv/webapp/index.html`. Make `webapp` start only after `prepare` has succeeded,
and refuse to start if `prepare` fails. Prove the second part by breaking `prepare`.
*Verify:* with a broken `prepare`, `systemctl restart webapp` fails with a dependency
error; with it fixed, both are active.

**6.25 — start-limit-hit.** Create `lab06-flap.service` running `/bin/false` with
`Restart=always` and `RestartSec=200ms`. Start it and explain the final state. Then make
systemd willing to try it again.
*Verify:* status shows `start-limit-hit`; after your command, `systemctl start` is attempted
again (and fails again, as expected).

**6.26 — Who's knocking?** Generate three failed SSH logins
(`ssh nosuchuser@localhost` three times, wrong passwords). Using only `journalctl` and text
tools, print the number of failed attempts per source address in the last 15 minutes.
*Verify:* `::1` or `127.0.0.1` with a count of at least 3.

**6.27 — Harden it.** Run `systemd-analyze security webapp` and note the exposure score.
Add at least four hardening settings from the notes without breaking the service.
*Verify:* the score is lower; `curl -s localhost:8001` still works.

---

## Self-check

- [ ] I always check a service is both active **and** enabled
- [ ] I change vendor units only through drop-ins, and know to clear `ExecStart=` first
- [ ] I can write a service with user, environment, restart policy and limits from memory
- [ ] I can read `203/EXEC`, `217/USER` and `start-limit-hit` and know what to do
- [ ] I can write a timer, a crontab line and an `at` job, and verify each
- [ ] I can query the journal by unit, boot, time, priority and tag
- [ ] I can write and test a logrotate rule

➡ **Solutions:** [solutions/06-services.md](../solutions/06-services.md)
