# Part 42: Authentication OAuth กับ Nuxt.js

## OAuth 2.0 คืออะไร?

OAuth 2.0 เป็นมาตรฐานการ authorization ที่ช่วยให้ผู้ใช้สามารถ login ด้วยบัญชี Google, GitHub, Facebook ฯลฯ โดยไม่ต้องสร้าง password ใหม่ ทำให้ประสบการณ์ผู้ใช้ดีขึ้นและลดความเสี่ยงด้านความปลอดภัย

## 1. OAuth 2.0 Flow

```
ผู้ใช้ -> แอปของเรา -> OAuth Provider (Google/GitHub)
                            ↓
                    ผู้ใช้ให้สิทธิ์
                            ↓
              OAuth Provider -> แอปของเรา (authorization code)
                            ↓
              แอปของเรา -> OAuth Provider (แลก code เป็น token)
                            ↓
                  ได้รับ access_token + user info
```

### Authorization Code Flow (แนะนำ)

```typescript
// Flow ปลอดภัยที่สุดสำหรับ Web Apps
// 1. Redirect ไป OAuth Provider
const authUrl = new URL('https://accounts.google.com/o/oauth2/v2/auth')
authUrl.searchParams.set('client_id', GOOGLE_CLIENT_ID)
authUrl.searchParams.set('redirect_uri', 'http://localhost:3000/auth/google/callback')
authUrl.searchParams.set('response_type', 'code')
authUrl.searchParams.set('scope', 'openid email profile')
authUrl.searchParams.set('state', generateRandomState()) // ป้องกัน CSRF

// 2. รับ code จาก callback URL
// 3. แลก code เป็น token ที่ฝั่ง Server (ปลอดภัย)
// 4. ดึงข้อมูลผู้ใช้จาก token
```

## 2. ติดตั้ง nuxt-auth-utils

```bash
# ติดตั้ง module
npx nuxi module add auth-utils

# หรือ
npm install nuxt-auth-utils
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-auth-utils'],
  
  runtimeConfig: {
    // Secret keys (server-side only)
    oauth: {
      google: {
        clientId: '',
        clientSecret: '',
      },
      github: {
        clientId: '',
        clientSecret: '',
      }
    }
  }
})
```

## 3. Google OAuth

### ตั้งค่า Google OAuth Credentials

