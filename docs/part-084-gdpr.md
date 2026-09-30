# Part 84: GDPR Compliance

## GDPR คืออะไร?

GDPR (General Data Protection Regulation) คือกฎหมายคุ้มครองข้อมูลส่วนบุคคลของ EU ที่บังคับใช้ตั้งแต่ปี 2018 สำหรับ application ที่มีผู้ใช้จาก EU ต้องปฏิบัติตาม GDPR หรือเสี่ยงถูกปรับสูงถึง 4% ของรายได้ทั่วโลก

---

## 1. GDPR Overview

### สิทธิ์ของ Data Subject (ผู้ใช้)

```
1. Right to be informed    - รู้ว่าเก็บข้อมูลอะไร
2. Right of access         - ขอดูข้อมูลของตัวเองได้
3. Right to rectification  - แก้ไขข้อมูลได้
4. Right to erasure        - ลบข้อมูลได้ (Right to be forgotten)
5. Right to data portability - ขอ export ข้อมูลได้
6. Right to object         - คัดค้านการประมวลผล
```

---

## 2. Cookie Consent

### Cookie Categories

```typescript
// types/cookies.ts
export type CookieCategory = 
  | 'necessary'   // จำเป็น (ไม่ต้องขอ consent)
  | 'functional'  // ช่วยให้ site ทำงานได้ดีขึ้น
  | 'analytics'   // วัดผล traffic
  | 'marketing'   // โฆษณา

export interface CookieConsent {
  necessary: true  // Always true
  functional: boolean
  analytics: boolean
  marketing: boolean
  timestamp: string
  version: string
}
```

### Cookie Consent Store

```typescript
// stores/cookie-consent.ts
import { defineStore } from 'pinia'

export const useCookieConsentStore = defineStore('cookie-consent', () => {
  const consent = useLocalStorage<CookieConsent | null>('cookie_consent', null)
  const showBanner = ref(false)

  const hasConsented = computed(() => consent.value !== null)

  const isAllowed = (category: CookieCategory): boolean => {
    if (!consent.value) return category === 'necessary'
    return consent.value[category]
  }

  function accept(categories: Partial<Omit<CookieConsent, 'necessary' | 'timestamp' | 'version'>>) {
    consent.value = {
      necessary: true,
      functional: categories.functional ?? false,
      analytics: categories.analytics ?? false,
      marketing: categories.marketing ?? false,
      timestamp: new Date().toISOString(),
      version: '1.0'
    }
    showBanner.value = false

    // Reload analytics if consented
    if (categories.analytics) {
      initAnalytics()
    }

    // Save to backend
    $fetch('/api/privacy/consent', {
      method: 'POST',
      body: consent.value
    })
  }

  function acceptAll() {
    accept({ functional: true, analytics: true, marketing: true })
  }

  function acceptNecessaryOnly() {
    accept({ functional: false, analytics: false, marketing: false })
  }

  function revokeConsent() {
    consent.value = null
    showBanner.value = true
    // Remove analytics cookies, etc.
    clearNonNecessaryCookies()
  }

  function checkAndShowBanner() {
    if (!hasConsented.value) {
      showBanner.value = true
    } else {
      // ตรวจสอบว่า consent เก่าเกิน 1 ปีหรือไม่
      const consentDate = new Date(consent.value!.timestamp)
      const oneYearAgo = new Date()
      oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1)

      if (consentDate < oneYearAgo) {
        consent.value = null
        showBanner.value = true
      }
    }
  }

  function clearNonNecessaryCookies() {
    const cookies = document.cookie.split(';')
    const necessaryCookies = ['session', 'csrf_token', 'auth_token']

    cookies.forEach(cookie => {
      const name = cookie.split('=')[0].trim()
      if (!necessaryCookies.some(nc => name.includes(nc))) {
        document.cookie = `${name}=;expires=Thu, 01 Jan 1970 00:00:00 GMT;path=/`
      }
    })
  }

  return {
    consent,
    showBanner,
    hasConsented,
    isAllowed,
    accept,
    acceptAll,
    acceptNecessaryOnly,
    revokeConsent,
    checkAndShowBanner
  }
})
```

