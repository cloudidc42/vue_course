# Part 52: Server-Side Rendering Deep Dive

## ทำความเข้าใจ Rendering Modes

Nuxt.js รองรับหลาย rendering strategies ที่ตอบโจทย์แต่ละ use case:

```
CSR  (Client-Side Rendering)  - render ใน browser ทั้งหมด
SSR  (Server-Side Rendering)  - render ที่ server ทุก request
SSG  (Static Site Generation) - render ตอน build time
ISR  (Incremental Static Regeneration) - hybrid ระหว่าง SSR กับ SSG
```

## 1. SSR vs CSR vs SSG vs ISR

### เปรียบเทียบแต่ละ mode

```
| Feature          | CSR      | SSR      | SSG      | ISR      |
|------------------|----------|----------|----------|----------|
| Initial Load     | ช้า     | เร็ว    | เร็วมาก  | เร็วมาก  |
| SEO              | แย่     | ดี      | ดีมาก    | ดีมาก    |
| Dynamic Content  | ✅       | ✅       | ❌        | ✅ (TTL) |
| Server Load      | ต่ำ     | สูง     | ไม่มี    | ต่ำ      |
| Build Time       | เร็ว   | เร็ว    | ช้า      | เร็ว     |
| Real-time Data   | ✅       | ✅       | ❌        | บาง      |
```

### ตั้งค่าใน Nuxt

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // Global SSR (default)
  ssr: true,
  
  // Hybrid rendering per route
  routeRules: {
    // Homepage - prerendered at build time
    '/': { prerender: true },
    
    // Blog posts - ISG with 1 hour cache
    '/blog/**': { swr: 3600 },
    
    // Admin panel - CSR only (client-side)
    '/admin/**': { ssr: false },
    
    // API routes - no caching
    '/api/**': { cors: true, headers: { 'cache-control': 'no-store' } },
    
    // Static pages
    '/about': { prerender: true },
    '/contact': { prerender: true },
    
    // Product pages - cache 1 hour
    '/products/**': { isr: 3600 },
  }
})
```

## 2. Hydration Process

```
Server:
1. รับ Request
2. Execute Vue components
3. สร้าง HTML string
4. ส่ง HTML + JSON data ไปยัง Client

Client:
1. แสดง HTML ทันที (Fast First Contentful Paint)
2. โหลด JavaScript
3. Vue "hydrates" - attach event handlers กับ existing DOM
4. แอปพร้อมใช้งาน (Time to Interactive)
```

### Hydration Mismatch Issues

```vue
<!-- ปัญหา: server กับ client render ไม่ตรงกัน -->
<template>
  <!-- ❌ ผิด! Date.now() ต่างกันระหว่าง server และ client -->
  <p>เวลาปัจจุบัน: {{ Date.now() }}</p>
  
  <!-- ❌ ผิด! window ไม่มีใน server -->
  <p v-if="window.innerWidth > 768">Desktop View</p>
  
  <!-- ✅ ถูก! ใช้ ClientOnly -->
  <ClientOnly>
    <p>เวลา: {{ currentTime }}</p>
    <template #fallback>
      <p>กำลังโหลด...</p>
    </template>
  </ClientOnly>
</template>

<script setup>
const currentTime = ref('')

// อัปเดตเฉพาะ client-side
onMounted(() => {
  currentTime.value = new Date().toLocaleTimeString('th-TH')
  setInterval(() => {
    currentTime.value = new Date().toLocaleTimeString('th-TH')
  }, 1000)
})
</script>
```

## 3. Server/Client Code Separation

```typescript
// composables/usePlatform.ts

// ตรวจสอบว่าอยู่ที่ไหน
export const useIsServer = () => !process.client
export const useIsClient = () => process.client

