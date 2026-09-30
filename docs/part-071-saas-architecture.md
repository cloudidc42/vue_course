# Part 71: Building SaaS Products

## SaaS Architecture คืออะไร?

SaaS (Software as a Service) คือรูปแบบการให้บริการซอฟต์แวร์ผ่านอินเทอร์เน็ต ที่ผู้ใช้เข้าถึงได้ผ่าน web browser โดยไม่ต้องติดตั้ง การสร้าง SaaS Product ที่ดีต้องออกแบบ Architecture ให้รองรับผู้ใช้หลายรายพร้อมกัน (Multi-tenancy) และมีระบบ Subscription Management ที่ชัดเจน

---

## 1. SaaS Architecture Patterns

### Single-tenant vs Multi-tenant

```
Single-tenant:
┌─────────────────┐  ┌─────────────────┐
│   Customer A    │  │   Customer B    │
│   Application   │  │   Application   │
│   Database      │  │   Database      │
└─────────────────┘  └─────────────────┘

Multi-tenant (Shared Database):
┌─────────────────────────────────────┐
│           Application               │
│  ┌─────────────────────────────┐   │
│  │   Shared Database            │   │
│  │   tenant_id on every table  │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

### เลือก Pattern ที่เหมาะสม

| Pattern | Pros | Cons | ใช้เมื่อ |
|---------|------|------|---------|
| Single-tenant | Security, Isolation | ราคาแพง | Enterprise, Compliance |
| Shared DB | ประหยัด, ง่าย | Security risk | Startup, SME |
| Separate Schema | Balance | Maintenance | Mid-size |

---

## 2. Multi-tenancy Implementation

### Database Schema Design

```sql
-- ตาราง tenants
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  plan VARCHAR(50) DEFAULT 'free',
  status VARCHAR(20) DEFAULT 'active',
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- ตาราง users (สัมพันธ์กับ tenants)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  email VARCHAR(255) NOT NULL,
  name VARCHAR(255),
  role VARCHAR(50) DEFAULT 'member',
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(tenant_id, email)
);

-- ตาราง projects (ตัวอย่าง entity)
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Index สำหรับ performance
CREATE INDEX idx_users_tenant_id ON users(tenant_id);
CREATE INDEX idx_projects_tenant_id ON projects(tenant_id);
```

### Prisma Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Tenant {
  id        String   @id @default(uuid())
  name      String
  slug      String   @unique
  plan      String   @default("free")
  status    String   @default("active")
  settings  Json     @default("{}")
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  users        User[]
  projects     Project[]
  subscription Subscription?

  @@map("tenants")
}

model User {
  id        String   @id @default(uuid())
  tenantId  String
  email     String
  name      String?
  role      String   @default("member")
  createdAt DateTime @default(now())

  tenant  Tenant    @relation(fields: [tenantId], references: [id])
  projects Project[] @relation("ProjectCreator")

  @@unique([tenantId, email])
  @@map("users")
}

model Subscription {
  id                   String    @id @default(uuid())
  tenantId             String    @unique
  stripeCustomerId     String?   @unique
  stripeSubscriptionId String?   @unique
  plan                 String    @default("free")
  status               String    @default("active")
  currentPeriodStart   DateTime?
  currentPeriodEnd     DateTime?
  cancelAtPeriodEnd    Boolean   @default(false)

  tenant Tenant @relation(fields: [tenantId], references: [id])

  @@map("subscriptions")
}
```

### Tenant Context Middleware (Nuxt)

```typescript
// server/middleware/tenant.ts
import { prisma } from '~/server/lib/prisma'

export default defineEventHandler(async (event) => {
  // ดึง tenant จาก subdomain หรือ header
  const host = getHeader(event, 'host') || ''
  const subdomain = host.split('.')[0]
  
  // หรือจาก custom header
  const tenantSlug = getHeader(event, 'x-tenant-slug') || subdomain

  if (!tenantSlug || tenantSlug === 'www' || tenantSlug === 'app') {
    return // ไม่ต้องหา tenant สำหรับ main domain
  }

  const tenant = await prisma.tenant.findUnique({
    where: { slug: tenantSlug, status: 'active' }
  })

  if (!tenant) {
    throw createError({
      statusCode: 404,
      message: 'Tenant not found'
    })
  }

  // เก็บ tenant ใน event context
  event.context.tenant = tenant
  event.context.tenantId = tenant.id
})
```

### Tenant-aware Repository Pattern

