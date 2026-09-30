# Part 13: Provide / Inject

## บทนำ

Provide/Inject เป็น mechanism ใน Vue.js สำหรับส่งข้อมูลจาก ancestor component ไปยัง descendant component โดยไม่ต้องผ่าน props ทุกชั้น (แก้ปัญหา "Prop Drilling")

```
App (provide: theme)
  └── Layout
       └── Sidebar
            └── MenuItem ← inject: theme (ได้รับโดยตรง ไม่ต้องผ่าน Layout, Sidebar)
```

---

## 1. Provide/Inject พื้นฐาน

### การใช้ provide() ใน Parent

```vue
<!-- components/ParentComponent.vue -->
<template>
  <div>
    <h2>Parent Component</h2>
    <ChildComponent />
  </div>
</template>

<script setup>
import { provide, ref } from 'vue'
import ChildComponent from './ChildComponent.vue'

// ส่งค่าแบบ static
provide('appName', 'Vue Course App')

// ส่งค่าแบบ reactive
const count = ref(0)
provide('count', count)

// ส่งฟังก์ชัน
function increment() {
  count.value++
}
provide('incrementCount', increment)

// ส่งหลายค่าพร้อมกัน (object)
provide('userConfig', {
  language: 'th',
  timezone: 'Asia/Bangkok',
  currency: 'THB'
})
</script>
```

### การใช้ inject() ใน Child

```vue
<!-- components/DeepChildComponent.vue -->
<template>
  <div>
    <h4>Deep Child Component</h4>
    <p>ชื่อแอป: {{ appName }}</p>
    <p>นับ: {{ count }}</p>
    <button @click="incrementCount">เพิ่ม</button>
    <p>ภาษา: {{ userConfig.language }}</p>
    <p>เขตเวลา: {{ userConfig.timezone }}</p>
  </div>
</template>

<script setup>
import { inject } from 'vue'

// รับค่าที่ถูก provide ไว้
const appName = inject('appName')
const count = inject('count')
const incrementCount = inject('incrementCount')
const userConfig = inject('userConfig')
</script>
```

---

## 2. Provide ที่ App Level

การ provide ที่ระดับ App ทำให้ทุก component ในแอปสามารถ inject ได้

```javascript
// main.js
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Provide ที่ app level - ใช้ได้ทุก component
app.provide('apiUrl', 'https://api.example.com')
app.provide('appVersion', '1.0.0')
app.provide('maxUploadSize', 5 * 1024 * 1024) // 5MB

app.mount('#app')
```

```vue
<!-- components/AnyComponent.vue - ใช้ได้จากทุก component -->
<template>
  <div>
    <p>API URL: {{ apiUrl }}</p>
    <p>Version: {{ appVersion }}</p>
  </div>
</template>

<script setup>
import { inject } from 'vue'

const apiUrl = inject('apiUrl')
const appVersion = inject('appVersion')
</script>
```

---

## 3. Inject with Default Values

เมื่อ inject ค่าที่ไม่แน่ใจว่าจะมีหรือไม่ ควรใส่ default value

```vue
<script setup>
import { inject, ref } from 'vue'

// inject พร้อม default value
const theme = inject('theme', 'light')

// inject พร้อม default function (lazy evaluation - ดีกว่าสำหรับ object/array)
const config = inject('config', () => ({
  fontSize: 16,
  language: 'th'
}), true) // true = treat second arg as factory function

// inject reactive ref พร้อม default
const user = inject('currentUser', ref(null))

// inject ฟังก์ชัน พร้อม no-op default
const logout = inject('logout', () => {
  console.warn('logout function not provided')
})
</script>
```

---

## 4. Reactive Provide/Inject

เพื่อให้ child component รับรู้การเปลี่ยนแปลงข้อมูล ต้อง provide ค่าที่เป็น reactive

