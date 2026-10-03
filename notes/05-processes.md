# Chapter 05 — Processes and performance

When a server is slow, something is consuming a resource. This chapter is about finding
out *what*, *which resource*, and *why* — and then stopping it without making things worse.

LFCS: *Operations Deployment* (diagnose and manage processes) and *Essential Commands*
(monitor performance, determine application constraints).

## Objectives

After this chapter you can:

- Read `ps` and `top` output field by field, including process states.
- Explain load average correctly, including why it can be high with an idle CPU.
- Send the right signal, and explain why `kill -9` is the last resort.
- Find a process's open files, network sockets, limits and environment through `/proc`.
- Find the disk space held by deleted files.
- Watch a process's system calls with `strace`.
- Control priority with `nice`/`ionice` and limits with `ulimit` and systemd.
- Triage a slow machine resource by resource, including what happened hours ago with `sar`.
- Determine every limit and constraint acting on a running application.

---

## 1. What a process is

A process is a running program plus its state: memory, open files, current directory,
user and group IDs, environment, and limits. Each has a **PID** and a parent (**PPID**).

Every process is created by another one forking. PID 1 — `systemd` on modern systems —
is the ancestor of everything else, and adopts any process whose parent dies.

```bash
pstree -p | head -20        # the family tree with PIDs
ps -o pid,ppid,user,cmd -p $$   # $$ is your shell's own PID
```

---

## 2. ps — a snapshot

Two syntaxes exist for historical reasons. Learn one of each:

```bash
ps aux             # BSD style: every process, with CPU/MEM
ps -ef             # System V style: every process, with PPID
```

What you will actually type most is a custom format:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd --sort=-%mem | head
ps -o pid,stat,wchan:20,cmd -p 1234     # what is this PID waiting on?
ps -C nginx -o pid,cmd                  # by command name
ps -u alice                             # by user
```

### Process states — the STAT column

| State | Meaning | What to think |
|---|---|---|
| `R` | running or runnable | using, or waiting for, a CPU |
| `S` | interruptible sleep | waiting for something — normal for most processes |
| `D` | **uninterruptible sleep** | stuck in the kernel, usually on I/O — cannot be killed |
| `T` | stopped | paused by a signal (Ctrl-Z) or a debugger |
| `Z` | zombie | finished, but its parent hasn't collected its exit status |
| `I` | idle kernel thread | ignore |

Extra characters after the letter: `s` session leader, `+` foreground, `<` high
priority, `N` low priority, `l` multi-threaded.

### D state — the one that matters

A process in `D` is waiting inside the kernel, almost always for storage or a network
filesystem. A true uninterruptible sleep **ignores every signal, including SIGKILL**, until
the I/O completes. Some kernel waits — NFS among them — use a *killable* variant that also
shows as `D` but does respond to SIGKILL. Either way, many `D` processes at once usually
means a dead NFS server or a failing disk: the fix is the storage, not the processes.

### Zombies

A zombie has already exited. It uses no CPU or memory; it is just an entry in the
process table holding an exit code nobody has read. You cannot kill it — it is already
dead. You fix it by making its **parent** collect it, or by killing the parent so
PID 1 adopts and reaps it.

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'     # find zombies and their parents
```

A handful is harmless. Thousands mean a buggy parent, and eventually the PID space runs
out.

---

## 3. top — live view

```bash
top
```

| Key | Does |
|---|---|
| `P` / `M` / `T` | sort by CPU / memory / time |
| `1` | show each CPU separately |
| `c` | show full command lines |
| `k` | kill a PID |
| `r` | renice a PID |
| `H` | show threads |
| `u` | filter by user |
| `q` | quit |

The CPU line is the most informative line on the screen:

```text
%Cpu(s): 12.0 us,  3.0 sy,  0.0 ni, 60.0 id, 25.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

| Field | Means | High value suggests |
|---|---|---|
| `us` | user code | the application is busy computing |
| `sy` | kernel code | many syscalls, context switches |
| `wa` | waiting for I/O | **storage is the bottleneck, not CPU** |
| `st` | stolen by the hypervisor | a noisy neighbour on your VM host |

`htop` is friendlier if installed. `top` is always there.

---

## 4. Load average

```bash
uptime
 10:14:22 up 3 days,  2 users,  load average: 4.12, 2.80, 1.20
