# HTTP Status Codes Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## 1xx — Informational

| Code | Name | Use Case |
|------|------|---------|
| 100 | Continue | Server received headers, client should proceed with body |
| 101 | Switching Protocols | Upgrading to WebSocket or HTTP/2 |
| 102 | Processing | Server received request, still processing (WebDAV) |
| 103 | Early Hints | Server sends preliminary headers while preparing response |

---

## 2xx — Success

| Code | Name | Use Case |
|------|------|---------|
| **200** | OK | Standard successful response (GET, PUT, PATCH) |
| **201** | Created | Resource created (POST with new resource; include `Location` header) |
| **202** | Accepted | Request accepted for async processing (queued jobs) |
| **204** | No Content | Success with no body (DELETE, PUT with no return) |
| 205 | Reset Content | Client should reset form/view |
| 206 | Partial Content | Response to `Range` header request (file streaming, resumable downloads) |
| 207 | Multi-Status | Multiple status codes for multiple operations (WebDAV) |
| 208 | Already Reported | Member of collection already enumerated (WebDAV) |
| 226 | IM Used | Response is result of delta encoding |

**API patterns:**
```
POST   /users       → 201 Created  + Location: /users/42
GET    /users/42    → 200 OK       + body
PATCH  /users/42    → 200 OK       + body (or 204 if no body)
DELETE /users/42    → 204 No Content
POST   /jobs        → 202 Accepted + { jobId: "..." }
```

---

## 3xx — Redirection

| Code | Name | Use Case |
|------|------|---------|
| 300 | Multiple Choices | Multiple representations available |
| **301** | Moved Permanently | Resource permanently at new URL; update bookmarks |
| **302** | Found | Temporary redirect (avoid — use 303 or 307) |
| **303** | See Other | After POST, redirect to GET (Post/Redirect/Get pattern) |
| 304 | Not Modified | Cached version is fresh; no body sent (ETag / If-Modified-Since) |
| 307 | Temporary Redirect | Temporary redirect; method must not change |
| **308** | Permanent Redirect | Permanent redirect; method must not change |

**Redirect guidance:**

| Scenario | Use |
|----------|-----|
| Domain change, permanent | 301 |
| HTTP → HTTPS, permanent | 301 |
| After form POST | 303 |
| API endpoint renamed, permanent | 308 |
| Maintenance redirect, temporary | 307 |
| Cached response still valid | 304 |

---

## 4xx — Client Errors

| Code | Name | Use Case |
|------|------|---------|
| **400** | Bad Request | Malformed syntax, invalid body, failed validation |
| **401** | Unauthorized | Authentication required or credentials invalid (include `WWW-Authenticate`) |
| **403** | Forbidden | Authenticated but not authorized for this resource |
| **404** | Not Found | Resource does not exist (also used to hide existence of forbidden resources) |
| **405** | Method Not Allowed | HTTP method not supported on this endpoint (include `Allow` header) |
| 406 | Not Acceptable | Server can't produce response matching `Accept` header |
| 407 | Proxy Authentication Required | Must authenticate with proxy |
| **408** | Request Timeout | Client took too long to send request |
| **409** | Conflict | State conflict — duplicate resource, optimistic lock failure |
| 410 | Gone | Resource permanently deleted (stronger than 404) |
| 411 | Length Required | `Content-Length` header required |
| 412 | Precondition Failed | Conditional request precondition not met (ETag mismatch on PUT) |
| 413 | Content Too Large | Request body too large |
| 414 | URI Too Long | URL too long |
| 415 | Unsupported Media Type | Content-Type not supported |
| **416** | Range Not Satisfiable | Requested range not available |
| 417 | Expectation Failed | `Expect` header cannot be met |
| **418** | I'm a Teapot | Easter egg (RFC 2324) — sometimes used for intentionally refused requests |
| 421 | Misdirected Request | Request directed at server unable to respond |
| **422** | Unprocessable Entity | Body syntactically correct but semantically invalid (preferred for validation errors) |
| 423 | Locked | Resource is locked (WebDAV) |
| 424 | Failed Dependency | Operation failed because dependent operation failed |
| 425 | Too Early | Replay attack risk (Early Data) |
| **426** | Upgrade Required | Client must upgrade protocol (e.g., TLS) |
| **429** | Too Many Requests | Rate limit exceeded (include `Retry-After` header) |
| 431 | Request Header Fields Too Large | Headers too large |
| 451 | Unavailable For Legal Reasons | Censored for legal reasons |

