# Solutions A — Bash scripting

Scripts marked **(tested)** were run against the lab data; the rest depend on systemd,
`getent`, `flock` or `logger` and follow the same patterns. Every script should pass
`shellcheck` — run it.

---

## 🟢 Warm-up

**A.1 — paths.sh** (tested)

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ $# -eq 1 ]] || { echo "Usage: ${0##*/} FILE" >&2; exit 2; }
path=$1
file=${path##*/}

if [[ $path == */* ]]; then dir=${path%/*}; else dir=.; fi
[[ -z $dir ]] && dir=/

if [[ $file == *.* && $file != .* ]]; then
  ext=${file##*.}
  stem=${file%.*}
else
  ext=""
  stem=$file
fi

printf 'dir:  %s\nfile: %s\next:  %s\nstem: %s\n' "$dir" "$file" "$ext" "$stem"
```

The edge cases are the lesson: a bare name has no `/` (directory `.`), `/etc` has an empty
directory part (`/`), and a dotfile like `.bashrc` has no extension.

**A.2 — userinfo.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ $# -eq 1 ]] || { echo "Usage: ${0##*/} USER" >&2; exit 2; }

entry=$(getent passwd "$1") || { echo "no such user: $1" >&2; exit 1; }
IFS=: read -r name _ uid _ _ home shell <<< "$entry"
printf 'user:  %s\nuid:   %s\nhome:  %s\nshell: %s\n' "$name" "$uid" "$home" "$shell"
```

`getent` covers LDAP users too (chapter 04); `grep /etc/passwd` wouldn't.

**A.3 — svc-check.sh**

```bash
#!/usr/bin/env bash
set -uo pipefail

services=(ssh cron systemd-journald nosuch)
failed=0
for svc in "${services[@]}"; do
  if systemctl is-active --quiet "$svc"; then
    state=OK
  else
    state=DOWN
    failed=1
  fi
  printf '%-20s %s\n' "$svc" "$state"
done
exit "$failed"
```

No `set -e` here: a down service is an expected outcome to report, not an error to abort on.

**A.4 — shells.sh** (tested)

```bash
#!/usr/bin/env bash
set -euo pipefail

passwd=${1:-/etc/passwd}
declare -A count=()

while IFS=: read -r _ _ _ _ _ _ shell; do
  [[ -n $shell ]] || continue
  count[$shell]=$(( ${count[$shell]:-0} + 1 ))
done < "$passwd"

for shell in "${!count[@]}"; do
  printf '%7d %s\n' "${count[$shell]}" "$shell"
done | sort -rn
```

`${count[$shell]:-0}` matters under `set -u`: the first time a shell is seen, its entry is
unset. Associative arrays iterate in no particular order, hence the final `sort`.

---

## 🔵 Practical

**A.5 / A.20 — mkusers.sh** (with the dry-run mode A.20 asks for)

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ $# -eq 1 ]] || { echo "Usage: ${0##*/} FILE" >&2; exit 2; }
DRY_RUN=${DRY_RUN:-false}
if [[ $DRY_RUN != true && $EUID -ne 0 ]]; then
  echo "run as root (or DRY_RUN=true)" >&2; exit 1
fi

# act MESSAGE CMD... — run CMD and print MESSAGE, or just show CMD in dry-run mode
act() {
  local msg=$1; shift
  if [[ $DRY_RUN == true ]]; then
    echo "+ $*"
  else
    "$@" && echo "$msg"
  fi
}

while IFS=, read -r user group || [[ -n $user ]]; do
  [[ -z $user || $user == \#* ]] && continue

  if getent group "$group" > /dev/null; then
    echo "group $group exists"
  else
    act "created group $group" groupadd "$group"
  fi

  if id "$user" &> /dev/null; then
    echo "user $user exists"
  else
    act "created user $user" useradd -m -s /bin/bash "$user"
  fi

  if id -nG "$user" 2> /dev/null | tr ' ' '\n' | grep -qx "$group"; then
    echo "$user already in $group"
  else
    act "added $user to $group" usermod -aG "$group" "$user"
  fi
done < "$1"
```

Every action is preceded by a check, so the second run only prints "exists" lines.
`|| [[ -n $user ]]` keeps a final line that lacks a trailing newline — a common CSV
export quirk.

**A.6 — backup.sh** (tested)

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly PROG=${0##*/}

usage() {
  echo "Usage: $PROG -s SRC -d DEST [-k KEEP] [-n]" >&2
  exit 2
}

die() { echo "$PROG: $*" >&2; exit 1; }

run() {
  if $dry_run; then echo "+ $*"; else "$@"; fi
}

src="" dest="" keep=5 dry_run=false
while getopts ':s:d:k:nh' opt; do
  case $opt in
    s) src=$OPTARG ;;
    d) dest=$OPTARG ;;
    k) keep=$OPTARG ;;
    n) dry_run=true ;;
    h) usage ;;
    :) echo "$PROG: option -$OPTARG needs an argument" >&2; usage ;;
    *) echo "$PROG: unknown option -$OPTARG" >&2; usage ;;
  esac
