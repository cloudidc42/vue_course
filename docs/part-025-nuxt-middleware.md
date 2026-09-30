# Part 25: Nuxt Middleware

## Route Middleware คืออะไร?

Route Middleware คือ function ที่ทำงานก่อนที่ page จะ render โดย middleware สามารถ:
- ตรวจสอบ authentication
- Redirect ไปหน้าอื่น
- เปลี่ยน route meta
- Log analytics

### ประเภทของ Middleware

```
Nuxt Middleware
├── Route Middleware (middleware/)
│   ├── Inline    - เขียนใน page component
│   ├── Named     - ไฟล์ใน middleware/
│   └── Global    - ไฟล์ที่ลงท้ายด้วย .global.ts
└── Server Middleware (server/middleware/)
    └── ทำงานทุก request บน server
```

---

## Inline Middleware

Inline middleware เขียนตรงใน `definePageMeta()` ของ page

```vue
<!-- pages/profile.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: [
    // Inline middleware function
    (to, from) => {
      const auth = useAuthStore()
      
      if (!auth.isLoggedIn) {
        return navigateTo('/login')
      }
      
      // ไม่ return อะไร = ผ่านได้
    }
  ]
})
</script>
```

### ตัวอย่าง Inline Middleware ที่ซับซ้อนขึ้น

```vue
<!-- pages/admin/index.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: [
    // ตรวจสอบ auth
    async (to, from) => {
      const { user, isLoggedIn } = useAuth()
      
      // รอ auth check เสร็จก่อน
      await until(isLoggedIn).toBeTruthy({ timeout: 3000 })
      
      if (!isLoggedIn.value) {
        return navigateTo({
          path: '/login',
          query: { redirect: to.fullPath }
        })
      }
    },
    
    // ตรวจสอบ role
    (to) => {
      const { user } = useAuth()
      
      if (user.value?.role !== 'admin') {
        return navigateTo('/403')
      }
    }
  ]
})
</script>
```

---

## Named Middleware

Named middleware เป็นไฟล์ใน `middleware/` directory

### สร้าง Named Middleware

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const auth = useAuthStore()
  
  // ตรวจสอบว่า login หรือยัง
  if (!auth.isAuthenticated) {
    // Redirect ไป login พร้อม redirect URL
    return navigateTo({
      path: '/login',
      query: { redirect: to.fullPath }
    })
  }
})
```

```typescript
// middleware/admin.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const auth = useAuthStore()
  
  if (!auth.user) {
    return navigateTo('/login')
  }
  
  if (!['admin', 'superadmin'].includes(auth.user.role)) {
    // ไม่มีสิทธิ์ - ส่งไปหน้า 403
    return abortNavigation(
      createError({
        statusCode: 403,
        statusMessage: 'Forbidden: คุณไม่มีสิทธิ์เข้าถึงส่วนนี้'
      })
    )
  }
})
```

### ใช้ Named Middleware ใน Page

```vue
<!-- pages/dashboard.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: ['auth']  // ใช้ middleware/auth.ts
})
</script>
```

```vue
<!-- pages/admin/users.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: ['auth', 'admin']  // หลาย middleware (รันตามลำดับ)
})
</script>
```

---

## Global Middleware

Global middleware ทำงานกับทุก route โดยอัตโนมัติ ไม่ต้องระบุใน page

### สร้าง Global Middleware

```typescript
// middleware/analytics.global.ts
// ไฟล์ที่ลงท้ายด้วย .global.ts จะทำงานกับทุก route

export default defineNuxtRouteMiddleware((to, from) => {
  // Track pageview
  if (process.client) {
    // ส่ง analytics event
    console.log(`[Analytics] Navigate to: ${to.path}`)
    
    // gtag, mixpanel, etc.
    // gtag('event', 'page_view', { page_path: to.path })
  }
})
```

```typescript
// middleware/maintenance.global.ts
export default defineNuxtRouteMiddleware((to) => {
  const config = useRuntimeConfig()
  
  // ถ้าอยู่ใน maintenance mode
  if (config.public.maintenanceMode && to.path !== '/maintenance') {
    return navigateTo('/maintenance')
  }
})
```

### ลำดับการทำงานของ Middleware

```
Request เข้ามา
    │
    ▼
Global Middleware (.global.ts) - รันทุก route
    │
    ▼
Named Middleware (กำหนดใน definePageMeta)
    │
    ▼
Inline Middleware (กำหนดใน definePageMeta array)
    │
    ▼
