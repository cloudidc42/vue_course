# Part 35: SEO และ Meta Tags

## SEO ใน Nuxt.js คืออะไร?

Nuxt.js มีความสามารถ Server-Side Rendering (SSR) ที่ช่วยให้ Search Engine Crawlers เข้าถึงเนื้อหาได้ง่าย ทำให้ SEO ดีกว่า SPA ทั่วไป Nuxt มี composables ที่ช่วยจัดการ meta tags, structured data, และ sitemap ได้อย่างสะดวก

---

## 1. SEO ใน Nuxt.js พื้นฐาน

```ts
// nuxt.config.ts - Global SEO defaults
export default defineNuxtConfig({
  app: {
    head: {
      charset: 'utf-8',
      viewport: 'width=device-width, initial-scale=1',
      title: 'My Website',
      titleTemplate: '%s | My Website',
      meta: [
        { name: 'description', content: 'คำอธิบายเว็บไซต์ของคุณ' },
        { name: 'keywords', content: 'keyword1, keyword2, keyword3' },
        { name: 'author', content: 'ชื่อผู้เขียน' },
        { property: 'og:type', content: 'website' },
        { property: 'og:site_name', content: 'My Website' }
      ],
      link: [
        { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' },
        { rel: 'canonical', href: 'https://mywebsite.com' }
      ]
    }
  }
})
```

---

## 2. useHead() Composable

```vue
<!-- pages/about.vue -->
<script setup lang="ts">
useHead({
  title: 'เกี่ยวกับเรา',
  meta: [
    { name: 'description', content: 'เรียนรู้เพิ่มเติมเกี่ยวกับทีมของเรา' },
    { name: 'keywords', content: 'เกี่ยวกับ, ทีม, บริษัท' }
  ],
  link: [
    { rel: 'canonical', href: 'https://mywebsite.com/about' }
  ],
  script: [
    {
      type: 'application/ld+json',
      innerHTML: JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'Organization',
        name: 'My Company'
      })
    }
  ]
})
</script>

<template>
  <main>
    <h1>เกี่ยวกับเรา</h1>
  </main>
</template>
```

```vue
<!-- pages/blog/[slug].vue - Dynamic SEO -->
<script setup lang="ts">
const route = useRoute()
const { data: post } = await useFetch(`/api/posts/${route.params.slug}`)

// useHead กับ dynamic content
useHead({
  title: () => post.value?.title ?? 'กำลังโหลด...',
  meta: [
    {
      name: 'description',
      content: () => post.value?.excerpt ?? ''
    }
  ]
})
</script>
```

---

## 3. useSeoMeta() Composable

```vue
<!-- pages/product/[id].vue -->
<script setup lang="ts">
const route = useRoute()
const { data: product } = await useFetch(`/api/products/${route.params.id}`)

useSeoMeta({
  // Basic SEO
  title: () => product.value?.name ?? '',
  description: () => product.value?.description ?? '',
  keywords: () => product.value?.tags?.join(', ') ?? '',

  // Open Graph
  ogTitle: () => product.value?.name ?? '',
  ogDescription: () => product.value?.description ?? '',
  ogImage: () => product.value?.imageUrl ?? '',
  ogUrl: () => `https://myshop.com/product/${route.params.id}`,
  ogType: 'product',
  ogSiteName: 'My Shop',
  ogLocale: 'th_TH',

  // Twitter Card
  twitterCard: 'summary_large_image',
  twitterTitle: () => product.value?.name ?? '',
  twitterDescription: () => product.value?.description ?? '',
  twitterImage: () => product.value?.imageUrl ?? '',
  twitterSite: '@myshop',
  twitterCreator: '@author',

  // Article specific (สำหรับ blog)
  articleAuthor: 'John Doe',
  articlePublishedTime: () => product.value?.createdAt ?? '',
  articleModifiedTime: () => product.value?.updatedAt ?? ''
})
</script>
```

---

## 4. Open Graph Tags

```vue
<!-- composables/useSocialMeta.ts -->
<script lang="ts">
interface SocialMetaOptions {
  title: string
  description: string
  imageUrl: string
  url: string
  type?: 'website' | 'article' | 'product'
}

