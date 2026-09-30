# Part 85: Interview Preparation

## การเตรียมตัวสัมภาษณ์งาน Vue.js/Nuxt.js

การเตรียมตัวที่ดีจะช่วยให้มั่นใจมากขึ้นในการสัมภาษณ์ เน้นทั้ง Technical Knowledge, Problem-solving และ Communication Skills

---

## 1. Vue.js Interview Questions: Basic

### คำถามพื้นฐาน

**Q: Vue 3 Composition API vs Options API ต่างกันอย่างไร?**

```typescript
// Options API
export default {
  data() {
    return { count: 0 }
  },
  computed: {
    doubled() { return this.count * 2 }
  },
  methods: {
    increment() { this.count++ }
  }
}

// Composition API - ตอบ
// 1. Logic ที่เกี่ยวข้องกันอยู่ด้วยกัน (ไม่กระจาย)
// 2. Reusable logic ผ่าน composables
// 3. TypeScript support ดีกว่า
// 4. Tree-shakeable
export function useCounter() {
  const count = ref(0)
  const doubled = computed(() => count.value * 2)
  const increment = () => count.value++
  return { count, doubled, increment }
}
```

**Q: ref vs reactive ต่างกันอย่างไร?**

```typescript
// ref - สำหรับ primitive values
const count = ref(0)
count.value++ // ต้องใช้ .value

// reactive - สำหรับ objects
const state = reactive({ count: 0, name: 'John' })
state.count++ // ไม่ต้องใช้ .value

// ข้อควรระวัง:
const { count } = reactive({ count: 0 }) // ❌ reactivity หาย!
const countRef = toRef(reactive({ count: 0 }), 'count') // ✅
```

**Q: computed vs watch ใช้เมื่อไหร่?**

```typescript
// computed: สำหรับ derived state ที่ sync
const fullName = computed(() => `${firstName.value} ${lastName.value}`)

// watch: สำหรับ side effects เมื่อ state เปลี่ยน
watch(userId, async (newId) => {
  userData.value = await fetchUser(newId) // async side effect
}, { immediate: true })

// watchEffect: ติดตาม dependencies อัตโนมัติ
watchEffect(async () => {
  userData.value = await fetchUser(userId.value) // ติดตาม userId อัตโนมัติ
})
```

**Q: Vue lifecycle hooks มีอะไรบ้างและใช้เมื่อไหร่?**

```typescript
// onMounted: DOM พร้อมแล้ว - fetch data, init libraries
onMounted(() => {
  chart = new Chart(canvasRef.value)
})

// onUpdated: component re-rendered - ระวัง infinite loop
onUpdated(() => {
  console.log('DOM updated')
})

// onUnmounted: cleanup - remove listeners, cancel requests
onUnmounted(() => {
  chart.destroy()
  controller.abort()
})

// onErrorCaptured: catch errors from children
onErrorCaptured((err) => {
  logError(err)
  return false // prevent propagation
})
```

---

## 2. Vue.js Interview Questions: Intermediate

**Q: Vue Reactivity System ทำงานอย่างไร?**

```typescript
// Vue 3 ใช้ Proxy สำหรับ reactivity
// เมื่อเข้าถึง property → track dependency
// เมื่อเปลี่ยน property → trigger effects

const state = reactive({ count: 0 })

// Internally:
// new Proxy({ count: 0 }, {
//   get(target, key) {
//     track(target, key)  // ลงทะเบียน dependency
//     return target[key]
//   },
//   set(target, key, value) {
//     target[key] = value
//     trigger(target, key)  // trigger effects
//     return true
//   }
// })

// Effect ที่ subscribe อยู่จะ run เมื่อ state.count เปลี่ยน
effect(() => {
  console.log(state.count) // subscribe to state.count
})

state.count++ // trigger effect → console.log runs again
```

**Q: อธิบาย Virtual DOM และ Reconciliation**