1. ไปที่ [Google Cloud Console](https://console.cloud.google.com)
2. สร้าง Project ใหม่
3. เปิดใช้งาน Google+ API
4. สร้าง OAuth 2.0 Credentials
5. เพิ่ม Authorized redirect URIs

```bash
# .env
NUXT_OAUTH_GOOGLE_CLIENT_ID=your-google-client-id
NUXT_OAUTH_GOOGLE_CLIENT_SECRET=your-google-client-secret
NUXT_SESSION_PASSWORD=your-super-secret-password-32-chars-min
```

### Server Handler สำหรับ Google OAuth

```typescript
// server/routes/auth/google.get.ts
export default defineOAuthGoogleEventHandler({
  config: {
    scope: ['openid', 'email', 'profile'],
    // เพิ่ม params เพิ่มเติม
    authorizationParams: {
      access_type: 'offline', // สำหรับ refresh token
      prompt: 'select_account'
    }
  },
  
  async onSuccess(event, { user, tokens }) {
    // user มีข้อมูล: sub, email, name, picture
    console.log('Google user:', user)
    console.log('Tokens:', tokens)
    
    // บันทึกหรืออัปเดต user ในฐานข้อมูล
    const dbUser = await prisma.user.upsert({
      where: { email: user.email },
      update: {
        name: user.name,
        avatar: user.picture,
        lastLoginAt: new Date()
      },
      create: {
        email: user.email,
        name: user.name,
        avatar: user.picture,
        provider: 'google',
        providerId: user.sub,
        role: 'user'
      }
    })
    
    // สร้าง session
    await setUserSession(event, {
      user: {
        id: dbUser.id,
        email: dbUser.email,
        name: dbUser.name,
        avatar: dbUser.avatar,
        role: dbUser.role
      },
      loggedInAt: new Date()
    })
    
    // Redirect ไปหน้า dashboard
    return sendRedirect(event, '/dashboard')
  },
  
  onError(event, error) {
    console.error('Google OAuth error:', error)
    return sendRedirect(event, '/login?error=google')
  }
})
```

## 4. GitHub OAuth

```bash
# .env
NUXT_OAUTH_GITHUB_CLIENT_ID=your-github-client-id
NUXT_OAUTH_GITHUB_CLIENT_SECRET=your-github-client-secret
```

```typescript
// server/routes/auth/github.get.ts
export default defineOAuthGitHubEventHandler({
  config: {
    scope: ['user:email', 'read:user']
  },
  
  async onSuccess(event, { user, tokens }) {
    // ดึง email เพิ่มเติม (GitHub อาจไม่ return email ใน user object)
    let email = user.email
    
    if (!email) {
      // ดึง email จาก GitHub API
      const emails = await $fetch<Array<{ email: string; primary: boolean; verified: boolean }>>(
        'https://api.github.com/user/emails',
        {
          headers: {
            Authorization: `Bearer ${tokens.access_token}`,
            'User-Agent': 'MyNuxtApp'
          }
        }
      )
      
      const primaryEmail = emails.find(e => e.primary && e.verified)
      email = primaryEmail?.email || ''
    }
    
    if (!email) {
      return sendRedirect(event, '/login?error=no-email')
    }
    
    // Upsert user
    const dbUser = await prisma.user.upsert({
      where: { email },
      update: {
        name: user.name || user.login,
        avatar: user.avatar_url,
        githubUsername: user.login
      },
      create: {
        email,
        name: user.name || user.login,
        avatar: user.avatar_url,
        provider: 'github',
        providerId: String(user.id),
        githubUsername: user.login,
        role: 'user'
      }
    })
    
    await setUserSession(event, {
      user: {
        id: dbUser.id,
        email: dbUser.email,
        name: dbUser.name,
        avatar: dbUser.avatar,
        role: dbUser.role
      }
    })
    
    return sendRedirect(event, '/dashboard')
  },
  
  onError(event, error) {
    return sendRedirect(event, '/login?error=github')
  }
})
```

## 5. Social Login UI Component

```vue
<!-- components/auth/SocialLogin.vue -->
<template>
  <div class="social-login">
    <p class="divider-text">หรือเข้าสู่ระบบด้วย</p>
    
    <div class="social-buttons">
      <!-- Google Login -->
      <a href="/auth/google" class="social-btn google-btn">
        <svg class="social-icon" viewBox="0 0 24 24">
          <path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/>
          <path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/>
          <path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/>
          <path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/>
        </svg>
        เข้าสู่ระบบด้วย Google
      </a>
      
      <!-- GitHub Login -->
      <a href="/auth/github" class="social-btn github-btn">
        <svg class="social-icon" viewBox="0 0 24 24" fill="currentColor">
          <path d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z"/>
        </svg>
        เข้าสู่ระบบด้วย GitHub
      </a>
      
      <!-- Discord Login (ถ้ามี) -->
      <a href="/auth/discord" class="social-btn discord-btn">
        <svg class="social-icon" viewBox="0 0 24 24" fill="#5865F2">
          <path d="M20.317 4.492c-1.53-.69-3.17-1.2-4.885-1.49a.075.075 0 0 0-.079.036c-.21.369-.444.85-.608 1.23a18.566 18.566 0 0 0-5.487 0 12.36 12.36 0 0 0-.617-1.23A.077.077 0 0 0 8.562 3c-1.714.29-3.354.8-4.885 1.491a.07.07 0 0 0-.032.027C.533 9.093-.32 13.555.099 17.961a.08.08 0 0 0 .031.055 20.03 20.03 0 0 0 5.993 2.98.078.078 0 0 0 .084-.026 13.83 13.83 0 0 0 1.226-1.963.074.074 0 0 0-.041-.104 13.201 13.201 0 0 1-1.872-.878.075.075 0 0 1-.008-.125c.126-.093.252-.19.372-.287a.075.075 0 0 1 .078-.01c3.927 1.764 8.18 1.764 12.061 0a.075.075 0 0 1 .079.009c.12.098.245.195.372.288a.075.075 0 0 1-.006.125c-.598.344-1.22.635-1.873.877a.075.075 0 0 0-.041.105c.36.687.772 1.341 1.225 1.962a.077.077 0 0 0 .084.028 19.963 19.963 0 0 0 6.002-2.981.076.076 0 0 0 .032-.054c.5-5.094-.838-9.52-3.549-13.442a.06.06 0 0 0-.031-.028z"/>
        </svg>
        เข้าสู่ระบบด้วย Discord
      </a>
    </div>
  </div>
</template>

<style scoped>
.social-login { margin: 24px 0; }

.divider-text {
  text-align: center;
  position: relative;
  color: #666;
  margin: 16px 0;
}

.divider-text::before,
.divider-text::after {
  content: '';
  position: absolute;
  top: 50%;
  width: 40%;
  height: 1px;
  background: #e0e0e0;
}

.divider-text::before { left: 0; }
.divider-text::after { right: 0; }

.social-buttons {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.social-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 12px 24px;
  border-radius: 8px;
  font-size: 0.95em;
  font-weight: 500;
  text-decoration: none;
  transition: all 0.2s;
  border: 1px solid #e0e0e0;
}

.social-btn:hover { transform: translateY(-1px); box-shadow: 0 4px 12px rgba(0,0,0,0.1); }

.google-btn { background: white; color: #333; }
.google-btn:hover { background: #f8f8f8; }

.github-btn { background: #24292e; color: white; }
.github-btn:hover { background: #2f363d; }

.discord-btn { background: #5865F2; color: white; }
.discord-btn:hover { background: #4752C4; }

.social-icon { width: 20px; height: 20px; }
</style>
```

## 6. Token Management และ Session

```typescript
// composables/useAuth.ts
export const useAuth = () => {
  const { loggedIn, user, session, fetch: refreshSession } = useUserSession()
  
  // Login ด้วย provider
  const loginWith = (provider: 'google' | 'github' | 'discord') => {
    window.location.href = `/auth/${provider}`
  }
  
  // Logout
  const logout = async () => {
    await $fetch('/api/auth/logout', { method: 'POST' })
    await refreshSession()
    await navigateTo('/login')
  }
  
  // ตรวจสอบว่า user มี role ที่ต้องการ
  const hasRole = (role: string | string[]) => {
    if (!user.value) return false
    const roles = Array.isArray(role) ? role : [role]
    return roles.includes(user.value.role)
  }
  
  // ตรวจสอบว่า user มี permission
  const can = (permission: string) => {
    if (!user.value) return false
    return user.value.permissions?.includes(permission) || user.value.role === 'admin'
  }
  
  return {
    loggedIn,
    user,
    session,
    loginWith,
    logout,
    hasRole,
    can,
    refreshSession
  }
}
```

```typescript
// server/api/auth/logout.post.ts
export default defineEventHandler(async (event) => {
  await clearUserSession(event)
  return { success: true }
})
```

```typescript
// server/api/auth/me.get.ts
export default defineEventHandler(async (event) => {
  const session = await getUserSession(event)
  
  if (!session.user) {
    throw createError({ statusCode: 401, message: 'ยังไม่ได้เข้าสู่ระบบ' })
  }
  
  // ดึงข้อมูลล่าสุดจากฐานข้อมูล
  const user = await prisma.user.findUnique({
    where: { id: session.user.id },
    select: {
      id: true,
      email: true,
      name: true,
      avatar: true,
      role: true,
      createdAt: true,
      _count: {
        select: { posts: true, comments: true }
      }
    }
  })
  
  if (!user) {
    await clearUserSession(event)
    throw createError({ statusCode: 404, message: 'ไม่พบผู้ใช้' })
  }
  
  return user
})
```

## 7. Middleware สำหรับป้องกันเส้นทาง

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware(() => {
  const { loggedIn } = useUserSession()
  
  if (!loggedIn.value) {
    return navigateTo('/login?redirect=' + useRoute().fullPath)
  }
})
```

```typescript
// middleware/guest.ts - สำหรับหน้าที่ต้องยังไม่ login
export default defineNuxtRouteMiddleware(() => {
  const { loggedIn } = useUserSession()
  
  if (loggedIn.value) {
    return navigateTo('/dashboard')
  }
})
```

## 8. ตัวอย่าง: Multi-provider Auth System

```vue
<!-- pages/login.vue -->
<template>
  <div class="login-page">
    <div class="login-card">
      <div class="login-header">
        <img src="/logo.svg" alt="Logo" class="logo" />
        <h1>เข้าสู่ระบบ</h1>
        <p>ยินดีต้อนรับกลับ!</p>
      </div>
      
      <!-- Error Alert -->
      <div v-if="errorMessage" class="error-alert">
        {{ errorMessage }}
      </div>
      
      <!-- Email/Password Form -->
      <form v-if="showEmailForm" @submit.prevent="loginWithEmail">
        <div class="form-group">
          <label>อีเมล</label>
          <input
            v-model="email"
            type="email"
            placeholder="your@email.com"
            required
            class="form-input"
          />
        </div>
        
        <div class="form-group">
          <label>รหัสผ่าน</label>
          <div class="password-input">
            <input
              v-model="password"
              :type="showPassword ? 'text' : 'password'"
              placeholder="รหัสผ่าน"
              required
              class="form-input"
            />
            <button
              type="button"
              @click="showPassword = !showPassword"
              class="toggle-password"
            >
              {{ showPassword ? '🙈' : '👁️' }}
            </button>
          </div>
        </div>
        
        <div class="form-options">
          <label class="checkbox-label">
            <input v-model="rememberMe" type="checkbox" />
            จำฉันไว้
          </label>
          <NuxtLink to="/forgot-password" class="forgot-link">
            ลืมรหัสผ่าน?
          </NuxtLink>
        </div>
        
        <button
          type="submit"
          :disabled="isLoading"
          class="btn-login"
        >
          {{ isLoading ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ' }}
        </button>
      </form>
      
      <!-- Social Login -->
      <SocialLogin />
      
      <!-- Toggle Email Form -->
      <button
        @click="showEmailForm = !showEmailForm"
        class="btn-toggle-form"
      >
        {{ showEmailForm ? 'ซ่อนฟอร์ม Email' : 'เข้าสู่ระบบด้วย Email' }}
      </button>
      
      <!-- Register Link -->
      <p class="register-text">
        ยังไม่มีบัญชี?
        <NuxtLink to="/register">สมัครสมาชิก</NuxtLink>
      </p>
    </div>
  </div>
</template>

<script setup>
definePageMeta({
  middleware: 'guest'
})

const route = useRoute()
const { loginWith } = useAuth()

const email = ref('')
const password = ref('')
const rememberMe = ref(false)
const showPassword = ref(false)
const showEmailForm = ref(false)
const isLoading = ref(false)
const errorMessage = ref('')

// แสดง error message จาก URL query
const errorParam = route.query.error
if (errorParam) {
  const errors = {
    google: 'ไม่สามารถเข้าสู่ระบบด้วย Google ได้',
    github: 'ไม่สามารถเข้าสู่ระบบด้วย GitHub ได้',
    'no-email': 'ไม่พบอีเมลจากบัญชีของคุณ',
    unauthorized: 'คุณยังไม่ได้เข้าสู่ระบบ',
  }
  errorMessage.value = errors[errorParam] || 'เกิดข้อผิดพลาด'
}

const loginWithEmail = async () => {
  isLoading.value = true
  errorMessage.value = ''
  
  try {
    const result = await $fetch('/api/auth/login', {
      method: 'POST',
      body: { email: email.value, password: password.value }
    })
    
    const redirect = route.query.redirect as string || '/dashboard'
    await navigateTo(redirect)
  } catch (error) {
    errorMessage.value = error.data?.message || 'อีเมลหรือรหัสผ่านไม่ถูกต้อง'
  } finally {
    isLoading.value = false
  }
}
</script>

<style scoped>
.login-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 24px;
}

.login-card {
  background: white;
  border-radius: 16px;
  padding: 40px;
  width: 100%;
  max-width: 420px;
  box-shadow: 0 20px 60px rgba(0,0,0,0.2);
}

.login-header {
  text-align: center;
  margin-bottom: 32px;
}

.logo { width: 60px; height: 60px; margin-bottom: 16px; }

.login-header h1 {
  font-size: 1.8em;
  color: #333;
  margin: 0;
}

.login-header p {
  color: #666;
  margin-top: 4px;
}

.error-alert {
  background: #fee;
  border: 1px solid #fcc;
  color: #c33;
  padding: 12px;
  border-radius: 8px;
  margin-bottom: 16px;
  font-size: 0.9em;
}

.form-group {
  margin-bottom: 16px;
}

.form-group label {
  display: block;
  font-weight: 500;
  margin-bottom: 6px;
  color: #444;
  font-size: 0.9em;
}

.form-input {
  width: 100%;
  padding: 10px 14px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 1em;
  transition: border-color 0.2s;
  box-sizing: border-box;
}

.form-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.password-input {
  position: relative;
}

.password-input .form-input {
  padding-right: 44px;
}

.toggle-password {
  position: absolute;
  right: 12px;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  cursor: pointer;
  padding: 0;
  font-size: 16px;
}

.form-options {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  font-size: 0.9em;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
}

.forgot-link { color: #667eea; text-decoration: none; }
.forgot-link:hover { text-decoration: underline; }

.btn-login {
  width: 100%;
  padding: 12px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1em;
  font-weight: 600;
  cursor: pointer;
  transition: opacity 0.2s;
}

.btn-login:disabled { opacity: 0.6; cursor: not-allowed; }

.btn-toggle-form {
  width: 100%;
  padding: 10px;
  background: none;
  border: 1px dashed #ccc;
  border-radius: 8px;
  color: #666;
  cursor: pointer;
  font-size: 0.9em;
  margin-top: 8px;
}

.register-text {
  text-align: center;
  color: #666;
  font-size: 0.9em;
  margin-top: 20px;
}

.register-text a { color: #667eea; text-decoration: none; font-weight: 500; }
</style>
```

```typescript
// server/api/auth/login.post.ts
import bcrypt from 'bcrypt'

export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const { email, password } = body
  
  if (!email || !password) {
    throw createError({ statusCode: 400, message: 'กรุณากรอกอีเมลและรหัสผ่าน' })
  }
  
  // ค้นหา user
  const user = await prisma.user.findUnique({
    where: { email },
    select: {
      id: true,
      email: true,
      name: true,
      avatar: true,
      password: true,
      role: true,
      emailVerified: true
    }
  })
  
  if (!user || !user.password) {
    throw createError({ statusCode: 401, message: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง' })
  }
  
  // ตรวจสอบ password
  const isValidPassword = await bcrypt.compare(password, user.password)
  if (!isValidPassword) {
    throw createError({ statusCode: 401, message: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง' })
  }
  
  if (!user.emailVerified) {
    throw createError({ statusCode: 403, message: 'กรุณายืนยันอีเมลก่อนเข้าสู่ระบบ' })
  }
  
  // สร้าง session
  await setUserSession(event, {
    user: {
      id: user.id,
      email: user.email,
      name: user.name,
      avatar: user.avatar,
      role: user.role
    },
    loggedInAt: new Date().toISOString()
  })
  
  return {
    success: true,
    user: {
      id: user.id,
      email: user.email,
      name: user.name
    }
  }
})
```

```typescript
// server/api/auth/register.post.ts
import bcrypt from 'bcrypt'

export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const { name, email, password } = body
  
  // ตรวจสอบ email ซ้ำ
  const existingUser = await prisma.user.findUnique({ where: { email } })
  if (existingUser) {
    throw createError({ statusCode: 409, message: 'อีเมลนี้ถูกใช้งานแล้ว' })
  }
  
  // Hash password
  const hashedPassword = await bcrypt.hash(password, 12)
  
  // สร้าง user
  const user = await prisma.user.create({
    data: {
      name,
      email,
      password: hashedPassword,
      role: 'user',
      provider: 'email'
    }
  })
  
  // ส่ง verification email (ในโปรเจกต์จริง)
  // await sendVerificationEmail(user.email, verificationToken)
  
  return {
    success: true,
    message: 'สมัครสมาชิกสำเร็จ กรุณายืนยันอีเมล'
  }
})
```

## 9. User Profile Page

```vue
<!-- pages/profile.vue -->
<template>
  <div class="profile-page">
    <div class="profile-header">
      <div class="avatar-section">
        <img
          :src="user?.avatar || '/default-avatar.png'"
          :alt="user?.name"
          class="avatar"
        />
        <div class="avatar-upload">
          <label for="avatar-input" class="btn-change-avatar">เปลี่ยนรูป</label>
          <input
            id="avatar-input"
            type="file"
            accept="image/*"
            @change="updateAvatar"
            style="display: none"
          />
        </div>
      </div>
      
      <div class="user-info">
        <h1>{{ user?.name }}</h1>
        <p>{{ user?.email }}</p>
        <span class="role-badge" :class="user?.role">{{ user?.role }}</span>
      </div>
    </div>
    
    <!-- Connected Accounts -->
    <div class="connected-accounts">
      <h2>บัญชีที่เชื่อมต่อ</h2>
      
      <div class="account-list">
        <div class="account-item" v-for="provider in providers" :key="provider.id">
          <div class="account-info">
            <span class="account-icon">{{ provider.icon }}</span>
            <div>
              <strong>{{ provider.name }}</strong>
              <p v-if="provider.connected" class="account-status connected">
                เชื่อมต่อแล้ว ({{ provider.username }})
              </p>
              <p v-else class="account-status">ยังไม่ได้เชื่อมต่อ</p>
            </div>
          </div>
          
          <div class="account-actions">
            <a
              v-if="!provider.connected"
              :href="`/auth/${provider.id}?link=true`"
              class="btn-connect"
            >
              เชื่อมต่อ
            </a>
            <button
              v-else
              @click="disconnectProvider(provider.id)"
              class="btn-disconnect"
            >
              ยกเลิกการเชื่อมต่อ
            </button>
          </div>
        </div>
      </div>
    </div>
    
    <!-- Danger Zone -->
    <div class="danger-zone">
      <h2>Danger Zone</h2>
      <button @click="confirmDeleteAccount" class="btn-danger">
        ลบบัญชี
      </button>
    </div>
  </div>
</template>

<script setup>
definePageMeta({ middleware: 'auth' })

const { user, logout, refreshSession } = useAuth()

const providers = ref([
  { id: 'google', name: 'Google', icon: '🔵', connected: false, username: null },
  { id: 'github', name: 'GitHub', icon: '⚫', connected: false, username: null },
])

// โหลดข้อมูล connected accounts
onMounted(async () => {
  const profile = await $fetch('/api/user/profile')
  providers.value.forEach(p => {
    if (profile.connectedProviders?.includes(p.id)) {
      p.connected = true
      p.username = profile[p.id + 'Username'] || ''
    }
  })
})

const updateAvatar = async (event) => {
  const file = event.target.files[0]
  if (!file) return
  
  const formData = new FormData()
  formData.append('avatar', file)
  
  const result = await $fetch('/api/user/avatar', {
    method: 'POST',
    body: formData
  })
  
  await refreshSession()
}

const disconnectProvider = async (providerId) => {
  if (!confirm(`ยืนยันการยกเลิกการเชื่อมต่อกับ ${providerId}?`)) return
  
  await $fetch(`/api/user/providers/${providerId}`, { method: 'DELETE' })
  const provider = providers.value.find(p => p.id === providerId)
  if (provider) {
    provider.connected = false
    provider.username = null
  }
}

const confirmDeleteAccount = async () => {
  if (!confirm('คุณแน่ใจหรือไม่ที่จะลบบัญชี? การกระทำนี้ไม่สามารถยกเลิกได้')) return
  
  await $fetch('/api/user/account', { method: 'DELETE' })
  await logout()
}
</script>
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **OAuth 2.0 Flow** - Authorization Code Flow ที่ปลอดภัยที่สุด
2. **Google OAuth** - ตั้งค่าและใช้งาน Google Sign-In
3. **GitHub OAuth** - ตั้งค่าและใช้งาน GitHub Sign-In
4. **nuxt-auth-utils** - Module ที่ง่ายต่อการใช้งาน
5. **Social Login UI** - Component ที่สวยงามและใช้งานได้จริง
6. **Token & Session Management** - การจัดการ session อย่างปลอดภัย
7. **Multi-provider System** - ระบบ Auth ที่รองรับหลาย provider
