# Quiz 06 — Services, scheduling and logs

20 questions. Answers: [solutions/06-services.md](../solutions/06-services.md#quiz-answers)

---

**Q1.** A task says "ensure nginx is running". What two states must be true, and which one
command achieves both?

**Q2.** There is a unit file at `/usr/lib/systemd/system/app.service` and another at
`/etc/systemd/system/app.service`. Which one does systemd use?

**Q3.** What is the difference between `systemctl edit app` and `systemctl edit --full app`?
Which should you normally use, and why?

**Q4.** In a drop-in you write a new `ExecStart=` line, and the service now refuses to
start. What did you forget?

**Q5.** You edited a unit file, restarted the service, and nothing changed. What step was
missed?

**Q6.** What does `After=postgresql.service` do on its own? What does it *not* do?

**Q7.** `Wants=` versus `Requires=` — what happens to your service if the dependency fails
to start?

**Q8.** What is wrong with this line?

```ini
ExecStart=/usr/local/bin/app --port 8080 >> /var/log/app.log
```

**Q9.** A service fails with status `203/EXEC`. Name two likely causes.

**Q10.** A service fails with `217/USER`. What is wrong?

**Q11.** How do you limit a service to 512 MB of memory, and what happens when it exceeds it?

**Q12.** What is the difference between `systemctl disable` and `systemctl mask`?

**Q13.** You created `backup.service` and `backup.timer`. Which one do you enable?

**Q14.** What does `Persistent=true` in a timer do?

**Q15.** Write a crontab line that runs `/usr/local/bin/report.sh` at 06:45 every Monday.

**Q16.** This cron line runs but produces a file called `backup-` with nothing after the
dash. Why?

```text
0 1 * * * tar czf /backup/backup-$(date +%F).tgz /etc
```

**Q17.** What field does a line in `/etc/cron.d/` have that a user's crontab line doesn't?

**Q18.** `journalctl -b -1` says there is no previous boot. Why, and how do you fix it for
the future?

**Q19.** Show only errors and worse from the `ssh` unit since this boot.

**Q20.** After logrotate runs, the application keeps writing to `app.log.1` instead of the
new `app.log`. Name two ways to fix the logrotate configuration.
