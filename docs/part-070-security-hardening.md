# Part 70: Security Hardening สำหรับ Nuxt.js

## OWASP Top 10 กับ Vue/Nuxt

OWASP Top 10 คือรายการช่องโหว่ความปลอดภัยที่พบบ่อยที่สุดในเว็บแอปพลิเคชัน

### OWASP Top 10 (2021)
1. **A01: Broken Access Control**
2. **A02: Cryptographic Failures**
3. **A03: Injection (SQL, XSS)**
4. **A04: Insecure Design**
5. **A05: Security Misconfiguration**
6. **A06: Vulnerable Components**
7. **A07: Identification & Authentication Failures**
8. **A08: Software & Data Integrity Failures**
9. **A09: Security Logging Failures**
10. **A10: Server-Side Request Forgery**

## SQL Injection Prevention

```typescript
// WRONG: Vulnerable to SQL injection
const getUser = async (username: string) => {
  // Never do this!
  return prisma.$queryRaw`SELECT * FROM users WHERE username = '${username}'`
}

// CORRECT: Use Prisma ORM (automatically parameterized)
const getUser = async (username: string) => {
  return prisma.user.findFirst({
    where: { username }
  })
}

// CORRECT: If you need raw SQL, use Prisma's tagged template
const getUserSafe = async (username: string) => {
  return prisma.$queryRaw`SELECT id, name FROM users WHERE username = ${username}`
  // Prisma automatically uses parameterized queries
}
```

```typescript
// Input validation middleware
import { z } from 'zod'

export const createUserSchema = z.object({
  name: z.string()
    .min(2, 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร')
    .max(100, 'ชื่อต้องไม่เกิน 100 ตัวอักษร')
    .regex(/^[฀-๿A-Za-z\s]+$/, 'ชื่อต้องประกอบด้วยตัวอักษรเท่านั้น'),
  
  email: z.string()
    .email('รูปแบบอีเมลไม่ถูกต้อง')
    .toLowerCase()
    .max(255),
  
  password: z.string()
    .min(8, 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
    .max(100)
    .regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, 
      'รหัสผ่านต้องมีตัวพิมพ์เล็ก ตัวพิมพ์ใหญ่ และตัวเลข'),
  
  role: z.enum(['user', 'admin']).default('user')
})

export const validateBody = <T>(schema: z.ZodSchema<T>) => {
  return async (event: H3Event) => {
    const body = await readBody(event)
    const result = schema.safeParse(body)
    
    if (!result.success) {
      throw createError({
        statusCode: 422,
        message: 'Validation failed',
        data: result.error.flatten()
      })
    }
    
    return result.data
  }
}
```

## XSS Prevention

```typescript
// server/utils/sanitize.ts
import DOMPurify from 'isomorphic-dompurify'
import { JSDOM } from 'jsdom'

const window = new JSDOM('').window
const purify = DOMPurify(window)

export function sanitizeHTML(dirty: string): string {
  return purify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li', 
                   'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'blockquote', 'pre', 'code'],
    ALLOWED_ATTR: ['href', 'target', 'rel', 'class'],
    ALLOW_DATA_ATTR: false,
    ADD_ATTR: ['rel'],
    FORCE_BODY: false,
    // Force all links to be external with noopener
    TRANSFORM_TAGS: {
      'a': (tagName, attribs) => ({
        tagName,
        attribs: {
          ...attribs,
          target: '_blank',
          rel: 'noopener noreferrer'
        }
      })
    }
  })
}

export function escapeHTML(text: string): string {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;')
}
```

```vue
<!-- Vue XSS prevention -->
<template>
  <!-- SAFE: Vue escapes HTML by default -->
  <p>{{ userInput }}</p>
  
  <!-- SAFE: Use v-html only with sanitized content -->
  <div v-html="sanitizedContent"></div>
  
  <!-- DANGEROUS: Never use raw user input in v-html -->
  <!-- <div v-html="userInput"></div> -->
</template>

<script setup lang="ts">
import { sanitizeHTML } from '@/utils/sanitize'

const props = defineProps<{
  userContent: string
}>()

const sanitizedContent = computed(() => sanitizeHTML(props.userContent))
</script>
```

## CSRF Protection

