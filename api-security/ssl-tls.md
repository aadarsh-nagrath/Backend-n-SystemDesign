# SSL/TLS

TLS is what turns HTTP into HTTPS — the protocol that encrypts, authenticates, and checks the integrity of data between a client and a server. This is the mechanics file: the handshake step by step, certificate chains and validation, cipher suites, mutual TLS, and server configuration. For the practical HTTP-layer side (redirects, HSTS, mixed content), see `https.md`.

## TL;DR
- **SSL** is the deprecated 1990s predecessor; **TLS** is the modern protocol. "SSL" is still used colloquially to mean TLS — but never actually deploy SSLv2/SSLv3.
- TLS combines **asymmetric crypto** (for authentication and initial key exchange — slow but solves the "how do we agree on a secret with no prior shared secret" problem) with **symmetric crypto** (fast, used for the actual data once a shared key exists).
- The handshake's job: agree on a protocol version and cipher suite, authenticate the server via its certificate, and derive a shared symmetric key — all before a byte of application data moves.
- TLS 1.3 cuts the handshake to 1 round trip (down from 2 in TLS 1.2), removes legacy/weak ciphers, and is always forward-secret.
- Certificate validation means checking: is it signed by a CA my client trusts, is it for the right hostname, has it expired, has it been revoked.
- Use TLS 1.2 (compatibility) and TLS 1.3 (preferred); disable everything older.

## What is SSL? What is TLS?

- **SSL (Secure Sockets Layer)** — the original web encryption protocol, introduced by Netscape in the 1990s to secure early e-commerce. Deprecated and insecure; SSLv2 and SSLv3 must never be used (SSLv3 is broken by the POODLE attack).
- **TLS (Transport Layer Security)** — the modern successor, standardized by the IETF starting in 1999 as a renamed, cleaned-up continuation of SSL 3.0. "SSL" persists as informal shorthand for TLS in casual usage (certs are still commonly called "SSL certificates"), but actual deployments should run TLS 1.2 or, preferably, TLS 1.3.
- A site running TLS correctly serves `https://` and browsers show a padlock in the address bar.

### Timeline

| Version | Year | Notes | Status today |
|---|---|---|---|
| SSL 1.0 | 1994 | Netscape's first attempt; never publicly released | Never used |
| SSL 2.0 | 1995 | First public version, seriously flawed | Deprecated, insecure |
| SSL 3.0 | 1996 | Improved handshake, still flawed | Deprecated (POODLE attack, 2014) |
| TLS 1.0 | 1999 | IETF-standardized successor to SSL 3.0 | Deprecated by all major browsers since 2020 |
| TLS 1.1 | 2006 | Fixed some CBC-mode vulnerabilities | Deprecated, not recommended |
| TLS 1.2 | 2008 | Added AES, SHA-2; still widely deployed | Fine for compatibility; migrate to 1.3 where possible |
| TLS 1.3 | 2018 | Faster handshake, hides more of the negotiation, drops weak ciphers | Preferred/default target |

## HTTP vs HTTPS

- **HTTP** transmits data in plaintext — eavesdroppers can read or modify it in transit.
- **HTTPS (HTTP over TLS)** encrypts and authenticates data in transit, preventing interception and tampering, and authenticating the server's identity. See `https.md` for redirects, HSTS, and mixed-content handling once TLS itself is configured.

## Why TLS matters

- **Confidentiality** — encryption prevents eavesdropping on credit cards, credentials, PII.
- **Integrity** — detects/blocks tampering (man-in-the-middle proxies altering content in transit).
- **Authentication** — certificates bind a domain to a public key; clients verify the server is who it claims to be.
- **Trust/UI** — browsers flag non-HTTPS as "Not Secure"; some browser APIs require a secure context outright.

### Why this is non-negotiable for modern sites
- **Browser enforcement** — HTTP sites get flagged "Not Secure," and features like geolocation and camera/mic access are blocked outright on insecure origins.
- **SEO** — search engines rank HTTPS sites higher.
- **API requirements** — most modern REST/GraphQL APIs require HTTPS.
- **Regulatory compliance** — PCI-DSS, GDPR, HIPAA effectively mandate encryption in transit.
- **User trust** — users expect and look for the padlock.

