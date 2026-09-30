# Part 63: Push Notifications ใน Nuxt.js

## Web Push Notifications

Web Push Notifications ช่วยให้เว็บแอปส่งการแจ้งเตือนถึงผู้ใช้ได้ แม้ว่าผู้ใช้จะไม่ได้เปิดหน้าเว็บอยู่

## Firebase Cloud Messaging (FCM)

FCM คือ service จาก Google ที่ช่วยส่ง push notifications ข้ามแพลตฟอร์ม

### การติดตั้ง Firebase

```bash
npm install firebase
npm install firebase-admin
```

```typescript
// plugins/firebase.client.ts
import { initializeApp } from 'firebase/app'
import { getMessaging, getToken, onMessage } from 'firebase/messaging'

export default defineNuxtPlugin(async () => {
  const config = useRuntimeConfig()
  
  const firebaseConfig = {
    apiKey: config.public.firebaseApiKey,
    authDomain: config.public.firebaseAuthDomain,
    projectId: config.public.firebaseProjectId,
    storageBucket: config.public.firebaseStorageBucket,
    messagingSenderId: config.public.firebaseMessagingSenderId,
    appId: config.public.firebaseAppId
  }
  
  const app = initializeApp(firebaseConfig)
  const messaging = getMessaging(app)
  
  return {
    provide: {
      firebase: app,
      messaging
    }
  }
})
```

### Service Worker สำหรับ FCM

```javascript
// public/firebase-messaging-sw.js
importScripts('https://www.gstatic.com/firebasejs/10.0.0/firebase-app-compat.js')
importScripts('https://www.gstatic.com/firebasejs/10.0.0/firebase-messaging-compat.js')

firebase.initializeApp({
  apiKey: self.__WEB_APP_FIREBASE_API_KEY__,
  authDomain: self.__WEB_APP_FIREBASE_AUTH_DOMAIN__,
  projectId: self.__WEB_APP_FIREBASE_PROJECT_ID__,
  messagingSenderId: self.__WEB_APP_FIREBASE_MESSAGING_SENDER_ID__,
  appId: self.__WEB_APP_FIREBASE_APP_ID__
})

const messaging = firebase.messaging()

// Handle background messages
messaging.onBackgroundMessage((payload) => {
  console.log('Background message:', payload)
  
  const notificationTitle = payload.notification?.title || 'New Notification'
  const notificationOptions = {
    body: payload.notification?.body,
    icon: payload.notification?.icon || '/icons/notification-icon.png',
    badge: '/icons/badge-icon.png',
    data: payload.data,
    actions: payload.data?.actions ? JSON.parse(payload.data.actions) : [],
    vibrate: [200, 100, 200],
    requireInteraction: payload.data?.requireInteraction === 'true'
  }
  
  self.registration.showNotification(notificationTitle, notificationOptions)
})

// Handle notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close()
  
  const url = event.notification.data?.url || '/'
  
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then((clientList) => {
      // Check if there's already a window open
      for (const client of clientList) {
        if (client.url === url && 'focus' in client) {
          return client.focus()
        }
      }
      // Open new window
      return clients.openWindow(url)
    })
  )
})
```

## Notification Permission

```typescript
// composables/useNotifications.ts
import { ref, computed } from 'vue'
import { getToken, onMessage, type Messaging } from 'firebase/messaging'

interface NotificationPermissionState {
  status: NotificationPermission
  isSupported: boolean
  token: string | null
}

export function useNotifications() {
  const { $messaging } = useNuxtApp()
  const permissionState = ref<NotificationPermissionState>({
    status: 'default',
    isSupported: false,
    token: null
  })
  
  const isGranted = computed(() => permissionState.value.status === 'granted')
  const isDenied = computed(() => permissionState.value.status === 'denied')
  const canAsk = computed(() => permissionState.value.status === 'default')

  const checkSupport = () => {
    const supported = 'Notification' in window &&
      'serviceWorker' in navigator &&
      'PushManager' in window
    
    permissionState.value.isSupported = supported
    if (supported) {
      permissionState.value.status = Notification.permission
    }
    return supported
  }

  const requestPermission = async (): Promise<boolean> => {
    if (!checkSupport()) {
      return false
    }
    
    const permission = await Notification.requestPermission()
    permissionState.value.status = permission
    
    if (permission === 'granted') {
      await getFCMToken()
      return true
    }
    
    return false
  }

  const getFCMToken = async () => {
    const config = useRuntimeConfig()
    
    try {
      const token = await getToken($messaging as Messaging, {
        vapidKey: config.public.firebaseVapidKey,
        serviceWorkerRegistration: await navigator.serviceWorker.ready
      })
      
      permissionState.value.token = token
      
      // Send token to server
      await $fetch('/api/notifications/register-token', {
        method: 'POST',
        body: { token }
      })
      
      return token
    } catch (error) {
      console.error('Failed to get FCM token:', error)
      return null
    }
  }

  const listenForMessages = (callback: (payload: any) => void) => {
    return onMessage($messaging as Messaging, callback)
  }

  // Initialize
  if (import.meta.client) {
    checkSupport()
  }

  return {
    permissionState,
    isGranted,
    isDenied,
    canAsk,
    requestPermission,
    getFCMToken,
    listenForMessages
  }
}
```

