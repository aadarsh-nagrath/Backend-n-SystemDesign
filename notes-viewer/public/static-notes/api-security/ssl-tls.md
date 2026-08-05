## SSL/TLS: Complete Guide for Backend Engineers

This guide explains SSL/TLS end to end: concepts, how it works, why it matters, certificate types and validation, modern TLS versions and cipher suites, mutual TLS, SNI/ECH, ALPN and HTTP/2/3, configuration examples for Nginx and Apache, automation with ACME/Certbot, hardening, troubleshooting, and best practices.

### What is SSL? What is TLS?
- **SSL (Secure Sockets Layer)** is the original web encryption protocol from the 1990s. It is deprecated and insecure (SSLv2/SSLv3 must not be used).
- **TLS (Transport Layer Security)** is the modern successor. Today “SSL” is commonly used as shorthand for TLS, but implementations should use TLS 1.2+ (prefer TLS 1.3).
- Websites using TLS advertise `https://` and the browser shows a lock icon.

### HTTP vs HTTPS
- **HTTP** transmits data in plaintext; eavesdroppers can read or modify it.
- **HTTPS (HTTP over TLS)** encrypts and authenticates data in transit, preventing interception and tampering, and authenticating the server’s identity.

### Why SSL/TLS is Important
- **Confidentiality**: Encryption prevents eavesdropping (credit cards, credentials, PII).
- **Integrity**: Detects/blocks tampering (man-in-the-middle, proxies changing content).
- **Authentication**: Certificates bind a domain to a public key; clients verify server identity.
- **Trust/UI**: Browsers flag non‑HTTPS as "Not Secure"; some APIs require HTTPS only.

#### Why Most Websites Require HTTPS Today
Modern web security demands HTTPS for several reasons:
- **Browser enforcement**: Browsers flag HTTP sites as "Not Secure" and block many features (geolocation, camera, etc.) on HTTP
- **SEO impact**: Search engines prioritize HTTPS sites in rankings
- **API requirements**: Most modern APIs (REST, GraphQL) require HTTPS for security
- **Regulatory compliance**: PCI DSS, GDPR, and other standards mandate encryption
- **User trust**: Users expect the lock icon and green address bar

---

### How TLS Works (High Level)
TLS provides:
- Encryption using symmetric keys (fast) negotiated securely.
- Authentication using X.509 certificates and a public key infrastructure (PKI).
- Integrity via MACs or AEAD modes (e.g., AES‑GCM, ChaCha20‑Poly1305).

#### TLS 1.2 Handshake (simplified)
1) ClientHello → supported TLS versions, cipher suites, random, SNI, extensions
2) ServerHello → chosen version/cipher, random; Certificate; (ServerKeyExchange if needed); ServerHelloDone
3) Client verifies certificate chain (CA trust, hostname, validity, revocation)
4) Key exchange (e.g., ECDHE) to derive shared secret
5) Both sides generate symmetric keys; Finished messages verify handshake integrity

#### TLS 1.3 Handshake (faster, simpler)
- 1‑RTT by default; supports 0‑RTT (replay‑risky; disable for state‑changing requests)
- Removes legacy ciphers; uses AEAD only (AES‑GCM or ChaCha20‑Poly1305)
- Always forward‑secret (ECDHE)

#### Perfect Forward Secrecy (PFS)
- Achieved via ephemeral Diffie‑Hellman (ECDHE). Protects past sessions if the server private key is later compromised.

#### Detailed TLS Handshake Walkthrough (TLS 1.2)
Let's break down the TLS handshake step by step to understand how secure communication is established:

**Step 1: TCP Connection**
- Browser establishes a TCP connection with the server (just like HTTP)
- This is the foundation for the TLS handshake

**Step 2: Client Hello**
- Client sends a "ClientHello" message containing:
  - Supported TLS versions (1.2, 1.3, etc.)
  - Supported cipher suites (encryption algorithms)
  - Random data for security
  - Server Name Indication (SNI) for virtual hosting
  - Extensions (ALPN for HTTP/2, etc.)

**Step 3: Server Hello**
- Server responds with "ServerHello" containing:
  - Chosen TLS version and cipher suite
  - Server's random data
- Server sends its certificate containing:
  - Public key for the server
  - Domain name and validity period
  - Digital signature from Certificate Authority
- Server may send additional messages (ServerKeyExchange, CertificateRequest)

