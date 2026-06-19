# AI Time-Management Agent - Security Architecture

## Document Information

**Version:** 1.0  
**Date:** 2026-06-19  
**Status:** Draft  
**Author:** Security Team

---

## 1. Overview

This document defines the comprehensive security architecture for the AI Time-Management Agent, covering authentication, authorization, data protection, network security, and compliance requirements.

---

## 2. Security Principles

### 2.1 Core Principles

1. **Defense in Depth**: Multiple layers of security controls
2. **Least Privilege**: Minimum necessary access rights
3. **Zero Trust**: Never trust, always verify
4. **Security by Design**: Security built into every component
5. **Data Privacy**: Protect user data at all times
6. **Compliance First**: Meet all regulatory requirements
7. **Continuous Monitoring**: Detect and respond to threats

### 2.2 Security Standards

- **OWASP Top 10**: Address all OWASP vulnerabilities
- **CIS Controls**: Implement CIS security benchmarks
- **ISO 27001**: Information security management
- **SOC 2 Type II**: Security, availability, confidentiality
- **GDPR**: Data protection and privacy
- **CCPA**: California consumer privacy

---

## 3. Authentication Architecture

### 3.1 Authentication Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Flow                       │
└─────────────────────────────────────────────────────────────┘

User → Login Request
         │
         ▼
    ┌─────────────────┐
    │  API Gateway    │
    │  - Rate Limit   │
    │  - CAPTCHA      │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │  User Service   │
    │  - Validate     │
    │    Credentials  │
    └────────┬────────┘
             │
             ├─> Password Hash Verification (bcrypt)
             │
             ├─> MFA Check (if enabled)
             │   └─> TOTP/SMS/Email verification
             │
             ▼
    ┌─────────────────┐
    │  Token Service  │
    │  - Generate JWT │
    │  - Create       │
    │    Session      │
    └────────┬────────┘
             │
             ▼
    Return: {
      accessToken: "JWT",
      refreshToken: "UUID",
      expiresIn: 900
    }
```

### 3.2 Password Security

**Password Requirements**:
```typescript
const passwordPolicy = {
  minLength: 12,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: true,
  preventCommonPasswords: true,
  preventUserInfo: true,  // No username, email in password
  maxAge: 90,  // days
  historyCount: 5  // Can't reuse last 5 passwords
}
```

**Password Hashing**:
```typescript
import bcrypt from 'bcrypt'

const SALT_ROUNDS = 12

async function hashPassword(password: string): Promise<string> {
  // Validate password meets policy
  validatePasswordPolicy(password)
  
  // Generate salt and hash
  const salt = await bcrypt.genSalt(SALT_ROUNDS)
  const hash = await bcrypt.hash(password, salt)
  
  return hash
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash)
}
```

### 3.3 Multi-Factor Authentication (MFA)

**MFA Methods**:
1. **TOTP (Time-based One-Time Password)**: Google Authenticator, Authy
2. **SMS**: Text message verification codes
3. **Email**: Email verification codes
4. **Backup Codes**: One-time use recovery codes

**TOTP Implementation**:
```typescript
import speakeasy from 'speakeasy'
import QRCode from 'qrcode'

async function setupTOTP(userId: string): Promise<{secret: string, qrCode: string}> {
  // Generate secret
  const secret = speakeasy.generateSecret({
    name: `TimeManagement (${user.email})`,
    issuer: 'TimeManagement'
  })
  
  // Generate QR code
  const qrCode = await QRCode.toDataURL(secret.otpauth_url)
  
  // Store secret (encrypted)
  await storeUserMFASecret(userId, encrypt(secret.base32))
  
  return {
    secret: secret.base32,
    qrCode
  }
}

