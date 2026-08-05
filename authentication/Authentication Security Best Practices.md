# Authentication security best practices

Concrete, implementation-level defenses for the vulnerabilities and threat models discussed conceptually in [4-Authentication Concepts and Theory.md](<4-Authentication Concepts and Theory.md>). This file is code-first — each section is a working (or near-working) reference implementation of a specific control, in Node.js/Express, covering the OWASP Authentication Cheat Sheet's practical guidance.

## TL;DR
- Hash passwords with **bcrypt/scrypt/Argon2** (adaptive, salted) — never a fast general-purpose hash like raw SHA-256.
- Rate-limit and lock out brute-force attempts; regenerate session IDs on login to prevent session fixation.
- JWT security is about explicit configuration: pin algorithms, set short expiry, blacklist by `jti` for the tokens that need instant revocation (full reasoning in the [JWT file](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>)).
- OAuth security centers on `state` (CSRF) and PKCE (code interception) — full reasoning in the [OAuth file](<3-Oauth Guide.md>); this file just shows the implementation.
- CSRF, input validation, security headers, audit logging, and rate limiting are independent layers — each closes a different gap, and skipping any one of them leaves a real hole even if the others are solid.

## 1. Password security

### Password policy validation
Enforce length, character variety, and block common/breached passwords — length matters more than complexity rules in practice (NIST 800-63B specifically recommends favoring length over forced character-class complexity), but a baseline policy still catches the worst cases.

```javascript
const passwordValidator = {
  minLength: 12,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: true
};

function validatePassword(password) {
  const errors = [];
  if (password.length < passwordValidator.minLength) {
    errors.push(`Password must be at least ${passwordValidator.minLength} characters`);
  }
  if (!/[A-Z]/.test(password)) errors.push('Password must contain at least one uppercase letter');
  if (!/[a-z]/.test(password)) errors.push('Password must contain at least one lowercase letter');
  if (!/\d/.test(password)) errors.push('Password must contain at least one number');
  if (!/[!@#$%^&*(),.?":{}|<>]/.test(password)) errors.push('Password must contain at least one special character');

  const commonPasswords = ['password', '123456', 'qwerty', 'admin'];
  if (commonPasswords.includes(password.toLowerCase())) errors.push('Password cannot be a common password');
  if (/(.)\1{2,}/.test(password)) errors.push('Password cannot contain repeated characters');

  return { isValid: errors.length === 0, errors };
}
```

