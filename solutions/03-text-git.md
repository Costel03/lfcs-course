# Solutions 03 — Text processing and Git

Every answer below was run against the lab's setup data.

---

## Part A — Text processing

### 🟢 Warm-up

**3.1**

```bash
awk '$9 == 500' access.log | wc -l          # 10
grep -c '" 500 ' access.log                  # also 10
```

The awk version compares the status *field*. The grep version only works because the
pattern is anchored by the closing quote and a space — `grep -c 500` alone would also
count lines where `500` appears in a byte count.

**3.2**

```bash
grep -Ev '^\s*(#|$)' app.conf
```

**3.3**

```bash
awk -F: '$7 == "/bin/bash" {print $1}' /etc/passwd
grep -E ':/bin/bash$' /etc/passwd | cut -d: -f1
```

**3.4**

```bash
grep -n 'port' app.conf
```

**3.5**

```bash
sed -n '3,5p' app.conf
```

**3.6**

```bash
tr 'a-z' 'A-Z' < app.conf
```

### 🔵 Practical

**3.7**

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -3
```

```text
     15 10.0.0.3
     15 10.0.0.2
     14 10.0.0.7
```

**3.8**

```bash
awk '{print $9}' access.log | sort | uniq -c
awk '{n[$9]++} END {for (s in n) print s, n[s]}' access.log    # awk-only
```

**3.9**

```bash
awk '{sum += $NF} END {print sum}' access.log     # 50500
```

**3.10**

```bash
awk '$9 == 500 {print $1}' access.log | sort -u
```

**3.11**

```bash
grep 'Failed password' auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

The IP is always 4th from the end (`from IP port N ssh2`), whatever comes before it —
counting from the end is more robust than counting from the start here, because
"invalid user X" adds two words to some lines.

**3.12**

```bash
grep -o 'invalid user [^ ]*' auth.log | awk '{print $3}'
sed -nE 's/.*invalid user ([^ ]+).*/\1/p' auth.log
```

**3.13**

```bash
sed -i.bak -E 's/^#\s*debug = true/debug = true/' app.conf
```

**3.14**

```bash
sed -i '/^\[database\]/,/^\[/ s/^port = .*/port = 5433/' app.conf
```

The address `/^\[database\]/,/^\[/` limits the substitution to lines from `[database]`
up to the next section header (or end of file). A plain `s/port = 5432/…/` would also
work here, but only because the values differ — the range is what you'd use when both
sections have the same value.

**3.15**

```bash
sed -i '/^\[main\]/a timeout = 30' app.conf
```

**3.16**

```bash
awk -F: '$3 >= 1000 {printf "%-12s %6s  %s\n", $1, $3, $6}' /etc/passwd
awk -F: '$3 >= 1000 {print $1":"$3":"$6}' /etc/passwd | column -t -s:
```

`nobody` has UID 65534 and shows up too; add `&& $3 < 65534` to exclude it.

**3.17**

```bash
cp app.conf original.conf
cp app.conf app2.conf && sed -i 's/info/warn/' app2.conf
diff -u original.conf app2.conf > change.patch
cp original.conf patched.conf
patch patched.conf < change.patch
diff app2.conf patched.conf          # no output
```

**3.18**

```bash
sudo touch /etc/lab03-resolv.conf
echo 'nameserver 9.9.9.9' | sudo tee -a /etc/lab03-resolv.conf > /dev/null
```

`> /dev/null` stops `tee` echoing the line back to your terminal.

### 🔴 Challenge

**3.19**

```bash
sed -E -i 's/^#?\s*debug\s*=.*/debug = true/' app.conf
grep -q '^debug = true' app.conf || sed -i '/^\[main\]/a debug = true' app.conf
```

