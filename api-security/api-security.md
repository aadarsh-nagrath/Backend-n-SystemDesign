# API Security

APIs are the backbone of modern software — mobile apps, cloud integrations, service-to-service calls — and that makes them the default attack surface. Unlike a traditional web app where the server controls the whole UI, an API hands a contract straight to the client, and every endpoint is a door someone can knock on directly (no browser sandboxing, no UI to hide behind). Securing an API means covering authentication, authorization, input handling, transport, and operational visibility — not just "add a login."

## TL;DR
- Auth (who you are) and access control (what you can do) are different problems — solve both, separately.
- Never hand-roll auth. Use OAuth 2.0/OIDC, JWT with proper validation, or vetted API-key schemes.
- Validate everything coming in (method, content-type, body shape); reveal as little as possible going out (headers, error detail, object fields).
- Rate limit and throttle every public endpoint — it's your cheapest defense against brute force and DoS.
- The OWASP API Security Top 10 is the standard checklist — BOLA (broken object-level authorization) is consistently the #1 real-world API vulnerability.
- Log and monitor centrally, but never log secrets, tokens, or full credentials.

## Threat modeling for APIs

Before writing endpoint code, model the threats. Two common frameworks:
- **STRIDE** — Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege. A checklist to run per endpoint.
- **PASTA** (Process for Attack Simulation and Threat Analysis) — a more business-risk-driven flow: map the app, list threats, simulate attacks, assess impact.

Practical steps:
1. Map API flows — use your OpenAPI/Swagger spec as the source of truth for what exists.
2. Identify entry points — every endpoint, every parameter, every header the server trusts.
3. Assess risk against the OWASP API Top 10 (below).
4. Prioritize with **DREAD** — Damage, Reproducibility, Exploitability, Affected users, Discoverability — to decide what to fix first.

**Example**: for `/users/{id}/orders`, the obvious threat is IDOR/BOLA — can a logged-in user change `{id}` and see someone else's orders? Mitigate with a server-side ownership check on every request, not just at login.

Tools: OWASP Threat Dragon, Microsoft Threat Modeling Tool.

## Authentication

Authentication answers "who are you." Get this wrong and you have OWASP **API2: Broken Authentication**.

