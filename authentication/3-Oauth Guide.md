# OAuth 2.0

OAuth 2.0 (RFC 6749) is an **authorization** framework — it lets a third-party application (client) get scoped access to a user's resources without ever seeing the user's password. It is not an authentication protocol by itself; [OpenID Connect](<Advanced Authentication Concepts.md>) layers identity on top of it. For where OAuth fits among other auth mechanisms, see [1-authentication.md](<1-authentication.md>); for JWT as the usual token format, see [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>).

## TL;DR
- Four roles: **resource owner** (user), **client** (app), **resource server** (API), **authorization server** (issues tokens).
- The client never touches the user's credentials — it gets a scoped, revocable, short-lived **access token** instead, obtained via one of several **grant types (flows)**.
- **Authorization Code + PKCE** is the flow to default to today, for both public and confidential clients — implicit and password grants are deprecated.
- **PKCE** is what makes the authorization code flow safe for clients that can't keep a secret (SPAs, mobile, and now recommended even for confidential clients as defense-in-depth).
- OAuth solves the "password anti-pattern": before OAuth, delegating access meant literally handing your password to a third party.

## Why OAuth 2.0 exists

Before OAuth, if you wanted a photo-printing app to access your Google Photos, you gave it your Google password directly — the "password anti-pattern." That's bad because the app can store/misuse the password, you have no way to grant *partial* access (just "print my vacation album," not "read all my email too"), and revoking access means changing your password everywhere it's shared.

OAuth 2.0 replaces that with: the user authenticates directly with the service that owns their data (Google), explicitly consents to specific **scopes** ("read photos," not "everything"), and the client receives an **access token** — a credential that's scoped, time-limited, and revocable independently of the user's actual password.

## Roles and components

