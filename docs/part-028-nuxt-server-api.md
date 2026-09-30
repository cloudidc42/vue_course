# Part 28: Nuxt Server Routes (API)

## server/api/ directory

Nuxt มี built-in server engine ชื่อ Nitro ที่ช่วยให้สร้าง API endpoints ได้โดยตรงในโปรเจกต์ โดยวางไฟล์ใน `server/api/`

### โครงสร้าง server/api/

```
server/
├── api/
│   ├── users/
│   │   ├── index.ts          → GET/POST /api/users
│   │   └── [id].ts           → GET/PUT/DELETE /api/users/:id
│   ├── posts/
│   │   ├── index.ts          → GET/POST /api/posts
│   │   └── [id]/
│   │       ├── index.ts      → /api/posts/:id
│   │       └── comments.ts   → /api/posts/:id/comments
│   └── auth/
│       ├── login.post.ts     → POST /api/auth/login เท่านั้น
│       ├── logout.post.ts    → POST /api/auth/logout
│       └── me.get.ts         → GET /api/auth/me
├── middleware/
│   ├── logger.ts             → ทำงานกับทุก request
│   └── auth.ts
└── utils/
    ├── db.ts                 → Database utilities
    └── jwt.ts                → JWT utilities
```

### HTTP Method ใน Filename

```
users.get.ts      → GET /api/users
users.post.ts     → POST /api/users
users.put.ts      → PUT /api/users
users.delete.ts   → DELETE /api/users
users.patch.ts    → PATCH /api/users

users.ts          → ทุก method (ต้องตรวจ method เอง)
```

---

## สร้าง API Endpoints

### GET Endpoint พื้นฐาน

```typescript
// server/api/hello.ts
export default defineEventHandler((event) => {
  return {
    message: 'Hello from Nuxt API!',
    timestamp: new Date().toISOString()
  }
})
```

### GET กับ Query Parameters

```typescript
// server/api/posts.get.ts
export default defineEventHandler(async (event) => {
  // ดึง query parameters
  const query = getQuery(event)
  
  const page = parseInt(query.page as string || '1')
  const limit = parseInt(query.limit as string || '10')
  const category = query.category as string
  const search = query.q as string
  
  // Validate
  if (page < 1 || limit > 100) {
    throw createError({
      statusCode: 400,
      statusMessage: 'Invalid pagination parameters'
    })
  }
  
  // ดึงข้อมูลจาก database (ตัวอย่าง)
  const posts = await db.posts.findAll({
    where: {
      ...(category && { category }),
      ...(search && {
        OR: [
          { title: { contains: search } },
          { content: { contains: search } }
        ]
      })
    },
    limit,
    offset: (page - 1) * limit,
    orderBy: { createdAt: 'desc' }
  })
  
  const total = await db.posts.count()
  
  return {
    data: posts,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit)
    }
  }
})
```

---

## HTTP Methods (GET, POST, PUT, DELETE)

### Complete CRUD Endpoints

```typescript
// server/api/posts/index.ts - GET all posts and POST new post

export default defineEventHandler(async (event) => {
  const method = getMethod(event)
  
  if (method === 'GET') {
    const query = getQuery(event)
    const posts = await getAllPosts(query)
    return posts
  }
  
  if (method === 'POST') {
    await requireAuth(event)  // ต้อง login
    const body = await readBody(event)
    const post = await createPost(body, event.context.user)
    
    setResponseStatus(event, 201)
    return post
  }
  
  throw createError({ statusCode: 405, statusMessage: 'Method Not Allowed' })
})
```

```typescript
// server/api/posts/[id].ts - GET, PUT, DELETE single post

export default defineEventHandler(async (event) => {
  const method = getMethod(event)
  const id = getRouterParam(event, 'id')
  
  if (!id) {
    throw createError({ statusCode: 400, statusMessage: 'ID is required' })
  }
  
  if (method === 'GET') {
    const post = await db.post.findUnique({ where: { id: parseInt(id) } })
    
    if (!post) {
      throw createError({ statusCode: 404, statusMessage: 'Post not found' })
    }
    
    return post
  }
  
  if (method === 'PUT') {
    await requireAuth(event)
    const body = await readBody(event)
    
    const post = await db.post.update({
      where: { id: parseInt(id) },
      data: body
    })
    
    return post
  }
  
  if (method === 'DELETE') {
    await requireAuth(event)
    await db.post.delete({ where: { id: parseInt(id) } })
    
    setResponseStatus(event, 204)
    return null
  }
  
  throw createError({ statusCode: 405, statusMessage: 'Method Not Allowed' })
})
```

