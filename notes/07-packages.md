# Chapter 07 — Software and packages

Installing software is one command. Knowing where it came from, which version you have,
what files it put where, how to stop it upgrading, how to add a repository safely and how
to undo all of it — that is the administrator's job.

LFCS: *Operations Deployment* — search for, install, validate and maintain software
packages or repositories.

## Objectives

After this chapter you can:

- Install, upgrade, remove and purge packages, and pin a version so it doesn't change.
- Find which package owns a file, and which package provides a command you don't have.
- List the files a package installed, and verify they haven't been modified.
- Add a third-party repository with a properly scoped signing key.
- Build a small local repository and install from it.
- Read the package history and undo a change.
- Build software from source when there is no package, without polluting the system.
- Do all of the above on both the Debian and the RedHat family.

---

## 1. How package management fits together

```text
repository (HTTP)  ──apt update──▶  local index  ──apt install──▶  dpkg  ──▶  files on disk
  signed metadata                    /var/lib/apt/lists               /var/lib/dpkg (database)
```

- A **package** is an archive of files plus metadata: version, dependencies, and scripts
  that run on install and removal.
- A **repository** is a web directory of packages plus signed index files.
- The **high-level tool** (`apt`, `dnf`) resolves dependencies and downloads.
- The **low-level tool** (`dpkg`, `rpm`) installs one file and records it in the database.

Use the high-level tool for anything involving repositories. Use the low-level tool to
*query* the database.

---

## 2. apt — the Debian/Ubuntu high-level tool

```bash
sudo apt update                    # refresh the index — do this first, always
apt list --upgradable
sudo apt upgrade                   # upgrade, never removing packages
sudo apt full-upgrade              # upgrade, removing packages if needed (kernel/major changes)
sudo apt install nginx
sudo apt install nginx=1.24.0-2ubuntu7    # a specific version
sudo apt install ./tool_1.0_amd64.deb     # a local file, WITH dependency resolution
sudo apt reinstall nginx
sudo apt remove nginx              # remove program files, KEEP configuration
sudo apt purge nginx               # remove program AND configuration
sudo apt autoremove                # remove dependencies nobody needs any more
apt search 'web server'
apt show nginx                     # description, dependencies, size
apt policy nginx                   # installed version, candidate, and from which repo
apt list --installed | grep nginx
apt-cache depends nginx            # what it needs
apt-cache rdepends nginx           # what needs it
apt download nginx                 # fetch the .deb without installing
```

`apt` is meant for people; its output format may change. In scripts use `apt-get` and
`apt-cache`, which are stable: `sudo DEBIAN_FRONTEND=noninteractive apt-get install -y pkg`.

### Hold — stop a package from changing

```bash
sudo apt-mark hold nginx
apt-mark showhold
sudo apt-mark unhold nginx
```

### History

```bash
less /var/log/apt/history.log      # every install/upgrade/remove, with the command line
zgrep -h 'Commandline' /var/log/apt/history.log*
less /var/log/dpkg.log             # every individual package state change
```

apt has no "undo"; you read the history and reverse it by hand.

---

## 3. dpkg — querying the database

```bash
dpkg -l                       # every package with its state
dpkg -l 'nginx*'
dpkg -L nginx-core            # files this package installed
dpkg -S /usr/sbin/nginx       # which package owns this file
dpkg -s nginx                 # status and metadata
dpkg -I file.deb              # inspect a .deb before installing
dpkg -c file.deb              # list its contents
dpkg -V nginx-core            # verify installed files against the package checksums
sudo dpkg-reconfigure tzdata  # re-run a package's configuration questions
dpkg-query -W -f='${Package} ${Version}\n' | sort   # custom format
```

The two letters at the start of `dpkg -l` lines:

| Code | Meaning |
|---|---|
| `ii` | installed, fine |
| `rc` | **removed, config files remain** — what `apt remove` leaves behind |
| `iU` / `iF` | unpacked but not configured / failed configuration — fix with `sudo dpkg --configure -a` |
| `hi` | held, installed |

Clean up `rc` leftovers: `dpkg -l | awk '/^rc/ {print $2}' | xargs -r sudo apt purge -y`.

### Which package provides a command I don't have?

```bash
apt-file search bin/dig         # sudo apt install apt-file && sudo apt-file update
```

Ubuntu's `command-not-found` often prints the answer itself when you mistype.

