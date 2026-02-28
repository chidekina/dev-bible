# How the Web Works

> Every senior developer must be able to walk from a keypress to a rendered pixel. This file covers the entire journey: DNS resolution, TCP handshake, TLS negotiation, HTTP semantics, browser rendering, and caching strategies.

---

## 1. What & Why

When a user types `https://google.com` and presses Enter, a chain of at least ten distinct technical operations happens before a single pixel appears. Most developers learn HTTP and move on, but understanding the full stack — from DNS UDP packets to browser compositing — is what separates engineers who can debug production mysteries from those who guess.

Knowing this material lets you:
- Debug "why is my site slow?" by isolating which layer is the bottleneck
- Fix CDN caching bugs that serve stale assets after a deploy
- Understand why HTTPS matters and what TLS actually does
- Optimize Time-to-First-Byte (TTFB) and Core Web Vitals
- Reason about CORS, cookies, and redirects correctly

---

## 2. Core Concepts

### DNS — Domain Name System

DNS translates human-readable names (`google.com`) into IP addresses (`142.250.80.46`). It is a distributed, hierarchical system.

**The four actors in a DNS lookup:**

1. **Recursive Resolver** (your ISP or 8.8.8.8): The server your OS talks to first. It does the heavy lifting, querying other servers on your behalf and caching results.
2. **Root Name Server**: There are 13 logical root server clusters worldwide. They know which servers are authoritative for TLDs (`.com`, `.org`, `.io`).
3. **TLD Name Server**: The `.com` TLD server knows which authoritative name server holds records for `google.com`.
4. **Authoritative Name Server**: The final authority. This server holds the actual A record (IPv4) or AAAA record (IPv6) for `google.com`.

**DNS record types:**
- `A` — maps name to IPv4 address
- `AAAA` — maps name to IPv6 address
- `CNAME` — canonical name alias (`www.example.com → example.com`)
- `MX` — mail exchange server
- `TXT` — arbitrary text (used for SPF, DKIM, domain verification)
- `NS` — name server records
- `SOA` — Start of Authority, zone metadata

**TTL (Time To Live):** Every DNS record has a TTL in seconds. Caches honour this TTL. A TTL of 300 means resolvers re-query after 5 minutes. During a DNS migration, lowering TTL beforehand (to 60s) lets changes propagate faster.

---

### TCP/IP — Transmission Control Protocol

TCP is a connection-oriented, reliable transport protocol. Before any data is exchanged, a **three-way handshake** establishes the connection:

```
Client                          Server
  |                               |
  |—— SYN (seq=x) ——————————————>|   "I want to connect"
  |                               |
  |<—— SYN-ACK (seq=y, ack=x+1) —|   "OK, I heard you"
  |                               |
  |—— ACK (seq=x+1, ack=y+1) ——>|   "I heard your reply"
  |                               |
  |   ← connection established →  |
```

This handshake adds one **round-trip time (RTT)** before any data flows. On a mobile network with 100ms RTT, that is 100ms before TLS even starts.

TCP also handles:
- **Segmentation**: breaking data into packets
- **Ordering**: reassembling out-of-order packets
- **Flow control**: receiver tells sender how fast to send (window size)
- **Congestion control**: slow-start algorithm ramps up bandwidth

---

### TLS — Transport Layer Security

TLS (successor to SSL) encrypts the connection. TLS 1.3 (current) requires only **1 RTT** for the handshake (TLS 1.2 needed 2 RTTs).

**TLS 1.3 handshake:**

```
Client                                Server
  |                                     |
  |—— ClientHello ——————————————————>  |
  |   (supported ciphers, key share,    |
  |    TLS version, random nonce)       |
  |                                     |
  |<—— ServerHello ——————————————————  |
  |   (chosen cipher, key share,        |
  |    certificate, Finished)           |
  |                                     |
  |—— Finished ————————————————————>   |
  |                                     |
  |   ← encrypted data flows →         |
```

