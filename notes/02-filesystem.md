# Chapter 02 — The file system

An administrator spends most of the day reading and changing files, so it pays to know
exactly what a file *is*: a name pointing at an inode, which points at data. That model
explains hard links, the disk-full-but-du-says-no mystery, and why some deletions don't
free space.

## Objectives

After this chapter you can:

- Say where any kind of file belongs in the hierarchy, and where to look for it.
- Identify all seven file types and explain what an inode holds.
- Explain the difference between hard and symbolic links, and when each breaks.
- Read and reason about the three timestamps, including which one cannot be faked.
- Find any file by name, size, age, owner, type or permission, and act on the results.
- Archive, compress and copy trees without losing permissions or ownership.
- Find what is consuming disk space, including inodes.

---

## 1. The hierarchy, with purpose

The layout is standardised by the Filesystem Hierarchy Standard (`man 7 hier`). What
matters is not memorising it but knowing **which directories change, which are yours,
and which survive a reboot**.

| Directory | Holds | Changes? | Yours to edit? |
|---|---|---|---|
| `/usr` | the OS itself: binaries, libraries, docs | only on package install | **no** — packages own it |
| `/usr/local` | software you installed by hand | when you install | yes |
| `/opt` | self-contained third-party packages | rarely | via their installer |
| `/etc` | host configuration — plain text | when you configure | **yes, this is your job** |
| `/var/log` | logs | constantly | read, rotate |
| `/var/lib` | application state: databases, package DBs | constantly | through the application |
| `/var/cache` | regenerable cache | constantly | safe to clear |
| `/var/spool` | queued work: mail, print, cron | constantly | rarely |
| `/var/tmp` | temporary files that **survive reboot** | yes | yes |
| `/tmp` | temporary files, often RAM-backed, cleared at boot | yes | yes |
| `/run` | runtime state since boot: PID files, sockets | yes | no |
| `/srv` | data this machine serves (web roots, FTP) | yes | yes |
| `/home`, `/root` | users' and root's home directories | yes | the users' |
| `/boot` | kernel, initramfs, bootloader config | on kernel update | carefully |
| `/dev` | device files | as hardware appears | no |
| `/proc`, `/sys` | kernel's live view — not on disk | always | some tunables |

On current distributions `/bin`, `/sbin` and `/lib` are symlinks into `/usr` — the "usr
merge". You will still see old scripts reference `/bin/bash`; it works through the link.

### Vendor config versus admin config

A pattern repeated across modern Linux, worth learning once:

| Location | Owned by | Rule |
|---|---|---|
| `/usr/lib/systemd/system/` | the package | never edit — an upgrade overwrites it |
| `/etc/systemd/system/` | you | your files here **override** the vendor's |

The same split exists for `sysctl.d`, `tmpfiles.d`, `modprobe.d` and others. The
principle: packages ship defaults under `/usr`, you override under `/etc`, upgrades never
destroy your changes. Many services also read a `.d/` directory — `/etc/sudoers.d/`,
`/etc/ssh/sshd_config.d/` — so you can add a file instead of editing the main one. Prefer
that: your change is isolated, easy to remove, and easy to manage with automation.

---

## 2. Seven types of file

`ls -l` shows the type as the first character:

| Char | Type | Example |
|---|---|---|
| `-` | regular file | `/etc/hosts` |
| `d` | directory | `/etc` |
| `l` | symbolic link | `/bin → usr/bin` |
| `b` | block device — random access, buffered | `/dev/sda` |
| `c` | character device — a stream | `/dev/tty`, `/dev/null` |
| `p` | named pipe (FIFO) | made with `mkfifo` |
| `s` | socket | `/run/systemd/journal/socket` |

```bash
ls -l /dev/sda /dev/null /run/systemd/journal/socket
file /bin/ls /etc/hosts /dev/sda
stat /etc/hosts
```

`file` inspects content, not the name — it will tell you a `.txt` is really a gzip
archive.

### The useful device files

