# Lab 03 — Text processing and Git

30 tasks on **ubuntu1** (Git remote tasks also use **ubuntu2**). Do the quiz first.

**Setup** — builds sample files with known contents, so every answer is checkable:

```bash
mkdir -p ~/lab03 && cd ~/lab03

# A web access log: 100 requests from 7 IPs, with known status codes and sizes
for i in $(seq 1 100); do
  ip="10.0.0.$(( i % 7 + 1 ))"
  if   (( i % 10 == 0 )); then st=500
  elif (( i % 5  == 0 )); then st=404
  else st=200; fi
  printf '%s - - [03/Oct/2026:10:%02d:00 +0000] "GET /page%d HTTP/1.1" %d %d\n' \
         "$ip" $(( i % 60 )) $(( i % 4 )) "$st" $(( i * 10 ))
done > access.log

# An SSH auth log
cat > auth.log <<'EOF'
Oct  3 10:00:01 ubuntu1 sshd[901]: Failed password for invalid user admin from 203.0.113.5 port 50122 ssh2
Oct  3 10:00:04 ubuntu1 sshd[903]: Failed password for root from 203.0.113.5 port 50124 ssh2
Oct  3 10:00:09 ubuntu1 sshd[905]: Accepted publickey for vagrant from 192.168.56.1 port 50210 ssh2
Oct  3 10:01:15 ubuntu1 sshd[910]: Failed password for invalid user test from 198.51.100.7 port 41000 ssh2
Oct  3 10:01:18 ubuntu1 sshd[912]: Failed password for root from 203.0.113.5 port 50130 ssh2
Oct  3 10:02:30 ubuntu1 sshd[920]: Failed password for invalid user oracle from 198.51.100.7 port 41010 ssh2
Oct  3 10:03:00 ubuntu1 sshd[925]: Accepted password for alice from 192.168.56.12 port 51000 ssh2
EOF

# A config file to edit
cat > app.conf <<'EOF'
# Application settings
[main]
port = 8080
#debug = true
log_level = info

[database]
host = db.local
port = 5432
EOF
```

**Cleanup:** `rm -rf ~/lab03` (and `/srv/git/lab03.git` on ubuntu2)

---

## Part A — Text processing

### 🟢 Warm-up

**3.1** Count the requests in `access.log` that returned `500`.
*Verify:* `10`.

**3.2** Show only the active lines of `app.conf` (no comments, no blanks).
*Verify:* 6 lines, none starting with `#`.

**3.3** List every username in `/etc/passwd` whose login shell is `/bin/bash`.
*Verify:* `vagrant` and `root` are in the list.

**3.4** Show the line numbers of every `port` setting in `app.conf`.
*Verify:* two lines, prefixed `3:` and `9:`.

**3.5** Print lines 3–5 of `app.conf` using `sed`.
*Verify:* exactly three lines, starting with `port = 8080`.

**3.6** Convert the contents of `app.conf` to upper case on screen without changing the file.
*Verify:* output is upper case; `head -1 app.conf` is unchanged.

### 🔵 Practical

**3.7** Print the 3 IP addresses with the most requests, with their counts, highest first.
*Verify:* the top two are `10.0.0.2` and `10.0.0.3`, each with `15`.

**3.8** Count requests per HTTP status code.
*Verify:* `80` for 200, `10` for 404, `10` for 500.

**3.9** Total the bytes sent (last field) across all requests.
*Verify:* `50500`.

**3.10** List the IPs that caused at least one `500`, each once.
*Verify:* seven IPs, sorted, no duplicates.

**3.11** From `auth.log`, list each IP with failed password attempts and how many, highest
first.
*Verify:* `203.0.113.5` with `3`, `198.51.100.7` with `2`.

**3.12** From `auth.log`, extract just the usernames of *invalid users* that were tried.
*Verify:* `admin`, `test`, `oracle`.

**3.13** Using `sed` in place with a backup, uncomment `debug = true` in `app.conf`.
*Verify:* `grep '^debug = true' app.conf` matches; `app.conf.bak` exists.

**3.14** Change the `port` in the `[database]` section only to `5433` — leave `[main]`
alone.
*Verify:* `grep -n port app.conf` shows `8080` on line 3 and `5433` in the database section.

