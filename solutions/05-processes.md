# Solutions 05 — Processes and performance

---

## 🟢 Warm-up

**5.1**

```bash
echo "PID $$  PPID $PPID"
pstree -ps $$
```

`-s` shows the ancestors of the given PID; `-p` adds PIDs. Over SSH the chain is
`systemd → sshd → sshd → sshd → bash`: the listener, the per-connection privileged
process, the unprivileged session process, and your shell.

**5.2**

```bash
ps -eo pid,user,%mem,rss,cmd --sort=-%mem | head -6
```

`rss` is resident memory in KiB — what the process actually has in RAM right now.

**5.3**

```bash
nproc
uptime
```

"Load 0.15 on 2 CPUs — idle." Always state the load relative to `nproc`.

**5.4**

```bash
free -h
```

**5.5**

```bash
kill -l
kill -l TERM KILL HUP STOP      # prints the numbers
man 7 signal
```

**5.6**

```bash
pgrep -a sshd
pid=$(pgrep -o -x sshd)          # -o: the oldest, i.e. the listener
tr '\0' ' ' < /proc/$pid/cmdline; echo
```

---

## 🔵 Practical

**5.7**

```bash
sleep 600
# Ctrl-Z
jobs            # [1]+ Stopped  sleep 600
bg %1
jobs            # [1]+ Running  sleep 600 &
fg %1
# Ctrl-C   (SIGINT — the polite interactive interrupt)
```

**5.8**

```bash
nohup sleep 900 > /dev/null 2>&1 &
exit
# reconnect
pgrep -a sleep
```

Alternatives: `setsid sleep 900 &` (new session, no controlling terminal), or start it in
`tmux`. Note that on systems where systemd-logind has `KillUserProcesses=yes`, even
`nohup`ed processes die at logout — Ubuntu and RHEL default to `no`. For anything that
matters, use a service (chapter 06) or `systemd-run --user`.

**5.9**

```bash
nice -n 15 yes > /dev/null &
ps -o pid,ni,cmd -p $!
renice -n 5 -p $!          # Permission denied
sudo renice -n 5 -p $!
ps -o pid,ni,cmd -p $!
kill $!
```

Unprivileged users may only make their processes *nicer* (raise the number). Lowering it —
asking for more priority — needs root, or an `RLIMIT_NICE` granted through `limits.conf`.

**5.10**

```bash
yes > /dev/null & yes > /dev/null &
top                        # press P, then q
pkill -u "$USER" -x yes
```

`-x` matches the exact process name; `-u` restricts to your processes.

**5.11**

```bash
env LAB_COLOR=blue sleep 300 &
tr '\0' '\n' < /proc/$!/environ | grep LAB_COLOR
```

Note it shows the environment *at start*. A process that changes its own variables later
doesn't update this file.

**5.12**

```bash
sudo ss -tlnp 'sport = :22'
sudo lsof -i :22 -sTCP:LISTEN
```

`-t` TCP, `-l` listening, `-n` numeric, `-p` process — `sudo` is needed to see other
users' processes. On Ubuntu with socket activation you may see `systemd` holding port 22
as well as, or instead of, `sshd` — systemd owns the listening socket and passes it to
`sshd`. That's chapter 06's socket units at work.

**5.13**

```bash
bash -c 'sleep 1 & exec sleep 300' &
sleep 2
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/'
```

```text
 5120  5119 Z    [sleep] <defunct>
```

```bash
ps -o pid,cmd -p 5119            # the parent: "sleep 300"
kill 5119
ps -eo stat | grep -c '^Z'       # 0
```

`bash` started `sleep 1` as a child, then `exec` replaced bash with `sleep 300` — same PID,
but the new program has no idea it has a child and never calls `wait()`. When `sleep 1`
exits it stays a zombie. Killing the parent hands the zombie to PID 1, which reaps it
immediately. Killing the zombie itself does nothing — it is already dead.

**5.14**

