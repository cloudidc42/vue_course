# Part 74: A/B Testing

## A/B Testing คืออะไร?

A/B Testing (หรือ Split Testing) คือการทดสอบ 2 versions ของ UI/Feature กับผู้ใช้จริง เพื่อดูว่า version ไหน perform ดีกว่า โดยวัดจาก metric จริง เช่น Conversion Rate, Click-through Rate

---

## 1. A/B Testing Concepts

### วงจร A/B Testing

```
1. Define Hypothesis
   "เราคิดว่า CTA สีเขียวจะมี CTR สูงกว่าสีน้ำเงิน 15%"

2. Design Experiment
   - Control: CTA สีน้ำเงิน (version A)
   - Variant: CTA สีเขียว (version B)
   - Traffic split: 50/50

3. Run Experiment
   - ต้องรัน minimum 2 สัปดาห์
   - ต้องมี sample size เพียงพอ

4. Analyze Results
   - Statistical significance > 95%
   - Practical significance

5. Make Decision
   - Roll out winner
   - หรือ run follow-up test
```

---

## 2. Client-side A/B Testing

### Experiment Engine

```typescript
// composables/useABTest.ts

interface Experiment {
  id: string
  name: string
  variants: {
    id: string
    name: string
    weight: number // 0-100 (ต้องรวมเป็น 100)
  }[]
  isActive: boolean
}

interface Assignment {
  experimentId: string
  variantId: string
  assignedAt: string
}

export function useABTest(experimentId: string) {
  const assignments = useLocalStorage<Record<string, Assignment>>(
    'ab_assignments', 
    {}
  )

  const { data: experiment } = useFetch<Experiment>(
    `/api/experiments/${experimentId}`
  )

  const variant = computed(() => {
    if (!experiment.value?.isActive) return null

    // ตรวจสอบว่าเคย assign แล้วหรือยัง (sticky assignment)
    const existing = assignments.value[experimentId]
    if (existing) {
      return experiment.value.variants.find(v => v.id === existing.variantId)
    }

    // Random assignment ตาม weights
    const assigned = weightedRandom(experiment.value.variants)

    // บันทึก assignment
    assignments.value[experimentId] = {
      experimentId,
      variantId: assigned.id,
      assignedAt: new Date().toISOString()
    }

    // Track exposure
    trackExposure(experimentId, assigned.id)

    return assigned
  })

  function trackExposure(expId: string, variantId: string) {
    $fetch('/api/analytics/track', {
      method: 'POST',
      body: {
        event: 'experiment_exposure',
        properties: { experimentId: expId, variantId }
      }
    })
  }

  function trackConversion(goal: string, value?: number) {
    if (!variant.value) return

    $fetch('/api/analytics/track', {
      method: 'POST',
      body: {
        event: 'experiment_conversion',
        properties: {
          experimentId,
          variantId: variant.value.id,
          goal,
          value
        }
      }
    })
  }

  return { experiment, variant, trackConversion }
}

function weightedRandom<T extends { weight: number }>(items: T[]): T {
  const totalWeight = items.reduce((sum, item) => sum + item.weight, 0)
  let random = Math.random() * totalWeight

  for (const item of items) {
    random -= item.weight
    if (random <= 0) return item
  }

  return items[items.length - 1]
}
```

---

## 3. Server-side A/B Testing

### Server Experiment Handler

