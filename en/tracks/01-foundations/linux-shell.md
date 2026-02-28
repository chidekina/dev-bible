# Linux & Shell

> The terminal is the developer's power tool. Proficiency with Linux and shell scripting dramatically increases your effectiveness: automating repetitive tasks, debugging servers, managing deployments, and navigating complex codebases at speed.

---

## 1. What & Why

Linux powers most of the world's servers, cloud infrastructure, CI/CD systems, and containers. MacOS shares most of its POSIX foundation. Understanding the shell means:
- Debugging production issues on a remote server without a GUI
- Writing automation scripts that run in CI pipelines
- Managing file permissions, processes, and network configurations
- Using tools like Docker, SSH, and system logs effectively
- Building muscle memory that makes you dramatically faster

---

## 2. Core Concepts

### Filesystem Hierarchy Standard (FHS)

```
/                   root — the top of the filesystem tree
├── etc/            system configuration files (nginx.conf, passwd, fstab)
├── var/            variable data — content that changes at runtime
│   ├── log/        system and application logs
│   ├── cache/      cached data (apt cache, etc.)
│   └── lib/        persistent application state
├── opt/            optional/third-party software packages
├── home/           user home directories (/home/alice, /home/bob)
├── root/           home directory for the root user
├── usr/            user utilities and applications
│   ├── bin/        most user commands (ls, grep, curl)
│   ├── lib/        shared libraries for /usr/bin
│   └── local/      locally compiled/installed software
├── bin/            → symlink to /usr/bin on modern systems
├── sbin/           → symlink to /usr/sbin — system admin binaries
├── tmp/            temporary files, cleared on reboot
├── proc/           virtual filesystem — kernel exposes process info here
│   ├── 1234/       directory per PID with mem, fd, status, etc.
│   └── cpuinfo     CPU information
├── dev/            device files (block devices, char devices)
│   ├── sda         first SCSI/SATA disk
│   ├── null        black hole — discard all writes, EOF on read
│   └── random      random number source
├── sys/            virtual filesystem — kernel parameters and hardware info
├── run/            runtime data (PID files, sockets) — cleared on reboot
└── mnt/            temporary mount points for external filesystems
```

---

## 3. How It Works

### Essential Commands

**Navigation:**
```bash
pwd                    # print working directory
ls -la                 # list all files (including hidden) with permissions and size
ls -lah                # same, human-readable file sizes
cd /etc/nginx          # change to absolute path
cd ..                  # go up one level
cd -                   # go to previous directory
tree -L 2              # visual tree view, 2 levels deep
tree -L 3 --gitignore  # respects .gitignore
```

**File operations:**
```bash
cp file.txt backup.txt         # copy file
cp -r src/ dst/                # copy directory recursively
mv old.txt new.txt             # move/rename
rm file.txt                    # delete file
rm -rf dir/                    # delete directory recursively (no confirmation)
mkdir -p path/to/nested/dir    # create nested directories
touch file.txt                 # create empty file or update timestamp
ln -s /path/to/target link     # create symbolic link
stat file.txt                  # show file metadata (size, permissions, timestamps)
```

**Viewing files:**
```bash
cat file.txt                   # print entire file
less file.txt                  # paginated viewer (q to quit, / to search)
head -n 20 file.txt            # first 20 lines
tail -n 20 file.txt            # last 20 lines
tail -f /var/log/app.log       # follow — print new lines as they are written
tail -f -n 100 app.log         # follow last 100 lines
wc -l file.txt                 # count lines
wc -c file.txt                 # count bytes
```

**Searching:**
```bash
# find — filesystem search
find . -name "*.log"                    # files matching pattern
find . -name "*.log" -mtime -7          # modified in last 7 days
find . -name "*.ts" -not -path "*/node_modules/*"
find /var/log -size +100M               # files larger than 100MB
find . -type d -name ".git"             # only directories named .git
find . -empty                           # empty files and directories
find . -name "*.sh" -exec chmod +x {} \;  # execute command on each result

# grep — content search
grep -r "TODO" src/                     # search recursively
grep -r "pattern" --include="*.ts"      # only search .ts files
grep -r "pattern" --exclude-dir="node_modules"
grep -n "error" app.log                 # show line numbers
grep -v "DEBUG" app.log                 # lines NOT matching (invert)
grep -i "error" app.log                 # case-insensitive
grep -c "error" app.log                 # count matching lines
grep -l "TODO" src/**/*.ts              # only filenames (not matching lines)
grep -A 3 -B 3 "error" app.log         # 3 lines of context before and after
grep -E "error|warn|fatal" app.log     # extended regex (ERE)
```

