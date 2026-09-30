# Part 48: Nuxt Security

## ทำไมต้องใส่ใจเรื่อง Security?

เว็บแอปพลิเคชันมีความเสี่ยงจากการโจมตีหลายรูปแบบ การใส่ใจด้าน security ตั้งแต่ต้นช่วยป้องกันข้อมูลผู้ใช้และระบบของเรา

## 1. ติดตั้ง nuxt-security

```bash
npx nuxi module add security
# หรือ
npm install nuxt-security
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  
  security: {
    // ตั้งค่าทั่วไป
    enabled: true,
    
    // HTTPS Strict Transport Security
    headers: {
      crossOriginEmbedderPolicy: 'require-corp',
      crossOriginOpenerPolicy: 'same-origin',
      crossOriginResourcePolicy: 'same-origin',
      
      // Content Security Policy
      contentSecurityPolicy: {
        'base-uri': ["'self'"],
        'font-src': ["'self'", 'https:', 'data:'],
        'form-action': ["'self'"],
        'frame-ancestors': ["'none'"],
        'img-src': ["'self'", 'data:', 'https:'],
        'object-src': ["'none'"],
        'script-src-attr': ["'none'"],
        'style-src': ["'self'", 'https:', "'unsafe-inline'"],
        'script-src': ["'self'", "'nonce-{{nonce}}'"],
        'upgrade-insecure-requests': true
      },
      
      // Permission Policy
      permissionsPolicy: {
        camera: [],
        microphone: [],
        geolocation: [],
        'payment': ['self']
      },
      
      // HSTS
      strictTransportSecurity: {
        maxAge: 63072000, // 2 ปี
        includeSubDomains: true,
        preload: true
      },
      
      // X-Frame-Options
      xFrameOptions: 'DENY',
      xContentTypeOptions: 'nosniff',
      xXSSProtection: '1; mode=block',
      referrerPolicy: 'no-referrer'
    },
    
    // Rate Limiting
    rateLimiter: {
      tokensPerInterval: 150,
      interval: 300000, // 5 minutes
      fireImmediately: false,
      headers: true
    },
    
    // Request Size Limit
    requestSizeLimiter: {
      maxRequestSizeInBytes: 2000000, // 2 MB
      maxUploadFileRequestInBytes: 10000000, // 10 MB
      throwError: true
    },
    
    // CSRF Protection
    csrf: {
      enabled: true,
      cookieKey: '__Host-csrf',
      cookie: {
        path: '/',
        httpOnly: true,
        sameSite: 'strict',
        secure: process.env.NODE_ENV === 'production'
      },
      methodsToProtect: ['POST', 'PUT', 'PATCH', 'DELETE'],
      headerName: 'csrf-token'
    },
    
    // Hidden Routes
    hidePoweredBy: true,
    
    // CORS
    corsHandler: {
      origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
      methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
      credentials: true
    }
  }
})
```

## 2. CSRF Protection

```typescript
// server/api/posts.post.ts - ต้องการ CSRF token
export default defineEventHandler(async (event) => {
  // nuxt-security จัดการ CSRF validation อัตโนมัติ
  // แต่ถ้าต้องการ manual:
  
  const csrfToken = getHeader(event, 'csrf-token')
  if (!csrfToken) {
    throw createError({ statusCode: 403, message: 'Missing CSRF token' })
  }
  
  // ดำเนินการต่อ...
  const body = await readBody(event)
  return { success: true }
})
```

```vue
<!-- การใช้ CSRF token ใน forms -->
<template>
  <form @submit.prevent="submitForm">
    <!-- nuxt-security เพิ่ม CSRF token อัตโนมัติ -->
    <input type="hidden" name="_csrf" :value="csrfToken" />
    
    <input v-model="email" type="email" />
    <button type="submit">ส่ง</button>
  </form>
</template>

<script setup>
// ดึง CSRF token
const { csrf: csrfToken } = useCsrf()

const submitForm = async () => {
  await $fetch('/api/contact', {
    method: 'POST',
    headers: {
      'csrf-token': csrfToken.value
    },
    body: { email: email.value }
  })
}
</script>
```

## 3. XSS Prevention

```typescript
// utils/sanitize.ts
import DOMPurify from 'isomorphic-dompurify'

export const sanitizeHTML = (dirty: string): string => {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: [
      'p', 'br', 'b', 'i', 'em', 'strong', 'a',
      'ul', 'ol', 'li', 'blockquote', 'code', 'pre',
      'h1', 'h2', 'h3', 'h4', 'img'
    ],
    ALLOWED_ATTR: ['href', 'src', 'alt', 'title', 'class'],
    ALLOWED_URI_REGEXP: /^(?:(?:(?:f|ht)tps?|mailto|tel):|[^a-z]|[a-z+\-.]+(?:[^a-z+\-.]:|$))/i,
    FORBID_CONTENTS: ['script', 'style']
  })
}

export const sanitizeText = (text: string): string => {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
}
```

