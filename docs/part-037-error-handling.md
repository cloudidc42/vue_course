# Part 37: Error Handling

## ทำไม Error Handling ถึงสำคัญ?

Error ที่ไม่ได้รับการจัดการอย่างถูกต้องสร้างปัญหาหลายอย่าง:
- **UX ที่แย่** - ผู้ใช้เห็น error message ที่ไม่เข้าใจ
- **Data loss** - โปรแกรมพังกลางคัน
- **Security risks** - Stack trace รั่วไหลสู่ผู้ใช้
- **Debugging ยาก** - ไม่รู้ว่า error เกิดจากอะไร

---

## 1. Error Handling Patterns ใน Vue

### Try-Catch ใน async functions

```ts
// composables/useApi.ts
export function useApi() {
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function execute<T>(fn: () => Promise<T>): Promise<T | null> {
    loading.value = true
    error.value = null

    try {
      return await fn()
    } catch (err) {
      if (err instanceof Error) {
        error.value = err
      } else {
        error.value = new Error('เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ')
      }
      return null
    } finally {
      loading.value = false
    }
  }

  return { loading, error, execute }
}
```

### Error States ใน Component

```vue
<!-- components/DataFetcher.vue -->
<script setup lang="ts">
type Status = 'idle' | 'loading' | 'success' | 'error'

interface State<T> {
  data: T | null
  status: Status
  error: string | null
}

const state = reactive<State<any>>({
  data: null,
  status: 'idle',
  error: null
})

async function fetchData() {
  state.status = 'loading'
  state.error = null

  try {
    const response = await $fetch('/api/data')
    state.data = response
    state.status = 'success'
  } catch (err: any) {
    state.status = 'error'
    state.error = err?.message ?? 'เกิดข้อผิดพลาด'
  }
}

onMounted(fetchData)
</script>

<template>
  <div>
    <div v-if="state.status === 'loading'" class="loading">
      กำลังโหลด...
    </div>
    <div v-else-if="state.status === 'error'" class="error">
      <p>{{ state.error }}</p>
      <button @click="fetchData">ลองอีกครั้ง</button>
    </div>
    <div v-else-if="state.status === 'success'">
      <slot :data="state.data" />
    </div>
  </div>
</template>
```

---

## 2. onErrorCaptured Hook

`onErrorCaptured` ดัก errors จาก child components ทั้งหมด

```vue
<!-- components/ErrorWrapper.vue -->
<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue'

interface CapturedError {
  message: string
  stack?: string
  component?: string
  info?: string
}

const capturedError = ref<CapturedError | null>(null)

// ดัก error จาก children
onErrorCaptured((err: Error, instance, info) => {
  capturedError.value = {
    message: err.message,
    stack: err.stack,
    component: instance?.$options?.name ?? 'Unknown',
    info
  }

  // log ไปยัง error tracking service
  console.error('[Error Captured]', {
    message: err.message,
    component: instance?.$options?.name,
    info
  })

  // return false เพื่อหยุดไม่ให้ error propagate ต่อ
  return false
})

function retry() {
  capturedError.value = null
}
</script>

<template>
  <div>
    <div v-if="capturedError" class="error-boundary">
      <h3>เกิดข้อผิดพลาด</h3>
      <p>{{ capturedError.message }}</p>
      <button @click="retry">ลองอีกครั้ง</button>
    </div>
    <slot v-else />
  </div>
</template>
```

---

## 3. Error Boundary Component

```vue
<!-- components/ErrorBoundary.vue - Reusable Error Boundary -->
<script setup lang="ts">
import { onErrorCaptured, ref, type Component } from 'vue'

interface Props {
  fallback?: Component
  onError?: (err: Error, info: string) => void
}

const props = defineProps<Props>()

const hasError = ref(false)
const errorInfo = ref<{ message: string; stack?: string } | null>(null)

onErrorCaptured((err: Error, instance, info) => {
  hasError.value = true
  errorInfo.value = {
    message: err.message,
    stack: import.meta.dev ? err.stack : undefined
  }

  // เรียก callback ถ้ามี
  props.onError?.(err, info)

  // ส่ง error ไป error tracking service
  reportError(err, { info })

  return false
})

function reset() {
  hasError.value = false
  errorInfo.value = null
}

// Error tracking function
function reportError(err: Error, context?: Record<string, unknown>) {
  // ส่งไป Sentry, LogRocket, หรือ custom service
  if (typeof window !== 'undefined' && (window as any).Sentry) {
    (window as any).Sentry.captureException(err, { extra: context })
  }
}
</script>

<template>
  <component
    v-if="hasError && fallback"
    :is="fallback"
    :error="errorInfo"
    @retry="reset"
  />
  <div v-else-if="hasError" class="error-boundary-default">
    <h2>เกิดข้อผิดพลาด</h2>
    <p>{{ errorInfo?.message }}</p>
    <pre v-if="errorInfo?.stack" class="stack-trace">
      {{ errorInfo.stack }}
    </pre>
    <button @click="reset">รีเซ็ต</button>
  </div>
  <slot v-else :reset="reset" />
</template>
```