---

### Text Processing

```bash
# awk — column-based processing
awk '{print $2}' file.txt              # print second column (whitespace-delimited)
awk -F: '{print $1}' /etc/passwd       # first column, colon-delimited
awk '$3 > 100 {print $1, $3}' data     # conditional: print cols 1 and 3 if col 3 > 100
awk '{sum += $1} END {print sum}' nums # accumulate sum

# sed — stream editor (substitution is most common)
sed 's/old/new/' file.txt              # replace first occurrence per line
sed 's/old/new/g' file.txt             # replace all occurrences (global)
sed 's/old/new/gi' file.txt            # global, case-insensitive
sed -i 's/old/new/g' file.txt          # in-place edit (modifies the file)
sed -i.bak 's/old/new/g' file.txt      # in-place, keep backup as file.txt.bak
sed -n '10,20p' file.txt               # print lines 10-20
sed '/pattern/d' file.txt              # delete lines matching pattern

# cut — extract columns from delimited text
cut -d: -f1 /etc/passwd                # first field, colon delimiter
cut -d',' -f2,4 data.csv               # fields 2 and 4 from CSV
cut -c1-10 file.txt                    # characters 1-10

# sort and uniq
sort file.txt                          # alphabetical sort
sort -n file.txt                       # numeric sort
sort -rn file.txt                      # reverse numeric (largest first)
sort -t: -k3 -n /etc/passwd            # sort by 3rd field, colon-delimited
sort -u file.txt                       # sort and remove duplicates (like sort | uniq)
uniq -c file.txt                       # prefix lines with their count
sort | uniq -c | sort -rn              # frequency analysis (most common lines)

# tr — translate or delete characters
tr 'a-z' 'A-Z'                         # uppercase
tr -d '\r'                             # delete carriage returns (fix Windows line endings)
tr -s ' '                              # squeeze multiple spaces into one
```

---

### Pipes, Redirection, and xargs

```bash
# Pipes — send stdout of one command to stdin of the next
ls -la | grep ".log"
cat app.log | grep "ERROR" | tail -20
ps aux | grep node | awk '{print $2}'  # get PIDs of node processes

# Redirection
command > output.txt                   # stdout to file (overwrite)
command >> output.txt                  # stdout to file (append)
command 2> errors.txt                  # stderr to file
command 2>&1                           # redirect stderr to stdout (combine streams)
command > output.txt 2>&1             # both stdout and stderr to file
command 2>/dev/null                    # discard stderr
command > /dev/null 2>&1              # discard all output
command < input.txt                    # read stdin from file

# Here documents and here strings
cat << 'EOF' > config.txt
server {
  listen 80;
}
EOF

grep pattern <<< "some string to search"

# xargs — build commands from stdin
find . -name "*.log" | xargs rm                    # delete found files
find . -name "*.log" | xargs -I{} cp {} /backup/  # copy each with replacement
echo "one two three" | xargs -n1 echo             # one arg per command
find . -name "*.ts" | xargs grep -l "TODO"        # grep each found file
cat urls.txt | xargs -P4 curl -sO                 # download 4 URLs in parallel
```

---

### File Permissions

Linux permissions use a three-group model: **owner**, **group**, **others**.

```
-rwxr-xr--  1  alice  dev  4096  Jan 1  file.sh
 |||||||||||
 |└────────── permissions (user/group/others)
 └─────────── file type (- = regular file, d = directory, l = symlink)

Permissions breakdown:
  rwx = read(4) + write(2) + execute(1) = 7
  r-x = read(4) + execute(1) = 5
  r-- = read(4) = 4

So: rwxr-xr-- = 754
    rwxr-xr-x = 755  (common for directories and scripts)
    rw-r--r-- = 644  (common for files)
    rw------- = 600  (private key files)
```

