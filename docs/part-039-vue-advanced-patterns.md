# Part 39: Vue 3 Advanced Patterns

## ทำไมต้องเรียน Advanced Patterns?

Advanced patterns ใน Vue 3 ช่วยให้สร้าง components ที่ reusable, flexible, และ performant มากขึ้น เมื่อเข้าใจ patterns เหล่านี้ คุณจะสามารถสร้าง component libraries ได้อย่างมืออาชีพ

---

## 1. Render Functions

Render Functions ให้ความยืดหยุ่นสูงสุดในการสร้าง components

```ts
// components/DynamicTag.ts - Component ด้วย Render Function
import { defineComponent, h, type PropType } from 'vue'

type HeadingLevel = 1 | 2 | 3 | 4 | 5 | 6

export default defineComponent({
  name: 'DynamicTag',
  props: {
    tag: {
      type: String as PropType<string>,
      default: 'div'
    },
    level: {
      type: Number as PropType<HeadingLevel>,
      default: undefined
    },
    class: {
      type: [String, Array, Object],
      default: undefined
    }
  },
  setup(props, { slots, attrs }) {
    return () => {
      const tag = props.level ? `h${props.level}` : props.tag

      return h(
        tag,
        { ...attrs, class: props.class },
        slots.default?.()
      )
    }
  }
})
```

```ts
// components/List.ts - Render function สำหรับ dynamic list
import { defineComponent, h, type PropType } from 'vue'

export default defineComponent({
  name: 'DynamicList',
  props: {
    items: {
      type: Array as PropType<unknown[]>,
      required: true
    },
    tag: {
      type: String,
      default: 'ul'
    },
    itemTag: {
      type: String,
      default: 'li'
    }
  },
  setup(props, { slots }) {
    return () => h(
      props.tag,
      { class: 'dynamic-list' },
      props.items.map((item, index) =>
        h(
          props.itemTag,
          { key: index, class: 'dynamic-list-item' },
          slots.default?.({ item, index })
        )
      )
    )
  }
})
```

---

## 2. JSX ใน Vue 3

```ts
// components/DataTable.tsx - Table Component ด้วย JSX
import { defineComponent, ref, computed, type PropType } from 'vue'

interface Column<T = Record<string, unknown>> {
  key: string
  title: string
  sortable?: boolean
  render?: (value: unknown, row: T) => JSX.Element
}

interface Props<T = Record<string, unknown>> {
  data: T[]
  columns: Column<T>[]
  loading?: boolean
}

export const DataTable = defineComponent({
  name: 'DataTable',
  props: {
    data: { type: Array as PropType<Record<string, unknown>[]>, required: true },
    columns: { type: Array as PropType<Column[]>, required: true },
    loading: { type: Boolean, default: false }
  },
  setup(props) {
    const sortKey = ref<string | null>(null)
    const sortOrder = ref<'asc' | 'desc'>('asc')

    const sortedData = computed(() => {
      if (!sortKey.value) return props.data

      return [...props.data].sort((a, b) => {
        const valA = a[sortKey.value!]
        const valB = b[sortKey.value!]
        const compare = valA < valB ? -1 : valA > valB ? 1 : 0
        return sortOrder.value === 'asc' ? compare : -compare
      })
    })

    function handleSort(key: string) {
      if (sortKey.value === key) {
        sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc'
      } else {
        sortKey.value = key
        sortOrder.value = 'asc'
      }
    }

    return () => (
      <div class="data-table-wrapper">
        {props.loading ? (
          <div class="loading">กำลังโหลด...</div>
        ) : (
          <table class="data-table">
            <thead>
              <tr>
                {props.columns.map(col => (
                  <th
                    key={col.key}
                    class={{ sortable: col.sortable, active: sortKey.value === col.key }}
                    onClick={() => col.sortable && handleSort(col.key)}
                  >
                    {col.title}
                    {col.sortable && (
                      <span class="sort-icon">
                        {sortKey.value === col.key
                          ? sortOrder.value === 'asc' ? ' ↑' : ' ↓'
                          : ' ⇅'}
                      </span>
                    )}
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {sortedData.value.map((row, rowIndex) => (
                <tr key={rowIndex}>
                  {props.columns.map(col => (
                    <td key={col.key}>
                      {col.render
                        ? col.render(row[col.key], row)
                        : String(row[col.key] ?? '')
                      }
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    )
  }
})
```