```

The three numbers are averages over 1, 5 and 15 minutes of the number of processes
that are **runnable (R) or in uninterruptible sleep (D)**.

Interpret it against your CPU count (`nproc`):

- Load 4 on 8 CPUs — half busy.
- Load 4 on 1 CPU — four processes queued for one CPU.
- Load rising (1-min > 15-min) — it is getting worse; falling — recovering.

Because Linux counts `D` processes, **a high load with an idle CPU** is common and
means processes are blocked on I/O. Check `wa` in `top` and the `D` count in `ps` before
blaming CPU.

---

## 5. Memory

```bash
free -h
               total   used   free   shared  buff/cache   available
Mem:           1.9Gi  612Mi  158Mi    12Mi       1.1Gi       1.1Gi
Swap:          2.0Gi    0Bi  2.0Gi
```

Read **`available`**, not `free`. Linux deliberately fills unused RAM with disk cache
(`buff/cache`) and gives it back the moment a program needs it. Low `free` with healthy
`available` is a well-run system, not a problem.

Real memory pressure shows as: `available` near zero, swap in active use, and `si`/`so`
non-zero in `vmstat`.

```bash
vmstat 1 5          # every second, five times
```

| Column | Means |
|---|---|
| `r` | runnable processes |
| `b` | processes in D state |
| `si` / `so` | swapping in / out — sustained non-zero is bad |
| `wa` | CPU waiting for I/O |

### The OOM killer

When the system runs out of memory completely, the kernel picks a process and kills it
with SIGKILL. The victim cannot log anything. You find out from the kernel log:

```bash
journalctl -k | grep -i -E 'out of memory|killed process'
dmesg -T | grep -i oom
```

---

## 6. Signals

A signal is a small message to a process. Most can be caught, ignored, or handled.

| Signal | Number | Default | Typical use |
|---|---|---|---|
| `SIGHUP` | 1 | terminate | daemons: reload configuration |
| `SIGINT` | 2 | terminate | Ctrl-C |
| `SIGQUIT` | 3 | core dump | Ctrl-\ |
| `SIGKILL` | 9 | terminate | **cannot be caught or ignored** |
| `SIGTERM` | 15 | terminate | polite request to exit — the default for `kill` |
| `SIGSTOP` | 19 | stop | **cannot be caught** |
| `SIGCONT` | 18 | continue | resume a stopped process |
| `SIGTSTP` | 20 | stop | Ctrl-Z |

```bash
kill 1234              # SIGTERM
kill -HUP 1234         # reload
kill -9 1234           # SIGKILL
kill -l                # list all signals
pkill -u alice sleep   # by name and user
pgrep -a nginx         # find PIDs with their command lines
killall -s HUP nginx   # every process named nginx
```

### Why SIGTERM first

SIGTERM lets the program clean up: flush buffers, finish writing a file, release locks,
remove its PID file, close database connections cleanly. SIGKILL gives it no chance —
you can leave behind a corrupt file, a stale lock that stops the next start, or a
half-applied transaction.

Order of escalation: **TERM → wait a few seconds → KILL**. Which is exactly what
`systemctl stop` does for you.

---

## 7. Jobs — the shell's own processes

```bash
sleep 300 &        # run in background
jobs               # list this shell's jobs
fg %1              # bring job 1 to foreground
# Ctrl-Z           # stop the foreground job
bg %1              # resume it in background
kill %1            # signal by job number
```

When you log out, your shell sends SIGHUP to its jobs and they die. To keep something
running:

```bash
nohup long-task.sh > task.log 2>&1 &    # ignore SIGHUP from the start
disown %1                               # detach a job already running
```

For anything that matters, use `tmux`/`screen` or — properly — a systemd service
(chapter 06).

---

## 8. /proc — everything about a process

Every process has a directory `/proc/<PID>/`. The tools above mostly just read it.

| Path | Contains |
|---|---|
| `cmdline` | full command line, NUL-separated |
| `environ` | environment variables, NUL-separated |
| `cwd` | link to its current directory |
| `exe` | link to the binary |
| `fd/` | one link per open file descriptor |
| `limits` | its resource limits |
| `status` | state, memory, UIDs, threads, capabilities |

```bash
tr '\0' '\n' < /proc/1234/environ       # its environment
ls -l /proc/1234/fd                      # its open files
cat /proc/1234/limits                    # what it is allowed
ls -l /proc/1234/exe                     # which binary — even if deleted
```

`/proc/<PID>/exe` still points at the binary after it has been deleted or upgraded on
disk, shown as `(deleted)`. That is how you find processes running an old version
after a package update.

---

## 9. lsof — open files

On Linux everything is a file, including network sockets, so `lsof` answers a lot.

```bash
lsof -p 1234                 # everything a process has open
lsof /var/log/syslog         # who has this file open
lsof +D /mnt/data            # anything open under a directory
lsof -i :443                 # what is using port 443
lsof -u alice                # everything alice's processes have open
```

### The deleted-but-open file

The classic: `df` says the disk is full, `du` cannot find what is using it.

A process still has a file open after someone deleted it. The name is gone, so `du`
cannot see it, but the space is not freed until the last descriptor closes.

```bash
lsof +L1                     # open files with zero links — i.e. deleted
```

Fix it by restarting (or signalling to reopen logs) the process that holds it. In an
emergency you can truncate it through `/proc` without restarting:

```bash
: > /proc/1234/fd/7
```

The `fuser` command is a shorter way to ask "who is using this":

```bash
fuser -vm /mnt/data          # which processes use this mount — why umount says busy
```

---

## 10. strace — what is it doing?

When a process hangs or fails with a vague error, watch its system calls:

```bash
strace -p 1234                         # attach to a running process
strace -f -o trace.txt ./program       # trace a program and its children, to a file
strace -e trace=openat,read ls /tmp    # only some syscalls
strace -c ls                           # summary: count and time per syscall
```

Most useful pattern — "which file is it failing to find?":

```bash
strace -f -e trace=openat,stat,access ./program 2>&1 | grep ENOENT
```

A process stuck on one `read(` or `futex(` line is waiting — the question becomes what
it is waiting for.

---

## 11. Priority and limits

### CPU priority — nice

Niceness runs from **-20 (highest priority)** to **19 (lowest)**. Default is 0. Only root
can lower niceness (raise priority).

```bash
nice -n 10 ./backup.sh        # start with lower priority
renice -n 15 -p 1234          # change a running process
```

### I/O priority — ionice

```bash
ionice -c3 -p 1234            # idle class: only uses disk when nothing else wants it
ionice -c2 -n7 tar czf …      # best-effort, lowest level
```

A backup job reading the whole disk can starve a database even with the CPU idle.
`nice` does nothing for that; `ionice` does.

### Resource limits — ulimit

```bash
ulimit -a           # all limits for this shell
ulimit -n           # max open files (soft)
ulimit -Hn          # hard limit
ulimit -n 4096      # raise soft limit up to the hard limit
```

Set persistently for logins in `/etc/security/limits.conf` or `/etc/security/limits.d/`:

```text
alice   soft   nofile   8192
alice   hard   nofile   16384
```

**Services ignore `limits.conf`.** It is applied by PAM at login, and systemd services do
not log in. For a service, set it in the unit:

```ini
[Service]
LimitNOFILE=65536
```

"Too many open files" on a service whose `limits.conf` you already raised is this
gotcha, every time.

### cgroups — limits on groups of processes

systemd puts every service in its own control group, which is how it can limit and
account for a service's total CPU and memory:

```bash
systemd-cgtop                         # live resource use per cgroup
systemd-cgls                          # the tree
systemctl set-property nginx MemoryMax=512M CPUQuota=50%
```

This is the same mechanism containers use. A container is, largely, a process in its
own cgroups and namespaces.

---

## 12. Performance triage

When "the server is slow", check each resource for **utilisation, saturation and
errors** (the USE method) instead of guessing. The first minute:

```bash
uptime                       # load trend: rising or falling?
dmesg -T | tail              # kernel errors: OOM kills, disk errors, dropped packets
vmstat 1 5                   # r (CPU queue), b (D-state), si/so (swap), wa (I/O wait)
mpstat -P ALL 1 3            # per-CPU: one CPU at 100% = a single-threaded bottleneck
pidstat 1 3                  # per-process CPU
pidstat -d 1 3               # per-process disk I/O
iostat -xz 1 3               # per-disk utilisation and latency (chapter 09)
free -h                      # memory: read "available"
ss -s                        # socket summary: thousands in TIME_WAIT or SYN-RECV?
```

`mpstat`, `pidstat`, `iostat` and `sar` come in the `sysstat` package.

| Resource | Utilisation | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat` `%usr`+`%sys` | `vmstat` `r` > CPU count; load average | — |
| Memory | `free` used vs available | `vmstat` `si`/`so`; OOM kills | `dmesg` |
| Disk | `iostat -x` `%util` | `iostat -x` `aqu-sz`, `await` | `dmesg`, `smartctl` |
| Network | `sar -n DEV` throughput | `ss -s`, `netstat -s` retransmits | `ip -s link` errors/drops |

### History — sar

`top` only shows *now*. When someone says "it was slow at 3am", you need history. With
`sysstat` enabled, `sar` records every 10 minutes:

```bash
sudo systemctl enable --now sysstat
sar -u                 # CPU today
sar -r                 # memory
sar -q                 # load and run queue
sar -d                 # disks
sar -n DEV             # network
sar -u -s 03:00:00 -e 04:00:00     # a time window
```

On Ubuntu, collection is off until `ENABLED="true"` is set in `/etc/default/sysstat`
(some releases enable it through the systemd timer alone — check `systemctl status sysstat`
and whether files appear in `/var/log/sysstat/`).

---

## 13. What constrains an application?

The LFCS asks you to "determine application and service specific constraints" — in
practice: *why can't this program do what it's trying to do?* The candidates, and where
each is visible:

| Constraint | How to see it |
|---|---|
| Resource limits of the running process | `cat /proc/<pid>/limits` |
| Limits set by its systemd unit | `systemctl show nginx -p LimitNOFILE -p MemoryMax -p CPUQuota -p TasksMax` |
| cgroup usage against those limits | `systemd-cgtop`, `systemctl status nginx` (shows Memory and Tasks) |
| The user it runs as | `ps -o user,group,cmd -p <pid>` |
| Ports — already taken, or below 1024 | `ss -tlnp`, capabilities: `grep Cap /proc/<pid>/status` |
| Files it can't open | `strace -f -e trace=openat -p <pid>` — look for `EACCES`, `ENOENT` |
| Mandatory access control | `ausearch -m avc` (SELinux), `dmesg \| grep apparmor` |
| Kernel-wide limits | `sysctl fs.file-max`, `kernel.pid_max`, `net.core.somaxconn` |

Work from the error message: *Too many open files* → limits; *Address already in use* →
ports; *Permission denied* on a file that looks fine → MAC or a path component (chapter 04);
a process killed with no error → memory limit or OOM.

```bash
pid=$(pgrep -o nginx)
cat /proc/$pid/limits | grep -E 'open files|processes'
systemctl show nginx -p LimitNOFILE -p MemoryMax -p TasksMax
ls /proc/$pid/fd | wc -l           # how many it actually has open
```

---

## 14. Gotchas

**`kill -9` on a `D` process usually does nothing.** It is waiting in the kernel. (NFS
waits are the exception — killable — but new processes will hang the same way.) Fix the
storage or network filesystem it is waiting for.

**Killing a zombie does nothing.** Deal with its parent.

**`free` looks alarming on a healthy system.** Read `available`.

**`limits.conf` doesn't apply to services.** Use `LimitNOFILE=` in the unit.

**`nohup` output goes somewhere.** Without redirection it writes `nohup.out` in the
current directory, which has filled disks.

**`pkill` matches substrings.** `pkill ssh` also kills `sshd` — including the one serving
your session. Use `pgrep -a` first to see what would match, and `-x` for exact names.

---

## 15. Commands introduced

| Command | Purpose |
|---|---|
| `ps`, `pstree`, `pgrep` | list and find processes |
| `top`, `htop` | live process view |
| `uptime`, `nproc` | load average, CPU count |
| `free`, `vmstat` | memory and system activity |
| `kill`, `pkill`, `killall` | send signals |
| `jobs`, `fg`, `bg`, `nohup`, `disown` | job control |
| `lsof`, `fuser` | open files and who holds them |
| `strace` | trace system calls |
| `nice`, `renice`, `ionice` | CPU and I/O priority |
| `ulimit` | per-process limits |
| `systemd-cgtop`, `systemd-cgls` | cgroup view |
| `dmesg`, `journalctl -k` | kernel messages, including OOM kills |
| `mpstat`, `pidstat`, `sar` | per-CPU, per-process and historical statistics (sysstat) |
| `systemctl show -p` | a unit's effective limits |

---

➡ **Next:** [quiz](../quizzes/05-processes.md) → [lab](../labs/05-processes.md)
