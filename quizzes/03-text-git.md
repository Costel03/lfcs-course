# Quiz 03 — Text processing and Git

18 questions. Answers: [solutions/03-text-git.md](../solutions/03-text-git.md#quiz-answers)

---

**Q1.** Which needs `-E` (or escaping) for `+` to mean "one or more": `grep`, `awk`, or both?

**Q2.** What does `grep '10.0.0.1' log` match that `grep -F '10.0.0.1' log` does not?

**Q3.** Write a `grep` that shows the active lines of `/etc/ssh/sshd_config` — no comments,
no blank lines.

**Q4.** What does `-n` do in `sed -n '5,10p'`, and what happens without it?

**Q5.** What is the difference between these?

```bash
sed 's/a/b/' file
sed 's/a/b/g' file
```

**Q6.** Write a `sed` command that changes `PasswordAuthentication yes` to
`PasswordAuthentication no` in place, keeping a backup.

**Q7.** In `awk -F: '$3 >= 1000 {print $1}' /etc/passwd`, what is `$3`, and what does the
command list?

**Q8.** Why does `uniq -c` usually follow `sort`?

**Q9.** What does this pipeline produce?

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -3
```

**Q10.** Why does `sudo echo 'nameserver 1.1.1.1' > /etc/resolv.conf` fail for a normal
user, and what is the fix?

**Q11.** A script fails with `/bin/bash^M: bad interpreter`. What is wrong with the file
and how do you fix it?

**Q12.** The regex `".*"` applied to `say "hi" and "bye"` matches what? How do you match
only `"hi"`?

**Q13.** Put these in order for a new change: `git commit`, `git push`, `git add`,
editing the file.

**Q14.** What is the difference between `git diff` and `git diff --staged`?

**Q15.** A commit has already been pushed and others have pulled it. Which is the safe
way to undo it: `git reset --hard HEAD~1` or `git revert <commit>`? Why?

**Q16.** What is a bare repository, and why should the remote you push to be one?

**Q17.** During a merge you see `<<<<<<<`, `=======` and `>>>>>>>` in a file. What are the
three steps to finish the merge?

**Q18.** You committed `db-password.txt`, pushed it, then deleted it in the next commit.
Is the password safe now?
