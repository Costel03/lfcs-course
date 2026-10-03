# Solutions 07 — Software and packages

---

## 🟢 Warm-up

**7.1**

```bash
sudo apt update
apt list --upgradable 2>/dev/null | tail -n +2 | wc -l
```

`tail -n +2` drops the `Listing…` header; `2>/dev/null` hides apt's "unstable CLI" warning.

**7.2**

```bash
dpkg -S /usr/bin/ls /etc/ssh/sshd_config
```

On merged-`/usr` systems `dpkg -S /bin/ls` may say *no path found* — dpkg records the path
the package shipped, so ask with the `/usr/bin` path.

**7.3**

```bash
dpkg -L openssh-server | wc -l
```

The count includes directories, not only files.

**7.4**

```bash
apt policy curl
```

**7.5**

```bash
apt search 'network scan'
apt show nmap
dpkg -l nmap        # "no packages found" or a line starting "un"
```

---

## 🔵 Practical

**7.6**

```bash
sudo apt install -y vsftpd
echo '# lab07 was here' | sudo tee -a /etc/vsftpd.conf
sudo apt remove -y vsftpd
dpkg -l vsftpd                     # rc
ls -l /etc/vsftpd.conf             # still there
sudo apt install -y vsftpd
tail -1 /etc/vsftpd.conf           # your line survived
sudo apt purge -y vsftpd
ls /etc/vsftpd.conf                # No such file or directory
```

dpkg treats files listed as *conffiles* specially: it never overwrites your edits on
reinstall or upgrade (it asks, or keeps yours non-interactively), and only `purge` deletes
them.

**7.7**

```bash
sudo apt install -y apt-file && sudo apt-file update
apt-file search bin/dig
sudo apt install -y bind9-dnsutils         # "dnsutils" also works: a transitional package
dig +short ubuntu.com
```

**7.8**

```bash
sudo apt-mark hold curl
sudo apt upgrade                  # "The following packages have been kept back: curl" (if an update exists)
apt-mark showhold
sudo apt-mark unhold curl
```

**7.9**

```bash
apt policy curl
#   Installed: 8.5.0-2ubuntu10.4
#   Candidate: 8.5.0-2ubuntu10.4
#   Version table:
#  *** 8.5.0-2ubuntu10.4 500   …noble-updates…
#      8.5.0-2ubuntu10   500   …noble…
sudo apt install --allow-downgrades curl=8.5.0-2ubuntu10 libcurl4t64=8.5.0-2ubuntu10
sudo apt install curl                         # back to the candidate
```

Your version numbers will differ — copy them from `apt policy`. Downgrading one library
often requires downgrading its siblings from the same source package at the same time,
which is why `libcurl4t64` appears; apt tells you which.

**7.10**

```bash
sudo apt install -y tree
echo x | sudo tee -a /usr/share/man/man1/tree.1.gz > /dev/null
dpkg -V tree               # ??5??????   /usr/share/man/man1/tree.1.gz
sudo apt reinstall -y tree
dpkg -V tree               # nothing
```

`dpkg -V` compares files against the MD5 sums recorded at install time in
`/var/lib/dpkg/info/tree.md5sums`. Notice it's not a security tool against a determined
attacker — someone with root can rewrite that file too — but it's excellent at catching
accidental edits and broken files.

**7.11**

```bash
sudo install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $VERSION_CODENAME stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
apt policy docker-ce
```

Sourcing `/etc/os-release` gives `$VERSION_CODENAME` (`noble`, `jammy`), so the same
commands work on any Ubuntu release.

**7.12**

```bash
zgrep -h -B1 -A3 'Commandline:.*nginx' /var/log/apt/history.log*
```

Each block in `history.log` has `Start-Date`, `Commandline`, `Install/Upgrade/Remove` and
`End-Date`. `zgrep` reads the rotated, compressed logs as well.

**7.13**

```bash
sudo apt install -y dpkg-dev
sudo mkdir -p /srv/repo && sudo chown "$USER" /srv/repo && cd /srv/repo
apt download sl figlet
dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz
echo 'deb [trusted=yes] file:/srv/repo ./' | sudo tee /etc/apt/sources.list.d/local.list
sudo apt update
apt policy sl
sudo apt install -y sl && /usr/games/sl
```

The same `.deb` is also in the Ubuntu repositories, so `apt policy` shows both sources at
the same version. Whenever you add packages, rerun `dpkg-scanpackages` and `apt update`.

**7.14**

```bash
sudo tee /etc/apt/preferences.d/lab07 > /dev/null <<'EOF'
Package: apache2
Pin: release a=*
Pin-Priority: -1
EOF
apt policy apache2              # Candidate: (none)
sudo apt install apache2        # Package 'apache2' has no installation candidate
```

**7.15** (rocky1)

```bash
rpm -qf /usr/sbin/sshd               # openssh-server-…
sudo dnf install -y tree
sudo dnf history                     # newest transaction first
sudo dnf history undo last -y
rpm -q tree                          # package tree is not installed
```

**7.16** (rocky1)

```bash
sudo dnf install -y python3-dnf-plugin-versionlock
sudo dnf versionlock add curl
sudo dnf versionlock list
```

---

## 🔴 Challenge

**7.17**

```bash
sudo apt install -y build-essential
cd ~/lab07
curl -fsSLO https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz
tar xzf hello-2.12.1.tar.gz && cd hello-2.12.1
./configure --prefix=/opt/hello
make -j"$(nproc)"
sudo make install
echo 'export PATH="$PATH:/opt/hello/bin"' | sudo tee /etc/profile.d/hello.sh
bash -lc hello                      # Hello, world!
dpkg -S /opt/hello                  # no path found matching pattern
```

