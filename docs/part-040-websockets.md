# Part 40: Real-time กับ WebSockets

## WebSockets คืออะไร?

WebSocket เป็น protocol ที่ช่วยให้ client และ server สื่อสารกันแบบ two-way real-time โดยไม่ต้องส่ง HTTP request ทุกครั้ง ต่างจาก HTTP polling ที่ client ต้องถามซ้ำ WebSocket จะเปิด connection ค้างไว้ตลอด

**Use cases:**
- Chat applications
- Live notifications
- Real-time dashboards
- Collaborative editing
- Online gaming
- Stock price updates

---

## 1. WebSocket พื้นฐาน

```ts
// WebSocket API พื้นฐาน
const ws = new WebSocket('ws://localhost:3001/ws')
// หรือ Secure WebSocket
const wss = new WebSocket('wss://api.example.com/ws')

// Events
ws.addEventListener('open', (event) => {
  console.log('Connected to WebSocket server')
  ws.send(JSON.stringify({ type: 'hello', data: 'World' }))
})

ws.addEventListener('message', (event) => {
  const message = JSON.parse(event.data)
  console.log('Received:', message)
})

ws.addEventListener('error', (event) => {
  console.error('WebSocket error:', event)
})

ws.addEventListener('close', (event) => {
  console.log('Disconnected:', event.code, event.reason)
})

// ส่งข้อมูล
ws.send(JSON.stringify({ type: 'message', content: 'Hello!' }))

// ปิด connection
ws.close(1000, 'Done')
```

---

## 2. ใช้ WebSocket ใน Vue Composable

```ts
// composables/useWebSocket.ts
import { ref, onUnmounted, type Ref } from 'vue'

type MessageHandler<T = unknown> = (data: T) => void
type ConnectionStatus = 'connecting' | 'open' | 'closing' | 'closed'

interface WebSocketOptions {
  url: string
  autoReconnect?: boolean
  reconnectDelay?: number
  maxReconnectAttempts?: number
  onOpen?: () => void
  onClose?: (event: CloseEvent) => void
  onError?: (event: Event) => void
}

interface UseWebSocketReturn<T> {
  socket: Ref<WebSocket | null>
  status: Ref<ConnectionStatus>
  lastMessage: Ref<T | null>
  send: (data: T) => void
  connect: () => void
  disconnect: () => void
  reconnectAttempts: Ref<number>
}

export function useWebSocket<T = unknown>(
  options: WebSocketOptions
): UseWebSocketReturn<T> {
  const {
    url,
    autoReconnect = true,
    reconnectDelay = 3000,
    maxReconnectAttempts = 5,
    onOpen,
    onClose,
    onError
  } = options

  const socket = ref<WebSocket | null>(null) as Ref<WebSocket | null>
  const status = ref<ConnectionStatus>('closed')
  const lastMessage = ref<T | null>(null) as Ref<T | null>
  const reconnectAttempts = ref(0)
  const messageHandlers = new Map<string, MessageHandler[]>()

  let reconnectTimer: ReturnType<typeof setTimeout> | null = null

  function connect() {
    if (socket.value?.readyState === WebSocket.OPEN) return

    status.value = 'connecting'
    const ws = new WebSocket(url)

    ws.addEventListener('open', () => {
      status.value = 'open'
      reconnectAttempts.value = 0
      onOpen?.()
    })

    ws.addEventListener('message', (event) => {
      try {
        const data = JSON.parse(event.data) as T
        lastMessage.value = data

        // Dispatch ไปยัง handlers ตาม message type
        if (data && typeof data === 'object' && 'type' in data) {
          const type = (data as any).type as string
          const handlers = messageHandlers.get(type) ?? []
          handlers.forEach(handler => handler(data))
        }
      } catch {
        console.error('Failed to parse WebSocket message:', event.data)
      }
    })

    ws.addEventListener('error', (event) => {
      console.error('WebSocket error:', event)
      onError?.(event)
    })

    ws.addEventListener('close', (event) => {
      status.value = 'closed'
      socket.value = null
      onClose?.(event)

      // Auto reconnect
      if (autoReconnect && reconnectAttempts.value < maxReconnectAttempts) {
        reconnectAttempts.value++
        console.log(
          `Reconnecting... attempt ${reconnectAttempts.value}/${maxReconnectAttempts}`
        )
        reconnectTimer = setTimeout(connect, reconnectDelay * reconnectAttempts.value)
      }
    })

    socket.value = ws
  }

  function disconnect() {
    if (reconnectTimer) clearTimeout(reconnectTimer)
    if (socket.value) {
      status.value = 'closing'
      socket.value.close(1000, 'Client disconnected')
    }
  }

  function send(data: T) {
    if (socket.value?.readyState === WebSocket.OPEN) {
      socket.value.send(JSON.stringify(data))
    } else {
      console.warn('WebSocket is not connected')
    }
  }

  // ลงทะเบียน message type handler
  function on(type: string, handler: MessageHandler) {
    if (!messageHandlers.has(type)) {
      messageHandlers.set(type, [])
    }
    messageHandlers.get(type)!.push(handler)

    // Return unsubscribe function
    return () => {
      const handlers = messageHandlers.get(type) ?? []
      messageHandlers.set(type, handlers.filter(h => h !== handler))
    }
  }

  // Cleanup เมื่อ component unmount
  onUnmounted(() => {
    disconnect()
  })

  return {
    socket,
    status,
    lastMessage,
    send,
    connect,
    disconnect,
    reconnectAttempts
  }
}
```