export function useSocialMeta(options: MaybeRef<SocialMetaOptions>) {
  const opts = computed(() => unref(options))

  useSeoMeta({
    // Open Graph
    ogTitle: () => opts.value.title,
    ogDescription: () => opts.value.description,
    ogImage: () => opts.value.imageUrl,
    ogImageAlt: () => opts.value.title,
    ogImageWidth: '1200',
    ogImageHeight: '630',
    ogUrl: () => opts.value.url,
    ogType: () => opts.value.type ?? 'website',

    // Twitter
    twitterCard: 'summary_large_image',
    twitterTitle: () => opts.value.title,
    twitterDescription: () => opts.value.description,
    twitterImage: () => opts.value.imageUrl,
    twitterImageAlt: () => opts.value.title
  })
}
</script>
```

```vue
<!-- pages/blog/[slug].vue -->
<script setup lang="ts">
const route = useRoute()
const config = useRuntimeConfig()
const { data: article } = await useFetch(`/api/articles/${route.params.slug}`)

useSocialMeta({
  title: article.value?.title ?? '',
  description: article.value?.excerpt ?? '',
  imageUrl: article.value?.coverImage ?? `${config.public.siteUrl}/og-default.jpg`,
  url: `${config.public.siteUrl}/blog/${route.params.slug}`,
  type: 'article'
})
</script>
```

---

## 5. Twitter Cards

```vue
<!-- pages/post/[id].vue -->
<script setup lang="ts">
const post = ref({
  title: 'บทความน่าสนใจ',
  excerpt: 'สรุปเนื้อหาของบทความ',
  coverImage: 'https://example.com/cover.jpg',
  author: {
    name: 'สมชาย',
    twitter: '@somchai'
  }
})

useSeoMeta({
  // Twitter Card Types: summary, summary_large_image, app, player
  twitterCard: 'summary_large_image',
  twitterSite: '@mysite',
  twitterCreator: () => post.value.author.twitter,
  twitterTitle: () => post.value.title,
  twitterDescription: () => post.value.excerpt,
  twitterImage: () => post.value.coverImage,
  twitterImageAlt: () => `รูปปกของ: ${post.value.title}`
})
</script>
```

---

## 6. Structured Data (JSON-LD)

```vue
<!-- pages/blog/[slug].vue - Article Schema -->
<script setup lang="ts">
const route = useRoute()
const { data: article } = await useFetch(`/api/articles/${route.params.slug}`)

// JSON-LD Structured Data
useHead({
  script: [
    {
      type: 'application/ld+json',
      innerHTML: computed(() => JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'Article',
        headline: article.value?.title,
        description: article.value?.excerpt,
        image: article.value?.coverImage,
        datePublished: article.value?.publishedAt,
        dateModified: article.value?.updatedAt,
        author: {
          '@type': 'Person',
          name: article.value?.author?.name,
          url: `https://mysite.com/author/${article.value?.author?.slug}`
        },
        publisher: {
          '@type': 'Organization',
          name: 'My Blog',
          logo: {
            '@type': 'ImageObject',
            url: 'https://mysite.com/logo.png'
          }
        },
        mainEntityOfPage: {
          '@type': 'WebPage',
          '@id': `https://mysite.com/blog/${route.params.slug}`
        }
      }))
    }
  ]
})
</script>
```

```vue
<!-- pages/product/[id].vue - Product Schema -->
<script setup lang="ts">
const { data: product } = await useFetch(`/api/products/${useRoute().params.id}`)