```typescript
// server/repositories/base.repository.ts
import { PrismaClient } from '@prisma/client'

export abstract class BaseRepository<T> {
  constructor(
    protected prisma: PrismaClient,
    protected tenantId: string
  ) {}

  // ทุก query ต้องมี tenantId
  protected withTenant(data: any) {
    return { ...data, tenantId: this.tenantId }
  }

  protected tenantWhere(where: any = {}) {
    return { ...where, tenantId: this.tenantId }
  }
}

// server/repositories/project.repository.ts
import { BaseRepository } from './base.repository'

export class ProjectRepository extends BaseRepository<any> {
  async findAll(filters: { status?: string } = {}) {
    return this.prisma.project.findMany({
      where: this.tenantWhere(filters),
      orderBy: { createdAt: 'desc' }
    })
  }

  async findById(id: string) {
    return this.prisma.project.findFirst({
      where: this.tenantWhere({ id })
    })
  }

  async create(data: { name: string; description?: string; createdBy: string }) {
    return this.prisma.project.create({
      data: this.withTenant(data)
    })
  }

  async update(id: string, data: Partial<{ name: string; description: string }>) {
    // ตรวจสอบว่า project เป็นของ tenant นี้ก่อน
    const project = await this.findById(id)
    if (!project) throw new Error('Project not found')

    return this.prisma.project.update({
      where: { id },
      data
    })
  }

  async delete(id: string) {
    const project = await this.findById(id)
    if (!project) throw new Error('Project not found')

    return this.prisma.project.delete({ where: { id } })
  }
}
```

---

## 3. Subscription Management กับ Stripe

### Plan Configuration

```typescript
// config/plans.ts
export const PLANS = {
  free: {
    name: 'Free',
    price: 0,
    stripePriceId: null,
    limits: {
      projects: 3,
      users: 1,
      storage: 1024 * 1024 * 100, // 100MB
      apiCalls: 1000
    },
    features: {
      customDomain: false,
      analytics: false,
      priority_support: false,
      white_label: false
    }
  },
  starter: {
    name: 'Starter',
    price: 29,
    stripePriceId: process.env.STRIPE_STARTER_PRICE_ID,
    limits: {
      projects: 10,
      users: 5,
      storage: 1024 * 1024 * 1024, // 1GB
      apiCalls: 10000
    },
    features: {
      customDomain: true,
      analytics: true,
      priority_support: false,
      white_label: false
    }
  },
  pro: {
    name: 'Pro',
    price: 99,
    stripePriceId: process.env.STRIPE_PRO_PRICE_ID,
    limits: {
      projects: 50,
      users: 20,
      storage: 1024 * 1024 * 1024 * 10, // 10GB
      apiCalls: 100000
    },
    features: {
      customDomain: true,
      analytics: true,
      priority_support: true,
      white_label: false
    }
  },
  enterprise: {
    name: 'Enterprise',
    price: -1, // Custom pricing
    stripePriceId: null,
    limits: {
      projects: Infinity,
      users: Infinity,
      storage: Infinity,
      apiCalls: Infinity
    },
    features: {
      customDomain: true,
      analytics: true,
      priority_support: true,
      white_label: true
    }
  }
} as const

export type PlanName = keyof typeof PLANS
```

### Stripe Integration