---

## 3. Nuxt WebSocket Server

```ts
// server/plugins/websocket.ts - WebSocket Server ใน Nuxt
import { defineWebSocketHandler } from 'h3'

// Connected clients map
const clients = new Map<string, { send: (data: string) => void }>()

export default defineNitroPlugin((nitroApp) => {
  nitroApp.router.use('/ws', defineWebSocketHandler({
    open(peer) {
      console.log('[WS] Client connected:', peer.id)
      clients.set(peer.id, {
        send: (data: string) => peer.send(data)
      })

      // แจ้งเมื่อ connect สำเร็จ
      peer.send(JSON.stringify({
        type: 'connected',
        id: peer.id,
        timestamp: new Date().toISOString()
      }))
    },

    message(peer, message) {
      try {
        const data = JSON.parse(message.text())
        console.log('[WS] Message from', peer.id, ':', data)

        // Handle different message types
        switch (data.type) {
          case 'ping':
            peer.send(JSON.stringify({ type: 'pong', timestamp: Date.now() }))
            break

          case 'broadcast':
            // ส่งไปยัง clients ทั้งหมด
            broadcast({
              type: 'message',
              from: peer.id,
              content: data.content,
              timestamp: new Date().toISOString()
            }, peer.id)
            break

          case 'join_room':
            // TODO: implement rooms
            peer.send(JSON.stringify({ type: 'joined', room: data.room }))
            break

          default:
            peer.send(JSON.stringify({ type: 'error', message: 'Unknown message type' }))
        }
      } catch {
        peer.send(JSON.stringify({ type: 'error', message: 'Invalid JSON' }))
      }
    },

    close(peer, event) {
      console.log('[WS] Client disconnected:', peer.id, event.code)
      clients.delete(peer.id)

      // แจ้ง clients อื่น
      broadcast({
        type: 'user_left',
        userId: peer.id,
        timestamp: new Date().toISOString()
      })
    },

    error(peer, error) {
      console.error('[WS] Error from', peer.id, ':', error)
    }
  }))

  // Helper function broadcast
  function broadcast(data: unknown, excludeId?: string) {
    const message = JSON.stringify(data)
    clients.forEach((client, id) => {
      if (id !== excludeId) {
        client.send(message)
      }
    })
  }
})
```

---

## 4. Socket.io กับ Nuxt

```bash
# ติดตั้ง socket.io
npm install socket.io socket.io-client
```

```ts
// server/plugins/socket-io.ts
import { Server } from 'socket.io'

export default defineNitroPlugin((nitroApp) => {
  const io = new Server(3001, {
    cors: {
      origin: process.env.FRONTEND_URL || 'http://localhost:3000',
      methods: ['GET', 'POST']
    }
  })

  // Middleware
  io.use((socket, next) => {
    const token = socket.handshake.auth.token
    if (!token) return next(new Error('Authentication required'))
    // verify token...
    next()
  })

  // Chat namespace
  const chat = io.of('/chat')

  chat.on('connection', (socket) => {
    console.log('User connected:', socket.id)

    // Join room
    socket.on('join_room', (roomId: string) => {
      socket.join(roomId)
      socket.to(roomId).emit('user_joined', {
        userId: socket.id,
        timestamp: new Date()
      })
    })

    // Send message
    socket.on('send_message', (data: {
      roomId: string
      content: string
      type: 'text' | 'image'
    }) => {
      const message = {
        id: Date.now().toString(),
        userId: socket.id,
        content: data.content,
        type: data.type,
        timestamp: new Date()
      }

      // ส่งไปทุกคนในห้อง รวมทั้ง sender
      chat.to(data.roomId).emit('new_message', message)
      socket.emit('new_message', message)
    })

    // Typing indicator
    socket.on('typing', (data: { roomId: string; isTyping: boolean }) => {
      socket.to(data.roomId).emit('user_typing', {
        userId: socket.id,
        isTyping: data.isTyping
      })
    })

    socket.on('disconnect', () => {
      console.log('User disconnected:', socket.id)
    })
  })
})
```

