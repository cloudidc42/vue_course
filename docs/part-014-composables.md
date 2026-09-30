# Part 14: Composables (Custom Hooks)

## บทนำ

Composables คือ functions ที่ใช้ Vue Composition API เพื่อ encapsulate และ reuse stateful logic ระหว่าง components แนวคิดคล้าย Custom Hooks ใน React

---

## 1. Composable คืออะไร

Composable คือฟังก์ชันที่:
- ชื่อขึ้นต้นด้วย `use` (convention)
- ใช้ Composition API ภายใน (ref, computed, watch, lifecycle hooks)
- return ค่าที่ component ต้องการ
- สามารถนำกลับมาใช้ซ้ำได้

```
❌ แบบ Options API (ทำซ้ำใน mixin ซึ่งมีปัญหา)
✅ แบบ Composable (แยกชัดเจน ใช้ซ้ำได้ test ง่าย)
```

---

## 2. สร้าง Composable ครั้งแรก

### ตัวอย่าง: useCounter

```javascript
// composables/useCounter.js
import { ref, computed } from 'vue'

export function useCounter(initialValue = 0, options = {}) {
  const { min = -Infinity, max = Infinity, step = 1 } = options
  
  const count = ref(initialValue)
  
  // computed values
  const isAtMin = computed(() => count.value <= min)
  const isAtMax = computed(() => count.value >= max)
  const percentage = computed(() => {
    if (min === -Infinity || max === Infinity) return null
    return ((count.value - min) / (max - min)) * 100
  })
  
  // methods
  function increment() {
    if (count.value + step <= max) {
      count.value += step
    }
  }
  
  function decrement() {
    if (count.value - step >= min) {
      count.value -= step
    }
  }
  
  function reset() {
    count.value = initialValue
  }
  
  function set(value) {
    count.value = Math.max(min, Math.min(max, value))
  }
  
  return {
    count,
    isAtMin,
    isAtMax,
    percentage,
    increment,
    decrement,
    reset,
    set
  }
}
```

### การใช้งาน

```vue
<!-- components/CounterDemo.vue -->
<template>
  <div class="counter-demo">
    <!-- Counter พื้นฐาน -->
    <div class="counter-box">
      <h3>Counter พื้นฐาน</h3>
      <button @click="decrement" :disabled="isAtMin">-</button>
      <span class="count">{{ count }}</span>
      <button @click="increment" :disabled="isAtMax">+</button>
      <button @click="reset">Reset</button>
    </div>

    <!-- Progress bar จาก percentage -->
    <div v-if="percentage !== null" class="progress-bar">
      <div :style="{ width: percentage + '%' }" class="progress-fill"></div>
      <span>{{ Math.round(percentage) }}%</span>
    </div>

    <!-- ใช้ composable เดิมหลายครั้ง - state แยกกัน! -->
    <div class="multi-counter">
      <h3>หลาย Counter</h3>
      <div>Counter A: {{ counterA.count.value }}</div>
      <div>Counter B: {{ counterB.count.value }}</div>
      <button @click="counterA.increment">A++</button>
      <button @click="counterB.increment">B++</button>
    </div>
  </div>
</template>

<script setup>
import { useCounter } from '../composables/useCounter.js'

// Counter หลัก (min: 0, max: 100, step: 5)
const { count, isAtMin, isAtMax, percentage, increment, decrement, reset } = useCounter(0, {
  min: 0,
  max: 100,
  step: 5
})

// สร้าง counter แยกกัน - state ไม่ share กัน
const counterA = useCounter(0)
const counterB = useCounter(10)
</script>
```

---

## 3. Naming Conventions (use prefix)

```javascript
// ✅ ชื่อที่ถูกต้อง
export function useCounter() {}
export function useFetch() {}
export function useLocalStorage() {}
export function useEventListener() {}
export function useDebounce() {}
export function useMediaQuery() {}

// ❌ ชื่อที่ไม่ถูกต้อง
export function counter() {}    // ไม่มี use prefix
export function fetchData() {}  // ไม่ชัดเจนว่าเป็น composable
export function CounterHook() {} // ขึ้นต้นด้วย uppercase
```