```typescript
// server/middleware/csrf.ts
import { createHash, randomBytes } from 'crypto'

const CSRF_COOKIE = '__csrf'
const CSRF_HEADER = 'x-csrf-token'

function generateToken(): string {
  return randomBytes(32).toString('hex')
}

function hashToken(token: string, secret: string): string {
  return createHash('sha256').update(`${token}${secret}`).digest('hex')
}

export default defineEventHandler(async (event) => {
  const method = getMethod(event)
  
  // Only protect state-changing methods
  if (!['POST', 'PUT', 'PATCH', 'DELETE'].includes(method)) {
    // Generate token for GET requests
    if (method === 'GET') {
      const token = generateToken()
      setCookie(event, CSRF_COOKIE, token, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
        path: '/'
      })
      setResponseHeader(event, 'X-CSRF-Token', hashToken(token, process.env.JWT_SECRET!))
    }
    return
  }
  
  // Skip for API-only endpoints (using JWT bearer auth)
  const authorization = getHeader(event, 'authorization')
  if (authorization?.startsWith('Bearer ')) return
  
  // Validate CSRF token
  const cookie = getCookie(event, CSRF_COOKIE)
  const header = getHeader(event, CSRF_HEADER)
  
  if (!cookie || !header) {
    throw createError({ statusCode: 403, message: 'CSRF token missing' })
  }
  
  const expectedHash = hashToken(cookie, process.env.JWT_SECRET!)
  if (header !== expectedHash) {
    throw createError({ statusCode: 403, message: 'CSRF token invalid' })
  }
})
```

```typescript
// composables/useCSRF.ts
export function useCSRF() {
  const token = ref<string | null>(null)
  
  const getToken = async () => {
    // Token is set via cookie/header on first page load
    const meta = document.querySelector('meta[name="csrf-token"]')
    token.value = meta?.getAttribute('content') || null
    return token.value
  }
  
  const fetchWithCSRF = async (url: string, options: RequestInit = {}) => {
    const csrfToken = token.value || await getToken()
    
    return fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        'X-CSRF-Token': csrfToken || '',
        'Content-Type': 'application/json'
      }
    })
  }
  
  return { getToken, fetchWithCSRF }
}
```

## Authentication Security

```typescript
// server/utils/auth.ts
import bcrypt from 'bcryptjs'
import { SignJWT, jwtVerify } from 'jose'

const BCRYPT_ROUNDS = 12

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, BCRYPT_ROUNDS)
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash)
}

const jwtSecret = new TextEncoder().encode(process.env.JWT_SECRET)

export async function signJWT(payload: Record<string, any>, expiresIn = '7d'): Promise<string> {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime(expiresIn)
    .setIssuer(process.env.APP_URL!)
    .setAudience(process.env.APP_URL!)
    .sign(jwtSecret)
}

export async function verifyJWT<T>(token: string): Promise<T> {
  const { payload } = await jwtVerify(token, jwtSecret, {
    issuer: process.env.APP_URL,
    audience: process.env.APP_URL
  })
  return payload as T
}
```

```typescript
// server/middleware/auth.ts
export async function requireAuth(event: H3Event) {
  const authorization = getHeader(event, 'authorization')
  
  if (!authorization?.startsWith('Bearer ')) {
    throw createError({ statusCode: 401, message: 'Authentication required' })
  }
  
  const token = authorization.slice(7)
  
  try {
    const payload = await verifyJWT<{ userId: string; role: string }>(token)
    
    // Check if user still exists and is active
    const user = await prisma.user.findUnique({
      where: { id: payload.userId },
      select: { id: true, email: true, role: true, isActive: true }
    })
    
    if (!user || !user.isActive) {
      throw createError({ statusCode: 401, message: 'Account not found or inactive' })
    }
    
    event.context.user = user
    return user
  } catch (err: any) {
    if (err.code === 'ERR_JWT_EXPIRED') {
      throw createError({ statusCode: 401, message: 'Token expired' })
    }
    throw createError({ statusCode: 401, message: 'Invalid token' })
  }
}

export async function requireAdmin(event: H3Event) {
  const user = await requireAuth(event)
  
  if (user.role !== 'admin') {
    throw createError({ statusCode: 403, message: 'Admin access required' })
  }
  
  return user
}
```

## API Security

```typescript
// server/middleware/rate-limit.ts
import { getRedis } from '~/server/utils/redis'

interface RateLimitOptions {
  windowMs: number
  max: number
  keyPrefix?: string
}

export function createRateLimit(options: RateLimitOptions) {
  const { windowMs, max, keyPrefix = 'rl' } = options
  
  return defineEventHandler(async (event) => {
    const ip = getRequestIP(event, { xForwardedFor: true }) || 'unknown'
    const key = `${keyPrefix}:${ip}`
    const redis = getRedis()
    
    const requests = await redis.incr(key)
    
    if (requests === 1) {
      await redis.expire(key, Math.ceil(windowMs / 1000))
    }
    
    const remaining = Math.max(0, max - requests)
    const reset = await redis.ttl(key)
    
    setResponseHeader(event, 'X-RateLimit-Limit', String(max))
    setResponseHeader(event, 'X-RateLimit-Remaining', String(remaining))
    setResponseHeader(event, 'X-RateLimit-Reset', String(Date.now() + reset * 1000))
    
    if (requests > max) {
      setResponseHeader(event, 'Retry-After', String(reset))
      throw createError({
        statusCode: 429,
        message: 'Too many requests, please try again later'
      })
    }
  })
}

// Usage
export const apiRateLimit = createRateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100
})

export const authRateLimit = createRateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 10, // Only 10 login attempts per hour
  keyPrefix: 'auth'
})
```

