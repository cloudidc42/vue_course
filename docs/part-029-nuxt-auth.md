# Part 29: Authentication ใน Nuxt

## Authentication Patterns ใน Nuxt

Authentication (การยืนยันตัวตน) มีหลายรูปแบบที่ใช้กับ Nuxt ได้:

```
Authentication Patterns
├── JWT (JSON Web Token)
│   ├── Stateless - server ไม่เก็บ session
│   └── Access + Refresh tokens
├── Session-based
│   ├── Server เก็บ session data
│   └── Client เก็บแค่ session ID ใน cookie
├── OAuth/Social Login
│   └── Google, GitHub, Facebook, etc.
└── Magic Link / Passwordless
    └── ส่ง link ไปทาง email
```

---

## JWT Authentication

### สร้าง JWT Utilities

```typescript
// server/utils/jwt.ts
import jwt from 'jsonwebtoken'
import type { H3Event } from 'h3'

export interface TokenPayload {
  userId: number
  email: string
  role: string
}

export function generateTokens(payload: TokenPayload) {
  const config = useRuntimeConfig()
  
  // Access token - อายุสั้น (15 นาที - 1 ชั่วโมง)
  const accessToken = jwt.sign(payload, config.jwtAccessSecret, {
    expiresIn: '15m'
  })
  
  // Refresh token - อายุยาว (7-30 วัน)
  const refreshToken = jwt.sign(
    { userId: payload.userId },
    config.jwtRefreshSecret,
    { expiresIn: '30d' }
  )
  
  return { accessToken, refreshToken }
}

export function verifyAccessToken(token: string): TokenPayload {
  const config = useRuntimeConfig()
  return jwt.verify(token, config.jwtAccessSecret) as TokenPayload
}

export function verifyRefreshToken(token: string): { userId: number } {
  const config = useRuntimeConfig()
  return jwt.verify(token, config.jwtRefreshSecret) as { userId: number }
}

export async function requireAuth(event: H3Event): Promise<TokenPayload> {
  // ลองดึง token จาก Authorization header ก่อน
  const authHeader = getRequestHeader(event, 'authorization')
  let token = authHeader?.replace('Bearer ', '')
  
  // ถ้าไม่มีใน header ลองดูใน cookie
  if (!token) {
    token = getCookie(event, 'access-token') || undefined
  }
  
  if (!token) {
    throw createError({
      statusCode: 401,
      statusMessage: 'Authentication required'
    })
  }
  
  try {
    const payload = verifyAccessToken(token)
    event.context.user = payload
    return payload
  } catch (error: any) {
    if (error.name === 'TokenExpiredError') {
      throw createError({
        statusCode: 401,
        statusMessage: 'Token expired',
        data: { code: 'TOKEN_EXPIRED' }
      })
    }
    throw createError({
      statusCode: 401,
      statusMessage: 'Invalid token'
    })
  }
}
```

---

## Register, Login, Logout API

### Register

```typescript
// server/api/auth/register.post.ts
import bcrypt from 'bcryptjs'
import { z } from 'zod'
import { db } from '~/server/utils/db'
import { generateTokens } from '~/server/utils/jwt'

const RegisterSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email('อีเมลไม่ถูกต้อง'),
  password: z.string()
    .min(8, 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
    .regex(/[A-Z]/, 'ต้องมีตัวพิมพ์ใหญ่')
    .regex(/[0-9]/, 'ต้องมีตัวเลข'),
  confirmPassword: z.string()
}).refine(data => data.password === data.confirmPassword, {
  message: 'รหัสผ่านไม่ตรงกัน',
  path: ['confirmPassword']
})

export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  
  const result = RegisterSchema.safeParse(body)
  if (!result.success) {
    throw createError({
      statusCode: 422,
      statusMessage: 'Validation Error',
      data: result.error.flatten()
    })
  }
  
  const { name, email, password } = result.data
  
  // ตรวจสอบว่า email ซ้ำไหม
  const existing = await db.user.findUnique({ where: { email } })
  if (existing) {
    throw createError({
      statusCode: 409,
      statusMessage: 'อีเมลนี้มีการลงทะเบียนแล้ว'
    })
  }
  
  // Hash password
  const hashedPassword = await bcrypt.hash(password, 12)
  
  // สร้าง user
  const user = await db.user.create({
    data: {
      name,
      email,
      password: hashedPassword,
      role: 'user'
    },
    select: {
      id: true,
      name: true,
      email: true,
      role: true,
      createdAt: true
    }
  })
  
  // สร้าง tokens
  const tokens = generateTokens({
    userId: user.id,
    email: user.email,
    role: user.role
  })
  
  // บันทึก refresh token ใน DB
  await db.refreshToken.create({
    data: {
      token: tokens.refreshToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
    }
  })
  
  // Set cookies
  setCookie(event, 'access-token', tokens.accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 15 * 60  // 15 นาที
  })
  
  setCookie(event, 'refresh-token', tokens.refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 30 * 24 * 60 * 60  // 30 วัน
  })
  
  setResponseStatus(event, 201)
  
  return {
    user,
    token: tokens.accessToken  // ส่ง access token กลับด้วยสำหรับ client usage
  }
})
```