```typescript
// server/api/experiments/[id]/variant.get.ts
export default defineEventHandler(async (event) => {
  const experimentId = getRouterParam(event, 'id')!
  const userId = event.context.userId
  const tenantId = event.context.tenantId

  const experiment = await prisma.experiment.findUnique({
    where: { id: experimentId, isActive: true },
    include: { variants: true }
  })

  if (!experiment) {
    throw createError({ statusCode: 404, message: 'Experiment not found' })
  }

  // ตรวจสอบ existing assignment
  const existing = await prisma.experimentAssignment.findFirst({
    where: { experimentId, userId: userId || undefined }
  })

  if (existing) {
    const variant = experiment.variants.find(v => v.id === existing.variantId)
    return { variantId: existing.variantId, variant }
  }

  // Assign variant
  const variant = weightedRandom(experiment.variants)

  await prisma.experimentAssignment.create({
    data: {
      experimentId,
      variantId: variant.id,
      userId: userId || null,
      tenantId: tenantId || null
    }
  })

  // Track exposure
  await analyticsService.track({
    event: 'experiment_exposure',
    userId,
    properties: { experimentId, variantId: variant.id }
  })

  return { variantId: variant.id, variant }
})
```

---

## 4. Statistical Significance

### Calculator

```typescript
// utils/statistics.ts

export function calculateStatisticalSignificance(
  controlConversions: number,
  controlVisitors: number,
  variantConversions: number,
  variantVisitors: number
): {
  pValue: number
  significant: boolean
  confidence: number
  relativeUplift: number
} {
  const controlRate = controlConversions / controlVisitors
  const variantRate = variantConversions / variantVisitors

  // Z-test for proportions
  const pooledRate = (controlConversions + variantConversions) /
                     (controlVisitors + variantVisitors)

  const standardError = Math.sqrt(
    pooledRate * (1 - pooledRate) * (1/controlVisitors + 1/variantVisitors)
  )

  const zScore = Math.abs(variantRate - controlRate) / standardError

  // แปลง Z-score เป็น p-value (two-tailed)
  const pValue = 2 * (1 - normalCDF(Math.abs(zScore)))
  const confidence = (1 - pValue) * 100
  const relativeUplift = ((variantRate - controlRate) / controlRate) * 100

  return {
    pValue: Math.round(pValue * 10000) / 10000,
    significant: pValue < 0.05, // 95% confidence
    confidence: Math.round(confidence * 100) / 100,
    relativeUplift: Math.round(relativeUplift * 100) / 100
  }
}

// Normal CDF approximation
function normalCDF(x: number): number {
  const t = 1 / (1 + 0.2316419 * Math.abs(x))
  const d = 0.3989422820 * Math.exp(-x * x / 2)
  const p = d * t * (0.3193815 + t * (-0.3565638 + t * (1.7814779 + t * (-1.8212560 + t * 1.3302744))))
  return x > 0 ? 1 - p : p
}

export function calculateRequiredSampleSize(
  baselineConversionRate: number,
  minimumDetectableEffect: number, // relative effect e.g., 0.05 for 5%
  alpha = 0.05, // significance level
  beta = 0.20   // 1 - power
): number {
  const p1 = baselineConversionRate
  const p2 = p1 * (1 + minimumDetectableEffect)

  const zAlpha = 1.96 // z-score for alpha=0.05
  const zBeta = 0.84  // z-score for beta=0.20 (80% power)

  const numerator = Math.pow(zAlpha * Math.sqrt(2 * p1 * (1 - p1)) +
                             zBeta * Math.sqrt(p1 * (1 - p1) + p2 * (1 - p2)), 2)
  const denominator = Math.pow(p2 - p1, 2)

  return Math.ceil(numerator / denominator)
}
```

---

## 5. Analytics Integration

### Analytics Service

```typescript
// server/services/analytics.service.ts
export class AnalyticsService {
  async track(params: {
    event: string
    userId?: string
    tenantId?: string
    properties?: Record<string, unknown>
  }) {
    // บันทึกลง database
    await prisma.analyticsEvent.create({
      data: {
        event: params.event,
        userId: params.userId,
        tenantId: params.tenantId,
        properties: params.properties || {}
      }
    })

    // ส่งไปยัง external analytics (optional)
    if (process.env.MIXPANEL_TOKEN) {
      await this.sendToMixpanel(params)
    }
  }

  async getExperimentResults(experimentId: string) {
    const [exposures, conversions] = await Promise.all([
      prisma.analyticsEvent.groupBy({
        by: ['properties'],
        where: {
          event: 'experiment_exposure',
          properties: { path: ['experimentId'], equals: experimentId }
        },
        _count: true
      }),
      prisma.analyticsEvent.groupBy({
        by: ['properties'],
        where: {
          event: 'experiment_conversion',
          properties: { path: ['experimentId'], equals: experimentId }
        },
        _count: true
      })
    ])

    return { exposures, conversions }
  }
}
```

