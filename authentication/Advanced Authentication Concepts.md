# Advanced authentication concepts

Protocols and mechanisms beyond the basics covered in [1-authentication.md](<1-authentication.md>), [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>), and [3-Oauth Guide.md](<3-Oauth Guide.md>). For the theoretical framing behind these (why MFA works, threat models, compliance drivers), see [4-Authentication Concepts and Theory.md](<4-Authentication Concepts and Theory.md>).

## TL;DR
- **Passkeys/WebAuthn** are the modern replacement for passwords — public/private keypairs instead of shared secrets, phishing-resistant by construction because the credential is bound to the origin. This is the most important section in this file; read it even if you skim everything else.
- **SAML** is XML-based enterprise SSO, older than OAuth/OIDC but still dominant in large enterprises.
- **Certificate-based auth (mTLS)** authenticates via X.509 certificates instead of any shared secret — common for service-to-service and device auth.
- **Zero-knowledge proofs**, **risk-based auth**, **continuous auth**, and **auth orchestration** round out the advanced toolkit — all covered with working conceptual implementations below.

## 1. Passkeys and WebAuthn

### What a passkey actually is

A password is a shared secret: both you and the server know it, and if the server's database leaks, or you type it into a lookalike phishing site, the secret is compromised on both ends simultaneously. A **passkey** replaces that shared secret with a **public/private keypair**, generated on your device:

- The **private key** never leaves your device. It's generated inside — and typically never exportable from — a hardware-backed secure enclave (a phone's Secure Enclave/TPM, a laptop's TPM, or a dedicated hardware key like a YubiKey).
- The **public key** is sent to and stored by the server (the "relying party") at registration time. Public keys are, by design, safe to store in plaintext and safe to leak — an attacker with your public key still cannot sign anything, the same way knowing someone's house address doesn't let you forge their signature.

This is the single most important property to internalize: **there is no shared secret to steal.** A database breach at the server yields a pile of public keys, which are useless to an attacker for impersonation. Compare to passwords, where a database breach yields (hashed, but crackable) secrets that directly enable impersonation if cracked or reused elsewhere.

"Passkey" is the consumer-facing name for a WebAuthn/FIDO2 credential, typically synced across a user's devices via a platform ecosystem (iCloud Keychain, Google Password Manager, Windows Hello) so losing one device doesn't lose access — a meaningful usability improvement over the original FIDO2 model where a credential was pinned to one physical authenticator.

### The registration ceremony (conceptually)

1. User initiates "sign up" or "add a passkey" on a website (the **relying party**).
2. The relying party's server generates a random **challenge** and sends it to the browser, along with metadata (relying party ID — essentially the domain, user ID, acceptable algorithms).
3. The browser invokes the **WebAuthn API** (`navigator.credentials.create()`), which hands off to a **platform authenticator** (Face ID, Windows Hello, Touch ID) or a **roaming authenticator** (a physical security key).
4. The authenticator prompts the user to verify themselves locally — a fingerprint, face scan, or device PIN. This local verification never leaves the device and is never sent to the server; it's purely how the device confirms *you* are the one authorizing key generation, not how the server authenticates you.
5. On successful local verification, the authenticator generates a brand-new keypair **specific to this relying party's origin** (this domain-binding is the crux of the phishing resistance — more below), signs the server's challenge with the new private key, and returns the public key + signed challenge + a credential ID to the browser, which relays it to the server.
6. The server verifies the signature against the returned public key, confirms the challenge matches what it issued (preventing replay), and stores the public key + credential ID against the user's account. Registration is complete — no password was ever created or transmitted.

### The authentication ceremony (conceptually)

1. User initiates login; enters just a username (or, with "discoverable credentials"/resident keys, not even that — the authenticator itself surfaces which account to use).
2. Server generates a fresh, random **challenge**, sends it to the browser along with the relying party ID and (optionally) the list of credential IDs registered for this user.
3. Browser invokes `navigator.credentials.get()`, which routes to the authenticator holding the matching private key.
4. Authenticator prompts for local verification again (fingerprint/face/PIN) — proving the legitimate device owner is present.
5. Authenticator signs the challenge with the stored private key and returns the signature to the browser, which sends it to the server.
6. Server looks up the stored public key for that credential ID, verifies the signature against the challenge it issued, and — if valid — authenticates the user. No secret was ever typed, transmitted, or exposed to be intercepted.