The `sed` handles "commented" and "set to another value"; the `grep … ||` handles
"missing". Tested from all three starting states, three runs each: always exactly one
line. This shape — *fix if present, add if absent* — is what configuration management
tools do for you (Ansible's `lineinfile`), and you'll want it in every exam task that
says "ensure".

**3.20**

```bash
awk '{split($4, t, ":"); print t[2] ":" t[3]}' access.log | sort | uniq -c | awk '$1 > 1' | wc -l
```

`$4` is `[03/Oct/2026:10:41:00`; splitting on `:` gives hour in `t[2]` and minute in `t[3]`.
Answer: `40`.

**3.21**

```bash
awk '{n[$9]++; s[$9] += $NF} END {for (k in n) print k, s[k]/n[k]}' access.log | sort
```

```text
200 500
404 500
500 550
```

**3.22**

```bash
head -1 win.sh | cat -A     # #!/bin/bash^M$
sed -i 's/\r$//' win.sh     # or: dos2unix win.sh
./win.sh                    # hi
```

The kernel reads the shebang line literally, so it looks for an interpreter called
`/bin/bash\r`, which doesn't exist. `cat -A` shows the carriage return as `^M`.

---

## Part B — Git

### 🟢 Warm-up

**3.23**

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

**3.24**

```bash
mkdir -p ~/lab03/configs && cd ~/lab03/configs
cp ../app.conf .
echo '*.bak' > .gitignore
git init
git add .gitignore app.conf
git commit -m "Add app config"
```

### 🔵 Practical

**3.25**

```bash
# on ubuntu2
sudo mkdir -p /srv/git && sudo chown vagrant: /srv/git
git init --bare /srv/git/lab03.git
exit

# on ubuntu1
cd ~/lab03/configs
git remote add origin ubuntu2:/srv/git/lab03.git
git push -u origin main
```

`ubuntu2:/path` is SSH syntax — Git runs over the SSH keys you set up in the lab
environment.

**3.26**

```bash
git clone ubuntu2:/srv/git/lab03.git ~/lab03/clone
cd ~/lab03/clone
sed -i 's/log_level = info/log_level = debug/' app.conf
git commit -am "Raise log level to debug"
git push
cd ~/lab03/configs && git pull
```

`commit -a` stages every *tracked* file that changed — new files still need `git add`.

**3.27**

```bash
git switch -c tuning
sed -i 's/port = 8080/port = 9090/' app.conf
git commit -am "Move main port to 9090"
git switch main
sed -i 's/port = 8080/port = 8081/' app.conf
git commit -am "Move main port to 8081"
git merge tuning                   # CONFLICT (content): Merge conflict in app.conf
vim app.conf                       # keep "port = 9090", delete the three marker lines
git add app.conf
git commit -m "Merge tuning: keep port 9090"
```

**3.28**

```bash
git revert -m 1 --no-edit HEAD
```

A merge commit has two parents; `-m 1` says "reverse the changes relative to parent 1"
(the `main` side). The result is a new commit that restores `8081`, which is safe to push
because no history is rewritten.

### 🔴 Challenge

**3.29**

```bash
git add app.conf .gitignore
git restore --staged .gitignore
git commit -m "Only app.conf"
git show --stat HEAD
git diff                          # .gitignore changes still present
```

**3.30**

```bash
git tag -a v1.0 -m "First release"
git push origin v1.0              # or: git push --tags
git diff --name-only "$(git rev-list --max-parents=0 HEAD)" v1.0
```

`git rev-list --max-parents=0 HEAD` finds the root commit. Tags are not pushed by a plain
`git push` — forgetting this is a common reason a "released" tag is missing on the server.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | `grep` | `awk` always uses ERE; `grep` is BRE unless `-E` |
| 2 | Any character where each `.` is — e.g. `10a0b0c1` | `.` is a wildcard; `-F` makes it literal |
| 3 | `grep -Ev '^\s*(#\|$)' /etc/ssh/sshd_config` | |
| 4 | Suppresses automatic printing; without it every line prints, and 5–10 print twice | `p` prints in addition to the default output |
| 5 | First replaces the first match per line; `g` replaces all matches per line | |
| 6 | `sed -i.bak 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config` | |
| 7 | `$3` is the UID; it lists regular users (UID ≥ 1000) | `-F:` splits on colons |
| 8 | `uniq` only collapses *adjacent* duplicates | Sorting groups them |
| 9 | The three IPs with the most requests, with counts | extract → group → count → order → top |
| 10 | The `>` redirect is performed by the user's shell, before `sudo` runs | `echo … \| sudo tee /etc/resolv.conf` |
| 11 | Windows CRLF line endings; the interpreter name ends in `\r` | `sed -i 's/\r$//'` or `dos2unix` |
| 12 | `"hi" and "bye"` (greedy); use `"[^"]*"` | `.*` takes as much as possible |
| 13 | edit → `git add` → `git commit` → `git push` | |
| 14 | Unstaged changes vs changes already staged for the next commit | |
| 15 | `git revert` | `reset` rewrites history others have; `revert` adds a new commit |
| 16 | A repository with no working tree; pushes into a checked-out branch are refused | Nothing on the server to get out of sync |
| 17 | Edit the file to the final content and remove markers → `git add` → `git commit` | |
| 18 | **No** | It remains in history; rotate the password |
