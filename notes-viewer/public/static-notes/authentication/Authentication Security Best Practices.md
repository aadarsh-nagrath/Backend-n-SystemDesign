# Authentication Security Best Practices

## Introduction

This document provides comprehensive security best practices for implementing authentication systems. It covers common vulnerabilities, attack vectors, and defensive strategies to ensure your authentication system is robust and secure.

## 1. Password Security

### Password Policy Best Practices

#### Strong Password Requirements
```javascript
const passwordValidator = {
  minLength: 12,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: true,
  preventCommonPasswords: true,
  preventSequentialChars: true,
  preventRepeatedChars: true
};

function validatePassword(password) {
  const errors = [];
  
  if (password.length < passwordValidator.minLength) {
    errors.push(`Password must be at least ${passwordValidator.minLength} characters`);
  }
  
  if (!/[A-Z]/.test(password)) {
    errors.push('Password must contain at least one uppercase letter');
  }
  
  if (!/[a-z]/.test(password)) {
    errors.push('Password must contain at least one lowercase letter');
  }
  
  if (!/\d/.test(password)) {
    errors.push('Password must contain at least one number');
  }
  
  if (!/[!@#$%^&*(),.?":{}|<>]/.test(password)) {
    errors.push('Password must contain at least one special character');
  }
  
  // Check for common passwords
  const commonPasswords = ['password', '123456', 'qwerty', 'admin'];
  if (commonPasswords.includes(password.toLowerCase())) {
    errors.push('Password cannot be a common password');
  }
  
  // Check for sequential characters
  if (/(.)\1{2,}/.test(password)) {
    errors.push('Password cannot contain repeated characters');
  }
  
  return {
    isValid: errors.length === 0,
    errors
  };
}
```

#### Password Hashing
```javascript
const bcrypt = require('bcrypt');

class PasswordManager {
  constructor() {
    this.saltRounds = 12;
  }
  
  async hashPassword(password) {
    return await bcrypt.hash(password, this.saltRounds);
  }
  
  async verifyPassword(password, hash) {
    return await bcrypt.compare(password, hash);
  }
  
  async needsRehash(hash) {
    return await bcrypt.getRounds(hash) < this.saltRounds;
  }
}
```

### Password Attack Prevention

#### Brute Force Protection
```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('redis');

const loginLimiter = rateLimit({
  store: new RedisStore({
    client: redis.createClient(),
    prefix: 'login_limit:'
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts per window
  message: 'Too many login attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  skipSuccessfulRequests: true
});

// Progressive delays
const progressiveDelay = (attempts) => {
  const baseDelay = 1000; // 1 second
  const maxDelay = 300000; // 5 minutes
  return Math.min(baseDelay * Math.pow(2, attempts), maxDelay);
};
```

#### Account Lockout
```javascript
class AccountLockout {
  constructor() {
    this.maxAttempts = 5;
    this.lockoutDuration = 30 * 60 * 1000; // 30 minutes
    this.failedAttempts = new Map();
  }
  
  recordFailedAttempt(username) {
    const attempts = this.failedAttempts.get(username) || { count: 0, firstAttempt: Date.now() };
    attempts.count++;
    
    if (attempts.count === 1) {
      attempts.firstAttempt = Date.now();
    }
    
    this.failedAttempts.set(username, attempts);
  }
  
  isLocked(username) {
    const attempts = this.failedAttempts.get(username);
    if (!attempts) return false;
    
    if (attempts.count >= this.maxAttempts) {
      const timeSinceFirstAttempt = Date.now() - attempts.firstAttempt;
      if (timeSinceFirstAttempt < this.lockoutDuration) {
        return true;
      } else {
        // Reset after lockout period
        this.failedAttempts.delete(username);
        return false;
      }
    }
    
    return false;
  }
  
  resetAttempts(username) {
    this.failedAttempts.delete(username);
  }
}
```

## 2. Session Security

### Secure Session Configuration
```javascript
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

const sessionConfig = {
  store: new RedisStore({
    client: redis.createClient(),
    prefix: 'sess:'
  }),
  secret: process.env.SESSION_SECRET,
  name: 'sessionId', // Don't use default 'connect.sid'
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    domain: process.env.COOKIE_DOMAIN,
    path: '/'
  },
  rolling: true, // Extend session on activity
  unset: 'destroy' // Remove session from store on logout
};
```