```
Registration:                                Authentication:
Server --challenge--------> Browser          Server --challenge--------> Browser
                              |                                            |
                    (biometric prompt)                          (biometric prompt)
                              |                                            |
                    generate keypair,                            sign challenge with
                    sign challenge                                stored private key
                              |                                            |
Server <---public key + signature-- Browser  Server <---signature-------- Browser
  |                                             |
  verify + store public key                     verify against stored public key
```

### Why WebAuthn resists phishing where passwords and OTP don't

This is the property that makes WebAuthn categorically different from every "second factor" that came before it, worth spelling out precisely:

- **Passwords** are phishable because the secret is portable — you can type the same password into a lookalike site (`paypa1.com` instead of `paypal.com`) and it works identically from the user's perspective. The password itself carries no information about which site it's "supposed" to work on.
- **OTP/TOTP codes** are phishable for the same underlying reason — a 6-digit code is just another string a user can be tricked into typing into an attacker-controlled site, which the attacker then immediately relays to the real site in real time (a real-time relay/adversary-in-the-middle phishing kit — this is a well-documented, automated attack pattern, not theoretical).
- **WebAuthn credentials are origin-bound at the cryptographic level, not just by user vigilance.** During registration, the authenticator records which exact origin (domain) the keypair was created for. During authentication, the browser itself — not the user, not JavaScript the page controls — includes the actual origin the request is coming from as part of what gets signed. If a phishing site at `paypa1.com` tries to trigger a WebAuthn authentication for `paypal.com`'s credential, the browser will not even offer the credential — the mismatch is caught by the browser/OS before the user ever gets a chance to (mistakenly) approve anything. There is no "look carefully at the URL" step for the user to get wrong, because the check isn't the user's job.

This is why WebAuthn is described as "phishing-resistant" rather than merely "phishing-resistant if used correctly" — the resistance is structural, enforced by the browser and OS, not dependent on user attentiveness the way "check the URL before typing your password" always was and always failed to fully deliver on.

### Passkeys vs. TOTP-based MFA

Worth comparing directly since both get called "strong authentication," but they solve different problems and have different failure modes:

| | TOTP (authenticator app codes) | Passkeys (WebAuthn) |
|---|---|---|
| What's shared with the server | A symmetric secret (the TOTP seed), used to independently compute the same code both sides | Only a public key — nothing secret ever touches the server |
| Phishing resistance | None — a code can be relayed by a real-time phishing proxy just like a password | Structural — origin-bound, browser-enforced, can't be relayed |
| What if the server is breached | The TOTP seed itself can potentially be extracted and cloned if stored insecurely | Public keys leak with zero impersonation value |
| Typically used as | A *second* factor alongside a password | Can replace the password entirely (passwordless) or serve as a strong second factor |
| Cross-device story | Manual re-enrollment per device, or shared seed (weaker) | Increasingly synced via platform cloud keychains |
| User experience | Type a 6-digit code every login | Biometric/PIN prompt, no typing |

The practical implication: **TOTP is a meaningful improvement over password-only, but it does not close the phishing gap** the way passkeys do — an attacker who successfully phishes both the password and a real-time-relayed TOTP code still gets in. Passkeys close that gap structurally. Where a system can't yet support passkeys, TOTP (app-based, not SMS) remains a reasonable interim second factor — just go in aware it's a friction improvement over passwords alone, not a phishing fix.

### FIDO2, WebAuthn, and CTAP — how the pieces fit
- **FIDO2** is the umbrella standard/alliance effort.
- **WebAuthn** is the W3C browser API (`navigator.credentials`) that web pages use to trigger registration/authentication ceremonies — this is the piece described above.
- **CTAP (Client to Authenticator Protocol)** is the piece that lets a browser talk to an external/roaming authenticator (a USB/NFC/Bluetooth security key) rather than only the platform's built-in authenticator.

Together these let a single standard cover "log in with Face ID on this laptop" and "log in by tapping a YubiKey" with the same server-side verification logic.