**Step 4: Client Certificate Verification**
- Client verifies the server's certificate:
  - Checks if it's signed by a trusted CA
  - Validates the domain name matches
  - Checks expiration date
  - Verifies the certificate chain

**Step 5: Key Exchange**
- Client generates a random session key (symmetric encryption key)
- Client encrypts this session key using the server's public key
- Client sends the encrypted session key to the server
- Server decrypts the session key using its private key
- Now both client and server have the same session key

**Step 6: Secure Communication**
- Both sides use the session key and agreed cipher suite
- All subsequent data is encrypted with this shared key
- Communication is now secure and bidirectional

**Why Asymmetric + Symmetric Encryption?**
- **Asymmetric encryption** (RSA/ECDSA): Secure but computationally expensive
- **Symmetric encryption** (AES): Fast but requires a shared secret
- **Solution**: Use asymmetric encryption to securely exchange a symmetric key, then use symmetric encryption for all data

**TLS 1.3 Improvements**
- Reduces handshake from 2 round trips to 1 (faster)
- Removes RSA key exchange (uses only ECDHE for forward secrecy)
- Eliminates many legacy cipher suites
- Supports 0-RTT for even faster subsequent connections

---

### Public Key Infrastructure (PKI) Basics
- **Certificates (X.509)** bind identities (CN/SAN) to public keys.
- **Certificate Authorities (CAs)** issue certs; browsers/OS maintain a trust store of roots.
- **Chain of Trust**: leaf (your site) → intermediates → root (trusted by client).
- **Validation**: clients check expiration, hostname, signature chain, and optionally revocation.
- **Revocation**: OCSP/CRL; OCSP stapling improves privacy and performance.

---

### Certificate Types
- **By scope**:
  - Single‑domain: one FQDN (e.g., `example.com` or `www.example.com`).
  - Wildcard: one domain plus all first‑level subdomains (e.g., `*.example.com`).
  - Multi‑domain (SAN/UCC): multiple FQDNs in one certificate.
- **By validation level**:
  - DV (Domain Validation): proves control of the domain (DNS/HTTP/Email). Fast, common.
  - OV (Organization Validation): CA verifies organization details. Higher assurance.
  - EV (Extended Validation): stringent vetting; distinct UI signals have largely diminished.

Key algorithms: RSA (2048/3072+), ECDSA (P‑256/P‑384; smaller, faster signatures). Many sites deploy both via dual‑certs for broad client compatibility.

---

### Modern TLS Versions and Cipher Suites
- Disable SSLv2, SSLv3, TLS 1.0, TLS 1.1. Support TLS 1.2 and TLS 1.3.
- Prefer AEAD cipher suites: AES‑GCM or ChaCha20‑Poly1305.
- Prioritize ECDHE for PFS. Offer both ECDSA and RSA certificates if possible.

Example hardened policy reference: see Mozilla SSL/TLS guidelines (intermediate/modern configs).

---

### Advanced Concepts
- **SNI (Server Name Indication)**: Allows multiple HTTPS sites per IP. Essential for virtual hosting.
- **ECH (Encrypted ClientHello)**: Encrypts SNI to protect the requested hostname from on‑path observers (emerging support).
- **ALPN (Application‑Layer Protocol Negotiation)**: Negotiates HTTP/2 (`h2`) or HTTP/3 (`h3`).
- **HTTP/2**: Multiplexed streams over one TCP+TLS connection; lower latency.
- **HTTP/3 (QUIC)**: Runs over UDP with TLS 1.3; faster connection setup and loss recovery.
- **HSTS**: Strict‑Transport‑Security header enforces HTTPS and blocks downgrade/stripping.
- **mTLS (Mutual TLS)**: Clients present certificates for strong client authentication.
- **0‑RTT**: TLS 1.3 early data; disable for non‑idempotent requests due to replay risk.

---

### Config Examples

