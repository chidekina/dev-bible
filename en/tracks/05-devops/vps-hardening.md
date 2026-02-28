# VPS Hardening

## Overview

A freshly provisioned VPS is a liability. The default configuration prioritizes accessibility over security: the root account accepts password logins, all ports are accessible, and no intrusion detection is running. Within hours of a new VPS going online, it will be scanned by automated bots probing for weak passwords and known exploits.

Hardening is the process of systematically reducing the attack surface of a server. It doesn't make a system invincible — it makes exploitation hard enough that attackers move on to easier targets.

This chapter covers the essential hardening steps for a production Ubuntu 22.04/24.04 VPS: user management, SSH configuration, firewall rules, intrusion prevention, automatic updates, and application-level security for Docker deployments.

---

## Prerequisites

- Root or sudo access to a fresh Ubuntu 22.04/24.04 VPS
- A local SSH key pair (`ssh-keygen -t ed25519`)
- Basic Linux command line knowledge

---

## Core Concepts

### The Attack Surface

Every exposed service, open port, enabled account, and installed package is a potential entry point. Hardening systematically eliminates unnecessary ones:

```
Attack surface reduction:
  Accounts    → disable root login, use key-only SSH
  Network     → firewall blocks all but 22/80/443
  Services    → disable/remove unused daemons
  Software    → automatic security updates
  Monitoring  → detect and block brute-force attempts
  Containers  → run as non-root, read-only FS where possible
```

### Defense in Depth

No single control is sufficient. Layer multiple defenses so that bypassing one layer still leaves others:

```
Layer 1: Firewall (UFW)         — block unwanted traffic
Layer 2: SSH hardening          — key-only, non-root, fail2ban
Layer 3: OS hardening           — minimal packages, auto-updates
Layer 4: Application hardening  — non-root containers, no exposed DB ports
Layer 5: Monitoring             — logs, alerts, anomaly detection
```

---

## Hands-On Examples

### Step 1: Create a Non-Root User

```bash
# Log in as root initially
ssh root@YOUR_VPS_IP

# Create admin user
adduser deploy
usermod -aG sudo deploy

# Set up SSH key for deploy user
mkdir -p /home/deploy/.ssh
chmod 700 /home/deploy/.ssh

# Add your public key (from your local machine: cat ~/.ssh/id_ed25519.pub)
echo "ssh-ed25519 AAAA... your-key-comment" >> /home/deploy/.ssh/authorized_keys

chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh

# Test before closing the root session
# Open new terminal:
ssh deploy@YOUR_VPS_IP
sudo whoami   # should return "root"
```

### Step 2: Harden SSH

`/etc/ssh/sshd_config` (or create `/etc/ssh/sshd_config.d/hardening.conf`):

```bash
# Create override file (modern approach — survives package upgrades)
cat > /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
# Disable root login
PermitRootLogin no

# Key-only authentication — disable passwords
PasswordAuthentication no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
UsePAM yes

# Restrict to specific user(s)
AllowUsers deploy

# Use modern ciphers and algorithms
Protocol 2
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org

# Connection settings
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable unused features
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitEmptyPasswords no
PrintLastLog yes
EOF

# Test the new configuration
sshd -t

# Apply
systemctl restart sshd
```

**Critical:** Before restarting sshd, confirm you can log in with your SSH key from a separate terminal.

### Step 3: Configure UFW Firewall

```bash
# Install UFW (usually pre-installed on Ubuntu)
apt install -y ufw

# Default policies
ufw default deny incoming
ufw default allow outgoing

# Allow SSH (must do this before enabling!)
ufw allow ssh          # or: ufw allow 22/tcp

# Allow web traffic
ufw allow 80/tcp       # HTTP
ufw allow 443/tcp      # HTTPS

# If you need to expose specific internal ports temporarily (e.g., for diagnostics)
# ufw allow from YOUR.IP.ADDRESS to any port 5432

# Enable the firewall
ufw --force enable

# Verify rules
ufw status verbose
```

Example output:
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
```

### Step 4: Install and Configure Fail2Ban

Fail2ban monitors log files and bans IPs that show malicious behavior (too many failed login attempts):

```bash
apt install -y fail2ban

# Create local override (survives updates)
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
# Ban for 1 hour
bantime  = 3600
# Time window to count failures
findtime = 600
# Number of failures before ban
maxretry = 5
# Backend (systemd for Ubuntu 22.04+)
backend = systemd

[sshd]
enabled  = true
port     = ssh
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 86400  # ban SSH brute-force for 24 hours

[nginx-http-auth]
enabled  = true

[nginx-limit-req]
enabled  = true
EOF

systemctl enable fail2ban
systemctl start fail2ban

# Check status
fail2ban-client status
fail2ban-client status sshd
```

Check bans and unban:
```bash
# See current bans
fail2ban-client status sshd

# Unban an IP (if you accidentally locked yourself out)
fail2ban-client set sshd unbanip YOUR.IP.ADDRESS
```

### Step 5: Automatic Security Updates

```bash
apt install -y unattended-upgrades apt-listchanges

# Configure
cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};

