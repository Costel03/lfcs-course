# Lab A — Bash scripting

20 tasks on **ubuntu1**. Each task produces a script in `~/labA/`. Run `shellcheck` on
every script before you call it done — clean output is part of the Verify step.

**Setup:**

```bash
sudo apt install -y shellcheck
mkdir -p ~/labA && cd ~/labA
printf 'alice,devs\nbob,devs\ncarol,ops\n\ndave,ops\n' > users.csv
mkdir -p rename-me && touch 'rename-me/My Report.PDF' 'rename-me/Holiday Photo.JPG' rename-me/notes.TXT
```

---

## 🟢 Warm-up

**A.1** `paths.sh FILE` — given any path, print its directory, file name, extension and
name-without-extension, using only parameter expansion (no `basename`, `dirname`, `sed`, `cut`).
*Verify:* `./paths.sh /var/log/nginx/access.log.1` prints `/var/log/nginx`, `access.log.1`,
`1`, `access.log`.

**A.2** `userinfo.sh USER` — print the user's UID, home and shell. Exit `0` if found, `1` if
not, `2` with a usage message if no argument was given.
*Verify:* `./userinfo.sh vagrant; echo $?` → details and `0`; `nosuch` → `1`; no argument → `2`.

**A.3** `svc-check.sh` — check a list of services held in an array (`ssh`, `cron`,
`systemd-journald`, `nosuch`) and print `OK` or `DOWN` for each, aligned. Exit non-zero if any
is down.
*Verify:* three `OK`, one `DOWN`, exit status `1`.

**A.4** `shells.sh` — using an associative array, count how many accounts use each login shell
in `/etc/passwd`, printing the counts sorted highest first.
*Verify:* matches `cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn`.

---

## 🔵 Practical

**A.5** `mkusers.sh FILE` — read `users.csv` (`user,group`), create each group and user if they
don't exist, and add the user to the group. Skip blank lines. Running it twice must change
nothing the second time and must print what it did (or skipped).
*Verify:* first run creates; second run prints only "exists" messages; `id alice` shows `devs`.

**A.6** `backup.sh -s SRC -d DEST [-k N] [-n]` — create `DEST/<basename of SRC>-YYYY-MM-DD_HHMMSS.tar.gz`,
then delete all but the newest `N` (default 5) backups of that source. `-n` prints what would
happen without doing it. Use `getopts`; reject unknown options with exit `2`.
*Verify:* run 7 times with `-k 3` — exactly 3 archives remain; with `-n` nothing changes.

**A.7** `disk-alert.sh [THRESHOLD]` — print every mounted real filesystem (no `tmpfs`, `devtmpfs`,
`overlay`, `squashfs`) above `THRESHOLD`% (default 80) and exit `1` if any; else print `all ok`
and exit `0`.
*Verify:* `./disk-alert.sh 1` lists `/` and exits `1`; `./disk-alert.sh 99` exits `0`.

**A.8** `tmpwork.sh` — create a temporary directory, write files into it, and guarantee it is
removed when the script ends — normally, on error, or on Ctrl-C.
*Verify:* print the directory's path; after a normal run, after a forced `false` with
`set -e`, and after Ctrl-C during a `sleep`, the directory doesn't exist.

**A.9** `single.sh` — a script that sleeps 30 seconds but refuses to run if another copy is
already running. No PID files.
*Verify:* a second copy started in another terminal exits immediately with a message.

**A.10** `retry.sh` — a function `retry N DELAY cmd…` that runs a command up to N times,
doubling the delay after each failure, and returns the command's last status. Demonstrate it
on `curl -sf http://localhost:9999/` (nothing listening) and on `true`.
*Verify:* the failing case shows N attempts with growing waits and returns non-zero; `true`
succeeds on the first attempt.

