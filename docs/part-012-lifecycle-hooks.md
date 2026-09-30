# Part 12: Component Lifecycle Hooks

## บทนำ

ทุก Vue component มีวงจรชีวิต (lifecycle) ตั้งแต่การสร้าง การอัปเดต ไปจนถึงการทำลาย Lifecycle Hooks คือ functions ที่เราสามารถเรียกใช้ในแต่ละช่วงของวงจรนั้น

---

## 1. Lifecycle Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Component Setup                        │
│                         │                                 │
│              ┌──────────▼──────────┐                     │
│              │    setup() / <script setup>   │                     │
│              └──────────┬──────────┘                     │
│                         │                                 │
│              ┌──────────▼──────────┐                     │
│              │   onBeforeMount()   │                     │
│              └──────────┬──────────┘                     │
│                         │                                 │
│         ┌───────────────▼──────────────┐                 │
│         │  DOM created, child rendered │                 │
│         └───────────────┬──────────────┘                 │
│                         │                                 │
│              ┌──────────▼──────────┐                     │
│              │    onMounted()      │ ← เข้าถึง DOM ได้   │
│              └──────────┬──────────┘                     │
│                         │                                 │
│              ┌──────────▼──────────┐                     │
│              │  (data changes...)  │                     │
│              └──────────┬──────────┘                     │
│                         │                                 │
│              ┌──────────▼──────────┐                     │
│              │  onBeforeUpdate()   │                     │
│              └──────────┬──────────┘                     │
│                         │                                 │
│              ┌──────────▼──────────┐                     │
│              │    onUpdated()      │                     │
│              └──────────┬──────────┘                     │
│                         │                                 │
│              ┌──────────▼──────────┐                     │
│              │  onBeforeUnmount()  │                     │
│              └──────────┬──────────┘                     │
│                         │                                 │
│              ┌──────────▼──────────┐                     │
│              │    onUnmounted()    │ ← cleanup ที่นี่     │
│              └─────────────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

---

## 2. onBeforeMount และ onMounted

### onBeforeMount
เรียกก่อน component จะ mount เข้า DOM ยังเข้าถึง DOM element ไม่ได้

### onMounted
เรียกหลัง component mount เข้า DOM แล้ว สามารถเข้าถึง DOM element ได้

```vue
<template>
  <div>
    <div ref="containerRef" class="container">
      <h2>{{ title }}</h2>
      <canvas ref="canvasRef" width="400" height="200"></canvas>
      <p v-for="item in items" :key="item.id">{{ item.name }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, onBeforeMount, onMounted } from 'vue'

const containerRef = ref(null)
const canvasRef = ref(null)
const title = ref('กำลังโหลด...')
const items = ref([])

// เรียกก่อน mount - ยังเข้าถึง DOM ไม่ได้
onBeforeMount(() => {
  console.log('onBeforeMount - Component กำลังจะ mount')
  console.log('containerRef.value:', containerRef.value) // null
  
  // เหมาะสำหรับ setup ที่ไม่ต้องการ DOM
  title.value = 'Loading data...'
})

// เรียกหลัง mount - เข้าถึง DOM ได้แล้ว
onMounted(async () => {
  console.log('onMounted - Component mount แล้ว')
  console.log('containerRef.value:', containerRef.value) // DOM element
  
  // 1. เข้าถึง DOM element ได้
  const rect = containerRef.value.getBoundingClientRect()
  console.log('Container size:', rect.width, 'x', rect.height)
  
  // 2. ใช้ third-party libraries ที่ต้องการ DOM
  initChart(canvasRef.value)
  
  // 3. Fetch ข้อมูลจาก API
  await fetchData()
  
  // 4. Setup event listeners
  window.addEventListener('resize', handleResize)
})

async function fetchData() {
  const response = await fetch('https://api.example.com/items')
  items.value = await response.json()
  title.value = 'รายการสินค้า'
}

function initChart(canvas) {
  const ctx = canvas.getContext('2d')
  // วาด chart บน canvas
  ctx.fillStyle = '#4CAF50'
  ctx.fillRect(0, 0, 100, 100)
}

function handleResize() {
  console.log('Window resized')
}
</script>
```

---

## 3. onBeforeUpdate และ onUpdated

### onBeforeUpdate
เรียกก่อน DOM จะอัปเดต เหมาะสำหรับอ่านค่า DOM ก่อนการเปลี่ยนแปลง