```
1. Vue render function สร้าง Virtual DOM tree (VNode objects)
2. เมื่อ state เปลี่ยน → render function ทำงานใหม่ → VNode tree ใหม่
3. Vue เปรียบเทียบ old VNode กับ new VNode (diffing)
4. อัปเดตเฉพาะ DOM ที่เปลี่ยนจริง (patching)

Vue 3 optimizations:
- Static hoisting: nodes ที่ static ไม่ต้อง diff
- Patch flags: บอก runtime ว่าอะไรเปลี่ยน
- Block tree: ติดตามเฉพาะ dynamic nodes
```

**Q: สร้าง Custom Directive**

```typescript
// ตัวอย่าง v-focus directive
const vFocus = {
  mounted(el: HTMLElement) {
    el.focus()
  }
}

// v-click-outside
const vClickOutside = {
  mounted(el: HTMLElement, binding: DirectiveBinding) {
    const handler = (event: Event) => {
      if (!el.contains(event.target as Node)) {
        binding.value(event)
      }
    }
    el._clickOutsideHandler = handler
    document.addEventListener('click', handler)
  },
  unmounted(el: HTMLElement) {
    document.removeEventListener('click', el._clickOutsideHandler)
  }
}

// ใช้งาน
// <div v-click-outside="closeMenu">...</div>
```

**Q: Teleport คืออะไร ใช้เมื่อไหร่?**

```vue
<!-- Modal ที่ต้อง render นอก component hierarchy -->
<template>
  <Teleport to="body">
    <div v-if="isOpen" class="modal-overlay">
      <div class="modal">
        <slot />
      </div>
    </div>
  </Teleport>
</template>

<!-- ใช้สำหรับ: Modal, Toast, Tooltip, Dropdown
     เหตุผล: หลีกเลี่ยง z-index, overflow:hidden issues -->
```

---

## 3. Nuxt.js Interview Questions

**Q: SSR vs SSG vs CSR ต่างกันอย่างไร?**

```
SSR (Server Side Rendering):
- HTML render บน server ทุก request
- ดี: SEO, First load fast, dynamic data
- ไม่ดี: Server load สูง, ช้ากว่า static

SSG (Static Site Generation):
- HTML render ตอน build time
- ดี: เร็วมาก, cheap hosting, SEO
- ไม่ดี: ต้อง rebuild เมื่อข้อมูลเปลี่ยน

CSR (Client Side Rendering):
- HTML render ใน browser
- ดี: Interactive, no server needed
- ไม่ดี: SEO แย่, Slow first load

ISR (Incremental Static Regeneration):
- Hybrid: static + revalidate
- ดีที่สุดสำหรับ most use cases
```

**Q: useFetch vs $fetch ต่างกันอย่างไร?**

```typescript
// useFetch - สำหรับใช้ใน setup() หรือ <script setup>
// - ทำงานทั้ง server และ client
// - Auto-deduplication
// - Return reactive state
const { data, pending, error, refresh } = useFetch('/api/users')

// $fetch - สำหรับ event handlers, stores, functions
// - ไม่ return reactive state
// - ใช้ใน functions/actions
async function createUser(data) {
  const user = await $fetch('/api/users', {
    method: 'POST',
    body: data
  })
  return user
}
```

**Q: Nuxt Middleware ทำงานอย่างไร?**

```typescript
// Route middleware - client-side navigation guard
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to) => {
  const { isLoggedIn } = useAuth()
  
  if (!isLoggedIn.value) {
    return navigateTo('/login')
  }
})

// Server middleware - runs on every server request
// server/middleware/tenant.ts
export default defineEventHandler((event) => {
  const host = getHeader(event, 'host')
  // Set tenant context
})
```

---

## 4. Algorithm Questions กับ Vue Context

**Q: Implement debounce สำหรับ search input**

```typescript
// Debounce implementation
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout>

  return function (...args: Parameters<T>) {
    clearTimeout(timeoutId)
    timeoutId = setTimeout(() => fn(...args), delay)
  }
}

// Vue composable
function useDebouncedSearch(delay = 300) {
  const query = ref('')
  const results = ref([])
  const loading = ref(false)

  const search = debounce(async (q: string) => {
    if (!q.trim()) { results.value = []; return }
    
    loading.value = true
    try {
      results.value = await $fetch('/api/search', { params: { q } })
    } finally {
      loading.value = false
    }
  }, delay)

  watch(query, search)

  return { query, results, loading }
}
```