**Rules of thumb:**
- **Avoid Basic Auth** — credentials go over the wire base64-encoded (not encrypted) on every request; if TLS is ever misconfigured, it's a plaintext leak. Use token-based auth instead.
- **Don't reinvent auth** — use OAuth 2.0/OIDC, or a maintained library. Auth bugs are subtle and attackers actively hunt for them; a custom scheme is a liability, not a feature.
- **Rate-limit login attempts** — cap retries (e.g., 5 attempts) then lock the account temporarily ("jail"), to blunt brute force and credential stuffing.
- **Encrypt sensitive data** — AES-256 for data at rest; hash passwords with bcrypt or Argon2 (never plain SHA-256 — it's too fast, making brute force cheap).
- **MFA for sensitive operations** — TOTP or WebAuthn for anything high-value.
- **Secure session cookies** — `HttpOnly`, `Secure`, `SameSite=Strict` if the API is stateful and cookie-backed.
- **mTLS for machine-to-machine** — when both sides are services you control, mutual TLS gives strong bidirectional identity without a shared secret.

```javascript
// Node.js login with hashed password + short-lived JWT
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  const user = await getUserByUsername(username);
  if (user && await bcrypt.compare(password, user.hash)) {
    const token = jwt.sign({ sub: user.id }, process.env.SECRET, { expiresIn: '1h' });
    res.json({ token });
  } else {
    // Increment a per-account/per-IP failure counter here (rate limit)
    res.status(401).send('Invalid credentials');
  }
});
```

Validate the token on *every* protected request server-side — never trust a client's claim that it's authenticated. Support revocation (e.g., a denylist in Redis for tokens invalidated before their natural expiry, such as on logout or password change).

Other angles: zero-trust ("verify every request regardless of network location, VPN, or prior auth"), credential stuffing defenses (CAPTCHA after repeated failures, breached-password checks), and biometric auth on mobile clients (with a PIN/password fallback — biometrics should never be the *only* factor a backend trusts blindly).

## JWT (JSON Web Tokens)

JWTs are compact, signed (optionally encrypted) tokens for carrying claims between parties — the most common shape for stateless API auth. A JWT has three base64url segments: `header.payload.signature`.

**Rules that actually matter:**
- **Strong secret / key** — 256-bit random for HMAC (HS256), or an RSA/EC keypair for RS256/ES256. A weak or guessable secret means anyone can forge tokens.
- **Never trust the `alg` from the header** — a classic attack is sending a token with `"alg": "none"` or downgrading RS256 to HS256 (using the public key as an HMAC secret) to forge a valid-looking signature. The server must hardcode the expected algorithm and reject anything else, not read `alg` from the token and dispatch on it.
- **Short TTL** — 15–60 minutes for access tokens; use a separate, revocable refresh token for longer sessions.
- **No sensitive data in the payload** — the payload is base64-encoded, not encrypted; anyone can decode and read it. Keep PII out; stick to `sub`, `iss`, `aud`, `exp`, roles/scopes.
- **Small payload** — every claim adds bytes to every request; keep it to what authorization checks actually need.
- **Validate `iss`, `aud`, `exp`, `nbf`** server-side on every request — a token issued for a different audience or already expired must be rejected even if the signature is valid.
- **Revocation** — JWTs are stateless by design, meaning you can't "delete" one; use a short TTL plus a denylist (Redis, keyed by `jti` or user-id) for tokens that need explicit invalidation (logout, compromised account).

```python
from flask import Flask, jsonify
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity
import datetime

app = Flask(__name__)
app.config['JWT_SECRET_KEY'] = 'super-secret'  # load from env/secret manager, not hardcoded
jwt = JWTManager(app)

@app.route('/protected', methods=['GET'])
@jwt_required()
def protected():
    return jsonify(logged_in_as=get_jwt_identity()), 200

access_token = create_access_token(identity='user', expires_delta=datetime.timedelta(minutes=15))
```

Use a well-audited library (PyJWT, `jsonwebtoken`, etc.) rather than parsing/verifying manually — algorithm-confusion and signature-stripping bugs are exactly the kind of thing these libraries have already hardened against. For encrypted payloads, JWTs have a JWE variant (vs. the signed-only JWS); combine with OAuth by using the JWT as the access/ID token format.

## Access control

Access control answers "what are you allowed to do" — a different question from authentication, and mixing them up is a common root cause of OWASP **API1 (BOLA)**, **API3 (broken object property auth)**, and **API5 (broken function-level auth)**.

- **Rate limiting / throttling** — protects against brute force and DoS regardless of auth status. Token bucket or fixed/sliding window; apply per-IP and per-account.
- **HTTPS everywhere, strong ciphers** — TLS 1.3 preferred, AES-GCM ciphers; see `https.md` and `ssl-tls.md` for the mechanics.
- **HSTS** — forces browsers to use HTTPS on every subsequent visit, closing the window for SSL-stripping downgrade attacks.
- **IP allowlisting for private/internal APIs** — restrict to known IP ranges or VPCs rather than exposing internal endpoints publicly.
- **Disable directory listing** — an exposed file listing leaks structure and sometimes secrets (config files, backups).
- **RBAC/ABAC** — Role-Based (fixed roles → permissions) or Attribute-Based (dynamic rules on user/resource/context attributes) access control, applied server-side on every request, not just at the UI layer.
- **API gateways** for centralized enforcement (rate limits, auth, logging) — Kong, AWS API Gateway, Apigee.

```javascript
const rateLimit = require('express-rate-limit');
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100                  // limit each IP to 100 requests per window
});
app.use(limiter);
```

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

**BOLA/IDOR defense in practice**: don't rely on "the ID is hard to guess" (sequential integers are trivially enumerable — `GET /orders/1001`, `/orders/1002`, ...). Use UUIDs *and* still check server-side that the authenticated user owns/can access the requested object on every single request, not just at the route's outer auth check. This is the single most common real-world API vulnerability class — get it right.

Zero-Trust Network Access (verify identity/context per-request, not per-network) and service meshes (Istio for mTLS + policy between microservices) extend this model to service-to-service traffic.

## OAuth 2.0

OAuth 2.0 delegates authorization ("let app X act on my behalf with these permissions") without sharing the user's password; OpenID Connect (OIDC) layers identity (authentication) on top of it.

- **Validate `redirect_uri` server-side** against a registered allowlist — never accept it as free-form input, or an attacker can redirect the auth code/token to their own server (open redirect → account takeover).
- **Use the Authorization Code flow, not Implicit** (`response_type=token`) — the implicit flow returns the access token directly in the URL fragment, exposed to browser history, referrer leaks, and any script on the page. Authorization Code exchanges a short-lived code for a token server-side, keeping the token out of the browser's URL.
- **`state` parameter** — a random, unguessable value tied to the user's session, checked on callback, to prevent CSRF against the OAuth flow itself (an attacker tricking a victim into linking the attacker's account).
- **PKCE (Proof Key for Code Exchange)** — mandatory for public clients (SPAs, mobile apps) that can't keep a client secret; binds the authorization code to the client that requested it, blocking code-interception attacks.
- **Scopes** — request and grant the minimum needed (`read:users`, not `admin:*`); validate scope on every resource server call, not just at token issuance.
- **Token introspection** — resource servers validate opaque tokens by calling back to the authorization server rather than trusting them blindly.

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.oauth2Login()
            .authorizationEndpoint()
            .authorizationRequestResolver(new CustomAuthorizationRequestResolver(clientRegistrationRepository()));
    }
}
```

Threats: redirect URI manipulation, authorization code interception/replay (mitigated by PKCE), and scope creep (an app silently accumulating broader access than it needs).

## Input processing

Bad input handling is the root of OWASP **API7 (SSRF)** and classic injection attacks.

- **Restrict HTTP methods** to what the operation actually needs — reject anything else with `405 Method Not Allowed`.
- **Validate `Content-Type`** — reject requests whose declared type doesn't match what the endpoint expects (`application/json` expected but `text/plain` sent, etc.) rather than trying to parse anyway.
- **Sanitize/validate all input** — type, length, format, allowed character set. Use a schema validation library (Joi in Node, Pydantic in Python, Hibernate Validator in Java) rather than ad hoc checks scattered through handler code.
- **`Authorization` header for credentials** — not custom headers or query strings (query strings end up in server logs, browser history, referrer headers).
- **Server-side encryption** — encrypt sensitive fields before they hit storage, don't rely on disk/transport encryption alone.
- **API gateway** for centralized rate limiting, caching, and WAF rules, so every service behind it inherits the same baseline.
- **File uploads** — route through a CDN/object store (S3 presigned URLs) rather than the app server; scan for malware; cap size; validate file type by content, not just extension.

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    name: str
    age: int = Field(gt=0)

@app.post("/users")
def create_user(user: User):
    # Pydantic validates types/constraints before this line ever runs
    return user
```

