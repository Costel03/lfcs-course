# Appendix A — Bash scripting

The LFCS doesn't list scripting as an objective, but every domain rewards it: a five-line
loop beats twenty manual commands, under time pressure especially. And in the job,
scripts are how you stop doing things twice. This appendix builds on chapter 01's shell
mechanics; reread that first if expansion order isn't second nature.

## Objectives

After this appendix you can:

- Write scripts that fail loudly and clean up after themselves.
- Manipulate strings with parameter expansion instead of calling external tools.
- Use arrays and associative arrays.
- Write correct tests and loops, including reading files safely line by line.
- Structure scripts with functions, and parse options with `getopts`.
- Make scripts safe to run twice, safe to run concurrently, and safe to dry-run.
- Debug with `bash -x` and `shellcheck`.

---

## 1. Script skeleton

```bash
#!/usr/bin/env bash
#
# backup-etc — archive /etc to a target directory.
set -euo pipefail

readonly PROG=${0##*/}

usage() {
  echo "Usage: $PROG [-n] -d DEST" >&2
  exit 2
}

die() { echo "$PROG: $*" >&2; exit 1; }

main() {
  local dest="" dry_run=false
  while getopts ':nd:h' opt; do
    case $opt in
      d) dest=$OPTARG ;;
      n) dry_run=true ;;
      h) usage ;;
      :) die "option -$OPTARG needs an argument" ;;
      *) usage ;;
    esac
  done
  shift $((OPTIND - 1))
  [[ -n $dest ]] || usage
  [[ -d $dest ]] || die "no such directory: $dest"

  local file="$dest/etc-$(date +%F).tar.gz"
  if $dry_run; then
    echo "would create $file"
  else
    tar -czf "$file" -C / etc
    echo "created $file"
  fi
}

main "$@"
```

| Element | Why |
|---|---|
| `#!/usr/bin/env bash` | runs bash from `PATH`; `#!/bin/bash` is fine too. Not `#!/bin/sh` if you use bash features |
| `set -e` | exit when an unchecked command fails |
| `set -u` | error on unset variables — catches typos like `$dset` |
| `set -o pipefail` | a pipeline fails if any stage fails (chapter 01) |
| `main "$@"` at the end | the whole file is parsed before anything runs |
| messages to `>&2` | stdout stays clean for output another program might consume |
| exit `2` for usage errors | convention: 0 success, 1 failure, 2 misuse |

### What `set -e` does *not* catch

```bash
set -e
false || true          # no exit: the failure is "handled"
if false; then :; fi   # no exit: tested by if
cmd && other           # no exit if cmd fails: part of an && list
f() { false; echo "still here"; }
f || echo handled      # inside f, set -e is SUSPENDED because f is called in a test
local x=$(false)       # no exit: `local` itself succeeded
```

Treat `set -e` as a safety net with holes, not as error handling. Check what matters
explicitly. For the last case, declare and assign separately: `local x; x=$(cmd)`.

---

## 2. Variables and parameter expansion

String work without spawning `sed`, `cut`, `basename`:

```bash
path=/var/log/nginx/access.log.1

${path##*/}          # access.log.1         — strip longest prefix up to /   (basename)
${path%/*}           # /var/log/nginx       — strip shortest suffix from /   (dirname)
${path#*.}           # log.1                — after the first dot
${path##*.}          # 1                    — after the last dot (extension)
${path%.*}           # /var/log/nginx/access.log
${path/log/LOG}      # /var/LOG/nginx/access.log.1   — first match
${path//log/LOG}     # /var/LOG/nginx/access.LOG.1   — all matches
${#path}             # 27                   — length
${path:0:4}          # /var                 — substring: offset, length
${path^^}            # upper case            ${path,,}  lower case
```

Memory aid: `#` is on the left of `$` on a keyboard → removes from the left; `%` on the
right → removes from the right. Doubled means "longest match".

Defaults and checks:

```bash
${PORT:-8080}        # value, or 8080 if unset or empty
${PORT:=8080}        # same, and assign it
${CONFIG:?not set}   # exit with an error if unset or empty
${DEBUG:+-x}         # "-x" if DEBUG is set, else nothing
```

Integers:

```bash
count=$((count + 1))
(( count++ ))
(( total += size ))
echo $(( 7 / 2 ))    # 3 — integer division only; use awk or bc for decimals
```

---

## 3. Arrays