```typescript
// server/lib/stripe.ts
import Stripe from 'stripe'

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16'
})

// server/api/billing/create-checkout.post.ts
export default defineEventHandler(async (event) => {
  const { planName, successUrl, cancelUrl } = await readBody(event)
  const tenantId = event.context.tenantId
  const userId = event.context.userId

  const plan = PLANS[planName as PlanName]
  if (!plan || !plan.stripePriceId) {
    throw createError({ statusCode: 400, message: 'Invalid plan' })
  }

  // ดึงหรือสร้าง Stripe Customer
  let subscription = await prisma.subscription.findUnique({
    where: { tenantId },
    include: { tenant: true }
  })

  let customerId = subscription?.stripeCustomerId

  if (!customerId) {
    const customer = await stripe.customers.create({
      email: event.context.user.email,
      name: subscription?.tenant.name,
      metadata: { tenantId }
    })
    customerId = customer.id

    await prisma.subscription.upsert({
      where: { tenantId },
      update: { stripeCustomerId: customerId },
      create: { tenantId, stripeCustomerId: customerId }
    })
  }

  // สร้าง Checkout Session
  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    payment_method_types: ['card'],
    mode: 'subscription',
    line_items: [{ price: plan.stripePriceId, quantity: 1 }],
    success_url: successUrl,
    cancel_url: cancelUrl,
    metadata: { tenantId, planName }
  })

  return { url: session.url }
})

// server/api/billing/webhook.post.ts
export default defineEventHandler(async (event) => {
  const body = await readRawBody(event)
  const signature = getHeader(event, 'stripe-signature')!

  let stripeEvent: Stripe.Event

  try {
    stripeEvent = stripe.webhooks.constructEvent(
      body!,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch {
    throw createError({ statusCode: 400, message: 'Invalid signature' })
  }

  switch (stripeEvent.type) {
    case 'checkout.session.completed': {
      const session = stripeEvent.data.object as Stripe.CheckoutSession
      await handleCheckoutCompleted(session)
      break
    }
    case 'customer.subscription.updated': {
      const sub = stripeEvent.data.object as Stripe.Subscription
      await handleSubscriptionUpdated(sub)
      break
    }
    case 'customer.subscription.deleted': {
      const sub = stripeEvent.data.object as Stripe.Subscription
      await handleSubscriptionDeleted(sub)
      break
    }
    case 'invoice.payment_failed': {
      const invoice = stripeEvent.data.object as Stripe.Invoice
      await handlePaymentFailed(invoice)
      break
    }
  }

  return { received: true }
})

async function handleCheckoutCompleted(session: Stripe.CheckoutSession) {
  const tenantId = session.metadata?.tenantId
  const planName = session.metadata?.planName

  if (!tenantId || !planName) return

  const stripeSubscription = await stripe.subscriptions.retrieve(
    session.subscription as string
  )

  await prisma.subscription.update({
    where: { tenantId },
    data: {
      stripeSubscriptionId: stripeSubscription.id,
      plan: planName,
      status: 'active',
      currentPeriodStart: new Date(stripeSubscription.current_period_start * 1000),
      currentPeriodEnd: new Date(stripeSubscription.current_period_end * 1000)
    }
  })

  await prisma.tenant.update({
    where: { id: tenantId },
    data: { plan: planName }
  })
}
```

---

## 4. Usage Tracking

### Usage Service

```typescript
// server/services/usage.service.ts
import { redis } from '~/server/lib/redis'
import { prisma } from '~/server/lib/prisma'
import { PLANS } from '~/config/plans'

export class UsageService {
  private redisKey(tenantId: string, metric: string) {
    const month = new Date().toISOString().slice(0, 7)
    return `usage:${tenantId}:${month}:${metric}`
  }

  async increment(tenantId: string, metric: string, amount = 1) {
    const key = this.redisKey(tenantId, metric)
    const current = await redis.incrby(key, amount)
    
    // Set expiry ถ้ายังไม่มี (หมดอายุหลัง 35 วัน)
    const ttl = await redis.ttl(key)
    if (ttl === -1) {
      await redis.expire(key, 35 * 24 * 60 * 60)
    }

    return current
  }

  async get(tenantId: string, metric: string) {
    const key = this.redisKey(tenantId, metric)
    const value = await redis.get(key)
    return parseInt(value || '0', 10)
  }

  async getAll(tenantId: string) {
    const metrics = ['apiCalls', 'storage', 'projects', 'users']
    const values = await Promise.all(
      metrics.map(m => this.get(tenantId, m))
    )
    
    return metrics.reduce((acc, metric, i) => {
      acc[metric] = values[i]
      return acc
    }, {} as Record<string, number>)
  }

  async checkLimit(tenantId: string, metric: keyof typeof PLANS['free']['limits']) {
    const tenant = await prisma.tenant.findUnique({ where: { id: tenantId } })
    if (!tenant) throw new Error('Tenant not found')

    const plan = PLANS[tenant.plan as keyof typeof PLANS]
    const limit = plan.limits[metric]
    
    if (limit === Infinity) return true // ไม่มี limit

    const current = await this.get(tenantId, metric)
    
    if (current >= limit) {
      throw createError({
        statusCode: 429,
        message: `Limit exceeded for ${metric}. Current: ${current}, Limit: ${limit}. Please upgrade your plan.`
      })
    }

    return true
  }

  async getUsageStats(tenantId: string) {
    const tenant = await prisma.tenant.findUnique({
      where: { id: tenantId },
      include: { subscription: true }
    })

    if (!tenant) throw new Error('Tenant not found')

    const plan = PLANS[tenant.plan as keyof typeof PLANS]
    const usage = await this.getAll(tenantId)

    return {
      plan: tenant.plan,
      usage,
      limits: plan.limits,
      percentages: Object.entries(plan.limits).reduce((acc, [key, limit]) => {
        acc[key] = limit === Infinity ? 0 : Math.round((usage[key] || 0) / limit * 100)
        return acc
      }, {} as Record<string, number>)
    }
  }
}

export const usageService = new UsageService()
```

