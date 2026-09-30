# Part 24: Nuxt Data Fetching

## ภาพรวม Data Fetching ใน Nuxt

Nuxt มี composables สำหรับ data fetching ที่ทำงานได้ทั้ง server และ client โดยไม่ต้องเขียนโค้ดแยก

```
┌─────────────────────────────────────────────────────────┐
│              Data Fetching Methods                       │
├──────────────────┬──────────────────────────────────────┤
│   useFetch()     │  Universal - ทำงานทั้ง SSR + Client  │
│   useAsyncData() │  Universal - Custom async logic       │
│   $fetch         │  Client-side only (หรือ server API)  │
│   useLazyFetch() │  Non-blocking - ไม่ wait render      │
└──────────────────┴──────────────────────────────────────┘
```

---

## useFetch() - Universal data fetching

`useFetch()` คือ wrapper ของ `useAsyncData()` + `$fetch` ที่ทำงานได้ทั้ง server-side และ client-side

### Syntax พื้นฐาน

```typescript
const {
  data,        // Ref<T | null> - ข้อมูลที่ได้
  pending,     // Ref<boolean> - กำลัง loading
  error,       // Ref<Error | null> - error ถ้ามี
  refresh,     // () => Promise<void> - refresh ข้อมูล
  execute,     // () => Promise<void> - execute manually
  status       // Ref<'idle' | 'pending' | 'success' | 'error'>
} = await useFetch(url, options)
```

### ตัวอย่างพื้นฐาน

```vue
<!-- pages/posts/index.vue -->
<script setup lang="ts">
interface Post {
  id: number
  title: string
  excerpt: string
  author: string
  createdAt: string
  category: string
}

interface PostsResponse {
  posts: Post[]
  total: number
  page: number
  pageSize: number
}

// Basic fetch
const { data: postsData, pending, error } = await useFetch<PostsResponse>('/api/posts')
</script>

<template>
  <div>
    <div v-if="pending" class="loading">กำลังโหลด...</div>
    
    <div v-else-if="error" class="error">
      เกิดข้อผิดพลาด: {{ error.message }}
    </div>
    
    <div v-else>
      <p>พบ {{ postsData?.total }} บทความ</p>
      <div v-for="post in postsData?.posts" :key="post.id">
        <h2>{{ post.title }}</h2>
        <p>{{ post.excerpt }}</p>
      </div>
    </div>
  </div>
</template>
```

### Options ของ useFetch

```vue
<script setup lang="ts">
const page = ref(1)
const category = ref('vue')

const { data, pending, refresh } = await useFetch('/api/posts', {
  // Query parameters (reactive!)
  query: computed(() => ({
    page: page.value,
    category: category.value,
    limit: 10
  })),
  
  // Headers
  headers: {
    'Authorization': `Bearer ${useAuthStore().token}`,
    'Content-Type': 'application/json'
  },
  
  // Method
  method: 'GET',  // 'POST', 'PUT', 'DELETE', 'PATCH'
  
  // Body สำหรับ POST/PUT
  // body: { title: 'New Post', content: '...' },
  
  // Unique key for caching
  key: computed(() => `posts-${category.value}-page-${page.value}`),
  
  // ดึงข้อมูลเฉพาะส่วนที่ต้องการ (ลด memory)
  pick: ['posts', 'total'],
  
  // Transform response
  transform: (response) => ({
    posts: response.posts.map(p => ({
      ...p,
      formattedDate: new Date(p.createdAt).toLocaleDateString('th-TH')
    })),
    total: response.total
  }),
  
  // Watch reactive dependencies
  watch: [page, category],
  
  // ไม่ execute ทันที
  immediate: false,
  
  // Server only (ไม่ re-fetch บน client)
  server: true,
  
  // ไม่ cache ผล
  dedupe: 'cancel',  // 'cancel' | 'defer'
  
  // Timeout
  timeout: 5000,
  
  // Retry
  retry: 3,
  retryDelay: 500,
  
  // On Response callbacks
  onRequest({ request, options }) {
    console.log('Request:', request)
  },
  
  onResponse({ response }) {
    console.log('Response status:', response.status)
  },
  
  onResponseError({ response }) {
    console.error('Response error:', response._data)
  }
})

// Watch page changes แล้วค่อย refresh
watch(page, () => refresh())
</script>
```