useHead({
  script: [
    {
      type: 'application/ld+json',
      innerHTML: computed(() => JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'Product',
        name: product.value?.name,
        description: product.value?.description,
        image: product.value?.images,
        sku: product.value?.sku,
        brand: {
          '@type': 'Brand',
          name: product.value?.brand
        },
        offers: {
          '@type': 'Offer',
          price: product.value?.price,
          priceCurrency: 'THB',
          availability: product.value?.stock > 0
            ? 'https://schema.org/InStock'
            : 'https://schema.org/OutOfStock',
          priceValidUntil: '2025-12-31',
          seller: {
            '@type': 'Organization',
            name: 'My Shop'
          }
        },
        aggregateRating: product.value?.rating ? {
          '@type': 'AggregateRating',
          ratingValue: product.value.rating.average,
          reviewCount: product.value.rating.count
        } : undefined
      }))
    }
  ]
})
</script>
```

---

## 7. Sitemap

```bash
# ติดตั้ง @nuxtjs/sitemap
npm install -D @nuxtjs/sitemap
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/sitemap'],

  site: {
    url: 'https://mywebsite.com',
    name: 'My Website'
  },

  sitemap: {
    strictNuxtContentPaths: true,
    // สำหรับ static routes
    urls: [
      '/',
      '/about',
      '/contact'
    ]
  }
})
```

```ts
// server/routes/sitemap.xml.ts - Custom Dynamic Sitemap
export default defineEventHandler(async (event) => {
  const posts = await $fetch('/api/posts')
  const products = await $fetch('/api/products')

  const baseUrl = 'https://mywebsite.com'

  const staticUrls = [
    { loc: '/', priority: 1.0, changefreq: 'daily' },
    { loc: '/about', priority: 0.8, changefreq: 'monthly' },
    { loc: '/blog', priority: 0.9, changefreq: 'daily' }
  ]

  const postUrls = posts.map((post: any) => ({
    loc: `/blog/${post.slug}`,
    lastmod: post.updatedAt,
    priority: 0.7,
    changefreq: 'weekly'
  }))

  const productUrls = products.map((product: any) => ({
    loc: `/products/${product.id}`,
    lastmod: product.updatedAt,
    priority: 0.6,
    changefreq: 'weekly'
  }))

  const allUrls = [...staticUrls, ...postUrls, ...productUrls]

  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
${allUrls.map(url => `  <url>
    <loc>${baseUrl}${url.loc}</loc>
    ${url.lastmod ? `<lastmod>${new Date(url.lastmod).toISOString()}</lastmod>` : ''}
    <changefreq>${url.changefreq}</changefreq>
    <priority>${url.priority}</priority>
  </url>`).join('\n')}
</urlset>`

  setHeader(event, 'Content-Type', 'application/xml')
  return xml
})
```

---

## 8. Robots.txt

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  robots: {
    disallow: ['/admin', '/private', '/api/'],
    allow: '/',
    sitemap: 'https://mywebsite.com/sitemap.xml'
  }
})
```

```ts
// public/robots.txt หรือ server/routes/robots.txt.ts
export default defineEventHandler(() => {
  const config = useRuntimeConfig()
  const isProd = config.public.env === 'production'

  return isProd
    ? `User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/
Disallow: /private/
Sitemap: https://mywebsite.com/sitemap.xml`
    : `User-agent: *
Disallow: /`
})
```

---

## 9. ตัวอย่าง: Blog Post SEO

```vue
<!-- pages/blog/[slug].vue - Complete Blog SEO -->
<script setup lang="ts">
interface BlogPost {
  id: number
  title: string
  slug: string
  excerpt: string
  content: string
  coverImage: string
  author: { name: string; twitter: string }
  category: string
  tags: string[]
  publishedAt: string
  updatedAt: string
  readingTime: number
}

const route = useRoute()
const config = useRuntimeConfig()
const { data: post, error } = await useFetch<BlogPost>(
  `/api/posts/${route.params.slug}`
)

if (error.value || !post.value) {
  throw createError({ statusCode: 404, message: 'ไม่พบบทความ' })
}

const siteUrl = config.public.siteUrl
const postUrl = `${siteUrl}/blog/${route.params.slug}`

// Complete SEO setup
useSeoMeta({
  title: post.value.title,
  description: post.value.excerpt,
  keywords: post.value.tags.join(', '),
  author: post.value.author.name,

  // Open Graph
  ogTitle: post.value.title,
  ogDescription: post.value.excerpt,
  ogImage: post.value.coverImage,
  ogImageWidth: '1200',
  ogImageHeight: '630',
  ogUrl: postUrl,
  ogType: 'article',
  ogSiteName: 'My Blog',
  ogLocale: 'th_TH',

  // Article
  articlePublishedTime: post.value.publishedAt,
  articleModifiedTime: post.value.updatedAt,
  articleAuthor: post.value.author.name,
  articleSection: post.value.category,
  articleTag: post.value.tags,

  // Twitter
  twitterCard: 'summary_large_image',
  twitterSite: '@myblog',
  twitterCreator: post.value.author.twitter,
  twitterTitle: post.value.title,
  twitterDescription: post.value.excerpt,
  twitterImage: post.value.coverImage
})

