# Part 47: PWA (Progressive Web App) กับ Nuxt.js

## PWA คืออะไร?

Progressive Web App (PWA) คือเว็บแอปที่ใช้เทคโนโลยี modern web เพื่อให้ประสบการณ์ใกล้เคียงกับ Native App ผู้ใช้สามารถติดตั้งลงบนหน้าจอหลัก ใช้งาน offline ได้ และรับ push notification

## ประโยชน์ของ PWA

- **Installable** - ติดตั้งลง device ได้
- **Offline Support** - ทำงานได้แม้ไม่มี internet
- **Fast** - cache ทำให้โหลดเร็ว
- **Push Notifications** - แจ้งเตือนผู้ใช้ได้
- **Native-like UX** - ประสบการณ์เหมือน app จริง

## 1. ติดตั้ง @vite-pwa/nuxt

```bash
npm install -D @vite-pwa/nuxt
```

```typescript
// nuxt.config.ts
import { defineNuxtConfig } from 'nuxt/config'

export default defineNuxtConfig({
  modules: ['@vite-pwa/nuxt'],
  
  pwa: {
    registerType: 'autoUpdate',
    
    manifest: {
      name: 'My PWA App',
      short_name: 'MyApp',
      description: 'แอปพลิเคชันที่ทำงาน offline ได้',
      theme_color: '#4CAF50',
      background_color: '#ffffff',
      display: 'standalone',
      orientation: 'portrait',
      scope: '/',
      start_url: '/',
      icons: [
        {
          src: 'pwa-64x64.png',
          sizes: '64x64',
          type: 'image/png'
        },
        {
          src: 'pwa-192x192.png',
          sizes: '192x192',
          type: 'image/png'
        },
        {
          src: 'pwa-512x512.png',
          sizes: '512x512',
          type: 'image/png',
          purpose: 'any maskable'
        }
      ],
      screenshots: [
        {
          src: 'screenshot-wide.png',
          sizes: '1280x720',
          type: 'image/png',
          form_factor: 'wide',
          label: 'Home Screen'
        },
        {
          src: 'screenshot-narrow.png',
          sizes: '390x844',
          type: 'image/png',
          form_factor: 'narrow',
          label: 'Home Screen Mobile'
        }
      ]
    },
    
    workbox: {
      navigateFallback: '/',
      globPatterns: ['**/*.{js,css,html,ico,png,svg,webp}'],
      
      // Cache strategies
      runtimeCaching: [
        // Cache API calls
        {
          urlPattern: /^https:\/\/api\.example\.com\/.*/i,
          handler: 'NetworkFirst',
          options: {
            cacheName: 'api-cache',
            expiration: {
              maxEntries: 50,
              maxAgeSeconds: 60 * 60 // 1 hour
            },
            networkTimeoutSeconds: 10
          }
        },
        // Cache images
        {
          urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp|avif)$/,
          handler: 'CacheFirst',
          options: {
            cacheName: 'images-cache',
            expiration: {
              maxEntries: 100,
              maxAgeSeconds: 60 * 60 * 24 * 30 // 30 days
            }
          }
        },
        // Cache fonts
        {
          urlPattern: /^https:\/\/fonts\.googleapis\.com\/.*/i,
          handler: 'StaleWhileRevalidate',
          options: {
            cacheName: 'google-fonts-cache'
          }
        }
      ]
    },
    
    client: {
      installPrompt: true,
      periodicSyncForUpdates: 3600 // Check for updates every hour
    },
    
    devOptions: {
      enabled: true,
      suppressWarnings: false,
      navigateFallbackAllowlist: [/^\/$/],
      type: 'module'
    }
  }
})
```

## 2. Web App Manifest

```json
// public/manifest.json (ถ้าสร้าง manual)
{
  "name": "My Awesome PWA",
  "short_name": "AwesomePWA",
  "description": "แอปที่ทำงาน offline ได้และรองรับ Push Notifications",
  "theme_color": "#4CAF50",
  "background_color": "#ffffff",
  "display": "standalone",
  "orientation": "any",
  "scope": "/",
  "start_url": "/?source=pwa",
  "lang": "th",
  "dir": "ltr",
  "categories": ["productivity", "utilities"],
  "icons": [
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "shortcuts": [
    {
      "name": "สร้างโพสต์ใหม่",
      "short_name": "โพสต์ใหม่",
      "description": "สร้างบทความใหม่",
      "url": "/posts/create?source=shortcut",
      "icons": [{ "src": "/icons/create-icon.png", "sizes": "96x96" }]
    }
  ],
  "related_applications": [],
  "prefer_related_applications": false
}
```