### Usage Middleware

```typescript
// server/middleware/usage-tracking.ts
export default defineEventHandler(async (event) => {
  const tenantId = event.context.tenantId
  if (!tenantId) return

  // นับ API calls
  if (getRequestURL(event).pathname.startsWith('/api/')) {
    await usageService.increment(tenantId, 'apiCalls')
  }
})
```

---

## 5. Feature Flags per Plan

### Feature Guard Composable

```typescript
// composables/usePlanFeatures.ts
import { PLANS, type PlanName } from '~/config/plans'

export function usePlanFeatures() {
  const { data: tenantData } = useFetch('/api/tenant/current')
  
  const currentPlan = computed(() => 
    tenantData.value?.plan as PlanName || 'free'
  )

  const planConfig = computed(() => PLANS[currentPlan.value])

  const hasFeature = (feature: keyof typeof PLANS['free']['features']) => {
    return planConfig.value?.features[feature] ?? false
  }

  const canUse = (resource: keyof typeof PLANS['free']['limits'], current: number) => {
    const limit = planConfig.value?.limits[resource]
    if (limit === undefined) return false
    if (limit === Infinity) return true
    return current < limit
  }

  const requiresUpgrade = (feature: keyof typeof PLANS['free']['features']) => {
    return !hasFeature(feature)
  }

  const suggestUpgrade = (feature: keyof typeof PLANS['free']['features']) => {
    // หา plan ที่ถูกสุดที่มี feature นี้
    const plans = Object.entries(PLANS) as [PlanName, typeof PLANS['free']][]
    const eligiblePlan = plans.find(
      ([, config]) => config.features[feature] && config.price > (planConfig.value?.price || 0)
    )
    return eligiblePlan ? eligiblePlan[0] : 'enterprise'
  }

  return {
    currentPlan,
    planConfig,
    hasFeature,
    canUse,
    requiresUpgrade,
    suggestUpgrade
  }
}
```

### Feature Gate Component

```vue
<!-- components/FeatureGate.vue -->
<template>
  <slot v-if="hasAccess" />
  <slot v-else name="fallback">
    <div class="feature-gate">
      <div class="upgrade-prompt">
        <Icon name="lock" />
        <h3>{{ title }}</h3>
        <p>{{ description }}</p>
        <NuxtLink :to="`/billing/upgrade?plan=${suggestedPlan}`" class="upgrade-btn">
          Upgrade to {{ suggestedPlan }}
        </NuxtLink>
      </div>
    </div>
  </slot>
</template>

<script setup lang="ts">
import type { PLANS } from '~/config/plans'

const props = defineProps<{
  feature: keyof typeof PLANS['free']['features']
  title?: string
  description?: string
}>()

const { hasFeature, suggestUpgrade } = usePlanFeatures()

const hasAccess = computed(() => hasFeature(props.feature))
const suggestedPlan = computed(() => suggestUpgrade(props.feature))
</script>
```

---

## 6. Onboarding Flow

### Onboarding Store

