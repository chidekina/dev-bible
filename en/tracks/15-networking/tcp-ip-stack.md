# TCP/IP Stack

TCP (Transmission Control Protocol) running over IP (Internet Protocol) is the foundation of most internet communication. Understanding how they work together is essential for debugging network issues and designing reliable systems.

## IP — Internet Protocol

IP is responsible for **addressing** and **routing** packets across networks. It is connectionless and unreliable — it makes a best-effort delivery with no guarantees.

### IPv4 Addressing

```
Format: X.X.X.X where X is 0–255
Example: 192.168.1.100

Classes (historical, now replaced by CIDR):
Class A: 0.0.0.0   – 127.255.255.255  (/8,  16M hosts)
Class B: 128.0.0.0 – 191.255.255.255  (/16, 65K hosts)
Class C: 192.0.0.0 – 223.255.255.255  (/24, 254 hosts)

Private ranges (RFC 1918 — not routable on internet):
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

### CIDR Notation

```
192.168.1.0/24
              └── 24 bits for network, 8 bits for hosts
                  → 254 usable host addresses

Subnet mask: /24 = 255.255.255.0

Usable hosts = 2^(32-prefix) - 2
  /24 → 254 hosts
  /16 → 65,534 hosts
  /30 → 2 hosts (point-to-point links)
```

### IPv6

```
Format: 8 groups of 4 hex digits
Example: 2001:0db8:85a3:0000:0000:8a2e:0370:7334
Shortened: 2001:db8:85a3::8a2e:370:7334

Loopback: ::1
Link-local: fe80::/10
Global unicast: 2000::/3
```

## TCP — Transmission Control Protocol

TCP provides **reliable, ordered, error-checked** delivery of data between applications. Key features:

- **Connection-oriented**: establish connection before data transfer
- **Reliable**: guarantees delivery via acknowledgements and retransmission
- **Ordered**: data arrives in the order it was sent
- **Flow control**: receiver controls how fast sender sends
- **Congestion control**: adapts to network capacity

### Ports

```
Port range: 0–65535

Well-known ports (0–1023) — require root/admin:
  22   SSH
  25   SMTP
  53   DNS
  80   HTTP
  443  HTTPS
  5432 PostgreSQL
  3306 MySQL
  6379 Redis

Registered ports (1024–49151): assigned to services
Ephemeral ports (49152–65535): assigned dynamically to clients
```

### Three-Way Handshake (Connection Setup)

```
Client                    Server
  │                          │
  │──── SYN (seq=x) ────────►│  Client initiates
  │                          │
  │◄─── SYN-ACK (seq=y,      │  Server acknowledges
  │       ack=x+1) ──────────│
  │                          │
  │──── ACK (ack=y+1) ──────►│  Client confirms
  │                          │
  │    [Connection OPEN]     │
  │                          │
```

**SYN** = Synchronize sequence numbers  
**ACK** = Acknowledgement  
After handshake: both sides have agreed on initial sequence numbers.

### Four-Way Termination (Connection Teardown)

```
Client                    Server
  │                          │
  │──── FIN ────────────────►│  Client done sending
  │◄─── ACK ─────────────────│  Server acks
  │◄─── FIN ─────────────────│  Server done sending
  │──── ACK ────────────────►│  Client acks
  │                          │
  │    [Connection CLOSED]   │
```

Client enters **TIME_WAIT** (2×MSL ≈ 60–120s) before fully closing, to handle delayed packets.

### TCP Segment Structure

```
┌────────────┬────────────┐
│  Src Port  │  Dst Port  │  (16 bits each)
├────────────┴────────────┤
│     Sequence Number     │  (32 bits)
├─────────────────────────┤
│  Acknowledgement Number │  (32 bits)
├──────┬──────────────────┤
│ Data │    Flags         │  SYN, ACK, FIN, RST, PSH, URG
│ Off  │                  │
├──────┴──────────────────┤
│   Window Size           │  Flow control
├─────────────────────────┤
│   Checksum              │
├─────────────────────────┤
│   Data...               │
└─────────────────────────┘
```

### Flow Control — Sliding Window

```
Receiver advertises window size (how much buffer is available).
Sender cannot have more unacknowledged bytes than window size.

Window = 3 segments:
Sent+Ack'd  Sent+Unack'd  Can Send  Cannot Send Yet
[  1  2  ] [  3  4  5  ] [ 6 7 8 ] [ 9 10 ...    ]
```

### Congestion Control

```
Slow Start: begin with small window, double each RTT
Congestion Avoidance: grow linearly after threshold
Fast Retransmit: 3 duplicate ACKs → retransmit immediately
Fast Recovery: reduce window by half (not to 1)
```

## Sockets

A socket is the OS abstraction for a network connection, identified by:

```
(src_ip, src_port, dst_ip, dst_port, protocol)

Example:
192.168.1.5:54321 ──TCP──► 93.184.216.34:443
```

```typescript
// Node.js socket example
import net from 'net';

const client = net.createConnection({ port: 3000, host: 'localhost' }, () => {
  console.log('Connected');
  client.write('Hello!');
});

client.on('data', (data) => {
  console.log(data.toString());
  client.end();
});
```

## Common Diagnostics

```bash
# See all active TCP connections and listening ports
ss -tulnp
netstat -tulnp  # older alternative

# Check if port is open on remote host
nc -zv example.com 443

# View TCP connection states
ss -s

# Watch connection count in TIME_WAIT
ss -tan | grep TIME-WAIT | wc -l
```

## Common Issues

| Symptom | Likely Cause |
|---------|-------------|
| Connection refused | Nothing listening on that port |
| Connection timed out | Firewall dropping packets (no RST) |
| Too many TIME_WAIT | High connection churn, tune `tcp_tw_reuse` |
| RST received | Peer forcefully closed connection |
| Retransmissions | Packet loss, congestion |

## Practice

1. Run `ss -tulnp` on your machine. Identify what's listening on common ports.
2. Use `nc -zv google.com 80` and `nc -zv google.com 443`. What's the difference?
3. Explain why TIME_WAIT exists and what problems removing it could cause.