```javascript
app.use((req, res, next) => {
  if (!['GET', 'POST'].includes(req.method)) return res.status(405).send();
  next();
});
```

- **XXE (XML External Entity)** — disable external entity resolution in any XML parser; a permissive parser lets an attacker read local files or trigger SSRF via a crafted `<!ENTITY>`.
- **Insecure deserialization** — use allowlist-based (de)serializers (e.g., Jackson configured with a type deny/allow list) rather than generic object deserialization, which can be abused to instantiate arbitrary classes.
- **SQL/NoSQL/command injection** — always use parameterized queries/prepared statements; never string-concatenate user input into a query or shell command.

## Processing

Covers server-side execution hygiene — largely OWASP **API8: Security Misconfiguration**.

- **Auth-gate every non-public endpoint** — including internal/admin/debug routes that "shouldn't" be reachable; misconfiguration, not missing code, is usually the cause of exposure.
- **Don't expose internal IDs in URLs** (`/users/242/orders` leaks a sequential user ID) — prefer UUIDs, which are also non-enumerable.
- **Disable XML/YAML entity parsing** where not needed (see XXE above).
- **CDN for uploads** rather than serving user-provided files directly from the app server.
- **Async processing for large payloads** — offload to a queue (RabbitMQ, SQS) instead of blocking a request thread, which both improves resilience and avoids one large upload starving other requests.
- **Debug mode off in production** — verbose stack traces and debug endpoints leak internals (file paths, framework versions, sometimes secrets).
- **Non-executable stacks** where the OS/compiler supports it, as defense-in-depth against buffer-overflow style exploits.
- **Generic error messages externally**, detailed errors only in internal logs.

