# Chapter 03 — Text processing and Git

Configuration is text, logs are text, and most command output is text. An administrator
who can slice text quickly answers questions in one line that others write a script for.
The second half covers Git, which the LFCS lists as an Essential Command — and which is
how configuration is kept, reviewed and rolled back in any team you will work in.

## Objectives

After this chapter you can:

- Write basic and extended regular expressions and know which tool uses which.
- Search with `grep` using context, inversion, counts and recursive search.
- Edit files non-interactively with `sed`, safely, including in place.
- Extract, filter and summarise columns with `awk`.
- Build pipelines with `cut`, `sort`, `uniq`, `tr`, `wc`, `paste`, `column`, `tee`.
- Compare files with `diff`, and produce and apply patches.
- Use Git to clone, commit, branch, merge, resolve a conflict, and push to a remote.

---

## 1. Regular expressions

A regular expression describes a pattern of text. Two dialects matter:

| Dialect | Used by | Note |
|---|---|---|
| **Basic (BRE)** | `grep`, `sed` by default | `+ ? | ( ) { }` must be escaped to be special |
| **Extended (ERE)** | `grep -E`, `sed -E`, `awk` | those characters are special as written |

Use ERE whenever you can — it reads the way regexes look everywhere else.

| Pattern | Matches |
|---|---|
| `.` | any single character |
| `^` / `$` | start / end of line |
| `*` | zero or more of the previous item |
| `+` | one or more (ERE) |
| `?` | zero or one (ERE) |
| `{n,m}` | between n and m (ERE) |
| `[abc]` / `[^abc]` | one of / none of |
| `[a-z0-9]` | a range |
| `[[:digit:]]`, `[[:space:]]`, `[[:alpha:]]` | POSIX classes — locale-safe |
| `\b` | word boundary (GNU) |
| `(a|b)` | alternation and grouping (ERE) |
| `\1` | back-reference to group 1 |

Examples you will reuse:

```text
^#                         a comment line
^\s*$                      blank or whitespace-only line (GNU \s)
^[^#]                      a line that doesn't start with #
[0-9]{1,3}(\.[0-9]{1,3}){3}   an IPv4-looking address (ERE)
^[a-z_][a-z0-9_-]*$        a valid-looking username
```