### Attestation
Optionally, during registration the authenticator can provide **attestation** — a signed statement from the authenticator's manufacturer proving what kind of device generated the key (a genuine YubiKey model X, versus an unknown/emulated authenticator). High-security deployments (government, some enterprise) may require attestation from an approved authenticator list; most consumer applications skip it, since it adds friction and privacy trade-offs (attestation can leak device model information) without much marginal benefit for typical threat models.

### Conceptual server-side flow (Node.js, illustrative)

Real implementations should use a maintained library (`@simplewebauthn/server` for Node, or equivalent) rather than hand-rolling the CBOR/COSE key parsing — the snippet below shows the shape of the interaction, not production-ready crypto handling.

```javascript
const express = require('express');
const crypto = require('crypto');
const app = express();
app.use(express.json());

// Registration: step 1 — issue a challenge
app.post('/webauthn/register/options', (req, res) => {
  const challenge = crypto.randomBytes(32);
  req.session.currentChallenge = challenge.toString('base64url');

  res.json({
    challenge: challenge.toString('base64url'),
    rp: { name: 'Example Site', id: 'example.com' },
    user: {
      id: Buffer.from(req.session.userId).toString('base64url'),
      name: req.session.userEmail,
      displayName: req.session.userName
    },
    pubKeyCredParams: [{ type: 'public-key', alg: -7 }], // ES256
    authenticatorSelection: { userVerification: 'required' },
    timeout: 60000,
    attestation: 'none'
  });
});

// Registration: step 2 — verify the returned credential (use a library in production)
app.post('/webauthn/register/verify', async (req, res) => {
  const { credential } = req.body;
  // A real implementation verifies: challenge match, origin match, signature validity,
  // then stores credential.id + the returned public key against the user account.
  await db.credentials.create({
    userId: req.session.userId,
    credentialId: credential.id,
    publicKey: credential.response.publicKey,
    counter: 0
  });
  res.json({ verified: true });
});

// Authentication: step 1 — issue a challenge
app.post('/webauthn/authenticate/options', async (req, res) => {
  const challenge = crypto.randomBytes(32);
  req.session.currentChallenge = challenge.toString('base64url');
  const userCredentials = await db.credentials.findByUser(req.body.userId);

  res.json({
    challenge: challenge.toString('base64url'),
    rpId: 'example.com',
    allowCredentials: userCredentials.map(c => ({ type: 'public-key', id: c.credentialId })),
    userVerification: 'required',
    timeout: 60000
  });
});

// Authentication: step 2 — verify the signed assertion (use a library in production)
app.post('/webauthn/authenticate/verify', async (req, res) => {
  const { credential } = req.body;
  const stored = await db.credentials.findByCredentialId(credential.id);
  // A real implementation verifies the signature against stored.publicKey and checks
  // the signature counter increased (a stalled/repeated counter suggests key cloning).
  res.json({ verified: true, userId: stored.userId });
});

app.listen(3000);
```

## 2. SAML (Security Assertion Markup Language)

XML-based standard for exchanging authentication/authorization data, predating OAuth and still dominant in large enterprise SSO deployments (many enterprise IdPs — Okta, ADFS, PingFederate — support both SAML and OIDC side by side for this reason).

**Components**: **Assertions** (the actual signed XML statement about the user), **Protocol** (how requests/responses are exchanged), **Bindings** (transport — HTTP POST or Redirect), **Profiles** (specific use cases like Web Browser SSO or Single Logout).

**Flow**:
1. User attempts to access a protected resource at the Service Provider (SP).
2. SP redirects the user to the Identity Provider (IdP) with a SAML authentication request.
3. User authenticates with the IdP (if not already authenticated there).
4. IdP sends a signed SAML assertion back to the SP (via the browser, typically an auto-submitting HTML form using the POST binding).
5. SP validates the assertion's signature against the IdP's known public certificate and grants access.

```javascript
const express = require('express');
const passport = require('passport');
const SamlStrategy = require('passport-saml').Strategy;

const app = express();
app.use(passport.initialize());

passport.use(new SamlStrategy({
  entryPoint: 'https://idp.example.com/saml/sso',
  issuer: 'https://sp.example.com',
  callbackUrl: 'https://sp.example.com/saml/callback',
  cert: 'idp-public-cert.pem',
  validateInResponseTo: false
}, (profile, done) => {
  return done(null, { id: profile.nameID, email: profile.email, displayName: profile.displayName });
}));

app.get('/login', passport.authenticate('saml', { successRedirect: '/dashboard', failureRedirect: '/login' }));
app.post('/saml/callback', passport.authenticate('saml', { successRedirect: '/dashboard', failureRedirect: '/login' }));
```

