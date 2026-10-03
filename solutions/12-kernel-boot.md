# Solutions 12 — Kernel, boot and recovery

---

## 🟢 Warm-up

**12.1**

```bash
sysctl vm.swappiness net.ipv4.ip_forward fs.file-max
cat /proc/sys/vm/swappiness /proc/sys/net/ipv4/ip_forward /proc/sys/fs/file-max
```

**12.2**

```bash
ls /etc/sysctl.d /run/sysctl.d /usr/lib/sysctl.d 2>/dev/null
grep -r swappiness /etc/sysctl.d /usr/lib/sysctl.d /etc/sysctl.conf 2>/dev/null
```

Usually nothing sets it, and you get the kernel default of 60.

**12.3**

```bash
cat /proc/cmdline
uname -r
```

**12.4**

```bash
lsmod | head
modinfo loop
```

**12.5**

```bash
dpkg -l 'linux-image-*' | grep ^ii
grep -E "^\s*(menuentry|submenu) '" /boot/grub/grub.cfg | cut -d"'" -f2
```

---

## 🔵 Practical

**12.6**

```bash
sudo sysctl -w vm.swappiness=15
sysctl vm.swappiness           # 15
sudo reboot
sysctl vm.swappiness           # 60 again
```

**12.7**

```bash
sudo tee /etc/sysctl.d/90-lab12.conf > /dev/null <<'EOF'
vm.swappiness = 15
net.ipv4.icmp_echo_ignore_all = 1
EOF
sudo sysctl --system | tail -5
sysctl vm.swappiness net.ipv4.icmp_echo_ignore_all
```

From ubuntu2 `ping -c2 ubuntu1` now gets no answer. Set the icmp line back to `0` and run
`sysctl --system` again when you're done.

**12.8**

```bash
echo 'vm.swappiness = 80' | sudo tee /etc/sysctl.d/05-swap.conf
sudo sysctl --system | grep -E 'Applying|swappiness'
sysctl vm.swappiness           # 15
```

Files are applied in lexical order: `05-swap.conf` sets 80, later `90-lab12.conf` sets 15,
and the last write wins. `sudo rm /etc/sysctl.d/05-swap.conf` afterwards.

**12.9**

```bash
sudo modprobe nbd nbds_max=2
cat /sys/module/nbd/parameters/nbds_max
ls /dev/nbd*
sudo modprobe -r nbd
lsmod | grep nbd
```

If `nbd` is still loaded from chapter 09, `modprobe` with a parameter does nothing — the
module is already in memory. Unload it first.

**12.10**

```bash
echo nbd | sudo tee /etc/modules-load.d/nbd.conf
echo 'options nbd nbds_max=4' | sudo tee /etc/modprobe.d/nbd.conf
sudo reboot
cat /sys/module/nbd/parameters/nbds_max       # 4
```

**12.11**

```bash
sudo tee /etc/modprobe.d/cramfs.conf > /dev/null <<'EOF'
blacklist cramfs
install cramfs /bin/false
EOF
sudo modprobe cramfs; echo $?         # non-zero
modprobe -n -v cramfs                 # install /bin/false
```

`-n -v` is a dry run that prints what modprobe *would* do — the right way to check module
configuration.

**12.12**

```bash
sudoedit /etc/default/grub
#   GRUB_TIMEOUT_STYLE=menu
#   GRUB_TIMEOUT=5
#   GRUB_CMDLINE_LINUX="audit=1"
sudo update-grub
sudo reboot
cat /proc/cmdline
```

`GRUB_CMDLINE_LINUX` applies to every entry including recovery; `…_DEFAULT` only to normal
entries.

**12.13**

At the GRUB menu: `e`, delete `quiet splash` from the `linux` line, Ctrl-X. Nothing was saved,
so the next normal boot uses the configured line again. Vagrant boxes often have `quiet`
already removed — the point is the edit-and-boot procedure.

**12.14**

```bash
ls -l /boot/initrd.img-$(uname -r)
sudo update-initramfs -u
ls -l /boot/initrd.img-$(uname -r)
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'kernel/drivers|scripts/local' | head
```

**12.15**

On the console:

```bash
sudo systemctl isolate rescue.target
```

On an Ubuntu box with a locked root account you'll see *Cannot open access to console, the
root account is locked* and an offer to press Enter — which continues to the default
target. With a root password, you get a single-user shell; leave with `systemctl default`.

**12.16**

```bash
systemd-analyze
systemd-analyze critical-chain
last -x | head
```

`last -x` shows `reboot` and `shutdown` records. A `reboot` with no preceding `shutdown`
means the previous boot ended without a clean shutdown — a crash or a power cut.

---

## 🔴 Challenge

**12.17**

```bash
sudo passwd vagrant           # set something, then forget it
sudo reboot
```

On the console, at the GRUB menu: `e`, on the `linux` line replace `ro` (and `quiet splash`)
with `rw init=/bin/bash`, Ctrl-X. Then:

```bash
mount -o remount,rw /        # harmless if already rw
passwd vagrant
sync
exec /sbin/init
```

`passwd -S vagrant` then shows `P` and today's date. `exec /sbin/init` replaces the bash
shell (PID 1) with systemd, continuing the boot without a reset.

**12.18**

First attempt: GRUB → `e` → append `systemd.unit=emergency.target` → Ctrl-X → *the root
account is locked*. Emergency and rescue targets require root's password through
`sulogin`, and this box has none.

```bash
sudo passwd root             # set one, reboot, repeat the GRUB edit
```

In the emergency shell:

```bash
findmnt -no OPTIONS /        # ro,…
mount -o remount,rw /
findmnt -no OPTIONS /        # rw,…
systemctl default            # continue the boot
```

Decide afterwards whether root should keep a password. With it, console recovery is
easier; without it, only `init=/bin/bash` works — which needs console access anyway.

**12.19**

```bash
sudo mkdir -p /mnt/sysroot
sudo mount --bind / /mnt/sysroot
findmnt /boot     >/dev/null && sudo mount --bind /boot     /mnt/sysroot/boot
findmnt /boot/efi >/dev/null && sudo mount --bind /boot/efi /mnt/sysroot/boot/efi
for d in dev proc sys run; do
  sudo mount --rbind /$d /mnt/sysroot/$d
  sudo mount --make-rslave /mnt/sysroot/$d
done
sudo chroot /mnt/sysroot /bin/bash
update-grub
update-initramfs -u
exit
sudo umount -R /mnt/sysroot
findmnt -R /mnt/sysroot      # nothing
```

A plain `--bind /` does **not** include filesystems mounted under it — so a separate
`/boot` must be bound explicitly, or `update-grub` writes its configuration into the empty
`/boot` directory on the root filesystem, where the bootloader never looks. Check with
`findmnt` every time.

**12.20**

```bash
# beforehand, note the root device:
findmnt -no SOURCE /            # e.g. /dev/sda1, or /dev/mapper/ubuntu--vg-ubuntu--lv
sudo mv /boot/grub/grub.cfg /boot/grub/grub.cfg.bak
sudo reboot
```

At the `grub>` prompt (BIOS/GPT layout shown; yours may differ):

```text
grub> ls
(hd0) (hd0,gpt1) (hd0,gpt2)
grub> ls (hd0,gpt2)/
lost+found/ boot/ etc/ … vmlinuz initrd.img …
grub> set root=(hd0,gpt2)
grub> linux /boot/vmlinuz root=/dev/sda2 ro
grub> initrd /boot/initrd.img
grub> boot
```

If `/boot` is a separate partition, the paths are `/vmlinuz` and `/initrd.img` on *that*
partition, and `root=` still names the root filesystem. For LVM root, `insmod lvm` first
and use `root=/dev/mapper/…`. Tab completion works at the `grub>` prompt — use it to find
files.

Once booted:

```bash
sudo update-grub             # regenerates grub.cfg
sudo reboot                  # normal boot
```

**12.21** (rocky1, console)

GRUB → `e` → append `rd.break` to the `linux` line → Ctrl-X.

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
```

The next boot runs a full SELinux relabel (a few minutes, then an automatic reboot). The
faster alternative inside the chroot, before exiting: `load_policy -i` then
`restorecon -v /etc/shadow`, which fixes only the file `passwd` changed.

**12.22**

```bash
sudo tee /usr/local/bin/hwreport > /dev/null <<'EOF'
#!/usr/bin/env bash
echo "== Kernel errors this boot";  journalctl -k -p err -b --no-pager | tail -10
echo "== Failed units";             systemctl --failed --no-legend
echo "== SMART";                    for d in $(lsblk -dno NAME -e7,11); do
                                      printf '%s: ' "$d"
                                      smartctl -H /dev/$d 2>/dev/null | grep -E 'result|Status' || echo unsupported
                                    done
echo "== RAID";                     cat /proc/mdstat 2>/dev/null | grep -E '^md|\[' || echo none
echo "== Read-only mounts";         findmnt -rno TARGET,OPTIONS -t ext4,xfs,btrfs | awk '$2 ~ /^ro/'
echo "== Recent reboots";           last -x reboot | head -3
EOF
sudo chmod 755 /usr/local/bin/hwreport
sudo apt install -y smartmontools && sudo hwreport
```

`-e7,11` excludes loop devices (major 7) and optical drives (11) from the disk list.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | `-w` changes the running kernel only; the file is applied at every boot | |
| 2 | `sysctl --system` | |
| 3 | `90-b.conf` | Files apply in lexical order; the last write wins |
| 4 | `/proc/sys/net/ipv4/ip_forward` | Dots become slashes |
| 5 | `modprobe` resolves dependencies and reads `/etc/modprobe.d` | |
| 6 | `/etc/modules-load.d/nbd.conf` (`nbd`) and `/etc/modprobe.d/nbd.conf` (`options nbd nbds_max=4`) | |
| 7 | `blacklist` only stops alias-based auto-loading; `install usb-storage /bin/false` blocks all loads | |
| 8 | `/proc/cmdline` | |
| 9 | `update-grub`; `_LINUX` applies to all entries including recovery, `_DEFAULT` to normal entries only | |
| 10 | `grubby --update-kernel=ALL --args=…` | |
| 11 | It mounts the real root (drivers, RAID, LVM, LUKS); rebuild after changing `mdadm.conf`, `crypttab`, LVM/root device, or root-storage module options | |
| 12 | Rescue: local filesystems mounted, minimal services. Emergency: root only, read-only, nothing else | |
| 13 | `init=/bin/bash` on the kernel command line | |
| 14 | `touch /.autorelabel` (or `restorecon /etc/shadow`); otherwise `/etc/shadow`'s SELinux label is wrong and nobody can log in | |
| 15 | Tools inside the chroot need to see devices and kernel interfaces of the running system | |
| 16 | Choose the previous kernel under "Advanced options" in the GRUB menu | |
| 17 | `journalctl -k -p err` / `dmesg`, `smartctl`, `/proc/mdstat` | |
| 18 | Reboots automatically 10 s after a kernel panic instead of hanging | Unattended servers recover on their own |
