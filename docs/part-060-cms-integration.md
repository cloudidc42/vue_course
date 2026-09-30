# Part 60: CMS Integration กับ Nuxt.js

## Headless CMS แนวคิด

Headless CMS คือ CMS ที่แยก backend (การจัดการเนื้อหา) ออกจาก frontend (การแสดงผล) โดย CMS จะ expose เนื้อหาผ่าน API ทำให้ developer สามารถใช้ framework ใดก็ได้สำหรับ frontend

```
┌──────────────────┐         ┌──────────────────┐
│   Content Editor │         │   Nuxt Frontend  │
│   (Strapi/       │  REST/  │                  │
│    Contentful/   │ GraphQL │  - Blog          │
│    Sanity)       │ ──────► │  - Landing Pages │
│                  │         │  - E-commerce    │
└──────────────────┘         └──────────────────┘
```

### ข้อดีของ Headless CMS
- ยืดหยุ่นในการใช้ frontend framework
- Content editors ใช้ UI ที่คุ้นเคย
- Multi-channel delivery (web, mobile, etc.)
- Better performance ด้วย SSG/ISR

## Strapi กับ Nuxt

Strapi คือ open-source headless CMS ที่ใช้งานง่าย

### การติดตั้ง Strapi

```bash
# สร้าง Strapi project
npx create-strapi-app@latest my-cms --quickstart

# หรือด้วย TypeScript
npx create-strapi-app@latest my-cms --quickstart --typescript
```

### Content Types ใน Strapi

```typescript
// strapi/src/api/article/content-types/article/schema.json
{
  "kind": "collectionType",
  "collectionName": "articles",
  "info": {
    "singularName": "article",
    "pluralName": "articles",
    "displayName": "Article"
  },
  "attributes": {
    "title": {
      "type": "string",
      "required": true
    },
    "slug": {
      "type": "uid",
      "targetField": "title",
      "required": true
    },
    "content": {
      "type": "richtext"
    },
    "excerpt": {
      "type": "text",
      "maxLength": 500
    },
    "cover": {
      "type": "media",
      "multiple": false,
      "allowedTypes": ["images"]
    },
    "categories": {
      "type": "relation",
      "relation": "manyToMany",
      "target": "api::category.category"
    },
    "author": {
      "type": "relation",
      "relation": "manyToOne",
      "target": "plugin::users-permissions.user"
    },
    "publishedAt": {
      "type": "datetime"
    },
    "seo": {
      "type": "component",
      "component": "shared.seo",
      "repeatable": false
    }
  }
}
```

### Nuxt Module สำหรับ Strapi

```bash
npm install @nuxtjs/strapi
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/strapi'],
  
  strapi: {
    url: process.env.STRAPI_URL || 'http://localhost:1337',
    prefix: '/api',
    version: 'v4',
    cookie: {},
    cookieName: 'strapi_jwt'
  }
})
```

```vue
<!-- pages/blog/index.vue -->
<template>
  <div class="blog-list">
    <h1>บทความ</h1>
    
    <div class="articles-grid">
      <ArticleCard
        v-for="article in articles"
        :key="article.id"
        :article="article"
      />
    </div>
    
    <Pagination
      :current="page"
      :total="pageCount"
      @change="page = $event"
    />
  </div>
</template>

<script setup lang="ts">
const { find } = useStrapi()
const page = ref(1)
const pageSize = 12

const { data } = await useAsyncData(
  `articles-${page.value}`,
  () => find('articles', {
    populate: ['cover', 'categories', 'author.avatar'],
    sort: ['publishedAt:desc'],
    pagination: {
      page: page.value,
      pageSize
    },
    filters: {
      publishedAt: { $notNull: true }
    }
  })
)

const articles = computed(() => data.value?.data || [])
const pageCount = computed(() => data.value?.meta?.pagination?.pageCount || 0)

watch(page, () => {
  refreshNuxtData(`articles-${page.value}`)
})
</script>
```

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <article class="article-detail" v-if="article">
    <header class="article-header">
      <div class="article-meta">
        <span v-for="cat in article.categories" :key="cat.id" class="category-tag">
          {{ cat.name }}
        </span>
      </div>
      
      <h1>{{ article.title }}</h1>
      
      <div class="article-info">
        <img
          v-if="article.author?.avatar"
          :src="strapiImage(article.author.avatar.url)"
          :alt="article.author.username"
          class="author-avatar"
        />
        <span>{{ article.author?.username }}</span>
        <time :datetime="article.publishedAt">
          {{ formatDate(article.publishedAt) }}
        </time>
      </div>
    </header>
    
    <NuxtImg
      v-if="article.cover"
      :src="strapiImage(article.cover.url)"
      :alt="article.cover.alternativeText || article.title"
      class="article-cover"
      width="800"
      height="400"
    />
    
    <div 
      class="article-content"
      v-html="renderMarkdown(article.content)"
    ></div>
    
    <!-- Comments section -->
    <ArticleComments :article-id="article.id" />
  </article>