---

## 4. Repositories

### Where they are defined

Older style, one line per repository:

```text
# /etc/apt/sources.list or /etc/apt/sources.list.d/*.list
deb http://archive.ubuntu.com/ubuntu noble main restricted universe
```

Newer **deb822** style (Ubuntu 24.04 uses this for its own repositories):

```text
# /etc/apt/sources.list.d/ubuntu.sources
Types: deb
URIs: http://archive.ubuntu.com/ubuntu
Suites: noble noble-updates
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

| Term | Meaning |
|---|---|
| suite | release codename (`noble`, `jammy`) and channels (`-updates`, `-security`) |
| component | `main` (supported), `universe` (community), `restricted`, `multiverse` |

### Adding a third-party repository, properly

Every repository's metadata is signed. apt refuses unsigned or wrongly signed metadata.
The modern rule: **each repository's key is trusted only for that repository**.

```bash
sudo install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://example.com/repo/key.asc | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
echo "deb [signed-by=/etc/apt/keyrings/example.gpg] https://example.com/repo stable main" \
  | sudo tee /etc/apt/sources.list.d/example.list
sudo apt update
```

`apt-key add` is deprecated because it trusted a key for **every** repository — a key from
any vendor could sign packages that replace core system ones.

On Ubuntu, `sudo add-apt-repository ppa:owner/name` adds a Launchpad PPA in one step.

### Pinning — choosing between repositories

When the same package exists in two repositories, priorities decide. Files in
`/etc/apt/preferences.d/`:

```text
Package: nginx
Pin: origin nginx.org
Pin-Priority: 900
```

| Priority | Effect |
|---|---|
| > 1000 | install even if it means a downgrade |
| 990 | the target release, when `APT::Default-Release` is set |
| 500 | the default for every repository |
| 100 | the version currently installed |
| < 0 | never install this |

`apt policy nginx` shows the priorities in action.

### A local repository

Useful for an air-gapped machine, or for distributing your own packages:

```bash
sudo apt install dpkg-dev
sudo mkdir -p /srv/repo && cd /srv/repo
sudo apt download tree htop                 # or copy your own .deb files here
dpkg-scanpackages . /dev/null | gzip -9c | sudo tee Packages.gz > /dev/null
echo 'deb [trusted=yes] file:/srv/repo ./' | sudo tee /etc/apt/sources.list.d/local.list
sudo apt update
apt policy tree
```

`[trusted=yes]` skips signature checking — acceptable for a lab or a repository you fully
control on the same host, never for anything fetched over a network.

---

## 5. Validating packages and files

| Question | Debian | RedHat |
|---|---|---|
| Are installed files unmodified? | `dpkg -V pkg` (or `debsums -s`) | `rpm -V pkg` |
| Is this package file signed and intact? | apt checks repository signatures automatically | `rpm -K file.rpm` |
| Does this download match the published checksum? | `sha256sum -c file.sha256` | same |

`dpkg -V` / `rpm -V` print nothing for unmodified packages. A line means something
differs:

```text
??5??????   c /etc/nginx/nginx.conf
```

`5` = checksum differs; `c` = it's a configuration file — expected if you edited it. A
changed **binary** in `/usr/bin` is a red flag.

---

## 6. Keeping systems updated automatically

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
less /var/log/unattended-upgrades/unattended-upgrades.log
```

By default it installs only security updates. When `apt` says *Could not get lock
/var/lib/dpkg/lock-frontend*, it's usually this job running — wait for it rather than
deleting the lock file.

RedHat equivalent: `dnf-automatic` with `systemctl enable --now dnf-automatic.timer`.

---

## 7. The RedHat family — dnf and rpm

| Task | Debian | RedHat |
|---|---|---|
| Refresh metadata | `apt update` | automatic (`dnf makecache`) |
| Install | `apt install pkg` | `dnf install pkg` |
| Specific version | `apt install pkg=1.2-3` | `dnf install pkg-1.2-3` |
| Local file | `apt install ./x.deb` | `dnf install ./x.rpm` |
| Remove | `apt remove` / `purge` | `dnf remove` (configs saved as `.rpmsave` if modified) |
| Upgrade all | `apt upgrade` | `dnf upgrade` |
| Search | `apt search` | `dnf search` |
| Info | `apt show` | `dnf info` |
| Which package provides a file/command | `apt-file search` | `dnf provides /usr/bin/dig` |
| Files of a package | `dpkg -L` | `rpm -ql` |
| Owner of a file | `dpkg -S` | `rpm -qf` |
| List installed | `dpkg -l` | `rpm -qa` / `dnf list installed` |
| Verify files | `dpkg -V` | `rpm -V` |
| Hold a version | `apt-mark hold` | `dnf versionlock add` (plugin) |
| History / undo | log files, by hand | `dnf history`, `dnf history undo <id>` |
| Repositories | `/etc/apt/sources.list.d/` | `/etc/yum.repos.d/*.repo` |
| Package groups | tasks/metapackages | `dnf group install "Development Tools"` |