// ใช้ window อย่างปลอดภัย
export const useWindowSize = () => {
  const width = ref(0)
  const height = ref(0)
  
  if (process.client) {
    width.value = window.innerWidth
    height.value = window.innerHeight
    
    const handleResize = () => {
      width.value = window.innerWidth
      height.value = window.innerHeight
    }
    
    window.addEventListener('resize', handleResize)
    onUnmounted(() => window.removeEventListener('resize', handleResize))
  }
  
  return { width, height }
}
```

```typescript
// server/utils/db.ts - รันเฉพาะ server
export const getServerData = async (id: string) => {
  // ไฟล์นี้ถูก import เฉพาะใน server/
  const data = await prisma.item.findUnique({ where: { id } })
  return data
}
```

```typescript
// plugins/analytics.client.ts - suffix .client.ts = client only
export default defineNuxtPlugin(() => {
  // Google Analytics - รันเฉพาะ client
  window.gtag?.('config', 'GA-XXXXXXXX')
  
  const router = useRouter()
  router.afterEach((to) => {
    window.gtag?.('event', 'page_view', {
      page_path: to.path
    })
  })
})
```

```typescript
// plugins/logger.server.ts - suffix .server.ts = server only
export default defineNuxtPlugin((nuxtApp) => {
  // Server-side logging
  nuxtApp.hook('app:rendered', (ctx) => {
    console.log(`[SSR] Rendered: ${ctx.ssrContext?.url}`)
  })
})
```

## 4. useSSRData() Patterns

```typescript
// pattern 1: useAsyncData
const { data: posts, pending, error, refresh } = await useAsyncData(
  'posts',
  () => $fetch('/api/posts'),
  {
    // ตั้งค่า cache
    getCachedData: (key, nuxtApp) => {
      return nuxtApp.payload.data[key] ?? nuxtApp.static.data[key]
    },
    
    // Transform data
    transform: (data) => data.posts,
    
    // Pick specific fields (ลด payload size)
    pick: ['id', 'title', 'slug'],
    
    // Watch dependencies
    watch: [currentPage, categoryFilter],
    
    // ไม่ lazy โหลด - รอผลก่อน
    lazy: false,
    
    // Server only
    server: true,
    
    // Client only
    // client: true,
  }
)
```

```typescript
// pattern 2: useFetch
const { data: post } = await useFetch(`/api/posts/${route.params.slug}`, {
  key: `post-${route.params.slug}`,
  server: true,
  
  onRequest({ request, options }) {
    // เพิ่ม auth header
    options.headers = options.headers || {}
    options.headers.authorization = `Bearer ${userToken.value}`
  },
  
  onResponseError({ response }) {
    if (response.status === 404) {
      throw createError({ statusCode: 404, statusMessage: 'ไม่พบ' })
    }
  }
})
```

```typescript
// pattern 3: useNuxtData - share data ระหว่าง components
// parent component
const { data } = await useAsyncData('shared-posts', () => $fetch('/api/posts'))

// child component - ใช้ data เดิม ไม่ต้องเรียก API ใหม่
const { data: posts } = useNuxtData('shared-posts')
```

## 5. Streaming SSR

Streaming SSR ส่ง HTML เป็น chunks แทนที่จะรอทั้งหน้า ทำให้ Time to First Byte (TTFB) เร็วขึ้น

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  experimental: {
    renderJsonPayloads: true
  }
})
```

```vue
<!-- ใช้ Suspense กับ Streaming -->
<template>
  <div>
    <!-- ส่วนนี้ render ทันที -->
    <header>
      <Logo />
      <Navigation />
    </header>
    
    <!-- ส่วนนี้ stream เมื่อพร้อม -->
    <Suspense>
      <AsyncHeavyContent />
      <template #fallback>
        <ContentSkeleton />
      </template>
    </Suspense>
    
    <!-- ส่วนนี้ render ได้เลย -->
    <footer>
      <Footer />
    </footer>
  </div>
</template>
```

## 6. Edge SSR

```typescript
// nuxt.config.ts - ใช้ Edge Runtime
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages' // หรือ 'vercel-edge', 'netlify-edge'
  },
  
  routeRules: {
    // รัน SSR ที่ Edge (ใกล้ผู้ใช้)
    '/': { edge: true },
    '/products/**': { edge: true }
  }
})
```

