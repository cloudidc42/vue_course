# Part 22: Nuxt File-based Routing

## File-based Routing คืออะไร?

ใน Nuxt.js การสร้าง Route ทำได้โดยการสร้างไฟล์ใน `pages/` directory โดย Nuxt จะสร้าง Route อัตโนมัติตามชื่อและโครงสร้างของไฟล์ ไม่ต้องกำหนด router config เอง

### ตัวอย่างโครงสร้าง pages/

```
pages/
├── index.vue          → /
├── about.vue          → /about
├── contact.vue        → /contact
├── posts/
│   ├── index.vue      → /posts
│   └── [id].vue       → /posts/:id
├── users/
│   ├── index.vue      → /users
│   ├── [id]/
│   │   ├── index.vue  → /users/:id
│   │   └── edit.vue   → /users/:id/edit
│   └── create.vue     → /users/create
└── [...slug].vue      → /* (catch-all)
```

### ไฟล์ที่ Nuxt สร้าง Route ให้อัตโนมัติ

| ไฟล์ | Route | คำอธิบาย |
|------|-------|----------|
| `pages/index.vue` | `/` | หน้าแรก |
| `pages/about.vue` | `/about` | หน้าเกี่ยวกับ |
| `pages/posts/index.vue` | `/posts` | รายการ posts |
| `pages/posts/[id].vue` | `/posts/:id` | post แต่ละชิ้น |
| `pages/[...slug].vue` | `/*` | catch-all route |
| `pages/[[id]].vue` | `/:id?` | optional parameter |

---

## Dynamic Routes ([id].vue)

Dynamic Route คือ route ที่มีส่วนที่เปลี่ยนแปลงได้ โดยใช้วงเล็บเหลี่ยม `[paramName]`

### สร้าง Dynamic Route

```vue
<!-- pages/posts/[id].vue -->
<script setup lang="ts">
// useRoute auto-imported
const route = useRoute()

// ดึงค่า id จาก route params
const postId = route.params.id  // string หรือ string[]

// หรือใช้ defineProps (Nuxt 3)
// const { id } = defineProps<{ id: string }>()

// Fetch data based on id
const { data: post, pending, error } = await useFetch(`/api/posts/${postId}`)

// กำหนด SEO meta
useSeoMeta({
  title: () => post.value?.title || 'Loading...',
  description: () => post.value?.excerpt || ''
})
</script>

<template>
  <div class="post-page">
    <div v-if="pending" class="loading">
      <div class="spinner"></div>
      <p>กำลังโหลดบทความ...</p>
    </div>
    
    <div v-else-if="error" class="error">
      <h2>เกิดข้อผิดพลาด</h2>
      <p>{{ error.message }}</p>
      <NuxtLink to="/posts">กลับไปรายการบทความ</NuxtLink>
    </div>
    
    <article v-else-if="post" class="post">
      <header>
        <h1>{{ post.title }}</h1>
        <div class="post-meta">
          <span>โดย {{ post.author }}</span>
          <span>{{ new Date(post.createdAt).toLocaleDateString('th-TH') }}</span>
        </div>
      </header>
      <div class="post-content" v-html="post.content"></div>
    </article>
  </div>
</template>
```

### หลาย Dynamic Segments

```
pages/
└── users/
    └── [userId]/
        └── posts/
            └── [postId].vue   → /users/:userId/posts/:postId
```

```vue
<!-- pages/users/[userId]/posts/[postId].vue -->
<script setup lang="ts">
const route = useRoute()

const userId = route.params.userId    // จาก [userId]
const postId = route.params.postId    // จาก [postId]

const { data } = await useFetch(`/api/users/${userId}/posts/${postId}`)
</script>

<template>
  <div>
    <p>User: {{ $route.params.userId }}</p>
    <p>Post: {{ $route.params.postId }}</p>
  </div>
</template>
```

---

## Nested Routes

Nested Routes ช่วยให้มี Layout ที่ share กันใน route กลุ่มเดียวกัน

### โครงสร้างไฟล์

