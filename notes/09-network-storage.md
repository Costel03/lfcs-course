# Chapter 09 — Network storage and automounting

Chapter 08 was storage attached to the machine. This chapter is storage across the
network — shared filesystems with NFS, raw block devices with NBD — mounted on demand by
an automounter, plus the kernel's virtual filesystem layer that makes all filesystems look
alike, and the tools for measuring storage performance.

LFCS: *Storage* — use remote filesystems and network block devices; configure filesystem
automounters; manage and configure the virtual file system; monitor storage performance.

## Objectives

After this chapter you can:

- Explain the virtual filesystem layer and use pseudo-filesystems: `proc`, `sysfs`, `tmpfs`,
  bind and overlay mounts.
- Export directories over NFS with correct security options, and mount them on clients
  persistently.
- Mount an SMB/CIFS share with credentials kept out of `fstab`.
- Export and attach a network block device, and put a filesystem on it.
- Configure `autofs` direct and indirect maps, and systemd automounts.
- Measure storage performance and identify the process and the disk behind slow I/O.

---

## 1. The virtual filesystem (VFS)

Programs call `open()`, `read()`, `write()` — they never know whether the file lives on
ext4, XFS, NFS or in RAM. The kernel's **VFS** layer provides that one interface and passes
each call to the right filesystem driver.

```bash
cat /proc/filesystems         # filesystem types this kernel supports ("nodev" = no device needed)
findmnt -t proc,sysfs,tmpfs,devtmpfs,cgroup2
```

Many filesystems have **no disk at all** — the kernel generates them:

| Type | Mounted at | Contains |
|---|---|---|
| `proc` | `/proc` | processes and kernel state; tunables under `/proc/sys` |
| `sysfs` | `/sys` | devices, drivers, kernel objects |
| `devtmpfs` | `/dev` | device nodes |
| `tmpfs` | `/run`, `/dev/shm`, often `/tmp` | files in RAM |
| `cgroup2` | `/sys/fs/cgroup` | control groups (chapter 05) |
| `overlay` | container roots | a merged view of several directories |

These files are interfaces, not data:

```bash
cat /proc/meminfo                         # memory, generated on read
cat /proc/sys/vm/swappiness               # a tunable (chapter 12: sysctl)
cat /sys/block/sda/queue/scheduler        # I/O scheduler for a disk
cat /sys/class/net/eth0/address           # a MAC address
```

### tmpfs

```bash
sudo mount -t tmpfs -o size=100M,mode=1777 tmpfs /mnt/scratch
```

`size=` caps it (default: half of RAM). Contents use RAM and can be swapped out; they
vanish at unmount or reboot.

### Bind mounts

```bash
sudo mount --bind /srv/web /var/www/site          # same directory, second path
sudo mount -o remount,bind,ro /var/www/site       # read-only view of a writable directory
```

### Overlay mounts

An overlay presents a read-only **lower** directory with a writable **upper** directory on
top — writes go to upper, the lower stays untouched. This is exactly how container image
layers work (chapter 14).

```bash
mkdir -p /tmp/ov/{lower,upper,work,merged}
echo original > /tmp/ov/lower/file
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/ov/lower,upperdir=/tmp/ov/upper,workdir=/tmp/ov/work /tmp/ov/merged
echo changed | sudo tee /tmp/ov/merged/file
cat /tmp/ov/lower/file        # original — the lower layer is unchanged
cat /tmp/ov/upper/file        # changed  — the edit landed in upper
```

---

## 2. NFS — sharing a filesystem

### Server

```bash
sudo apt install nfs-kernel-server
sudo mkdir -p /srv/share
```

`/etc/exports` — one line per export:

```text
/srv/share    192.168.56.0/24(rw,sync,no_subtree_check)
/srv/public   *(ro,sync,all_squash,no_subtree_check)
/srv/backup   192.168.56.11(rw,sync,no_root_squash,no_subtree_check)
```

**No space between the client and the parenthesis.** `host (rw)` means "host with default
options, and *everyone else* read-write".

| Option | Meaning |
|---|---|
| `rw` / `ro` | read-write / read-only |
| `sync` | reply only after data is on disk — safe (default); `async` is faster and loses data on crash |
| `root_squash` | client root becomes `nobody` — **the default** |
| `no_root_squash` | client root stays root on the export — only for trusted hosts |
| `all_squash` | every client user becomes `nobody` (`anonuid=`/`anongid=` to choose who) |
| `no_subtree_check` | recommended; avoids problems with renamed files |

```bash
sudo exportfs -ra            # re-read /etc/exports
sudo exportfs -v             # what is exported, with effective options
showmount -e localhost       # what clients will see (NFSv3 query)
sudo systemctl enable --now nfs-kernel-server     # nfs-server on RedHat
```

### Client

```bash
sudo apt install nfs-common                       # nfs-utils on RedHat
showmount -e ubuntu2
sudo mount -t nfs ubuntu2:/srv/share /mnt/share
findmnt /mnt/share                                 # shows nfs4 and the negotiated options
```

`/etc/fstab`:

```text
ubuntu2:/srv/share  /mnt/share  nfs  defaults,_netdev,nofail  0 0
```