```typescript
// server/api/products.ts - Edge compatible
export default defineEventHandler(async (event) => {
  // ต้องใช้ Web APIs (ไม่ใช่ Node.js specific APIs)
  const url = new URL(event.node.req.url!, 'http://localhost')
  const page = url.searchParams.get('page') || '1'
  
  // ใช้ fetch แทน node-fetch
  const data = await fetch(`https://api.example.com/products?page=${page}`)
  return data.json()
})
```

## 7. ตัวอย่าง: SSR Performance Comparison

```vue
<!-- pages/ssr-demo.vue -->
<template>
  <div class="ssr-demo">
    <h1>SSR Performance Demo</h1>
    
    <!-- แสดงข้อมูลที่มาจาก server -->
    <section class="metrics">
      <div class="metric-card">
        <h3>Server Render Time</h3>
        <p class="metric-value">{{ serverRenderTime }}ms</p>
      </div>
      <div class="metric-card">
        <h3>Hydration Time</h3>
        <p class="metric-value">{{ hydrationTime }}ms</p>
      </div>
      <div class="metric-card">
        <h3>Total Load Time</h3>
        <p class="metric-value">{{ totalLoadTime }}ms</p>
      </div>
    </section>
    
    <!-- Deferred/Lazy loaded sections -->
    <section>
      <h2>Immediate Content (SSR)</h2>
      <div v-if="immediateData">
        <PostCard v-for="post in immediateData" :key="post.id" :post="post" />
      </div>
    </section>
    
    <section>
      <h2>Lazy Content (loads after hydration)</h2>
      <ClientOnly>
        <LazyDataSection />
        <template #fallback>
          <SkeletonLoader count="3" />
        </template>
      </ClientOnly>
    </section>
  </div>
</template>

<script setup>
// Timing เพื่อวัด SSR performance
const serverStartTime = useState('server-start-time', () => Date.now())
const serverRenderTime = ref(0)
const hydrationTime = ref(0)
const totalLoadTime = ref(0)

// SSR Data - rendered ที่ server
const { data: immediateData } = await useAsyncData(
  'immediate-posts',
  () => $fetch('/api/posts?limit=5')
)

// Measure hydration time
const hydrateStart = Date.now()
onMounted(() => {
  hydrationTime.value = Date.now() - hydrateStart
  
  // วัดจาก navigation timing API
  if (window.performance) {
    const perfEntries = performance.getEntriesByType('navigation')
    if (perfEntries.length > 0) {
      const nav = perfEntries[0] as PerformanceNavigationTiming
      serverRenderTime.value = Math.round(nav.responseStart - nav.requestStart)
      totalLoadTime.value = Math.round(nav.loadEventEnd - nav.startTime)
    }
  }
})

// SEO Meta
useSeoMeta({
  title: 'SSR Performance Demo',
  description: 'ทดสอบ Server-Side Rendering performance'
})
</script>
```

```typescript
// server/api/posts/[slug].ts - Server-side with caching
export default defineCachedEventHandler(
  async (event) => {
    const slug = getRouterParam(event, 'slug')!
    
    const post = await prisma.post.findUnique({
      where: { slug, published: true },
      include: {
        author: { select: { name: true, avatar: true } },
        category: true
      }
    })
    
    if (!post) {
      throw createError({ statusCode: 404 })
    }
    
    return post
  },
  {
    maxAge: 60,             // Cache 1 นาที
    staleMaxAge: 3600,      // Stale for 1 hour
    swr: true,              // Stale While Revalidate
    name: 'post',
    getKey: (event) => getRouterParam(event, 'slug')!
  }
)
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **SSR vs CSR vs SSG vs ISR** - เปรียบเทียบข้อดีข้อเสีย
2. **Hydration Process** - วิธี Vue attach กับ server HTML
3. **Server/Client Separation** - แยก code ได้อย่างถูกต้อง
4. **useSSRData Patterns** - useAsyncData, useFetch, useNuxtData
5. **Streaming SSR** - ส่ง HTML เป็น chunks
6. **Edge SSR** - render ที่ Edge network
7. **Performance Demo** - วัดและแสดง SSR metrics