```bash
chmod 755 script.sh            # rwxr-xr-x
chmod +x script.sh             # add execute for all
chmod u+x script.sh            # add execute for owner only
chmod go-w file.txt            # remove write from group and others
chmod -R 644 /var/www/         # recursively set permissions

chown alice file.txt           # change owner
chown alice:devteam file.txt   # change owner and group
chown -R alice:web /var/www/   # recursively

# umask — default permissions mask for new files
umask                          # show current umask (e.g., 022)
umask 022                      # new dirs: 755, new files: 644
# umask subtracts from 777 (dirs) or 666 (files)
```

---

### Process Management

```bash
# Viewing processes
ps aux                         # all processes with user, PID, CPU, MEM
ps aux | grep node             # find node processes
ps -ef --forest                # process tree view
top                            # interactive process monitor (q to quit)
htop                           # better interactive monitor (if installed)

# Killing processes
kill PID                       # send SIGTERM (polite request to stop)
kill -9 PID                    # send SIGKILL (force kill, cannot be ignored)
kill -15 PID                   # same as kill (SIGTERM)
pkill -f "node server.js"      # kill by process name or command pattern
killall node                   # kill all processes named "node"

# Background and foreground
command &                      # run in background
nohup command &                # run in background, immune to hangup (persists after logout)
nohup command > output.log 2>&1 &  # with output redirection
Ctrl+C                         # interrupt (SIGINT) foreground process
Ctrl+Z                         # suspend foreground process
bg                             # resume suspended process in background
fg                             # bring background process to foreground
jobs                           # list background jobs in current shell

# systemd service management
systemctl status nginx          # show service status
systemctl start nginx           # start service
systemctl stop nginx            # stop service
systemctl restart nginx         # restart service
systemctl reload nginx          # reload config without restart
systemctl enable nginx          # enable at boot
systemctl disable nginx         # disable at boot
systemctl is-active nginx       # check if running (exit code 0 = active)
journalctl -u nginx             # logs for a service
journalctl -u nginx -f          # follow logs
journalctl -u nginx --since "1 hour ago"
journalctl -n 100               # last 100 lines from all services
```

---

### Networking Commands

```bash
# HTTP requests
curl https://api.example.com/data
curl -X POST -H "Content-Type: application/json" \
  -d '{"name":"Alice"}' https://api.example.com/users
curl -o file.zip https://example.com/file.zip    # download to file
curl -I https://example.com                      # HEAD request (headers only)
curl -L https://example.com                      # follow redirects
curl -v https://example.com 2>&1 | head -50      # verbose (TLS, headers)
curl -s https://api.example.com | jq '.users[]'  # silent, pipe to jq

wget https://example.com/file.zip                # alternative downloader

# Port and socket inspection
ss -tlnp                                         # TCP listening sockets with PID
ss -tulnp                                        # TCP + UDP listening with PID
netstat -tlnp                                    # older alternative to ss
lsof -i :3000                                    # what's using port 3000
lsof -i -P -n | grep LISTEN                      # all listening sockets

# DNS
dig google.com                                   # DNS lookup
dig google.com A                                 # only A records
dig google.com +short                            # just the IP
dig @8.8.8.8 google.com                         # use specific resolver
dig +trace google.com                            # full DNS resolution walk
host google.com                                  # simpler DNS lookup
nslookup google.com                              # older DNS lookup tool

# Network diagnostics
ping -c 4 google.com                             # 4 ICMP pings
traceroute google.com                            # route to destination
mtr google.com                                   # combined ping + traceroute

# Scanning
nmap -p 80,443 example.com                       # check specific ports
nmap -p- localhost                               # all ports on localhost
```

---

### Disk Usage

```bash
df -h                          # disk usage for all filesystems, human-readable
df -h /var                     # disk usage for specific path
du -sh *                       # disk usage of items in current directory
du -sh /var/log/               # total size of a directory
du -sh * | sort -h             # sorted by size (smallest first)
du -sh * | sort -rh            # sorted by size (largest first)
du -sh /var/* | sort -rh | head  # top 10 largest items in /var
lsblk                          # list block devices and partitions
```

---

### Shell Scripting