### Login

```typescript
// server/api/auth/login.post.ts
import bcrypt from 'bcryptjs'
import { db } from '~/server/utils/db'
import { generateTokens } from '~/server/utils/jwt'

export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const { email, password } = body
  
  if (!email || !password) {
    throw createError({
      statusCode: 400,
      statusMessage: 'กรุณาระบุอีเมลและรหัสผ่าน'
    })
  }
  
  // ค้นหา user
  const user = await db.user.findUnique({
    where: { email: email.toLowerCase() }
  })
  
  if (!user) {
    // ไม่เปิดเผยว่า email ไม่มีในระบบ (security)
    throw createError({
      statusCode: 401,
      statusMessage: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง'
    })
  }
  
  // ตรวจสอบ password
  const isValid = await bcrypt.compare(password, user.password)
  if (!isValid) {
    throw createError({
      statusCode: 401,
      statusMessage: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง'
    })
  }
  
  // สร้าง tokens
  const tokens = generateTokens({
    userId: user.id,
    email: user.email,
    role: user.role
  })
  
  // บันทึก refresh token
  await db.refreshToken.upsert({
    where: { userId: user.id },
    update: {
      token: tokens.refreshToken,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
    },
    create: {
      token: tokens.refreshToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
    }
  })
  
  // Set HTTP-only cookies
  setCookie(event, 'access-token', tokens.accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 15 * 60
  })
  
  setCookie(event, 'refresh-token', tokens.refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 30 * 24 * 60 * 60
  })
  
  return {
    user: {
      id: user.id,
      name: user.name,
      email: user.email,
      role: user.role,
      avatar: user.avatar
    },
    token: tokens.accessToken
  }
})
```

### Logout

```typescript
// server/api/auth/logout.post.ts
import { db } from '~/server/utils/db'

export default defineEventHandler(async (event) => {
  const refreshToken = getCookie(event, 'refresh-token')
  
  if (refreshToken) {
    // ลบ refresh token จาก DB
    await db.refreshToken.deleteMany({
      where: { token: refreshToken }
    }).catch(() => {})
  }
  
  // ลบ cookies
  deleteCookie(event, 'access-token')
  deleteCookie(event, 'refresh-token')
  
  return { success: true, message: 'ออกจากระบบสำเร็จ' }
})
```

### Get Current User

```typescript
// server/api/auth/me.get.ts
import { db } from '~/server/utils/db'
import { requireAuth } from '~/server/utils/jwt'

export default defineEventHandler(async (event) => {
  const payload = await requireAuth(event)
  
  const user = await db.user.findUnique({
    where: { id: payload.userId },
    select: {
      id: true,
      name: true,
      email: true,
      role: true,
      avatar: true,
      createdAt: true
    }
  })
  
  if (!user) {
    throw createError({ statusCode: 404, statusMessage: 'User not found' })
  }
  
  return user
})
```

---

## Refresh Tokens

