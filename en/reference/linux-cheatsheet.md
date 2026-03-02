# Linux Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## File & Directory Operations

```bash
# Navigation
pwd                        # print working directory
ls -lah                    # long list, all files, human sizes
ls -lt                     # sort by modification time
cd -                       # go to previous directory
pushd /some/dir            # push dir onto stack
popd                       # pop back

# Create / delete
mkdir -p /deep/path/dir    # create with parents
touch file.txt             # create empty or update timestamp
rm file.txt                # remove file
rm -rf dir/                # remove directory recursively (careful!)
rmdir dir/                 # remove empty directory

# Copy / move
cp file.txt dest/
cp -r src/ dest/           # recursive copy
cp -a src/ dest/           # archive mode (preserves permissions, timestamps)
mv old.txt new.txt         # move or rename
mv dir/ /new/location/

# Links
ln -s /original /link      # symbolic link
ln /original /hardlink     # hard link

# Find files
find . -name "*.log"
find . -type f -size +100M
find . -mtime -7           # modified in last 7 days
find . -name "*.js" -not -path "*/node_modules/*"
find . -name "*.tmp" -exec rm {} \;
```

---

## Viewing Files

```bash
cat file.txt               # print whole file
less file.txt              # paginated viewer (q to quit)
head -n 20 file.txt        # first 20 lines
tail -n 50 file.txt        # last 50 lines
tail -f /var/log/app.log   # follow (live stream)
tail -F /var/log/app.log   # follow + reopen if rotated

wc -l file.txt             # count lines
wc -c file.txt             # count bytes

file mystery.bin           # detect file type
xxd file.bin | head        # hex dump
stat file.txt              # detailed file metadata
```

---

## Searching: grep & ripgrep

```bash
grep "pattern" file.txt
grep -r "pattern" .        # recursive
grep -i "pattern" file     # case-insensitive
grep -n "pattern" file     # show line numbers
grep -v "pattern" file     # invert (exclude matches)
grep -c "pattern" file     # count matching lines
grep -l "pattern" *.txt    # only filenames
grep -A 3 -B 3 "pattern" f # 3 lines context around match
grep -E "regex|pattern" f  # extended regex (ERE)
grep -P "\d{4}" f          # Perl-compatible regex

# ripgrep (faster, respects .gitignore)
rg "pattern"
rg -i "pattern" --type ts  # TypeScript files, case-insensitive
rg -l "pattern"            # filenames only
rg -n "pattern"            # with line numbers
```

---

## Text Processing: sed & awk

```bash
# sed — stream editor
sed 's/old/new/' file              # replace first match per line
sed 's/old/new/g' file             # replace all matches
sed 's/old/new/gi' file            # case-insensitive
sed -i 's/old/new/g' file          # in-place edit
sed -i.bak 's/old/new/g' file      # in-place with backup
sed -n '5,10p' file                # print lines 5-10
sed '/pattern/d' file              # delete lines matching pattern
sed 's/^/  /' file                 # indent every line
sed -E 's/(foo|bar)/baz/g' file    # extended regex

# awk — column/field processing
awk '{print $1}' file              # print first field
awk '{print $NF}' file             # print last field
awk -F: '{print $1}' /etc/passwd   # custom delimiter
awk '/pattern/{print}' file        # print matching lines
awk '{sum += $2} END {print sum}'  # sum column 2
awk 'NR==5' file                   # print line 5
awk 'NR>=5 && NR<=10' file         # print lines 5-10
awk '{print NR, $0}' file          # add line numbers
awk '!seen[$0]++'                  # deduplicate lines

# Combining
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -20
```

---

## Permissions

