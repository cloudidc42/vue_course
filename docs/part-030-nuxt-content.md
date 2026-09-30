# Part 30: Nuxt Content

## @nuxt/content คืออะไร?

`@nuxt/content` คือ module สำหรับสร้าง content-driven website โดยสามารถใช้ Markdown, JSON, YAML, CSV เป็น data source โดยไม่ต้องมี CMS backend

### ความสามารถหลัก

- **Markdown** - เขียนบทความด้วย Markdown
- **MDC** (Markdown Components) - ใช้ Vue components ใน Markdown
- **queryContent()** - Query content คล้าย database
- **Full-text search** - ค้นหาเนื้อหา
- **TypeScript support** - Type-safe content
- **Code highlighting** - Syntax highlighting อัตโนมัติ

---

## ติดตั้งและตั้งค่า

### การติดตั้ง

```bash
# ติดตั้ง module
npx nuxi module add @nuxt/content

# หรือ install manual
npm install @nuxt/content
```

### nuxt.config.ts

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/content'],
  
  content: {
    // ตำแหน่ง content (default: content/)
    sources: {
      // local content
      content: {
        driver: 'fs',
        prefix: '/docs',
        base: './content'
      }
    },
    
    // Markdown settings
    markdown: {
      anchorLinks: true,
      remarkPlugins: [],
      rehypePlugins: [],
      
      // Code highlighting (ใช้ Shiki)
      highlight: {
        theme: {
          default: 'github-light',
          dark: 'github-dark'
        },
        langs: ['js', 'ts', 'vue', 'json', 'bash', 'css', 'html']
      }
    },
    
    // Experimental features
    experimental: {
      search: true  // เปิด full-text search
    }
  }
})
```

---

## โครงสร้างโฟลเดอร์ content/

```
content/
├── index.md                → /
├── about.md                → /about
├── blog/
│   ├── index.md            → /blog
│   ├── hello-world.md      → /blog/hello-world
│   └── learn-nuxt.md       → /blog/learn-nuxt
├── docs/
│   ├── index.md            → /docs
│   ├── getting-started.md  → /docs/getting-started
│   └── api/
│       └── index.md        → /docs/api
└── _partials/              # _ prefix = ไม่สร้าง route
    └── header.md
```

---

## สร้าง Markdown Content

### Front Matter

```markdown
---
title: ยินดีต้อนรับสู่ Vue Blog
description: บทความแรกของเรา
date: 2024-01-15
category: tutorial
tags:
  - vue
  - nuxt
  - beginner
author:
  name: สมชาย ใจดี
  avatar: /avatars/somchai.jpg
published: true
featured: false
readingTime: 5
cover: /images/hello-world.jpg
---

# ยินดีต้อนรับสู่ Vue Blog

เนื้อหาบทความที่นี่...
```

### Markdown Features

```markdown
---
title: Markdown Examples
---

# Heading 1
## Heading 2
### Heading 3

**Bold text** and *italic text* and ~~strikethrough~~

