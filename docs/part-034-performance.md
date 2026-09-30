# Part 34: Performance Optimization

## ทำไม Performance ถึงสำคัญ?

Performance ของแอปพลิเคชันส่งผลโดยตรงต่อ:
- **User Experience** - ผู้ใช้ที่รอนานเกิน 3 วินาทีมักออกจากเว็บ
- **SEO** - Google ใช้ Core Web Vitals ในการจัดอันดับ
- **Business** - ทุก 100ms ที่ลดลงเพิ่ม conversion rate ได้ถึง 1%

---

## 1. Vue Performance Tips

### หลีกเลี่ยงการสร้าง Reactive Object ที่ใหญ่เกินความจำเป็น

```ts
// ❌ ไม่ดี - reactive ทั้ง array ขนาดใหญ่
const bigList = reactive(Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  name: `Item ${i}`,
  data: { /* nested objects */ }
})))

// ✅ ดีกว่า - ใช้ shallowRef สำหรับ list ขนาดใหญ่
const bigList = shallowRef(Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  name: `Item ${i}`
})))

// ✅ หรือใช้ markRaw สำหรับ object ที่ไม่ต้องการ reactivity
import { markRaw } from 'vue'

const nonReactiveData = markRaw({
  // data ที่ไม่เปลี่ยนแปลง เช่น config, constants
  apiConfig: { baseUrl: '/api', timeout: 5000 }
})
```

### Computed vs Method

```ts
// ❌ ไม่ดี - ใช้ method ใน template จะ re-execute ทุกครั้ง render
const expensiveResult = () => {
  return heavyCalculation(data.value) // ทุกครั้งที่ component re-render
}

// ✅ ดีกว่า - computed จะ cache ผลลัพธ์
const expensiveResult = computed(() => {
  return heavyCalculation(data.value) // คำนวณใหม่เฉพาะเมื่อ data เปลี่ยน
})
```

### ใช้ key attribute อย่างถูกต้อง

```vue
<template>
  <!-- ❌ ใช้ index เป็น key - ทำให้ Vue สับสนเมื่อ reorder -->
  <div v-for="(item, index) in items" :key="index">
    {{ item.name }}
  </div>

  <!-- ✅ ใช้ unique ID เป็น key -->
  <div v-for="item in items" :key="item.id">
    {{ item.name }}
  </div>
</template>
```

---

## 2. v-memo Directive

`v-memo` ช่วย skip การ re-render ของ sub-tree เมื่อ dependencies ไม่เปลี่ยน

```vue
<script setup lang="ts">
interface Product {
  id: number
  name: string
  price: number
  isSelected: boolean
}

const products = ref<Product[]>([])
</script>

<template>
  <!-- v-memo จะ skip re-render ถ้า id, name, price, isSelected ไม่เปลี่ยน -->
  <div
    v-for="product in products"
    :key="product.id"
    v-memo="[product.id, product.name, product.price, product.isSelected]"
  >
    <ProductCard :product="product" />
  </div>
</template>
```

```vue
<!-- v-memo กับ list ที่ซับซ้อน -->
<script setup lang="ts">
const selected = ref<number | null>(null)
const items = ref([...])
</script>

<template>
  <!-- Re-render เฉพาะ item ที่ selected status เปลี่ยน -->
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id === selected]"
    :class="{ active: item.id === selected }"
    @click="selected = item.id"
  >
    {{ item.name }}
  </div>
</template>
```

---

## 3. defineAsyncComponent (Lazy Loading)

```ts
// router/index.ts - Lazy load routes
import { createRouter, createWebHistory } from 'vue-router'
import { defineAsyncComponent } from 'vue'

export const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('@/pages/index.vue')  // lazy load
    },
    {
      path: '/admin',
      component: defineAsyncComponent({
        loader: () => import('@/pages/admin/index.vue'),
        loadingComponent: () => import('@/components/LoadingSpinner.vue'),
        errorComponent: () => import('@/components/ErrorDisplay.vue'),
        delay: 200,      // แสดง loading หลัง 200ms
        timeout: 10000   // timeout หลัง 10 วินาที
      })
    }
  ]
})
```

