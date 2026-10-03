# Quiz 13 — SELinux

16 questions. Answers: [solutions/13-selinux.md](../solutions/13-selinux.md#quiz-answers)

---

**Q1.** What is the difference between discretionary and mandatory access control?

**Q2.** In permissive mode, what happens to an action the policy forbids?

**Q3.** How do you switch to permissive until reboot, and how do you make enforcing the
persistent setting?

**Q4.** In `system_u:object_r:httpd_sys_content_t:s0`, which field does the targeted policy
mostly use, and what is it called for processes?

**Q5.** A file was created in your home directory and `mv`ed to `/var/www/html`. nginx returns
403 though the mode is 644. Why, and what's the quick fix?

**Q6.** Why is `chcon` the wrong way to label `/srv/web` permanently? What is the right way?

**Q7.** What does the regular expression in `semanage fcontext -a -t T '/srv/web(/.*)?'` match?

**Q8.** nginx won't start after you set `listen 8081;`. `journalctl` shows *Permission denied*
on bind. What's the fix?

**Q9.** `semanage port -a -t http_port_t -p tcp 8000` fails with *already defined*. What now?

**Q10.** nginx as a reverse proxy returns *502 Bad Gateway*; the backend works. Which boolean
is probably off?

**Q11.** What does `-P` do in `setsebool -P`?

**Q12.** Where are AVC denials logged, and which command searches them for the last ten
minutes?

**Q13.** Read this denial and name the fix:

```text
avc: denied { name_connect } for comm="nginx" dest=8000
scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:soundd_port_t:s0 tclass=tcp_socket
```

**Q14.** Why is `audit2allow -M` a last resort?

**Q15.** A service fails, but no AVC appears in the log. What might be hiding it, and how do
you reveal it?

**Q16.** What must you do when switching a system from SELinux *disabled* to enforcing?