**Q: Implement infinite scroll**

```typescript
function useInfiniteScroll(loadMore: () => Promise<boolean>) {
  const sentinel = ref<HTMLElement | null>(null)
  const loading = ref(false)
  const hasMore = ref(true)

  const observer = new IntersectionObserver(async ([entry]) => {
    if (entry.isIntersecting && !loading.value && hasMore.value) {
      loading.value = true
      try {
        hasMore.value = await loadMore()
      } finally {
        loading.value = false
      }
    }
  }, { threshold: 0.1 })

  onMounted(() => {
    if (sentinel.value) observer.observe(sentinel.value)
  })

  onUnmounted(() => observer.disconnect())

  return { sentinel, loading, hasMore }
}
```

---

## 5. System Design Questions

**Q: ออกแบบ Real-time Notification System**

```
Architecture:
1. Client → WebSocket → Server
2. Server → Pub/Sub (Redis) → Worker
3. Worker → Process notification → Redis
4. Server → Push to specific user via WebSocket

Components:
- WebSocket server (Nitro/Socket.io)
- Notification queue (Redis pub/sub)
- Notification store (Pinia)
- UI components (Bell icon, notification list)

Considerations:
- ผู้ใช้ offline: เก็บใน DB อ่านเมื่อ reconnect
- Scale: Redis pub/sub รองรับ multiple servers
- Security: ตรวจสอบ tenant ใน every notification
```

**Q: ออกแบบ File Upload System**

```
Flow:
1. Client → Request presigned URL จาก server
2. Server → สร้าง S3 presigned URL → return ให้ client
3. Client → Upload โดยตรงไปยัง S3 (ไม่ผ่าน server)
4. S3 → Trigger Lambda/webhook เมื่อ upload สำเร็จ
5. Server → Process file (resize image, scan virus, etc.)
6. Server → Update database

Vue Implementation:
```

```typescript
async function uploadFile(file: File) {
  // 1. ขอ presigned URL
  const { uploadUrl, fileKey } = await $fetch('/api/files/presigned-url', {
    method: 'POST',
    body: { filename: file.name, contentType: file.type }
  })

  // 2. Upload ไปยัง S3
  await fetch(uploadUrl, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': file.type }
  })

  // 3. แจ้ง server ว่า upload สำเร็จ
  return $fetch('/api/files/confirm', {
    method: 'POST',
    body: { fileKey }
  })
}
```

---

## 6. Coding Challenges

**Challenge: สร้าง Virtual List Component**

```vue
<!-- VirtualList.vue - render เฉพาะ items ที่อยู่ใน viewport -->
<template>
  <div
    ref="containerRef"
    class="virtual-list"
    :style="{ height: `${containerHeight}px`, overflow: 'auto' }"
    @scroll="handleScroll"
  >
    <!-- Spacer เพื่อให้ scroll height ถูกต้อง -->
    <div :style="{ height: `${totalHeight}px`, position: 'relative' }">
      <!-- Render เฉพาะ visible items -->
      <div
        v-for="item in visibleItems"
        :key="item.id"
        :style="{ 
          position: 'absolute',
          top: `${item.offset}px`,
          width: '100%'
        }"
      >
        <slot :item="item.data" />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts" generic="T extends { id: string | number }">
const props = defineProps<{
  items: T[]
  itemHeight: number
  containerHeight: number
  overscan?: number
}>()

const overscan = computed(() => props.overscan ?? 3)

const scrollTop = ref(0)
const containerRef = ref<HTMLElement>()

const totalHeight = computed(() => props.items.length * props.itemHeight)

const visibleItems = computed(() => {
  const start = Math.floor(scrollTop.value / props.itemHeight)
  const visibleCount = Math.ceil(props.containerHeight / props.itemHeight)
  
  const startIndex = Math.max(0, start - overscan.value)
  const endIndex = Math.min(
    props.items.length - 1,
    start + visibleCount + overscan.value
  )

  return props.items.slice(startIndex, endIndex + 1).map((data, i) => ({
    id: data.id,
    data,
    offset: (startIndex + i) * props.itemHeight
  }))
})

function handleScroll(e: Event) {
  scrollTop.value = (e.target as HTMLElement).scrollTop
}
</script>
```