**3.15** Insert a line `timeout = 30` directly after `[main]`.
*Verify:* line 3 of the file is `timeout = 30`.

**3.16** Print a table of `/etc/passwd` showing username, UID and home directory for
users with UID ≥ 1000, aligned in columns.
*Verify:* columns line up; `vagrant` appears with UID `1000`.

**3.17** Create a modified copy `app2.conf`, produce a unified diff as `change.patch`, then
apply it to a fresh copy of the original.
*Verify:* after `patch`, `diff app2.conf <patched file>` prints nothing.

**3.18** Append the line `nameserver 9.9.9.9` to a root-owned file
`/etc/lab03-resolv.conf` (create it first with sudo) **without** using `sudo bash -c` or
an editor.
*Verify:* `cat /etc/lab03-resolv.conf` shows the line; the file is owned by root.

### 🔴 Challenge

**3.19** Make the edit in 3.13 idempotent: write one command (or a short `&&`/`||`
combination) that sets `debug = true` active whether the line is commented, set to
`false`, or missing entirely. Run it three times.
*Verify:* exactly one `debug = true` line exists after three runs.

**3.20** Group the requests in `access.log` by *minute* (the `HH:MM` in the timestamp).
How many distinct minutes received more than one request?
*Verify:* `40`.

**3.21** Using only `awk` (no `sort`/`uniq`), print each status code and the average
response size for it.
*Verify:* three lines; the `500` average is `550`.

**3.22** A colleague's script fails with `bad interpreter`. Create the situation with
`printf '#!/bin/bash\r\necho hi\r\n' > win.sh && chmod +x win.sh`, prove the cause with
one command, and fix it.
*Verify:* `./win.sh` prints `hi`.

---

## Part B — Git

### 🟢 Warm-up

**3.23** Configure your Git name and email globally, and set the default branch to `main`.
*Verify:* `git config --global --list` shows all three.

**3.24** In `~/lab03/configs`, create a repository containing `app.conf`, with a
`.gitignore` that excludes `*.bak`. Make the first commit.
*Verify:* `git log --oneline` shows one commit; `git status` mentions no `.bak` file.

### 🔵 Practical

**3.25** On **ubuntu2**, create a bare repository `/srv/git/lab03.git`. Add it as remote
`origin` from ubuntu1 and push `main`.
*Verify:* on ubuntu2, `git --git-dir=/srv/git/lab03.git log --oneline` shows your commit.

**3.26** Clone the remote into `~/lab03/clone`, make a commit there changing
`log_level`, push it, then pull it into `~/lab03/configs`.
*Verify:* `git log --oneline` in both directories shows the same top commit.

**3.27** Create a branch `tuning`, change `port = 8080` to `9090` there, commit. Switch
to `main`, change the same line to `8081`, commit. Merge `tuning` into `main` and resolve
the conflict by keeping `9090`.
*Verify:* `git log --graph --oneline` shows a merge commit; `grep 9090 app.conf` matches;
no `<<<<<<<` markers remain.

**3.28** Revert the merge's effect safely, as you would after it had been pushed.
*Verify:* a new commit exists and `app.conf` shows `8081` again. (Hint: reverting a merge
needs `-m 1`.)

### 🔴 Challenge

**3.29** You staged two files but only want to commit one. Unstage the other without
losing its changes, commit, and show that the second file's edits are still there.
*Verify:* `git show --stat HEAD` lists one file; `git diff` shows the other's changes.

**3.30** Tag the current commit `v1.0` with a message, push the tag, and show the
difference between `v1.0` and the first commit as a list of changed files only.
*Verify:* on ubuntu2, `git --git-dir=/srv/git/lab03.git tag` lists `v1.0`.

---

## Self-check

- [ ] I know which tools use BRE and which use ERE
- [ ] I can show a config file's active lines in one command
- [ ] I can edit a config in place with `sed`, safely and idempotently
- [ ] I can write the extract → sort → count → sort → head pipeline without thinking
- [ ] I can write to a root-owned file with `tee`
- [ ] I can clone, commit, branch, merge, resolve a conflict and push
- [ ] I know when to `revert` instead of `reset`

➡ **Solutions:** [solutions/03-text-git.md](../solutions/03-text-git.md)