---

## Request Body, Query, Params

### อ่าน Request Data ทุกประเภท

```typescript
// server/api/example.ts
export default defineEventHandler(async (event) => {
  // 1. Route Params
  const id = getRouterParam(event, 'id')
  const userId = getRouterParam(event, 'userId')
  
  // 2. Query String
  const query = getQuery(event)
  const page = query.page as string
  const filter = query.filter as string
  
  // 3. Request Body (JSON)
  const body = await readBody(event)
  
  // 4. Form Data
  const formData = await readFormData(event)
  const name = formData.get('name') as string
  const file = formData.get('file') as File
  
  // 5. Raw body
  const rawBody = await readRawBody(event)
  
  // 6. Request Headers
  const contentType = getRequestHeader(event, 'content-type')
  const authHeader = getRequestHeader(event, 'authorization')
  const userAgent = getRequestHeader(event, 'user-agent')
  
  // 7. Cookies
  const token = getCookie(event, 'auth-token')
  const locale = getCookie(event, 'locale')
  
  // 8. Request info
  const method = getMethod(event)
  const path = event.path
  const ip = getRequestIP(event)
  
  return { id, query, method, ip }
})
```

### Validation ด้วย zod

```typescript
// server/api/posts.post.ts
import { z } from 'zod'

const CreatePostSchema = z.object({
  title: z.string().min(3).max(200),
  content: z.string().min(10),
  category: z.string(),
  tags: z.array(z.string()).optional(),
  published: z.boolean().default(false),
  slug: z.string().optional()
})

type CreatePostData = z.infer<typeof CreatePostSchema>

export default defineEventHandler(async (event) => {
  // ต้อง login
  const user = await requireAuth(event)
  
  // อ่าน body
  const rawBody = await readBody(event)
  
  // Validate
  const result = CreatePostSchema.safeParse(rawBody)
  
  if (!result.success) {
    throw createError({
      statusCode: 422,
      statusMessage: 'Validation Error',
      data: result.error.flatten()
    })
  }
  
  const data: CreatePostData = result.data
  
  // สร้าง slug ถ้าไม่มี
  if (!data.slug) {
    data.slug = data.title
      .toLowerCase()
      .replace(/[^a-z0-9ก-ฮ]+/g, '-')
      .replace(/^-|-$/g, '')
  }
  
  // บันทึก
  const post = await db.post.create({
    data: {
      ...data,
      authorId: user.id
    }
  })
  
  setResponseStatus(event, 201)
  return post
})
```

---

## Error Handling

### สร้าง Error responses

```typescript
// server/utils/errors.ts
export function notFound(message = 'Not found') {
  return createError({ statusCode: 404, statusMessage: message })
}

export function unauthorized(message = 'Unauthorized') {
  return createError({ statusCode: 401, statusMessage: message })
}

export function forbidden(message = 'Forbidden') {
  return createError({ statusCode: 403, statusMessage: message })
}

export function validationError(errors: any) {
  return createError({
    statusCode: 422,
    statusMessage: 'Validation Error',
    data: errors
  })
}

export function serverError(message = 'Internal Server Error') {
  return createError({ statusCode: 500, statusMessage: message })
}
```

### Global Error Handler

```typescript
// server/plugins/error-handler.ts
export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('error', (error) => {
    console.error('[Server Error]', {
      message: error.message,
      stack: error.stack,
      statusCode: error.statusCode
    })
  })
})
```

---

## Middleware สำหรับ API

### Auth Middleware

