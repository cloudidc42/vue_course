# Part 20: Vue DevTools

## บทนำ

Vue DevTools เป็น browser extension ที่ช่วยให้เราสามารถ debug, inspect และ profile Vue.js applications ได้อย่างมีประสิทธิภาพ

---

## 1. ติดตั้ง Vue DevTools

### Browser Extension

**Chrome:**
1. เปิด Chrome Web Store: https://chrome.google.com/webstore/
2. ค้นหา "Vue DevTools"
3. คลิก "Add to Chrome"

**Firefox:**
1. เปิด Firefox Add-ons: https://addons.mozilla.org/
2. ค้นหา "Vue DevTools"
3. คลิก "Add to Firefox"

### Standalone App (ไม่ต้องใช้ browser extension)

```bash
npm install -g @vue/devtools
vue-devtools
```

```javascript
// main.js - เชื่อมต่อ standalone devtools
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// ในกรณีใช้ standalone devtools
if (process.env.NODE_ENV === 'development') {
  import('@vue/devtools').then(devtools => {
    devtools.connect()
  })
}

app.mount('#app')
```

### Vite Plugin DevTools (Vue 3.4+)

```bash
npm install vite-plugin-vue-devtools -D
```

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import VueDevTools from 'vite-plugin-vue-devtools'

export default defineConfig({
  plugins: [
    vue(),
    VueDevTools() // เพิ่ม DevTools panel ใน browser
  ]
})
```

---

## 2. Component Inspector

### การ Inspect Component

```
DevTools Panel > Components Tab

ทำได้:
- ดู component tree ทั้งหมด
- เลือก component เพื่อดู state
- แก้ไข props/data แบบ live
- ไฮไลท์ component ใน DOM
```

### การเพิ่ม Component Name สำหรับ Debug

```vue
<!-- ✅ กำหนดชื่อ component ที่ชัดเจน -->
<script>
export default {
  name: 'UserProfileCard' // ชื่อนี้จะแสดงใน DevTools
}
</script>

<!-- Composition API - ชื่อจากชื่อไฟล์ -->
<!-- UserProfileCard.vue → จะแสดงเป็น "UserProfileCard" -->
<script setup>
// ตั้งชื่อแบบ explicit
defineOptions({
  name: 'UserProfileCard'
})
</script>
```

### Custom Inspector

```javascript
// เพิ่มข้อมูลใน DevTools Inspector
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()

// เพิ่ม custom data ที่จะแสดงใน DevTools
if (instance) {
  instance.appContext.config.globalProperties.$devtools = {
    // custom data
  }
}
```

---

## 3. State Inspector

### ดู Pinia State ใน DevTools

```javascript
// stores/myStore.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useMyStore = defineStore('myStore', () => {
  const user = ref({ name: 'สมชาย', role: 'admin' })
  const items = ref([1, 2, 3])
  const isLoading = ref(false)
  
  const itemCount = computed(() => items.value.length)
  
  function addItem(item) {
    items.value.push(item)
  }
  
  return { user, items, isLoading, itemCount, addItem }
})

// ใน DevTools จะเห็น:
// Stores > myStore
//   user: { name: "สมชาย", role: "admin" }
//   items: [1, 2, 3]
//   isLoading: false
//   itemCount: 3 (getter)
```

### Devtools API - Custom State Tracking

```javascript
// การใช้ Devtools API เพื่อ custom inspector
import { setupDevtoolsPlugin } from '@vue/devtools-api'

setupDevtoolsPlugin({
  id: 'my-plugin',
  label: 'My Plugin',
  packageName: 'my-package',
  homepage: 'https://example.com',
  app
}, (api) => {
  // เพิ่ม custom inspector
  api.addInspector({
    id: 'my-inspector',
    label: 'My Inspector',
    icon: 'storage'
  })
  
  // ตอบสนองต่อ inspector requests
  api.on.getInspectorTree((payload) => {
    if (payload.inspectorId === 'my-inspector') {
      payload.rootNodes = [
        {
          id: 'root',
          label: 'My Data',
          children: [
            { id: 'state', label: 'State' },
            { id: 'actions', label: 'Actions' }
          ]
        }
      ]
    }
  })
})
```

---

## 4. Timeline และ Events

Timeline แสดงลำดับ events ที่เกิดขึ้นใน application

```javascript
// Emit custom events ไปยัง Timeline
import { getCurrentInstance } from 'vue'