Its own prefix means everything — binary, man page, translations — lives under
`/opt/hello`, so `rm -rf /opt/hello /etc/profile.d/hello.sh` removes it completely. With
`--prefix=/usr/local` it would be mixed in with every other hand-installed tool.

GNU publishes a `.sig` file next to each tarball; `gpg --verify hello-2.12.1.tar.gz.sig`
checks it once you've imported the GNU keyring. For anything going to production, verify.

**7.18**

```bash
mkdir -p ~/lab07/labtool/DEBIAN ~/lab07/labtool/usr/local/bin
cat > ~/lab07/labtool/DEBIAN/control <<'EOF'
Package: labtool
Version: 1.0
Architecture: all
Maintainer: Lab <lab@example.com>
Description: Lab 07 example package
EOF
printf '#!/bin/sh\necho "labtool 1.0"\n' > ~/lab07/labtool/usr/local/bin/labtool
chmod 755 ~/lab07/labtool/usr/local/bin/labtool

cd ~/lab07 && dpkg-deb --build --root-owner-group labtool
sudo apt install -y ./labtool.deb
labtool
dpkg -S /usr/local/bin/labtool

cp labtool.deb /srv/repo/ && cd /srv/repo
dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz
sudo apt update && apt policy labtool
sudo apt purge -y labtool
```

`--root-owner-group` makes the files inside the package owned by root even though you
built it as a normal user. Real Debian packages shouldn't install into `/usr/local` (that
directory is reserved for the administrator), but dpkg allows it, and it's the honest
place for an in-house tool.

**7.19**

```bash
cp -r ~/lab07/labtool ~/lab07/brokentool
sed -i 's/^Package: labtool/Package: brokentool/' ~/lab07/brokentool/DEBIAN/control
mv ~/lab07/brokentool/usr/local/bin/labtool ~/lab07/brokentool/usr/local/bin/brokentool
printf '#!/bin/sh\nexit 1\n' > ~/lab07/brokentool/DEBIAN/postinst
chmod 755 ~/lab07/brokentool/DEBIAN/postinst
cd ~/lab07 && dpkg-deb --build --root-owner-group brokentool

sudo apt install -y ./brokentool.deb   # post-installation script … returned error exit status 1
dpkg -l brokentool                     # iF  — installed, half-configured
sudo apt install -y tree               # apt now complains about the broken package
```

Repair — either make the maintainer script succeed and finish configuring:

```bash
sudo sed -i 's/exit 1/exit 0/' /var/lib/dpkg/info/brokentool.postinst
sudo dpkg --configure -a
sudo apt purge -y brokentool
```

or force the removal:

```bash
sudo dpkg --remove --force-remove-reinstreq brokentool
```

Installed copies of maintainer scripts live in `/var/lib/dpkg/info/<pkg>.*`. Editing them
is a legitimate repair when a package's own script is what's broken — that's the real-
world version of this task, usually after an interrupted upgrade.

**7.20**

```bash
./answer      # error while loading shared libraries: libanswer.so: cannot open shared object file
echo /usr/local/lib/lab07 | sudo tee /etc/ld.so.conf.d/lab07.conf
sudo ldconfig
ldd ./answer | grep answer
./answer      # 42
```

`-L` told the **linker** where to find the library at build time. At run time the
**dynamic loader** looks in its cache (`/etc/ld.so.cache`, built by `ldconfig` from
`/etc/ld.so.conf.d/*`) and a few default directories. `/usr/local/lib/lab07` was in
neither. `LD_LIBRARY_PATH` would work for one shell — and is ignored for SUID programs and
easy to forget — hence the system-wide fix.

**7.21**

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
#   APT::Periodic::Update-Package-Lists "1";
#   APT::Periodic::Unattended-Upgrade "1";
systemctl list-timers apt-daily-upgrade.timer
sudo tail -20 /var/log/unattended-upgrades/unattended-upgrades.log
sudo unattended-upgrade --dry-run --debug 2>&1 | tail -20
```

**7.22**

```bash
apt-mark showmanual
dpkg-query -W -f='${Installed-Size}\t${Package}\n' | sort -n | tail -10
```

`Installed-Size` is in KiB. `apt-mark showmanual` is also the list you'd use to rebuild a
similar machine: everything else is pulled in as a dependency.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | `update` refreshes the package index; `upgrade` installs newer versions | Upgrade without update uses stale information |
| 2 | Configuration files (conffiles) | `purge` removes them too |
| 3 | Removed, configuration files remain | Clean with `apt purge` |
| 4 | `dpkg -S /usr/bin/curl`; `rpm -qf /usr/bin/curl` | |
| 5 | `apt-file search bin/dig`; `dnf provides /usr/bin/dig` (or `*/dig`) | |
| 6 | `apt-mark hold postgresql-16` | (or a pin) |
| 7 | It trusted a key for **every** repository | Per-repository keyrings with `signed-by=` |
| 8 | Only that key may sign that repository's metadata | Limits the damage of a compromised third-party key |
| 9 | Installed vs candidate version, every available version and its repository and priority | |
| 10 | No — `c` is a conffile you probably edited. A `5` on a binary in `/usr/bin` or `/usr/sbin` would | |
| 11 | `apt`'s output and behaviour may change between versions; `apt-get` is a stable interface | apt warns about this itself |
| 12 | Never install that package (from that source) | |
| 13 | Keeps it out of package-manager territory (`/usr`) and first on `PATH`; you lose updates, security fixes and clean removal | |
| 14 | The loader cache doesn't include that directory: add it to `/etc/ld.so.conf.d/` and run `ldconfig` | `-L` only affects build time |
| 15 | Find what holds it (`lsof`) — usually unattended-upgrades — and wait; never delete the lock file | Deleting it risks corrupting the dpkg database |
| 16 | `sudo dnf history undo last` | |