## Notification Permission UI

```vue
<!-- components/NotificationPermissionBanner.vue -->
<template>
  <Transition name="slide-down">
    <div 
      v-if="showBanner" 
      class="notification-banner"
      role="region"
      aria-label="การแจ้งเตือน"
    >
      <div class="banner-content">
        <div class="banner-icon">🔔</div>
        <div class="banner-text">
          <h3>รับการแจ้งเตือนสำคัญ</h3>
          <p>เปิดใช้งานการแจ้งเตือนเพื่อรับข่าวสาร, อัพเดตคำสั่งซื้อ, และข้อเสนอพิเศษ</p>
        </div>
        <div class="banner-actions">
          <button 
            @click="handleAllow" 
            class="btn-allow"
            :disabled="isRequesting"
          >
            {{ isRequesting ? 'กำลังตั้งค่า...' : 'อนุญาต' }}
          </button>
          <button 
            @click="handleDismiss"
            class="btn-dismiss"
          >
            ไม่ตอนนี้
          </button>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const { canAsk, requestPermission, isGranted } = useNotifications()
const isRequesting = ref(false)
const dismissed = ref(false)

const showBanner = computed(() => 
  canAsk.value && !dismissed.value && !isGranted.value
)

const DISMISSED_KEY = 'notification_banner_dismissed'
const DISMISS_DELAY = 7 * 24 * 60 * 60 * 1000 // 7 days

onMounted(() => {
  try {
    const dismissedAt = localStorage.getItem(DISMISSED_KEY)
    if (dismissedAt) {
      const timeSinceDismiss = Date.now() - parseInt(dismissedAt)
      dismissed.value = timeSinceDismiss < DISMISS_DELAY
    }
  } catch {}
})

const handleAllow = async () => {
  isRequesting.value = true
  try {
    const granted = await requestPermission()
    if (!granted) {
      // Show instructions for denied
    }
    dismissed.value = true
  } finally {
    isRequesting.value = false
  }
}

const handleDismiss = () => {
  dismissed.value = true
  try {
    localStorage.setItem(DISMISSED_KEY, Date.now().toString())
  } catch {}
}
</script>
```

## Service Worker กับ Notifications

```javascript
// public/sw.js
const CACHE_NAME = 'vue-course-v1'
const STATIC_ASSETS = [
  '/',
  '/icons/notification-icon.png',
  '/icons/badge-icon.png'
]

// Install
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(STATIC_ASSETS))
  )
  self.skipWaiting()
})

// Activate
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys => 
      Promise.all(keys.filter(key => key !== CACHE_NAME).map(key => caches.delete(key)))
    )
  )
  self.clients.claim()
})

// Push event
self.addEventListener('push', (event) => {
  if (!event.data) return
  
  const data = event.data.json()
  
  const options = {
    body: data.body,
    icon: data.icon || '/icons/notification-icon.png',
    badge: '/icons/badge-icon.png',
    image: data.image,
    data: data.data,
    actions: data.actions || [],
    vibrate: [200, 100, 200],
    tag: data.tag || 'default',
    renotify: data.renotify || false,
    silent: data.silent || false,
    requireInteraction: data.requireInteraction || false,
    timestamp: Date.now()
  }
  
  event.waitUntil(
    self.registration.showNotification(data.title, options)
  )
})

// Notification click
self.addEventListener('notificationclick', (event) => {
  const notification = event.notification
  const action = event.action
  
  notification.close()
  
  if (action === 'view') {
    event.waitUntil(clients.openWindow(notification.data?.url || '/'))
  } else if (action === 'dismiss') {
    // Just close
  } else {
    // Default click
    event.waitUntil(
      clients.matchAll({ type: 'window' }).then(clientList => {
        const url = notification.data?.url || '/'
        
        for (const client of clientList) {
          if (client.url.includes(url) && 'focus' in client) {
            return client.focus()
          }
        }
        
        return clients.openWindow(url)
      })
    )
  }
})
```

