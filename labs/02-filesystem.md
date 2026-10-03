# Lab 02 — The file system

28 tasks on **ubuntu1**. Do the quiz first.

**Setup:**

```bash
mkdir -p ~/lab02/{src/{docs,img,logs},dest,restore} && cd ~/lab02
for i in $(seq 1 5); do echo "doc $i" > "src/docs/doc$i.txt"; done
head -c 2M /dev/urandom > src/img/photo.bin
head -c 200M /dev/zero > src/logs/huge.log
touch -d '45 days ago' src/logs/old1.log src/logs/old2.log
touch -d '3 days ago'  src/logs/recent.log
touch 'src/docs/name with spaces.txt'
```

**Cleanup:** `rm -rf ~/lab02`

---

## 🟢 Warm-up

**2.1** Without searching the web, open the documentation that describes the purpose of
`/srv`, `/var/spool` and `/opt`.
*Verify:* you found all three in one manual page.

**2.2** Show the inode number of `src/docs/doc1.txt` and its link count.
*Verify:* `stat -c '%i %h' src/docs/doc1.txt` matches what you found.

**2.3** Identify the type of each of these: `/dev/sda`, `/dev/null`, `/etc/hosts`,
`/bin`, `/run/systemd/journal/socket`.
*Verify:* you named five different types.

**2.4** Show the three timestamps of `src/logs/old1.log`. Which two are 45 days old and
which is not?
*Verify:* `stat` shows mtime/atime in the past and ctime as today.

**2.5** List the five largest files under `~/lab02`, largest first, with human sizes.
*Verify:* `huge.log` is first.

**2.6** Show free space and free inodes for the filesystem holding `/home`.
*Verify:* two commands, one ending in `-h`, one in `-i`.

**2.7** Show the size of each top-level directory under `/var`, sorted.
*Verify:* output is sorted smallest to largest with human units.

---

## 🔵 Practical

**2.8** Create a hard link `docs-hard` to `src/docs/doc1.txt` and a symlink `docs-soft`
to it. Delete `src/docs/doc1.txt`. Which link still works? Explain using inodes.
*Verify:* `cat docs-hard` prints `doc 1`; `cat docs-soft` fails.

**2.9** Find every broken symbolic link under `~/lab02`.
*Verify:* `docs-soft` is listed.

**2.10** Try to hard-link a file from `/tmp` to `~/lab02`. Explain the result (check
`df ~/lab02 /tmp` first).
*Verify:* you can say whether they are on the same filesystem, and why it matters.

**2.11** Create a release layout and switch versions atomically:

```bash
mkdir -p app/releases/{v1,v2}
echo v1 > app/releases/v1/VERSION; echo v2 > app/releases/v2/VERSION
```

Make `app/current` point to `v1` using a **relative** link, then switch it to `v2` in a
single command.
*Verify:* `cat app/current/VERSION` prints `v2`; `readlink app/current` shows a relative path.

**2.12** Find every `.log` file under `src` older than 30 days. Then delete only those —
showing the list first.
*Verify:* `old1.log` and `old2.log` are gone; `recent.log` and `huge.log` remain.

**2.13** Find all files under `/etc` modified in the last 3 days.
*Verify:* the command uses a time test, and you suppressed permission errors.

**2.14** Find files under `src` larger than 1 MB but smaller than 100 MB.
*Verify:* only `photo.bin` matches.

**2.15** Find every file under `/home` not owned by an existing user.
*Verify:* the command runs cleanly; an empty result is a valid answer.

**2.16** Make every `.txt` file under `src/docs` read-only for everyone, in one command,
without breaking on the filename with spaces.
*Verify:* `ls -l src/docs` shows `r--r--r--` on all of them, including the spaced one.

**2.17** Search for `/etc` files containing the word `PermitRootLogin`, printing only
filenames, without searching `/proc` or other filesystems.
*Verify:* `/etc/ssh/sshd_config` is in the output.