### Session Fixation Prevention
```javascript
app.post('/login', (req, res) => {
  // Validate credentials
  if (isValidCredentials(req.body)) {
    // Regenerate session ID to prevent session fixation
    req.session.regenerate((err) => {
      if (err) {
        return res.status(500).send('Login failed');
      }
      
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

### Session Invalidation
```javascript
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).send('Logout failed');
    }
    
    // Clear session cookie
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

## 3. JWT Security

### Secure JWT Configuration
```javascript
const jwt = require('jsonwebtoken');

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
      jwtid: crypto.randomBytes(16).toString('hex') // Unique token ID
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
      return jwt.verify(token, this.secret, {
        algorithms: [this.algorithm],
        issuer: 'your-app',
        audience: audience
      });
    } catch (error) {
      throw new Error('Invalid token');
    }
  }
}
```

### JWT Token Blacklisting
```javascript
class TokenBlacklist {
  constructor() {
    this.blacklist = new Set();
  }
  
  blacklistToken(jti) {
    this.blacklist.add(jti);
  }
  
  isBlacklisted(jti) {
    return this.blacklist.has(jti);
  }
  
  // Clean up expired tokens periodically
  cleanup() {
    // Implementation depends on your storage mechanism
    // For Redis, you can set TTL on blacklisted tokens
  }
}

// Middleware to check blacklisted tokens
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

## 4. OAuth 2.0 Security

### Secure OAuth Implementation
```javascript
class OAuthSecurity {
  constructor() {
    this.stateStore = new Map();
    this.pkceStore = new Map();
  }
  
  generateState() {
    const state = crypto.randomBytes(32).toString('hex');
    this.stateStore.set(state, {
      timestamp: Date.now(),
      used: false
    });
    return state;
  }
  
  validateState(state) {
    const stateData = this.stateStore.get(state);
    if (!stateData || stateData.used) {
      return false;
    }
    
    // Check if state is not expired (5 minutes)
    if (Date.now() - stateData.timestamp > 5 * 60 * 1000) {
      this.stateStore.delete(state);
      return false;
    }
    
    stateData.used = true;
    return true;
  }
  
  generatePKCE() {
    const codeVerifier = crypto.randomBytes(32).toString('base64url');
    const codeChallenge = crypto
      .createHash('sha256')
      .update(codeVerifier)
      .digest('base64url');
    
    return { codeVerifier, codeChallenge };
  }
  
  storePKCE(codeChallenge, codeVerifier) {
    this.pkceStore.set(codeChallenge, {
      codeVerifier,
      timestamp: Date.now()
    });
  }
  
  validatePKCE(codeChallenge, codeVerifier) {
    const stored = this.pkceStore.get(codeChallenge);
    if (!stored || stored.codeVerifier !== codeVerifier) {
      return false;
    }
    
    // Check if PKCE is not expired (10 minutes)
    if (Date.now() - stored.timestamp > 10 * 60 * 1000) {
      this.pkceStore.delete(codeChallenge);
      return false;
    }
    
    this.pkceStore.delete(codeChallenge);
    return true;
  }
}
```

### Redirect URI Validation
```javascript
class RedirectURIValidator {
  constructor() {
    this.allowedRedirects = [
      'https://your-app.com/callback',
      'https://your-app.com/auth/callback',
      'http://localhost:3000/callback' // Development only
    ];
  }
  
  validateRedirectURI(redirectUri) {
    // Exact match validation
    if (!this.allowedRedirects.includes(redirectUri)) {
      return false;
    }
    
    // Additional validation for production
    if (process.env.NODE_ENV === 'production') {
      const url = new URL(redirectUri);
      if (url.protocol !== 'https:') {
        return false;
      }
    }
    
    return true;
  }
}
```

## 5. CSRF Protection

### CSRF Token Implementation
```javascript
const csrf = require('csurf');

// CSRF protection middleware
const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict'
  }
});

// Apply to routes that need CSRF protection
app.use('/api', csrfProtection);

// Error handler for CSRF
app.use((err, req, res, next) => {
  if (err.code === 'EBADCSRFTOKEN') {
    res.status(403).json({ error: 'CSRF token validation failed' });
  } else {
    next(err);
  }
});

