# Solutions 02 — The file system

---

## 🟢 Warm-up

**2.1**

```bash
man 7 hier
```

On Ubuntu also `man file-hierarchy`, which describes systemd's view of the same layout.

**2.2**

```bash
ls -i src/docs/doc1.txt
stat -c '%i %h' src/docs/doc1.txt     # inode, hard-link count
```

**2.3**

```bash
ls -ld /dev/sda /dev/null /etc/hosts /bin /run/systemd/journal/socket
```

`b` block device, `c` character device, `-` regular file, `l` symlink (`/bin → usr/bin`),
`s` socket. On some VMs the disk is `/dev/vda` instead of `/dev/sda` — use `lsblk` to see
what yours is called.

**2.4**

```bash
stat src/logs/old1.log
```

`touch -d` set Access and Modify to 45 days ago. **Change** is today, because changing a
timestamp is itself an inode change, and ctime cannot be set by `touch`.

**2.5**

```bash
find ~/lab02 -type f -printf '%s\t%p\n' | sort -rn | head -5 | numfmt --to=iec --field=1
# simpler, slightly less precise:
du -ah ~/lab02 | sort -rh | head -5
```

The `du` version includes directories in the list; the `find` version lists only files.

**2.6**

```bash
df -h /home
df -i /home
```

**2.7**

```bash
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h
```

`sort -h` understands `K`, `M`, `G` suffixes. Plain `sort -n` would put `900K` after `2G`.

---

## 🔵 Practical

**2.8**

```bash
ln src/docs/doc1.txt docs-hard
ln -s src/docs/doc1.txt docs-soft
rm src/docs/doc1.txt
cat docs-hard     # doc 1
cat docs-soft     # No such file or directory
```

The hard link is a second name for the same inode; removing the first name dropped the
link count from 2 to 1 and nothing else happened. The symlink stores the *path*
`src/docs/doc1.txt`, which no longer exists.

**2.9**

```bash
find ~/lab02 -xtype l
```

`-type l` matches every symlink; `-xtype l` matches symlinks whose target, followed, is
still a symlink or nothing — i.e. broken ones.

**2.10**

```bash
df ~/lab02 /tmp
touch /tmp/x && ln /tmp/x ~/lab02/x-hard
```

If `/tmp` is a separate `tmpfs` you get *Invalid cross-device link*. An inode number is
only meaningful inside its own filesystem, so a directory entry on one filesystem cannot
refer to an inode on another. If both happen to be on the root filesystem, the link
works — which is the point of checking `df` first.

**2.11**

```bash
mkdir -p app/releases/{v1,v2}
echo v1 > app/releases/v1/VERSION; echo v2 > app/releases/v2/VERSION
ln -s releases/v1 app/current
ln -sfn releases/v2 app/current
cat app/current/VERSION
```

The target `releases/v1` is relative to the link's directory (`app/`), not to where you
ran the command. `-f` replaces an existing link; `-n` treats an existing link to a
directory as a file — without `-n`, `ln -sf` would create `app/current/v2` *inside* v1.

Strictly, `ln -sfn` unlinks then creates, leaving a split second with no `current`.
Deployment tools make it fully atomic by creating a temporary link and renaming it over
the old one: `ln -s releases/v2 app/current.tmp && mv -T app/current.tmp app/current`.

**2.12**

```bash
find src -name '*.log' -type f -mtime +30            # look first
find src -name '*.log' -type f -mtime +30 -delete    # then delete
```

**2.13**

```bash
sudo find /etc -type f -mtime -3 2>/dev/null
```

`-mtime -3` is "less than 3 days ago".

**2.14**

```bash
find src -type f -size +1M -size -100M
```

`find` rounds sizes up to the unit you give, so `-size -100M` means "99 MB or less" in
whole megabytes. For exact limits use `c` (bytes): `-size -104857600c`.

**2.15**

```bash
sudo find /home -nouser -o -nogroup
```

