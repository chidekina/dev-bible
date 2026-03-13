# Firewalls & VPNs

Firewalls control which traffic is allowed in and out of a network. VPNs create encrypted tunnels that protect and route traffic through a different endpoint.

## Firewalls

A firewall inspects network traffic and decides whether to **allow** or **deny** it based on rules.

### Types of Firewalls

**Packet filter (stateless)**: inspects each packet independently
```
Rule: ALLOW TCP from any to 203.0.113.1:443
Rule: DENY all

Fast but can't understand connections — a response packet looks
the same as an attack packet if evaluated without context.
```

**Stateful inspection (connection tracking)**: tracks TCP connections
```
Rule: ALLOW ESTABLISHED,RELATED connections (return traffic)
Rule: ALLOW new TCP to :443
Rule: DENY all new inbound

Knows that TCP response on port 54321 belongs to an established
outbound connection — allows it without explicit rule.
```

**Application layer firewall (L7)**: inspects payload content
```
Can detect: SQL injection in HTTP params
Can block: specific URLs, file types, HTTP methods
Can inspect: TLS (with decryption proxy)
Used as: WAF (Web Application Firewall)
```

### iptables (Linux)

```bash
# View current rules
iptables -L -n -v
iptables -t nat -L -n -v  # NAT table

# Allow established connections (stateful)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from specific IP
iptables -A INPUT -p tcp -s 203.0.113.5 --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Default deny (put this last)
iptables -P INPUT DROP

# Port forwarding (DNAT)
iptables -t nat -A PREROUTING -p tcp --dport 8080 \
  -j DNAT --to-destination 192.168.1.10:80
iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 80 -j ACCEPT

# Save rules
iptables-save > /etc/iptables/rules.v4
```

### nftables (Modern Linux)

```bash
# nftables is the replacement for iptables
nft list ruleset

# Basic config at /etc/nftables.conf:
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    ct state established,related accept
    iif lo accept
    tcp dport { 22, 80, 443 } accept
  }
  chain output {
    type filter hook output priority 0; policy accept;
  }
}
```

### UFW (Ubuntu Simplified Firewall)

```bash
# UFW wraps iptables with simpler syntax
ufw status
ufw enable
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny from 192.168.1.100
ufw delete allow 80/tcp
```

### Cloud Security Groups

Cloud providers (AWS, GCP, Azure) provide virtual firewalls as Security Groups:

```
AWS Security Group — Web Server:
Inbound:
  HTTP   :80   from 0.0.0.0/0
  HTTPS  :443  from 0.0.0.0/0
  SSH    :22   from 10.0.0.0/8 (internal only)
Outbound:
  All traffic to 0.0.0.0/0

Key differences from iptables:
- Stateful by default (return traffic auto-allowed)
- Applied at instance level (in addition to subnet NACLs)
- Allow-only rules (no explicit deny — everything not allowed is denied)
```

## VPNs — Virtual Private Networks

A VPN creates an encrypted tunnel between two endpoints, making traffic appear to originate from the VPN server's location.

### How VPN Tunneling Works

```
Without VPN:
You → ISP → Internet → Server
     (plaintext visible to ISP)

With VPN:
You → [Encrypted tunnel] → VPN Server → Internet → Server
     (ISP sees only VPN traffic, not destination or content)
```

### VPN Protocols

**WireGuard** (modern, recommended):
```
- Minimal codebase (~4000 lines vs OpenVPN's ~400k)
- State-of-the-art cryptography (ChaCha20, Curve25519, BLAKE2)
- Very fast — comparable to unencrypted connections
- Built into Linux kernel 5.6+
- UDP-based (port 51820 by default)
```

**OpenVPN** (mature, widely deployed):
```
- TLS-based, TCP or UDP
- Very configurable, complex
- Slower than WireGuard
- Works through most firewalls (can use TCP:443)
```

**IPSec/IKEv2** (common on mobile, corporate VPNs):
```
- Native support in iOS, Android, Windows
- Fast reconnection on network change
- Complex configuration
```

### WireGuard Setup Example

```ini
# /etc/wireguard/wg0.conf (server)
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server-private-key>

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; \
         iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; \
           iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.8.0.2/32
```

```ini
# /etc/wireguard/wg0.conf (client)
[Interface]
Address = 10.8.0.2/24
PrivateKey = <client-private-key>
DNS = 1.1.1.1

[Peer]
PublicKey = <server-public-key>
Endpoint = 203.0.113.1:51820
AllowedIPs = 0.0.0.0/0  # Route all traffic through VPN
PersistentKeepalive = 25
```

```bash
# Start WireGuard
wg-quick up wg0

# Generate key pair
wg genkey | tee privatekey | wg pubkey > publickey
```

### Site-to-Site vs Client VPN

```
Client VPN (Remote Access):
Laptop ──[VPN]──► Corporate Network
  One device connects to a network

Site-to-Site VPN:
Office Network A ──[VPN]──► Cloud VPC
  Two networks connect to each other
  Often: corporate HQ ↔ AWS VPC
```

## Practice

1. Check which ports are open on your VPS with `nmap -sS -p 1-1000 <your-vps-ip>` (from another machine).
2. Add a UFW rule to allow only your IP to SSH. Test it works, then verify others are blocked.
3. Read the WireGuard whitepaper summary — why is its minimal codebase a security advantage?