```vue
<!-- การใช้งาน ErrorBoundary -->
<script setup lang="ts">
import ErrorFallback from '@/components/ErrorFallback.vue'

function handleError(err: Error, info: string) {
  console.error('Error caught:', err, info)
}
</script>

<template>
  <ErrorBoundary
    :fallback="ErrorFallback"
    :on-error="handleError"
  >
    <SomeRiskyComponent />
  </ErrorBoundary>
</template>
```

---

## 4. Nuxt Error Handling

```ts
// plugins/error-handler.ts - Global error handler สำหรับ Nuxt
export default defineNuxtPlugin((nuxtApp) => {
  // ดัก Vue errors
  nuxtApp.vueApp.config.errorHandler = (err, instance, info) => {
    console.error('[Vue Error]', err, info)
    // ส่งไป error tracking
  }

  // ดัก unhandled promise rejections
  nuxtApp.hook('app:error', (err) => {
    console.error('[App Error]', err)
  })
})
```

```vue
<!-- pages/error.vue - Custom Error Page -->
<script setup lang="ts">
import type { NuxtError } from '#app'

const props = defineProps<{
  error: NuxtError
}>()

const title = computed(() => {
  switch (props.error.statusCode) {
    case 404: return 'ไม่พบหน้าที่คุณต้องการ'
    case 403: return 'คุณไม่มีสิทธิ์เข้าถึง'
    case 500: return 'เซิร์ฟเวอร์เกิดข้อผิดพลาด'
    default: return `เกิดข้อผิดพลาด ${props.error.statusCode}`
  }
})

const description = computed(() => {
  switch (props.error.statusCode) {
    case 404: return 'หน้าที่คุณหาอาจถูกลบหรือ URL ไม่ถูกต้อง'
    case 403: return 'กรุณาเข้าสู่ระบบหรือติดต่อผู้ดูแล'
    case 500: return 'เกิดข้อผิดพลาดภายใน กรุณาลองอีกครั้ง'
    default: return props.error.message
  }
})

function handleError() {
  clearError({ redirect: '/' })
}
</script>

<template>
  <div class="error-page">
    <div class="error-content">
      <h1 class="error-code">{{ error.statusCode }}</h1>
      <h2>{{ title }}</h2>
      <p>{{ description }}</p>
      
      <div class="actions">
        <button @click="handleError">กลับหน้าหลัก</button>
        <button @click="$router.back()">กลับหน้าก่อน</button>
      </div>
    </div>
  </div>
</template>
```

---

## 5. createError() utility

```ts
// server/api/posts/[id].ts
export default defineEventHandler(async (event) => {
  const id = parseInt(getRouterParam(event, 'id') ?? '0')

  if (isNaN(id) || id <= 0) {
    throw createError({
      statusCode: 400,
      statusMessage: 'Bad Request',
      message: 'ID ต้องเป็นตัวเลขบวก',
      data: { field: 'id', received: getRouterParam(event, 'id') }
    })
  }

  const post = await db.posts.findUnique({ where: { id } })

  if (!post) {
    throw createError({
      statusCode: 404,
      statusMessage: 'Not Found',
      message: `ไม่พบบทความ ID: ${id}`
    })
  }

  if (!post.isPublished) {
    throw createError({
      statusCode: 403,
      statusMessage: 'Forbidden',
      message: 'บทความนี้ยังไม่ได้เผยแพร่'
    })
  }

  return post
})
```