```ts
// composables/useSocketIO.ts - Client-side Socket.IO
import { io, type Socket } from 'socket.io-client'
import { ref, onUnmounted } from 'vue'

export function useSocketIO(namespace = '/') {
  const socket = ref<Socket | null>(null)
  const isConnected = ref(false)

  function connect(token?: string) {
    socket.value = io(`http://localhost:3001${namespace}`, {
      auth: { token },
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionAttempts: 5
    })

    socket.value.on('connect', () => {
      isConnected.value = true
      console.log('Socket connected:', socket.value?.id)
    })

    socket.value.on('disconnect', (reason) => {
      isConnected.value = false
      console.log('Socket disconnected:', reason)
    })

    socket.value.on('connect_error', (err) => {
      console.error('Connection error:', err.message)
    })
  }

  function emit<T>(event: string, data?: T) {
    socket.value?.emit(event, data)
  }

  function on<T>(event: string, callback: (data: T) => void) {
    socket.value?.on(event, callback)
    return () => socket.value?.off(event, callback)
  }

  function disconnect() {
    socket.value?.disconnect()
    socket.value = null
  }

  onUnmounted(disconnect)

  return { socket, isConnected, connect, emit, on, disconnect }
}
```

---

## 5. Real-time State Sync

```ts
// composables/useRealtimeStore.ts - Store ที่ sync ผ่าน WebSocket
import { defineStore } from 'pinia'
import { useWebSocket } from './useWebSocket'

interface LiveData {
  timestamp: string
  users: number
  orders: number
  revenue: number
  recentActivity: Activity[]
}

interface Activity {
  id: string
  type: string
  description: string
  timestamp: string
}

export const useRealtimeStore = defineStore('realtime', () => {
  const liveData = ref<LiveData>({
    timestamp: new Date().toISOString(),
    users: 0,
    orders: 0,
    revenue: 0,
    recentActivity: []
  })

  const { status, connect, send } = useWebSocket<{ type: string; data: LiveData }>({
    url: `${useRuntimeConfig().public.wsUrl}/dashboard`,
    autoReconnect: true,
    onOpen: () => {
      // Subscribe to live data
      send({ type: 'subscribe', channel: 'dashboard' } as any)
    }
  })

  // Listen to websocket messages
  const { lastMessage } = useWebSocket<{ type: string; payload: Partial<LiveData> }>({
    url: `${useRuntimeConfig().public.wsUrl}/dashboard`
  })

  watch(lastMessage, (message) => {
    if (!message) return

    switch (message.type) {
      case 'data_update':
        Object.assign(liveData.value, message.payload)
        break

      case 'activity':
        liveData.value.recentActivity.unshift(message.payload as Activity)
        if (liveData.value.recentActivity.length > 50) {
          liveData.value.recentActivity.pop()
        }
        break
    }
  })

  onMounted(() => connect())

  return {
    liveData: readonly(liveData),
    status,
    connect
  }
})
```

---

## 6. Connection Management

```ts
// composables/useConnectionManager.ts - จัดการหลาย WebSocket connections
import { ref, computed, onUnmounted } from 'vue'

interface ConnectionConfig {
  id: string
  url: string
  onMessage: (data: unknown) => void
}

interface Connection {
  id: string
  ws: WebSocket
  status: 'connecting' | 'open' | 'closed'
  reconnectCount: number
}