`_netdev` tells systemd this mount needs the network, so it waits for it at boot and
unmounts it before the network goes down at shutdown.

### Permissions across machines

NFS sends **numeric** UIDs and GIDs. If alice is UID 1001 on the client and 1002 on the
server, she'll see files owned by someone else. Keep UIDs consistent across machines —
this is one of the main reasons organisations use LDAP (chapter 04).

### hard versus soft

| Mount option | If the server stops responding |
|---|---|
| `hard` (default) | processes wait — forever — until it comes back. No data loss, but hung processes in `D` state |
| `soft` | operations fail with an error after retries. Applications may see I/O errors and corrupt data |

`hard` is the safe default for data; the price is that a dead NFS server hangs every
process that touches the mount. They show as `D` in `ps`, but NFS waits are *killable*:
unlike a true uninterruptible sleep, `kill -9` does end them. New processes will hang the
same way, though, until you detach the mount with `umount -f -l`.

### Troubleshooting

```bash
rpcinfo -p ubuntu2           # RPC services (NFSv3 needs these; v4 needs only port 2049)
nc -zv ubuntu2 2049          # is the NFS port reachable?
nfsstat -m                   # mounted NFS filesystems and their options
journalctl -u nfs-server     # server side
```

| Symptom | Cause |
|---|---|
| `access denied by server` | client not matched in `/etc/exports`, or forgot `exportfs -ra` |
| hangs on mount | firewall blocking 2049, or server down |
| root can't write | `root_squash` — working as intended |
| wrong owner names | UID/GID mismatch between machines |

---

## 3. SMB/CIFS client

Windows shares and Samba servers use SMB:

```bash
sudo apt install cifs-utils
sudo tee /root/.smbcred > /dev/null <<'EOF'
username=alice
password=Secret123
domain=LAB
EOF
sudo chmod 600 /root/.smbcred
sudo mount -t cifs //fileserver/projects /mnt/projects \
  -o credentials=/root/.smbcred,uid=1001,gid=1001,file_mode=0640,dir_mode=0750
```

fstab:

```text
//fileserver/projects  /mnt/projects  cifs  credentials=/root/.smbcred,uid=1001,gid=1001,_netdev,nofail  0 0
```

Never put `password=` in `/etc/fstab` — it's world-readable. SMB has no Unix ownership, so
`uid=`, `gid=` and the `*_mode` options decide what the local permissions look like.

---

## 4. Network block devices

NFS shares a **filesystem**; the server owns it. A network block device shares a raw
**block device**; the client puts its own filesystem on it, exactly like a local disk.

### NBD

Server (`ubuntu2`):

```bash
sudo apt install nbd-server
sudo mkdir -p /srv/nbd && sudo truncate -s 1G /srv/nbd/disk1.img
sudo chown nbd: /srv/nbd/disk1.img
```

`/etc/nbd-server/config`:

```ini
[generic]
    user = nbd
    group = nbd
    allowlist = true

[disk1]
    exportname = /srv/nbd/disk1.img
```

```bash
sudo systemctl restart nbd-server        # listens on TCP 10809
```

Client (`ubuntu1`):

```bash
sudo apt install nbd-client
sudo modprobe nbd                        # the kernel driver; creates /dev/nbd0…
nbd-client -l ubuntu2                    # list exports
sudo nbd-client -N disk1 ubuntu2 /dev/nbd0
lsblk /dev/nbd0                          # a 1 GiB disk
sudo mkfs.ext4 /dev/nbd0 && sudo mount /dev/nbd0 /mnt/nbd
# … use it …
sudo umount /mnt/nbd
sudo nbd-client -d /dev/nbd0             # disconnect
```

Load the module at every boot: `echo nbd | sudo tee /etc/modules-load.d/nbd.conf`.
Persistent connections can be declared in `/etc/nbdtab`.

**Only one client may mount a given block device at a time** (with an ordinary
filesystem). Two clients writing ext4 on the same blocks corrupt it instantly — the
filesystem assumes it's the only writer. That's the difference from NFS.

### iSCSI — the enterprise version

iSCSI is the same idea with a richer protocol, used by SANs. Server side is `targetcli`;
client side is `iscsiadm`:

```bash
sudo iscsiadm -m discovery -t sendtargets -p storage01
sudo iscsiadm -m node --login
lsblk                                    # a new /dev/sdX appears
```

---

## 5. Automounting

Mounting network storage permanently has costs: boot waits for it, a dead server hangs
whatever touches it, and idle mounts hold resources. An **automounter** mounts on first
access and unmounts after a period of inactivity.

### autofs

```bash
sudo apt install autofs
```

**Indirect map** — a parent directory managed by autofs, with keys appearing beneath it:

```text
# /etc/auto.master.d/lab.autofs
/mnt/auto   /etc/auto.lab   --timeout=60
```

```text
# /etc/auto.lab
share    -rw,soft   ubuntu2:/srv/share
public   -ro        ubuntu2:/srv/public
```

```bash
sudo systemctl restart autofs
ls /mnt/auto                 # empty — nothing is mounted yet
ls /mnt/auto/share           # triggers the mount
findmnt /mnt/auto/share
```