### Cookie Banner Component

```vue
<!-- components/CookieBanner.vue -->
<template>
  <Transition name="slide-up">
    <div v-if="showBanner" class="cookie-banner" role="dialog" aria-label="Cookie Consent">
      <div class="cookie-content">
        <h3>We Value Your Privacy</h3>
        <p>
          We use cookies to enhance your experience. By clicking "Accept All", 
          you consent to our use of cookies.
          <NuxtLink to="/privacy-policy">Learn more</NuxtLink>
        </p>
        
        <div v-if="showDetails" class="cookie-categories">
          <div v-for="category in categories" :key="category.id" class="category">
            <div class="category-header">
              <label>
                <input
                  v-if="category.id !== 'necessary'"
                  type="checkbox"
                  v-model="preferences[category.id]"
                />
                <input v-else type="checkbox" checked disabled />
                <strong>{{ category.name }}</strong>
              </label>
            </div>
            <p>{{ category.description }}</p>
          </div>
        </div>

        <div class="cookie-actions">
          <button @click="acceptNecessaryOnly" class="btn-secondary">
            Necessary Only
          </button>
          <button @click="showDetails = !showDetails" class="btn-outline">
            {{ showDetails ? 'Hide Details' : 'Manage Preferences' }}
          </button>
          <button @click="savePreferences" class="btn-primary" v-if="showDetails">
            Save Preferences
          </button>
          <button @click="acceptAll" class="btn-primary" v-else>
            Accept All
          </button>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup lang="ts">
const store = useCookieConsentStore()
const { showBanner, acceptAll, acceptNecessaryOnly } = storeToRefs(store)

const showDetails = ref(false)
const preferences = reactive({
  functional: false,
  analytics: false,
  marketing: false
})

const categories = [
  {
    id: 'necessary',
    name: 'Necessary',
    description: 'Required for the website to function properly'
  },
  {
    id: 'functional',
    name: 'Functional',
    description: 'Enables enhanced features like saved preferences'
  },
  {
    id: 'analytics',
    name: 'Analytics',
    description: 'Helps us understand how visitors use our site'
  },
  {
    id: 'marketing',
    name: 'Marketing',
    description: 'Used for targeted advertising and remarketing'
  }
]

function savePreferences() {
  store.accept(preferences)
}
</script>
```

---

## 3. Right to be Forgotten

### Delete User Data API

```typescript
// server/api/privacy/delete-account.delete.ts
export default defineEventHandler(async (event) => {
  await requireAuth(event)

  const userId = event.context.userId
  const tenantId = event.context.tenantId

  // สร้าง deletion job (ทำแบบ async)
  const job = await prisma.dataDeletionRequest.create({
    data: {
      userId,
      tenantId,
      status: 'pending',
      requestedAt: new Date(),
      scheduledAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) // 30 วัน grace period
    }
  })

  // แจ้ง user
  await emailService.sendAccountDeletionScheduled(userId, job.scheduledAt)

  return {
    message: 'Account deletion scheduled. You have 30 days to cancel.',
    scheduledAt: job.scheduledAt,
    cancellationCode: job.id
  }
})

// server/jobs/process-deletions.ts (cron job)
export async function processDeletions() {
  const pendingDeletions = await prisma.dataDeletionRequest.findMany({
    where: {
      status: 'pending',
      scheduledAt: { lte: new Date() }
    }
  })

  for (const request of pendingDeletions) {
    try {
      await deleteUserData(request.userId, request.tenantId)
      await prisma.dataDeletionRequest.update({
        where: { id: request.id },
        data: { status: 'completed', completedAt: new Date() }
      })
    } catch (error) {
      await prisma.dataDeletionRequest.update({
        where: { id: request.id },
        data: { status: 'failed', errorMessage: String(error) }
      })
    }
  }
}

async function deleteUserData(userId: string, tenantId: string) {
  // ลำดับการลบข้อมูล (ตาม foreign key constraints)
  await prisma.$transaction([
    // Anonymize แทนการลบ (เพื่อ maintain referential integrity)
    prisma.comment.updateMany({
      where: { authorId: userId },
      data: { 
        authorId: null, 
        content: '[Deleted by user request]'
      }
    }),
    prisma.post.updateMany({
      where: { authorId: userId },
      data: { authorId: null }
    }),
    // ลบข้อมูลส่วนตัว
    prisma.userProfile.deleteMany({ where: { userId } }),
    prisma.session.deleteMany({ where: { userId } }),
    // Anonymize user record
    prisma.user.update({
      where: { id: userId },
      data: {
        email: `deleted-${userId}@deleted.local`,
        name: '[Deleted User]',
        deletedAt: new Date()
      }
    })
  ])

  // ลบจาก external services
  await Promise.allSettled([
    analyticsService.deleteUser(userId),
    emailService.unsubscribeAll(userId),
    storageService.deleteUserFiles(userId, tenantId)
  ])
}
```