---

## 4. Composable with Lifecycle

```javascript
// composables/useMousePosition.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMousePosition() {
  const x = ref(0)
  const y = ref(0)
  const isTracking = ref(false)
  
  function updatePosition(event) {
    x.value = event.clientX
    y.value = event.clientY
  }
  
  function startTracking() {
    if (isTracking.value) return
    window.addEventListener('mousemove', updatePosition)
    isTracking.value = true
  }
  
  function stopTracking() {
    window.removeEventListener('mousemove', updatePosition)
    isTracking.value = false
  }
  
  // Lifecycle hooks ใน composable
  onMounted(() => {
    startTracking()
  })
  
  onUnmounted(() => {
    stopTracking()
  })
  
  return {
    x,
    y,
    isTracking,
    startTracking,
    stopTracking
  }
}
```

```vue
<!-- components/MouseTracker.vue -->
<template>
  <div class="mouse-tracker">
    <p>Mouse Position: ({{ x }}, {{ y }})</p>
    <button @click="isTracking ? stopTracking() : startTracking()">
      {{ isTracking ? 'หยุดติดตาม' : 'เริ่มติดตาม' }}
    </button>
    
    <!-- แสดงจุด dot ตามตำแหน่ง mouse -->
    <div
      v-if="isTracking"
      class="cursor-dot"
      :style="{ left: x + 'px', top: y + 'px' }"
    ></div>
  </div>
</template>

<script setup>
import { useMousePosition } from '../composables/useMousePosition.js'

const { x, y, isTracking, startTracking, stopTracking } = useMousePosition()
</script>
```

---

## 5. Composable with Watchers

```javascript
// composables/useFormValidation.js
import { ref, reactive, computed, watch } from 'vue'

export function useFormValidation(schema) {
  const values = reactive({})
  const errors = reactive({})
  const touched = reactive({})
  
  // Initialize fields จาก schema
  Object.keys(schema).forEach(field => {
    values[field] = schema[field].defaultValue || ''
    errors[field] = []
    touched[field] = false
  })
  
  const isValid = computed(() => 
    Object.values(errors).every(fieldErrors => fieldErrors.length === 0)
  )
  
  const isDirty = computed(() =>
    Object.values(touched).some(Boolean)
  )
  
  // Validate หนึ่ง field
  function validateField(field) {
    const rules = schema[field].rules || []
    const value = values[field]
    const fieldErrors = []
    
    for (const rule of rules) {
      const result = rule(value, values)
      if (result !== true) {
        fieldErrors.push(result)
      }
    }
    
    errors[field] = fieldErrors
    return fieldErrors.length === 0
  }
  
  // Validate ทุก fields
  function validateAll() {
    let allValid = true
    Object.keys(schema).forEach(field => {
      touched[field] = true
      if (!validateField(field)) {
        allValid = false
      }
    })
    return allValid
  }
  
  // Watch แต่ละ field เพื่อ validate อัตโนมัติ
  Object.keys(schema).forEach(field => {
    watch(
      () => values[field],
      () => {
        if (touched[field]) {
          validateField(field)
        }
      }
    )
  })
  
  function reset() {
    Object.keys(schema).forEach(field => {
      values[field] = schema[field].defaultValue || ''
      errors[field] = []
      touched[field] = false
    })
  }
  
  return {
    values,
    errors,
    touched,
    isValid,
    isDirty,
    validateField,
    validateAll,
    reset
  }
}

// Validation rules
export const rules = {
  required: (msg = 'กรุณากรอกข้อมูล') => (value) => !!value.trim() || msg,
  email: (msg = 'อีเมลไม่ถูกต้อง') => (value) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value) || msg,
  minLength: (min, msg) => (value) => value.length >= min || msg || `ต้องมีอย่างน้อย ${min} ตัวอักษร`,
  maxLength: (max, msg) => (value) => value.length <= max || msg || `ต้องไม่เกิน ${max} ตัวอักษร`,
  matches: (field, msg) => (value, values) => value === values[field] || msg || 'ค่าไม่ตรงกัน'
}
```

