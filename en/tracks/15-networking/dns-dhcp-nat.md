# DNS, DHCP & NAT

Three protocols that make the internet usable: DNS translates names to addresses, DHCP assigns addresses automatically, and NAT lets many devices share a single public IP.

## DNS — Domain Name System

DNS is the internet's phone book. It translates human-readable domain names into IP addresses.

### DNS Hierarchy

```
.                              ← Root (13 root server clusters)
├── com.                       ← TLD (Top-Level Domain)
│   ├── google.com.            ← Authoritative zone
│   │   ├── www.google.com.    ← A record → 142.250.80.4
│   │   └── mail.google.com.   ← MX record
│   └── example.com.
└── br.
    └── aethostech.com.br.
```

### Resolution Process (Recursive)

```
Browser: "What is the IP for api.example.com?"

1. Check local cache → miss
2. Ask OS resolver (127.0.0.53 / /etc/resolv.conf)
3. Resolver asks Root Server → "ask .com TLD servers"
4. Resolver asks .com TLD Server → "ask example.com nameservers"
5. Resolver asks example.com NS → "api.example.com = 93.184.216.34"
6. Resolver caches result (TTL) and returns to browser
7. Browser connects to 93.184.216.34
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|--------|
| A | IPv4 address | `api.example.com → 93.184.216.34` |
| AAAA | IPv6 address | `api.example.com → 2606:2800::1` |
| CNAME | Alias | `www → example.com` |
| MX | Mail server | `example.com → mail.example.com` |
| TXT | Arbitrary text | SPF, DKIM, domain verification |
| NS | Nameserver | Delegates zone to nameserver |
| PTR | Reverse DNS | `34.216.184.93.in-addr.arpa → example.com` |
| SRV | Service location | `_http._tcp.example.com → host:port` |
| SOA | Zone authority | Start of Authority record |

### TTL and Caching

```
TTL (Time To Live): how long resolvers cache the record

Low TTL (60s):  changes propagate fast, more DNS queries
High TTL (3600s): fewer queries, slow propagation

Before changing DNS: lower TTL to 60s a day in advance
After change: raise TTL back to normal
```

### Diagnosing DNS

```bash
# Query DNS
dig api.example.com
dig api.example.com A
dig api.example.com MX

# Query specific nameserver
dig @8.8.8.8 example.com

# Reverse lookup
dig -x 93.184.216.34

# Trace resolution path
dig +trace example.com

# Quick lookup
nslookup example.com
```

### DNS Security

- **DNSSEC**: signs records cryptographically — prevents cache poisoning
- **DoH (DNS over HTTPS)**: encrypts DNS queries in HTTPS — prevents snooping
- **DoT (DNS over TLS)**: same as DoH but over TLS port 853

## DHCP — Dynamic Host Configuration Protocol

DHCP automatically assigns IP configuration to devices joining a network, eliminating manual configuration.

### What DHCP Assigns

```
- IP address (from pool)
- Subnet mask
- Default gateway
- DNS server(s)
- Lease duration
```

### DORA Process

```
Client                    DHCP Server
  │                           │
  │──── DISCOVER (broadcast) ►│  "Anyone have an IP for me?"
  │◄─── OFFER ────────────────│  "Here's 192.168.1.50 for 24h"
  │──── REQUEST (broadcast) ──►│  "I'd like that IP"
  │◄─── ACK ──────────────────│  "It's yours for 86400s"
  │                           │
  │  [Using 192.168.1.50]     │
```

**D**iscover → **O**ffer → **R**equest → **A**cknowledge

### DHCP Lease Renewal

```
At 50% of lease: client tries to renew with same server (unicast)
At 87.5% of lease: client broadcasts to any DHCP server
At expiry: client must stop using IP and restart DORA
```

### Static vs Dynamic Assignment

```
Dynamic: server assigns from pool → devices get different IPs each time
Static (DHCP reservation): server always assigns same IP based on MAC address
  → Best of both worlds: automatic + predictable
```

## NAT — Network Address Translation

NAT allows multiple devices on a private network to share a single public IP address. It was critical for stretching the limited IPv4 address space.

### How NAT Works

```
Private Network              Internet
192.168.1.10:54321  ─┐
192.168.1.11:54322  ─┤  Router/NAT   ──►  93.184.216.34:80
192.168.1.12:54323  ─┘  203.0.113.1

NAT Table:
 Private IP:Port      →  Public IP:Port
 192.168.1.10:54321  →  203.0.113.1:1024
 192.168.1.11:54322  →  203.0.113.1:1025
 192.168.1.12:54323  →  203.0.113.1:1026
```

Return traffic arrives at the public IP, NAT rewrites destination back to private IP.

### Types of NAT

**SNAT (Source NAT)**: changes source address — used for outbound traffic (most common)

**DNAT (Destination NAT)**: changes destination address — used for port forwarding

```bash
# Port forwarding example (iptables):
# Forward external :8080 to internal server :80
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT \
  --to-destination 192.168.1.10:80
```

**MASQUERADE**: dynamic SNAT where public IP is determined at runtime (for dynamic IPs)

### NAT and Docker

```bash
# Docker creates a private network: 172.17.0.0/16
# Containers get private IPs: 172.17.0.2, 172.17.0.3...
# Docker host NATs their traffic out through the host's public IP

# Port publishing creates DNAT:
docker run -p 8080:80 nginx
# → External :8080 → container 172.17.0.2:80
```

### NAT Limitations

- Breaks protocols that embed IP in payload (FTP, SIP) — needs ALG (Application Layer Gateway)
- Complicates peer-to-peer connectivity (NAT traversal, STUN/TURN needed)
- IPv6 makes NAT largely unnecessary (every device gets a global address)

## ARP — Address Resolution Protocol

ARP maps IP addresses to MAC addresses within a local network segment.

```
Device knows: destination IP = 192.168.1.1
Device needs: destination MAC address

1. Broadcast: "Who has 192.168.1.1? Tell 192.168.1.10"
2. Target responds: "192.168.1.1 is at AA:BB:CC:DD:EE:FF"
3. Requester caches result in ARP table

Check ARP table:
arp -n         # or: ip neigh show
```

## Practice

1. Run `dig +trace google.com` and identify each step of the resolution.
2. Run `ip neigh show` to see your ARP table. What devices are listed?
3. Check `ip addr show` — is your IP dynamically assigned? Verify with `ip route show` which interface has the default route.
