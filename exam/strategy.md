# Exam strategy

Knowing Linux and passing a performance-based exam are related but different skills. This
page is about the second: how to spend two hours so that what you know turns into marks.

Check the current rules on the Linux Foundation site before booking — details below were
correct when written (2026-10) and do change.

---

## The format, and what it means for you

| Fact | Consequence |
|---|---|
| 17–20 tasks in 2 hours | **~6 minutes per task** on average. Some take 2, some 15 |
| 67% to pass | you can drop several tasks entirely and still pass |
| Marked by scripts that inspect the machines afterwards | only the **final state** counts — not your commands, not your intent |
| Tasks done on named hosts via `ssh` | being on the wrong host scores zero for that task |
| `man` pages and `/usr/share/doc` only | everything in this course's "offline references" lines |
| Tasks have different weights (shown per task) | a 7% task deserves more time than a 2% one |
| Partial credit is often possible | finishing 3 of 4 sub-steps is worth something |

---

## Before the exam

- **Sit both mock exams under real conditions**: a 2-hour timer, no notes, no browser,
  `man` only. Score them honestly with the marking guide. Under 75%? Find the weak domain
  in the syllabus table and redo its 🔴 tasks.
- **Practise in the exam's terminal style**: one shell, `ssh <host>` to each machine,
  `sudo -i` for root, `exit` back to base. Never nest SSH sessions.
- **Know the editor cold.** Pick `vim` or `nano` and don't think about it on the day.
  In `vim`: `:set paste` before pasting, `u` undo, `/` search, `:wq`.
- **Know your man page searches**: `man -k keyword`, `/pattern` then `n`, section 5 for
  file formats. Know which man pages hold the examples you'll want: `man nmcli-examples`,
  `man netplan`, `man exports`, `man 5 crontab`, `man systemd.timer`, `/usr/share/doc/`.
- Run the Linux Foundation's system check on the exact computer, network and room you'll use.

---

## During the exam

### First five minutes

Read the whole task list quickly. Note for each: weight, host, and a gut feel —
**easy / medium / hard**. Don't solve anything yet.

### Three passes

1. **Pass 1 — the easy ones.** Anything you can do in under 5 minutes without thinking.
   This banks points and calms you down.
2. **Pass 2 — the medium ones.** Highest weight first.
3. **Pass 3 — the hard ones**, with whatever time remains.

Use the exam's flag feature to mark tasks to return to. **Never spend more than ~12 minutes
stuck on one task** — flag it and move on. Coming back fresh often makes it obvious.

### For every single task

```text
1. Read the task twice. Underline: host, names, paths, ports, "persistent", "at boot".
2. ssh <host>                 — check the prompt shows the right host
3. sudo -i                    — if the task needs root
4. Do it.
5. VERIFY it — the state the marker will check, including after a reboot.
6. exit back to base.
```

Step 5 is where this course's Verify habit pays off. The marker checks:

| You did | The marker also checks | So verify with |
|---|---|---|
| started a service | enabled at boot | `systemctl is-active X && systemctl is-enabled X` |
| mounted a filesystem | `/etc/fstab` entry works | `findmnt --verify && mount -a` |
| added swap | fstab entry | `swapoff -a && swapon -a && swapon --show` |
| set a sysctl | a file in `/etc/sysctl.d/` | `sysctl --system \| grep key` |
| added a firewall rule | it's saved / permanent | `firewall-cmd --list-all --permanent`, `nft list ruleset` after `systemctl restart nftables` |
| changed network config | it's in Netplan / nmcli | `netplan get`, `nmcli con show` |
| created a user | exact UID/groups/shell asked | `id user`, `getent passwd user` |
| set a boolean | `-P` | `semanage boolean -l \| grep name` |
| created a cron job | right user, right file | `crontab -l -u user`, `cat /etc/cron.d/…` |
| changed a config file | the service actually uses it | reload/restart, then check its behaviour |

### Things that cost marks for no reason

- Doing the task on the **wrong host**.
- **Typos in names** — `webserver` vs `web-server`, `/srv/data` vs `/srv/Data`. Copy names
  from the task text.
- Leaving a service **stopped** after changing its config, or not **reloading** it.
- **Runtime-only** changes: `ip addr add`, `sysctl -w`, `firewall-cmd` without
  `--permanent`, `setsebool` without `-P`.
- Breaking SSH on a host — then every later task on that host is lost. Keep the change
  minimal and test from base before moving on.
- **Rebooting the base host** — the instructions forbid it.
- Spending 25 minutes on a 3% task.

### When something won't work

1. Read the error message, all of it.
2. Check the service's journal: `journalctl -xeu <unit>`.
3. Check the config's own test: `nginx -t`, `sshd -t`, `named-checkconf`, `visudo -c`,
   `findmnt --verify`, `netplan generate`, `nft -c -f`.
4. SELinux/AppArmor? `ausearch -m AVC -ts recent`, `journalctl -k | grep -i denied`.
5. Still stuck after a few minutes? Flag it and move on.

---

## Time budget

| Time | Do |
|---|---|
| 0:00–0:05 | read everything, classify |
| 0:05–0:45 | easy tasks |
| 0:45–1:35 | medium tasks |
| 1:35–1:50 | hard tasks / flagged tasks |
| 1:50–2:00 | **verification sweep** — re-check persistence on every task you completed |

The final sweep catches more lost marks than any amount of extra solving.

---

## After the exam

Results arrive by email, usually within a day or two. If you don't pass, the free retake is
included — and you now know exactly which domains to strengthen.

---

➡ [Mock exam 1](mock-exam-1.md) · [Mock exam 2](mock-exam-2.md)