```typescript
// stores/onboarding.ts
import { defineStore } from 'pinia'

interface OnboardingStep {
  id: string
  title: string
  description: string
  completed: boolean
  required: boolean
}

export const useOnboardingStore = defineStore('onboarding', () => {
  const steps = ref<OnboardingStep[]>([
    {
      id: 'profile',
      title: 'Complete Your Profile',
      description: 'Add your name and avatar',
      completed: false,
      required: true
    },
    {
      id: 'team',
      title: 'Invite Team Members',
      description: 'Add your colleagues to the workspace',
      completed: false,
      required: false
    },
    {
      id: 'first-project',
      title: 'Create Your First Project',
      description: 'Start with a new project',
      completed: false,
      required: true
    },
    {
      id: 'integration',
      title: 'Connect an Integration',
      description: 'Connect GitHub, Slack, or other tools',
      completed: false,
      required: false
    }
  ])

  const currentStepIndex = ref(0)
  const isComplete = computed(() => 
    steps.value.filter(s => s.required).every(s => s.completed)
  )
  const progress = computed(() => {
    const completed = steps.value.filter(s => s.completed).length
    return Math.round(completed / steps.value.length * 100)
  })

  async function completeStep(stepId: string) {
    const step = steps.value.find(s => s.id === stepId)
    if (step) step.completed = true

    // บันทึกลง backend
    await $fetch('/api/onboarding/complete-step', {
      method: 'POST',
      body: { stepId }
    })

    // ย้ายไปขั้นตอนถัดไป
    const nextIncomplete = steps.value.findIndex(s => !s.completed)
    if (nextIncomplete !== -1) {
      currentStepIndex.value = nextIncomplete
    }
  }

  async function loadProgress() {
    const data = await $fetch<{ completedSteps: string[] }>('/api/onboarding/progress')
    data.completedSteps.forEach(stepId => {
      const step = steps.value.find(s => s.id === stepId)
      if (step) step.completed = true
    })
  }

  return {
    steps,
    currentStepIndex,
    isComplete,
    progress,
    completeStep,
    loadProgress
  }
})
```

### Onboarding Page

```vue
<!-- pages/onboarding/index.vue -->
<template>
  <div class="onboarding-page">
    <div class="onboarding-header">
      <h1>Welcome to {{ appName }}! 👋</h1>
      <p>Let's get you set up. This will take just a few minutes.</p>
      
      <div class="progress-bar">
        <div class="progress-fill" :style="{ width: `${progress}%` }"></div>
        <span>{{ progress }}% Complete</span>
      </div>
    </div>

    <div class="steps-grid">
      <div
        v-for="(step, index) in steps"
        :key="step.id"
        :class="['step-card', { 
          completed: step.completed, 
          active: index === currentStepIndex,
          required: step.required
        }]"
        @click="!step.completed && navigateTo(`/onboarding/${step.id}`)"
      >
        <div class="step-icon">
          <Icon v-if="step.completed" name="check-circle" class="text-green-500" />
          <Icon v-else-if="index === currentStepIndex" name="arrow-right-circle" class="text-blue-500" />
          <span v-else class="step-number">{{ index + 1 }}</span>
        </div>
        
        <div class="step-content">
          <h3>{{ step.title }}
            <span v-if="!step.required" class="optional-badge">Optional</span>
          </h3>
          <p>{{ step.description }}</p>
        </div>
      </div>
    </div>

    <div v-if="isComplete" class="completion-banner">
      <Icon name="party-popper" />
      <h2>You're all set!</h2>
      <NuxtLink to="/dashboard" class="go-to-dashboard">
        Go to Dashboard →
      </NuxtLink>
    </div>
  </div>
</template>

<script setup lang="ts">
definePageMeta({ layout: 'onboarding' })

const appName = useRuntimeConfig().public.appName
const store = useOnboardingStore()
const { steps, currentStepIndex, isComplete, progress } = storeToRefs(store)

onMounted(() => store.loadProgress())
</script>
```

---

## 7. ตัวอย่าง: SaaS App Architecture สมบูรณ์

### Project Structure

```
saas-app/
├── nuxt.config.ts
├── prisma/
│   └── schema.prisma
├── server/
│   ├── middleware/
│   │   ├── tenant.ts          # Multi-tenancy
│   │   ├── auth.ts            # Authentication
│   │   └── usage-tracking.ts  # Usage metrics
│   ├── api/
│   │   ├── tenant/
│   │   │   ├── current.get.ts
│   │   │   └── settings.put.ts
│   │   ├── billing/
│   │   │   ├── create-checkout.post.ts
│   │   │   ├── portal.post.ts
│   │   │   └── webhook.post.ts
│   │   ├── users/
│   │   │   ├── index.get.ts
│   │   │   ├── invite.post.ts
│   │   │   └── [id].delete.ts
│   │   └── projects/
│   │       ├── index.get.ts
│   │       ├── index.post.ts
│   │       └── [id].ts
│   ├── repositories/
│   │   ├── base.repository.ts
│   │   └── project.repository.ts
│   ├── services/
│   │   ├── usage.service.ts
│   │   └── email.service.ts
│   └── lib/
│       ├── prisma.ts
│       ├── redis.ts
│       └── stripe.ts
├── stores/
│   ├── auth.ts
│   ├── tenant.ts
│   └── onboarding.ts
├── composables/
│   ├── usePlanFeatures.ts
│   └── useUsage.ts
├── components/
│   ├── billing/
│   ├── onboarding/
│   └── shared/
│       └── FeatureGate.vue
└── pages/
    ├── dashboard/
    ├── billing/
    ├── onboarding/
    └── settings/
```

