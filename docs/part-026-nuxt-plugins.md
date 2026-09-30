# Part 26: Nuxt Plugins

## Nuxt Plugin คืออะไร?

Plugin ใน Nuxt คือ code ที่ทำงานก่อน Vue application เริ่มต้น (before app mounting) ใช้สำหรับ:
- ลงทะเบียน global components
- เพิ่ม global properties/methods
- ติดตั้ง third-party libraries
- ตั้งค่า directives
- Subscribe to events

### โครงสร้าง Plugin

```typescript
// plugins/my-plugin.ts
export default defineNuxtPlugin((nuxtApp) => {
  // Code ที่ทำงานก่อน app start
  
  // Optional: provide values ที่ใช้ใน app
  return {
    provide: {
      hello: (msg: string) => `Hello ${msg}!`
    }
  }
})
```

### ใช้ provided values ใน components

```vue
<script setup lang="ts">
const { $hello } = useNuxtApp()
console.log($hello('World'))  // "Hello World!"
</script>
```

---

## สร้าง Plugin

### Plugin พื้นฐาน

```typescript
// plugins/format.ts
export default defineNuxtPlugin(() => {
  return {
    provide: {
      // Format currency เป็นบาทไทย
      formatCurrency: (amount: number) => {
        return new Intl.NumberFormat('th-TH', {
          style: 'currency',
          currency: 'THB'
        }).format(amount)
      },
      
      // Format date เป็นภาษาไทย
      formatDate: (date: string | Date, format?: 'short' | 'long') => {
        const d = new Date(date)
        if (format === 'long') {
          return d.toLocaleDateString('th-TH', {
            weekday: 'long',
            year: 'numeric',
            month: 'long',
            day: 'numeric'
          })
        }
        return d.toLocaleDateString('th-TH')
      },
      
      // Truncate text
      truncate: (text: string, length: number = 100) => {
        if (text.length <= length) return text
        return text.substring(0, length) + '...'
      }
    }
  }
})
```

### Plugin กับ Vue API

```typescript
// plugins/directives.ts
export default defineNuxtPlugin((nuxtApp) => {
  // ลงทะเบียน custom directive
  nuxtApp.vueApp.directive('focus', {
    mounted(el: HTMLElement) {
      el.focus()
    }
  })
  
  nuxtApp.vueApp.directive('click-outside', {
    mounted(el: HTMLElement, binding) {
      const handler = (event: MouseEvent) => {
        if (!el.contains(event.target as Node)) {
          binding.value(event)
        }
      }
      el._clickOutsideHandler = handler
      document.addEventListener('click', handler)
    },
    unmounted(el: HTMLElement) {
      document.removeEventListener('click', el._clickOutsideHandler)
    }
  })
  
  // ลงทะเบียน global component
  nuxtApp.vueApp.component('Icon', defineComponent({
    props: { name: String, size: { type: Number, default: 24 } },
    template: `<span class="icon" :style="{ fontSize: size + 'px' }">{{ name }}</span>`
  }))
})
```

---

## Server/Client Only Plugins

### Client-only Plugin

```typescript
// plugins/analytics.client.ts
// ชื่อไฟล์ลงท้ายด้วย .client.ts = ทำงานเฉพาะ browser

export default defineNuxtPlugin(() => {
  // Google Analytics - ทำงานเฉพาะ client
  const config = useRuntimeConfig()
  
  if (config.public.gaId) {
    // โหลด GA script
    useHead({
      script: [
        {
          src: `https://www.googletagmanager.com/gtag/js?id=${config.public.gaId}`,
          async: true
        }
      ]
    })
    
    window.dataLayer = window.dataLayer || []
    function gtag(...args: any[]) { window.dataLayer.push(args) }
    gtag('js', new Date())
    gtag('config', config.public.gaId)
    
    return {
      provide: {
        gtag
      }
    }
  }
})
```

### Server-only Plugin

```typescript
// plugins/database.server.ts
// ชื่อไฟล์ลงท้ายด้วย .server.ts = ทำงานเฉพาะ server