#### Nginx (TLS 1.2/1.3, HTTP/2, HSTS)
```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:EECDH+AESGCM';
    ssl_prefer_server_ciphers on;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_stapling on;
    ssl_stapling_verify on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

Mutual TLS (Nginx):
```nginx
ssl_verify_client on;                         # or optional
ssl_client_certificate /etc/nginx/ca_chain.pem;
```

HTTP/3 (QUIC) note (requires QUIC build):
```nginx
server {
    listen 443 http3 reuseport;
    listen 443 ssl http2;
    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

#### Apache httpd (TLS + HSTS + HTTP/2)
```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
  ServerName example.com
  Protocols h2 http/1.1
  SSLEngine on
  SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

  SSLOpenSSLConfCmd Protocol "-ALL, TLSv1.2, TLSv1.3"
  SSLOpenSSLConfCmd Curves X25519:P-256:P-384
  SSLOpenSSLConfCmd ECDHParameters Automatic

  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
  Header set X-Content-Type-Options "nosniff"
  Header set X-Frame-Options "DENY"

  ProxyPass        "/"  "http://127.0.0.1:3000/"
  ProxyPassReverse "/"  "http://127.0.0.1:3000/"
</VirtualHost>
</IfModule>

<VirtualHost *:80>
  ServerName example.com
  Redirect permanent "/" "https://example.com/"
</VirtualHost>
```

Mutual TLS (Apache):
```apache
SSLVerifyClient require
SSLCACertificateFile /etc/apache2/ca_chain.pem
```

---

### Automation: ACME and Let’s Encrypt
- Use ACME clients to obtain and renew certificates automatically.
- **Certbot (Debian/Ubuntu)**:
```bash
sudo apt install -y certbot python3-certbot-nginx   # or python3-certbot-apache
sudo certbot --nginx   -d example.com -d www.example.com --redirect --hsts --agree-tos -m admin@example.com --non-interactive
# or
sudo certbot --apache  -d example.com -d www.example.com --redirect --hsts --agree-tos -m admin@example.com --non-interactive
```
- **DNS‑01 challenges** for wildcards/complex setups; use DNS plugins (e.g., Cloudflare, Route53).
- Ensure renewal timers run and reload servers on renewal (`--deploy-hook`).

---

### Security Headers Complementing TLS
- `Strict-Transport-Security` (HSTS): enforce HTTPS.
- `X-Content-Type-Options: nosniff`: mitigate MIME sniffing.
- `X-Frame-Options: DENY` or CSP `frame-ancestors`.
- `Referrer-Policy: strict-origin-when-cross-origin`.
- `Content-Security-Policy`: control allowed sources; prevents many XSS vectors.

---

### Operational Hardening Checklist
- Disable SSLv2, SSLv3, TLS 1.0, TLS 1.1.
- Prefer TLS 1.3, support TLS 1.2 for compatibility.
- Use modern AEAD ciphers; enable ECDHE for PFS.
- Use strong keys: RSA 2048+ or ECDSA P‑256/P‑384.
- Enable OCSP stapling.
- Implement HSTS after verifying HTTPS is stable across your site.
- Rotate/renew certificates automatically; protect private keys with strict permissions.
- Consider dual‑stack RSA+ECDSA certs for performance and compatibility.
- Disable 0‑RTT for state‑changing requests if using HTTP/3.

---

### Troubleshooting
- Handshake failures: check certificate chain (leaf + intermediates), key permissions, hostname mismatch, expired cert.
- Protocol mismatch: ensure TLS 1.2/1.3 enabled; disable legacy protocols.
- Cipher mismatch: align server ciphers with client capabilities; use modern policy presets.
- OCSP issues: enable stapling; verify outbound connectivity to CA OCSP responders.
- SNI problems: verify the correct `server_name`/`ServerName` vhost is matched.
- Performance: enable HTTP/2; consider HTTP/3; tune session resumption and TLS ticket keys.
- Tools: `openssl s_client -connect example.com:443 -servername example.com`, `curl -vkI https://example.com`, `ssllabs.com/ssltest`.

---

### Common Questions
- Are SSL and TLS the same? Colloquially yes; technically SSL is obsolete, TLS is modern.
- Do I need a certificate for HTTPS? Yes—issued by a trusted CA or via ACME (Let’s Encrypt).
- What about self‑signed certs? Fine for testing; clients will not trust them without pinning.
- Which validation level should I pick? DV suffices for most; OV/EV for additional org assurance.
- Is mTLS required? Only if you need strong client auth (APIs, internal services, B2B).

---

### References
- TLS 1.3 RFC: `https://www.rfc-editor.org/rfc/rfc8446`
- Mozilla TLS configuration: `https://mozilla.github.io/server-side-tls/`
- Let’s Encrypt/ACME: `https://letsencrypt.org/`, `https://certbot.eff.org/`
- OWASP TLS Cheat Sheet: `https://cheatsheetseries.owasp.org/cheatsheets/TLS_Cheat_Sheet.html`
- SSL Labs test: `https://www.ssllabs.com/ssltest/`