**Common 4xx decision tree:**
```
Missing/invalid credentials?   → 401
Credentials valid, no access?  → 403
Resource not found?            → 404
Duplicate / conflict?          → 409
Validation failed?             → 422 (preferred) or 400
Rate limited?                  → 429 + Retry-After
Wrong HTTP method?             → 405 + Allow header
```

---

## 5xx — Server Errors

| Code | Name | Use Case |
|------|------|---------|
| **500** | Internal Server Error | Unhandled exception, bug — generic server failure |
| **501** | Not Implemented | Method or feature not implemented yet |
| **502** | Bad Gateway | Upstream server returned invalid response (reverse proxy issue) |
| **503** | Service Unavailable | Server overloaded or down for maintenance (include `Retry-After`) |
| **504** | Gateway Timeout | Upstream server did not respond in time |
| 505 | HTTP Version Not Supported | HTTP version in request not supported |
| 506 | Variant Also Negotiates | Configuration error in content negotiation |
| 507 | Insufficient Storage | Server out of storage (WebDAV) |
| 508 | Loop Detected | Infinite loop detected (WebDAV) |
| 510 | Not Extended | Further extensions required |
| 511 | Network Authentication Required | Must authenticate to gain network access (captive portals) |

**5xx operational notes:**
- 500 → fix the bug; never leak stack traces
- 502/504 → check upstream service health and timeouts
- 503 → add retry logic in clients + return `Retry-After`

---

## Headers Commonly Paired with Status Codes

| Status | Common Headers |
|--------|---------------|
| 201 | `Location: /resources/42` |
| 204 | (no body, no Content-Type) |
| 301, 302, 307, 308 | `Location: https://new-url.com` |
| 304 | `ETag`, `Last-Modified`, `Cache-Control` |
| 401 | `WWW-Authenticate: Bearer` |
| 405 | `Allow: GET, POST` |
| 429 | `Retry-After: 60` (seconds) |
| 503 | `Retry-After: 120` |

---

## REST API Design Conventions

```
GET    /resources          → 200 OK         (list)
GET    /resources/:id      → 200 OK         (single)
                           → 404 Not Found  (missing)
POST   /resources          → 201 Created    (new resource)
                           → 400/422        (invalid body)
                           → 409 Conflict   (duplicate)
PUT    /resources/:id      → 200 OK         (replaced, return body)
                           → 204 No Content (replaced, no body)
                           → 404 Not Found
PATCH  /resources/:id      → 200 OK
                           → 422            (validation error)
DELETE /resources/:id      → 204 No Content (success)
                           → 404 Not Found
                           → 409 Conflict   (cannot delete — in use)
```

---

## Caching & Conditional Requests

| Header | Direction | Purpose |
|--------|-----------|---------|
| `Cache-Control: no-store` | Response | Never cache |
| `Cache-Control: no-cache` | Response | Cache but revalidate each time |
| `Cache-Control: max-age=3600` | Response | Cache for 1 hour |
| `Cache-Control: s-maxage=3600` | Response | Shared cache (CDN) TTL |
| `ETag: "abc123"` | Response | Resource version fingerprint |
| `Last-Modified: Thu, 01 Jan 2025 00:00:00 GMT` | Response | Last change time |
| `If-None-Match: "abc123"` | Request | Return 304 if ETag matches |
| `If-Modified-Since: ...` | Request | Return 304 if not modified |
| `Vary: Accept-Encoding` | Response | Cache separately per encoding |

---

## WebSocket Upgrade

```
Client: GET /ws HTTP/1.1
        Upgrade: websocket
        Connection: Upgrade

Server: HTTP/1.1 101 Switching Protocols
        Upgrade: websocket
        Connection: Upgrade
        Sec-WebSocket-Accept: <hash>
```
