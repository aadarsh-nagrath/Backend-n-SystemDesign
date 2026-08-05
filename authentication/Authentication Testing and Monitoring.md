# Authentication testing and monitoring

Testing strategies and operational monitoring for authentication systems — unit, integration, security, and load testing, plus the metrics/alerting/health-check layer that tells you the system is behaving correctly in production. Pairs with [Authentication Security Best Practices.md](<Authentication Security Best Practices.md>), which this file verifies.

## TL;DR
- **Unit tests** verify individual components (password validation, JWT signing, OAuth state/PKCE) in isolation.
- **Integration tests** verify the full request/response cycle through real endpoints — this is where rate limiting, token refresh, and logout actually get exercised end-to-end.
- **Security tests** deliberately probe for SQL injection, XSS, and CSRF gaps — the same classes of bug [Authentication Security Best Practices.md](<Authentication Security Best Practices.md>) defends against.
- **Monitoring** (metrics, health checks, alerting) is what tells you these defenses are still working *after* deployment, not just at test time — a rate limiter that silently stops working in production is exactly as bad as never having written it.

## 1. Unit testing authentication components

### Password validation
```javascript
const { expect } = require('chai');
const { validatePassword } = require('../utils/passwordValidator');

describe('Password Validation', () => {
  describe('Valid Passwords', () => {
    it('should accept a strong password', () => {
      const result = validatePassword('StrongPass123!');
      expect(result.isValid).to.be.true;
      expect(result.errors).to.have.length(0);
    });
  });

  describe('Invalid Passwords', () => {
    it('should reject short passwords', () => {
      const result = validatePassword('Short1!');
      expect(result.isValid).to.be.false;
      expect(result.errors).to.include('Password must be at least 12 characters');
    });

    it('should reject common passwords', () => {
      const result = validatePassword('password123!');
      expect(result.isValid).to.be.false;
      expect(result.errors).to.include('Password cannot be a common password');
    });
  });
});
```

### JWT operations
Tests the concrete claims/config from the [JWT security implementation](<Authentication Security Best Practices.md>) — signature, expiry, audience, and `jti` uniqueness are exactly the properties that matter for the revocation strategy discussed in the [JWT file](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>).

```javascript
const jwt = require('jsonwebtoken');
const { JWTSecurity } = require('../utils/jwtSecurity');

describe('JWT Security', () => {
  let jwtSecurity;
  const testPayload = { userId: '123', email: 'test@example.com' };

  beforeEach(() => { jwtSecurity = new JWTSecurity(); });

  describe('Token Generation', () => {
    it('should generate valid access tokens with correct claims', () => {
      const token = jwtSecurity.generateAccessToken(testPayload);
      const decoded = jwt.decode(token);
      expect(decoded.userId).to.equal('123');
      expect(decoded.iss).to.equal('your-app');
      expect(decoded.aud).to.equal('your-app-users');
    });

    it('should include a unique JWT ID on every token', () => {
      const token1 = jwtSecurity.generateAccessToken(testPayload);
      const token2 = jwtSecurity.generateAccessToken(testPayload);
      expect(jwt.decode(token1).jti).to.not.equal(jwt.decode(token2).jti);
    });
  });

  describe('Token Verification', () => {
    it('should verify valid tokens', () => {
      const token = jwtSecurity.generateAccessToken(testPayload);
      const decoded = jwtSecurity.verifyToken(token, 'your-app-users');
      expect(decoded.userId).to.equal('123');
    });

    it('should reject expired tokens', (done) => {
      const expiredToken = jwt.sign(testPayload, process.env.JWT_SECRET, {
        expiresIn: '1ms', issuer: 'your-app', audience: 'your-app-users'
      });
      setTimeout(() => {
        expect(() => jwtSecurity.verifyToken(expiredToken, 'your-app-users')).to.throw('Invalid token');
        done();
      }, 10);
    });

    it('should reject tokens with the wrong audience', () => {
      const token = jwtSecurity.generateAccessToken(testPayload);
      expect(() => jwtSecurity.verifyToken(token, 'wrong-audience')).to.throw('Invalid token');
    });

    it('should reject a forged alg:none token', () => {
      // Regression test for the alg:none attack — see the JWT file for the underlying vulnerability
      const header = Buffer.from(JSON.stringify({ alg: 'none', typ: 'JWT' })).toString('base64url');
      const payload = Buffer.from(JSON.stringify({ ...testPayload, roles: ['admin'] })).toString('base64url');
      const forgedToken = `${header}.${payload}.`;
      expect(() => jwtSecurity.verifyToken(forgedToken, 'your-app-users')).to.throw();
    });
  });
});
```

