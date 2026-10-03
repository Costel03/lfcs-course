# Mock exam 2

**18 tasks · 2 hours · pass mark 67%**

Same rules as mock 1: a 2-hour timer, `man` and `/usr/share/doc` only, base host
**ubuntu1**, everything marked on final state after a reboot. Mock 2 covers the
competencies mock 1 didn't, so together they test every LFCS objective.

Score yourself with the [marking guide](mock-exam-2-solutions.md) afterwards.

---

## Setup — before you start the timer

Restore clean snapshots of all three machines, then on **ubuntu1** run the setup. It
downloads packages and an image, so give it a few minutes. Don't read it for hints.

```bash
# --- ubuntu2: LDAP directory, NBD export, a backend app ---
ssh ubuntu2 'sudo bash -s' <<'EOF'
set -e
debconf-set-selections <<'D'
slapd slapd/password1 password admin
slapd slapd/password2 password admin
slapd slapd/domain string exam.local
slapd shared/organization string Exam
D
DEBIAN_FRONTEND=noninteractive apt-get install -y -qq slapd ldap-utils nbd-server > /dev/null
PW=$(slappasswd -s 'Exam123!')
cat > /tmp/exam.ldif <<L
dn: ou=people,dc=exam,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=exam,dc=local
objectClass: organizationalUnit
ou: groups

dn: cn=staff,ou=groups,dc=exam,dc=local
objectClass: posixGroup
cn: staff
gidNumber: 6000
memberUid: ldapuser

dn: uid=ldapuser,ou=people,dc=exam,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ldapuser
sn: User
cn: LDAP User
uidNumber: 6001
gidNumber: 6000
homeDirectory: /home/ldapuser
loginShell: /bin/bash
userPassword: $PW
L
ldapadd -x -D cn=admin,dc=exam,dc=local -w admin -f /tmp/exam.ldif > /dev/null
mkdir -p /srv/nbd && truncate -s 512M /srv/nbd/examdisk.img && chown nbd: /srv/nbd/examdisk.img
printf '[generic]\n    user = nbd\n    group = nbd\n    allowlist = true\n[examdisk]\n    exportname = /srv/nbd/examdisk.img\n' > /etc/nbd-server/config
systemctl restart nbd-server
mkdir -p /srv/backend && hostname > /srv/backend/index.html
printf '[Service]\nExecStart=/usr/bin/python3 -m http.server 8001 --directory /srv/backend\n[Install]\nWantedBy=multi-user.target\n' > /etc/systemd/system/backend.service
systemctl daemon-reload && systemctl enable --now backend
EOF

# --- rocky1: a second backend ---
ssh rocky1 'sudo bash -s' <<'EOF'
mkdir -p /srv/backend && hostname > /srv/backend/index.html
printf '[Service]\nExecStart=/usr/bin/python3 -m http.server 8001 --directory /srv/backend\n[Install]\nWantedBy=multi-user.target\n' > /etc/systemd/system/backend.service
systemctl daemon-reload && systemctl enable --now backend
firewall-cmd -q --permanent --add-port=8001/tcp && firewall-cmd -q --reload
EOF

# --- ubuntu1 ---
sudo bash -s <<'EOF'
set -e
apt-get update -qq
apt-get install -y -qq nginx chrony podman qemu-system-x86 libvirt-daemon-system \
  libvirt-clients virtinst > /dev/null
echo 'allow 192.168.56.0/24' > /etc/chrony/conf.d/exam-server.conf && systemctl restart chrony
curl -fsSL -o /var/lib/libvirt/images/cirros.img \
  https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

# a CA and a certificate request waiting to be signed
mkdir -p /root/ca && cd /root/ca
openssl req -x509 -newkey rsa:2048 -nodes -days 3650 -keyout ca.key -out ca.crt -subj "/CN=Exam Root CA" 2> /dev/null
cd /root
openssl req -newkey rsa:2048 -nodes -keyout exam.key -out exam.csr -subj "/CN=app.exam.local" 2> /dev/null

# an application service, and its log directory
mkdir -p /srv/examapp /var/log/examapp && echo ok > /srv/examapp/index.html
printf '[Unit]\nDescription=Exam app\n[Service]\nExecStart=/usr/bin/python3 -m http.server 9100 --directory /srv/examapp\n[Install]\nWantedBy=multi-user.target\n' > /etc/systemd/system/examapp.service
systemctl daemon-reload && systemctl enable --now examapp
for i in 1 2; do echo "entry $i" > /var/log/examapp/app$i.log; done

# a filesystem that no longer mounts
mkdir -p /var/lib/exam /srv/vol
truncate -s 200M /var/lib/exam/vol.img && mkfs.ext4 -q /var/lib/exam/vol.img
mount -o loop /var/lib/exam/vol.img /srv/vol && echo 'do not lose me' > /srv/vol/important.txt && umount /srv/vol
echo '/var/lib/exam/vol.img  /srv/vol  ext4  loop,nofail  0 0' >> /etc/fstab
dd if=/dev/zero of=/var/lib/exam/vol.img bs=1024 seek=1 count=1 conv=notrunc status=none

# something keeps the disk busy
nohup python3 -c "
import os
fd = os.open('/var/tmp/.cache-flush', os.O_WRONLY | os.O_CREAT)
buf = b'0' * 1048576
while True:
    os.lseek(fd, 0, 0)
    for _ in range(64):
        os.write(fd, buf)
    os.fsync(fd)
" > /dev/null 2>&1 &
EOF
echo 'setup done'
```