```bash
#!/bin/bash
# Always start with a shebang

# Strict mode — exit on error, undefined variables, and pipe failures
set -euo pipefail

# Variables
NAME="World"
echo "Hello, $NAME"

# Command substitution
DATE=$(date +%Y-%m-%d)
FILES=$(find . -name "*.log" | wc -l)
echo "Found $FILES log files on $DATE"

# Conditionals
if [ -f "/etc/nginx/nginx.conf" ]; then
  echo "nginx config exists"
elif [ -d "/etc/nginx" ]; then
  echo "nginx directory exists but no config"
else
  echo "nginx not found"
fi

# Comparison operators for strings and numbers
# [ "$a" = "$b" ]   — string equality
# [ "$a" != "$b" ]  — string inequality
# [ -z "$a" ]       — string is empty
# [ -n "$a" ]       — string is non-empty
# [ "$a" -eq "$b" ] — numeric equality
# [ "$a" -lt "$b" ] — numeric less than
# [ "$a" -gt "$b" ] — numeric greater than

# Loops
for f in *.log; do
  echo "Processing $f"
  gzip "$f"
done

for i in {1..10}; do
  echo "Count: $i"
done

while read -r line; do
  echo "Line: $line"
done < input.txt

# Arrays
FILES=("app.js" "server.js" "config.js")
echo "${FILES[0]}"         # first element
echo "${#FILES[@]}"        # array length
for f in "${FILES[@]}"; do
  echo "$f"
done

# Functions
log_info() {
  echo "[INFO] $(date +%H:%M:%S) $*"
}

log_error() {
  echo "[ERROR] $(date +%H:%M:%S) $*" >&2  # stderr
}

backup_dir() {
  local src="$1"                              # local variable
  local dst="$2"
  local timestamp
  timestamp=$(date +%Y%m%d_%H%M%S)

  if [ ! -d "$src" ]; then
    log_error "Source directory $src does not exist"
    return 1                                 # non-zero exit code = failure
  fi

  mkdir -p "$dst"
  cp -r "$src" "$dst/${timestamp}_$(basename "$src")"
  log_info "Backed up $src to $dst"
}

# Exit code checking
backup_dir /var/app /backups
if [ $? -ne 0 ]; then                        # $? = exit code of last command
  echo "Backup failed!"
  exit 1
fi

# Or use || for inline error handling
cp important.txt /backup/ || { echo "Copy failed"; exit 1; }

# String manipulation
FILE="backup_20240115.tar.gz"
echo "${FILE%.tar.gz}"         # remove suffix: backup_20240115
echo "${FILE#backup_}"         # remove prefix: 20240115.tar.gz
echo "${FILE//backup/copy}"    # replace all: copy_20240115.tar.gz
echo "${FILE:7:8}"             # substring: 20240115
echo "${#FILE}"                # length: 24
echo "${FILE^^}"               # uppercase (bash 4+)
```

---

### SSH

```bash
# Key generation
ssh-keygen -t ed25519 -C "alice@example.com"   # generate Ed25519 key (recommended)
ssh-keygen -t rsa -b 4096 -C "alice@example.com" # RSA 4096 (older compatibility)
# Keys created in ~/.ssh/id_ed25519 and ~/.ssh/id_ed25519.pub

# Copy public key to server
ssh-copy-id alice@server.example.com
# or manually:
cat ~/.ssh/id_ed25519.pub | ssh alice@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Basic connection
ssh alice@server.example.com
ssh -p 2222 alice@server.example.com          # non-standard port
ssh -i ~/.ssh/custom_key alice@server.example.com  # specify key

# ~/.ssh/config — aliases and options
# Host myserver
#   HostName server.example.com
#   User alice
#   Port 2222
#   IdentityFile ~/.ssh/id_ed25519
#   ServerAliveInterval 60
#
# After config:
ssh myserver

# SSH tunneling
# Local port forwarding — access remote service locally
ssh -L 5432:localhost:5432 alice@server     # access server's postgres locally
ssh -L 8080:internal-server:80 alice@bastion  # access internal server through bastion

# Remote port forwarding — expose local port on remote
ssh -R 8080:localhost:3000 alice@server    # server's :8080 → your :3000

# Jump hosts (-J replaces ProxyJump)
ssh -J bastion.example.com alice@internal.server.local

# File transfer
scp file.txt alice@server:~/              # copy to remote home
scp -r dir/ alice@server:~/backup/       # copy directory
scp alice@server:~/file.txt .            # copy from remote

# rsync — better for large or repeated transfers
rsync -avz --progress src/ alice@server:~/dst/
rsync -avz --delete --exclude="node_modules" ./app/ alice@server:/opt/app/
# -a = archive (preserve perms, timestamps, symlinks)
# -v = verbose
# -z = compress
# --delete = remove files on dest that don't exist in source
```