### OAuth security (state + PKCE)
Verifies the [OAuth security implementation](<Authentication Security Best Practices.md>) — single-use `state`, expiry, and PKCE verifier/challenge matching are exactly the properties the [OAuth file's PKCE walkthrough](<3-Oauth Guide.md>) depends on.

```javascript
const { OAuthSecurity } = require('../utils/oauthSecurity');

describe('OAuth Security', () => {
  let oauthSecurity;
  beforeEach(() => { oauthSecurity = new OAuthSecurity(); });

  describe('State Parameter', () => {
    it('should generate unique, sufficiently long state values', () => {
      const state1 = oauthSecurity.generateState();
      const state2 = oauthSecurity.generateState();
      expect(state1).to.not.equal(state2);
      expect(state1).to.have.length(64);
    });

    it('should reject a state value used twice (replay)', () => {
      const state = oauthSecurity.generateState();
      oauthSecurity.validateState(state);
      expect(oauthSecurity.validateState(state)).to.be.false;
    });

    it('should reject expired state', () => {
      const state = oauthSecurity.generateState();
      const stateData = oauthSecurity.stateStore.get(state);
      stateData.timestamp = Date.now() - (6 * 60 * 1000);
      expect(oauthSecurity.validateState(state)).to.be.false;
    });
  });

  describe('PKCE', () => {
    it('should generate a distinct verifier/challenge pair', () => {
      const { codeVerifier, codeChallenge } = oauthSecurity.generatePKCE();
      expect(codeVerifier).to.not.equal(codeChallenge);
    });

    it('should reject an incorrect verifier against a stored challenge', () => {
      const { codeVerifier, codeChallenge } = oauthSecurity.generatePKCE();
      oauthSecurity.storePKCE(codeChallenge, codeVerifier);
      expect(oauthSecurity.validatePKCE(codeChallenge, 'wrong-verifier')).to.be.false;
    });
  });
});
```

## 2. Integration testing

Exercises real endpoints end-to-end — this is where bugs in the *wiring* between components (not just the components themselves) surface: a rate limiter configured but never actually applied to the route, a refresh endpoint that doesn't reject a reused token, etc.

```javascript
const request = require('supertest');
const { expect } = require('chai');
const app = require('../app');
const { createTestUser, cleanupTestUser } = require('./testHelpers');

describe('Authentication Endpoints', () => {
  let testUser;

  before(async () => {
    testUser = await createTestUser({ email: 'test@example.com', password: 'TestPass123!' });
  });
  after(async () => { await cleanupTestUser(testUser.id); });

  describe('POST /auth/login', () => {
    it('should authenticate valid credentials', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({ email: 'test@example.com', password: 'TestPass123!' })
        .expect(200);
      expect(response.body).to.have.property('accessToken');
      expect(response.body).to.have.property('refreshToken');
    });

    it('should reject invalid credentials', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({ email: 'test@example.com', password: 'wrongpassword' })
        .expect(401);
      expect(response.body.error).to.equal('Invalid credentials');
    });

    it('should enforce rate limiting after repeated failures', async () => {
      const promises = Array(6).fill().map(() =>
        request(app).post('/auth/login').send({ email: 'test@example.com', password: 'wrongpassword' })
      );
      const responses = await Promise.all(promises);
      const lastResponse = responses[responses.length - 1];
      expect(lastResponse.status).to.equal(429);
    });
  });

  describe('POST /auth/refresh', () => {
    it('should issue a new access token, distinct from the original', async () => {
      const loginResponse = await request(app)
        .post('/auth/login')
        .send({ email: 'test@example.com', password: 'TestPass123!' });
      const { refreshToken } = loginResponse.body;

      const response = await request(app).post('/auth/refresh').send({ refreshToken }).expect(200);
      expect(response.body.accessToken).to.not.equal(loginResponse.body.accessToken);
    });

    it('should reject a refresh token that was already rotated (reuse detection)', async () => {
      const loginResponse = await request(app)
        .post('/auth/login')
        .send({ email: 'test@example.com', password: 'TestPass123!' });
      const { refreshToken } = loginResponse.body;

      await request(app).post('/auth/refresh').send({ refreshToken }); // first use — rotates it
      const secondAttempt = await request(app).post('/auth/refresh').send({ refreshToken }); // reuse
      expect(secondAttempt.status).to.equal(401);
    });
  });

  describe('POST /auth/logout', () => {
    it('should invalidate the session/token so it cannot be reused', async () => {
      const loginResponse = await request(app)
        .post('/auth/login')
        .send({ email: 'test@example.com', password: 'TestPass123!' });
      const { accessToken } = loginResponse.body;

      await request(app).post('/auth/logout').set('Authorization', `Bearer ${accessToken}`).expect(200);
      // The real assertion that matters: the token is actually dead afterward
      await request(app).get('/api/profile').set('Authorization', `Bearer ${accessToken}`).expect(401);
    });
  });
});
```

### Protected route access control
```javascript
describe('Protected Routes', () => {
  let accessToken;

  before(async () => {
    const loginResponse = await request(app)
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'TestPass123!' });
    accessToken = loginResponse.body.accessToken;
  });

  describe('GET /api/profile', () => {
    it('should return the profile with a valid token', async () => {
      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200);
      expect(response.body.user.email).to.equal('test@example.com');
    });

    it('should reject requests with no token', async () => {
      await request(app).get('/api/profile').expect(401);
    });

    it('should reject a syntactically invalid token', async () => {
      await request(app).get('/api/profile').set('Authorization', 'Bearer invalid-token').expect(401);
    });

    it('should reject an expired token', async () => {
      const expiredToken = jwt.sign({ userId: '123' }, process.env.JWT_SECRET, { expiresIn: '1ms' });
      await new Promise(resolve => setTimeout(resolve, 10));
      await request(app).get('/api/profile').set('Authorization', `Bearer ${expiredToken}`).expect(401);
    });
  });
});
```

## 3. Security testing

Deliberately probes for the exact vulnerability classes [Authentication Security Best Practices.md](<Authentication Security Best Practices.md>) defends against — this is regression testing for security controls, not a substitute for a real penetration test.

```javascript
describe('Security Testing', () => {
  describe('SQL Injection Prevention', () => {
    it('should not error or leak data on injection payloads', async () => {
      const maliciousPayloads = ["'; DROP TABLE users; --", "' OR '1'='1", "admin'--", "' UNION SELECT * FROM users --"];
      for (const payload of maliciousPayloads) {
        const response = await request(app).post('/auth/login').send({ email: payload, password: 'anypassword' });
        expect(response.status).to.not.equal(500); // a 500 here suggests the query broke, not that it was safely rejected
      }
    });
  });

  describe('XSS Prevention', () => {
    it('should sanitize or reject script payloads in registration fields', async () => {
      const xssPayloads = ['<script>alert("xss")</script>', '<img src="x" onerror="alert(1)">'];
      for (const payload of xssPayloads) {
        const response = await request(app).post('/auth/register').send({
          email: 'test@example.com', password: 'TestPass123!', confirmPassword: 'TestPass123!',
          firstName: payload, lastName: 'User'
        });
        expect(response.status).to.be.oneOf([400, 201]);
        // If accepted, assert the stored/returned value is escaped, not the raw payload
      }
    });
  });

  describe('CSRF Protection', () => {
    it('should reject state-changing requests without a valid CSRF token', async () => {
      const response = await request(app).post('/auth/register').send({
        email: 'test@example.com', password: 'TestPass123!', confirmPassword: 'TestPass123!',
        firstName: 'Test', lastName: 'User'
      });
      expect(response.status).to.equal(403);
    });
  });

  describe('Brute Force Protection', () => {
    it('should apply progressive delay across repeated failures', async () => {
      const startTime = Date.now();
      for (let i = 0; i < 5; i++) {
        await request(app).post('/auth/login').send({ email: 'test@example.com', password: 'wrongpassword' });
      }
      expect(Date.now() - startTime).to.be.at.least(1000);
    });
  });
});
```

## 4. Load testing

Authentication endpoints are frequently the first thing to buckle under real traffic — they do expensive work (bcrypt hashing) by design, which makes them a natural target for both legitimate spikes and DoS attempts. Load-test them specifically, not just as part of general API load tests.

```javascript
const autocannon = require('autocannon');

describe('Load Testing', () => {
  it('should handle concurrent login requests within latency budget', async () => {
    const result = await autocannon({
      url: 'http://localhost:3000/auth/login',
      connections: 100,
      duration: 10,
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email: 'test@example.com', password: 'TestPass123!' })
    });

    expect(result.errors).to.equal(0);
    expect(result.timeouts).to.equal(0);
    expect(result.latency.p99).to.be.below(1000); // p99 under 1s
  });
});
```

⚠️ **bcrypt is deliberately slow** — that's the security property, not a bug — so login endpoints will always have a materially higher latency floor than the rest of your API. Load-test with that expectation, and make sure your rate limiter (not raw server capacity) is what caps abusive traffic, since scaling hardware to make brute-forcing cheaper for legitimate load also makes it cheaper for attackers.

## 5. Monitoring and alerting

Testing proves the system works *at test time*; monitoring proves it's still working *right now*, in production, under real traffic and real attackers.

### Metrics
```javascript
const prometheus = require('prom-client');

class AuthMetrics {
  constructor() {
    this.loginAttempts = new prometheus.Counter({
      name: 'auth_login_attempts_total', help: 'Total login attempts', labelNames: ['success', 'method']
    });
    this.tokenRefreshes = new prometheus.Counter({
      name: 'auth_token_refreshes_total', help: 'Total token refreshes', labelNames: ['success']
    });
    this.loginDuration = new prometheus.Histogram({
      name: 'auth_login_duration_seconds', help: 'Login request duration', labelNames: ['success']
    });
    this.activeSessions = new prometheus.Gauge({
      name: 'auth_active_sessions', help: 'Number of active sessions'
    });
    this.failedLoginAttempts = new prometheus.Gauge({
      name: 'auth_failed_login_attempts', help: 'Failed login attempts in the last hour'
    });
  }

  recordLoginAttempt(success, method = 'password') {
    this.loginAttempts.inc({ success: success.toString(), method });
  }
  recordLoginDuration(success, duration) {
    this.loginDuration.observe({ success: success.toString() }, duration);
  }
}

const authMetrics = new AuthMetrics();

app.post('/auth/login', async (req, res) => {
  const startTime = Date.now();
  try {
    const user = await authenticateUser(req.body.email, req.body.password);
    const duration = (Date.now() - startTime) / 1000;
    authMetrics.recordLoginAttempt(!!user, 'password');
    authMetrics.recordLoginDuration(!!user, duration);

    if (user) {
      req.session.userId = user.id;
      res.json({ message: 'Login successful' });
    } else {
      res.status(401).json({ error: 'Invalid credentials' });
    }
  } catch (error) {
    authMetrics.recordLoginAttempt(false, 'password');
    res.status(500).json({ error: 'Login error' });
  }
});
```

The metrics worth tracking beyond simple counts: **login success rate over time** (a sudden drop can mean an outage; a sudden rise in failures can mean credential-stuffing), **p95/p99 latency** (catches both performance regressions and, indirectly, DoS attempts hammering the bcrypt-hashing path), and **active session/token count** (a sanity check against expected traffic — an unexplained spike is worth investigating).

### Health checks
Distinguish **liveness** (is the process up at all) from **readiness** (are its actual dependencies — DB, cache, email — reachable) — a process can be alive but unable to serve real requests if Redis is down, and a load balancer needs to know the difference to route traffic correctly.

```javascript
app.get('/health', async (req, res) => {
  const health = { status: 'ok', timestamp: new Date().toISOString(), checks: {} };

  try { await db.query('SELECT 1'); health.checks.database = 'ok'; }
  catch { health.checks.database = 'error'; health.status = 'error'; }

  try { await redis.ping(); health.checks.redis = 'ok'; }
  catch { health.checks.redis = 'error'; health.status = 'error'; }

  health.checks.jwt = process.env.JWT_SECRET ? 'ok' : 'error';
  if (health.checks.jwt === 'error') health.status = 'error';

  res.status(health.status === 'ok' ? 200 : 503).json(health);
});

app.get('/health/ready', async (req, res) => {
  const ready = { status: 'ready', services: {} };
  const checks = [
    { name: 'database', check: () => db.query('SELECT 1') },
    { name: 'redis', check: () => redis.ping() },
    { name: 'email', check: () => emailService.ping() }
  ];

  for (const { name, check } of checks) {
    try { await check(); ready.services[name] = 'ready'; }
    catch { ready.services[name] = 'not ready'; ready.status = 'not ready'; }
  }
  res.status(ready.status === 'ready' ? 200 : 503).json(ready);
});
```

### Alerting rules
Alert thresholds should reflect what's actually actionable — a spike that resolves itself in 30 seconds shouldn't page anyone, but a sustained trend should.

```yaml
groups:
  - name: authentication_alerts
    rules:
      - alert: HighLoginFailureRate
        expr: rate(auth_login_attempts_total{success="false"}[5m]) > 0.1
        for: 2m
        labels: { severity: warning }
        annotations:
          summary: "High login failure rate detected"
          description: "Login failure rate is {{ $value }} per second — possible credential-stuffing attack"

      - alert: TooManyFailedLogins
        expr: auth_failed_login_attempts > 100
        for: 1m
        labels: { severity: critical }
        annotations:
          summary: "Too many failed login attempts in the last hour"

      - alert: HighLoginLatency
        expr: histogram_quantile(0.95, rate(auth_login_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "95th percentile login latency is {{ $value }}s — check DB/bcrypt cost or DoS"

      - alert: AuthServiceDown
        expr: up{job="authentication-service"} == 0
        for: 1m
        labels: { severity: critical }
        annotations:
          summary: "Authentication service has been unreachable for over 1 minute"
```

## 6. Performance testing

Sets explicit latency budgets for the operations that matter most, and verifies the deliberately-slow parts (password hashing) stay within their intended cost — too fast and you've probably weakened the hash config; too slow and legitimate logins suffer.

```javascript
describe('Performance Testing', () => {
  it('should complete login within the acceptable budget', async () => {
    const startTime = Date.now();
    await request(app).post('/auth/login').send({ email: 'test@example.com', password: 'TestPass123!' }).expect(200);
    expect(Date.now() - startTime).to.be.below(500);
  });

  it('should hash and verify passwords within expected bcrypt cost', async () => {
    const passwordManager = new PasswordManager();
    const password = 'TestPassword123!';

    const hashStart = Date.now();
    const hash = await passwordManager.hashPassword(password);
    expect(Date.now() - hashStart).to.be.below(1000); // bcrypt at rounds=12 typically lands well under this

    const verifyStart = Date.now();
    const isValid = await passwordManager.verifyPassword(password, hash);
    expect(isValid).to.be.true;
    expect(Date.now() - verifyStart).to.be.below(1000);
  });

  it('should sign and verify JWTs quickly (these should be cheap, unlike password hashing)', async () => {
    const jwtSecurity = new JWTSecurity();
    const payload = { userId: '123' };

    const signStart = Date.now();
    const token = jwtSecurity.generateAccessToken(payload);
    expect(Date.now() - signStart).to.be.below(10);

    const verifyStart = Date.now();
    jwtSecurity.verifyToken(token, 'your-app-users');
    expect(Date.now() - verifyStart).to.be.below(10);
  });
});
```

## 7. Automated security scanning

CI-level checks that catch known-vulnerable dependencies and common web vulnerability classes before they reach production — complements, but doesn't replace, the manual security testing above and periodic real penetration testing.

```javascript
const { exec } = require('child_process');
const { promisify } = require('util');
const execAsync = promisify(exec);

describe('Security Scanning', () => {
  it('should have no moderate-or-higher npm audit findings', async () => {
    const { stdout } = await execAsync('npm audit --audit-level=moderate');
    expect(stdout).to.not.include('found');
  });

  it('should pass an OWASP ZAP baseline scan', async () => {
    const { stdout } = await execAsync('zap-baseline.py -t http://localhost:3000');
    expect(stdout).to.include('PASS');
  });
});
```

## Quick reference
- **Unit**: individual functions (password validation, JWT sign/verify, PKCE/state generation) in isolation.
- **Integration**: real endpoints, full request cycle — this is where rate limiting, refresh rotation, and logout invalidation actually get proven.
- **Security**: deliberate injection/XSS/CSRF probing, plus regression tests for known attack classes (e.g. `alg:none`).
- **Load**: concurrent traffic against auth endpoints specifically, with realistic bcrypt-cost latency expectations.
- **Monitoring**: liveness vs. readiness health checks, Prometheus metrics (attempt counts, latency percentiles, active sessions), alerting tuned to be actionable, not noisy.
- Run all of this continuously in CI, not just before a release — auth is the component where a regression is most costly to ship silently.

## Further reading
- [Authentication Security Best Practices.md](<Authentication Security Best Practices.md>) — the controls this file tests.
- [2-Comprehensive Guide to JSON Web Tokens (JWT).md](<2-Comprehensive Guide to JSON Web Tokens (JWT).md>) and [3-Oauth Guide.md](<3-Oauth Guide.md>) — the mechanisms behind the JWT/OAuth test suites above.
- OWASP Testing Guide (authentication testing chapter).