## 3. Service Workers

```typescript
// public/sw.js (custom service worker)
const CACHE_NAME = 'my-app-v1'
const OFFLINE_URL = '/offline'

// ไฟล์ที่ต้อง cache เสมอ
const PRECACHE_URLS = [
  '/',
  '/offline',
  '/css/main.css',
  '/js/main.js',
  '/icons/icon-192x192.png'
]

// Install event
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(PRECACHE_URLS))
      .then(() => self.skipWaiting())
  )
})

// Activate event - ลบ cache เก่า
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames
          .filter(name => name !== CACHE_NAME)
          .map(name => caches.delete(name))
      )
    }).then(() => self.clients.claim())
  )
})

// Fetch event - จัดการ network requests
self.addEventListener('fetch', (event) => {
  const { request } = event
  const url = new URL(request.url)
  
  // Skip non-GET requests
  if (request.method !== 'GET') return
  
  // API requests: Network First
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(request))
    return
  }
  
  // Images: Cache First
  if (request.destination === 'image') {
    event.respondWith(cacheFirst(request))
    return
  }
  
  // HTML: Network First with offline fallback
  if (request.headers.get('accept')?.includes('text/html')) {
    event.respondWith(
      fetch(request)
        .catch(() => caches.match(OFFLINE_URL))
    )
    return
  }
  
  // Default: Stale While Revalidate
  event.respondWith(staleWhileRevalidate(request))
})

// Network First strategy
async function networkFirst(request) {
  try {
    const response = await fetch(request)
    const cache = await caches.open(CACHE_NAME)
    cache.put(request, response.clone())
    return response
  } catch {
    return caches.match(request)
  }
}

// Cache First strategy
async function cacheFirst(request) {
  const cached = await caches.match(request)
  if (cached) return cached
  
  const response = await fetch(request)
  const cache = await caches.open(CACHE_NAME)
  cache.put(request, response.clone())
  return response
}

// Stale While Revalidate
async function staleWhileRevalidate(request) {
  const cached = await caches.match(request)
  
  const networkPromise = fetch(request).then(response => {
    caches.open(CACHE_NAME).then(cache => cache.put(request, response.clone()))
    return response
  })
  
  return cached || networkPromise
}

// Background Sync
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-pending-posts') {
    event.waitUntil(syncPendingPosts())
  }
})

async function syncPendingPosts() {
  const db = await openDB()
  const pendingPosts = await db.getAll('pendingPosts')
  
  for (const post of pendingPosts) {
    try {
      await fetch('/api/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(post)
      })
      await db.delete('pendingPosts', post.id)
    } catch {
      break // หยุดถ้า network ยังไม่พร้อม
    }
  }
}
```

## 4. Offline Support

```vue
<!-- components/pwa/OfflineIndicator.vue -->
<template>
  <Transition name="slide-down">
    <div v-if="!isOnline" class="offline-banner">
      <span class="offline-icon">📡</span>
      <span>คุณกำลังใช้งาน offline - บางฟีเจอร์อาจไม่พร้อมใช้งาน</span>
    </div>
  </Transition>
</template>

<script setup>
const isOnline = useOnline()
</script>

<style scoped>
.offline-banner {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  background: #ff9800;
  color: white;
  padding: 10px 16px;
  text-align: center;
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  font-size: 0.9em;
}

.slide-down-enter-active,
.slide-down-leave-active {
  transition: transform 0.3s ease;
}

.slide-down-enter-from,
.slide-down-leave-to {
  transform: translateY(-100%);
}
</style>
```

```typescript
// composables/useOnline.ts
export const useOnline = () => {
  const isOnline = ref(true)
  
  if (process.client) {
    isOnline.value = navigator.onLine
    
    window.addEventListener('online', () => { isOnline.value = true })
    window.addEventListener('offline', () => { isOnline.value = false })
    
    onUnmounted(() => {
      window.removeEventListener('online', () => {})
      window.removeEventListener('offline', () => {})
    })
  }
  
  return isOnline
}
```

