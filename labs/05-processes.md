# Lab 05 — Processes and performance

26 tasks on **ubuntu1**. Several tasks need two terminals — open a second `vagrant ssh
ubuntu1`. Do the quiz first.

**Setup:**

```bash
sudo apt install -y sysstat strace psmisc
mkdir -p ~/lab05 && cd ~/lab05
```

**Cleanup:** `pkill -u "$USER" -x sleep; pkill -u "$USER" -x yes; rm -rf ~/lab05 /tmp/lab05*`

---

## 🟢 Warm-up

**5.1** Show your shell's PID, its parent's PID, and its full ancestry up to PID 1.
*Verify:* the ancestry ends at `systemd`.

**5.2** List the five processes using the most memory, showing PID, user, `%MEM`, resident
size and command.
*Verify:* the list is sorted by memory, largest first.

**5.3** Report the number of CPUs and the three load averages. In one sentence, say whether
this machine is busy.
*Verify:* your sentence compares the load to the CPU count.

**5.4** How much memory could programs use right now without swapping? How much is
currently disk cache?
*Verify:* you quoted the `available` and `buff/cache` columns, not `free`.

**5.5** Find the signal numbers of `SIGHUP`, `SIGTERM`, `SIGKILL` and `SIGSTOP` without
looking them up online.
*Verify:* 1, 15, 9, 19.

**5.6** Find the PID of the main `sshd` process and print its full command line from
`/proc`.
*Verify:* you read `/proc/<pid>/cmdline`, and the NUL separators are turned into spaces.

---

## 🔵 Practical

**5.7** Start `sleep 600` in the foreground. Suspend it, list your jobs, continue it in the
background, bring it back to the foreground, then end it politely.
*Verify:* `jobs` showed it as `Stopped`, then `Running`; at the end `jobs` is empty.

**5.8** Start `sleep 900` so that it survives you logging out. Log out, log back in, and
find it.
*Verify:* `pgrep -a sleep` still shows it after reconnecting.

**5.9** Start `yes > /dev/null` with a niceness of 15. Then change it to 5.
*Verify:* `ps -o pid,ni,cmd -p <pid>` shows `15`, then `5` — and you needed `sudo` for
the second step. Why?

**5.10** Start two CPU hogs (`yes > /dev/null &` twice). Find them with `top` sorted by CPU,
and kill both with one command that would not touch anyone else's `yes`.
*Verify:* `pgrep -u "$USER" -x yes` prints nothing.

**5.11** Run `env LAB_COLOR=blue sleep 300 &`. Without asking the process, show that its
environment contains `LAB_COLOR=blue`.
*Verify:* the variable appears, read from `/proc`.

**5.12** Which process is listening on port 22, as which user? Show it two different ways.
*Verify:* both `ss` and `lsof` name the same PID.

**5.13** Create a zombie:

```bash
bash -c 'sleep 1 & exec sleep 300' &
```

Wait two seconds. Find the zombie and its parent, explain why it exists, and get rid of it.
*Verify:* `ps -eo stat | grep -c '^Z'` returns `0` at the end.

**5.14** Use `strace` to find exactly which files `ls /nonexistent` tries to access, and
then produce a per-syscall summary of `ls /etc`.
*Verify:* the first output shows the failing `statx`/`lstat`/`newfstatat` with `ENOENT`;
the second is a table.

**5.15** Enable `sysstat` data collection, and show the CPU utilisation sampled three
times at one-second intervals with `sar`.
*Verify:* `sar -u 1 3` prints three rows plus an Average line; `systemctl is-enabled sysstat`
says `enabled`.

**5.16** In one terminal start this load:

```bash
( while :; do :; done ) &
dd if=/dev/zero of=/var/tmp/lab05.bin bs=1M count=3000 oflag=direct status=none &
```

Using `vmstat`, `mpstat` and `pidstat`, determine which process saturates the CPU and
which one is responsible for I/O wait. Then stop both.
*Verify:* you can point at the column in each tool that told you; `rm /var/tmp/lab05.bin`.

**5.17** Show a process failing because of an open-files limit:

```bash
bash -c 'ulimit -n 50; python3 -c "f=[open(\"/etc/hosts\") for _ in range(100)]"'
```

Explain the error, then make the same command succeed by changing only the `ulimit`.
*Verify:* the first run raises `[Errno 24] Too many open files`; the second runs silently.

**5.18** Run a transient service with a custom open-files limit:

```bash
sudo systemd-run --unit=lab05-limits -p LimitNOFILE=1234 sleep 600
```

Prove the running process has that limit, using two different sources.
*Verify:* `/proc/<pid>/limits` and `systemctl show` both report 1234.

**5.19** Run a memory hog under a 50 MB cgroup limit:

```bash
sudo systemd-run --unit=lab05-mem -p MemoryMax=50M -p MemorySwapMax=0 \
  python3 -c 'x = bytearray(200 * 1024 * 1024); import time; time.sleep(60)'
```

Find out what happened to it, from systemd *and* from the kernel.
*Verify:* `systemctl status lab05-mem` reports `oom-kill`; the kernel log shows the kill.
Clean up with `sudo systemctl reset-failed lab05-mem`.

---

## 🔴 Challenge

**5.20 — Address already in use.** Start `python3 -m http.server 8080 &`, then try to start
a second one on the same port. Identify the process that owns the port — PID, user and
how long it has been running — then free the port.
*Verify:* the second server starts after you're done.

**5.21 — Graceful versus forced.** Write a script that creates `/tmp/lab05.lock`, traps
`SIGTERM` to print `cleaning up` and delete the lock, then sleeps. Show that `kill` leaves
no lock behind, and that `kill -9` does.
*Verify:* after `kill`, the lock is gone; after `kill -9`, it remains.

**5.22 — Reload, don't restart.** Send `sshd` the reload signal directly (not through
`systemctl`). Prove the main process kept its PID, and find the log line that shows it
reloaded.
*Verify:* PID unchanged; `journalctl -u ssh` shows `Received SIGHUP`.

**5.23 — Target is busy.** Mount a small `tmpfs` at `/mnt/lab05`, `cd` into it in your
second terminal, and try to unmount it from the first. Find out who is blocking the
unmount, and resolve it without killing your second shell.
*Verify:* `findmnt /mnt/lab05` prints nothing at the end.

**5.24 — Per-user priority.** Start three `sleep 500` processes as alice. Lower the
priority of *all* alice's processes to niceness 10 with one command.
*Verify:* `ps -o ni,user,cmd -u alice` shows 10 for every one.

**5.25 — Threads.** Which process on the system has the most threads? Show the count.
*Verify:* your command sorted by the `nlwp` (number of lightweight processes) column.

**5.26 — What is it waiting for?** Run:

```bash
mkfifo /tmp/lab05.fifo
cat /tmp/lab05.fifo > /dev/null &
```

`cat` hangs forever. Without reading this command, use `ps` and `strace` to find out
which system call it is blocked in and on which file. Then release it.
*Verify:* you can name the syscall and the path; after your fix, `jobs` shows it finished.

---

## Self-check

- [ ] I can read every column of `ps`, `top` and `vmstat` I use
- [ ] I can explain a high load average with idle CPUs
- [ ] I always try SIGTERM before SIGKILL, and know why
- [ ] I can find a process's limits, open files, environment and syscalls through `/proc` and `strace`
- [ ] I know limits for services go in the unit, not `limits.conf`
- [ ] I can triage a slow machine resource by resource, and look backwards in time with `sar`

➡ **Solutions:** [solutions/05-processes.md](../solutions/05-processes.md)
