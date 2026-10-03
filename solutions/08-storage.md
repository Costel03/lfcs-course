# Solutions 08 — Disks, filesystems, LVM and swap

Device names assume `/dev/sdb` and `/dev/sdc` are the spare disks.

---

## 🟢 Warm-up

**8.1**

```bash
lsblk -f
```

**8.2**

```bash
df -hT
findmnt -T /var/log
```

**8.3**

```bash
swapon --show
free -h
```

**8.4**

```bash
sudo blkid -s UUID -o value "$(findmnt -no SOURCE /)"
lsblk -no UUID "$(findmnt -no SOURCE /)"
```

If the root filesystem is on LVM, `findmnt` returns `/dev/mapper/…`, which both commands
accept.

---

## 🔵 Practical

**8.5**

```bash
sudo parted -s /dev/sdb mklabel gpt
sudo parted -s /dev/sdb mkpart data ext4 1MiB 1GiB
sudo parted -s /dev/sdb mkpart logs xfs 1GiB 2GiB
sudo parted -s /dev/sdb mkpart swap linux-swap 2GiB 2.5GiB
sudo parted -s /dev/sdb mkpart lvm 2.5GiB 100%
sudo parted -s /dev/sdb set 4 lvm on
sudo parted /dev/sdb print
lsblk /dev/sdb
```

With `fdisk`: `g`, then `n` four times (sizes `+1G`, `+1G`, `+512M`, Enter for the rest),
`t` to set partition 3 to `swap` and partition 4 to `lvm`, then `w`.

The type field in `parted` (`ext4`, `xfs`) only sets a hint in the partition table — it
doesn't create a filesystem.

**8.6**

```bash
sudo mkfs.ext4 -L DATA /dev/sdb1
sudo mkdir -p /mnt/data
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb1)  /mnt/data  ext4  defaults  0 2" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo findmnt --verify
sudo mount -a
findmnt /mnt/data
```

**8.7**

```bash
sudo mkfs.xfs -L LOGS /dev/sdb2
sudo mkdir -p /mnt/logs
echo 'LABEL=LOGS  /mnt/logs  xfs  noatime,nodev,nosuid  0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount -a
findmnt -o TARGET,OPTIONS /mnt/logs
```

**8.8**

```bash
sudo mkswap /dev/sdb3
sudo swapon /dev/sdb3
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb3)  none  swap  sw  0 0" | sudo tee -a /etc/fstab
sudo swapoff -a && sudo swapon -a && swapon --show
```

`swapon -a` is the swap equivalent of `mount -a`: it activates everything in fstab and is
the proof that your entry is right.

**8.9**

```bash
sudo fallocate -l 256M /swapfile2
sudo chmod 600 /swapfile2
sudo mkswap /swapfile2
sudo swapon /swapfile2
echo '/swapfile2  none  swap  sw  0 0' | sudo tee -a /etc/fstab
swapon --show
```

**8.10**

```bash
sudo mount -o remount,ro /mnt/data
sudo touch /mnt/data/x            # Read-only file system
sudo mount -o remount,rw /mnt/data
sudo touch /mnt/data/x
```

**8.11**

```bash
sudo e2label /dev/sdb1 DATA2                 # ext4: works mounted

sudo umount /mnt/logs
sudo xfs_admin -L APPLOGS /dev/sdb2          # xfs: must be unmounted
sudo sed -i 's/^LABEL=LOGS /LABEL=APPLOGS /' /etc/fstab
sudo systemctl daemon-reload
sudo findmnt --verify && sudo mount -a
lsblk -f /dev/sdb
```

`/mnt/data` is mounted by UUID, so its label change breaks nothing. `/mnt/logs` was
mounted by label, so the fstab line had to change too — the trap in this task.

**8.12**

```bash
sudo pvcreate /dev/sdb4 /dev/sdc
sudo vgcreate vgdata /dev/sdb4 /dev/sdc
sudo vgs vgdata
```

**8.13**