function verifyTOTP(userId: string, token: string): boolean {
  const secret = decrypt(getUserMFASecret(userId))
  
  return speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token,
    window: 2  // Allow 2 time steps before/after
  })
}
```

### 3.4 JWT Token Structure

**Access Token**:
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-id-2026"
  },
  "payload": {
    "sub": "user-uuid",
    "email": "user@example.com",
    "role": "user",
    "permissions": ["read:tasks", "write:tasks"],
    "iat": 1718789400,
    "exp": 1718790300,
    "iss": "time-management-api",
    "aud": "time-management-app"
  },
  "signature": "..."
}
```

**Token Generation**:
```typescript
import jwt from 'jsonwebtoken'
import fs from 'fs'

const privateKey = fs.readFileSync('private-key.pem')
const publicKey = fs.readFileSync('public-key.pem')

function generateAccessToken(user: User): string {
  const payload = {
    sub: user.id,
    email: user.email,
    role: user.role,
    permissions: user.permissions
  }
  
  return jwt.sign(payload, privateKey, {
    algorithm: 'RS256',
    expiresIn: '15m',
    issuer: 'time-management-api',
    audience: 'time-management-app',
    keyid: 'key-id-2026'
  })
}

function verifyAccessToken(token: string): TokenPayload {
  try {
    return jwt.verify(token, publicKey, {
      algorithms: ['RS256'],
      issuer: 'time-management-api',
      audience: 'time-management-app'
    })
  } catch (error) {
    throw new UnauthorizedError('Invalid token')
  }
}
```

**Refresh Token**:
```typescript
interface RefreshToken {
  id: string
  userId: string
  token: string  // Hashed
  expiresAt: Date
  createdAt: Date
  lastUsedAt: Date
  deviceInfo: string
  ipAddress: string
}

async function generateRefreshToken(userId: string, deviceInfo: string): Promise<string> {
  const token = crypto.randomBytes(32).toString('hex')
  const hashedToken = await bcrypt.hash(token, 10)
  
  await db.refreshTokens.create({
    userId,
    token: hashedToken,
    expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),  // 30 days
    deviceInfo,
    ipAddress: req.ip
  })
  
  return token
}
```

### 3.5 Session Management

**Session Storage**:
```typescript
interface Session {
  id: string
  userId: string
  accessToken: string
  refreshToken: string
  expiresAt: Date
  deviceInfo: {
    userAgent: string
    ip: string
    location: string
  }
  lastActivity: Date
}

// Store in Redis for fast access
const sessionTTL = 15 * 60  // 15 minutes

await redis.setex(
  `session:${sessionId}`,
  sessionTTL,
  JSON.stringify(session)
)
```

**Session Validation**:
```typescript
async function validateSession(sessionId: string): Promise<Session> {
  const session = await redis.get(`session:${sessionId}`)
  
  if (!session) {
    throw new UnauthorizedError('Session expired')
  }
  
  const sessionData = JSON.parse(session)
  
  // Check if session is still valid
  if (new Date() > sessionData.expiresAt) {
    await redis.del(`session:${sessionId}`)
    throw new UnauthorizedError('Session expired')
  }
  
  // Update last activity
  sessionData.lastActivity = new Date()
  await redis.setex(`session:${sessionId}`, sessionTTL, JSON.stringify(sessionData))
  
  return sessionData
}
```

---

## 4. Authorization Architecture

### 4.1 Role-Based Access Control (RBAC)

**Role Hierarchy**:
```
┌─────────────────────────────────────────────────────────────┐
│                      Role Hierarchy                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐                                           │
│  │    Admin     │  (Full system access)                     │
│  └──────┬───────┘                                           │
│         │                                                    │
│         ├─────────────┬─────────────┐                       │
│         │             │             │                       │
│  ┌──────▼───────┐ ┌──▼──────────┐ ┌▼─────────────┐        │
│  │  Team Lead   │ │  Manager    │ │  Support     │        │
│  └──────┬───────┘ └──┬──────────┘ └┬─────────────┘        │
│         │            │              │                       │
│         └────────────┴──────────────┘                       │
│                      │                                      │
│               ┌──────▼───────┐                              │
│               │     User     │  (Standard access)           │
│               └──────────────┘                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Permission Model**:
```typescript
enum Permission {
  // Task permissions
  READ_OWN_TASKS = 'read:own:tasks',
  WRITE_OWN_TASKS = 'write:own:tasks',
  DELETE_OWN_TASKS = 'delete:own:tasks',
  READ_ALL_TASKS = 'read:all:tasks',
  WRITE_ALL_TASKS = 'write:all:tasks',
  
