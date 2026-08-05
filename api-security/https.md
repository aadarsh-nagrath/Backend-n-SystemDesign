# HTTPS

HTTPS is HTTP layered over TLS — the same request/response protocol you already know, but encrypted and authenticated in transit. If you're looking for the handshake mechanics, certificate chain validation, and cipher suite details, see `ssl-tls.md` — this file focuses on the practical side of running HTTPS in production: redirects, HSTS, mixed content, ports, and day-to-day setup.

## TL;DR
- HTTPS = HTTP + TLS. Same protocol, same methods/status codes, but the connection is encrypted and the server is authenticated via a certificate.
- Port 443 (vs. HTTP's 80); browsers show a padlock for HTTPS, flag HTTP as "Not Secure."
- Always redirect HTTP → HTTPS (301), then add HSTS so browsers skip the insecure request entirely on repeat visits.
- Mixed content (an HTTPS page loading an HTTP sub-resource) gets blocked or flagged by browsers — audit for it after migrating.
- For the handshake itself, certificate chains, and cipher suite configuration: see `ssl-tls.md`.

## 1. What HTTPS is and why it matters

HTTP sends data in plaintext — anyone on the network path (your ISP, a coffee-shop Wi-Fi eavesdropper, a compromised router) can read or modify it in transit. HTTPS wraps that same HTTP traffic in TLS encryption, so the content is unreadable to anyone except the two endpoints, and tampering is detectable.

**Why it matters in practice**:
- **Sensitive data protection** — logins, payment details, personal data are unreadable in transit.
- **Browser signals** — modern browsers show a padlock for HTTPS and flag HTTP explicitly as "Not Secure," which affects user trust directly.
- **SEO** — search engines treat HTTPS as a ranking signal; HTTP sites are penalized.
- **Feature gating** — many browser APIs (geolocation, camera/mic access, service workers, clipboard) are restricted to secure contexts (HTTPS or `localhost`) and simply don't work over plain HTTP.
- **Universal expectation** — any site handling logins, forms, or user data needs it; even static content sites benefit from avoiding the "Not Secure" flag and the SEO penalty.

**Example**: shopping on an e-commerce site over public Wi-Fi without HTTPS — a passive eavesdropper running a packet sniffer can read your card number directly off the wire. With HTTPS, the same traffic is ciphertext.

For *how* the encryption and authentication actually get established (the handshake, certificates, key exchange), see `ssl-tls.md` — that's the mechanics; this file is about running HTTPS correctly in a real deployment.

## 2. HTTP vs. HTTPS

| Aspect | HTTP | HTTPS |
|---|---|---|
| Security | Plaintext — readable/modifiable in transit | Encrypted and integrity-checked |
| Port | 80 | 443 |
| Overhead | None | Handshake cost, mostly amortized (session resumption, TLS 1.3's 1-RTT) |
| Browser treatment | "Not Secure" warning; some APIs disabled | Padlock; full API access |
| SEO | No ranking benefit | Ranking signal |
| Typical use | Internal-only tooling, if anything | Everything public-facing |

## 3. Ports and protocol layering

- **Port 443** is the default for HTTPS; **port 80** for HTTP. Firewalls typically keep 443 open and may restrict 80 to redirect-only traffic.
- HTTPS is not a separate protocol from HTTP — it's HTTP running on top of a TLS-encrypted TCP (or, for HTTP/3, UDP/QUIC) connection. The HTTP semantics (methods, headers, status codes) are unchanged; only the transport is different.
- HTTP/2 multiplexes multiple requests over one TLS connection, cutting connection overhead; HTTP/3 (QUIC) replaces TCP with UDP and folds TLS 1.3 into the transport handshake itself, improving performance especially on lossy networks (mobile).

## 4. Redirecting HTTP to HTTPS

Once HTTPS is configured, every HTTP request should be redirected to the HTTPS equivalent — never served in parallel indefinitely, since that leaves users who happen to hit the HTTP URL first (bookmarks, typed URLs, old links) exposed on that first request.

**Nginx**:
```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

**Apache**:
```apache
<VirtualHost *:80>
  ServerName example.com
  Redirect permanent "/" "https://example.com/"
</VirtualHost>
```

**`.htaccess`**:
```apache
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

Use a `301 Moved Permanently` (not `302`), so browsers and crawlers cache the redirect and search engines consolidate ranking signal onto the HTTPS URL rather than treating them as two separate pages.

⚠️ A redirect alone still means the *first* request to `http://` goes out in plaintext before the redirect is received — an on-path attacker can intercept that first request (SSL stripping). HSTS (next section) is what actually closes this gap on repeat visits.

## 5. HSTS (HTTP Strict Transport Security)

HSTS is a response header that tells the browser: "never make a plain HTTP request to this host again — always upgrade to HTTPS automatically, without even trying HTTP first." This is what prevents SSL-stripping attacks on subsequent visits, since the browser stops sending the vulnerable first HTTP request entirely.

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

- **`max-age`** — how long (seconds) the browser should remember this. `31536000` = 1 year, the common production value.
- **`includeSubDomains`** — applies the policy to every subdomain too, not just the exact host.
- **`preload`** — opts into browser vendors' hardcoded HSTS preload list, so *even the very first request ever made to this domain*, from a browser that's never visited before, is forced to HTTPS. Submission is via `hstspreload.org`, and it's effectively permanent — removal from the preload list is slow and painful, so only add `preload` once you're confident every subdomain will support HTTPS indefinitely.

**Rollout order matters**: verify HTTPS works reliably across the entire site (and all subdomains, if using `includeSubDomains`) before turning HSTS on — once a client has cached the HSTS policy, it will refuse to connect over HTTP even if you need to temporarily roll back, until `max-age` expires or you serve a header with `max-age=0` to explicitly clear it.

## 6. Mixed content

**Mixed content** is when an HTTPS page loads a sub-resource (script, stylesheet, image, iframe) over plain HTTP. This partially defeats the purpose of HTTPS — the sub-resource is fetchable/modifiable by a network attacker, and script-level mixed content is a real code-injection vector even though the parent page itself is "secure."

Browsers handle this by severity:
- **Active mixed content** (scripts, stylesheets, iframes — things that can execute code or alter the page) — **blocked by default** in modern browsers. The resource simply fails to load.
- **Passive mixed content** (images, video, audio) — historically allowed with a browser warning, though browsers have been tightening this too (Chrome auto-upgrades passive mixed content to HTTPS where possible, and blocks if the upgrade fails).

**Fixing it**:
- Update hardcoded `http://` URLs in HTML/CSS/JS to `https://`, or use protocol-relative/relative URLs.
- Audit with browser DevTools (Console flags mixed content warnings/errors) after any HTTP→HTTPS migration.
- `Content-Security-Policy: upgrade-insecure-requests` — instructs the browser to automatically rewrite `http://` sub-resource requests to `https://` before sending them, as a blanket fix during migration (see `csp-owasp-server-security.md` for full CSP directive coverage).

```http
Content-Security-Policy: upgrade-insecure-requests
```

This doesn't fix a resource that's genuinely only available over HTTP (the upgraded request will just fail) — it's a safety net for stale hardcoded URLs during migration, not a substitute for actually updating them.

## 7. Setting up HTTPS: practical steps

1. **Get a certificate** — free DV certs from Let's Encrypt (auto-renewing via ACME) or your CDN/host (Cloudflare's Universal SSL is instant on their free tier). See `ssl-tls.md` for certificate types (DV/OV/EV) and validation levels.
2. **Install on the server** — point your web server config at the cert and key files (see `ssl-tls.md` for full Nginx/Apache config examples).
3. **Redirect HTTP → HTTPS** (section 4).
4. **Add HSTS** once HTTPS is verified stable (section 5).
5. **Audit for mixed content** (section 6).
6. **Automate renewal** — Certbot (`certbot --nginx`) or your host's built-in automation; expired certificates are one of the most common self-inflicted outages.
7. **Verify**: SSL Labs (`ssllabs.com/ssltest`) for a full external grade, or `curl -vI https://yoursite.com` for a quick manual check.

## 8. Common issues

| Issue | Cause | Fix |
|---|---|---|
| Mixed content warnings/blocks | HTTP sub-resources on an HTTPS page | Update URLs to HTTPS; audit with DevTools; consider `upgrade-insecure-requests` |
| "Not Secure" despite having a cert | Redirect not configured, or cert not installed on all vhosts | Verify redirect rules and that every subdomain/vhost serving traffic has a valid cert |
| HSTS lockout during rollback | Browser cached the HSTS policy and refuses HTTP fallback | Serve `Strict-Transport-Security: max-age=0` to clear it, or wait out `max-age` |
| Cert expired | Renewal not automated | Automate with Certbot/ACME; monitor expiry dates |
| SEO/traffic drop after migration | Redirect using `302` instead of `301`, or broken redirect chains | Use `301`; verify redirects with `curl -I` |

## 9. Why HTTPS is non-negotiable today

- **Privacy** — encrypts traffic so ISPs, network operators, and passive attackers see ciphertext, not content.
- **Integrity** — a network attacker can't silently modify responses in transit (injecting ads, altering prices, swapping download links).
- **Authentication** — the certificate proves you're actually talking to the domain you think you are, not an impersonator (see `ssl-tls.md` for how certificate validation establishes this).
- **Compliance** — GDPR, PCI-DSS, and HIPAA all effectively require encryption of data in transit.
- **No silent ad/content injection** — some ISPs have historically injected ads or tracking into unencrypted HTTP traffic; HTTPS makes this impossible.

## Further reading
- `ssl-tls.md` — the TLS handshake step by step, certificate chains and validation, cipher suites, mTLS, and server config examples.
- `csp-owasp-server-security.md` — Content-Security-Policy directives including `upgrade-insecure-requests`.
- SSL Labs test: `https://www.ssllabs.com/ssltest/`
- HSTS preload list submission: `https://hstspreload.org/`
- Let's Encrypt / Certbot: `https://letsencrypt.org/`, `https://certbot.eff.org/`
