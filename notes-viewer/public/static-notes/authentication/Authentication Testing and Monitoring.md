# Authentication Testing and Monitoring

## Introduction

This document covers comprehensive testing strategies and monitoring approaches for authentication systems. It includes unit testing, integration testing, security testing, and monitoring techniques to ensure your authentication system is robust, secure, and reliable.

## 1. Unit Testing Authentication Components

### Testing Password Validation
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

    it('should accept password with special characters', () => {
      const result = validatePassword('My@Secure#Pass1');
      expect(result.isValid).to.be.true;
    });
  });

  describe('Invalid Passwords', () => {
    it('should reject short passwords', () => {
      const result = validatePassword('Short1!');
      expect(result.isValid).to.be.false;
      expect(result.errors).to.include('Password must be at least 12 characters');
    });

    it('should reject passwords without uppercase', () => {
      const result = validatePassword('lowercase123!');
      expect(result.isValid).to.be.false;
      expect(result.errors).to.include('Password must contain at least one uppercase letter');
    });

    it('should reject common passwords', () => {
      const result = validatePassword('password123!');
      expect(result.isValid).to.be.false;
      expect(result.errors).to.include('Password cannot be a common password');
    });

    it('should reject passwords with repeated characters', () => {
      const result = validatePassword('StrongPass111!');
      expect(result.isValid).to.be.false;
      expect(result.errors).to.include('Password cannot contain repeated characters');
    });
  });
});
```

### Testing JWT Operations
```javascript
const { expect } = require('chai');
const jwt = require('jsonwebtoken');
const { JWTSecurity } = require('../utils/jwtSecurity');

describe('JWT Security', () => {
  let jwtSecurity;
  const testPayload = { userId: '123', email: 'test@example.com' };

  beforeEach(() => {
    jwtSecurity = new JWTSecurity();
  });

  describe('Token Generation', () => {
    it('should generate valid access tokens', () => {
      const token = jwtSecurity.generateAccessToken(testPayload);
      expect(token).to.be.a('string');
      
      const decoded = jwt.decode(token);
      expect(decoded.userId).to.equal('123');
      expect(decoded.email).to.equal('test@example.com');
      expect(decoded.iss).to.equal('your-app');
      expect(decoded.aud).to.equal('your-app-users');
    });

    it('should generate valid refresh tokens', () => {
      const token = jwtSecurity.generateRefreshToken(testPayload);
      expect(token).to.be.a('string');
      
      const decoded = jwt.decode(token);
      expect(decoded.aud).to.equal('your-app-refresh');
    });

    it('should include unique JWT ID', () => {
      const token1 = jwtSecurity.generateAccessToken(testPayload);
      const token2 = jwtSecurity.generateAccessToken(testPayload);
      
      const decoded1 = jwt.decode(token1);
      const decoded2 = jwt.decode(token2);
      
      expect(decoded1.jti).to.not.equal(decoded2.jti);
    });
  });

  describe('Token Verification', () => {
    it('should verify valid tokens', () => {
      const token = jwtSecurity.generateAccessToken(testPayload);
      const decoded = jwtSecurity.verifyToken(token, 'your-app-users');
      expect(decoded.userId).to.equal('123');
    });

    it('should reject expired tokens', () => {
      const expiredToken = jwt.sign(testPayload, process.env.JWT_SECRET, {
        expiresIn: '1ms',
        issuer: 'your-app',
        audience: 'your-app-users'
      });

      // Wait for token to expire
      setTimeout(() => {
        expect(() => {
          jwtSecurity.verifyToken(expiredToken, 'your-app-users');
        }).to.throw('Invalid token');
      }, 10);
    });

    it('should reject tokens with wrong audience', () => {
      const token = jwtSecurity.generateAccessToken(testPayload);
      expect(() => {
        jwtSecurity.verifyToken(token, 'wrong-audience');
      }).to.throw('Invalid token');
    });
  });
});
```

### Testing OAuth Security
```javascript
const { expect } = require('chai');
const { OAuthSecurity } = require('../utils/oauthSecurity');