```bash
sudo lvcreate -n lvweb -L 2G vgdata
sudo lvcreate -n lvdb -l 50%FREE vgdata
sudo mkfs.ext4 /dev/vgdata/lvweb
sudo mkfs.xfs /dev/vgdata/lvdb
sudo mkdir -p /srv/web /srv/db
sudo tee -a /etc/fstab > /dev/null <<'EOF'
/dev/vgdata/lvweb  /srv/web  ext4  defaults  0 2
/dev/vgdata/lvdb   /srv/db   xfs   defaults  0 0
EOF
sudo systemctl daemon-reload && sudo mount -a
df -h /srv/web /srv/db
```

LVM paths are stable, so they're fine in fstab — UUIDs would work too.

**8.14**

```bash
echo '<h1>lab08</h1>' | sudo tee /srv/web/index.html
sudo lvextend -r -L +500M /dev/vgdata/lvweb
df -h /srv/web
```

**8.15**

```bash
sudo lvextend -L +1G /dev/vgdata/lvdb
sudo lvs vgdata; df -h /srv/db          # LV bigger, filesystem not
sudo xfs_growfs /srv/db                 # XFS takes the mount point
df -h /srv/db
```

**8.16**

```bash
sudo umount /srv/web
sudo lvreduce -r -L 1G /dev/vgdata/lvweb
sudo e2fsck -fn /dev/vgdata/lvweb
sudo mount /srv/web
cat /srv/web/index.html
```

`lvreduce -r` runs `e2fsck -f`, then `resize2fs` to shrink the filesystem to the new size,
and only then reduces the LV. Done in the other order — or without `-r` — the end of the
filesystem is simply cut off.

**8.17**

```bash
sudo lvcreate -s -n webSnap -L 300M /dev/vgdata/lvweb
echo 'changed' | sudo tee /srv/web/index.html
sudo mkdir -p /mnt/snap
sudo mount -o ro /dev/vgdata/webSnap /mnt/snap
sudo cp /mnt/snap/index.html /srv/web/index.html
sudo umount /mnt/snap
sudo lvremove -y /dev/vgdata/webSnap
cat /srv/web/index.html
```

To roll back *everything* instead of one file: `sudo lvconvert --merge
/dev/vgdata/webSnap`, then unmount `/srv/web` and re-activate the LV (`sudo lvchange -an
vgdata/lvweb && sudo lvchange -ay vgdata/lvweb`) for the merge to run.

---

## 🔴 Challenge

**8.18**

```bash
echo 'UUID=00000000-0000-0000-0000-000000000000  /mnt/ghost  ext4  defaults  0 2' | sudo tee -a /etc/fstab
sudo mkdir -p /mnt/ghost
sudo findmnt --verify         # [E] unreachable source: UUID=00000000-…
sudo mount -a; echo $?        # can't find UUID=…, non-zero
sudo sed -i 's#/mnt/ghost  ext4  defaults  0 2#/mnt/ghost  ext4  defaults,nofail  0 0#' /etc/fstab
sudo systemctl daemon-reload
sudo mount -a; echo $?        # 0
```

With `nofail` the boot continues without the mount; pass `0` avoids a pointless fsck
attempt. `findmnt --verify` still notes the missing source — it's a warning about reality,
not a syntax error. `nofail` is right for removable or optional storage, wrong for
something a service depends on: there you'd rather fail loudly.

**8.19**

```bash
sudo mount /mnt/data          # fails: can't find UUID=… (the UUID lives in the superblock)
sudo mke2fs -n /dev/sdb1      # DRY RUN: "Superblock backups stored on blocks: 32768, 98304, …"
sudo e2fsck -y -b 32768 /dev/sdb1
sudo mount /mnt/data
ls /mnt/data
```

Because the UUID is stored in the superblock, `blkid` can't see the filesystem at all until
it's repaired — that's why the error mentions the UUID rather than a bad superblock. `mke2fs
-n` computes where backups *would* be for a filesystem of this size made with default
options; it matches as long as the filesystem was created with defaults. `dumpe2fs` can't
help here because it needs the primary superblock.

