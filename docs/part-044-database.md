# Part 44: Database Integration ด้วย Prisma ORM กับ Nuxt.js

## Prisma คืออะไร?

Prisma เป็น Next-generation ORM (Object-Relational Mapping) สำหรับ Node.js และ TypeScript ที่ช่วยให้การทำงานกับฐานข้อมูลง่ายขึ้น มี type-safety และ auto-completion ใน IDE

## 1. Prisma ORM Setup กับ Nuxt

### ติดตั้ง Prisma

```bash
# ติดตั้ง Prisma
npm install prisma @prisma/client

# หรือถ้าใช้กับ Nuxt
npm install prisma @prisma/client
npx prisma init
```

### โครงสร้างโปรเจกต์

```
nuxt-app/
├── prisma/
│   ├── schema.prisma    # Database schema
│   ├── migrations/      # Migration files
│   └── seed.ts          # Database seeding
├── server/
│   ├── api/
│   │   ├── posts/
│   │   └── users/
│   └── utils/
│       └── prisma.ts    # Prisma client instance
├── .env
└── nuxt.config.ts
```

```bash
# .env
DATABASE_URL="postgresql://username:password@localhost:5432/blog_db?schema=public"
# หรือ SQLite สำหรับ development
DATABASE_URL="file:./dev.db"
```

```typescript
// server/utils/prisma.ts
import { PrismaClient } from '@prisma/client'

// ป้องกัน Prisma Client instances มากเกินไปใน development
declare global {
  var __prisma: PrismaClient | undefined
}

export const prisma = globalThis.__prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development'
    ? ['query', 'info', 'warn', 'error']
    : ['error']
})

if (process.env.NODE_ENV !== 'production') {
  globalThis.__prisma = prisma
}
```

## 2. Blog Database Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ============ USER ============
model User {
  id             String    @id @default(cuid())
  email          String    @unique
  name           String
  password       String?
  avatar         String?
  bio            String?
  role           Role      @default(viewer)
  provider       String?   // 'email', 'google', 'github'
  providerId     String?
  emailVerified  Boolean   @default(false)
  
  // Relations
  posts          Post[]
  comments       Comment[]
  likes          Like[]
  followers      Follow[]  @relation("Following")
  following      Follow[]  @relation("Follower")
  
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt
  
  @@index([email])
  @@index([provider, providerId])
}

enum Role {
  admin
  editor
  author
  viewer
}

// ============ POST ============
model Post {
  id          String      @id @default(cuid())
  title       String
  slug        String      @unique
  excerpt     String?
  content     String
  coverImage  String?
  published   Boolean     @default(false)
  publishedAt DateTime?
  viewCount   Int         @default(0)
  
  // Author
  authorId    String
  author      User        @relation(fields: [authorId], references: [id])
  
  // Category
  categoryId  String?
  category    Category?   @relation(fields: [categoryId], references: [id])
  
  // Relations
  tags        PostTag[]
  comments    Comment[]
  likes       Like[]
  
  // SEO
  metaTitle       String?
  metaDescription String?
  
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt
  
  @@index([authorId])
  @@index([categoryId])
  @@index([published, publishedAt])
  @@fulltext([title, content])
}

// ============ CATEGORY ============
model Category {
  id          String   @id @default(cuid())
  name        String   @unique
  slug        String   @unique
  description String?
  color       String?  // สีสำหรับ UI
  
  posts       Post[]
  
  createdAt   DateTime @default(now())
}

// ============ TAG ============
model Tag {
  id    String    @id @default(cuid())
  name  String    @unique
  slug  String    @unique
  
  posts PostTag[]
}

// Many-to-Many: Post <-> Tag
model PostTag {
  postId String
  tagId  String
  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag    Tag    @relation(fields: [tagId], references: [id], onDelete: Cascade)
  
  @@id([postId, tagId])
}

// ============ COMMENT ============
model Comment {
  id        String   @id @default(cuid())
  content   String
  
  // Author
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
  
  // Post
  postId    String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  
  // Parent comment (สำหรับ replies)
  parentId  String?
  parent    Comment? @relation("Replies", fields: [parentId], references: [id])
  replies   Comment[] @relation("Replies")
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([postId])
  @@index([authorId])
}

