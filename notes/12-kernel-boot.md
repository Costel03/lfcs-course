# Chapter 12 — Kernel, boot and recovery

This chapter is about the layer under everything else: tuning the running kernel, loading
its modules, controlling how it boots — and getting a machine back when it doesn't. The
recovery procedures are the ones you hope never to need and must be able to do from memory
when you do, usually at a console, under pressure.

LFCS: *Operations Deployment* — configure kernel parameters, persistent and non-persistent;
recover from hardware, operating system or filesystem failures.

## Objectives

After this chapter you can:

- Read and change kernel parameters with `sysctl`, temporarily and persistently.
- Load, unload, configure and blacklist kernel modules.
- Change the kernel command line for one boot and permanently, on Ubuntu and RedHat.
- Explain each boot stage and rebuild the initramfs when it matters.
- Boot into rescue and emergency targets, and reset a lost root password.
- Repair an unbootable system from a chroot.
- Choose the right diagnosis path for hardware, OS and filesystem failures.

---

## 1. Kernel parameters — sysctl

Thousands of kernel tunables live under `/proc/sys`; `sysctl` reads and writes them using
dotted names that mirror the paths.

```bash
sysctl -a | wc -l
sysctl vm.swappiness                     # same as: cat /proc/sys/vm/swappiness
sysctl net.ipv4.ip_forward
```

### Non-persistent — until reboot

```bash
sudo sysctl -w vm.swappiness=10
echo 10 | sudo tee /proc/sys/vm/swappiness      # equivalent
```

### Persistent

```bash
echo 'vm.swappiness = 10' | sudo tee /etc/sysctl.d/60-swappiness.conf
sudo sysctl --system                      # load every sysctl file, show what was applied
sudo sysctl -p /etc/sysctl.d/60-swappiness.conf   # load just one file
```

At boot, `systemd-sysctl` collects `*.conf` files from `/etc/sysctl.d/`, `/run/sysctl.d/`
and `/usr/lib/sysctl.d/`. A file in `/etc` replaces a same-named file in the other two;
then **all** files are applied in lexical order of their names, whichever directory they
came from — so for a given key, the alphabetically last file wins. That's why your
overrides get high numbers like `90-`. On Debian, `/etc/sysctl.conf` is read through the
link `/etc/sysctl.d/99-sysctl.conf`. Files not ending in `.conf` are ignored.

### Parameters worth knowing

| Parameter | Effect |
|---|---|
| `vm.swappiness` | 0–200: how readily the kernel swaps; lower favours keeping apps in RAM |
| `vm.dirty_ratio`, `vm.dirty_background_ratio` | how much dirty page cache before writeback |
| `vm.overcommit_memory` | memory overcommit policy (0 heuristic, 1 always, 2 strict) |
| `net.ipv4.ip_forward` | route packets (chapter 10) |
| `net.ipv4.icmp_echo_ignore_all` | ignore pings |
| `net.ipv4.conf.all.rp_filter` | reverse-path filtering — drop spoofed source addresses |
| `net.core.somaxconn` | maximum listen backlog for busy servers |
| `fs.file-max` | system-wide open file limit |
| `kernel.pid_max` | maximum PID |
| `kernel.panic` | seconds before rebooting after a panic (0 = hang) |
| `kernel.sysrq` | which Magic SysRq functions are allowed |

`man 5 sysctl.d`, and the kernel's own documentation under
`/usr/share/doc/linux-doc` (if installed), describe them.

---

## 2. Kernel modules

Drivers and many features are **modules** loaded on demand.

```bash
lsmod                          # loaded modules, size, users
modinfo nbd                    # file, description, parameters it accepts
sudo modprobe nbd              # load, with its dependencies
sudo modprobe nbd nbds_max=4   # with a parameter
sudo modprobe -r nbd           # unload (fails if in use)
cat /sys/module/nbd/parameters/nbds_max     # a loaded module's current parameters
```

Use `modprobe`, not `insmod`/`rmmod`: `modprobe` resolves dependencies.

| Need | File |
|---|---|
| Load a module at every boot | `/etc/modules-load.d/<name>.conf` — one module name per line |
| Set a module's options | `/etc/modprobe.d/<name>.conf` — `options nbd nbds_max=4` |
| Stop a module auto-loading | `/etc/modprobe.d/<name>.conf` — `blacklist floppy` |
| Make it impossible to load at all | `install floppy /bin/false` |

