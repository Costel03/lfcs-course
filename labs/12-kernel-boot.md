# Lab 12 — Kernel, boot and recovery

22 tasks on **ubuntu1**, plus 12.21 on **rocky1**. Several tasks need the **VirtualBox
console** — open the VM's window from the VirtualBox Manager ("Show"). Take a snapshot
first: `vagrant snapshot save pre-boot`. Do the quiz first.

---

## 🟢 Warm-up

**12.1** Show the current values of `vm.swappiness`, `net.ipv4.ip_forward` and `fs.file-max`
two ways: with `sysctl` and from `/proc/sys`.
*Verify:* the two methods agree.

**12.2** List the files from which sysctl settings are loaded at boot, and find which file
(if any) sets `vm.swappiness`.
*Verify:* `grep -r swappiness /etc/sysctl.d /usr/lib/sysctl.d /etc/sysctl.conf 2>/dev/null`.

**12.3** Show the kernel command line of the running system, and the kernel version.
*Verify:* `/proc/cmdline` and `uname -r` agree on the version.

**12.4** List loaded modules, then show the description, file and parameters of the `loop`
(or `nbd`) module.
*Verify:* `modinfo` lists `parm:` lines.

**12.5** List the installed kernels and the GRUB menu entries.
*Verify:* each installed kernel has an entry.

---

## 🔵 Practical

**12.6** Set `vm.swappiness` to `15` **without** persistence. Reboot. What is it now?
*Verify:* 15 before the reboot; the original value after.

**12.7** Make `vm.swappiness = 15` and `net.ipv4.icmp_echo_ignore_all = 1` persistent in one
file, and load it without rebooting.
*Verify:* `sysctl vm.swappiness net.ipv4.icmp_echo_ignore_all`; ubuntu2 can no longer ping
ubuntu1; survives a reboot. (Then set icmp back to 0.)

**12.8** Create a second file `05-swap.conf` setting `vm.swappiness = 80`. After
`sysctl --system`, which value applies, and why?
*Verify:* 15 — and you can explain the ordering.

**12.9** Load `nbd` with `nbds_max=2`, check the parameter took effect, then unload it.
*Verify:* `ls /dev/nbd*` shows exactly two devices; afterwards `lsmod | grep nbd` is empty.

**12.10** Make `nbd` load at every boot with `nbds_max=4`.
*Verify:* after a reboot, `cat /sys/module/nbd/parameters/nbds_max` prints `4`.

**12.11** Block the `cramfs` filesystem module completely — no auto-loading, and an explicit
`modprobe cramfs` must fail too.
*Verify:* `sudo modprobe cramfs` returns an error; `modprobe -n -v cramfs` shows
`install /bin/false`.

**12.12** Make the GRUB menu visible for 5 seconds at every boot, and permanently add the
kernel parameter `audit=1` to normal **and** recovery entries.
*Verify:* after a reboot the menu appears; `/proc/cmdline` contains `audit=1`.

**12.13** For one boot only, remove `quiet splash` from the kernel command line so you can
watch the boot messages.
*Verify:* the messages scroll on the console; after the next normal reboot `/proc/cmdline`
has `quiet` again (or never had it on your box — check first).

**12.14** Rebuild the initramfs for the running kernel, and list three modules or files
inside it.
*Verify:* the timestamp of `/boot/initrd.img-$(uname -r)` changed; `lsinitramfs` output.

**12.15** From a running system, switch to `rescue.target` (on the console — SSH will drop),
then return to normal.
*Verify:* the console shows the rescue prompt (or the "root account is locked" message —
explain it); `systemctl default` returns you; `systemctl get-default` unchanged.

**12.16** Find how long the last boot took, which unit the boot waited on longest, and
whether the previous boot ended cleanly.
*Verify:* `systemd-analyze`, `systemd-analyze critical-chain`, and `last -x | head`.

---

## 🔴 Challenge

**12.17 — Lost password.** Set a password for `vagrant`, then "forget" it: on the console,
reset it at boot **without** knowing the old one or any root password, and boot normally.
*Verify:* you log in on the console with the new password; `sudo passwd -S vagrant` shows
today's change date.

**12.18 — Boot into emergency mode by hand.** At the GRUB menu, add
`systemd.unit=emergency.target`. What happens on this box, and why? Then make it work by
setting a root password first, and repeat. In emergency mode, find whether `/` is
read-only and make it writable.
*Verify:* you saw *root account is locked* the first time; the second time you got a shell and
`findmnt -no OPTIONS /` showed `ro`, then `rw`.

**12.19 — Practise the chroot.** On the running ubuntu1, rehearse the rescue-media procedure
safely: bind-mount `/` at `/mnt/sysroot`, add the `dev`, `proc`, `sys` and `run` bind mounts,
`chroot` in, run `update-grub` and `update-initramfs -u` inside, exit, and unmount everything
cleanly.
*Verify:* both commands succeed inside the chroot; at the end `findmnt -R /mnt/sysroot` prints
nothing.

**12.20 — Corrupted GRUB configuration.** Rename `/boot/grub/grub.cfg` to `grub.cfg.bak` and
reboot. Boot the system **by hand** from the `grub>` prompt, then fix it permanently.
*Hints:* `ls`, `set root=(hd0,gpt2)` (find yours with `ls`), `linux /vmlinuz root=… ro`,
`initrd /initrd.img`, `boot`. On LVM root, find the device from `/etc/fstab` beforehand.
*Verify:* the system boots from the prompt; after `update-grub` it boots normally again.

**12.21 — RedHat root password.** On **rocky1**, reset the root password with `rd.break`,
handling SELinux correctly.
*Verify:* you log in as root on the console with the new password; the next boot showed
the relabel (or you restored the label by hand and can say how).

**12.22 — Hardware triage report.** Write a one-screen script that prints: kernel errors this
boot, failed units, SMART health of each disk (or "unsupported"), RAID status, filesystems
mounted read-only that should be read-write, and the last three reboots.
*Verify:* each section is produced by one command; it runs without errors on ubuntu1.

---

## Self-check

- [ ] I can change a kernel parameter for now, for good, and explain which file wins
- [ ] I can load, configure, blacklist and fully block a module
- [ ] I can change the kernel command line once at GRUB, and permanently on both families
- [ ] I know when to rebuild the initramfs
- [ ] I can reset a root password on Ubuntu and on Rocky
- [ ] I can repair a system from a chroot, and boot by hand from a `grub>` prompt

➡ **Solutions:** [solutions/12-kernel-boot.md](../solutions/12-kernel-boot.md)