### POST Request

```vue
<script setup lang="ts">
interface CreatePostBody {
  title: string
  content: string
  category: string
  tags: string[]
}

const newPost = reactive<CreatePostBody>({
  title: '',
  content: '',
  category: '',
  tags: []
})

// POST ด้วย useFetch
const { data, pending, error, execute } = useFetch('/api/posts', {
  method: 'POST',
  body: newPost,
  immediate: false,  // ไม่ execute ทันที รอ manual
  watch: false
})

async function submitPost() {
  await execute()
  if (!error.value) {
    await navigateTo(`/posts/${data.value?.id}`)
  }
}
</script>

<template>
  <form @submit.prevent="submitPost">
    <input v-model="newPost.title" placeholder="ชื่อบทความ" required />
    <textarea v-model="newPost.content" placeholder="เนื้อหา" required></textarea>
    <button type="submit" :disabled="pending">
      {{ pending ? 'กำลังบันทึก...' : 'บันทึก' }}
    </button>
  </form>
</template>
```

---

## useAsyncData() - Custom async logic

`useAsyncData()` ใช้เมื่อต้องการ custom async logic ที่ซับซ้อนกว่า simple HTTP request

### ความแตกต่างกับ useFetch

```typescript
// useFetch = ทำสิ่งนี้ให้อัตโนมัติ:
useFetch('/api/posts')

// useAsyncData = เขียนเอง:
useAsyncData('posts', () => $fetch('/api/posts'))
```

### ตัวอย่าง useAsyncData

```vue
<script setup lang="ts">
const userId = useRoute().params.id as string

// Fetch หลาย API พร้อมกัน
const { data, pending, error } = await useAsyncData(
  `user-profile-${userId}`,  // unique key
  async () => {
    // ทำ async operations ที่ซับซ้อน
    const [user, posts, followers] = await Promise.all([
      $fetch(`/api/users/${userId}`),
      $fetch(`/api/users/${userId}/posts`),
      $fetch(`/api/users/${userId}/followers`)
    ])
    
    return {
      user,
      posts,
      followers,
      stats: {
        totalPosts: posts.length,
        totalFollowers: followers.length
      }
    }
  },
  {
    // Options
    watch: [() => userId],
    
    // Transform result
    transform: (data) => ({
      ...data,
      user: {
        ...data.user,
        fullName: `${data.user.firstName} ${data.user.lastName}`
      }
    }),
    
    // Pick only needed fields
    pick: ['user', 'stats']
  }
)
</script>

<template>
  <div v-if="data">
    <h1>{{ data.user.fullName }}</h1>
    <p>{{ data.stats.totalPosts }} บทความ | {{ data.stats.totalFollowers }} followers</p>
  </div>
</template>
```

### เปรียบเทียบ useFetch vs useAsyncData

```typescript
// useFetch - เหมาะกับ simple API calls
const { data } = await useFetch('/api/users')

// useAsyncData - เหมาะกับ:
// 1. หลาย API calls
const { data } = await useAsyncData('dashboard', async () => {
  const [users, posts, stats] = await Promise.all([
    $fetch('/api/users/count'),
    $fetch('/api/posts/count'),
    $fetch('/api/analytics/today')
  ])
  return { users, posts, stats }
})

// 2. Business logic ก่อน return
const { data } = await useAsyncData('posts-processed', async () => {
  const rawPosts = await $fetch('/api/posts')
  const processedPosts = rawPosts
    .filter(p => p.isPublished)
    .sort((a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime())
    .map(p => ({
      ...p,
      readingTime: Math.ceil(p.content.split(' ').length / 200)
    }))
  return processedPosts
})

// 3. Database หรือ service calls บน server
const { data } = await useAsyncData('posts-db', async () => {
  const posts = await db.post.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' }
  })
  return posts
})
```

---

## $fetch - Client-side and Server fetching