### onUpdated
เรียกหลัง DOM อัปเดตแล้ว ระวัง: ไม่ควรแก้ไข state ใน onUpdated เพราะอาจทำให้ loop ไม่สิ้นสุด

```vue
<template>
  <div>
    <input v-model="searchText" placeholder="ค้นหา..." />
    <ul ref="listRef">
      <li v-for="item in filteredItems" :key="item.id">
        {{ item.name }}
      </li>
    </ul>
    <p>จำนวนผลลัพธ์: {{ filteredItems.length }}</p>
  </div>
</template>

<script setup>
import { ref, computed, onBeforeUpdate, onUpdated } from 'vue'

const listRef = ref(null)
const searchText = ref('')
const items = ref([
  { id: 1, name: 'แอปเปิ้ล' },
  { id: 2, name: 'กล้วย' },
  { id: 3, name: 'มะม่วง' },
  { id: 4, name: 'ส้ม' }
])

const filteredItems = computed(() =>
  items.value.filter(item =>
    item.name.toLowerCase().includes(searchText.value.toLowerCase())
  )
)

let prevItemCount = 0

// เรียกก่อน DOM update
onBeforeUpdate(() => {
  prevItemCount = listRef.value?.children.length || 0
  console.log('Before update - item count:', prevItemCount)
})

// เรียกหลัง DOM update
onUpdated(() => {
  const currentCount = listRef.value?.children.length || 0
  console.log('After update - item count:', currentCount)
  
  if (currentCount !== prevItemCount) {
    console.log(`รายการเปลี่ยนจาก ${prevItemCount} เป็น ${currentCount}`)
  }
  
  // ระวัง! ไม่ควรแก้ไข reactive state ที่นี่ เพราะจะทำให้ update loop
  // ❌ searchText.value = 'bad idea' // ทำให้ infinite loop!
})
</script>
```

---

## 4. onBeforeUnmount และ onUnmounted

### onBeforeUnmount
เรียกก่อน component จะถูกทำลาย ยังเข้าถึง DOM ได้

### onUnmounted
เรียกหลัง component ถูกทำลายแล้ว ใช้สำหรับ cleanup

```vue
<template>
  <div>
    <p>เวลาปัจจุบัน: {{ currentTime }}</p>
    <p>ข้อความจาก WebSocket: {{ wsMessage }}</p>
    <p>จำนวน Event Listeners: {{ listenerCount }}</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, onUnmounted } from 'vue'

const currentTime = ref('')
const wsMessage = ref('กำลังเชื่อมต่อ...')
const listenerCount = ref(0)

let timer = null
let ws = null
const cleanupFunctions = []

onMounted(() => {
  // 1. ตั้ง timer
  timer = setInterval(() => {
    currentTime.value = new Date().toLocaleTimeString('th-TH')
  }, 1000)

  // 2. เชื่อมต่อ WebSocket
  ws = new WebSocket('wss://echo.websocket.org')
  
  ws.onopen = () => {
    wsMessage.value = 'เชื่อมต่อแล้ว'
    ws.send('Hello from Vue!')
  }
  
  ws.onmessage = (event) => {
    wsMessage.value = event.data
  }
  
  ws.onerror = () => {
    wsMessage.value = 'เกิดข้อผิดพลาด'
  }

  // 3. เพิ่ม event listeners
  function handleScroll() {
    console.log('Scrolling...')
  }
  
  function handleKeydown(e) {
    console.log('Key pressed:', e.key)
  }
  
  window.addEventListener('scroll', handleScroll)
  window.addEventListener('keydown', handleKeydown)
  listenerCount.value = 2
  
  // เก็บ cleanup functions ไว้ใช้ใน onUnmounted
  cleanupFunctions.push(
    () => window.removeEventListener('scroll', handleScroll),
    () => window.removeEventListener('keydown', handleKeydown)
  )
})

// เรียกก่อน unmount
onBeforeUnmount(() => {
  console.log('กำลังจะทำลาย component...')
  // ยังเข้าถึง DOM ได้อยู่
  // บันทึก scroll position หรือสถานะต่างๆ
})

// เรียกหลัง unmount - cleanup ทุกอย่าง
onUnmounted(() => {
  console.log('Component ถูกทำลายแล้ว')
  
  // 1. หยุด timer
  if (timer) {
    clearInterval(timer)
    timer = null
  }
  
  // 2. ปิด WebSocket
  if (ws) {
    ws.close()
    ws = null
  }
  
  // 3. ลบ event listeners
  cleanupFunctions.forEach(cleanup => cleanup())
  cleanupFunctions.length = 0
  listenerCount.value = 0
  
  console.log('Cleanup เสร็จสิ้น')
})
</script>
```