These appear after deleting a user without `-r`: the UID remains on the files, and any
new user created later with the same UID silently inherits them.

**2.16**

```bash
find src/docs -name '*.txt' -exec chmod a-w {} +
```

`-exec … +` passes names directly as arguments, never through word splitting — spaces
are safe.

**2.17**

```bash
sudo grep -rl PermitRootLogin /etc 2>/dev/null
```

`grep -r` does not follow into `/proc` because you only gave it `/etc`. If you used
`find /` instead, add `-xdev` to stay on one filesystem.

**2.18**

```bash
time tar -czf src.tar.gz src
time tar -cJf src.tar.xz src
time tar --zstd -cf src.tar.zst src
ls -lh src.tar.*
```

The 200 MB of zeros compresses to almost nothing in all three — compressors excel at
repetition. The 2 MB of random data doesn't compress at all. Real-world ranking on
logs: xz smallest and slowest, zstd nearly as small and many times faster, gzip in
between on both.

**2.19**

```bash
tar -tzf src.tar.gz
tar -xzf src.tar.gz -C restore src/docs/doc2.txt
find restore
```

The member name must match exactly as listed, including the leading `src/`.

**2.20**

```bash
rsync -a src/ dest/
rm src/docs/doc3.txt
rsync -avn --delete src/ dest/      # dry run: shows "deleting docs/doc3.txt"
rsync -a --delete src/ dest/
diff -r src dest
```

**2.21**

```bash
mkdir t1 t2
rsync -a src/ t1/
rsync -a src  t2/
find t1 t2 -maxdepth 2 -type d
```

`t1/docs`, but `t2/src/docs`. The trailing slash on the **source** means "the contents
of"; the slash on the destination makes no difference.

---

## 🔴 Challenge

**2.22**

```bash
df -h /var/tmp                     # still includes ~300 MB
sudo du -sh /var/tmp               # doesn't
sudo lsof +L1 | grep ghost         # sleep, PID 4321, FD 3r, (deleted)
sudo truncate -s 0 /proc/4321/fd/3
df -h /var/tmp                     # space returned
```

The directory entry is gone, so `du` cannot see the file; the inode survives because a
process still has it open. Truncating through `/proc/<pid>/fd/<n>` releases the blocks
without disturbing the process. In real life this is a log file someone deleted instead
of rotating — the correct long-term fix is to make the service reopen its logs, usually
via `logrotate`'s `postrotate` or `copytruncate`.

**2.23**

```bash
cd /mnt/tiny
for i in $(seq 1 1000); do touch "f$i" || break; done
df -h /mnt/tiny; df -i /mnt/tiny
```

`touch` fails with *No space left on device* after about 500 files while `df -h` still
shows most of the 20 MB free. The kernel uses the same error for "no blocks" and "no
inodes", so you must check both.

**2.24**

```bash
cd ~ && sudo umount /mnt/tiny
sudo dd if=/dev/zero of=/mnt/tiny/hidden.bin bs=1M count=50   # lands on the root fs
sudo mount -o loop /tmp/tiny.img /mnt/tiny
ls /mnt/tiny            # hidden.bin not visible
sudo mkdir -p /mnt/rootview
sudo mount --bind / /mnt/rootview
ls -l /mnt/rootview/mnt/tiny/hidden.bin     # there it is
sudo umount /mnt/rootview
```

A bind mount of `/` shows the root filesystem *without* the filesystems mounted on top
of it, revealing what is underneath. This is how you find space that `du` can't account
for after a mount failed at boot and services wrote into the empty mount point.

**2.25**

```bash
stat /tmp/innocent.sh
```

Modify says 2021; **Change** says today. `touch -d` can set mtime and atime, but setting
them is itself a change to the inode, so ctime records the moment of tampering. If the
filesystem records it, **Birth** gives it away too.

**2.26**