// Reboot if needed (e.g., kernel updates)
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";

// Email notification (optional)
// Unattended-Upgrade::Mail "admin@myapp.com";
EOF

cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
EOF

# Test
unattended-upgrade --dry-run
```

### Step 6: Disable Unused Services

```bash
# List running services
systemctl list-units --type=service --state=running

# Common services to disable on a minimal server
systemctl disable --now snapd.service   # Snap daemon (if not using snaps)
systemctl disable --now avahi-daemon    # mDNS (not needed on servers)
systemctl disable --now cups            # Printing (not on servers)
systemctl disable --now bluetooth       # Bluetooth

# Check which services are listening on network ports
ss -tlnp
# or
netstat -tlnp
```

### Step 7: Kernel and Network Hardening (sysctl)

```bash
cat > /etc/sysctl.d/99-security.conf << 'EOF'
# Prevent IP spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Disable IP source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Log suspicious packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Protect against SYN flood attacks
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# Disable IPv6 if not used
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# Don't accept router advertisements
net.ipv6.conf.all.accept_ra = 0

# Protect against time-wait assassination
net.ipv4.tcp_rfc1337 = 1

# Randomize virtual address space (ASLR)
kernel.randomize_va_space = 2

# Disable core dumps for setuid programs
fs.suid_dumpable = 0
EOF

# Apply immediately
sysctl -p /etc/sysctl.d/99-security.conf
```

### Step 8: Docker Security

```bash
# Docker daemon configuration
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true,
  "live-restore": true,
  "userland-proxy": false,
  "icc": false
}
EOF

systemctl restart docker
```

| Setting | Purpose |
|---------|---------|
| `no-new-privileges` | Prevents containers from gaining additional privileges |
| `live-restore` | Keeps containers running when Docker daemon restarts |
| `userland-proxy` | Disable for better performance (use kernel's NAT instead) |
| `icc: false` | Disable inter-container communication by default |

Docker security in compose.yml:
```yaml
services:
  api:
    image: myapi:latest
    # Never run as root
    user: "1000:1000"

    # Read-only root filesystem (mount writable paths explicitly)
    read_only: true
    tmpfs:
      - /tmp

    # Drop all capabilities and add only what you need
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE   # only if binding to port < 1024

    # Prevent privilege escalation
    security_opt:
      - no-new-privileges:true

    # Limit resources
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
        reservations:
          memory: 128M
```

---

## Common Patterns & Best Practices

### SSH Key Management

```bash
# Generate a strong key locally
ssh-keygen -t ed25519 -C "deploy@myapp.com" -f ~/.ssh/myapp_deploy

# Add to server's authorized_keys
ssh-copy-id -i ~/.ssh/myapp_deploy.pub deploy@YOUR_VPS_IP

# Use in SSH config (~/.ssh/config on your local machine)
Host myapp
  HostName YOUR_VPS_IP
  User deploy
  IdentityFile ~/.ssh/myapp_deploy
  IdentitiesOnly yes

# Then connect with:
ssh myapp
```

### Two-Factor Authentication for SSH

```bash
# Install Google Authenticator PAM module
apt install -y libpam-google-authenticator

# Configure for deploy user
su - deploy
google-authenticator   # follow prompts, save the QR code

# Configure PAM
echo "auth required pam_google_authenticator.so" >> /etc/pam.d/sshd

# Update sshd_config
echo "AuthenticationMethods publickey,keyboard-interactive" >> /etc/ssh/sshd_config.d/hardening.conf
sed -i 's/KbdInteractiveAuthentication no/KbdInteractiveAuthentication yes/' /etc/ssh/sshd_config.d/hardening.conf

systemctl restart sshd
```

### Port Knocking (optional, for SSH)

Change SSH to a non-standard port to reduce log noise:

```bash
# Change SSH port (in sshd_config.d/hardening.conf)
Port 2222

# Update UFW
ufw delete allow ssh
ufw allow 2222/tcp

# Update your local ~/.ssh/config
Host myapp
  Port 2222
```

This doesn't improve security significantly but dramatically reduces brute-force log noise.

### Immutable /etc/hosts

Prevent DNS rebinding and ensure consistent hostname resolution:

```bash
# Add to /etc/hosts
127.0.0.1   localhost
YOUR_VPS_IP  myapp.com

# Make immutable (prevents accidental changes)
chattr +i /etc/hosts
```

---

## Anti-Patterns to Avoid

### Disabling the Firewall "Just for Testing"

```bash
# Never do this in production
ufw disable