</template>

<script setup lang="ts">
const route = useRoute()
const { findOne } = useStrapi()
const config = useRuntimeConfig()

const { data: article } = await useAsyncData(
  `article-${route.params.slug}`,
  () => findOne('articles', route.params.slug as string, {
    populate: ['cover', 'categories', 'author.avatar', 'seo'],
    filters: {
      slug: { $eq: route.params.slug }
    }
  }).then(res => res.data[0])
)

if (!article.value) {
  throw createError({ statusCode: 404, message: 'Article not found' })
}

const strapiImage = (url: string) => {
  if (url.startsWith('http')) return url
  return `${config.public.strapiUrl}${url}`
}

const formatDate = (dateStr: string) => new Date(dateStr).toLocaleDateString('th-TH', {
  year: 'numeric',
  month: 'long',
  day: 'numeric'
})

// SEO
useSeoMeta({
  title: article.value?.seo?.metaTitle || article.value?.title,
  description: article.value?.seo?.metaDescription || article.value?.excerpt,
  ogImage: article.value?.cover?.url 
    ? strapiImage(article.value.cover.url) 
    : undefined
})

// JSON-LD
useSchemaOrg([
  defineArticle({
    headline: article.value?.title,
    datePublished: article.value?.publishedAt,
    dateModified: article.value?.updatedAt,
    author: [{ name: article.value?.author?.username }]
  })
])
</script>
```

## Contentful Integration

```typescript
// plugins/contentful.ts
import { createClient } from 'contentful'

export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()
  
  const client = createClient({
    space: config.contentfulSpaceId,
    accessToken: config.contentfulAccessToken,
    environment: config.contentfulEnvironment || 'master'
  })
  
  const previewClient = createClient({
    space: config.contentfulSpaceId,
    accessToken: config.contentfulPreviewToken,
    host: 'preview.contentful.com',
    environment: config.contentfulEnvironment || 'master'
  })
  
  return {
    provide: {
      contentful: client,
      contentfulPreview: previewClient
    }
  }
})
```

```typescript
// composables/useContentful.ts
export function useContentful() {
  const { $contentful, $contentfulPreview } = useNuxtApp()
  const route = useRoute()
  
  const isPreview = computed(() => route.query.preview === 'true')
  const client = computed(() => isPreview.value ? $contentfulPreview : $contentful)

  const getEntries = async <T>(
    contentType: string,
    params: Record<string, any> = {}
  ) => {
    const entries = await client.value.getEntries<T>({
      content_type: contentType,
      ...params
    })
    return entries
  }

  const getEntry = async <T>(id: string) => {
    return client.value.getEntry<T>(id)
  }

  const getEntriesBySlug = async <T>(
    contentType: string,
    slug: string
  ) => {
    const entries = await client.value.getEntries<T>({
      content_type: contentType,
      'fields.slug': slug,
      limit: 1,
      include: 3
    })
    return entries.items[0] || null
  }

  return { getEntries, getEntry, getEntriesBySlug }
}
```

```vue
<!-- pages/blog/[slug].vue (Contentful version) -->
<script setup lang="ts">
const route = useRoute()
const { getEntriesBySlug } = useContentful()

interface ArticleFields {
  title: string
  slug: string
  content: Document
  excerpt: string
  coverImage: Asset
  author: Entry<AuthorFields>
  publishDate: string
  seo: Entry<SeoFields>
}

const { data: article } = await useAsyncData(
  `article-${route.params.slug}`,
  () => getEntriesBySlug<ArticleFields>('blogPost', route.params.slug as string)
)