**A.11** `normalise.sh DIR [-n]` — rename every file in DIR to lower case with spaces replaced
by underscores. `-n` shows the renames without doing them. Never overwrite an existing file.
*Verify:* `rename-me/` ends up with `my_report.pdf`, `holiday_photo.jpg`, `notes.txt`.

**A.12** `log.sh` — a `log LEVEL MESSAGE` function that writes `2026-10-03 10:00:00 [LEVEL]
message` to stderr **and** sends it to the journal with tag `labA` at the matching priority
(`INFO` → info, `WARN` → warning, `ERROR` → err).
*Verify:* `journalctl -t labA -p warning` shows only the WARN and ERROR lines.

**A.13** `health.sh URL…` — for each URL print the HTTP status code and response time, with a
5-second timeout; exit with the number of URLs that failed.
*Verify:* `./health.sh https://ubuntu.com http://localhost:9 ; echo $?` → one success, one
failure, exit `1`.

**A.14** `sysreport.sh` — print a formatted summary table: hostname, OS, kernel, uptime, CPU
count, load, memory used/available, root filesystem use, number of logged-in users, failed
units.
*Verify:* aligned with `printf`; every value comes from a command, nothing hard-coded.

**A.15** `set-kv.sh FILE KEY VALUE` — ensure `KEY = VALUE` is set in an INI-style file: replace
an existing (possibly commented-out) `KEY` line, or append it if missing. Idempotent.
*Verify:* run against chapter 03's `app.conf` three times — exactly one active line for the key.

---

## 🔴 Challenge

**A.16 — Fix this script.** Save it as `broken.sh`, run `shellcheck`, and fix every problem —
then explain each one in a comment.

```bash
#!/bin/bash
set -e
target=$1
count=0
for f in $(ls $target/*.log); do
  size=$(stat -c %s $f)
  if [ $size -gt 1000 ]; then
    count=$((count+1))
  fi
done
cat $target/*.log | while read line; do
  total=$((total+1))
done
echo "big files: $count, lines: $total"
cd $target/old
rm -rf *
```

*Verify:* `shellcheck broken.sh` is clean; the script works on a directory whose log names
contain spaces; it never runs `rm` outside `$target/old`.

**A.17 — Menu.** `menu.sh` — an interactive menu (`select`) offering: show disk usage, show top
5 memory processes, show failed units, quit. Invalid choices print a message and redisplay.
*Verify:* each option works; `q`/quit exits `0`.

**A.18 — From script to service.** Turn `disk-alert.sh` into a systemd service + timer that runs
every 15 minutes, logs its output to the journal, and is visibly *failed* when a filesystem is
over the threshold.
*Verify:* `systemctl list-timers`; with threshold 1, `systemctl status` shows the service failed
and the journal shows the filesystems.

**A.19 — Parallel.** `pingsweep.sh 192.168.56` — ping hosts `.1`–`.20` in parallel (one ping each,
1 s timeout) and print the ones that answered, sorted numerically. Must finish in a few seconds.
*Verify:* lists ubuntu1, ubuntu2, rocky1 (and the host); completes in under 5 s.

**A.20 — Dry-run everywhere.** Add a `DRY_RUN=true` environment mode to `mkusers.sh` using a `run`
wrapper so that **every** state-changing command is printed instead of executed.
*Verify:* `DRY_RUN=true ./mkusers.sh users.csv` changes nothing (`getent passwd` unchanged) and
prints `+ useradd …` lines for users that don't exist yet.

---

## Self-check

- [ ] My scripts start with a shebang and `set -euo pipefail`, and I know its gaps
- [ ] I do string surgery with parameter expansion
- [ ] I read files with `while IFS= read -r`, redirected, not piped
- [ ] I parse options with `getopts` and print usage to stderr with exit 2
- [ ] I clean up with `trap … EXIT` and `mktemp`, and lock with `flock`
- [ ] My scripts are idempotent and can dry-run
- [ ] `shellcheck` is clean

➡ **Solutions:** [solutions/A-scripting.md](../solutions/A-scripting.md)
