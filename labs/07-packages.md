# Lab 07 — Software and packages

22 tasks on **ubuntu1**; 7.15 and 7.16 on **rocky1**. Do the quiz first.

**Setup:** `sudo apt update && mkdir -p ~/lab07 && cd ~/lab07`

**Cleanup:**

```bash
sudo apt purge -y vsftpd nmap tree sl figlet labtool brokentool 2>/dev/null
sudo apt-mark unhold curl
sudo rm -f /etc/apt/sources.list.d/{docker,local}.list /etc/apt/keyrings/docker.gpg \
           /etc/apt/preferences.d/lab07 /etc/ld.so.conf.d/lab07.conf /etc/profile.d/hello.sh
sudo rm -rf /srv/repo /opt/hello /usr/local/lib/lab07 ~/lab07
sudo ldconfig; sudo apt update
```

---

## 🟢 Warm-up

**7.1** Refresh the package index and show how many packages can be upgraded.
*Verify:* a number, from `apt list --upgradable`.

**7.2** Which packages own `/usr/bin/ls` and `/etc/ssh/sshd_config`?
*Verify:* `coreutils` and `openssh-server`.

**7.3** List the files installed by `openssh-server` and count them.
*Verify:* the count came from `dpkg -L … | wc -l`.

**7.4** For `curl`, show the installed version, the candidate version, and the repository it
comes from.
*Verify:* one command answered all three.

**7.5** Search for packages related to network scanning, then show the details of `nmap`
without installing it.
*Verify:* `dpkg -l nmap` reports it is not installed.

---

## 🔵 Practical

**7.6** Install `vsftpd`. Add the line `# lab07 was here` to `/etc/vsftpd.conf`. Remove the
package (not purge) and show its state. Reinstall it and check your line. Then purge it.
*Verify:* after `remove`, `dpkg -l vsftpd` starts with `rc` and the config still exists;
after reinstall your line is still there; after `purge` the file is gone.

**7.7** The `dig` command is missing on a fresh machine. Find which package provides it
using `apt-file`, and install it.
*Verify:* `dig +short ubuntu.com` returns addresses; you can name the package.

**7.8** Hold `curl` at its current version. Show that `apt upgrade` keeps it back, list all
held packages, then release the hold.
*Verify:* `apt-mark showhold` lists `curl`, then nothing.

**7.9** Find a package with more than one version available (try
`apt policy curl openssl libssl3t64 tzdata`). Install the **older** version, then go back
to the newest.
*Verify:* `apt policy <pkg>` shows `Installed:` changing between the two versions.

**7.10** Install `tree`. Append a byte to its man page
(`/usr/share/man/man1/tree.1.gz`), show that package verification detects it, then repair
the package.
*Verify:* `dpkg -V tree` prints a line with `5`, then nothing after the repair.

**7.11** Add Docker's official repository the modern way — key in `/etc/apt/keyrings/`,
scoped with `signed-by`, codename taken from `/etc/os-release`. Do **not** install Docker.
*Key:* `https://download.docker.com/linux/ubuntu/gpg` ·
*Repository:* `https://download.docker.com/linux/ubuntu <codename> stable`
*Verify:* `apt policy docker-ce` shows a candidate from `download.docker.com`.

**7.12** Using apt's logs, find when `nginx` was installed and the exact command used.
*Verify:* you found a `Commandline:` line mentioning nginx, with its date.

**7.13** Build a local repository in `/srv/repo` containing the `.deb` files for `sl` and
`figlet`, add it as a source, and install `sl` from it.
*Verify:* `apt policy sl` lists `file:/srv/repo`; `/usr/games/sl` runs.

**7.14** Make it impossible for `apache2` to be installed on this machine, using pinning.
*Verify:* `sudo apt install apache2` fails with *has no installation candidate*.

**7.15** On **rocky1**: which package owns `/usr/sbin/sshd`? Install `tree`, show the
transaction in `dnf history`, then undo that transaction.
*Verify:* `rpm -q tree` reports it is not installed after the undo.

**7.16** On **rocky1**: lock `curl` at its current version with the versionlock plugin and
list the locks.
*Verify:* `dnf versionlock list` shows curl.

---

## 🔴 Challenge

**7.17 — Build from source.** Download GNU hello
(`https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz`), build it, and install it under
`/opt/hello` so that it can be removed with a single `rm -rf`. Make `hello` available on
every user's `PATH`.
*Verify:* in a new login shell, `hello` prints `Hello, world!`; `dpkg -S /opt/hello` finds
no owning package.

**7.18 — Make your own package.** Build a `.deb` called `labtool` (version `1.0`) that
installs a script `/usr/local/bin/labtool` printing `labtool 1.0`. Install it, show that
dpkg knows which package owns the file, add it to your `/srv/repo`, and remove it.
*Verify:* `dpkg -S /usr/local/bin/labtool` → `labtool`; `apt policy labtool` shows the
local repository.

**7.19 — A package that won't configure.** Build `brokentool` the same way, but with a
`DEBIAN/postinst` script that exits `1`. Install it and observe the state dpkg leaves it
in. Repair the system so `apt` works normally again, and remove the package.
*Verify:* `dpkg -l brokentool` showed a half-configured state (`iF`); at the end `sudo apt
install -y tree` runs without dpkg errors.

**7.20 — Missing library.** Compile a tiny shared library into `/usr/local/lib/lab07/`
and a program that uses it:

```bash
mkdir -p ~/lab07/lib && cd ~/lab07/lib
printf 'int answer(void){return 42;}\n' > answer.c
printf '#include <stdio.h>\nint answer(void);\nint main(void){printf("%%d\\n",answer());}\n' > main.c
gcc -shared -fPIC -o libanswer.so answer.c
sudo mkdir -p /usr/local/lib/lab07 && sudo cp libanswer.so /usr/local/lib/lab07/
gcc -o answer main.c -L/usr/local/lib/lab07 -lanswer
./answer
```

Explain the error, and fix it system-wide without setting `LD_LIBRARY_PATH`.
*Verify:* `./answer` prints `42`; `ldd ./answer` resolves `libanswer.so`.

**7.21 — Automatic security updates.** Confirm unattended upgrades are enabled, find when
they last ran, and do a dry run.
*Verify:* `/etc/apt/apt.conf.d/20auto-upgrades` has both settings at `1`; the dry run
completes.

**7.22 — Inventory.** Produce (a) the list of packages installed *manually* rather than as
dependencies, and (b) the ten largest installed packages by installed size.
*Verify:* (a) includes `nginx`; (b) is sorted with sizes.

---

## Self-check

- [ ] I can install, hold, downgrade, remove and purge, and know what each leaves behind
- [ ] I can map any file to its package and any missing command to a package
- [ ] I can add a repository with a scoped key, and pin or block packages
- [ ] I can verify installed files and recognise a suspicious change
- [ ] I can build from source into its own prefix, and fix library lookup
- [ ] I can do the common tasks on Rocky with `dnf` and `rpm`

➡ **Solutions:** [solutions/07-packages.md](../solutions/07-packages.md)
