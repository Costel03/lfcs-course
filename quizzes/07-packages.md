# Quiz 07 — Software and packages

16 questions. Answers: [solutions/07-packages.md](../solutions/07-packages.md#quiz-answers)

---

**Q1.** What is the difference between `apt update` and `apt upgrade`?

**Q2.** `apt remove nginx` versus `apt purge nginx` — what stays behind after the first?

**Q3.** `dpkg -l` shows `rc` in front of a package. What does that mean?

**Q4.** Which command tells you which installed package owns `/usr/bin/curl` — on Debian,
and on RedHat?

**Q5.** You need the `dig` command but it isn't installed. How do you find which package
provides it, on each family?

**Q6.** How do you stop `apt upgrade` from ever changing the version of `postgresql-16`?

**Q7.** Why is `apt-key add` deprecated, and what replaces it?

**Q8.** In a sources line, what does `[signed-by=/etc/apt/keyrings/x.gpg]` achieve?

**Q9.** What does `apt policy nginx` show that `apt show nginx` doesn't?

**Q10.** `dpkg -V nginx-core` prints `??5??????   c /etc/nginx/nginx.conf`. Should you be
worried? What would worry you?

**Q11.** Why should scripts use `apt-get` rather than `apt`?

**Q12.** A pin priority of `-1` means what?

**Q13.** You built a tool from source with `--prefix=/usr/local`. Why that prefix, and what
do you lose compared with a package?

**Q14.** After a source install, running the tool gives *error while loading shared
libraries: libfoo.so.1*. The file exists in `/usr/local/lib`. What's missing?

**Q15.** `apt` says *Could not get lock /var/lib/dpkg/lock-frontend*. What is the right
response, and the wrong one?

**Q16.** On a Rocky machine, how do you undo the last package transaction?