**8.20**

```bash
sudo passwd root
echo '/dev/nosuchdisk  /mnt/broken  ext4  defaults  0 2' | sudo tee -a /etc/fstab
sudo reboot
```

The boot waits about 90 seconds for the device, then drops to emergency mode. In the
VirtualBox console, enter the root password, then:

```bash
mount -o remount,rw /          # root may be read-only in emergency mode
vi /etc/fstab                  # delete or fix the bad line (or add nofail)
systemctl daemon-reload
systemctl default              # continue to the normal target (or: reboot)
```

`journalctl -xb` in the emergency shell shows exactly which mount unit failed and why.

Without a root password — the default on Ubuntu cloud images and many Vagrant boxes — the
emergency shell refuses you: *Cannot open access to console, the root account is locked*.
Then you need the GRUB method from chapter 12 (`init=/bin/bash` or
`systemd.unit=…`). That's why the task sets a password first, and why you should know
both routes.

**8.21**

```bash
sudo dd if=/dev/zero of=/srv/web/big bs=1M status=none     # No space left on device
df -h /srv/web                                             # 100%
sudo vgs vgdata                                            # check VFree first
sudo lvextend -r -L +200M /dev/vgdata/lvweb
sudo fallocate -l 150M /srv/web/more
df -h /srv/web
```

**8.22**

```bash
sudo pvs                                   # PFree on sdc must exceed PSize−PFree of sdb4
sudo pvmove /dev/sdb4
sudo vgreduce vgdata /dev/sdb4
sudo pvremove /dev/sdb4
sudo pvs; sudo vgs vgdata
ls /srv/web /srv/db
```

`pvmove` copies extents to other PVs while the LVs stay in use, then switches the mapping.
It can take hours on real disks; it's restartable if interrupted (run `pvmove` again
with no arguments to resume).

**8.23**

```bash
yes | sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 "$L1" "$L2"
cat /proc/mdstat
sudo mkfs.ext4 -q /dev/md0
sudo mkdir -p /mnt/raid && sudo mount /dev/md0 /mnt/raid
echo important | sudo tee /mnt/raid/file

sudo mdadm /dev/md0 --fail "$L2"
cat /proc/mdstat                       # [U_] — degraded, still working
sudo mdadm /dev/md0 --remove "$L2"
sudo mdadm /dev/md0 --add "$L3"
watch -n1 cat /proc/mdstat             # recovery … then [UU]
sudo mdadm --detail /dev/md0
cat /mnt/raid/file
```

`yes |` answers mdadm's question about the metadata format. Note the file stayed readable
the whole time — that's the point of RAID: the failure is an alert, not an outage.

**8.24**

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

The `ARRAY` line records the array's UUID, so it's found whatever the member devices are
called. The initramfs update matters when the array holds a filesystem needed early in
boot. With loop devices, the members themselves don't exist at boot — `losetup` would have
to run first, which is why real arrays are built on partitions or whole disks.

**8.25**

```bash
truncate -s 100M ~/dup.img
L=$(sudo losetup -f --show ~/dup.img)
sudo mkfs.ext4 -q -L APPLOGS "$L"
sudo blkid -L APPLOGS                 # resolves to ONE of the two — which one is not guaranteed
sudo blkid | grep APPLOGS             # two devices
```

With duplicate labels, the device that answers to `LABEL=APPLOGS` depends on discovery
order — at boot, `/mnt/logs` could get the wrong filesystem with nothing to warn you.
Fix: make the label unique (`sudo e2label "$L" SCRATCH`), and make fstab refer to the XFS
filesystem by UUID instead of label. Then `sudo losetup -d "$L"`.

**8.26**

```bash
sudo mkdir -p /mnt/ram /var/www/site
sudo tee -a /etc/fstab > /dev/null <<'EOF'
tmpfs     /mnt/ram       tmpfs  size=64M,mode=1777  0 0
/srv/web  /var/www/site  none   bind                0 0
EOF
sudo systemctl daemon-reload && sudo mount -a
findmnt /mnt/ram /var/www/site
sudo touch /srv/web/bindtest && ls /var/www/site/bindtest
```

