# Mock exam 1

**20 tasks · 2 hours · pass mark 67%**

Rules — the same as the real exam:

- Set a timer for **2:00:00** and stop when it rings.
- Help allowed: `man`, `--help`, `/usr/share/doc`. **No browser, no notes, no course files.**
- Work from **ubuntu1** as your base; `ssh` to the host each task names; `exit` back before
  the next task.
- Every task is marked on the **final state**, including what would survive a reboot.

Score yourself with the [marking guide](mock-exam-1-solutions.md) afterwards.

---

## Setup — before you start the timer

Restore clean machines, then run the setup. Don't read the setup to find answers.

```bash
# on your host
vagrant snapshot restore ubuntu1 clean
vagrant snapshot restore ubuntu2 clean
vagrant snapshot restore rocky1 clean
vagrant ssh ubuntu1
```

On **ubuntu1** (re-create the SSH keys from the lab environment if the snapshot predates
them):

```bash
ssh ubuntu2 'sudo bash -s' <<'EOF'
set -e
apt-get update -qq && apt-get install -y -qq git nfs-kernel-server > /dev/null
mkdir -p /srv/git /srv/exports/shared /srv/examapp
git init -q --bare -b main /srv/git/exam.git
tmp=$(mktemp -d); git -C "$tmp" init -q -b main
printf '[server]\nport = 80\nworkers = 2\n' > "$tmp/app.conf"
git -C "$tmp" add app.conf
git -C "$tmp" -c user.name=setup -c user.email=setup@exam commit -qm "Initial config"
git -C "$tmp" push -q /srv/git/exam.git main
chown -R vagrant: /srv/git
echo shared-ok > /srv/exports/shared/README
echo '/srv/exports/shared 192.168.56.0/24(rw,sync,no_subtree_check)' >> /etc/exports
exportfs -ra
echo 'examapp ok' > /srv/examapp/index.html
EOF

ssh rocky1 'sudo bash -s' <<'EOF'
dnf install -y -q nginx policycoreutils-python-utils
mkdir -p /srv/exam && echo 'rocky exam' > /srv/exam/index.html
EOF

sudo bash -s <<'EOF'
apt-get update -qq && apt-get install -y -qq telnet nginx > /dev/null 2>&1
systemctl stop nginx
mkdir -p /var/log/exam
for i in 1 2 3; do echo "log $i" > /var/log/exam/app$i.log; done
head -c 2M /dev/urandom > /var/log/exam/big-recent.log
head -c 3M /dev/urandom > /var/log/exam/big-old.log
touch -d '30 days ago' /var/log/exam/big-old.log
mkdir -p /srv/exam-web && echo 'exam web' > /srv/exam-web/index.html
printf '#!/bin/bash\nfind /tmp -maxdepth 1 -name "exam-*" -mtime +1 -delete\n' > /usr/local/bin/cleanup.sh
chmod 755 /usr/local/bin/cleanup.sh
dd if=/dev/zero of=/var/tmp/holder.dat bs=1M count=300 status=none
nohup bash -c 'exec 3</var/tmp/holder.dat; exec -a examholder sleep 100000' > /dev/null 2>&1 &
sleep 1; rm -f /var/tmp/holder.dat
EOF
echo 'setup done'
```

**Start the timer now.**

---

## Essential Commands — 20%

**Task 1** · ubuntu1 · 3%

Find every regular file under `/var/log` that is larger than 1 MiB **and** was modified in
the last 7 days. Write their absolute paths, one per line, sorted alphabetically, to
`/root/bigrecent.txt`.

**Task 2** · ubuntu1 · 5%

The Git repository `ubuntu2:/srv/git/exam.git` holds an application config. As user
`vagrant`:

- clone it to `/home/vagrant/exam`,
- create a branch `fix-port`,
- on that branch change `port = 80` to `port = 8080` in `app.conf`,
- commit with the message `Change port to 8080`,
- push the branch to the remote.

**Task 3** · ubuntu2 · 5%

Create a systemd service `examapp.service` that runs
`/usr/bin/python3 -m http.server 9000 --directory /srv/examapp` as user `www-data`. It must
restart automatically if it fails, be running now, and start at boot.

**Task 4** · ubuntu1 · 4%

Create a 2048-bit RSA private key `/etc/ssl/exam/web.key` and a self-signed certificate
`/etc/ssl/exam/web.crt` for `web.exam.local`, valid for 90 days, with a Subject Alternative
Name of `DNS:web.exam.local`. The key must be readable only by root.