```vue
<!-- ใช้ v-html อย่างปลอดภัย -->
<template>
  <div class="content">
    <!-- ไม่ดี! อาจถูก XSS -->
    <!-- <div v-html="rawContent" /> -->
    
    <!-- ดี! sanitize ก่อน -->
    <div v-html="sanitizedContent" />
  </div>
</template>

<script setup>
import { sanitizeHTML } from '~/utils/sanitize'

const props = defineProps<{ content: string }>()
const sanitizedContent = computed(() => sanitizeHTML(props.content))
</script>
```

## 4. Input Sanitization

```typescript
// server/utils/validation.ts
import { z } from 'zod'
import { sanitizeText } from '~/utils/sanitize'

// Schema ตรวจสอบ input
export const createPostSchema = z.object({
  title: z.string()
    .min(1, 'ต้องกรอก title')
    .max(200, 'title ยาวเกิน 200 ตัวอักษร')
    .transform(sanitizeText),
  
  content: z.string()
    .min(10, 'content ต้องมีอย่างน้อย 10 ตัวอักษร')
    .max(50000, 'content ยาวเกิน'),
  
  email: z.string()
    .email('รูปแบบอีเมลไม่ถูกต้อง')
    .toLowerCase()
    .trim(),
  
  url: z.string()
    .url('URL ไม่ถูกต้อง')
    .refine(url => {
      // ห้าม javascript: และ data: URLs
      const parsed = new URL(url)
      return ['http:', 'https:'].includes(parsed.protocol)
    }, 'Protocol ไม่อนุญาต'),
  
  tags: z.array(
    z.string().max(50).regex(/^[a-z0-9-]+$/, 'tag ต้องเป็น lowercase และ alphanumeric')
  ).max(10, 'มี tag ได้สูงสุด 10 อัน')
})

// Server handler พร้อม validation
export const validateBody = async <T>(event: H3Event, schema: z.ZodSchema<T>): Promise<T> => {
  const body = await readBody(event)
  
  try {
    return schema.parse(body)
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw createError({
        statusCode: 400,
        message: 'ข้อมูลไม่ถูกต้อง',
        data: error.errors.map(e => ({
          field: e.path.join('.'),
          message: e.message
        }))
      })
    }
    throw error
  }
}
```

## 5. Rate Limiting

```typescript
// server/middleware/rateLimit.ts
import { LRUCache } from 'lru-cache'

const rateLimitMap = new LRUCache<string, { count: number; resetTime: number }>({
  max: 10000
})

interface RateLimitOptions {
  limit: number
  windowMs: number
  keyPrefix?: string
}

export const createRateLimiter = (options: RateLimitOptions) => {
  const { limit, windowMs, keyPrefix = 'rate' } = options
  
  return (event: H3Event) => {
    const ip = getHeader(event, 'x-forwarded-for') ||
               getHeader(event, 'x-real-ip') ||
               event.node.req.socket.remoteAddress ||
               'unknown'
    
    const key = `${keyPrefix}:${ip}`
    const now = Date.now()
    
    const current = rateLimitMap.get(key)
    
    if (!current || now > current.resetTime) {
      rateLimitMap.set(key, { count: 1, resetTime: now + windowMs })
      return
    }
    
    if (current.count >= limit) {
      const retryAfter = Math.ceil((current.resetTime - now) / 1000)
      
      setResponseHeaders(event, {
        'X-RateLimit-Limit': String(limit),
        'X-RateLimit-Remaining': '0',
        'X-RateLimit-Reset': String(current.resetTime),
        'Retry-After': String(retryAfter)
      })
      
      throw createError({
        statusCode: 429,
        message: `Too many requests. Try again in ${retryAfter} seconds.`
      })
    }
    
    current.count++
    setResponseHeaders(event, {
      'X-RateLimit-Limit': String(limit),
      'X-RateLimit-Remaining': String(limit - current.count),
      'X-RateLimit-Reset': String(current.resetTime)
    })
  }
}

// ใช้ใน API
export const apiRateLimit = createRateLimiter({
  limit: 100,
  windowMs: 60 * 1000 // 1 minute
})

export const authRateLimit = createRateLimiter({
  limit: 5,
  windowMs: 15 * 60 * 1000, // 15 minutes
  keyPrefix: 'auth'
})
```

## 6. HTTPS และ Security Headers