## Notification Preferences

```vue
<!-- pages/settings/notifications.vue -->
<template>
  <div class="notification-settings">
    <h1>ตั้งค่าการแจ้งเตือน</h1>
    
    <!-- Permission Status -->
    <div class="permission-status">
      <div v-if="!isSupported" class="status-unsupported">
        <p>⚠️ เบราว์เซอร์ของคุณไม่รองรับการแจ้งเตือน</p>
      </div>
      <div v-else-if="isDenied" class="status-denied">
        <p>🚫 การแจ้งเตือนถูกปิดใช้งาน</p>
        <p>กรุณาเปิดการแจ้งเตือนในการตั้งค่าเบราว์เซอร์</p>
      </div>
      <div v-else-if="isGranted" class="status-granted">
        <p>✓ การแจ้งเตือนเปิดใช้งานอยู่</p>
      </div>
      <div v-else class="status-default">
        <p>การแจ้งเตือนยังไม่ได้รับอนุญาต</p>
        <button @click="requestPermission">เปิดการแจ้งเตือน</button>
      </div>
    </div>
    
    <!-- Notification Types -->
    <div v-if="isGranted" class="notification-types">
      <h2>ประเภทการแจ้งเตือน</h2>
      
      <div v-for="pref in preferences" :key="pref.key" class="pref-item">
        <div class="pref-info">
          <span class="pref-icon">{{ pref.icon }}</span>
          <div>
            <h3>{{ pref.name }}</h3>
            <p>{{ pref.description }}</p>
          </div>
        </div>
        <label class="toggle">
          <input
            type="checkbox"
            v-model="pref.enabled"
            @change="updatePreference(pref.key, pref.enabled)"
          />
          <span class="toggle-slider"></span>
        </label>
      </div>
      
      <!-- Test Notification -->
      <button @click="sendTestNotification" class="test-btn">
        ส่งการแจ้งเตือนทดสอบ
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

const { isGranted, isDenied, permissionState, requestPermission } = useNotifications()
const isSupported = computed(() => permissionState.value.isSupported)

const preferences = ref([
  {
    key: 'orders',
    name: 'อัพเดตคำสั่งซื้อ',
    description: 'แจ้งเตือนเมื่อสถานะคำสั่งซื้อเปลี่ยนแปลง',
    icon: '📦',
    enabled: true
  },
  {
    key: 'promotions',
    name: 'โปรโมชั่น',
    description: 'รับข่าวสารและส่วนลดพิเศษ',
    icon: '🎁',
    enabled: true
  },
  {
    key: 'messages',
    name: 'ข้อความใหม่',
    description: 'แจ้งเตือนเมื่อมีข้อความใหม่',
    icon: '💬',
    enabled: true
  },
  {
    key: 'reminders',
    name: 'เตือนความจำ',
    description: 'เตือนสินค้าในตะกร้าที่ยังไม่ได้ซื้อ',
    icon: '⏰',
    enabled: false
  }
])

const updatePreference = async (key: string, enabled: boolean) => {
  await $fetch('/api/notifications/preferences', {
    method: 'PUT',
    body: { [key]: enabled }
  })
}

const sendTestNotification = async () => {
  if ('Notification' in window && Notification.permission === 'granted') {
    const registration = await navigator.serviceWorker.ready
    await registration.showNotification('ทดสอบการแจ้งเตือน', {
      body: 'การแจ้งเตือนทำงานได้ถูกต้อง!',
      icon: '/icons/notification-icon.png',
      badge: '/icons/badge-icon.png'
    })
  }
}

onMounted(async () => {
  try {
    const prefs = await $fetch<Record<string, boolean>>('/api/notifications/preferences')
    preferences.value.forEach(p => {
      if (prefs[p.key] !== undefined) {
        p.enabled = prefs[p.key]
      }
    })
  } catch {}
})
</script>
```

## Server-side: ส่ง Push Notification