### Nuxt Config สำหรับ SaaS

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@nuxtjs/prisma',
    '@pinia/nuxt',
    'nuxt-stripe',
    '@nuxtjs/tailwindcss'
  ],

  runtimeConfig: {
    // Private (server-side only)
    stripeSecretKey: process.env.STRIPE_SECRET_KEY,
    stripeWebhookSecret: process.env.STRIPE_WEBHOOK_SECRET,
    jwtSecret: process.env.JWT_SECRET,
    redisUrl: process.env.REDIS_URL,

    // Public (client + server)
    public: {
      appName: process.env.APP_NAME || 'SaaS App',
      stripePublishableKey: process.env.STRIPE_PUBLISHABLE_KEY,
      appUrl: process.env.APP_URL || 'http://localhost:3000'
    }
  },

  // Rate limiting
  nitro: {
    routeRules: {
      '/api/**': {
        headers: {
          'X-Content-Type-Options': 'nosniff',
          'X-Frame-Options': 'DENY',
          'X-XSS-Protection': '1; mode=block'
        }
      }
    }
  }
})
```

### Billing Portal Component

```vue
<!-- components/billing/BillingPortal.vue -->
<template>
  <div class="billing-portal">
    <div class="current-plan">
      <h2>Current Plan: <span class="plan-badge">{{ currentPlan }}</span></h2>
      <p class="renewal-date">
        Renews on {{ formatDate(subscription?.currentPeriodEnd) }}
      </p>
    </div>

    <div class="usage-overview">
      <h3>Usage This Month</h3>
      <div class="usage-meters">
        <UsageMeter
          v-for="(value, metric) in usageStats?.percentages"
          :key="metric"
          :label="formatMetricLabel(metric)"
          :current="usageStats?.usage[metric]"
          :limit="usageStats?.limits[metric]"
          :percentage="value"
        />
      </div>
    </div>

    <div class="plan-cards">
      <PlanCard
        v-for="(plan, name) in PLANS"
        :key="name"
        :plan-name="name"
        :plan="plan"
        :is-current="name === currentPlan"
        @select="selectPlan(name)"
      />
    </div>

    <div class="billing-actions">
      <button @click="openBillingPortal" class="btn-secondary">
        Manage Payment Methods
      </button>
      <button v-if="subscription" @click="cancelSubscription" class="btn-danger">
        Cancel Subscription
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { PLANS, type PlanName } from '~/config/plans'

const { currentPlan } = usePlanFeatures()
const { data: subscription } = useFetch('/api/billing/subscription')
const { data: usageStats } = useFetch('/api/billing/usage')

async function selectPlan(planName: PlanName) {
  const { url } = await $fetch('/api/billing/create-checkout', {
    method: 'POST',
    body: {
      planName,
      successUrl: `${window.location.origin}/billing/success`,
      cancelUrl: `${window.location.origin}/billing`
    }
  })
  navigateTo(url, { external: true })
}

async function openBillingPortal() {
  const { url } = await $fetch('/api/billing/portal', { method: 'POST' })
  navigateTo(url, { external: true })
}

function formatDate(date?: string) {
  if (!date) return 'N/A'
  return new Intl.DateTimeFormat('th-TH').format(new Date(date))
}

function formatMetricLabel(metric: string) {
  const labels: Record<string, string> = {
    apiCalls: 'API Calls',
    storage: 'Storage',
    projects: 'Projects',
    users: 'Team Members'
  }
  return labels[metric] || metric
}
</script>
```

---

## สรุป

การสร้าง SaaS Product ที่ดีต้องคำนึงถึง:

1. **Multi-tenancy** - แยกข้อมูลของแต่ละ tenant อย่างชัดเจน
2. **Subscription Management** - ใช้ Stripe สำหรับจัดการ billing
3. **Usage Tracking** - ติดตาม usage ด้วย Redis สำหรับ performance
4. **Feature Flags** - ควบคุม feature access ตาม plan
5. **Onboarding** - ช่วยให้ผู้ใช้ใหม่เริ่มต้นได้ง่าย
6. **Security** - ตรวจสอบ tenantId ทุก query เสมอ