## How TLS works, conceptually

TLS provides three guarantees over the raw TCP connection:
- **Encryption** using symmetric keys (fast), negotiated securely during the handshake.
- **Authentication** using X.509 certificates and a public key infrastructure (PKI).
- **Integrity** via MACs or AEAD modes (e.g., AES-GCM, ChaCha20-Poly1305) — tampering with ciphertext in transit is detectable.

### Why both asymmetric and symmetric crypto

- **Asymmetric (public-key) encryption** (RSA/ECDSA/ECDHE): a keypair where one key encrypts and the other decrypts. Secure and solves the "no prior shared secret" bootstrap problem, but computationally expensive — too slow to encrypt an entire session's worth of data.
- **Symmetric encryption** (AES, ChaCha20): a single shared key both sides use to encrypt/decrypt. Fast, but requires both sides to already have the same secret — which is exactly the problem asymmetric crypto solves.
- **The solution**: use asymmetric crypto only for the handshake — authenticating the server and securely agreeing on a symmetric key — then switch to symmetric encryption for all actual data. You get the bootstrapping security of asymmetric crypto with the speed of symmetric crypto.

**The carrier pigeon analogy**: Alice (browser) wants to send Bob (server) a message, with Mallory (an attacker) able to intercept pigeons mid-flight.
- **Naive/no encryption**: Alice ties a note to a pigeon in the clear. Mallory reads or swaps it — Bob has no way to know.
- **Symmetric key alone (like a Caesar cipher)**: Alice and Bob agree on a shift code so messages are unreadable to Mallory — but how do they agree on the code in the first place without Mallory seeing *that* exchange too? They can't, without another mechanism — this is exactly the "key distribution problem," and solving it naively is a man-in-the-middle opportunity.
- **Asymmetric crypto (boxes and locks)**: Alice publishes an open box (public key) — anyone, including Mallory, can see it, but a certificate signed by a trusted party (Ted, the CA) proves the box really belongs to Alice. Bob locks his message in Alice's box and sends it back; only Alice's private key opens it. No shared secret had to be exchanged in the clear first.
- **Hybrid (the actual answer)**: use the heavy, slow boxes (asymmetric) just once, to securely exchange a lightweight shared code (symmetric key) — then switch to the fast shared-code system for the actual conversation. This is precisely what TLS does: asymmetric crypto bootstraps a symmetric session key, then symmetric crypto carries the real traffic.

## The TLS handshake, step by step

### TLS 1.2 handshake (the classic 2-round-trip version)

**Step 1 — TCP connection**: the browser opens a TCP connection to the server first, same as for plain HTTP. TLS then runs on top of this.