`blacklist` only stops *automatic* loading by alias; an explicit `modprobe` or a dependency
still loads it. `install … /bin/false` blocks every load — the form security benchmarks
require for unused filesystems like `cramfs` or protocols like `dccp`.

If a module is needed to mount the root filesystem, it's loaded from the **initramfs** —
so after changing its options or blacklisting it, rebuild the initramfs (section 4).

---

## 3. The kernel command line

Parameters passed by the bootloader configure the kernel and systemd at startup.

```bash
cat /proc/cmdline
# BOOT_IMAGE=/vmlinuz-6.8.0-45-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro quiet splash
```

| Parameter | Effect |
|---|---|
| `ro` / `rw` | mount root read-only (normal — fsck then remount rw) or read-write |
| `quiet`, `splash` | hide boot messages — remove them when debugging |
| `systemd.unit=rescue.target` | boot into rescue mode |
| `systemd.unit=emergency.target` | boot into emergency mode |
| `init=/bin/bash` | skip systemd entirely; a root shell — last resort |
| `rd.break` | RedHat: stop inside the initramfs, before switching to the real root |
| `fsck.mode=force` | check filesystems on this boot |
| `single` / `1` | legacy synonyms for rescue |
| `nomodeset` | disable kernel mode-setting (display problems) |
| `selinux=0` / `enforcing=0` | disable SELinux / boot permissive (RedHat) |

### For one boot only — at the GRUB menu

1. Reboot and get to the GRUB menu. Ubuntu hides it on single-OS machines: hold **Shift**
   (BIOS) or press **Esc** (UEFI) during boot.
2. Highlight the entry and press **`e`**.
3. Find the line starting `linux` and add your parameter at the end.
4. Press **Ctrl-X** (or F10) to boot. Nothing is saved.

Make the menu always visible (in `/etc/default/grub`):

```text
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=5
```

### Permanently

Ubuntu/Debian:

```bash
sudoedit /etc/default/grub
#   GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"   ← normal boots only
#   GRUB_CMDLINE_LINUX="audit=1"                ← every boot, including recovery entries
sudo update-grub                              # regenerates /boot/grub/grub.cfg
```

RedHat — `grubby` edits every installed kernel's entry:

```bash
sudo grubby --update-kernel=ALL --args="audit=1"
sudo grubby --update-kernel=ALL --remove-args="quiet"
sudo grubby --info=DEFAULT
```

(or edit `/etc/default/grub` and run `grub2-mkconfig -o /boot/grub2/grub.cfg`).

**Never edit `grub.cfg` directly** — the next kernel update regenerates it.

### Choosing a kernel

```bash
dpkg -l 'linux-image-*' | grep ^ii          # installed kernels (rpm -q kernel on RedHat)
grep -E "menuentry '" /boot/grub/grub.cfg | cut -d"'" -f2
sudo grub-set-default 'Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-40-generic'
#  needs GRUB_DEFAULT=saved in /etc/default/grub, then update-grub
sudo grubby --set-default /boot/vmlinuz-5.14.0-427.el9.x86_64     # RedHat
```

After a kernel update that won't boot, choose the previous kernel under **Advanced
options** in the GRUB menu — package managers keep at least one older kernel for this.

---

## 4. The boot sequence, in depth

| Stage | What happens | Breaks look like | Fix with |
|---|---|---|---|
| **Firmware** (UEFI/BIOS) | POST, finds a boot device | "no bootable device" | firmware boot order; disk failure |
| **GRUB** | reads `grub.cfg`, loads kernel + initramfs from `/boot` | `grub>` or `grub rescue>` prompt | rescue media → chroot → `grub-install`, `update-grub` |
| **Kernel** | initialises hardware, unpacks initramfs | kernel panic | previous kernel from GRUB menu |
| **initramfs** | loads drivers, assembles RAID/LVM, unlocks LUKS, mounts real root | dracut/initramfs emergency shell; "cannot find root device" | rebuild initramfs; fix `root=` / UUIDs |
| **systemd** | mounts `/etc/fstab`, starts units to the default target | emergency or rescue mode | fix the failed unit/mount (`journalctl -xb`) |
| **Login** | getty, sshd | no prompt / SSH refused | the service, from console |

