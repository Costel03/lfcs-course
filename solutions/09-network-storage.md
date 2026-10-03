# Solutions 09 — Network storage and automounting

---

## 🟢 Warm-up

**9.1**

```bash
cat /proc/filesystems
```

Modules load on demand, so `xfs` or `nfs` may appear only after first use.

**9.2**

```bash
findmnt -t proc,sysfs,tmpfs,devtmpfs,cgroup2
```

**9.3**

```bash
cat /proc/sys/vm/swappiness
cat /sys/block/sda/queue/scheduler          # the active one is in [brackets]
cat /sys/class/net/eth0/address             # use: ls /sys/class/net to find the name
```

**9.4**

```bash
sudo mkdir -p /mnt/scratch
sudo mount -t tmpfs -o size=50M tmpfs /mnt/scratch
echo hi | sudo tee /mnt/scratch/f
df -h /mnt/scratch
sudo umount /mnt/scratch && sudo mount -t tmpfs -o size=50M tmpfs /mnt/scratch
ls /mnt/scratch                              # empty
```

The data existed only in that tmpfs instance's memory; unmounting freed it.

---

## 🔵 Practical

**9.5**

```bash
mkdir -p ~/ov/{lower,upper,work} && sudo mkdir -p /mnt/merged
echo v1 > ~/ov/lower/config.txt
echo bye > ~/ov/lower/remove-me.txt
sudo mount -t overlay overlay -o lowerdir=$HOME/ov/lower,upperdir=$HOME/ov/upper,workdir=$HOME/ov/work /mnt/merged
echo v2 | sudo tee /mnt/merged/config.txt
sudo rm /mnt/merged/remove-me.txt
cat ~/ov/lower/config.txt          # v1
ls -l ~/ov/upper                   # config.txt (v2) and  c--------- 0,0 remove-me.txt
```

A deletion can't touch the read-only lower layer, so overlayfs records a **whiteout** — a
`0,0` character device — in upper that hides the lower file. Container images delete files
the same way, which is why deleting a file in a later Dockerfile layer doesn't shrink the
image.

**9.6** (ubuntu2)

```bash
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/share && echo hello | sudo tee /srv/share/hello.txt
echo '/srv/share  192.168.56.0/24(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
sudo exportfs -v
```

From ubuntu1: `sudo apt install -y nfs-common && showmount -e ubuntu2`.

**9.7** (ubuntu1)

```bash
sudo mkdir -p /mnt/share
echo 'ubuntu2:/srv/share  /mnt/share  nfs  defaults,_netdev,nofail  0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount -a
findmnt /mnt/share
```

**9.8**

```bash
sudo touch /mnt/share/fromroot          # ubuntu1
ls -l /srv/share/fromroot               # ubuntu2: nobody nogroup
```

`root_squash` (the default) maps the client's UID 0 to the anonymous user. Root on a
client machine is not trusted to be root on the server's files.

Note `/srv/share` itself must be writable by `nobody` for this to succeed — if the
`touch` failed with *Permission denied*, that's `root_squash` too: as `nobody`, root can't
write to a root-owned directory. `sudo chmod 1777 /srv/share` on the server for the lab.

**9.9**

```bash
# ubuntu2
sudo mkdir -p /srv/public && echo public | sudo tee /srv/public/readme
echo '/srv/public  *(ro,sync,all_squash,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
# ubuntu1
sudo mkdir -p /mnt/public && sudo mount ubuntu2:/srv/public /mnt/public
sudo touch /mnt/public/x                # Read-only file system
```

**9.10**

```bash
# ubuntu1                                   # ubuntu2
sudo useradd -m -u 1501 nfsuser            sudo useradd -m -u 1502 nfsuser
sudo -u nfsuser touch /mnt/share/mine
ls -l /mnt/share/mine    # nfsuser          ls -ln /srv/share/mine   # 1501 — not nfsuser
```

Fix on the server:

```bash
sudo usermod -u 1501 nfsuser               # ubuntu2
sudo find /home/nfsuser -user 1502 -exec chown -h 1501 {} +   # usermod fixes the home dir; check others
ls -l /srv/share/mine                      # nfsuser
```

With `sec=sys` (the default), the client simply sends numbers. NFSv4 can map *names*
(via `idmapd` and a shared domain), but Linux disables that for `sec=sys` by default — so
consistent UIDs, usually via LDAP, are the real fix.

**9.11**

