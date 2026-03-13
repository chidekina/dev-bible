# OSI Model

The Open Systems Interconnection (OSI) model is a conceptual framework that standardizes network communication into 7 layers. Each layer has a specific responsibility and communicates only with the layers directly above and below it.

## The 7 Layers

```
┌─────────────────────────────────────────┐
│  7 - Application   (HTTP, DNS, SMTP)    │
├─────────────────────────────────────────┤
│  6 - Presentation  (TLS, encoding)      │
├─────────────────────────────────────────┤
│  5 - Session       (session management) │
├─────────────────────────────────────────┤
│  4 - Transport     (TCP, UDP)           │
├─────────────────────────────────────────┤
│  3 - Network       (IP, routing)        │
├─────────────────────────────────────────┤
│  2 - Data Link     (Ethernet, MAC)      │
├─────────────────────────────────────────┤
│  1 - Physical      (cables, signals)    │
└─────────────────────────────────────────┘
```

Mnemonic: **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

## Layer Breakdown

### Layer 1 — Physical
- Transmits raw bits over physical media
- Examples: Ethernet cables, fiber optic, Wi-Fi radio waves
- Devices: hubs, repeaters, network interface cards (NICs)
- Unit: **bit**

### Layer 2 — Data Link
- Node-to-node data transfer on the same network segment
- Adds MAC addresses, error detection (CRC)
- Examples: Ethernet frames, Wi-Fi (802.11), ARP
- Devices: switches, bridges
- Unit: **frame**

```
Ethernet Frame:
┌──────────┬──────────┬──────┬──────────────┬─────┐
│ Dst MAC  │ Src MAC  │ Type │   Payload    │ CRC │
│ 6 bytes  │ 6 bytes  │ 2 B  │  46-1500 B   │ 4 B │
└──────────┴──────────┴──────┴──────────────┴─────┘
```

### Layer 3 — Network
- Logical addressing and routing between networks
- Determines the best path for data to travel
- Examples: IP (v4/v6), ICMP, routing protocols (OSPF, BGP)
- Devices: routers
- Unit: **packet**

```
IPv4 Header (simplified):
┌──────────┬───────┬──────────┬──────┬──────────┐
│ Version  │  TTL  │ Protocol │ Src  │   Dst    │
│  4 bits  │ 8 bits│  8 bits  │ IP   │   IP     │
└──────────┴───────┴──────────┴──────┴──────────┘
```

### Layer 4 — Transport
- End-to-end communication, segmentation, flow control
- Examples: TCP (reliable), UDP (fast)
- Devices: firewalls (L4), load balancers
- Unit: **segment** (TCP) / **datagram** (UDP)
- Key concepts: ports (0–65535), multiplexing

### Layer 5 — Session
- Manages sessions between applications
- Handles setup, maintenance, and teardown of connections
- Examples: RPC, NetBIOS, SQL sessions
- In practice: often merged with L4/L6 in TCP/IP

### Layer 6 — Presentation
- Data translation, encryption, compression
- Examples: TLS/SSL encryption, JPEG compression, character encoding (UTF-8)
- In practice: TLS is the dominant L6 protocol

### Layer 7 — Application
- Closest to the user; provides network services to applications
- Examples: HTTP, HTTPS, DNS, SMTP, FTP, SSH, WebSocket
- Devices: application-layer firewalls, API gateways, CDNs
- Unit: **data** / **message**

## Encapsulation

As data travels down the stack, each layer **wraps** the payload with its own header (encapsulation). Going up, each layer **unwraps** (decapsulation).

```
Sending:
App data
→ [TCP header | App data]                    (Segment)
→ [IP header | TCP header | App data]         (Packet)
→ [ETH header | IP | TCP | App data | CRC]    (Frame)
→ 101010010111...                             (Bits)

Receiving: reverse order, headers stripped layer by layer
```

## TCP/IP Model vs OSI

In practice, the TCP/IP model (4 layers) is what the internet uses. The OSI model is used for conceptual understanding.

```
OSI              TCP/IP
────────────     ────────────
7 Application  ┐
6 Presentation ├─ Application
5 Session      ┘
4 Transport    ── Transport
3 Network      ── Internet
2 Data Link    ┐
1 Physical     ┘── Network Access
```

## Common Interview Questions

**Q: At which layer does a router operate?**
A: Layer 3 (Network). It reads IP addresses to make routing decisions.

**Q: At which layer does a switch operate?**
A: Layer 2 (Data Link). It uses MAC addresses to forward frames within a LAN.

**Q: Where does TLS fit?**
A: Conceptually Layer 6 (Presentation), but in TCP/IP it sits between Transport and Application.

**Q: What is a Layer 7 load balancer?**
A: One that inspects HTTP content (headers, URL, cookies) to make routing decisions — vs a Layer 4 load balancer that only sees IP/port.

## Key Protocols by Layer

| Layer | Protocols |
|-------|----------|
| 7 Application | HTTP, HTTPS, DNS, SMTP, FTP, SSH, WebSocket, gRPC |
| 6 Presentation | TLS/SSL, MIME, JPEG, ASCII |
| 5 Session | RPC, NetBIOS |
| 4 Transport | TCP, UDP, QUIC |
| 3 Network | IPv4, IPv6, ICMP, OSPF, BGP |
| 2 Data Link | Ethernet, Wi-Fi (802.11), ARP, VLAN |
| 1 Physical | Ethernet cable, fiber, coax, radio |

## Practice

1. Open Wireshark and capture an HTTP request. Identify each layer's headers in the capture.
2. Run `ping 8.8.8.8`. Which OSI layer does ping operate at? (Hint: ICMP)
3. Explain why a switch doesn't need an IP address but a router does.