---

## 3. Functional Components

```ts
// components/Badge.ts - Functional Component
import { defineComponent, h } from 'vue'

const Badge = defineComponent({
  functional: true,
  props: {
    type: {
      type: String as () => 'primary' | 'success' | 'warning' | 'danger',
      default: 'primary'
    },
    rounded: Boolean
  },
  render(props, { slots }) {
    return h(
      'span',
      {
        class: [
          'badge',
          `badge-${props.type}`,
          { 'badge-rounded': props.rounded }
        ]
      },
      slots.default?.()
    )
  }
})

export default Badge
```

```vue
<!-- Functional Component ด้วย Vue SFC -->
<script lang="ts">
// Functional SFC - ไม่มี state, เร็วกว่า stateful component
export default {
  functional: true
}
</script>

<script setup lang="ts">
// ใช้ defineProps โดยไม่มี setup function
defineProps<{
  label: string
  count?: number
}>()
</script>

<template>
  <div class="stat-widget">
    <span class="label">{{ label }}</span>
    <span class="value">{{ count ?? 0 }}</span>
  </div>
</template>
```

---

## 4. Teleport Component

```vue
<!-- components/Modal.vue - Modal ด้วย Teleport -->
<script setup lang="ts">
interface Props {
  modelValue: boolean
  title?: string
  size?: 'sm' | 'md' | 'lg' | 'xl'
  persistent?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  size: 'md',
  persistent: false
})

const emit = defineEmits<{
  'update:modelValue': [value: boolean]
  close: []
}>()

function close() {
  if (!props.persistent) {
    emit('update:modelValue', false)
    emit('close')
  }
}

// ปิดด้วย Escape key
onMounted(() => {
  function handleKeyDown(e: KeyboardEvent) {
    if (e.key === 'Escape' && props.modelValue) close()
  }
  window.addEventListener('keydown', handleKeyDown)
  onUnmounted(() => window.removeEventListener('keydown', handleKeyDown))
})

// ป้องกัน scroll เมื่อ modal เปิด
watch(() => props.modelValue, (isOpen) => {
  document.body.style.overflow = isOpen ? 'hidden' : ''
})
</script>

<template>
  <!-- Teleport ไปที่ body - ป้องกัน z-index issues -->
  <Teleport to="body">
    <Transition name="modal">
      <div
        v-if="modelValue"
        class="modal-overlay"
        @click.self="close"
      >
        <div :class="`modal modal-${size}`" role="dialog" aria-modal="true">
          <!-- Header -->
          <div v-if="title || $slots.header" class="modal-header">
            <slot name="header">
              <h2>{{ title }}</h2>
            </slot>
            <button
              v-if="!persistent"
              @click="close"
              class="modal-close"
              aria-label="ปิด"
            >
              ×
            </button>
          </div>

          <!-- Body -->
          <div class="modal-body">
            <slot />
          </div>

          <!-- Footer -->
          <div v-if="$slots.footer" class="modal-footer">
            <slot name="footer" />
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: white;
  border-radius: 8px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  max-height: 90vh;
  overflow-y: auto;
}

.modal-sm { width: 400px; }
.modal-md { width: 600px; }
.modal-lg { width: 800px; }
.modal-xl { width: 1200px; }

.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}
</style>
```

---

## 5. Suspense Component

```vue
<!-- components/AsyncUserProfile.vue - Async component ที่ใช้กับ Suspense -->
<script setup lang="ts">
interface Props {
  userId: number
}

const props = defineProps<Props>()

// async setup - ทำงานกับ Suspense
const user = await $fetch(`/api/users/${props.userId}`)
const posts = await $fetch(`/api/users/${props.userId}/posts`)
</script>

<template>
  <div class="user-profile">
    <h2>{{ user.name }}</h2>
    <ul>
      <li v-for="post in posts" :key="post.id">{{ post.title }}</li>
    </ul>
  </div>
</template>
```

