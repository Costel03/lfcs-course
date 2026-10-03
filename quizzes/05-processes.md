# Quiz 05 — Processes and performance

18 questions. Answers: [solutions/05-processes.md](../solutions/05-processes.md#quiz-answers)

---

**Q1.** What do the process states `R`, `S`, `D`, `Z` and `T` mean?

**Q2.** Why does `kill -9` have no effect on a process in state `D`, and what should you
investigate instead?

**Q3.** A machine has 4 CPUs and a load average of `8.00, 6.00, 2.00`. Is it getting
better or worse, and is it overloaded?

**Q4.** Load average is 12, but `top` shows the CPUs 90% idle. What is the most likely
explanation, and which field in `top` confirms it?

**Q5.** `free -h` shows 150 MB free and 1.4 GB available. Is the machine short of memory?

**Q6.** How do you get rid of a zombie process?

**Q7.** Which signals cannot be caught or ignored?

**Q8.** Why should you send `SIGTERM` before `SIGKILL`?

**Q9.** Which signal is conventionally used to tell a daemon to reload its configuration?

**Q10.** What range does niceness take, which end is *higher* priority, and who may
raise a process's priority?

**Q11.** A backup job saturates the disk and slows the database, while the CPU is idle.
Which tool changes its priority usefully — `nice` or `ionice`?

**Q12.** How do you see the environment variables of a running process with PID 4242?

**Q13.** You raised `nofile` in `/etc/security/limits.conf`, yet `cat /proc/<pid>/limits`
for your systemd service still shows the old value. Why, and where must the limit go?

**Q14.** What does `lsof +L1` show, and what problem does it help diagnose?

**Q15.** Which command shows the *system calls* a process is making?

**Q16.** A service died overnight. `top` is no help now. Which tool can show CPU and
memory usage for 03:00–04:00, and what must have been enabled beforehand?

**Q17.** A process exited with code 137. What happened to it?

**Q18.** `pkill ssh` — what might it kill besides the process you intended?