```bash
strace ls /nonexistent 2>&1 | grep nonexistent
strace -c ls /etc > /dev/null
```

```text
statx(AT_FDCWD, "/nonexistent", AT_STATX_SYNC_AS_STAT|AT_SYMLINK_NOFOLLOW, …) = -1 ENOENT
```

**5.15**

```bash
sudo sed -i 's/^ENABLED=.*/ENABLED="true"/' /etc/default/sysstat
sudo systemctl enable --now sysstat
sar -u 1 3
```

Live sampling (`sar -u 1 3`) works without collection enabled; enabling it is what makes
`sar -u` with no interval show *history* tomorrow.

**5.16**

```bash
vmstat 1 5          # high "us" from the loop; high "wa" and "b" from dd
mpstat -P ALL 1 3   # one CPU near 100% usr: a single-threaded hog
pidstat 1 3         # the bash loop at ~100% CPU
pidstat -d 1 3      # dd with large kB_wr/s
kill %1 %2
rm /var/tmp/lab05.bin
```

`oflag=direct` bypasses the page cache, so `dd` really waits for the disk — without it
the writes land in RAM and the I/O wait would hide behind the cache.

**5.17**

```bash
bash -c 'ulimit -n 50; python3 -c "f=[open(\"/etc/hosts\") for _ in range(100)]"'
# OSError: [Errno 24] Too many open files: '/etc/hosts'
bash -c 'ulimit -n 200; python3 -c "f=[open(\"/etc/hosts\") for _ in range(100)]"'
```

Errno 24 is `EMFILE`: the process hit its per-process descriptor limit. Running it inside
`bash -c` keeps the reduced limit out of your interactive shell — a soft limit, once
lowered, can be raised again only up to the hard limit, and a lowered *hard* limit can't
be raised at all without root.

**5.18**

```bash
pid=$(systemctl show -p MainPID --value lab05-limits)
grep 'open files' /proc/$pid/limits
systemctl show lab05-limits -p LimitNOFILE
sudo systemctl stop lab05-limits
```

`systemd-run` creates a *transient* unit — a real service with all the normal unit
properties, without writing a file. Very handy for experiments like this.

**5.19**

```bash
systemctl status lab05-mem          # Active: failed (Result: oom-kill)
journalctl -k | grep -i -E 'memory cgroup out of memory|killed process' | tail -3
sudo systemctl reset-failed lab05-mem
```

The kill came from the **cgroup's** memory limit, not from the machine running out of
memory: plenty of RAM was free. The kernel log line says `Memory cgroup out of memory`,
which distinguishes it from a system-wide OOM. The same mechanism kills containers and
Kubernetes pods that exceed their memory limit — that's the familiar `OOMKilled`.

---

## 🔴 Challenge

**5.20**

```bash
python3 -m http.server 8080 &
python3 -m http.server 8080         # OSError: [Errno 98] Address already in use
sudo ss -tlnp 'sport = :8080'
ps -o pid,user,etime,cmd -p <pid>
kill <pid>
python3 -m http.server 8080 &       # now starts
```

`etime` is elapsed time since start — useful for "has this been running since before the
deploy?"

**5.21**

```bash
cat > ~/lab05/graceful.sh <<'EOF'
#!/usr/bin/env bash
touch /tmp/lab05.lock
trap 'echo cleaning up; rm -f /tmp/lab05.lock; exit 0' TERM
while :; do sleep 1; done
EOF
chmod +x ~/lab05/graceful.sh

~/lab05/graceful.sh & sleep 1; kill $!;    sleep 2; ls /tmp/lab05.lock   # gone
~/lab05/graceful.sh & sleep 1; kill -9 $!; sleep 1; ls /tmp/lab05.lock   # still there
```

The loop uses `sleep 1` deliberately: bash runs a trap only between commands, so a single
`sleep 3600` would delay the cleanup until the sleep ended. That's also why a stale lock
from a `kill -9` is a real production problem — the next start sees the lock and refuses.

