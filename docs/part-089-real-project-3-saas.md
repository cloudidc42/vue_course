# Part 89: โปรเจกต์จริง 3 - SaaS App

## Overview

สร้าง SaaS Application ที่สมบูรณ์: Project Management Tool (คล้าย Trello/Linear)
- Multi-tenant Architecture
- Subscription Plans
- User Management + Roles
- Analytics Dashboard  
- API Keys สำหรับ External Access
- Billing และ Invoicing

---

## 1. SaaS Product Planning

### Product Vision

```
ProjectFlow - The modern project management tool for small teams

Plans:
Free:    1 project, 3 members, basic features
Starter: 10 projects, 10 members, integrations ($29/mo)
Pro:     Unlimited projects, 50 members, analytics ($99/mo)
Enterprise: Custom
```

---

## 2. Database Schema

```prisma
// prisma/schema.prisma

model Tenant {
  id           String    @id @default(cuid())
  name         String
  slug         String    @unique
  plan         String    @default("free")
  logo         String?
  createdAt    DateTime  @default(now())
  
  members      Member[]
  projects     Project[]
  subscription Subscription?
  apiKeys      ApiKey[]
  invoices     Invoice[]

  @@map("tenants")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  avatar    String?
  password  String
  createdAt DateTime @default(now())

  memberships Member[]
  assignedTasks Task[]

  @@map("users")
}

model Member {
  id       String @id @default(cuid())
  userId   String
  user     User   @relation(fields: [userId], references: [id])
  tenantId String
  tenant   Tenant @relation(fields: [tenantId], references: [id])
  role     MemberRole @default(MEMBER)
  joinedAt DateTime @default(now())

  @@unique([userId, tenantId])
  @@map("members")
}

enum MemberRole {
  OWNER
  ADMIN
  MEMBER
  VIEWER
}

model Project {
  id          String  @id @default(cuid())
  tenantId    String
  tenant      Tenant  @relation(fields: [tenantId], references: [id])
  name        String
  description String?
  status      String  @default("active")
  color       String  @default("#3b82f6")
  boards      Board[]
  createdAt   DateTime @default(now())

  @@map("projects")
}

model Board {
  id        String  @id @default(cuid())
  projectId String
  project   Project @relation(fields: [projectId], references: [id])
  name      String
  position  Int     @default(0)
  tasks     Task[]

  @@map("boards")
}

model Task {
  id          String   @id @default(cuid())
  boardId     String
  board       Board    @relation(fields: [boardId], references: [id])
  title       String
  description String?
  priority    Priority @default(MEDIUM)
  dueDate     DateTime?
  position    Int      @default(0)
  
  assigneeId  String?
  assignee    User?    @relation(fields: [assigneeId], references: [id])
  
  labels      TaskLabel[]
  comments    TaskComment[]
  attachments TaskAttachment[]

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@map("tasks")
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}

model ApiKey {
  id          String   @id @default(cuid())
  tenantId    String
  tenant      Tenant   @relation(fields: [tenantId], references: [id])
  name        String
  key         String   @unique // hashed
  prefix      String   // First 8 chars for display
  permissions String[] // ['read:projects', 'write:tasks']
  lastUsedAt  DateTime?
  expiresAt   DateTime?
  createdAt   DateTime @default(now())

  @@map("api_keys")
}

model Subscription {
  id                   String   @id @default(cuid())
  tenantId             String   @unique
  tenant               Tenant   @relation(fields: [tenantId], references: [id])
  plan                 String
  status               String
  stripeCustomerId     String?
  stripeSubscriptionId String?
  currentPeriodEnd     DateTime?

  @@map("subscriptions")
}

model Invoice {
  id          String   @id @default(cuid())
  tenantId    String
  tenant      Tenant   @relation(fields: [tenantId], references: [id])
  amount      Decimal  @db.Decimal(10, 2)
  status      String   // paid, pending, failed
  paidAt      DateTime?
  stripeInvoiceId String?
  createdAt   DateTime @default(now())

  @@map("invoices")
}
```

---

## 3. Multi-tenant Setup

```typescript
// server/middleware/01.tenant.ts
export default defineEventHandler(async (event) => {
  const host = getHeader(event, 'host') || ''
  const subdomain = extractSubdomain(host)

  if (!subdomain || subdomain === 'www' || subdomain === 'app') {
    return
  }

  const tenant = await prisma.tenant.findUnique({
    where: { slug: subdomain },
    include: { subscription: true }
  })

  if (!tenant) {
    throw createError({ statusCode: 404, message: 'Workspace not found' })
  }

  event.context.tenant = tenant
  event.context.tenantId = tenant.id
  event.context.plan = tenant.plan
})

function extractSubdomain(host: string): string {
  const parts = host.split('.')
  if (parts.length > 2) return parts[0]
  return ''
}
```