```vue
<!-- components/ThemeProvider.vue -->
<template>
  <div :class="['theme-container', `theme-${currentTheme}`]">
    <slot></slot>
  </div>
</template>

<script setup>
import { provide, ref, computed, readonly } from 'vue'

const currentTheme = ref('light')
const fontSize = ref(16)

// computed values
const isDark = computed(() => currentTheme.value === 'dark')

// ฟังก์ชัน mutations
function toggleTheme() {
  currentTheme.value = currentTheme.value === 'light' ? 'dark' : 'light'
}

function setFontSize(size) {
  fontSize.value = Math.max(12, Math.min(24, size)) // clamp 12-24
}

// ✅ ใช้ readonly() เพื่อป้องกัน child แก้ไขโดยตรง
provide('theme', readonly(currentTheme))
provide('isDark', isDark)
provide('fontSize', readonly(fontSize))

// ส่งฟังก์ชันสำหรับ mutations แทน
provide('toggleTheme', toggleTheme)
provide('setFontSize', setFontSize)
</script>
```

### Child Component รับ Reactive Values

```vue
<!-- components/ThemeToggle.vue -->
<template>
  <div>
    <button @click="toggleTheme">
      {{ isDark ? '☀️ โหมดสว่าง' : '🌙 โหมดมืด' }}
    </button>
    <span>Theme: {{ theme }}</span>
  </div>
</template>

<script setup>
import { inject } from 'vue'

// รับ reactive values
const theme = inject('theme')
const isDark = inject('isDark')

// รับ mutation functions
const toggleTheme = inject('toggleTheme')
</script>
```

```vue
<!-- components/FontSizeControl.vue -->
<template>
  <div class="font-control">
    <button @click="setFontSize(fontSize - 1)">A-</button>
    <span :style="{ fontSize: fontSize + 'px' }">ขนาดตัวอักษร: {{ fontSize }}px</span>
    <button @click="setFontSize(fontSize + 1)">A+</button>
  </div>
</template>

<script setup>
import { inject } from 'vue'

const fontSize = inject('fontSize')
const setFontSize = inject('setFontSize')
</script>
```

---

## 5. Symbols as Injection Keys

การใช้ Symbol เป็น injection key ช่วยป้องกันการชนกันของชื่อ และให้ TypeScript support ที่ดีกว่า

```javascript
// keys.js - ไฟล์รวม injection keys
export const ThemeKey = Symbol('theme')
export const AuthKey = Symbol('auth')
export const RouterKey = Symbol('router')
export const StoreKey = Symbol('store')
```

```vue
<!-- providers/AuthProvider.vue -->
<script setup>
import { provide, reactive } from 'vue'
import { AuthKey } from '../keys.js'

const auth = reactive({
  user: null,
  isAuthenticated: false,
  isLoading: false
})

async function login(credentials) {
  auth.isLoading = true
  try {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    })
    
    if (!response.ok) throw new Error('Login failed')
    
    const data = await response.json()
    auth.user = data.user
    auth.isAuthenticated = true
    
    // บันทึก token
    localStorage.setItem('token', data.token)
    
    return data
  } finally {
    auth.isLoading = false
  }
}

async function logout() {
  await fetch('/api/auth/logout', { method: 'POST' })
  auth.user = null
  auth.isAuthenticated = false
  localStorage.removeItem('token')
}

// ใช้ Symbol เป็น key
provide(AuthKey, {
  auth,
  login,
  logout
})
</script>

<template>
  <slot></slot>
</template>
```

```vue
<!-- components/LoginButton.vue -->
<script setup>
import { inject } from 'vue'
import { AuthKey } from '../keys.js'

// inject ด้วย Symbol key
const { auth, login, logout } = inject(AuthKey)
</script>

<template>
  <div>
    <div v-if="auth.isAuthenticated">
      <span>สวัสดี, {{ auth.user?.name }}</span>
      <button @click="logout">ออกจากระบบ</button>
    </div>
    <div v-else>
      <button @click="login({ email: 'user@example.com', password: '1234' })">
        เข้าสู่ระบบ
      </button>
    </div>
  </div>
</template>
```

---

## 6. ตัวอย่างจริง: Theme System