```java
@Entity
public class Order {
    @Id
    @GeneratedValue(generator = "uuid2")
    @GenericGenerator(name = "uuid2", strategy = "uuid2")
    private UUID id;
}
```

```python
app.run(debug=False)
```

Also worth automating: dependency vulnerability scanning (OWASP Dependency-Check, Snyk) and container image scanning with least-privilege runtime settings.

## Output

Prevents client-side attacks (XSS, clickjacking, MIME confusion) and data leakage from responses.

- **`X-Content-Type-Options: nosniff`** — stops browsers from guessing (sniffing) a different content type than declared, which can turn a file upload into executable script in some legacy scenarios.
- **`X-Frame-Options: DENY`** (or CSP `frame-ancestors 'none'`) — prevents the response being framed by another site (clickjacking).
- **`Content-Security-Policy: default-src 'none'`** for pure API responses — an API returning JSON has no legitimate need to load scripts/styles/images, so lock it down entirely. See `csp-owasp-server-security.md` for full CSP directive coverage.
- **Remove fingerprinting headers** — `X-Powered-By`, server version banners — these hand attackers a shortlist of known CVEs to try.
- **Force the `Content-Type`** on responses explicitly rather than letting the framework guess.
- **Never return sensitive fields** — password hashes, internal tokens, other users' PII — filter/redact at the serialization layer, not by hoping the frontend won't display them.
- **Correct status codes** — `204` for no-content success, `403` vs `404` deliberately (see the BOLA note below) — inconsistency itself can leak information.

```javascript
const helmet = require('helmet');
app.use(helmet());
// Adds X-Content-Type-Options, X-Frame-Options, a baseline CSP, etc.
```

```python
def get_user(id):
    user = fetch_user(id)
    return {k: v for k, v in user.items() if k != 'password'}
```

CORS headers belong here too — set them as narrowly as the client actually needs (see `cors.md`). A `403 Forbidden` vs `404 Not Found` distinction on an authorization failure is itself a minor information leak (it confirms the resource exists); some APIs deliberately return `404` for both "doesn't exist" and "exists but you can't see it" to avoid enumerating valid IDs to unauthorized callers.

## CI/CD and dependency hygiene

Security has to be a pipeline stage, not a pre-launch checklist.

- **Automated tests** — unit tests for auth logic, integration tests for full request flows; run API-focused scanners like OWASP ZAP against a staging environment.
- **Mandatory code review** — no self-approval, even for "small" changes; a second set of eyes catches logic-level auth bugs that tests miss.
- **SAST/DAST/SCA in the pipeline** — static analysis (SonarQube), dynamic analysis, and software composition analysis (Snyk) run automatically on every push, not manually before releases.
- **Automated dependency scanning** — Dependabot, Trivy, or similar catching known-CVE dependencies before they merge.
- **Rollback plan** — blue-green deploys or feature flags so a bad security-relevant change can be reverted in seconds, not hours.

```yaml
name: Security Scan
on: [push]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: snyk/actions/node@master
        with: { command: test }
```

Shift security left — catching a broken-auth bug in code review is orders of magnitude cheaper than catching it in production. Gate deploys on scan results rather than treating them as advisory.

## Monitoring