```vue
<!-- การใช้ defineAsyncComponent ใน component -->
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

// Lazy load component ที่ใหญ่
const HeavyChart = defineAsyncComponent({
  loader: () => import('@/components/HeavyChart.vue'),
  loadingComponent: {
    template: '<div class="skeleton h-64 w-full animate-pulse rounded" />'
  },
  delay: 300
})

// Lazy load เฉพาะเมื่อต้องการ
const showChart = ref(false)
</script>

<template>
  <button @click="showChart = true">แสดงกราฟ</button>
  
  <!-- HeavyChart จะถูก load เฉพาะเมื่อ showChart เป็น true -->
  <Suspense v-if="showChart">
    <HeavyChart />
    <template #fallback>
      <div>กำลังโหลดกราฟ...</div>
    </template>
  </Suspense>
</template>
```

---

## 4. KeepAlive Component

`<KeepAlive>` ช่วย cache component state เพื่อหลีกเลี่ยงการ re-create

```vue
<!-- App.vue - Cache route components -->
<script setup lang="ts">
import { useRoute } from 'vue-router'

const route = useRoute()
</script>

<template>
  <router-view v-slot="{ Component, route }">
    <KeepAlive 
      :include="['ProductList', 'SearchPage']"
      :max="5"
    >
      <component
        :is="Component"
        :key="route.meta.usePathKey ? route.path : undefined"
      />
    </KeepAlive>
  </router-view>
</template>
```

```vue
<!-- ProductList.vue - ใช้ lifecycle hooks ของ KeepAlive -->
<script setup lang="ts">
import { onActivated, onDeactivated } from 'vue'

// ทำงานเมื่อ component ถูก activate จาก cache
onActivated(() => {
  console.log('ProductList activated - refresh if needed')
  // ตรวจว่าต้อง refresh data ไหม
  if (needsRefresh.value) {
    fetchProducts()
  }
})

// ทำงานเมื่อ component ถูก deactivate ไปยัง cache
onDeactivated(() => {
  console.log('ProductList deactivated - saved in cache')
  // cleanup subscriptions ถ้าจำเป็น
})
</script>
```

---

## 5. Virtualize Large Lists

```bash
# ติดตั้ง vue-virtual-scroller
npm install vue-virtual-scroller
```

```vue
<!-- components/VirtualProductList.vue -->
<script setup lang="ts">
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

interface Product {
  id: number
  name: string
  price: number
  imageUrl: string | null
}

interface Props {
  products: Product[]
}

defineProps<Props>()
</script>

<template>
  <!-- แสดง 10,000 items โดยไม่ค้าง - render เฉพาะที่มองเห็น -->
  <RecycleScroller
    class="scroller"
    :items="products"
    :item-size="80"
    key-field="id"
    v-slot="{ item }"
  >
    <div class="product-row">
      <img
        v-if="item.imageUrl"
        :src="item.imageUrl"
        :alt="item.name"
        class="product-image"
      />
      <div class="product-info">
        <h3>{{ item.name }}</h3>
        <span>฿{{ item.price.toLocaleString() }}</span>
      </div>
    </div>
  </RecycleScroller>
</template>

<style>
.scroller {
  height: 600px;
  overflow-y: auto;
}

.product-row {
  display: flex;
  align-items: center;
  height: 80px;
  padding: 8px 16px;
  border-bottom: 1px solid #eee;
}
</style>
```

```vue
<!-- Dynamic size items ด้วย DynamicScroller -->
<script setup lang="ts">
import { DynamicScroller, DynamicScrollerItem } from 'vue-virtual-scroller'

const messages = ref([...]) // array ขนาดใหญ่
</script>

<template>
  <DynamicScroller
    :items="messages"
    :min-item-size="54"
    class="chat-scroller"
  >
    <template #default="{ item, index, active }">
      <DynamicScrollerItem
        :item="item"
        :active="active"
        :size-dependencies="[item.content]"
        :data-index="index"
      >
        <div class="message">
          <strong>{{ item.author }}</strong>
          <p>{{ item.content }}</p>
        </div>
      </DynamicScrollerItem>
    </template>
  </DynamicScroller>
</template>
```

---

## 6. Bundle Splitting

```ts
// nuxt.config.ts - ตั้งค่า bundle splitting
export default defineNuxtConfig({
  vite: {
    build: {
      rollupOptions: {
        output: {
          manualChunks: {
            'vendor-charts': ['chart.js', 'vue-chartjs'],
            'vendor-ui': ['@headlessui/vue', '@heroicons/vue'],
            'vendor-utils': ['lodash-es', 'date-fns']
          }
        }
      }
    }
  }
})
```