---

## 4. Data Export

### Export User Data API

```typescript
// server/api/privacy/export.get.ts
export default defineEventHandler(async (event) => {
  await requireAuth(event)
  const userId = event.context.userId

  // เก็บข้อมูลทั้งหมดของ user
  const [user, profile, posts, comments, orders] = await Promise.all([
    prisma.user.findUnique({ where: { id: userId } }),
    prisma.userProfile.findUnique({ where: { userId } }),
    prisma.post.findMany({ where: { authorId: userId } }),
    prisma.comment.findMany({ where: { authorId: userId } }),
    prisma.order.findMany({ where: { userId } })
  ])

  // สร้าง export package
  const exportData = {
    exportedAt: new Date().toISOString(),
    user: {
      id: user?.id,
      email: user?.email,
      name: user?.name,
      createdAt: user?.createdAt
    },
    profile,
    content: { posts, comments },
    orders,
    dataCategories: [
      'Account information',
      'Profile data',
      'Posts and comments',
      'Order history'
    ]
  }

  // ส่งเป็น JSON file
  setHeader(event, 'Content-Type', 'application/json')
  setHeader(event, 'Content-Disposition', 
    `attachment; filename="my-data-export-${new Date().toISOString().slice(0,10)}.json"`)

  return exportData
})
```

---

## 5. Audit Logs

### Audit Log Service

```typescript
// server/services/audit-log.service.ts
type AuditAction = 
  | 'USER_LOGIN' | 'USER_LOGOUT'
  | 'DATA_ACCESS' | 'DATA_CREATE' | 'DATA_UPDATE' | 'DATA_DELETE'
  | 'PRIVACY_CONSENT_GIVEN' | 'PRIVACY_CONSENT_REVOKED'
  | 'DATA_EXPORT_REQUESTED' | 'ACCOUNT_DELETION_REQUESTED'

export class AuditLogService {
  async log(params: {
    action: AuditAction
    userId?: string
    tenantId?: string
    resourceType?: string
    resourceId?: string
    changes?: Record<string, { before: unknown; after: unknown }>
    ipAddress?: string
    userAgent?: string
    metadata?: Record<string, unknown>
  }) {
    return prisma.auditLog.create({
      data: {
        action: params.action,
        userId: params.userId,
        tenantId: params.tenantId,
        resourceType: params.resourceType,
        resourceId: params.resourceId,
        changes: params.changes,
        ipAddress: params.ipAddress,
        userAgent: params.userAgent,
        metadata: params.metadata,
        timestamp: new Date()
      }
    })
  }
}

export const auditLog = new AuditLogService()
```

### Audit Middleware

