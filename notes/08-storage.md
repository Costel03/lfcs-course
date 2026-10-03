# Chapter 08 — Disks, filesystems, LVM and swap

Storage tasks are where exam candidates lose the most marks to *persistence*: the
filesystem mounts fine now, but there's no `/etc/fstab` entry, or the entry has a typo and
the machine won't boot. This chapter builds the whole stack — disk, partition, LVM,
filesystem, mount, swap — and the habit of verifying each layer.

LFCS: *Storage* — configure and manage LVM; create, manage and troubleshoot filesystems;
configure and manage swap. *Operations* — recover from hardware and filesystem failures.

## Objectives

After this chapter you can:

- Identify disks, partitions and filesystems, and refer to them in a way that survives reboots.
- Partition a disk with GPT using `fdisk` or `parted`.
- Create, label, check, grow and (for ext4) shrink filesystems.
- Mount filesystems with the right options, persistently, and test `/etc/fstab` before rebooting.
- Build, extend, reduce, snapshot and dismantle LVM volumes.
- Add swap as a partition or a file.
- Repair a corrupted filesystem and replace a failed disk in a RAID 1 mirror.

---

## 1. The storage stack

```text
 filesystem  (ext4, xfs)        ← mounted at a directory, e.g. /srv/web
     │
 logical volume (optional)      ← /dev/vgdata/lvweb   — LVM
 volume group                   ← a pool built from physical volumes
 physical volume
     │
 partition                      ← /dev/sdb1
     │
 disk                           ← /dev/sdb (SATA/SCSI), /dev/vdb (virtio), /dev/nvme0n1
```

Every layer has its own command to inspect it:

```bash
lsblk                     # the tree of disks, partitions, LVs, mount points
lsblk -f                  # with filesystem type, label, UUID
sudo blkid                # UUIDs and types of every block device
df -hT                    # mounted filesystems, type and usage
findmnt                   # mount tree
sudo pvs; sudo vgs; sudo lvs     # LVM layers
swapon --show
```

### Naming that survives reboots

`/dev/sdb` is assigned in detection order and **can change** when disks are added or a
controller is slower to start. Persistent names:

| Name | Example | Stable when |
|---|---|---|
| UUID | `UUID=3e6be9de-…` | always — it belongs to the filesystem |
| Label | `LABEL=DATA` | while labels are unique |
| by-id | `/dev/disk/by-id/ata-VBOX_HARDDISK_VB1234…` | per physical disk |
| LVM path | `/dev/vgdata/lvweb`, `/dev/mapper/vgdata-lvweb` | always — LVM names are stable |

Use UUIDs (or LVM paths) in `/etc/fstab`.

---

## 2. Partitioning

### MBR or GPT

| | MBR (msdos) | GPT |
|---|---|---|
| Max disk size | 2 TiB | effectively unlimited |
| Partitions | 4 primary (or extended + logical) | 128 by default |
| Redundancy | none | backup table at end of disk |
| Use | legacy BIOS only | **everything new** |

### fdisk — interactive

```bash
sudo fdisk /dev/sdb
```

| Key | Action |
|---|---|
| `g` | new GPT label (`o` for MBR) |
| `n` | new partition — accept the default start; size like `+1G` |
| `t` | change type — `swap`, `lvm`, `linux` (type `L` to list) |
| `p` | print |
| `d` | delete |
| `w` | **write** and exit |
| `q` | quit without saving |

Nothing changes until `w`. If you made a mess, `q`.

### parted — scriptable

```bash
sudo parted -s /dev/sdb mklabel gpt
sudo parted -s /dev/sdb mkpart data ext4 1MiB 1GiB
sudo parted -s /dev/sdb mkpart swap linux-swap 1GiB 1.5GiB
sudo parted -s /dev/sdb mkpart lvm 1.5GiB 100%
sudo parted -s /dev/sdb set 3 lvm on
sudo parted /dev/sdb print
```

Start the first partition at **1 MiB** — it aligns partitions with the physical sectors of
modern disks; misaligned partitions can halve write performance.

After changing a disk that's in use, tell the kernel: `sudo partprobe /dev/sdb`. Check with
`lsblk`.

---

## 3. Filesystems

| | ext4 | XFS | Btrfs |
|---|---|---|---|
| Default on | Debian, Ubuntu | RHEL, Rocky | Fedora, openSUSE |
| Grow | online | online | online |
| **Shrink** | **offline** | **never** | online |
| Repair tool | `e2fsck` | `xfs_repair` | `btrfs check` |
| Label tool | `e2label`, `tune2fs -L` | `xfs_admin -L` (unmounted) | `btrfs filesystem label` |
| Strengths | mature, shrinkable | large files, parallel I/O | snapshots, checksums |