`$fetch` คือ HTTP client ที่ใช้ได้ทั้ง component, middleware, server handlers

### ใช้ $fetch ใน Component

```vue
<script setup lang="ts">
// $fetch ใช้โดยตรงใน event handlers
async function deletePost(id: number) {
  try {
    await $fetch(`/api/posts/${id}`, {
      method: 'DELETE'
    })
    // refresh list after delete
    await refreshNuxtData('posts')
  } catch (error) {
    console.error('Delete failed:', error)
  }
}

async function likePost(id: number) {
  const result = await $fetch<{ likes: number }>(`/api/posts/${id}/like`, {
    method: 'POST'
  })
  return result.likes
}

// ใช้ $fetch ใน useAsyncData
const { data } = await useAsyncData('post', () => 
  $fetch(`/api/posts/${route.params.id}`)
)
</script>
```

### $fetch Options

```typescript
const data = await $fetch('/api/endpoint', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer token',
    'X-Custom-Header': 'value'
  },
  body: { key: 'value' },
  query: { page: 1, limit: 10 },
  parseResponse: (txt) => JSON.parse(txt),
  responseType: 'json',  // 'json' | 'text' | 'blob' | 'arrayBuffer'
  
  // Retry
  retry: 3,
  retryDelay: 500,
  
  // Timeout
  timeout: 10000,
  
  // Base URL
  baseURL: 'https://api.example.com',
  
  // Interceptors
  onRequest({ options }) {
    options.headers = {
      ...options.headers,
      timestamp: Date.now()
    }
  },
  
  onResponseError({ response }) {
    if (response.status === 401) {
      navigateTo('/login')
    }
  }
})
```

---

## useLazyFetch() และ useLazyAsyncData()

Lazy versions ไม่ block การ render ของหน้า แต่แสดง loading state แทน

### เปรียบเทียบ

```typescript
// useFetch - BLOCKS rendering จนกว่าจะ fetch เสร็จ (SSR)
const { data } = await useFetch('/api/posts')  // หน้าจะรอ

// useLazyFetch - ไม่ block render แสดง pending state แทน
const { data, pending } = useLazyFetch('/api/posts')  // ไม่ต้อง await
```

### เมื่อไหรควรใช้ Lazy?

```vue
<!-- ใช้ useLazyFetch เมื่อ data ไม่จำเป็นสำหรับ initial render -->
<script setup lang="ts">
// Comments - ไม่จำเป็นต้อง block render รอ comments
const { data: comments, pending } = useLazyFetch('/api/posts/1/comments')

// Related posts - secondary content
const { data: related, pending: relatedPending } = useLazyFetch('/api/posts/related', {
  query: { category: 'vue' }
})
</script>

<template>
  <div>
    <!-- Main content render ทันที -->
    <article>
      <h1>บทความ</h1>
      <p>เนื้อหาหลัก...</p>
    </article>
    
    <!-- Comments section - แสดง loading -->
    <section class="comments">
      <h2>ความคิดเห็น</h2>
      <div v-if="pending" class="skeleton">
        <div v-for="i in 3" :key="i" class="skeleton-item"></div>
      </div>
      <div v-else>
        <div v-for="comment in comments" :key="comment.id">
          {{ comment.text }}
        </div>
      </div>
    </section>
    
    <!-- Related posts - แสดง loading -->
    <section class="related">
      <h2>บทความที่เกี่ยวข้อง</h2>
      <div v-if="relatedPending">กำลังโหลด...</div>
      <div v-else class="related-grid">
        <div v-for="post in related" :key="post.id">{{ post.title }}</div>
      </div>
    </section>
  </div>
</template>
```

---

## Error Handling

### จัดการ Errors แบบต่างๆ

