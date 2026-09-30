# Part 75: Performance at Scale

## ทำไม Performance at Scale ถึงสำคัญ?

Application ที่ทำงานได้ดีในขณะมีผู้ใช้น้อย อาจล่มเมื่อผู้ใช้มากขึ้น การออกแบบสำหรับ Scale ต้องคิดตั้งแต่ต้น

---

## 1. Horizontal Scaling

### Stateless Application Design

```typescript
// ❌ ไม่ดี - เก็บ state ใน memory
const sessions: Map<string, User> = new Map()

export default defineEventHandler((event) => {
  sessions.set(userId, user) // หาย เมื่อ restart หรือ scale
})

// ✅ ดี - เก็บ state ใน Redis
export default defineEventHandler(async (event) => {
  const redis = useRedis()
  await redis.setex(`session:${sessionId}`, 3600, JSON.stringify(user))
})
```

### Load Balancer Configuration (Nginx)

```nginx
# nginx.conf
upstream nuxt_app {
  least_conn; # ส่ง request ไปยัง server ที่มี connections น้อยสุด
  
  server app1:3000 weight=1 max_fails=3 fail_timeout=30s;
  server app2:3000 weight=1 max_fails=3 fail_timeout=30s;
  server app3:3000 weight=1 max_fails=3 fail_timeout=30s;
  
  keepalive 32; # Connection pooling
}

server {
  listen 80;
  server_name myapp.com;

  location / {
    proxy_pass http://nuxt_app;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_bypass $http_upgrade;
    
    # Timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
  }
}
```

### Docker Compose สำหรับ Scaling

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
      - NODE_ENV=production
    deploy:
      replicas: 3  # Scale แบบ horizontal
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data

  pgbouncer:
    image: edoburu/pgbouncer
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/myapp
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 20

volumes:
  postgres_data:
  redis_data:
```

---

## 2. Database Optimization

### Query Optimization

```typescript
// ❌ N+1 Problem
const posts = await prisma.post.findMany()
for (const post of posts) {
  const author = await prisma.user.findUnique({ where: { id: post.authorId } })
  // N+1 queries!
}

// ✅ ดี - ใช้ include
const posts = await prisma.post.findMany({
  include: { author: true }
})

// ✅ ดีกว่า - เลือกเฉพาะ fields ที่ต้องการ
const posts = await prisma.post.findMany({
  select: {
    id: true,
    title: true,
    author: {
      select: { name: true, avatar: true }
    }
  }
})
```

### Database Indexing

```sql
-- Index สำหรับ queries ที่ใช้บ่อย
CREATE INDEX CONCURRENTLY idx_posts_author_created 
ON posts(author_id, created_at DESC);

CREATE INDEX CONCURRENTLY idx_posts_search 
ON posts USING gin(to_tsvector('english', title || ' ' || content));

-- Partial index - index เฉพาะ published posts
CREATE INDEX CONCURRENTLY idx_posts_published 
ON posts(published_at DESC) 
WHERE status = 'published';

-- Composite index สำหรับ tenant queries
CREATE INDEX CONCURRENTLY idx_projects_tenant_status 
ON projects(tenant_id, status, created_at DESC);
```

### Connection Pooling

```typescript
// server/lib/prisma.ts
import { PrismaClient } from '@prisma/client'

declare global {
  var __prisma: PrismaClient | undefined
}

function createPrismaClient() {
  return new PrismaClient({
    log: process.env.NODE_ENV === 'development' 
      ? ['query', 'error', 'warn'] 
      : ['error'],
    datasources: {
      db: {
        url: process.env.DATABASE_URL
      }
    }
  })
}

// Singleton pattern - prevent multiple connections in dev
export const prisma = globalThis.__prisma ?? createPrismaClient()

if (process.env.NODE_ENV !== 'production') {
  globalThis.__prisma = prisma
}