Detection is what catches OWASP **API9 (Improper Inventory Management)** and everything else that slips past prevention.

- **Centralized logging** — ELK stack, Splunk, or similar; log requests/responses/errors from every service in one place, correlated by request ID.
- **Instrumentation** — OpenTelemetry traces/metrics across services so a slow or failing request can be traced end-to-end.
- **Alerting** — thresholds on suspicious patterns (a spike in `401`s, unusual request volume from one IP) routed to Slack/email/PagerDuty.
- **Never log secrets** — tokens, passwords, full credit card numbers, or raw PII; mask/redact at the logging layer, and treat this as a compliance requirement (GDPR) as much as a security one.
- **IDS/IPS or WAAP** — Snort or a Web App and API Protection layer for runtime intrusion detection.
- **Continuous API inventory/discovery** — shadow APIs (deployed but undocumented) and zombie APIs (deprecated but still reachable) are a top real-world source of breaches precisely because nobody's watching them; automated discovery tools catch what documentation misses.

```python
import logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')
logging.info('Request: %s', request.path)  # never log body/headers with secrets
```

## OWASP API Security Top 10 (2023)

This is the standard reference for API-specific risk — distinct from the general OWASP Top 10 (which is web-app focused). Each entry below is a real, common vulnerability class with concrete cause and fix, not just a name.

1. **API1: Broken Object Level Authorization (BOLA / IDOR)**
   The most common and most damaging API vulnerability. An endpoint takes an object ID (`GET /orders/{id}`) and returns data without verifying the *authenticated* caller actually owns or is permitted to see that object — only that *some* valid token was presented. An attacker just increments or guesses IDs to pull other users' data.
   **Fix**: enforce an ownership/permission check on every object access, every single request — not once at login, not just at the route level. Prefer UUIDs over sequential IDs to make guessing harder (defense-in-depth, not a substitute for the authz check).

2. **API2: Broken Authentication**
   Weak, missing, or bypassable authentication — think reusable/never-expiring tokens, weak password policies, no brute-force protection, or JWT validation flaws (accepting `alg: none`, not checking `exp`).
   **Fix**: standard OAuth2/OIDC flows, strong credential storage (bcrypt/Argon2), MFA on sensitive operations, rate-limited login endpoints, and strict token validation (signature, expiry, issuer, audience) on every request.

3. **API3: Broken Object Property Level Authorization**
   Related to BOLA but at the *field* level: an endpoint correctly checks the user can see the object, but returns (or accepts updates to) properties they shouldn't touch — e.g., a `PATCH /users/me` that lets a normal user set their own `role: admin` field because the endpoint blindly accepts whatever JSON keys are sent (mass assignment), or a response that includes an internal `is_flagged_fraud` field never meant for the client.
   **Fix**: explicit allowlists for both readable and writable fields per role — never bind request bodies directly onto internal models.

4. **API4: Unrestricted Resource Consumption**
   No limits on request size, rate, or resource-intensive operations (large file uploads, expensive search queries, pagination with no max page size) lets a single client exhaust CPU, memory, or third-party API quota/cost.
   **Fix**: rate limiting, request size caps, pagination limits, and timeouts on expensive operations; consider cost-based throttling for endpoints that call metered third-party services.

5. **API5: Broken Function Level Authorization**
   Object-level checks might be fine, but the *endpoint itself* isn't protected — e.g., an admin-only `DELETE /users/{id}` endpoint exists and works for any authenticated user because nobody checks role before executing it, only that a valid token was sent.
   **Fix**: RBAC/ABAC enforced centrally (middleware/gateway) per endpoint and HTTP method, "deny by default," tested explicitly for privilege escalation (can a normal-role token call an admin-role endpoint?).

6. **API6: Unrestricted Access to Sensitive Business Flows**
   The API works exactly as designed, but the business logic itself can be abused at scale — e.g., no limit on how many times a user can apply a discount code, or a ticket-purchasing API with no bot/rate protection that lets scalpers buy out inventory in seconds.
   **Fix**: identify sensitive flows during threat modeling (not just technical checks); apply CAPTCHA, device fingerprinting, or business-rule limits (e.g., "one redemption per account") specifically to those flows.