---

## 7. Behavioral Questions

**Q: บอกตัวอย่างที่คุณแก้ปัญหาที่ยากมาก**

ตัวอย่างโครงสร้างคำตอบ (STAR Method):
```
Situation: "ในโปรเจกต์ที่ผ่านมา performance ของ dashboard ช้ามากเมื่อมีข้อมูล 10,000+ rows"

Task: "ต้องทำให้ load time < 2 วินาที โดยไม่ refactor ทั้ง codebase"

Action: 
1. Profiled เพื่อหา bottleneck → N+1 queries
2. ใช้ Prisma include แทน nested queries
3. เพิ่ม Redis caching สำหรับ expensive queries
4. ใช้ Virtual List สำหรับ large tables
5. Lazy load charts ด้วย Intersection Observer

Result: "Load time ลดจาก 8 วินาที เหลือ 1.2 วินาที (85% improvement)"
```

---

## 8. การเตรียม Portfolio

### Portfolio Checklist

```markdown
## Portfolio Project Requirements

### GitHub Profile
- [ ] README.md ที่ดี (bio, skills, projects)
- [ ] Pinned repositories (2-6 projects ที่ดีที่สุด)
- [ ] Regular contributions (green squares)
- [ ] Contributing to open source

### For Each Project
- [ ] Live demo URL
- [ ] Good README with screenshots
- [ ] Technologies used
- [ ] Architecture decisions explained
- [ ] Test coverage > 70%
- [ ] CI/CD pipeline

### Recommended Projects
1. SaaS Application (Full-stack)
   - Multi-tenant, Stripe billing, authentication
   
2. E-commerce (Nuxt + Stripe)
   - Cart, checkout, admin panel
   
3. Open Source Package
   - Vue plugin หรือ Nuxt module
   
4. Real-time Application
   - Chat, notifications, live dashboard
```

---

## 9. Mock Interview Scenarios

### Scenario 1: Component Optimization

```
"เรา component ที่ re-render บ่อยมากจนทำให้ UX แย่ คุณจะแก้ยังไง?"

Answer Framework:
1. Profile ก่อน (Vue DevTools, performance.mark)
2. ตรวจสอบ unnecessary re-renders
3. เพิ่ม computed ที่เหมาะสม
4. ใช้ v-memo สำหรับ list items
5. shallowRef สำหรับ large objects ที่ไม่ต้อง deep watch
6. debounce/throttle events ที่ fire บ่อย
```

### Scenario 2: State Management

```
"Application มี state ซับซ้อนมาก ทำยังไงให้จัดการได้ง่าย?"

Answer:
1. Categorize state:
   - UI state → component-local ref/reactive
   - Shared UI state → Pinia store
   - Server state → useQuery (TanStack Query)
   - URL state → useRoute/useRouter
   
2. ใช้ Pinia modules pattern
3. Separate concerns (auth store, cart store, etc.)
4. ใช้ TypeScript สำหรับ type safety
```

---

## สรุป

การเตรียมสัมภาษณ์ที่ดี:
1. **รู้จริง** - ไม่ท่อง แต่เข้าใจจริงๆ
2. **STAR method** - สำหรับ behavioral questions
3. **Portfolio** - project จริงที่ deploy แล้ว
4. **Practice** - ทำ coding challenges บน LeetCode, HackerRank
5. **Ask questions** - แสดงว่าสนใจและคิดลึก
6. **Mock interviews** - ฝึกกับเพื่อนหรือ interviewing.io