// Graceful shutdown
process.on('beforeExit', async () => {
  await prisma.$disconnect()
})
```

### Read Replicas

```typescript
// server/lib/prisma-replicas.ts
import { PrismaClient } from '@prisma/client'

// Write operations -> Primary
export const primaryDb = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_PRIMARY_URL } }
})

// Read operations -> Replica
export const replicaDb = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_REPLICA_URL } }
})

// Wrapper
export class DatabaseRouter {
  async read<T>(fn: (db: PrismaClient) => Promise<T>): Promise<T> {
    return fn(replicaDb)
  }

  async write<T>(fn: (db: PrismaClient) => Promise<T>): Promise<T> {
    return fn(primaryDb)
  }
}

export const db = new DatabaseRouter()

// ใช้งาน
const users = await db.read(d => d.user.findMany())
const user = await db.write(d => d.user.create({ data: newUser }))
```

---

## 3. CDN Strategy

### Nuxt CDN Configuration

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    // ส่ง static assets ผ่าน CDN
    cdnURL: process.env.CDN_URL || ''
  },

  nitro: {
    // Route rules สำหรับ caching
    routeRules: {
      // Static pages - cache ที่ CDN นาน
      '/': { 
        prerender: true,
        headers: { 'cache-control': 'public, max-age=3600, s-maxage=86400' }
      },
      
      // API - ไม่ cache หรือ cache สั้น
      '/api/**': { 
        headers: { 'cache-control': 'no-store' }
      },
      
      // Public content - cache ที่ CDN
      '/blog/**': { 
        swr: 3600, // Stale While Revalidate
        headers: { 'cache-control': 'public, max-age=60, s-maxage=3600' }
      },
      
      // Static assets - cache นาน
      '/_nuxt/**': { 
        headers: { 'cache-control': 'public, max-age=31536000, immutable' }
      }
    }
  }
})
```

### Image Optimization

```vue
<!-- components/OptimizedImage.vue -->
<template>
  <picture>
    <!-- WebP สำหรับ modern browsers -->
    <source 
      :srcset="webpSrcset" 
      type="image/webp"
    />
    <!-- Fallback -->
    <img
      :src="src"
      :srcset="imgSrcset"
      :sizes="sizes"
      :alt="alt"
      :width="width"
      :height="height"
      loading="lazy"
      decoding="async"
    />
  </picture>
</template>

<script setup lang="ts">
const props = defineProps<{
  src: string
  alt: string
  width: number
  height: number
  sizes?: string
}>()

const CDN_URL = useRuntimeConfig().public.cdnUrl

function getImageUrl(src: string, width: number, format: string) {
  return `${CDN_URL}/image?url=${encodeURIComponent(src)}&w=${width}&f=${format}`
}

const webpSrcset = computed(() =>
  [320, 640, 1024, 1440].map(w =>
    `${getImageUrl(props.src, w, 'webp')} ${w}w`
  ).join(', ')
)

const imgSrcset = computed(() =>
  [320, 640, 1024, 1440].map(w =>
    `${getImageUrl(props.src, w, 'jpg')} ${w}w`
  ).join(', ')
)
</script>
```

---

## 4. Edge Computing

### Nuxt Edge Rendering

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages' // หรือ 'vercel-edge'
  }
})

// server/middleware/edge-cache.ts
export default defineEventHandler(async (event) => {
  // ใช้ Cache API ใน Edge
  if (typeof caches !== 'undefined') {
    const cache = await caches.open('v1')
    const cachedResponse = await cache.match(event.node.req)
    
    if (cachedResponse) {
      // Return cached response
      event.node.res.setHeader('X-Cache', 'HIT')
      return cachedResponse.text()
    }
  }
})
```

---

## 5. Cache Invalidation

### Cache Manager

```typescript
// server/lib/cache.ts
import { redis } from './redis'

export class CacheManager {
  private defaultTTL = 3600