## Dependency Security (Snyk)

```bash
# ติดตั้ง Snyk
npm install -g snyk

# Login
snyk auth

# Check for vulnerabilities
snyk test

# Monitor project
snyk monitor

# Fix vulnerabilities automatically (when possible)
snyk fix
```

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1' # Weekly on Monday

jobs:
  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
  
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
```

## Security Audit

```typescript
// scripts/security-check.ts
import { execSync } from 'child_process'

async function runSecurityAudit() {
  console.log('Running security audit...\n')
  
  // npm audit
  try {
    console.log('1. Checking npm vulnerabilities...')
    execSync('npm audit --audit-level=moderate', { stdio: 'inherit' })
    console.log('✓ No moderate+ vulnerabilities\n')
  } catch {
    console.error('⚠️ Vulnerabilities found. Run npm audit fix\n')
  }
  
  // Check for hardcoded secrets
  console.log('2. Checking for hardcoded secrets...')
  try {
    execSync('npx trufflesecurity/trufflehog filesystem . --no-update', { stdio: 'inherit' })
    console.log('✓ No secrets detected\n')
  } catch {
    console.error('⚠️ Potential secrets found!\n')
  }
  
  // Check security headers
  console.log('3. Security headers check configured in nuxt-security module')
  
  console.log('Security audit complete!')
}

runSecurityAudit()
```

## ตัวอย่าง: Secure Nuxt Application

```typescript
// nuxt.config.ts (secure configuration)
import { defineNuxtConfig } from 'nuxt/config'

export default defineNuxtConfig({
  modules: ['nuxt-security'],
  
  security: {
    strict: false,
    
    headers: {
      crossOriginResourcePolicy: 'same-origin',
      crossOriginOpenerPolicy: 'same-origin',
      crossOriginEmbedderPolicy: 'require-corp',
      contentSecurityPolicy: {
        'base-uri': ["'none'"],
        'default-src': ["'none'"],
        'connect-src': ["'self'", 'https://api.stripe.com'],
        'font-src': ["'self'", 'https://fonts.gstatic.com'],
        'form-action': ["'self'"],
        'frame-ancestors': ["'none'"],
        'img-src': ["'self'", 'data:', 'https:'],
        'manifest-src': ["'self'"],
        'media-src': ["'self'"],
        'object-src': ["'none'"],
        'script-src-attr': ["'none'"],
        'style-src': ["'self'", "'unsafe-inline'"],
        'upgrade-insecure-requests': true
      },
      originAgentCluster: '?1',
      referrerPolicy: 'no-referrer',
      strictTransportSecurity: {
        maxAge: 31536000,
        includeSubdomains: true,
        preload: true
      },
      xContentTypeOptions: 'nosniff',
      xDNSPrefetchControl: 'off',
      xDownloadOptions: 'noopen',
      xFrameOptions: 'DENY',
      xPermittedCrossDomainPolicies: 'none',
      xXSSProtection: '1; mode=block'
    },
    
    rateLimiter: {
      tokensPerInterval: 150,
      interval: 'hour',
      headers: true
    },
    
    requestSizeLimiter: {
      maxRequestSizeInBytes: 2000000,  // 2MB
      maxUploadFileRequestInBytes: 8000000 // 8MB
    },
    
    xssValidator: {
      throwError: true
    },
    
    corsHandler: {
      origin: process.env.APP_URL,
      methods: ['GET', 'POST', 'PUT', 'DELETE'],
      allowHeaders: ['Content-Type', 'Authorization'],
      exposeHeaders: ['X-Request-ID'],
      credentials: true,
      maxAge: '3600'
    },
    
    allowedMethodsRestricter: {
      methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS']
    },
    
    hidePoweredBy: true,
    basicAuth: false,
    enabled: true,
    csrf: true
  }
})
```

## File Upload Security

```typescript
// server/api/upload.post.ts
import { createHash } from 'crypto'
import path from 'path'