```bash
servers=(web01 web02 db01)
servers+=(cache01)
echo "${servers[0]}"            # web01
echo "${#servers[@]}"           # 4
for s in "${servers[@]}"; do echo "$s"; done
echo "${servers[@]:1:2}"        # web02 db01 — slice

mapfile -t users < <(cut -d: -f1 /etc/passwd)   # file lines → array
files=(/etc/*.conf)                              # glob → array (with nullglob for safety)
```

`"${arr[@]}"` — quoted, with `@` — expands to each element as a separate word, spaces
preserved. It's the array version of `"$@"`.

### Associative arrays

```bash
declare -A port=([http]=80 [https]=443 [ssh]=22)
port[dns]=53
echo "${port[https]}"
for svc in "${!port[@]}"; do echo "$svc=${port[$svc]}"; done   # ! gives the keys
[[ -v port[ftp] ]] || echo "no ftp"

declare -A count
while read -r shell; do (( count[$shell]++ )); done < <(cut -d: -f7 /etc/passwd)
```

---

## 4. Tests

Use `[[ … ]]` in bash; `[ … ]` is the POSIX command and trips over empty variables.

```bash
[[ -f $file ]]       # regular file          [[ -d $dir ]]   directory
[[ -e $p ]]          # exists                [[ -L $p ]]     symlink
[[ -r / -w / -x $f ]]  # readable / writable / executable by me
[[ -s $f ]]          # exists and not empty
[[ $a -nt $b ]]      # a newer than b

[[ -z $s ]]          # empty                 [[ -n $s ]]     not empty
[[ $s == "exact" ]]
[[ $s == *.log ]]    # glob pattern — right side UNquoted
[[ $s =~ ^[0-9]+$ ]] # regex — right side unquoted; captures in BASH_REMATCH
[[ $a != "$b" ]]

(( n > 10 ))         # arithmetic comparison — clearer than [[ $n -gt 10 ]]
(( n % 2 == 0 ))
```

Combine with `&&`, `||`, `!` inside `[[ ]]`.

### if and case

```bash
if [[ $EUID -ne 0 ]]; then
  echo "run as root" >&2; exit 1
elif ! command -v rsync >/dev/null; then
  echo "rsync missing" >&2; exit 1
fi

case $1 in
  start|up)   start_it ;;
  stop)       stop_it ;;
  *.tar.gz)   extract "$1" ;;
  ''|-h|--help) usage ;;
  *)          echo "unknown: $1" >&2; exit 2 ;;
esac
```

`if` tests a command's **exit status**, so `if grep -q root /etc/passwd; then` needs no
brackets at all.

---

## 5. Loops

```bash
for host in web01 web02; do ssh "$host" uptime; done
for f in /var/log/*.log; do gzip -- "$f"; done
for ((i = 1; i <= 5; i++)); do echo "$i"; done
for i in {1..5}; do …; done

while (( attempts < 5 )); do …; (( attempts++ )); done
until ping -c1 -W1 db01 >/dev/null; do sleep 2; done
```

### Reading a file line by line — the right way

```bash
while IFS= read -r line; do
  printf '%s\n' "$line"
done < input.txt
```

- `IFS=` keeps leading and trailing whitespace.
- `-r` keeps backslashes literal.
- Redirecting into the loop (not `cat file |`) keeps variables set inside it (chapter 01).

Splitting fields as you read:

```bash
while IFS=: read -r user _ uid _ _ home shell; do
  (( uid >= 1000 )) && echo "$user $home"
done < /etc/passwd

while IFS=, read -r name group; do …; done < users.csv
```

`_` is a conventional throwaway variable. If the last line has no trailing newline,
`read` returns false and drops it; `|| [[ -n $line ]]` after `read` rescues it.

---

## 6. Functions

```bash
log() { printf '%s %s\n' "$(date '+%F %T')" "$*" >&2; }

is_active() {
  local svc=$1
  systemctl is-active --quiet "$svc"
}

user_home() {
  local user=$1 entry
  entry=$(getent passwd "$user") || return 1
  echo "${entry}" | cut -d: -f6
}

if is_active nginx; then log "nginx up"; fi
home=$(user_home alice) || log "no such user"
```

| Rule | Why |
|---|---|
| `local` for every variable | otherwise they leak into the caller |
| `return` a status, `echo` data | `$(f)` captures output; `if f` tests status |
| arguments as `$1`, `"$@"` | same as a script |

---

## 7. Options and arguments

```bash
$0  script name     $1…$9 ${10}  positional     $#  count
"$@" all, separately    $?  last exit status    $$  this PID    $!  last background PID
shift               # drop $1, renumber the rest
```

