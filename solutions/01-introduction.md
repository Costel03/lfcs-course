# Solutions 01 — Introduction

---

## Part A — Orientation

**1.1**

```bash
cat /etc/os-release      # or: hostnamectl
uname -r
```

`/etc/os-release` is a cross-distribution standard, which is why scripts should read it
rather than guess from files like `/etc/redhat-release` that exist on one family only.

**1.2**

```bash
id vagrant
```

On rocky1 `vagrant` is in `wheel`; on ubuntu1 it is in `sudo`. Both are granted rights by
a line in `/etc/sudoers` (`%wheel ALL=(ALL) ALL` / `%sudo ALL=(ALL:ALL) ALL`). Same idea,
different name — one of the differences that breaks copy-pasted instructions.

**1.3**

```bash
man 5 fstab
man 5 crontab
```

Both are section 5, file formats. `man crontab` alone gives section 1, the `crontab`
command, which does not describe the line format at all.

**1.4**

```bash
type -a cd ls echo ll time python3
```

| Name | What it is |
|---|---|
| `cd` | builtin — it *must* be: a child process cannot change its parent's directory |
| `ls` | usually an alias (`ls --color=auto`) **and** a file, `/usr/bin/ls` |
| `echo` | builtin **and** a file, `/usr/bin/echo` — the builtin wins |
| `ll` | alias, on distributions that define it |
| `time` | shell keyword **and** possibly `/usr/bin/time` — they behave differently |
| `python3` | file on disk |

The `cd` answer is the one worth understanding: a program runs as a child process, and a
child cannot change its parent's working directory. So `cd` has to be built into the shell.

**1.5**

```bash
systemd-analyze
systemd-analyze blame | head -5
```

**1.6**

```bash
strace -c ls /etc > /dev/null
```

Redirect stdout so the listing doesn't mix with the summary (strace writes to stderr).
The top entries are usually `openat`, `read`, `close`, `fstat`/`newfstatat`, `mmap` —
mostly the dynamic loader opening shared libraries before `ls` does any work of its own.
`man 2 openat` documents the call; section 2 is system calls.

---

## Part B — The shell

### 🟢 Warm-up

**1.7**

```text
{1..5}              → 1 2 3 4 5
n=5; echo {1..$n}   → {1..$n}      brace expansion runs BEFORE parameter expansion
$((2 + 3 * 4))      → 14           arithmetic honours precedence
${UNSET_VAR:-fallback} → fallback
"$HOME" '$HOME'     → /home/you $HOME
```

The second is the lesson: braces are step 1, variables are step 3, so by the time `$n`
has a value the brace is long gone. Use `seq` or a C-style `for ((i=1;i<=n;i++))`.

**1.8**

```bash
files=(tmp/*.tmp); echo "${#files[@]}"
```

An array holds the glob result; `${#array[@]}` is its length. Parsing `ls` would break
on any name containing a newline.

**1.9**

```bash
nosuchcommand; echo $?      # 127
```

127 is "command not found", 126 is "found but not executable" — usually a missing `+x`
or a script with a bad shebang.

**1.10**

```bash
ls /nonexistent /etc 2> err.txt
```

**1.11**

```bash
find / -name '*.conf' > /dev/null 2>&1
find / -name '*.conf' &> /dev/null        # bash shorthand
```

**1.12**

```bash
echo "${MYVAR:-default}"
declare -p MYVAR          # bash: MYVAR: not found
```

`:-` substitutes without assigning. `:=` would assign — that is the difference.

---

### 🔵 Practical

**1.13**

```bash
for f in data/*.pdf; do printf '%s\n' "$f"; done
```

The glob produces two words regardless of spaces, because splitting never happens to
glob results. Quoting `"$f"` keeps it one argument when passed on.

**1.14**

```bash
for f in $(ls data/*.pdf); do echo "$f"; done    # three lines
```

Word splitting (step 6) runs on the *result* of command substitution (step 4), so
`my report.pdf` is chopped at the space. Globs are not subject to this.

**1.15**

```bash
comm -13 <(sort data/a.txt) <(sort data/b.txt)
```

`comm` needs two sorted files; process substitution supplies them as `/dev/fd/N`
without touching the disk. `-13` suppresses columns 1 and 3, leaving "only in b".

**1.16**

```bash
ls /etc /nonexistent > all.log 2>&1       # both
ls /etc /nonexistent 2>&1 > only-out.log  # stderr went to the terminal
```

`2>&1` copies stdout's *current* destination. Do it before stdout is redirected and
you have copied the terminal.

**1.17**

```bash
count=0
while read -r line; do
  [[ $line == ERROR* ]] && count=$((count+1))
done < logs/app.log
echo "$count"        # 2
```

Redirecting the file into the loop keeps it in the current shell.

**1.18**

```bash
count=0
cat logs/app.log | while read -r line; do count=$((count+1)); done
echo "$count"        # 0
```

Every stage of a pipeline is its own process. The loop incremented a copy of `count`
in a subshell, which then exited, taking the value with it. `shopt -s lastpipe` changes
this in non-interactive bash, but relying on it is fragile.

**1.19**

```bash
find tmp -name '*.tmp' -type f -exec rm {} +
find tmp -name '*.tmp' -type f -print0 | xargs -0 rm
```

`-exec … +` batches as many files per `rm` as fit, like xargs but without a pipe.
`-exec … \;` would fork one `rm` per file — correct but far slower on large trees.

**1.20**

```bash
false | true; echo $?                      # 0
false | true; echo "${PIPESTATUS[@]}"      # 1 0
set -o pipefail; false | true; echo $?     # 1
```

Default behaviour reports only the last stage. In any script that pipes, `pipefail` is
the difference between noticing a failure and shipping corrupt output.

**1.21**

