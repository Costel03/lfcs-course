# Lab 08 — Disks, filesystems, LVM and swap

27 tasks on **ubuntu1**, using its two spare 5 GB disks. Every persistent task must pass
two checks: it works **now**, and `findmnt --verify` + `mount -a` / `swapon -a` succeed —
the proof it will survive a reboot. Do the quiz first.

**Take a snapshot first:** on your host, `vagrant snapshot save ubuntu1 pre-storage`.
Task 8.20 deliberately breaks the boot.

**Find your spare disks:**

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

The tasks call them **`/dev/sdb`** and **`/dev/sdc`** — the two 5 GB disks with nothing on
them. If yours have other names (`vdb`, `vdc`), substitute throughout. **Never** run these
commands against the disk holding `/`.

**Cleanup:** `vagrant snapshot restore ubuntu1 pre-storage` is the reliable way back. Task
8.27 practises tearing everything down by hand.

---

## 🟢 Warm-up

**8.1** Show all block devices as a tree with filesystem type, label, UUID and mount point.
Identify the root filesystem's device and the two spare disks.
*Verify:* the spare disks show no partitions and no filesystem.

**8.2** Show every mounted filesystem with its type and usage, then show only the mount that
holds `/var/log`.
*Verify:* two commands; the second uses `findmnt -T` or `df` on the path.

**8.3** Show the current swap devices and how much swap is in use.
*Verify:* `swapon --show` and `free -h` agree.

**8.4** Find the UUID of the root filesystem two different ways.
*Verify:* `blkid` and `lsblk -f` print the same value.

---

## 🔵 Practical

**8.5** Partition `/dev/sdb` with GPT as follows:

| Partition | Size | Purpose |
|---|---|---|
| 1 | 1 GiB | ext4 data |
| 2 | 1 GiB | XFS logs |
| 3 | 512 MiB | swap |
| 4 | the rest | LVM |

*Verify:* `lsblk /dev/sdb` shows four partitions of those sizes; `sudo parted /dev/sdb print`
shows a `gpt` table, and partition 4 has the `lvm` flag (or the Linux LVM type in `fdisk`).

**8.6** Create an ext4 filesystem labelled `DATA` on `sdb1`, and mount it on `/mnt/data`
persistently **by UUID**.
*Verify:* `findmnt /mnt/data`; the fstab line starts with `UUID=`; `findmnt --verify` is clean.

**8.7** Create an XFS filesystem labelled `LOGS` on `sdb2`, and mount it on `/mnt/logs`
persistently by **label**, with `noatime`, `nodev` and `nosuid`.
*Verify:* `findmnt -o TARGET,OPTIONS /mnt/logs` shows the three options.

**8.8** Turn `sdb3` into swap, active now and at boot.
*Verify:* `swapon --show` lists it; `sudo swapoff -a && sudo swapon -a && swapon --show` brings it back.

**8.9** Add a 256 MiB swap **file** `/swapfile2`, active now and at boot, with safe
permissions.
*Verify:* `swapon --show` lists both swap areas; `ls -l /swapfile2` shows `-rw-------`.

**8.10** Remount `/mnt/data` read-only without unmounting it, prove writes fail, then make it
read-write again.
*Verify:* `touch /mnt/data/x` fails with *Read-only file system*, then succeeds.

**8.11** Change the `DATA` label to `DATA2` and the `LOGS` label to `APPLOGS`. Fix anything
that would break at boot as a result.
*Verify:* `lsblk -f` shows the new labels; `findmnt --verify` is clean; `mount -a` works.

**8.12** Turn `sdb4` and the whole of `sdc` into physical volumes, and create a volume group
`vgdata` from them.
*Verify:* `sudo vgs vgdata` shows 2 PVs and about 7.4 GiB.

**8.13** Create `lvweb` (2 GiB, ext4) mounted on `/srv/web`, and `lvdb` (50% of the
remaining free space, XFS) mounted on `/srv/db`, both persistent.
*Verify:* `sudo lvs` shows both; `df -h /srv/web /srv/db`; `mount -a` is clean.

**8.14** Put a file `/srv/web/index.html` in place, then grow `lvweb` by 500 MiB **while
mounted**, filesystem included, in one command.
*Verify:* `df -h /srv/web` shows ~2.5 GiB; the file is still there.

**8.15** Grow `lvdb` by 1 GiB in two separate steps: first the LV, then the filesystem.
*Verify:* `sudo lvs` grew after step one, `df -h /srv/db` only after step two.

**8.16** Shrink `lvweb` to exactly 1 GiB, safely.
*Verify:* `sudo lvs` shows `1.00g`; `index.html` is intact; `sudo e2fsck -fn /dev/vgdata/lvweb`
(run while unmounted) reports no errors.

**8.17** Take a 300 MiB snapshot of `lvweb`. Change `index.html`, then retrieve the original
version from the snapshot and remove the snapshot.
*Verify:* `index.html` has its original content; `sudo lvs` no longer lists the snapshot.

---

## 🔴 Challenge

**8.18 — Catch it before the reboot.** Add this line to `/etc/fstab`:

```text
UUID=00000000-0000-0000-0000-000000000000  /mnt/ghost  ext4  defaults  0 2
```

Show two different commands that would have warned you. Then change the line so a missing
device can never stop the machine from booting, and confirm with the same commands.
*Verify:* `findmnt --verify` reports the problem before your fix; after the fix `mount -a`
returns exit status 0.

**8.19 — Read it from the kernel.** Remove the ghost line. Unmount `/mnt/data` and corrupt
its first superblock deliberately:

```bash
sudo umount /mnt/data
sudo dd if=/dev/zero of=/dev/sdb1 bs=1024 seek=1 count=1
sudo mount /mnt/data
```

Repair it using a backup superblock and remount it. Your files must survive.
*Verify:* the first `mount` fails; after repair `mount /mnt/data` works and `ls /mnt/data` is intact.

**8.20 — Emergency mode.** Set a root password first (`sudo passwd root`). Add a bad fstab
entry **without** `nofail` and reboot. Open the VM's console from the VirtualBox window —
SSH won't be available. Repair the system from emergency mode and boot normally.
*Verify:* after your repair, `systemctl is-system-running` reports `running` (or
`degraded` only for unrelated units), and `vagrant ssh ubuntu1` works again.

**8.21 — Disk full, live.** Fill `/srv/web` until writes fail:

```bash
sudo dd if=/dev/zero of=/srv/web/big bs=1M status=none    # stops at "No space left on device"
```

Without unmounting, and without deleting anything, make room for 200 MiB more.
*Verify:* `sudo fallocate -l 150M /srv/web/more` succeeds; `df -h /srv/web` shows the larger size.

**8.22 — Retire a disk.** `sdb4` is "failing". Move every extent off it while `/srv/web` and
`/srv/db` stay mounted, then remove it from the volume group and erase its LVM label.
Check first that the remaining disk has enough free space.
*Verify:* `sudo pvs` no longer lists `sdb4`; `sudo vgs vgdata` shows 1 PV; both
filesystems still mounted and readable.

**8.23 — RAID 1 and a failing disk.** Using loop devices so your spare disks stay as they are:

```bash
for i in 1 2 3; do sudo truncate -s 300M /var/tmp/raid$i.img; done
L1=$(sudo losetup -f --show /var/tmp/raid1.img)
L2=$(sudo losetup -f --show /var/tmp/raid2.img)
L3=$(sudo losetup -f --show /var/tmp/raid3.img)
echo "$L1 $L2 $L3"
```

Create a RAID 1 array `/dev/md0` from the first two, put ext4 on it, mount it and write a
file. Then fail one member, remove it, add the third as a replacement and watch the
rebuild.
*Verify:* `cat /proc/mdstat` shows `[UU]` at the end; `mdadm --detail /dev/md0` lists the
third loop device as active; your file is intact.

**8.24 — Persist the array.** Make `/dev/md0` assemble automatically at boot (on real
disks; with loop devices, write the configuration and explain what would still be missing).
*Verify:* `/etc/mdadm/mdadm.conf` contains an `ARRAY` line for md0; you ran the
initramfs update.

**8.25 — Labels vs UUIDs.** Create a second filesystem labelled `APPLOGS` on a loop file
(`truncate -s 100M`, `losetup -f --show`, `mkfs.ext4 -L APPLOGS`). `LABEL=APPLOGS` in
fstab is now ambiguous. Show which device the system resolves the label to, explain why
that's dangerous at boot, and remove the ambiguity.
*Verify:* `sudo blkid -L APPLOGS` before and after; at the end `lsblk -f` shows one
`APPLOGS` and fstab no longer depends on a label that could be duplicated.

**8.26 — Special mounts.** Add two persistent mounts that have no disk behind them:
a 64 MiB `tmpfs` at `/mnt/ram` writable by everyone with the sticky bit, and a **bind
mount** that makes `/srv/web` also appear at `/var/www/site`.
*Verify:* `findmnt /mnt/ram` shows `tmpfs` with `size=65536k`; a file created in
`/srv/web` is visible in `/var/www/site`; `mount -a` clean.

**8.27 — Tear it all down.** Remove everything you built in this lab — mounts, fstab lines,
swap, LVs, VG, PVs, partitions, the array and loop devices — leaving the machine able to
reboot cleanly.
*Verify:* `lsblk /dev/sdb /dev/sdc` shows bare disks; `findmnt --verify` clean;
`swapon --show` shows only the original swap; `sudo reboot` comes back without errors.

---

## Self-check

- [ ] I identify disks with `lsblk -f` before every destructive command
- [ ] I can partition with `fdisk` and with `parted -s`
- [ ] I write fstab lines by UUID and test them with `findmnt --verify` and `mount -a`
- [ ] I can build, grow, shrink, snapshot and move LVM volumes
- [ ] I know which filesystems shrink, which tool grows which, and what `-r` does
- [ ] I can add swap two ways and make it persistent
- [ ] I can repair a filesystem from a backup superblock and replace a RAID member
- [ ] I can get a machine out of emergency mode

➡ **Solutions:** [solutions/08-storage.md](../solutions/08-storage.md)