// JSON-LD
useHead({
  link: [
    { rel: 'canonical', href: postUrl }
  ],
  script: [
    {
      type: 'application/ld+json',
      innerHTML: JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'BlogPosting',
        headline: post.value.title,
        description: post.value.excerpt,
        image: post.value.coverImage,
        url: postUrl,
        datePublished: post.value.publishedAt,
        dateModified: post.value.updatedAt,
        author: {
          '@type': 'Person',
          name: post.value.author.name
        },
        publisher: {
          '@type': 'Organization',
          name: 'My Blog',
          logo: { '@type': 'ImageObject', url: `${siteUrl}/logo.png` }
        },
        keywords: post.value.tags.join(', '),
        wordCount: post.value.content.split(' ').length,
        timeRequired: `PT${post.value.readingTime}M`
      })
    }
  ]
})
</script>

<template>
  <article>
    <header>
      <h1>{{ post?.title }}</h1>
      <p>{{ post?.excerpt }}</p>
      <img :src="post?.coverImage" :alt="post?.title" />
      <time :datetime="post?.publishedAt">
        {{ new Date(post?.publishedAt ?? '').toLocaleDateString('th-TH') }}
      </time>
    </header>
    <div v-html="post?.content" />
  </article>
</template>
```

---

## 10. ตัวอย่าง: E-commerce Product SEO

```vue
<!-- pages/products/[id].vue - Complete Product SEO -->
<script setup lang="ts">
interface Product {
  id: number
  name: string
  description: string
  images: string[]
  price: number
  comparePrice?: number
  sku: string
  brand: string
  stock: number
  rating: { average: number; count: number }
  tags: string[]
  createdAt: string
  updatedAt: string
}

const route = useRoute()
const config = useRuntimeConfig()
const { data: product } = await useFetch<Product>(`/api/products/${route.params.id}`)

if (!product.value) {
  throw createError({ statusCode: 404, statusMessage: 'ไม่พบสินค้า' })
}

const productUrl = `${config.public.siteUrl}/products/${route.params.id}`

useSeoMeta({
  title: `${product.value.name} | My Shop`,
  description: product.value.description.slice(0, 160),
  keywords: [product.value.brand, ...product.value.tags].join(', '),

  ogTitle: product.value.name,
  ogDescription: product.value.description.slice(0, 300),
  ogImage: product.value.images[0],
  ogUrl: productUrl,
  ogType: 'product',

  twitterCard: 'summary_large_image',
  twitterTitle: product.value.name,
  twitterDescription: product.value.description.slice(0, 200),
  twitterImage: product.value.images[0]
})

useHead({
  link: [{ rel: 'canonical', href: productUrl }],
  script: [{
    type: 'application/ld+json',
    innerHTML: computed(() => JSON.stringify({
      '@context': 'https://schema.org',
      '@type': 'Product',
      name: product.value?.name,
      description: product.value?.description,
      image: product.value?.images,
      sku: product.value?.sku,
      brand: { '@type': 'Brand', name: product.value?.brand },
      offers: {
        '@type': 'Offer',
        url: productUrl,
        price: product.value?.price,
        priceCurrency: 'THB',
        availability: (product.value?.stock ?? 0) > 0
          ? 'https://schema.org/InStock'
          : 'https://schema.org/OutOfStock',
        priceValidUntil: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
          .toISOString().split('T')[0]
      },
      aggregateRating: {
        '@type': 'AggregateRating',
        ratingValue: product.value?.rating.average,
        reviewCount: product.value?.rating.count,
        bestRating: 5,
        worstRating: 1
      }
    }))
  }]
})
</script>

<template>
  <div class="product-page">
    <h1 itemprop="name">{{ product?.name }}</h1>
    <div class="price">
      <span class="current-price">฿{{ product?.price.toLocaleString() }}</span>
      <span v-if="product?.comparePrice" class="compare-price">
        ฿{{ product.comparePrice.toLocaleString() }}
      </span>
    </div>
    <p itemprop="description">{{ product?.description }}</p>
  </div>
</template>
```

---

## สรุป

SEO ใน Nuxt.js มีเครื่องมือครบครัน:

1. **Global Config** - ตั้งค่า default SEO ใน `nuxt.config.ts`
2. **useHead()** - จัดการ `<head>` tags อย่างยืดหยุ่น
3. **useSeoMeta()** - Type-safe meta tags สำหรับ SEO/Social
4. **Open Graph** - meta tags สำหรับการแชร์บน Facebook/Line
5. **Twitter Cards** - meta tags สำหรับการแชร์บน Twitter/X
6. **JSON-LD** - Structured Data สำหรับ Rich Snippets ใน Google
7. **Sitemap** - ช่วย Search Engine ค้นหา pages ได้ง่ายขึ้น
8. **Robots.txt** - ควบคุม crawling behavior