---

## 6. VueUse Library

VueUse คือ collection ของ composables สำเร็จรูปกว่า 200 ตัว

```bash
npm install @vueuse/core
```

```vue
<template>
  <div>
    <!-- ใช้ useDark จาก VueUse -->
    <p>Dark mode: {{ isDark ? 'เปิด' : 'ปิด' }}</p>
    <button @click="toggleDark()">Toggle Dark</button>
    
    <!-- ใช้ useWindowSize -->
    <p>Window: {{ width }}x{{ height }}</p>
    
    <!-- ใช้ useBattery -->
    <p v-if="isSupported">Battery: {{ Math.round(level * 100) }}%</p>
  </div>
</template>

<script setup>
import { useDark, useToggle, useWindowSize, useBattery } from '@vueuse/core'

const isDark = useDark()
const toggleDark = useToggle(isDark)

const { width, height } = useWindowSize()

const { level, isSupported } = useBattery()
</script>
```

---

## 7. useFetch() - Data Fetching

```javascript
// composables/useFetch.js
import { ref, watch, shallowRef } from 'vue'

export function useFetch(url, options = {}) {
  const {
    immediate = true,
    initialData = null,
    transform = (data) => data
  } = options
  
  const data = shallowRef(initialData)
  const error = ref(null)
  const loading = ref(false)
  const status = ref(null)
  
  let abortController = null
  
  async function execute(overrideUrl) {
    const targetUrl = overrideUrl || (typeof url === 'function' ? url() : url)
    
    // ยกเลิก request ก่อนหน้า
    if (abortController) {
      abortController.abort()
    }
    
    abortController = new AbortController()
    
    loading.value = true
    error.value = null
    
    try {
      const response = await fetch(targetUrl, {
        ...options.fetchOptions,
        signal: abortController.signal
      })
      
      status.value = response.status
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }
      
      const rawData = await response.json()
      data.value = transform(rawData)
      
    } catch (err) {
      if (err.name === 'AbortError') return
      error.value = err
      console.error('Fetch error:', err)
    } finally {
      loading.value = false
    }
  }
  
  function refetch() {
    return execute()
  }
  
  // Watch url changes
  if (typeof url === 'function') {
    watch(url, execute)
  }
  
  // Auto execute
  if (immediate) {
    execute()
  }
  
  return {
    data,
    error,
    loading,
    status,
    execute,
    refetch
  }
}
```

### การใช้ useFetch

```vue
<!-- components/PostsList.vue -->
<template>
  <div>
    <div v-if="loading" class="loading-spinner">กำลังโหลด...</div>
    
    <div v-else-if="error" class="error-message">
      ❌ {{ error.message }}
      <button @click="refetch">ลองใหม่</button>
    </div>
    
    <div v-else>
      <div class="filter-bar">
        <input v-model="search" placeholder="ค้นหา..." />
        <select v-model="selectedCategory">
          <option value="">ทุกหมวด</option>
          <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
        </select>
      </div>
      
      <div class="posts-grid">
        <article v-for="post in filteredPosts" :key="post.id" class="post-card">
          <h3>{{ post.title }}</h3>
          <p>{{ post.excerpt }}</p>
          <div class="meta">
            <span>{{ post.category }}</span>
            <span>{{ post.date }}</span>
          </div>
        </article>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useFetch } from '../composables/useFetch.js'

const search = ref('')
const selectedCategory = ref('')

// Fetch posts
const { data: posts, loading, error, refetch } = useFetch(
  'https://api.example.com/posts',
  {
    transform: (data) => data.posts || data
  }
)

// Computed
const categories = computed(() => {
  if (!posts.value) return []
  return [...new Set(posts.value.map(p => p.category))]
})

const filteredPosts = computed(() => {
  if (!posts.value) return []
  return posts.value.filter(post => {
    const matchSearch = post.title.toLowerCase().includes(search.value.toLowerCase())
    const matchCategory = !selectedCategory.value || post.category === selectedCategory.value
    return matchSearch && matchCategory
  })
})
</script>
```

---