import { PrismaClient } from '@prisma/client'

export default defineNuxtPlugin(() => {
  // สร้าง database connection บน server เท่านั้น
  const prisma = new PrismaClient()
  
  return {
    provide: {
      prisma
    }
  }
})
```

### ตรวจสอบ context ใน plugin

```typescript
// plugins/universal.ts
export default defineNuxtPlugin((nuxtApp) => {
  // ตรวจสอบว่ากำลังทำงานที่ไหน
  if (process.server) {
    console.log('Running on server')
    // Server-specific setup
  }
  
  if (process.client) {
    console.log('Running on client')
    // Client-specific setup
    
    // ดักจับ unhandled errors บน client
    window.addEventListener('unhandledrejection', (event) => {
      console.error('Unhandled promise rejection:', event.reason)
    })
  }
})
```

---

## Plugin Ordering

### ควบคุมลำดับ Plugin

```typescript
// plugins/01.setup.ts
// Nuxt เรียงลำดับ plugin ตามชื่อ (alphabetically)
// ใส่ตัวเลขนำหน้าเพื่อควบคุมลำดับ

export default defineNuxtPlugin((nuxtApp) => {
  console.log('Plugin 1: Setup')
  nuxtApp.$setup = true
})
```

```typescript
// plugins/02.auth.ts
export default defineNuxtPlugin(async (nuxtApp) => {
  // ต้องรอ plugin 01 ก่อน
  console.log('Plugin 2: Auth')
  
  // สามารถใช้ค่าจาก plugin อื่น
  // const isSetup = nuxtApp.$setup
})
```

### Parallel vs Sequential

```typescript
// plugins/heavy.ts
export default defineNuxtPlugin({
  name: 'heavy-plugin',
  parallel: true,  // ทำงานพร้อมกับ plugin อื่น (ไม่รอกัน)
  async setup(nuxtApp) {
    await heavyOperation()
  }
})
```

---

## นำเข้า Third-party Libraries

### Vue Plugin Integration

```typescript
// plugins/vuetify.ts
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'
import 'vuetify/styles'

export default defineNuxtPlugin((nuxtApp) => {
  const vuetify = createVuetify({
    components,
    directives,
    theme: {
      defaultTheme: 'light',
      themes: {
        light: {
          colors: {
            primary: '#00dc82',
            secondary: '#1a1a2e'
          }
        }
      }
    }
  })
  
  nuxtApp.vueApp.use(vuetify)
})
```

### Chart.js Integration

```typescript
// plugins/chartjs.client.ts
import {
  Chart,
  CategoryScale,
  LinearScale,
  BarElement,
  LineElement,
  PointElement,
  ArcElement,
  Title,
  Tooltip,
  Legend
} from 'chart.js'

export default defineNuxtPlugin(() => {
  Chart.register(
    CategoryScale,
    LinearScale,
    BarElement,
    LineElement,
    PointElement,
    ArcElement,
    Title,
    Tooltip,
    Legend
  )
})
```

---

## Custom $composables ผ่าน Plugin

### การ Provide Composables

```typescript
// plugins/composables.ts
export default defineNuxtPlugin(() => {
  return {
    provide: {
      // Provide toast notification
      toast: {
        success(message: string) {
          useToastStore().add({ type: 'success', message })
        },
        error(message: string) {
          useToastStore().add({ type: 'error', message })
        },
        info(message: string) {
          useToastStore().add({ type: 'info', message })
        },
        warning(message: string) {
          useToastStore().add({ type: 'warning', message })
        }
      }
    }
  }
})
```

---

## ตัวอย่าง: Axios Plugin

```typescript
// plugins/axios.ts
import axios, { type AxiosInstance } from 'axios'