---

## 4. Project Board (Kanban)

### Board API

```typescript
// server/api/projects/[id]/boards.get.ts
export default defineEventHandler(async (event) => {
  await requireAuth(event)

  const projectId = getRouterParam(event, 'id')!

  // ตรวจสอบว่าเป็น project ของ tenant นี้
  const project = await prisma.project.findFirst({
    where: { id: projectId, tenantId: event.context.tenantId }
  })

  if (!project) {
    throw createError({ statusCode: 404, message: 'Project not found' })
  }

  const boards = await prisma.board.findMany({
    where: { projectId },
    include: {
      tasks: {
        include: {
          assignee: { select: { name: true, avatar: true } },
          labels: true,
          _count: { select: { comments: true, attachments: true } }
        },
        orderBy: { position: 'asc' }
      }
    },
    orderBy: { position: 'asc' }
  })

  return boards
})

// Drag-and-drop reorder
// server/api/tasks/[id]/move.patch.ts
export default defineEventHandler(async (event) => {
  await requireAuth(event)

  const taskId = getRouterParam(event, 'id')!
  const { targetBoardId, position } = await readBody(event)

  const task = await prisma.task.findFirst({
    where: {
      id: taskId,
      board: { project: { tenantId: event.context.tenantId } }
    }
  })

  if (!task) {
    throw createError({ statusCode: 404, message: 'Task not found' })
  }

  // Re-order tasks ใน board
  await prisma.$transaction(async (tx) => {
    // Update task's board and position
    await tx.task.update({
      where: { id: taskId },
      data: { boardId: targetBoardId, position }
    })

    // Re-order other tasks ใน target board
    await tx.task.updateMany({
      where: {
        boardId: targetBoardId,
        id: { not: taskId },
        position: { gte: position }
      },
      data: { position: { increment: 1 } }
    })
  })

  return { success: true }
})
```

### Kanban Board Component

```vue
<!-- components/KanbanBoard.vue -->
<template>
  <div class="kanban-board">
    <div
      v-for="board in boards"
      :key="board.id"
      class="board-column"
      @dragover.prevent
      @drop="handleDrop($event, board.id)"
    >
      <div class="board-header">
        <h3>{{ board.name }}</h3>
        <span class="task-count">{{ board.tasks.length }}</span>
        <button @click="addTask(board.id)">+</button>
      </div>

      <div class="tasks-list">
        <TaskCard
          v-for="task in board.tasks"
          :key="task.id"
          :task="task"
          draggable="true"
          @dragstart="handleDragStart($event, task)"
          @click="openTask(task)"
        />
      </div>
    </div>

    <!-- Add Board -->
    <button @click="addBoard" class="add-board-btn">+ Add Column</button>
  </div>
</template>

<script setup lang="ts">
const props = defineProps<{ projectId: string }>()
const { data: boards, refresh } = await useFetch(`/api/projects/${props.projectId}/boards`)

const draggedTask = ref<any>(null)

function handleDragStart(event: DragEvent, task: any) {
  draggedTask.value = task
  event.dataTransfer!.setData('taskId', task.id)
}

async function handleDrop(event: DragEvent, targetBoardId: string) {
  const taskId = event.dataTransfer!.getData('taskId')

  await $fetch(`/api/tasks/${taskId}/move`, {
    method: 'PATCH',
    body: { targetBoardId, position: 0 }
  })

  refresh()
}

async function addTask(boardId: string) {
  const title = prompt('Task title:')
  if (!title) return

  await $fetch('/api/tasks', {
    method: 'POST',
    body: { boardId, title }
  })

  refresh()
}
</script>
```

---

## 5. API Keys Management