**SAML vs. OAuth 2.0/OIDC**: SAML is heavier (XML signing, metadata exchange) but more mature for enterprise SSO with rich attribute exchange; OAuth/OIDC is lighter (JSON over HTTPS) and better suited to APIs, mobile, and modern web apps. New enterprise integrations increasingly default to OIDC, but SAML remains entrenched wherever it's already deployed — migrating an established enterprise SSO integration off SAML is rarely worth the effort on its own.

## 3. OpenID Connect (OIDC) — implementation detail

OIDC's conceptual role (identity layer on OAuth 2.0) and flow are covered in [3-Oauth Guide.md](<3-Oauth Guide.md>). Implementation shape:

```javascript
const express = require('express');
const axios = require('axios');
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const app = express();

const OIDC_CONFIG = {
  issuer: 'https://accounts.google.com',
  clientId: 'your-client-id',
  clientSecret: 'your-client-secret',
  redirectUri: 'http://localhost:3000/callback'
};

async function getOIDCConfig() {
  const response = await axios.get(`${OIDC_CONFIG.issuer}/.well-known/openid_configuration`);
  return response.data; // discovery — dynamically learn all provider endpoints
}

app.get('/login', async (req, res) => {
  const config = await getOIDCConfig();
  const state = crypto.randomBytes(16).toString('hex');
  const authUrl = `${config.authorization_endpoint}?response_type=code&client_id=${OIDC_CONFIG.clientId}` +
    `&redirect_uri=${OIDC_CONFIG.redirectUri}&scope=openid email profile&state=${state}`;
  res.redirect(authUrl);
});

app.get('/callback', async (req, res) => {
  const { code } = req.query;
  const config = await getOIDCConfig();
  const tokenResponse = await axios.post(config.token_endpoint, {
    grant_type: 'authorization_code', code,
    redirect_uri: OIDC_CONFIG.redirectUri,
    client_id: OIDC_CONFIG.clientId, client_secret: OIDC_CONFIG.clientSecret
  }, { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } });

  const { access_token, id_token } = tokenResponse.data;
  const decoded = jwt.decode(id_token); // production: verify signature against config.jwks_uri
  res.json({ user: decoded, access_token });
});
```

## 4. Certificate-based authentication (mTLS)

Uses X.509 digital certificates to verify identity — no password or token at all, just proof of possessing a private key whose corresponding certificate was signed by a trusted Certificate Authority. Common for service-to-service auth (mutual TLS, where *both* client and server present certificates) and long-lived device identity (IoT).

```javascript
const https = require('https');
const fs = require('fs');
const express = require('express');
const app = express();

const options = {
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem'),
  ca: fs.readFileSync('ca-cert.pem'),
  requestCert: true,
  rejectUnauthorized: true // reject any connection without a valid client cert
};

app.use((req, res, next) => {
  const cert = req.socket.getPeerCertificate();
  if (cert.subject) {
    req.user = { commonName: cert.subject.CN, organization: cert.subject.O };
  }
  next();
});

app.get('/protected', (req, res) => {
  if (req.user) res.json({ message: 'Access granted', user: req.user });
  else res.status(401).send('Certificate required');
});

https.createServer(options, app).listen(3000);
```

**Trade-offs**: strong identity guarantees and no credential to phish (the private key never leaves the holder), but certificate lifecycle management (issuance, rotation, revocation via CRL/OCSP) is real operational overhead most teams underestimate when adopting mTLS.

## 5. Zero-knowledge proofs — implementation shape

Conceptual background (completeness/soundness/zero-knowledge properties, zk-SNARKs/STARKs/Bulletproofs) is in [4-Authentication Concepts and Theory.md](<4-Authentication Concepts and Theory.md>). The shape of a ZK authentication scheme:

```javascript
// Conceptual only — real ZK schemes require a proper cryptographic library
// (e.g. circom/snarkjs for zk-SNARKs), never hand-rolled primitives.
class ZeroKnowledgeAuth {
  constructor() {
    this.secret = crypto.randomBytes(32);
    this.publicKey = this.derivePublicKey(this.secret);
  }

  generateProof(challenge) {
    // Prove knowledge of `secret` corresponding to `publicKey`, for this specific
    // challenge, without transmitting `secret` itself.
    return this.createProof(this.secret, challenge);
  }

  verifyProof(proof, publicKey, challenge) {
    // Verifier checks the proof against publicKey + challenge, learning only
    // "yes, they know the secret" — nothing about the secret's value.
    return this.verifyProofLogic(proof, publicKey, challenge);
  }
}
```

## 6. Risk-based authentication — implementation

Conceptual background in [4-Authentication Concepts and Theory.md](<4-Authentication Concepts and Theory.md>). A minimal scoring implementation:

```javascript
const geoip = require('geoip-lite');

class RiskBasedAuth {
  constructor() {
    this.riskThresholds = { low: 0.3, medium: 0.7 };
  }

  calculateRisk(req) {
    let riskScore = 0;
    const geo = geoip.lookup(req.ip);
    if (geo && geo.country !== req.session.usualCountry) riskScore += 0.3;

    const userAgent = req.headers['user-agent'];
    if (!req.session.knownDevices?.includes(userAgent)) riskScore += 0.2;

    const hour = new Date().getHours();
    if (hour < 6 || hour > 22) riskScore += 0.2;

    return Math.min(riskScore, 1.0);
  }

  getAuthRequirements(riskScore) {
    if (riskScore < this.riskThresholds.low) return { mfa: false, captcha: false };
    if (riskScore < this.riskThresholds.medium) return { mfa: true, captcha: false };
    return { mfa: true, captcha: true };
  }
}
```

## 7. Continuous authentication — implementation

```javascript
class ContinuousAuth {
  constructor() {
    this.behaviorProfile = {};
    this.currentSession = {};
    this.threshold = 0.8;
  }

  startMonitoring() {
    this.monitorKeystrokes();
    this.monitorMouseMovements();
  }

  monitorKeystrokes() {
    let lastKeyTime = Date.now();
    let intervals = [];
    document.addEventListener('keydown', () => {
      const now = Date.now();
      intervals.push(now - lastKeyTime);
      if (intervals.length > 10) intervals.shift();
      this.updateBehaviorProfile('keystrokes', intervals);
      lastKeyTime = now;
    });
  }

  monitorMouseMovements() {
    let movements = [];
    document.addEventListener('mousemove', (event) => {
      movements.push({ x: event.clientX, y: event.clientY, timestamp: Date.now() });
      if (movements.length > 50) movements.shift();
      this.updateBehaviorProfile('mouse', movements);
    });
  }

  updateBehaviorProfile(type, data) {
    this.currentSession[type] = data;
    if (this.calculateSimilarity(this.behaviorProfile, this.currentSession) < this.threshold) {
      this.triggerReauthentication();
    }
  }

  calculateSimilarity(profile, session) {
    return 0.9; // placeholder — real implementations use a trained similarity model
  }

  triggerReauthentication() {
    // Step-up challenge, not an immediate hard logout — see theory file for why
  }
}
```

## 8. Passwordless via magic links — implementation

The weakest passwordless method security-wise (see the comparison table in the theory file), but simple and still common for low-stakes applications.

```javascript
const crypto = require('crypto');
const magicLinks = new Map();

app.post('/auth/magic-link', async (req, res) => {
  const { email } = req.body;
  const token = crypto.randomBytes(32).toString('hex');
  const expiresAt = Date.now() + (10 * 60 * 1000); // 10 minutes
  magicLinks.set(token, { email, expiresAt });

  await sendEmail(email, `Login link: http://localhost:3000/auth/verify?token=${token} (expires in 10 minutes)`);
  res.json({ message: 'Magic link sent' });
});