  async get<T>(key: string): Promise<T | null> {
    const data = await redis.get(key)
    return data ? JSON.parse(data) : null
  }

  async set<T>(key: string, value: T, ttl?: number): Promise<void> {
    await redis.setex(key, ttl || this.defaultTTL, JSON.stringify(value))
  }

  async invalidate(pattern: string): Promise<void> {
    // ใช้ SCAN แทน KEYS เพื่อ performance ที่ดีกว่า
    let cursor = 0
    do {
      const [nextCursor, keys] = await redis.scan(
        cursor, 'MATCH', pattern, 'COUNT', 100
      )
      cursor = parseInt(nextCursor)
      if (keys.length > 0) {
        await redis.del(...keys)
      }
    } while (cursor !== 0)
  }

  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl?: number
  ): Promise<T> {
    const cached = await this.get<T>(key)
    if (cached !== null) return cached

    const fresh = await fetcher()
    await this.set(key, fresh, ttl)
    return fresh
  }

  // Tag-based invalidation
  async setWithTags<T>(
    key: string, 
    value: T, 
    tags: string[],
    ttl?: number
  ): Promise<void> {
    await this.set(key, value, ttl)
    
    // เก็บ key -> tags mapping
    for (const tag of tags) {
      await redis.sadd(`tag:${tag}`, key)
    }
  }

  async invalidateByTag(tag: string): Promise<void> {
    const keys = await redis.smembers(`tag:${tag}`)
    if (keys.length > 0) {
      await redis.del(...keys)
      await redis.del(`tag:${tag}`)
    }
  }
}

export const cache = new CacheManager()

// ใช้งาน
async function getProducts(tenantId: string) {
  return cache.getOrSet(
    `products:${tenantId}`,
    () => prisma.product.findMany({ where: { tenantId } }),
    300 // 5 minutes
  )
}

// เมื่อ update product
async function updateProduct(id: string, tenantId: string, data: any) {
  await prisma.product.update({ where: { id }, data })
  await cache.invalidateByTag(`products:${tenantId}`)
}
```

---

## 6. Load Testing กับ k6

### k6 Scripts

```javascript
// load-tests/api-test.js
import http from 'k6/http'
import { sleep, check } from 'k6'
import { Rate, Trend, Counter } from 'k6/metrics'

// Custom metrics
const errorRate = new Rate('errors')
const apiDuration = new Trend('api_duration')
const apiErrors = new Counter('api_errors')

export const options = {
  scenarios: {
    // Smoke test
    smoke: {
      executor: 'constant-vus',
      vus: 1,
      duration: '30s'
    },
    
    // Load test
    load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 50 },    // Ramp up
        { duration: '5m', target: 50 },    // Stay at 50 users
        { duration: '2m', target: 100 },   // Ramp up more
        { duration: '5m', target: 100 },   // Stay at 100 users
        { duration: '2m', target: 0 }      // Ramp down
      ]
    },
    
    // Stress test
    stress: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 200 },
        { duration: '2m', target: 300 },
        { duration: '5m', target: 300 },
        { duration: '2m', target: 0 }
      ]
    }
  },
  
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // 95% ต้องเสร็จใน 500ms
    errors: ['rate<0.01'],                           // Error rate < 1%
    http_req_failed: ['rate<0.01']
  }
}

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000'

export function setup() {
  // Login และได้ token
  const res = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: 'test@example.com',
    password: 'password123'
  }), { headers: { 'Content-Type': 'application/json' } })
  
  return { token: res.json('token') }
}