A `.repo` file:

```ini
[example]
name=Example repository
baseurl=https://example.com/el9/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://example.com/RPM-GPG-KEY-example
```

`dnf history undo` is a real advantage of the RedHat side: every transaction is recorded
and reversible.

---

## 8. Other formats

- **Snap** (Ubuntu) and **Flatpak** bundle an application with its dependencies and run
  it confined. `snap list`, `snap install`, `snap refresh`. They update themselves.
- **Container images** (chapter 14) are the server-side equivalent of bundling.
- **Language package managers** — `pip`, `npm`, `gem` — install outside the system
  package manager. Never `sudo pip install` into the system Python; use a virtual
  environment (`python3 -m venv`). Ubuntu 24.04 refuses it outright
  (`externally-managed-environment`).

---

## 9. Building from source

When no package exists:

```bash
sudo apt install build-essential          # gcc, make, libc headers
# for a project's dependencies, install the -dev packages it needs, or:
sudo apt build-dep <package>              # needs deb-src lines enabled

tar xzf tool-2.1.tar.gz && cd tool-2.1
./configure --prefix=/usr/local           # check for dependencies, generate Makefiles
make -j"$(nproc)"                         # compile
sudo make install                         # copy into /usr/local
```

Why `--prefix=/usr/local`: it keeps your hand-built software out of the paths the package
manager owns (`/usr`), so the two never overwrite each other. `/usr/local/bin` comes first
in `$PATH`, so your build wins over a packaged version.

The cost: the package manager doesn't know about it — no updates, no clean removal, no
security fixes. Keep the build directory (`make uninstall` often exists), or install into
its own prefix (`--prefix=/opt/tool-2.1`) so removal is `rm -rf`.

### Shared libraries

```bash
ldd /usr/local/bin/tool                    # which libraries it needs, and where they resolve
echo /usr/local/lib | sudo tee /etc/ld.so.conf.d/local.conf
sudo ldconfig                              # rebuild the library cache
```

*error while loading shared libraries: libfoo.so.1* after a source install almost always
means `ldconfig` hasn't been told about the directory the library went into.

---

## 10. Gotchas

**Forgetting `apt update`** — you install an old version, or get *Unable to locate package*.

**`remove` leaves configuration.** Reinstalling later brings back your old config, not the
default one. `purge` when you mean "gone".

**Held packages block upgrades silently.** `apt upgrade` reports them as *kept back*; run
`apt-mark showhold`.

**Mixing distribution releases** — adding repositories for a different Ubuntu release or
from Debian — ends in a dependency mess. Match the codename.

**Deleting the dpkg lock** to "fix" a stuck apt can corrupt the database. Find the process
holding it (`sudo lsof /var/lib/dpkg/lock-frontend`) instead.

**Configuration-file prompts** during upgrades block unattended runs. Non-interactively:
`-o Dpkg::Options::=--force-confold` keeps your version.

---

## 11. Commands introduced

| Command | Purpose |
|---|---|
| `apt`, `apt-get`, `apt-cache`, `apt-mark`, `apt-file` | Debian high-level tools |
| `dpkg`, `dpkg-query`, `dpkg-reconfigure`, `dpkg-scanpackages` | Debian low-level tools |
| `add-apt-repository` | Ubuntu PPAs |
| `dnf`, `rpm` | RedHat equivalents |
| `sha256sum -c` | verify a download |
| `./configure`, `make`, `make install` | build from source |
| `ldd`, `ldconfig` | shared libraries |
| `man apt`, `man sources.list`, `man apt_preferences`, `man dpkg` | offline references |

---

➡ **Next:** [quiz](../quizzes/07-packages.md) → [lab](../labs/07-packages.md)