export function useConnectionManager() {
  const connections = ref<Map<string, Connection>>(new Map())

  const totalConnections = computed(() => connections.value.size)
  const activeConnections = computed(() =>
    [...connections.value.values()].filter(c => c.status === 'open').length
  )

  function addConnection(config: ConnectionConfig) {
    if (connections.value.has(config.id)) {
      console.warn(`Connection ${config.id} already exists`)
      return
    }

    function createWs() {
      const ws = new WebSocket(config.url)
      const connection: Connection = {
        id: config.id,
        ws,
        status: 'connecting',
        reconnectCount: 0
      }

      ws.addEventListener('open', () => {
        connection.status = 'open'
        connection.reconnectCount = 0
        connections.value.set(config.id, { ...connection })
      })

      ws.addEventListener('message', (event) => {
        try {
          config.onMessage(JSON.parse(event.data))
        } catch {
          config.onMessage(event.data)
        }
      })

      ws.addEventListener('close', () => {
        connection.status = 'closed'
        connections.value.set(config.id, { ...connection })

        // Auto reconnect หลัง 3 วินาที (max 5 ครั้ง)
        if (connection.reconnectCount < 5) {
          setTimeout(() => {
            connection.reconnectCount++
            createWs()
          }, 3000 * (connection.reconnectCount + 1))
        }
      })

      connections.value.set(config.id, connection)
    }

    createWs()
  }

  function removeConnection(id: string) {
    const conn = connections.value.get(id)
    if (conn) {
      conn.ws.close()
      connections.value.delete(id)
    }
  }

  function sendTo(id: string, data: unknown) {
    const conn = connections.value.get(id)
    if (conn?.status === 'open') {
      conn.ws.send(JSON.stringify(data))
    }
  }

  function broadcastAll(data: unknown) {
    connections.value.forEach((conn) => {
      if (conn.status === 'open') {
        conn.ws.send(JSON.stringify(data))
      }
    })
  }

  onUnmounted(() => {
    connections.value.forEach((conn) => conn.ws.close())
    connections.value.clear()
  })

  return {
    connections: readonly(connections),
    totalConnections,
    activeConnections,
    addConnection,
    removeConnection,
    sendTo,
    broadcastAll
  }
}
```

---

## 7. ตัวอย่าง: Real-time Chat App

```vue
<!-- pages/chat.vue - Real-time Chat App -->
<script setup lang="ts">
import { useSocketIO } from '@/composables/useSocketIO'
import { useAuthStore } from '@/stores/auth'

interface Message {
  id: string
  userId: string
  userName: string
  content: string
  timestamp: string
  type: 'text' | 'image' | 'system'
}

interface TypingUser {
  userId: string
  userName: string
}

const authStore = useAuthStore()
const { connect, emit, on, isConnected } = useSocketIO('/chat')

const messages = ref<Message[]>([])
const typingUsers = ref<TypingUser[]>([])
const newMessage = ref('')
const currentRoom = ref('general')
const messagesContainer = ref<HTMLElement | null>(null)

// Connect เมื่อ component mount
onMounted(async () => {
  connect(authStore.token ?? undefined)

  // Join default room
  emit('join_room', currentRoom.value)

  // Listen to events
  on<Message>('new_message', (message) => {
    messages.value.push(message)
    nextTick(() => scrollToBottom())
  })

  on<{ userId: string; userName: string; isTyping: boolean }>('user_typing', (data) => {
    if (data.isTyping) {
      if (!typingUsers.value.find(u => u.userId === data.userId)) {
        typingUsers.value.push({ userId: data.userId, userName: data.userName })
      }
    } else {
      typingUsers.value = typingUsers.value.filter(u => u.userId !== data.userId)
    }
  })

  on<{ userId: string }>('user_joined', (data) => {
    messages.value.push({
      id: Date.now().toString(),
      userId: 'system',
      userName: 'System',
      content: `${data.userId} เข้าร่วมห้องสนทนา`,
      timestamp: new Date().toISOString(),
      type: 'system'
    })
  })

  // Load history
  await loadMessageHistory()
})

async function loadMessageHistory() {
  const history = await $fetch<Message[]>(`/api/chat/${currentRoom.value}/messages`)
  messages.value = history
  nextTick(() => scrollToBottom())
}

function sendMessage() {
  if (!newMessage.value.trim() || !isConnected.value) return

  emit('send_message', {
    roomId: currentRoom.value,
    content: newMessage.value,
    type: 'text'
  })

  newMessage.value = ''
  stopTyping()
}

// Debounced typing indicator
let typingTimer: ReturnType<typeof setTimeout> | null = null

function handleTyping() {
  emit('typing', { roomId: currentRoom.value, isTyping: true })
  if (typingTimer) clearTimeout(typingTimer)
  typingTimer = setTimeout(stopTyping, 1000)
}

function stopTyping() {
  emit('typing', { roomId: currentRoom.value, isTyping: false })
  if (typingTimer) {
    clearTimeout(typingTimer)
    typingTimer = null
  }
}