A bind mount shows the same directory tree in a second place — the same inodes, not a
copy. It's how you expose a directory inside a chroot or container, or give a path a
second name a program expects. `tmpfs` lives in RAM (and swap) and is empty after every
reboot.

**8.27**

```bash
# 1. stop using everything
sudo umount /mnt/data /mnt/logs /srv/web /srv/db /mnt/ram /var/www/site /mnt/raid 2>/dev/null
sudo swapoff /dev/sdb3 /swapfile2

# 2. remove every fstab line this lab added (inspect first!)
sudo grep -nE 'mnt/(data|logs|ghost|broken|ram)|srv/(web|db)|swapfile2|var/www/site|sdb3' /etc/fstab
sudo sed -i -E '/mnt\/(data|logs|ghost|broken|ram)|srv\/(web|db)|swapfile2|var\/www\/site/d' /etc/fstab
sudo sed -i "/$(sudo blkid -s UUID -o value /dev/sdb3)/d" /etc/fstab
sudo systemctl daemon-reload && sudo findmnt --verify

# 3. RAID and loops
sudo mdadm --stop /dev/md0
sudo sed -i '/^ARRAY \/dev\/md0/d' /etc/mdadm/mdadm.conf && sudo update-initramfs -u
sudo losetup -D; sudo rm -f /var/tmp/raid*.img ~/dup.img

# 4. LVM, top-down
sudo lvremove -y vgdata
sudo vgremove vgdata
sudo pvremove /dev/sdc

# 5. signatures and partitions
sudo wipefs -a /dev/sdb1 /dev/sdb2 /dev/sdb3
sudo wipefs -a /dev/sdb /dev/sdc
sudo rm -f /swapfile2
lsblk /dev/sdb /dev/sdc
sudo reboot
```

Order matters: unmount before removing, remove fstab lines before the volumes vanish,
LVs before VGs before PVs. `wipefs -a` erases filesystem, RAID, LVM and partition-table
signatures so nothing tries to assemble or mount them later.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | Device names follow detection order and can change; a UUID belongs to the filesystem | |
| 2 | >2 TiB disks; 128 partitions; backup table (any two) | |
| 3 | `lsblk -f` | |
| 4 | ext4 only, and only unmounted | XFS can only grow |
| 5 | `UUID=abcd-1234  /srv/data  xfs  noatime  0 0` | XFS pass is 0 |
| 6 | `findmnt --verify` (checks the file) and `mount -a` (actually mounts it) — plus `systemctl daemon-reload` | |
| 7 | Boot stops in emergency mode; `nofail` | |
| 8 | `noexec,nosuid,nodev` | |
| 9 | `fallocate` → `chmod 600` → `mkswap` → `swapon` → fstab | |
| 10 | Physical volume (a prepared disk/partition), volume group (pool), logical volume (a slice), physical extent (allocation unit) | |
| 11 | `-L` takes sizes, `-l` takes extents or percentages; `-l 50%FREE` is valid | |
| 12 | The filesystem wasn't resized; `resize2fs /dev/vg/lv` or `xfs_growfs <mountpoint>` — or use `lvextend -r` | |
| 13 | Checks and shrinks the filesystem first; without it you truncate the filesystem | |
| 14 | `pvmove /dev/<disk>` | then `vgreduce` and `pvremove` |
| 15 | It becomes invalid and unusable | |
| 16 | Only when it's unmounted (or mounted read-only, for checking) | |
| 17 | `mke2fs -n /dev/X` (or `dumpe2fs` if it still reads); `e2fsck -b <block> /dev/X` | `-n` makes it a dry run |
| 18 | Remounted it read-only to prevent more damage; `dmesg` / `journalctl -k` | |
| 19 | `--fail`, `--remove`, `--add` | |
| 20 | The mount fails; without `nofail` the boot drops to emergency mode | |