```typescript
// composables/useOfflineStorage.ts
// ใช้ IndexedDB สำหรับ offline storage
export const useOfflineStorage = () => {
  const isOnline = useOnline()
  
  // บันทึก post ไว้ offline
  const savePostOffline = async (post: any) => {
    const db = await openDB('my-app', 1, {
      upgrade(db) {
        db.createObjectStore('pendingPosts', { keyPath: 'id' })
      }
    })
    
    post.id = post.id || Date.now().toString()
    await db.put('pendingPosts', { ...post, savedAt: new Date() })
    
    // Register background sync ถ้า SW รองรับ
    if ('serviceWorker' in navigator && 'SyncManager' in window) {
      const registration = await navigator.serviceWorker.ready
      await registration.sync.register('sync-pending-posts')
    }
    
    return post
  }
  
  // ดึง pending posts
  const getPendingPosts = async () => {
    const db = await openDB('my-app', 1)
    return db.getAll('pendingPosts')
  }
  
  // Sync เมื่อ online
  watchEffect(async () => {
    if (isOnline.value) {
      const pending = await getPendingPosts()
      if (pending.length > 0) {
        // Sync pending posts to server
        for (const post of pending) {
          try {
            await $fetch('/api/posts', { method: 'POST', body: post })
            const db = await openDB('my-app', 1)
            await db.delete('pendingPosts', post.id)
          } catch {
            break
          }
        }
      }
    }
  })
  
  return { savePostOffline, getPendingPosts }
}
```

## 5. Push Notifications

```typescript
// composables/usePushNotification.ts
export const usePushNotification = () => {
  const isSupported = computed(() => {
    if (!process.client) return false
    return 'Notification' in window && 'serviceWorker' in navigator && 'PushManager' in window
  })
  
  const permission = ref<NotificationPermission>('default')
  
  if (process.client) {
    permission.value = Notification.permission
  }
  
  const requestPermission = async () => {
    const result = await Notification.requestPermission()
    permission.value = result
    return result
  }
  
  const subscribe = async () => {
    if (permission.value !== 'granted') {
      await requestPermission()
    }
    
    if (permission.value !== 'granted') return null
    
    const registration = await navigator.serviceWorker.ready
    
    const subscription = await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(
        process.env.VITE_VAPID_PUBLIC_KEY!
      )
    })
    
    // บันทึก subscription ใน server
    await $fetch('/api/push/subscribe', {
      method: 'POST',
      body: subscription.toJSON()
    })
    
    return subscription
  }
  
  const unsubscribe = async () => {
    const registration = await navigator.serviceWorker.ready
    const subscription = await registration.pushManager.getSubscription()
    
    if (subscription) {
      await subscription.unsubscribe()
      await $fetch('/api/push/unsubscribe', {
        method: 'POST',
        body: { endpoint: subscription.endpoint }
      })
    }
  }
  
  const sendTestNotification = async () => {
    await $fetch('/api/push/test', { method: 'POST' })
  }
  
  return {
    isSupported,
    permission,
    requestPermission,
    subscribe,
    unsubscribe,
    sendTestNotification
  }
}

function urlBase64ToUint8Array(base64String: string) {
  const padding = '='.repeat((4 - base64String.length % 4) % 4)
  const base64 = (base64String + padding)
    .replace(/-/g, '+')
    .replace(/_/g, '/')
  const rawData = window.atob(base64)
  return new Uint8Array([...rawData].map(char => char.charCodeAt(0)))
}
```

## 6. Install Prompt