---

## 5. onErrorCaptured

ใช้สำหรับ catch errors จาก child components เหมาะสำหรับสร้าง Error Boundary

```vue
<!-- components/ErrorBoundary.vue -->
<template>
  <div>
    <div v-if="hasError" class="error-container">
      <h3>⚠️ เกิดข้อผิดพลาด</h3>
      <p class="error-message">{{ errorMessage }}</p>
      <p class="error-info">{{ errorInfo }}</p>
      <button @click="resetError">ลองใหม่</button>
    </div>
    <slot v-else></slot>
  </div>
</template>

<script setup>
import { ref, onErrorCaptured } from 'vue'

const hasError = ref(false)
const errorMessage = ref('')
const errorInfo = ref('')

// ดักจับ error จาก child components
onErrorCaptured((error, instance, info) => {
  console.error('Error captured:', error)
  console.error('Component instance:', instance)
  console.error('Error info:', info)
  
  hasError.value = true
  errorMessage.value = error.message || 'Unknown error occurred'
  errorInfo.value = info
  
  // ส่ง error ไปยัง logging service
  logErrorToService(error, info)
  
  // return false เพื่อหยุดไม่ให้ error propagate ขึ้นไป
  return false
})

function resetError() {
  hasError.value = false
  errorMessage.value = ''
  errorInfo.value = ''
}

function logErrorToService(error, info) {
  // ส่ง error ไปยัง Sentry, LogRocket หรือ service อื่นๆ
  console.log('Logging error to service:', { error: error.message, info })
}
</script>

<style scoped>
.error-container {
  padding: 20px;
  background: #fff3cd;
  border: 1px solid #ffc107;
  border-radius: 8px;
  text-align: center;
}
.error-message {
  color: #856404;
  font-weight: bold;
}
.error-info {
  color: #666;
  font-size: 12px;
}
</style>
```

### Component ที่อาจเกิด Error

```vue
<!-- components/BuggyComponent.vue -->
<template>
  <div>
    <!-- component ที่อาจ throw error -->
    <p>{{ userData.name.toUpperCase() }}</p>
  </div>
</template>

<script setup>
import { ref } from 'vue'

// userData อาจเป็น null ทำให้ .name.toUpperCase() error
const userData = ref(null)
</script>
```

### การใช้ ErrorBoundary

```vue
<!-- App.vue -->
<template>
  <div>
    <!-- wrap component ที่อาจ error ด้วย ErrorBoundary -->
    <ErrorBoundary>
      <BuggyComponent />
    </ErrorBoundary>

    <!-- ส่วนอื่นของ app ยังทำงานได้ปกติ -->
    <p>ส่วนนี้ยังทำงานได้แม้ BuggyComponent จะ error</p>
  </div>
</template>

<script setup>
import ErrorBoundary from './components/ErrorBoundary.vue'
import BuggyComponent from './components/BuggyComponent.vue'
</script>
```

---

## 6. onActivated และ onDeactivated (keep-alive)

เมื่อใช้ `<KeepAlive>` component จะไม่ถูกทำลายเมื่อซ่อน แต่จะเรียก `onDeactivated` แทน

```vue
<!-- App.vue -->
<template>
  <div>
    <nav>
      <button @click="currentTab = 'home'">หน้าแรก</button>
      <button @click="currentTab = 'profile'">โปรไฟล์</button>
      <button @click="currentTab = 'settings'">ตั้งค่า</button>
    </nav>

    <!-- KeepAlive เก็บ component state ไว้เมื่อ switch tab -->
    <KeepAlive>
      <component :is="currentComponent" />
    </KeepAlive>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import HomeTab from './components/HomeTab.vue'
import ProfileTab from './components/ProfileTab.vue'
import SettingsTab from './components/SettingsTab.vue'

const currentTab = ref('home')

const currentComponent = computed(() => {
  const map = {
    home: HomeTab,
    profile: ProfileTab,
    settings: SettingsTab
  }
  return map[currentTab.value]
})
</script>
```