```bash
rsync -a src/ snap/day1/
echo changed >> src/docs/doc2.txt
rsync -a --link-dest=../day1 src/ snap/day2/
ls -i snap/day1/docs/doc4.txt snap/day2/docs/doc4.txt    # same inode
ls -i snap/day1/docs/doc2.txt snap/day2/docs/doc2.txt    # different
du -sh snap/day1 snap/day2 snap
```

`--link-dest` is relative to the destination directory. For each unchanged file rsync
creates a hard link to the copy in `day1` instead of copying data — each snapshot looks
complete, but only changed files consume new space. Tools like `rsnapshot` and Time
Machine use exactly this idea.

**2.27**

```bash
mkdir target && touch target/{a,b,c} && ln -s target link
rm -rf link/      # empties target/ — follows the link
ls target         # nothing
touch target/{a,b,c}
rm -rf link       # removes only the symlink
ls target         # a b c
```

The trailing slash forces path resolution through the symlink, so `rm` operates on the
directory it points at. To remove a symlink, never use a trailing slash; `unlink link`
makes the intent unambiguous.

**2.28**

```bash
sudo mkdir /tmp/owned && sudo touch /tmp/owned/f && sudo chown -R nobody: /tmp/owned
sudo tar -czf /tmp/owned.tgz -C /tmp owned
sudo mkdir /tmp/r-root && sudo tar -xzf /tmp/owned.tgz -C /tmp/r-root
mkdir /tmp/r-user && tar -xzf /tmp/owned.tgz -C /tmp/r-user
ls -l /tmp/r-root/owned /tmp/r-user/owned
```

Root's extraction restores `nobody`; yours makes the files yours, because only root may
give files to other users. tar stores owners by **name and number**. On restore it maps
by name — so if `www-data` is UID 33 on one host and 48 on another, files follow the
name. `--numeric-owner` uses the number instead, which you want when restoring a backup
into a chroot or container whose `/etc/passwd` is not the host's.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | A package upgrade overwrites `/usr/lib`; put overrides in `/etc/systemd/system/` (e.g. `systemctl edit nginx`) | Packages own `/usr`, you own `/etc` |
| 2 | **C** | `/tmp` is cleared at boot and often RAM; `/run` is runtime state |
| 3 | **B** | Names live in directory entries |
| 4 | A rename changes one directory entry; between filesystems the data must be copied and the original deleted | Inode numbers are per-filesystem |
| 5 | Inode exhaustion — `df -i` | Same error for no blocks and no inodes |
| 6 | Cross filesystems; link to a directory (also: point to something that doesn't exist) | Hard links are inode references |
| 7 | The directory containing the link: `/srv/app/current/` | Relative targets resolve from the link's location |
| 8 | **ctime** — any change to the inode, including `touch -d`, updates it | Backdating mtime leaves ctime pointing at the real time |
| 9 | `*.log` is unquoted — the shell may expand it first, so `find` searches the wrong name (or errors); and it deletes without a dry run | Quote patterns: `-name '*.log'` |
| 10 | The first deletes **everything** under `/backups` | `find` evaluates left to right; `-delete` is an action |
| 11 | **No** — 7.5 days rounds down to 7, and `+7` means more than 7 | Use `-mmin` for precision |
| 12 | block device, character device, socket, named pipe | |
| 13 | A deleted file still held open; files hidden under a mount point (also: reserved blocks, other directories on the same filesystem) | `lsof +L1`, bind-mount `/` |
| 14 | First: contents of `src` in `dest`. Second: `dest/src/…` | Trailing slash on the source |
| 15 | It deletes the **contents** of `/srv/app/releases/v42` | The slash resolves through the link |
| 16 | `zstd` | Fast, with a ratio close to xz |
| 17 | Archives with relative names can't overwrite system files when extracted elsewhere | Extract with `-C /` when you mean it |
| 18 | Ownership, mode, timestamps, symlinks as links (and hard links) | `-a` = `-dR --preserve=all` |