```vue
<!-- pages/profile/[id].vue - ใช้ Suspense -->
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const AsyncUserProfile = defineAsyncComponent(
  () => import('@/components/AsyncUserProfile.vue')
)
</script>

<template>
  <div class="profile-page">
    <Suspense>
      <!-- Content จะแสดงเมื่อ async setup เสร็จ -->
      <template #default>
        <AsyncUserProfile :user-id="Number($route.params.id)" />
      </template>

      <!-- แสดงระหว่างรอ -->
      <template #fallback>
        <div class="skeleton-profile">
          <div class="skeleton w-24 h-24 rounded-full" />
          <div class="skeleton w-48 h-4 mt-4" />
          <div class="skeleton w-32 h-4 mt-2" />
        </div>
      </template>
    </Suspense>
  </div>
</template>
```

---

## 6. Async Components กับ Suspense

```vue
<!-- App.vue - Suspense กับ error handling -->
<script setup lang="ts">
import { defineAsyncComponent, ref } from 'vue'

const HeavyDashboard = defineAsyncComponent({
  loader: () => import('./components/HeavyDashboard.vue'),
  loadingComponent: {
    template: '<div class="loading">กำลังโหลด Dashboard...</div>'
  },
  errorComponent: {
    template: '<div class="error">โหลด Dashboard ไม่สำเร็จ</div>'
  },
  delay: 200,
  timeout: 30000
})

const error = ref<Error | null>(null)

function onSuspenseError(err: Error) {
  error.value = err
}
</script>

<template>
  <div>
    <div v-if="error" class="error">
      {{ error.message }}
      <button @click="error = null">ลองอีกครั้ง</button>
    </div>
    
    <Suspense @resolve="console.log('Loaded!')" @pending="console.log('Loading...')">
      <HeavyDashboard />
      <template #fallback>
        <DashboardSkeleton />
      </template>
    </Suspense>
  </div>
</template>
```

---

## 7. Higher-Order Components (HOC)

```ts
// composables/withLoading.ts - HOC Pattern
import { defineComponent, h, ref, type Component } from 'vue'

export function withLoading<T extends Component>(WrappedComponent: T) {
  return defineComponent({
    name: `WithLoading(${(WrappedComponent as any).name ?? 'Component'})`,
    props: {
      loading: {
        type: Boolean,
        default: false
      },
      loadingText: {
        type: String,
        default: 'กำลังโหลด...'
      }
    },
    setup(props, { attrs, slots }) {
      return () => {
        if (props.loading) {
          return h('div', { class: 'loading-wrapper' }, [
            h('div', { class: 'loading-spinner' }),
            h('span', { class: 'loading-text' }, props.loadingText)
          ])
        }
        return h(WrappedComponent, attrs, slots)
      }
    }
  })
}
```

```ts
// composables/withPermission.ts - HOC สำหรับ permission
import { defineComponent, h, type Component } from 'vue'
import { useAuthStore } from '@/stores/auth'

export function withPermission<T extends Component>(
  WrappedComponent: T,
  requiredPermission: string
) {
  return defineComponent({
    name: `WithPermission(${(WrappedComponent as any).name ?? 'Component'})`,
    setup(_, { attrs, slots }) {
      const authStore = useAuthStore()
      const hasPermission = authStore.hasPermission(requiredPermission as any)

      return () => {
        if (!hasPermission) {
          return h('div', { class: 'no-permission' }, 'ไม่มีสิทธิ์เข้าถึง')
        }
        return h(WrappedComponent, attrs, slots)
      }
    }
  })
}
```

---

## 8. Compound Components Pattern

```vue
<!-- components/Select/index.ts - Compound Select Component -->
<script lang="ts">
// Context for compound component
import { provide, inject, ref, computed, type Ref } from 'vue'

const SELECT_KEY = Symbol('select')

interface SelectContext {
  selectedValue: Ref<unknown>
  isOpen: Ref<boolean>
  select: (value: unknown) => void
  toggle: () => void
  close: () => void
}

export function useSelectContext(): SelectContext {
  const context = inject<SelectContext>(SELECT_KEY)
  if (!context) {
    throw new Error('Select components must be used inside <Select>')
  }
  return context
}

export function provideSelectContext(context: SelectContext) {
  provide(SELECT_KEY, context)
}
</script>
```

