# Quiz 02 — The file system

18 questions. Answers: [solutions/02-filesystem.md](../solutions/02-filesystem.md#quiz-answers)

---

**Q1.** You want to change how the `nginx` service starts. The unit file lives in
`/usr/lib/systemd/system/nginx.service`. Why should you *not* edit it there, and where
does your change go instead?

**Q2.** Which directory should hold temporary files that must survive a reboot?

- A) `/tmp`  B) `/run`  C) `/var/tmp`  D) `/var/cache`

**Q3.** What does an inode **not** contain?

- A) the owner  B) the file's name  C) the size  D) the link count

**Q4.** Why is renaming a 50 GB file within one filesystem instant, but moving it to
another filesystem slow?

**Q5.** `df -h` shows 40% used but writes fail with *No space left on device*. What do
you check next, and with which command?

**Q6.** Name two things a hard link cannot do that a symbolic link can.

**Q7.** `ln -s ../config/app.yml /srv/app/current/app.yml` — relative to which
directory is `../config/app.yml` resolved?

**Q8.** Which timestamp cannot be set by an ordinary user with `touch`, and why does that
make it useful during an investigation?

**Q9.** What is wrong with this command, and what could it do?

```bash
find . -name *.log -delete
```

**Q10.** What is the difference between these two?

```bash
find /backups -delete -name '*.old'
find /backups -name '*.old' -delete
```

**Q11.** `find /var/log -mtime +7` — does a file modified exactly 7.5 days ago match?

**Q12.** What does the first character mean in each line?

```text
brw-rw----  …  /dev/sda
crw-rw-rw-  …  /dev/null
srw-rw-rw-  …  /run/systemd/journal/socket
prw-r--r--  …  /tmp/pipe
```

**Q13.** `du -sh /data` reports 2 GB, but `df` says the filesystem holding `/data` has
30 GB used. Give two possible explanations.

**Q14.** What does each of these produce?

```bash
rsync -a src/ dest/
rsync -a src dest/
```

**Q15.** You run `rm -rf current/` where `current` is a symlink to
`/srv/app/releases/v42`. What happens?

**Q16.** Which compression tool would you pick for nightly backups where speed matters
and the ratio should still be good?

**Q17.** When `tar -czf` prints *Removing leading '/' from member names*, why is that a
safety feature?

**Q18.** `cp -r` and `cp -a` both copy a directory tree. What does `-a` preserve that
`-r` does not?