**Step 2 — Client Hello**: the client sends a `ClientHello` containing:
- Supported TLS versions
- Supported cipher suites (encryption algorithm combinations it's willing to use)
- A random value (used later in key derivation)
- **SNI (Server Name Indication)** — the hostname being requested, needed so a server hosting multiple HTTPS sites on one IP knows which certificate to present
- Extensions (e.g., ALPN, to negotiate HTTP/2)

**Step 3 — Server Hello**: the server responds with:
- The chosen TLS version and cipher suite (picked from what the client offered)
- Its own random value
- Its **certificate** — containing its public key, domain name, validity period, and a signature from a CA
- Possibly additional messages (`ServerKeyExchange`, `CertificateRequest` for mTLS), then `ServerHelloDone`

**Step 4 — Certificate verification**: the client checks the server's certificate:
- Is it signed by a CA in the client's trust store (directly or via a chain of intermediates)?
- Does the domain name (or a SAN entry) match the hostname being requested?
- Is it within its validity period (not expired, not "not yet valid")?
- Has it been revoked (via OCSP/CRL)?

**Step 5 — Key exchange**: the client generates a random symmetric session key, encrypts it using the server's public key (from the certificate), and sends it over. The server decrypts it with its private key. (Modern deployments use ephemeral Diffie-Hellman — ECDHE — instead of RSA key transport here, for forward secrecy; see below.) Both sides now hold the same symmetric key without it ever having traveled in plaintext.

**Step 6 — Secure communication**: both sides use the negotiated symmetric key and cipher suite for all further data. `Finished` messages on both sides verify the handshake itself wasn't tampered with, by including a MAC over the handshake transcript.

### TLS 1.3 handshake (faster, simpler)

- **1-RTT by default** — client and server can agree on parameters and derive keys in a single round trip, versus TLS 1.2's two. TLS 1.3 achieves this by having the client guess/send its preferred key-exchange parameters in the first flight, rather than waiting for the server to pick first.
- **0-RTT (optional)** — for resumed connections, the client can send application data in its very first flight, before the handshake even finishes. This is *replay-risky* (an attacker who captures a 0-RTT request can resend it) — disable 0-RTT for anything non-idempotent (state-changing requests like payments).
- **AEAD ciphers only** — legacy/weak cipher suites are removed entirely from the protocol; you get AES-GCM or ChaCha20-Poly1305, full stop.
- **Always forward-secret** — TLS 1.3 mandates ECDHE (ephemeral key exchange) for every handshake; static RSA key exchange is gone.
- **More of the handshake is encrypted** — TLS 1.3 hides more of the negotiation (including the server certificate) from passive on-path observers compared to 1.2.

### Perfect Forward Secrecy (PFS)

Achieved via ephemeral Diffie-Hellman (ECDHE): each session negotiates a *fresh*, temporary key pair used only for that session's key derivation, then discarded. If the server's long-term private key is later stolen, past recorded sessions **cannot** be decrypted retroactively, because their session keys were never derivable from the long-term key alone — they depended on ephemeral values that no longer exist anywhere. Without PFS (e.g., old-style RSA key exchange), a compromised private key can decrypt every past session ever recorded, which is the scenario PFS specifically defeats.

## Public Key Infrastructure (PKI) basics

- **Certificates (X.509)** bind an identity (Common Name / Subject Alternative Names) to a public key.
- **Certificate Authorities (CAs)** issue certificates after verifying some level of control/ownership; browsers and operating systems ship a trust store of root CAs they trust by default.
- **Chain of trust**: leaf certificate (your site) → one or more intermediate CA certificates → a root CA certificate the client already trusts. Servers must send the full chain (leaf + intermediates) — a missing intermediate is one of the most common real-world "cert works in some clients, fails in others" bugs, since some clients cache intermediates from prior connections and others don't.
- **Validation** on the client side checks: expiration, hostname match, signature chain integrity up to a trusted root, and (optionally) revocation status.
- **Revocation**: OCSP (Online Certificate Status Protocol) or CRLs (Certificate Revocation Lists) let a client check if a cert was revoked before its expiry (e.g., after a private key leak). **OCSP stapling** — the server periodically fetches its own OCSP response and includes ("staples") it directly in the handshake — improves both performance (client skips an extra round trip to the CA) and privacy (the CA doesn't see every client that's checking a given cert's status).

## Certificate types

**By scope** (how many hostnames one certificate covers):
- **Single-domain** — one FQDN (e.g., `example.com` or `www.example.com`, but not both unless listed as SANs).
- **Wildcard** — covers one domain plus all first-level subdomains (`*.example.com` covers `api.example.com` but not `api.staging.example.com`).
- **Multi-domain (SAN/UCC)** — multiple distinct FQDNs in a single certificate via the Subject Alternative Name extension.

**By validation level** (how much the CA verified before issuing):

| Type | Validation performed | Typical cost | Best for | Example issuer |
|---|---|---|---|---|
| **Domain Validation (DV)** | Proves control of the domain only (DNS/HTTP/email challenge) — automatable, minutes | Free–$10/yr | Blogs, personal sites, most APIs | Let's Encrypt |
| **Organization Validation (OV)** | CA verifies the requesting organization's business documents | $50–$200/yr | E-commerce, apps handling user data | Sectigo OV |
| **Extended Validation (EV)** | Strict legal/business vetting | $100–$500/yr | Banks, high-trust institutional sites | DigiCert EV |

DV now covers the vast majority of certificates in the wild — automation (ACME/Let's Encrypt) made it essentially free and instant, and browsers stopped giving EV certificates a distinct UI treatment (the old "green address bar"), which reduced EV's practical user-facing benefit. OV/EV still carry more legal weight for organizational accountability, but they don't change the cryptographic strength of the connection itself — a DV cert encrypts exactly as strongly as an EV one.

**Key algorithms**: RSA (2048-bit minimum, 3072+ for longer-term security) or ECDSA (P-256/P-384 — smaller keys and faster signature operations than RSA at equivalent security levels). Many production sites deploy **dual certificates** (both RSA and ECDSA) so older clients that don't support ECDSA still connect, while modern clients get ECDSA's performance benefit.

## Modern TLS versions and cipher suites

- Disable SSLv2, SSLv3, TLS 1.0, TLS 1.1 entirely. Support TLS 1.2 and TLS 1.3.
- Prefer AEAD cipher suites: AES-GCM or ChaCha20-Poly1305 (ChaCha20 is notably faster on devices without AES hardware acceleration, e.g., some mobile chips).
- Prioritize ECDHE key exchange for forward secrecy; offer both ECDSA and RSA certificates if broad client compatibility matters.
- Reference: Mozilla's SSL/TLS configuration generator provides current "modern" and "intermediate" policy presets — use it over hand-rolling a cipher list from memory, since guidance shifts as attacks against specific ciphers are found.

## Advanced concepts

- **SNI (Server Name Indication)** — lets one IP address serve multiple HTTPS sites, each with its own certificate, by having the client announce the target hostname *before* the server picks which cert to present. Essential for any shared/virtual hosting setup.
- **ECH (Encrypted ClientHello)** — encrypts the SNI field itself, so even the hostname being requested is hidden from on-path observers (SNI is otherwise sent in the clear even under TLS 1.3). Emerging support, not yet universal.
- **ALPN (Application-Layer Protocol Negotiation)** — a TLS extension that negotiates the application protocol (`h2` for HTTP/2, `h3` for HTTP/3) during the handshake itself, avoiding an extra round trip to upgrade after connecting.
- **HTTP/2** — multiplexes many requests over a single TCP+TLS connection, reducing per-request connection overhead.
- **HTTP/3 (QUIC)** — runs over UDP with TLS 1.3 integrated into the transport handshake itself; faster connection setup and better loss recovery than TCP-based HTTP/2, especially on unreliable networks.
- **mTLS (Mutual TLS)** — the client also presents a certificate, so the server authenticates the client too, not just the reverse. Common for service-to-service and B2B API auth.
- **0-RTT** — see TLS 1.3 handshake above; disable for non-idempotent requests due to replay risk.

## Config examples

### Nginx (TLS 1.2/1.3, HTTP/2, HSTS)
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

HTTP/3 (QUIC) note (requires a QUIC-enabled build):
```nginx
server {
    listen 443 http3 reuseport;
    listen 443 ssl http2;
    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

### Apache httpd (TLS + HSTS + HTTP/2)
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

## Automation: ACME and Let's Encrypt

- Use ACME clients to obtain and renew certificates automatically — manual certificate management is how expired-cert outages happen.
- **Certbot (Debian/Ubuntu)**:
```bash
sudo apt install -y certbot python3-certbot-nginx   # or python3-certbot-apache
sudo certbot --nginx   -d example.com -d www.example.com --redirect --hsts --agree-tos -m admin@example.com --non-interactive
# or
sudo certbot --apache  -d example.com -d www.example.com --redirect --hsts --agree-tos -m admin@example.com --non-interactive
```
- **DNS-01 challenges** are required for wildcard certs (and useful for complex setups where the server isn't directly reachable on port 80); use a DNS provider plugin (Cloudflare, Route53, etc.).
- Ensure renewal timers actually run and reload the web server on renewal (`--deploy-hook`) — a cert that renews on disk but is never reloaded into the running server process still expires from the client's perspective.

## Security headers complementing TLS

TLS secures the transport; these headers close related gaps at the HTTP layer:
- `Strict-Transport-Security` (HSTS) — enforces HTTPS on future requests (full detail in `https.md`).
- `X-Content-Type-Options: nosniff` — mitigates MIME-type sniffing.
- `X-Frame-Options: DENY` or CSP `frame-ancestors` — clickjacking defense.
- `Referrer-Policy: strict-origin-when-cross-origin` — limits what's leaked in the `Referer` header on cross-origin navigation.
- `Content-Security-Policy` — controls allowed content sources, closing off many XSS vectors (see `csp-owasp-server-security.md`).

## Operational hardening checklist

- Disable SSLv2, SSLv3, TLS 1.0, TLS 1.1.
- Prefer TLS 1.3; support TLS 1.2 for compatibility.
- Use modern AEAD ciphers; enable ECDHE for PFS.
- Use strong keys: RSA 2048+ or ECDSA P-256/P-384.
- Enable OCSP stapling.
- Implement HSTS only after verifying HTTPS is stable site-wide.
- Automate certificate rotation/renewal; protect private keys with strict file permissions.
- Consider dual-stack RSA+ECDSA certs for performance and compatibility.
- Disable 0-RTT for state-changing requests if using HTTP/3/TLS 1.3 early data.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Handshake failures | Broken certificate chain (missing intermediates), key permission issues, hostname mismatch, expired cert | Bundle the full chain; check file permissions; verify SAN/CN matches; check expiry |
| Protocol mismatch errors | Client and server don't share a supported TLS version | Ensure TLS 1.2/1.3 enabled server-side; disable legacy protocols |
| Cipher mismatch | No common cipher suite between client and server | Align server cipher list with client capabilities; use a modern policy preset |
| OCSP issues | Stapling misconfigured, or outbound connectivity to CA's OCSP responder blocked | Enable/verify stapling; check firewall egress rules |
| SNI problems | Wrong vhost/server block matched | Verify `server_name`/`ServerName` matches exactly |
| Slow TLS | Old TLS version, no session resumption, no HTTP/2 | Upgrade to 1.3; enable HTTP/2; tune session resumption and TLS ticket keys |

Diagnostic tools:
```bash
openssl s_client -connect example.com:443 -servername example.com
curl -vkI https://example.com
```
Plus `ssllabs.com/ssltest` for a full external grade covering protocol support, cipher strength, and certificate chain correctness.

## Common questions

- **Are SSL and TLS the same?** Colloquially yes; technically SSL is obsolete and TLS is the protocol actually in use.
- **Do I need a certificate for HTTPS?** Yes — issued by a trusted CA or via ACME (Let's Encrypt is free).
- **What about self-signed certs?** Fine for local testing; browsers/clients won't trust them by default without manual pinning/import.
- **Which validation level should I pick?** DV is sufficient for most sites and APIs; OV/EV add organizational assurance but not cryptographic strength.
- **Is mTLS required?** Only when you need strong client authentication — internal service-to-service calls, B2B integrations, zero-trust architectures.

## Further reading
- `https.md` — HTTP-layer practicalities: redirects, HSTS rollout, mixed content.
- `csp-owasp-server-security.md` — Content-Security-Policy directives.
- TLS 1.3 RFC: `https://www.rfc-editor.org/rfc/rfc8446`
- Mozilla TLS configuration generator: `https://mozilla.github.io/server-side-tls/`
- Let's Encrypt / ACME: `https://letsencrypt.org/`, `https://certbot.eff.org/`
- OWASP TLS Cheat Sheet: `https://cheatsheetseries.owasp.org/cheatsheets/TLS_Cheat_Sheet.html`
- SSL Labs test: `https://www.ssllabs.com/ssltest/`