done
shift $((OPTIND - 1))

[[ -n $src && -n $dest ]] || usage
[[ $keep =~ ^[1-9][0-9]*$ ]] || die "-k must be a positive integer"
[[ -d $src ]] || die "source is not a directory: $src"
[[ -d $dest ]] || die "destination is not a directory: $dest"

src=$(realpath -- "$src")          # absolute, no trailing slash
name=${src##*/}
parent=${src%/*}
parent=${parent:-/}                # a source directly under /, e.g. /etc
archive="$dest/$name-$(date +%F_%H%M%S).tar.gz"

if ! run tar -czf "$archive" -C "$parent" "$name"; then
  rm -f -- "$archive"              # never leave a half-written archive behind
  die "tar failed"
fi
$dry_run || echo "created $archive"

# newest first; everything after the first $keep is deleted
mapfile -t old < <(find "$dest" -maxdepth 1 -name "$name-*.tar.gz" -printf '%T@\t%p\n' \
                     | sort -rn | cut -f2- | tail -n +"$((keep + 1))")
for f in "${old[@]}"; do
  run rm -f -- "$f"
done
```

Testing caught a real bug in the first draft: with a relative source like `src`,
`${src%/*}` returned `src` itself, so `tar -C src src` failed — and left a half-written
archive each time, so the rotation never ran. `realpath` first, then splitting, fixes it.
Sorting by modification time with `find -printf` instead of parsing `ls -t` keeps
`shellcheck` quiet and handles odd names.

**A.7 — disk-alert.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

threshold=${1:-80}
[[ $threshold =~ ^[0-9]+$ ]] || { echo "Usage: ${0##*/} [THRESHOLD]" >&2; exit 2; }

over=0
while read -r pcent target; do
  use=${pcent%\%}
  if (( use > threshold )); then
    printf '%-30s %3s%%\n' "$target" "$use"
    over=1
  fi
done < <(df --output=pcent,target -x tmpfs -x devtmpfs -x overlay -x squashfs | tail -n +2)

if (( over )); then exit 1; fi
echo "all ok"
```

Putting `pcent` first means `read` gives the *rest* of the line — the mount point, spaces
included — to `target`. `df --output` can't be combined with `-P`.

**A.8 — tmpwork.sh** (tested)

```bash
#!/usr/bin/env bash
set -euo pipefail

tmp=$(mktemp -d)
trap 'rm -rf "$tmp"' EXIT

echo "$tmp"
date > "$tmp/a"
hostname > "$tmp/b"

case ${1:-} in
  fail)  false ;;          # set -e exits here; the EXIT trap still runs
  sleep) sleep 60 ;;       # interrupt with Ctrl-C
esac
echo "finished normally"
```

Tested four ways — normal exit, `set -e` failure, SIGINT, SIGTERM — and the directory was
gone every time. One surprise from testing: a script started in the background with `&`
from another script **ignores SIGINT** (non-interactive shells start background jobs that
way), so `kill -INT` only took effect once `sleep` finished. Ctrl-C on a foreground script
behaves as you'd expect.

**A.9 — single.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

exec 9> /run/lock/labA-single.lock
if ! flock -n 9; then
  echo "${0##*/}: already running" >&2
  exit 1
fi

echo "running as PID $$"
sleep 30
```

The kernel releases the lock when descriptor 9 closes — at exit, even after `kill -9`.
`/run/lock` is world-writable with the sticky bit on Ubuntu, and cleared at boot.

**A.10 — retry.sh** (tested)

```bash
#!/usr/bin/env bash
set -uo pipefail

# retry N DELAY cmd... — run cmd up to N times, doubling DELAY after each failure.
retry() {
  local tries=$1 delay=$2 attempt=1 status
  shift 2
  while true; do
    "$@" && return 0
    status=$?
    if (( attempt >= tries )); then
      echo "retry: giving up after $attempt attempts (status $status)" >&2
      return "$status"
    fi
    echo "retry: attempt $attempt failed (status $status), waiting ${delay}s" >&2
    sleep "$delay"
    (( attempt++ ))
    (( delay *= 2 ))
  done
}

retry 4 1 curl -sf http://localhost:9999/
echo "failing case returned $?"
retry 4 1 true
echo "true returned $?"
```

