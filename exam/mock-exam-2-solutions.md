# Mock exam 2 — marking guide

Reboot all three machines before marking (except where a task says its state needn't
survive). For each task: the checks, then a model solution.

| Task | Weight | Your score |
|---|---|---|
| 1 | 4 | |
| 2 | 5 | |
| 3 | 6 | |
| 4 | 5 | |
| 5 | 3 | |
| 6 | 3 | |
| 7 | 4 | |
| 8 | 6 | |
| 9 | 7 | |
| 10 | 7 | |
| 11 | 6 | |
| 12 | 6 | |
| 13 | 5 | |
| 14 | 8 | |
| 15 | 5 | |
| 16 | 8 | |
| 17 | 7 | |
| 18 | 5 | |
| **Total** | **100** | |

---

### Task 1 — I/O hog (4%)

**Checks:** `/root/io-hog.pid` holds the PID of the python process writing
`/var/tmp/.cache-flush`; that process no longer runs.

```bash
sudo apt install -y sysstat           # if pidstat/iostat are missing
pidstat -d 1 3                        # one python3 process with large kB_wr/s
sudo lsof /var/tmp/.cache-flush       # confirms the file it writes
pid=$(sudo lsof -t /var/tmp/.cache-flush)
echo "$pid" | sudo tee /root/io-hog.pid
sudo kill "$pid"
```

`iotop -o` works too. Writing the PID *before* killing it matters — afterwards it's gone.

### Task 2 — service constraints (5%)

**Checks:** `systemctl show examapp -p MemoryMax -p CPUQuotaPerSecUSec -p LimitNOFILE` →
`209715200`, `300ms`, `4096`; the unit file in `/etc/systemd/system/examapp.service` is
unchanged; the service is active.

```bash
sudo systemctl edit examapp
#   [Service]
#   MemoryMax=200M
#   CPUQuota=30%
#   LimitNOFILE=4096
sudo systemctl restart examapp         # LimitNOFILE applies at process start
```

`systemctl set-property` handles the first two but not `LimitNOFILE` — the drop-in does all
three at once.

### Task 3 — certificate signing (6%)

**Checks:** `openssl verify -CAfile /root/ca/ca.crt /root/exam.crt` → OK; SAN contains
`app.exam.local`; validity 365 days; the CA is in the system store
(`/etc/ssl/certs/ca-certificates.crt` contains it; `openssl verify /root/exam.crt` with no
`-CAfile` succeeds).

```bash
cd /root
printf 'subjectAltName=DNS:app.exam.local\n' | sudo tee /root/san.ext
sudo openssl x509 -req -in exam.csr -CA ca/ca.crt -CAkey ca/ca.key -CAcreateserial \
  -days 365 -out exam.crt -extfile san.ext
sudo cp /root/ca/ca.crt /usr/local/share/ca-certificates/exam-root-ca.crt
sudo update-ca-certificates
sudo openssl verify /root/exam.crt
```

### Task 4 — journal and logrotate (5%)