```ts
// nuxt.config.ts - ใช้ dynamic imports สำหรับ heavy libraries
export default defineNuxtConfig({
  // ไม่ import ใน global - ใช้ dynamic import แทน
  plugins: [
    // ❌ ไม่ดี
    // '~/plugins/heavyLibrary.client.ts'
  ]
})
```

```ts
// composables/useChart.ts - Lazy load chart library
export async function useChart() {
  // โหลด chart.js เฉพาะเมื่อต้องการ
  const { Chart, registerables } = await import('chart.js')
  Chart.register(...registerables)
  return Chart
}
```

---

## 7. Image Optimization

```vue
<!-- components/OptimizedImage.vue -->
<script setup lang="ts">
interface Props {
  src: string
  alt: string
  width?: number
  height?: number
  loading?: 'lazy' | 'eager'
}

const props = withDefaults(defineProps<Props>(), {
  loading: 'lazy'
})

// สร้าง srcset สำหรับ responsive images
const srcset = computed(() => {
  const sizes = [320, 640, 960, 1280]
  return sizes
    .map(w => `${props.src}?w=${w} ${w}w`)
    .join(', ')
})
</script>

<template>
  <!-- Nuxt Image component (แนะนำ) -->
  <NuxtImg
    v-if="$nuxt"
    :src="src"
    :alt="alt"
    :width="width"
    :height="height"
    :loading="loading"
    format="webp"
    quality="80"
    sizes="sm:100vw md:50vw lg:400px"
  />
  
  <!-- Fallback standard img -->
  <img
    v-else
    :src="src"
    :srcset="srcset"
    sizes="(max-width: 600px) 100vw, 50vw"
    :alt="alt"
    :width="width"
    :height="height"
    :loading="loading"
    decoding="async"
  />
</template>
```

```ts
// nuxt.config.ts - ตั้งค่า NuxtImage
export default defineNuxtConfig({
  modules: ['@nuxt/image'],
  image: {
    quality: 80,
    formats: ['webp', 'avif'],
    screens: {
      sm: 640,
      md: 768,
      lg: 1024,
      xl: 1280
    },
    provider: 'cloudinary',
    cloudinary: {
      baseURL: 'https://res.cloudinary.com/my-account/image/upload/'
    }
  }
})
```

---

## 8. Nuxt Performance Tools

```ts
// nuxt.config.ts - Performance configurations
export default defineNuxtConfig({
  // เปิดใช้ Nuxt DevTools
  devtools: { enabled: true },

  // ใช้ Nitro caching
  nitro: {
    routeRules: {
      '/api/products': { cache: { maxAge: 60 } },      // cache 60 วินาที
      '/blog/**': { swr: 3600 },                        // stale-while-revalidate
      '/admin/**': { ssr: false }                       // client-only
    }
  },

  // เปิดใช้ compression
  experimental: {
    renderJsonPayloads: true,
    inlineSSRStyles: false,    // ลด initial HTML size
  }
})
```

```ts
// server/api/products.ts - Caching ใน server
import { defineEventHandler, getQuery } from 'h3'

export default defineCachedEventHandler(async (event) => {
  const products = await fetchFromDatabase()
  return products
}, {
  maxAge: 60 * 5,  // cache 5 นาที
  name: 'products',
  getKey: (event) => {
    const query = getQuery(event)
    return `products-${JSON.stringify(query)}`
  }
})
```

---

## 9. Lighthouse Optimization

```ts
// plugins/performance.client.ts - ติดตาม Core Web Vitals
export default defineNuxtPlugin(() => {
  if (process.client) {
    // Measure LCP (Largest Contentful Paint)
    new PerformanceObserver((list) => {
      const entries = list.getEntries()
      const lastEntry = entries[entries.length - 1]
      console.log('LCP:', lastEntry.startTime)
    }).observe({ entryTypes: ['largest-contentful-paint'] })

    // Measure CLS (Cumulative Layout Shift)
    let clsScore = 0
    new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!(entry as any).hadRecentInput) {
          clsScore += (entry as any).value
        }
      }
      console.log('CLS:', clsScore)
    }).observe({ entryTypes: ['layout-shift'] })

    // Measure FID/INP (Interaction to Next Paint)
    new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        console.log('INP:', (entry as any).processingStart - entry.startTime)
      }
    }).observe({ entryTypes: ['first-input'] })
  }
})
```