```
pages/
└── users/
    ├── index.vue          → /users
    ├── [id].vue           → /users/:id (parent)
    └── [id]/
        ├── profile.vue    → /users/:id/profile (nested)
        ├── settings.vue   → /users/:id/settings (nested)
        └── posts.vue      → /users/:id/posts (nested)
```

### Parent Route Component

```vue
<!-- pages/users/[id].vue - Parent -->
<script setup lang="ts">
const route = useRoute()
const { data: user } = await useFetch(`/api/users/${route.params.id}`)
</script>

<template>
  <div class="user-layout">
    <!-- ข้อมูล user ที่ share ใน child routes -->
    <div class="user-sidebar">
      <img :src="user?.avatar" :alt="user?.name" />
      <h2>{{ user?.name }}</h2>
      <p>{{ user?.email }}</p>
      
      <nav class="user-nav">
        <NuxtLink :to="`/users/${route.params.id}/profile`">โปรไฟล์</NuxtLink>
        <NuxtLink :to="`/users/${route.params.id}/posts`">บทความ</NuxtLink>
        <NuxtLink :to="`/users/${route.params.id}/settings`">ตั้งค่า</NuxtLink>
      </nav>
    </div>
    
    <!-- Child route content จะ render ที่นี่ -->
    <div class="user-content">
      <NuxtPage />
    </div>
  </div>
</template>

<style scoped>
.user-layout {
  display: grid;
  grid-template-columns: 250px 1fr;
  gap: 2rem;
}
</style>
```

### Child Route Components

```vue
<!-- pages/users/[id]/profile.vue -->
<script setup lang="ts">
const route = useRoute()
const { data: profile } = await useFetch(`/api/users/${route.params.id}/profile`)
</script>

<template>
  <div>
    <h3>โปรไฟล์</h3>
    <p>ชื่อ: {{ profile?.name }}</p>
    <p>อีเมล: {{ profile?.email }}</p>
    <p>สมาชิกตั้งแต่: {{ profile?.joinedAt }}</p>
  </div>
</template>
```

```vue
<!-- pages/users/[id]/posts.vue -->
<script setup lang="ts">
const route = useRoute()
const { data: posts } = await useFetch(`/api/users/${route.params.id}/posts`)
</script>

<template>
  <div>
    <h3>บทความของฉัน ({{ posts?.length }} บทความ)</h3>
    <div v-for="post in posts" :key="post.id" class="post-item">
      <NuxtLink :to="`/posts/${post.id}`">{{ post.title }}</NuxtLink>
      <span>{{ post.createdAt }}</span>
    </div>
  </div>
</template>
```

---

## Catch-all Routes ([...slug].vue)

Catch-all Route จะจับ URL ทุกอย่างที่ไม่ match กับ route อื่น

### สร้าง Catch-all Route

```vue
<!-- pages/[...slug].vue -->
<script setup lang="ts">
const route = useRoute()

// slug จะเป็น array ของ path segments
// URL: /a/b/c → slug = ['a', 'b', 'c']
const slugArray = route.params.slug as string[]
const fullPath = slugArray.join('/')

console.log('Path segments:', slugArray)
// URL: /blog/2024/my-post → ['blog', '2024', 'my-post']
</script>

<template>
  <div>
    <h1>Catch-all Route</h1>
    <p>Path: /{{ $route.params.slug.join('/') }}</p>
    
    <!-- ใช้สร้าง dynamic breadcrumbs -->
    <nav class="breadcrumbs">
      <NuxtLink to="/">หน้าแรก</NuxtLink>
      <template v-for="(segment, index) in slugArray" :key="index">
        <span>/</span>
        <NuxtLink :to="'/' + slugArray.slice(0, index + 1).join('/')">
          {{ segment }}
        </NuxtLink>
      </template>
    </nav>
  </div>
</template>
```

### ใช้กับ Content Management