export function useAnalytics() {
  const instance = getCurrentInstance()
  
  function trackEvent(eventName, data) {
    // Log ไปยัง DevTools Timeline
    if (process.env.NODE_ENV === 'development') {
      instance?.appContext.config.globalProperties.$devtools?.emit(
        eventName, data
      )
    }
    
    // Production analytics
    window.gtag?.('event', eventName, data)
  }
  
  return { trackEvent }
}
```

### ดู Vue Router Events ใน Timeline

```javascript
// router/index.js
const router = createRouter({ ... })

// Events เหล่านี้จะแสดงใน DevTools Timeline
router.beforeEach((to, from) => {
  console.log('🔵 Navigate:', from.path, '→', to.path)
})

router.afterEach((to) => {
  console.log('✅ Navigated to:', to.path)
})

router.onError((error) => {
  console.error('❌ Router error:', error)
})
```

---

## 5. Performance Profiling

### ใช้ Vue DevTools Performance Tab

```vue
<!-- ทดสอบ performance -->
<template>
  <div>
    <!-- Component ที่มี heavy computation -->
    <div v-for="item in expensiveList" :key="item.id">
      {{ item.processedValue }}
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const data = ref(/* large array */)

// ❌ ไม่ดี - คำนวณทุก render
const badList = data.value.map(item => heavyProcess(item))

// ✅ ดีกว่า - cache ด้วย computed
const expensiveList = computed(() =>
  data.value.map(item => ({
    id: item.id,
    processedValue: heavyProcess(item)
  }))
)

function heavyProcess(item) {
  // complex calculation...
  return item.value * 2
}
</script>
```

### Performance Tips จาก DevTools

```javascript
// 1. ใช้ v-memo สำหรับ static content
// <div v-memo="[item.id]">...</div>

// 2. ใช้ shallowRef สำหรับ large objects
import { shallowRef, triggerRef } from 'vue'

const largeData = shallowRef({ /* huge object */ })

function updateData() {
  largeData.value.someProperty = 'new value'
  triggerRef(largeData) // manual trigger
}

// 3. ใช้ defineAsyncComponent
import { defineAsyncComponent } from 'vue'

const HeavyComponent = defineAsyncComponent(() =>
  import('./HeavyComponent.vue')
)

// 4. Virtual scrolling สำหรับ long lists
// ใช้ @vueuse/virtual-list หรือ vue-virtual-scroller
```

---

## 6. ใช้ DevTools กับ Pinia

```javascript
// stores/debug.js - Store สำหรับ debug
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useDebugStore = defineStore('debug', () => {
  const logs = ref([])
  const startTime = ref(Date.now())
  
  function log(message, data = {}) {
    logs.value.push({
      time: Date.now() - startTime.value,
      message,
      data,
      timestamp: new Date().toISOString()
    })
  }
  
  function clearLogs() {
    logs.value = []
  }
  
  return { logs, log, clearLogs }
})
```

### Pinia State ใน DevTools

```
DevTools Panel > Pinia Tab

สิ่งที่เห็น:
├── Stores
│   ├── counter
│   │   ├── count: 42
│   │   └── history: [...]
│   ├── auth
│   │   ├── user: { name: "สมชาย" }
│   │   └── token: "eyJ..."
│   └── cart
│       ├── items: [...]
│       └── total: 1500

Actions ที่เรียก:
├── counter/increment (2ms)
├── auth/login (245ms)
└── cart/addItem (1ms)
```

---

## 7. ใช้ DevTools กับ Vue Router

```javascript
// Vue Router จะแสดงข้อมูลใน DevTools อัตโนมัติ