```typescript
// server/services/pushNotification.ts
import { getMessaging } from 'firebase-admin/messaging'
import { initializeApp, getApps, cert } from 'firebase-admin/app'

// Initialize Firebase Admin
if (!getApps().length) {
  initializeApp({
    credential: cert({
      projectId: process.env.FIREBASE_PROJECT_ID,
      clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
      privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n')
    })
  })
}

interface PushNotificationPayload {
  title: string
  body: string
  icon?: string
  image?: string
  data?: Record<string, string>
  url?: string
  actions?: Array<{ action: string; title: string }>
}

export async function sendPushToUser(
  userId: string,
  payload: PushNotificationPayload
) {
  const tokens = await getUserFCMTokens(userId)
  if (tokens.length === 0) return

  const message = {
    notification: {
      title: payload.title,
      body: payload.body,
      imageUrl: payload.image
    },
    data: {
      ...payload.data,
      url: payload.url || '/',
      actions: JSON.stringify(payload.actions || [])
    },
    webpush: {
      notification: {
        icon: payload.icon || '/icons/notification-icon.png',
        badge: '/icons/badge-icon.png',
        requireInteraction: false
      },
      fcmOptions: {
        link: payload.url || '/'
      }
    },
    tokens
  }

  try {
    const response = await getMessaging().sendEachForMulticast(message)
    
    // Handle invalid tokens
    response.responses.forEach(async (resp, idx) => {
      if (!resp.success && resp.error?.code === 'messaging/registration-token-not-registered') {
        await removeUserFCMToken(userId, tokens[idx])
      }
    })
    
    return response
  } catch (error) {
    console.error('Push notification error:', error)
    throw error
  }
}

export async function sendPushToTopic(
  topic: string,
  payload: PushNotificationPayload
) {
  const message = {
    notification: {
      title: payload.title,
      body: payload.body
    },
    data: payload.data || {},
    webpush: {
      notification: {
        icon: payload.icon || '/icons/notification-icon.png'
      }
    },
    topic
  }
  
  return getMessaging().send(message)
}

async function getUserFCMTokens(userId: string): Promise<string[]> {
  const tokens = await prisma.fcmToken.findMany({
    where: { userId, isValid: true },
    select: { token: true }
  })
  return tokens.map(t => t.token)
}

async function removeUserFCMToken(userId: string, token: string) {
  await prisma.fcmToken.updateMany({
    where: { userId, token },
    data: { isValid: false }
  })
}
```

## Real-time Notification System

```typescript
// server/api/notifications/register-token.post.ts
export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  const { token } = await readBody(event)
  
  // Upsert FCM token
  await prisma.fcmToken.upsert({
    where: { token },
    update: {
      userId: user.id,
      isValid: true,
      lastUsed: new Date()
    },
    create: {
      token,
      userId: user.id,
      isValid: true
    }
  })
  
  // Subscribe to user-specific topic
  await getMessaging().subscribeToTopic([token], `user-${user.id}`)
  
  return { success: true }
})
```

```vue
<!-- composables/useRealtimeNotifications.ts -->
<script setup lang="ts">
// composables/useRealtimeNotifications.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useRealtimeNotifications() {
  const notifications = ref<any[]>([])
  const unreadCount = ref(0)
  const { listenForMessages, isGranted } = useNotifications()
  
  let unsubscribe: (() => void) | null = null
  
  onMounted(() => {
    if (isGranted.value) {
      unsubscribe = listenForMessages((payload) => {
        const notification = {
          id: Date.now(),
          title: payload.notification?.title,
          body: payload.notification?.body,
          data: payload.data,
          timestamp: new Date(),
          read: false
        }
        
        notifications.value.unshift(notification)
        unreadCount.value++
        
        // Show in-app notification
        showInAppNotification(notification)
      })
    }
    
    fetchNotifications()
  })
  
  onUnmounted(() => {
    unsubscribe?.()
  })
  
  const fetchNotifications = async () => {
    try {
      const data = await $fetch('/api/notifications')
      notifications.value = data.notifications
      unreadCount.value = data.unreadCount
    } catch {}
  }
  
  const markAsRead = async (notificationId: number) => {
    await $fetch(`/api/notifications/${notificationId}/read`, { method: 'PUT' })
    const notif = notifications.value.find(n => n.id === notificationId)
    if (notif) {
      notif.read = true
      unreadCount.value = Math.max(0, unreadCount.value - 1)
    }
  }
  
  const markAllAsRead = async () => {
    await $fetch('/api/notifications/read-all', { method: 'PUT' })
    notifications.value.forEach(n => { n.read = true })
    unreadCount.value = 0
  }
  
  const showInAppNotification = (notification: any) => {
    // Use toast or notification component
    useToast().show({
      title: notification.title,
      message: notification.body,
      type: 'info',
      duration: 5000
    })
  }
  
  return {
    notifications,
    unreadCount,
    markAsRead,
    markAllAsRead,
    refresh: fetchNotifications
  }
}
</script>
```

## สรุป

Push Notifications ต้องการ:
1. Service Worker สำหรับ background notifications
2. FCM สำหรับ cross-platform delivery
3. Permission request ที่ user-friendly
4. Token management บน server
5. Notification preferences
6. In-app notification display