**Checks:** `systemd-analyze cat-config systemd/journald.conf` shows `SystemMaxUse=200M`
and persistent storage; a logrotate file covering `/var/log/examapp/*.log` with `weekly`,
`rotate 8`, `compress`, `missingok`, `notifempty`; `logrotate -d` on it reports no errors.

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\nSystemMaxUse=200M\n' | sudo tee /etc/systemd/journald.conf.d/exam.conf
sudo systemctl restart systemd-journald
sudo tee /etc/logrotate.d/examapp > /dev/null <<'EOF'
/var/log/examapp/*.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
}
EOF
sudo logrotate -d /etc/logrotate.d/examapp
```

### Task 5 — sudo (3%)

**Checks:** `visudo -c` clean; a member of `helpdesk` sees exactly the two commands in
`sudo -l`, both `NOPASSWD`; `sudo -n cat /etc/shadow` is refused.

```bash
sudo groupadd helpdesk
echo '%helpdesk ALL=(root) NOPASSWD: /usr/bin/systemctl restart examapp, /usr/bin/journalctl -u examapp' \
  | sudo tee /etc/sudoers.d/helpdesk
sudo chmod 440 /etc/sudoers.d/helpdesk
sudo visudo -c
```

(Or create it with `sudo visudo -f /etc/sudoers.d/helpdesk`.) Test with a throwaway member:
`sudo useradd -m -G helpdesk hd1 && sudo -iu hd1 sudo -l`.

### Task 6 — environment (3%)

**Checks:** `/etc/environment` contains `EXAM_ENV=production`; a file in `/etc/profile.d/`
appends `/opt/exam/bin`; `sudo -iu vagrant env` shows both;
`ssh localhost 'echo $EXAM_ENV'` (non-interactive) prints `production`.

```bash
echo 'EXAM_ENV=production' | sudo tee -a /etc/environment
echo 'export PATH="$PATH:/opt/exam/bin"' | sudo tee /etc/profile.d/exam-path.sh
```

`/etc/environment` is read by `pam_env` for every PAM session — logins, SSH commands, su —
which a `profile.d` script (login shells only) can't do.

### Task 7 — LDAP client (4%)

**Checks:** `getent passwd ldapuser` → UID 6001; `id ldapuser` shows group `staff`; `su -
ldapuser` with `Exam123!` works; home directory created; no `ldapuser` line in
`/etc/passwd`; sssd enabled.

```bash
sudo apt install -y sssd-ldap libnss-sss libpam-sss ldap-utils
sudo tee /etc/sssd/sssd.conf > /dev/null <<'EOF'
[sssd]
services = nss, pam
domains = exam

[domain/exam]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ubuntu2
ldap_search_base = dc=exam,dc=local
cache_credentials = true
ldap_auth_disable_tls_never_use_in_production = true
EOF
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl enable --now sssd && sudo systemctl restart sssd
sudo pam-auth-update --enable mkhomedir
getent passwd ldapuser && id ldapuser
```

### Task 8 — partition and mount (6%)

**Checks:** `/dev/sdc` has a GPT label and one ~1 GiB partition; ext4 labelled `ARCHIVE`;
the fstab entry uses `UUID=` with `noexec,nodev`; mounted on `/archive` after reboot.

```bash
sudo parted -s /dev/sdc mklabel gpt mkpart archive ext4 1MiB 1025MiB
sudo mkfs.ext4 -L ARCHIVE /dev/sdc1
sudo mkdir -p /archive
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdc1)  /archive  ext4  noexec,nodev  0 2" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount -a
findmnt -o TARGET,OPTIONS /archive
```

### Task 9 — repair (7%)

**Checks:** `/srv/vol` mounted after reboot; `/srv/vol/important.txt` contains `do not lose
me`; `e2fsck -fn` on the image (unmounted) reports a clean filesystem.

```bash
sudo mount /srv/vol                            # wrong fs type, bad option, bad superblock …
sudo mke2fs -n -F /var/lib/exam/vol.img        # DRY RUN — lists backup superblocks
sudo e2fsck -y -b <first backup> /var/lib/exam/vol.img
sudo mount /srv/vol && cat /srv/vol/important.txt
```

The first backup is `32768` for 4 KiB blocks but `8193` for 1 KiB blocks, which `mke2fs`
may choose for a filesystem this small — read the number from `mke2fs -n` rather than
assuming. The fstab line already exists; once the filesystem is repaired it mounts at boot.

### Task 10 — NBD (7%)

**Checks:** `/etc/modules-load.d/` loads `nbd`; at marking time (before reboot)
`/mnt/examdisk` is an XFS filesystem on `/dev/nbd0`.

```bash
sudo apt install -y nbd-client xfsprogs
sudo modprobe nbd
echo nbd | sudo tee /etc/modules-load.d/nbd.conf
sudo nbd-client -N examdisk ubuntu2 /dev/nbd0
sudo mkfs.xfs /dev/nbd0
sudo mkdir -p /mnt/examdisk && sudo mount /dev/nbd0 /mnt/examdisk
findmnt /mnt/examdisk
```

### Task 11 — bridge (6%)

**Checks:** after reboot `bridge link` shows `eth2` in `br0`; `br0` has `10.98.0.11/24`; STP off.

```bash
sudo tee /etc/netplan/62-bridge.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    eth2: {}
  bridges:
    br0:
      interfaces: [eth2]
      addresses: [10.98.0.11/24]
      parameters:
        stp: false
EOF
sudo chmod 600 /etc/netplan/62-bridge.yaml
sudo netplan apply
```

### Task 12 — SSH (6%)

**Checks:** `sshd -T` shows `permitrootlogin no`, `passwordauthentication no`, `port 22`
and `port 2222`; `ssh ubuntu2` and `ssh -p 2222 ubuntu2` from ubuntu1 both work; a password
login is refused.

```bash
sudo tee /etc/ssh/sshd_config.d/40-exam.conf > /dev/null <<'EOF'
Port 22
Port 2222
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
sudo sshd -t
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket 2>/dev/null; sudo systemctl restart ssh
sudo ss -tlnp | grep -E ':(22|2222)\b'
```

Keep your existing session open while testing from a new one.

### Task 13 — time (5%)

**Checks:** `chronyc sources` on ubuntu2 lists `ubuntu1` (or `192.168.56.11`) marked
`prefer` in config and reachable; chrony enabled.

```bash
sudo apt install -y chrony
echo 'server ubuntu1 iburst prefer' | sudo tee /etc/chrony/conf.d/exam.conf
sudo systemctl enable --now chrony && sudo systemctl restart chrony
chronyc sources -v
```

### Task 14 — load balancer (8%)

**Checks:** `nginx -t` OK; enabled; repeated `curl lb.exam.local:8080` alternates
`ubuntu2` / `rocky1`; with one backend stopped every request still succeeds.

```bash
echo '127.0.0.1 lb.exam.local' | sudo tee -a /etc/hosts
sudo tee /etc/nginx/sites-available/lb.conf > /dev/null <<'EOF'
upstream exam_pool {
    server 192.168.56.12:8001;
    server 192.168.56.13:8001;
}
server {
    listen 8080;
    server_name lb.exam.local;
    location / {
        proxy_pass http://exam_pool;
        proxy_set_header Host $host;
        proxy_connect_timeout 2s;
    }
}
EOF
sudo ln -sf /etc/nginx/sites-available/lb.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl enable --now nginx && sudo systemctl reload nginx
for i in 1 2 3 4; do curl -s lb.exam.local:8080; done
```

### Task 15 — kernel (5%)

**Checks:** after reboot `/proc/cmdline` contains `audit=1`, and `/etc/default/grub`'s
`GRUB_CMDLINE_LINUX` has it (all entries, including recovery); `modprobe -n -v usb_storage`
shows `install /bin/false` and `sudo modprobe usb_storage` fails.

```bash
sudo sed -i 's/^GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 audit=1"/' /etc/default/grub
grep ^GRUB_CMDLINE_LINUX= /etc/default/grub
sudo update-grub
printf 'blacklist usb_storage\ninstall usb_storage /bin/false\n' | sudo tee /etc/modprobe.d/exam-usb.conf
sudo update-initramfs -u                 # the initramfs may otherwise carry the module
```

### Task 16 — libvirt VM (8%)

**Checks:** `virsh dominfo exam-vm` → running, 256 MiB, 1 CPU, autostart enabled; its disk
is not `/var/lib/libvirt/images/cirros.img`; `cirros.img` unchanged; network `default`.

```bash
cd /var/lib/libvirt/images
sudo qemu-img create -f qcow2 -F qcow2 -b cirros.img exam-vm.qcow2 1G
sudo virt-install --name exam-vm --memory 256 --vcpus 1 \
  --disk path=/var/lib/libvirt/images/exam-vm.qcow2,format=qcow2 --import \
  --osinfo detect=on,require=off --network network=default \
  --graphics none --noautoconsole
sudo virsh autostart exam-vm
sudo virsh dominfo exam-vm
```

`sudo cp cirros.img exam-vm.img` and using the copy is equally correct; the overlay is
just faster and smaller.

### Task 17 — Quadlet (7%)

**Checks:** `/etc/containers/systemd/web.container` exists; `systemctl is-active web`;
`curl localhost:8090` returns `ok`; the volume is read-only; running after reboot.

```bash
sudo tee /etc/containers/systemd/web.container > /dev/null <<'EOF'
[Container]
Image=docker.io/library/nginx:1.27
PublishPort=8090:80
Volume=/srv/examapp:/usr/share/nginx/html:ro

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl start web
curl -s localhost:8090
```

### Task 18 — cron and at (5%)

**Checks:** `crontab -l -u vagrant` contains `*/30 * * * 1-5 /usr/bin/logger exam-cron`;
`atq` shows a job for tomorrow 03:00 whose `at -c` script ends with `/usr/bin/logger exam-at`.

```bash
( crontab -l 2>/dev/null; echo '*/30 * * * 1-5 /usr/bin/logger exam-cron' ) | crontab -
sudo apt install -y at && sudo systemctl enable --now atd
echo '/usr/bin/logger exam-at' | at 03:00 tomorrow
atq
```

`( crontab -l; echo … ) | crontab -` appends without opening an editor — handy, and safe
when no crontab exists yet thanks to `2>/dev/null`.

---

## Reading your score

Compare with mock 1. Two scores above 80 mean you're ready to book. A domain that lost
points in **both** mocks is the one to spend your last study week on.