```typescript
// server/api/api-keys/index.post.ts
import crypto from 'crypto'
import { createHash } from 'crypto'

export default defineEventHandler(async (event) => {
  await requireAuth(event)
  await requireRole(event, ['OWNER', 'ADMIN'])

  const { name, permissions, expiresAt } = await readBody(event)

  // สร้าง API key
  const rawKey = `pfk_${crypto.randomBytes(32).toString('hex')}`
  const prefix = rawKey.slice(0, 12) // "pfk_xxxxxxxx"
  const hashedKey = createHash('sha256').update(rawKey).digest('hex')

  const apiKey = await prisma.apiKey.create({
    data: {
      tenantId: event.context.tenantId,
      name,
      key: hashedKey,
      prefix,
      permissions: permissions || ['read:projects'],
      expiresAt: expiresAt ? new Date(expiresAt) : null
    }
  })

  // Return raw key ครั้งเดียว ไม่เก็บใน DB
  return {
    id: apiKey.id,
    name: apiKey.name,
    prefix: apiKey.prefix,
    key: rawKey, // ส่งให้ user ครั้งเดียวเท่านั้น!
    createdAt: apiKey.createdAt
  }
})

// API Key Authentication Middleware
// server/middleware/02.api-key-auth.ts
export default defineEventHandler(async (event) => {
  const apiKeyHeader = getHeader(event, 'x-api-key')
  if (!apiKeyHeader) return

  const hashedKey = createHash('sha256').update(apiKeyHeader).digest('hex')

  const apiKey = await prisma.apiKey.findUnique({
    where: { key: hashedKey },
    include: { tenant: true }
  })

  if (!apiKey) {
    throw createError({ statusCode: 401, message: 'Invalid API key' })
  }

  if (apiKey.expiresAt && apiKey.expiresAt < new Date()) {
    throw createError({ statusCode: 401, message: 'API key expired' })
  }

  // Set context สำหรับ request
  event.context.tenantId = apiKey.tenantId
  event.context.tenant = apiKey.tenant
  event.context.apiKeyPermissions = apiKey.permissions
  event.context.isApiKeyAuth = true

  // Update last used
  await prisma.apiKey.update({
    where: { id: apiKey.id },
    data: { lastUsedAt: new Date() }
  })
})
```

---

## 6. Analytics Dashboard

### Analytics API

```typescript
// server/api/analytics/overview.get.ts
export default defineEventHandler(async (event) => {
  await requireAuth(event)

  const tenantId = event.context.tenantId
  const { period = '30' } = getQuery(event)
  const daysAgo = new Date()
  daysAgo.setDate(daysAgo.getDate() - Number(period))

  const [projectCount, taskCount, memberCount, tasksByPriority, taskCompletionRate] =
    await Promise.all([
      prisma.project.count({ where: { tenantId } }),

      prisma.task.count({
        where: {
          board: { project: { tenantId } },
          createdAt: { gte: daysAgo }
        }
      }),

      prisma.member.count({ where: { tenantId } }),

      prisma.task.groupBy({
        by: ['priority'],
        where: { board: { project: { tenantId } } },
        _count: { _all: true }
      }),

      // % of tasks completed
      prisma.$queryRaw<[{ completed: bigint; total: bigint }]>`
        SELECT 
          COUNT(*) FILTER (WHERE b.name = 'Done') as completed,
          COUNT(*) as total
        FROM tasks t
        JOIN boards b ON t.board_id = b.id
        JOIN projects p ON b.project_id = p.id
        WHERE p.tenant_id = ${tenantId}
      `
    ])

  const completionRate = taskCompletionRate[0]
    ? Math.round(Number(taskCompletionRate[0].completed) / Number(taskCompletionRate[0].total) * 100)
    : 0

  return {
    projectCount,
    taskCount,
    memberCount,
    tasksByPriority,
    completionRate
  }
})
```

### Analytics Dashboard

```vue
<!-- pages/analytics/index.vue -->
<template>
  <div class="analytics">
    <h1>Analytics</h1>

    <!-- Period Selector -->
    <div class="period-selector">
      <button v-for="p in periods" :key="p.value"
        :class="{ active: period === p.value }"
        @click="period = p.value"
      >
        {{ p.label }}
      </button>
    </div>

    <!-- Overview Cards -->
    <div class="stats-grid">
      <StatCard icon="folder" label="Active Projects" :value="stats?.projectCount" />
      <StatCard icon="check" label="Tasks (Period)" :value="stats?.taskCount" />
      <StatCard icon="users" label="Team Members" :value="stats?.memberCount" />
      <StatCard icon="chart" label="Completion Rate" :value="`${stats?.completionRate}%`" />
    </div>

    <!-- Charts -->
    <div class="charts-grid">
      <div class="chart-card">
        <h3>Tasks by Priority</h3>
        <PriorityDonutChart :data="stats?.tasksByPriority" />
      </div>

      <div class="chart-card">
        <h3>Task Velocity</h3>
        <VelocityLineChart :period="Number(period)" />
      </div>

      <div class="chart-card">
        <h3>Team Activity</h3>
        <ActivityHeatmap :period="Number(period)" />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
const period = ref('30')
const periods = [
  { label: '7 Days', value: '7' },
  { label: '30 Days', value: '30' },
  { label: '90 Days', value: '90' }
]

const { data: stats } = useFetch('/api/analytics/overview', {
  query: computed(() => ({ period: period.value }))
})
</script>
```

---

## 7. Billing & Invoicing