```ts
// nuxt.config.ts - Critical CSS และ Font optimization
export default defineNuxtConfig({
  app: {
    head: {
      link: [
        // Preconnect to font servers
        { rel: 'preconnect', href: 'https://fonts.googleapis.com' },
        { rel: 'preconnect', href: 'https://fonts.gstatic.com', crossorigin: 'anonymous' },
        // Preload critical font
        {
          rel: 'preload',
          as: 'font',
          type: 'font/woff2',
          href: '/fonts/sarabun.woff2',
          crossorigin: 'anonymous'
        }
      ]
    }
  },
  
  // CSS
  css: ['@/assets/css/critical.css'],
  
  vite: {
    css: {
      devSourcemap: false  // ปิด sourcemap ใน production
    }
  }
})
```

---

## 10. ตัวอย่าง: Performance Profiling และแก้ไข

```vue
<!-- BEFORE: Component ที่มีปัญหา Performance -->
<script setup lang="ts">
// ❌ ปัญหาที่ 1: Expensive computation ใน computed ที่ไม่ได้ cache
function filterAndSort(items: any[]) {
  return items
    .filter(item => item.active)
    .sort((a, b) => b.date - a.date)
    .slice(0, 50)
}

const products = ref([/* 10000 items */])

// ❌ ปัญหาที่ 2: Watch ที่กว้างเกินไป
watch(products, () => {
  // ทำงานทุกครั้งที่ products เปลี่ยน แม้แค่ 1 item
  syncToServer(products.value)
}, { deep: true })
</script>

<template>
  <!-- ❌ ปัญหาที่ 3: ไม่มี key ที่ถูกต้อง -->
  <div v-for="(p, i) in filterAndSort(products)" :key="i">
    {{ p.name }}
  </div>
</template>
```

```vue
<!-- AFTER: แก้ไข Performance ปัญหา -->
<script setup lang="ts">
import { RecycleScroller } from 'vue-virtual-scroller'

const products = ref([/* 10000 items */])

// ✅ แก้ที่ 1: ใช้ computed พร้อม memo
const visibleProducts = computed(() => {
  return products.value
    .filter(item => item.active)
    .sort((a, b) => b.date - a.date)
    .slice(0, 50)
})

// ✅ แก้ที่ 2: Watch เฉพาะ IDs ไม่ใช่ deep watch
const productIds = computed(() => products.value.map(p => p.id).join(','))
watch(productIds, () => {
  syncToServer(products.value)
})

// ✅ แก้ที่ 3: Debounce การ sync
const debouncedSync = useDebounceFn(syncToServer, 1000)
</script>

<template>
  <!-- ✅ ใช้ Virtual Scroller + proper key -->
  <RecycleScroller
    :items="visibleProducts"
    :item-size="60"
    key-field="id"
    v-slot="{ item }"
  >
    <div v-memo="[item.id, item.name, item.price]">
      {{ item.name }} - ฿{{ item.price }}
    </div>
  </RecycleScroller>
</template>
```

---

## สรุป

Performance Optimization ใน Vue/Nuxt มีหลายระดับ:

1. **Vue Tips** - หลีกเลี่ยง reactive ที่ไม่จำเป็น, ใช้ computed แทน method
2. **v-memo** - Skip re-render สำหรับ list items ที่ไม่เปลี่ยน
3. **Lazy Loading** - โหลด component เฉพาะเมื่อต้องการด้วย `defineAsyncComponent`
4. **KeepAlive** - Cache component state เพื่อหลีกเลี่ยง re-creation
5. **Virtual Lists** - แสดง 100,000 items ด้วย `vue-virtual-scroller`
6. **Bundle Splitting** - แยก bundle ด้วย `manualChunks`
7. **Image Optimization** - ใช้ `@nuxt/image` พร้อม WebP/AVIF
8. **Server Caching** - ใช้ Nitro `routeRules` cache
9. **Lighthouse** - ติดตาม Core Web Vitals และแก้ไข