describe('OAuth Security', () => {
  let oauthSecurity;

  beforeEach(() => {
    oauthSecurity = new OAuthSecurity();
  });

  describe('State Parameter', () => {
    it('should generate unique state values', () => {
      const state1 = oauthSecurity.generateState();
      const state2 = oauthSecurity.generateState();
      
      expect(state1).to.not.equal(state2);
      expect(state1).to.have.length(64);
    });

    it('should validate correct state', () => {
      const state = oauthSecurity.generateState();
      expect(oauthSecurity.validateState(state)).to.be.true;
    });

    it('should reject used state', () => {
      const state = oauthSecurity.generateState();
      oauthSecurity.validateState(state);
      expect(oauthSecurity.validateState(state)).to.be.false;
    });

    it('should reject expired state', () => {
      const state = oauthSecurity.generateState();
      
      // Simulate time passing
      const stateData = oauthSecurity.stateStore.get(state);
      stateData.timestamp = Date.now() - (6 * 60 * 1000); // 6 minutes ago
      
      expect(oauthSecurity.validateState(state)).to.be.false;
    });
  });

  describe('PKCE', () => {
    it('should generate valid PKCE pair', () => {
      const { codeVerifier, codeChallenge } = oauthSecurity.generatePKCE();
      
      expect(codeVerifier).to.be.a('string');
      expect(codeChallenge).to.be.a('string');
      expect(codeVerifier).to.not.equal(codeChallenge);
    });

    it('should validate correct PKCE', () => {
      const { codeVerifier, codeChallenge } = oauthSecurity.generatePKCE();
      oauthSecurity.storePKCE(codeChallenge, codeVerifier);
      
      expect(oauthSecurity.validatePKCE(codeChallenge, codeVerifier)).to.be.true;
    });

    it('should reject incorrect PKCE', () => {
      const { codeVerifier, codeChallenge } = oauthSecurity.generatePKCE();
      oauthSecurity.storePKCE(codeChallenge, codeVerifier);
      
      expect(oauthSecurity.validatePKCE(codeChallenge, 'wrong-verifier')).to.be.false;
    });
  });
});
```

## 2. Integration Testing

### Testing Authentication Endpoints
```javascript
const request = require('supertest');
const { expect } = require('chai');
const app = require('../app');
const { createTestUser, cleanupTestUser } = require('./testHelpers');