```typescript
// server/api/auth/refresh.post.ts
import { db } from '~/server/utils/db'
import { generateTokens, verifyRefreshToken } from '~/server/utils/jwt'

export default defineEventHandler(async (event) => {
  const refreshToken = getCookie(event, 'refresh-token')
  
  if (!refreshToken) {
    throw createError({ statusCode: 401, statusMessage: 'No refresh token' })
  }
  
  try {
    const payload = verifyRefreshToken(refreshToken)
    
    // ตรวจสอบว่า refresh token ยังอยู่ใน DB
    const storedToken = await db.refreshToken.findFirst({
      where: { token: refreshToken, userId: payload.userId }
    })
    
    if (!storedToken || storedToken.expiresAt < new Date()) {
      throw createError({ statusCode: 401, statusMessage: 'Refresh token expired' })
    }
    
    // ดึงข้อมูล user
    const user = await db.user.findUnique({
      where: { id: payload.userId }
    })
    
    if (!user) {
      throw createError({ statusCode: 404, statusMessage: 'User not found' })
    }
    
    // สร้าง tokens ใหม่ (Token Rotation)
    const tokens = generateTokens({
      userId: user.id,
      email: user.email,
      role: user.role
    })
    
    // อัพเดต refresh token ใน DB
    await db.refreshToken.update({
      where: { id: storedToken.id },
      data: {
        token: tokens.refreshToken,
        expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
      }
    })
    
    // Set cookies ใหม่
    setCookie(event, 'access-token', tokens.accessToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 15 * 60
    })
    
    setCookie(event, 'refresh-token', tokens.refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 30 * 24 * 60 * 60
    })
    
    return {
      token: tokens.accessToken,
      user: { id: user.id, name: user.name, email: user.email, role: user.role }
    }
  } catch (error: any) {
    // ลบ invalid cookies
    deleteCookie(event, 'access-token')
    deleteCookie(event, 'refresh-token')
    
    throw createError({ statusCode: 401, statusMessage: 'Invalid refresh token' })
  }
})
```

---

## User State Management

### Auth Store (Pinia)

```typescript
// stores/auth.ts
import { defineStore } from 'pinia'

export interface User {
  id: number
  name: string
  email: string
  role: string
  avatar?: string
}

export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const isInitialized = ref(false)
  
  const isAuthenticated = computed(() => !!user.value)
  const isAdmin = computed(() => user.value?.role === 'admin')
  const isPremium = computed(() => ['premium', 'admin'].includes(user.value?.role || ''))
  
  // Initialize - เรียกครั้งเดียวตอน app start
  async function initialize() {
    if (isInitialized.value) return
    
    try {
      user.value = await $fetch<User>('/api/auth/me')
    } catch {
      user.value = null
    } finally {
      isInitialized.value = true
    }
  }
  
  async function register(data: {
    name: string
    email: string
    password: string
    confirmPassword: string
  }) {
    const response = await $fetch<{ user: User; token: string }>('/api/auth/register', {
      method: 'POST',
      body: data
    })
    user.value = response.user
    return response
  }
  
  async function login(email: string, password: string) {
    const response = await $fetch<{ user: User; token: string }>('/api/auth/login', {
      method: 'POST',
      body: { email, password }
    })
    user.value = response.user
    return response
  }
  
  async function logout() {
    try {
      await $fetch('/api/auth/logout', { method: 'POST' })
    } catch {}
    user.value = null
    await navigateTo('/login')
  }
  
  async function refreshToken() {
    try {
      const response = await $fetch<{ user: User }>('/api/auth/refresh', {
        method: 'POST'
      })
      user.value = response.user
      return true
    } catch {
      user.value = null
      return false
    }
  }
  
  async function updateProfile(data: Partial<User>) {
    const updated = await $fetch<User>('/api/auth/profile', {
      method: 'PATCH',
      body: data
    })
    user.value = updated
    return updated
  }
  
  return {
    user,
    isInitialized,
    isAuthenticated,
    isAdmin,
    isPremium,
    initialize,
    register,
    login,
    logout,
    refreshToken,
    updateProfile
  }
})
```

### Plugin สำหรับ Initialize Auth

```typescript
// plugins/auth.ts
export default defineNuxtPlugin(async () => {
  const authStore = useAuthStore()
  
  // Initialize auth state ตอน app start
  await authStore.initialize()
})
```

---

## Protected Routes

### Middleware

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware(async (to) => {
  const authStore = useAuthStore()
  
  if (!authStore.isInitialized) {
    await authStore.initialize()
  }
  
  if (!authStore.isAuthenticated) {
    return navigateTo({
      path: '/login',
      query: to.path !== '/' ? { redirect: to.fullPath } : {}
    })
  }
})
```

### ใช้ใน Pages

```vue
<!-- pages/dashboard.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: ['auth']
})

const authStore = useAuthStore()
const { user, isAdmin } = storeToRefs(authStore)
</script>

