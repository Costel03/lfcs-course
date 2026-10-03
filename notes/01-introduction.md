# Chapter 01 — Introduction

What Linux actually is, how its pieces fit, and then the shell — the tool you will use
for every other chapter. The shell half is where most people discover they had been
guessing: almost every strange bug in a script comes from an expansion you did not expect.

## Objectives

After this chapter you can:

- Explain the split between kernel space and user space, and what a system call is.
- Tell the major distribution families apart and know what differs between them.
- Describe the boot sequence from firmware to login prompt at a high level.
- Find authoritative documentation on the machine itself.
- State the order in which bash expands a command line, and use it to predict output.
- Explain why `for f in $(ls)` is broken and what to write instead.
- Redirect any file descriptor, including swapping stdout and stderr.
- Use process substitution to feed a command where a filename is expected.
- Build safe pipelines with `find -exec` and `xargs -0`, and read `PIPESTATUS`.

---

## 1. What "Linux" means

Strictly, **Linux is a kernel** — the program that owns the hardware, schedules
processes, manages memory, and enforces permissions. Everything you interact with — the
shell, `ls`, `systemd`, `ssh` — is ordinary software running *on top of* the kernel.

```text
┌───────────────────────────────────────────────────────────┐
│  user space     bash   nginx   sshd   systemd   ls   vim  │
│                 ─────────── C library (glibc) ─────────── │
├──────────────────────── system calls ─────────────────────┤
│  kernel space   scheduler · memory · VFS · network stack  │
│                 drivers · security (SELinux, namespaces)  │
├───────────────────────────────────────────────────────────┤
│  hardware       CPU · RAM · disks · NICs                  │
└───────────────────────────────────────────────────────────┘
```

A program in user space cannot touch hardware directly. To open a file, send a packet or
start a process it asks the kernel through a **system call** — `openat()`, `sendto()`,
`fork()`, `execve()`. This boundary is why one crashing program cannot take down the
machine, and why `strace` (chapter 05) can show you exactly what any program is doing:
it watches the system calls.

```bash
uname -r            # kernel release: 5.14.0-427.el9.x86_64
uname -a            # everything uname knows
cat /proc/version   # the kernel describing itself
```

---

## 2. Distributions

A **distribution** is the kernel plus a chosen set of user-space software, a package
manager, defaults, and a support policy. The commands are 95% the same everywhere; the
remaining 5% is what this table is for.

| | Debian family | RedHat family |
|---|---|---|
| Examples | Debian, Ubuntu | RHEL, Rocky, AlmaLinux, Fedora, CentOS Stream |
| Package format | `.deb` | `.rpm` |
| Package tools | `apt`, `dpkg` | `dnf`, `rpm` |
| Admin group | `sudo` | `wheel` |
| Mandatory access control | AppArmor | SELinux |
| Default firewall front-end | `ufw` | `firewalld` |
| Network config | Netplan (Ubuntu), `/etc/network/interfaces` | NetworkManager (`nmcli`) |
| Main system log file | `/var/log/syslog` | `/var/log/messages` |

Enterprise distributions (RHEL, Ubuntu LTS) trade new features for a long, stable support
window — RHEL major versions are supported for ten years. That is why production often
runs older software than your laptop, and why "it works on my machine" is a real problem.

```bash
cat /etc/os-release      # which distribution and version — works everywhere
hostnamectl              # the same, plus kernel and hardware
```

---

## 3. The boot sequence at a glance

Chapter 12 covers this in depth. You need the outline now, because several later
chapters (storage, services) can break a boot, and you should know which stage you broke.

| Stage | What runs | Configured in | If it fails you see |
|---|---|---|---|
| 1. Firmware | UEFI or BIOS: finds a bootable disk | firmware setup | no boot device |
| 2. Bootloader | GRUB2: loads kernel + initramfs | `/etc/default/grub` | GRUB prompt or menu |
| 3. Kernel | initialises hardware, mounts initramfs | kernel command line | kernel panic |
| 4. initramfs | finds and mounts the real root filesystem | `dracut` / `update-initramfs` | dracut emergency shell |
| 5. systemd (PID 1) | starts services, mounts `/etc/fstab` | `/etc/systemd/` | emergency or rescue mode |
| 6. Login | `getty`, `sshd`, display manager | unit files | no prompt, or SSH refused |

```bash
systemd-analyze               # how long each stage took
systemd-analyze blame | head  # slowest services
journalctl -b                 # everything logged since this boot
```

---

## 4. Where help lives

The machine you are fixing has its documentation installed. Production machines are
often not on the internet; the habit of reading the manual first pays off there.

**In the LFCS exam this is all you get:** `man` pages and `/usr/share/doc`, no browser.
Every chapter of this course points you to the man page for what it teaches. Use them
from day one, so that on exam day finding an option in `man nmcli` or an example in
`/usr/share/doc` takes seconds rather than minutes.