Key concepts:
- **Certificate**: an X.509 document signed by a Certificate Authority (CA) that proves the server owns the domain. The browser has a built-in list of trusted CAs.
- **Key Exchange**: using Diffie-Hellman Ephemeral (DHE) or Elliptic Curve DHE (ECDHE), both sides derive a shared secret without ever transmitting it — perfect forward secrecy.
- **Cipher Suite**: e.g. `TLS_AES_256_GCM_SHA384` — specifies the encryption algorithm (AES-256-GCM) and hash function (SHA-384).

---

### HTTP — HyperText Transfer Protocol

HTTP is a request/response protocol on top of TCP+TLS.

**Request anatomy:**
```
GET /api/users HTTP/1.1          ← method + path + version
Host: api.example.com            ← required header in HTTP/1.1
Accept: application/json
Authorization: Bearer eyJ...
                                 ← blank line separates headers from body
```

**Response anatomy:**
```
HTTP/1.1 200 OK                  ← version + status code + reason phrase
Content-Type: application/json
Content-Length: 142
Cache-Control: max-age=3600
ETag: "abc123"
                                 ← blank line
{ "users": [...] }               ← body
```

**HTTP Methods:**

| Method  | Idempotent | Safe | Body |
|---------|-----------|------|------|
| GET     | Yes       | Yes  | No (technically allowed, ignored by convention) |
| POST    | No        | No   | Yes |
| PUT     | Yes       | No   | Yes |
| PATCH   | No        | No   | Yes |
| DELETE  | Yes       | No   | Optional |
| HEAD    | Yes       | Yes  | No |
| OPTIONS | Yes       | Yes  | No |

**Status Code classes:**
- `1xx` — Informational (100 Continue, 101 Switching Protocols)
- `2xx` — Success (200 OK, 201 Created, 204 No Content)
- `3xx` — Redirection (301 Moved Permanently, 302 Found, 304 Not Modified)
- `4xx` — Client Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests)
- `5xx` — Server Error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable)

---

## 3. How It Works

### Step-by-step: "What happens when you type google.com"

**Step 1 — Browser cache check**
Before anything happens on the network, the browser checks its own DNS cache and then the OS DNS cache (`/etc/hosts` on Linux/macOS, the Windows hosts file).

**Step 2 — DNS resolution**
```
Browser → OS resolver → Recursive resolver (8.8.8.8)
                           ↓ (cache miss)
                       Root server (.com NS?)
                           ↓
                       TLD server (google.com NS?)
                           ↓
                       Authoritative NS (216.239.32.10)
                           ↓
                       A record: 142.250.80.46
                           ↑
                       Returned and cached (TTL: 300s)
```

**Step 3 — TCP handshake**
With the IP address in hand, the browser opens a TCP socket to port 443 (HTTPS) or 80 (HTTP). The three-way handshake completes in 1 RTT.

**Step 4 — TLS handshake**
ClientHello → ServerHello + Certificate + Finished → Client Finished. TLS 1.3 adds only 1 RTT. Total cost up to this point: 2 RTTs (1 TCP + 1 TLS).

**Step 5 — HTTP request**
The browser sends the GET request including headers like `Accept-Encoding: gzip, br`, cookies, `User-Agent`, and `Referer`.

**Step 6 — Server processing**
The server (nginx / CDN edge → app server → database) builds the response. TTFB (Time to First Byte) is measured here — from the moment the request leaves the browser to the moment the first byte of the response arrives.

**Step 7 — Response transmission**
The server sends headers first, then the body. With HTTP/2, multiple resources can be streamed concurrently over the same connection.

**Step 8 — Browser rendering pipeline**

1. Parse HTML → build **DOM** (Document Object Model) tree
2. Parse CSS → build **CSSOM** (CSS Object Model) tree
3. Execute synchronous JavaScript (blocks parsing unless `defer`/`async`)
4. Combine DOM + CSSOM → **Render Tree** (only visible nodes; `display: none` excluded)
5. **Layout** (a.k.a. reflow): calculate size and position of each node
6. **Paint**: fill in pixels (text, colors, images, borders)
7. **Composite**: combine painted layers on the GPU (transforms, opacity are composited cheaply)

