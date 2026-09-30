# Part 73: Feature Flags

## Feature Flags คืออะไร?

Feature Flag (หรือ Feature Toggle) คือเทคนิคที่ช่วยให้เราเปิด/ปิด feature ของ application ได้โดยไม่ต้อง deploy code ใหม่ ทำให้ทีมสามารถ:
- ทดสอบ feature ใหม่กับผู้ใช้เฉพาะกลุ่ม
- Rollout feature แบบค่อยเป็นค่อยไป
- ปิด feature ที่มีปัญหาได้ทันที

---

## 1. ประเภทของ Feature Flags

```
Release Flags    - เปิด feature เมื่อพร้อม
Experiment Flags - A/B Testing
Ops Flags        - Emergency kill switch
Permission Flags - Feature per user/plan
```

---

## 2. สร้าง Feature Flag System เอง

### Database Schema

```typescript
// prisma/schema.prisma (เพิ่มใน schema)
model FeatureFlag {
  id          String   @id @default(uuid())
  key         String   @unique
  name        String
  description String?
  enabled     Boolean  @default(false)
  rollout     Float    @default(0) // 0-100 percentage
  conditions  Json     @default("[]")
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@map("feature_flags")
}
```

### Feature Flag Service

```typescript
// server/services/feature-flags.service.ts
import { prisma } from '~/server/lib/prisma'
import { redis } from '~/server/lib/redis'

interface FlagCondition {
  type: 'tenant' | 'user' | 'plan' | 'percentage'
  operator: 'in' | 'not_in' | 'equals' | 'less_than' | 'greater_than'
  value: string | string[] | number
}

interface EvaluationContext {
  tenantId?: string
  userId?: string
  plan?: string
  userHash?: number // 0-100 สำหรับ percentage rollout
}

export class FeatureFlagService {
  private cachePrefix = 'ff:'
  private cacheTTL = 60 // 60 seconds

  async getFlag(key: string) {
    // ลองดึงจาก cache ก่อน
    const cached = await redis.get(`${this.cachePrefix}${key}`)
    if (cached) return JSON.parse(cached)

    const flag = await prisma.featureFlag.findUnique({ where: { key } })
    if (flag) {
      await redis.setex(
        `${this.cachePrefix}${key}`,
        this.cacheTTL,
        JSON.stringify(flag)
      )
    }
    return flag
  }

  async isEnabled(key: string, context: EvaluationContext = {}): Promise<boolean> {
    const flag = await this.getFlag(key)
    if (!flag) return false
    if (!flag.enabled) return false

    const conditions: FlagCondition[] = flag.conditions

    // ถ้าไม่มี conditions ให้ใช้ rollout percentage
    if (conditions.length === 0) {
      return this.checkRollout(flag.rollout, context)
    }

    // ตรวจสอบทุก condition (AND logic)
    return conditions.every(condition => 
      this.evaluateCondition(condition, context)
    )
  }

  private checkRollout(rolloutPercentage: number, context: EvaluationContext): boolean {
    if (rolloutPercentage >= 100) return true
    if (rolloutPercentage <= 0) return false

    // ใช้ userId หรือ tenantId เพื่อให้ผู้ใช้คนเดิมได้ผลเหมือนเดิม
    const hash = this.hashString(context.userId || context.tenantId || Math.random().toString())
    return (hash % 100) < rolloutPercentage
  }

  private hashString(str: string): number {
    let hash = 0
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i)
      hash = hash & hash // Convert to 32bit integer
    }
    return Math.abs(hash)
  }

  private evaluateCondition(condition: FlagCondition, context: EvaluationContext): boolean {
    let value: string | number | undefined

    switch (condition.type) {
      case 'tenant': value = context.tenantId; break
      case 'user': value = context.userId; break
      case 'plan': value = context.plan; break
      case 'percentage':
        const hash = this.hashString(context.userId || context.tenantId || '')
        value = hash % 100
        break
    }

    if (value === undefined) return false

    switch (condition.operator) {
      case 'in':
        return Array.isArray(condition.value) && condition.value.includes(String(value))
      case 'not_in':
        return Array.isArray(condition.value) && !condition.value.includes(String(value))
      case 'equals':
        return String(value) === String(condition.value)
      case 'less_than':
        return Number(value) < Number(condition.value)
      case 'greater_than':
        return Number(value) > Number(condition.value)
      default:
        return false
    }
  }

  async getAllFlags(context: EvaluationContext = {}) {
    const flags = await prisma.featureFlag.findMany()
    const results: Record<string, boolean> = {}

    await Promise.all(
      flags.map(async (flag) => {
        results[flag.key] = await this.isEnabled(flag.key, context)
      })
    )

    return results
  }

  async invalidateCache(key: string) {
    await redis.del(`${this.cachePrefix}${key}`)
  }
}

export const featureFlagService = new FeatureFlagService()
```