```typescript
// server/middleware/security.ts
export default defineEventHandler((event) => {
  const headers = {
    // ป้องกัน Clickjacking
    'X-Frame-Options': 'DENY',
    
    // ป้องกัน MIME type sniffing
    'X-Content-Type-Options': 'nosniff',
    
    // XSS Protection (for older browsers)
    'X-XSS-Protection': '1; mode=block',
    
    // Referrer Policy
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    
    // Feature Policy
    'Permissions-Policy': [
      'camera=()',
      'microphone=()',
      'geolocation=()',
      'interest-cohort=()'
    ].join(', '),
    
    // Remove server info
    'X-Powered-By': ''
  }
  
  Object.entries(headers).forEach(([key, value]) => {
    if (value) setResponseHeader(event, key, value)
    else removeResponseHeader(event, key)
  })
})
```

## 7. ตัวอย่าง: Secure API Endpoints

```typescript
// server/api/auth/login.post.ts
import { authRateLimit } from '~/server/utils/rateLimit'
import { z } from 'zod'
import bcrypt from 'bcrypt'

const loginSchema = z.object({
  email: z.string().email().toLowerCase().trim(),
  password: z.string().min(8).max(128)
})

export default defineEventHandler(async (event) => {
  // Rate limit สำหรับ login
  authRateLimit(event)
  
  // Validate input
  const { email, password } = await validateBody(event, loginSchema)
  
  // ค้นหา user
  const user = await prisma.user.findUnique({
    where: { email },
    select: {
      id: true,
      email: true,
      name: true,
      password: true,
      role: true,
      emailVerified: true,
      loginAttempts: true,
      lockedUntil: true
    }
  })
  
  // ตรวจสอบว่า account ถูก lock หรือไม่
  if (user?.lockedUntil && user.lockedUntil > new Date()) {
    const minutesLeft = Math.ceil((user.lockedUntil.getTime() - Date.now()) / 60000)
    throw createError({
      statusCode: 423,
      message: `บัญชีถูกล็อค กรุณาลองใหม่ในอีก ${minutesLeft} นาที`
    })
  }
  
  // Timing-safe comparison (ป้องกัน timing attack)
  const isValid = user?.password
    ? await bcrypt.compare(password, user.password)
    : await bcrypt.compare(password, '$2b$12$invalidhashfortimingsafety')
  
  if (!user || !isValid) {
    // บันทึก failed attempt
    if (user) {
      const attempts = (user.loginAttempts || 0) + 1
      const lockedUntil = attempts >= 5 ? new Date(Date.now() + 30 * 60 * 1000) : null
      
      await prisma.user.update({
        where: { id: user.id },
        data: { loginAttempts: attempts, lockedUntil }
      })
    }
    
    // ไม่บอกว่า email มีอยู่หรือไม่ (ป้องกัน user enumeration)
    throw createError({ statusCode: 401, message: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง' })
  }
  
  // Reset login attempts
  await prisma.user.update({
    where: { id: user.id },
    data: { loginAttempts: 0, lockedUntil: null, lastLoginAt: new Date() }
  })
  
  // สร้าง session
  await setUserSession(event, {
    user: {
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role
    },
    loggedInAt: new Date().toISOString()
  })
  
  return { success: true }
})
```

```typescript
// server/api/admin/users.get.ts - ตัวอย่าง secure admin endpoint
export default defineEventHandler(async (event) => {
  // ตรวจสอบ authentication
  const session = await getUserSession(event)
  if (!session?.user) {
    throw createError({ statusCode: 401, message: 'ยังไม่ได้เข้าสู่ระบบ' })
  }
  
  // ตรวจสอบ authorization
  if (session.user.role !== 'admin') {
    throw createError({ statusCode: 403, message: 'ต้องเป็น admin เท่านั้น' })
  }
  
  const query = getQuery(event)
  
  // Sanitize query parameters
  const page = Math.max(1, parseInt(String(query.page || '1')))
  const limit = Math.min(100, Math.max(1, parseInt(String(query.limit || '20'))))
  
  const users = await prisma.user.findMany({
    skip: (page - 1) * limit,
    take: limit,
    select: {
      id: true,
      email: true,
      name: true,
      role: true,
      createdAt: true,
      // ไม่เลือก password!
    },
    orderBy: { createdAt: 'desc' }
  })
  
  return { users }
})
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **nuxt-security** - Module ที่ครอบคลุมด้าน security
2. **CSRF Protection** - ป้องกัน Cross-Site Request Forgery
3. **XSS Prevention** - ป้องกัน Cross-Site Scripting
4. **Content Security Policy** - ควบคุม resources ที่โหลดได้
5. **Rate Limiting** - ป้องกัน brute force และ abuse
6. **Input Sanitization** - ตรวจสอบและทำความสะอาด input
7. **Security Headers** - HTTP headers ที่ช่วยเพิ่มความปลอดภัย
8. **Secure API Endpoints** - ตัวอย่าง login ที่ปลอดภัย