```javascript
// composables/useTheme.js
import { inject, provide, ref, computed, watch } from 'vue'

// Symbol key
export const THEME_KEY = Symbol('theme')

// Default theme config
const defaultTheme = {
  primary: '#4CAF50',
  secondary: '#2196F3',
  danger: '#F44336',
  background: '#ffffff',
  surface: '#f5f5f5',
  text: '#333333',
  border: '#dddddd'
}

const darkTheme = {
  primary: '#66BB6A',
  secondary: '#42A5F5',
  danger: '#EF5350',
  background: '#1a1a2e',
  surface: '#16213e',
  text: '#e0e0e0',
  border: '#444444'
}

// สร้าง theme provider
export function createThemeProvider() {
  const mode = ref('light')
  const customColors = ref({})
  
  const colors = computed(() => {
    const base = mode.value === 'dark' ? darkTheme : defaultTheme
    return { ...base, ...customColors.value }
  })
  
  const isDark = computed(() => mode.value === 'dark')
  
  function setMode(newMode) {
    mode.value = newMode
    localStorage.setItem('theme-mode', newMode)
  }
  
  function toggleMode() {
    setMode(mode.value === 'light' ? 'dark' : 'light')
  }
  
  function setCustomColor(key, value) {
    customColors.value = { ...customColors.value, [key]: value }
  }
  
  // Apply CSS variables
  watch(colors, (newColors) => {
    const root = document.documentElement
    Object.entries(newColors).forEach(([key, value]) => {
      root.style.setProperty(`--color-${key}`, value)
    })
  }, { immediate: true })
  
  // Load saved preference
  const savedMode = localStorage.getItem('theme-mode')
  if (savedMode) mode.value = savedMode
  
  const themeContext = {
    mode,
    colors,
    isDark,
    setMode,
    toggleMode,
    setCustomColor
  }
  
  provide(THEME_KEY, themeContext)
  
  return themeContext
}

// สำหรับ inject ใน child components
export function useTheme() {
  const theme = inject(THEME_KEY)
  if (!theme) {
    throw new Error('useTheme() must be used inside ThemeProvider')
  }
  return theme
}
```

```vue
<!-- App.vue - Setup ThemeProvider -->
<template>
  <div :class="['app', isDark ? 'dark' : 'light']">
    <TheHeader />
    <TheMain />
    <TheFooter />
  </div>
</template>

<script setup>
import { createThemeProvider } from './composables/useTheme.js'
import TheHeader from './components/TheHeader.vue'
import TheMain from './components/TheMain.vue'
import TheFooter from './components/TheFooter.vue'

// สร้าง theme context
const { isDark } = createThemeProvider()
</script>

<style>
.app {
  background-color: var(--color-background);
  color: var(--color-text);
  min-height: 100vh;
  transition: all 0.3s ease;
}
</style>
```

```vue
<!-- components/TheHeader.vue -->
<template>
  <header :style="{ borderBottom: `2px solid ${colors.primary}` }">
    <nav>
      <a href="/" class="logo">Vue Course</a>
      <div class="nav-links">
        <a href="/courses">คอร์สเรียน</a>
        <a href="/blog">บทความ</a>
      </div>
      <button @click="toggleMode" class="theme-toggle">
        {{ isDark ? '☀️' : '🌙' }}
      </button>
    </nav>
  </header>
</template>

<script setup>
import { useTheme } from '../composables/useTheme.js'

// inject theme context
const { colors, isDark, toggleMode } = useTheme()
</script>
```

---

## 7. ตัวอย่างจริง: Auth Context