```vue
<!-- components/Select/Select.vue -->
<script setup lang="ts">
import { provideSelectContext } from './index'

const props = defineProps<{
  modelValue?: unknown
}>()

const emit = defineEmits<{
  'update:modelValue': [value: unknown]
}>()

const isOpen = ref(false)
const selectedValue = ref(props.modelValue)

function select(value: unknown) {
  selectedValue.value = value
  emit('update:modelValue', value)
  isOpen.value = false
}

function toggle() {
  isOpen.value = !isOpen.value
}

function close() {
  isOpen.value = false
}

provideSelectContext({ selectedValue, isOpen, select, toggle, close })
</script>

<template>
  <div class="select-container" v-click-outside="close">
    <slot />
  </div>
</template>
```

```vue
<!-- components/Select/SelectTrigger.vue -->
<script setup lang="ts">
import { useSelectContext } from './index'

const { isOpen, toggle, selectedValue } = useSelectContext()
</script>

<template>
  <button
    @click="toggle"
    :aria-expanded="isOpen"
    class="select-trigger"
  >
    <slot :selected-value="selectedValue" :is-open="isOpen" />
    <span class="chevron" :class="{ open: isOpen }">▼</span>
  </button>
</template>
```

```vue
<!-- components/Select/SelectContent.vue -->
<script setup lang="ts">
import { useSelectContext } from './index'

const { isOpen } = useSelectContext()
</script>

<template>
  <Transition name="dropdown">
    <div v-if="isOpen" class="select-content">
      <slot />
    </div>
  </Transition>
</template>
```

```vue
<!-- components/Select/SelectItem.vue -->
<script setup lang="ts">
import { useSelectContext } from './index'

const props = defineProps<{
  value: unknown
  disabled?: boolean
}>()

const { select, selectedValue } = useSelectContext()

const isSelected = computed(() => selectedValue.value === props.value)
</script>

<template>
  <div
    :class="['select-item', { selected: isSelected, disabled: disabled }]"
    @click="!disabled && select(value)"
    :aria-selected="isSelected"
  >
    <slot :is-selected="isSelected" />
  </div>
</template>
```

```vue
<!-- การใช้งาน Compound Select Component -->
<template>
  <Select v-model="selectedCountry">
    <SelectTrigger>
      <template #default="{ selectedValue }">
        {{ selectedValue ? selectedValue : 'เลือกประเทศ' }}
      </template>
    </SelectTrigger>

    <SelectContent>
      <SelectItem value="th">
        <template #default="{ isSelected }">
          🇹🇭 ไทย {{ isSelected ? '✓' : '' }}
        </template>
      </SelectItem>
      <SelectItem value="en">
        🇺🇸 สหรัฐอเมริกา
      </SelectItem>
      <SelectItem value="ja">
        🇯🇵 ญี่ปุ่น
      </SelectItem>
    </SelectContent>
  </Select>
</template>
```

---

## 9. ตัวอย่าง: Modal System

```ts
// composables/useModal.ts - Modal System
import { defineComponent, h, ref, shallowRef, type Component, type App } from 'vue'

interface ModalOptions {
  component: Component
  props?: Record<string, unknown>
  onClose?: () => void
}

const modals = ref<Array<ModalOptions & { id: number }>>([])
let nextId = 0

export function useModal() {
  function open(options: ModalOptions): number {
    const id = nextId++
    modals.value.push({ ...options, id })
    return id
  }

  function close(id?: number) {
    if (id !== undefined) {
      const index = modals.value.findIndex(m => m.id === id)
      if (index !== -1) {
        modals.value[index].onClose?.()
        modals.value.splice(index, 1)
      }
    } else {
      // ปิด modal ล่าสุด
      const last = modals.value[modals.value.length - 1]
      if (last) {
        last.onClose?.()
        modals.value.pop()
      }
    }
  }

  function closeAll() {
    modals.value.forEach(m => m.onClose?.())
    modals.value = []
  }

  return { modals: readonly(modals), open, close, closeAll }
}

// Modal Container Component
export const ModalContainer = defineComponent({
  name: 'ModalContainer',
  setup() {
    const { modals, close } = useModal()
    return () => modals.value.map(modal =>
      h(modal.component, {
        key: modal.id,
        ...modal.props,
        onClose: () => close(modal.id)
      })
    )
  }
})
```