  // Calendar permissions
  READ_OWN_CALENDAR = 'read:own:calendar',
  WRITE_OWN_CALENDAR = 'write:own:calendar',
  READ_SHARED_CALENDAR = 'read:shared:calendar',
  
  // User permissions
  READ_OWN_PROFILE = 'read:own:profile',
  WRITE_OWN_PROFILE = 'write:own:profile',
  READ_ALL_USERS = 'read:all:users',
  MANAGE_USERS = 'manage:users',
  
  // Admin permissions
  MANAGE_SYSTEM = 'manage:system',
  VIEW_ANALYTICS = 'view:analytics'
}

const rolePermissions = {
  user: [
    Permission.READ_OWN_TASKS,
    Permission.WRITE_OWN_TASKS,
    Permission.DELETE_OWN_TASKS,
    Permission.READ_OWN_CALENDAR,
    Permission.WRITE_OWN_CALENDAR,
    Permission.READ_OWN_PROFILE,
    Permission.WRITE_OWN_PROFILE
  ],
  
  team_lead: [
    ...rolePermissions.user,
    Permission.READ_ALL_TASKS,
    Permission.READ_SHARED_CALENDAR,
    Permission.VIEW_ANALYTICS
  ],
  
  admin: [
    ...rolePermissions.team_lead,
    Permission.WRITE_ALL_TASKS,
    Permission.READ_ALL_USERS,
    Permission.MANAGE_USERS,
    Permission.MANAGE_SYSTEM
  ]
}
```

**Authorization Middleware**:
```typescript
function requirePermission(...permissions: Permission[]) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const user = req.user
    
    if (!user) {
      return res.status(401).json({ error: 'Unauthorized' })
    }
    
    const hasPermission = permissions.every(permission =>
      user.permissions.includes(permission)
    )
    
    if (!hasPermission) {
      return res.status(403).json({ error: 'Forbidden' })
    }
    
    next()
  }
}

// Usage
app.get('/tasks', 
  authenticate,
  requirePermission(Permission.READ_OWN_TASKS),
  getTasksHandler
)

app.delete('/users/:id',
  authenticate,
  requirePermission(Permission.MANAGE_USERS),
  deleteUserHandler
)
```

### 4.2 Resource-Level Authorization

**Ownership Verification**:
```typescript
async function verifyTaskOwnership(userId: string, taskId: string): Promise<boolean> {
  const task = await db.tasks.findById(taskId)
  
  if (!task) {
    throw new NotFoundError('Task not found')
  }
  
  if (task.userId !== userId) {
    throw new ForbiddenError('Not authorized to access this task')
  }
  
  return true
}

// Middleware
async function requireTaskOwnership(req: Request, res: Response, next: NextFunction) {
  try {
    await verifyTaskOwnership(req.user.id, req.params.taskId)
    next()
  } catch (error) {
    next(error)
  }
}
```

---

## 5. Data Protection

### 5.1 Encryption at Rest

**Database Encryption**:
```yaml
# RDS encryption configuration
database:
  encryption:
    enabled: true
    kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/..."
    algorithm: AES-256
```

**Application-Level Encryption**:
```typescript
import crypto from 'crypto'

const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY  // 32 bytes
const ALGORITHM = 'aes-256-gcm'