Page Component Render
```

---

## Auth Middleware

### Auth Middleware แบบสมบูรณ์

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware(async (to, from) => {
  // ดึง auth state
  const authStore = useAuthStore()
  
  // ถ้ายังไม่ได้ initialize auth
  if (!authStore.initialized) {
    await authStore.init()
  }
  
  // หน้าที่ต้อง login
  const requiresAuth = to.meta.requiresAuth !== false  // default: ต้อง auth
  const guestOnly = to.meta.guestOnly === true          // หน้าสำหรับ guest เท่านั้น
  
  if (requiresAuth && !authStore.isAuthenticated) {
    // เก็บ URL ที่ต้องการไปหลัง login
    const redirectUrl = to.fullPath
    
    return navigateTo({
      path: '/login',
      query: redirectUrl !== '/' ? { redirect: redirectUrl } : {}
    })
  }
  
  if (guestOnly && authStore.isAuthenticated) {
    // ถ้า login แล้วพยายามเข้าหน้า login/register
    return navigateTo('/dashboard')
  }
})
```

### Auth Store (Pinia)

```typescript
// stores/auth.ts
import { defineStore } from 'pinia'

interface User {
  id: number
  name: string
  email: string
  role: string
  avatar?: string
}

export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)
  const initialized = ref(false)
  
  const isAuthenticated = computed(() => !!user.value && !!token.value)
  
  async function init() {
    // ดึง token จาก cookie
    const cookieToken = useCookie('auth-token')
    
    if (cookieToken.value) {
      token.value = cookieToken.value
      try {
        // ดึงข้อมูล user จาก API
        user.value = await $fetch('/api/auth/me', {
          headers: { Authorization: `Bearer ${token.value}` }
        })
      } catch {
        // Token หมดอายุหรือไม่ valid
        await logout()
      }
    }
    
    initialized.value = true
  }
  
  async function login(email: string, password: string) {
    const response = await $fetch<{ user: User; token: string }>('/api/auth/login', {
      method: 'POST',
      body: { email, password }
    })
    
    user.value = response.user
    token.value = response.token
    
    // บันทึก token ใน cookie
    const cookieToken = useCookie('auth-token', {
      maxAge: 60 * 60 * 24 * 7,  // 7 วัน
      secure: true,
      httpOnly: true,
      sameSite: 'lax'
    })
    cookieToken.value = response.token
  }
  
  async function logout() {
    try {
      await $fetch('/api/auth/logout', { method: 'POST' })
    } catch {}
    
    user.value = null
    token.value = null
    
    const cookieToken = useCookie('auth-token')
    cookieToken.value = null
    
    await navigateTo('/login')
  }
  
  return { user, token, initialized, isAuthenticated, init, login, logout }
})
```

### ใช้ในหน้าต่างๆ

```vue
<!-- pages/dashboard.vue - ต้อง login -->
<script setup lang="ts">
definePageMeta({
  requiresAuth: true,   // custom meta (default ใน middleware แล้ว)
  middleware: ['auth']
})
</script>

<!-- pages/login.vue - guest only -->
<script setup lang="ts">
definePageMeta({
  guestOnly: true,      // redirect ถ้า login แล้ว
  middleware: ['auth'],
  layout: false
})

const route = useRoute()
const authStore = useAuthStore()

async function handleLogin(credentials: { email: string; password: string }) {
  await authStore.login(credentials.email, credentials.password)
  
  // Redirect to original destination
  const redirect = route.query.redirect as string || '/dashboard'
  await navigateTo(redirect)
}
</script>
```

---

## Redirect Logic

### Redirect Patterns

```typescript
// middleware/redirect.ts

export default defineNuxtRouteMiddleware((to, from) => {
  // 1. Simple redirect
  if (to.path === '/old-path') {
    return navigateTo('/new-path', { redirectCode: 301 })
  }
  
  // 2. Redirect กับ query params
  if (to.path === '/search' && !to.query.q) {
    return navigateTo({ path: '/search', query: { q: '', type: 'all' } })
  }
  
  // 3. Redirect ด้วย replace (ไม่เพิ่ม history)
  if (to.path === '/temp') {
    return navigateTo('/permanent', { replace: true })
  }
  
  // 4. External redirect
  if (to.path === '/docs') {
    return navigateTo('https://docs.example.com', { external: true })
  }
  
  // 5. Abort navigation
  if (to.meta.disabled) {
    return abortNavigation()  // หยุด navigation โดยไม่ redirect
  }
  
  // 6. Abort กับ error
  if (to.path.includes('forbidden')) {
    return abortNavigation(createError({ statusCode: 403 }))
  }
})
```

### Locale Redirect Middleware