```bash
man ls                 # the manual page
man 5 passwd           # section 5: the FILE FORMAT, not the command
man -k partition       # search all page descriptions (same as apropos)
ls --help              # short summary
info coreutils         # longer GNU documentation
type cd                # builtin, alias, function or file?
command -v python3     # what would run — script-safe version of `which`
ls /usr/share/doc/     # package documentation and examples
```

Manual pages are divided into sections. The ones you will use:

| Section | Contains | Example |
|---|---|---|
| 1 | user commands | `man 1 passwd` — the program |
| 5 | file formats and config files | `man 5 passwd`, `man 5 crontab`, `man 5 fstab` |
| 7 | overviews and conventions | `man 7 signal`, `man 7 regex`, `man 7 hier` |
| 8 | administration commands | `man 8 mount`, `man 8 useradd` |

When a name is both a command and a config file — `passwd`, `crontab`, `fstab` — you must
ask for section 5 to read about the file. `man 7 hier` is the filesystem layout, and is
where chapter 02 begins.

Inside `man`: `/word` searches, `n` jumps to the next match, `G` goes to the end, `q` quits.

---

## 5. The shell

When you type a command you are not talking to Linux. You are talking to a **shell** —
usually `bash` — a user-space program that reads your line, rewrites it, and then asks
the kernel to run something. The rewriting is the important part: by the time a program
starts, the shell has already expanded variables, globs and quotes, and the program never
sees what you typed. The rest of this chapter is about that rewriting.

```bash
echo $SHELL          # your login shell
echo $0              # the shell running right now
echo $BASH_VERSION
```

---

## 6. Expansion order — the thing everything else depends on

Bash rewrites your line before any program runs, in a fixed order:

| # | Expansion | Example in → out |
|---|---|---|
| 1 | Brace | `f{1,2}.txt` → `f1.txt f2.txt` |
| 2 | Tilde | `~/log` → `/home/costel/log` |
| 3 | Parameter | `$USER`, `${VAR:-default}` |
| 4 | Command substitution | `$(hostname)` → `web01` |
| 5 | Arithmetic | `$((2+3))` → `5` |
| 6 | **Word splitting** | `a b c` becomes three words |
| 7 | Pathname (globbing) | `*.log` → matching filenames |
| 8 | Quote removal | the quotes themselves are stripped |

Step 6 is where scripts die. Word splitting happens **after** variables are expanded,
so an unquoted variable containing spaces becomes several arguments:

```bash
f="my report.pdf"
rm $f        # rm receives TWO arguments: "my" and "report.pdf"
rm "$f"      # correct — one argument
```

Brace expansion happens *first*, before variables exist:

```bash
n=3
echo {1..$n}      # prints the literal {1..3} — braces expanded before $n
eval echo {1..$n} # prints 1 2 3, but eval is a last resort
seq 1 "$n"        # what you should actually write
```

> **Rule:** quote every variable expansion unless you have a specific reason not to.
> The exceptions are rare and deliberate; the failures are common and silent.

### Why `for f in $(ls)` is wrong

```bash
for f in $(ls); do echo "$f"; done      # broken
for f in *; do echo "$f"; done          # correct
```

`$(ls)` produces one string, which word splitting then chops at every space — so
`my report.pdf` becomes two iterations. A glob produces a proper list of names, spaces
intact, and works when a directory is empty only if you check. `ls` output is for
humans; globs and `find` are for programs.

---

## 7. Quoting, precisely

| Form | Variables | Globs | Word splitting | Use for |
|---|---|---|---|---|
| `bare` | yes | yes | yes | almost never with variables |
| `"double"` | yes | no | no | nearly always |
| `'single'` | no | no | no | literal text, regex, awk programs |
| `$'ansi'` | escapes | no | no | `$'\t'`, `$'\n'` |

```bash
printf '%s\n' 'a	b'      # literal tab typed in
printf '%s\n' $'a\tb'    # tab via escape
```

`"$@"` versus `"$*"` is the classic trap:

```bash
set -- "one two" three
printf '[%s]\n' "$@"     # [one two] [three]   — each argument preserved
printf '[%s]\n' "$*"     # [one two three]     — joined into one string
```

`"$@"` is what you want when passing arguments through to another command. Always.

---

## 8. File descriptors and redirection

Every process starts with three open descriptors:

| FD | Name | Default |
|---|---|---|
| 0 | stdin | keyboard |
| 1 | stdout | terminal |
| 2 | stderr | terminal |

```bash
cmd > out.log            # stdout to file (truncates)
cmd >> out.log           # append
cmd 2> err.log           # stderr only
cmd > all.log 2>&1       # both to one file — order matters
cmd &> all.log           # bash shorthand for the same
cmd < input.txt          # stdin from file
cmd 2>/dev/null          # discard errors
```

**Order matters and catches everyone:**

```bash
cmd > file 2>&1    # stdout→file, then stderr→wherever stdout now points = file ✓
cmd 2>&1 > file    # stderr→terminal (stdout still terminal), then stdout→file ✗
```