> 💡 JavaScript blocks HTML parsing by default. Use `defer` (runs after parsing) or `async` (runs as soon as downloaded) on `<script>` tags to avoid render-blocking.

---

### HTTP/1.1 vs HTTP/2 vs HTTP/3

**HTTP/1.1 (1997):**
- One request at a time per TCP connection — head-of-line blocking
- Workaround: browsers open 6 parallel TCP connections per domain
- Headers sent as plain text, no compression
- Keep-Alive: connections can be reused across multiple requests, but requests are still serial per connection

**HTTP/2 (2015):**
- **Multiplexing**: multiple requests and responses simultaneously over a single TCP connection, using binary frames and stream IDs
- **Header compression** (HPACK): headers are compressed using a shared dictionary — huge savings on repeated headers like `Cookie` and `User-Agent`
- **Server push**: server can proactively send resources (e.g., CSS, fonts) it knows the client will need — in practice, rarely used well and removed in some implementations
- Still uses TCP underneath — a single lost TCP packet pauses ALL streams (TCP head-of-line blocking)

**HTTP/3 (2022):**
- Runs over **QUIC** (Quick UDP Internet Connections) instead of TCP
- QUIC is built on UDP with reliability, ordering, and congestion control implemented in userspace
- **No head-of-line blocking**: each QUIC stream is independent; a lost packet only blocks its own stream
- **0-RTT resumption**: previously connected clients can send data with the first packet
- **Connection migration**: QUIC connections survive IP address changes (mobile handoffs between WiFi and LTE)

> 💡 HTTP/3 is now supported by all major browsers and CDNs. Cloudflare, Fastly, and AWS CloudFront all support it. Check support with `curl --http3`.

---

### Caching

**Cache-Control directives:**

| Directive | Meaning |
|-----------|---------|
| `max-age=3600` | Cache for 3600 seconds from when it was received |
| `no-cache` | Cache it, but **revalidate** with server before each use |
| `no-store` | Do **not** store at all (sensitive data) |
| `must-revalidate` | Once stale, must revalidate — do not serve stale even if offline |
| `public` | Shared caches (CDN) may cache this |
| `private` | Only browser cache (not CDN) — e.g., personalized pages |
| `immutable` | Content will never change; no revalidation needed |
| `s-maxage=3600` | Like max-age but only for shared caches (CDN); overrides max-age for CDN |
| `stale-while-revalidate=60` | Serve stale content while revalidating in background |

**ETag and Last-Modified (conditional requests):**
```http
# First request — server includes validators
GET /app.js  →  200 OK
ETag: "abc123"
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT

# Subsequent request — browser sends validators
GET /app.js
If-None-Match: "abc123"
If-Modified-Since: Mon, 01 Jan 2024 00:00:00 GMT

# Server: nothing changed
→  304 Not Modified   (no body, browser uses its cached copy)

# Server: file changed
→  200 OK + new ETag  (full response with new content)
```

**CDN caching:**
CDN edge nodes (PoPs — Points of Presence) cache content geographically close to users. The CDN's cache TTL is controlled by `Cache-Control: s-maxage=3600` (overrides `max-age` for shared caches). When a user in São Paulo requests a file cached at a São Paulo PoP, the response comes from ~5ms away instead of a distant origin server.

---

## 4. Code Examples

### curl — inspect headers and timing
```bash
# See only response headers (no body)
curl -I https://google.com

# Follow redirects, show verbose TLS and connection info
curl -vL https://google.com 2>&1 | head -80

# Measure detailed timing breakdown
curl -o /dev/null -s -w "\
DNS lookup:        %{time_namelookup}s\n\
TCP connect:       %{time_connect}s\n\
TLS handshake:     %{time_appconnect}s\n\
TTFB:              %{time_starttransfer}s\n\
Total:             %{time_total}s\n" \
  https://google.com

# Send a JSON POST with auth
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"name": "Alice", "email": "alice@example.com"}'

# Check which HTTP version is being used
curl -vso /dev/null https://cloudflare.com 2>&1 | grep "< HTTP"
```