```vue
<script setup lang="ts">
// ดักจับ error จาก useFetch
const { data, error } = await useFetch('/api/posts')

// จัดการ error ใน component
if (error.value) {
  // Log error
  console.error('Fetch error:', error.value)
  
  // Throw error ให้ Nuxt จัดการ (แสดง error.vue)
  if (error.value.statusCode === 404) {
    throw createError({
      statusCode: 404,
      statusMessage: 'Posts not found'
    })
  }
}
</script>

<template>
  <div>
    <!-- แสดง error inline -->
    <div v-if="error" class="error-container">
      <div class="error-icon">⚠️</div>
      <h2>เกิดข้อผิดพลาด</h2>
      <p>{{ getErrorMessage(error) }}</p>
      <button @click="refresh()">ลองใหม่</button>
    </div>
    
    <div v-else>
      <!-- content -->
    </div>
  </div>
</template>

<script setup lang="ts">
function getErrorMessage(error: any): string {
  if (error.statusCode === 404) return 'ไม่พบข้อมูลที่ต้องการ'
  if (error.statusCode === 403) return 'คุณไม่มีสิทธิ์เข้าถึงข้อมูลนี้'
  if (error.statusCode === 500) return 'เกิดข้อผิดพลาดจาก server'
  if (error.message === 'Network Error') return 'ไม่สามารถเชื่อมต่อ internet ได้'
  return error.message || 'เกิดข้อผิดพลาดไม่ทราบสาเหตุ'
}
</script>
```

### Global Error Handler

```typescript
// plugins/error-handler.ts
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.hook('app:error', (error) => {
    console.error('Global error:', error)
  })
  
  nuxtApp.hook('vue:error', (error, instance, info) => {
    console.error('Vue error:', error, info)
  })
})
```

### Error Page

```vue
<!-- error.vue -->
<script setup lang="ts">
interface NuxtError {
  statusCode: number
  statusMessage: string
  message: string
  description?: string
}

const props = defineProps<{ error: NuxtError }>()

const handleError = () => clearError({ redirect: '/' })

const errorMessages: Record<number, string> = {
  404: 'ไม่พบหน้าที่คุณค้นหา',
  403: 'คุณไม่มีสิทธิ์เข้าถึงหน้านี้',
  500: 'เกิดข้อผิดพลาดจาก server',
  503: 'Service ไม่พร้อมใช้งานชั่วคราว'
}
</script>

<template>
  <div class="error-page">
    <div class="error-code">{{ props.error.statusCode }}</div>
    <h1>{{ errorMessages[props.error.statusCode] || props.error.statusMessage }}</h1>
    <p>{{ props.error.message }}</p>
    <div class="error-actions">
      <button @click="handleError">กลับหน้าแรก</button>
      <button @click="$router.back()">กลับหน้าก่อน</button>
    </div>
  </div>
</template>
```

---

## Refresh and Re-fetching

### refresh() และ execute()

```vue
<script setup lang="ts">
const page = ref(1)

const { data, pending, refresh, execute } = await useFetch('/api/posts', {
  query: { page },
  // immediate: false ถ้าไม่ต้องการ execute ทันที
})

// Manual refresh
async function loadMore() {
  page.value++
  await refresh()  // re-fetch with new page value
}

// refreshNuxtData - refresh ข้อมูลทุก key ที่กำหนด
async function refreshAll() {
  await refreshNuxtData()  // refresh ทั้งหมด
  // หรือ refresh เฉพาะ key
  await refreshNuxtData(['posts', 'users'])
}

// clearNuxtData - ล้าง cached data
function clearCache() {
  clearNuxtData('posts')  // ล้าง cache ของ key 'posts'
  clearNuxtData()  // ล้างทั้งหมด
}
</script>

<template>
  <div>
    <div v-for="post in data?.posts" :key="post.id">{{ post.title }}</div>
    
    <div class="actions">
      <button @click="refresh()" :disabled="pending">
        {{ pending ? 'กำลังโหลด...' : 'รีเฟรช' }}
      </button>
      <button @click="loadMore()">โหลดเพิ่ม</button>
    </div>
  </div>
</template>
```

### Auto-refresh ด้วย Watch

```vue
<script setup lang="ts">
const filter = ref('all')
const sortBy = ref('date')

// เมื่อ filter หรือ sortBy เปลี่ยน จะ refresh อัตโนมัติ
const { data } = await useFetch('/api/posts', {
  query: computed(() => ({
    filter: filter.value,
    sort: sortBy.value
  })),
  watch: [filter, sortBy]
})
</script>
```

