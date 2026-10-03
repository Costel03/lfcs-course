# Quiz 14 — Virtual machines and containers

18 questions. Answers: [solutions/14-vms-containers.md](../solutions/14-vms-containers.md#quiz-answers)

---

**Q1.** What does each of KVM, QEMU and libvirt contribute?

**Q2.** How do you check whether a host can use hardware-accelerated virtualization?

**Q3.** What is the difference between `virsh destroy vm1` and `virsh undefine vm1`?

**Q4.** Which command makes a VM start when the host boots?

**Q5.** What does `virt-install --import` do, compared with `--cdrom`?

**Q6.** You run `virsh setmem vm1 1G --config`. When does the change take effect?

**Q7.** What is the advantage of creating a VM's disk with `qemu-img create -b base.img`?

**Q8.** On libvirt's `default` network, can other machines on your LAN reach the guests
directly? What would allow it?

**Q9.** Name the kernel features that make a process a container.

**Q10.** Why can't you run a Windows container on a Linux host, while you can run a Windows VM?

**Q11.** `docker run ubuntu` returns immediately and `docker ps` shows nothing. Is something
broken?

**Q12.** What happens to files a container wrote in its own filesystem when you `docker rm` it,
and how do you keep them?

**Q13.** What does `-p 8080:80` mean, and how do you make the published port reachable only
from the host itself?

**Q14.** Why is membership of the `docker` group considered equivalent to root?

**Q15.** You need `docker push ubuntu1:5000/app:1.0` to work from ubuntu2 to a plain-HTTP
registry. What must you configure, and where?

**Q16.** Which `docker run` option makes a container come back after a reboot, unless you
deliberately stopped it?

**Q17.** How does Podman run containers as systemd services today?

**Q18.** ufw denies port 8080, yet a container published with `-p 8080:80` is reachable from
other machines. Why?