// ============ LIKE ============
model Like {
  id      String @id @default(cuid())
  userId  String
  postId  String
  user    User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  post    Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  
  createdAt DateTime @default(now())
  
  @@unique([userId, postId])
  @@index([postId])
}

// ============ FOLLOW ============
model Follow {
  followerId  String
  followingId String
  follower    User   @relation("Follower", fields: [followerId], references: [id])
  following   User   @relation("Following", fields: [followingId], references: [id])
  
  createdAt DateTime @default(now())
  
  @@id([followerId, followingId])
}

// ============ MEDIA ============
model Media {
  id        String   @id @default(cuid())
  filename  String
  url       String
  publicId  String?  // Cloudinary/S3 ID
  type      String   // 'image', 'video', 'document'
  size      Int
  width     Int?
  height    Int?
  
  uploadedBy String
  
  createdAt  DateTime @default(now())
  
  @@index([uploadedBy])
}
```

## 3. Database Migrations

```bash
# สร้าง migration ครั้งแรก
npx prisma migrate dev --name init

# สร้าง migration หลังแก้ไข schema
npx prisma migrate dev --name add_view_count_to_posts

# Apply migrations ใน production
npx prisma migrate deploy

# Reset database (ลบข้อมูลทั้งหมด)
npx prisma migrate reset

# ดู migration status
npx prisma migrate status

# Generate Prisma Client หลังแก้ไข schema
npx prisma generate
```

### ตัวอย่าง Migration File

```sql
-- prisma/migrations/20241001000000_init/migration.sql
-- CreateEnum
CREATE TYPE "Role" AS ENUM ('admin', 'editor', 'author', 'viewer');

-- CreateTable
CREATE TABLE "User" (
    "id" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "password" TEXT,
    "avatar" TEXT,
    "bio" TEXT,
    "role" "Role" NOT NULL DEFAULT 'viewer',
    "provider" TEXT,
    "providerId" TEXT,
    "emailVerified" BOOLEAN NOT NULL DEFAULT false,
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "User_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "User_email_key" ON "User"("email");
```

## 4. CRUD Operations ด้วย Prisma

### API สำหรับ Posts

```typescript
// server/api/posts/index.get.ts - ดึงรายการโพสต์
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  
  const page = Number(query.page) || 1
  const limit = Math.min(Number(query.limit) || 10, 100)
  const skip = (page - 1) * limit
  
  const where: any = {
    published: true
  }
  
  // Filter by category
  if (query.category) {
    where.category = { slug: query.category }
  }
  
  // Filter by tag
  if (query.tag) {
    where.tags = {
      some: { tag: { slug: query.tag } }
    }
  }
  
  // Search
  if (query.search) {
    where.OR = [
      { title: { contains: String(query.search), mode: 'insensitive' } },
      { excerpt: { contains: String(query.search), mode: 'insensitive' } }
    ]
  }
  
  const [posts, total] = await prisma.$transaction([
    prisma.post.findMany({
      where,
      skip,
      take: limit,
      orderBy: { publishedAt: 'desc' },
      include: {
        author: {
          select: { id: true, name: true, avatar: true }
        },
        category: {
          select: { id: true, name: true, slug: true, color: true }
        },
        tags: {
          include: { tag: { select: { id: true, name: true, slug: true } } }
        },
        _count: {
          select: { comments: true, likes: true }
        }
      }
    }),
    prisma.post.count({ where })
  ])
  
  return {
    posts,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
      hasNextPage: skip + limit < total,
      hasPrevPage: page > 1
    }
  }
})
```

```typescript
// server/api/posts/index.post.ts - สร้างโพสต์ใหม่
import { generateSlug } from '~/utils/slug'