**5.22**

```bash
pid=$(systemctl show -p MainPID --value ssh)
sudo kill -HUP "$pid"
systemctl show -p MainPID --value ssh          # same number
journalctl -u ssh --since '-2min' | grep -i sighup
```

sshd re-executes itself on SIGHUP, keeping its PID and existing sessions. `systemctl
reload ssh` does exactly this — `ExecReload=` in the unit sends the HUP. Not every daemon
reloads on HUP; check the man page or the unit's `ExecReload=`.

**5.23**

```bash
sudo mkdir -p /mnt/lab05
sudo mount -t tmpfs -o size=10M tmpfs /mnt/lab05
# second terminal: cd /mnt/lab05
sudo umount /mnt/lab05            # target is busy
sudo fuser -vm /mnt/lab05         # shows the bash with access "c" (current directory)
sudo lsof +D /mnt/lab05
# second terminal: cd ~
sudo umount /mnt/lab05
```

The right fix is to make the user leave. `umount -l` (lazy) also works — it detaches the
mount immediately and finishes when the last user leaves — but processes keep reading a
filesystem nobody can see, which can surprise you later.

**5.24**

```bash
for i in 1 2 3; do sudo -u alice sleep 500 & done
sudo renice -n 10 -u alice
ps -o ni,user,cmd -u alice
```

**5.25**

```bash
ps -eo nlwp,pid,cmd --sort=-nlwp | head -3
```

**5.26**

```bash
ps -o pid,stat,wchan:30,cmd -C cat        # S state, waiting in a fifo-related kernel function
sudo strace -p "$(pgrep -n -x cat)"       # openat(AT_FDCWD, "/tmp/lab05.fifo", O_RDONLY  <blocked>
echo done > /tmp/lab05.fifo               # a writer arrives; open() returns, cat reads and exits
```

Opening a FIFO for reading blocks until something opens it for writing. `strace -p` shows
the syscall it's sitting in — often the fastest answer to "why is this hung?". `wchan`
names the kernel function it's sleeping in, which works even when you can't attach
`strace`.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | Running/runnable, interruptible sleep, uninterruptible sleep (usually I/O), zombie, stopped | |
| 2 | It is blocked inside the kernel and handles no signals until the I/O completes (NFS waits are a killable exception) | Investigate the storage / NFS server it waits on |
| 3 | Getting worse (1-minute > 15-minute); 8 on 4 CPUs is twice the capacity — overloaded now | Compare with `nproc` |
| 4 | Processes blocked on I/O (D state count toward load); high `wa` in `top` | Linux load includes D state |
| 5 | **No** | `available` counts reclaimable cache |
| 6 | Make the parent `wait()` for it, or kill the parent so PID 1 reaps it | The zombie is already dead |
| 7 | `SIGKILL` and `SIGSTOP` | |
| 8 | TERM lets the program flush data, release locks and exit cleanly | KILL can leave corruption and stale locks |
| 9 | `SIGHUP` | `systemctl reload` usually sends it |
| 10 | -20 to 19; -20 is highest; only root may lower niceness | |
| 11 | `ionice` (e.g. `-c3`) | `nice` affects CPU scheduling only |
| 12 | `tr '\0' '\n' < /proc/4242/environ` | NUL-separated |
| 13 | Services don't pass through PAM; set `LimitNOFILE=` in the unit | `limits.conf` is applied by `pam_limits` at login |
| 14 | Open files with zero links — deleted but still held open | Disk full while `du` finds nothing |
| 15 | `strace` | |
| 16 | `sar` with a time window (`-s`, `-e`); `sysstat` collection must have been enabled | `top` has no history |
| 17 | Killed by SIGKILL (128 + 9) — often the OOM killer | Check `journalctl -k` |
| 18 | `sshd` — possibly the one serving your own session | `pkill` matches substrings; use `pgrep -a` first and `-x` |