if (!article.value) {
  throw createError({ statusCode: 404 })
}
</script>
```

## Sanity Integration

```bash
npm install @sanity/nuxt @sanity/client
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@sanity/nuxt'],
  
  sanity: {
    projectId: process.env.SANITY_PROJECT_ID,
    dataset: process.env.SANITY_DATASET || 'production',
    apiVersion: '2024-01-01',
    useCdn: process.env.NODE_ENV === 'production'
  }
})
```

```typescript
// composables/useSanity.ts
export function useSanityQueries() {
  const sanity = useSanityClient()
  
  const getArticles = (page = 1, perPage = 10) => {
    const start = (page - 1) * perPage
    const end = start + perPage
    
    return sanity.fetch(`
      *[_type == "article" && defined(slug.current)] | order(publishedAt desc) [$start...$end] {
        _id,
        title,
        "slug": slug.current,
        excerpt,
        publishedAt,
        "cover": cover.asset->url,
        "author": author->{name, "avatar": image.asset->url},
        "categories": categories[]->{_id, title, "slug": slug.current}
      }
    `, { start, end })
  }

  const getArticle = (slug: string) => {
    return sanity.fetch(`
      *[_type == "article" && slug.current == $slug][0] {
        _id,
        title,
        "slug": slug.current,
        body,
        publishedAt,
        "cover": cover.asset->url,
        "author": author->{name, bio, "avatar": image.asset->url},
        "categories": categories[]->{_id, title},
        seo {
          title,
          description,
          "image": image.asset->url
        }
      }
    `, { slug })
  }

  const getRelatedArticles = (articleId: string, categoryIds: string[]) => {
    return sanity.fetch(`
      *[
        _type == "article" && 
        _id != $articleId && 
        count((categories[]->_id)[@ in $categoryIds]) > 0
      ] | order(publishedAt desc)[0...3] {
        _id,
        title,
        "slug": slug.current,
        "cover": cover.asset->url,
        publishedAt
      }
    `, { articleId, categoryIds })
  }

  return { getArticles, getArticle, getRelatedArticles }
}
```

## Preview Mode

Preview mode ช่วยให้ content editors เห็น draft content ก่อน publish

```typescript
// server/api/preview.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const { secret, slug, redirect } = query

  // Validate secret
  if (secret !== process.env.PREVIEW_SECRET) {
    throw createError({ statusCode: 401, message: 'Invalid preview secret' })
  }

  if (!slug) {
    throw createError({ statusCode: 400, message: 'Slug is required' })
  }

  // Set preview cookie
  setCookie(event, '__preview', 'true', {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 3600 // 1 hour
  })

  // Redirect to content
  return sendRedirect(event, `${redirect || '/blog'}/${slug}?preview=true`)
})
```

```typescript
// server/api/exit-preview.get.ts
export default defineEventHandler(async (event) => {
  deleteCookie(event, '__preview')
  return sendRedirect(event, '/')
})
```

```typescript
// middleware/preview.ts
export default defineNuxtRouteMiddleware((to) => {
  const cookie = useCookie('__preview')
  
  if (cookie.value && to.query.preview !== 'true') {
    return navigateTo({ ...to, query: { ...to.query, preview: 'true' } })
  }
})
```

## Webhook for ISR

```typescript
// server/api/webhooks/strapi.post.ts
import { createHmac } from 'crypto'

export default defineEventHandler(async (event) => {
  const signature = getHeader(event, 'x-strapi-signature')
  const body = await readRawBody(event)
  
  if (!body) {
    throw createError({ statusCode: 400 })
  }
  
  // Verify webhook signature
  const expectedSignature = createHmac('sha256', process.env.STRAPI_WEBHOOK_SECRET!)
    .update(body)
    .digest('hex')
  
  if (signature !== expectedSignature) {
    throw createError({ statusCode: 401, message: 'Invalid signature' })
  }
  
  const payload = JSON.parse(body)
  const { event: eventType, model, entry } = payload
  
  // Revalidate affected pages
  if (model === 'article') {
    const paths: string[] = []
    
    if (entry?.slug) {
      paths.push(`/blog/${entry.slug}`)
    }
    paths.push('/blog')
    
    // Purge CDN cache or revalidate ISR
    await Promise.all(paths.map(path => purgePath(path)))
    
    console.log(`Revalidated paths: ${paths.join(', ')}`)
  }
  
  return { revalidated: true }
})