### Raw HTTP/1.1 request (via netcat)
```bash
# Manually send an HTTP request to see the raw response
printf "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" \
  | nc example.com 80
```

### Cache-Control header examples
```http
# Static assets with content hash in filename (e.g., app.a3f8e1.js)
# Cache for 1 year, never revalidate — file name guarantees freshness
Cache-Control: public, max-age=31536000, immutable

# API responses — personalized, no CDN caching, browser can cache 5 minutes
Cache-Control: private, max-age=300

# HTML pages — browser always revalidates (uses ETag/304 to save bandwidth)
Cache-Control: no-cache

# Sensitive data (sessions, tokens, banking) — never store anywhere
Cache-Control: no-store

# News feed — serve stale while fetching fresh in background
Cache-Control: public, max-age=60, stale-while-revalidate=300

# CDN caches for 1 hour, browser caches for 5 minutes
Cache-Control: public, s-maxage=3600, max-age=300
```

### Minimal HTTP/1.1 server in Node.js (raw TCP)
```javascript
const net = require('net');

const server = net.createServer((socket) => {
  let raw = '';

  socket.on('data', (chunk) => {
    raw += chunk.toString();

    // Wait until we have the full headers (blank line)
    if (!raw.includes('\r\n\r\n')) return;

    const [requestLine] = raw.split('\r\n');
    const [method, path] = requestLine.split(' ');

    let statusCode = 200;
    let body;

    if (path === '/') {
      body = JSON.stringify({ message: 'Hello, World!', method, path });
    } else {
      statusCode = 404;
      body = JSON.stringify({ error: 'Not Found', path });
    }

    const response = [
      `HTTP/1.1 ${statusCode} ${statusCode === 200 ? 'OK' : 'Not Found'}`,
      'Content-Type: application/json',
      `Content-Length: ${Buffer.byteLength(body)}`,
      'Connection: close',
      '',
      body,
    ].join('\r\n');

    socket.write(response);
    socket.end();
  });
});

server.listen(3000, () => {
  console.log('Raw HTTP server on http://localhost:3000');
  console.log('Try: curl http://localhost:3000/');
  console.log('Try: curl http://localhost:3000/unknown');
});
```