### Polling (auto-refresh ทุก N วินาที)

```vue
<script setup lang="ts">
const { data: liveData, refresh } = await useFetch('/api/stats/live')

// Poll ทุก 5 วินาที
const pollingInterval = setInterval(refresh, 5000)

// Cleanup เมื่อ component unmount
onUnmounted(() => {
  clearInterval(pollingInterval)
})
</script>
```

---

## Caching กับ key

### Key สำหรับ Cache

```typescript
// Key ที่ unique สำหรับแต่ละ request ช่วย cache ข้อมูล
const { data } = await useFetch('/api/posts', {
  key: 'all-posts'  // static key
})

// Dynamic key ตาม params
const { data } = await useFetch(`/api/posts/${id}`, {
  key: `post-${id}`  // dynamic key based on id
})

// Key ตาม query
const { data } = await useFetch('/api/posts', {
  key: computed(() => `posts-${page.value}-${category.value}`)
})
```

### Payload Hydration

Nuxt จะ serialize ข้อมูลที่ fetch บน server และส่งไปให้ client ผ่าน payload เพื่อไม่ต้อง fetch ซ้ำ:

```typescript
// ข้อมูลจาก SSR จะถูก hydrate เข้า client โดยอัตโนมัติ
// ไม่ต้อง fetch ใหม่บน client ถ้า key เดียวกัน
const { data } = await useFetch('/api/posts', {
  key: 'posts-homepage'  // ถ้า SSR fetch แล้ว client จะใช้ data เดิม
})
```

---

## Server-side vs Client-side Fetching

### กำหนด Server/Client behavior

```typescript
// ทำงานเฉพาะ server (ไม่ re-fetch บน client)
const { data } = await useFetch('/api/posts', {
  server: true,   // default: true
  lazy: false
})

// ทำงานเฉพาะ client
const { data } = useLazyFetch('/api/posts', {
  server: false  // ข้าม server-side, fetch บน client เท่านั้น
})

// ทำงานทั้งคู่ (fetch บน server, re-fetch บน client ด้วย)
const { data } = await useFetch('/api/posts', {
  server: true,
  lazy: false,
  // ค่า default คือ fetch บน server ส่ง hydrate ไป client
})
```

### ตัวอย่าง: เลือก Fetch Method ที่เหมาะสม

```vue
<script setup lang="ts">
// 1. SEO-critical content → useFetch (server-side)
const { data: hero } = await useFetch('/api/hero-content', {
  key: 'hero'
  // server: true (default) - render บน server เพื่อ SEO
})

// 2. Secondary content → useLazyFetch (client-side)
const { data: trending, pending: trendingPending } = useLazyFetch('/api/trending')

// 3. User-specific data → client-only
const { data: notifications } = useLazyFetch('/api/notifications', {
  server: false,  // ไม่ต้องการใน SSR
  // เพราะ user-specific ไม่เหมาะกับ cache
})

// 4. Analytics, tracking → client-only $fetch
onMounted(() => {
  $fetch('/api/analytics/pageview', {
    method: 'POST',
    body: { page: route.path, timestamp: Date.now() }
  })
})
</script>
```

---

## ตัวอย่าง: Blog Posts API และ User Profile

### Blog Posts System