In production, check against a breached-password list (the Have I Been Pwned API's k-anonymity range search is the standard approach) rather than a hardcoded list of four common passwords — the point is catching passwords already known to attackers, not just obvious ones.

### Password hashing
Never store passwords in plaintext or with a fast hash (MD5, SHA-256 alone) — those are designed to be fast, which is exactly wrong for password storage, since it makes offline brute-forcing cheap. Use an adaptive, deliberately slow algorithm with a per-password salt built in.

```javascript
const bcrypt = require('bcrypt');

class PasswordManager {
  constructor() {
    this.saltRounds = 12; // tune upward as hardware gets faster; benchmark on your target hardware
  }

  async hashPassword(password) {
    return await bcrypt.hash(password, this.saltRounds);
  }

  async verifyPassword(password, hash) {
    return await bcrypt.compare(password, hash);
  }

  async needsRehash(hash) {
    return await bcrypt.getRounds(hash) < this.saltRounds; // re-hash on next successful login if rounds increased
  }
}
```

Argon2id is the current OWASP-recommended default over bcrypt where available (better resistance to GPU/ASIC cracking), but bcrypt remains a solid, widely-supported choice.

### Brute-force and account-lockout protection
Two complementary layers: rate limiting (slows down *any* client hammering the endpoint) and account lockout (protects one specific account regardless of source IP, since credential-stuffing attacks come from many IPs).

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('redis');

const loginLimiter = rateLimit({
  store: new RedisStore({ client: redis.createClient(), prefix: 'login_limit:' }),
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Too many login attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  skipSuccessfulRequests: true
});

// Progressive delay as an additional layer, independent of the hard rate limit
const progressiveDelay = (attempts) => {
  const baseDelay = 1000;
  const maxDelay = 300000; // 5 minutes
  return Math.min(baseDelay * Math.pow(2, attempts), maxDelay);
};
```

```javascript
class AccountLockout {
  constructor() {
    this.maxAttempts = 5;
    this.lockoutDuration = 30 * 60 * 1000;
    this.failedAttempts = new Map(); // use Redis in production — this Map is per-process and won't survive a restart or scale across instances
  }

  recordFailedAttempt(username) {
    const attempts = this.failedAttempts.get(username) || { count: 0, firstAttempt: Date.now() };
    attempts.count++;
    if (attempts.count === 1) attempts.firstAttempt = Date.now();
    this.failedAttempts.set(username, attempts);
  }

  isLocked(username) {
    const attempts = this.failedAttempts.get(username);
    if (!attempts) return false;
    if (attempts.count >= this.maxAttempts) {
      if (Date.now() - attempts.firstAttempt < this.lockoutDuration) return true;
      this.failedAttempts.delete(username); // lockout window expired, reset
      return false;
    }
    return false;
  }

  resetAttempts(username) {
    this.failedAttempts.delete(username);
  }
}
```

⚠️ **Lockout gotcha**: a naive account-lockout policy is itself a denial-of-service vector — an attacker who knows a victim's username can deliberately trigger lockouts to deny them access. Mitigate by combining lockout with rate limiting per-IP as well as per-account, and consider a CAPTCHA challenge instead of a hard lockout past a certain attempt threshold.

## 2. Session security

Concept and trade-offs of session-based auth are in [1-authentication.md](<1-authentication.md>); this is the concrete secure configuration.

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

const sessionConfig = {
  store: new RedisStore({ client: redis.createClient(), prefix: 'sess:' }),
  secret: process.env.SESSION_SECRET,
  name: 'sessionId', // never ship the framework default ('connect.sid') — it fingerprints your stack
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000,
    domain: process.env.COOKIE_DOMAIN,
    path: '/'
  },
  rolling: true, // extend session on activity rather than a fixed absolute expiry
  unset: 'destroy'
};
```

### Session fixation prevention
An attacker who can set a victim's session ID *before* login (e.g. via a crafted link) can hijack the session once the victim authenticates, if the server keeps using the same ID post-login. Regenerating the session ID on successful authentication closes this:

```javascript
app.post('/login', (req, res) => {
  if (isValidCredentials(req.body)) {
    req.session.regenerate((err) => {
      if (err) return res.status(500).send('Login failed');
      req.session.userId = user.id;
      req.session.userRole = user.role;
      req.session.loginTime = Date.now();
      res.redirect('/dashboard');
    });
  } else {
    res.status(401).send('Invalid credentials');
  }
});
```

### Session invalidation on logout
```javascript
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) return res.status(500).send('Logout failed');
    res.clearCookie('sessionId', {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      domain: process.env.COOKIE_DOMAIN,
      path: '/'
    });
    res.redirect('/login');
  });
});
```

## 3. JWT security

Full reasoning on revocation strategy, algorithm attacks, and storage trade-offs is in [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>). This is the concrete implementation.

```javascript
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

class JWTSecurity {
  constructor() {
    this.secret = process.env.JWT_SECRET;
    this.algorithm = 'HS256';
    this.accessTokenExpiry = '15m';
    this.refreshTokenExpiry = '7d';
  }

  generateAccessToken(payload) {
    return jwt.sign(payload, this.secret, {
      algorithm: this.algorithm,
      expiresIn: this.accessTokenExpiry,
      issuer: 'your-app',
      audience: 'your-app-users',
      jwtid: crypto.randomBytes(16).toString('hex')
    });
  }

  generateRefreshToken(payload) {
    return jwt.sign(payload, this.secret, {
      algorithm: this.algorithm,
      expiresIn: this.refreshTokenExpiry,
      issuer: 'your-app',
      audience: 'your-app-refresh',
      jwtid: crypto.randomBytes(16).toString('hex')
    });
  }

  verifyToken(token, audience) {
    try {
      // Always pin `algorithms` explicitly — never let the token's own header dictate this
      return jwt.verify(token, this.secret, { algorithms: [this.algorithm], issuer: 'your-app', audience });
    } catch (error) {
      throw new Error('Invalid token');
    }
  }
}
```

### Token blacklisting (denylist)
For the subset of tokens needing instant revocation (see the JWT file's revocation strategy comparison table) — an extra lookup on every request, which is exactly why this is applied selectively, not universally.

```javascript
class TokenBlacklist {
  constructor() {
    this.blacklist = new Set(); // use Redis with a TTL matching token exp in production
  }

  blacklistToken(jti) {
    this.blacklist.add(jti);
  }

  isBlacklisted(jti) {
    return this.blacklist.has(jti);
  }
}

const checkBlacklist = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (token) {
    try {
      const decoded = jwt.decode(token);
      if (decoded && tokenBlacklist.isBlacklisted(decoded.jti)) {
        return res.status(401).json({ error: 'Token has been revoked' });
      }
    } catch (error) {
      return res.status(401).json({ error: 'Invalid token' });
    }
  }
  next();
};
```

## 4. OAuth 2.0 security

Full reasoning on PKCE mechanics and the `state` parameter is in [3-Oauth Guide.md](<3-Oauth Guide.md>). Implementation:

```javascript
class OAuthSecurity {
  constructor() {
    this.stateStore = new Map(); // use Redis in production
    this.pkceStore = new Map();
  }

  generateState() {
    const state = crypto.randomBytes(32).toString('hex');
    this.stateStore.set(state, { timestamp: Date.now(), used: false });
    return state;
  }

  validateState(state) {
    const stateData = this.stateStore.get(state);
    if (!stateData || stateData.used) return false;
    if (Date.now() - stateData.timestamp > 5 * 60 * 1000) {
      this.stateStore.delete(state);
      return false;
    }
    stateData.used = true; // single-use — prevents replay of a captured callback URL
    return true;
  }

  generatePKCE() {
    const codeVerifier = crypto.randomBytes(32).toString('base64url');
    const codeChallenge = crypto.createHash('sha256').update(codeVerifier).digest('base64url');
    return { codeVerifier, codeChallenge };
  }

  storePKCE(codeChallenge, codeVerifier) {
    this.pkceStore.set(codeChallenge, { codeVerifier, timestamp: Date.now() });
  }

  validatePKCE(codeChallenge, codeVerifier) {
    const stored = this.pkceStore.get(codeChallenge);
    if (!stored || stored.codeVerifier !== codeVerifier) return false;
    if (Date.now() - stored.timestamp > 10 * 60 * 1000) {
      this.pkceStore.delete(codeChallenge);
      return false;
    }
    this.pkceStore.delete(codeChallenge); // single-use
    return true;
  }
}
```

### Redirect URI validation
Exact-match allowlisting only — never substring match or wildcard, both of which are routinely bypassed (`https://your-app.com.attacker.com` passes a naive substring check).

```javascript
class RedirectURIValidator {
  constructor() {
    this.allowedRedirects = [
      'https://your-app.com/callback',
      'https://your-app.com/auth/callback',
      'http://localhost:3000/callback' // development only — strip before deploying
    ];
  }

  validateRedirectURI(redirectUri) {
    if (!this.allowedRedirects.includes(redirectUri)) return false;
    if (process.env.NODE_ENV === 'production') {
      const url = new URL(redirectUri);
      if (url.protocol !== 'https:') return false;
    }
    return true;
  }
}
```

## 5. CSRF protection

Relevant primarily to cookie/session-based auth (a Bearer-token API that never auto-attaches credentials isn't CSRF-exposed the same way) — see [1-authentication.md](<1-authentication.md>)'s cookie section for why `SameSite` alone isn't always considered sufficient defense-in-depth.

```javascript
const csrf = require('csurf');

const csrfProtection = csrf({
  cookie: { httpOnly: true, secure: process.env.NODE_ENV === 'production', sameSite: 'strict' }
});

app.use('/api', csrfProtection);

app.use((err, req, res, next) => {
  if (err.code === 'EBADCSRFTOKEN') {
    res.status(403).json({ error: 'CSRF token validation failed' });
  } else {
    next(err);
  }
});

app.get('/api/csrf-token', (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});
```

### Double-submit cookie pattern
An alternative that doesn't require server-side session storage of the CSRF token — the server just checks that a value readable by JS (in a non-httpOnly cookie) matches a value the client explicitly sent in a header, which a cross-site attacker can't do since they can't read the victim's cookies.

```javascript
class DoubleSubmitCSRF {
  constructor() {
    this.cookieName = 'XSRF-TOKEN';
  }

  generateToken() {
    return crypto.randomBytes(32).toString('hex');
  }

  setCookie(res, token) {
    res.cookie(this.cookieName, token, {
      httpOnly: false, // must be JS-readable so the client can echo it back in a header
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict'
    });
  }

  validateToken(cookieToken, headerToken) {
    if (!cookieToken || !headerToken) return false;
    // Constant-time compare — a naive === comparison leaks timing information
    return crypto.timingSafeEqual(Buffer.from(cookieToken, 'hex'), Buffer.from(headerToken, 'hex'));
  }
}
```

## 6. Input validation and sanitization

Authentication endpoints are a favorite injection target precisely because they're unauthenticated by definition — validate everything before it touches a query or a template.

```javascript
const Joi = require('joi');

const validationSchemas = {
  login: Joi.object({
    email: Joi.string().email().required(),
    password: Joi.string().min(8).required()
  }),
  register: Joi.object({
    email: Joi.string().email().required(),
    password: Joi.string().min(12).pattern(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/
    ).required(),
    confirmPassword: Joi.string().valid(Joi.ref('password')).required(),
    firstName: Joi.string().min(2).max(50).required(),
    lastName: Joi.string().min(2).max(50).required()
  })
};

const validate = (schema) => (req, res, next) => {
  const { error } = schema.validate(req.body);
  if (error) {
    return res.status(400).json({ error: 'Validation failed', details: error.details.map(d => d.message) });
  }
  next();
};
```

### SQL injection prevention
Parameterized queries, always — string concatenation into a query is the root cause of essentially every SQL injection vulnerability, in login forms most of all since attackers specifically target them.

```javascript
const mysql = require('mysql2/promise');

class SecureDatabase {
  constructor() {
    this.pool = mysql.createPool({
      host: process.env.DB_HOST, user: process.env.DB_USER,
      password: process.env.DB_PASSWORD, database: process.env.DB_NAME,
      connectionLimit: 10
    });
  }

  async authenticateUser(email, password) {
    const [rows] = await this.pool.execute(
      'SELECT id, email, password_hash FROM users WHERE email = ?',
      [email] // parameterized — never `WHERE email = '${email}'`
    );
    if (rows.length === 0) return null;
    const user = rows[0];
    const isValid = await bcrypt.compare(password, user.password_hash);
    return isValid ? user : null;
  }
}
```

## 7. Rate limiting and DDoS protection

Different endpoints warrant different limits — login attempts are cheap to make and worth heavily throttling; registration and password reset can be abused for enumeration/spam and warrant tighter limits than general API traffic.

```javascript
const rateLimits = {
  login: rateLimit({
    store: new RedisStore({ client: redis.createClient(), prefix: 'login_limit:' }),
    windowMs: 15 * 60 * 1000, max: 5, message: 'Too many login attempts'
  }),
  register: rateLimit({
    store: new RedisStore({ client: redis.createClient(), prefix: 'register_limit:' }),
    windowMs: 60 * 60 * 1000, max: 3, message: 'Too many registration attempts'
  }),
  passwordReset: rateLimit({
    store: new RedisStore({ client: redis.createClient(), prefix: 'reset_limit:' }),
    windowMs: 60 * 60 * 1000, max: 3, message: 'Too many password reset attempts'
  }),
  api: rateLimit({
    store: new RedisStore({ client: redis.createClient(), prefix: 'api_limit:' }),
    windowMs: 60 * 1000, max: 100, message: 'Too many API requests'
  })
};
```

A hand-rolled sliding-window limiter, for context on what libraries like `express-rate-limit` do internally:

```javascript
class IPRateLimiter {
  constructor() {
    this.limits = new Map();
    this.cleanupInterval = setInterval(() => this.cleanup(), 60000);
  }

  isAllowed(ip, limit, windowMs) {
    const key = `${ip}:${limit}:${windowMs}`;
    const now = Date.now();
    const windowStart = now - windowMs;
    const requests = this.limits.get(key) || [];
    const validRequests = requests.filter(t => t > windowStart);

    if (validRequests.length >= limit) return false;
    validRequests.push(now);
    this.limits.set(key, validRequests);
    return true;
  }

  cleanup() {
    const now = Date.now();
    for (const [key, requests] of this.limits.entries()) {
      const windowMs = parseInt(key.split(':')[2]);
      const validRequests = requests.filter(t => t > now - windowMs);
      if (validRequests.length === 0) this.limits.delete(key);
      else this.limits.set(key, validRequests);
    }
  }
}
```

## 8. Security headers

HTTP response headers are a cheap, high-leverage layer — they instruct the *browser* to enforce restrictions the server can't otherwise guarantee (blocking inline scripts, refusing to be framed, forcing HTTPS).

```javascript
const helmet = require('helmet');

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"]
    }
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  noSniff: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  frameguard: { action: 'deny' },
  xssFilter: true
}));

app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
  next();
});
```

## 9. Audit logging

Every authentication event — success and failure — should produce a structured log entry with enough context to reconstruct an incident timeline after the fact. Logs are worthless for incident response if they only capture failures, or if they lack IP/user-agent/timestamp granularity.

```javascript
const winston = require('winston');

class AuditLogger {
  constructor() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(winston.format.timestamp(), winston.format.json()),
      transports: [new winston.transports.File({ filename: 'audit.log' }), new winston.transports.Console()]
    });
  }

  logAuthEvent(event) {
    this.logger.info('Authentication event', {
      timestamp: new Date().toISOString(),
      event: event.type,
      userId: event.userId,
      ipAddress: event.ipAddress,
      userAgent: event.userAgent,
      success: event.success,
      details: event.details,
      sessionId: event.sessionId
    });
  }

  logSecurityEvent(event) {
    this.logger.warn('Security event', {
      timestamp: new Date().toISOString(),
      event: event.type,
      severity: event.severity,
      ipAddress: event.ipAddress,
      userAgent: event.userAgent,
      details: event.details,
      action: event.action
    });
  }
}
```

⚠️ **Never log raw passwords, tokens, or session IDs** — even in a failure event where you're tempted to log "what was submitted" for debugging. Log a hash or a redacted form if you need to correlate failed attempts.

## 10. Secure development practices

### Environment configuration and validation
Secrets belong in environment variables (or a proper secrets manager) never in source — and validate that required config is actually present and well-formed at boot, so misconfiguration fails loudly at startup instead of silently at the first request that needs it.

```javascript
// .env.example — a template, never commit real values
NODE_ENV=development
SESSION_SECRET=your-super-secret-session-key
JWT_SECRET=your-super-secret-jwt-key
DB_HOST=localhost
REDIS_URL=redis://localhost:6379
```

```javascript
const Joi = require('joi');

const configSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
  PORT: Joi.number().port().default(3000),
  SESSION_SECRET: Joi.string().min(32).required(),
  JWT_SECRET: Joi.string().min(32).required(),
  DB_HOST: Joi.string().required(),
  REDIS_URL: Joi.string().uri().required()
});

function validateConfig() {
  const { error, value } = configSchema.validate(process.env);
  if (error) throw new Error(`Configuration validation failed: ${error.message}`);
  return value;
}
```

### Error handling
Don't leak stack traces or internal error details in production responses — they're a reconnaissance gift to an attacker probing your auth endpoints.

```javascript
app.use((err, req, res, next) => {
  console.error(err.stack); // full detail to server logs, always

  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' }); // generic to the client
  } else {
    res.status(500).json({ error: err.message, stack: err.stack });
  }
});

app.use((req, res) => res.status(404).json({ error: 'Not found' }));
```

## Quick reference checklist
- [ ] Passwords hashed with bcrypt/Argon2id, never a fast hash.
- [ ] Rate limiting on login, register, and password-reset endpoints, separately tuned.
- [ ] Account lockout paired with IP-based limiting to avoid becoming a DoS vector itself.
- [ ] Session ID regenerated on login (fixation prevention); destroyed on logout.
- [ ] JWT algorithm explicitly pinned on every verify call; short expiry + refresh rotation.
- [ ] OAuth `state` validated and single-use; PKCE used for all client types.
- [ ] Redirect URIs exact-match allowlisted.
- [ ] CSRF protection on all state-changing, cookie-authenticated endpoints.
- [ ] All user input validated server-side; all queries parameterized.
- [ ] Security headers set (CSP, HSTS, X-Frame-Options, etc.).
- [ ] Auth events audit-logged, without ever logging raw credentials/tokens.
- [ ] Config validated at boot; secrets never in source; generic error messages in production.

## Further reading
- OWASP Authentication Cheat Sheet, OWASP ASVS (Application Security Verification Standard).
- [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>) for the reasoning behind the JWT controls here.
- [3-Oauth Guide.md](<3-Oauth Guide.md>) for the reasoning behind the OAuth controls here.
- [Authentication Testing and Monitoring.md](<Authentication Testing and Monitoring.md>) for verifying these controls actually work in practice.
