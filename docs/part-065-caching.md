# Part 65: Caching Strategies ใน Nuxt.js

## Caching Patterns

การ caching ช่วยเพิ่ม performance โดยการเก็บข้อมูลที่ใช้บ่อยไว้ในที่ที่เข้าถึงได้เร็ว

### Types of Caching
1. **Browser Cache** - ข้อมูลอยู่ใน client's browser
2. **CDN Cache** - ข้อมูลอยู่ที่ edge servers ใกล้ผู้ใช้
3. **Application Cache** - ข้อมูลอยู่ใน application memory (Redis)
4. **Route Cache** - Nuxt caches route responses

## Redis Integration

```bash
npm install ioredis @upstash/redis
```

```typescript
// server/utils/redis.ts
import Redis from 'ioredis'

let redis: Redis | null = null

export function getRedis(): Redis {
  if (!redis) {
    redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: Number(process.env.REDIS_PORT) || 6379,
      password: process.env.REDIS_PASSWORD,
      db: Number(process.env.REDIS_DB) || 0,
      maxRetriesPerRequest: 3,
      lazyConnect: true,
      retryStrategy: (times) => {
        if (times > 3) return null
        return Math.min(times * 200, 1000)
      }
    })
    
    redis.on('error', (err) => {
      console.error('Redis error:', err)
    })
  }
  return redis
}

// Cache helper
export const cache = {
  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await getRedis().get(key)
      return value ? JSON.parse(value) : null
    } catch {
      return null
    }
  },
  
  async set(key: string, value: any, ttlSeconds = 300): Promise<void> {
    try {
      await getRedis().setex(key, ttlSeconds, JSON.stringify(value))
    } catch (err) {
      console.warn('Cache set error:', err)
    }
  },
  
  async del(key: string): Promise<void> {
    try {
      await getRedis().del(key)
    } catch {}
  },
  
  async invalidatePattern(pattern: string): Promise<void> {
    try {
      const keys = await getRedis().keys(pattern)
      if (keys.length > 0) {
        await getRedis().del(...keys)
      }
    } catch {}
  },
  
  async remember<T>(
    key: string,
    ttl: number,
    callback: () => Promise<T>
  ): Promise<T> {
    const cached = await cache.get<T>(key)
    if (cached !== null) return cached
    
    const value = await callback()
    await cache.set(key, value, ttl)
    return value
  }
}
```

## HTTP Cache Headers

```typescript
// server/middleware/cache-headers.ts
export default defineEventHandler((event) => {
  const url = getRequestURL(event)
  const path = url.pathname
  
  // Static assets - cache 1 year
  if (path.match(/\.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)(\?.*)?$/)) {
    setResponseHeader(event, 'Cache-Control', 'public, max-age=31536000, immutable')
    return
  }
  
  // API routes - no cache by default
  if (path.startsWith('/api/')) {
    setResponseHeader(event, 'Cache-Control', 'no-cache, no-store, must-revalidate')
    return
  }
  
  // Pages - short cache with revalidation
  if (getMethod(event) === 'GET') {
    setResponseHeader(event, 'Cache-Control', 'public, max-age=0, s-maxage=60, stale-while-revalidate=300')
  }
})
```

```typescript
// server/api/products/[id].get.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  
  // Check cache
  const cacheKey = `product:${id}`
  const cached = await cache.get(cacheKey)
  
  if (cached) {
    // Cache hit - add header
    setResponseHeader(event, 'X-Cache', 'HIT')
    setResponseHeader(event, 'Cache-Control', 'public, max-age=300, s-maxage=3600')
    return cached
  }
  
  // Cache miss
  const product = await prisma.product.findUnique({
    where: { id },
    include: { images: true, category: true }
  })
  
  if (!product) {
    throw createError({ statusCode: 404 })
  }
  
  // Cache for 5 minutes
  await cache.set(cacheKey, product, 300)
  
  setResponseHeader(event, 'X-Cache', 'MISS')
  setResponseHeader(event, 'Cache-Control', 'public, max-age=300, s-maxage=3600')
  
  return product
})
```

## Nuxt Route Cache

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // Static pages - cached indefinitely
    '/': { prerender: true },
    '/about': { prerender: true },
    
    // Blog - ISR with 1 hour revalidation
    '/blog/**': { isr: 3600 },
    
    // Product pages - ISR with 5 minutes revalidation
    '/products/**': { isr: 300 },
    
    // API routes - no cache
    '/api/**': { cache: false },
    
    // Admin - no cache, CSR only
    '/admin/**': { ssr: false, cache: false },
    
    // SWR - serve stale while revalidating
    '/dashboard': { swr: true }
  }
})
```

## SWR Pattern

```typescript
// composables/useSWR.ts
import { ref, computed, onMounted, onUnmounted } from 'vue'