### The initramfs

A small archive containing just enough to find and mount the real root filesystem.
Rebuild it after changing anything it needs: storage drivers, `mdadm.conf`, LVM, LUKS
(`/etc/crypttab`), module options for those drivers, or the root device.

```bash
sudo update-initramfs -u                 # Debian/Ubuntu — current kernel
sudo update-initramfs -u -k all          # every installed kernel
sudo dracut -f                           # RedHat — current kernel
lsinitramfs /boot/initrd.img-$(uname -r) | grep -i mdadm    # what's inside (lsinitrd on RedHat)
```

### Targets for recovery

| Target | You get | Needs root password? |
|---|---|---|
| `rescue.target` | single-user root shell, local filesystems mounted, minimal services | yes |
| `emergency.target` | root shell, root mounted **read-only**, nothing else | yes |
| `init=/bin/bash` | a bare shell as PID 1, root read-only, no systemd at all | **no** |

From a running system: `systemctl rescue` / `systemctl isolate rescue.target` (drops all
network sessions). Leave with `systemctl default`.

Ubuntu Vagrant boxes and cloud images have **no root password**: rescue and emergency
targets refuse with *the root account is locked*. Then `init=/bin/bash` is the way in.

---

## 5. Resetting a lost root password

### Ubuntu / Debian — `init=/bin/bash`