## 8. useLocalStorage() - Local Storage

```javascript
// composables/useLocalStorage.js
import { ref, watch } from 'vue'

export function useLocalStorage(key, defaultValue = null) {
  // อ่านค่าจาก localStorage
  function readFromStorage() {
    try {
      const item = localStorage.getItem(key)
      if (item === null) return defaultValue
      return JSON.parse(item)
    } catch {
      return defaultValue
    }
  }
  
  const storedValue = ref(readFromStorage())
  
  // Watch การเปลี่ยนแปลงและบันทึกลง localStorage
  watch(
    storedValue,
    (newValue) => {
      try {
        if (newValue === null || newValue === undefined) {
          localStorage.removeItem(key)
        } else {
          localStorage.setItem(key, JSON.stringify(newValue))
        }
      } catch (error) {
        console.error('localStorage write error:', error)
      }
    },
    { deep: true }
  )
  
  // Sync กับ tab อื่น
  function handleStorageEvent(event) {
    if (event.key === key) {
      try {
        storedValue.value = event.newValue ? JSON.parse(event.newValue) : defaultValue
      } catch {
        storedValue.value = defaultValue
      }
    }
  }
  
  window.addEventListener('storage', handleStorageEvent)
  
  function remove() {
    storedValue.value = defaultValue
    localStorage.removeItem(key)
  }
  
  return [storedValue, remove]
}
```

### การใช้ useLocalStorage

```vue
<!-- components/UserSettings.vue -->
<template>
  <div class="settings">
    <h3>การตั้งค่า</h3>
    
    <div class="setting-item">
      <label>ธีม</label>
      <select v-model="settings.theme">
        <option value="light">สว่าง</option>
        <option value="dark">มืด</option>
      </select>
    </div>
    
    <div class="setting-item">
      <label>ภาษา</label>
      <select v-model="settings.language">
        <option value="th">ภาษาไทย</option>
        <option value="en">English</option>
      </select>
    </div>
    
    <div class="setting-item">
      <label>แจ้งเตือน</label>
      <input type="checkbox" v-model="settings.notifications" />
    </div>
    
    <button @click="resetSettings">รีเซ็ต</button>
    <p class="hint">การตั้งค่าจะถูกบันทึกอัตโนมัติ</p>
  </div>
</template>

<script setup>
import { useLocalStorage } from '../composables/useLocalStorage.js'

const defaultSettings = {
  theme: 'light',
  language: 'th',
  notifications: true
}

const [settings, removeSettings] = useLocalStorage('user-settings', defaultSettings)

function resetSettings() {
  removeSettings()
}
</script>
```

---

## 9. useDebounce() - Debounce

```javascript
// composables/useDebounce.js
import { ref, watch } from 'vue'

export function useDebounce(value, delay = 300) {
  const debouncedValue = ref(typeof value === 'function' ? value() : value)
  let timer = null
  
  // ถ้า value เป็น ref หรือ reactive
  watch(
    () => (typeof value === 'function' ? value() : value),
    (newValue) => {
      if (timer) clearTimeout(timer)
      timer = setTimeout(() => {
        debouncedValue.value = newValue
      }, delay)
    }
  )
  
  return debouncedValue
}

// Debounce function (ไม่ใช่ value)
export function useDebounceFn(fn, delay = 300) {
  let timer = null
  
  function debouncedFn(...args) {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => {
      fn(...args)
    }, delay)
  }
  
  function cancel() {
    if (timer) {
      clearTimeout(timer)
      timer = null
    }
  }
  
  return { fn: debouncedFn, cancel }
}
```

### การใช้ useDebounce