interface SWROptions<T> {
  key: string
  fetcher: () => Promise<T>
  revalidateOnFocus?: boolean
  revalidateOnReconnect?: boolean
  refreshInterval?: number
  dedupingInterval?: number
  fallbackData?: T
  onSuccess?: (data: T) => void
  onError?: (error: Error) => void
}

const swrCache = new Map<string, { data: any; timestamp: number }>()
const pendingRequests = new Map<string, Promise<any>>()

export function useSWR<T>(options: SWROptions<T>) {
  const {
    key,
    fetcher,
    revalidateOnFocus = true,
    revalidateOnReconnect = true,
    refreshInterval,
    dedupingInterval = 2000,
    fallbackData,
    onSuccess,
    onError
  } = options
  
  const data = ref<T | undefined>(fallbackData)
  const error = ref<Error | null>(null)
  const isLoading = ref(false)
  const isValidating = ref(false)
  
  let refreshTimer: NodeJS.Timeout | null = null
  
  const revalidate = async (dedupe = true) => {
    // Check deduplication
    const now = Date.now()
    const cached = swrCache.get(key)
    
    if (dedupe && cached && now - cached.timestamp < dedupingInterval) {
      return
    }
    
    // Deduplicate concurrent requests
    if (pendingRequests.has(key)) {
      try {
        const result = await pendingRequests.get(key)
        data.value = result
        return
      } catch {}
    }
    
    isValidating.value = true
    if (!data.value) {
      isLoading.value = true
    }
    
    const fetchPromise = fetcher()
    pendingRequests.set(key, fetchPromise)
    
    try {
      const result = await fetchPromise
      data.value = result
      error.value = null
      
      swrCache.set(key, { data: result, timestamp: Date.now() })
      onSuccess?.(result)
    } catch (err: any) {
      error.value = err
      onError?.(err)
    } finally {
      isLoading.value = false
      isValidating.value = false
      pendingRequests.delete(key)
    }
  }
  
  // Load cached data immediately
  const cached = swrCache.get(key)
  if (cached) {
    data.value = cached.data
  }
  
  onMounted(() => {
    // Initial fetch
    revalidate(false)
    
    // Revalidate on focus
    if (revalidateOnFocus) {
      const handleFocus = () => revalidate()
      window.addEventListener('focus', handleFocus)
      onUnmounted(() => window.removeEventListener('focus', handleFocus))
    }
    
    // Revalidate on reconnect
    if (revalidateOnReconnect) {
      const handleOnline = () => revalidate()
      window.addEventListener('online', handleOnline)
      onUnmounted(() => window.removeEventListener('online', handleOnline))
    }
    
    // Refresh interval
    if (refreshInterval && refreshInterval > 0) {
      refreshTimer = setInterval(() => revalidate(), refreshInterval)
    }
  })
  
  onUnmounted(() => {
    if (refreshTimer) clearInterval(refreshTimer)
  })
  
  const mutate = (newData: T) => {
    data.value = newData
    swrCache.set(key, { data: newData, timestamp: Date.now() })
  }
  
  return {
    data: computed(() => data.value),
    error: computed(() => error.value),
    isLoading: computed(() => isLoading.value),
    isValidating: computed(() => isValidating.value),
    revalidate,
    mutate
  }
}
```

## CDN Caching

```typescript
// server/utils/cdn.ts
export async function purgeCDNCache(paths: string[]) {
  const provider = process.env.CDN_PROVIDER
  
  switch (provider) {
    case 'cloudflare':
      await purgeCloudflare(paths)
      break
    case 'vercel':
      await purgeVercel(paths)
      break
    case 'fastly':
      await purgeFastly(paths)
      break
  }
}