export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`
  }

  // Test 1: List users
  const start = new Date()
  const usersRes = http.get(`${BASE_URL}/api/v2/users?limit=20`, { headers })
  apiDuration.add(new Date() - start)

  check(usersRes, {
    'users status is 200': r => r.status === 200,
    'users has data': r => r.json('data') !== null,
    'response time < 500ms': r => r.timings.duration < 500
  }) || (errorRate.add(1), apiErrors.add(1))

  // Test 2: Get single user
  const userId = usersRes.json('data')[0]?.id
  if (userId) {
    const userRes = http.get(`${BASE_URL}/api/v2/users/${userId}`, { headers })
    check(userRes, { 'user status is 200': r => r.status === 200 })
  }

  sleep(1) // Think time
}

export function teardown(data) {
  console.log('Test completed')
}
```

### Performance Budget

```typescript
// performance-budget.ts
export const PERFORMANCE_BUDGET = {
  // Core Web Vitals
  LCP: 2500,  // Largest Contentful Paint < 2.5s
  FID: 100,   // First Input Delay < 100ms
  CLS: 0.1,   // Cumulative Layout Shift < 0.1

  // Additional metrics
  FCP: 1800,  // First Contentful Paint < 1.8s
  TTI: 3800,  // Time to Interactive < 3.8s
  TBT: 200,   // Total Blocking Time < 200ms

  // Bundle sizes (gzipped)
  jsMain: 150 * 1024,     // 150KB
  jsVendor: 200 * 1024,   // 200KB
  css: 50 * 1024,         // 50KB
  
  // API response times (ms)
  api: {
    p50: 100,   // 50th percentile
    p95: 500,   // 95th percentile
    p99: 1000   // 99th percentile
  }
}
```

---

## 7. ตัวอย่าง: Scaled Nuxt App Configuration

```typescript
// nuxt.config.ts - Production-ready configuration
export default defineNuxtConfig({
  // Compression
  nitro: {
    compressPublicAssets: {
      gzip: true,
      brotli: true
    },
    
    // Server-side caching
    storage: {
      redis: {
        driver: 'redis',
        url: process.env.REDIS_URL
      }
    },
    
    // Route rules
    routeRules: {
      '/api/**': { cors: true },
      '/health': { cache: false }
    },
    
    // Prerender static pages
    prerender: {
      crawlLinks: true,
      routes: ['/sitemap.xml', '/robots.txt']
    }
  },

  // Vite optimizations
  vite: {
    build: {
      rollupOptions: {
        output: {
          manualChunks: {
            vendor: ['vue', 'vue-router', 'pinia'],
            ui: ['@headlessui/vue', '@heroicons/vue']
          }
        }
      }
    },
    optimizeDeps: {
      include: ['lodash-es', 'date-fns']
    }
  },

  // Performance monitoring
  modules: ['@nuxtjs/web-vitals'],
  
  webVitals: {
    provider: 'log',
    debug: false,
    disabled: false
  }
})
```

### Health Check Endpoint

```typescript
// server/api/health.get.ts
export default defineEventHandler(async (event) => {
  const checks = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || '1.0.0',
    checks: {
      database: false,
      redis: false,
      memory: false
    }
  }

  // Database check
  try {
    await prisma.$queryRaw`SELECT 1`
    checks.checks.database = true
  } catch {}

  // Redis check
  try {
    await redis.ping()
    checks.checks.redis = true
  } catch {}

  // Memory check
  const memUsage = process.memoryUsage()
  const memUsedMB = memUsage.rss / 1024 / 1024
  checks.checks.memory = memUsedMB < 512 // OK ถ้าใช้น้อยกว่า 512MB

  const allHealthy = Object.values(checks.checks).every(Boolean)
  
  if (!allHealthy) {
    setResponseStatus(event, 503)
    checks.status = 'degraded'
  }

  return checks
})
```

---

## สรุป

Performance at Scale ต้องการ:
1. **Stateless design** - ให้ scale ได้ง่าย
2. **Database optimization** - Index, connection pooling, read replicas
3. **CDN** - ลด latency ด้วย edge caching
4. **Cache everywhere** - Redis, HTTP cache headers
5. **Load testing** - ทดสอบก่อน production
6. **Performance budget** - กำหนด threshold และวัดผลเสมอ