# If you need to temporarily allow something, be specific
ufw allow from YOUR.IP to any port 5432
# Remove it when done
ufw delete allow from YOUR.IP to any port 5432
```

### Keeping Root SSH Enabled

Password-based root SSH is the single most common VPS attack vector. Disable it immediately:

```
PermitRootLogin no
PasswordAuthentication no
```

### Storing Private Keys on the Server

Never put your private SSH key on the VPS. The VPS holds only public keys in `authorized_keys`.

### Running Everything as Root

Even in Docker, running as root means a container escape gives the attacker root on the host. Always use non-root users in containers.

### Not Rotating Secrets

SSH keys, API tokens, and database passwords should be rotated periodically. If a key is compromised and you don't know it, rotation limits the exposure window.

---

## Debugging & Troubleshooting

### Locked Out via SSH

If you accidentally locked yourself out:

1. Most VPS providers offer a web console / VNC access — use it
2. From the console, fix sshd_config and restart sshd
3. Or temporarily re-enable password auth, fix the issue, disable again

```bash
# From web console
sshd -T | grep -E "passwordauth|permitrootlogin|allowusers"
systemctl restart sshd
```

### Fail2ban Blocking Legitimate Access

```bash
# Check if your IP is banned
fail2ban-client status sshd
iptables -L INPUT -n | grep YOUR.IP

# Unban
fail2ban-client set sshd unbanip YOUR.IP

# Whitelist your IP permanently
echo "ignoreip = 127.0.0.1/8 ::1 YOUR.IP" >> /etc/fail2ban/jail.local
systemctl restart fail2ban
```

### UFW Blocking Something Unexpectedly

```bash
ufw status numbered           # see rules with numbers
ufw status verbose            # detailed output
ufw delete 5                  # delete rule by number

# Temporarily disable to test if UFW is the issue
ufw disable
# ... test ...
ufw enable
```

### Check Who Has Access

```bash
# Who is currently logged in
who
w

# Recent login history
last -n 20
lastb -n 20   # failed login attempts

# Check authorized_keys
cat /home/deploy/.ssh/authorized_keys

# Check sudo access
cat /etc/sudoers
ls -la /etc/sudoers.d/
```

### Security Audit

```bash
# Install lynis for automated security auditing
apt install -y lynis
lynis audit system

# It will give a score and specific recommendations
```

---

## Real-World Scenarios

### Scenario 1: Post-Breach Response

If you suspect a breach:

```bash
# 1. Check who is currently logged in
who
ps aux | grep -v "^\[" | sort -k2

# 2. Check recent commands
last -n 30
cat /root/.bash_history
cat /home/deploy/.bash_history

# 3. Check for unexpected processes
ps aux | grep -E "nc|ncat|socat|python|perl|ruby" | grep -v grep

# 4. Check for unusual network connections
ss -tlnp
ss -tnp | grep ESTABLISHED

# 5. Check recently modified files
find /etc /usr/bin /usr/sbin -newer /etc/passwd -type f 2>/dev/null
find /tmp /var/tmp -type f -newer /var/log/auth.log 2>/dev/null

# 6. If breach confirmed: take snapshot, terminate, rebuild
```

### Scenario 2: Hardening Checklist for New Server

```bash
#!/usr/bin/env bash
# Quick hardening check — run after initial setup

echo "=== SSH Config ==="
sshd -T | grep -E "^(passwordauth|permitrootlogin|allowusers|port)"

echo "=== Firewall ==="
ufw status

echo "=== Fail2ban ==="
systemctl is-active fail2ban

echo "=== Auto-updates ==="
systemctl is-active unattended-upgrades

echo "=== Docker daemon ==="
cat /etc/docker/daemon.json

echo "=== Listening ports ==="
ss -tlnp

echo "=== Last logins ==="
last -n 5
```

### Scenario 3: Rotating SSH Keys

```bash
# 1. Generate new key pair on local machine
ssh-keygen -t ed25519 -C "deploy-new-$(date +%Y%m)" -f ~/.ssh/myapp_new

# 2. Add new public key to server (while old key still works)
ssh deploy@myapp "echo '$(cat ~/.ssh/myapp_new.pub)' >> ~/.ssh/authorized_keys"

# 3. Test new key works
ssh -i ~/.ssh/myapp_new deploy@myapp whoami

# 4. Remove old key from server
ssh -i ~/.ssh/myapp_new deploy@myapp \
  "sed -i '/old-key-comment/d' ~/.ssh/authorized_keys"

# 5. Update your SSH config to use new key
```

---

## Further Reading

- [Ubuntu Security Guide](https://ubuntu.com/security/certifications/docs/2204)
- [CIS Benchmarks for Ubuntu](https://www.cisecurity.org/benchmark/ubuntu_linux)
- [Lynis — security auditing tool](https://cisofy.com/lynis/)
- [Fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/MANUAL_0_8)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)

---

## Summary

| Control | Command / Config |
|---------|-----------------|
| Non-root user | `adduser deploy && usermod -aG sudo deploy` |
| SSH keys only | `PasswordAuthentication no` in sshd_config |
| Disable root SSH | `PermitRootLogin no` |
| Firewall | `ufw default deny incoming` + allow 22/80/443 |
| Brute-force protection | `fail2ban` with `maxretry=3`, `bantime=86400` for SSH |
| Auto updates | `unattended-upgrades` with reboot at 3 AM |
| Kernel hardening | sysctl: SYN cookies, ASLR, no ICMP redirects |
| Docker security | `no-new-privileges`, non-root containers, `icc=false` |
| Audit | `lynis audit system` — run monthly |

Security is a process, not a state. Harden on day one, audit regularly, rotate credentials, and monitor logs. The goal is to be a harder target than the other VPS running default configs.
