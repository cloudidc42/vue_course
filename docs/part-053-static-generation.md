# Part 53: Static Site Generation (SSG) ด้วย Nuxt.js

## SSG คืออะไร?

Static Site Generation (SSG) คือการ pre-render ทุกหน้าตอน build time และเก็บเป็น HTML files ที่พร้อม serve โดยตรง ทำให้เว็บโหลดเร็วมากและ deploy ได้ง่ายบน CDN

## เมื่อไหร่ควรใช้ SSG?

- บล็อก, documentation
- หน้า marketing
- Portfolio
- ข้อมูลที่ไม่ค่อยเปลี่ยน

## 1. SSG ใน Nuxt

```typescript
// nuxt.config.ts - Fully Static
export default defineNuxtConfig({
  // ทำให้ทั้งแอปเป็น static
  ssr: true, // ยังต้องใช้ SSR mode สำหรับ generate
  
  // หรือใช้ hybrid rendering
  routeRules: {
    '/': { prerender: true },
    '/about': { prerender: true },
    '/blog/**': { prerender: true }
  }
})
```

## 2. nuxt generate command

```bash
# Generate static site
npx nuxt generate

# Output จะอยู่ใน .output/public/
# โครงสร้าง:
# .output/public/
# ├── index.html
# ├── about/
# │   └── index.html
# ├── blog/
# │   ├── index.html
# │   └── my-first-post/
# │       └── index.html
# └── _nuxt/
#     ├── app.xxx.js
#     └── styles.xxx.css

# Preview static site
npx nuxt preview
```

## 3. Pre-rendering Dynamic Routes

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    prerender: {
      // Pre-render routes เหล่านี้
      routes: [
        '/',
        '/about',
        '/sitemap.xml'
      ],
      
      // Crawl links อัตโนมัติ
      crawlLinks: true,
      
      // ไม่ pre-render routes เหล่านี้
      ignore: ['/admin/**', '/api/**']
    }
  }
})
```

### สร้าง Dynamic Routes ด้วย API

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  hooks: {
    async 'nitro:config'(nitroConfig) {
      // เพิ่ม routes แบบ dynamic จาก API/database
      const posts = await fetchAllPosts()
      
      nitroConfig.prerender = nitroConfig.prerender || {}
      nitroConfig.prerender.routes = nitroConfig.prerender.routes || []
      
      posts.forEach(post => {
        nitroConfig.prerender!.routes!.push(`/blog/${post.slug}`)
      })
    }
  }
})

async function fetchAllPosts() {
  const response = await fetch('https://api.example.com/posts?per_page=1000')
  return response.json()
}
```

```vue
<!-- pages/blog/[slug].vue -->
<script setup>
// สำหรับ SSG ต้องมี useFetch หรือ useAsyncData
const route = useRoute()
const { data: post } = await useFetch(`/api/blog/${route.params.slug}`)

if (!post.value) {
  throw createError({ statusCode: 404, statusMessage: 'ไม่พบบทความ' })
}

// SEO
useSeoMeta({
  title: post.value.title,
  ogTitle: post.value.title,
  description: post.value.excerpt,
  ogImage: post.value.coverImage
})
</script>
```

## 4. Incremental Static Generation (ISR)

```typescript
// nuxt.config.ts - ISR Configuration
export default defineNuxtConfig({
  routeRules: {
    // Revalidate ทุก 1 ชั่วโมง
    '/blog/**': { isr: 3600 },
    
    // Stale While Revalidate
    '/products/**': { swr: 300 },
    
    // ไม่มี revalidation
    '/about': { prerender: true }
  }
})
```

```typescript
// server/api/blog/[slug].ts - กับ caching
export default defineCachedEventHandler(
  async (event) => {
    const slug = getRouterParam(event, 'slug')
    
    const post = await prisma.post.findUnique({
      where: { slug, published: true }
    })
    
    if (!post) throw createError({ statusCode: 404 })
    
    return post
  },
  {
    maxAge: 3600,
    swr: true,
    getKey: (event) => `blog:${getRouterParam(event, 'slug')}`
  }
)
```