```typescript
// server/utils/auth.ts
import jwt from 'jsonwebtoken'

export interface JWTPayload {
  userId: number
  email: string
  role: string
}

export async function requireAuth(event: H3Event): Promise<JWTPayload> {
  const authHeader = getRequestHeader(event, 'authorization')
  const cookieToken = getCookie(event, 'auth-token')
  
  const token = authHeader?.replace('Bearer ', '') || cookieToken
  
  if (!token) {
    throw createError({
      statusCode: 401,
      statusMessage: 'กรุณาเข้าสู่ระบบ'
    })
  }
  
  try {
    const config = useRuntimeConfig()
    const payload = jwt.verify(token, config.jwtSecret) as JWTPayload
    
    // เพิ่ม user เข้า event context
    event.context.user = payload
    return payload
  } catch (error) {
    throw createError({
      statusCode: 401,
      statusMessage: 'Token ไม่ถูกต้องหรือหมดอายุ'
    })
  }
}

export async function requireRole(event: H3Event, roles: string[]): Promise<JWTPayload> {
  const user = await requireAuth(event)
  
  if (!roles.includes(user.role)) {
    throw createError({
      statusCode: 403,
      statusMessage: 'คุณไม่มีสิทธิ์ทำรายการนี้'
    })
  }
  
  return user
}
```

---

## Database Connection (Prisma)

### ติดตั้ง Prisma

```bash
npm install @prisma/client
npm install -D prisma

# Initialize Prisma
npx prisma init

# Generate client หลัง schema เปลี่ยน
npx prisma generate

# Migrate database
npx prisma migrate dev --name init

# Open Prisma Studio
npx prisma studio
```

### Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  password  String
  role      String   @default("user")
  avatar    String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  posts     Post[]
  comments  Comment[]
  todos     Todo[]
}

model Post {
  id          Int       @id @default(autoincrement())
  title       String
  slug        String    @unique
  content     String
  excerpt     String?
  published   Boolean   @default(false)
  authorId    Int
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  
  author      User      @relation(fields: [authorId], references: [id])
  comments    Comment[]
}

model Comment {
  id        Int      @id @default(autoincrement())
  content   String
  postId    Int
  authorId  Int
  createdAt DateTime @default(now())
  
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  author    User     @relation(fields: [authorId], references: [id])
}

model Todo {
  id          Int      @id @default(autoincrement())
  title       String
  completed   Boolean  @default(false)
  priority    String   @default("medium")
  userId      Int
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  user        User     @relation(fields: [userId], references: [id])
}
```

### Prisma Client Singleton

```typescript
// server/utils/db.ts
import { PrismaClient } from '@prisma/client'

let prisma: PrismaClient

declare global {
  var __prisma: PrismaClient | undefined
}

if (process.env.NODE_ENV === 'production') {
  prisma = new PrismaClient()
} else {
  // ใน development ใช้ global instance เพื่อไม่ให้ create ใหม่ทุกครั้ง
  if (!global.__prisma) {
    global.__prisma = new PrismaClient({
      log: ['query', 'info', 'warn', 'error']
    })
  }
  prisma = global.__prisma
}

export { prisma as db }
```

---

## ตัวอย่าง: Full CRUD API สำหรับ Todo App

### Todo API - รายการ

```typescript
// server/api/todos/index.ts
import { z } from 'zod'
import { db } from '~/server/utils/db'
import { requireAuth } from '~/server/utils/auth'

const CreateTodoSchema = z.object({
  title: z.string().min(1, 'กรุณาระบุชื่อ Todo').max(500),
  priority: z.enum(['low', 'medium', 'high']).default('medium'),
  dueDate: z.string().optional()
})