1. GRUB menu → `e` on the entry.
2. On the `linux` line, replace `ro quiet splash` with `rw init=/bin/bash`.
3. Ctrl-X. You get a root shell, no password.
4. ```bash
   mount -o remount,rw /         # if it's still read-only
   passwd root                   # or: passwd someuser
   sync
   exec /sbin/init               # continue booting normally
   ```
   (or `reboot -f` — with no systemd running, plain `reboot` doesn't work).

### RedHat — `rd.break`

1. GRUB → `e` → append `rd.break` to the `linux` line → Ctrl-X.
2. ```bash
   mount -o remount,rw /sysroot
   chroot /sysroot
   passwd root
   touch /.autorelabel           # SELinux: relabel the changed /etc/shadow at next boot
   exit; exit
   ```

**Forget `/.autorelabel` and nobody can log in**: `/etc/shadow` gets the wrong SELinux
label when `passwd` rewrites it outside the running system. The relabel takes a few
minutes on the next boot. (Restoring just that file's label works too:
`load_policy -i; restorecon -v /etc/shadow` before exiting.)

### Why this is possible — and how to stop it

Anyone at the console can do this. Physical or console access is root access unless you
set a **GRUB password** (`grub-mkpasswd-pbkdf2`, then `set superusers` / `password_pbkdf2`
in `/etc/grub.d/40_custom`) and encrypt the disk. On cloud VMs, the console is protected by
the provider's account security.

---

## 6. Repairing from a chroot

When the installed system can't boot at all — GRUB wiped, `/boot` broken, a package
half-removed — boot **rescue media** (the installer ISO's "rescue" or a live USB), mount
the broken system, and *become* it with `chroot`:

```bash
lsblk -f                                   # find the root (and /boot, /boot/efi) partitions
sudo vgchange -ay                          # if root is on LVM
sudo mount /dev/mapper/ubuntu--vg-ubuntu--lv /mnt
sudo mount /dev/sda2 /mnt/boot             # if /boot is separate
sudo mount /dev/sda1 /mnt/boot/efi         # UEFI systems
for d in dev proc sys run; do
  sudo mount --rbind /$d /mnt/$d
  sudo mount --make-rslave /mnt/$d       # see below
done
sudo chroot /mnt /bin/bash
```

Inside, the broken system's own tools work against its own files:

```bash
grub-install /dev/sda          # reinstall the bootloader (BIOS); on UEFI: grub-install with no device
update-grub
update-initramfs -u -k all
apt install --reinstall linux-image-$(ls /lib/modules | tail -1)
passwd root
exit
```

```bash
sudo umount -R /mnt
sudo reboot
```

The bind mounts of `/dev`, `/proc`, `/sys` and `/run` are what make tools like
`grub-install` and `update-initramfs` work inside the chroot — without them they can't see
the hardware.

`--make-rslave` matters on any systemd machine. There, mounts use *shared* propagation, so
an `--rbind` copy stays linked to the original: a later `umount -R /mnt` would propagate
back and unmount the running system's own `/dev/pts`, `/dev/shm` and so on. Making the
copies *slaves* lets changes flow in but not back out. (`arch-chroot` and similar helpers
do exactly this.)

---

## 7. Diagnosing failures

### Hardware

```bash
journalctl -k -p err -b          # kernel errors this boot
dmesg -T | grep -iE 'error|fail|i/o|ata|nvme|mce|edac'
sudo smartctl -H -a /dev/sda     # disk health
cat /proc/mdstat                 # RAID degraded? (chapter 08)
sudo edac-util -v                # memory errors on servers with ECC (package edac-utils)
sensors                          # temperatures (lm-sensors)
```

Read-only filesystems, I/O errors in `dmesg`, SMART *reallocated* or *pending* sectors, and
RAID `[U_]` all point at storage hardware: back up first, then replace.

### Operating system

```bash
systemctl --failed
journalctl -b -p err
journalctl -b -1 -p err          # the previous boot, if it crashed (persistent journal)
systemd-analyze critical-chain   # what the boot waited on
last -x | head                   # reboots, shutdowns, runlevel changes
```

A kernel panic leaves no journal on disk — the system stopped before writing. Set
`kernel.panic = 10` so the machine at least reboots on its own, and look at `kdump`
(`kdump-tools` / `kexec-tools`) to capture crash dumps.

### Filesystems

Chapter 08: `e2fsck`/`xfs_repair` on unmounted filesystems, backup superblocks, and
`fsck.mode=force` on the kernel command line for the root filesystem.

### Magic SysRq — when nothing else responds

If the console is frozen but the kernel isn't, the SysRq key combination talks to the
kernel directly. The safe-reboot sequence is **Alt+SysRq+R, E, I, S, U, B**: keyboard raw,
terminate, kill, **sync**, remount read-only, reboot. From a shell:
`echo s | sudo tee /proc/sysrq-trigger` syncs disks; `kernel.sysrq` controls what's allowed.

---

## 8. Gotchas

**`sysctl -w` without a file** — gone at reboot.

**A sysctl file not ending in `.conf`** — silently ignored.

**Forgetting `update-grub` / `grub2-mkconfig`** after editing `/etc/default/grub`.

**Editing `grub.cfg` by hand** — overwritten on the next kernel update.

**Not rebuilding the initramfs** after changing RAID, LVM, LUKS or root-device driver
settings — the next boot can't find root.

**`blacklist` doesn't block explicit loads** — use `install <module> /bin/false`.

**Rescue mode with no root password** — Ubuntu boxes need `init=/bin/bash`.

**`rd.break` without `touch /.autorelabel`** on RedHat — nobody can log in.

**`reboot` inside `init=/bin/bash`** — no systemd to ask; use `exec /sbin/init` or `reboot -f`
after `sync`.

---

## 9. Commands introduced

| Command | Purpose |
|---|---|
| `sysctl`, `sysctl --system`, `/etc/sysctl.d/` | kernel parameters |
| `lsmod`, `modinfo`, `modprobe`, `modprobe -r` | modules |
| `/etc/modules-load.d/`, `/etc/modprobe.d/` | module loading, options, blacklisting |
| `/proc/cmdline` | the running kernel's command line |
| `update-grub`, `grub2-mkconfig`, `grubby`, `grub-set-default` | bootloader configuration |
| `update-initramfs`, `dracut`, `lsinitramfs`, `lsinitrd` | initramfs |
| `systemctl rescue`, `systemd.unit=`, `init=/bin/bash`, `rd.break` | recovery boots |
| `chroot`, `mount --rbind` | repair from rescue media |
| `journalctl -k`, `dmesg`, `smartctl`, `systemd-analyze critical-chain`, `last -x` | diagnosis |
| `man sysctl.d`, `man modprobe.d`, `man bootparam`, `man kernel-command-line`, `man dracut.cmdline` | offline references |

---

➡ **Next:** [quiz](../quizzes/12-kernel-boot.md) → [lab](../labs/12-kernel-boot.md)