```vue
<!-- pages/[...slug].vue - สำหรับ CMS pages -->
<script setup lang="ts">
const route = useRoute()
const slug = route.params.slug as string[]

// ดึง page content ตาม path
const { data: page, error } = await useFetch(`/api/pages/${slug.join('/')}`)

if (error.value?.statusCode === 404) {
  throw createError({
    statusCode: 404,
    statusMessage: 'Page Not Found'
  })
}

useSeoMeta({
  title: () => page.value?.title,
  description: () => page.value?.description
})
</script>

<template>
  <div v-if="page">
    <h1>{{ page.title }}</h1>
    <div v-html="page.content"></div>
  </div>
</template>
```

---

## Optional Parameters ([[id]].vue)

Double bracket หมายถึง parameter ที่ไม่จำเป็นต้องมี (optional)

```
pages/
├── [[id]].vue         → / หรือ /:id
└── posts/
    └── [[page]].vue   → /posts หรือ /posts/:page
```

### ตัวอย่าง Optional Parameter

```vue
<!-- pages/posts/[[page]].vue -->
<script setup lang="ts">
const route = useRoute()

// page อาจเป็น undefined ถ้าไม่มีใน URL
const currentPage = computed(() => {
  const page = route.params.page
  return page ? parseInt(page as string) : 1
})

const pageSize = 10

const { data } = await useFetch('/api/posts', {
  query: {
    page: currentPage,
    limit: pageSize
  }
})
</script>

<template>
  <div>
    <h1>บทความ</h1>
    <p>หน้า {{ currentPage }}</p>
    
    <div v-for="post in data?.posts" :key="post.id">
      <h2>{{ post.title }}</h2>
    </div>
    
    <!-- Pagination -->
    <div class="pagination">
      <NuxtLink
        v-if="currentPage > 1"
        :to="currentPage === 2 ? '/posts' : `/posts/${currentPage - 1}`"
      >
        ← ก่อนหน้า
      </NuxtLink>
      
      <span>{{ currentPage }} / {{ data?.totalPages }}</span>
      
      <NuxtLink
        v-if="currentPage < data?.totalPages"
        :to="`/posts/${currentPage + 1}`"
      >
        ถัดไป →
      </NuxtLink>
    </div>
  </div>
</template>
```

---

## Named Routes

ใน Nuxt สามารถระบุ route name ได้ใน `definePageMeta()`

### กำหนด Route Name

```vue
<!-- pages/posts/[id].vue -->
<script setup lang="ts">
definePageMeta({
  name: 'post-detail'  // กำหนดชื่อ route
})
</script>
```

### Navigate ด้วย Name

```vue
<script setup lang="ts">
const router = useRouter()

// Navigate ด้วย route name
function goToPost(id: number) {
  router.push({
    name: 'post-detail',
    params: { id }
  })
}

// หรือใช้ navigateTo
async function viewPost(id: number) {
  await navigateTo({
    name: 'post-detail',
    params: { id }
  })
}
</script>

<template>
  <!-- NuxtLink กับ route name -->
  <NuxtLink :to="{ name: 'post-detail', params: { id: 1 } }">
    ดูบทความ
  </NuxtLink>
</template>
```

---

## useRoute() และ useRouter()

### useRoute()

```vue
<script setup lang="ts">
const route = useRoute()

// Properties ของ route
console.log(route.path)          // '/posts/123'
console.log(route.fullPath)      // '/posts/123?page=1#section1'
console.log(route.params)        // { id: '123' }
console.log(route.query)         // { page: '1', filter: 'vue' }
console.log(route.hash)          // '#section1'
console.log(route.name)          // 'post-detail'
console.log(route.meta)          // { layout: 'blog', requiresAuth: true }
console.log(route.matched)       // array ของ matched routes

// Watch route changes
watch(() => route.params.id, (newId, oldId) => {
  console.log(`ID เปลี่ยนจาก ${oldId} เป็น ${newId}`)
  // reload data...
})
</script>
```

### useRouter()

```vue
<script setup lang="ts">
const router = useRouter()

// Navigation methods
router.push('/about')
router.push({ path: '/about' })
router.push({ name: 'about' })
router.push({ path: '/posts', query: { page: 2 } })

// Replace (ไม่เพิ่ม history entry)
router.replace('/login')

// Go back/forward
router.back()
router.forward()
router.go(-2)  // ย้อน 2 ขั้น

// Guards
router.beforeEach((to, from) => {
  console.log('Navigating to:', to.path)
  // return false เพื่อ cancel navigation
})
</script>
```