**2.18** Archive `src` as `src.tar.gz`, `src.tar.xz` and `src.tar.zst`. Compare the
three sizes and the time each took.
*Verify:* you have a three-row comparison; `huge.log` (all zeros) makes them all small.

**2.19** List the contents of `src.tar.gz` without extracting, then extract only
`doc2.txt` into `restore/`.
*Verify:* `restore/` contains only the path to `doc2.txt`.

**2.20** Mirror `src/` into `dest/` with rsync. Then delete a file from `src`, and make
`dest` match again — doing a dry run first.
*Verify:* the dry run listed `deleting …`; afterwards `diff -r src dest` prints nothing.

**2.21** Demonstrate the rsync trailing-slash rule: run the copy both ways into fresh
directories and show the difference with `tree` or `find`.
*Verify:* one result contains a nested `src/` directory, the other doesn't.

---

## 🔴 Challenge

**2.22 — Disk full, du says no.** Run:

```bash
sudo dd if=/dev/zero of=/var/tmp/ghost.bin bs=1M count=300
sudo bash -c 'exec 3</var/tmp/ghost.bin; sleep 3600' &
sudo rm /var/tmp/ghost.bin
```

Show that `df` still counts the 300 MB while `du` cannot find it. Find the process
holding it and free the space **without killing that process**.
*Verify:* `df` drops by ~300 MB and the `sleep` is still running.

**2.23 — Inode exhaustion.** Create a small filesystem and exhaust its inodes while it
still has free space:

```bash
dd if=/dev/zero of=/tmp/tiny.img bs=1M count=20
mkfs.ext4 -q -N 512 /tmp/tiny.img          # only 512 inodes
sudo mkdir -p /mnt/tiny && sudo mount -o loop /tmp/tiny.img /mnt/tiny
sudo chown "$USER" /mnt/tiny
```

Fill it with empty files until it fails. Show `df -h` and `df -i` side by side.
*Verify:* the error is *No space left on device*, `df -h` shows space free, `df -i` shows 100%.

**2.24 — Hidden under a mount point.** Write a 50 MB file into `/mnt/tiny` *after
unmounting it*, then mount it again. Where did the file go, is the space still used, and
how do you get at it without unmounting?
*Verify:* you can `ls` the hidden file while `/mnt/tiny` stays mounted. (Hint: a bind
mount of `/` elsewhere shows what's underneath.)

**2.25 — Forensics.** A file has been planted with a backdated mtime:

```bash
echo payload > /tmp/innocent.sh && touch -d '2021-03-01' /tmp/innocent.sh
```

Prove from metadata alone that it was not really created in 2021.
*Verify:* you can point to the timestamp that gives it away.

**2.26 — Snapshot backups with hard links.** Make two daily snapshots of `src` where
unchanged files take no extra space:

```bash
rsync -a src/ snap/day1/
# change one file in src
rsync -a --link-dest=../day1 src/ snap/day2/
```

Prove that unchanged files in `day1` and `day2` share an inode while the changed one does not.
*Verify:* `ls -i` shows matching inode numbers for unchanged files; `du -sh snap` is
barely more than one copy.

**2.27 — The dangerous slash.** Create `target/` with three files and a symlink `link`
to it. Predict the effect of `rm -rf link` and `rm -rf link/` and then test **on this
lab directory only**.
*Verify:* you can state which command emptied `target/`.

**2.28 — Restore with ownership.** As root, create a tar archive of a directory owned by
`nobody`. Extract it once as root and once as your user. Compare ownership, and explain
when you'd want `--numeric-owner`.
*Verify:* root's extraction keeps `nobody`; yours makes the files yours.

---

## Self-check

- [ ] I know which directories are mine to edit and which belong to packages
- [ ] I can explain a hard link and a symlink in terms of inodes
- [ ] I know which timestamp cannot be faked
- [ ] I check `df -i` when the disk is "full" but has space
- [ ] I always dry-run `find -delete` and `rsync --delete`
- [ ] I know the rsync trailing-slash rule without looking it up

➡ **Solutions:** [solutions/02-filesystem.md](../solutions/02-filesystem.md)