### Feature Flags API

```typescript
// server/api/feature-flags/index.get.ts
export default defineEventHandler(async (event) => {
  const context = {
    tenantId: event.context.tenantId,
    userId: event.context.userId,
    plan: event.context.tenant?.plan
  }

  const flags = await featureFlagService.getAllFlags(context)
  return flags
})

// server/api/feature-flags/[key].get.ts
export default defineEventHandler(async (event) => {
  const key = getRouterParam(event, 'key')!
  const context = {
    tenantId: event.context.tenantId,
    userId: event.context.userId,
    plan: event.context.tenant?.plan
  }

  const enabled = await featureFlagService.isEnabled(key, context)
  return { key, enabled }
})
```

---

## 3. LaunchDarkly Integration

### Setup

```typescript
// server/lib/launchdarkly.ts
import * as ld from '@launchdarkly/node-server-sdk'

let ldClient: ld.LDClient | null = null

export async function getLDClient(): Promise<ld.LDClient> {
  if (!ldClient) {
    ldClient = ld.init(process.env.LAUNCHDARKLY_SDK_KEY!)
    await ldClient.waitForInitialization()
  }
  return ldClient
}

export async function getFlag<T = boolean>(
  flagKey: string,
  user: ld.LDUser,
  defaultValue: T
): Promise<T> {
  const client = await getLDClient()
  return client.variation(flagKey, user, defaultValue) as T
}
```

### LaunchDarkly Composable

```typescript
// composables/useLaunchDarkly.ts
import * as LDClient from 'launchdarkly-js-client-sdk'

let ldClient: LDClient.LDClient | null = null

export function useLaunchDarkly() {
  const { user } = useAuth()

  async function initialize() {
    if (ldClient) return

    const context: LDClient.LDContext = {
      kind: 'user',
      key: user.value?.id || 'anonymous',
      email: user.value?.email,
      custom: {
        tenantId: user.value?.tenantId,
        plan: user.value?.plan
      }
    }

    ldClient = LDClient.initialize(
      useRuntimeConfig().public.launchDarklyClientId,
      context
    )

    await ldClient.waitForInitialization()
  }

  async function getFlag<T = boolean>(flagKey: string, defaultValue: T): Promise<T> {
    await initialize()
    return ldClient!.variation(flagKey, defaultValue)
  }

  function useFlagRef<T = boolean>(flagKey: string, defaultValue: T) {
    const flagValue = ref<T>(defaultValue)

    onMounted(async () => {
      await initialize()
      flagValue.value = ldClient!.variation(flagKey, defaultValue)

      // Listen for changes
      ldClient!.on(`change:${flagKey}`, (newValue: T) => {
        flagValue.value = newValue
      })
    })

    onUnmounted(() => {
      ldClient?.off(`change:${flagKey}`)
    })

    return flagValue
  }

  return { getFlag, useFlagRef }
}
```

---

## 4. A/B Testing กับ Feature Flags