## 5. Hybrid Rendering

```typescript
// nuxt.config.ts - Hybrid Rendering
export default defineNuxtConfig({
  routeRules: {
    // Static pages
    '/': { prerender: true },
    '/about': { prerender: true },
    '/pricing': { prerender: true },
    
    // SSR with cache
    '/blog': { swr: 60 },           // cache 1 minute
    '/blog/**': { swr: 3600 },      // cache 1 hour
    
    // Dynamic (SSR, no cache)
    '/account/**': { ssr: true },
    '/checkout/**': { ssr: true },
    
    // CSR only
    '/admin/**': { ssr: false },
    '/dashboard/**': { ssr: false }
  }
})
```

## 6. Deploy to CDN

### Deploy to Netlify

```bash
# Build static site
npx nuxt generate

# Netlify config
```

```toml
# netlify.toml
[build]
  command = "nuxt generate"
  publish = ".output/public"

[[headers]]
  for = "/_nuxt/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "/*.html"
  [headers.values]
    Cache-Control = "public, max-age=0, must-revalidate"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

### Deploy to GitHub Pages

```yaml
# .github/workflows/deploy-pages.yml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      
      - run: npm ci
      
      - name: Generate static site
        run: npx nuxt generate
        env:
          NUXT_PUBLIC_BASE_URL: https://username.github.io/repo-name
      
      - uses: actions/configure-pages@v4
      
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .output/public
      
      - id: deployment
        uses: actions/deploy-pages@v4
```

## 7. ตัวอย่าง: Fully Static Blog

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@nuxt/content', // สำหรับ markdown-based blog
    '@nuxt/image',
    '@nuxtjs/sitemap'
  ],
  
  routeRules: {
    '/': { prerender: true },
    '/blog/**': { prerender: true }
  },
  
  content: {
    highlight: {
      theme: 'github-dark',
      langs: ['js', 'ts', 'vue', 'bash', 'json']
    }
  },
  
  sitemap: {
    hostname: 'https://myblog.com',
    gzip: true
  }
})
```

```vue
<!-- pages/blog/index.vue -->
<template>
  <div class="blog-index">
    <h1>Blog</h1>
    
    <!-- Search (client-side หลัง hydration) -->
    <input
      v-model="searchQuery"
      type="search"
      placeholder="ค้นหาบทความ..."
      class="search-input"
    />
    
    <!-- Category Filter -->
    <div class="categories">
      <button
        v-for="cat in categories"
        :key="cat"
        @click="selectedCategory = cat"
        :class="{ active: selectedCategory === cat }"
      >
        {{ cat }}
      </button>
    </div>
    
    <!-- Posts Grid -->
    <div class="posts-grid">
      <article
        v-for="post in filteredPosts"
        :key="post._path"
        class="post-card"
      >
        <NuxtLink :to="post._path">
          <NuxtImg
            v-if="post.cover"
            :src="post.cover"
            :alt="post.title"
            width="400"
            height="250"
            loading="lazy"
          />
          <div class="post-content">
            <span class="category-badge">{{ post.category }}</span>
            <h2>{{ post.title }}</h2>
            <p>{{ post.description }}</p>
            <div class="post-meta">
              <time>{{ formatDate(post.date) }}</time>
              <span>{{ post.readTime }} นาที</span>
            </div>
          </div>
        </NuxtLink>
      </article>
    </div>
    
    <!-- Empty State -->
    <div v-if="filteredPosts.length === 0" class="empty">
      <p>ไม่พบบทความที่ตรงกับเงื่อนไข</p>
    </div>
  </div>
</template>

<script setup>
// ดึงบทความจาก @nuxt/content
const { data: posts } = await useAsyncData('blog-posts', () =>
  queryContent('blog')
    .where({ draft: { $ne: true } })
    .sort({ date: -1 })
    .find()
)

const searchQuery = ref('')
const selectedCategory = ref('ทั้งหมด')

const categories = computed(() => {
  const cats = new Set(['ทั้งหมด', ...posts.value?.map(p => p.category) || []])
  return [...cats]
})

const filteredPosts = computed(() => {
  if (!posts.value) return []
  
  return posts.value.filter(post => {
    const matchesSearch = !searchQuery.value ||
      post.title.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      post.description?.toLowerCase().includes(searchQuery.value.toLowerCase())
    
    const matchesCategory = selectedCategory.value === 'ทั้งหมด' ||
      post.category === selectedCategory.value
    
    return matchesSearch && matchesCategory
  })
})

const formatDate = (date) => {
  return new Date(date).toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

useSeoMeta({
  title: 'Blog - Vue.js และ Nuxt.js',
  description: 'บทความเกี่ยวกับ Vue.js, Nuxt.js และการพัฒนา Web'
})
</script>
```

