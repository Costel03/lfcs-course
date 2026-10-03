# Quiz 10 — Networking

20 questions. Answers: [solutions/10-networking.md](../solutions/10-networking.md#quiz-answers)

---

**Q1.** You added an address with `ip addr add`. What happens to it at reboot?

**Q2.** `ip link` shows `<BROADCAST,MULTICAST,UP>` but not `LOWER_UP`. What does that mean?

**Q3.** Which command applies a Netplan change but rolls it back automatically if you lose
access?

**Q4.** What is wrong with indenting a Netplan file using tabs?

**Q5.** With `hosts: files dns` in `/etc/nsswitch.conf`, a name is in both `/etc/hosts`
and DNS with different addresses. Which wins, and which command shows you what
applications will get?

**Q6.** Why might `dig server01` return nothing while `ping server01` works?

**Q7.** On Ubuntu, `/etc/resolv.conf` contains `nameserver 127.0.0.53`. What is that, and
where do you see the real upstream DNS servers?

**Q8.** Two routes match a destination: `10.0.0.0/8` and `10.20.0.0/16`. Which is used?

**Q9.** Which command tells you which route, interface and source address a packet to
`8.8.8.8` would use?

**Q10.** What setting turns a Linux machine into a router, and how do you make it survive
a reboot?

**Q11.** What is `fe80::a00:27ff:fe3a:1b9c`, and why does `ping -6` to it need `%eth1`?

**Q12.** Which bonding mode provides failover without any configuration on the switch?

**Q13.** Where can you see which member of a bond is currently active?

**Q14.** You add `eth2` to bridge `br0`. Where should the IP address now be configured?

**Q15.** Why does wrong system time break TLS and LDAP logins?

**Q16.** In `chronyc sources`, what do `^*` and `^?` mean?

**Q17.** A service works with `curl localhost:8080` but not from another machine. `ss -tlnp`
shows `127.0.0.1:8080`. What's wrong?

**Q18.** From a client, `nc -zv host 9000` returns *Connection refused* in one case and hangs
until timeout in another. What does each tell you?

**Q19.** Put these troubleshooting steps in a sensible bottom-up order: name resolution,
link state, port reachability, routing, address.

**Q20.** Why use `tcpdump -n`?