The keys are invisible until accessed (`--ghost` or `browse_mode = yes` shows them).
`/mnt/auto` belongs to autofs: don't create subdirectories in it or list it in fstab.

**Wildcard map** — the classic home-directories setup:

```text
# /etc/auto.master.d/home.autofs
/nethome   /etc/auto.home

# /etc/auto.home
*   -rw   ubuntu2:/home/&
```

`*` matches any key; `&` is replaced by it. `cd /nethome/alice` mounts
`ubuntu2:/home/alice`.

**Direct map** — full paths anywhere:

```text
# /etc/auto.master.d/direct.autofs
/-   /etc/auto.direct

# /etc/auto.direct
/srv/reports   -ro   ubuntu2:/srv/public
```

Debug: `sudo automount -f -v` runs in the foreground with verbose output (stop the
service first).

### systemd automount through fstab

No extra package — add options to an fstab line:

```text
ubuntu2:/srv/share  /mnt/share  nfs  noauto,x-systemd.automount,x-systemd.idle-timeout=60,_netdev  0 0
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart remote-fs.target
systemctl list-units --type=automount
ls /mnt/share                # triggers it
```

The automount unit is created even with `noauto`; the real mount happens on first access
and is dropped after 60 idle seconds.

| | autofs | systemd automount |
|---|---|---|
| Extra package | yes | no |
| Wildcards (`*` / `&`) | yes | no |
| Maps from LDAP | yes | no |
| Configuration | its own map files | fstab options |

---

## 6. Storage performance

```bash
iostat -xz 1                 # extended per-device stats, skip idle devices (sysstat)
sudo iotop -o                # per-process I/O, only processes doing I/O
pidstat -d 1                 # per-process I/O, no root needed for your own
vmstat 1                     # "b" (blocked) and "wa" (I/O wait)
sar -d                       # history (chapter 05)
```

Reading `iostat -x`:

| Column | Meaning | Worry when |
|---|---|---|
| `r/s`, `w/s` | read/write operations per second (IOPS) | near the device's limit |
| `rkB/s`, `wkB/s` | throughput | near the device's limit |
| `r_await`, `w_await` | average ms per request, **including queueing** | high and rising — this is what users feel |
| `aqu-sz` | average queue length | consistently above 1 per device |
| `%util` | time the device was busy | near 100% on a single spinning disk |

`%util` is misleading for SSDs, NVMe and RAID: they serve many requests in parallel, so a
device can be "100% busy" and still have capacity. Latency (`await`) is the better signal.

### Measuring a device

```bash
# sequential write throughput, bypassing the page cache
dd if=/dev/zero of=/mnt/data/test bs=1M count=500 oflag=direct status=progress
# random I/O — what databases do — with fio
sudo apt install fio
fio --name=randread --filename=/mnt/data/fio.test --size=256M --rw=randread \
    --bs=4k --direct=1 --runtime=20 --time_based --group_reporting
```

Without `oflag=direct` / `--direct=1`, you measure RAM, not the disk.

### I/O scheduler

```bash
cat /sys/block/sda/queue/scheduler          # [mq-deadline] kyber bfq none
echo none | sudo tee /sys/block/sda/queue/scheduler     # NVMe/virtual disks often prefer none
```

Changes through `/sys` are lost at reboot; persist them with a udev rule.

---

## 7. Gotchas

**A space in `/etc/exports`** between host and options silently exports read-write to the
world.

**Forgetting `exportfs -ra`** after editing `/etc/exports`.

**`no_root_squash` on an export** lets any root user on any allowed client own the files —
grant it only to hosts you control.

**UID mismatch** makes NFS files appear to belong to the wrong user.

**Hard mounts and a dead server** — every process that touches the mount hangs in `D`
state. `kill -9` ends each one, but only `umount -f -l` stops new ones hanging.

**NFS in fstab without `_netdev`** can stall the boot on systems where the network comes up
late; add `nofail` for anything optional.

**Mounting the same NBD device on two clients** destroys the filesystem.

**Creating directories inside an autofs-managed directory** — autofs owns it.

---

## 8. Commands introduced

| Command | Purpose |
|---|---|
| `/proc/filesystems`, `findmnt -t` | supported and mounted filesystem types |
| `mount -t tmpfs`, `mount --bind`, `mount -t overlay` | virtual filesystems |
| `exportfs`, `showmount`, `rpcinfo`, `nfsstat` | NFS |
| `mount -t nfs`, `mount -t cifs` | network filesystem clients |
| `nbd-server`, `nbd-client`, `modprobe nbd` | network block devices |
| `iscsiadm` | iSCSI client |
| `automount`, `/etc/auto.master.d/` | autofs |
| `x-systemd.automount` | systemd automount |
| `iostat -x`, `iotop`, `pidstat -d`, `fio` | storage performance |
| `man exports`, `man nfs`, `man autofs`, `man auto.master`, `man 5 nbd-server` | offline references |

---

➡ **Next:** [quiz](../quizzes/09-network-storage.md) → [lab](../labs/09-network-storage.md)