function scrollToBottom() {
  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

const typingText = computed(() => {
  if (typingUsers.value.length === 0) return ''
  if (typingUsers.value.length === 1) {
    return `${typingUsers.value[0].userName} กำลังพิมพ์...`
  }
  return `${typingUsers.value.length} คนกำลังพิมพ์...`
})
</script>

<template>
  <div class="chat-page">
    <div class="chat-sidebar">
      <h2>ห้องสนทนา</h2>
      <button
        v-for="room in ['general', 'tech', 'random']"
        :key="room"
        :class="['room-btn', { active: currentRoom === room }]"
        @click="currentRoom = room"
      >
        # {{ room }}
      </button>
    </div>

    <div class="chat-main">
      <!-- Header -->
      <div class="chat-header">
        <h3># {{ currentRoom }}</h3>
        <span :class="['status', isConnected ? 'online' : 'offline']">
          {{ isConnected ? 'เชื่อมต่อแล้ว' : 'ไม่ได้เชื่อมต่อ' }}
        </span>
      </div>

      <!-- Messages -->
      <div ref="messagesContainer" class="messages">
        <div
          v-for="msg in messages"
          :key="msg.id"
          :class="[
            'message',
            msg.type,
            { own: msg.userId === authStore.user?.id?.toString() }
          ]"
        >
          <template v-if="msg.type === 'system'">
            <p class="system-message">{{ msg.content }}</p>
          </template>
          <template v-else>
            <div class="message-header">
              <strong>{{ msg.userName }}</strong>
              <time>{{ new Date(msg.timestamp).toLocaleTimeString('th-TH') }}</time>
            </div>
            <p class="message-content">{{ msg.content }}</p>
          </template>
        </div>
      </div>

      <!-- Typing Indicator -->
      <div v-if="typingText" class="typing-indicator">
        <span>{{ typingText }}</span>
      </div>

      <!-- Input -->
      <div class="message-input">
        <input
          v-model="newMessage"
          type="text"
          placeholder="พิมพ์ข้อความ..."
          @keyup.enter="sendMessage"
          @input="handleTyping"
          :disabled="!isConnected"
        />
        <button
          @click="sendMessage"
          :disabled="!newMessage.trim() || !isConnected"
        >
          ส่ง
        </button>
      </div>
    </div>
  </div>
</template>
```

---

## 8. ตัวอย่าง: Live Dashboard

```vue
<!-- pages/dashboard/live.vue - Live Dashboard -->
<script setup lang="ts">
import { useWebSocket } from '@/composables/useWebSocket'

interface DashboardMetric {
  key: string
  label: string
  value: number
  change: number
  unit: string
}

interface ChartDataPoint {
  time: string
  value: number
}

interface DashboardData {
  type: 'metrics' | 'chart_update' | 'alert'
  metrics?: DashboardMetric[]
  chart?: { key: string; point: ChartDataPoint }
  alert?: { level: 'info' | 'warning' | 'critical'; message: string }
}

const metrics = ref<DashboardMetric[]>([
  { key: 'users', label: 'ผู้ใช้ออนไลน์', value: 0, change: 0, unit: 'คน' },
  { key: 'orders', label: 'คำสั่งซื้อวันนี้', value: 0, change: 0, unit: 'รายการ' },
  { key: 'revenue', label: 'ยอดขายวันนี้', value: 0, change: 0, unit: '฿' },
  { key: 'errors', label: 'Error rate', value: 0, change: 0, unit: '%' }
])

const chartData = ref<Map<string, ChartDataPoint[]>>(new Map())
const alerts = ref<Array<{ level: string; message: string; time: Date }>>([])

const { status, connect } = useWebSocket<DashboardData>({
  url: `${useRuntimeConfig().public.wsUrl}/dashboard`,
  autoReconnect: true,
  onOpen: () => {
    console.log('Dashboard connected')
  }
})

const { lastMessage } = useWebSocket<DashboardData>({
  url: `${useRuntimeConfig().public.wsUrl}/dashboard`
})

