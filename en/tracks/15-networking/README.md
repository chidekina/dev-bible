# Track 15 — Networking

Understand how data moves across networks — from physical bits to application protocols.

## Chapters

| # | Chapter | Topics |
|---|---------|--------|
| 1 | [OSI Model](./osi-model.md) | 7 layers, responsibilities, encapsulation |
| 2 | [TCP/IP Stack](./tcp-ip-stack.md) | TCP internals, three-way handshake, ports, sockets |
| 3 | [TCP vs UDP](./tcp-vs-udp.md) | Reliability, flow control, use cases |
| 4 | [DNS, DHCP & NAT](./dns-dhcp-nat.md) | Name resolution, IP assignment, address translation |
| 5 | [HTTP Versions](./http-versions.md) | HTTP/1.1 vs HTTP/2 vs HTTP/3, QUIC |
| 6 | [TLS & Certificates](./tls-certificates.md) | TLS handshake, PKI, mTLS, certificate chains |
| 7 | [Firewalls & VPNs](./firewalls-vpn.md) | Packet filtering, stateful inspection, VPN tunnels |
| 8 | [Network Troubleshooting](./network-troubleshooting.md) | ping, traceroute, dig, netstat, curl, tcpdump |

## Prerequisites

- Basic Linux command line (Track 05)
- Docker networking concepts (Track 05)

## Why This Track Matters

Every web request, database query, and microservice call crosses a network. Understanding networking lets you:
- Debug latency and connection issues
- Design secure architectures
- Configure firewalls, proxies, and load balancers
- Understand what happens when things fail