async function purgePath(path: string) {
  // Implement based on your CDN/deployment
  // For Vercel:
  await $fetch(`https://api.vercel.com/v1/deployments/${process.env.VERCEL_DEPLOYMENT_ID}/cache`, {
    method: 'DELETE',
    body: { path }
  })
}
```

## ตัวอย่าง: Nuxt + Strapi Blog

```typescript
// server/api/blog/articles.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const { page = 1, pageSize = 10, category, search } = query
  
  const strapiUrl = process.env.STRAPI_URL
  
  const params = new URLSearchParams({
    'populate[0]': 'cover',
    'populate[1]': 'categories',
    'populate[2]': 'author.avatar',
    'sort[0]': 'publishedAt:desc',
    'pagination[page]': String(page),
    'pagination[pageSize]': String(pageSize)
  })
  
  if (category) {
    params.set('filters[categories][slug][$eq]', String(category))
  }
  
  if (search) {
    params.set('filters[$or][0][title][$containsi]', String(search))
    params.set('filters[$or][1][content][$containsi]', String(search))
  }
  
  const data = await $fetch(`${strapiUrl}/api/articles?${params}`, {
    headers: {
      Authorization: `Bearer ${process.env.STRAPI_API_TOKEN}`
    }
  })
  
  return data
})
```

```vue
<!-- components/blog/ArticleCard.vue -->
<template>
  <article class="article-card">
    <NuxtLink :to="`/blog/${article.slug}`">
      <div class="card-image">
        <NuxtImg
          v-if="article.cover"
          :src="imageUrl"
          :alt="article.cover.alternativeText || article.title"
          width="400"
          height="250"
          loading="lazy"
        />
        <div v-else class="card-image-placeholder">
          <span>📝</span>
        </div>
      </div>
      
      <div class="card-body">
        <div class="card-categories">
          <span
            v-for="cat in article.categories"
            :key="cat.id"
            class="category-badge"
          >
            {{ cat.name }}
          </span>
        </div>
        
        <h2 class="card-title">{{ article.title }}</h2>
        <p class="card-excerpt">{{ article.excerpt }}</p>
        
        <footer class="card-footer">
          <div class="author" v-if="article.author">
            <img
              v-if="article.author.avatar"
              :src="authorAvatarUrl"
              :alt="article.author.username"
              class="author-avatar"
            />
            <span>{{ article.author.username }}</span>
          </div>
          <time :datetime="article.publishedAt">
            {{ formatDate(article.publishedAt) }}
          </time>
        </footer>
      </div>
    </NuxtLink>
  </article>
</template>

<script setup lang="ts">
import { computed } from 'vue'

interface Article {
  id: number
  title: string
  slug: string
  excerpt: string
  publishedAt: string
  cover?: {
    url: string
    alternativeText?: string
  }
  author?: {
    username: string
    avatar?: { url: string }
  }
  categories: Array<{ id: number; name: string }>
}

const props = defineProps<{ article: Article }>()

const config = useRuntimeConfig()
const strapiUrl = config.public.strapiUrl

const imageUrl = computed(() => {
  if (!props.article.cover) return ''
  const url = props.article.cover.url
  return url.startsWith('http') ? url : `${strapiUrl}${url}`
})

const authorAvatarUrl = computed(() => {
  if (!props.article.author?.avatar) return ''
  const url = props.article.author.avatar.url
  return url.startsWith('http') ? url : `${strapiUrl}${url}`
})

const formatDate = (dateStr: string) =>
  new Date(dateStr).toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
</script>
```

## Rich Text Renderer

```vue
<!-- components/blog/RichTextRenderer.vue -->
<template>
  <div class="rich-text" v-html="renderedContent"></div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { marked } from 'marked'
import DOMPurify from 'isomorphic-dompurify'

const props = defineProps<{
  content: string
  type?: 'markdown' | 'html'
}>()

const renderedContent = computed(() => {
  if (props.type === 'html') {
    return DOMPurify.sanitize(props.content)
  }
  // Markdown
  return DOMPurify.sanitize(marked(props.content) as string)
})
</script>
```

## สรุป

Headless CMS Integration ต้องการ:
1. เลือก CMS ที่เหมาะกับ use case (Strapi สำหรับ self-hosted, Contentful/Sanity สำหรับ cloud)
2. ตั้งค่า API integration ด้วย Nuxt modules
3. ใช้ Preview mode สำหรับ draft content
4. ตั้งค่า Webhooks สำหรับ ISR revalidation
5. Optimize images ผ่าน Nuxt Image module