### CORS headers
```http
# Browser preflight for cross-origin POST
OPTIONS /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type

# Server response — grants permission
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Confusing `no-cache` and `no-store`**: `no-cache` means "you CAN store it, but you MUST revalidate before serving". `no-store` means "do not store anywhere". Many developers use `no-cache` when they mean `no-store` for sensitive data, accidentally caching passwords and tokens in intermediate caches.

> ⚠️ **Assuming DNS is instant**: DNS lookups can take 20–200ms. TTL caching helps, but the first visitor after TTL expiry always pays the full lookup time. Solution: use `<link rel="dns-prefetch">` and `<link rel="preconnect">` hints in your HTML for third-party origins.

> ⚠️ **Treating HTTP and HTTPS as equivalent**: HTTPS is not just "encrypted HTTP". It changes cookie semantics (`Secure` flag only works on HTTPS), enables certain browser APIs (geolocation, Service Workers, Web Bluetooth only available on secure origins), and affects SEO ranking. Never serve mixed content (HTTP resources on an HTTPS page — modern browsers block it silently).

> ⚠️ **Misunderstanding 301 vs 302**: 301 (Moved Permanently) is cached by browsers and search engines indefinitely, passing SEO link equity to the new URL. 302 (Found / Temporary Redirect) is not cached and does not transfer SEO value. Using 301 incorrectly for a temporary redirect means you cannot undo it easily — the browser ignores it for months.

> ⚠️ **Ignoring head-of-line blocking in HTTP/2**: Developers sometimes think HTTP/2 completely solves all latency issues. A single lost TCP packet still pauses all streams on that connection. HTTP/3 over QUIC solves this. Also, server push was removed from Chrome in 2022 — don't rely on it.

> ⚠️ **Setting `max-age` without content hashing**: If you cache `app.js` for 1 year but deploy a new version at the same URL, users are stuck with the old file until their cache expires. Solution: include a content hash in the filename (`app.a3f8e1b2.js`) and set `immutable`. All modern bundlers (Webpack, Vite, esbuild) do this by default.

---

## 6. When to Use / Not Use

**Use aggressive caching (`max-age=31536000, immutable`):**
- Static assets with content hashes in their filename
- Fonts, vendor bundles, images

**Use `no-cache` (always revalidate, use 304 for bandwidth):**
- HTML pages (so users always get a fresh shell that references the latest hashed assets)
- API responses that might change

**Use `no-store`:**
- Responses containing sensitive data (banking, healthcare, auth tokens, PII)

**Use `private`:**
- Personalized content that CDNs should never cache (user dashboards, account pages)

**Use `stale-while-revalidate`:**
- Feeds, dashboards, leaderboards — tolerate short staleness for speed

**Use `s-maxage`:**
- When CDN TTL should differ from browser TTL (longer CDN cache, shorter browser cache)

---

## 7. Real-World Scenario

### The "stale JS after deploy" bug

**Situation:** You deploy a new version of your single-page app. Some users continue seeing the old JavaScript bundle and experience a broken UI mixing old and new code structures.

**Investigation:**
```bash
# Check what cache headers the CDN is returning
curl -I https://yourapp.com/assets/app.js

# HTTP/2 200 OK
# cache-control: public, max-age=86400
# etag: "old-hash-abc123"
# age: 43210
# x-cache: HIT
```

The file is being served from CDN cache with a 24-hour TTL. You deployed a new file at the same URL path, but the CDN will not fetch the new version for another ~11 hours (86400 - 43210 = ~43190 seconds).

**Root cause:** Your build outputs `app.js` (no content hash in filename), and you set a 24-hour CDN cache without a way to invalidate.

**Fix 1 — Content-hash filenames + immutable caching (correct long-term solution):**

Configure your bundler to output hashed filenames:
```javascript
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash][extname]',
      },
    },
  },
};
```

Then set cache headers:
```nginx
# nginx — HTML pages always revalidate
location ~* \.html$ {
  add_header Cache-Control "no-cache";
}

