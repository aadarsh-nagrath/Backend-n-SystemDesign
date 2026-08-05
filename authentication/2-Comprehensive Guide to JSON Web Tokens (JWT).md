# JSON Web Tokens (JWT)

JWT (RFC 7519) is a compact, URL-safe, digitally signed format for transmitting claims between two parties. It's the dominant concrete implementation of "token-based authentication" (see [1-authentication.md](<1-authentication.md>) for how it fits among other mechanisms) and the usual format for OAuth 2.0 access/ID tokens (see [3-Oauth Guide.md](<3-Oauth Guide.md>)).

## TL;DR
- Structure: `header.payload.signature`, each part Base64Url-encoded, dot-separated.
- **Signed, not encrypted** — anyone can decode and read the payload. Never put secrets in it.
- Stateless: the signature alone proves integrity, so the server doesn't need to store session data.
- The hard problem is **revocation** — a valid JWT stays valid until it expires, no matter what happens server-side. Short-lived access tokens + refresh token rotation + a denylist for the rare "kill this token now" case is the standard answer.
- Storage is a genuine trade-off, not a solved problem: localStorage is vulnerable to XSS exfiltration, httpOnly cookies are vulnerable to CSRF (mitigated with `SameSite`).
- Watch for algorithm confusion attacks and the `alg: none` attack — both are about the *verifier* trusting attacker-controlled input about how to verify the token.

## Structure of a JWT

Three Base64Url-encoded parts joined by dots: `xxxxx.yyyyy.zzzzz`.

### 1. Header
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
`alg` names the signing algorithm; `typ` is always `"JWT"`.

### 2. Payload
Claims — statements about the entity plus metadata. Three categories:
- **Registered claims** (predefined, optional but recommended): `iss` (issuer), `sub` (subject/user ID), `aud` (audience), `exp` (expiration, Unix timestamp), `iat` (issued at), `nbf` (not before), `jti` (unique token ID — used for revocation and replay prevention).
- **Public claims**: registered in the IANA JSON Web Token Registry for cross-application reuse.
- **Private claims**: application-specific, e.g. `roles`, `permissions`.

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622,
  "roles": ["admin"]
}
```

### 3. Signature
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```
Signs the concatenated header+payload with the algorithm named in the header, using a shared secret (symmetric, e.g. HS256) or a private key (asymmetric, e.g. RS256/ES256). The signature is what makes tampering detectable — flip one bit in the payload and verification fails — but it does **not** hide the payload's contents. Anyone can Base64-decode the first two segments and read them in plaintext; paste any JWT into jwt.io to see this directly.

### Signing algorithms
- **Symmetric (HS256, HS384, HS512)**: one secret key both signs and verifies. Simple and fast, but every service that verifies tokens must hold the same secret — a leak anywhere lets an attacker forge tokens.
- **Asymmetric (RS256, ES256, PS256)**: a private key signs, a public key verifies. The verifying service only ever needs the public key, so a compromised resource server can't forge new tokens — better fit for distributed systems and third-party verification (this is what OIDC ID tokens use).

## How JWT auth works end to end

1. **Login**: client sends credentials; server validates against its user store.
2. **Issue**: server builds header + payload, signs it, returns the JWT.
3. **Store**: client stores the JWT (see storage trade-offs below).
4. **Use**: client sends `Authorization: Bearer <JWT>` on each request.
5. **Verify**: server checks the signature, then validates claims (`exp`, `nbf`, `aud`, `iss`) — signature validity alone is not enough, an expired-but-correctly-signed token must still be rejected.
6. **Expire**: short-lived access tokens force periodic re-issuance via a refresh token.

```javascript
const jwt = require('jsonwebtoken');

const payload = {
  sub: '1234567890',
  name: 'John Doe',
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + (15 * 60), // 15 minutes
  roles: ['admin']
};

const token = jwt.sign(payload, process.env.JWT_SECRET, { algorithm: 'HS256' });

// Verification — always pin the expected algorithm(s) explicitly
const decoded = jwt.verify(token, process.env.JWT_SECRET, { algorithms: ['HS256'] });
```

## JWT vs. session-based authentication

| Aspect | Session-based | JWT-based |
|--------|---------------|-----------|
| State | Stateful (server stores session data) | Stateless (data lives in the token, on the client) |
| Storage | Server-side (DB/cache) | Client-side (cookie, memory, localStorage) |
| Scalability | Limited by the shared session store | High — no server storage needed |
| Revocation | Easy — delete the session record | Hard — see revocation strategies below |
| Main risk | CSRF, session hijacking | Token theft, payload exposure |
| Best fit | Centralized apps needing instant revocation | Distributed APIs, microservices, SPAs |