export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()
  const authStore = useAuthStore()
  
  const axiosInstance: AxiosInstance = axios.create({
    baseURL: config.public.apiBase,
    timeout: 10000,
    headers: {
      'Content-Type': 'application/json'
    }
  })
  
  // Request Interceptor
  axiosInstance.interceptors.request.use(
    (requestConfig) => {
      // เพิ่ม auth token
      if (authStore.token) {
        requestConfig.headers.Authorization = `Bearer ${authStore.token}`
      }
      return requestConfig
    },
    (error) => Promise.reject(error)
  )
  
  // Response Interceptor
  axiosInstance.interceptors.response.use(
    (response) => response.data,
    async (error) => {
      const originalRequest = error.config
      
      // Token expired - try to refresh
      if (error.response?.status === 401 && !originalRequest._retry) {
        originalRequest._retry = true
        
        try {
          await authStore.refreshToken()
          originalRequest.headers.Authorization = `Bearer ${authStore.token}`
          return axiosInstance(originalRequest)
        } catch {
          await authStore.logout()
          return Promise.reject(error)
        }
      }
      
      return Promise.reject(error)
    }
  )
  
  return {
    provide: {
      axios: axiosInstance
    }
  }
})
```

---

## ตัวอย่าง: Toast Plugin สมบูรณ์

### Toast Store

```typescript
// stores/toast.ts
import { defineStore } from 'pinia'

export interface Toast {
  id: string
  type: 'success' | 'error' | 'info' | 'warning'
  message: string
  title?: string
  duration?: number
  persistent?: boolean
}

export const useToastStore = defineStore('toast', () => {
  const toasts = ref<Toast[]>([])
  
  function add(toast: Omit<Toast, 'id'>) {
    const id = Math.random().toString(36).substring(2)
    const newToast: Toast = { ...toast, id, duration: toast.duration || 4000 }
    toasts.value.push(newToast)
    
    if (!newToast.persistent) {
      setTimeout(() => remove(id), newToast.duration)
    }
    
    return id
  }
  
  function remove(id: string) {
    const index = toasts.value.findIndex(t => t.id === id)
    if (index > -1) toasts.value.splice(index, 1)
  }
  
  function clear() {
    toasts.value = []
  }
  
  return { toasts, add, remove, clear }
})
```

### Toast Plugin

```typescript
// plugins/toast.ts
export default defineNuxtPlugin(() => {
  const store = useToastStore()
  
  const toast = {
    success(message: string, title?: string) {
      return store.add({ type: 'success', message, title })
    },
    error(message: string, title?: string) {
      return store.add({ type: 'error', message, title, duration: 6000 })
    },
    info(message: string, title?: string) {
      return store.add({ type: 'info', message, title })
    },
    warning(message: string, title?: string) {
      return store.add({ type: 'warning', message, title })
    },
    promise<T>(
      promise: Promise<T>,
      messages: { loading: string; success: string; error: string }
    ) {
      const id = store.add({
        type: 'info',
        message: messages.loading,
        persistent: true
      })
      
      return promise
        .then((result) => {
          store.remove(id)
          store.add({ type: 'success', message: messages.success })
          return result
        })
        .catch((error) => {
          store.remove(id)
          store.add({ type: 'error', message: messages.error })
          throw error
        })
    }
  }
  
  return {
    provide: { toast }
  }
})
```

### Toast Component

```vue
<!-- components/AppToast.vue -->
<script setup lang="ts">
const toastStore = useToastStore()

const icons = {
  success: '✅',
  error: '❌',
  info: 'ℹ️',
  warning: '⚠️'
}

const colors = {
  success: '#10b981',
  error: '#ef4444',
  info: '#3b82f6',
  warning: '#f59e0b'
}
</script>