---

## 10. ตัวอย่าง: Toast Notification System

```ts
// composables/useToast.ts
export type ToastType = 'info' | 'success' | 'warning' | 'error'

export interface Toast {
  id: number
  type: ToastType
  title?: string
  message: string
  duration?: number
}

const toasts = ref<Toast[]>([])
let nextId = 0

export function useToast() {
  function show(options: Omit<Toast, 'id'>): number {
    const id = nextId++
    const toast: Toast = {
      id,
      duration: 3000,
      ...options
    }

    toasts.value.push(toast)

    if (toast.duration && toast.duration > 0) {
      setTimeout(() => dismiss(id), toast.duration)
    }

    return id
  }

  const success = (message: string, title?: string) =>
    show({ type: 'success', message, title })

  const error = (message: string, title?: string) =>
    show({ type: 'error', message, title, duration: 5000 })

  const warning = (message: string, title?: string) =>
    show({ type: 'warning', message, title })

  const info = (message: string, title?: string) =>
    show({ type: 'info', message, title })

  function dismiss(id: number) {
    toasts.value = toasts.value.filter(t => t.id !== id)
  }

  function dismissAll() {
    toasts.value = []
  }

  return {
    toasts: readonly(toasts),
    show,
    success,
    error,
    warning,
    info,
    dismiss,
    dismissAll
  }
}
```

```vue
<!-- components/ToastContainer.vue -->
<script setup lang="ts">
import { useToast } from '@/composables/useToast'

const { toasts, dismiss } = useToast()
</script>

<template>
  <Teleport to="body">
    <div class="toast-container" aria-live="polite">
      <TransitionGroup name="toast">
        <div
          v-for="toast in toasts"
          :key="toast.id"
          :class="['toast', `toast-${toast.type}`]"
          role="alert"
        >
          <div class="toast-content">
            <strong v-if="toast.title" class="toast-title">{{ toast.title }}</strong>
            <p class="toast-message">{{ toast.message }}</p>
          </div>
          <button @click="dismiss(toast.id)" class="toast-close" aria-label="ปิด">
            ×
          </button>
        </div>
      </TransitionGroup>
    </div>
  </Teleport>
</template>

<style scoped>
.toast-container {
  position: fixed;
  top: 1rem;
  right: 1rem;
  z-index: 9999;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.toast {
  display: flex;
  align-items: start;
  padding: 1rem;
  border-radius: 8px;
  min-width: 300px;
  max-width: 500px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.toast-success { background: #f0fdf4; border-left: 4px solid #22c55e; }
.toast-error { background: #fef2f2; border-left: 4px solid #ef4444; }
.toast-warning { background: #fffbeb; border-left: 4px solid #f59e0b; }
.toast-info { background: #eff6ff; border-left: 4px solid #3b82f6; }

.toast-enter-active,
.toast-leave-active {
  transition: all 0.3s ease;
}

.toast-enter-from {
  opacity: 0;
  transform: translateX(100%);
}

.toast-leave-to {
  opacity: 0;
  transform: translateX(100%);
}
</style>
```

---

## สรุป

Vue 3 Advanced Patterns ช่วยสร้าง components ที่มีคุณภาพสูง:

1. **Render Functions** - ความยืดหยุ่นสูงสุดในการสร้าง components
2. **JSX** - เขียน template ด้วย JavaScript syntax
3. **Functional Components** - components ที่ไม่มี state ทำงานเร็วกว่า
4. **Teleport** - render elements นอก DOM hierarchy ปัจจุบัน
5. **Suspense** - จัดการ async components อย่างสวยงาม
6. **Async Components** - lazy load components เพื่อ performance
7. **HOC Pattern** - เพิ่ม functionality ให้ components ด้วย wrapper
8. **Compound Components** - components ที่แชร์ state ผ่าน Context
9. **Modal System** - ระบบ modal ที่จัดการได้จากทุกที่
10. **Toast System** - ระบบ notification ที่ reusable