Use session-based when you need instant, guaranteed revocation (banking, admin tools) or already have a centralized session store. Use JWT when you need statelessness across many servers/services and can tolerate revocation being "eventually enforced" rather than instant.

## 🔴 The revocation problem

This is the central trade-off of JWTs and worth understanding precisely, not just as a bullet point. A session ID is a pointer to server-side state — delete the record, the session is dead everywhere, instantly. A JWT is *not* a pointer; it's a self-contained, cryptographically valid statement of fact ("this user has these roles until this timestamp"). The server that issued it has no ongoing say in whether it's still "live" — any server holding the verification key will accept it right up until `exp`, even if you've since banned the user, they've logged out, or their device was stolen. There is no way to make a stateless token "not stateless" after the fact without reintroducing state somewhere.

Given that, revocation is handled with a layered strategy rather than one mechanism:

**1. Short-lived access tokens (the primary defense)**
Make `exp` short — 5 to 15 minutes is typical. This bounds the maximum damage window of a stolen or "revoked" token to a small, known interval. It doesn't solve revocation; it shrinks the blast radius so that most of the time you don't need to solve it.

**2. Refresh tokens, rotated on every use**
A long-lived refresh token (stored server-side, associated with a user/device) is exchanged for a new short-lived access token when the old one expires. Rotation means: every time a refresh token is used, the server invalidates it and issues a brand-new one. This turns theft detection into something actionable — if a stolen refresh token gets used *after* the legitimate client already rotated it, the server sees a reused/invalid refresh token and can treat that as a signal to revoke the entire token family (all descendant tokens from that refresh chain), not just the one token. Without rotation, a stolen refresh token is a standing backdoor for its full lifetime.

```javascript
// Simplified refresh rotation
async function refreshAccessToken(oldRefreshToken) {
  const record = await db.refreshTokens.findOne({ token: oldRefreshToken });
  if (!record || record.revoked) {
    // Reuse of an already-rotated or revoked token — treat as compromise
    await db.refreshTokens.revokeFamily(record?.familyId);
    throw new Error('Refresh token reuse detected — session family revoked');
  }
  await db.refreshTokens.revoke(record.id);
  const newRefreshToken = await db.refreshTokens.create({ familyId: record.familyId, userId: record.userId });
  const newAccessToken = signAccessToken({ sub: record.userId });
  return { accessToken: newAccessToken, refreshToken: newRefreshToken };
}
```

**3. Denylists (blacklists) for the "kill this token right now" case**
For genuine immediate revocation (user clicks "log out everywhere," admin bans an account, a token is known-compromised), maintain a store — Redis with a TTL matching the token's remaining `exp` is the standard choice — of revoked `jti` values. Every verification does an extra lookup: signature valid *and* `jti` not in the denylist. This reintroduces a stateful check, which is exactly the cost JWTs were meant to avoid — so it's used sparingly, for the minority of tokens that genuinely need instant kill, not as the default path for every request.

```javascript
async function verifyWithDenylist(token) {
  const decoded = jwt.verify(token, SECRET, { algorithms: ['HS256'] });
  if (await redis.exists(`revoked:${decoded.jti}`)) {
    throw new Error('Token has been revoked');
  }
  return decoded;
}

// On logout / ban:
async function revokeToken(jti, exp) {
  const ttlSeconds = exp - Math.floor(Date.now() / 1000);
  if (ttlSeconds > 0) await redis.set(`revoked:${jti}`, '1', 'EX', ttlSeconds);
}
```

**4. Versioned claims as a coarser alternative**
Store a `tokenVersion` (or `passwordChangedAt`) on the user record and embed it as a claim. Bump the version on password change/ban; verification checks the claim's version against the current DB value. This revokes *all* of a user's outstanding tokens at once with one write, without needing a growing denylist — coarser than per-token revocation but much cheaper to operate.

| Strategy | Revocation granularity | Latency to take effect | Operational cost |
|---|---|---|---|
| Short expiry alone | None (just waits it out) | Up to `exp` | Free |
| Refresh rotation | Per refresh-token family | Next refresh attempt | Low (one DB table) |
| Denylist | Per token (`jti`) | Immediate | Moderate (extra lookup every request) |
| Version claim | Per user (all tokens) | Immediate | Low (one field, one lookup) |

## 🔴 Algorithm attacks

