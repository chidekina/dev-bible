# TCP vs UDP

TCP and UDP are the two primary transport layer protocols. Choosing between them is a fundamental design decision in any networked application.

## Side-by-Side Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented (handshake) | Connectionless |
| Reliability | Guaranteed delivery | Best-effort |
| Ordering | In-order delivery | No ordering guarantee |
| Error checking | Checksum + retransmission | Checksum only (optional) |
| Flow control | Yes (sliding window) | No |
| Congestion control | Yes | No |
| Speed | Slower (overhead) | Faster |
| Header size | 20–60 bytes | 8 bytes |
| Use cases | HTTP, email, SSH, databases | DNS, video streaming, gaming, VoIP |

## TCP Deep Dive

TCP trades speed for reliability. Every byte sent is acknowledged. If an ACK doesn't arrive within the timeout, TCP retransmits.

```
Data transfer:
Sender ──── [Seg 1] ───────────────────────► Receiver
Sender ◄─── [ACK 2] ────────────────────────
Sender ──── [Seg 2] ───────────────────────►
Sender ◄─── [ACK 3] ────────────────────────

Packet loss:
Sender ──── [Seg 2] ──── LOST ──── ✗
                                         (timeout)
Sender ──── [Seg 2] ─── retransmit ─────►
Sender ◄─── [ACK 3] ────────────────────────
```

### When TCP Head-of-Line Blocking Hurts

In HTTP/1.1 over TCP, if one packet is lost, **all** subsequent data is held up waiting for retransmission — even data for independent resources. This is TCP head-of-line blocking.

```
Request A: chunks 1, 2, [3 lost], 4, 5
Request B: chunks 1, 2, 3

Both stall until chunk 3 of Request A is retransmitted.
```

HTTP/2 multiplexes streams over one TCP connection — but TCP HOL blocking still affects all streams. HTTP/3 moves to QUIC (UDP-based) to eliminate this.

## UDP Deep Dive

UDP is a thin wrapper around IP. It adds only ports and a checksum.

```
UDP Header (8 bytes):
┌────────────┬────────────┐
│  Src Port  │  Dst Port  │
├────────────┼────────────┤
│   Length   │  Checksum  │
└────────────┴────────────┘
│   Data...               │
```

### When UDP Is the Right Choice

**Real-time applications** that can tolerate some loss but not delay:
- VoIP: a dropped audio packet sounds like a click; waiting for retransmission sounds like silence
- Online gaming: old position data is useless; fresh data matters more
- Live video streaming: viewers tolerate a glitch, not a pause

**Simple request-response at scale:**
- DNS queries: one packet out, one packet in — no need for a connection

**Multicast/broadcast:**
- UDP supports sending to multiple recipients; TCP is unicast only

## QUIC — The Best of Both

QUIC (used in HTTP/3) is built on UDP but implements:
- Reliable delivery per stream (not per connection)
- TLS 1.3 integrated (0-RTT connection setup)
- No head-of-line blocking at the transport layer
- Connection migration (works across IP changes — great for mobile)

```
HTTP/1.1: TCP + TLS 1.2 = 3 RTTs before data
HTTP/2:   TCP + TLS 1.3 = 2 RTTs before data
HTTP/3:   QUIC (UDP)    = 1 RTT (or 0-RTT for resumed connections)
```

## Reliability on Top of UDP

When you need some reliability guarantees but not full TCP overhead, you can build it at the application layer:

```typescript
// Simplified example: ACK-based reliability over UDP
const dgram = require('dgram');
const socket = dgram.createSocket('udp4');
const pending = new Map<number, NodeJS.Timeout>();

function sendReliable(seq: number, data: Buffer, host: string, port: number) {
  socket.send(data, port, host);
  // Retransmit if no ACK in 500ms
  const timer = setTimeout(() => sendReliable(seq, data, host, port), 500);
  pending.set(seq, timer);
}

function onAck(seq: number) {
  const timer = pending.get(seq);
  if (timer) { clearTimeout(timer); pending.delete(seq); }
}
```

Libraries like **KCP**, **QUIC**, and game networking SDKs (ENet, Steam Networking) do this properly.

## Decision Guide

```
Need reliable delivery?
  Yes → TCP (HTTP, SSH, databases, email)
  No  → Is latency critical?
          Yes → UDP (gaming, VoIP, live streaming)
          No  → Still TCP (simpler to implement correctly)

Need multicast?
  Yes → UDP

Building on top of HTTP?
  HTTP/1.1, HTTP/2 → TCP (automatic)
  HTTP/3, WebTransport → QUIC (automatic)
```

## Practice

1. Use `tcpdump -i any port 53` and run `dig google.com`. Confirm DNS uses UDP.
2. Start a simple UDP server in Node.js. Send it a message from another terminal with `nc -u`.
3. Research QUIC: why does 0-RTT resumption exist, and what security trade-off does it make?