```typescript
// server/middleware/audit.ts
export default defineEventHandler(async (event) => {
  const url = getRequestURL(event)
  const method = event.method

  // ตรวจสอบ actions ที่ต้อง audit
  if (method !== 'GET' && url.pathname.startsWith('/api/')) {
    const userId = event.context.userId
    const tenantId = event.context.tenantId

    const actionMap: Record<string, AuditAction> = {
      POST: 'DATA_CREATE',
      PUT: 'DATA_UPDATE',
      PATCH: 'DATA_UPDATE',
      DELETE: 'DATA_DELETE'
    }

    await auditLog.log({
      action: actionMap[method] || 'DATA_ACCESS',
      userId,
      tenantId,
      resourceType: url.pathname.split('/')[3], // /api/v2/users → 'users'
      ipAddress: getHeader(event, 'x-forwarded-for') || 'unknown',
      userAgent: getHeader(event, 'user-agent')
    })
  }
})
```

---

## 6. ตัวอย่าง: GDPR-compliant Nuxt App

### Privacy Settings Page

```vue
<!-- pages/privacy/settings.vue -->
<template>
  <div class="privacy-settings">
    <h1>Privacy Settings</h1>
    
    <!-- Data Overview -->
    <section class="data-overview">
      <h2>Your Data</h2>
      <p>Account created: {{ formatDate(user.createdAt) }}</p>
      <p>Last login: {{ formatDate(user.lastLoginAt) }}</p>
      
      <button @click="exportData" :loading="exporting">
        Export My Data
      </button>
    </section>

    <!-- Cookie Preferences -->
    <section class="cookie-preferences">
      <h2>Cookie Preferences</h2>
      <button @click="store.revokeConsent">
        Manage Cookies
      </button>
    </section>

    <!-- Communication Preferences -->
    <section class="communication-prefs">
      <h2>Email Preferences</h2>
      <div v-for="pref in emailPrefs" :key="pref.id">
        <label>
          <input type="checkbox" v-model="pref.enabled" @change="updatePref(pref)" />
          {{ pref.label }}
        </label>
        <p class="pref-description">{{ pref.description }}</p>
      </div>
    </section>

    <!-- Account Deletion -->
    <section class="danger-zone">
      <h2>Delete Account</h2>
      <p class="warning">
        This will permanently delete your account and all associated data.
        You have 30 days to cancel after requesting deletion.
      </p>
      <button @click="showDeleteModal = true" class="btn-danger">
        Request Account Deletion
      </button>
    </section>

    <!-- Delete Confirmation Modal -->
    <Modal v-model="showDeleteModal">
      <h3>Confirm Account Deletion</h3>
      <p>Please type <strong>DELETE</strong> to confirm:</p>
      <input v-model="deleteConfirmation" placeholder="Type DELETE" />
      <button 
        @click="deleteAccount"
        :disabled="deleteConfirmation !== 'DELETE'"
        class="btn-danger"
      >
        Delete My Account
      </button>
    </Modal>
  </div>
</template>

<script setup lang="ts">
definePageMeta({ middleware: 'auth' })

const { data: user } = useFetch('/api/user/me')
const store = useCookieConsentStore()
const showDeleteModal = ref(false)
const deleteConfirmation = ref('')
const exporting = ref(false)

async function exportData() {
  exporting.value = true
  try {
    const response = await $fetch('/api/privacy/export', { 
      responseType: 'blob' 
    })
    const url = URL.createObjectURL(response as Blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `my-data-${new Date().toISOString().slice(0,10)}.json`
    a.click()
  } finally {
    exporting.value = false
  }
}

async function deleteAccount() {
  await $fetch('/api/privacy/delete-account', { method: 'DELETE' })
  showDeleteModal.value = false
  await signOut()
  navigateTo('/goodbye')
}
</script>
```

---

## สรุป

GDPR Compliance ต้องการ:
1. **Cookie consent** - ขอ consent ก่อนใช้ non-essential cookies
2. **Privacy policy** - อธิบายว่าเก็บข้อมูลอะไร ทำไม
3. **Data access** - ให้ user ดูและ export ข้อมูลตัวเองได้
4. **Right to erasure** - ลบข้อมูลได้ (แต่ใช้ grace period)
5. **Audit logs** - บันทึกการเข้าถึงข้อมูล
6. **Data minimization** - เก็บเฉพาะที่จำเป็น
