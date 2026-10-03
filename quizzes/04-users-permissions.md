# Quiz 04 — Users, groups and permissions

24 questions. Q1–Q15 cover permissions, Q16–Q24 users and groups. Answers: [solutions/04-users-permissions.md](../solutions/04-users-permissions.md#quiz-answers)

---

**Q1.** Convert to octal: `rwxr-x---`

**Q2.** Convert to symbolic: `640`

**Q3.** A file is `----r--r--` and owned by alice. Can alice read it? Why?

**Q4.** You have `x` but not `r` on a directory. Which of these work?

- A) `ls dir`
- B) `cat dir/known-file.txt` (file is readable)
- C) `ls dir/known-file.txt`
- D) Tab-completing filenames in `dir`

**Q5.** The umask is `027`. What modes do a new file and a new directory get?

**Q6.** Can any umask value make a newly created file executable? Why or why not?

**Q7.** What does `chmod -R a+rX dir/` do differently from `chmod -R a+rx dir/`?

**Q8.** Match each bit to its effect on a **directory**:

| Bit | Effect |
|---|---|
| SUID | ? |
| SGID | ? |
| Sticky | ? |

**Q9.** `ls -l` shows `-rwSr--r--`. What does the capital `S` tell you?

**Q10.** You `chmod u+s` a shell script owned by root. A normal user runs it. Who does
it run as on Linux, and why?

**Q11.** bob is not in group `devs`. A file is `rw-r-----` owned by `alice:devs`. You
run `setfacl -m u:bob:rw file`, and later someone runs `chmod 640 file`. What can bob
do now, and why?

**Q12.** What does the `+` in `-rw-r-----+` mean?

**Q13.** root runs `rm /etc/important.conf` and gets *Operation not permitted*. The file
is `rw-r--r-- root root`. What is the most likely cause, and what command confirms it?

**Q14.** A script is `rwxr-xr-x`, you own it, every directory above it is fine, yet
`./script.sh` says *Permission denied*. Name two causes worth checking that are not the
file's own mode.

**Q15.** You need a directory where the whole `devs` team can create and edit each
other's files, and files automatically belong to `devs`. Which two settings achieve it?

---

**Q16.** Which file holds password hashes, and who can read it?

**Q17.** alice is in groups `sudo` and `devs`. You run `usermod -G docker alice`. Which
supplementary groups does she have now?

**Q18.** What does `useradd -r` change compared with a normal `useradd`?

**Q19.** After `passwd -l alice`, can alice still log in with an SSH key? How do you disable
the account completely?

**Q20.** You create `/etc/sudoers.d/web.conf` with a valid rule, but it has no effect. Why?

**Q21.** Why is `alice ALL=(root) /usr/bin/vim /etc/hosts` dangerous, and what should you
grant instead?

**Q22.** Which file sets a variable for every user's session through PAM, and what syntax
restriction does it have?

**Q23.** You add `nginx hard nofile 65535` to `/etc/security/limits.conf`, but the running
nginx service still reports a limit of 1024 after a restart. Why, and what is the fix?

**Q24.** `ldapsearch` finds `ldapuser1`, but `getent passwd ldapuser1` returns nothing.
Name two things to check on the client.