```typescript
// middleware/locale.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const supportedLocales = ['th', 'en', 'ja']
  const defaultLocale = 'th'
  
  // ดึง locale จาก cookie หรือ browser preference
  const localeCookie = useCookie('locale')
  const browserLocale = process.client
    ? navigator.language.split('-')[0]
    : 'th'
  
  const currentLocale = localeCookie.value || browserLocale
  
  // ตรวจสอบว่า path มี locale prefix
  const pathLocale = to.path.split('/')[1]
  const hasLocalePrefix = supportedLocales.includes(pathLocale)
  
  if (!hasLocalePrefix) {
    const locale = supportedLocales.includes(currentLocale)
      ? currentLocale
      : defaultLocale
    
    return navigateTo(`/${locale}${to.fullPath}`, { redirectCode: 301 })
  }
  
  // บันทึก locale ที่ใช้
  localeCookie.value = pathLocale
})
```

---

## Server Middleware

Server middleware ทำงานบน server สำหรับทุก request (ก่อน API routes)

### สร้าง Server Middleware

```typescript
// server/middleware/logger.ts
export default defineEventHandler((event) => {
  const start = Date.now()
  
  // เมื่อ response ส่งออกไป
  event.node.res.on('finish', () => {
    const duration = Date.now() - start
    const method = event.node.req.method
    const url = event.node.req.url
    const status = event.node.res.statusCode
    
    console.log(`[${new Date().toISOString()}] ${method} ${url} ${status} ${duration}ms`)
  })
})
```

```typescript
// server/middleware/cors.ts
export default defineEventHandler((event) => {
  // กำหนด CORS headers
  setResponseHeaders(event, {
    'Access-Control-Allow-Origin': process.env.ALLOWED_ORIGIN || '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    'Access-Control-Max-Age': '86400'
  })
  
  // Handle preflight request
  if (event.node.req.method === 'OPTIONS') {
    event.node.res.statusCode = 204
    return ''
  }
})
```

```typescript
// server/middleware/auth.ts - Server-side auth check
export default defineEventHandler(async (event) => {
  // เฉพาะ API routes เท่านั้น
  if (!event.path.startsWith('/api/')) return
  
  // บาง API ไม่ต้องการ auth
  const publicPaths = ['/api/auth/login', '/api/auth/register', '/api/posts']
  if (publicPaths.some(path => event.path.startsWith(path))) return
  
  // ตรวจสอบ token
  const authHeader = getRequestHeader(event, 'authorization')
  
  if (!authHeader?.startsWith('Bearer ')) {
    throw createError({
      statusCode: 401,
      statusMessage: 'Unauthorized: Missing token'
    })
  }
  
  const token = authHeader.substring(7)
  
  try {
    // Verify JWT token
    const payload = verifyJWT(token)
    
    // เพิ่ม user info เข้า event context
    event.context.user = payload
  } catch (error) {
    throw createError({
      statusCode: 401,
      statusMessage: 'Unauthorized: Invalid token'
    })
  }
})
```

---

## ตัวอย่าง: Auth Protected Routes และ Locale Redirect

### โครงสร้างสมบูรณ์

```
middleware/
├── auth.ts              # Authentication check
├── admin.ts             # Admin role check
├── guest.ts             # Guest-only pages
├── verified.ts          # Email verified check
└── analytics.global.ts  # Global analytics tracking

server/middleware/
├── logger.ts            # Request logging
├── cors.ts              # CORS headers
└── rate-limit.ts        # Rate limiting
```

### Complete Auth System

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware(async (to) => {
  const { $auth } = useNuxtApp()
  
  // รอ auth initialization
  if (!$auth.initialized) {
    await $auth.initialize()
  }
  
  if (!$auth.loggedIn) {
    return navigateTo(`/login?redirect=${encodeURIComponent(to.fullPath)}`)
  }
  
  // ตรวจสอบ email verification
  if (to.meta.requiresVerification && !$auth.user.emailVerified) {
    return navigateTo('/verify-email')
  }
})
```

```typescript
// middleware/guest.ts
export default defineNuxtRouteMiddleware(() => {
  const { $auth } = useNuxtApp()
  
  if ($auth.loggedIn) {
    return navigateTo('/dashboard')
  }
})
```

```typescript
// middleware/admin.ts
export default defineNuxtRouteMiddleware(() => {
  const { $auth } = useNuxtApp()
  const allowedRoles = ['admin', 'superadmin', 'editor']
  
  if (!$auth.loggedIn) {
    return navigateTo('/login')
  }
  
  if (!allowedRoles.includes($auth.user.role)) {
    throw createError({
      statusCode: 403,
      statusMessage: 'คุณไม่มีสิทธิ์เข้าถึงส่วนนี้'
    })
  }
})
```

### Rate Limiting Middleware

```typescript
// server/middleware/rate-limit.ts
const requestCounts = new Map<string, { count: number; resetAt: number }>()