---

## 4. Code Examples

### Backup script with timestamp
```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="${1:?Usage: $0 <source_dir> [dest_dir]}"
DEST_DIR="${2:-/backups}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="$(basename "$BACKUP_DIR")_${TIMESTAMP}"
ARCHIVE="${DEST_DIR}/${BACKUP_NAME}.tar.gz"
MAX_BACKUPS=10

log() { echo "[$(date '+%H:%M:%S')] $*"; }
die() { echo "[ERROR] $*" >&2; exit 1; }

[ -d "$BACKUP_DIR" ] || die "Source directory '$BACKUP_DIR' not found"
mkdir -p "$DEST_DIR" || die "Cannot create backup directory '$DEST_DIR'"

log "Backing up $BACKUP_DIR → $ARCHIVE"
tar -czf "$ARCHIVE" -C "$(dirname "$BACKUP_DIR")" "$(basename "$BACKUP_DIR")"
log "Archive size: $(du -sh "$ARCHIVE" | cut -f1)"

# Rotate old backups — keep only MAX_BACKUPS
log "Rotating old backups (keeping $MAX_BACKUPS)"
ls -t "$DEST_DIR"/*.tar.gz 2>/dev/null | tail -n +"$((MAX_BACKUPS + 1))" | xargs -r rm
log "Done. $(ls "$DEST_DIR"/*.tar.gz | wc -l) backups retained."
```

### Monitor a log file and alert on errors
```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="${1:?Usage: $0 <log_file>}"
ALERT_PATTERN="${2:-ERROR}"
ALERT_CMD="${3:-echo}"  # replace with: 'curl -s -X POST https://hooks.slack.com/...'

[ -f "$LOG_FILE" ] || { echo "Log file not found: $LOG_FILE"; exit 1; }

echo "Monitoring $LOG_FILE for '$ALERT_PATTERN' ..."

tail -f -n 0 "$LOG_FILE" | while IFS= read -r line; do
  if echo "$line" | grep -qi "$ALERT_PATTERN"; then
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    MESSAGE="[$TIMESTAMP] ALERT on $LOG_FILE: $line"
    $ALERT_CMD "$MESSAGE"
  fi
done
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **`rm -rf /` (spaces matter!)**: `rm -rf /important-dir` is safe (deletes that directory). `rm -rf / important-dir` (extra space) deletes root first! Always double-check paths. Use `--` to separate options from filenames: `rm -rf -- "$dir"`.

> ⚠️ **Not quoting variables**: Unquoted variables undergo word splitting and glob expansion. `rm $file` fails or does unexpected things if `$file` contains spaces or wildcards. Always quote: `rm "$file"`.
> ```bash
> # Bug — word splits on spaces in filename
> file="my important file.txt"
> rm $file     # tries to delete "my", "important", "file.txt"
> rm "$file"   # correct — treats as single argument
> ```

> ⚠️ **Not using `set -euo pipefail`**: Without `set -e`, scripts continue running after errors. Without `set -u`, unset variables silently expand to empty string (causing `rm -rf "$DIR/"` to become `rm -rf /` if `$DIR` is unset). Without `set -o pipefail`, `false | true` exits 0.

> ⚠️ **Ignoring exit codes**: Every command exits with a code (0 = success, non-zero = failure). In scripts, check exit codes with `$?`, `if command`, or `command || handle_error`.

> ⚠️ **Using `cp -r` when `rsync` is better**: For large directories, `rsync` is faster (only transfers changed files), handles partial transfers, and preserves permissions better. Use rsync for deployments and backups.

> ⚠️ **Hardcoding paths instead of using variables**: Use variables for paths in scripts. If the path changes, you only update one line.

---

## 6. When to Use / Not Use

**Use shell scripts for:**
- Orchestrating other commands (glue code)
- Automation tasks that are fundamentally file and process manipulation
- CI/CD pipeline steps
- Quick one-off data processing

**Do NOT use shell scripts for:**
- Complex logic with data structures, error handling, or business rules (use Python/Node.js)
- Anything that needs to be unit tested (shell test frameworks exist but are painful)
- When you find yourself writing more than ~100 lines of non-trivial logic

**Use `find` when:**
- You need filesystem-based filtering (by type, size, date, permissions)
- Combined with `-exec` or `xargs` for batch operations

**Use `grep` when:**
- Searching file contents
- Filtering command output

**Use `awk` when:**
- Processing columns of structured text output (log lines, CSV, `ps aux` output)

**Use `sed` when:**
- Stream substitution in files or pipelines

---

## 7. Real-World Scenario

### Diagnosing a production server issue

You are paged at 2am: "App is slow, some requests are timing out."

```bash
# 1. Check if the service is running
systemctl status app-server
journalctl -u app-server -n 50