```javascript
// composables/useAuth.js
import { provide, inject, ref, computed } from 'vue'

export const AUTH_KEY = Symbol('auth')

export function createAuthContext() {
  const token = ref(localStorage.getItem('auth_token'))
  const user = ref(null)
  const isLoading = ref(false)
  
  const isAuthenticated = computed(() => !!token.value && !!user.value)
  const isAdmin = computed(() => user.value?.role === 'admin')
  
  // Initialize - load user if token exists
  async function initialize() {
    if (!token.value) return
    
    try {
      isLoading.value = true
      const response = await fetch('/api/auth/me', {
        headers: { Authorization: `Bearer ${token.value}` }
      })
      
      if (response.ok) {
        user.value = await response.json()
      } else {
        // Token หมดอายุ
        clearAuth()
      }
    } catch {
      clearAuth()
    } finally {
      isLoading.value = false
    }
  }
  
  async function login(email, password) {
    isLoading.value = true
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password })
      })
      
      if (!response.ok) {
        const error = await response.json()
        throw new Error(error.message || 'เข้าสู่ระบบล้มเหลว')
      }
      
      const data = await response.json()
      token.value = data.token
      user.value = data.user
      localStorage.setItem('auth_token', data.token)
      
      return { success: true }
    } finally {
      isLoading.value = false
    }
  }
  
  async function register(userData) {
    isLoading.value = true
    try {
      const response = await fetch('/api/auth/register', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(userData)
      })
      
      if (!response.ok) {
        const error = await response.json()
        throw new Error(error.message)
      }
      
      const data = await response.json()
      token.value = data.token
      user.value = data.user
      localStorage.setItem('auth_token', data.token)
      
      return { success: true }
    } finally {
      isLoading.value = false
    }
  }
  
  function clearAuth() {
    token.value = null
    user.value = null
    localStorage.removeItem('auth_token')
  }
  
  async function logout() {
    try {
      await fetch('/api/auth/logout', {
        method: 'POST',
        headers: { Authorization: `Bearer ${token.value}` }
      })
    } finally {
      clearAuth()
    }
  }
  
  const authContext = {
    user,
    token,
    isLoading,
    isAuthenticated,
    isAdmin,
    initialize,
    login,
    register,
    logout
  }
  
  provide(AUTH_KEY, authContext)
  
  return authContext
}

export function useAuth() {
  const auth = inject(AUTH_KEY)
  if (!auth) {
    throw new Error('useAuth() must be called inside AuthProvider')
  }
  return auth
}
```

```vue
<!-- App.vue -->
<template>
  <div>
    <TheNavbar />
    <router-view />
  </div>
</template>

<script setup>
import { onMounted } from 'vue'
import { createAuthContext } from './composables/useAuth.js'
import TheNavbar from './components/TheNavbar.vue'

// สร้าง auth context ที่ app level
const { initialize } = createAuthContext()

// โหลดข้อมูล user เมื่อเปิดแอป
onMounted(initialize)
</script>
```

```vue
<!-- components/LoginForm.vue -->
<template>
  <form @submit.prevent="handleLogin" class="login-form">
    <h2>เข้าสู่ระบบ</h2>

    <div v-if="errorMessage" class="error-alert">
      {{ errorMessage }}
    </div>

    <div class="form-group">
      <label>อีเมล</label>
      <input v-model="form.email" type="email" required />
    </div>

    <div class="form-group">
      <label>รหัสผ่าน</label>
      <input v-model="form.password" type="password" required />
    </div>

    <button type="submit" :disabled="isLoading">
      {{ isLoading ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ' }}
    </button>
  </form>
</template>

<script setup>
import { ref } from 'vue'
import { useAuth } from '../composables/useAuth.js'
import { useRouter } from 'vue-router'

const { login, isLoading } = useAuth()
const router = useRouter()

const form = ref({ email: '', password: '' })
const errorMessage = ref('')

async function handleLogin() {
  errorMessage.value = ''
  try {
    await login(form.value.email, form.value.password)
    router.push('/')
  } catch (error) {
    errorMessage.value = error.message
  }
}
</script>
```

---

## 8. ตัวอย่างจริง: Locale System (i18n)