export default defineEventHandler((event) => {
  // เฉพาะ API เท่านั้น
  if (!event.path.startsWith('/api/')) return
  
  const ip = getRequestIP(event) || 'unknown'
  const now = Date.now()
  const windowMs = 60 * 1000  // 1 นาที
  const maxRequests = 100
  
  const current = requestCounts.get(ip)
  
  if (!current || now > current.resetAt) {
    requestCounts.set(ip, { count: 1, resetAt: now + windowMs })
    return
  }
  
  current.count++
  
  if (current.count > maxRequests) {
    throw createError({
      statusCode: 429,
      statusMessage: 'Too Many Requests',
      message: 'คุณส่ง request มากเกินไป กรุณารอสักครู่'
    })
  }
  
  // เพิ่ม rate limit headers
  setResponseHeaders(event, {
    'X-RateLimit-Limit': String(maxRequests),
    'X-RateLimit-Remaining': String(maxRequests - current.count),
    'X-RateLimit-Reset': String(Math.ceil(current.resetAt / 1000))
  })
})
```

### หน้า Login สมบูรณ์

```vue
<!-- pages/login.vue -->
<script setup lang="ts">
definePageMeta({
  layout: false,
  middleware: ['guest']
})

const route = useRoute()

const form = reactive({
  email: '',
  password: '',
  remember: false
})

const loading = ref(false)
const error = ref<string | null>(null)

const authStore = useAuthStore()

async function handleLogin() {
  if (!form.email || !form.password) {
    error.value = 'กรุณากรอกอีเมลและรหัสผ่าน'
    return
  }
  
  loading.value = true
  error.value = null
  
  try {
    await authStore.login(form.email, form.password)
    
    // Redirect กลับไปหน้าที่ต้องการ
    const redirect = route.query.redirect as string || '/dashboard'
    await navigateTo(decodeURIComponent(redirect))
  } catch (err: any) {
    if (err.statusCode === 401) {
      error.value = 'อีเมลหรือรหัสผ่านไม่ถูกต้อง'
    } else if (err.statusCode === 429) {
      error.value = 'พยายาม login มากเกินไป กรุณารอสักครู่'
    } else {
      error.value = 'เกิดข้อผิดพลาด กรุณาลองใหม่'
    }
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="login-page">
    <div class="login-card">
      <h1>เข้าสู่ระบบ</h1>
      
      <div v-if="error" class="error-alert">{{ error }}</div>
      
      <form @submit.prevent="handleLogin">
        <div class="form-group">
          <label>อีเมล</label>
          <input
            v-model="form.email"
            type="email"
            placeholder="your@email.com"
            :disabled="loading"
            autocomplete="email"
          />
        </div>
        
        <div class="form-group">
          <label>รหัสผ่าน</label>
          <input
            v-model="form.password"
            type="password"
            placeholder="••••••••"
            :disabled="loading"
            autocomplete="current-password"
          />
        </div>
        
        <div class="form-options">
          <label>
            <input v-model="form.remember" type="checkbox" />
            จำฉันไว้
          </label>
          <NuxtLink to="/forgot-password">ลืมรหัสผ่าน?</NuxtLink>
        </div>
        
        <button type="submit" :disabled="loading" class="btn-primary">
          {{ loading ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ' }}
        </button>
      </form>
      
      <p class="register-link">
        ยังไม่มีบัญชี?
        <NuxtLink to="/register">สมัครสมาชิก</NuxtLink>
      </p>
    </div>
  </div>
</template>

<style scoped>
.login-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.login-card {
  background: white;
  border-radius: 16px;
  padding: 2.5rem;
  width: 100%;
  max-width: 400px;
  box-shadow: 0 20px 60px rgba(0,0,0,0.2);
}

.form-group {
  margin-bottom: 1.25rem;
}

.form-group label {
  display: block;
  font-weight: 600;
  margin-bottom: 0.5rem;
  color: #374151;
}

.form-group input {
  width: 100%;
  padding: 0.75rem 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.2s;
}

.form-group input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.btn-primary {
  width: 100%;
  padding: 0.875rem;
  background: #667eea;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: opacity 0.2s;
}

.btn-primary:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.error-alert {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #dc2626;
  padding: 0.75rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.875rem;
}
</style>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Route Middleware** - ทำงานก่อน page render
2. **Inline Middleware** - เขียนตรงใน `definePageMeta()`
3. **Named Middleware** - ไฟล์ใน `middleware/`
4. **Global Middleware** - ทำงานกับทุก route (`.global.ts`)
5. **Auth Middleware** - ตรวจสอบ authentication/authorization
6. **Redirect Logic** - `navigateTo()`, `abortNavigation()`
7. **Server Middleware** - ทำงานบน server สำหรับทุก request

**ถัดไป**: Part 26 - Nuxt Plugins