function encrypt(text: string): string {
  const iv = crypto.randomBytes(16)
  const cipher = crypto.createCipheriv(ALGORITHM, Buffer.from(ENCRYPTION_KEY, 'hex'), iv)
  
  let encrypted = cipher.update(text, 'utf8', 'hex')
  encrypted += cipher.final('hex')
  
  const authTag = cipher.getAuthTag()
  
  // Return: iv:authTag:encrypted
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`
}

function decrypt(encryptedText: string): string {
  const [ivHex, authTagHex, encrypted] = encryptedText.split(':')
  
  const iv = Buffer.from(ivHex, 'hex')
  const authTag = Buffer.from(authTagHex, 'hex')
  
  const decipher = crypto.createDecipheriv(ALGORITHM, Buffer.from(ENCRYPTION_KEY, 'hex'), iv)
  decipher.setAuthTag(authTag)
  
  let decrypted = decipher.update(encrypted, 'hex', 'utf8')
  decrypted += decipher.final('utf8')
  
  return decrypted
}

// Usage: Encrypt sensitive fields
interface User {
  id: string
  email: string
  mfaSecret?: string  // Encrypted
  oauthTokens?: string  // Encrypted
}

async function saveUser(user: User): Promise<void> {
  if (user.mfaSecret) {
    user.mfaSecret = encrypt(user.mfaSecret)
  }
  if (user.oauthTokens) {
    user.oauthTokens = encrypt(user.oauthTokens)
  }
  
  await db.users.save(user)
}
```

### 5.2 Encryption in Transit

**TLS Configuration**:
```nginx
# NGINX TLS configuration
server {
  listen 443 ssl http2;
  server_name api.time-management.com;
  
  # SSL certificates
  ssl_certificate /etc/ssl/certs/cert.pem;
  ssl_certificate_key /etc/ssl/private/key.pem;
  
  # TLS version
  ssl_protocols TLSv1.3 TLSv1.2;
  
  # Cipher suites
  ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
  ssl_prefer_server_ciphers on;
  
  # HSTS
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
  
  # OCSP stapling
  ssl_stapling on;
  ssl_stapling_verify on;
  
  # Session cache
  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 10m;
}
```

**Certificate Management**:
```yaml
# Cert-manager for automatic certificate renewal
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-tls
  namespace: production
spec:
  secretName: api-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.time-management.com
  renewBefore: 720h  # 30 days
```

### 5.3 Data Masking and Anonymization

**PII Masking**:
```typescript
function maskEmail(email: string): string {
  const [local, domain] = email.split('@')
  const maskedLocal = local.charAt(0) + '***' + local.charAt(local.length - 1)
  return `${maskedLocal}@${domain}`
}

function maskPhone(phone: string): string {
  return phone.replace(/\d(?=\d{4})/g, '*')
}

// Usage in logs
logger.info('User logged in', {
  userId: user.id,
  email: maskEmail(user.email),
  ip: maskIP(req.ip)
})
```

**Data Anonymization for Analytics**:
```typescript
function anonymizeUserData(user: User): AnonymizedUser {
  return {
    id: hashUserId(user.id),  // One-way hash
    ageGroup: getAgeGroup(user.birthDate),  // 18-25, 26-35, etc.
    country: user.country,
    // Remove all PII
  }
}
```

---

## 6. API Security

### 6.1 Rate Limiting

**Rate Limit Configuration**:
```typescript
import rateLimit from 'express-rate-limit'

// General API rate limit
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,  // 100 requests per window
  message: 'Too many requests, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    res.status(429).json({
      error: 'Rate limit exceeded',
      retryAfter: req.rateLimit.resetTime
    })
  }
})

// Stricter limit for authentication endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,  // 5 attempts per 15 minutes
  skipSuccessfulRequests: true
})

// Per-user rate limiting
const userLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: async (req) => {
    const user = req.user
    return user.role === 'premium' ? 1000 : 100
  },
  keyGenerator: (req) => req.user.id
})

app.use('/api/', apiLimiter)
app.use('/auth/login', authLimiter)
app.use('/api/tasks', authenticate, userLimiter)
```

### 6.2 Input Validation

**Request Validation**:
```typescript
import { z } from 'zod'

const createTaskSchema = z.object({
  title: z.string().min(1).max(200),
  description: z.string().max(5000).optional(),
  dueDate: z.string().datetime().optional(),
  priority: z.enum(['low', 'medium', 'high']),
  tags: z.array(z.string()).max(10).optional()
})

function validateRequest(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body)
      next()
    } catch (error) {
      res.status(400).json({
        error: 'Validation failed',
        details: error.errors
      })
    }
  }
}

app.post('/tasks',
  authenticate,
  validateRequest(createTaskSchema),
  createTaskHandler
)
```

**SQL Injection Prevention**:
```typescript
// Use parameterized queries
const tasks = await db.query(
  'SELECT * FROM tasks WHERE user_id = ? AND status = ?',
  [userId, status]
)

// Never concatenate user input
// BAD: `SELECT * FROM tasks WHERE user_id = '${userId}'`
```

**XSS Prevention**:
```typescript
import DOMPurify from 'isomorphic-dompurify'

function sanitizeHTML(html: string): string {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href']
  })
}

// Sanitize user input before storing
task.description = sanitizeHTML(task.description)
```

### 6.3 CORS Configuration

```typescript
import cors from 'cors'

const corsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://app.time-management.com',
      'https://staging.time-management.com'
    ]
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true)
    } else {
      callback(new Error('Not allowed by CORS'))
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Total-Count', 'X-Page-Count'],
  maxAge: 86400  // 24 hours
}

app.use(cors(corsOptions))
```

### 6.4 Security Headers

```typescript
import helmet from 'helmet'

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.time-management.com'],
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
  xssFilter: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
}))

// Additional security headers
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff')
  res.setHeader('X-Frame-Options', 'DENY')
  res.setHeader('X-XSS-Protection', '1; mode=block')
  res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()')
  next()
})
```

---

## 7. Network Security

### 7.1 Network Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Internet                                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   WAF (Web Application Firewall)             │
│  - DDoS protection                                           │
│  - SQL injection prevention                                  │
│  - XSS prevention                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Load Balancer (Public)                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      VPC                                     │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Public Subnet                                     │    │
│  │  - NAT Gateway                                     │    │
│  │  - Bastion Host                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Private Subnet (Application)                      │    │
│  │  - Kubernetes Cluster                              │    │
│  │  - Application Pods                                │    │
│  │  - No direct internet access                       │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Private Subnet (Data)                             │    │
│  │  - RDS Database                                    │    │
│  │  - ElastiCache                                     │    │
│  │  - No internet access                              │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Security Groups

**Application Security Group**:
```yaml
securityGroup:
  name: app-sg
  inbound:
    - port: 443
      protocol: tcp
      source: load-balancer-sg
    - port: 8080
      protocol: tcp
      source: load-balancer-sg
  outbound:
    - port: 5432
      protocol: tcp
      destination: database-sg
    - port: 6379
      protocol: tcp
      destination: cache-sg
    - port: 443
      protocol: tcp
      destination: 0.0.0.0/0  # For external API calls
```

**Database Security Group**:
```yaml
securityGroup:
  name: database-sg
  inbound:
    - port: 5432
      protocol: tcp
      source: app-sg
  outbound:
    - port: 443
      protocol: tcp
      destination: 0.0.0.0/0  # For backups to S3
```

### 7.3 VPN and Bastion Access

**Bastion Host Configuration**:
```yaml
bastionHost:
  instanceType: t3.micro
  publicIP: true
  securityGroup:
    inbound:
      - port: 22
        protocol: tcp
        source: office-ip-range
    outbound:
      - port: 22
        protocol: tcp
        destination: private-subnet
  
  sshConfig:
    permitRootLogin: no
    passwordAuthentication: no
    pubkeyAuthentication: yes
    allowUsers: [admin, devops]
```

---

## 8. Secrets Management

### 8.1 AWS Secrets Manager

```typescript
import { SecretsManager } from '@aws-sdk/client-secrets-manager'

const secretsManager = new SecretsManager({ region: 'us-east-1' })

async function getSecret(secretName: string): Promise<string> {
  try {
    const response = await secretsManager.getSecretValue({
      SecretId: secretName
    })
    
    return response.SecretString
  } catch (error) {
    logger.error('Failed to retrieve secret', { secretName, error })
    throw error
  }
}

// Usage
const dbPassword = await getSecret('prod/database/password')
const apiKey = await getSecret('prod/external-api/key')
```

### 8.2 Secret Rotation

```yaml
secretRotation:
  enabled: true
  schedule: "rate(30 days)"
  lambda: rotate-secrets-function
  
  secrets:
    - name: database-password
      rotationDays: 30
    - name: api-keys
      rotationDays: 90
    - name: encryption-keys
      rotationDays: 180
```

---

## 9. Audit Logging

### 9.1 Audit Log Structure

```typescript
interface AuditLog {
  id: string
  timestamp: Date
  userId: string
  action: string
  resource: string
  resourceId: string
  changes: {
    before: any
    after: any
  }
  ipAddress: string
  userAgent: string
  result: 'success' | 'failure'
  errorMessage?: string
}

async function logAudit(log: AuditLog): Promise<void> {
  await db.auditLogs.create(log)
  
  // Also send to centralized logging
  logger.audit(log)
}
```

### 9.2 Auditable Events

```typescript
// Log all sensitive operations
const auditableActions = [
  'user.login',
  'user.logout',
  'user.password_change',
  'user.mfa_enable',
  'user.mfa_disable',
  'user.delete',
  'task.create',
  'task.update',
  'task.delete',
  'calendar.share',
  'integration.connect',
  'integration.disconnect',
  'admin.user_impersonate',
  'admin.permission_change'
]

// Middleware to log auditable actions
function auditMiddleware(action: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const startTime = Date.now()
    
    res.on('finish', async () => {
      if (auditableActions.includes(action)) {
        await logAudit({
          id: generateId(),
          timestamp: new Date(),
          userId: req.user?.id,
          action,
          resource: req.path,
          resourceId: req.params.id,
          changes: {
            before: req.body._before,
            after: req.body
          },
          ipAddress: req.ip,
          userAgent: req.headers['user-agent'],
          result: res.statusCode < 400 ? 'success' : 'failure',
          duration: Date.now() - startTime
        })
      }
    })
    
    next()
  }
}
```

---

## 10. Compliance

### 10.1 GDPR Compliance

**Data Subject Rights**:
```typescript
// Right to Access
async function exportUserData(userId: string): Promise<UserDataExport> {
  return {
    profile: await getUserProfile(userId),
    tasks: await getUserTasks(userId),
    events: await getUserEvents(userId),
    timeEntries: await getUserTimeEntries(userId),
    preferences: await getUserPreferences(userId)
  }
}

// Right to Erasure
async function deleteUserData(userId: string): Promise<void> {
  await db.transaction(async (trx) => {
    await trx.tasks.deleteMany({ userId })
    await trx.events.deleteMany({ userId })
    await trx.timeEntries.deleteMany({ userId })
    await trx.preferences.delete({ userId })
    await trx.users.delete({ id: userId })
  })
  
  // Anonymize audit logs
  await anonymizeAuditLogs(userId)
}

// Right to Portability
async function exportUserDataPortable(userId: string): Promise<string> {
  const data = await exportUserData(userId)
  return JSON.stringify(data, null, 2)
}
```

### 10.2 Data Retention

```typescript
const retentionPolicies = {
  auditLogs: 7 * 365,  // 7 years
  userActivity: 2 * 365,  // 2 years
  deletedUsers: 30,  // 30 days (soft delete)
  sessions: 30,  // 30 days
  backups: 90  // 90 days
}

// Automated cleanup job
async function cleanupExpiredData(): Promise<void> {
  const now = new Date()
  
  // Delete old audit logs
  await db.auditLogs.deleteMany({
    timestamp: { $lt: subDays(now, retentionPolicies.auditLogs) }
  })
  
  // Delete old activity logs
  await db.activityLogs.deleteMany({
    timestamp: { $lt: subDays(now, retentionPolicies.userActivity) }
  })
  
  // Permanently delete soft-deleted users
  await db.users.deleteMany({
    deletedAt: { $lt: subDays(now, retentionPolicies.deletedUsers) }
  })
}
```

---

## 11. Security Monitoring

### 11.1 Security Metrics

```typescript
const securityMetrics = {
  // Authentication
  failedLoginAttempts: 0,
  successfulLogins: 0,
  mfaBypassAttempts: 0,
  
  // Authorization
  unauthorizedAccessAttempts: 0,
  privilegeEscalationAttempts: 0,
  
  // API Security
  rateLimitExceeded: 0,
  invalidTokens: 0,
  suspiciousRequests: 0,
  
  // Data Security
  encryptionFailures: 0,
  dataLeakAttempts: 0
}
```

### 11.2 Security Alerts

```typescript
// Alert on suspicious activity
async function detectSuspiciousActivity(event: SecurityEvent): Promise<void> {
  const alerts = []
  
  // Multiple failed login attempts
  if (event.type === 'login_failed') {
    const recentFailures = await getRecentFailedLogins(event.userId, 15 * 60)
    if (recentFailures >= 5) {
      alerts.push({
        severity: 'high',
        message: 'Multiple failed login attempts',
        userId: event.userId,
        count: recentFailures
      })
    }
  }
  
  // Access from unusual location
  if (event.type === 'login_success') {
    const isUnusualLocation = await checkUnusualLocation(event.userId, event.ipAddress)
    if (isUnusualLocation) {
      alerts.push({
        severity: 'medium',
        message: 'Login from unusual location',
        userId: event.userId,
        location: event.location
      })
    }
  }
  
  // Send alerts
  for (const alert of alerts) {
    await sendSecurityAlert(alert)
  }
}
```

---

## 12. Incident Response

### 12.1 Incident Response Plan

**Phases**:
1. **Detection**: Identify security incident
2. **Containment**: Limit damage and prevent spread
3. **Eradication**: Remove threat from environment
4. **Recovery**: Restore systems to normal operation
5. **Post-Incident**: Review and improve

**Response Team**:
- Security Lead
- DevOps Engineer
- Backend Engineer
- Legal/Compliance Officer

### 12.2 Incident Playbooks

**Data Breach Response**:
```
1. Immediately isolate affected systems
2. Assess scope of breach
3. Notify security team and management
4. Preserve evidence for forensics
5. Notify affected users (within 72 hours for GDPR)
6. Notify regulatory authorities if required
7. Conduct post-mortem analysis
8. Implement preventive measures
```

---

## 13. Security Checklist

### 13.1 Pre-Deployment Security Checklist

- [ ] All secrets stored in Secrets Manager
- [ ] TLS 1.3 enabled for all endpoints
- [ ] Rate limiting configured
- [ ] Input validation implemented
- [ ] SQL injection prevention verified
- [ ] XSS prevention verified
- [ ] CSRF protection enabled
- [ ] Security headers configured
- [ ] Authentication implemented
- [ ] Authorization implemented
- [ ] Audit logging enabled
- [ ] Encryption at rest enabled
- [ ] Encryption in transit enabled
- [ ] Security groups configured
- [ ] WAF rules configured
- [ ] Backup strategy implemented
- [ ] Incident response plan documented
- [ ] Security monitoring enabled
- [ ] Vulnerability scanning scheduled
- [ ] Penetration testing completed

---

**Document Status:** Draft  
**Next Review Date:** TBD  
**Approval Required From:** Security Team, Compliance Officer

---

*End of Security Architecture Document*