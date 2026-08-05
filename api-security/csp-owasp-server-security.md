# CSP, OWASP ASVS, and server hardening

A backend developer's job isn't just writing correct endpoints — it's making sure the code, the responses it sends, and the machine it runs on are all hardened against the standard attack playbook. This covers three complementary layers: **OWASP ASVS** (a checklist for whether your code is secure by design), **Content Security Policy** (response headers that block browser-side injection attacks), and **server hardening** (securing the box the code runs on). ASVS levels: L1 (baseline, every app), L2 (apps handling real user data), L3 (high-risk — finance, health).

## TL;DR
- ASVS gives you a structured checklist across auth, sessions, access control, input/output, crypto, and API security — use it to audit, not just to read once.
- CSP is a response header that tells the browser which sources of scripts/styles/images/etc. are allowed to load — its main job is blocking XSS payloads from executing even if one slips into your HTML.
- `unsafe-inline` in `script-src` defeats most of CSP's XSS protection — **nonces or hashes are the modern replacement** and should be your default, not an afterthought.
- Server hardening (patching, minimal attack surface, firewalls, least privilege) matters even with perfect application code — a hardened app on an unhardened box is still compromised.

## 1. OWASP ASVS: a checklist for secure backend code

ASVS (Application Security Verification Standard) is a list of concrete, testable security requirements — 350+ rules across chapters like design, authentication, sessions, access control, input/output handling, cryptography, error handling, data protection, API security, and connections. Rule IDs like `v5.0.0-1.2.5` encode version-chapter-section-rule, so they're referenceable in audits and tickets.

Why use a standard instead of ad hoc review: it stops common, well-understood attack classes (data leaks, broken auth, injection) from being reinvented-and-missed project by project, and it gives you a level (L1/L2/L3) to target based on the app's actual risk instead of guessing how much security is "enough."

| Area | What it covers | Level examples | Example + why |
|---|---|---|---|
| **Design & threats (V1)** | Plan the app to catch risks early | L1: document security-relevant parts. L2: threat modeling. L3: security built into the dev process itself | `app.use(rateLimiter)` — stops resource-exhaustion DoS that can also be used to mask credential-stuffing attempts |
| **Login security (V2)** | Make logins hard to crack | L1: 12+ char passwords. L2: MFA. L3: hardware security keys | `hashlib.pbkdf2_hmac('sha256', password, salt, 100000)` — weak hashing lets attackers brute-force leaked hashes offline |
| **Sessions (V3)** | Keep logins short-lived and unguessable | L1: random session IDs. L2: idle timeout (~15 min). L3: bind session to device/IP | `res.cookie('sid', token, {secure: true, httpOnly: true, sameSite: 'strict'})` — stops session hijacking via XSS or network sniffing |
| **Access control (V4)** | Only let users see/do what they're permitted | L1: check per user and per URL. L2: deny-by-default. L3: re-check on every request, not just at entry | `if (!user.isAdmin) return res.sendStatus(403);` — prevents privilege escalation to admin-only functionality |
| **Input/output safety (V5)** | Validate data in, encode data out | L1: validate all input. L2: context-appropriate output encoding. L3: parameterized queries everywhere, no exceptions | `cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))` — parameterization is the actual fix for SQL injection, string-escaping is not reliable |
| **Cryptography (V6)** | Protect sensitive data with real crypto | L1: strong, standard algorithms (AES-256). L2: key rotation policy. L3: key usage auditing | `crypto.createCipheriv('aes-256-cbc', key, iv)` — if the server/DB is breached, properly encrypted data is still protected |
| **Errors & logs (V7)** | Fail without leaking, log without leaking | L1: no secrets in error responses. L2: log auth failures. L3: retain security logs (e.g., 1 year) | `logger.error('Login failed', {ip: req.ip})` — never log passwords/tokens; verbose errors hand attackers a roadmap |
| **Data privacy (V8)** | Handle personal data minimally and carefully | L1: collect only what's needed. L2: encrypt personal data at rest | `crypto.createHash('sha256').update(email)` for lookups where the raw email isn't needed — regulatory exposure (GDPR) scales with what you retain |
| **APIs & services (V9)** | Secure every endpoint, not just the UI-facing ones | L1: auth all APIs. L2: rate limits + input validation. L3: enforce request/response schemas | CSRF tokens on state-changing requests; wrap JSON responses (`{"data": [...]}`) rather than a bare top-level array, a historical defense against old-browser JSON-array hijacking |
| **Connections (V10)** | Only allow secure transport | L1: TLS 1.3 only where possible. L2: force HTTPS everywhere, no plaintext fallback | `ssl_protocols TLSv1.3;` in Nginx — stops on-path attackers from reading or altering traffic (see `ssl-tls.md`) |
| **Logging & monitoring (V14)** | Make incidents detectable | L1: log access/logins. L2: ship logs to a central system (SIEM) | Centralized logging turns "we got breached and found out three months later" into "we got alerted within minutes" |