// ประมวลผล messages
watch(lastMessage, (message) => {
  if (!message) return

  switch (message.type) {
    case 'metrics':
      if (message.metrics) {
        message.metrics.forEach(newMetric => {
          const existing = metrics.value.find(m => m.key === newMetric.key)
          if (existing) {
            existing.change = newMetric.value - existing.value
            existing.value = newMetric.value
          }
        })
      }
      break

    case 'chart_update':
      if (message.chart) {
        const { key, point } = message.chart
        if (!chartData.value.has(key)) {
          chartData.value.set(key, [])
        }
        const points = chartData.value.get(key)!
        points.push(point)
        // เก็บแค่ 60 points ล่าสุด
        if (points.length > 60) points.shift()
      }
      break

    case 'alert':
      if (message.alert) {
        alerts.value.unshift({
          ...message.alert,
          time: new Date()
        })
        if (alerts.value.length > 10) alerts.value.pop()
      }
      break
  }
})

onMounted(() => connect())

// Format helpers
function formatValue(metric: DashboardMetric): string {
  if (metric.unit === '฿') {
    return `฿${metric.value.toLocaleString()}`
  }
  return `${metric.value.toLocaleString()} ${metric.unit}`
}

function getChangeClass(change: number): string {
  if (change > 0) return 'increase'
  if (change < 0) return 'decrease'
  return 'neutral'
}

function formatChange(change: number): string {
  const prefix = change > 0 ? '+' : ''
  return `${prefix}${change.toLocaleString()}`
}
</script>

<template>
  <div class="live-dashboard">
    <div class="dashboard-header">
      <h1>Live Dashboard</h1>
      <div class="connection-badge" :class="status">
        <span class="dot" />
        {{ status === 'open' ? 'LIVE' : 'Disconnected' }}
      </div>
    </div>

    <!-- Metric Cards -->
    <div class="metrics-grid">
      <div v-for="metric in metrics" :key="metric.key" class="metric-card">
        <div class="metric-label">{{ metric.label }}</div>
        <div class="metric-value">{{ formatValue(metric) }}</div>
        <div :class="['metric-change', getChangeClass(metric.change)]">
          {{ formatChange(metric.change) }}
        </div>
      </div>
    </div>

    <!-- Alerts -->
    <div v-if="alerts.length > 0" class="alerts-section">
      <h2>การแจ้งเตือน</h2>
      <div
        v-for="(alert, i) in alerts"
        :key="i"
        :class="['alert', `alert-${alert.level}`]"
      >
        <span class="alert-time">
          {{ alert.time.toLocaleTimeString('th-TH') }}
        </span>
        <span class="alert-message">{{ alert.message }}</span>
      </div>
    </div>

    <!-- Connection Status -->
    <div class="status-bar">
      <span>สถานะ: {{ status }}</span>
      <button v-if="status === 'closed'" @click="connect">
        เชื่อมต่อใหม่
      </button>
    </div>
  </div>
</template>

<style scoped>
.live-dashboard {
  padding: 1.5rem;
}

.dashboard-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 2rem;
}

.connection-badge {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.875rem;
  font-weight: 600;
}

.connection-badge.open { background: #dcfce7; color: #16a34a; }
.connection-badge.closed { background: #fee2e2; color: #dc2626; }
.connection-badge.connecting { background: #fef9c3; color: #ca8a04; }

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: currentColor;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}

.metrics-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
  margin-bottom: 2rem;
}

.metric-card {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 1.25rem;
}

.metric-label { color: #6b7280; font-size: 0.875rem; margin-bottom: 0.5rem; }
.metric-value { font-size: 1.75rem; font-weight: 700; margin-bottom: 0.25rem; }
.metric-change { font-size: 0.875rem; }
.increase { color: #22c55e; }
.decrease { color: #ef4444; }
.neutral { color: #6b7280; }

.alert { padding: 0.75rem 1rem; border-radius: 4px; margin-bottom: 0.5rem; }
.alert-info { background: #eff6ff; }
.alert-warning { background: #fffbeb; }
.alert-critical { background: #fef2f2; }
</style>
```

---

## สรุป

WebSockets ใน Vue/Nuxt ช่วยสร้าง real-time applications ได้:

1. **WebSocket พื้นฐาน** - เข้าใจ API และ lifecycle events
2. **useWebSocket Composable** - จัดการ connection พร้อม auto-reconnect
3. **Nuxt Server** - สร้าง WebSocket server ด้วย Nitro/H3
4. **Socket.io** - library ที่มี features เพิ่มเติม เช่น rooms, namespaces
5. **Real-time State Sync** - sync Pinia store ผ่าน WebSocket
6. **Connection Manager** - จัดการหลาย connections พร้อมกัน
7. **Chat App** - ระบบ chat แบบ real-time ที่สมบูรณ์
8. **Live Dashboard** - dashboard ที่อัปเดต metrics แบบ real-time
