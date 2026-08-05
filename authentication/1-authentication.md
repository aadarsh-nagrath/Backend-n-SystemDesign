# Authentication mechanisms

Authentication answers "who are you?" — verifying the identity of a user, device, or system before granting access. It's distinct from **authorization** ("what are you allowed to do?"), which happens after authentication succeeds. This file is the catalog of concrete mechanisms: how each one works on the wire, when to use it, and its trade-offs. For the underlying theory (factors, threat models, architecture patterns) see [4-Authentication Concepts and Theory.md](<4-Authentication Concepts and Theory.md>); for JWT and OAuth 2.0 specifics see their dedicated files.

## TL;DR
- **Basic Auth**: username:password Base64-encoded in a header. Simple, stateless, insecure without HTTPS, no revocation.
- **Session-based**: server stores session state, client holds an opaque session ID cookie. Easy to revoke, but stateful (doesn't scale horizontally without a shared store).
- **Token-based / JWT**: client holds a self-contained signed token. Stateless and scalable, but hard to revoke before expiry. Full JWT depth: [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>).
- **OAuth 2.0**: delegated authorization — a third-party app gets scoped access without your password. Full depth: [3-Oauth Guide.md](<3-Oauth Guide.md>).
- **SSO**: authenticate once with an identity provider, access many apps. Protocol details (SAML/OIDC): [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>).
- **MFA / Biometric / API Key**: covered in depth below — each solves a different problem (extra factor, device-bound identity, machine-to-machine access).

## Comparison at a glance

| Mechanism | Stateful/Stateless | Scalability | Security | Revocation | Best fit |
|-----------|--------------------|-------------|----------|------------|----------|
| Basic Auth | Stateless | High | Low (without HTTPS) | N/A | Simple APIs, server-to-server |
| Session-based | Stateful | Moderate | Moderate (CSRF risk) | Easy | Traditional web apps |
| Token-based | Stateless | High | Moderate (token theft) | Difficult | APIs, SPAs, microservices |
| JWT | Stateless | High | Moderate (payload exposure) | Difficult | APIs, microservices, cross-domain |
| OAuth 2.0 | Stateless | High | High (with best practices) | Moderate (refresh tokens) | Third-party access, APIs, SSO |
| SSO (SAML/OIDC) | Varies | Moderate | High (with IdP security) | Easy | Enterprise apps, federated identity |
| Cookie-based | Stateful | Moderate | Moderate (CSRF risk) | Easy | Browser-based web apps |
| MFA | Varies | Varies | Very high | Varies | High-security, compliance-driven apps |
| Biometric | Varies | Varies | High | N/A | Mobile/device authentication |
| API Key | Stateless | High | Low | Moderate | Public APIs, server-to-server |

## 🟢 Basic Authentication

RFC 7617. The client sends `username:password` Base64-encoded in the `Authorization` header. It's one of the oldest HTTP auth mechanisms — no session, no token, just credentials on every request.

```http
GET /api/protected HTTP/1.1
Host: example.com
Authorization: Basic YWRtaW46c2VjcmV0
```

`YWRtaW46c2VjcmV0` is just `admin:secret` Base64-encoded — **not encrypted**. Anyone who intercepts the request over plain HTTP reads the password directly.

**How it works**: client combines credentials as `username:password`, Base64-encodes the string, sends it with the `Basic` prefix. Server decodes, checks against stored (hashed) credentials, returns `200` or `401`.

**Advantages**: trivial to implement, universally supported, stateless (no server-side session storage).

**Disadvantages**: insecure without HTTPS (Base64 is encoding, not encryption), no session management (credentials resent every request, widening the exposure window), no scopes/permissions, and it trains clients to store raw passwords — a genuine anti-pattern.

**Use it for**: quick prototyping, low-security internal APIs, server-to-server calls where credentials live in a secrets manager, not a browser.

```javascript
const express = require('express');
const app = express();

app.get('/protected', (req, res) => {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith('Basic ')) {
    res.set('WWW-Authenticate', 'Basic realm="Restricted Area"');
    return res.status(401).send('Authentication required');
  }

  const credentials = Buffer.from(authHeader.split(' ')[1], 'base64').toString('ascii');
  const [username, password] = credentials.split(':');

  if (username === 'admin' && password === 'secret') { // replace with real DB check + hash compare
    res.json({ message: 'Access granted' });
  } else {
    res.status(401).send('Invalid credentials');
  }
});

app.listen(3000);
```

## 🟢 Session-based authentication

Stateful: the server creates and owns a session record; the client only holds a random session ID (usually in a cookie).

**Flow**:
1. Client submits credentials to `/login`.
2. Server validates, creates a session record (user ID, expiry, etc.) in a store (Redis, DB), generates a random session ID.
3. Server responds with `Set-Cookie: session_id=...`.
4. Browser auto-attaches the cookie to every subsequent request; server looks up the session ID in its store to authenticate.
5. Logout / expiry / admin action destroys the session server-side — instantly invalidating it.

```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=john&password=pass123

HTTP/1.1 200 OK
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
```

**Advantages**: instant server-side revocation (delete the session record), sensitive data never leaves the server, mature framework support (Express, Django, Rails all have this built in).

**Disadvantages**: stateful — needs a shared session store (Redis) once you have more than one app server, adding latency and an operational dependency; vulnerable to CSRF unless mitigated; depends on cookies being enabled.

**Security musts**: HTTPS, `HttpOnly` (blocks JS access, mitigates XSS token theft), `Secure` (cookie only sent over TLS), `SameSite=Strict` or `Lax` (mitigates CSRF), reasonable expiry, CSRF tokens on state-changing forms.

```javascript
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis')(session);
const redis = require('redis');

const app = express();
const redisClient = redis.createClient();

app.use(express.urlencoded({ extended: true }));
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, httpOnly: true, sameSite: 'strict', maxAge: 24 * 60 * 60 * 1000 }
}));

app.post('/login', (req, res) => {
  const { username, password } = req.body;
  if (username === 'john' && password === 'pass123') { // replace with real check
    req.session.userId = '12345';
    res.redirect('/dashboard');
  } else {
    res.status(401).send('Invalid credentials');
  }
});

app.get('/logout', (req, res) => {
  req.session.destroy(() => res.redirect('/login'));
});
```

**Best fit**: traditional server-rendered web apps, anything needing instant revocation (banking, admin panels), centralized architectures with an existing session store.

## 🟡 Token-based authentication (general)

Stateless: the server issues a token at login; the token itself (not a server-side lookup) proves identity on every subsequent request. JWT (below) is the dominant concrete implementation, but the pattern is broader — an opaque random string checked against a database works too.

**Flow**: client sends credentials → server validates and issues a token → client stores it (memory, cookie, or localStorage — see storage trade-offs in the [JWT file](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>)) → client sends the token in `Authorization: Bearer <token>` on every request → server validates it (signature check for JWTs, DB lookup for opaque tokens) without needing a session store.

```http
POST /login HTTP/1.1
Content-Type: application/json

{"username":"john","password":"pass123"}

200 OK
{"token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}

GET /api/protected HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Advantages**: no server-side session storage, scales cleanly across many stateless servers, flexible storage on the client.

**Disadvantages**: a stolen token is valid until it expires (mitigate with short lifetimes + refresh tokens), revocation before expiry needs extra machinery (denylists), payloads can bloat request size if overloaded with claims.

**Best fit**: REST/GraphQL APIs, SPAs, mobile apps, cross-domain auth. For the full mechanics of the JWT variant — structure, signing, revocation strategies, algorithm attacks, storage trade-offs — see [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>).

## 🟡 JWT authentication — summary

JWTs (RFC 7519) are the standard structured token: `header.payload.signature`, each Base64Url-encoded, self-contained and digitally signed. They're the concrete mechanism behind most modern "token-based" and OAuth 2.0 bearer-token systems.

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Full structure breakdown, signing algorithms, claim reference, revocation strategies (short expiry + refresh rotation + denylists), the `alg: none` and algorithm-confusion attacks, and storage trade-offs (localStorage XSS vs httpOnly cookie CSRF) live in [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>) — read that file for anything beyond "what does a JWT look like."

## 🟡 OAuth 2.0 — summary

OAuth 2.0 (RFC 6749) is a delegated **authorization** framework: a third-party client gets scoped, revocable access to a user's resources without ever seeing the user's password. It defines four roles (resource owner, client, resource server, authorization server) and several grant flows (authorization code + PKCE, client credentials, device code; implicit and password grants are deprecated).

```http
GET /oauth/authorize?response_type=code&client_id=CLIENT_ID&redirect_uri=https://client.com/callback&scope=read&state=xyz123

302 Found
Location: https://client.com/callback?code=AUTH_CODE&state=xyz123
```

Full flow-by-flow breakdown, PKCE mechanics, the flow decision table, tokens, and security considerations live in [3-Oauth Guide.md](<3-Oauth Guide.md>) — read that file for anything beyond the two-paragraph summary above. Note OAuth 2.0 alone is *authorization*, not authentication — [OpenID Connect](<Advanced Authentication Concepts.md>) adds the identity layer on top.

## 🟡 Single sign-on (SSO) — summary

SSO lets a user authenticate once with an identity provider (IdP) and access many independent applications without re-authenticating, typically via SAML or OpenID Connect (OIDC, itself built on OAuth 2.0).

```http
GET /oauth/authorize?response_type=code&client_id=CLIENT_ID&redirect_uri=https://app.com/callback&scope=openid email&state=xyz123
Host: idp.com
```

**Advantages**: one login for many apps, centralized access management/revocation, consistent security policy across the org. **Disadvantages**: the IdP becomes a single point of failure, and all connected apps depend on its availability. Protocol-level detail on SAML and OIDC (assertions, bindings, claims, flows) lives in [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>).

**Best fit**: enterprise applications (Office 365, Salesforce), cross-org federated identity.

## 🟡 Cookie-based authentication

A special case of session-based auth where the session identifier specifically lives in a browser cookie — worth calling out separately because the cookie flags carry most of the security weight.

```http
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=86400
```

- **`HttpOnly`**: blocks JavaScript from reading the cookie — the primary defense against XSS stealing the session token.
- **`Secure`**: cookie is only sent over HTTPS.
- **`SameSite=Strict|Lax`**: browser withholds the cookie on cross-site requests, mitigating CSRF. `Strict` blocks it even on top-level navigation from another site (e.g. clicking a link into your app while logged in elsewhere loses the cookie); `Lax` (the modern default) allows it on top-level GET navigations but blocks it on cross-site POSTs/forms/fetches — the common CSRF vector.
- **`Max-Age` / `Expires`**: bounds exposure if the cookie leaks.

**Disadvantages**: still needs CSRF tokens for defense-in-depth on state-changing requests even with `SameSite`, still stateful, and depends on cookies being enabled (rare to be disabled today, but some embedded/webview contexts restrict them).

**Best fit**: any browser-first web app wanting the browser to handle credential transport automatically.

## 🟡 Multi-factor authentication (MFA)

MFA layers a second (or third) independent factor on top of primary authentication, drawn from a different category so that compromising one factor alone isn't enough.

**Factors**:
- **Knowledge** — password, PIN, security question.
- **Possession** — phone with an authenticator app, hardware key (YubiKey), SMS/email code.
- **Inherence** — fingerprint, face, voice.
- **Context** (increasingly used as a soft fourth factor) — IP address, device fingerprint, time of day, feeding risk-based/adaptive MFA.

**Flow**: user passes primary auth (e.g. password) → server challenges for a second factor (TOTP code, push approval, hardware key tap) → server validates both before issuing a session/token.

```javascript
const express = require('express');
const speakeasy = require('speakeasy');
const app = express();
app.use(express.json());

app.post('/mfa/setup', (req, res) => {
  const secret = speakeasy.generateSecret({ length: 20 });
  res.json({ secret: secret.base32 }); // show as QR code to the user
});

app.post('/mfa/verify', (req, res) => {
  const { token, secret } = req.body;
  const verified = speakeasy.totp.verify({ secret, encoding: 'base32', token, window: 1 });
  res.status(verified ? 200 : 401).json({ verified });
});
```

TOTP (`window: 1`) accepts codes from one 30-second step before/after the current one, absorbing clock drift without materially weakening the 30-second window.

**Trade-offs**: significantly cuts credential-stuffing and phishing success rates and satisfies most compliance regimes (PCI-DSS effectively mandates it), but adds login friction and requires a recovery path (backup codes) for lost devices. See [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>) for how MFA compares to phishing-resistant passkeys/WebAuthn.

## 🟡 Biometric authentication

Uses a physical or behavioral trait — fingerprint, face, voice — as a factor. In practice almost always device-local: the biometric template never leaves the device's secure enclave/TEE, and the server only ever sees a cryptographic signature (this is the same trust model WebAuthn uses — see [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>)).

**Flow**: enroll (capture and store a template in secure hardware) → authenticate (compare live sample to template on-device) → device asserts success to the app/server, typically by unlocking a private key used to sign a challenge.

**Advantages**: fast, convenient, hard to replicate at scale. **Disadvantages**: irrevocable if the underlying template is ever compromised (you can't rotate a fingerprint), false accept/reject rates depend on sensor quality, and it always needs a fallback method (PIN/password) for enrollment failures or sensor issues.

**Best fit**: mobile device unlock, step-up auth combined with WebAuthn/passkeys, not usually a sole/standalone factor for high-value actions.

## 🟢 API key authentication

A static, opaque credential issued to a client (usually another service, not a human user) and passed on every request.

```http
GET /api/data?api_key=abc123 HTTP/1.1
```

Passing it as a query parameter is common but leaks the key into server logs, browser history, and referrer headers — prefer a header (`X-API-Key: abc123` or `Authorization: Bearer <key>`).

**Advantages**: trivial to generate and use, stateless, minimal integration burden for API consumers.

**Disadvantages**: keys are static (don't expire unless manually rotated), easy to leak (committed to a public repo, logged accidentally), and carry no user-specific context — an API key authenticates a *client*, not a person, so it's a poor fit for anything needing per-user authorization.

**Best fit**: public APIs, server-to-server integrations, developer tooling — always with rotation, per-key scoping, and rate limiting.

```javascript
const express = require('express');
const app = express();
const VALID_API_KEY = 'abc123';

app.get('/api/data', (req, res) => {
  const apiKey = req.headers['x-api-key'];
  if (apiKey === VALID_API_KEY) { // real implementation: hash-compare against a DB
    res.json({ data: 'Protected data' });
  } else {
    res.status(401).send('Invalid API key');
  }
});
```

## 🔴 Choosing a mechanism

- **Security needs dominate** → MFA and OAuth 2.0 + OIDC, layered.
- **Scalability dominates** → JWT, OAuth 2.0, API keys (all stateless).
- **User experience dominates** → SSO and passkeys/biometrics reduce friction most.
- **Architecture shape**: monoliths lean session/cookie-based; microservices lean JWT/OAuth 2.0, since there's no single shared session store to check.
- **Compliance** (GDPR, HIPAA, PCI-DSS): MFA and SSO with strong IdP security satisfy most audit requirements out of the box.

For most modern applications, **OAuth 2.0 + OpenID Connect, layered with MFA (or passkeys) for sensitive operations**, is the default answer — it balances security, scalability, and delegated access without forcing you to build session infrastructure. Pure session-based auth remains the right, simpler choice for a monolithic server-rendered app with no third-party integration needs.

## Further reading
- RFC 7617 (Basic Authentication), RFC 7519 (JWT), RFC 6749 (OAuth 2.0).
- [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>)
- [3-Oauth Guide.md](<3-Oauth Guide.md>)
- [4-Authentication Concepts and Theory.md](<4-Authentication Concepts and Theory.md>)
- [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>)