```typescript
// server/api/billing/invoices.get.ts
export default defineEventHandler(async (event) => {
  await requireAuth(event)
  await requireRole(event, ['OWNER', 'ADMIN'])

  const invoices = await prisma.invoice.findMany({
    where: { tenantId: event.context.tenantId },
    orderBy: { createdAt: 'desc' }
  })

  return invoices
})

// server/api/billing/invoices/[id]/download.get.ts
import PDFDocument from 'pdfkit'

export default defineEventHandler(async (event) => {
  await requireAuth(event)

  const invoiceId = getRouterParam(event, 'id')!
  const invoice = await prisma.invoice.findFirst({
    where: { id: invoiceId, tenantId: event.context.tenantId },
    include: { tenant: true }
  })

  if (!invoice) {
    throw createError({ statusCode: 404, message: 'Invoice not found' })
  }

  // สร้าง PDF
  const doc = new PDFDocument()
  const buffers: Buffer[] = []

  doc.on('data', data => buffers.push(data))

  doc.fontSize(20).text('INVOICE', 50, 50)
  doc.fontSize(12).text(`Invoice #${invoice.id.slice(-8).toUpperCase()}`, 50, 80)
  doc.text(`Date: ${new Date(invoice.createdAt).toLocaleDateString('th-TH')}`, 50, 100)
  doc.text(`Company: ${invoice.tenant.name}`, 50, 120)
  doc.text(`Amount: $${invoice.amount}`, 50, 140)
  doc.text(`Status: ${invoice.status.toUpperCase()}`, 50, 160)

  doc.end()

  await new Promise(resolve => doc.on('end', resolve))

  const pdfBuffer = Buffer.concat(buffers)

  setHeader(event, 'Content-Type', 'application/pdf')
  setHeader(event, 'Content-Disposition', `attachment; filename="invoice-${invoice.id.slice(-8)}.pdf"`)

  return pdfBuffer
})
```

### Invoices Page

```vue
<!-- pages/billing/invoices.vue -->
<template>
  <div class="billing-invoices">
    <h1>Invoices</h1>

    <div class="invoices-table">
      <table>
        <thead>
          <tr>
            <th>Invoice #</th>
            <th>Date</th>
            <th>Amount</th>
            <th>Status</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="invoice in invoices" :key="invoice.id">
            <td>{{ invoice.id.slice(-8).toUpperCase() }}</td>
            <td>{{ formatDate(invoice.createdAt) }}</td>
            <td>${{ invoice.amount }}</td>
            <td>
              <Badge :variant="invoice.status === 'paid' ? 'success' : 'warning'">
                {{ invoice.status }}
              </Badge>
            </td>
            <td>
              <a :href="`/api/billing/invoices/${invoice.id}/download`" 
                 target="_blank"
                 class="download-link">
                Download PDF
              </a>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup lang="ts">
definePageMeta({ middleware: ['auth', 'admin'] })

const { data: invoices } = useFetch('/api/billing/invoices')

function formatDate(date: string) {
  return new Intl.DateTimeFormat('th-TH').format(new Date(date))
}
</script>
```

---

## 8. Real-time Updates กับ WebSocket

```typescript
// server/plugins/websocket.ts
import { Server } from 'socket.io'

export default defineNitroPlugin((nitroApp) => {
  const io = new Server(nitroApp.h3App.server)

  io.use(async (socket, next) => {
    const token = socket.handshake.auth.token
    try {
      const user = await verifyToken(token)
      socket.data.userId = user.id
      socket.data.tenantId = user.tenantId
      next()
    } catch {
      next(new Error('Authentication failed'))
    }
  })

  io.on('connection', (socket) => {
    const { tenantId } = socket.data

    // Join tenant room
    socket.join(`tenant:${tenantId}`)

    socket.on('join-project', (projectId: string) => {
      socket.join(`project:${projectId}`)
    })

    socket.on('leave-project', (projectId: string) => {
      socket.leave(`project:${projectId}`)
    })

    socket.on('task-moved', async (data) => {
      // Broadcast to everyone in the project
      socket.to(`project:${data.projectId}`).emit('task-moved', data)
    })
  })

  // Expose io for server events
  nitroApp.hooks.hook('request', (event) => {
    event.context.io = io
  })
})
```

---

## สรุป

SaaS App project ครอบคลุม:
1. **Multi-tenancy** - แต่ละ workspace แยกกัน
2. **Role-based access** - Owner, Admin, Member, Viewer
3. **Kanban board** - Drag-and-drop
4. **API Keys** - สำหรับ external integrations
5. **Analytics** - Usage insights
6. **Billing** - Stripe + PDF invoices
7. **Real-time** - WebSocket สำหรับ live collaboration

นี่คือ project ที่ซับซ้อนที่สุดในหลักสูตร และแสดงให้เห็นว่าคุณสามารถสร้าง production-ready SaaS ได้แล้ว