```bash
sudo mkfs.ext4 -L DATA /dev/sdb1
sudo mkfs.xfs  -L LOGS /dev/sdb2
sudo mkfs.ext4 -q -N 100000 /dev/sdX      # set the number of inodes (ch. 02)

sudo tune2fs -l /dev/sdb1 | less          # every ext4 parameter
sudo dumpe2fs -h /dev/sdb1                # header only
sudo xfs_info /mnt/logs                   # XFS geometry (mounted)
sudo e2label /dev/sdb1 DATA2              # rename an ext4 label
```

`mkfs` destroys whatever was there. Check `lsblk -f` first — if it shows a filesystem or a
mount point, stop and think.

---

## 4. Mounting

```bash
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
sudo mount -o ro,noexec /dev/sdb1 /mnt/data
sudo mount -o remount,ro /mnt/data        # change options on a mounted filesystem
sudo umount /mnt/data
findmnt /mnt/data
findmnt -T /some/path                     # which filesystem holds this path?
```

| Option | Effect |
|---|---|
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` |
| `ro` / `rw` | read-only / read-write |
| `noexec` | can't execute binaries from it |
| `nosuid` | SUID/SGID bits ignored |
| `nodev` | device files ignored |
| `noatime` | don't update access times — less write I/O |
| `nofail` | don't fail the boot if this device is missing |
| `x-systemd.automount` | mount on first access (chapter 09) |
| `_netdev` | needs the network — wait for it (chapter 09) |

`noexec,nosuid,nodev` on `/tmp`, `/var/tmp`, removable media and any data-only filesystem
is a common hardening baseline.

---

## 5. /etc/fstab

```text
# <device>                                 <mount point>  <type>  <options>              <dump> <pass>
UUID=3e6be9de-8f2a-4b1e-9d3a-0c1f7e5a2b11  /mnt/data      ext4    defaults               0      2
LABEL=LOGS                                 /mnt/logs      xfs     noatime,nodev,nosuid   0      0
/dev/vgdata/lvweb                          /srv/web       ext4    defaults               0      2
UUID=7d0c…                                 none           swap    sw                     0      0
```

| Field | Meaning |
|---|---|
| device | `UUID=`, `LABEL=`, or an LVM path |
| mount point | a directory that exists; `none` for swap |
| type | `ext4`, `xfs`, `swap`, `nfs`, `tmpfs`… |
| options | comma-separated, no spaces |
| dump | legacy backup flag — `0` |
| pass | fsck order at boot: `1` root, `2` others, `0` never (XFS: always 0) |

### Always test before rebooting

```bash
sudo findmnt --verify            # syntax, devices that don't exist, unknown types
sudo systemctl daemon-reload     # systemd turns fstab into .mount units — refresh them
sudo mount -a                    # mount everything listed that isn't mounted
findmnt /mnt/data
```

A bad entry without `nofail` drops the next boot into **emergency mode** — on a remote
server, that's an outage you can only fix from a console. `mount -a` succeeding is the
minimum bar before you reboot. Chapter 12 covers recovering when you didn't.

Get the UUID without typing it:

```bash
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb1)  /mnt/data  ext4  defaults  0 2" | sudo tee -a /etc/fstab
```

---

## 6. Swap

Swap gives the kernel somewhere to put memory pages that haven't been used recently, so
RAM is free for active work. It is not a substitute for enough memory.

### A swap partition

```bash
sudo mkswap -L SWAP2 /dev/sdb3
sudo swapon /dev/sdb3
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb3)  none  swap  sw  0 0" | sudo tee -a /etc/fstab
swapon --show
```

### A swap file

```bash
sudo fallocate -l 512M /swapfile2        # or: dd if=/dev/zero of=/swapfile2 bs=1M count=512
sudo chmod 600 /swapfile2                # swapon warns if others can read it
sudo mkswap /swapfile2
sudo swapon /swapfile2
echo '/swapfile2  none  swap  sw  0 0' | sudo tee -a /etc/fstab
```

On filesystems where `fallocate` produces files with holes (some XFS and Btrfs setups),
`swapon` refuses the file — use `dd` instead.

```bash
sudo swapoff /swapfile2          # stop using it (needs enough free RAM)
cat /proc/swaps
sudo swapon -a                   # activate everything in fstab — the test for swap entries
```

`swapon --priority` and `pri=` in fstab decide which swap is used first. How eagerly the
kernel swaps is the `vm.swappiness` sysctl (chapter 12).

---

## 7. LVM

LVM puts a flexible layer between disks and filesystems: volumes can grow across disks,
move between disks while mounted, and be snapshotted.

```text
  /dev/sdb4 ─┐                      ┌─ lvweb  (2 GiB) → ext4 → /srv/web
             ├─ PVs ─▶ VG vgdata ─▶ ┤
  /dev/sdc  ─┘   (pool of extents)  └─ lvdb   (3 GiB) → xfs  → /srv/db