export default defineEventHandler(async (event) => {
  const user = await requirePermission(event, 'posts:create')
  const body = await readBody(event)
  
  const { title, content, excerpt, categoryId, tagIds, coverImage, published } = body
  
  if (!title || !content) {
    throw createError({ statusCode: 400, message: 'ต้องกรอก title และ content' })
  }
  
  // สร้าง slug จาก title
  let slug = generateSlug(title)
  
  // ตรวจสอบว่า slug ซ้ำหรือไม่
  const existingPost = await prisma.post.findUnique({ where: { slug } })
  if (existingPost) {
    slug = `${slug}-${Date.now()}`
  }
  
  const post = await prisma.post.create({
    data: {
      title,
      slug,
      content,
      excerpt,
      coverImage,
      published: published || false,
      publishedAt: published ? new Date() : null,
      authorId: user.id,
      categoryId: categoryId || null,
      tags: tagIds ? {
        create: tagIds.map((tagId: string) => ({ tagId }))
      } : undefined
    },
    include: {
      author: { select: { id: true, name: true } },
      category: true,
      tags: { include: { tag: true } }
    }
  })
  
  setResponseStatus(event, 201)
  return post
})
```

```typescript
// server/api/posts/[id].put.ts - อัปเดตโพสต์
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)
  
  // ดึงโพสต์เพื่อตรวจสอบ ownership
  const post = await prisma.post.findUnique({ where: { id } })
  if (!post) throw createError({ statusCode: 404, message: 'ไม่พบโพสต์' })
  
  // ตรวจสอบสิทธิ์
  await requireOwnerOrPermission(event, post.authorId, 'posts:update')
  
  const { title, content, excerpt, categoryId, tagIds, coverImage, published } = body
  
  const updatedPost = await prisma.post.update({
    where: { id },
    data: {
      ...(title && { title }),
      ...(content && { content }),
      ...(excerpt !== undefined && { excerpt }),
      ...(categoryId !== undefined && { categoryId }),
      ...(coverImage !== undefined && { coverImage }),
      ...(published !== undefined && {
        published,
        publishedAt: published && !post.published ? new Date() : post.publishedAt
      }),
      ...(tagIds && {
        tags: {
          deleteMany: {},
          create: tagIds.map((tagId: string) => ({ tagId }))
        }
      })
    },
    include: {
      author: { select: { id: true, name: true } },
      category: true,
      tags: { include: { tag: true } }
    }
  })
  
  return updatedPost
})
```

## 5. Relations - One-to-Many และ Many-to-Many

```typescript
// server/api/posts/[slug]/comments/index.get.ts
export default defineEventHandler(async (event) => {
  const slug = getRouterParam(event, 'slug')
  const query = getQuery(event)
  
  const post = await prisma.post.findUnique({ where: { slug } })
  if (!post) throw createError({ statusCode: 404 })
  
  // ดึง top-level comments พร้อม replies
  const comments = await prisma.comment.findMany({
    where: {
      postId: post.id,
      parentId: null // เฉพาะ top-level
    },
    include: {
      author: {
        select: { id: true, name: true, avatar: true }
      },
      replies: {
        include: {
          author: {
            select: { id: true, name: true, avatar: true }
          }
        },
        orderBy: { createdAt: 'asc' }
      }
    },
    orderBy: { createdAt: 'desc' }
  })
  
  return { comments }
})
```

```typescript
// server/api/posts/[slug]/like.post.ts - Toggle Like
export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  const slug = getRouterParam(event, 'slug')
  
  const post = await prisma.post.findUnique({ where: { slug } })
  if (!post) throw createError({ statusCode: 404 })
  
  // ตรวจสอบว่า like แล้วหรือยัง
  const existingLike = await prisma.like.findUnique({
    where: {
      userId_postId: { userId: user.id, postId: post.id }
    }
  })
  
  if (existingLike) {
    // Unlike
    await prisma.like.delete({
      where: { userId_postId: { userId: user.id, postId: post.id } }
    })
    return { liked: false }
  } else {
    // Like
    await prisma.like.create({
      data: { userId: user.id, postId: post.id }
    })
    return { liked: true }
  }
})
```

## 6. Transactions

```typescript
// server/api/posts/[id]/publish.post.ts - Publish with Transaction
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  const user = await requirePermission(event, 'posts:publish')
  
  // ใช้ transaction เพื่อให้ operation ทั้งหมด atomic
  const result = await prisma.$transaction(async (tx) => {
    const post = await tx.post.findUnique({ where: { id } })
    if (!post) throw createError({ statusCode: 404 })
    if (post.published) throw createError({ statusCode: 400, message: 'โพสต์เผยแพร่แล้ว' })
    
    // อัปเดตสถานะโพสต์
    const publishedPost = await tx.post.update({
      where: { id },
      data: {
        published: true,
        publishedAt: new Date()
      }
    })
    
    // บันทึก log
    await tx.activityLog.create({
      data: {
        action: 'POST_PUBLISHED',
        userId: user.id,
        resourceId: id,
        resourceType: 'Post',
        metadata: { postTitle: post.title }
      }
    })
    
    // ส่ง notification ให้ followers (ถ้ามี)
    const author = await tx.user.findUnique({
      where: { id: post.authorId },
      include: { followers: true }
    })
    
    if (author?.followers.length > 0) {
      await tx.notification.createMany({
        data: author.followers.map(follow => ({
          userId: follow.followerId,
          type: 'NEW_POST',
          message: `${author.name} ได้เผยแพร่โพสต์ใหม่: ${post.title}`,
          link: `/posts/${post.slug}`
        }))
      })
    }
    
    return publishedPost
  })
  
  return result
})
```

```typescript
// server/api/users/[id]/transfer-posts.post.ts - Transfer posts ด้วย transaction
export default defineEventHandler(async (event) => {
  const fromId = getRouterParam(event, 'id')
  const { toId } = await readBody(event)
  
  await requirePermission(event, 'users:manage_roles')
  
  // Transaction: ย้ายโพสต์ทั้งหมดและลบ user เดิม
  const result = await prisma.$transaction([
    prisma.post.updateMany({
      where: { authorId: fromId },
      data: { authorId: toId }
    }),
    prisma.comment.updateMany({
      where: { authorId: fromId },
      data: { authorId: toId }
    }),
    prisma.user.delete({
      where: { id: fromId }
    })
  ])
  
  return {
    postsTransferred: result[0].count,
    commentsTransferred: result[1].count
  }
})
```

## 7. Database Seeding

```typescript
// prisma/seed.ts
import { PrismaClient, Role } from '@prisma/client'
import bcrypt from 'bcrypt'

