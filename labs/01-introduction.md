# Lab 01 — Introduction

30 tasks. Part A runs on **rocky1** and **ubuntu1**; Part B anywhere with bash.
Do not open the solutions until you have an answer you believe.

**Setup:**

```bash
mkdir -p ~/lab01/{data,logs,tmp} && cd ~/lab01
printf 'alpha\nbravo\ncharlie\n' > data/a.txt
printf 'bravo\ncharlie\ndelta\n' > data/b.txt
touch 'data/my report.pdf' 'data/normal.pdf' data/.hidden
printf 'ERROR disk\nINFO ok\nERROR net\nWARN slow\n' > logs/app.log
for i in $(seq 1 20); do echo "line $i" > "tmp/file$i.tmp"; done
```

**Cleanup:** `rm -rf ~/lab01`

---

## Part A — Orientation

**1.1** On both rocky1 and ubuntu1, print the distribution name and version, and the
kernel release, using commands that work identically on both.
*Verify:* you used the same two commands on each machine.

**1.2** On each machine, find out which group grants `sudo` rights, without looking at
the sudoers file. (Hint: who is the `vagrant` user, and what is it a member of?)
*Verify:* rocky1 answers `wheel`, ubuntu1 answers `sudo`.

**1.3** Find the documentation for the *format* of `/etc/fstab` and the *format* of a
crontab line. Name the manual section each lives in.
*Verify:* both are section 5.

**1.4** For each of these, find out whether it is a shell builtin, an alias, a function,
or a file on disk: `cd`, `ls`, `echo`, `ll`, `time`, `python3`.
*Verify:* `type -a` agrees with your answers — note that `echo` and `time` are *both*.

**1.5** Show how long the last boot took and the five slowest units.
*Verify:* the output has a `Startup finished in …` line and five unit names.

**1.6** Use `strace -c` on `ls /etc` to count the system calls it makes. Which syscall
appears most often?
*Verify:* the summary table names it; you can explain what it does using its man page in
section 2.

---

## Part B — The shell

### 🟢 Warm-up

**1.7** Predict, then check, what each prints. Name the expansion responsible for each
surprise.

```bash
echo {1..5}
n=5; echo {1..$n}
echo $((2 + 3 * 4))
echo ${UNSET_VAR:-fallback}
echo "$HOME" '$HOME'
```
*Verify:* your predictions match exactly.

**1.8** Print only the *number* of `.tmp` files in `tmp/`, using a glob, not `ls | wc`.
*Verify:* prints `20`.

**1.9** Show the exit code of a command that does not exist.
*Verify:* you see `127`.

**1.10** Redirect only errors from `ls /nonexistent /etc` to a file, letting normal
output stay on screen.
*Verify:* `cat` of the file shows the error and nothing else.

**1.11** Discard all output, both streams, from `find / -name '*.conf'`.
*Verify:* nothing appears on your terminal at all.

**1.12** Use `${VAR:-}` to print a default for an unset variable without assigning it.
Then prove it is still unset.
*Verify:* `declare -p MYVAR` reports it as not found.

---

### 🔵 Practical

**1.13** The file `data/my report.pdf` has a space in its name. Write a loop that prints
each `.pdf` file in `data/` on its own line, correctly, including that one.
*Verify:* exactly two lines, and the name with the space is intact on one of them.

**1.14** Now write the *broken* version using `$(ls)` and observe the difference.
Explain in one sentence which expansion step caused it.
*Verify:* the broken version prints three lines.

**1.15** Show the lines that appear in `data/b.txt` but not in `data/a.txt`, without
creating any temporary files.
*Verify:* output is exactly `delta`.

**1.16** Redirect both stdout and stderr of `ls /etc /nonexistent` into `all.log`,
then prove that reversing the order of the redirections gives a different result.
*Verify:* one file contains both streams, the other contains only stdout.

**1.17** Count the `ERROR` lines in `logs/app.log`, storing the count in a variable
that still holds the value after the loop ends. Use a `while read` loop, not `grep -c`.
*Verify:* `echo "$count"` prints `2` after the loop.

**1.18** Do the same thing with the pipeline form (`cat … | while read …`) and observe
that it prints `0`. Explain why.
*Verify:* you can state which process the variable was set in.

**1.19** Delete every `.tmp` file under `tmp/` using `find` with `-exec … +`, then
recreate them and delete them again using `xargs -0`.
*Verify:* `ls tmp | wc -l` prints `0` both times.

**1.20** Write a pipeline where the first command fails and the last succeeds. Show
that `$?` is 0, then make the failure visible two different ways.
*Verify:* you can show a non-zero result using both `PIPESTATUS` and `pipefail`.

**1.21** Open file descriptor 3 pointing at `trace.log`, write three lines to it from
different points in a script, then close it.
*Verify:* `wc -l trace.log` prints `3`.

**1.22** Find all files under `/etc` modified more recently than `/etc/hostname`,
counting them, with all permission errors suppressed.
*Verify:* you get a number and no `Permission denied` noise.

**1.23** Write a one-liner that gzips every `.log` under `~/lab01`, four at a time in
parallel, safe against spaces in filenames.
*Verify:* `logs/app.log.gz` exists.

**1.24** Demonstrate the difference between `"$@"` and `"$*"` using a function that
prints each argument in brackets. Call it with an argument containing a space.
*Verify:* the two forms produce a visibly different number of bracketed items.

---

### 🔴 Challenge

**1.25** Pipe **only stderr** through `grep -c .` to count error lines, while stdout
passes to your terminal untouched. Test with `ls /etc /nonexistent`.

This needs the descriptor-swap idiom — a pipe only ever carries stdout, so the two
streams must exchange places first.

*Verify:* you see the `/etc` listing on screen, and a count of `1` from the grep.

**1.26** Create a file whose name contains a newline character. Then delete it safely.
Show that `ls | xargs rm` would have failed on it.
*Verify:* the file is gone and no other file was removed.

**1.27** A glob matching nothing is passed through literally. Write a loop over
`*.absent` that runs zero times, and prove the default behaviour runs it once.
*Verify:* one version prints nothing, the other prints `*.absent`.

**1.28** Capture a command's output into a variable while *also* seeing it on screen in
real time, and preserve the command's exit code — not `tee`'s.
*Verify:* the variable holds the output, you saw it live, and `$?` reflects the real
command.

**1.29** Explain what this prints and why, before running it:

```bash
x=1
echo $x | read x
echo $x
```
*Verify:* your explanation names the subshell, and you can write a version that
actually updates `x`.

**1.30** You are handed a directory of 50,000 files and told to delete those older than
30 days. `rm *` fails with "Argument list too long". Write a solution that works, and
explain the limit you hit.
*Verify:* your command uses no glob expansion of the full list, and you can name the
kernel limit involved.

---

## Self-check

- [ ] I can draw the kernel/user-space boundary and say where a system call crosses it
- [ ] I know which commands differ between the Debian and RedHat families
- [ ] I can find any config file's format in `man` section 5

- [ ] I can list the expansion order from memory and use it to debug
- [ ] I always quote variable expansions, and know the exceptions
- [ ] I can redirect any descriptor, in the right order
- [ ] I know why a piped `while` loop loses its variables
- [ ] I use `-print0`/`-0` whenever filenames come from user data
- [ ] I check `PIPESTATUS` or set `pipefail` before trusting a pipeline

➡ **Solutions:** [solutions/01-introduction.md](../solutions/01-introduction.md)