```bash
ls -l                      # show permissions
chmod 755 file             # rwxr-xr-x
chmod 644 file             # rw-r--r--
chmod +x script.sh         # add execute for all
chmod -R 755 dir/          # recursive
chmod u+x,g-w,o-rwx file  # symbolic notation

chown user:group file      # change owner and group
chown -R user:group dir/   # recursive
chgrp developers file      # change group only

# Permission bits
# r=4  w=2  x=1
# 7=rwx  6=rw-  5=r-x  4=r--  0=---
# User Group Other
# e.g. 755 = rwxr-xr-x

# Special bits
chmod u+s file             # SUID — run as owner
chmod g+s dir/             # SGID — inherit group
chmod +t /tmp              # sticky — only owner can delete

# umask (default permission mask)
umask                      # show current (e.g., 022)
umask 027                  # new files: 640, dirs: 750
```

---

## Processes

```bash
# Listing
ps aux                     # all processes, detailed
ps aux | grep nginx        # find specific process
pgrep nginx                # get PID(s) by name
pgrep -l nginx             # with process name

# Monitoring
top                        # interactive process viewer
htop                       # nicer interactive viewer
watch -n 2 'ps aux'        # refresh every 2 seconds

# Signals
kill PID                   # SIGTERM (graceful shutdown)
kill -9 PID                # SIGKILL (force quit)
kill -HUP PID              # SIGHUP (reload config)
killall nginx              # kill by name
pkill -f "node app.js"     # kill by command match

# Background & foreground
command &                  # run in background
Ctrl+Z                     # suspend foreground job
bg                         # resume in background
fg                         # bring to foreground
jobs                       # list background jobs
disown %1                  # detach job from shell

# Priority
nice -n 10 command         # start with lower priority (-20 to 19)
renice -n 5 -p PID         # change priority of running process

# Information
lsof -p PID                # files opened by process
lsof -i :3000              # process on port 3000
strace -p PID              # trace system calls
```

---

## Disk & Memory

```bash
df -h                      # disk usage per filesystem
df -h /                    # specific mount point
du -sh /var/log            # size of directory
du -sh *                   # size of everything in current dir
du -sh * | sort -h         # sorted by size

free -h                    # memory usage
vmstat 1                   # virtual memory stats (every 1s)

lsblk                      # list block devices
fdisk -l                   # list disk partitions (root)
mount | column -t          # list mounted filesystems

# Find large files
find / -type f -size +1G 2>/dev/null
du -a /var | sort -rh | head -20
```

---

## Networking

```bash
# IP and interfaces
ip addr                    # show interfaces and IPs
ip addr show eth0          # specific interface
ip link                    # link layer status
ip route                   # routing table

# DNS
dig example.com            # DNS lookup
dig example.com A          # A record
dig @8.8.8.8 example.com   # use specific DNS server
nslookup example.com       # alternative DNS lookup
host example.com           # simple lookup

# Connectivity
ping -c 4 google.com       # 4 pings
traceroute google.com      # hop-by-hop path
mtr google.com             # combines ping + traceroute

# Port scanning & sockets
ss -tlnp                   # listening TCP sockets
ss -ulnp                   # listening UDP sockets
netstat -tlnp              # older equivalent
lsof -i :8080              # what's on port 8080?

# HTTP
curl -v https://example.com
curl -X POST -H "Content-Type: application/json" \
  -d '{"key":"val"}' https://api.example.com/endpoint
curl -o output.html https://example.com
curl -I https://example.com          # headers only
curl -u user:pass https://example.com
wget https://example.com/file.zip    # download file

# Firewall (ufw)
ufw status
ufw allow 80/tcp
ufw allow from 192.168.1.0/24 to any port 5432
ufw deny 22
ufw enable
```

---

## Systemd

```bash
# Service management
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx          # reload config without restart
systemctl status nginx
systemctl enable nginx          # start on boot
systemctl disable nginx
systemctl is-active nginx
systemctl is-enabled nginx

# System state
systemctl list-units            # all active units
systemctl list-units --failed   # failed units
systemctl daemon-reload         # reload unit files (after editing)
systemctl reboot
systemctl poweroff

# Logs (journald)
journalctl -u nginx             # logs for nginx service
journalctl -u nginx -f          # follow
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -n 50       # last 50 lines
journalctl --since today
journalctl -p err               # only errors
journalctl --disk-usage
journalctl --vacuum-time=7d     # clean logs older than 7 days
```