```vue
<!-- components/HomeTab.vue -->
<template>
  <div>
    <h2>หน้าแรก</h2>
    <p>จำนวนครั้งที่ activate: {{ activationCount }}</p>
    <p>ข้อมูลที่กรอก: {{ formData }}</p>
    <input v-model="formData" placeholder="กรอกข้อมูล (จะถูกเก็บไว้)" />
  </div>
</template>

<script setup>
import { ref, onActivated, onDeactivated, onMounted, onUnmounted } from 'vue'

const activationCount = ref(0)
const formData = ref('')

onMounted(() => {
  console.log('HomeTab: mounted (ครั้งแรก)')
  startPolling()
})

onUnmounted(() => {
  console.log('HomeTab: unmounted')
  stopPolling()
})

// เรียกทุกครั้งที่ tab ถูกแสดง (รวมครั้งแรกด้วย)
onActivated(() => {
  activationCount.value++
  console.log('HomeTab: activated')
  
  // เริ่ม polling ใหม่เมื่อ tab กลับมาแสดง
  startPolling()
  
  // อัปเดตข้อมูลที่อาจ stale
  refreshData()
})

// เรียกเมื่อ tab ถูกซ่อน
onDeactivated(() => {
  console.log('HomeTab: deactivated')
  
  // หยุด polling เมื่อ tab ซ่อนอยู่ ประหยัด resources
  stopPolling()
  
  // บันทึก state ก่อนซ่อน
  saveState()
})

let pollingInterval = null

function startPolling() {
  if (pollingInterval) return // ป้องกัน duplicate
  pollingInterval = setInterval(() => {
    console.log('Polling data...')
  }, 5000)
}

function stopPolling() {
  if (pollingInterval) {
    clearInterval(pollingInterval)
    pollingInterval = null
  }
}

async function refreshData() {
  console.log('Refreshing data...')
}

function saveState() {
  localStorage.setItem('homeTabState', JSON.stringify({ formData: formData.value }))
}
</script>
```

---

## 7. Server-Side Hooks (onServerPrefetch)

ใช้สำหรับ Nuxt.js หรือ Vue SSR เพื่อ fetch ข้อมูลก่อน server render

```vue
<!-- components/ArticleList.vue (Nuxt.js) -->
<template>
  <div>
    <h2>บทความทั้งหมด</h2>
    <article v-for="article in articles" :key="article.id">
      <h3>{{ article.title }}</h3>
      <p>{{ article.summary }}</p>
    </article>
  </div>
</template>

<script setup>
import { ref, onServerPrefetch } from 'vue'

const articles = ref([])

// เรียกเฉพาะบน server ก่อน render
// ทำให้ HTML ที่ส่งมีข้อมูลอยู่แล้ว (ดี SEO)
onServerPrefetch(async () => {
  // Fetch ข้อมูลบน server
  const response = await fetch('https://api.example.com/articles')
  articles.value = await response.json()
  
  console.log('Articles fetched on server:', articles.value.length)
})
</script>
```

```vue
<!-- pages/index.vue (Nuxt.js) - ใช้ useAsyncData แทนก็ได้ -->
<template>
  <div>
    <ArticleList />
  </div>
</template>

<script setup>
// ใน Nuxt.js มักใช้ useAsyncData หรือ useFetch แทน onServerPrefetch
const { data: pageData } = await useAsyncData('page', async () => {
  return await $fetch('/api/page-data')
})
</script>
```

---

## 8. ตัวอย่างจริง: API Fetch ใน Lifecycle