---

## Route Params, Query, Hash

### การทำงานกับ Query Parameters

```vue
<!-- pages/search.vue -->
<script setup lang="ts">
const route = useRoute()
const router = useRouter()

// อ่าน query parameters
const searchQuery = computed(() => route.query.q as string || '')
const currentPage = computed(() => parseInt(route.query.page as string || '1'))
const sortBy = computed(() => route.query.sort as string || 'date')

// อัพเดต query parameters
function updateSearch(query: string) {
  router.push({
    query: {
      ...route.query,  // เก็บ query เดิมไว้
      q: query,
      page: '1'  // reset กลับหน้า 1 เมื่อ search ใหม่
    }
  })
}

function changePage(page: number) {
  router.push({
    query: { ...route.query, page: String(page) }
  })
}

// Fetch ด้วย query params
const { data, refresh } = await useFetch('/api/posts', {
  query: computed(() => ({
    q: searchQuery.value,
    page: currentPage.value,
    sort: sortBy.value
  }))
})

// Watch query changes แล้ว refresh
watch(() => route.query, () => {
  refresh()
})
</script>

<template>
  <div class="search-page">
    <h1>ค้นหา</h1>
    
    <input
      :value="searchQuery"
      @input="updateSearch(($event.target as HTMLInputElement).value)"
      placeholder="ค้นหาบทความ..."
    />
    
    <div class="filters">
      <button
        v-for="sort in ['date', 'views', 'likes']"
        :key="sort"
        :class="{ active: sortBy === sort }"
        @click="router.push({ query: { ...route.query, sort } })"
      >
        {{ sort }}
      </button>
    </div>
    
    <p>พบ {{ data?.total }} ผลลัพธ์ สำหรับ "{{ searchQuery }}"</p>
    
    <div v-for="post in data?.posts" :key="post.id">
      <NuxtLink :to="`/posts/${post.id}`">{{ post.title }}</NuxtLink>
    </div>
  </div>
</template>
```

### การทำงานกับ Hash

```vue
<script setup lang="ts">
const route = useRoute()
const router = useRouter()

// อ่าน hash
const currentSection = computed(() => route.hash.replace('#', ''))

// Navigate ไป hash
function scrollToSection(sectionId: string) {
  router.push({ hash: `#${sectionId}` })
}
</script>

<template>
  <div>
    <!-- Table of Contents -->
    <nav class="toc">
      <button @click="scrollToSection('intro')">บทนำ</button>
      <button @click="scrollToSection('details')">รายละเอียด</button>
      <button @click="scrollToSection('conclusion')">สรุป</button>
    </nav>
    
    <section id="intro">
      <h2>บทนำ</h2>
      <p>...</p>
    </section>
    
    <section id="details">
      <h2>รายละเอียด</h2>
      <p>...</p>
    </section>
    
    <section id="conclusion">
      <h2>สรุป</h2>
      <p>...</p>
    </section>
  </div>
</template>
```

---

## Navigation (navigateTo, useRouter().push)

### navigateTo()

```vue
<script setup lang="ts">
// navigateTo - Nuxt 3 navigation helper
async function handleLogin() {
  const success = await performLogin()
  
  if (success) {
    // Navigate หลัง login สำเร็จ
    await navigateTo('/dashboard')
    
    // หรือกับ params
    await navigateTo({ name: 'dashboard' })
    
    // External URL
    await navigateTo('https://example.com', { external: true })
    
    // Replace history (ไม่ let กด back กลับมา)
    await navigateTo('/dashboard', { replace: true })
    
    // Redirect code สำหรับ server-side
    await navigateTo('/dashboard', { redirectCode: 301 })
  } else {
    await navigateTo('/login?error=invalid-credentials')
  }
}