```bash
sudo umount /mnt/share
sudo mount -o soft ubuntu2:/srv/share /mnt/share && nfsstat -m
sudo umount /mnt/share
sudo mount -o hard ubuntu2:/srv/share /mnt/share && findmnt -o OPTIONS /mnt/share
```

Look also at `vers=4.2`, `rsize`/`wsize`, `timeo` and `retrans` — the client negotiated
them with the server.

**9.12** (ubuntu2)

```bash
sudo apt install -y nbd-server
sudo mkdir -p /srv/nbd && sudo truncate -s 1G /srv/nbd/disk1.img
sudo chown nbd: /srv/nbd/disk1.img
sudo tee /etc/nbd-server/config > /dev/null <<'EOF'
[generic]
    user = nbd
    group = nbd
    allowlist = true

[disk1]
    exportname = /srv/nbd/disk1.img
EOF
sudo systemctl restart nbd-server
sudo ss -tlnp | grep 10809
```

**9.13** (ubuntu1)

```bash
sudo apt install -y nbd-client
sudo modprobe nbd
nbd-client -l ubuntu2
sudo nbd-client -N disk1 ubuntu2 /dev/nbd0
lsblk /dev/nbd0
sudo mkfs.ext4 -q /dev/nbd0
sudo mkdir -p /mnt/nbd && sudo mount /dev/nbd0 /mnt/nbd
echo block | sudo tee /mnt/nbd/file
sudo umount /mnt/nbd
sudo nbd-client -d /dev/nbd0
```

Always unmount before disconnecting: pulling a block device out from under a mounted
filesystem is like unplugging a disk.

**9.14**

```bash
echo nbd | sudo tee /etc/modules-load.d/nbd.conf
```

`systemd-modules-load.service` reads every file in `/etc/modules-load.d/` at boot.

**9.15** (ubuntu1)

```bash
sudo sed -i '\#/mnt/share#d' /etc/fstab && sudo systemctl daemon-reload
sudo umount /mnt/share 2>/dev/null
sudo apt install -y autofs
echo '/mnt/auto  /etc/auto.lab  --timeout=30' | sudo tee /etc/auto.master.d/lab.autofs
echo 'share  -rw  ubuntu2:/srv/share' | sudo tee /etc/auto.lab
sudo systemctl restart autofs
findmnt /mnt/auto/share; ls /mnt/auto/share; findmnt /mnt/auto/share
sleep 60; findmnt /mnt/auto/share       # gone again
```

Files in `/etc/auto.master.d/` must end in `.autofs` to be read.

**9.16**

```bash
# ubuntu2
sudo mkdir -p /srv/homes/{alice,bob}
echo '/srv/homes  192.168.56.0/24(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
# ubuntu1
echo '/nethome  /etc/auto.home  --timeout=60' | sudo tee /etc/auto.master.d/home.autofs
echo '*  -rw  ubuntu2:/srv/homes/&' | sudo tee /etc/auto.home
sudo systemctl restart autofs
ls /nethome/alice; ls /nethome/bob
findmnt -t nfs4
```

**9.17**

```bash
sudo rm /etc/auto.master.d/lab.autofs && sudo systemctl restart autofs
sudo mkdir -p /mnt/share
echo 'ubuntu2:/srv/share  /mnt/share  nfs  noauto,x-systemd.automount,x-systemd.idle-timeout=120,_netdev  0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo systemctl restart remote-fs.target
systemctl list-units --type=automount
ls /mnt/share && findmnt /mnt/share
```

The unit name `mnt-share.automount` is the mount path with `/` turned into `-`.

**9.18**

```bash
dd if=/dev/zero of=/var/tmp/load bs=1M count=2000 oflag=direct status=none &
iostat -xz 1 5
```

Read the line for the device holding `/var/tmp` (`findmnt -T /var/tmp` → its disk):
`wkB/s` throughput, `w_await` latency, `%util` busy time.

**9.19**

```bash
pidstat -d 1 3
sudo iotop -o -b -n 3
rm /var/tmp/load
```

**9.20**

```bash
sudo apt install -y fio
fio --name=rr --filename=/var/tmp/fio.test --size=256M --rw=randread --bs=4k \
    --direct=1 --runtime=20 --time_based --group_reporting
rm /var/tmp/fio.test
```

On a VM the number reflects your host's disk and VirtualBox's caching. The method is what
transfers — on a real server, compare the result with the device's specification.

---

## 🔴 Challenge

**9.21**