- **Resource owner** — the user who owns the data.
- **Client** — the application requesting access (e.g., the photo-printing app).
- **Resource server** — hosts the protected data (e.g., Google's Photos API).
- **Authorization server** — authenticates the resource owner and issues tokens (e.g., Google's OAuth server). Often the same entity as the resource server, but not always.

**Tokens**:
- **Access token** — short-lived (minutes to an hour), used on every API call, typically `Authorization: Bearer <token>`.
- **Refresh token** — long-lived, used to get new access tokens without bothering the user again.

**Endpoints**: the **authorization endpoint** (user-facing, handles login + consent) and the **token endpoint** (machine-to-machine, exchanges a grant for tokens).

**Client types**:
- **Confidential clients** — can securely hold a `client_secret` (server-side web apps).
- **Public clients** — cannot (SPAs, mobile apps, CLI tools — the secret would ship in the client binary/bundle, which isn't a secret at all). Public clients rely on PKCE instead of a client secret for security.

## 🟢 Authorization Code Flow (with PKCE)

The default flow for essentially everything today — server-rendered apps, SPAs, and mobile apps alike, the last two via mandatory PKCE.

### Why PKCE exists

Without PKCE, the authorization code flow has a gap: the authorization code returned in step 3 below travels through the browser's redirect (a URL) and is exchanged for tokens in step 4 using the `client_id` + `client_secret`. For a confidential client that's fine — a `client_secret` proves the token-exchange request really comes from the legitimate client. But a public client (SPA, mobile app) *has no secret to present* — anything shipped in the client is visible to anyone who inspects the app. So if an attacker can intercept the authorization code (e.g., a malicious app registered to the same custom URL scheme on a mobile device intercepts the redirect), nothing stops them from exchanging that stolen code for tokens themselves.

**PKCE (Proof Key for Code Exchange, RFC 7636, pronounced "pixy")** closes this gap by binding the authorization request to the token exchange with a client-generated secret that never touches the authorization server until the very last step — so an attacker who only intercepts the *code* still can't complete the exchange.

### PKCE mechanics, step by step

1. **Client generates a `code_verifier`** — a high-entropy random string (43–128 characters), kept only in the client, never sent anywhere yet.
   ```javascript
   const codeVerifier = crypto.randomBytes(32).toString('base64url');
   ```
2. **Client derives a `code_challenge`** from it, normally via SHA-256 (`code_challenge_method=S256`):
   ```javascript
   const codeChallenge = crypto.createHash('sha256').update(codeVerifier).digest('base64url');
   ```
3. **Authorization request** includes the `code_challenge` (not the verifier):
   ```
   GET https://authorization-server.com/oauth/authorize
     ?response_type=code
     &client_id=CLIENT_ID
     &redirect_uri=https://client.com/callback
     &scope=read
     &state=xyz123
     &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
     &code_challenge_method=S256
   ```
4. **User authenticates and consents**; the authorization server stores the `code_challenge` alongside the issued authorization code, and redirects back with the code:
   ```
   https://client.com/callback?code=AUTH_CODE&state=xyz123
   ```
5. **Token request** — the client now sends the original `code_verifier` (the raw secret, for the first time) along with the code:
   ```
   POST https://authorization-server.com/oauth/token
   Content-Type: application/x-www-form-urlencoded

   grant_type=authorization_code
   &code=AUTH_CODE
   &redirect_uri=https://client.com/callback
   &client_id=CLIENT_ID
   &code_verifier=<the original random string from step 1>
   ```
6. **Server verifies**: it hashes the received `code_verifier` with the same method and checks it matches the `code_challenge` it stored in step 4. Only then does it issue tokens.

**Why this works**: an attacker who intercepts the authorization code in step 4 (e.g., by capturing the redirect) does *not* have the `code_verifier` — that value never left the legitimate client until step 5, and it can't be derived from the `code_challenge` (that's the point of hashing it — SHA-256 is one-way). So a stolen code alone is now useless.

### Full request/response sequence

```
Client                    Browser                 Authorization Server        Token Endpoint
  |--generate verifier+------->|                          |                        |
  |  challenge (local)         |                          |                        |
  |--redirect w/ code_challenge--------------------------->|                        |
  |                            |<---login + consent UI-----|                        |
  |                            |----user approves---------->|                        |
  |<----redirect w/ code-------|                            |                        |
  |--POST code + code_verifier--------------------------------------------------->|
  |<----------------------------------------access_token + refresh_token----------|
```

### Advantages / disadvantages
Server-side apps get the original flow's benefit (secret never exposed to the browser) plus PKCE's protection against code interception; public clients get equivalent protection without needing a secret at all. Downsides are purely implementation complexity — one more crypto step and one more thing to get right (using `S256`, not the weaker `plain` method where `code_challenge == code_verifier`, which offers no protection).

**Modern guidance**: use PKCE even for confidential clients — it's defense-in-depth against authorization code interception regardless of client type, and OAuth 2.1 (the informal consolidation of OAuth 2.0 best practices) makes it mandatory for all clients, not just public ones.

## 🟢 Client Credentials Flow

For machine-to-machine access with no user involved — the client is acting on its own behalf, not a user's.

```
POST https://authorization-server.com/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=CLIENT_ID&client_secret=CLIENT_SECRET&scope=read
```

```json
{"access_token": "2YotnFZFEjr1zCsicMWpAA", "token_type": "Bearer", "expires_in": 3600, "scope": "read"}
```

**Use for**: service-to-service calls, background jobs hitting an API, accessing an app's own non-user-specific resources. Always a confidential client (a `client_secret` is required, so this flow doesn't apply to public clients).

## 🟢 Device Code Flow

For input-constrained devices — smart TVs, CLI tools, game consoles — that can't easily host a browser-based redirect flow.

1. **Device requests a code**:
   ```
   POST https://authorization-server.com/device
   client_id=CLIENT_ID
   ```
2. **Server responds** with a `device_code` (for the device), a short human-friendly `user_code`, and a `verification_uri`:
   ```json
   {"device_code": "IO2RUI3SAH0IQuESHAEBAeYOO8UPAI", "user_code": "RSIK-KRAM", "verification_uri": "https://authorization-server.com/device", "interval": 5, "expires_in": 1800}
   ```
3. **Device displays** the `user_code` and `verification_uri` (often as a QR code); the user opens that URL on a *different* device (phone/laptop) and enters the code.
4. **Device polls** the token endpoint at the given `interval` until the user approves (`authorization_pending` → success), denies (`access_denied`), or the codes expire (`expired_token`).

**Use for**: smart TVs, CLIs, IoT devices — anything with no convenient browser/keyboard for direct login.

## 🔴 Deprecated flows — and why

### Implicit Flow
Returned the access token directly in the redirect URL fragment, skipping the code-exchange step, because early browsers made a POST-based token exchange awkward for pure JS apps. **Why deprecated**: the token ends up in the browser's URL/history/referrer and server logs, has no client authentication step at all, and doesn't support refresh tokens (forcing frequent re-auth via invisible iframe tricks that browsers' third-party cookie restrictions have since broken anyway). **Replacement**: Authorization Code + PKCE — it achieves everything the implicit flow was trying to enable for public clients, without exposing the token in a URL.

### Resource Owner Password Credentials Flow
Let the client collect the user's username/password directly and trade them for a token in one request. **Why deprecated**: it's the exact password anti-pattern OAuth was invented to eliminate — the client sees the raw password, defeating the entire point of delegated authorization. Still occasionally used for first-party migration scenarios (a company's own legacy app moving to OAuth) but never for third-party clients.

## Which flow for which scenario

| Scenario | Flow | Why |
|---|---|---|
| Server-rendered web app (Node/Django/Rails), secret stays server-side | Authorization Code (+ PKCE recommended) | Confidential client; PKCE adds defense-in-depth against code interception |
| SPA / JS-only frontend | Authorization Code + PKCE | Public client, cannot hold a secret; PKCE is the only safe option — never implicit flow |
| Native mobile app | Authorization Code + PKCE | Same as SPA — app binaries can be decompiled, so no secret is truly secret |
| Backend service calling an API with no user context | Client Credentials | No resource owner involved; client authenticates as itself |
| Smart TV, CLI tool, IoT device with no/limited browser | Device Code | Can't host a redirect-based flow; user completes auth on a secondary device |
| Legacy first-party app migrating off direct password storage | Resource Owner Password Credentials (temporary only) | Acceptable only as a stopgap for a trusted first-party client, never third-party |
| Anything client-side returning tokens via URL fragment | ❌ Implicit (deprecated) | Use Authorization Code + PKCE instead |

## Tokens

**Access tokens**: short-lived (15 min–1 hr), sent as `Authorization: Bearer <token>`. Often a JWT (see [the JWT file](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>)) so resource servers can verify locally, but the spec only requires it be a string — opaque tokens validated via introspection are equally valid.

**Refresh tokens**: long-lived, stored server-side, exchanged for new access tokens without re-prompting the user:
```
POST https://authorization-server.com/oauth/token
grant_type=refresh_token&refresh_token=REFRESH_TOKEN&client_id=CLIENT_ID&client_secret=CLIENT_SECRET
```
Rotate on every use (see the JWT file's revocation section for why rotation matters) — issuing a new refresh token and invalidating the old one on each refresh call limits how long a stolen refresh token stays useful.

## Security considerations

| Risk | Mitigation |
|---|---|
| Token theft (interception or insecure storage) | HTTPS everywhere, short-lived access tokens, secure storage (see JWT storage trade-offs) |
| Authorization code interception | PKCE (see above) |
| CSRF against the authorization request | The `state` parameter — a random value the client generates, includes in the auth request, and verifies matches on the callback |
| Redirect URI manipulation | Exact-match redirect URI allowlisting at client registration — never wildcard match |
| Client secret leakage | Only relevant to confidential clients; store secrets in a secrets manager, never in client-side code or public repos |
| Overly broad access | Request the minimum scopes needed; resource servers should reject requests exceeding a token's granted scope |

The `state` parameter deserves a concrete example — it's easy to under-implement:
```javascript
// Before redirecting to the authorization server
const state = crypto.randomBytes(32).toString('hex');
req.session.oauthState = state; // store server-side, tied to the user's session

// On the callback
if (req.query.state !== req.session.oauthState) {
  throw new Error('State mismatch — possible CSRF');
}
```

## OpenID Connect (OIDC)

OAuth 2.0 is authorization-only; **OpenID Connect** extends it with an identity layer:
- **`id_token`** — a JWT asserting who the user is (`sub`, `email`, `name`, etc.), returned alongside the access token.
- **UserInfo endpoint** — a REST endpoint clients can call with the access token to fetch additional profile claims.
- **Standard scopes**: `openid` (required to trigger OIDC behavior at all), plus `profile`, `email`, `address`, `phone`.
- **Discovery**: providers expose `/.well-known/openid-configuration` so clients can dynamically learn all the provider's endpoints and capabilities instead of hardcoding them.

```
GET https://authorization-server.com/oauth/authorize?response_type=code&client_id=CLIENT_ID&redirect_uri=https://client.com/callback&scope=openid email&state=xyz123

# Token response now includes an id_token:
{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjFlOWdkazcifQ..."
}
```

The client must validate the `id_token`'s signature, `iss`, `aud`, and `exp` before trusting its claims — exactly like any other JWT.

## OAuth 2.0 vs. SAML

| | SAML | OAuth 2.0 (+ OIDC) |
|---|---|---|
| Format | XML assertions | JSON tokens |
| Primary purpose | SSO / authentication | Delegated authorization (auth via OIDC extension) |
| Fit | Enterprise SSO, legacy systems | APIs, mobile, modern web |
| Complexity | Heavier — XML signing, metadata exchange | Lighter — JSON over HTTPS |

## Practical implementation

```javascript
const express = require('express');
const crypto = require('crypto');
const axios = require('axios');
const app = express();

const CLIENT_ID = 'your-client-id';
const REDIRECT_URI = 'http://localhost:3000/callback';
const AUTH_SERVER = 'https://authorization-server.com';

app.get('/login', (req, res) => {
  const state = crypto.randomBytes(16).toString('hex');
  const codeVerifier = crypto.randomBytes(32).toString('base64url');
  const codeChallenge = crypto.createHash('sha256').update(codeVerifier).digest('base64url');

  req.session.oauthState = state;
  req.session.codeVerifier = codeVerifier;

  const url = `${AUTH_SERVER}/oauth/authorize?response_type=code&client_id=${CLIENT_ID}` +
    `&redirect_uri=${REDIRECT_URI}&scope=openid%20email&state=${state}` +
    `&code_challenge=${codeChallenge}&code_challenge_method=S256`;
  res.redirect(url);
});

app.get('/callback', async (req, res) => {
  const { code, state } = req.query;
  if (state !== req.session.oauthState) return res.status(400).send('Invalid state');

  try {
    const response = await axios.post(`${AUTH_SERVER}/oauth/token`, new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: REDIRECT_URI,
      client_id: CLIENT_ID,
      code_verifier: req.session.codeVerifier
    }), { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } });

    res.json(response.data);
  } catch (error) {
    res.status(500).send('Token exchange failed');
  }
});

app.listen(3000);
```

## Use cases
1. API authorization with scoped access tokens.
2. SSO across applications via OIDC.
3. Device/IoT authentication via device code flow.
4. Federated identity — sign in with Google, Okta, Microsoft, etc.

## Tools and libraries
- Node.js: `openid-client`, `passport-oauth2`. Python: `authlib`, `oauthlib`. Java: `spring-security-oauth2`. Providers: Google, Okta, Auth0, Keycloak.

## Quick reference
- Roles: resource owner, client, resource server, authorization server.
- Default flow: Authorization Code + PKCE, for public *and* confidential clients.
- PKCE: `code_verifier` (secret, client-only) → `code_challenge` (hash, sent upfront) → verifier revealed only at token exchange.
- Never use implicit or password grants for new development.
- `state` prevents CSRF on the authorization request; PKCE prevents code interception.
- OIDC = OAuth 2.0 + `id_token` (JWT) + UserInfo endpoint = actual authentication.

## Further reading
- RFC 6749 (OAuth 2.0), RFC 7636 (PKCE), OpenID Connect Core spec.
- [1-authentication.md](<1-authentication.md>) for OAuth's place among other mechanisms.
- [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>) for the token format OAuth usually carries.
- [Advanced Authentication Concepts.md](<Advanced Authentication Concepts.md>) for SAML and OIDC protocol-level detail.