```vue
<!-- pages/blog/[id].vue - จัดการ Nuxt errors -->
<script setup lang="ts">
const route = useRoute()

const { data: post, error } = await useFetch(`/api/posts/${route.params.id}`)

// ถ้า error ให้โยน Nuxt error เพื่อแสดง error page
if (error.value) {
  throw createError({
    statusCode: error.value.statusCode ?? 500,
    statusMessage: error.value.statusMessage ?? 'Error',
    fatal: true  // fatal: true = แสดง full error page
  })
}
</script>
```

---

## 6. Custom Error Pages

```vue
<!-- error.vue - Root error page -->
<script setup lang="ts">
import type { NuxtError } from '#app'

const { error } = defineProps<{ error: NuxtError }>()

const statusCode = computed(() => error?.statusCode ?? 500)

const errorData = computed(() => ({
  404: {
    title: '404 - ไม่พบหน้า',
    description: 'หน้าที่คุณต้องการไม่มีอยู่',
    icon: '🔍',
    action: 'กลับหน้าหลัก'
  },
  403: {
    title: '403 - ไม่มีสิทธิ์',
    description: 'คุณไม่ได้รับอนุญาตให้เข้าถึงหน้านี้',
    icon: '🚫',
    action: 'เข้าสู่ระบบ'
  },
  500: {
    title: '500 - เซิร์ฟเวอร์ผิดพลาด',
    description: 'เกิดข้อผิดพลาดภายใน กรุณาลองอีกครั้ง',
    icon: '⚠️',
    action: 'ลองอีกครั้ง'
  }
}[statusCode.value] ?? {
  title: `${statusCode.value} - เกิดข้อผิดพลาด`,
  description: error?.message ?? 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ',
  icon: '❌',
  action: 'กลับหน้าหลัก'
}))

function handleError() {
  clearError({ redirect: '/' })
}
</script>

<template>
  <NuxtLayout>
    <div class="min-h-screen flex items-center justify-center">
      <div class="text-center max-w-md">
        <div class="text-6xl mb-4">{{ errorData.icon }}</div>
        <h1 class="text-2xl font-bold mb-2">{{ errorData.title }}</h1>
        <p class="text-gray-600 mb-6">{{ errorData.description }}</p>
        <div class="flex gap-4 justify-center">
          <button @click="handleError" class="btn-primary">
            {{ errorData.action }}
          </button>
        </div>
      </div>
    </div>
  </NuxtLayout>
</template>
```

---

## 7. API Error Handling

```ts
// composables/useApiError.ts - จัดการ API errors อย่างเป็นระบบ
export interface ApiError {
  statusCode: number
  message: string
  details?: Record<string, string[]>
}

export function useApiError() {
  function parseError(err: unknown): ApiError {
    // Nuxt/H3 error
    if (err && typeof err === 'object' && 'statusCode' in err) {
      return {
        statusCode: (err as any).statusCode,
        message: (err as any).statusMessage ?? (err as any).message ?? 'Unknown error',
        details: (err as any).data?.details
      }
    }

    // Standard Error
    if (err instanceof Error) {
      return { statusCode: 500, message: err.message }
    }

    return { statusCode: 500, message: 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ' }
  }

  function getErrorMessage(err: ApiError): string {
    switch (err.statusCode) {
      case 400: return err.details
        ? Object.values(err.details).flat().join(', ')
        : 'ข้อมูลไม่ถูกต้อง'
      case 401: return 'กรุณาเข้าสู่ระบบ'
      case 403: return 'คุณไม่มีสิทธิ์ดำเนินการนี้'
      case 404: return 'ไม่พบข้อมูลที่ต้องการ'
      case 422: return 'ข้อมูลไม่ผ่านการตรวจสอบ'
      case 429: return 'คำขอถูกส่งบ่อยเกินไป กรุณารอสักครู่'
      case 500: return 'เซิร์ฟเวอร์เกิดข้อผิดพลาด กรุณาลองอีกครั้ง'
      default: return err.message
    }
  }

  return { parseError, getErrorMessage }
}
```

---

## 8. ตัวอย่าง: Global Error Handler และ Retry Logic