```bash
# make sure /mnt/share is a plain hard mount (not the automount from 9.17)
sudo umount /mnt/share; sudo mount -o hard ubuntu2:/srv/share /mnt/share
ssh ubuntu2 sudo systemctl stop nfs-kernel-server
ls /mnt/share &
sleep 5; ps -o pid,stat,cmd -p $!       # D
kill -9 $!; sleep 1; ps -p $!           # gone: NFS waits are killable
ls /mnt/share &                         # hangs again
sudo umount -f -l /mnt/share            # detach; anything still waiting gets an error
ssh ubuntu2 sudo systemctl start nfs-kernel-server
```

`-f` aborts outstanding NFS requests; `-l` detaches the mount from the tree immediately.
Together they're the standard way to free a client from a dead server. Since kernel
2.6.25, processes waiting on NFS sleep in a *killable* state: `ps` shows `D`, but SIGKILL is
delivered. Ordinary disk I/O waits are not like that.

**9.22**

```bash
# ubuntu2
sudo sed -i 's#^/srv/share .*#/srv/share  192.168.56.99(rw,sync,no_subtree_check)#' /etc/exports
sudo exportfs -ra
# ubuntu1
sudo mount ubuntu2:/srv/share /mnt/share              # access denied by server
sudo mount -o vers=3 ubuntu2:/srv/share /mnt/share    # same error
# ubuntu2
journalctl --since '-5min' | grep -i 'refused mount'
#  rpc.mountd: refused mount request from 192.168.56.11 for /srv/share (/srv/share): unmatched host
```

NFSv3 uses a separate MOUNT protocol served by `rpc.mountd`, which logs every refusal.
NFSv4 has no MOUNT protocol — the kernel server checks exports internally — so the v4
attempt fails without that log line. Restore the original line and `exportfs -ra`.

**9.23**

```bash
sudo sed -i 's#^/srv/public .*#/srv/public 192.168.56.11 (rw)#' /etc/exports
sudo exportfs -ra
sudo exportfs -v | grep -A1 public
#  /srv/public   192.168.56.11(ro,…)      ← the host, with DEFAULT options (ro is the default)
#  /srv/public   <world>(rw,…)           ← "(rw)" became a SEPARATE entry for everyone
```

The space split one entry into two: `192.168.56.11` with defaults, and `(rw)` applied to
any host. Fix: `/srv/public 192.168.56.11(rw,sync,no_subtree_check)`. `exportfs` usually
warns about this — read its output.

**9.24** (ubuntu2)

```bash
sudo mkdir -p /srv/share-ro
echo '/srv/share  /srv/share-ro  none  bind,ro  0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount -a
sudo touch /srv/share-ro/x       # Read-only file system
sudo touch /srv/share/x          # works
findmnt /srv/share-ro            # shows ro
```

Current util-linux performs `bind,ro` as a bind followed by a read-only remount in one
step. On very old systems you needed a second `mount -o remount,bind,ro` — `findmnt`
confirms which you got.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | Gives every filesystem one common interface for system calls, routing each to the right driver | |
| 2 | `proc` (processes, kernel state), `sysfs` (devices), `tmpfs` (RAM files), `devtmpfs`, `cgroup2`, `overlay` — any three | |
| 3 | First: rw to that host. Second: that host with defaults (ro), **and the world rw** | The space separates the option from the host |
| 4 | Owned by `nobody` | `root_squash` is the default |
| 5 | `exportfs -ra` | |
| 6 | Her UID differs between client and server | NFS sends numbers, not names |
| 7 | Marks it as needing the network: wait for the network at boot, unmount before it goes down | |
| 8 | `hard`; it never silently loses or corrupts data; `umount -f -l` (each hung process can be `kill -9`ed — NFS waits are killable) | |
| 9 | fstab is world-readable; use `credentials=` pointing to a root-only (`600`) file | |
| 10 | NFS shares a filesystem (server manages it); NBD shares raw blocks (client puts its own filesystem on it) | |
| 11 | Filesystem corruption — ext4 assumes a single writer | |
| 12 | `*` matches any key looked up; `&` is replaced by that key | |
| 13 | No — keys only appear when accessed (unless browse mode is on) | |
| 14 | `noauto,x-systemd.automount,x-systemd.idle-timeout=300,_netdev` | |
| 15 | `r_await`/`w_await` (latency including queueing); SSDs process many requests in parallel, so "busy" ≠ saturated | |
| 16 | Otherwise writes land in the page cache and you measure RAM | |