### The `alg: none` attack
The JWT spec technically allows `"alg": "none"` — an unsigned token. If a server's verification code doesn't explicitly reject this, an attacker can take any JWT, change the payload (e.g. `"roles": ["admin"]`), set the header to `{"alg":"none","typ":"JWT"}`, drop the signature segment entirely, and submit `header.payload.` — a forged, "verified" admin token, no key required.

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhdHRhY2tlciIsInJvbGVzIjpbImFkbWluIl19.
```

**Mitigation**: never trust the `alg` value from the token itself as the algorithm to verify with. Modern libraries (`jsonwebtoken`, etc.) require you to pass an explicit allowlist of acceptable algorithms to `verify()`, and reject `none` by default — but only if you actually pass that option:

```javascript
// Vulnerable pattern — do not do this:
jwt.verify(token, secretOrPublicKey); // some older/looser libraries infer alg from the token

// Safe pattern — always pin the allowed algorithm(s) explicitly:
jwt.verify(token, secretOrPublicKey, { algorithms: ['RS256'] });
```

### Algorithm confusion (RS256 → HS256)
A more subtle version of the same root cause. Say a server signs tokens with RS256 (asymmetric: private key signs, public key verifies) and publishes its public key (common — OIDC providers expose these at a JWKS endpoint for anyone to fetch). If the verification code says "use whatever `alg` the token header claims," an attacker can:

1. Take the server's known-public RS256 public key.
2. Forge a new token, set the header to `"alg": "HS256"` (symmetric).
3. Sign the forged token using the public key *as if it were an HMAC secret*.
4. Send it to a server whose verification logic naively does `jwt.verify(token, publicKey)` without pinning the algorithm.

The server, told by the (attacker-controlled) header to treat this as HS256, uses its public key as the HMAC secret to verify — and since the attacker signed it with that exact same string, verification succeeds. The attacker has turned a public key into a symmetric secret the server unknowingly trusts.

**Mitigation**: identical to `alg:none` — pin the expected algorithm explicitly server-side and never let the token's own header dictate verification behavior. If you support both symmetric and asymmetric tokens in the same system (rare, and worth avoiding), use separate verification code paths keyed off which *type* of credential issued the token, not off what the token claims about itself.

### Other JWT-specific risks
- **Weak/guessable HMAC secrets**: HS256 security is only as strong as the secret. Use a long, random, high-entropy secret (32+ bytes) — a short or dictionary-guessable secret can be brute-forced offline against a captured token.
- **Missing claim validation**: verifying the signature is necessary but not sufficient — also check `exp`, `nbf`, `aud` (is this token meant for *this* service?), and `iss` (did *this* trusted issuer create it?). Skipping `aud` validation is a common cause of token confusion between services that share a signing key.
- **Replay**: a stolen-but-not-yet-expired token can be replayed freely since JWTs carry no built-in single-use guarantee. `jti` + a short-lived denylist (or requiring `jti` uniqueness for particularly sensitive operations) mitigates this.

## 🔴 Storage: where does the client keep the JWT?

There's no storage location that's simply "safe" — each has a different attacker it's vulnerable to, and the right choice depends on what you're already defending against.

| Storage | Vulnerable to | Notes |
|---|---|---|
| `localStorage` / `sessionStorage` | **XSS**: any injected script can read `localStorage` and exfiltrate the token | Not sent automatically — must be manually attached to each request, which is why SPAs like it. But if an attacker gets *any* script execution on your page (a compromised dependency, an unsanitized user-generated field), they read every token in storage, no cookie flags can stop them |
| `httpOnly` cookie | **CSRF**: browser auto-attaches the cookie to *any* request to your domain, including ones triggered by a malicious third-party page | JavaScript cannot read an `httpOnly` cookie at all, so XSS can't directly exfiltrate it — but XSS can still *use* it by making authenticated requests through the victim's own browser, it just can't steal the raw token value |
| In-memory (JS variable, never persisted) | Lost on page refresh/tab close; still readable by XSS while the page is open | Best resistance to long-term exfiltration, worst UX (re-auth on every reload) — often paired with a refresh token in an `httpOnly` cookie so refresh survives reloads |

**Practical recommendation, in order of preference for browser apps**:
1. **Access token in memory, refresh token in an `httpOnly`, `Secure`, `SameSite=Strict` cookie.** The short-lived access token being memory-only limits exfiltration value (it's gone on reload and expires in minutes anyway); the long-lived refresh token is XSS-proof because JS can't read it. Mitigate the residual CSRF exposure on the refresh endpoint with a `SameSite=Strict` cookie (blocks it being sent cross-site at all) plus a CSRF token or requiring the refresh call to be same-origin.
2. **httpOnly cookie for everything**, if you don't need the token accessible to JS at all (e.g. no cross-origin API calls) — simplest, and pairs `SameSite=Strict`/`Lax` with standard CSRF-token defense on state-changing endpoints.
3. **localStorage** is the common real-world default for pure SPA/mobile-backend architectures, but it means your XSS defense (strict CSP, output encoding, dependency hygiene) *is* your token-theft defense — there's no cookie flag backstopping it. Treat any XSS finding in an app using localStorage tokens as a full account-takeover bug, not a low-severity issue.

The underlying trade-off doesn't disappear by picking one option — it moves. Cookies push the risk to CSRF (which `SameSite` has made much easier to mitigate over the last few years); JS-accessible storage pushes the risk to XSS (which is harder to fully rule out in any app rendering any user or third-party content).

## JSON Web Encryption (JWE)

JWT signs; it does not encrypt. If a payload must carry data that shouldn't be readable by anyone holding the token (rare — usually better to just not put sensitive data in the token and look it up server-side instead), JWE encrypts the claims. Structure: `header.encrypted_key.iv.ciphertext.authentication_tag`, five segments instead of three. Supports symmetric (A256GCM) and asymmetric (RSA-OAEP) encryption. JWTs are commonly nested inside a JWE for "signed then encrypted" when both integrity and confidentiality are required.

## JWT in OAuth 2.0 and OIDC

- **OAuth 2.0**: access tokens are frequently implemented as JWTs so the resource server can verify them locally (checking the signature against the authorization server's public key) without a network round-trip back to the authorization server on every request.
- **OpenID Connect**: the `id_token` is *always* a JWT, carrying identity claims (`sub`, `email`, `name`, ets.) about the authenticated user — this is the piece that turns OAuth's authorization into actual authentication. See [3-Oauth Guide.md](<3-Oauth Guide.md>) for the full OIDC flow.

## Best practices checklist

1. Short expiry on access tokens (5–15 min); long-lived, rotated refresh tokens for renewal.
2. Explicitly pin allowed algorithm(s) on every `verify()` call — never trust the token's own `alg` header.
3. Validate every relevant claim (`exp`, `nbf`, `aud`, `iss`), not just the signature.
4. Never put secrets, passwords, or anything sensitive in the payload — it's readable by anyone, always.
5. Prefer asymmetric algorithms (RS256/ES256) when multiple services need to verify tokens independently.
6. Use a high-entropy secret for HMAC algorithms; rotate signing keys periodically (supporting overlapping old/new keys during rotation via a `kid` header claim).
7. Keep payloads minimal — every claim adds bytes to every request.
8. Pick a storage strategy deliberately based on your XSS/CSRF threat model, not by default.
9. Layer denylisting or version claims for the subset of tokens that genuinely need instant revocation; don't try to make every token instantly revocable, that's what session-based auth is for.
10. Always transmit over HTTPS.

## Practical implementation

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');

const app = express();
app.use(express.json());
const secret = process.env.JWT_SECRET;

app.post('/login', (req, res) => {
  const { username, password } = req.body;
  if (username === 'user' && password === 'pass') { // replace with real check
    const payload = {
      sub: '1234567890',
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + (15 * 60),
      roles: ['admin']
    };
    const token = jwt.sign(payload, secret, { algorithm: 'HS256' });
    res.json({ access_token: token });
  } else {
    res.status(401).json({ error: 'Invalid credentials' });
  }
});

app.get('/protected', (req, res) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Token missing' });

  try {
    const decoded = jwt.verify(token, secret, { algorithms: ['HS256'] });
    res.json({ message: 'Access granted', user: decoded });
  } catch (err) {
    res.status(403).json({ error: 'Invalid token' });
  }
});

app.listen(3000);
```