**Quote every regex in single quotes.** Otherwise the shell expands `*`, `$` and `\`
before the tool sees them.

`man 7 regex` documents the syntax — and it is available in the exam.

---

## 2. grep

```bash
grep 'error' app.log              # lines containing error
grep -i 'error' app.log           # case-insensitive
grep -v '^#' sshd_config          # lines NOT matching
grep -c 'Failed' auth.log         # count of matching lines
grep -n 'listen' nginx.conf       # with line numbers
grep -w 'root' /etc/passwd        # whole word: not "chroot"
grep -o '[0-9]\+' file            # print only the matched part
grep -E 'warn|error' app.log      # alternation
grep -F '1.2.3.4' access.log      # fixed string — dots are literal
grep -r 'PermitRootLogin' /etc    # recursive
grep -rl 'TODO' src/              # only filenames
grep -A3 -B1 'panic' kern.log     # 3 lines after, 1 before
grep -C2 'OOM' syslog             # 2 lines of context each side
grep -q 'pattern' file && echo found   # quiet: just the exit code
```

The most useful idiom on any config file — show what is actually set:

```bash
grep -Ev '^\s*(#|$)' /etc/ssh/sshd_config
```

`-F` matters more than people think: in `grep '1.2.3.4'`, each `.` matches any character,
so it also matches `1a2b3c4`.

---

## 3. sed — the stream editor

`sed` reads input line by line, applies commands, and prints the result.

### Substitute

```bash
sed 's/old/new/' file         # first match on each line
sed 's/old/new/g' file        # every match
sed 's/old/new/2' file        # only the second match
sed 's/old/new/gI' file       # case-insensitive (GNU)
sed 's#/usr/local#/opt#g' f   # any delimiter — useful with paths
```

### Addresses — which lines to act on

```bash
sed -n '5p' file              # print line 5 only (-n: no default output)
sed -n '10,20p' file          # a range
sed -n '/^\[mysqld\]/,/^\[/p' my.cnf    # from one pattern to the next
sed '/^#/d' file              # delete comment lines
sed '/^$/d' file              # delete blank lines
sed '3s/a/b/' file            # substitute only on line 3
sed '/^Port/s/22/2222/' sshd_config     # substitute only on matching lines
```

### Insert, append, change

```bash
sed '/^\[main\]/a newkey=value' file      # line after the match
sed '/^\[main\]/i # comment' file         # line before the match
sed '/^PasswordAuthentication/c PasswordAuthentication no' sshd_config
```

### Groups and back-references

```bash
echo 'John Smith' | sed -E 's/(\w+) (\w+)/\2, \1/'      # Smith, John
sed -E 's/^(\s*)#\s*(PermitRootLogin)/\1\2/' sshd_config  # uncomment a directive
```

### In-place editing

```bash
sed -i.bak 's/^Port 22/Port 2222/' /etc/ssh/sshd_config
```

`-i` rewrites the file; `-i.bak` keeps a copy first. On a config file, always keep the
backup or check with `sed -n` before using `-i`. Note that `sed -i` replaces the file
with a new one — the inode changes, which matters for hard links and for SELinux
contexts on some systems.

### Idempotent edits

A sed edit that is safe to run twice is worth getting right:

```bash
# set Port to 2222 whether the line is commented, set to something else, or already 2222
sed -E -i 's/^#?\s*Port\s+.*/Port 2222/' /etc/ssh/sshd_config
grep -q '^Port 2222' /etc/ssh/sshd_config || echo 'Port 2222' >> /etc/ssh/sshd_config
```

---

## 4. awk — columns and logic

`awk` splits each line into fields: `$1`, `$2`, …, `$NF` (last). `$0` is the whole line.

```bash
awk '{print $1}' access.log                 # first field (split on whitespace)
awk -F: '{print $1, $7}' /etc/passwd        # custom separator
awk -F: '$3 >= 1000 {print $1}' /etc/passwd # condition: regular users
awk '/ERROR/ {print $2}' app.log            # pattern, then action
awk 'NR==1 || NR==5' file                   # line numbers
awk 'NF > 0' file                           # non-empty lines
awk '{print $NF}' file                      # last field
awk '{print NR": "$0}' file                 # number the lines
```

### Summing and counting

```bash
awk '{sum += $10} END {print sum}' access.log              # total bytes sent
awk '{n[$1]++} END {for (ip in n) print n[ip], ip}' access.log | sort -rn | head
df -P | awk 'NR>1 && $5+0 > 80 {print $6, $5}'            # filesystems over 80%
```

`BEGIN` runs before input, `END` after it. Associative arrays (`n[$1]++`) are what make
awk powerful for summaries.

### printf for aligned output

```bash
awk -F: '{printf "%-15s %5d %s\n", $1, $3, $7}' /etc/passwd
```

---

## 5. The rest of the toolkit

```bash
cut -d: -f1,7 /etc/passwd         # fields 1 and 7, colon-separated
cut -c1-10 file                   # characters 1–10
sort file                         # alphabetical
sort -n / -h / -r                 # numeric / human sizes / reverse
sort -t: -k3,3n /etc/passwd       # by field 3, numerically
sort -u file                      # sorted and deduplicated
uniq -c                           # count adjacent duplicates — sort first!
tr 'a-z' 'A-Z'                    # translate characters
tr -d '\r' < win.txt > unix.txt   # strip Windows line endings
tr -s ' '                         # squeeze repeated spaces
wc -l / -w / -c                   # lines / words / bytes
paste -sd, file                   # join all lines with commas
column -t -s:                     # align into a table
tee file                          # copy stdin to a file AND stdout
head -n 5 / tail -n 5             # first / last lines
nl file                           # number lines
```

`uniq` only removes **adjacent** duplicates, which is why it is almost always `sort | uniq`.

### The canonical pipeline

The top 10 client IPs in a web log:

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
```

Read it as: extract → group identical values together → count each group → order by
count → keep the top. Most log questions are this shape.

### Writing with sudo

```bash
sudo echo 'x' > /etc/file          # FAILS: the redirect runs in YOUR shell, not sudo's
echo 'x' | sudo tee /etc/file      # works
echo 'x' | sudo tee -a /etc/file   # append
```

---

## 6. diff and patch

```bash
diff old.conf new.conf             # classic output
diff -u old.conf new.conf          # unified format — what everyone reads
diff -r dir1 dir2                  # recursive
diff <(sort a) <(sort b)           # compare sorted versions without temp files
```

A unified diff can be applied elsewhere:

```bash
diff -u original.conf fixed.conf > fix.patch
patch original.conf < fix.patch    # apply
patch -R original.conf < fix.patch # reverse
```

---

## 7. Git

Git tracks the history of a directory: who changed what, when, and why. For an
administrator it is a safety net — every config change can be reviewed and reverted.

### Concepts