<template>
  <Teleport to="body">
    <div class="toast-container">
      <TransitionGroup name="toast" tag="div">
        <div
          v-for="toast in toastStore.toasts"
          :key="toast.id"
          class="toast"
          :style="{ borderLeftColor: colors[toast.type] }"
        >
          <span class="toast-icon">{{ icons[toast.type] }}</span>
          <div class="toast-content">
            <strong v-if="toast.title">{{ toast.title }}</strong>
            <p>{{ toast.message }}</p>
          </div>
          <button @click="toastStore.remove(toast.id)" class="toast-close">×</button>
        </div>
      </TransitionGroup>
    </div>
  </Teleport>
</template>

<style scoped>
.toast-container {
  position: fixed;
  top: 1rem;
  right: 1rem;
  z-index: 9999;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  max-width: 400px;
}

.toast {
  background: white;
  border-radius: 8px;
  border-left: 4px solid;
  padding: 0.75rem 1rem;
  display: flex;
  align-items: flex-start;
  gap: 0.75rem;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.toast-content p { margin: 0; font-size: 0.875rem; }
.toast-close {
  margin-left: auto;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1.25rem;
  color: #9ca3af;
}

/* Transitions */
.toast-enter-active { transition: all 0.3s ease; }
.toast-leave-active { transition: all 0.3s ease; }
.toast-enter-from { transform: translateX(100%); opacity: 0; }
.toast-leave-to { transform: translateX(100%); opacity: 0; }
</style>
```

### ใช้ Toast Plugin

```vue
<script setup lang="ts">
const { $toast } = useNuxtApp()

async function saveUser(data: any) {
  try {
    await $toast.promise(
      $fetch('/api/users', { method: 'POST', body: data }),
      {
        loading: 'กำลังบันทึก...',
        success: 'บันทึกสำเร็จ!',
        error: 'เกิดข้อผิดพลาดในการบันทึก'
      }
    )
  } catch {}
}

function showNotifications() {
  $toast.success('บันทึกข้อมูลสำเร็จ')
  $toast.error('เกิดข้อผิดพลาด', 'Error Title')
  $toast.info('มีการอัพเดตใหม่')
  $toast.warning('Session ใกล้หมดอายุ')
}
</script>
```

---

## ตัวอย่าง: Analytics Plugin

```typescript
// plugins/analytics.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()
  const router = useRouter()
  
  // Track pageviews ทุกครั้งที่ navigate
  router.afterEach((to) => {
    if (window.gtag) {
      window.gtag('event', 'page_view', {
        page_title: document.title,
        page_location: window.location.href,
        page_path: to.path
      })
    }
  })
  
  return {
    provide: {
      analytics: {
        track(event: string, params?: Record<string, any>) {
          if (window.gtag) {
            window.gtag('event', event, params)
          }
          // ส่งไป analytics service อื่นด้วย
          $fetch('/api/analytics/event', {
            method: 'POST',
            body: { event, params, timestamp: Date.now() }
          }).catch(() => {}) // silent fail
        },
        
        trackClick(element: string, metadata?: any) {
          this.track('click', { element, ...metadata })
        },
        
        trackSearch(query: string, results: number) {
          this.track('search', { query, results })
        },
        
        setUser(userId: string, properties?: any) {
          if (window.gtag) {
            window.gtag('config', config.public.gaId, {
              user_id: userId,
              user_properties: properties
            })
          }
        }
      }
    }
  }
})
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Plugin คืออะไร** - Code ที่ทำงานก่อน app เริ่ม
2. **สร้าง Plugin** - ใช้ `defineNuxtPlugin()`
3. **Server/Client Plugins** - `.server.ts` และ `.client.ts`
4. **Plugin Ordering** - ควบคุมลำดับด้วยชื่อไฟล์
5. **Third-party Libraries** - ติดตั้ง Vue plugins
6. **Custom $composables** - Provide values ผ่าน plugin
7. **Axios Plugin** - HTTP client พร้อม interceptors
8. **Toast Plugin** - Notification system
9. **Analytics Plugin** - Event tracking

**ถัดไป**: Part 27 - Nuxt Modules
