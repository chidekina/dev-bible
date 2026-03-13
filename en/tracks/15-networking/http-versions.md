# HTTP Versions

HTTP has evolved significantly from its origins. Understanding the differences between versions explains why modern web performance looks the way it does.

## HTTP/0.9 and HTTP/1.0 (Historical)

```
HTTP/0.9 (1991): GET only, no headers, no status codes
HTTP/1.0 (1996): headers, status codes, one request per connection
```

Each HTTP/1.0 request required a new TCP connection — extremely inefficient.

## HTTP/1.1 (1997 — still widely used)

Major improvements:
- **Persistent connections**: `Connection: keep-alive` — reuse TCP connections
- **Pipelining**: send multiple requests without waiting for responses (rarely used in practice)
- **Chunked transfer encoding**: stream responses without knowing Content-Length
- **Host header**: enables virtual hosting (multiple domains on one IP)
- **Cache control headers**: `Cache-Control`, `ETag`, `If-Modified-Since`

### HTTP/1.1 Limitations

```
Head-of-line blocking:
Request 1 (large) ─────────────────────────► Response 1 (slow)
Request 2 (small) ─── waiting ──────────────► Response 2
Request 3 (small) ─────────── waiting ───────► Response 3

Browsers worked around this with 6 parallel TCP connections per domain.
```

### HTTP/1.1 Message Format

```http
# Request
GET /api/users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJ...
Accept: application/json

# Response
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 248
Cache-Control: max-age=60

{"users": [...]}
```

## HTTP/2 (2015)

HTTP/2 keeps the same semantics (methods, headers, status codes) but completely changes how data is transmitted.

### Key Features

**Binary framing**: instead of text, HTTP/2 uses binary frames — more efficient to parse

```
HTTP/1.1: GET /index.html HTTP/1.1\r\nHost: example.com\r\n\r\n  (text)
HTTP/2:   [HEADERS frame: binary-encoded headers]               (binary)
```

**Multiplexing**: multiple requests/responses interleaved on one TCP connection

```
Stream 1: ─── H ─── D1 ─── D2 ─────────────── END
Stream 3: ────────── H ─── D1 ──────── END
Stream 5: ─────────────────── H ─── D ─── END
          ─────────────────────────────────────────► (one TCP connection)
```

**Header compression (HPACK)**: headers are compressed and cached across requests

```
Request 1: sends full headers
Request 2: only sends headers that changed (delta compression)
→ typical 70-85% reduction in header bytes
```

**Server Push**: server can proactively send resources the client will need

```
Client: GET /index.html
Server: [index.html] + PUSH_PROMISE [style.css] + PUSH_PROMISE [app.js]
→ Client receives resources before it parses HTML and discovers them
```

**Stream prioritization**: client can signal which resources matter most

### HTTP/2 Still Has TCP HOL Blocking

HTTP/2 eliminates application-level HOL blocking through multiplexing, but TCP-level HOL blocking remains:

```
If a TCP packet is lost, ALL HTTP/2 streams on that connection stall
until retransmission arrives — even streams with no dependency on
the lost packet.
```

## HTTP/3 (2022)

HTTP/3 replaces TCP with **QUIC** (built on UDP), eliminating TCP's fundamental limitations.

### QUIC Features

**No HOL blocking**: each QUIC stream has independent reliability

```
Stream 1 lost packet: only stream 1 stalls
Stream 2, 3: unaffected — continue delivering data
```

**Integrated TLS 1.3**: no separate TLS handshake

```
HTTP/1.1 + TLS: TCP SYN → TCP SYN-ACK → TCP ACK → TLS ClientHello
                → TLS ServerHello → TLS Finished → First request
                = 3 round trips

HTTP/3 + QUIC:  QUIC Initial (includes TLS ClientHello)
                → QUIC Response (includes TLS Finished + HTTP data)
                = 1 round trip (0-RTT for resumed connections)
```

**Connection migration**: identified by connection ID, not IP:port

```
Mobile user switches from WiFi to 4G → IP changes
HTTP/3: connection survives (same connection ID)
HTTP/1.1/2: connection drops, must reconnect
```

### HTTP/3 Adoption

```bash
# Check if a site supports HTTP/3
curl -I --http3 https://cloudflare.com

# Or look for Alt-Svc header in HTTP/2 response:
Alt-Svc: h3=":443"; ma=86400
```

## Comparison Summary

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Protocol | Text | Binary | Binary |
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | No (6 conns) | Yes (1 conn) | Yes (1 conn) |
| HOL Blocking | Yes (app+TCP) | TCP only | No |
| Header compression | No | HPACK | QPACK |
| TLS required | No | Recommended | Yes (built-in) |
| Connection setup | 1-3 RTT | 1-3 RTT | 0-1 RTT |
| Server Push | No | Yes | Yes (limited) |

## HTTP Methods

| Method | Idempotent | Safe | Body | Use |
|--------|-----------|------|------|-----|
| GET | Yes | Yes | No | Retrieve |
| POST | No | No | Yes | Create |
| PUT | Yes | No | Yes | Replace |
| PATCH | No | No | Yes | Partial update |
| DELETE | Yes | No | No | Delete |
| HEAD | Yes | Yes | No | Metadata only |
| OPTIONS | Yes | Yes | No | CORS preflight |

## Common Status Codes

```
1xx Informational
  100 Continue

2xx Success
  200 OK             201 Created
  204 No Content     206 Partial Content

3xx Redirection
  301 Moved Permanently   302 Found (temp redirect)
  304 Not Modified        307 Temporary Redirect
  308 Permanent Redirect

4xx Client Error
  400 Bad Request    401 Unauthorized    403 Forbidden
  404 Not Found      405 Method Not Allowed
  409 Conflict       422 Unprocessable Entity
  429 Too Many Requests

5xx Server Error
  500 Internal Server Error   502 Bad Gateway
  503 Service Unavailable     504 Gateway Timeout
```

## Practice

1. Open DevTools Network tab. Filter by protocol and find which resources use h2 vs h3.
2. Run `curl -v https://example.com 2>&1 | grep -E 'HTTP|<'` and read the response headers.
3. Explain why HTTP/2 multiplexing doesn't fully solve the HOL blocking problem.