```text
working tree ──git add──▶ staging area ──git commit──▶ local repository ──git push──▶ remote
     ▲                                                          │
     └─────────────────────── git restore / checkout ───────────┘
```

- **Working tree** — the files you edit.
- **Staging area (index)** — what will go into the next commit.
- **Commit** — a snapshot with an author, a message and a parent.
- **Branch** — a movable name pointing at a commit.
- **Remote** — another copy of the repository, usually on a server.

### First-time setup

```bash
git config --global user.name  "Costel Iacob"
git config --global user.email "costel@example.com"
git config --global init.defaultBranch main
git config --list --show-origin
```

### Everyday cycle

```bash
git init                         # new repository here
git clone url-or-path            # copy an existing one
git status                       # what changed — run it constantly
git add file / git add -p        # stage a file / stage hunks interactively
git commit -m "Set SSH port to 2222"
git log --oneline --graph -10    # history
git diff                         # unstaged changes
git diff --staged                # staged changes
git show HEAD                    # the last commit
```

### Undoing

| Want to | Command |
|---|---|
| Discard edits to a file | `git restore file` |
| Unstage a file | `git restore --staged file` |
| Change the last commit's message (not pushed) | `git commit --amend` |
| Undo a pushed commit safely | `git revert <commit>` — new commit that reverses it |
| Throw away local commits | `git reset --hard <commit>` — **destructive** |

`revert` is for anything others may have pulled; it adds history instead of rewriting it.

### Branches and merging

```bash
git branch                       # list
git switch -c feature-x          # create and switch (older: git checkout -b)
git switch main
git merge feature-x
git branch -d feature-x          # delete when merged
```

### Conflicts

When both branches changed the same lines, `git merge` stops:

```text
<<<<<<< HEAD
Port 2222
=======
Port 2200
>>>>>>> feature-x
```

Edit the file to the version you want, remove the markers, then:

```bash
git add sshd_config
git commit                       # completes the merge
git merge --abort                # or give up and return to before the merge
```

### Remotes

```bash
git remote -v
git remote add origin ssh://git@server/srv/git/configs.git
git push -u origin main          # first push; sets the upstream
git pull                         # fetch + merge (or rebase, if configured)
git fetch                        # download without merging
```

### A server-side repository over SSH

No hosting service needed — any machine with SSH and Git can be a remote:

```bash
# on the server
sudo mkdir -p /srv/git && sudo chown "$USER" /srv/git
git init --bare /srv/git/configs.git

# on the client
git clone ubuntu2:/srv/git/configs.git
```

A **bare** repository has no working tree — just the history. Remotes you push to should
be bare; pushing into a checked-out branch of a non-bare repository is refused.

### .gitignore and tags

```bash
echo '*.bak' >> .gitignore
echo 'secrets.env' >> .gitignore
git tag -a v1.0 -m "First production config"
git push --tags
```

Never commit secrets. Once pushed, a secret is in the history even if you delete the file
in a later commit — rotate it.

`man git`, `man git-commit`, `man gitignore` and `git help <cmd>` are all available offline.

---

## 8. Gotchas

**`uniq` without `sort`** gives wrong counts silently.

**Greedy matching.** `s/".*"/X/` on `"a" and "b"` replaces from the first quote to the
*last*. Use `"[^"]*"` to match a single quoted string.

**`sudo` with a redirect** — the redirect happens before sudo; use `tee`.

**`sed -i` on a symlink** replaces the link with a regular file. Use
`sed -i --follow-symlinks`.

**CRLF line endings.** A file edited on Windows has `\r` at line ends; scripts fail with
`bad interpreter: /bin/bash^M`, and `grep 'value$'` stops matching. `cat -A` shows them as
`^M`; `tr -d '\r'` or `dos2unix` removes them.

**`git add .` adds everything** including files you meant to ignore. Write `.gitignore`
before the first commit.

---

## 9. Commands introduced

| Command | Purpose |
|---|---|
| `grep`, `grep -E`, `grep -F` | search |
| `sed` | stream editing, in place |
| `awk` | field processing and summaries |
| `cut`, `sort`, `uniq`, `tr`, `wc`, `paste`, `column`, `nl` | shaping text |
| `tee` | write a stream to a file and stdout (and with sudo) |
| `diff -u`, `patch` | compare and apply changes |
| `git init/clone/add/commit/log/diff/branch/switch/merge/push/pull/revert/tag` | version control |
| `man 7 regex` | regex reference, available offline |

---

➡ **Next:** [quiz](../quizzes/03-text-git.md) → [lab](../labs/03-text-git.md)
