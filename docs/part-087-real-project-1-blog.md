# Part 87: โปรเจกต์จริง 1 - Full-stack Blog

## Overview

ในบทนี้เราจะสร้าง Full-stack Blog ที่สมบูรณ์ด้วย Nuxt 3 + Prisma + PostgreSQL โดยมีระบบ:
- User Authentication
- Posts, Categories, Tags
- Comments
- Admin Panel
- SEO Optimization
- Deploy to Production

---

## 1. Requirements Analysis

### User Stories

```
ผู้ใช้ทั่วไป:
- อ่าน blog posts
- ค้นหาบทความ
- Filter ตาม category/tag
- แสดงความคิดเห็น (ต้อง login)

Author:
- เขียนและแก้ไข posts
- Upload images
- Manage comments ของตัวเอง

Admin:
- Manage users ทั้งหมด
- Manage posts ทั้งหมด
- Manage categories/tags
- View analytics
```

---

## 2. Database Schema Design

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  username  String   @unique
  password  String
  bio       String?
  avatar    String?
  role      Role     @default(READER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  posts    Post[]
  comments Comment[]
  sessions Session[]

  @@map("users")
}

enum Role {
  READER
  AUTHOR
  ADMIN
}

model Post {
  id          String    @id @default(cuid())
  title       String
  slug        String    @unique
  excerpt     String?
  content     String    @db.Text
  coverImage  String?
  status      PostStatus @default(DRAFT)
  publishedAt DateTime?
  metaTitle   String?
  metaDesc    String?
  
  authorId    String
  author      User      @relation(fields: [authorId], references: [id])
  
  categories  Category[]
  tags        Tag[]
  comments    Comment[]
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@map("posts")
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

model Category {
  id          String  @id @default(cuid())
  name        String  @unique
  slug        String  @unique
  description String?
  posts       Post[]

  @@map("categories")
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  slug  String @unique
  posts Post[]

  @@map("tags")
}

model Comment {
  id        String   @id @default(cuid())
  content   String
  approved  Boolean  @default(false)
  
  postId    String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
  
  parentId  String?
  parent    Comment? @relation("CommentReplies", fields: [parentId], references: [id])
  replies   Comment[] @relation("CommentReplies")
  
  createdAt DateTime @default(now())

  @@map("comments")
}

model Session {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  token     String   @unique
  expiresAt DateTime
  createdAt DateTime @default(now())

  @@map("sessions")
}
```

---

## 3. Nuxt Configuration

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@nuxtjs/prisma',
    '@pinia/nuxt',
    '@nuxtjs/tailwindcss',
    '@nuxtjs/seo',
    '@nuxt/image',
    'nuxt-auth-utils'
  ],

  runtimeConfig: {
    jwtSecret: process.env.JWT_SECRET,
    public: {
      appName: 'My Blog',
      appUrl: process.env.APP_URL || 'http://localhost:3000'
    }
  },

  nitro: {
    routeRules: {
      '/': { prerender: true },
      '/blog/**': { swr: 3600 },
      '/api/**': { cors: true }
    }
  },

  seo: {
    baseUrl: process.env.APP_URL
  }
})
```

---

## 4. Authentication System

```typescript
// server/api/auth/register.post.ts
import bcrypt from 'bcryptjs'
import { z } from 'zod'

const RegisterSchema = z.object({
  name: z.string().min(2),
  username: z.string().min(3).regex(/^[a-zA-Z0-9_]+$/),
  email: z.string().email(),
  password: z.string().min(8)
})

export default defineEventHandler(async (event) => {
  const body = await readValidatedBody(event, RegisterSchema.parse)

  const existingUser = await prisma.user.findFirst({
    where: {
      OR: [
        { email: body.email },
        { username: body.username }
      ]
    }
  })

  if (existingUser) {
    throw createError({
      statusCode: 409,
      message: existingUser.email === body.email
        ? 'Email already registered'
        : 'Username already taken'
    })
  }

  const hashedPassword = await bcrypt.hash(body.password, 12)

  const user = await prisma.user.create({
    data: {
      name: body.name,
      username: body.username,
      email: body.email,
      password: hashedPassword
    },
    select: {
      id: true,
      name: true,
      username: true,
      email: true,
      role: true
    }
  })

  // Create session
  const session = await createSession(user.id)

  setCookie(event, 'auth_token', session.token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 30 * 24 * 60 * 60 // 30 days
  })

  return { user }
})

async function createSession(userId: string) {
  const token = generateSecureToken()
  const expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)

  return prisma.session.create({
    data: { userId, token, expiresAt }
  })
}
```

---

## 5. Blog Posts API

```typescript
// server/api/posts/index.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const { page = 1, limit = 10, category, tag, search, status } = query

  const where: any = { status: 'PUBLISHED' }

  if (category) where.categories = { some: { slug: String(category) } }
  if (tag) where.tags = { some: { slug: String(tag) } }
  if (search) {
    where.OR = [
      { title: { contains: String(search), mode: 'insensitive' } },
      { excerpt: { contains: String(search), mode: 'insensitive' } }
    ]
  }

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      where,
      select: {
        id: true,
        title: true,
        slug: true,
        excerpt: true,
        coverImage: true,
        publishedAt: true,
        author: {
          select: { name: true, username: true, avatar: true }
        },
        categories: {
          select: { name: true, slug: true }
        },
        tags: {
          select: { name: true, slug: true }
        },
        _count: { select: { comments: true } }
      },
      orderBy: { publishedAt: 'desc' },
      skip: (Number(page) - 1) * Number(limit),
      take: Number(limit)
    }),
    prisma.post.count({ where })
  ])

  return {
    posts,
    meta: {
      total,
      page: Number(page),
      limit: Number(limit),
      totalPages: Math.ceil(total / Number(limit))
    }
  }
})

// server/api/posts/[slug].get.ts
export default defineEventHandler(async (event) => {
  const slug = getRouterParam(event, 'slug')!

  const post = await prisma.post.findUnique({
    where: { slug, status: 'PUBLISHED' },
    include: {
      author: {
        select: { name: true, username: true, avatar: true, bio: true }
      },
      categories: true,
      tags: true,
      comments: {
        where: { approved: true, parentId: null },
        include: {
          author: { select: { name: true, username: true, avatar: true } },
          replies: {
            where: { approved: true },
            include: {
              author: { select: { name: true, username: true, avatar: true } }
            }
          }
        },
        orderBy: { createdAt: 'desc' }
      }
    }
  })

  if (!post) {
    throw createError({ statusCode: 404, message: 'Post not found' })
  }

  return post
})

// server/api/posts/index.post.ts (Admin/Author only)
export default defineEventHandler(async (event) => {
  await requireAuth(event)
  await requireRole(event, ['AUTHOR', 'ADMIN'])

  const body = await readBody(event)

  const { title, content, excerpt, categories, tags, status, coverImage } = body

  // Generate slug จาก title
  const baseSlug = title.toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')
    .replace(/\s+/g, '-')

  // ตรวจสอบว่า slug ซ้ำไหม
  const existingSlug = await prisma.post.findUnique({ where: { slug: baseSlug } })
  const slug = existingSlug
    ? `${baseSlug}-${Date.now()}`
    : baseSlug

  const post = await prisma.post.create({
    data: {
      title,
      slug,
      content,
      excerpt,
      coverImage,
      status: status || 'DRAFT',
      publishedAt: status === 'PUBLISHED' ? new Date() : null,
      metaTitle: title,
      metaDesc: excerpt,
      authorId: event.context.userId,
      categories: {
        connect: categories?.map((id: string) => ({ id })) || []
      },
      tags: {
        connectOrCreate: tags?.map((tag: string) => ({
          where: { name: tag },
          create: {
            name: tag,
            slug: tag.toLowerCase().replace(/\s+/g, '-')
          }
        })) || []
      }
    },
    include: {
      author: true,
      categories: true,
      tags: true
    }
  })

  return post
})
```

---

## 6. Frontend Implementation

### Blog Home Page

```vue
<!-- pages/index.vue -->
<template>
  <div>
    <!-- Hero Section -->
    <section class="hero">
      <h1>Welcome to My Blog</h1>
      <p>Sharing knowledge about Vue.js, Nuxt, and modern web development</p>
    </section>

    <!-- Featured Post -->
    <section v-if="featuredPost" class="featured">
      <h2>Featured Post</h2>
      <PostCard :post="featuredPost" featured />
    </section>

    <!-- Category Filter -->
    <div class="category-filter">
      <button
        v-for="category in categories"
        :key="category.slug"
        :class="['category-btn', { active: selectedCategory === category.slug }]"
        @click="selectCategory(category.slug)"
      >
        {{ category.name }}
      </button>
    </div>

    <!-- Post Grid -->
    <div class="posts-grid">
      <PostCard
        v-for="post in posts"
        :key="post.id"
        :post="post"
      />
    </div>

    <!-- Pagination -->
    <Pagination
      :current-page="page"
      :total-pages="totalPages"
      @change="changePage"
    />
  </div>
</template>

<script setup lang="ts">
const route = useRoute()
const router = useRouter()

const page = computed(() => Number(route.query.page) || 1)
const selectedCategory = computed(() => route.query.category as string || '')

const { data, pending } = await useFetch('/api/posts', {
  query: computed(() => ({
    page: page.value,
    limit: 9,
    category: selectedCategory.value || undefined
  }))
})

const posts = computed(() => data.value?.posts || [])
const featuredPost = computed(() => posts.value[0])
const totalPages = computed(() => data.value?.meta?.totalPages || 1)

const { data: categories } = await useFetch('/api/categories')

function selectCategory(slug: string) {
  router.push({ query: { category: slug === selectedCategory.value ? undefined : slug } })
}

function changePage(newPage: number) {
  router.push({ query: { ...route.query, page: newPage } })
}

// SEO
useHead({
  title: 'My Blog - Vue.js & Web Development',
  meta: [
    { name: 'description', content: 'Articles about Vue.js, Nuxt.js, and modern web development' }
  ]
})
</script>
```

### Blog Post Page

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <article class="blog-post" v-if="post">
    <!-- Cover Image -->
    <NuxtImg
      v-if="post.coverImage"
      :src="post.coverImage"
      :alt="post.title"
      class="cover-image"
      width="1200"
      height="630"
    />

    <!-- Post Header -->
    <header>
      <div class="categories">
        <NuxtLink
          v-for="cat in post.categories"
          :key="cat.slug"
          :to="`/?category=${cat.slug}`"
          class="category-badge"
        >
          {{ cat.name }}
        </NuxtLink>
      </div>

      <h1>{{ post.title }}</h1>

      <div class="post-meta">
        <AuthorCard :author="post.author" />
        <time :datetime="post.publishedAt">
          {{ formatDate(post.publishedAt) }}
        </time>
        <span>{{ readingTime(post.content) }} min read</span>
      </div>
    </header>

    <!-- Content -->
    <div class="post-content prose" v-html="renderedContent" />

    <!-- Tags -->
    <div class="tags">
      <NuxtLink
        v-for="tag in post.tags"
        :key="tag.slug"
        :to="`/?tag=${tag.slug}`"
        class="tag"
      >
        #{{ tag.name }}
      </NuxtLink>
    </div>

    <!-- Share Buttons -->
    <ShareButtons :title="post.title" />

    <!-- Comments Section -->
    <CommentsSection :post-id="post.id" :comments="post.comments" />
  </article>
</template>

<script setup lang="ts">
import { marked } from 'marked'

const route = useRoute()
const { data: post, error } = await useFetch(`/api/posts/${route.params.slug}`)

if (error.value) {
  throw createError({ statusCode: 404, message: 'Post not found' })
}

const renderedContent = computed(() =>
  post.value ? marked(post.value.content) : ''
)

function readingTime(content: string) {
  const wordsPerMinute = 200
  const wordCount = content.split(/\s+/).length
  return Math.ceil(wordCount / wordsPerMinute)
}

// SEO
useSeoMeta({
  title: () => post.value?.metaTitle || post.value?.title,
  description: () => post.value?.metaDesc || post.value?.excerpt,
  ogImage: () => post.value?.coverImage,
  ogType: 'article',
  articlePublishedTime: () => post.value?.publishedAt,
  articleAuthor: () => post.value?.author.name
})
</script>
```

---

## 7. Admin Panel

```vue
<!-- pages/admin/posts/index.vue -->
<template>
  <div class="admin-posts">
    <div class="page-header">
      <h1>Posts</h1>
      <NuxtLink to="/admin/posts/new" class="btn-primary">
        New Post
      </NuxtLink>
    </div>

    <div class="filters">
      <select v-model="statusFilter">
        <option value="">All</option>
        <option value="DRAFT">Draft</option>
        <option value="PUBLISHED">Published</option>
        <option value="ARCHIVED">Archived</option>
      </select>
      <input v-model="searchQuery" placeholder="Search posts..." />
    </div>

    <DataTable
      :columns="columns"
      :rows="posts"
      :loading="pending"
      @row-click="editPost"
    >
      <template #status="{ value }">
        <Badge :variant="getStatusVariant(value)">{{ value }}</Badge>
      </template>
      <template #actions="{ row }">
        <button @click.stop="publishPost(row.id)" v-if="row.status === 'DRAFT'">
          Publish
        </button>
        <button @click.stop="deletePost(row.id)" class="btn-danger">
          Delete
        </button>
      </template>
    </DataTable>
  </div>
</template>

<script setup lang="ts">
definePageMeta({ middleware: 'admin' })

const statusFilter = ref('')
const searchQuery = ref('')

const { data, pending, refresh } = useFetch('/api/admin/posts', {
  query: computed(() => ({
    status: statusFilter.value || undefined,
    search: searchQuery.value || undefined
  }))
})

const posts = computed(() => data.value?.posts || [])

const columns = [
  { key: 'title', label: 'Title', sortable: true },
  { key: 'author', label: 'Author' },
  { key: 'status', label: 'Status' },
  { key: 'publishedAt', label: 'Published', sortable: true },
  { key: '_count.comments', label: 'Comments' },
  { key: 'actions', label: '' }
]

async function publishPost(id: string) {
  await $fetch(`/api/admin/posts/${id}/publish`, { method: 'POST' })
  refresh()
}

async function deletePost(id: string) {
  if (!confirm('Delete this post?')) return
  await $fetch(`/api/admin/posts/${id}`, { method: 'DELETE' })
  refresh()
}
</script>
```

---

## 8. SEO Optimization

```typescript
// composables/useBlogSeo.ts
export function useBlogSeo(post: Ref<Post | null>) {
  const config = useRuntimeConfig()

  useSeoMeta({
    title: () => post.value?.metaTitle || post.value?.title,
    description: () => post.value?.metaDesc || post.value?.excerpt || '',
    ogTitle: () => post.value?.title,
    ogDescription: () => post.value?.excerpt || '',
    ogImage: () => post.value?.coverImage || `${config.public.appUrl}/og-default.jpg`,
    ogType: 'article',
    twitterCard: 'summary_large_image',
    twitterTitle: () => post.value?.title,
    twitterDescription: () => post.value?.excerpt || '',
    articlePublishedTime: () => post.value?.publishedAt?.toString(),
    articleAuthor: () => [post.value?.author.name || ''],
    articleTag: () => post.value?.tags.map(t => t.name) || []
  })

  useHead({
    link: [
      {
        rel: 'canonical',
        href: () => `${config.public.appUrl}/blog/${post.value?.slug}`
      }
    ],
    script: [
      {
        type: 'application/ld+json',
        innerHTML: () => JSON.stringify({
          '@context': 'https://schema.org',
          '@type': 'BlogPosting',
          headline: post.value?.title,
          description: post.value?.excerpt,
          image: post.value?.coverImage,
          author: {
            '@type': 'Person',
            name: post.value?.author.name
          },
          datePublished: post.value?.publishedAt
        })
      }
    ]
  })
}
```

---

## 9. Deploy

### Deployment Steps

```yaml
# .github/workflows/deploy.yml
name: Deploy Blog

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          JWT_SECRET: ${{ secrets.JWT_SECRET }}
          
      # Deploy to Railway/Vercel/etc
      - name: Deploy to Railway
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
        run: npx railway up
```

---

## สรุป

Blog project นี้ครอบคลุม:
- Full-stack development ด้วย Nuxt 3
- Database design ด้วย Prisma
- Authentication system
- CRUD operations
- Admin panel
- SEO optimization
- Production deployment

ใช้เป็น portfolio project ที่แสดงความสามารถได้ครบถ้วน