```typescript
// composables/useExperiment.ts
export function useExperiment(experimentKey: string) {
  const { data: variant } = useFetch(`/api/experiments/${experimentKey}/variant`)
  const { trackEvent } = useAnalytics()

  function trackExposure() {
    trackEvent('experiment_exposure', {
      experimentKey,
      variant: variant.value
    })
  }

  function trackConversion(goal: string) {
    trackEvent('experiment_conversion', {
      experimentKey,
      variant: variant.value,
      goal
    })
  }

  onMounted(trackExposure)

  return { variant, trackConversion }
}
```

---

## 5. Gradual Rollout

### Rollout Manager

```typescript
// server/services/rollout.service.ts
export class RolloutService {
  async setRollout(flagKey: string, percentage: number) {
    if (percentage < 0 || percentage > 100) {
      throw new Error('Percentage must be between 0 and 100')
    }

    await prisma.featureFlag.update({
      where: { key: flagKey },
      data: { rollout: percentage }
    })

    await featureFlagService.invalidateCache(flagKey)
  }

  async gradualRollout(
    flagKey: string,
    targetPercentage: number,
    options: {
      startPercentage?: number
      stepSize?: number
      intervalMs?: number
    } = {}
  ) {
    const { startPercentage = 0, stepSize = 10, intervalMs = 60000 } = options

    let current = startPercentage

    const rollout = async () => {
      if (current >= targetPercentage) return

      current = Math.min(current + stepSize, targetPercentage)
      await this.setRollout(flagKey, current)
      console.log(`[Rollout] ${flagKey}: ${current}%`)

      if (current < targetPercentage) {
        setTimeout(rollout, intervalMs)
      }
    }

    await rollout()
  }
}
```

---

## 6. ตัวอย่าง: Feature Flag Composable สมบูรณ์

```typescript
// composables/useFeatureFlags.ts
interface FeatureFlags {
  newDashboard: boolean
  betaEditor: boolean
  aiAssistant: boolean
  advancedAnalytics: boolean
}

export function useFeatureFlags() {
  const flags = useState<FeatureFlags>('feature_flags', () => ({
    newDashboard: false,
    betaEditor: false,
    aiAssistant: false,
    advancedAnalytics: false
  }))

  const loading = ref(false)

  async function loadFlags() {
    loading.value = true
    try {
      const data = await $fetch<FeatureFlags>('/api/feature-flags')
      Object.assign(flags.value, data)
    } finally {
      loading.value = false
    }
  }

  function isEnabled(key: keyof FeatureFlags): boolean {
    return flags.value[key] ?? false
  }

  return { flags, loading, loadFlags, isEnabled }
}
```

### FeatureFlag Component

```vue
<!-- components/FeatureFlag.vue -->
<template>
  <slot v-if="enabled" />
  <slot v-else name="fallback" />
</template>

<script setup lang="ts">
const props = defineProps<{
  flag: string
  inverted?: boolean
}>()

const { isEnabled } = useFeatureFlags()

const enabled = computed(() => {
  const value = isEnabled(props.flag as any)
  return props.inverted ? !value : value
})
</script>
```

### การใช้งาน

```vue
<template>
  <div>
    <!-- ใช้ component -->
    <FeatureFlag flag="newDashboard">
      <NewDashboard />
      <template #fallback>
        <OldDashboard />
      </template>
    </FeatureFlag>

    <!-- ใช้ composable โดยตรง -->
    <button v-if="flags.aiAssistant" @click="openAI">
      AI Assistant ✨
    </button>
  </div>
</template>

<script setup lang="ts">
const { flags } = useFeatureFlags()
</script>
```

---

## สรุป

Feature Flags ช่วยให้:
1. Deploy ได้บ่อยขึ้นโดยไม่ risk
2. Test feature กับ user จริงก่อน release
3. Rollback ได้ทันทีหากมีปัญหา
4. ทำ A/B Testing ได้ง่าย