const prisma = new PrismaClient()

async function main() {
  console.log('🌱 เริ่มการ seed ฐานข้อมูล...')
  
  // ============ สร้าง Users ============
  const adminPassword = await bcrypt.hash('admin123456', 12)
  
  const admin = await prisma.user.upsert({
    where: { email: 'admin@example.com' },
    update: {},
    create: {
      email: 'admin@example.com',
      name: 'Admin User',
      password: adminPassword,
      role: Role.admin,
      emailVerified: true,
      bio: 'ผู้ดูแลระบบ'
    }
  })
  
  const editor = await prisma.user.upsert({
    where: { email: 'editor@example.com' },
    update: {},
    create: {
      email: 'editor@example.com',
      name: 'Editor User',
      password: await bcrypt.hash('editor123', 12),
      role: Role.editor,
      emailVerified: true
    }
  })
  
  const author = await prisma.user.upsert({
    where: { email: 'author@example.com' },
    update: {},
    create: {
      email: 'author@example.com',
      name: 'Author User',
      password: await bcrypt.hash('author123', 12),
      role: Role.author,
      emailVerified: true
    }
  })
  
  console.log('✅ สร้าง Users สำเร็จ')
  
  // ============ สร้าง Categories ============
  const categories = await Promise.all([
    prisma.category.upsert({
      where: { slug: 'technology' },
      update: {},
      create: { name: 'เทคโนโลยี', slug: 'technology', color: '#2196F3' }
    }),
    prisma.category.upsert({
      where: { slug: 'programming' },
      update: {},
      create: { name: 'การเขียนโปรแกรม', slug: 'programming', color: '#4CAF50' }
    }),
    prisma.category.upsert({
      where: { slug: 'tutorial' },
      update: {},
      create: { name: 'Tutorial', slug: 'tutorial', color: '#FF9800' }
    })
  ])
  
  console.log('✅ สร้าง Categories สำเร็จ')
  
  // ============ สร้าง Tags ============
  const tags = await Promise.all([
    'vue', 'nuxt', 'react', 'typescript', 'prisma', 'postgresql'
  ].map(name =>
    prisma.tag.upsert({
      where: { slug: name },
      update: {},
      create: { name, slug: name }
    })
  ))
  
  console.log('✅ สร้าง Tags สำเร็จ')
  
  // ============ สร้าง Posts ============
  const samplePosts = [
    {
      title: 'เริ่มต้นกับ Vue.js 3',
      slug: 'getting-started-with-vuejs-3',
      excerpt: 'เรียนรู้พื้นฐาน Vue.js 3 ตั้งแต่ต้น',
      content: `# เริ่มต้นกับ Vue.js 3

Vue.js 3 เป็น framework ที่ทรงพลังสำหรับการสร้าง UI...

## การติดตั้ง

\`\`\`bash
npm create vue@latest my-app
\`\`\`

## Composition API

Vue 3 แนะนำ Composition API ที่ช่วยให้จัดการ logic ได้ดีขึ้น
      `,
      authorId: author.id,
      categoryId: categories[1].id,
      tagIds: [tags[0].id, tags[2].id],
      published: true
    },
    {
      title: 'Prisma ORM คู่มือฉบับสมบูรณ์',
      slug: 'prisma-orm-complete-guide',
      excerpt: 'ทุกสิ่งที่คุณต้องรู้เกี่ยวกับ Prisma',
      content: `# Prisma ORM

Prisma เป็น modern ORM ที่ทำให้การทำงานกับฐานข้อมูลง่ายขึ้น...`,
      authorId: editor.id,
      categoryId: categories[1].id,
      tagIds: [tags[4].id, tags[5].id],
      published: true
    }
  ]
  
  for (const postData of samplePosts) {
    const { tagIds, ...data } = postData
    await prisma.post.upsert({
      where: { slug: data.slug },
      update: {},
      create: {
        ...data,
        publishedAt: new Date(),
        tags: {
          create: tagIds.map(tagId => ({ tagId }))
        }
      }
    })
  }
  
  console.log('✅ สร้าง Posts สำเร็จ')
  console.log('🎉 Seed เสร็จสมบูรณ์!')
}

