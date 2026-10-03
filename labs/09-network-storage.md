# Lab 09 — Network storage and automounting

24 tasks. **ubuntu2** is the server, **ubuntu1** the client, unless a task says otherwise.
Remember the exam workflow: start on ubuntu1, `ssh ubuntu2` for server tasks, `exit` back.
Do the quiz first.

**Setup (both nodes):** `sudo apt update && sudo apt install -y sysstat`

**Cleanup:** restore the `clean` snapshots of ubuntu1 and ubuntu2.

---

## 🟢 Warm-up

**9.1** List the filesystem types your kernel supports, marking which need no device.
*Verify:* `ext4`, `xfs` (if loaded), `tmpfs`, `proc` and `nfs` (after installing it) appear;
the `nodev` ones are pseudo-filesystems.

**9.2** Show every pseudo-filesystem currently mounted (`proc`, `sysfs`, `tmpfs`,
`devtmpfs`, `cgroup2`).
*Verify:* one `findmnt` command with `-t`.

**9.3** Read, without any tool other than `cat`: your system's swappiness, the I/O
scheduler of your main disk, and the MAC address of `eth0`/`enp0s3`.
*Verify:* three values from `/proc/sys`, `/sys/block` and `/sys/class/net`.

**9.4** Mount a 50 MiB `tmpfs` at `/mnt/scratch`, write a file, unmount and remount it. Where
did the file go?
*Verify:* `df -h /mnt/scratch` shows 50M; the file is gone after remount.

---

## 🔵 Practical

**9.5** Build an overlay: a lower directory containing `config.txt` with `v1`, an upper and a
work directory, merged at `/mnt/merged`. Change the file through the merged view, then
delete another lower file through it.
*Verify:* lower still says `v1`; upper contains the changed file and a *whiteout* entry
(a character device `0,0`) for the deleted one.

**9.6** On **ubuntu2**, export `/srv/share` read-write to the `192.168.56.0/24` network with
synchronous writes. Put a file `hello.txt` in it.
*Verify:* `sudo exportfs -v` shows the export; `showmount -e ubuntu2` from ubuntu1 lists it.

**9.7** On **ubuntu1**, mount it on `/mnt/share`, persistently, so that boot waits for the
network and doesn't fail if ubuntu2 is down.
*Verify:* `findmnt /mnt/share` shows `nfs4`; the fstab line has `_netdev` and `nofail`;
`mount -a` clean.

**9.8** As root on ubuntu1, create a file on the share. Who owns it on ubuntu2, and why?
*Verify:* `ls -l` on the server shows `nobody` (or `nobody:nogroup`).

**9.9** On ubuntu2, add a read-only export `/srv/public` for any host, mapping every client
user to `nobody`. Mount it on ubuntu1 and prove you cannot write to it, even as root.
*Verify:* `touch` fails with *Read-only file system* (or *Permission denied*).

**9.10** Create user `nfsuser` on **both** machines but with **different** UIDs (1501 on
ubuntu1, 1502 on ubuntu2). Have nfsuser create a file on the share from ubuntu1, and look at
it from both sides. Then fix the mismatch.
*Verify:* before the fix the server shows a different owner; after aligning UIDs both
sides show `nfsuser`.

**9.11** Mount the share by hand with the `soft` option and then with `hard`, and record the
effective options each time from the client's view.
*Verify:* `nfsstat -m` or `findmnt -o OPTIONS` shows `soft`/`hard` and the NFS version.

**9.12** On **ubuntu2**, export a 1 GiB file as a network block device named `disk1`.
*Verify:* `ss -tlnp | grep 10809` shows `nbd-server` listening.

**9.13** On **ubuntu1**, list the exports, attach `disk1` as `/dev/nbd0`, create an ext4
filesystem on it, mount it on `/mnt/nbd`, write a file, then cleanly detach.
*Verify:* `lsblk` shows `nbd0` at 1G while attached; after detaching, `lsblk /dev/nbd0`
reports size 0 or nothing.