```vue
<!-- pages/blog/index.vue -->
<script setup lang="ts">
interface Post {
  id: number
  title: string
  slug: string
  excerpt: string
  author: { name: string; avatar: string }
  category: string
  tags: string[]
  publishedAt: string
  readingTime: number
  viewCount: number
}

interface PostsApiResponse {
  data: Post[]
  meta: {
    total: number
    page: number
    pageSize: number
    totalPages: number
  }
}

// State
const page = ref(1)
const pageSize = ref(12)
const category = ref<string>('')
const sortBy = ref<'newest' | 'popular' | 'trending'>('newest')
const searchQuery = ref('')

// Debounced search
const debouncedSearch = useDebounce(searchQuery, 300)

// Fetch posts
const {
  data: postsResponse,
  pending,
  error,
  refresh
} = await useFetch<PostsApiResponse>('/api/posts', {
  query: computed(() => ({
    page: page.value,
    limit: pageSize.value,
    category: category.value || undefined,
    sort: sortBy.value,
    q: debouncedSearch.value || undefined
  })),
  watch: [page, category, sortBy, debouncedSearch],
  key: computed(() => `blog-${page.value}-${category.value}-${sortBy.value}`)
})

// Categories สำหรับ filter
const { data: categories } = await useFetch('/api/categories')

// Computed
const posts = computed(() => postsResponse.value?.data || [])
const totalPages = computed(() => postsResponse.value?.meta.totalPages || 1)
const totalPosts = computed(() => postsResponse.value?.meta.total || 0)

// Methods
function changePage(newPage: number) {
  page.value = newPage
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function filterByCategory(cat: string) {
  category.value = cat
  page.value = 1
}

// SEO
useSeoMeta({
  title: computed(() => {
    const parts = ['บทความ']
    if (category.value) parts.push(category.value)
    if (page.value > 1) parts.push(`หน้า ${page.value}`)
    parts.push('Vue Blog')
    return parts.join(' - ')
  })
})
</script>

<template>
  <div class="blog-index">
    <!-- Search and Filters -->
    <div class="blog-filters">
      <div class="search-bar">
        <input
          v-model="searchQuery"
          type="search"
          placeholder="ค้นหาบทความ..."
          class="search-input"
        />
      </div>
      
      <div class="category-filters">
        <button
          :class="{ active: !category }"
          @click="filterByCategory('')"
        >ทั้งหมด</button>
        <button
          v-for="cat in categories"
          :key="cat.slug"
          :class="{ active: category === cat.slug }"
          @click="filterByCategory(cat.slug)"
        >{{ cat.name }}</button>
      </div>
      
      <div class="sort-options">
        <select v-model="sortBy">
          <option value="newest">ล่าสุด</option>
          <option value="popular">ยอดนิยม</option>
          <option value="trending">กำลังฮิต</option>
        </select>
      </div>
    </div>
    
    <!-- Results count -->
    <p class="results-info">
      <span v-if="searchQuery">
        พบ {{ totalPosts }} ผลลัพธ์ สำหรับ "{{ searchQuery }}"
      </span>
      <span v-else>
        บทความทั้งหมด {{ totalPosts }} บทความ
      </span>
    </p>
    
    <!-- Loading State -->
    <div v-if="pending" class="posts-skeleton">
      <div v-for="i in pageSize" :key="i" class="skeleton-card">
        <div class="skeleton-image"></div>
        <div class="skeleton-content">
          <div class="skeleton-line short"></div>
          <div class="skeleton-line"></div>
          <div class="skeleton-line medium"></div>
        </div>
      </div>
    </div>
    
    <!-- Error State -->
    <div v-else-if="error" class="error-state">
      <p>เกิดข้อผิดพลาดในการโหลดบทความ</p>
      <button @click="refresh()">ลองใหม่</button>
    </div>
    
    <!-- Empty State -->
    <div v-else-if="posts.length === 0" class="empty-state">
      <p>ไม่พบบทความที่ต้องการ</p>
      <button @click="filterByCategory('')">ดูบทความทั้งหมด</button>
    </div>
    
    <!-- Posts Grid -->
    <div v-else class="posts-grid">
      <article
        v-for="post in posts"
        :key="post.id"
        class="post-card"
      >
        <NuxtLink :to="`/blog/${post.slug}`">
          <div class="post-category">{{ post.category }}</div>
          <h2 class="post-title">{{ post.title }}</h2>
          <p class="post-excerpt">{{ post.excerpt }}</p>
          <div class="post-footer">
            <span class="post-author">{{ post.author.name }}</span>
            <span class="post-date">
              {{ new Date(post.publishedAt).toLocaleDateString('th-TH') }}
            </span>
            <span class="post-read">{{ post.readingTime }} นาที</span>
          </div>
        </NuxtLink>
      </article>
    </div>
    
    <!-- Pagination -->
    <nav v-if="totalPages > 1" class="pagination">
      <button
        :disabled="page <= 1"
        @click="changePage(page - 1)"
      >← ก่อนหน้า</button>
      
      <template v-for="p in totalPages" :key="p">
        <button
          v-if="Math.abs(p - page) < 3 || p === 1 || p === totalPages"
          :class="{ active: p === page }"
          @click="changePage(p)"
        >{{ p }}</button>
        <span v-else-if="Math.abs(p - page) === 3">...</span>
      </template>
      
      <button
        :disabled="page >= totalPages"
        @click="changePage(page + 1)"
      >ถัดไป →</button>
    </nav>
  </div>
</template>
```