<template>
  <div>
    <h1>ยินดีต้อนรับ, {{ user?.name }}</h1>
    
    <div v-if="isAdmin" class="admin-section">
      <NuxtLink to="/admin">ไปที่ Admin Panel</NuxtLink>
    </div>
    
    <div class="user-content">
      <!-- user-specific content -->
    </div>
  </div>
</template>
```

---

## ตัวอย่าง: Full Auth System

### หน้า Register

```vue
<!-- pages/register.vue -->
<script setup lang="ts">
definePageMeta({
  layout: false,
  middleware: ['guest']
})

const authStore = useAuthStore()
const route = useRoute()

const form = reactive({
  name: '',
  email: '',
  password: '',
  confirmPassword: ''
})

const errors = reactive<Record<string, string>>({})
const loading = ref(false)
const serverError = ref('')

function validateForm() {
  Object.keys(errors).forEach(k => delete errors[k])
  
  if (!form.name || form.name.length < 2) errors.name = 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร'
  if (!form.email) errors.email = 'กรุณาระบุอีเมล'
  else if (!/\S+@\S+\.\S+/.test(form.email)) errors.email = 'อีเมลไม่ถูกต้อง'
  if (!form.password || form.password.length < 8) errors.password = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร'
  if (form.password !== form.confirmPassword) errors.confirmPassword = 'รหัสผ่านไม่ตรงกัน'
  
  return Object.keys(errors).length === 0
}

async function handleRegister() {
  if (!validateForm()) return
  
  loading.value = true
  serverError.value = ''
  
  try {
    await authStore.register(form)
    const redirect = route.query.redirect as string || '/dashboard'
    await navigateTo(redirect)
  } catch (error: any) {
    if (error.data?.fieldErrors) {
      Object.assign(errors, error.data.fieldErrors)
    } else if (error.statusCode === 409) {
      errors.email = 'อีเมลนี้มีการลงทะเบียนแล้ว'
    } else {
      serverError.value = error.statusMessage || 'เกิดข้อผิดพลาด กรุณาลองใหม่'
    }
  } finally {
    loading.value = false
  }
}

// Password strength indicator
const passwordStrength = computed(() => {
  const p = form.password
  if (!p) return { score: 0, label: '', color: '' }
  
  let score = 0
  if (p.length >= 8) score++
  if (p.length >= 12) score++
  if (/[A-Z]/.test(p)) score++
  if (/[0-9]/.test(p)) score++
  if (/[^A-Za-z0-9]/.test(p)) score++
  
  const levels = [
    { label: 'อ่อนมาก', color: 'text-red-500' },
    { label: 'อ่อน', color: 'text-orange-500' },
    { label: 'ปานกลาง', color: 'text-yellow-500' },
    { label: 'แข็งแรง', color: 'text-blue-500' },
    { label: 'แข็งแรงมาก', color: 'text-green-500' }
  ]
  
  return { score, ...levels[Math.min(score, 4)] }
})
</script>

<template>
  <div class="auth-page">
    <div class="auth-card">
      <div class="auth-header">
        <h1>สมัครสมาชิก</h1>
        <p>สร้างบัญชีเพื่อเริ่มใช้งาน</p>
      </div>
      
      <div v-if="serverError" class="error-alert">{{ serverError }}</div>
      
      <form @submit.prevent="handleRegister" class="auth-form" novalidate>
        <!-- Name -->
        <div class="form-field">
          <label>ชื่อ-นามสกุล</label>
          <input
            v-model="form.name"
            type="text"
            placeholder="สมชาย ใจดี"
            :class="{ error: errors.name }"
            autocomplete="name"
          />
          <span v-if="errors.name" class="field-error">{{ errors.name }}</span>
        </div>
        
        <!-- Email -->
        <div class="form-field">
          <label>อีเมล</label>
          <input
            v-model="form.email"
            type="email"
            placeholder="your@email.com"
            :class="{ error: errors.email }"
            autocomplete="email"
          />
          <span v-if="errors.email" class="field-error">{{ errors.email }}</span>
        </div>
        
        <!-- Password -->
        <div class="form-field">
          <label>รหัสผ่าน</label>
          <input
            v-model="form.password"
            type="password"
            placeholder="อย่างน้อย 8 ตัวอักษร"
            :class="{ error: errors.password }"
            autocomplete="new-password"
          />
          <!-- Strength indicator -->
          <div v-if="form.password" class="password-strength">
            <div class="strength-bars">
              <span
                v-for="i in 5"
                :key="i"
                :class="['bar', i <= passwordStrength.score ? 'filled' : '']"
              ></span>
            </div>
            <span :class="passwordStrength.color">{{ passwordStrength.label }}</span>
          </div>
          <span v-if="errors.password" class="field-error">{{ errors.password }}</span>
        </div>
        
        <!-- Confirm Password -->
        <div class="form-field">
          <label>ยืนยันรหัสผ่าน</label>
          <input
            v-model="form.confirmPassword"
            type="password"
            placeholder="พิมพ์รหัสผ่านอีกครั้ง"
            :class="{ error: errors.confirmPassword }"
            autocomplete="new-password"
          />
          <span v-if="errors.confirmPassword" class="field-error">{{ errors.confirmPassword }}</span>
        </div>
        
        <button type="submit" :disabled="loading" class="btn-submit">
          {{ loading ? 'กำลังสมัคร...' : 'สมัครสมาชิก' }}
        </button>
      </form>
      
      <p class="auth-footer">
        มีบัญชีแล้ว?
        <NuxtLink to="/login">เข้าสู่ระบบ</NuxtLink>
      </p>
    </div>
  </div>