**Process checklist**: map risks at design time (cheaper to fix on paper) → validate everything server-side at code time (blocks the bulk of injection-class bugs) → run security scans before shipping (catches what code review misses) → audit periodically against the ASVS level you're targeting.

## 2. Content Security Policy (CSP)

CSP is a response header that tells the browser which sources of content (scripts, styles, images, fonts, frames, connections) are legitimate for this page. Its primary job is mitigating **XSS** — even if an attacker manages to inject a `<script>` tag into your page (via a stored XSS bug, a compromised third-party widget, etc.), a correctly configured CSP stops the browser from executing it, because the script's source doesn't match the allowed list.

### Core directives

| Directive | What it does | Example |
|---|---|---|
| `default-src` | Fallback for any directive not explicitly set | `default-src 'self'` |
| `script-src` | Controls where JavaScript can load/execute from | `script-src 'self'` |
| `style-src` | Controls stylesheet sources | `style-src 'self'` |
| `img-src` | Controls image sources | `img-src 'self' https://cdn.example.com` |
| `connect-src` | Controls fetch/XHR/WebSocket targets | `connect-src 'self' https://api.example.com` |
| `frame-ancestors` | Controls who may embed this page in an iframe (clickjacking defense) | `frame-ancestors 'none'` |
| `object-src` | Controls `<object>`/`<embed>`/`<applet>` — usually disabled entirely | `object-src 'none'` |
| `base-uri` | Restricts what `<base href>` can be set to (prevents base-tag hijacking) | `base-uri 'self'` |
| `upgrade-insecure-requests` | Auto-upgrades `http://` sub-resource requests to `https://` | `upgrade-insecure-requests` |
| `report-to` / `report-uri` | Where the browser sends violation reports | `report-to csp-endpoint` |

```javascript
// Node.js: set a baseline API-appropriate policy
res.set('Content-Security-Policy', "default-src 'none'; frame-ancestors 'none'");
```

### Why `unsafe-inline` defeats the point of CSP

The naive way to get an existing app's inline `<script>` tags working under CSP is:
```
script-src 'self' 'unsafe-inline'
```
This technically "fixes" the console errors, but it also **reintroduces the exact vulnerability CSP exists to prevent**: `unsafe-inline` tells the browser to execute *any* inline script, including one an attacker injected via XSS. At that point CSP is providing essentially no script-injection protection — you've kept the header but removed its teeth.

### Nonce-based CSP — the modern default

A **nonce** ("number used once") is a random, unguessable, base64-encoded value generated fresh **per response** by the server, included both in the CSP header and as an attribute on each legitimate `<script>` tag. Only script tags carrying the matching nonce execute; anything else — including injected script — is blocked, even though inline scripts are technically allowed.

```http
Content-Security-Policy: script-src 'self' 'nonce-r4nd0mBase64Value123'
```
```html
<script nonce="r4nd0mBase64Value123">
  console.log('this runs — nonce matches');
</script>
<script>
  console.log('this is blocked — no nonce, or wrong nonce');
</script>
```