### User Profile Page

```vue
<!-- pages/users/[id].vue -->
<script setup lang="ts">
const route = useRoute()
const userId = route.params.id as string

// Fetch user profile
const { data: user, error: userError } = await useFetch(`/api/users/${userId}`)

if (userError.value?.statusCode === 404) {
  throw createError({ statusCode: 404, statusMessage: 'User not found' })
}

// Fetch user's posts (lazy - secondary content)
const { data: userPosts, pending: postsPending } = useLazyFetch(`/api/users/${userId}/posts`, {
  query: { limit: 10, page: 1 }
})

// Fetch user stats
const { data: userStats } = await useAsyncData(
  `user-stats-${userId}`,
  async () => {
    const [posts, followers, following] = await Promise.all([
      $fetch<{ count: number }>(`/api/users/${userId}/posts/count`),
      $fetch<{ count: number }>(`/api/users/${userId}/followers/count`),
      $fetch<{ count: number }>(`/api/users/${userId}/following/count`)
    ])
    return { posts: posts.count, followers: followers.count, following: following.count }
  }
)

useSeoMeta({
  title: () => `${user.value?.name} - Vue Blog`,
  description: () => user.value?.bio || `โปรไฟล์ของ ${user.value?.name}`
})
</script>

<template>
  <div class="user-profile" v-if="user">
    <!-- Profile Header -->
    <div class="profile-header">
      <img :src="user.avatar" :alt="user.name" class="avatar" />
      <div class="profile-info">
        <h1>{{ user.name }}</h1>
        <p class="username">@{{ user.username }}</p>
        <p class="bio">{{ user.bio }}</p>
        <div class="stats">
          <div class="stat">
            <span class="stat-value">{{ userStats?.posts }}</span>
            <span class="stat-label">บทความ</span>
          </div>
          <div class="stat">
            <span class="stat-value">{{ userStats?.followers }}</span>
            <span class="stat-label">ผู้ติดตาม</span>
          </div>
          <div class="stat">
            <span class="stat-value">{{ userStats?.following }}</span>
            <span class="stat-label">กำลังติดตาม</span>
          </div>
        </div>
      </div>
    </div>
    
    <!-- User Posts -->
    <div class="user-posts">
      <h2>บทความของ {{ user.name }}</h2>
      
      <div v-if="postsPending" class="loading">กำลังโหลด...</div>
      
      <div v-else-if="userPosts?.data?.length === 0" class="empty">
        ยังไม่มีบทความ
      </div>
      
      <div v-else class="posts-list">
        <article
          v-for="post in userPosts?.data"
          :key="post.id"
          class="post-item"
        >
          <NuxtLink :to="`/blog/${post.slug}`">{{ post.title }}</NuxtLink>
          <time>{{ new Date(post.publishedAt).toLocaleDateString('th-TH') }}</time>
        </article>
      </div>
    </div>
  </div>
</template>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **useFetch()** - Universal data fetching สำหรับ SSR + Client
2. **useAsyncData()** - Custom async logic พร้อม caching
3. **$fetch** - HTTP client สำหรับ event handlers
4. **useLazyFetch()** - Non-blocking fetch สำหรับ secondary content
5. **Error Handling** - จัดการ errors อย่างถูกวิธี
6. **Refresh** - refresh(), execute(), refreshNuxtData()
7. **Caching** - ใช้ key เพื่อ cache และ hydrate data
8. **Server vs Client** - เลือก mode ที่เหมาะสมกับ use case

**ถัดไป**: Part 25 - Nuxt Middleware