---

## 6. ตัวอย่าง: A/B Test Landing Page

```vue
<!-- pages/landing.vue -->
<template>
  <div>
    <!-- A/B Test Hero Section -->
    <section class="hero">
      <!-- Variant A: Blue CTA -->
      <template v-if="heroVariant?.id === 'control'">
        <h1>Build Better Products</h1>
        <p>The all-in-one platform for product teams</p>
        <button class="cta-blue" @click="handleCTA">
          Start Free Trial
        </button>
      </template>

      <!-- Variant B: Green CTA with urgency -->
      <template v-else-if="heroVariant?.id === 'variant_b'">
        <h1>🚀 Build Better Products Faster</h1>
        <p>Join 10,000+ teams shipping with confidence</p>
        <button class="cta-green" @click="handleCTA">
          Get Started Free — No Credit Card
        </button>
        <p class="urgency">⚡ Limited time: 3 months free</p>
      </template>
    </section>
  </div>
</template>

<script setup lang="ts">
const { variant: heroVariant, trackConversion } = useABTest('hero-cta-experiment')

function handleCTA() {
  trackConversion('cta_click')
  navigateTo('/signup')
}
</script>
```

### Results Dashboard Component

```vue
<!-- components/ExperimentResults.vue -->
<template>
  <div class="experiment-results">
    <h2>{{ experiment?.name }}</h2>
    
    <div class="variants-comparison">
      <div v-for="variant in variantResults" :key="variant.id" class="variant-card">
        <h3>{{ variant.name }}</h3>
        <div class="stats">
          <div class="stat">
            <label>Visitors</label>
            <value>{{ variant.visitors.toLocaleString() }}</value>
          </div>
          <div class="stat">
            <label>Conversions</label>
            <value>{{ variant.conversions.toLocaleString() }}</value>
          </div>
          <div class="stat">
            <label>Conversion Rate</label>
            <value>{{ (variant.conversionRate * 100).toFixed(2) }}%</value>
          </div>
        </div>
        
        <div v-if="variant.id !== 'control'" class="significance">
          <div :class="['badge', significance?.significant ? 'significant' : 'not-significant']">
            {{ significance?.significant ? '✓ Significant' : '⏳ Not Significant' }}
          </div>
          <p>Confidence: {{ significance?.confidence }}%</p>
          <p>Uplift: {{ significance?.relativeUplift > 0 ? '+' : '' }}{{ significance?.relativeUplift }}%</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { calculateStatisticalSignificance } from '~/utils/statistics'

const props = defineProps<{ experimentId: string }>()
const { data: results } = useFetch(`/api/experiments/${props.experimentId}/results`)

const variantResults = computed(() => results.value?.variants || [])

const significance = computed(() => {
  const control = variantResults.value.find(v => v.id === 'control')
  const variant = variantResults.value.find(v => v.id !== 'control')
  
  if (!control || !variant) return null
  
  return calculateStatisticalSignificance(
    control.conversions,
    control.visitors,
    variant.conversions,
    variant.visitors
  )
})
</script>
```

---

## สรุป

A/B Testing ที่ดีต้องการ:
1. **Clear hypothesis** - รู้ว่าต้องการทดสอบอะไร
2. **Sufficient sample size** - คำนวณ sample size ก่อนเริ่ม
3. **Statistical significance** - อย่าตัดสินใจโดยไม่มี data เพียงพอ
4. **Single variable** - เปลี่ยนทีละอย่าง
5. **Proper tracking** - วัด conversion ให้ถูกต้อง