export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  const method = getMethod(event)
  
  // GET - ดึงรายการ todos ของ user
  if (method === 'GET') {
    const query = getQuery(event)
    const filter = query.filter as string  // 'all' | 'active' | 'completed'
    const priority = query.priority as string
    const page = parseInt(query.page as string || '1')
    const limit = parseInt(query.limit as string || '20')
    
    const where: any = { userId: user.userId }
    
    if (filter === 'active') where.completed = false
    if (filter === 'completed') where.completed = true
    if (priority) where.priority = priority
    
    const [todos, total] = await Promise.all([
      db.todo.findMany({
        where,
        orderBy: [
          { completed: 'asc' },
          { priority: 'desc' },
          { createdAt: 'desc' }
        ],
        skip: (page - 1) * limit,
        take: limit
      }),
      db.todo.count({ where })
    ])
    
    return {
      data: todos,
      meta: {
        total,
        page,
        limit,
        totalPages: Math.ceil(total / limit),
        stats: {
          total: await db.todo.count({ where: { userId: user.userId } }),
          completed: await db.todo.count({ where: { userId: user.userId, completed: true } }),
          active: await db.todo.count({ where: { userId: user.userId, completed: false } })
        }
      }
    }
  }
  
  // POST - สร้าง todo ใหม่
  if (method === 'POST') {
    const rawBody = await readBody(event)
    const result = CreateTodoSchema.safeParse(rawBody)
    
    if (!result.success) {
      throw createError({
        statusCode: 422,
        statusMessage: 'Validation Error',
        data: result.error.flatten()
      })
    }
    
    const todo = await db.todo.create({
      data: {
        title: result.data.title,
        priority: result.data.priority,
        userId: user.userId
      }
    })
    
    setResponseStatus(event, 201)
    return todo
  }
  
  throw createError({ statusCode: 405, statusMessage: 'Method Not Allowed' })
})
```

### Todo API - Single Item

```typescript
// server/api/todos/[id].ts
import { db } from '~/server/utils/db'
import { requireAuth } from '~/server/utils/auth'

export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  const method = getMethod(event)
  const id = parseInt(getRouterParam(event, 'id') || '0')
  
  if (!id) {
    throw createError({ statusCode: 400, statusMessage: 'Invalid ID' })
  }
  
  // ดึง todo และตรวจสอบเจ้าของ
  const todo = await db.todo.findUnique({ where: { id } })
  
  if (!todo) {
    throw createError({ statusCode: 404, statusMessage: 'Todo not found' })
  }
  
  if (todo.userId !== user.userId) {
    throw createError({ statusCode: 403, statusMessage: 'Forbidden' })
  }
  
  // GET
  if (method === 'GET') {
    return todo
  }
  
  // PUT - อัพเดตทั้งหมด
  if (method === 'PUT') {
    const body = await readBody(event)
    
    const updated = await db.todo.update({
      where: { id },
      data: {
        title: body.title,
        completed: body.completed,
        priority: body.priority,
        updatedAt: new Date()
      }
    })
    
    return updated
  }
  
  // PATCH - อัพเดตบางส่วน
  if (method === 'PATCH') {
    const body = await readBody(event)
    
    const updated = await db.todo.update({
      where: { id },
      data: {
        ...(body.title !== undefined && { title: body.title }),
        ...(body.completed !== undefined && { completed: body.completed }),
        ...(body.priority !== undefined && { priority: body.priority }),
        updatedAt: new Date()
      }
    })
    
    return updated
  }
  
  // DELETE
  if (method === 'DELETE') {
    await db.todo.delete({ where: { id } })
    setResponseStatus(event, 204)
    return null
  }
  
  throw createError({ statusCode: 405, statusMessage: 'Method Not Allowed' })
})
```

### Bulk Operations

```typescript
// server/api/todos/bulk.post.ts
import { db } from '~/server/utils/db'
import { requireAuth } from '~/server/utils/auth'

export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  const body = await readBody(event)
  
  const { action, ids } = body
  
  if (!action || !Array.isArray(ids)) {
    throw createError({ statusCode: 400, statusMessage: 'Invalid request body' })
  }
  
  // ตรวจสอบว่า todos เป็นของ user
  const todos = await db.todo.findMany({
    where: { id: { in: ids }, userId: user.userId }
  })
  
  if (todos.length !== ids.length) {
    throw createError({ statusCode: 403, statusMessage: 'Some todos not found or forbidden' })
  }
  
  if (action === 'complete') {
    await db.todo.updateMany({
      where: { id: { in: ids } },
      data: { completed: true }
    })
    return { success: true, message: `Completed ${ids.length} todos` }
  }
  
  if (action === 'delete') {
    await db.todo.deleteMany({
      where: { id: { in: ids } }
    })
    return { success: true, message: `Deleted ${ids.length} todos` }
  }
  
  throw createError({ statusCode: 400, statusMessage: 'Unknown action' })
})
```

### Todo Frontend Component

```vue
<!-- pages/todos/index.vue -->
<script setup lang="ts">
definePageMeta({ middleware: ['auth'] })