**Task 5** · ubuntu1 · 3%

A process is holding a deleted file that still occupies about 300 MB in `/var/tmp`. Free
the space **without stopping that process**. Write the PID of the process to
`/root/holder.pid`.

---

## Users and Groups — 10%

**Task 6** · ubuntu1 · 4%

- Create a group `auditors` with GID `4000`.
- Create a user `jdoe` with UID `2500`, a home directory, login shell `/bin/bash`, and
  supplementary group `auditors`.
- jdoe's password must expire every **45 days**, and the account must expire on
  **2026-12-31**.

**Task 7** · ubuntu1 · 3%

Give group `auditors` read-only access to `/var/log/exam` and every file in it — including
files created there in the future — without changing any owner, group or mode bits.

**Task 8** · ubuntu1 · 3%

Configure resource limits: members of `auditors` may run at most **50 processes** (hard
limit); user `jdoe` may open at most **1024 files** (soft and hard).

---

## Storage — 20%

**Task 9** · ubuntu1 · 8%

Using the whole of the spare disk `/dev/sdb`:

- create a volume group `vgexam`,
- create a logical volume `lvdata` of **1.5 GiB** with an **XFS** filesystem,
- mount it on `/data` persistently,
- then grow `lvdata` and its filesystem by **500 MiB** without unmounting it.

**Task 10** · ubuntu1 · 5%

Add a **512 MiB** swap file `/swap.exam`. It must be active now and after a reboot.

**Task 11** · ubuntu1 · 7%

ubuntu2 exports `/srv/exports/shared` over NFS. Configure **autofs** on ubuntu1 so that
accessing `/mnt/auto/shared` mounts it automatically, read-write, and it is unmounted after
**60 seconds** of inactivity.

---

## Networking — 25%

**Task 12** · ubuntu2 · 6%

Persistently, on ubuntu2's lab interface (the one holding `192.168.56.12`):

- add the IPv4 address `192.168.56.112/24`,
- add the IPv6 address `fd00:56::12/64`,
- add a static route to `10.50.0.0/16` via `192.168.56.13`.

The existing address and connectivity must keep working.

**Task 13** · ubuntu1 · 6%

Create a bond `bond0` from the spare interfaces `eth2` and `eth3` in **active-backup** mode,
with the address `10.99.0.11/24`, persistently.

**Task 14** · ubuntu2 · 7%

Configure a persistent **nftables** firewall on ubuntu2:

- incoming traffic is dropped by default,
- allow established/related traffic, loopback, ICMP and SSH,
- allow TCP port `9000` only from `192.168.56.0/24`,
- connections to TCP port `80` must be redirected to port `9000`.

**Task 15** · ubuntu1 · 6%

Configure nginx on ubuntu1 as a reverse proxy: requests for `http://exam.local/` must be
served by `http://192.168.56.12:9000/`, passing the client address in the
`X-Forwarded-For` header. nginx must be running and enabled. (`exam.local` may resolve to
ubuntu1 via `/etc/hosts`.)

---

## Operations Deployment — 25%

**Task 16** · ubuntu1 · 5%

- Set `vm.swappiness` to `20` and enable IPv4 forwarding — both persistently.
- Make the `nbd` kernel module load at boot with the option `nbds_max=4`.

**Task 17** · ubuntu1 · 5%

Run `/usr/local/bin/cleanup.sh` every day at **02:15** using a **systemd timer** named
`cleanup.timer`. If the machine was off at that time, the job must run at the next boot.

**Task 18** · ubuntu1 · 4%

- Install the package that provides the `dig` command.
- Remove the `telnet` package completely.
- Prevent the `curl` package from being upgraded.

**Task 19** · rocky1 · 6%

Configure nginx on rocky1 to serve the content of `/srv/exam` on TCP port **8088** (as the
document root of a server listening on that port). SELinux must remain **enforcing**, and
the configuration must survive a reboot and a filesystem relabel. Open the port in the
firewall.

**Task 20** · ubuntu1 · 5%

Run a container named `exam-web` from the image `nginx:1.27`, publishing container port 80
on host port **8085**, serving `/srv/exam-web` from the host **read-only** as nginx's
document root, and restarting automatically unless explicitly stopped. Install a container
engine if needed.

---

**Stop.** Spend your last ten minutes on a verification sweep — then compare with the
[marking guide](mock-exam-1-solutions.md).