```vue
<!-- components/SearchInput.vue -->
<template>
  <div class="search-container">
    <input
      v-model="searchQuery"
      placeholder="ค้นหา..."
      class="search-input"
    />
    
    <div v-if="isSearching" class="searching-indicator">
      🔍 กำลังค้นหา...
    </div>
    
    <div class="results">
      <p v-if="debouncedQuery">ผลลัพธ์สำหรับ: "{{ debouncedQuery }}"</p>
      <div v-for="result in searchResults" :key="result.id">
        {{ result.title }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'
import { useDebounce } from '../composables/useDebounce.js'

const searchQuery = ref('')
const searchResults = ref([])
const isSearching = ref(false)

// Debounce การค้นหา 500ms
const debouncedQuery = useDebounce(() => searchQuery.value, 500)

watch(debouncedQuery, async (query) => {
  if (!query.trim()) {
    searchResults.value = []
    return
  }
  
  isSearching.value = true
  try {
    const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`)
    searchResults.value = await response.json()
  } finally {
    isSearching.value = false
  }
})
</script>
```

---

## 10. useMediaQuery() - Responsive

```javascript
// composables/useMediaQuery.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMediaQuery(query) {
  const matches = ref(false)
  
  let mediaQuery = null
  
  function updateMatches() {
    matches.value = mediaQuery?.matches || false
  }
  
  onMounted(() => {
    mediaQuery = window.matchMedia(query)
    updateMatches()
    
    // Modern API
    if (mediaQuery.addEventListener) {
      mediaQuery.addEventListener('change', updateMatches)
    } else {
      // Fallback for older browsers
      mediaQuery.addListener(updateMatches)
    }
  })
  
  onUnmounted(() => {
    if (mediaQuery) {
      if (mediaQuery.removeEventListener) {
        mediaQuery.removeEventListener('change', updateMatches)
      } else {
        mediaQuery.removeListener(updateMatches)
      }
    }
  })
  
  return matches
}

// Predefined breakpoints
export function useBreakpoints() {
  const sm = useMediaQuery('(min-width: 640px)')
  const md = useMediaQuery('(min-width: 768px)')
  const lg = useMediaQuery('(min-width: 1024px)')
  const xl = useMediaQuery('(min-width: 1280px)')
  const xxl = useMediaQuery('(min-width: 1536px)')
  const dark = useMediaQuery('(prefers-color-scheme: dark)')
  const reduced = useMediaQuery('(prefers-reduced-motion: reduce)')
  
  return { sm, md, lg, xl, xxl, dark, reduced }
}
```

---

## 11. useEventListener() - Event Listener

```javascript
// composables/useEventListener.js
import { onMounted, onUnmounted, isRef } from 'vue'

export function useEventListener(target, event, handler, options = {}) {
  function getTarget() {
    return isRef(target) ? target.value : target
  }
  
  onMounted(() => {
    const el = getTarget()
    if (el) {
      el.addEventListener(event, handler, options)
    }
  })
  
  onUnmounted(() => {
    const el = getTarget()
    if (el) {
      el.removeEventListener(event, handler, options)
    }
  })
}
```

```vue
<!-- components/KeyboardShortcuts.vue -->
<template>
  <div>
    <p>กด Ctrl+K เพื่อค้นหา</p>
    <p>กด Escape เพื่อปิด</p>
    <p>กด Ctrl+S เพื่อบันทึก</p>
    
    <div v-if="showSearch" class="search-overlay">
      <input ref="searchInput" v-model="searchQuery" placeholder="ค้นหา..." />
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import { useEventListener } from '../composables/useEventListener.js'

const showSearch = ref(false)
const searchQuery = ref('')
const searchInput = ref(null)

useEventListener(window, 'keydown', async (event) => {
  // Ctrl+K หรือ Cmd+K
  if ((event.ctrlKey || event.metaKey) && event.key === 'k') {
    event.preventDefault()
    showSearch.value = true
    await nextTick()
    searchInput.value?.focus()
  }
  
  // Escape
  if (event.key === 'Escape') {
    showSearch.value = false
  }
  
  // Ctrl+S
  if ((event.ctrlKey || event.metaKey) && event.key === 's') {
    event.preventDefault()
    handleSave()
  }
})

function handleSave() {
  console.log('Saving...')
}
</script>
```

---

## 12. useClipboard() - Copy to Clipboard

```javascript
// composables/useClipboard.js
import { ref } from 'vue'