```vue
<!-- pages/blog/[...slug].vue -->
<template>
  <article class="blog-post" v-if="post">
    <header class="post-header">
      <NuxtImg
        v-if="post.cover"
        :src="post.cover"
        :alt="post.title"
        width="1200"
        height="600"
        class="cover-image"
        :preload="true"
      />
      
      <div class="post-meta-header">
        <NuxtLink :to="`/blog?category=${post.category}`" class="category">
          {{ post.category }}
        </NuxtLink>
        <time>{{ formatDate(post.date) }}</time>
        <span>{{ post.readTime }} นาทีอ่าน</span>
      </div>
      
      <h1>{{ post.title }}</h1>
      <p class="excerpt">{{ post.description }}</p>
      
      <div class="author">
        <img :src="post.author.avatar" :alt="post.author.name" />
        <span>{{ post.author.name }}</span>
      </div>
    </header>
    
    <!-- Rendered Markdown Content -->
    <div class="post-body">
      <ContentRenderer :value="post" />
    </div>
    
    <!-- Table of Contents -->
    <aside class="toc">
      <h3>เนื้อหา</h3>
      <ContentNavigation :query="queryContent(`/blog/${post._file?.replace('.md', '')}`)" />
    </aside>
    
    <!-- Related Posts -->
    <section class="related">
      <h2>บทความที่เกี่ยวข้อง</h2>
      <div class="related-grid">
        <NuxtLink
          v-for="related in relatedPosts"
          :key="related._path"
          :to="related._path"
          class="related-card"
        >
          <strong>{{ related.title }}</strong>
          <span>{{ related.description }}</span>
        </NuxtLink>
      </div>
    </section>
  </article>
</template>

<script setup>
const route = useRoute()

// ดึงบทความ
const { data: post } = await useAsyncData(route.path, () =>
  queryContent(route.path).findOne()
)

if (!post.value) {
  throw createError({ statusCode: 404, statusMessage: 'ไม่พบบทความ' })
}

// Related posts
const { data: relatedPosts } = await useAsyncData(
  `related-${route.path}`,
  () =>
    queryContent('blog')
      .where({
        category: post.value?.category,
        _path: { $ne: route.path },
        draft: { $ne: true }
      })
      .limit(3)
      .find()
)

const formatDate = (date) => new Date(date).toLocaleDateString('th-TH', {
  year: 'numeric', month: 'long', day: 'numeric'
})

// SEO
useSeoMeta({
  title: post.value.title,
  ogTitle: post.value.title,
  description: post.value.description,
  ogImage: post.value.cover,
  twitterCard: 'summary_large_image',
  articlePublishedTime: post.value.date,
  articleAuthor: post.value.author?.name
})

// Schema.org structured data
useSchemaOrg([
  defineArticle({
    headline: post.value.title,
    description: post.value.description,
    image: post.value.cover,
    datePublished: post.value.date,
    author: post.value.author?.name
  })
])
</script>
```

