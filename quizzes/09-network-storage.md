# Quiz 09 — Network storage and automounting

16 questions. Answers: [solutions/09-network-storage.md](../solutions/09-network-storage.md#quiz-answers)

---

**Q1.** What does the kernel's VFS layer do, in one sentence?

**Q2.** Name three filesystems that have no disk behind them, and what each is for.

**Q3.** What is the difference between these two lines in `/etc/exports`?

```text
/srv/share 192.168.56.11(rw)
/srv/share 192.168.56.11 (rw)
```

**Q4.** By default, what happens when root on an NFS client creates a file on the export?

**Q5.** You edited `/etc/exports`. What command makes the change take effect without
restarting the server?

**Q6.** alice sees files on an NFS mount owned by `bob`, though she created them. Why?

**Q7.** What does `_netdev` do in an fstab line?

**Q8.** An NFS server died, and every process that touches the mount hangs. Which mount
option causes that behaviour, why is it still the default, and how do you release the
client?

**Q9.** Why should an SMB password never be written in `/etc/fstab`, and what do you use
instead?

**Q10.** What is the fundamental difference between NFS and NBD?

**Q11.** Two clients mount the same NBD export with ext4 at the same time. What happens?

**Q12.** In the autofs map line `* -rw server:/home/&`, what do `*` and `&` mean?

**Q13.** You run `ls /mnt/auto` and it's empty, though the map defines `share`. Is autofs
broken?

**Q14.** Which fstab options make systemd mount an NFS share on first access and unmount it
after 5 idle minutes?

**Q15.** In `iostat -x`, which column best reflects what users experience as "slow disk",
and why is `%util` unreliable on SSDs?

**Q16.** Why must a disk benchmark with `dd` use `oflag=direct`?