```vue
<!-- components/UserProfile.vue -->
<template>
  <div class="user-profile">
    <!-- Loading State -->
    <div v-if="state.loading" class="skeleton">
      <div class="skeleton-avatar"></div>
      <div class="skeleton-text"></div>
      <div class="skeleton-text short"></div>
    </div>

    <!-- Error State -->
    <div v-else-if="state.error" class="error-state">
      <p>{{ state.error }}</p>
      <button @click="loadUser">ลองใหม่</button>
    </div>

    <!-- Success State -->
    <div v-else-if="state.data" class="profile">
      <img :src="state.data.avatar" :alt="state.data.name" class="avatar" />
      <h2>{{ state.data.name }}</h2>
      <p>{{ state.data.email }}</p>
      <p class="bio">{{ state.data.bio }}</p>

      <div class="stats">
        <div class="stat">
          <span class="stat-value">{{ state.data.posts }}</span>
          <span class="stat-label">โพสต์</span>
        </div>
        <div class="stat">
          <span class="stat-value">{{ state.data.followers }}</span>
          <span class="stat-label">ผู้ติดตาม</span>
        </div>
        <div class="stat">
          <span class="stat-value">{{ state.data.following }}</span>
          <span class="stat-label">กำลังติดตาม</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { reactive, watch, onMounted, onUnmounted } from 'vue'

const props = defineProps({
  userId: {
    type: Number,
    required: true
  }
})

// ใช้ reactive object แทน ref หลายตัว
const state = reactive({
  loading: false,
  error: null,
  data: null
})

let abortController = null

async function loadUser() {
  // ยกเลิก request ก่อนหน้า (ถ้ามี)
  if (abortController) {
    abortController.abort()
  }
  
  abortController = new AbortController()
  
  state.loading = true
  state.error = null
  
  try {
    const response = await fetch(
      `https://api.example.com/users/${props.userId}`,
      { signal: abortController.signal }
    )
    
    if (!response.ok) {
      throw new Error(`ไม่พบข้อมูลผู้ใช้ (${response.status})`)
    }
    
    state.data = await response.json()
  } catch (error) {
    if (error.name === 'AbortError') {
      console.log('Request cancelled')
      return
    }
    state.error = error.message
  } finally {
    state.loading = false
  }
}

// โหลดข้อมูลครั้งแรก
onMounted(loadUser)

// Reload เมื่อ userId เปลี่ยน
watch(() => props.userId, loadUser)

// Cleanup เมื่อ component ถูกทำลาย
onUnmounted(() => {
  if (abortController) {
    abortController.abort()
  }
})
</script>

<style scoped>
.user-profile {
  max-width: 400px;
  margin: 0 auto;
  padding: 20px;
}

.avatar {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  object-fit: cover;
}

.stats {
  display: flex;
  gap: 20px;
  margin-top: 16px;
}

.stat {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.stat-value {
  font-size: 24px;
  font-weight: bold;
  color: #4CAF50;
}

.skeleton-avatar {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  background: #ddd;
  animation: pulse 1.5s infinite;
}

.skeleton-text {
  height: 16px;
  background: #ddd;
  border-radius: 4px;
  margin: 8px 0;
  animation: pulse 1.5s infinite;
}

.skeleton-text.short {
  width: 60%;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}
</style>
```

---

## 9. ตัวอย่างจริง: WebSocket Connection

```vue
<!-- components/LiveChat.vue -->
<template>
  <div class="chat">
    <div class="chat-status">
      <span :class="['status-dot', connectionStatus]"></span>
      {{ statusText }}
    </div>

    <div ref="messageContainer" class="messages">
      <div
        v-for="msg in messages"
        :key="msg.id"
        :class="['message', msg.type]"
      >
        <span class="sender">{{ msg.sender }}</span>
        <p>{{ msg.text }}</p>
        <span class="time">{{ msg.time }}</span>
      </div>
    </div>

    <div class="chat-input">
      <input
        v-model="newMessage"
        @keyup.enter="sendMessage"
        placeholder="พิมพ์ข้อความ..."
        :disabled="connectionStatus !== 'connected'"
      />
      <button
        @click="sendMessage"
        :disabled="connectionStatus !== 'connected'"
      >
        ส่ง
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'

const props = defineProps({
  roomId: String,
  username: String
})

const messages = ref([])
const newMessage = ref('')
const messageContainer = ref(null)
const connectionStatus = ref('disconnected') // 'connecting', 'connected', 'disconnected', 'error'

let ws = null
let reconnectTimer = null
let reconnectAttempts = 0
const MAX_RECONNECT_ATTEMPTS = 5

const statusText = computed(() => {
  const texts = {
    connecting: 'กำลังเชื่อมต่อ...',
    connected: 'เชื่อมต่อแล้ว',
    disconnected: 'ตัดการเชื่อมต่อ',
    error: 'เกิดข้อผิดพลาด'
  }
  return texts[connectionStatus.value] || 'ไม่ทราบสถานะ'
})

function connect() {
  connectionStatus.value = 'connecting'
  
  try {
    ws = new WebSocket(`wss://chat.example.com/rooms/${props.roomId}`)
    
    ws.onopen = () => {
      connectionStatus.value = 'connected'
      reconnectAttempts = 0
      
      // ส่ง join message
      ws.send(JSON.stringify({
        type: 'join',
        username: props.username
      }))
      
      addSystemMessage('เชื่อมต่อสำเร็จ')
    }
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data)
      
      messages.value.push({
        id: Date.now() + Math.random(),
        sender: data.sender || 'System',
        text: data.text,
        type: data.sender === props.username ? 'sent' : 'received',
        time: new Date().toLocaleTimeString('th-TH')
      })
      
      // Scroll to bottom
      nextTick(() => scrollToBottom())
    }
    
    ws.onclose = () => {
      connectionStatus.value = 'disconnected'
      addSystemMessage('ตัดการเชื่อมต่อ')
      
      // Reconnect อัตโนมัติ
      if (reconnectAttempts < MAX_RECONNECT_ATTEMPTS) {
        scheduleReconnect()
      }
    }
    
    ws.onerror = (error) => {
      connectionStatus.value = 'error'
      console.error('WebSocket error:', error)
    }
    
  } catch (error) {
    connectionStatus.value = 'error'
    console.error('Connection failed:', error)
  }
}