```bash
cmd > /dev/null                     # discard
head -c 1M /dev/zero > zeros.bin    # a stream of zero bytes
head -c 32 /dev/urandom | base64    # random bytes
```

### Named pipes

```bash
mkfifo /tmp/pipe
cat /tmp/pipe &          # reader waits
echo hello > /tmp/pipe   # writer delivers — the reader prints it
```

A pipe on disk with no data on disk — it connects two processes that were not started
together.

---

## 3. Inodes — what a file really is

A file has two parts:

- An **inode**: a numbered record holding the type, mode, owner, group, size, timestamps,
  link count, and pointers to the data blocks. **No name.**
- One or more **directory entries**: a name mapped to an inode number.

```bash
ls -i /etc/hosts            # inode number first: 1234567 /etc/hosts
stat /etc/hosts
```

```text
  File: /etc/hosts
  Size: 158         Blocks: 8          IO Block: 4096   regular file
Device: fd00h/64768d    Inode: 1234567     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-10-03 09:12:01.000000000 +0300
Modify: 2026-09-28 14:00:12.000000000 +0300
Change: 2026-09-28 14:00:12.000000000 +0300
 Birth: 2026-09-01 10:00:00.000000000 +0300
```

The name lives in the directory, not in the file. That single fact explains:

- **Renaming is instant** at any size — only the directory entry changes.
- **`mv` within one filesystem never copies data.** Across filesystems it must copy then
  delete, because an inode number only means something within its own filesystem.
- **Deleting is unlinking.** `rm` removes a name. The data is freed only when the link
  count reaches zero **and** no process has the file open (chapter 05's deleted-but-open
  file).

### Running out of inodes

Each filesystem has a fixed number of inodes set at format time (ext4) or allocated
dynamically (XFS). Millions of tiny files can exhaust them while plenty of space remains:

```bash
df -h /var        # 40% used
df -i /var        # IUse% 100% — "No space left on device" anyway
```

When you get *No space left on device* and `df -h` shows space, check `df -i`. The usual
culprits are session files, mail queues, or a cache directory that nobody cleans.

---

## 4. Hard links and symbolic links

### Hard links — another name for the same inode

```bash
echo data > original
ln original hardlink
ls -li original hardlink
```

```text
1311 -rw-r--r-- 2 you you 5 … hardlink
1311 -rw-r--r-- 2 you you 5 … original
```

Same inode, link count 2. Neither is "the original" — they are equal names for one file.
Delete either and the other still works.

Limits: a hard link **cannot cross filesystems** (inode numbers are per-filesystem) and
you **cannot hard-link a directory** (it could create loops the kernel cannot resolve).

### Symbolic links — a file containing a path

```bash
ln -s /etc/hosts hosts-link
ls -l hosts-link            # lrwxrwxrwx … hosts-link -> /etc/hosts
readlink hosts-link         # /etc/hosts
readlink -f hosts-link      # fully resolved, through every link
```

A symlink is its own inode whose content is a path. It can cross filesystems and point
at directories — and it can point at nothing:

```bash
ln -s /does/not/exist dangling
cat dangling                # No such file or directory
find . -xtype l             # find broken symlinks
```

### Relative versus absolute targets

```bash
ln -s /opt/app/current/bin/app /usr/local/bin/app     # absolute
ln -s ../lib/libfoo.so.2 libfoo.so                     # relative to the LINK's directory
```

A relative target is resolved from the directory **containing the link**, not from where
you were when you created it. Relative links survive the whole tree being moved or
mounted elsewhere (inside a container, a chroot, a backup); absolute ones do not.

### When to use which

| Need | Use |
|---|---|
| Point to a directory | symlink |
| Point across filesystems | symlink |
| Switch versions atomically (`current -> releases/v42`) | symlink |
| A file that survives deletion of the "original" name | hard link |
| Space-efficient snapshots of mostly-unchanged trees | hard links (`rsync --link-dest`, `cp -al`) |

---

## 5. Timestamps

