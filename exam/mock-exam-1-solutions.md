# Mock exam 1 — marking guide

For each task: the **checks** an exam script would run (all must pass for full marks;
award proportional partial credit if most pass), then a model solution. Total your score
out of 100. Reboot every machine **before** marking — persistence is part of every task.

| Task | Weight | Your score |
|---|---|---|
| 1 | 3 | |
| 2 | 5 | |
| 3 | 5 | |
| 4 | 4 | |
| 5 | 3 | |
| 6 | 4 | |
| 7 | 3 | |
| 8 | 3 | |
| 9 | 8 | |
| 10 | 5 | |
| 11 | 7 | |
| 12 | 6 | |
| 13 | 6 | |
| 14 | 7 | |
| 15 | 6 | |
| 16 | 5 | |
| 17 | 5 | |
| 18 | 4 | |
| 19 | 6 | |
| 20 | 5 | |
| **Total** | **100** | |

Task 5 can't survive a reboot by its nature — mark it before rebooting.

---

### Task 1 — files (3%)

**Checks:** `/root/bigrecent.txt` equals `find /var/log -type f -size +1M -mtime -7 | sort`;
contains `/var/log/exam/big-recent.log`, not `big-old.log`.

```bash
sudo bash -c 'find /var/log -type f -size +1M -mtime -7 | sort > /root/bigrecent.txt'
```

The redirect must happen as root — hence `bash -c`. Journal files in `/var/log/journal/`
legitimately appear too.

### Task 2 — Git (5%)

**Checks:** on ubuntu2, `git --git-dir=/srv/git/exam.git log fix-port -1 --format=%s` is
`Change port to 8080`; `git --git-dir=… show fix-port:app.conf` contains `port = 8080`;
`main` is unchanged.

```bash
git clone ubuntu2:/srv/git/exam.git ~/exam && cd ~/exam
git switch -c fix-port
sed -i 's/^port = 80$/port = 8080/' app.conf
git commit -am "Change port to 8080"
git push -u origin fix-port
```

If Git complains about identity, set `user.name`/`user.email` first.

### Task 3 — service (5%)

**Checks:** `systemctl is-active examapp` and `is-enabled` both succeed; process user is
`www-data`; `Restart=on-failure` (or `always`); `curl ubuntu2:9000` (from ubuntu2) returns
`examapp ok`.

```bash
sudo tee /etc/systemd/system/examapp.service > /dev/null <<'EOF'
[Unit]
Description=Exam application
After=network.target

[Service]
User=www-data
ExecStart=/usr/bin/python3 -m http.server 9000 --directory /srv/examapp
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now examapp
```

### Task 4 — certificate (4%)

**Checks:** `openssl x509 -in web.crt -noout -ext subjectAltName` shows `web.exam.local`;
validity ≈ 90 days; key is RSA 2048; key and certificate public keys match; key mode 600,
owner root.

```bash
sudo mkdir -p /etc/ssl/exam
sudo openssl req -x509 -newkey rsa:2048 -nodes -days 90 \
  -keyout /etc/ssl/exam/web.key -out /etc/ssl/exam/web.crt \
  -subj "/CN=web.exam.local" -addext "subjectAltName=DNS:web.exam.local"
sudo chmod 600 /etc/ssl/exam/web.key
```

### Task 5 — deleted file (3%)

**Checks:** the `examholder` process is still running; `df` no longer counts the 300 MB;
`/root/holder.pid` contains its PID.

```bash
sudo lsof +L1 | grep holder.dat            # PID and FD number
pid=$(pgrep -f examholder)
fd=$(sudo ls -l /proc/$pid/fd | awk '/holder.dat/ {print $9}')
sudo truncate -s 0 /proc/$pid/fd/$fd
echo "$pid" | sudo tee /root/holder.pid
```

### Task 6 — user and group (4%)

**Checks:** `getent group auditors` → GID 4000; `id jdoe` → UID 2500, groups include
auditors; shell `/bin/bash`; home exists; `chage -l jdoe` → maximum 45, account expires
Dec 31, 2026.

```bash
sudo groupadd -g 4000 auditors
sudo useradd -m -u 2500 -s /bin/bash -G auditors jdoe
sudo chage -M 45 -E 2026-12-31 jdoe
```

### Task 7 — ACL (3%)

**Checks:** `getfacl /var/log/exam` shows `group:auditors:r-x` and
`default:group:auditors:r-x`; existing files show `group:auditors:r--`; owners and modes
unchanged; a newly created file inherits the entry.