describe('Authentication Endpoints', () => {
  let testUser;

  before(async () => {
    testUser = await createTestUser({
      email: 'test@example.com',
      password: 'TestPass123!'
    });
  });

  after(async () => {
    await cleanupTestUser(testUser.id);
  });

  describe('POST /auth/login', () => {
    it('should authenticate valid credentials', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({
          email: 'test@example.com',
          password: 'TestPass123!'
        })
        .expect(200);

      expect(response.body).to.have.property('accessToken');
      expect(response.body).to.have.property('refreshToken');
      expect(response.body.user.email).to.equal('test@example.com');
    });

    it('should reject invalid credentials', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({
          email: 'test@example.com',
          password: 'wrongpassword'
        })
        .expect(401);

      expect(response.body.error).to.equal('Invalid credentials');
    });

    it('should enforce rate limiting', async () => {
      const promises = Array(6).fill().map(() =>
        request(app)
          .post('/auth/login')
          .send({
            email: 'test@example.com',
            password: 'wrongpassword'
          })
      );

      const responses = await Promise.all(promises);
      const lastResponse = responses[responses.length - 1];
      
      expect(lastResponse.status).to.equal(429);
      expect(lastResponse.body.error).to.include('Too many login attempts');
    });
  });

  describe('POST /auth/register', () => {
    it('should register new user with valid data', async () => {
      const response = await request(app)
        .post('/auth/register')
        .send({
          email: 'newuser@example.com',
          password: 'NewUserPass123!',
          confirmPassword: 'NewUserPass123!',
          firstName: 'New',
          lastName: 'User'
        })
        .expect(201);

      expect(response.body).to.have.property('accessToken');
      expect(response.body.user.email).to.equal('newuser@example.com');
    });

    it('should reject weak passwords', async () => {
      const response = await request(app)
        .post('/auth/register')
        .send({
          email: 'weak@example.com',
          password: 'weak',
          confirmPassword: 'weak',
          firstName: 'Weak',
          lastName: 'User'
        })
        .expect(400);

      expect(response.body.error).to.equal('Validation failed');
      expect(response.body.details).to.include('Password must be at least 12 characters');
    });

    it('should reject duplicate emails', async () => {
      const response = await request(app)
        .post('/auth/register')
        .send({
          email: 'test@example.com', // Already exists
          password: 'TestPass123!',
          confirmPassword: 'TestPass123!',
          firstName: 'Test',
          lastName: 'User'
        })
        .expect(409);

      expect(response.body.error).to.equal('Email already exists');
    });
  });

  describe('POST /auth/refresh', () => {
    it('should refresh valid tokens', async () => {
      // First login to get tokens
      const loginResponse = await request(app)
        .post('/auth/login')
        .send({
          email: 'test@example.com',
          password: 'TestPass123!'
        });

      const { refreshToken } = loginResponse.body;

      // Use refresh token
      const response = await request(app)
        .post('/auth/refresh')
        .send({ refreshToken })
        .expect(200);

      expect(response.body).to.have.property('accessToken');
      expect(response.body.accessToken).to.not.equal(loginResponse.body.accessToken);
    });

    it('should reject invalid refresh tokens', async () => {
      const response = await request(app)
        .post('/auth/refresh')
        .send({ refreshToken: 'invalid-token' })
        .expect(401);

      expect(response.body.error).to.equal('Invalid refresh token');
    });
  });

  describe('POST /auth/logout', () => {
    it('should logout successfully', async () => {
      // First login to get tokens
      const loginResponse = await request(app)
        .post('/auth/login')
        .send({
          email: 'test@example.com',
          password: 'TestPass123!'
        });

      const { accessToken } = loginResponse.body;

      const response = await request(app)
        .post('/auth/logout')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200);

      expect(response.body.message).to.equal('Logged out successfully');
    });
  });
});
```

### Testing Protected Routes
```javascript
describe('Protected Routes', () => {
  let accessToken;

  before(async () => {
    const loginResponse = await request(app)
      .post('/auth/login')
      .send({
        email: 'test@example.com',
        password: 'TestPass123!'
      });

    accessToken = loginResponse.body.accessToken;
  });

  describe('GET /api/profile', () => {
    it('should return user profile with valid token', async () => {
      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200);

      expect(response.body.user.email).to.equal('test@example.com');
    });

    it('should reject requests without token', async () => {
      const response = await request(app)
        .get('/api/profile')
        .expect(401);

      expect(response.body.error).to.equal('Access token missing');
    });

    it('should reject invalid tokens', async () => {
      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', 'Bearer invalid-token')
        .expect(401);

      expect(response.body.error).to.equal('Invalid token');
    });

    it('should reject expired tokens', async () => {
      // Create an expired token
      const expiredToken = jwt.sign(
        { userId: '123' },
        process.env.JWT_SECRET,
        { expiresIn: '1ms' }
      );

      // Wait for token to expire
      await new Promise(resolve => setTimeout(resolve, 10));

      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${expiredToken}`)
        .expect(401);

      expect(response.body.error).to.equal('Token expired');
    });
  });
});
```

## 3. Security Testing

### Penetration Testing Scenarios
```javascript
describe('Security Testing', () => {
  describe('SQL Injection Prevention', () => {
    it('should prevent SQL injection in login', async () => {
      const maliciousPayloads = [
        "'; DROP TABLE users; --",
        "' OR '1'='1",
        "admin'--",
        "' UNION SELECT * FROM users --"
      ];

      for (const payload of maliciousPayloads) {
        const response = await request(app)
          .post('/auth/login')
          .send({
            email: payload,
            password: 'anypassword'
          });

        // Should not crash and should return proper error
        expect(response.status).to.not.equal(500);
        expect(response.body.error).to.be.a('string');
      }
    });
  });

  describe('XSS Prevention', () => {
    it('should prevent XSS in user input', async () => {
      const xssPayloads = [
        '<script>alert("xss")</script>',
        'javascript:alert("xss")',
        '<img src="x" onerror="alert(\'xss\')">',
        '"><script>alert("xss")</script>'
      ];

      for (const payload of xssPayloads) {
        const response = await request(app)
          .post('/auth/register')
          .send({
            email: 'test@example.com',
            password: 'TestPass123!',
            confirmPassword: 'TestPass123!',
            firstName: payload,
            lastName: 'User'
          });

        // Should sanitize or reject malicious input
        expect(response.status).to.be.oneOf([400, 201]);
      }
    });
  });

  describe('CSRF Protection', () => {
    it('should require CSRF token for state-changing operations', async () => {
      const response = await request(app)
        .post('/auth/register')
        .send({
          email: 'test@example.com',
          password: 'TestPass123!',
          confirmPassword: 'TestPass123!',
          firstName: 'Test',
          lastName: 'User'
        });

      // Should require CSRF token
      expect(response.status).to.equal(403);
      expect(response.body.error).to.equal('CSRF token validation failed');
    });
  });

  describe('Brute Force Protection', () => {
    it('should implement progressive delays', async () => {
      const startTime = Date.now();
      
      // Make multiple failed attempts
      for (let i = 0; i < 5; i++) {
        await request(app)
          .post('/auth/login')
          .send({
            email: 'test@example.com',
            password: 'wrongpassword'
          });
      }

      const endTime = Date.now();
      const totalTime = endTime - startTime;
      
      // Should take at least 1 second due to progressive delays
      expect(totalTime).to.be.at.least(1000);
    });
  });
});
```

## 4. Load Testing

### Authentication Load Testing
```javascript
const autocannon = require('autocannon');

describe('Load Testing', () => {
  it('should handle concurrent login requests', async () => {
    const result = await autocannon({
      url: 'http://localhost:3000/auth/login',
      connections: 100,
      duration: 10,
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        email: 'test@example.com',
        password: 'TestPass123!'
      })
    });

    expect(result.errors).to.equal(0);
    expect(result.timeouts).to.equal(0);
    expect(result.latency.p99).to.be.below(1000); // 99th percentile under 1s
  });

  it('should handle concurrent registration requests', async () => {
    const result = await autocannon({
      url: 'http://localhost:3000/auth/register',
      connections: 50,
      duration: 10,
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        email: 'loadtest@example.com',
        password: 'LoadTestPass123!',
        confirmPassword: 'LoadTestPass123!',
        firstName: 'Load',
        lastName: 'Test'
      })
    });

    expect(result.errors).to.equal(0);
    expect(result.timeouts).to.equal(0);
  });
});
```

## 5. Monitoring and Alerting

### Authentication Metrics
```javascript
const prometheus = require('prom-client');

class AuthMetrics {
  constructor() {
    // Counters
    this.loginAttempts = new prometheus.Counter({
      name: 'auth_login_attempts_total',
      help: 'Total number of login attempts',
      labelNames: ['success', 'method']
    });

    this.registrations = new prometheus.Counter({
      name: 'auth_registrations_total',
      help: 'Total number of user registrations',
      labelNames: ['success']
    });

    this.tokenRefreshes = new prometheus.Counter({
      name: 'auth_token_refreshes_total',
      help: 'Total number of token refreshes',
      labelNames: ['success']
    });

    // Histograms
    this.loginDuration = new prometheus.Histogram({
      name: 'auth_login_duration_seconds',
      help: 'Login request duration in seconds',
      labelNames: ['success']
    });

    this.tokenValidationDuration = new prometheus.Histogram({
      name: 'auth_token_validation_duration_seconds',
      help: 'Token validation duration in seconds'
    });

    // Gauges
    this.activeSessions = new prometheus.Gauge({
      name: 'auth_active_sessions',
      help: 'Number of active sessions'
    });

    this.failedLoginAttempts = new prometheus.Gauge({
      name: 'auth_failed_login_attempts',
      help: 'Number of failed login attempts in the last hour'
    });
  }

  recordLoginAttempt(success, method = 'password') {
    this.loginAttempts.inc({ success: success.toString(), method });
  }

  recordLoginDuration(success, duration) {
    this.loginDuration.observe({ success: success.toString() }, duration);
  }

  recordRegistration(success) {
    this.registrations.inc({ success: success.toString() });
  }

  recordTokenRefresh(success) {
    this.tokenRefreshes.inc({ success: success.toString() });
  }

  recordTokenValidation(duration) {
    this.tokenValidationDuration.observe(duration);
  }

  setActiveSessions(count) {
    this.activeSessions.set(count);
  }

  setFailedLoginAttempts(count) {
    this.failedLoginAttempts.set(count);
  }
}

// Usage in authentication routes
const authMetrics = new AuthMetrics();

app.post('/auth/login', async (req, res) => {
  const startTime = Date.now();
  
  try {
    const user = await authenticateUser(req.body.email, req.body.password);
    
    if (user) {
      authMetrics.recordLoginAttempt(true, 'password');
      const duration = (Date.now() - startTime) / 1000;
      authMetrics.recordLoginDuration(true, duration);
      
      // Set up session
      req.session.userId = user.id;
      res.json({ message: 'Login successful' });
    } else {
      authMetrics.recordLoginAttempt(false, 'password');
      const duration = (Date.now() - startTime) / 1000;
      authMetrics.recordLoginDuration(false, duration);
      
      res.status(401).json({ error: 'Invalid credentials' });
    }
  } catch (error) {
    authMetrics.recordLoginAttempt(false, 'password');
    res.status(500).json({ error: 'Login error' });
  }
});
```

### Health Checks
```javascript
app.get('/health', async (req, res) => {
  const health = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    checks: {}
  };

  // Database health check
  try {
    await db.query('SELECT 1');
    health.checks.database = 'ok';
  } catch (error) {
    health.checks.database = 'error';
    health.status = 'error';
  }

  // Redis health check
  try {
    await redis.ping();
    health.checks.redis = 'ok';
  } catch (error) {
    health.checks.redis = 'error';
    health.status = 'error';
  }

  // JWT secret check
  if (process.env.JWT_SECRET) {
    health.checks.jwt = 'ok';
  } else {
    health.checks.jwt = 'error';
    health.status = 'error';
  }

  const statusCode = health.status === 'ok' ? 200 : 503;
  res.status(statusCode).json(health);
});

app.get('/health/ready', async (req, res) => {
  // More comprehensive readiness check
  const ready = {
    status: 'ready',
    timestamp: new Date().toISOString(),
    services: {}
  };

  // Check all required services
  const checks = [
    { name: 'database', check: () => db.query('SELECT 1') },
    { name: 'redis', check: () => redis.ping() },
    { name: 'email', check: () => emailService.ping() }
  ];

  for (const { name, check } of checks) {
    try {
      await check();
      ready.services[name] = 'ready';
    } catch (error) {
      ready.services[name] = 'not ready';
      ready.status = 'not ready';
    }
  }

  const statusCode = ready.status === 'ready' ? 200 : 503;
  res.status(statusCode).json(ready);
});
```

### Alerting Rules
```javascript
// Example Prometheus alerting rules
const alertingRules = `
groups:
  - name: authentication_alerts
    rules:
      - alert: HighLoginFailureRate
        expr: rate(auth_login_attempts_total{success="false"}[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High login failure rate detected"
          description: "Login failure rate is {{ $value }} per second"

      - alert: TooManyFailedLogins
        expr: auth_failed_login_attempts > 100
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Too many failed login attempts"
          description: "{{ $value }} failed login attempts in the last hour"

      - alert: HighLoginLatency
        expr: histogram_quantile(0.95, rate(auth_login_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High login latency detected"
          description: "95th percentile login latency is {{ $value }} seconds"

      - alert: DatabaseConnectionFailed
        expr: up{job="authentication-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Authentication service is down"
          description: "Authentication service has been down for more than 1 minute"
`;
```

## 6. Performance Testing

### Authentication Performance Benchmarks
```javascript
describe('Performance Testing', () => {
  it('should complete login within acceptable time', async () => {
    const startTime = Date.now();
    
    await request(app)
      .post('/auth/login')
      .send({
        email: 'test@example.com',
        password: 'TestPass123!'
      })
      .expect(200);

    const duration = Date.now() - startTime;
    expect(duration).to.be.below(500); // Should complete within 500ms
  });

  it('should handle password hashing efficiently', async () => {
    const passwordManager = new PasswordManager();
    const password = 'TestPassword123!';
    
    const startTime = Date.now();
    const hash = await passwordManager.hashPassword(password);
    const hashDuration = Date.now() - startTime;
    
    expect(hashDuration).to.be.below(1000); // Should hash within 1 second
    
    const verifyStartTime = Date.now();
    const isValid = await passwordManager.verifyPassword(password, hash);
    const verifyDuration = Date.now() - verifyStartTime;
    
    expect(isValid).to.be.true;
    expect(verifyDuration).to.be.below(100); // Should verify within 100ms
  });

  it('should handle JWT operations efficiently', async () => {
    const jwtSecurity = new JWTSecurity();
    const payload = { userId: '123', email: 'test@example.com' };
    
    const signStartTime = Date.now();
    const token = jwtSecurity.generateAccessToken(payload);
    const signDuration = Date.now() - signStartTime;
    
    expect(signDuration).to.be.below(10); // Should sign within 10ms
    
    const verifyStartTime = Date.now();
    const decoded = jwtSecurity.verifyToken(token, 'your-app-users');
    const verifyDuration = Date.now() - verifyStartTime;
    
    expect(decoded.userId).to.equal('123');
    expect(verifyDuration).to.be.below(10); // Should verify within 10ms
  });
});
```

## 7. Security Scanning

### Automated Security Testing
```javascript
const { exec } = require('child_process');
const { promisify } = require('util');

const execAsync = promisify(exec);

describe('Security Scanning', () => {
  it('should pass npm audit', async () => {
    const { stdout } = await execAsync('npm audit --audit-level=moderate');
    expect(stdout).to.not.include('found');
  });

  it('should pass OWASP ZAP scan', async () => {
    // This would require OWASP ZAP to be installed and configured
    const { stdout } = await execAsync('zap-baseline.py -t http://localhost:3000');
    expect(stdout).to.include('PASS');
  });

  it('should have no critical vulnerabilities', async () => {
    const { stdout } = await execAsync('npm audit --audit-level=high');
    expect(stdout).to.not.include('critical');
  });
});
```

## Conclusion

Comprehensive testing and monitoring are essential for maintaining a secure and reliable authentication system. This guide covers:

1. **Unit Testing**: Testing individual components in isolation
2. **Integration Testing**: Testing how components work together
3. **Security Testing**: Identifying vulnerabilities and attack vectors
4. **Load Testing**: Ensuring performance under stress
5. **Monitoring**: Real-time visibility into system health
6. **Alerting**: Proactive notification of issues
7. **Performance Testing**: Ensuring acceptable response times
8. **Security Scanning**: Automated vulnerability detection

Remember to:
- Run tests continuously in your CI/CD pipeline
- Monitor metrics in production
- Set up appropriate alerting thresholds
- Regularly review and update test cases
- Perform security audits and penetration testing
- Keep dependencies updated and secure

A well-tested and monitored authentication system provides confidence in its security and reliability while enabling quick detection and response to issues. 