```javascript
// composables/useLocale.js
import { provide, inject, ref, computed } from 'vue'

export const LOCALE_KEY = Symbol('locale')

// Translation dictionaries
const translations = {
  th: {
    common: {
      save: 'บันทึก',
      cancel: 'ยกเลิก',
      delete: 'ลบ',
      edit: 'แก้ไข',
      loading: 'กำลังโหลด...',
      error: 'เกิดข้อผิดพลาด',
      success: 'สำเร็จ'
    },
    nav: {
      home: 'หน้าแรก',
      about: 'เกี่ยวกับ',
      courses: 'คอร์สเรียน',
      contact: 'ติดต่อ'
    },
    auth: {
      login: 'เข้าสู่ระบบ',
      logout: 'ออกจากระบบ',
      register: 'สมัครสมาชิก',
      email: 'อีเมล',
      password: 'รหัสผ่าน',
      forgotPassword: 'ลืมรหัสผ่าน?'
    }
  },
  en: {
    common: {
      save: 'Save',
      cancel: 'Cancel',
      delete: 'Delete',
      edit: 'Edit',
      loading: 'Loading...',
      error: 'An error occurred',
      success: 'Success'
    },
    nav: {
      home: 'Home',
      about: 'About',
      courses: 'Courses',
      contact: 'Contact'
    },
    auth: {
      login: 'Login',
      logout: 'Logout',
      register: 'Register',
      email: 'Email',
      password: 'Password',
      forgotPassword: 'Forgot password?'
    }
  }
}

export function createLocaleContext() {
  const locale = ref(localStorage.getItem('locale') || 'th')
  
  const t = computed(() => (key) => {
    const keys = key.split('.')
    let value = translations[locale.value]
    
    for (const k of keys) {
      value = value?.[k]
    }
    
    return value || key
  })
  
  function setLocale(newLocale) {
    if (translations[newLocale]) {
      locale.value = newLocale
      localStorage.setItem('locale', newLocale)
      document.documentElement.lang = newLocale
    }
  }
  
  const availableLocales = computed(() => Object.keys(translations))
  
  const localeContext = {
    locale,
    t,
    setLocale,
    availableLocales
  }
  
  provide(LOCALE_KEY, localeContext)
  
  return localeContext
}

export function useLocale() {
  const locale = inject(LOCALE_KEY)
  if (!locale) {
    throw new Error('useLocale() must be used inside LocaleProvider')
  }
  return locale
}
```

```vue
<!-- components/LocaleSwitch.vue -->
<template>
  <div class="locale-switch">
    <button
      v-for="loc in availableLocales"
      :key="loc"
      :class="['locale-btn', { active: locale === loc }]"
      @click="setLocale(loc)"
    >
      {{ loc === 'th' ? '🇹🇭 ไทย' : '🇬🇧 English' }}
    </button>
  </div>
</template>

<script setup>
import { useLocale } from '../composables/useLocale.js'

const { locale, setLocale, availableLocales } = useLocale()
</script>
```

```vue
<!-- components/TheNavbar.vue -->
<template>
  <nav class="navbar">
    <a href="/">{{ t('nav.home') }}</a>
    <a href="/courses">{{ t('nav.courses') }}</a>
    <a href="/about">{{ t('nav.about') }}</a>
    <a href="/contact">{{ t('nav.contact') }}</a>
    <LocaleSwitch />
  </nav>
</template>

<script setup>
import { useLocale } from '../composables/useLocale.js'
import LocaleSwitch from './LocaleSwitch.vue'

const { t } = useLocale()
</script>
```

---

## สรุป

| เมื่อไหร่ควรใช้ Provide/Inject | เมื่อไหร่ไม่ควรใช้ |
|-------------------------------|-------------------|
| Theme / UI config ทั้งแอป | ส่งข้อมูลระหว่าง sibling components |
| Auth context | เมื่อ component tree ไม่ลึกมาก |
| Locale / i18n | State management ซับซ้อน (ใช้ Pinia แทน) |
| Plugin-like functionality | |

### Pattern ที่ดี

```javascript
// ✅ ให้ readonly กับ state
provide('count', readonly(count))

// ✅ ส่งฟังก์ชัน mutations แยก
provide('setCount', (val) => { count.value = val })

// ✅ ใช้ Symbol เป็น key เพื่อป้องกัน collision
export const MY_KEY = Symbol('myKey')
provide(MY_KEY, data)

// ✅ สร้าง composable สำหรับ inject
export function useMyFeature() {
  const ctx = inject(MY_KEY)
  if (!ctx) throw new Error('Must be inside Provider')
  return ctx
}
```
