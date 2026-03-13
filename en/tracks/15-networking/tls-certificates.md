# TLS & Certificates

TLS (Transport Layer Security) provides confidentiality, integrity, and authentication for network communication. It's the "S" in HTTPS.

## Why TLS Matters

Without TLS, anyone on the network path can:
- **Read** your data (passwords, tokens, PII)
- **Modify** your data in transit (inject content, tamper responses)
- **Impersonate** servers (serve you a fake bank site)

TLS addresses all three through encryption, MACs, and certificate authentication.

## TLS Handshake (TLS 1.3)

TLS 1.3 completes in 1 RTT (vs 2 RTTs in TLS 1.2):

```
Client                          Server
  │                                │
  │──── ClientHello ──────────────►│
  │     (supported ciphers,        │
  │      key share, TLS version)   │
  │                                │
  │◄─── ServerHello ───────────────│
  │     (chosen cipher,            │
  │      key share,                │
  │      Certificate,              │
  │      CertificateVerify,        │
  │      Finished)                 │
  │                                │
  │──── Finished ─────────────────►│
  │                                │
  │    [Encrypted data flow]       │
```

### Key Exchange (ECDHE)

TLS 1.3 uses only **Ephemeral Diffie-Hellman** for key exchange:

```
1. Client generates: private key a, public key A = a×G
2. Server generates: private key b, public key B = b×G
3. Both share public keys
4. Client computes: S = a×B
5. Server computes: S = b×A
6. S is the same (shared secret) — never sent over the wire

Session keys derived from S using HKDF
```

**Forward secrecy**: each session uses ephemeral keys. Compromising the server's private key doesn't decrypt past sessions.

## PKI — Public Key Infrastructure

PKI is the system of trust that answers: "How do I know this certificate belongs to example.com?"

### Certificate Chain

```
Root CA (self-signed, pre-installed in OS/browser)
  └── Intermediate CA (signed by Root)
        └── Server Certificate (signed by Intermediate)
              example.com, valid 2025-01-01 to 2026-01-01
```

Browsers trust a fixed set of Root CAs (Mozilla, Apple, Microsoft each maintain lists). Your certificate is trusted if it chains to a trusted root.

### X.509 Certificate Contents

```
Subject:      CN=example.com, O=Example Inc, C=US
Issuer:       CN=Let's Encrypt R11, O=Let's Encrypt
Valid:        2025-01-01 to 2025-04-01 (90 days)
SANs:         example.com, www.example.com
Public Key:   EC 256-bit (ECDSA)
Signature:    SHA256withECDSA by issuer

Extensions:
  Key Usage: Digital Signature
  Extended Key Usage: TLS Web Server Authentication
  OCSP: http://ocsp.lencr.org/
```

### Certificate Validation Steps

```
1. Chain validation: server cert → intermediate → root (trusted?)
2. Expiry: is current date within validity period?
3. Revocation: check OCSP or CRL (is cert revoked?)
4. Hostname: does cert's CN/SANs match requested hostname?
5. Key usage: is cert allowed for TLS server auth?
```

### SANs — Subject Alternative Names

```
A certificate can cover multiple hostnames:
SANs: example.com, www.example.com, api.example.com

Wildcard: *.example.com (covers one level deep only)
  Covers: api.example.com, www.example.com
  Doesn't cover: sub.api.example.com
```

## Let's Encrypt & ACME

Let's Encrypt provides free, automated certificates via the ACME protocol.

```bash
# HTTP-01 challenge: prove you control the domain
# ACME places a token at /.well-known/acme-challenge/<token>
# Let's Encrypt fetches it over HTTP to verify

# Using Certbot:
certbot certonly --webroot -w /var/www/html -d example.com

# Auto-renew (certificates last 90 days):
certbot renew --quiet

# Or use Traefik/Caddy — they handle ACME automatically
```

## Certificate Inspection

```bash
# View certificate details
openssl s_client -connect example.com:443 -servername example.com </dev/null \
  | openssl x509 -noout -text

# Check expiry date
echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -dates

# View cert in browser: click padlock → Certificate → Details

# Check TLS version and cipher suite negotiated
curl -v https://example.com 2>&1 | grep -E 'SSL|TLS|cipher'
```

## mTLS — Mutual TLS

In standard TLS, only the server presents a certificate. In mTLS, **both** client and server authenticate:

```
Client                          Server
  │──── ClientHello ─────────────►│
  │◄─── ServerHello + Cert ───────│
  │──── Client Cert + Verify ────►│  ← new: client proves identity
  │◄─── Finished ─────────────────│
```

### mTLS Use Cases

```
Service-to-service in microservices (zero-trust networking)
API clients (instead of API keys)
VPN authentication
IoT device authentication

Example: Kubernetes uses mTLS between pods via service meshes
(Istio, Linkerd) so all pod-to-pod communication is encrypted
and authenticated without application code changes.
```

```bash
# Call API with client certificate
curl --cert client.crt --key client.key \
     --cacert ca.crt \
     https://api.internal/endpoint
```

## Common TLS Issues

| Error | Cause | Fix |
|-------|-------|-----|
| `certificate has expired` | Cert past validity | Renew certificate |
| `certificate is not trusted` | Unknown CA | Install CA cert, use trusted CA |
| `hostname mismatch` | CN/SANs don't match | Get cert with correct SANs |
| `self-signed certificate` | No CA chain | Use Let's Encrypt or real CA |
| `SSL_ERROR_RX_RECORD_TOO_LONG` | HTTP port used for HTTPS | Fix port/protocol |
| `handshake failure` | Cipher suite mismatch | Update TLS config |

## TLS Best Practices

```nginx
# Nginx TLS config (modern)
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:
            ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_stapling on;
ssl_stapling_verify on;
add_header Strict-Transport-Security "max-age=63072000" always;
```

## Practice

1. Run `openssl s_client -connect google.com:443` — identify the TLS version and certificate chain.
2. Find a certificate using Let's Encrypt on any public site. Check its validity period.
3. Explain what forward secrecy means and why it matters if a server's private key is leaked.