interface Todo {
  id: number
  title: string
  completed: boolean
  priority: 'low' | 'medium' | 'high'
  createdAt: string
}

// State
const filter = ref<'all' | 'active' | 'completed'>('all')
const newTodoTitle = ref('')
const isAdding = ref(false)
const selectedIds = ref<Set<number>>(new Set())

// Fetch todos
const { data, pending, refresh } = await useFetch('/api/todos', {
  query: computed(() => ({ filter: filter.value }))
})

const todos = computed(() => data.value?.data || [])
const stats = computed(() => data.value?.meta.stats)

// Add todo
async function addTodo() {
  if (!newTodoTitle.value.trim()) return
  
  isAdding.value = true
  try {
    await $fetch('/api/todos', {
      method: 'POST',
      body: { title: newTodoTitle.value.trim() }
    })
    newTodoTitle.value = ''
    await refresh()
  } catch (error) {
    console.error('Failed to add todo')
  } finally {
    isAdding.value = false
  }
}

// Toggle complete
async function toggleTodo(id: number, completed: boolean) {
  await $fetch(`/api/todos/${id}`, {
    method: 'PATCH',
    body: { completed: !completed }
  })
  await refresh()
}

// Delete todo
async function deleteTodo(id: number) {
  await $fetch(`/api/todos/${id}`, { method: 'DELETE' })
  await refresh()
}

// Bulk actions
async function bulkComplete() {
  if (selectedIds.value.size === 0) return
  await $fetch('/api/todos/bulk', {
    method: 'POST',
    body: { action: 'complete', ids: Array.from(selectedIds.value) }
  })
  selectedIds.value.clear()
  await refresh()
}

async function bulkDelete() {
  if (selectedIds.value.size === 0) return
  if (!confirm(`ลบ ${selectedIds.value.size} รายการ?`)) return
  await $fetch('/api/todos/bulk', {
    method: 'POST',
    body: { action: 'delete', ids: Array.from(selectedIds.value) }
  })
  selectedIds.value.clear()
  await refresh()
}

const priorityColors = {
  high: 'text-red-600',
  medium: 'text-yellow-600',
  low: 'text-green-600'
}

const priorityLabels = { high: 'สูง', medium: 'กลาง', low: 'ต่ำ' }
</script>