```vue
<!-- components/pwa/InstallPrompt.vue -->
<template>
  <div v-if="showPrompt" class="install-prompt">
    <div class="prompt-content">
      <img src="/icons/icon-72x72.png" alt="App Icon" class="app-icon" />
      <div class="prompt-text">
        <strong>ติดตั้ง MyApp</strong>
        <p>เพิ่มลงหน้าจอหลักเพื่อเข้าใช้งานได้เร็วขึ้น</p>
      </div>
      <div class="prompt-actions">
        <button @click="dismissPrompt" class="btn-later">ไว้ทีหลัง</button>
        <button @click="installApp" class="btn-install">ติดตั้ง</button>
      </div>
    </div>
  </div>
</template>

<script setup>
let deferredPrompt: any = null

const showPrompt = ref(false)

if (process.client) {
  window.addEventListener('beforeinstallprompt', (e) => {
    e.preventDefault()
    deferredPrompt = e
    
    // แสดง prompt หลังจาก 30 วินาที
    setTimeout(() => {
      const dismissed = localStorage.getItem('pwa-install-dismissed')
      if (!dismissed) {
        showPrompt.value = true
      }
    }, 30000)
  })
  
  window.addEventListener('appinstalled', () => {
    showPrompt.value = false
    deferredPrompt = null
    console.log('PWA was installed')
  })
}

const installApp = async () => {
  if (!deferredPrompt) return
  
  deferredPrompt.prompt()
  const { outcome } = await deferredPrompt.userChoice
  
  if (outcome === 'accepted') {
    console.log('User accepted the install prompt')
  }
  
  deferredPrompt = null
  showPrompt.value = false
}

const dismissPrompt = () => {
  showPrompt.value = false
  localStorage.setItem('pwa-install-dismissed', 'true')
}
</script>

<style scoped>
.install-prompt {
  position: fixed;
  bottom: 20px;
  left: 20px;
  right: 20px;
  background: white;
  border-radius: 16px;
  box-shadow: 0 8px 40px rgba(0,0,0,0.2);
  z-index: 1000;
  animation: slideUp 0.3s ease;
}

@keyframes slideUp {
  from { transform: translateY(100px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}

.prompt-content {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
}

.app-icon {
  width: 56px;
  height: 56px;
  border-radius: 12px;
  flex-shrink: 0;
}

.prompt-text {
  flex: 1;
}

.prompt-text strong {
  display: block;
  font-size: 1em;
  margin-bottom: 4px;
}

.prompt-text p {
  font-size: 0.85em;
  color: #666;
  margin: 0;
}

.prompt-actions {
  display: flex;
  gap: 8px;
  flex-direction: column;
}

.btn-later {
  padding: 8px 16px;
  background: none;
  border: 1px solid #ccc;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.85em;
}

.btn-install {
  padding: 8px 16px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.85em;
  font-weight: 600;
}
</style>
```

## 7. ตัวอย่าง: PWA App ที่ทำงาน Offline ได้

```vue
<!-- app.vue -->
<template>
  <div>
    <OfflineIndicator />
    <InstallPrompt />
    <NuxtPage />
    <UpdateAvailable />
  </div>
</template>
```

```vue
<!-- components/pwa/UpdateAvailable.vue -->
<template>
  <div v-if="updateAvailable" class="update-banner">
    <p>มีการอัปเดตใหม่!</p>
    <button @click="updateApp">อัปเดตทันที</button>
    <button @click="updateAvailable = false">ไว้ทีหลัง</button>
  </div>
</template>

<script setup>
const { needRefresh, updateServiceWorker } = useRegisterSW({
  onRegistered(r) {
    console.log('SW Registered:', r)
  },
  onRegisterError(error) {
    console.error('SW registration error', error)
  }
})

const updateAvailable = needRefresh

const updateApp = async () => {
  await updateServiceWorker(true)
  window.location.reload()
}
</script>

<style scoped>
.update-banner {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: #333;
  color: white;
  padding: 16px;
  border-radius: 12px;
  z-index: 1000;
  display: flex;
  align-items: center;
  gap: 12px;
}

.update-banner button {
  padding: 6px 14px;
  border-radius: 6px;
  border: none;
  cursor: pointer;
}

.update-banner button:first-of-type {
  background: #4CAF50;
  color: white;
}

.update-banner button:last-of-type {
  background: rgba(255,255,255,0.2);
  color: white;
}
</style>
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **PWA Basics** - หลักการและประโยชน์ของ PWA
2. **@vite-pwa/nuxt** - ตั้งค่า module สำหรับ Nuxt
3. **Web App Manifest** - การตั้งค่า app metadata
4. **Service Workers** - Cache strategies ต่างๆ
5. **Offline Support** - ทำงานได้แม้ไม่มี internet
6. **Push Notifications** - แจ้งเตือนผู้ใช้
7. **Install Prompt** - ให้ผู้ใช้ติดตั้งแอป
