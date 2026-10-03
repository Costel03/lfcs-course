# Quiz 12 — Kernel, boot and recovery

18 questions. Answers: [solutions/12-kernel-boot.md](../solutions/12-kernel-boot.md#quiz-answers)

---

**Q1.** What is the difference between `sysctl -w vm.swappiness=10` and a line in
`/etc/sysctl.d/90-swap.conf`?

**Q2.** Which command loads all sysctl configuration files right now?

**Q3.** Two files set `net.core.somaxconn`: `10-a.conf` and `90-b.conf`. Which value applies?

**Q4.** What does `sysctl net.ipv4.ip_forward` correspond to under `/proc`?

**Q5.** Why use `modprobe` rather than `insmod`?

**Q6.** You need the `nbd` module loaded at boot with `nbds_max=4`. Which two files do you
create?

**Q7.** `blacklist usb-storage` is in `/etc/modprobe.d/`, yet `modprobe usb-storage` still
loads it. Why, and what blocks it completely?

**Q8.** Where do you see the kernel command line the system booted with?

**Q9.** On Ubuntu, you edited `GRUB_CMDLINE_LINUX` in `/etc/default/grub`. What must you run,
and what is the difference from `GRUB_CMDLINE_LINUX_DEFAULT`?

**Q10.** Which RedHat tool adds a kernel argument to every installed kernel's boot entry?

**Q11.** What is the initramfs for, and name two changes after which you must rebuild it.

**Q12.** What is the difference between `rescue.target` and `emergency.target`?

**Q13.** An Ubuntu VM has no root password. How do you get a root shell at boot?

**Q14.** On Rocky, after resetting the root password through `rd.break`, what one extra step
is essential, and what happens if you skip it?

**Q15.** Why must `/dev`, `/proc`, `/sys` and `/run` be bind-mounted before chrooting into
a system to run `grub-install`?

**Q16.** A kernel update leaves the machine unbootable. What's the fastest way to boot again?

**Q17.** Which three places would you check first for signs of failing hardware?

**Q18.** What does `kernel.panic = 10` do, and why set it on a server?