| Stamp | Updated when | Set it with |
|---|---|---|
| **mtime** — modify | the file's *content* changes | `touch -m`, `touch -d` |
| **ctime** — change | the *inode* changes: content, mode, owner, links, rename | **cannot be set by users** |
| **atime** — access | the file is read (see below) | `touch -a` |
| **btime** — birth | the file is created (if the filesystem records it) | cannot be set |

```bash
touch -d '2020-01-01 12:00' file      # fake mtime and atime
stat file                             # ctime still shows NOW
```

That asymmetry matters in an investigation: an attacker can backdate `mtime` to make a
planted file look old, but the `ctime` records when they did it.

### atime and relatime

Updating atime on every read would turn every read into a write. Linux mounts with
`relatime` by default: atime is updated only if it is older than mtime, or more than a
day old. So atime tells you roughly "read since last modified", not exact access times.

```bash
ls -l      # shows mtime
ls -lc     # shows ctime
ls -lu     # shows atime
```

---

## 6. Finding files

### find — the one to master

`find` walks a tree and evaluates an expression on every entry, left to right.

```bash
find /etc -name '*.conf'                 # by name (quote the glob!)
find / -iname 'readme*'                  # case-insensitive
find /var -type f -size +100M            # files over 100 MB
find /var/log -type f -mtime +30         # modified over 30 days ago
find /tmp -type f -mmin -10              # modified in the last 10 minutes
find /home -user alice -type f           # owned by alice
find / -nouser -o -nogroup               # owner no longer exists
find . -empty                            # empty files and dirs
find /etc -newer /etc/passwd             # newer than a reference file
find . -maxdepth 1 -type d               # don't descend
```

**Quote the pattern.** `find . -name *.conf` lets the shell expand `*.conf` first — if
the current directory contains one match, `find` searches for that exact name only.

#### Time units

`-mtime N` counts **24-hour periods, rounded down**. `-mtime +7` means "more than 7 full
days ago", i.e. at least 8 days. `-mtime 0` means "within the last 24 hours". When
precision matters, use `-mmin`.

#### Combining

```bash
find . \( -name '*.log' -o -name '*.tmp' \) -type f
find . -type f ! -name '*.gz'
find / -path /proc -prune -o -name 'core' -print     # skip a subtree
```

The parentheses must be escaped for the shell. `-prune` stops `find` descending into a
path; the `-o … -print` is needed because pruning returns true.

#### Acting on results

```bash
find /var/log -name '*.log' -mtime +30 -delete
find . -name '*.sh' -exec chmod +x {} +
find . -type f -name '*.conf' -exec grep -l 'listen' {} +
find . -type f -print0 | xargs -0 ls -l
```

`-delete` evaluates in order, so **it must come last**. `find . -delete -name '*.tmp'`
deletes everything, then tests names. Always run the command once without `-delete` and
read the list.

### locate — fast, but stale

```bash
sudo apt install plocate    # dnf install plocate on RedHat
sudo updatedb               # build the index
locate sshd_config
```

`locate` searches a database built by `updatedb` (usually nightly). Instant, but it won't
find files created since the last update — and it filters out what you can't read.

### which, whereis, type

```bash
command -v nginx            # path that would run
whereis nginx               # binary, source and man page locations
```

---

## 7. Disk usage

```bash
df -h                       # space per mounted filesystem
df -hT                      # with filesystem type
df -i                       # inodes
du -sh /var/log             # total for one directory
du -h --max-depth=1 /var | sort -h     # which subdirectory is big
du -xh / --max-depth=1 2>/dev/null | sort -h   # -x: stay on one filesystem
```

`df` asks the filesystem; `du` adds up files it can see. They disagree when:

- a deleted file is still held open (chapter 05: `lsof +L1`),
- files are hidden **under a mount point** — written to `/data` before something was
  mounted on top, so they still use space on the parent filesystem,
- reserved blocks: ext4 keeps 5% for root by default.

`ncdu` is an interactive `du` worth installing.

---

## 8. Archives and compression

### tar