async function purgeCloudflare(paths: string[]) {
  const zoneId = process.env.CF_ZONE_ID
  const apiToken = process.env.CF_API_TOKEN
  
  await $fetch(`https://api.cloudflare.com/client/v4/zones/${zoneId}/purge_cache`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiToken}`,
      'Content-Type': 'application/json'
    },
    body: {
      files: paths.map(p => `${process.env.APP_URL}${p}`)
    }
  })
}

async function purgeVercel(paths: string[]) {
  const token = process.env.VERCEL_TOKEN
  const projectId = process.env.VERCEL_PROJECT_ID
  
  for (const path of paths) {
    await $fetch(`https://api.vercel.com/v1/projects/${projectId}/cache`, {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${token}` },
      body: { path }
    })
  }
}
```

## Browser Cache

```typescript
// composables/useLocalCache.ts
interface CacheEntry<T> {
  data: T
  expiry: number
}

export function useLocalCache<T>(key: string, ttlMs = 300000) {
  const get = (): T | null => {
    try {
      const item = localStorage.getItem(key)
      if (!item) return null
      
      const entry: CacheEntry<T> = JSON.parse(item)
      if (Date.now() > entry.expiry) {
        localStorage.removeItem(key)
        return null
      }
      
      return entry.data
    } catch {
      return null
    }
  }
  
  const set = (data: T): void => {
    try {
      const entry: CacheEntry<T> = {
        data,
        expiry: Date.now() + ttlMs
      }
      localStorage.setItem(key, JSON.stringify(entry))
    } catch (err) {
      // Handle storage quota exceeded
      if (err instanceof DOMException && err.code === 22) {
        // Clear old cache entries
        clearExpired()
      }
    }
  }
  
  const remove = (): void => {
    try {
      localStorage.removeItem(key)
    } catch {}
  }
  
  const clearExpired = (): void => {
    try {
      const now = Date.now()
      const keysToRemove: string[] = []
      
      for (let i = 0; i < localStorage.length; i++) {
        const k = localStorage.key(i)
        if (!k) continue
        
        try {
          const item = localStorage.getItem(k)
          if (item) {
            const entry = JSON.parse(item)
            if (entry.expiry && entry.expiry < now) {
              keysToRemove.push(k)
            }
          }
        } catch {}
      }
      
      keysToRemove.forEach(k => localStorage.removeItem(k))
    } catch {}
  }
  
  return { get, set, remove }
}
```

## ตัวอย่าง: Cached API Endpoints

```typescript
// server/api/products/featured.get.ts
import { cache } from '~/server/utils/redis'
import { defineEventHandler, setResponseHeader } from 'h3'

const CACHE_KEY = 'products:featured'
const CACHE_TTL = 600 // 10 minutes

export default defineEventHandler(async (event) => {
  // ETag support
  const etag = `"featured-${CACHE_TTL}"`
  const ifNoneMatch = getHeader(event, 'if-none-match')
  
  if (ifNoneMatch === etag) {
    setResponseStatus(event, 304)
    return
  }
  
  const products = await cache.remember(
    CACHE_KEY,
    CACHE_TTL,
    async () => {
      return prisma.product.findMany({
        where: {
          isFeatured: true,
          isActive: true
        },
        include: {
          images: { orderBy: { position: 'asc' }, take: 1 },
          category: true
        },
        orderBy: { sortOrder: 'asc' },
        take: 8
      })
    }
  )
  
  // Set cache headers
  setResponseHeader(event, 'ETag', etag)
  setResponseHeader(event, 'Cache-Control', 'public, max-age=0, s-maxage=600, stale-while-revalidate=120')
  setResponseHeader(event, 'Vary', 'Accept-Encoding')
  
  return products
})
```

```typescript
// server/middleware/conditional-cache.ts
export default defineEventHandler(async (event) => {
  const url = getRequestURL(event)
  
  // Only cache GET requests
  if (getMethod(event) !== 'GET') return
  
  const cacheKey = `route:${url.pathname}${url.search}`
  const cached = await cache.get<{ body: any; headers: Record<string, string> }>(cacheKey)
  
  if (cached) {
    // Restore cached headers
    Object.entries(cached.headers).forEach(([key, value]) => {
      setResponseHeader(event, key, value)
    })
    setResponseHeader(event, 'X-Cache', 'HIT')
    return cached.body
  }
  
  setResponseHeader(event, 'X-Cache', 'MISS')
})
```

## สรุป

Caching Strategy ที่ดีต้องมี:
1. เลือก cache ที่เหมาะกับข้อมูล
2. กำหนด TTL ที่เหมาะสม
3. Cache invalidation เมื่อข้อมูลเปลี่ยน
4. Monitor cache hit rate
5. Graceful degradation เมื่อ cache ล้มเหลว
6. SWR สำหรับ UX ที่ดี