export function useClipboard(options = {}) {
  const { successDuration = 2000 } = options
  
  const copied = ref(false)
  const error = ref(null)
  let resetTimer = null
  
  async function copy(text) {
    try {
      if (navigator.clipboard?.writeText) {
        // Modern API
        await navigator.clipboard.writeText(text)
      } else {
        // Fallback
        const textarea = document.createElement('textarea')
        textarea.value = text
        textarea.style.position = 'fixed'
        textarea.style.opacity = '0'
        document.body.appendChild(textarea)
        textarea.select()
        document.execCommand('copy')
        document.body.removeChild(textarea)
      }
      
      copied.value = true
      error.value = null
      
      // Reset หลังจาก successDuration
      if (resetTimer) clearTimeout(resetTimer)
      resetTimer = setTimeout(() => {
        copied.value = false
      }, successDuration)
      
    } catch (err) {
      error.value = err
      copied.value = false
    }
  }
  
  return { copy, copied, error }
}
```

```vue
<!-- components/CopyButton.vue -->
<template>
  <button
    @click="copy(text)"
    :class="['copy-btn', { 'copied': copied }]"
    :disabled="copied"
  >
    <span v-if="copied">✅ คัดลอกแล้ว!</span>
    <span v-else>📋 {{ label }}</span>
  </button>
</template>

<script setup>
import { useClipboard } from '../composables/useClipboard.js'

const props = defineProps({
  text: String,
  label: { type: String, default: 'คัดลอก' }
})

const { copy, copied } = useClipboard({ successDuration: 2000 })
</script>
```

---

## 13. useGeolocation() - GPS Location

```javascript
// composables/useGeolocation.js
import { ref, onUnmounted } from 'vue'

export function useGeolocation(options = {}) {
  const {
    enableHighAccuracy = true,
    timeout = 10000,
    maximumAge = 0,
    watch = false
  } = options
  
  const coords = ref(null)
  const error = ref(null)
  const loading = ref(false)
  const isSupported = 'geolocation' in navigator
  
  let watchId = null
  
  function onSuccess(position) {
    coords.value = {
      latitude: position.coords.latitude,
      longitude: position.coords.longitude,
      accuracy: position.coords.accuracy,
      altitude: position.coords.altitude,
      altitudeAccuracy: position.coords.altitudeAccuracy,
      heading: position.coords.heading,
      speed: position.coords.speed,
      timestamp: position.timestamp
    }
    loading.value = false
    error.value = null
  }
  
  function onError(err) {
    const errorMessages = {
      1: 'ปฏิเสธสิทธิ์การเข้าถึงตำแหน่ง',
      2: 'ไม่สามารถหาตำแหน่งได้',
      3: 'หมดเวลา'
    }
    error.value = errorMessages[err.code] || err.message
    loading.value = false
  }
  
  const geoOptions = { enableHighAccuracy, timeout, maximumAge }
  
  function getCurrentPosition() {
    if (!isSupported) {
      error.value = 'Browser ไม่รองรับ Geolocation'
      return
    }
    
    loading.value = true
    navigator.geolocation.getCurrentPosition(onSuccess, onError, geoOptions)
  }
  
  function startWatching() {
    if (!isSupported || watchId !== null) return
    
    loading.value = true
    watchId = navigator.geolocation.watchPosition(onSuccess, onError, geoOptions)
  }
  
  function stopWatching() {
    if (watchId !== null) {
      navigator.geolocation.clearWatch(watchId)
      watchId = null
    }
  }
  
  if (watch) {
    startWatching()
  }
  
  onUnmounted(() => {
    stopWatching()
  })
  
  return {
    coords,
    error,
    loading,
    isSupported,
    getCurrentPosition,
    startWatching,
    stopWatching
  }
}
```

### การใช้ useGeolocation

```vue
<!-- components/LocationMap.vue -->
<template>
  <div class="location-map">
    <h3>ตำแหน่งของคุณ</h3>
    
    <div v-if="!isSupported" class="not-supported">
      Browser ของคุณไม่รองรับ Geolocation
    </div>
    
    <div v-else>
      <button @click="getCurrentPosition" :disabled="loading">
        {{ loading ? '⏳ กำลังหาตำแหน่ง...' : '📍 หาตำแหน่งของฉัน' }}
      </button>
      
      <div v-if="error" class="error">
        ❌ {{ error }}
      </div>
      
      <div v-if="coords" class="location-info">
        <p>📍 ละติจูด: {{ coords.latitude.toFixed(6) }}</p>
        <p>📍 ลองจิจูด: {{ coords.longitude.toFixed(6) }}</p>
        <p>🎯 ความแม่นยำ: {{ Math.round(coords.accuracy) }} เมตร</p>
        
        <a
          :href="`https://www.google.com/maps?q=${coords.latitude},${coords.longitude}`"
          target="_blank"
          class="map-link"
        >
          เปิดใน Google Maps
        </a>
      </div>
    </div>
  </div>
