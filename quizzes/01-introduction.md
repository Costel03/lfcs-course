# Quiz 01 — Introduction

20 questions. Write your answers down before checking. Score under 16? Reread the notes
before starting the lab.

Answers: [solutions/01-introduction.md](../solutions/01-introduction.md#quiz-answers)

---

**Q1.** Which expansion happens first?

- A) Parameter expansion (`$VAR`)
- B) Pathname expansion (`*.txt`)
- C) Brace expansion (`{a,b}`)
- D) Command substitution (`$(cmd)`)

**Q2.** What does this print?

```bash
n=3; echo {1..$n}
```

**Q3.** `f="a b.txt"`. How many arguments does `rm $f` pass to `rm`?

- A) 1  B) 2  C) 0  D) it depends on the current directory

**Q4.** Which of these sends *both* stdout and stderr to `out.log`?

- A) `cmd 2>&1 > out.log`
- B) `cmd > out.log 2>&1`
- C) `cmd 2> out.log 1>&2 out.log`
- D) `cmd | out.log 2>&1`

**Q5.** What does this print, and why?

```bash
count=0
printf 'a\nb\nc\n' | while read -r l; do count=$((count+1)); done
echo "$count"
```

**Q6.** After `false | true`, what is `$?` by default? What single setting changes it?

**Q7.** A service exits with status **137** and its own log shows no error. What is the
most likely cause, and where do you look first?

**Q8.** What is `<(sort file)` replaced with before the command runs?

- A) the sorted contents of the file
- B) a path such as `/dev/fd/63`
- C) a temporary file in `/tmp`
- D) nothing — it is a syntax error outside scripts

**Q9.** Why is `find … -print0 | xargs -0` safer than `find … | xargs`?

**Q10.** True or false: `set -e` causes a script to exit when a command inside an
`if` condition fails.

**Q11.** What is the difference between `"$@"` and `"$*"` when passed to another
command?

**Q12.** `rm *` in a directory of 200,000 files fails with *Argument list too long*.
Which program actually produced the failure — the shell, `rm`, or the kernel?

**Q13.** Spot the bug:

```bash
for f in $(ls *.log); do
  gzip $f
done
```

**Q14.** In a non-matching directory, what does `for f in *.csv; do echo "$f"; done`
print by default? What option changes that?

**Q15.** Which is the correct way to swap stdout and stderr so that only stderr goes
down a pipe?

- A) `cmd 2>&1 | grep x`
- B) `cmd 3>&1 1>&2 2>&3 3>&- | grep x`
- C) `cmd 1>&2 2>&1 | grep x`
- D) `cmd |& grep x`

**Q16.** A program wants to read a file. Which component actually reads the disk, and
how does the program ask it to?

**Q17.** Why must `cd` be a shell builtin rather than a program in `/usr/bin`?

**Q18.** You are on an unfamiliar server. Which single file tells you the distribution
and version on any modern Linux?

**Q19.** `man passwd` shows you how to change a password. Where do you find the format
of `/etc/passwd`?

**Q20.** The machine stops at an emergency-mode prompt right after a change to
`/etc/fstab`. Which boot stage failed, and which program was running when it did?