```bash
exec 3> trace.log
echo "step one" >&3
echo "step two" >&3
echo "step three" >&3
exec 3>&-
wc -l trace.log        # 3
```

**1.22**

```bash
find /etc -newer /etc/hostname -type f 2>/dev/null | wc -l
```

**1.23**

```bash
find ~/lab01 -name '*.log' -type f -print0 | xargs -0 -P4 -n1 gzip
```

**1.24**

```bash
show() { printf '[%s]\n' "$@"; }
show "one two" three      # [one two] / [three]

show2() { printf '[%s]\n' "$*"; }
show2 "one two" three     # [one two three]
```

`"$@"` preserves argument boundaries; `"$*"` joins everything with the first character
of `IFS`. When forwarding arguments to another command, `"$@"` is always correct.

---

### 🔴 Challenge

**1.25**

```bash
ls /etc /nonexistent 3>&1 1>&2 2>&3 3>&- | grep -c .
```

Step by step: `3>&1` saves the pipe (which is where stdout currently points) in 3;
`1>&2` sends stdout to the terminal; `2>&3` sends stderr into the saved pipe; `3>&-`
closes the spare. A pipe only ever carries descriptor 1, so the two streams must change
places before `|` can see the errors.

**1.26**

```bash
touch $'bad\nname'
find . -maxdepth 1 -name 'bad*' -print0 | xargs -0 rm
# or simply
rm $'bad\nname'
```

`ls | xargs rm` splits on whitespace including newlines, so it would try to remove two
files called `bad` and `name` — and if a file called `name` existed, it would delete the
wrong one. NUL separation is the only safe form because NUL is the single byte a
filename may not contain.

**1.27**

```bash
for f in *.absent; do echo "$f"; done          # prints *.absent once
shopt -s nullglob
for f in *.absent; do echo "$f"; done          # prints nothing
```

Default POSIX behaviour leaves an unmatched glob as literal text. Any loop that might
match nothing needs `nullglob`, or a guard like `[[ -e $f ]] || continue`.

**1.28**

```bash
set -o pipefail
output=$(ls /etc /nonexistent 2>&1 | tee /dev/tty); rc=$?
echo "exit=$rc"
```

Or without `pipefail`:

```bash
output=$(cmd | tee /dev/tty); rc=${PIPESTATUS[0]}
```

The trap is that `$?` after a pipeline is `tee`'s status, and `tee` almost always
succeeds — so the real failure is masked. `/dev/tty` writes to the terminal directly,
bypassing the capture.

**1.29**

```text
prints 1
```

`echo $x | read x` runs `read` in a subshell on the right-hand side of the pipe. It
does set `x` — in that subshell, which then exits. The parent's `x` is untouched.

Working versions:

```bash
read -r x < <(echo 5)     # process substitution: read runs in this shell
x=$(echo 5)               # simpler, when you just want the output
```

**1.30**

```bash
find /path -type f -mtime +30 -delete
find /path -type f -mtime +30 -print0 | xargs -0 rm
```

The limit is `ARG_MAX` — the kernel's cap on the combined size of the argument vector
and environment passed to `execve()`, typically around 2 MB (`getconf ARG_MAX`). The
glob is expanded *by the shell*, so `rm *` tries to hand the kernel 50,000 filenames in
one call and fails with `E2BIG` before `rm` even starts.

`find -delete` never builds that list. `xargs` builds several lists, each sized to fit.

Note `-delete` must come after the filters: `find -delete -mtime +30` deletes
everything, because `find` evaluates left to right.

---

## If you got stuck

The three worth repeating tomorrow are **1.18** (subshell variable loss), **1.25**
(descriptor swap) and **1.30** (ARG_MAX). Each represents a class of bug you will meet
in real scripts, and each is invisible until it bites.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | **C** | Brace expansion is step 1, before variables exist |
| 2 | `{1..$n}`, literally | Braces expand before `$n` has a value |
| 3 | **B** | Word splitting runs on the unquoted expansion |
| 4 | **B** | `2>&1` copies stdout's destination *at that moment*; in A it copies the terminal |
| 5 | `0` | Each pipeline stage is a subshell; the loop's `count` dies with it |
| 6 | `0`; `set -o pipefail` | A pipeline reports the last stage only |
| 7 | SIGKILL (128+9), usually the OOM killer — check `dmesg` / `journalctl -k` | The process cannot log its own SIGKILL |
| 8 | **B** | Process substitution is a path backed by a pipe |
| 9 | NUL cannot appear in a filename; newlines and spaces can | Default `xargs` splits on whitespace |
| 10 | **False** | `set -e` is suppressed inside `if`, `while`, `&&`, `\|\|` |
| 11 | `"$@"` keeps each argument separate; `"$*"` joins them into one | Use `"$@"` to forward arguments |
| 12 | The **kernel** — `execve()` returns `E2BIG`; the shell reports it and `rm` never starts | The glob was expanded by the shell into one huge argument list |
| 13 | `$(ls …)` word-splits names with spaces; `$f` is unquoted | Write `for f in *.log; do gzip -- "$f"; done` |
| 14 | `*.csv` once, literally; `shopt -s nullglob` | Unmatched globs pass through unchanged |
| 15 | **B** | D (`\|&`) pipes *both* streams; A also pipes both; C sends both to stderr |
| 16 | The kernel; the program makes system calls (`openat`, `read`) | User space never touches hardware directly |
| 17 | A child process cannot change its parent's working directory | `cd` must run inside the shell itself |
| 18 | `/etc/os-release` | A cross-distribution standard |
| 19 | `man 5 passwd` | Section 5 is file formats |
| 20 | Stage 5 — systemd (PID 1) failed to mount an fstab entry | The kernel and initramfs had already succeeded |
