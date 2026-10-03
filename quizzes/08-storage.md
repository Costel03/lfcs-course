# Quiz 08 — Disks, filesystems, LVM and swap

20 questions. Answers: [solutions/08-storage.md](../solutions/08-storage.md#quiz-answers)

---

**Q1.** Why should `/etc/fstab` refer to `UUID=…` rather than `/dev/sdb1`?

**Q2.** Name two advantages of GPT over MBR.

**Q3.** Which command shows every block device with its filesystem type, label and UUID in
a tree?

**Q4.** Which filesystem can be shrunk: ext4, XFS, both, or neither? Under what condition?

**Q5.** Write the `/etc/fstab` line that mounts the XFS filesystem with UUID `abcd-1234` on
`/srv/data` without access-time updates, and never fsck-checks it at boot.

**Q6.** You added an fstab line. Which two commands should you run before rebooting, and
what does each prove?

**Q7.** What happens at boot if an fstab entry refers to a device that doesn't exist, and
which option prevents it?

**Q8.** Which mount options would you use for `/tmp` to stop binaries, SUID programs and
device files working from it?

**Q9.** Put these in order to create a swap file: `mkswap`, `fallocate`, `swapon`, `chmod 600`,
add to fstab.

**Q10.** In LVM, what are a PV, a VG, an LV and a PE?

**Q11.** What is the difference between `lvcreate -L 50%FREE` and `lvcreate -l 50%FREE`?
Which one is valid?

**Q12.** You ran `lvextend -L +2G /dev/vg/lv` and `df -h` shows no change. Why, and how do you
fix it for ext4 and for XFS?

**Q13.** What does `-r` add to `lvreduce`, and why is it essential?

**Q14.** A disk in volume group `vgdata` is reporting errors. Which command moves its data to
the other disks while the filesystems stay mounted?

**Q15.** What happens to an LVM snapshot when more data changes than the snapshot's size?

**Q16.** When is it safe to run `e2fsck` on a filesystem?

**Q17.** ext4 won't mount: *bad superblock*. Which command lists where the backup
superblocks are **without** changing anything, and how do you use one?

**Q18.** After a disk error, writes fail with *Read-only file system*. What did the kernel do,
and where do you look for the cause?

**Q19.** In a RAID 1 array, one disk fails. Which three `mdadm` operations replace it?

**Q20.** You removed an LV and its volume group but left its line in `/etc/fstab`. What
happens at the next boot?