const ALLOWED_MIME_TYPES = new Set([
  'image/jpeg', 'image/png', 'image/gif', 'image/webp',
  'application/pdf'
])

const MAX_FILE_SIZE = 5 * 1024 * 1024 // 5MB

export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  
  const formData = await readFormData(event)
  const file = formData.get('file') as File
  
  if (!file) {
    throw createError({ statusCode: 400, message: 'No file provided' })
  }
  
  // Validate file size
  if (file.size > MAX_FILE_SIZE) {
    throw createError({ statusCode: 413, message: 'File too large (max 5MB)' })
  }
  
  // Validate MIME type from Content-Type header
  if (!ALLOWED_MIME_TYPES.has(file.type)) {
    throw createError({ statusCode: 415, message: 'File type not allowed' })
  }
  
  // Read file content and verify magic bytes
  const buffer = Buffer.from(await file.arrayBuffer())
  
  if (!isValidFileType(buffer, file.type)) {
    throw createError({ statusCode: 415, message: 'Invalid file content' })
  }
  
  // Generate safe filename
  const hash = createHash('sha256').update(buffer).digest('hex').slice(0, 16)
  const ext = path.extname(file.name).toLowerCase()
  const safeFilename = `${user.id}-${hash}${ext}`
  
  // Store file (use cloud storage in production)
  // await uploadToS3(buffer, safeFilename, file.type)
  
  return {
    url: `/uploads/${safeFilename}`,
    filename: safeFilename
  }
})

function isValidFileType(buffer: Buffer, mimeType: string): boolean {
  // Check magic bytes
  const signatures: Record<string, number[][]> = {
    'image/jpeg': [[0xFF, 0xD8, 0xFF]],
    'image/png': [[0x89, 0x50, 0x4E, 0x47]],
    'image/gif': [[0x47, 0x49, 0x46, 0x38]],
    'application/pdf': [[0x25, 0x50, 0x44, 0x46]]
  }
  
  const sigs = signatures[mimeType]
  if (!sigs) return false
  
  return sigs.some(sig => sig.every((byte, i) => buffer[i] === byte))
}
```

## Environment Variables Security

```typescript
// server/utils/validateEnv.ts
import { z } from 'zod'

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(64, 'JWT_SECRET must be at least 64 characters'),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
  REDIS_URL: z.string().optional()
})

export function validateEnv() {
  const result = envSchema.safeParse(process.env)
  
  if (!result.success) {
    const errors = result.error.errors.map(e => `${e.path.join('.')}: ${e.message}`)
    throw new Error(`Invalid environment variables:\n${errors.join('\n')}`)
  }
  
  return result.data
}

// Call during app startup
if (process.env.NODE_ENV !== 'test') {
  validateEnv()
}
```

## Security Testing

```typescript
// tests/security/auth.test.ts
import { describe, it, expect } from 'vitest'

describe('Authentication Security', () => {
  it('rejects requests without token', async () => {
    const response = await $fetch('/api/admin/users', {
      ignoreResponseError: true
    })
    expect(response.statusCode).toBe(401)
  })
  
  it('rejects expired tokens', async () => {
    const expiredToken = 'eyJhbGciOiJIUzI1NiJ9.eyJleHAiOjB9.invalid'
    
    const response = await $fetch('/api/profile', {
      headers: { Authorization: `Bearer ${expiredToken}` },
      ignoreResponseError: true
    })
    expect(response.statusCode).toBe(401)
  })
  
  it('blocks brute force on login', async () => {
    // Make 11 failed login attempts
    for (let i = 0; i < 11; i++) {
      await $fetch('/api/auth/login', {
        method: 'POST',
        body: { email: 'test@example.com', password: 'wrong' },
        ignoreResponseError: true
      })
    }
    
    // 11th attempt should be rate limited
    const response = await $fetch('/api/auth/login', {
      method: 'POST',
      body: { email: 'test@example.com', password: 'correct' },
      ignoreResponseError: true
    })
    
    expect(response.statusCode).toBe(429)
  })
})
```

## สรุป

Security Hardening ต้องทำอย่างครบถ้วน:
1. Validate และ sanitize ทุก input ด้วย Zod
2. ใช้ parameterized queries (Prisma) ป้องกัน SQL injection
3. Implement CSRF protection สำหรับ form submissions
4. Rate limiting บน sensitive endpoints
5. JWT best practices (strong secret, short expiry)
6. Security headers ผ่าน nuxt-security module
7. Regular dependency updates และ security audits
8. Security scanning ใน CI/CD ด้วย Snyk/Semgrep
9. File upload validation ตรวจสอบ magic bytes
10. Security testing เป็นส่วนหนึ่งของ test suite