<template>
  <div class="todo-app max-w-2xl mx-auto py-8 px-4">
    <h1 class="text-3xl font-bold text-gray-900 mb-8">รายการสิ่งที่ต้องทำ</h1>
    
    <!-- Stats -->
    <div class="stats grid grid-cols-3 gap-4 mb-6">
      <div class="stat-card bg-white rounded-xl p-4 shadow-sm text-center">
        <div class="text-2xl font-bold">{{ stats?.total || 0 }}</div>
        <div class="text-gray-500 text-sm">ทั้งหมด</div>
      </div>
      <div class="stat-card bg-white rounded-xl p-4 shadow-sm text-center">
        <div class="text-2xl font-bold text-blue-600">{{ stats?.active || 0 }}</div>
        <div class="text-gray-500 text-sm">กำลังทำ</div>
      </div>
      <div class="stat-card bg-white rounded-xl p-4 shadow-sm text-center">
        <div class="text-2xl font-bold text-green-600">{{ stats?.completed || 0 }}</div>
        <div class="text-gray-500 text-sm">เสร็จแล้ว</div>
      </div>
    </div>
    
    <!-- Add Todo Form -->
    <form @submit.prevent="addTodo" class="flex gap-2 mb-6">
      <input
        v-model="newTodoTitle"
        type="text"
        placeholder="เพิ่มสิ่งที่ต้องทำ..."
        class="flex-1 border border-gray-200 rounded-lg px-4 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
        :disabled="isAdding"
      />
      <button
        type="submit"
        :disabled="isAdding || !newTodoTitle.trim()"
        class="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700 disabled:opacity-50"
      >
        {{ isAdding ? '...' : 'เพิ่ม' }}
      </button>
    </form>
    
    <!-- Filter Tabs -->
    <div class="flex gap-1 mb-4 bg-gray-100 rounded-lg p-1">
      <button
        v-for="f in ['all', 'active', 'completed']"
        :key="f"
        @click="filter = f as any"
        :class="[
          'flex-1 py-1.5 rounded-md text-sm font-medium transition-all',
          filter === f ? 'bg-white shadow text-gray-900' : 'text-gray-500 hover:text-gray-700'
        ]"
      >
        {{ f === 'all' ? 'ทั้งหมด' : f === 'active' ? 'กำลังทำ' : 'เสร็จแล้ว' }}
      </button>
    </div>
    
    <!-- Bulk Actions -->
    <div v-if="selectedIds.size > 0" class="flex gap-2 mb-4 p-3 bg-blue-50 rounded-lg">
      <span class="text-sm text-blue-700">เลือก {{ selectedIds.size }} รายการ</span>
      <button @click="bulkComplete" class="text-sm text-green-600 hover:underline ml-auto">
        ทำเครื่องหมายเสร็จ
      </button>
      <button @click="bulkDelete" class="text-sm text-red-600 hover:underline">
        ลบที่เลือก
      </button>
    </div>
    
    <!-- Todo List -->
    <div v-if="pending" class="space-y-2">
      <div v-for="i in 5" :key="i" class="h-16 bg-gray-100 rounded-xl animate-pulse"></div>
    </div>
    
    <div v-else-if="todos.length === 0" class="text-center py-12 text-gray-400">
      <p class="text-lg">ไม่มีรายการ</p>
    </div>
    
    <TransitionGroup v-else name="list" tag="div" class="space-y-2">
      <div
        v-for="todo in todos"
        :key="todo.id"
        class="flex items-center gap-3 bg-white rounded-xl p-4 shadow-sm
               border border-transparent hover:border-gray-200 transition-all"
        :class="{ 'opacity-60': todo.completed }"
      >
        <!-- Checkbox -->
        <input
          type="checkbox"
          :checked="selectedIds.has(todo.id)"
          @change="selectedIds.has(todo.id) ? selectedIds.delete(todo.id) : selectedIds.add(todo.id)"
          class="w-4 h-4 rounded"
        />
        
        <!-- Complete Toggle -->
        <button
          @click="toggleTodo(todo.id, todo.completed)"
          :class="[
            'w-5 h-5 rounded-full border-2 flex items-center justify-center transition-all',
            todo.completed
              ? 'bg-green-500 border-green-500 text-white'
              : 'border-gray-300 hover:border-green-400'
          ]"
        >
          <span v-if="todo.completed" class="text-xs">✓</span>
        </button>
        
        <!-- Title -->
        <span
          class="flex-1 text-sm"
          :class="todo.completed ? 'line-through text-gray-400' : 'text-gray-900'"
        >
          {{ todo.title }}
        </span>
        
        <!-- Priority -->
        <span :class="['text-xs font-medium', priorityColors[todo.priority]]">
          {{ priorityLabels[todo.priority] }}
        </span>
        
        <!-- Delete -->
        <button
          @click="deleteTodo(todo.id)"
          class="text-gray-300 hover:text-red-500 transition-colors"
        >
          ×
        </button>
      </div>
    </TransitionGroup>
  </div>
</template>

<style scoped>
.list-enter-active, .list-leave-active {
  transition: all 0.3s ease;
}
.list-enter-from, .list-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}
</style>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **server/api/ directory** - โครงสร้าง API ใน Nuxt
2. **สร้าง Endpoints** - defineEventHandler()
3. **HTTP Methods** - GET, POST, PUT, PATCH, DELETE
4. **Request Data** - body, query, params, headers, cookies
5. **Validation** - ด้วย zod
6. **Error Handling** - createError()
7. **Server Middleware** - auth, logging
8. **Prisma Integration** - Database ORM
9. **Full CRUD Todo API** - ตัวอย่างสมบูรณ์

**ถัดไป**: Part 29 - Authentication ใน Nuxt