## Common use cases
1. API authentication (bearer tokens for REST/GraphQL).
2. SSO — a JWT issued by an auth service, verified by multiple downstream apps sharing a public key.
3. Microservices — no shared session store needed across services.
4. Mobile apps and serverless functions — stateless verification fits both well.

## Tools and libraries
- **jwt.io** — decode/debug JWTs interactively.
- Node.js: `jsonwebtoken`. Python: `pyjwt`. Java: `jjwt`. Ruby: `ruby-jwt`.

## Quick reference
- Structure: `header.payload.signature`, all Base64Url, dot-separated.
- Signed ≠ encrypted — payload is always readable.
- Revocation = short expiry + rotated refresh tokens + denylist/version-claim for instant-kill cases.
- Always pin `algorithms` on verify; never trust the token's own `alg` claim.
- Storage: memory (best XSS resistance, worst UX) > httpOnly cookie (CSRF risk, mitigated by SameSite) > localStorage (XSS risk, no mitigation available at the storage layer).

## Further reading
- RFC 7519 (JWT), RFC 7516 (JWE), RFC 7518 (JWA — algorithms).
- [1-authentication.md](<1-authentication.md>) for how JWT fits among other auth mechanisms.
- [3-Oauth Guide.md](<3-Oauth Guide.md>) for JWT's role as an OAuth 2.0 access/ID token.