```

| Term | Is |
|---|---|
| PV — physical volume | a disk or partition prepared for LVM |
| VG — volume group | a pool of storage from one or more PVs |
| LV — logical volume | a slice of a VG, used like a partition |
| PE — physical extent | the allocation unit, 4 MiB by default |

### Build

```bash
sudo pvcreate /dev/sdb4 /dev/sdc
sudo vgcreate vgdata /dev/sdb4 /dev/sdc
sudo lvcreate -n lvweb -L 2G vgdata
sudo lvcreate -n lvdb  -l 50%FREE vgdata       # -l: extents or percentages
sudo mkfs.ext4 /dev/vgdata/lvweb
sudo mkfs.xfs  /dev/vgdata/lvdb
sudo pvs; sudo vgs; sudo lvs
```

`-L` takes a size (`500M`, `2G`, `+1G`); `-l` takes extents or percentages (`100%FREE`,
`50%VG`).

### Grow — online

```bash
sudo lvextend -r -L +1G /dev/vgdata/lvweb         # -r: resize the filesystem too
sudo lvextend -r -l +100%FREE /dev/vgdata/lvdb
```

`-r` calls `resize2fs` or `xfs_growfs` for you. Without it, the LV grows but the
filesystem — and `df` — stays the same size; then run the resize tool by hand:

```bash
sudo resize2fs /dev/vgdata/lvweb       # ext4: device path
sudo xfs_growfs /srv/db                # xfs: MOUNT POINT
```

Out of space in the VG? Add a disk:

```bash
sudo pvcreate /dev/sdd
sudo vgextend vgdata /dev/sdd
```

### Shrink — ext4 only, offline

```bash
sudo umount /srv/web
sudo lvreduce -r -L 1G /dev/vgdata/lvweb      # -r runs e2fsck and resize2fs first
sudo mount /srv/web
```

Shrinking without `-r` cuts the end off a filesystem that still uses it — data loss.
**XFS cannot be shrunk**; the only way is back up, recreate smaller, restore.

### Move data off a disk — online

```bash
sudo pvmove /dev/sdb4                  # move all extents to other PVs in the VG
sudo vgreduce vgdata /dev/sdb4         # take the PV out of the VG
sudo pvremove /dev/sdb4
```

This is how you replace a disk that's showing errors, with the filesystem mounted and in
use.

### Snapshots

```bash
sudo lvcreate -s -n webSnap -L 500M /dev/vgdata/lvweb      # copy-on-write snapshot
sudo mount -o ro /dev/vgdata/webSnap /mnt/snap             # browse the frozen state
sudo lvconvert --merge /dev/vgdata/webSnap                 # roll the origin back to it
```

A snapshot stores only blocks that change after it is taken. If more than its size
changes, it becomes **invalid** — size it for the expected change rate, and don't keep
them for weeks. A merge into a mounted origin is deferred until the origin is next
activated (unmount and `lvchange -an`/`-ay`, or reboot).

### Rename and remove

```bash
sudo lvrename vgdata lvweb lvsite
sudo umount /srv/web && sudo lvremove /dev/vgdata/lvweb
sudo vgremove vgdata
sudo pvremove /dev/sdb4 /dev/sdc
```

Remove the `/etc/fstab` line **first**, or the next boot fails on it.

---

## 8. Checking and repairing filesystems

Filesystems get damaged by power loss, failing hardware and kernel bugs. The kernel then
often **remounts the filesystem read-only** (`errors=remount-ro`) to stop further damage.
Signs: *Read-only file system* errors, and messages in `dmesg`.

```bash
dmesg -T | grep -iE 'ext4|xfs|i/o error|remount'
```

Repair **only unmounted** filesystems:

```bash
sudo umount /mnt/data
sudo e2fsck -f /dev/sdb1          # ext4: -f forces a check; -y answers yes to everything
sudo xfs_repair /dev/sdb2         # xfs
```

Running `fsck` on a mounted read-write filesystem can destroy it.

For the root filesystem, which can't be unmounted while running, force a check at the
next boot: on systemd systems add `fsck.mode=force` to the kernel command line once
(chapter 12), or boot rescue media.

### ext4's backup superblocks

If the primary superblock is damaged the filesystem won't mount or check. ext4 keeps
copies:

```bash
sudo mke2fs -n /dev/sdb1          # -n: DRY RUN — prints where the backups are, writes nothing
sudo dumpe2fs /dev/sdb1 | grep -i 'backup superblock'
sudo e2fsck -b 32768 /dev/sdb1    # repair using a backup
```

The `-n` on `mke2fs` matters more than any other flag in this chapter. Without it, you've
just formatted the disk you were trying to rescue.

### Disk health

```bash
sudo apt install smartmontools
sudo smartctl -H /dev/sda         # overall health (not available on most virtual disks)
sudo smartctl -a /dev/sda         # reallocated sectors, pending sectors, error log
```

---

## 9. Software RAID — surviving a disk failure

LVM pools disks; RAID protects against losing one. Linux software RAID is `mdadm`:

| Level | Disks | Survives | Capacity |
|---|---|---|---|
| RAID 0 | 2+ | nothing | sum of all |
| RAID 1 | 2+ | all but one | one disk |
| RAID 5 | 3+ | one | n − 1 |
| RAID 6 | 4+ | two | n − 2 |
| RAID 10 | 4+ | one per mirror | half |

```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
cat /proc/mdstat                          # state and resync progress
sudo mdadm --detail /dev/md0
sudo mkfs.ext4 /dev/md0