### Service Unit File

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node App
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node dist/index.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=/opt/myapp/.env

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now myapp
```

---

## Cron

```bash
crontab -e                  # edit user crontab
crontab -l                  # list user crontab
crontab -r                  # remove user crontab
crontab -u www-data -e      # edit another user's crontab

# System-wide
/etc/crontab
/etc/cron.d/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

```
# Cron format: MIN HOUR DOM MON DOW COMMAND
# MIN  0-59
# HOUR 0-23
# DOM  1-31 (day of month)
# MON  1-12
# DOW  0-7  (0 and 7 = Sunday)

0 * * * *        /script.sh    # every hour at :00
0 2 * * *        /backup.sh    # daily at 2am
0 2 * * 0        /weekly.sh    # every Sunday at 2am
0 2 1 * *        /monthly.sh   # 1st of month at 2am
*/15 * * * *     /check.sh     # every 15 minutes
0 9-17 * * 1-5   /job.sh       # weekdays 9am-5pm hourly

# Special strings
@reboot          /startup.sh   # on boot
@daily           /daily.sh     # daily
@weekly          /weekly.sh    # weekly
```

---

## SSH

```bash
ssh user@host
ssh -p 2222 user@host             # custom port
ssh -i ~/.ssh/id_rsa user@host    # specific key
ssh -J jump@bastion user@target   # jump host (ProxyJump)
ssh -L 8080:localhost:80 user@host # local port forward
ssh -R 9090:localhost:3000 user@h  # remote port forward
ssh -N -f user@host -L 5432:db:5432 # background tunnel, no shell

# Key management
ssh-keygen -t ed25519 -C "you@example.com"
ssh-copy-id user@host             # install public key on server
cat ~/.ssh/id_ed25519.pub | ssh user@host 'cat >> ~/.ssh/authorized_keys'

# SSH config (~/.ssh/config)
Host myserver
  HostName 192.168.1.100
  User deploy
  Port 2222
  IdentityFile ~/.ssh/id_ed25519
  ServerAliveInterval 60
```

---

## Environment & Shell

```bash
# Variables
export VAR=value            # set and export
echo $VAR                   # use variable
unset VAR                   # remove
printenv                    # print all env vars
env | grep NODE             # filter env vars

# PATH
export PATH="$HOME/.local/bin:$PATH"

# Redirection
command > file              # stdout to file (overwrite)
command >> file             # stdout to file (append)
command 2> file             # stderr to file
command &> file             # stdout + stderr to file
command 2>&1 | tee file     # stdout + stderr to file and screen
command < file              # stdin from file

# Pipes & substitution
cmd1 | cmd2                 # pipe stdout to next
cmd1 && cmd2                # run cmd2 only if cmd1 succeeds
cmd1 || cmd2                # run cmd2 only if cmd1 fails
cmd1 ; cmd2                 # run sequentially regardless
$(command)                  # command substitution
<(command)                  # process substitution

# Useful one-liners
history | grep ssh          # search history
Ctrl+R                      # reverse search history
!!                          # repeat last command
!$                          # last argument of previous command
!nginx                      # last command starting with nginx
```

---

## Package Management

```bash
# Debian / Ubuntu (apt)
apt update
apt upgrade
apt install nginx
apt remove nginx
apt purge nginx             # remove + config files
apt autoremove              # remove orphaned packages
apt search nginx
apt show nginx
apt list --installed

# RHEL / CentOS (dnf/yum)
dnf install nginx
dnf remove nginx
dnf update
dnf search nginx

# Check what provides a command
apt-file search /usr/bin/dig
which dig && dpkg -S $(which dig)
```