**9.14** Make the `nbd` kernel module load at every boot.
*Verify:* the file you created in `/etc/modules-load.d/`; after a reboot `lsmod | grep nbd`.

**9.15** Configure **autofs** on ubuntu1 so that `/mnt/auto/share` mounts
`ubuntu2:/srv/share` on access and unmounts after 30 idle seconds. Remove the `/mnt/share`
fstab entry first so the two don't compete.
*Verify:* `findmnt /mnt/auto/share` is empty, then shows `nfs4` after `ls /mnt/auto/share`,
and is empty again a minute later.

**9.16** Add a **wildcard** autofs map: create `/srv/homes/alice` and `/srv/homes/bob` on
ubuntu2 (exported), so that on ubuntu1 `cd /nethome/<name>` mounts the matching directory.
*Verify:* `ls /nethome/alice` and `ls /nethome/bob` each trigger a separate mount.

**9.17** Replace autofs for `/mnt/share` with a **systemd automount** declared in fstab, idle
timeout 2 minutes.
*Verify:* `systemctl list-units --type=automount` shows `mnt-share.automount`; first access
mounts it.

**9.18** Watch storage performance live while generating load: run
`dd if=/dev/zero of=/var/tmp/load bs=1M count=2000 oflag=direct` and observe it with
`iostat -xz 1`. Identify the device, its write throughput, `w_await` and `%util`.
*Verify:* you can quote the four values from one line of output.

**9.19** Identify the process responsible for the I/O in 9.18 with two different tools.
*Verify:* both `pidstat -d` and `iotop -o` name `dd`.

**9.20** Measure 4 KiB random-read IOPS on `/var/tmp` with `fio`, bypassing the page cache.
*Verify:* the `read: IOPS=` line in fio's output.

---

## 🔴 Challenge

**9.21 — The dead server.** With `/mnt/share` mounted *hard* (no automount), stop
`nfs-kernel-server` on ubuntu2 and run `ls /mnt/share &` on ubuntu1. Show its process state.
Show that `kill -9` *does* end it, but that the next `ls` hangs again. Then free the client
for good. Restart the server afterwards.
*Verify:* `ps -o stat` shows `D`; after `kill -9` the process is gone; after your fix,
`findmnt /mnt/share` is empty and nothing hangs.

**9.22 — access denied.** On ubuntu2, change the `/srv/share` export so that only
`192.168.56.99` may mount it, `exportfs -ra`, and try to mount from ubuntu1. Read the error,
find the server-side log message, and restore access.
*Verify:* `mount` fails with *access denied by server*. Then mount with `-o vers=3` and
find the `refused mount request … unmatched host` line in ubuntu2's journal — and explain
why the plain NFSv4 attempt left no such line.

**9.23 — The world-writable typo.** Add this line to `/etc/exports` on ubuntu2 and
`exportfs -ra`:

```text
/srv/public 192.168.56.11 (rw)
```

Use `exportfs -v` to show what was actually exported, explain it, and fix it.
*Verify:* `exportfs -v` shows two entries for `/srv/public` before the fix — one of them
`rw` for `*`.

**9.24 — Read-only view.** Using only bind mounts, make `/srv/share` on ubuntu2 also
available at `/srv/share-ro` as a **read-only** view, while the original stays writable.
Make it persistent.
*Verify:* writing under `/srv/share-ro` fails, writing under `/srv/share` succeeds, and the
fstab entries survive `mount -a`.

---

## Self-check

- [ ] I can explain what VFS is and name the pseudo-filesystems
- [ ] I can use tmpfs, bind mounts and overlay mounts
- [ ] I can export NFS with the right squash and sync options, and mount it persistently
- [ ] I know why UIDs must match across NFS clients and servers
- [ ] I can attach a network block device and know why it can't be shared
- [ ] I can configure autofs indirect, wildcard and direct maps, and a systemd automount
- [ ] I can find the busy disk and the process behind it

➡ **Solutions:** [solutions/09-network-storage.md](../solutions/09-network-storage.md)
