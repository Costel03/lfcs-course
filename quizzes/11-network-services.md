# Quiz 11 — Network services and certificates

20 questions. Answers: [solutions/11-network-services.md](../solutions/11-network-services.md#quiz-answers)

---

**Q1.** Key login fails, and the server log says *Authentication refused: bad ownership or
modes*. What are the correct modes for `~/.ssh` and `authorized_keys`?

**Q2.** What does `ssh -L 8080:localhost:5432 db01` make possible?

**Q3.** How do you reach `internal01`, which only `bastion` can see, in one `ssh` command?

**Q4.** In `sshd_config` on Ubuntu, a setting in `/etc/ssh/sshd_config.d/50-x.conf`
conflicts with one later in `sshd_config`. Which wins, and why?

**Q5.** Which two commands show (a) whether `sshd_config` is valid, (b) the effective value
of `PasswordAuthentication`?

**Q6.** You changed `Port 2222` on Ubuntu 24.04 and restarted `ssh.service`, but sshd still
listens only on 22. What's missing?

**Q7.** What are the five netfilter hooks, and in which do DNAT and masquerade happen?

**Q8.** Why must `ct state established,related accept` appear in a default-drop input chain?

**Q9.** What does `table inet` mean in nftables?

**Q10.** With firewalld, you run `firewall-cmd --add-service=http`. After a reboot, http is
blocked again. Why?

**Q11.** What is the difference between masquerade and DNAT?

**Q12.** You redirect port 80 to 8080 in the `prerouting` hook. Remote clients work, but
`curl localhost` still reaches port 80. Why?

**Q13.** A DNAT rule sends ubuntu1:8080 to a backend on the same subnet as the client, and
connections hang. What's the usual missing piece?

**Q14.** What do the `X-Forwarded-For` and `X-Real-IP` headers fix?

**Q15.** With `location /api/`, how do `proxy_pass http://app;` and `proxy_pass http://app/;`
differ?

**Q16.** Name nginx's three load-balancing methods and when you'd pick `ip_hash`.

**Q17.** A certificate's CN is `app.lab.local` but browsers and curl reject it for that
name. What is it most likely missing?

**Q18.** How do you check that a private key and a certificate belong together?

**Q19.** Why use `-servername` with `openssl s_client`?

**Q20.** How do you make every tool on an Ubuntu machine trust your private CA?