// Route to get CSRF token for frontend
app.get('/api/csrf-token', (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});
```

### Double Submit Cookie Pattern
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
      httpOnly: false, // Must be accessible to JavaScript
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict'
    });
  }
  
  validateToken(cookieToken, headerToken) {
    if (!cookieToken || !headerToken) {
      return false;
    }
    
    return crypto.timingSafeEqual(
      Buffer.from(cookieToken, 'hex'),
      Buffer.from(headerToken, 'hex')
    );
  }
}
```

## 6. Input Validation and Sanitization

### Comprehensive Input Validation
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
  }),
  
  passwordReset: Joi.object({
    email: Joi.string().email().required(),
    token: Joi.string().length(64).required(),
    newPassword: Joi.string().min(12).pattern(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/
    ).required()
  })
};

// Validation middleware
const validate = (schema) => {
  return (req, res, next) => {
    const { error } = schema.validate(req.body);
    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(d => d.message)
      });
    }
    next();
  };
};
```

### SQL Injection Prevention
```javascript
const mysql = require('mysql2/promise');

class SecureDatabase {
  constructor() {
    this.pool = mysql.createPool({
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
      connectionLimit: 10
    });
  }
  
  async authenticateUser(email, password) {
    // Use parameterized queries to prevent SQL injection
    const [rows] = await this.pool.execute(
      'SELECT id, email, password_hash FROM users WHERE email = ?',
      [email]
    );
    
    if (rows.length === 0) {
      return null;
    }
    
    const user = rows[0];
    const isValid = await bcrypt.compare(password, user.password_hash);
    
    return isValid ? user : null;
  }
  
  async createUser(userData) {
    const { email, passwordHash, firstName, lastName } = userData;
    
    const [result] = await this.pool.execute(
      'INSERT INTO users (email, password_hash, first_name, last_name) VALUES (?, ?, ?, ?)',
      [email, passwordHash, firstName, lastName]
    );
    
    return result.insertId;
  }
}
```

## 7. Rate Limiting and DDoS Protection

### Advanced Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

// Different rate limits for different endpoints
const rateLimits = {
  login: rateLimit({
    store: new RedisStore({
      client: redis.createClient(),
      prefix: 'login_limit:'
    }),
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // 5 attempts
    message: 'Too many login attempts',
    standardHeaders: true,
    legacyHeaders: false
  }),
  
  register: rateLimit({
    store: new RedisStore({
      client: redis.createClient(),
      prefix: 'register_limit:'
    }),
    windowMs: 60 * 60 * 1000, // 1 hour
    max: 3, // 3 attempts
    message: 'Too many registration attempts'
  }),
  
  passwordReset: rateLimit({
    store: new RedisStore({
      client: redis.createClient(),
      prefix: 'reset_limit:'
    }),
    windowMs: 60 * 60 * 1000, // 1 hour
    max: 3, // 3 attempts
    message: 'Too many password reset attempts'
  }),
  
  api: rateLimit({
    store: new RedisStore({
      client: redis.createClient(),
      prefix: 'api_limit:'
    }),
    windowMs: 60 * 1000, // 1 minute
    max: 100, // 100 requests per minute
    message: 'Too many API requests'
  })
};
```

### IP-based Rate Limiting
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
    
    if (!this.limits.has(key)) {
      this.limits.set(key, []);
    }
    
    const requests = this.limits.get(key);
    
    // Remove old requests outside the window
    const validRequests = requests.filter(timestamp => timestamp > windowStart);
    
    if (validRequests.length >= limit) {
      return false;
    }
    
    validRequests.push(now);
    this.limits.set(key, validRequests);
    
    return true;
  }
  
  cleanup() {
    const now = Date.now();
    for (const [key, requests] of this.limits.entries()) {
      const windowMs = parseInt(key.split(':')[2]);
      const validRequests = requests.filter(timestamp => 
        timestamp > now - windowMs
      );
      
      if (validRequests.length === 0) {
        this.limits.delete(key);
      } else {
        this.limits.set(key, validRequests);
      }
    }
  }
}
```

## 8. Security Headers

### Security Headers Middleware
```javascript
const helmet = require('helmet');