```bash
tar -czf backup.tar.gz /etc          # create, gzip
tar -cJf backup.tar.xz /etc          # create, xz — smaller, slower
tar --zstd -cf backup.tar.zst /etc   # zstd — fast and good
tar -tzf backup.tar.gz | head        # list without extracting
tar -xzf backup.tar.gz -C /restore   # extract into a directory
tar -xzf backup.tar.gz etc/hosts     # extract one file
tar -czf app.tar.gz --exclude='*.log' app/
```

Modern GNU tar detects compression when extracting, so `tar -xf file` works for any of
them.

Things tar does that matter:

- It **strips the leading `/`** when creating (`Removing leading '/' from member names`),
  so extracting never overwrites system files by accident. Extract with `-C /` when you
  genuinely mean to restore in place.
- As root it restores **ownership and permissions** by default; as a normal user files
  become yours. `--numeric-owner` avoids remapping by name when restoring on another host.
- It preserves **symlinks as symlinks** and hard links as hard links.
- ACLs and SELinux labels need `--acls --selinux --xattrs`.

### Compression trade-offs

| Tool | Speed | Ratio | Use for |
|---|---|---|---|
| `gzip` | fast | ok | compatibility — everything reads it |
| `bzip2` | slow | better | legacy; mostly superseded |
| `xz` | slow | best | distribution, long-term archives |
| `zstd` | very fast | very good | backups, logs, anything modern |

```bash
gzip -k big.log        # -k keeps the original
zcat big.log.gz | grep ERROR
zstd -T0 big.log       # all cores
```

### rsync — copying done properly

```bash
rsync -a src/ dest/                 # archive: recursive, perms, times, links, owner
rsync -aHAX src/ dest/              # + hard links, ACLs, xattrs
rsync -avn --delete src/ dest/      # -n: DRY RUN — always first with --delete
rsync -az src/ user@host:/backup/   # over SSH, compressed
```

**The trailing slash changes everything:**

| Command | Result |
|---|---|
| `rsync -a src/ dest/` | contents of `src` go into `dest` |
| `rsync -a src dest/` | creates `dest/src/` |

`--delete` makes `dest` an exact mirror, removing anything not in `src`. A wrong trailing
slash plus `--delete` has wiped many backup directories. Dry run first.

---

## 9. Gotchas

**`rm -rf link/` follows the link.** With a trailing slash, `rm -rf` on a symlink to a
directory deletes the *contents of the target directory*. Without the slash it removes
only the link. Use `rm link` (no `-r`) or `unlink link` to remove a symlink.

**`cp -r` loses metadata.** It copies with your ownership and umask. `cp -a` preserves
mode, ownership, timestamps and links.

**`mv` across filesystems is copy-then-delete.** Interrupt it and you may have two
half-copies. For large moves between filesystems use `rsync` then delete.

**`/tmp` may be RAM and is cleaned.** It is often `tmpfs` and cleared at boot, and
`systemd-tmpfiles` deletes old files in it on a schedule. Never keep anything there you
want tomorrow — use `/var/tmp` or a real path.

**Files hidden under a mount point.** If a filesystem fails to mount at boot, services
write into the empty directory underneath. When it mounts later, that data vanishes from
view but still fills the parent disk.

---

## 10. Commands introduced

| Command | Purpose |
|---|---|
| `stat` | every inode field |
| `file` | identify content |
| `ls -i`, `ls -lc`, `ls -lu` | inode, ctime, atime |
| `ln`, `ln -s`, `readlink -f`, `unlink` | links |
| `touch -d`, `touch -m`, `touch -a` | set timestamps |
| `find`, `locate`, `updatedb` | find files |
| `df -h`, `df -i`, `du -sh`, `ncdu` | space and inode usage |
| `tar`, `gzip`, `xz`, `zstd` | archive and compress |
| `rsync` | copy and synchronise |
| `mkfifo` | named pipe |
| `man 7 hier` | the hierarchy, documented |

---

➡ **Next:** [quiz](../quizzes/02-filesystem.md) → [lab](../labs/02-filesystem.md)