```bash
sudo setfacl -R -m g:auditors:rX /var/log/exam
sudo setfacl -d -m g:auditors:rX /var/log/exam
```

### Task 8 — limits (3%)

**Checks:** a file under `/etc/security/limits.d/` (or `limits.conf`) contains the three
entries; a login session for jdoe reports `ulimit -Sn` and `-Hn` = 1024 and `-Hu` = 50.

```bash
sudo tee /etc/security/limits.d/exam.conf > /dev/null <<'EOF'
@auditors  hard  nproc   50
jdoe       soft  nofile  1024
jdoe       hard  nofile  1024
EOF
```

(`jdoe - nofile 1024` sets both in one line.)

### Task 9 — LVM (8%)

**Checks:** `vgs vgexam` uses `/dev/sdb`; `lvs vgexam/lvdata` ≈ 2.0 GiB; filesystem type
XFS; `/data` mounted after reboot from an fstab entry; `df -h /data` ≈ 2 GiB.

```bash
sudo pvcreate /dev/sdb
sudo vgcreate vgexam /dev/sdb
sudo lvcreate -n lvdata -L 1.5G vgexam
sudo mkfs.xfs /dev/vgexam/lvdata
sudo mkdir -p /data
echo '/dev/vgexam/lvdata  /data  xfs  defaults  0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount -a
sudo lvextend -r -L +500M /dev/vgexam/lvdata
```

### Task 10 — swap (5%)

**Checks:** `swapon --show` lists `/swap.exam` at 512M after reboot; mode 600.

```bash
sudo fallocate -l 512M /swap.exam
sudo chmod 600 /swap.exam
sudo mkswap /swap.exam && sudo swapon /swap.exam
echo '/swap.exam  none  swap  sw  0 0' | sudo tee -a /etc/fstab
```

### Task 11 — autofs (7%)

**Checks:** `autofs` active and enabled; `ls /mnt/auto/shared` shows `README`; `findmnt`
shows it mounted rw from `ubuntu2:/srv/exports/shared`; the timeout is 60.

```bash
sudo apt install -y autofs nfs-common
echo '/mnt/auto  /etc/auto.exam  --timeout=60' | sudo tee /etc/auto.master.d/exam.autofs
echo 'shared  -rw  ubuntu2:/srv/exports/shared' | sudo tee /etc/auto.exam
sudo systemctl enable --now autofs && sudo systemctl restart autofs
ls /mnt/auto/shared
```

### Task 12 — addressing (6%)

**Checks:** after reboot, `ip addr` on ubuntu2 shows `192.168.56.12`, `192.168.56.112` and
`fd00:56::12`; `ip route` shows `10.50.0.0/16 via 192.168.56.13`; SSH from ubuntu1 still
works.

```bash
sudo tee /etc/netplan/60-exam.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    eth1:
      addresses:
        - 192.168.56.112/24
        - "fd00:56::12/64"
      routes:
        - to: 10.50.0.0/16
          via: 192.168.56.13
EOF
sudo chmod 600 /etc/netplan/60-exam.yaml
sudo netplan get ethernets.eth1        # 192.168.56.12/24 must still be listed
sudo netplan try
```

### Task 13 — bond (6%)

**Checks:** `/proc/net/bonding/bond0` shows `active-backup` and both slaves; `bond0` has
`10.99.0.11/24`; survives reboot.

```bash
sudo tee /etc/netplan/61-bond.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    eth2: {}
    eth3: {}
  bonds:
    bond0:
      interfaces: [eth2, eth3]
      addresses: [10.99.0.11/24]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
EOF
sudo chmod 600 /etc/netplan/61-bond.yaml
sudo netplan apply
```

### Task 14 — firewall (7%)

**Checks:** `nftables` enabled; after reboot the ruleset has policy drop on input; SSH from
ubuntu1 works; `curl ubuntu2:9000` and `curl ubuntu2` (port 80) from ubuntu1 return
`examapp ok`; port 9000 from an address outside 192.168.56.0/24 is not accepted.

```bash
sudo tee /etc/nftables.conf > /dev/null <<'EOF'
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        ct state established,related accept
        iif "lo" accept
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept
        tcp dport 22 accept
        ip saddr 192.168.56.0/24 tcp dport 9000 accept
    }
}

table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        tcp dport 80 redirect to :9000
    }
}
EOF
sudo nft -c -f /etc/nftables.conf
sudo systemctl enable --now nftables && sudo systemctl restart nftables
```