# 2. Check CPU and memory
top
# Press 'c' to see full command, 'M' to sort by memory, 'P' for CPU

# 3. Check disk space (common culprit)
df -h
du -sh /var/log/* | sort -rh | head -10

# 4. Check for large log files eating disk
find /var/log -size +500M -ls

# 5. Check which ports are in use
ss -tlnp | grep :3000

# 6. Check for too many open connections
ss -s  # connection statistics
lsof -i :3000 | wc -l  # how many connections on port 3000

# 7. Check recent application errors
tail -100 /var/log/app/error.log | grep -i "error\|warn\|fatal"

# 8. Check system logs for OOM killer
journalctl -k | grep -i "killed\|oom"
dmesg | grep -i "killed\|oom\|memory"

# 9. Check postgres connections (if applicable)
ss -tlnp | grep :5432
# From inside psql:
# SELECT count(*), state FROM pg_stat_activity GROUP BY state;

# Root cause found: disk 95% full due to unrotated access logs
# Fix:
find /var/log/app -name "access.log.*" -mtime +30 -delete
gzip /var/log/app/access.log
# Then configure logrotate for automatic rotation
```

---

## 8. Interview Questions

**Q1: How do you find all files modified in the last 24 hours?**

A: `find . -mtime -1 -type f`. The `-mtime -1` means "modification time less than 1 day ago". For hours: `find . -mmin -60` (modified in last 60 minutes). To also see the modification times: `find . -mtime -1 -type f -ls`. To find files and do something with them: `find . -mtime -1 -name "*.log" -exec gzip {} \;` or `find . -mtime -1 | xargs ls -la`.

---

**Q2: What does `2>&1` mean?**

A: It redirects file descriptor 2 (stderr) to wherever file descriptor 1 (stdout) is currently pointing. When you write `command > output.txt 2>&1`, stdout is redirected to the file first, then stderr is redirected to the same place as stdout (the file). Order matters: `command 2>&1 > output.txt` would redirect stderr to the terminal (where stdout was before redirection) and stdout to the file. `/dev/null` is a special device that discards everything written to it: `command > /dev/null 2>&1` silences all output.

---

**Q3: Explain file permissions in Linux.**

A: Every file has three permission groups: owner (user), group, and others. Each group has three permission bits: read (r=4), write (w=2), execute (x=1). The numeric permission is the sum: rwx=7, r-x=5, r--=4. `chmod 755` sets owner=rwx(7), group=r-x(5), others=r-x(5). For directories, execute means "can enter and list". For files, execute means "can run as program". `chown user:group file` changes ownership. `umask` subtracts from the default (666 for files, 777 for directories) when creating new files.

---

**Q4: How does SSH key authentication work?**

A: You have a key pair: a private key (stays on your machine, never shared) and a public key (placed in `~/.ssh/authorized_keys` on the server). When you connect, the server sends a challenge encrypted with your public key. Only your private key can decrypt it. Your client signs the challenge with your private key and sends it back. The server verifies the signature using the public key — this proves you have the private key without ever transmitting it. No password is exchanged. The private key is protected by a passphrase (optional but recommended), which is only used to decrypt the key locally.

---

**Q5: What is the difference between `kill` and `kill -9`?**

A: `kill PID` (default signal: SIGTERM=15) sends a polite termination request. The process receives it and can clean up: close files, flush buffers, save state, then exit. `kill -9 PID` (SIGKILL) is sent directly to the kernel — the process never receives it and cannot intercept it. The kernel immediately terminates it. Use SIGTERM first (allow graceful shutdown), wait a few seconds, then SIGKILL only if it does not stop. `kill -9` can leave open files, unclosed database connections, and incomplete transactions.

---

**Q6: How do you run a process in the background so it persists after logout?**

A: Several options: (1) `nohup command &` — runs in background, immune to SIGHUP (hangup signal sent on logout), output goes to `nohup.out`. (2) `nohup command > app.log 2>&1 &` — redirect output. (3) `tmux` or `screen` — terminal multiplexers that persist sessions. (4) `systemd` service — `systemctl start myapp` — best for production services (auto-restart, logging, dependency management). (5) `disown` after starting: `command &; disown` — detach background job from current shell without nohup.

---

**Q7: What is a pipe and how does it work?**

A: A pipe (`|`) connects the stdout of one command to the stdin of the next. `ps aux | grep node | awk '{print $2}'` — `ps aux` writes to the pipe, `grep node` reads from it, filters, and writes to the next pipe, `awk` reads and prints the second column. Pipes are implemented in the kernel as a kernel buffer between processes. Commands in a pipeline run concurrently — the right side starts reading as the left side starts writing. This is why `tail -f log | grep ERROR` works as a live filter.

---

**Q8: How do you check which process is using a specific port?**

A: `ss -tlnp | grep :3000` or `lsof -i :3000`. `ss` (socket statistics) is the modern replacement for `netstat`. The flags `-t` (TCP), `-l` (listening), `-n` (numeric, no hostname resolution), `-p` (show process). On some systems, `-p` requires root privileges to see other users' processes: `sudo ss -tlnp | grep :3000`. You get the PID and process name in the output. Then you can investigate with `ps -p PID -f` or `kill PID`.

---

## 9. Exercises

**Exercise 1 — Backup script:**

Write a bash script `backup.sh <source> [destination]` that:
- Creates a `.tar.gz` archive of the source directory
- Names the archive with a timestamp (`sourcedir_YYYYMMDD_HHMMSS.tar.gz`)
- Rotates old backups (keep only the 5 most recent)
- Prints progress messages with timestamps
- Uses `set -euo pipefail`
- Handles missing source directory gracefully with a clear error message

---

**Exercise 2 — Find and kill all node processes:**

```bash
# Write a one-liner that:
# 1. Finds all PIDs for processes matching "node"
# 2. Excludes the grep process itself
# 3. Sends SIGTERM to each, then waits 3 seconds
# 4. Sends SIGKILL to any still running

# Hint 1: ps aux | grep node | grep -v grep | awk '{print $2}'
# Hint 2: kill $(command) vs xargs kill
# Hint 3: check if process still exists: kill -0 PID 2>/dev/null
```

---

**Exercise 3 — Log monitoring script:**

Write `monitor.sh <log_file> <pattern>` that:
- Tails the log file in real time
- Alerts (prints to stderr with a timestamp) whenever a line matches the pattern
- Counts how many alerts have been triggered
- Prints a summary on Ctrl+C (trap SIGINT)
- Has a rate-limit to avoid more than 1 alert per 5 seconds for the same pattern

---

**Exercise 4 — Set up SSH key auth to a remote server:**

1. Generate an Ed25519 key pair: `ssh-keygen -t ed25519 -C "your@email.com"`
2. Copy the public key to your server: `ssh-copy-id user@server`
3. Add an entry to `~/.ssh/config` with a short alias
4. Verify you can connect without a password
5. (Bonus) Disable password auth on the server: set `PasswordAuthentication no` in `/etc/ssh/sshd_config` and restart sshd

---

## 10. Further Reading

- **"The Linux Command Line" by William Shotts** (free online): https://linuxcommand.org/tlcl.php — the best beginner-to-intermediate Linux book
- **Bash manual**: https://www.gnu.org/software/bash/manual/bash.html — complete reference
- **"Advanced Bash-Scripting Guide"**: https://tldp.org/LDP/abs/html/ — comprehensive scripting guide
- **explainshell.com**: https://explainshell.com — paste any shell command to see what each part does
- **ShellCheck**: https://www.shellcheck.net — static analysis for shell scripts (catches common bugs)
- **`man` pages**: `man bash`, `man find`, `man grep`, `man ssh`, `man curl` — always authoritative
- **tldr pages**: https://tldr.sh — community-maintained simplified man pages with examples
- **"Linux Pocket Guide" by Daniel Barrett** — quick reference for the most common commands
- **SSH mastery by Michael W. Lucas** — definitive book on SSH configuration and use
- **Filesystem Hierarchy Standard**: https://refspecs.linuxfoundation.org/fhs.shtml