</template>

<script setup>
import { useGeolocation } from '../composables/useGeolocation.js'

const { coords, error, loading, isSupported, getCurrentPosition } = useGeolocation()
</script>
```

---

## 14. ตัวอย่าง Composite Composables

Composables สามารถใช้ composables อื่นภายในได้

```javascript
// composables/useInfiniteScroll.js
import { ref, onMounted, onUnmounted } from 'vue'
import { useFetch } from './useFetch.js'
import { useDebounce } from './useDebounce.js'

export function useInfiniteScroll(baseUrl, options = {}) {
  const { pageSize = 10, threshold = 200 } = options
  
  const page = ref(1)
  const allItems = ref([])
  const hasMore = ref(true)
  const isLoadingMore = ref(false)
  
  const url = () => `${baseUrl}?page=${page.value}&limit=${pageSize}`
  
  const { data, loading, error, execute } = useFetch(url, { immediate: false })
  
  async function loadMore() {
    if (!hasMore.value || isLoadingMore.value || loading.value) return
    
    isLoadingMore.value = true
    await execute()
    
    if (data.value) {
      if (data.value.length < pageSize) {
        hasMore.value = false
      }
      allItems.value = [...allItems.value, ...data.value]
      page.value++
    }
    
    isLoadingMore.value = false
  }
  
  function handleScroll() {
    const scrollBottom = window.innerHeight + window.scrollY
    const pageHeight = document.documentElement.scrollHeight
    
    if (pageHeight - scrollBottom < threshold) {
      loadMore()
    }
  }
  
  onMounted(() => {
    loadMore() // Load ครั้งแรก
    window.addEventListener('scroll', handleScroll)
  })
  
  onUnmounted(() => {
    window.removeEventListener('scroll', handleScroll)
  })
  
  function reset() {
    page.value = 1
    allItems.value = []
    hasMore.value = true
    loadMore()
  }
  
  return {
    items: allItems,
    loading,
    isLoadingMore,
    error,
    hasMore,
    loadMore,
    reset
  }
}
```

---

## สรุป Best Practices

```javascript
// ✅ ควรทำ
export function useMyFeature() {
  // 1. ประกาศ reactive state ที่ top level
  const data = ref(null)
  const loading = ref(false)
  
  // 2. ใช้ computed สำหรับ derived state
  const isEmpty = computed(() => !data.value)
  
  // 3. Cleanup ใน onUnmounted
  onUnmounted(() => {
    // cleanup code
  })
  
  // 4. Return เฉพาะสิ่งที่จำเป็น
  return { data, loading, isEmpty }
}

// ❌ ไม่ควรทำ
export function badComposable() {
  // อย่า call hooks แบบ conditional
  if (someCondition) {
    onMounted(() => {}) // ❌
  }
  
  // อย่า call hooks ใน loop
  for (const item of items) {
    watch(item, handler) // ❌
  }
}
```

| Composable | ใช้สำหรับ |
|-----------|-----------|
| `useFetch` | ดึงข้อมูลจาก API |
| `useLocalStorage` | เก็บข้อมูลใน browser |
| `useDebounce` | ลด event frequency |
| `useMediaQuery` | ตรวจสอบ screen size |
| `useEventListener` | จัดการ DOM events |
| `useClipboard` | คัดลอกข้อความ |
| `useGeolocation` | หาตำแหน่ง GPS |