function disconnect() {
  if (reconnectTimer) {
    clearTimeout(reconnectTimer)
    reconnectTimer = null
  }
  
  if (ws) {
    ws.close()
    ws = null
  }
}

function scheduleReconnect() {
  reconnectAttempts++
  const delay = Math.min(1000 * Math.pow(2, reconnectAttempts), 30000) // Exponential backoff
  
  addSystemMessage(`จะลองเชื่อมต่อใหม่ใน ${delay / 1000} วินาที... (ครั้งที่ ${reconnectAttempts})`)
  
  reconnectTimer = setTimeout(() => {
    connect()
  }, delay)
}

function sendMessage() {
  if (!newMessage.value.trim() || connectionStatus.value !== 'connected') return
  
  ws.send(JSON.stringify({
    type: 'message',
    text: newMessage.value,
    sender: props.username
  }))
  
  newMessage.value = ''
}

function addSystemMessage(text) {
  messages.value.push({
    id: Date.now() + Math.random(),
    sender: 'System',
    text,
    type: 'system',
    time: new Date().toLocaleTimeString('th-TH')
  })
}

function scrollToBottom() {
  if (messageContainer.value) {
    messageContainer.value.scrollTop = messageContainer.value.scrollHeight
  }
}

// เชื่อมต่อเมื่อ component mount
onMounted(() => {
  connect()
})

// Reconnect เมื่อ roomId เปลี่ยน
watch(() => props.roomId, () => {
  disconnect()
  messages.value = []
  connect()
})

// Cleanup เมื่อ component ถูกทำลาย
onUnmounted(() => {
  disconnect()
})
</script>

<style scoped>
.chat {
  display: flex;
  flex-direction: column;
  height: 500px;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.chat-status {
  padding: 8px 16px;
  background: #f5f5f5;
  border-bottom: 1px solid #ddd;
  display: flex;
  align-items: center;
  gap: 8px;
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.status-dot.connected { background: #4CAF50; }
.status-dot.connecting { background: #FF9800; animation: pulse 1s infinite; }
.status-dot.disconnected { background: #9E9E9E; }
.status-dot.error { background: #F44336; }

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.message {
  margin-bottom: 12px;
}

.message.sent {
  text-align: right;
}

.message.system {
  text-align: center;
  color: #999;
  font-size: 12px;
}

.chat-input {
  padding: 16px;
  display: flex;
  gap: 8px;
  border-top: 1px solid #ddd;
}

.chat-input input {
  flex: 1;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}
</style>
```

---

## สรุป Lifecycle Hooks

| Hook | เมื่อไหร่ถูกเรียก | ใช้สำหรับ |
|------|-------------------|-----------|
| `onBeforeMount` | ก่อน DOM mount | setup ที่ไม่ต้องการ DOM |
| `onMounted` | หลัง DOM mount | DOM manipulation, API fetch, Third-party libs |
| `onBeforeUpdate` | ก่อน DOM update | อ่านค่า DOM ก่อนเปลี่ยน |
| `onUpdated` | หลัง DOM update | DOM ที่ขึ้นกับ state |
| `onBeforeUnmount` | ก่อนทำลาย | บันทึก state สุดท้าย |
| `onUnmounted` | หลังทำลาย | Cleanup: timers, websockets, event listeners |
| `onErrorCaptured` | เมื่อ child error | Error boundaries |
| `onActivated` | KeepAlive: แสดง | Resume activities |
| `onDeactivated` | KeepAlive: ซ่อน | Pause activities |
| `onServerPrefetch` | SSR: ก่อน render | Fetch data บน server |