[Link text](https://example.com)

![Image alt](./image.jpg)

> Blockquote text

- List item 1
- List item 2
  - Nested item

1. Ordered item 1
2. Ordered item 2

| Column 1 | Column 2 | Column 3 |
|----------|----------|----------|
| Cell 1   | Cell 2   | Cell 3   |
| Cell 4   | Cell 5   | Cell 6   |

\`\`\`javascript
// Code block กับ syntax highlighting
const greeting = 'Hello, Nuxt!'
console.log(greeting)
\`\`\`

Inline `code` ก็ได้
```

### Nested Content

```markdown
---
title: Vue Composables Guide
description: เรียนรู้ Vue Composables ทั้งหมด
---

## useState

`useState` ใช้สำหรับ...

::: tip
นี่คือ tip box ใน Nuxt Content
:::

::: warning
นี่คือ warning
:::

::: danger
นี่คือ danger message
:::
```

---

## queryContent() API

### Basic Queries

```typescript
// ดึง content ทั้งหมดใน blog/
const posts = await queryContent('/blog').find()

// ดึง content เดียว
const post = await queryContent('/blog/hello-world').findOne()

// ดึงพร้อม filter
const vuePosts = await queryContent('/blog')
  .where({ tags: { $contains: 'vue' } })
  .find()

// Sort
const latest = await queryContent('/blog')
  .sort({ date: -1 })
  .limit(10)
  .find()

// Select specific fields
const titles = await queryContent('/blog')
  .only(['title', 'date', '_path'])
  .find()

// Search
const results = await queryContent('/blog')
  .where({ $text: { $search: 'nuxt' } })
  .find()
```

### ใช้ใน Pages

```vue
<!-- pages/blog/index.vue -->
<script setup lang="ts">
// queryContent ใช้กับ useAsyncData เพื่อ SSR
const { data: posts } = await useAsyncData('blog-posts', () =>
  queryContent('/blog')
    .where({ published: { $ne: false } })
    .sort({ date: -1 })
    .without(['body'])  // ไม่เอา body เพื่อ performance
    .find()
)

const { data: featuredPosts } = await useAsyncData('featured-posts', () =>
  queryContent('/blog')
    .where({ featured: true })
    .limit(3)
    .find()
)
</script>

<template>
  <div>
    <h1>บทความทั้งหมด</h1>
    
    <div v-for="post in posts" :key="post._path">
      <NuxtLink :to="post._path">
        <h2>{{ post.title }}</h2>
        <p>{{ post.description }}</p>
        <time>{{ new Date(post.date).toLocaleDateString('th-TH') }}</time>
      </NuxtLink>
    </div>
  </div>
</template>
```

### ดึงบทความแต่ละชิ้น

```vue
<!-- pages/blog/[...slug].vue -->
<script setup lang="ts">
const route = useRoute()
const slug = route.params.slug as string[]

// ดึง content ตาม path
const { data: post } = await useAsyncData(`post-${slug.join('-')}`, () =>
  queryContent(`/blog/${slug.join('/')}`)
    .findOne()
)

// Handle 404
if (!post.value) {
  throw createError({ statusCode: 404, statusMessage: 'Post not found' })
}

// SEO
useSeoMeta({
  title: () => `${post.value?.title} | Vue Blog`,
  description: () => post.value?.description,
  ogImage: () => post.value?.cover
})

// Navigation (prev/next post)
const { data: surroundingPosts } = await useAsyncData(`surround-${slug.join('-')}`, () =>
  queryContent('/blog')
    .only(['_path', 'title', 'date'])
    .sort({ date: 1 })
    .findSurround(`/blog/${slug.join('/')}`)
)

const [prevPost, nextPost] = surroundingPosts.value || []
</script>

<template>
  <article class="blog-post" v-if="post">
    <!-- Hero -->
    <div class="post-hero">
      <img v-if="post.cover" :src="post.cover" :alt="post.title" />
      <div class="post-header">
        <h1>{{ post.title }}</h1>
        <div class="post-meta">
          <span>{{ post.author?.name }}</span>
          <time>{{ new Date(post.date).toLocaleDateString('th-TH') }}</time>
          <span>{{ post.readingTime }} นาที</span>
        </div>
        <div class="tags">
          <NuxtLink
            v-for="tag in post.tags"
            :key="tag"
            :to="`/tags/${tag}`"
          >
            #{{ tag }}
          </NuxtLink>
        </div>
      </div>
    </div>
    
    <!-- Content - ใช้ ContentRenderer render Markdown -->
    <ContentRenderer :value="post" class="prose prose-lg max-w-none" />
    
    <!-- Navigation -->
    <nav class="post-nav">
      <NuxtLink v-if="prevPost" :to="prevPost._path" class="prev">
        ← {{ prevPost.title }}
      </NuxtLink>
      <NuxtLink v-if="nextPost" :to="nextPost._path" class="next">
        {{ nextPost.title }} →
      </NuxtLink>
    </nav>
  </article>
</template>

<style>
/* Prose styles for markdown content */
.prose {
  color: #1e293b;
  max-width: 65ch;
  line-height: 1.75;
}

.prose h2 { font-size: 1.5rem; font-weight: 700; margin: 2rem 0 1rem; }
.prose h3 { font-size: 1.25rem; font-weight: 600; margin: 1.5rem 0 0.75rem; }
.prose p { margin-bottom: 1.25rem; }
.prose pre { background: #1e293b; border-radius: 8px; padding: 1rem; overflow-x: auto; }
.prose code { font-size: 0.875em; }
.prose ul, .prose ol { padding-left: 1.5rem; margin-bottom: 1.25rem; }
.prose blockquote { border-left: 3px solid #00dc82; padding-left: 1rem; color: #64748b; }
</style>
```

---

## Content กับ TypeScript

### กำหนด Type สำหรับ Content

```typescript
// types/content.ts
export interface BlogPost {
  _path: string
  _type: 'markdown'
  title: string
  description: string
  date: string
  category: string
  tags: string[]
  author: {
    name: string
    avatar: string
  }
  published: boolean
  featured: boolean
  readingTime: number
  cover?: string
  body: any
}

export interface DocPage {
  _path: string
  title: string
  description: string
  section: string
  order: number
  body: any
}
```

### ใช้ Type ใน Query

```typescript
// pages/blog/index.vue
import type { BlogPost } from '~/types/content'

const { data: posts } = await useAsyncData<BlogPost[]>('posts', () =>
  queryContent<BlogPost>('/blog')
    .where({ published: true })
    .sort({ date: -1 })
    .find()
)
```

---

## MDC (Markdown Components)

### สร้าง Custom Components สำหรับ Markdown

```vue
<!-- components/content/CallOut.vue -->
<!-- ใช้ใน markdown: ::call-out{type="warning"} -->
<script setup lang="ts">
defineProps<{
  type?: 'info' | 'warning' | 'danger' | 'tip'
  title?: string
}>()
</script>

<template>
  <div :class="`callout callout-${type || 'info'}`">
    <div v-if="title" class="callout-title">{{ title }}</div>
    <slot />
  </div>
</template>

<style scoped>
.callout {
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin: 1.25rem 0;
  border-left: 4px solid;
}

.callout-info { background: #eff6ff; border-color: #3b82f6; }
.callout-warning { background: #fffbeb; border-color: #f59e0b; }
.callout-danger { background: #fef2f2; border-color: #ef4444; }
.callout-tip { background: #f0fdf4; border-color: #00dc82; }

.callout-title {
  font-weight: 700;
  margin-bottom: 0.5rem;
}
</style>
```

### ใช้ MDC ใน Markdown

```markdown
---
title: MDC Example
---

# MDC Example

ใช้ component ใน Markdown:

::call-out{type="tip" title="เคล็ดลับ"}
นี่คือ tip component ที่เราสร้างเอง
::

::call-out{type="warning"}
**ระวัง**: อย่าลืมตรวจสอบข้อมูลก่อนบันทึก
::

ใช้ inline component:
นี่คือ :badge[ใหม่]{type="success"} feature!
```

---

## ตัวอย่าง: Blog สมบูรณ์ด้วย Nuxt Content

### โครงสร้าง

```
content/blog/
├── 2024-01-hello-world.md
├── 2024-02-vue-composables.md
├── 2024-03-nuxt-routing.md
└── 2024-04-pinia-guide.md
```

### ตัวอย่างไฟล์ Markdown

```markdown
---
title: เรียนรู้ Vue Composables จากศูนย์
description: คู่มือสมบูรณ์สำหรับ Vue Composables
date: 2024-03-15
category: tutorial
tags:
  - vue
  - composables
  - javascript
author:
  name: สมชาย ใจดี
  avatar: /avatars/somchai.jpg
published: true
featured: true
readingTime: 8
cover: /blog/vue-composables.jpg
---

## Vue Composables คืออะไร?

Composable คือ function ที่ใช้ Vue Composition API เพื่อ encapsulate
และ reuse stateful logic

\`\`\`javascript
// composables/useCounter.js
export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  
  function increment() { count.value++ }
  function decrement() { count.value-- }
  function reset() { count.value = initialValue }
  
  return { count, increment, decrement, reset }
}
\`\`\`

::call-out{type="tip" title="Best Practice"}
ตั้งชื่อ Composable ด้วย prefix `use` เสมอ
::
```

### Blog Home Page

```vue
<!-- pages/blog/index.vue -->
<script setup lang="ts">
useSeoMeta({
  title: 'Blog - Vue & Nuxt.js',
  description: 'บทความเกี่ยวกับ Vue.js และ Nuxt.js'
})

const selectedCategory = ref('')
const searchQuery = ref('')

// ดึงบทความทั้งหมด
const { data: allPosts } = await useAsyncData('all-posts', () =>
  queryContent('/blog')
    .where({ published: true })
    .sort({ date: -1 })
    .without(['body'])
    .find()
)

// ดึง featured posts
const { data: featuredPosts } = await useAsyncData('featured', () =>
  queryContent('/blog')
    .where({ featured: true, published: true })
    .sort({ date: -1 })
    .limit(3)
    .without(['body'])
    .find()
)

// Categories จากบทความ
const categories = computed(() => {
  const cats = new Set(allPosts.value?.map(p => p.category) || [])
  return ['ทั้งหมด', ...Array.from(cats)]
})

// Filtered posts
const filteredPosts = computed(() => {
  let posts = allPosts.value || []
  
  if (selectedCategory.value && selectedCategory.value !== 'ทั้งหมด') {
    posts = posts.filter(p => p.category === selectedCategory.value)
  }
  
  if (searchQuery.value) {
    const q = searchQuery.value.toLowerCase()
    posts = posts.filter(p =>
      p.title.toLowerCase().includes(q) ||
      p.description?.toLowerCase().includes(q) ||
      p.tags?.some((t: string) => t.toLowerCase().includes(q))
    )
  }
  
  return posts
})
</script>

<template>
  <div class="blog-page">
    <!-- Featured Posts -->
    <section v-if="featuredPosts?.length && !searchQuery && !selectedCategory" class="featured-section">
      <h2>บทความแนะนำ</h2>
      <div class="featured-grid">
        <article v-for="post in featuredPosts" :key="post._path" class="featured-card">
          <NuxtLink :to="post._path">
            <img v-if="post.cover" :src="post.cover" :alt="post.title" />
            <div class="card-content">
              <span class="category-badge">{{ post.category }}</span>
              <h3>{{ post.title }}</h3>
              <p>{{ post.description }}</p>
              <div class="meta">
                <span>{{ post.author?.name }}</span>
                <span>{{ new Date(post.date).toLocaleDateString('th-TH') }}</span>
                <span>{{ post.readingTime }} นาที</span>
              </div>
            </div>
          </NuxtLink>
        </article>
      </div>
    </section>
    
    <!-- Search & Filter -->
    <section class="search-filter">
      <input
        v-model="searchQuery"
        type="search"
        placeholder="ค้นหาบทความ..."
        class="search-input"
      />
      <div class="categories">
        <button
          v-for="cat in categories"
          :key="cat"
          :class="{ active: selectedCategory === cat || (cat === 'ทั้งหมด' && !selectedCategory) }"
          @click="selectedCategory = cat === 'ทั้งหมด' ? '' : cat"
        >
          {{ cat }}
        </button>
      </div>
    </section>
    
    <!-- Posts Grid -->
    <section>
      <p class="results-count">
        {{ filteredPosts.length }} บทความ
        <span v-if="searchQuery"> สำหรับ "{{ searchQuery }}"</span>
      </p>
      
      <div v-if="filteredPosts.length === 0" class="empty">
        ไม่พบบทความที่ต้องการ
      </div>
      
      <div class="posts-grid">
        <article
          v-for="post in filteredPosts"
          :key="post._path"
          class="post-card"
        >
          <NuxtLink :to="post._path">
            <div v-if="post.cover" class="post-cover">
              <img :src="post.cover" :alt="post.title" loading="lazy" />
            </div>
            <div class="post-body">
              <span class="category">{{ post.category }}</span>
              <h2>{{ post.title }}</h2>
              <p>{{ post.description }}</p>
              <div class="post-footer">
                <div class="tags">
                  <span v-for="tag in post.tags?.slice(0, 3)" :key="tag" class="tag">
                    #{{ tag }}
                  </span>
                </div>
                <div class="meta">
                  <span>{{ new Date(post.date).toLocaleDateString('th-TH') }}</span>
                  <span>{{ post.readingTime }} นาที</span>
                </div>
              </div>
            </div>
          </NuxtLink>
        </article>
      </div>
    </section>
  </div>
</template>

<style scoped>
.blog-page { max-width: 1200px; margin: 0 auto; padding: 2rem 1rem; }

.featured-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1.5rem;
  margin-top: 1rem;
}

.featured-card {
  border-radius: 12px;
  overflow: hidden;
  border: 1px solid #e5e7eb;
  transition: transform 0.2s, box-shadow 0.2s;
}

.featured-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 24px rgba(0,0,0,0.1);
}

.featured-card img { width: 100%; height: 200px; object-fit: cover; }

.card-content { padding: 1.25rem; }

.category-badge {
  background: #dcfce7;
  color: #166534;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.2rem 0.6rem;
  border-radius: 999px;
}

.search-filter { margin: 2rem 0; }
.search-input { width: 100%; border: 1px solid #e5e7eb; border-radius: 10px; padding: 0.75rem 1rem; font-size: 1rem; margin-bottom: 1rem; }

.categories { display: flex; flex-wrap: wrap; gap: 0.5rem; }
.categories button { padding: 0.4rem 1rem; border: 1px solid #e5e7eb; border-radius: 999px; background: white; cursor: pointer; transition: all 0.2s; }
.categories button.active, .categories button:hover { background: #00dc82; color: white; border-color: #00dc82; }

.posts-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 1.5rem; }

.post-card { background: white; border: 1px solid #e5e7eb; border-radius: 12px; overflow: hidden; transition: transform 0.2s; }
.post-card:hover { transform: translateY(-3px); box-shadow: 0 8px 20px rgba(0,0,0,0.08); }
.post-card a { text-decoration: none; color: inherit; display: block; }

.post-cover img { width: 100%; height: 180px; object-fit: cover; }
.post-body { padding: 1.25rem; }
.post-body h2 { font-size: 1.1rem; font-weight: 700; margin: 0.5rem 0; color: #1e293b; }
.post-body p { color: #64748b; font-size: 0.875rem; line-height: 1.6; }

.post-footer { display: flex; justify-content: space-between; align-items: flex-end; margin-top: 0.75rem; }
.tags { display: flex; gap: 0.25rem; flex-wrap: wrap; }
.tag { font-size: 0.7rem; color: #00dc82; }
.meta { font-size: 0.75rem; color: #94a3b8; display: flex; gap: 0.5rem; }
.category { font-size: 0.75rem; color: #00dc82; font-weight: 600; text-transform: uppercase; }
</style>
```

### Blog Post Page

```vue
<!-- pages/blog/[...slug].vue -->
<script setup lang="ts">
const route = useRoute()
const slug = route.params.slug as string[]

const { data: post } = await useAsyncData(`post-${slug.join('-')}`, () =>
  queryContent(`/blog/${slug.join('/')}`).findOne()
)

if (!post.value) {
  throw createError({ statusCode: 404, statusMessage: 'ไม่พบบทความ' })
}

useSeoMeta({
  title: () => post.value?.title,
  description: () => post.value?.description,
  ogTitle: () => post.value?.title,
  ogDescription: () => post.value?.description,
  ogImage: () => post.value?.cover
})

const [prev, next] = await queryContent('/blog')
  .only(['_path', 'title'])
  .sort({ date: 1 })
  .findSurround(`/blog/${slug.join('/')}`)
</script>

<template>
  <div class="post-page" v-if="post">
    <!-- Cover -->
    <div v-if="post.cover" class="post-cover">
      <img :src="post.cover" :alt="post.title" />
    </div>
    
    <!-- Header -->
    <header class="post-header">
      <div class="breadcrumbs">
        <NuxtLink to="/">หน้าแรก</NuxtLink>
        <span>/</span>
        <NuxtLink to="/blog">บทความ</NuxtLink>
        <span>/</span>
        <span>{{ post.title }}</span>
      </div>
      
      <div class="category-tag">{{ post.category }}</div>
      <h1>{{ post.title }}</h1>
      <p class="description">{{ post.description }}</p>
      
      <div class="post-meta">
        <div class="author">
          <img v-if="post.author?.avatar" :src="post.author.avatar" :alt="post.author.name" />
          <span>{{ post.author?.name }}</span>
        </div>
        <time>{{ new Date(post.date).toLocaleDateString('th-TH', { year: 'numeric', month: 'long', day: 'numeric' }) }}</time>
        <span>{{ post.readingTime }} นาทีอ่าน</span>
      </div>
      
      <div class="tags">
        <NuxtLink v-for="tag in post.tags" :key="tag" :to="`/tags/${tag}`" class="tag">
          #{{ tag }}
        </NuxtLink>
      </div>
    </header>
    
    <!-- Content -->
    <ContentRenderer :value="post" class="prose" />
    
    <!-- Table of Contents -->
    <aside class="toc">
      <h4>สารบัญ</h4>
      <ul>
        <li v-for="link in post.body?.toc?.links" :key="link.id">
          <a :href="`#${link.id}`">{{ link.text }}</a>
          <ul v-if="link.children">
            <li v-for="child in link.children" :key="child.id">
              <a :href="`#${child.id}`">{{ child.text }}</a>
            </li>
          </ul>
        </li>
      </ul>
    </aside>
    
    <!-- Navigation -->
    <nav class="post-nav">
      <div class="prev-post" v-if="prev">
        <span>บทความก่อนหน้า</span>
        <NuxtLink :to="prev._path">{{ prev.title }}</NuxtLink>
      </div>
      <div class="next-post" v-if="next">
        <span>บทความถัดไป</span>
        <NuxtLink :to="next._path">{{ next.title }}</NuxtLink>
      </div>
    </nav>
  </div>
</template>

<style scoped>
.post-page { max-width: 860px; margin: 0 auto; padding: 2rem 1rem; }
.post-cover img { width: 100%; max-height: 400px; object-fit: cover; border-radius: 16px; margin-bottom: 2rem; }

.post-header { margin-bottom: 3rem; }
.breadcrumbs { display: flex; gap: 0.5rem; font-size: 0.875rem; color: #64748b; margin-bottom: 1rem; }
.breadcrumbs a { color: #00dc82; text-decoration: none; }

.category-tag { display: inline-block; background: #dcfce7; color: #166534; font-size: 0.75rem; font-weight: 700; padding: 0.25rem 0.75rem; border-radius: 999px; margin-bottom: 0.75rem; text-transform: uppercase; letter-spacing: 0.05em; }

h1 { font-size: 2.25rem; font-weight: 800; color: #0f172a; line-height: 1.2; margin-bottom: 0.75rem; }
.description { font-size: 1.125rem; color: #475569; margin-bottom: 1.5rem; }

.post-meta { display: flex; align-items: center; gap: 1.5rem; font-size: 0.875rem; color: #64748b; margin-bottom: 1rem; }
.author { display: flex; align-items: center; gap: 0.5rem; }
.author img { width: 32px; height: 32px; border-radius: 50%; }

.tags { display: flex; gap: 0.5rem; flex-wrap: wrap; }
.tag { background: #f0fdf4; color: #00dc82; font-size: 0.8rem; padding: 0.2rem 0.6rem; border-radius: 6px; text-decoration: none; }

.prose { line-height: 1.8; color: #1e293b; }

.toc { background: #f8fafc; border-radius: 12px; padding: 1.5rem; margin: 2rem 0; }
.toc h4 { font-weight: 700; margin-bottom: 0.75rem; }
.toc a { color: #475569; text-decoration: none; font-size: 0.875rem; }
.toc a:hover { color: #00dc82; }

.post-nav { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; margin-top: 3rem; padding-top: 2rem; border-top: 1px solid #e5e7eb; }
.prev-post, .next-post { background: #f8fafc; border-radius: 12px; padding: 1rem 1.25rem; }
.next-post { text-align: right; }
.post-nav span { display: block; font-size: 0.75rem; color: #94a3b8; margin-bottom: 0.25rem; }
.post-nav a { font-weight: 600; color: #1e293b; text-decoration: none; }
.post-nav a:hover { color: #00dc82; }
</style>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **@nuxt/content คืออะไร** - Content-driven framework บน Nuxt
2. **ติดตั้งและตั้งค่า** - Module configuration
3. **สร้าง Markdown Content** - Front matter, syntax
4. **queryContent()** - Query API แบบ database
5. **TypeScript** - Type-safe content
6. **MDC** - Vue components ใน Markdown
7. **Blog สมบูรณ์** - ตัวอย่าง full blog implementation

---

## สรุปทั้งหมด Series Nuxt (Part 21-30)

| Part | หัวข้อ | ความสำคัญ |
|------|--------|-----------|
| 21 | Nuxt Intro | SSR, SSG, Project structure |
| 22 | File-based Routing | Dynamic routes, catch-all |
| 23 | Layouts | Default, named, dynamic layouts |
| 24 | Data Fetching | useFetch, useAsyncData, $fetch |
| 25 | Middleware | Auth, global, server middleware |
| 26 | Plugins | Client/server plugins, third-party |
| 27 | Modules | Community modules, Tailwind, Images |
| 28 | Server API | CRUD API ด้วย Nitro |
| 29 | Authentication | JWT, sessions, protected routes |
| 30 | Nuxt Content | Markdown CMS, queryContent |

**จบ Series Nuxt.js!** - ถัดไปสามารถเรียนรู้เรื่อง Deployment, Testing, และ Performance Optimization