# JS/CSS/images with hash in name — cache 1 year, immutable
location ~* \.[0-9a-f]{8}\.(js|css|woff2|png|webp)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}
```

**Fix 2 — CDN cache purge on deploy (emergency fix or legacy systems):**
```bash
# Cloudflare — purge specific file after deploy
curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/purge_cache" \
  -H "Authorization: Bearer ${CF_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{"files":["https://yourapp.com/app.js"]}'

# Or purge everything (blunt but effective)
--data '{"purge_everything":true}'
```

**The correct architecture:**
```
index.html      →  Cache-Control: no-cache
app.a3f8e1.js   →  Cache-Control: public, max-age=31536000, immutable
vendor.8c2d1f.js→  Cache-Control: public, max-age=31536000, immutable
style.4b9e2a.css→  Cache-Control: public, max-age=31536000, immutable
/api/*          →  Cache-Control: private, no-store
```

---

## 8. Interview Questions

**Q1: Walk me through everything that happens when you type a URL and press Enter.**

A: Browser checks its own DNS cache, then the OS cache and `/etc/hosts`. On a cache miss, the OS queries the configured recursive resolver (e.g., 8.8.8.8) via UDP. The resolver walks the DNS hierarchy: root servers → TLD name servers → authoritative name server for the domain, returning the A/AAAA record and caching it per TTL. With the IP, the browser opens a TCP socket via a three-way handshake (SYN → SYN-ACK → ACK), costing 1 RTT. For HTTPS, a TLS 1.3 handshake follows (ClientHello, ServerHello + certificate + Finished, client Finished), costing another 1 RTT. The browser sends the HTTP GET request. The server processes it and returns a response. The browser parses the HTML into a DOM, parses CSS into a CSSOM, executes JavaScript, merges DOM+CSSOM into a render tree, runs layout to calculate positions, paints pixels, and composites layers on the GPU.

---

**Q2: What is the TLS handshake and why does it matter?**

A: TLS provides encryption, authentication, and integrity for the connection. In TLS 1.3, the handshake is 1 RTT: the client sends a ClientHello with supported cipher suites and a Diffie-Hellman public key share. The server responds with its chosen cipher, its DH key share, a certificate signed by a trusted CA (proving its identity), and a Finished message. Both sides independently compute the same shared secret from the DH exchange — no secret is ever transmitted. From that point, all data is encrypted. It matters because without TLS, any network intermediary (ISP, coffee shop router, government) can read or silently modify traffic.

---

**Q3: What is the difference between HTTP/2 and HTTP/1.1?**

A: HTTP/2 uses binary framing instead of plain text, enabling multiplexing — multiple concurrent request/response streams over a single TCP connection, eliminating the serial-request limitation of HTTP/1.1. It also adds HPACK header compression (huge savings for repeated headers like cookies) and optional server push. HTTP/1.1 required hacks like domain sharding and resource bundling to compensate for its per-connection serialization. The key limitation HTTP/2 does not fix is TCP head-of-line blocking — a single lost packet stalls all streams. HTTP/3 + QUIC solves this by using UDP with independent per-stream reliability.

---

**Q4: What does `Cache-Control: no-cache` mean exactly?**

A: It means the response CAN be stored in cache, but the cache MUST revalidate with the origin server before serving it on subsequent requests. If the server's ETag or Last-Modified indicates nothing changed, it returns 304 Not Modified (no body) and the cached version is served — saving bandwidth without sacrificing freshness. It does NOT mean "do not cache". If you want to prevent storage entirely (for sensitive data), use `Cache-Control: no-store`.

---

**Q5: What is a CDN and how does it work?**

A: A Content Delivery Network is a globally distributed network of servers called edge nodes or Points of Presence (PoPs). When a user requests a resource, DNS or Anycast routing directs them to the geographically nearest edge node. If the edge has the content cached (because the response had `Cache-Control: public`), it responds immediately without reaching your origin server — reducing both latency (geographically closer) and origin load. When the cache is cold or stale, the edge fetches from origin, stores the response, and serves future requests from cache.

---

**Q6: What is CORS and why does it exist?**

A: CORS (Cross-Origin Resource Sharing) is a browser mechanism that enforces the Same-Origin Policy: JavaScript can only read responses from the same origin (scheme + hostname + port) as the page, unless the server explicitly permits it. It exists to prevent malicious scripts on `evil.com` from silently reading authenticated responses from `yourbank.com` using the victim's cookies. CORS allows servers to whitelist specific origins via `Access-Control-Allow-Origin`. The restriction is browser-only — server-to-server requests and curl are never subject to CORS.

---

**Q7: What is the difference between a 301 and 302 redirect?**

A: 301 Moved Permanently — both browsers and search engines cache this redirect indefinitely and transfer SEO "link equity" to the destination URL. Once cached, the browser skips the redirect entirely and goes directly to the new URL. 302 Found (Temporary Redirect) — browsers do not cache it; search engines keep the original URL indexed. Use 301 when permanently moving content (domain migration, URL restructuring). Use 302 for temporary situations (A/B testing, maintenance pages). Also relevant: 307 (Temporary Redirect) and 308 (Permanent Redirect) preserve the original HTTP method (a POST stays a POST), while 301/302 typically convert POST to GET.

---

**Q8: What is DNS TTL and how does it affect deployments?**

A: TTL (Time To Live) is the number of seconds a DNS record should be cached by resolvers before they re-query. A TTL of 3600 means your domain's IP address change may not be visible for up to 1 hour after you update it, because all resolvers that already cached it will keep using the old value. For migrations: lower TTL to 60 seconds at least 24 hours before the migration (to let existing long-TTL caches expire), perform the DNS change, verify it works, then raise TTL back to normal (3600+). After changes, use `dig @8.8.8.8 example.com` to check propagation without hitting your local cache.

---

## 9. Exercises

**Exercise 1 — Trace a full request with curl:**
```bash
# Measure the timing breakdown for several sites and compare
for site in github.com google.com cloudflare.com; do
  echo "=== $site ==="
  curl -o /dev/null -s -w "\
  DNS:   %{time_namelookup}s\n\
  TCP:   %{time_connect}s\n\
  TLS:   %{time_appconnect}s\n\
  TTFB:  %{time_starttransfer}s\n\
  Total: %{time_total}s\n\n" \
    "https://$site"
done
```

Goals:
1. Use `curl -I` to see only response headers — identify Cache-Control, ETag, and Content-Type
2. Use `curl -v` to see the TLS certificate chain and the exact TLS version used
3. Compare `--http1.1` vs `--http2` for the same URL — observe header differences
4. Find a site serving HTTP/3 and use `curl --http3` if your curl supports it

---

**Exercise 2 — Implement a minimal HTTP/1.1 server in Node.js:**

Build a raw TCP server (using the `net` module, not `http`) that:
- Parses the first line of an HTTP request to extract method and path
- Returns `200 OK` with JSON body `{ "path": "..." }` for `GET /`
- Returns `404 Not Found` with JSON error for unknown paths
- Returns `405 Method Not Allowed` for non-GET requests
- Correctly sets `Content-Length` header (not Content-Transfer-Encoding)
- Can be tested end-to-end with `curl http://localhost:3000/`

Hint: HTTP headers and body are separated by `\r\n\r\n`. The request line is `METHOD PATH HTTP/1.1`. All line endings in HTTP are `\r\n`, not just `\n`.

---

**Exercise 3 — Analyze a waterfall in Chrome DevTools:**

1. Open Chrome DevTools → Network tab, check "Disable cache", reload
2. Visit a content-heavy page (a news site or e-commerce homepage)
3. Answer these questions:
   - What is the TTFB for the HTML document?
   - Are there any render-blocking resources (look for long "Waiting" bars on CSS/JS before first paint)?
   - Which files are served with content-hash filenames vs generic names (`app.js` vs `app.a3f8e1.js`)?
   - Are images optimized as WebP or AVIF, or still JPEG/PNG?
   - Does the site use HTTP/2 or HTTP/3? (Protocol column)
4. Enable cache and reload — which resources return 304? Which return from disk/memory cache?

---

## 10. Further Reading

- **MDN — HTTP**: https://developer.mozilla.org/en-US/docs/Web/HTTP — comprehensive reference for all HTTP concepts
- **"High Performance Browser Networking" by Ilya Grigorik** (free online): https://hpbn.co — the definitive book on TCP, TLS, HTTP/1.1, HTTP/2, WebSockets, and WebRTC
- **RFC 9114 — HTTP/3**: https://www.rfc-editor.org/rfc/rfc9114
- **"How Browsers Work"**: https://web.dev/articles/howbrowserswork — detailed breakdown of the rendering pipeline
- **web.dev — Core Web Vitals**: https://web.dev/vitals/ — Google's performance metrics framework (LCP, INP, CLS)
- **Cloudflare Learning Center**: https://www.cloudflare.com/learning/ — excellent DNS, CDN, and TLS explanations with diagrams
- **"What happens when..."** (GitHub): https://github.com/alex/what-happens-when — exhaustive, community-maintained deep dive into every layer
- **Google's HTTP/2 guide**: https://developers.google.com/web/fundamentals/performance/http2
- **dig command reference**: `man dig` — use `dig +trace example.com` to see the full DNS resolution walk