## 8. Sitemap และ RSS Feed

```typescript
// server/routes/sitemap.xml.ts
export default defineEventHandler(async (event) => {
  setHeader(event, 'Content-Type', 'application/xml')
  
  const posts = await queryContent('blog').find()
  const baseUrl = 'https://myblog.com'
  
  const urls = [
    { loc: '/', lastmod: new Date().toISOString(), changefreq: 'daily', priority: 1.0 },
    { loc: '/blog', lastmod: new Date().toISOString(), changefreq: 'daily', priority: 0.9 },
    ...posts.map(post => ({
      loc: post._path,
      lastmod: post.date || new Date().toISOString(),
      changefreq: 'weekly',
      priority: 0.7
    }))
  ]
  
  return `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  ${urls.map(url => `
  <url>
    <loc>${baseUrl}${url.loc}</loc>
    <lastmod>${url.lastmod}</lastmod>
    <changefreq>${url.changefreq}</changefreq>
    <priority>${url.priority}</priority>
  </url>`).join('')}
</urlset>`
})
```

```typescript
// server/routes/rss.xml.ts
export default defineEventHandler(async (event) => {
  setHeader(event, 'Content-Type', 'application/rss+xml')
  
  const posts = await queryContent('blog')
    .sort({ date: -1 })
    .limit(20)
    .find()
  
  const baseUrl = 'https://myblog.com'
  
  return `<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>MyBlog</title>
    <link>${baseUrl}</link>
    <description>บทความเกี่ยวกับ Vue.js และ Nuxt.js</description>
    <language>th</language>
    ${posts.map(post => `
    <item>
      <title>${post.title}</title>
      <link>${baseUrl}${post._path}</link>
      <description>${post.description || ''}</description>
      <pubDate>${new Date(post.date).toUTCString()}</pubDate>
      <guid>${baseUrl}${post._path}</guid>
    </item>`).join('')}
  </channel>
</rss>`
})
```

## 9. Open Graph Images แบบ Dynamic

```typescript
// server/routes/og/[...slug].png.ts
import { createCanvas, loadImage, registerFont } from 'canvas'

export default defineEventHandler(async (event) => {
  const slug = getRouterParam(event, 'slug')
  const post = await queryContent(`/blog/${slug}`).findOne()
  
  if (!post) throw createError({ statusCode: 404 })
  
  const canvas = createCanvas(1200, 630)
  const ctx = canvas.getContext('2d')
  
  // Background
  ctx.fillStyle = '#1a1a2e'
  ctx.fillRect(0, 0, 1200, 630)
  
  // Title text
  ctx.font = 'bold 56px sans-serif'
  ctx.fillStyle = '#ffffff'
  ctx.fillText(post.title || '', 80, 280)
  
  // Author
  ctx.font = '32px sans-serif'
  ctx.fillStyle = '#94a3b8'
  ctx.fillText(post.author?.name || 'MyBlog', 80, 550)
  
  setHeader(event, 'Content-Type', 'image/png')
  setHeader(event, 'Cache-Control', 'public, max-age=31536000')
  
  return canvas.toBuffer('image/png')
})
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **SSG Basics** - ทำความเข้าใจ Static Site Generation
2. **nuxt generate** - Build และ preview static site
3. **Dynamic Routes** - Pre-render dynamic pages
4. **ISR** - Incremental Static Regeneration
5. **Hybrid Rendering** - ผสม SSG, SSR, CSR ตาม route
6. **Deploy to CDN** - Netlify, GitHub Pages
7. **Static Blog** - ตัวอย่าง blog ด้วย @nuxt/content
8. **Sitemap & RSS** - สร้าง sitemap.xml และ rss.xml อัตโนมัติ
9. **Dynamic OG Images** - สร้าง Open Graph images แบบ dynamic