7. **API7: Server-Side Request Forgery (SSRF)**
   Any endpoint that fetches a URL supplied (directly or indirectly) by the client — webhook registration, "import from URL," image proxies — can be tricked into making the *server* issue requests to internal-only resources (`http://169.254.169.254/` cloud metadata endpoints, internal admin panels) that the attacker couldn't reach directly.
   **Fix**: allowlist permitted destination hosts/schemes, block requests to private/link-local IP ranges, and don't follow redirects blindly (a redirect can retarget an allowlisted URL to an internal one).

8. **API8: Security Misconfiguration**
   The broadest category: default credentials left in place, verbose stack traces in production, unnecessary HTTP methods enabled, permissive CORS, missing security headers, outdated software with known CVEs, cloud storage buckets left public.
   **Fix**: hardened, version-controlled configuration; automated config auditing (CIS benchmarks); disable debug mode and defaults before shipping; consistent config across all environments.

9. **API9: Improper Inventory Management**
   Old API versions (`/v1/users` still live after `/v2` shipped), staging/test endpoints reachable from the internet, or endpoints entirely undocumented and unknown to the security team — attackers actively scan for exactly this kind of forgotten surface, and it's often less hardened than the current, documented API.
   **Fix**: maintain a live API inventory (auto-generated from OpenAPI specs where possible), deprecate and actually remove old versions, and never expose non-production environments publicly.

10. **API10: Unsafe Consumption of APIs**
    Trusting data or behavior from third-party/upstream APIs without validation — assuming a partner API's response is well-formed and safe just because it's "internal" or "trusted," which can propagate injection payloads or malformed data into your own system.
    **Fix**: validate and sanitize responses from external APIs exactly as you would client input; apply timeouts, circuit breakers, and allowlisted redirect handling to any outbound API integration.

## Advanced topics

- **API posture governance** — structured, ongoing management of the API lifecycle (design → deploy → deprecate) rather than one-time security review; tools like Traceable specialize in this.
- **GraphQL-specific security** — a single flexible endpoint changes the threat model: enforce query depth/complexity limits (a deeply nested query can be a DoS vector), disable introspection in production, and rate-limit by query cost rather than just request count.
- **Serverless APIs** — least-privilege IAM roles per function (a function should only have the exact permissions it needs, not a shared broad role), and monitor invocation patterns for anomalies since traditional network-perimeter monitoring doesn't apply.
- **AI/ML-backed APIs** — validate inputs rigorously against prompt injection when an endpoint forwards user input to an LLM; treat model output as untrusted input to whatever consumes it next, not as safe-by-construction.
- **Compliance** — PCI-DSS mandates encryption of card data in transit and at rest; GDPR requires minimizing and anonymizing logged personal data, with a lawful basis for processing.
- **WAAP/WAF** — a Web App and API Protection layer in front of the API for runtime blocking of known attack patterns, complementing (not replacing) in-app fixes.

## Tools and resources

- **Testing**: Postman, Burp Suite, OWASP ZAP.
- **Gateways**: Kong, Apigee, AWS API Gateway.
- **Monitoring**: Datadog, New Relic, ELK/Splunk.
- **Reference material**: OWASP API Security Project (Top 10 + cheat sheets), NIST SP 800-204C, OWASP REST Security Cheat Sheet.

## Quick reference

| Layer | Key controls |
|---|---|
| Authentication | OAuth2/OIDC or JWT, MFA, bcrypt/Argon2, rate-limited login |
| Authorization | Per-object + per-function checks on every request, RBAC/ABAC, deny by default |
| Input | Schema validation, method/content-type restriction, parameterized queries |
| Output | Security headers (Helmet/CSP), field allowlisting, generic errors |
| Transport | TLS 1.3, HSTS, mTLS for service-to-service |
| Operations | Centralized logging (no secrets), rate limiting, dependency scanning, API inventory |