app.get('/auth/verify', (req, res) => {
  const { token } = req.query;
  const linkData = magicLinks.get(token);
  if (!linkData || Date.now() > linkData.expiresAt) {
    magicLinks.delete(token);
    return res.status(400).send('Invalid or expired token');
  }
  magicLinks.delete(token); // single-use
  const accessToken = jwt.sign({ email: linkData.email }, process.env.JWT_SECRET, { expiresIn: '1h' });
  res.json({ accessToken });
});
```

## 9. Social login integration

Authenticating via an existing provider (Google, GitHub, Microsoft) is functionally an OAuth 2.0 + OIDC client integration — see [3-Oauth Guide.md](<3-Oauth Guide.md>) for the underlying flow. The value: reduced signup friction, faster onboarding, and delegated trust in the provider's own authentication strength (which is often stronger than what a small app would build itself, especially if the provider offers passkeys).

```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  callbackURL: 'http://localhost:3000/auth/google/callback'
}, (accessToken, refreshToken, profile, done) => {
  const user = { id: profile.id, email: profile.emails[0].value, name: profile.displayName };
  return done(null, user); // find-or-create against your own user store here
}));

app.get('/auth/google', passport.authenticate('google', { scope: ['profile', 'email'] }));
app.get('/auth/google/callback', passport.authenticate('google', {
  successRedirect: '/dashboard', failureRedirect: '/login'
}));
```

**Caveat worth designing for**: a user's social account getting compromised (or the provider itself having an incident) becomes your app's problem too — social login inherits the provider's security posture, for better and worse.

## 10. Authentication orchestration

For systems supporting multiple authentication methods (password, MFA, social, passkeys) selected dynamically by risk or policy, an orchestration layer coordinates which method(s) are required for a given login attempt rather than hardcoding one path.

```javascript
class AuthOrchestrator {
  constructor() {
    this.authMethods = {
      password: this.passwordAuth.bind(this),
      mfa: this.mfaAuth.bind(this),
      passkey: this.passkeyAuth.bind(this),
      social: this.socialAuth.bind(this)
    };
    this.policies = {
      lowRisk: ['password'],
      mediumRisk: ['password', 'mfa'],
      highRisk: ['passkey'] // prefer phishing-resistant auth for high-risk logins
    };
  }

  async authenticate(req, res) {
    const { method, credentials, riskLevel } = req.body;
    const requiredMethods = this.policies[riskLevel] || this.policies.mediumRisk;

    if (!requiredMethods.includes(method)) {
      return res.status(400).json({ error: 'Invalid method for this risk level' });
    }

    const result = await this.authMethods[method](credentials);
    if (!result.success) return res.status(401).json({ error: 'Authentication failed' });

    const nextMethod = this.getNextMethod(method, requiredMethods);
    if (nextMethod) {
      return res.json({ status: 'partial', nextMethod, sessionId: result.sessionId });
    }
    return res.json({ status: 'complete', accessToken: result.accessToken });
  }

  getNextMethod(current, required) {
    return required[required.indexOf(current) + 1] || null;
  }

  async passwordAuth(credentials) { /* ... */ return { success: true, sessionId: 'session-123' }; }
  async mfaAuth(credentials) { /* ... */ return { success: true, accessToken: 'token-123' }; }
  async passkeyAuth(credentials) { /* ... */ return { success: true, accessToken: 'token-456' }; }
  async socialAuth(credentials) { /* ... */ return { success: true, accessToken: 'token-789' }; }
}
```

## Choosing among these

- **Default for new consumer/B2B web apps in 2026**: passkeys as the primary method, with TOTP or email/magic-link as a fallback for users on unsupported devices/browsers — password-only is increasingly the exception, not the norm, for new products.
- **Enterprise SSO into an existing large organization**: SAML if that's what their IdP already speaks, OIDC if greenfield or the org has already modernized.
- **Service-to-service**: mTLS or OAuth 2.0 client credentials, not user-facing methods at all.
- **High-risk/regulated actions within an otherwise normal session**: step-up authentication — require a fresh passkey/MFA challenge specifically before the sensitive action (a large transfer, a password change), even if the session itself is already authenticated.
- **Risk-based/continuous auth**: layer onto any of the above as an additional signal, not a replacement for a real primary authentication method.

## Further reading
- [4-Authentication Concepts and Theory.md](<4-Authentication Concepts and Theory.md>) — factor theory, threat models, ZKP/RBA/continuous-auth conceptual background.
- [1-authentication.md](<1-authentication.md>), [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>), [3-Oauth Guide.md](<3-Oauth Guide.md>) — foundational mechanisms these build on.
- W3C WebAuthn Level 2/3 spec; FIDO Alliance FIDO2 documentation; `webauthn.io` for a live interactive demo of the registration/authentication ceremonies.