### Task 15 — reverse proxy (6%)

**Checks:** `nginx -t` passes; nginx active and enabled; `curl -H 'Host: exam.local'
localhost` on ubuntu1 returns `examapp ok`; config contains `proxy_set_header
X-Forwarded-For`.

```bash
echo '127.0.0.1 exam.local' | sudo tee -a /etc/hosts
sudo tee /etc/nginx/sites-available/exam.conf > /dev/null <<'EOF'
server {
    listen 80;
    server_name exam.local;
    location / {
        proxy_pass http://192.168.56.12:9000/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
sudo ln -sf /etc/nginx/sites-available/exam.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl enable --now nginx && sudo systemctl reload nginx
curl -s http://exam.local/
```

### Task 16 — kernel (5%)

**Checks:** after reboot `sysctl vm.swappiness` = 20, `net.ipv4.ip_forward` = 1;
`lsmod | grep nbd`; `/sys/module/nbd/parameters/nbds_max` = 4.

```bash
printf 'vm.swappiness = 20\nnet.ipv4.ip_forward = 1\n' | sudo tee /etc/sysctl.d/90-exam.conf
sudo sysctl --system
echo nbd | sudo tee /etc/modules-load.d/nbd.conf
echo 'options nbd nbds_max=4' | sudo tee /etc/modprobe.d/nbd.conf
```

### Task 17 — timer (5%)

**Checks:** `cleanup.timer` enabled and listed by `systemctl list-timers` with next run at
02:15; `Persistent=true`; the service runs `/usr/local/bin/cleanup.sh` successfully
(`systemctl start cleanup.service` exits 0).

```bash
sudo tee /etc/systemd/system/cleanup.service > /dev/null <<'EOF'
[Unit]
Description=Exam cleanup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cleanup.sh
EOF
sudo tee /etc/systemd/system/cleanup.timer > /dev/null <<'EOF'
[Unit]
Description=Run cleanup daily at 02:15

[Timer]
OnCalendar=*-*-* 02:15:00
Persistent=true

[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now cleanup.timer
```

### Task 18 — packages (4%)

**Checks:** `command -v dig` succeeds; `dpkg -l telnet` shows no `ii` or `rc` line;
`apt-mark showhold` lists `curl`.

```bash
sudo apt install -y bind9-dnsutils
sudo apt purge -y telnet
sudo apt-mark hold curl
```

### Task 19 — SELinux (6%)

**Checks:** `getenforce` = Enforcing; `curl rocky1:8088` from ubuntu1 returns `rocky exam`;
`semanage fcontext -l -C` has a rule for `/srv/exam`; `semanage port -l` maps 8088 to
`http_port_t`; firewalld permanent config has 8088/tcp; nginx enabled.

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/exam(/.*)?'
sudo restorecon -Rv /srv/exam
sudo semanage port -l | grep -w 8088 \
  && sudo semanage port -m -t http_port_t -p tcp 8088 \
  || sudo semanage port -a -t http_port_t -p tcp 8088
sudo tee /etc/nginx/conf.d/exam.conf > /dev/null <<'EOF'
server {
    listen 8088;
    root /srv/exam;
}
EOF
sudo nginx -t && sudo systemctl enable --now nginx && sudo systemctl restart nginx
sudo firewall-cmd --permanent --add-port=8088/tcp && sudo firewall-cmd --reload
```

"Survive a relabel" rules out `chcon` — that's what the wording was testing.

### Task 20 — container (5%)

**Checks:** `docker ps` (or `podman ps`) shows `exam-web` from `nginx:1.27`, port
`8085->80`; `curl localhost:8085` returns `exam web`; the mount is read-only; restart policy
`unless-stopped` (or a systemd unit / Quadlet that achieves the same); running after reboot.

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo docker run -d --name exam-web --restart unless-stopped -p 8085:80 \
  -v /srv/exam-web:/usr/share/nginx/html:ro nginx:1.27
curl -s localhost:8085
```

---

## Reading your score

| Score | Meaning |
|---|---|
| 85+ | Ready. Do mock 2 to confirm, then book |
| 67–84 | Passing, with no margin. Redo the 🔴 tasks of every chapter where you lost marks |
| under 67 | Find the domain where most points went; redo that chapter's whole lab before mock 2 |

Lost points to persistence (works now, not after reboot)? That's a habit, not knowledge —
the fastest marks you'll ever gain. Re-read [the verification table](strategy.md#for-every-single-task).