# persist the array definition, then rebuild the initramfs so it assembles at boot
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf     # /etc/mdadm.conf on RedHat
sudo update-initramfs -u                                           # dracut -f on RedHat
```

Replacing a failed disk:

```bash
sudo mdadm /dev/md0 --fail /dev/sdc1      # (the kernel does this itself on real failure)
sudo mdadm /dev/md0 --remove /dev/sdc1
# physically replace, partition the new disk identically
sudo mdadm /dev/md0 --add /dev/sdd1
watch cat /proc/mdstat                    # resync
```

RAID is not a backup. It protects against a disk dying, not against `rm -rf`, corruption,
or ransomware — those replicate to both halves instantly.

---

## 10. Gotchas

**Persistence.** The mount works, but you didn't add it to `/etc/fstab`. Or you added it
but never ran `mount -a`. Exam scripts check after a reboot.

**`/dev/sdX` in fstab** works today and breaks the day a disk is added.

**Forgetting `-r` on `lvextend`.** The LV is bigger; `df` isn't. Run `resize2fs` or
`xfs_growfs`.

**`xfs_growfs` takes the mount point**, `resize2fs` takes the device.

**Shrinking XFS** — impossible. Shrinking ext4 without `-r` — data loss.

**Removing a volume but not its fstab line** — next boot goes to emergency mode.

**`umount: target is busy`** — someone's shell is inside it (`fuser -vm`, chapter 05).

**Running `mkfs` on the wrong device.** `lsblk -f` before every `mkfs`, every time.

---

## 11. Commands introduced

| Command | Purpose |
|---|---|
| `lsblk -f`, `blkid`, `findmnt`, `df -hT` | inspect devices and mounts |
| `fdisk`, `gdisk`, `parted`, `partprobe` | partitioning |
| `mkfs.ext4`, `mkfs.xfs`, `tune2fs`, `dumpe2fs`, `e2label`, `xfs_admin`, `xfs_info` | filesystems |
| `mount`, `umount`, `mount -a`, `findmnt --verify` | mounting |
| `mkswap`, `swapon`, `swapoff` | swap |
| `pvcreate/pvs/pvmove/pvremove`, `vgcreate/vgextend/vgreduce/vgs`, `lvcreate/lvextend/lvreduce/lvs/lvremove` | LVM |
| `resize2fs`, `xfs_growfs` | grow filesystems |
| `e2fsck`, `xfs_repair`, `mke2fs -n` | check and repair |
| `mdadm`, `/proc/mdstat` | software RAID |
| `smartctl` | disk health |
| `man fstab`, `man lvm`, `man lvmraid`, `man mdadm` | offline references |

---

➡ **Next:** [quiz](../quizzes/08-storage.md) → [lab](../labs/08-storage.md)