```text
retry: attempt 1 failed (status 7), waiting 1s
retry: attempt 2 failed (status 7), waiting 2s
retry: attempt 3 failed (status 7), waiting 4s
retry: giving up after 4 attempts (status 7)
failing case returned 7
true returned 0
```

No `set -e`: the function exists to handle failures. `(( attempt++ ))` would return status
1 when `attempt` is 0 and kill a `set -e` script — another reason to leave it off here.

**A.11 — normalise.sh** (tested)

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() { echo "Usage: ${0##*/} DIR [-n]" >&2; exit 2; }

[[ $# -ge 1 ]] || usage
dir=$1
dry_run=false
[[ ${2:-} == -n ]] && dry_run=true
[[ -d $dir ]] || { echo "not a directory: $dir" >&2; exit 1; }

shopt -s nullglob
for path in "$dir"/*; do
  [[ -f $path ]] || continue
  old=${path##*/}
  new=${old,,}
  new=${new// /_}
  [[ $old == "$new" ]] && continue
  if [[ -e $dir/$new ]]; then
    echo "skip: $old -> $new (target exists)" >&2
    continue
  fi
  if $dry_run; then
    echo "would rename: $old -> $new"
  else
    mv -n -- "$path" "$dir/$new"
    echo "renamed: $old -> $new"
  fi
done
```

`mv -n` is a second guard against overwriting — the `-e` check and the `mv` aren't atomic,
so another process could create the target in between.

**A.12 — log.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

log() {
  local level=$1 prio
  shift
  case $level in
    INFO)  prio=info ;;
    WARN)  prio=warning ;;
    ERROR) prio=err ;;
    *)     prio=notice ;;
  esac
  printf '%s [%s] %s\n' "$(date '+%F %T')" "$level" "$*" >&2
  logger -t labA -p "user.$prio" -- "$*"
}

log INFO "starting"
log WARN "disk at 85%"
log ERROR "backup failed"
```

**A.13 — health.sh** (tested)

```bash
#!/usr/bin/env bash
set -uo pipefail

(( $# > 0 )) || { echo "Usage: ${0##*/} URL..." >&2; exit 2; }

failed=0
for url in "$@"; do
  if out=$(curl -s -o /dev/null -m 5 -w '%{http_code} %{time_total}' "$url"); then
    read -r code secs <<< "$out"
    secs="${secs}s"
    if (( code >= 200 && code < 400 )); then
      status=OK
    else
      status=FAIL; failed=$((failed + 1))
    fi
  else
    code=000 secs=- status=FAIL
    failed=$((failed + 1))
  fi
  printf '%-4s %3s %9s  %s\n' "$status" "$code" "$secs" "$url"
done
exit "$failed"
```

```text
OK   200 0.503806s  https://ubuntu.com
FAIL 000         -  http://localhost:9
```

curl returns non-zero for a connection failure but zero for an HTTP 500 — hence checking
both the exit status and the code.

**A.14 — sysreport.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

# shellcheck source=/dev/null
. /etc/os-release
row() { printf '%-14s %s\n' "$1:" "$2"; }

row "Hostname"     "$(hostname -f 2>/dev/null || hostname)"
row "OS"           "$PRETTY_NAME"
row "Kernel"       "$(uname -r)"
row "Uptime"       "$(uptime -p)"
row "CPUs"         "$(nproc)"
row "Load"         "$(cut -d' ' -f1-3 /proc/loadavg)"
row "Memory"       "$(free -h | awk '/^Mem:/ {print $3 " used, " $7 " available"}')"
row "Root fs"      "$(df -h --output=pcent,size / | awk 'NR==2 {print $1 " of " $2}')"
row "Users"        "$(who | wc -l)"
row "Failed units" "$(systemctl --failed --no-legend | wc -l)"
```

**A.15 — set-kv.sh** (tested)

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ $# -eq 3 ]] || { echo "Usage: ${0##*/} FILE KEY VALUE" >&2; exit 2; }
file=$1 key=$2 value=$3
[[ -f $file ]] || { echo "no such file: $file" >&2; exit 1; }
[[ $key =~ ^[A-Za-z0-9_.-]+$ ]] || { echo "invalid key: $key" >&2; exit 2; }

# escape characters that are special in a sed replacement
esc_value=$(printf '%s' "$value" | sed 's/[&\\|]/\\&/g')

if grep -Eq "^[#;]?[[:space:]]*${key}[[:space:]]*=" "$file"; then
  # replace the FIRST matching line (active or commented), delete any later duplicates
  sed -E -i "0,/^[#;]?[[:space:]]*${key}[[:space:]]*=.*/s||${key} = ${esc_value}|" "$file"
  sed -E -i "0,/^${key} = /!{/^[#;]?[[:space:]]*${key}[[:space:]]*=/d}" "$file"
else
  printf '%s = %s\n' "$key" "$value" >> "$file"
fi
```

Tested three runs from a commented `#debug = true` — one active line — and a value
containing `|`, `&` and `\`, which would otherwise break the `sed` replacement. Two GNU sed
features carry it: `0,/re/` limits the change to the first match, and an empty pattern
(`s||…|`) reuses the last regex. Validating `KEY` keeps it from injecting sed syntax.

---

## 🔴 Challenge

**A.16 — the fixed script** (tested)

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Usage check instead of running with an empty $target.
[[ $# -eq 1 ]] || { echo "Usage: ${0##*/} DIR" >&2; exit 2; }
target=$1
[[ -d $target ]] || { echo "not a directory: $target" >&2; exit 1; }

count=0
total=0

# 2. A glob, not $(ls …): names with spaces stay whole. nullglob: no match → no iterations.
shopt -s nullglob
logs=("$target"/*.log)

for f in "${logs[@]}"; do
  # 3. "$f" quoted — otherwise stat receives a split name.
  size=$(stat -c %s -- "$f")
  # 4. (( )) for numbers; [ $size -gt … ] breaks if $size is empty.
  if (( size > 1000 )); then
    # 5. ((count++)) returns status 1 when count is 0 — under set -e that EXITS the script.
    count=$((count + 1))
  fi
done

# 6. Redirect into the loop instead of piping into it: the pipe ran the loop in a
#    subshell, so $total was always empty afterwards. Also read -r.
if (( ${#logs[@]} > 0 )); then
  while IFS= read -r _; do
    total=$((total + 1))
  done < <(cat -- "${logs[@]}")
fi

echo "big files: $count, lines: $total"

# 7. Never `cd dir; rm -rf *`: if cd fails, rm runs in the CURRENT directory.
#    Use the path directly, and only if it exists.
old="$target/old"
if [[ -d $old ]]; then
  find "$old" -mindepth 1 -delete
fi
```

Tested with a log named `big one.log`, an empty directory, and no arguments.

**A.17 — menu.sh**

```bash
#!/usr/bin/env bash
PS3="Choose (1-4): "
options=("Disk usage" "Top memory" "Failed units" "Quit")
select _ in "${options[@]}"; do
  case $REPLY in
    1) df -h -x tmpfs -x devtmpfs ;;
    2) ps -eo pid,user,%mem,cmd --sort=-%mem | head -6 ;;
    3) systemctl --failed ;;
    4|q|quit) exit 0 ;;
    *) echo "invalid choice: $REPLY" ;;
  esac
done
```

Matching on `$REPLY` (what was typed) rather than the selected item lets `q` work too.

**A.18 — disk-alert as a service and timer**

```bash
sudo install -m 755 ~/labA/disk-alert.sh /usr/local/bin/disk-alert
sudo tee /etc/systemd/system/disk-alert.service > /dev/null <<'EOF'
[Unit]
Description=Alert on full filesystems

[Service]
Type=oneshot
Environment=THRESHOLD=80
ExecStart=/usr/local/bin/disk-alert ${THRESHOLD}
EOF
sudo tee /etc/systemd/system/disk-alert.timer > /dev/null <<'EOF'
[Unit]
Description=Check filesystems every 15 minutes

[Timer]
OnCalendar=*:0/15
Persistent=true

[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now disk-alert.timer

# test the failure path
sudo systemctl edit disk-alert       # [Service]  Environment=THRESHOLD=1
sudo systemctl start disk-alert
systemctl status disk-alert          # failed (exit code 1)
journalctl -u disk-alert -n 10       # the filesystems over 1%
```

The script's exit code becomes the unit's result — `failed` is visible in
`systemctl --failed`, which monitoring can watch. That's the advantage over cron, where a
failing job is silent.

**A.19 — pingsweep.sh**

```bash
#!/usr/bin/env bash
set -uo pipefail

[[ $# -eq 1 && $1 =~ ^([0-9]{1,3}\.){2}[0-9]{1,3}$ ]] || { echo "Usage: ${0##*/} A.B.C" >&2; exit 2; }
prefix=$1
tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT

for i in $(seq 1 20); do
  { ping -c1 -W1 "$prefix.$i" > /dev/null 2>&1 && echo "$prefix.$i" >> "$tmp"; } &
done
wait
sort -t. -k4,4n "$tmp"
```

All twenty pings run at once; `wait` blocks until every background job finishes, so the
whole sweep takes about one timeout. Short appends to a file opened with `>>` don't
interleave.

**A.20** — see A.5: `DRY_RUN=true ./mkusers.sh users.csv` prints `+ groupadd …`,
`+ useradd …` lines and changes nothing. The `act` wrapper is the whole trick: every
state-changing command goes through one function, so one switch controls them all.