</template>

<style scoped>
.auth-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #00dc82 0%, #36e4da 100%);
  padding: 2rem 1rem;
}

.auth-card {
  background: white;
  border-radius: 20px;
  padding: 2.5rem;
  width: 100%;
  max-width: 440px;
  box-shadow: 0 25px 50px rgba(0,0,0,0.15);
}

.auth-header { text-align: center; margin-bottom: 2rem; }
.auth-header h1 { font-size: 1.75rem; font-weight: 700; color: #1e293b; }
.auth-header p { color: #64748b; margin-top: 0.25rem; }

.form-field { margin-bottom: 1.25rem; }
.form-field label { display: block; font-weight: 600; color: #374151; margin-bottom: 0.4rem; font-size: 0.875rem; }
.form-field input {
  width: 100%;
  padding: 0.75rem 1rem;
  border: 1.5px solid #e5e7eb;
  border-radius: 10px;
  font-size: 0.9rem;
  transition: border-color 0.2s;
}
.form-field input:focus { outline: none; border-color: #00dc82; box-shadow: 0 0 0 3px rgba(0, 220, 130, 0.1); }
.form-field input.error { border-color: #ef4444; }

.field-error { color: #ef4444; font-size: 0.8rem; margin-top: 0.25rem; display: block; }

.password-strength { display: flex; align-items: center; gap: 0.5rem; margin-top: 0.5rem; }
.strength-bars { display: flex; gap: 2px; }
.bar { width: 30px; height: 4px; background: #e5e7eb; border-radius: 2px; }
.bar.filled { background: #00dc82; }

.btn-submit {
  width: 100%;
  padding: 0.875rem;
  background: linear-gradient(135deg, #00dc82, #36e4da);
  color: white;
  border: none;
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: opacity 0.2s;
  margin-top: 0.5rem;
}
.btn-submit:disabled { opacity: 0.7; cursor: not-allowed; }

.error-alert { background: #fef2f2; border: 1px solid #fecaca; color: #dc2626; padding: 0.75rem; border-radius: 8px; margin-bottom: 1rem; font-size: 0.875rem; }
.auth-footer { text-align: center; margin-top: 1.5rem; color: #64748b; font-size: 0.875rem; }
.auth-footer a { color: #00dc82; font-weight: 600; text-decoration: none; }
</style>
```

### หน้า Profile

```vue
<!-- pages/profile.vue -->
<script setup lang="ts">
definePageMeta({ middleware: ['auth'] })

const authStore = useAuthStore()
const { user } = storeToRefs(authStore)

const form = reactive({
  name: user.value?.name || '',
  avatar: user.value?.avatar || ''
})

const passwordForm = reactive({
  currentPassword: '',
  newPassword: '',
  confirmPassword: ''
})

const saving = ref(false)
const changingPassword = ref(false)
const successMessage = ref('')

async function updateProfile() {
  saving.value = true
  try {
    await authStore.updateProfile(form)
    successMessage.value = 'บันทึกข้อมูลสำเร็จ'
    setTimeout(() => successMessage.value = '', 3000)
  } catch (error: any) {
    alert(error.statusMessage || 'เกิดข้อผิดพลาด')
  } finally {
    saving.value = false
  }
}

async function changePassword() {
  if (passwordForm.newPassword !== passwordForm.confirmPassword) {
    alert('รหัสผ่านใหม่ไม่ตรงกัน')
    return
  }
  
  changingPassword.value = true
  try {
    await $fetch('/api/auth/change-password', {
      method: 'POST',
      body: {
        currentPassword: passwordForm.currentPassword,
        newPassword: passwordForm.newPassword
      }
    })
    Object.assign(passwordForm, { currentPassword: '', newPassword: '', confirmPassword: '' })
    alert('เปลี่ยนรหัสผ่านสำเร็จ')
  } catch (error: any) {
    alert(error.statusMessage || 'เกิดข้อผิดพลาด')
  } finally {
    changingPassword.value = false
  }
}
</script>

<template>
  <div class="profile-page max-w-2xl mx-auto py-8 px-4">
    <h1 class="text-2xl font-bold mb-6">โปรไฟล์ของฉัน</h1>
    
    <div v-if="successMessage" class="mb-4 p-3 bg-green-50 border border-green-200 text-green-700 rounded-lg">
      {{ successMessage }}
    </div>
    
    <!-- Profile Form -->
    <div class="bg-white rounded-xl shadow-sm p-6 mb-6">
      <h2 class="text-lg font-semibold mb-4">ข้อมูลส่วนตัว</h2>
      
      <div class="flex items-center gap-4 mb-6">
        <img
          :src="form.avatar || '/default-avatar.png'"
          :alt="user?.name"
          class="w-20 h-20 rounded-full object-cover"
        />
        <div>
          <p class="font-medium">{{ user?.email }}</p>
          <p class="text-sm text-gray-500">สมาชิกตั้งแต่ {{ new Date(user?.createdAt || '').toLocaleDateString('th-TH') }}</p>
        </div>
      </div>
      
      <form @submit.prevent="updateProfile">
        <div class="mb-4">
          <label class="block text-sm font-medium mb-1">ชื่อ</label>
          <input v-model="form.name" type="text" class="w-full border rounded-lg px-3 py-2" />
        </div>
        <div class="mb-4">
          <label class="block text-sm font-medium mb-1">URL รูปโปรไฟล์</label>
          <input v-model="form.avatar" type="url" class="w-full border rounded-lg px-3 py-2" placeholder="https://..." />
        </div>
        <button type="submit" :disabled="saving" class="bg-blue-600 text-white px-4 py-2 rounded-lg disabled:opacity-50">
          {{ saving ? 'กำลังบันทึก...' : 'บันทึก' }}
        </button>
      </form>
    </div>
    
    <!-- Change Password -->
    <div class="bg-white rounded-xl shadow-sm p-6">
      <h2 class="text-lg font-semibold mb-4">เปลี่ยนรหัสผ่าน</h2>
      <form @submit.prevent="changePassword">
        <div class="mb-4">
          <label class="block text-sm font-medium mb-1">รหัสผ่านปัจจุบัน</label>
          <input v-model="passwordForm.currentPassword" type="password" class="w-full border rounded-lg px-3 py-2" />
        </div>
        <div class="mb-4">
          <label class="block text-sm font-medium mb-1">รหัสผ่านใหม่</label>
          <input v-model="passwordForm.newPassword" type="password" class="w-full border rounded-lg px-3 py-2" />
        </div>
        <div class="mb-4">
          <label class="block text-sm font-medium mb-1">ยืนยันรหัสผ่านใหม่</label>
          <input v-model="passwordForm.confirmPassword" type="password" class="w-full border rounded-lg px-3 py-2" />
        </div>
        <button type="submit" :disabled="changingPassword" class="bg-orange-600 text-white px-4 py-2 rounded-lg disabled:opacity-50">
          {{ changingPassword ? 'กำลังเปลี่ยน...' : 'เปลี่ยนรหัสผ่าน' }}
        </button>
      </form>
    </div>
  </div>
</template>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Authentication Patterns** - JWT, Session, OAuth
2. **JWT Authentication** - Access + Refresh tokens
3. **Register/Login/Logout API** - Full implementation
4. **Refresh Tokens** - Token rotation
5. **User State** - Pinia store สำหรับ auth
6. **Protected Routes** - Middleware สำหรับ auth
7. **Full Auth UI** - Register, Login, Profile pages

**ถัดไป**: Part 30 - Nuxt Content