// ดู Route ปัจจุบัน:
// DevTools > Router Tab
//   Current Route:
//     path: "/dashboard/profile"
//     name: "dashboard-profile"
//     params: {}
//     query: { tab: "info" }
//     meta: { requiresAuth: true }

// Route History จะแสดงการ navigate ทั้งหมด
```

---

## 8. Tips and Tricks

### Console Shortcuts

```javascript
// ใน Browser Console สามารถใช้ shortcut เหล่านี้:

// เข้าถึง Vue component ที่ inspect อยู่
$vm  // ใน Vue DevTools → คลิก component → กด [</>]

// ตัวอย่าง
$vm.count = 99    // แก้ไข state
$vm.increment()   // เรียก method
console.log($vm.$data) // ดู data

// เข้าถึง Pinia store
$pinia  // Pinia instance
$pinia._s  // Map ของ stores ทั้งหมด
$pinia._s.get('cart').items  // เข้าถึง cart store
```

### Custom Formatters

```javascript
// main.js - เพิ่ม custom formatter สำหรับ console
if (process.env.NODE_ENV === 'development') {
  window.devtools_formatters = [
    {
      header(object) {
        if (object instanceof Date) {
          return ['span', {style: 'color: blue'}, `📅 ${object.toLocaleDateString('th-TH')}`]
        }
        return null
      },
      body(object) {
        return null
      },
      hasBody(object) {
        return false
      }
    }
  ]
}
```

### Vue App Config

```javascript
// ตั้งค่า performance tracking
const app = createApp(App)

// เปิด performance tracking
app.config.performance = true

// กำหนด error handler
app.config.errorHandler = (err, instance, info) => {
  console.error('Vue Error:', err, info)
  // ส่ง error ไปยัง error tracking service
}

// กำหนด warn handler
app.config.warnHandler = (msg, instance, trace) => {
  console.warn('Vue Warning:', msg)
}
```

### ใช้ inject ใน DevTools Console

```javascript
// เข้าถึง router และ pinia จาก component ใน console
// 1. คลิก component ใน DevTools
// 2. กด icon [</>] เพื่อเข้า console context
// 3. ใช้ $vm

$vm.$router.push('/dashboard')      // navigate
$vm.$route.params                    // ดู params

// หรือเข้าถึงผ่าน window (ต้อง expose เอง)
// main.js
if (import.meta.env.DEV) {
  const { useAuthStore } = await import('./stores/auth')
  const { useCartStore } = await import('./stores/cart')
  
  window.__AUTH__ = useAuthStore()
  window.__CART__ = useCartStore()
  window.__ROUTER__ = router
}

// ใน console:
// __AUTH__.login({ email: 'test@test.com', password: '1234' })
// __CART__.addItem({ id: 1, name: 'test', price: 100 })
```

---

## สรุป

| ฟีเจอร์ | ใช้สำหรับ |
|--------|-----------|
| Component Tab | Inspect component tree และ props/state |
| Pinia Tab | Monitor และแก้ไข store state |
| Router Tab | ดู route history และ navigation |
| Timeline Tab | Debug events และ performance |
| Performance Tab | Profiling render performance |

### Keyboard Shortcuts ใน DevTools

```
F12 หรือ Ctrl+Shift+I  → เปิด DevTools
Ctrl+Shift+J           → เปิด Console
Alt+Shift+I            → เปิด Vue DevTools panel

ใน Vue DevTools Panel:
Ctrl+F / Cmd+F         → Search components
```

### Development vs Production

```javascript
// DevTools ทำงานเฉพาะใน development mode
// Production build จะ disable DevTools อัตโนมัติ

// ถ้าต้องการ enable ใน production (ไม่แนะนำ):
app.config.devtools = true

// ตรวจสอบ environment
console.log(import.meta.env.DEV)      // true ใน development
console.log(import.meta.env.PROD)     // true ใน production
console.log(import.meta.env.MODE)     // 'development' หรือ 'production'
```