Read `2>&1` as "make descriptor 2 point at whatever descriptor 1 points at *right now*".
It copies the current destination, it does not create a permanent link.

Swapping the two streams needs a spare descriptor:

```bash
cmd 3>&1 1>&2 2>&3 3>&-
```

Save stdout in 3, point stdout at stderr, point stderr at the saved copy, close 3.

Useful in scripts:

```bash
exec 3> /tmp/trace.log      # open FD 3 for the rest of the script
echo "step one" >&3         # write to it
exec 3>&-                   # close it

exec > >(tee -a run.log)    # send everything the script prints to a log as well
exec 2>&1
```

---

## 9. Process substitution

Some commands insist on a filename and will not read a pipe. Process substitution
gives them one backed by a running command.

```bash
diff <(sort a.txt) <(sort b.txt)          # compare without temp files
comm -13 <(sort old) <(sort new)          # lines only in new
while read -r line; do :; done < <(cmd)   # no subshell — see below
```

`<(cmd)` is replaced by a path like `/dev/fd/63`. The command runs in parallel and its
output appears when the path is read.

The last example matters more than it looks:

```bash
count=0
cat file | while read -r l; do count=$((count+1)); done
echo "$count"        # prints 0 — the loop ran in a subshell

count=0
while read -r l; do count=$((count+1)); done < <(cat file)
echo "$count"        # prints the real count
```

Each stage of a pipeline is a separate process. Variables set inside a piped `while`
loop die with it. Redirecting from process substitution keeps the loop in your shell.

---

## 10. Exit codes and pipelines

```bash
cmd; echo $?         # 0 = success, anything else = failure
```

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | general error |
| 2 | misuse of a builtin |
| 126 | found but not executable |
| 127 | command not found |
| 130 | killed by Ctrl-C (128 + SIGINT 2) |
| 137 | killed by SIGKILL (128 + 9) — often the OOM killer |

A pipeline reports only the **last** command's status:

```bash
false | true; echo $?        # 0 — the failure is invisible
```

Two fixes:

```bash
echo "${PIPESTATUS[@]}"      # status of every stage: "1 0"
set -o pipefail              # pipeline fails if any stage fails
```

`137` is worth memorising. When a container or service dies with 137 and no error in
its own logs, the kernel killed it — check `dmesg` for the OOM killer before assuming
the application crashed.

---

## 11. find and xargs, safely

```bash
find /var/log -name '*.log' -type f -mtime +7
find . -maxdepth 2 -type d -name '.git'
find /etc -newer /etc/hostname
find . -type f -size +100M
```

Acting on results — three ways, in increasing order of quality:

```bash
find . -name '*.tmp' -exec rm {} \;      # one rm per file: correct but slow
find . -name '*.tmp' -exec rm {} +       # batches files into few calls: better
find . -name '*.tmp' -print0 | xargs -0 rm    # parallel-capable, handles anything
```

The `-print0` / `-0` pairing separates names with a NUL byte instead of a newline.
Since NUL is the one byte a filename cannot contain, this is the only form that is
safe against names containing spaces, quotes or newlines — and a filename containing
a newline is legal and has been used deliberately in attacks.

```bash
find . -name '*.log' -print0 | xargs -0 -P4 -n10 gzip
```

`-P4` runs four in parallel, `-n10` passes ten files per invocation.

---

## 12. Gotchas

**`set -e` does not do what you think.** It exits on an unchecked failure, but not
inside `if`, `while`, `&&`, `||`, or for any command but the last in a pipeline
without `pipefail`. Appendix A covers this properly; for now, do not treat it as a
safety net.

**Globs that match nothing are passed through literally.**

```bash
ls *.nothing        # ls: cannot access '*.nothing'
shopt -s nullglob   # make non-matching globs expand to nothing instead
```

A loop over `*.log` in an empty directory runs once with the literal string `*.log`.

**`$(...)` strips trailing newlines**, all of them. Usually helpful, occasionally the
cause of a corrupted file when you round-trip content through a variable.

**Aliases do not exist in scripts.** They are an interactive feature. A script that
works when pasted into your terminal and fails when run is usually hitting this.

**`~` is expanded by the shell, never by the program.** In a config file, a quoted
string, or a variable read from a file, `~` stays a literal tilde.

---

## 13. Commands and constructs introduced

| Item | Purpose |
|---|---|
| `${VAR:-default}` | value, or a fallback if unset |
| `$(( ))` | arithmetic |
| `$( )` | command substitution |
| `<( )` `>( )` | process substitution |
| `exec` | redirect the shell's own descriptors |
| `PIPESTATUS` | exit status of every pipeline stage |
| `set -o pipefail` | fail a pipeline if any stage fails |
| `shopt -s nullglob` | non-matching globs vanish |
| `find -print0` / `xargs -0` | NUL-safe file handling |
| `"$@"` | all arguments, each preserved |
| `type`, `command -v` | what is this name, really |

---

➡ **Next:** [quiz](../quizzes/01-introduction.md) → [lab](../labs/01-introduction.md)