`getopts` parses short options (section 1 shows a full example). In the option string
`':nd:h'`: a letter is a flag, a letter followed by `:` takes an argument, and a leading `:`
lets you handle errors yourself (`:` for a missing argument, `?` for an unknown option).

Long options (`--dest`) need a manual `while (( $# )); do case $1 in …` loop with `shift`;
`getopts` doesn't do them.

---

## 8. Input and output

```bash
read -rp "Hostname: " host
read -rsp "Password: " pass; echo        # -s: don't echo
printf '%-15s %5d\n' "$name" "$count"   # formatted; prefer printf to echo for data

cat > /etc/motd <<EOF                    # here-document: variables expanded
Welcome to $(hostname)
EOF
cat > script.sh <<'EOF'                  # quoted delimiter: nothing expanded
echo "$HOME stays literal"
EOF
grep -q x <<< "$var"                     # here-string
```

---

## 9. Robustness

### Temporary files and cleanup

```bash
tmp=$(mktemp -d)
trap 'rm -rf "$tmp"' EXIT          # runs however the script ends: success, error, Ctrl-C
```

Never use fixed names like `/tmp/myscript.tmp` — predictable names in `/tmp` are a classic
security hole (another user can pre-create them as a symlink).

### Errors

```bash
trap 'echo "error on line $LINENO: $BASH_COMMAND" >&2' ERR
```

### One instance at a time

```bash
exec 9>/run/lock/backup.lock
flock -n 9 || { echo "already running" >&2; exit 1; }
```

The lock is released automatically when the script exits and file descriptor 9 closes —
even after `kill -9`. Far safer than a PID file.

### Idempotency and dry runs

A script is **idempotent** if running it twice leaves the same result as running it once:

```bash
id deploy &>/dev/null || useradd -m deploy
grep -qx 'vm.swappiness = 10' /etc/sysctl.d/90-swap.conf 2>/dev/null \
  || echo 'vm.swappiness = 10' > /etc/sysctl.d/90-swap.conf
mkdir -p /srv/app
```

Check, then act. That's what configuration-management tools do — and what the exam's
"ensure" tasks mean.

A `run` wrapper gives every script a dry-run mode for free:

```bash
DRY_RUN=${DRY_RUN:-false}
run() { if $DRY_RUN; then echo "+ $*"; else "$@"; fi; }
run useradd -m deploy
```

---

## 10. Debugging

```bash
bash -n script.sh               # syntax check only
bash -x script.sh               # trace every command after expansion
PS4='+ ${BASH_SOURCE##*/}:${LINENO}: ' bash -x script.sh    # trace with line numbers
set -x … set +x                 # trace just one section
shellcheck script.sh            # static analysis: quoting bugs, $(ls) loops, and more
```

`shellcheck` (`sudo apt install shellcheck`) finds most of chapter 01's mistakes before you
run anything. Each warning has a code (`SC2086`) with an explanation — `shellcheck -W` or
the code number in its output.

---

## 11. Gotchas

**Unquoted variables** — word splitting and globbing (chapter 01). Quote everything.

**`for f in $(ls)`** — use a glob.

**`cat file | while read`** — variables vanish in the subshell; redirect instead.

**`set -e` holes** — functions called from `if`, `&&` chains, `local x=$(…)`.

**Windows line endings** — `bad interpreter: /bin/bash^M` (chapter 03).

**`[ $var == x ]` with empty `$var`** — `[: ==: unary operator expected`. Use `[[ ]]`.

**Integers only** — `$(( 7 / 2 ))` is 3.

**`cd` inside a script** doesn't change the caller's directory. And `cd dir; rm -rf *` is a
disaster if `cd` fails — use `cd dir || exit 1`, or `set -e`.

---

## 12. Commands introduced

| Item | Purpose |
|---|---|
| `set -euo pipefail` | strict mode |
| `${var#}`, `${var%}`, `${var/}`, `${var:-}`, `${var:?}` | parameter expansion |
| arrays, `declare -A`, `mapfile` | lists and maps |
| `[[ ]]`, `(( ))` | tests |
| `while IFS= read -r` | line-by-line input |
| `getopts`, `shift` | options |
| `trap … EXIT/ERR`, `mktemp`, `flock` | cleanup, errors, locking |
| `bash -n`, `bash -x`, `shellcheck` | debugging |
| `man bash` (`/^PARAMETERS`, `/Parameter Expansion`), `help getopts`, `help read` | offline references |

---

➡ **Next:** [lab](../labs/A-scripting.md)