// ใน middleware
export default defineNuxtRouteMiddleware((to, from) => {
  const isAuthenticated = false
  
  if (!isAuthenticated) {
    return navigateTo('/login')
  }
})
</script>
```

### NuxtLink Component

```vue
<template>
  <div>
    <!-- Basic link -->
    <NuxtLink to="/about">เกี่ยวกับ</NuxtLink>
    
    <!-- ด้วย object -->
    <NuxtLink :to="{ name: 'post-detail', params: { id: 1 } }">
      ดูบทความ
    </NuxtLink>
    
    <!-- External link -->
    <NuxtLink to="https://nuxt.com" external target="_blank">
      Nuxt Docs
    </NuxtLink>
    
    <!-- Active class -->
    <NuxtLink
      to="/posts"
      active-class="router-link-active"
      exact-active-class="router-link-exact-active"
    >
      บทความ
    </NuxtLink>
    
    <!-- Prefetch (default: true) -->
    <NuxtLink to="/about" :prefetch="false">
      เกี่ยวกับ (no prefetch)
    </NuxtLink>
  </div>
</template>
```

---

## definePageMeta()

```vue
<!-- pages/admin/dashboard.vue -->
<script setup lang="ts">
definePageMeta({
  // กำหนด layout
  layout: 'admin',
  
  // กำหนด middleware
  middleware: ['auth', 'admin'],
  
  // Route name
  name: 'admin-dashboard',
  
  // Custom meta
  requiresAuth: true,
  roles: ['admin', 'superadmin'],
  
  // Page transition
  pageTransition: {
    name: 'slide-right',
    mode: 'out-in'
  },
  
  // SEO
  title: 'Admin Dashboard',
  
  // Keep-alive
  keepalive: true,
  
  // Validate params
  validate: async (route) => {
    return /^\d+$/.test(route.params.id as string)
  }
})
</script>
```

---

## ตัวอย่าง: Blog App Routing Structure

### โครงสร้างสมบูรณ์

```
pages/
├── index.vue                    → / (หน้าแรก)
├── blog/
│   ├── index.vue                → /blog (รายการบทความ)
│   ├── [[page]].vue             → /blog หรือ /blog/2 (pagination)
│   └── [slug].vue               → /blog/:slug (บทความแต่ละชิ้น)
├── categories/
│   ├── index.vue                → /categories
│   └── [category]/
│       ├── index.vue            → /categories/:category
│       └── [[page]].vue         → /categories/:category/:page?
├── tags/
│   └── [tag].vue                → /tags/:tag
├── author/
│   └── [username].vue           → /author/:username
├── search.vue                   → /search
└── [...slug].vue                → /* (404 catch-all)
```

### หน้าแรก Blog

```vue
<!-- pages/index.vue -->
<script setup lang="ts">
useSeoMeta({
  title: 'Vue Blog - บทความเกี่ยวกับ Vue.js',
  description: 'แหล่งเรียนรู้ Vue.js และ Nuxt.js ภาษาไทย'
})

// ดึงบทความล่าสุด
const { data: latestPosts } = await useFetch('/api/posts', {
  query: { limit: 6, sort: 'createdAt', order: 'desc' }
})

// ดึง featured posts
const { data: featuredPosts } = await useFetch('/api/posts', {
  query: { featured: true, limit: 3 }
})

// ดึง categories
const { data: categories } = await useFetch('/api/categories')
</script>

<template>
  <div class="home">
    <!-- Hero Section -->
    <section class="hero">
      <h1>Vue Blog</h1>
      <p>เรียนรู้ Vue.js และ Nuxt.js ภาษาไทย</p>
      <NuxtLink to="/blog" class="cta-button">อ่านบทความทั้งหมด</NuxtLink>
    </section>
    
    <!-- Featured Posts -->
    <section class="featured">
      <h2>บทความแนะนำ</h2>
      <div class="posts-grid">
        <article v-for="post in featuredPosts" :key="post.id" class="post-card featured">
          <img :src="post.coverImage" :alt="post.title" />
          <div class="post-info">
            <span class="badge">แนะนำ</span>
            <h3>
              <NuxtLink :to="`/blog/${post.slug}`">{{ post.title }}</NuxtLink>
            </h3>
            <p>{{ post.excerpt }}</p>
            <div class="post-meta">
              <NuxtLink :to="`/author/${post.author.username}`">
                {{ post.author.name }}
              </NuxtLink>
              <time :datetime="post.createdAt">
                {{ new Date(post.createdAt).toLocaleDateString('th-TH') }}
              </time>
            </div>
          </div>
        </article>
      </div>
    </section>
    
    <!-- Latest Posts -->
    <section class="latest">
      <h2>บทความล่าสุด</h2>
      <div class="posts-list">
        <article v-for="post in latestPosts" :key="post.id" class="post-item">
          <div class="post-thumbnail">
            <NuxtLink :to="`/blog/${post.slug}`">
              <img :src="post.thumbnail" :alt="post.title" />
            </NuxtLink>
          </div>
          <div class="post-details">
            <div class="post-tags">
              <NuxtLink
                v-for="tag in post.tags"
                :key="tag"
                :to="`/tags/${tag}`"
                class="tag"
              >
                #{{ tag }}
              </NuxtLink>
            </div>
            <h3>
              <NuxtLink :to="`/blog/${post.slug}`">{{ post.title }}</NuxtLink>
            </h3>
            <p>{{ post.excerpt }}</p>
            <div class="post-meta">
              <NuxtLink :to="`/categories/${post.category}`" class="category">
                {{ post.category }}
              </NuxtLink>
              <span>{{ post.readingTime }} นาที</span>
            </div>
          </div>
        </article>
      </div>
    </section>
    
    <!-- Categories -->
    <section class="categories-section">
      <h2>หมวดหมู่</h2>
      <div class="categories-grid">
        <NuxtLink
          v-for="cat in categories"
          :key="cat.slug"
          :to="`/categories/${cat.slug}`"
          class="category-card"
        >
          <span class="category-icon">{{ cat.icon }}</span>
          <span class="category-name">{{ cat.name }}</span>
          <span class="category-count">{{ cat.postCount }} บทความ</span>
        </NuxtLink>
      </div>
    </section>
  </div>
</template>
```

### Blog Post Detail

```vue
<!-- pages/blog/[slug].vue -->
<script setup lang="ts">
const route = useRoute()
const slug = route.params.slug as string

// Fetch post
const { data: post, error } = await useFetch(`/api/posts/${slug}`)

// Handle 404
if (!post.value) {
  throw createError({
    statusCode: 404,
    statusMessage: 'Post not found'
  })
}

// SEO
useSeoMeta({
  title: () => `${post.value?.title} | Vue Blog`,
  description: () => post.value?.excerpt,
  ogTitle: () => post.value?.title,
  ogDescription: () => post.value?.excerpt,
  ogImage: () => post.value?.coverImage,
  articlePublishedTime: () => post.value?.createdAt,
  articleAuthor: () => [post.value?.author.name]
})

// Related posts
const { data: relatedPosts } = await useFetch('/api/posts', {
  query: {
    category: post.value?.category,
    exclude: post.value?.id,
    limit: 3
  }
})

// Table of contents
const headings = computed(() => {
  if (!post.value?.content) return []
  const matches = post.value.content.matchAll(/<h([2-3])[^>]*id="([^"]+)"[^>]*>(.*?)<\/h[2-3]>/g)
  return Array.from(matches).map(([, level, id, text]) => ({
    level: parseInt(level),
    id,
    text: text.replace(/<[^>]+>/g, '')
  }))
})
</script>

<template>
  <article class="blog-post" v-if="post">
    <!-- Hero -->
    <div class="post-hero">
      <img :src="post.coverImage" :alt="post.title" class="cover-image" />
      <div class="post-hero-content">
        <div class="breadcrumbs">
          <NuxtLink to="/">หน้าแรก</NuxtLink>
          <span>/</span>
          <NuxtLink to="/blog">บทความ</NuxtLink>
          <span>/</span>
          <span>{{ post.title }}</span>
        </div>
        <h1>{{ post.title }}</h1>
        <div class="post-meta">
          <NuxtLink :to="`/author/${post.author.username}`" class="author">
            <img :src="post.author.avatar" :alt="post.author.name" />
            {{ post.author.name }}
          </NuxtLink>
          <time>{{ new Date(post.createdAt).toLocaleDateString('th-TH') }}</time>
          <span>{{ post.readingTime }} นาทีอ่าน</span>
          <span>{{ post.views }} ครั้งที่ดู</span>
        </div>
        <div class="post-tags">
          <NuxtLink v-for="tag in post.tags" :key="tag" :to="`/tags/${tag}`">
            #{{ tag }}
          </NuxtLink>
        </div>
      </div>
    </div>
    
    <!-- Content Layout -->
    <div class="post-layout">
      <!-- Table of Contents -->
      <aside class="toc" v-if="headings.length > 0">
        <h4>สารบัญ</h4>
        <nav>
          <a
            v-for="heading in headings"
            :key="heading.id"
            :href="`#${heading.id}`"
            :class="`toc-h${heading.level}`"
          >
            {{ heading.text }}
          </a>
        </nav>
      </aside>
      
      <!-- Post Content -->
      <div class="post-content" v-html="post.content"></div>
    </div>
    
    <!-- Related Posts -->
    <section class="related-posts" v-if="relatedPosts?.length">
      <h2>บทความที่เกี่ยวข้อง</h2>
      <div class="related-grid">
        <article v-for="related in relatedPosts" :key="related.id">
          <NuxtLink :to="`/blog/${related.slug}`">
            <img :src="related.thumbnail" :alt="related.title" />
            <h3>{{ related.title }}</h3>
          </NuxtLink>
        </article>
      </div>
    </section>
  </article>
</template>
```

### Category Page

```vue
<!-- pages/categories/[category]/[[page]].vue -->
<script setup lang="ts">
const route = useRoute()
const category = route.params.category as string
const currentPage = computed(() => {
  const p = route.params.page
  return p ? parseInt(p as string) : 1
})

const { data: categoryInfo } = await useFetch(`/api/categories/${category}`)
const { data: posts } = await useFetch('/api/posts', {
  query: computed(() => ({
    category,
    page: currentPage.value,
    limit: 10
  }))
})

useSeoMeta({
  title: () => `${categoryInfo.value?.name} - Vue Blog`,
  description: () => categoryInfo.value?.description
})
</script>

<template>
  <div>
    <header class="category-header">
      <h1>{{ categoryInfo?.name }}</h1>
      <p>{{ categoryInfo?.description }}</p>
      <span>{{ posts?.total }} บทความ</span>
    </header>
    
    <div class="posts-list">
      <article v-for="post in posts?.items" :key="post.id">
        <NuxtLink :to="`/blog/${post.slug}`">
          <h2>{{ post.title }}</h2>
        </NuxtLink>
        <p>{{ post.excerpt }}</p>
      </article>
    </div>
    
    <!-- Pagination -->
    <nav class="pagination">
      <NuxtLink
        v-if="currentPage > 1"
        :to="currentPage === 2 
          ? `/categories/${category}` 
          : `/categories/${category}/${currentPage - 1}`"
      >← ก่อนหน้า</NuxtLink>
      
      <span>หน้า {{ currentPage }} จาก {{ posts?.totalPages }}</span>
      
      <NuxtLink
        v-if="currentPage < posts?.totalPages"
        :to="`/categories/${category}/${currentPage + 1}`"
      >ถัดไป →</NuxtLink>
    </nav>
  </div>
</template>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **File-based Routing** - สร้าง route โดยสร้างไฟล์ใน pages/
2. **Dynamic Routes** - ใช้ `[param]` สำหรับ dynamic segments
3. **Nested Routes** - ใช้ folder structure สร้าง nested routes
4. **Catch-all Routes** - ใช้ `[...slug]` จับทุก URL
5. **Optional Parameters** - ใช้ `[[param]]` สำหรับ optional
6. **Named Routes** - กำหนดชื่อ route ด้วย `definePageMeta()`
7. **useRoute() & useRouter()** - Composables สำหรับจัดการ routes
8. **Navigation** - ใช้ `navigateTo()`, `useRouter().push()`, `NuxtLink`

**ถัดไป**: Part 23 - Nuxt Layouts