Server-side, generate a new cryptographically random nonce on every request (never reuse it — a reusable nonce is not a nonce, it's just another `unsafe-inline` bypass) and inject it into both the header and the template:

```javascript
const crypto = require('crypto');

app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString('base64');
  res.set(
    'Content-Security-Policy',
    `default-src 'self'; script-src 'self' 'nonce-${res.locals.nonce}'; object-src 'none'; base-uri 'self'`
  );
  next();
});
```
```html
<!-- template, using the nonce injected above -->
<script nonce="<%= nonce %>">
  initApp();
</script>
```

`strict-dynamic` pairs well with nonces for apps that load additional scripts dynamically (bundlers, third-party tags): a script with a valid nonce is trusted to load further scripts, without needing to allowlist every third-party domain individually.
```
Content-Security-Policy: script-src 'nonce-r4nd0m' 'strict-dynamic'
```

### Hash-based CSP — for static inline scripts

If an inline script's content never changes (no per-request templating), you can allowlist it by the **SHA hash of its exact content** instead of a nonce:

```html
<script>alert('static content, never changes');</script>
```
```http
Content-Security-Policy: script-src 'sha256-<base64-encoded-hash-of-exact-script-content>'
```

Compute the hash with the browser's own algorithm (`sha256`, `sha384`, or `sha512`) over the exact bytes between the `<script>` tags (whitespace-sensitive):
```bash
echo -n "alert('static content, never changes');" | openssl dgst -sha256 -binary | openssl base64
```
Any change to the script content — even a single whitespace character — invalidates the hash, which is exactly the point: it guarantees the script that runs is byte-identical to what you approved. Hashes work well for a handful of small, static inline scripts (e.g., a fixed analytics snippet); nonces are the better fit for dynamically rendered pages with many inline scripts, since you don't need to compute/maintain a hash per script.

**Nonce vs. hash — when to use which**:

| | Nonce | Hash |
|---|---|---|
| Best for | Server-rendered pages, dynamic content | Static inline scripts that never change |
| Requires | Fresh random value generated per request | Precomputed hash of exact script content |
| Breaks if | Nonce reused/predictable | Script content changes even slightly |
| Server involvement | Must inject nonce into every response | Can be computed once and hardcoded |

Both approaches let you keep inline scripts (avoiding a large refactor to move everything into external files) while still blocking injected script — which is the actual goal `unsafe-inline` fails to achieve.

### Rollout process

1. **Start in report-only mode** — `Content-Security-Policy-Report-Only` instead of the enforcing header. Violations are reported (via `report-to`/`report-uri`) but nothing is blocked, so you can see what would break before it does.
2. **Move to nonces/hashes** for anything currently relying on `unsafe-inline`.
3. **Enforce** — switch to the real `Content-Security-Policy` header once reports show no unexpected violations.
4. **From general web security practice**: avoid `eval()` and similar dynamic-code-execution patterns entirely — CSP can block `eval` via omitting `'unsafe-eval'`, but the safer fix is just not using it, since eval'd strings are a direct code-injection vector independent of CSP.

CSP is defense-in-depth: even if an XSS bug exists in your code, a correctly scoped policy (no `unsafe-inline`, no wildcard `script-src`) stops it from actually executing in the victim's browser.

## 3. Server hardening

Hardening reduces the server's attack surface — the set of things an attacker could possibly interact with. Application-layer security doesn't help if the underlying box has an open port, a default credential, or a six-month-old unpatched CVE. Follow established baselines (CIS Benchmarks, NIST guidelines) rather than improvising.

| Area | What to do | Linux example | Windows example | Why |
|---|---|---|---|---|
| Patching | Apply updates promptly | `apt upgrade` | Enable auto-updates | Patches close known, actively-exploited holes — unpatched CVEs are the single most common initial-access vector |
| Access | Strong auth, lock out failures | SSH key-only login, disable password auth | Require MFA | Blunts brute-force and credential-stuffing attempts against the box itself |
| Firewall | Only expose needed ports | `ufw allow 443` | Inbound rule for HTTPS only | Every closed port is one less thing for a scanner to find |
| Services | Disable what you don't run | `systemctl disable telnet` | Remove unused print/file services | Fewer running services = fewer possible vulnerabilities |
| Disk/network encryption | Encrypt data at rest and in transit | LUKS full-disk encryption | BitLocker | Protects data even if physical hardware or a disk image is stolen |
| Logging/monitoring | Watch for anomalies | Fail2ban for automated banning | Ship event logs to a central system | Early detection turns a breach attempt into a non-event |
| Backups | Keep independent, tested copies | `rsync` to offsite/cloud storage | Managed cloud backups | Ransomware and destructive attacks are survivable if backups are current and isolated from the compromised host |
| Remote access | Restrict and harden entry points | No root SSH login | RDP with Network Level Authentication | Limits how an attacker who gets partial access can pivot further |

**Process checklist**: lock down access first (least privilege, no shared/default credentials) → configure the firewall (deny by default, allow only what's needed) → patch on a schedule, not "when someone remembers" → minimize installed software (every extra package is extra attack surface and extra patching burden) → back up and monitor continuously, because prevention eventually fails and detection/recovery is what limits the damage.

## How these three layers connect

- **ASVS + CSP**: ASVS's API security chapter (V9) expects output that's safe for the browser to consume — CSP headers are part of satisfying that in practice.
- **ASVS + hardening**: ASVS's connections chapter (V10) — TLS-only, no plaintext fallback — is enforced at the server/firewall level, not just in application code.
- **CSP + hardening**: CSP violation reports are themselves a signal worth logging on a properly monitored server — a spike in reports can indicate an active XSS attempt in progress, not just a misconfigured policy.

None of these three replace the others. Secure code (ASVS) running on a compromised server is still compromised; a hardened server running code with injection bugs still gets breached through the application; strong headers (CSP) on a server with an exposed admin port still gets popped through the port. Full coverage requires all three layers.