main()
  .catch(console.error)
  .finally(async () => {
    await prisma.$disconnect()
  })
```

```json
// package.json - เพิ่ม seed script
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

```bash
# รัน seed
npx prisma db seed
```

## 8. Environment Configuration

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // Server-side only (ปลอดภัย)
    databaseUrl: process.env.DATABASE_URL,
    
    // Client-side ด้วย (ระวัง!)
    public: {
      apiBase: '/api'
    }
  }
})
```

```bash
# .env.local (development)
DATABASE_URL="postgresql://postgres:password@localhost:5432/blog_dev"

# .env.production (production - ไม่ commit ขึ้น git!)
DATABASE_URL="postgresql://prod_user:prod_password@prod-host:5432/blog_prod"
```

## 9. ตัวอย่าง: Blog API พร้อม Pagination และ Search

```typescript
// server/api/blog/index.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  
  const page = Math.max(1, Number(query.page) || 1)
  const limit = Math.min(Math.max(1, Number(query.limit) || 12), 50)
  const skip = (page - 1) * limit
  
  const categorySlug = query.category as string | undefined
  const tagSlug = query.tag as string | undefined
  const search = query.search as string | undefined
  const authorId = query.author as string | undefined
  
  // Build where clause
  const where = {
    published: true,
    publishedAt: { lte: new Date() },
    ...(categorySlug && {
      category: { slug: categorySlug }
    }),
    ...(tagSlug && {
      tags: { some: { tag: { slug: tagSlug } } }
    }),
    ...(search && {
      OR: [
        { title: { contains: search, mode: 'insensitive' as const } },
        { excerpt: { contains: search, mode: 'insensitive' as const } },
        { content: { contains: search, mode: 'insensitive' as const } }
      ]
    }),
    ...(authorId && { authorId })
  }
  
  const [posts, total, categories, popularTags] = await prisma.$transaction([
    prisma.post.findMany({
      where,
      skip,
      take: limit,
      orderBy: [
        { publishedAt: 'desc' }
      ],
      select: {
        id: true,
        title: true,
        slug: true,
        excerpt: true,
        coverImage: true,
        publishedAt: true,
        viewCount: true,
        author: {
          select: { id: true, name: true, avatar: true }
        },
        category: {
          select: { id: true, name: true, slug: true, color: true }
        },
        tags: {
          select: {
            tag: { select: { id: true, name: true, slug: true } }
          }
        },
        _count: {
          select: { comments: true, likes: true }
        }
      }
    }),
    
    prisma.post.count({ where }),
    
    prisma.category.findMany({
      include: {
        _count: { select: { posts: { where: { published: true } } } }
      },
      orderBy: { posts: { _count: 'desc' } },
      take: 10
    }),
    
    prisma.tag.findMany({
      include: {
        _count: { select: { posts: true } }
      },
      orderBy: { posts: { _count: 'desc' } },
      take: 20
    })
  ])
  
  return {
    posts: posts.map(post => ({
      ...post,
      tags: post.tags.map(pt => pt.tag)
    })),
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
      hasNext: skip + limit < total,
      hasPrev: page > 1
    },
    meta: {
      categories: categories.map(c => ({
        ...c,
        postCount: c._count.posts
      })),
      popularTags: popularTags.map(t => ({
        ...t,
        postCount: t._count.posts
      }))
    }
  }
})
```

```typescript
// server/api/blog/[slug].get.ts
export default defineEventHandler(async (event) => {
  const slug = getRouterParam(event, 'slug')!
  
  const post = await prisma.post.findUnique({
    where: { slug, published: true },
    include: {
      author: {
        select: {
          id: true, name: true, avatar: true, bio: true,
          _count: { select: { posts: true, followers: true } }
        }
      },
      category: true,
      tags: { include: { tag: true } },
      comments: {
        where: { parentId: null },
        include: {
          author: { select: { id: true, name: true, avatar: true } },
          replies: {
            include: {
              author: { select: { id: true, name: true, avatar: true } }
            },
            orderBy: { createdAt: 'asc' }
          }
        },
        orderBy: { createdAt: 'desc' },
        take: 20
      },
      _count: {
        select: { comments: true, likes: true }
      }
    }
  })
  
  if (!post) {
    throw createError({ statusCode: 404, message: 'ไม่พบโพสต์' })
  }
  
  // เพิ่ม view count (ไม่รอผล)
  prisma.post.update({
    where: { id: post.id },
    data: { viewCount: { increment: 1 } }
  }).catch(console.error)
  
  // ดึง related posts
  const relatedPosts = await prisma.post.findMany({
    where: {
      published: true,
      id: { not: post.id },
      OR: [
        { categoryId: post.categoryId },
        { tags: { some: { tagId: { in: post.tags.map(t => t.tagId) } } } }
      ]
    },
    take: 4,
    orderBy: { publishedAt: 'desc' },
    select: {
      id: true, title: true, slug: true,
      excerpt: true, coverImage: true, publishedAt: true,
      author: { select: { name: true } },
      _count: { select: { comments: true, likes: true } }
    }
  })
  
  return {
    post: {
      ...post,
      tags: post.tags.map(pt => pt.tag)
    },
    relatedPosts
  }
})
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Prisma Setup** - การติดตั้งและตั้งค่า Prisma กับ Nuxt
2. **Schema Design** - การออกแบบ schema สำหรับ Blog ที่สมบูรณ์
3. **Migrations** - การจัดการ database migrations
4. **CRUD Operations** - Create, Read, Update, Delete ด้วย Prisma
5. **Relations** - One-to-Many และ Many-to-Many relationships
6. **Transactions** - การทำ atomic operations
7. **Seeding** - การเติมข้อมูลเริ่มต้น
8. **Environment Config** - การจัดการ environment variables อย่างปลอดภัย