// Comprehensive security headers
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
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
  noSniff: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  frameguard: { action: 'deny' },
  xssFilter: true
}));

// Additional custom headers
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
  next();
});
```

## 9. Audit Logging

### Comprehensive Audit Logging
```javascript
const winston = require('winston');

class AuditLogger {
  constructor() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.File({ filename: 'audit.log' }),
        new winston.transports.Console()
      ]
    });
  }
  
  logAuthEvent(event) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      event: event.type,
      userId: event.userId,
      ipAddress: event.ipAddress,
      userAgent: event.userAgent,
      success: event.success,
      details: event.details,
      sessionId: event.sessionId
    };
    
    this.logger.info('Authentication event', logEntry);
  }
  
  logSecurityEvent(event) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      event: event.type,
      severity: event.severity,
      ipAddress: event.ipAddress,
      userAgent: event.userAgent,
      details: event.details,
      action: event.action
    };
    
    this.logger.warn('Security event', logEntry);
  }
}

// Usage in authentication routes
app.post('/login', async (req, res) => {
  const auditLogger = new AuditLogger();
  
  try {
    const user = await authenticateUser(req.body.email, req.body.password);
    
    if (user) {
      auditLogger.logAuthEvent({
        type: 'LOGIN_SUCCESS',
        userId: user.id,
        ipAddress: req.ip,
        userAgent: req.headers['user-agent'],
        success: true,
        sessionId: req.sessionID
      });
      
      // Set up session
      req.session.userId = user.id;
      res.redirect('/dashboard');
    } else {
      auditLogger.logAuthEvent({
        type: 'LOGIN_FAILURE',
        ipAddress: req.ip,
        userAgent: req.headers['user-agent'],
        success: false,
        details: 'Invalid credentials'
      });
      
      res.status(401).send('Invalid credentials');
    }
  } catch (error) {
    auditLogger.logSecurityEvent({
      type: 'LOGIN_ERROR',
      severity: 'HIGH',
      ipAddress: req.ip,
      userAgent: req.headers['user-agent'],
      details: error.message,
      action: 'BLOCK_IP'
    });
    
    res.status(500).send('Login error');
  }
});
```

## 10. Secure Development Practices

### Environment Configuration
```javascript
// .env.example
NODE_ENV=development
PORT=3000
SESSION_SECRET=your-super-secret-session-key
JWT_SECRET=your-super-secret-jwt-key
DB_HOST=localhost
DB_USER=dbuser
DB_PASSWORD=dbpassword
DB_NAME=myapp
REDIS_URL=redis://localhost:6379
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

### Configuration Validation
```javascript
const Joi = require('joi');

const configSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
  PORT: Joi.number().port().default(3000),
  SESSION_SECRET: Joi.string().min(32).required(),
  JWT_SECRET: Joi.string().min(32).required(),
  DB_HOST: Joi.string().required(),
  DB_USER: Joi.string().required(),
  DB_PASSWORD: Joi.string().required(),
  DB_NAME: Joi.string().required(),
  REDIS_URL: Joi.string().uri().required(),
  GOOGLE_CLIENT_ID: Joi.string().required(),
  GOOGLE_CLIENT_SECRET: Joi.string().required()
});

function validateConfig() {
  const { error, value } = configSchema.validate(process.env);
  if (error) {
    throw new Error(`Configuration validation failed: ${error.message}`);
  }
  return value;
}

const config = validateConfig();
```

### Error Handling
```javascript
// Global error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  
  // Don't expose internal errors in production
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' });
  } else {
    res.status(500).json({ 
      error: err.message,
      stack: err.stack 
    });
  }
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({ error: 'Not found' });
});
```

## Conclusion

Implementing these security best practices will significantly improve the security posture of your authentication system. Remember to:

1. **Regularly update dependencies** to patch security vulnerabilities
2. **Conduct security audits** and penetration testing
3. **Monitor logs** for suspicious activity
4. **Implement defense in depth** with multiple security layers
5. **Stay informed** about new security threats and best practices
6. **Train your team** on security best practices
7. **Have an incident response plan** ready

Security is an ongoing process, not a one-time implementation. Continuously review and improve your security measures based on new threats and best practices. 