**Start the timer now.**

---

## Essential Commands — 20%

**Task 1** · ubuntu1 · 4%

Something on ubuntu1 is writing to disk continuously. Identify the process, write its PID to
`/root/io-hog.pid`, and stop it.

**Task 2** · ubuntu1 · 5%

Constrain the existing `examapp` service **without editing its unit file**: at most
**200 MiB** of memory, **30%** of one CPU, and **4096** open files per process. The service
must be running with these limits.

**Task 3** · ubuntu1 · 6%

A CA lives in `/root/ca` (`ca.crt`, `ca.key`). Sign the request `/root/exam.csr` with it to
produce `/root/exam.crt`, valid **365 days**, with a Subject Alternative Name of
`DNS:app.exam.local`. Then make ubuntu1 trust the CA system-wide.

**Task 4** · ubuntu1 · 5%

- Limit the systemd journal to **200 MiB** of persistent storage.
- Rotate `/var/log/examapp/*.log` **weekly**, keeping **8** compressed copies, without
  errors if the files are missing or empty.

---

## Users and Groups — 10%

**Task 5** · ubuntu1 · 3%

Create a group `helpdesk`. Its members may run exactly these commands as root **without a
password**, and nothing else:
`/usr/bin/systemctl restart examapp` and `/usr/bin/journalctl -u examapp`.

**Task 6** · ubuntu1 · 3%

- Every user must have the variable `EXAM_ENV=production` in **every** session, including
  non-interactive ones started by PAM.
- Every user's **login shell** must have `/opt/exam/bin` appended to `PATH`.

**Task 7** · ubuntu1 · 4%

ubuntu2 runs an LDAP directory with base `dc=exam,dc=local`. Configure ubuntu1 so that the
directory user `ldapuser` (password `Exam123!`) is resolved, can log in, and gets a home
directory created at first login. Don't create the user locally. (Unencrypted LDAP is
acceptable for this task.)

---

## Storage — 20%

**Task 8** · ubuntu1 · 6%

On the spare disk `/dev/sdc`, create a **GPT** partition table and a **1 GiB** partition
with an **ext4** filesystem labelled `ARCHIVE`. Mount it on `/archive` persistently **by
UUID**, with the options `noexec` and `nodev`. Leave the rest of the disk unpartitioned.

**Task 9** · ubuntu1 · 7%

The filesystem that should be mounted on `/srv/vol` (from `/var/lib/exam/vol.img`) no
longer mounts. Repair it so that it mounts at boot with its existing data intact.

**Task 10** · ubuntu1 · 7%

ubuntu2 exports a network block device named `examdisk`. Attach it to ubuntu1 as
`/dev/nbd0`, create an **XFS** filesystem on it, and mount it on `/mnt/examdisk`. Make the
`nbd` module load at boot. (The NBD connection itself need not persist across reboot.)

---

## Networking — 25%

**Task 11** · ubuntu1 · 6%

Create a bridge `br0` containing the spare interface `eth2`, with the address
`10.98.0.11/24`, persistently. STP disabled.

**Task 12** · ubuntu2 · 6%

Configure the SSH server on ubuntu2 so that:

- root cannot log in,
- password authentication is disabled,
- it listens on port **22 and port 2222**.

Your key-based access from ubuntu1 must keep working on both ports.

**Task 13** · ubuntu2 · 5%

ubuntu1 runs an NTP server. Configure ubuntu2 to synchronise its clock with **ubuntu1** as
its preferred source, using chrony.

**Task 14** · ubuntu1 · 8%

Configure nginx on ubuntu1 so that requests to `http://lb.exam.local:8080/` are
**load-balanced round-robin** across `ubuntu2:8001` and `rocky1:8001`. If one backend fails,
requests must go to the other. nginx must be enabled. (`lb.exam.local` may resolve via
`/etc/hosts`.)

---

## Operations Deployment — 25%

**Task 15** · ubuntu1 · 5%

- Add the kernel parameter `audit=1` permanently to every boot entry.
- Ensure the `usb_storage` module can **never** be loaded, not even manually.

**Task 16** · ubuntu1 · 8%

Using libvirt, create a VM named `exam-vm` from the image
`/var/lib/libvirt/images/cirros.img` (don't modify that file — give the VM its own disk),
with **256 MiB** RAM, **1** vCPU, on the `default` network. It must be running and start
automatically with the host.

**Task 17** · ubuntu1 · 7%

Using **Podman** and **Quadlet**, run `docker.io/library/nginx:1.27` as a **system** service
named `web.service`, publishing container port 80 on host port **8090** and serving
`/srv/examapp` read-only as the document root. It must start at boot.

**Task 18** · ubuntu1 · 5%

- As user `vagrant`, schedule with **cron**: run `/usr/bin/logger exam-cron` every
  **30 minutes**, Monday to Friday.
- Schedule a one-time job with **at** that runs `/usr/bin/logger exam-at` tomorrow at
  **03:00**.

---

**Stop.** Use your last ten minutes on a verification sweep, then go to the
[marking guide](mock-exam-2-solutions.md).