```ts
// composables/useRetry.ts - Retry Logic
interface RetryOptions {
  maxAttempts?: number
  delay?: number
  backoff?: 'linear' | 'exponential'
  shouldRetry?: (err: unknown, attempt: number) => boolean
}

export function useRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
) {
  const {
    maxAttempts = 3,
    delay = 1000,
    backoff = 'exponential',
    shouldRetry = (err, attempt) => attempt < maxAttempts
  } = options

  const attempts = ref(0)
  const isRetrying = ref(false)
  const lastError = ref<Error | null>(null)

  async function execute(): Promise<T> {
    attempts.value = 0
    lastError.value = null

    while (true) {
      try {
        attempts.value++
        const result = await fn()
        return result
      } catch (err) {
        lastError.value = err instanceof Error ? err : new Error(String(err))

        if (!shouldRetry(err, attempts.value)) {
          throw lastError.value
        }

        // คำนวณ delay
        const waitTime = backoff === 'exponential'
          ? delay * Math.pow(2, attempts.value - 1)
          : delay * attempts.value

        isRetrying.value = true
        await new Promise(resolve => setTimeout(resolve, waitTime))
        isRetrying.value = false
      }
    }
  }

  return { execute, attempts: readonly(attempts), isRetrying: readonly(isRetrying), lastError: readonly(lastError) }
}
```

```vue
<!-- components/RetryableDataLoader.vue -->
<script setup lang="ts">
import { useRetry } from '@/composables/useRetry'

interface Props {
  endpoint: string
}

const props = defineProps<Props>()

const data = ref<unknown>(null)
const error = ref<string | null>(null)
const isLoading = ref(false)

const { execute, attempts, isRetrying } = useRetry(
  () => $fetch(props.endpoint),
  {
    maxAttempts: 3,
    delay: 1000,
    backoff: 'exponential',
    shouldRetry: (err: any, attempt) => {
      // Retry เฉพาะ network errors และ 5xx
      const status = err?.statusCode ?? err?.status
      return attempt < 3 && (!status || status >= 500)
    }
  }
)

async function loadData() {
  isLoading.value = true
  error.value = null
  try {
    data.value = await execute()
  } catch (err: any) {
    error.value = err?.message ?? 'โหลดข้อมูลไม่สำเร็จหลังจากลอง 3 ครั้ง'
  } finally {
    isLoading.value = false
  }
}

onMounted(loadData)
</script>

<template>
  <div>
    <div v-if="isLoading" class="loading">
      <span v-if="isRetrying">
        กำลังลองใหม่... (ครั้งที่ {{ attempts }})
      </span>
      <span v-else>กำลังโหลด...</span>
    </div>
    <div v-else-if="error" class="error">
      <p>{{ error }}</p>
      <button @click="loadData">ลองอีกครั้ง</button>
    </div>
    <slot v-else :data="data" />
  </div>
</template>
```

```ts
// plugins/global-error-handler.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  // จัดการ unhandled promise rejections
  window.addEventListener('unhandledrejection', (event) => {
    console.error('[Unhandled Promise Rejection]', event.reason)
    event.preventDefault()
    // ส่งไป error tracking
  })

  // จัดการ uncaught errors
  window.addEventListener('error', (event) => {
    console.error('[Uncaught Error]', event.error)
    // ส่งไป error tracking
  })

  // Vue error handler
  nuxtApp.vueApp.config.errorHandler = (err, instance, info) => {
    console.error('[Vue Error Handler]', {
      error: err,
      component: instance?.$options?.name ?? 'Unknown',
      info
    })
    // ส่งไป error tracking เช่น Sentry
    if ((window as any).Sentry) {
      (window as any).Sentry.captureException(err, {
        extra: { info, component: instance?.$options?.name }
      })
    }
  }
})
```

---

## สรุป

Error Handling ที่ดีช่วยให้แอปพลิเคชัน robust:

1. **Try-Catch Patterns** - จัดการ async errors อย่างเป็นระบบ
2. **onErrorCaptured** - ดัก errors จาก child components
3. **Error Boundary** - component ที่ทำหน้าที่เป็น safety net
4. **Nuxt Error Pages** - custom pages สำหรับ 404, 403, 500
5. **createError()** - สร้าง HTTP errors ที่ consistent
6. **API Error Handling** - parse และแสดง error messages ที่เป็นมิตร
7. **Retry Logic** - พยายามใหม่อัตโนมัติสำหรับ network errors
8. **Global Error Handler** - ดัก unhandled errors ทั่วทั้งแอป
