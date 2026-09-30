# Part 9: Components พื้นฐาน

## บทนำ

Components เป็นหัวใจสำคัญของ Vue.js ที่ช่วยให้เราแบ่ง UI ออกเป็นชิ้นส่วนที่ reusable ได้ การเข้าใจวิธีสร้าง component ที่ดี การ register และการใช้ Single File Components (SFC) จะทำให้โค้ดของเราสะอาด maintainable และ scalable มากขึ้น

---

## 1. สร้าง Component ครั้งแรก

```vue
<!-- HelloWorld.vue - Component แรก -->
<script setup lang="ts">
import { ref, defineProps } from 'vue'

// Props ที่รับมาจาก parent
const props = defineProps<{
  name?: string
  greeting?: string
}>()

const count = ref(0)
const message = `${props.greeting || 'สวัสดี'}, ${props.name || 'โลก'}!`
</script>

<template>
  <div class="hello-world">
    <h1>{{ message }}</h1>
    <p>คลิกไปแล้ว {{ count }} ครั้ง</p>
    <button @click="count++">คลิกฉัน</button>
  </div>
</template>

<style scoped>
.hello-world {
  text-align: center;
  padding: 2rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}
</style>
```

```vue
<!-- App.vue - ใช้ HelloWorld component -->
<script setup lang="ts">
import HelloWorld from './components/HelloWorld.vue'
</script>

<template>
  <HelloWorld name="Vue.js" greeting="ยินดีต้อนรับสู่" />
  <HelloWorld name="โลก" />
</template>
```

---

## 2. Single File Component (SFC) Structure

SFC คือไฟล์ `.vue` ที่รวม template, script และ style ไว้ในไฟล์เดียว

```vue
<!-- FullSFC.vue - โครงสร้าง SFC สมบูรณ์ -->

<!-- SCRIPT BLOCK: logic ของ component -->
<script setup lang="ts">
// 1. Imports
import { ref, computed, onMounted } from 'vue'
import type { PropType } from 'vue'
import ChildComponent from './ChildComponent.vue'

// 2. Types/Interfaces
interface User {
  id: number
  name: string
  role: string
}

// 3. Props
const props = defineProps<{
  title: string
  users?: User[]
}>()

// 4. Emits
const emit = defineEmits<{
  'user-selected': [user: User]
  'close': []
}>()

// 5. Reactive state
const isExpanded = ref(false)
const selectedUser = ref<User | null>(null)
const searchQuery = ref('')

// 6. Computed
const filteredUsers = computed(() =>
  (props.users || []).filter(u =>
    u.name.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
)

// 7. Methods
function selectUser(user: User) {
  selectedUser.value = user
  emit('user-selected', user)
}

// 8. Lifecycle hooks
onMounted(() => {
  console.log('Component mounted:', props.title)
})

// 9. Expose (ถ้าจำเป็น)
defineExpose({ isExpanded, selectedUser })
</script>

<!-- TEMPLATE BLOCK: HTML structure -->
<template>
  <div class="component-wrapper">
    <header class="component-header">
      <h2>{{ title }}</h2>
      <button @click="isExpanded = !isExpanded">
        {{ isExpanded ? 'ซ่อน' : 'แสดง' }}
      </button>
    </header>

    <div v-if="isExpanded" class="component-body">
      <input v-model="searchQuery" placeholder="ค้นหา..." />

      <ul v-if="filteredUsers.length > 0">
        <li
          v-for="user in filteredUsers"
          :key="user.id"
          @click="selectUser(user)"
          :class="{ selected: selectedUser?.id === user.id }"
        >
          {{ user.name }} - {{ user.role }}
        </li>
      </ul>
      <p v-else>ไม่พบผู้ใช้</p>

      <ChildComponent v-if="selectedUser" :user="selectedUser" />
    </div>
  </div>
</template>

<!-- STYLE BLOCK: scoped styles -->
<style scoped>
/* scoped: styles เฉพาะ component นี้ ไม่รั่วออกไปที่อื่น */
.component-wrapper {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow: hidden;
}

.component-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  background: #f9fafb;
  border-bottom: 1px solid #e5e7eb;
}

.component-body {
  padding: 1rem;
}

li.selected {
  background: #eff6ff;
}

/* :deep() เพื่อ style child components จาก parent */
:deep(.child-title) {
  color: #1d4ed8;
}
</style>
```

### Script Block Variants

```vue
<!-- Option 1: <script setup> - แนะนำ (Vue 3.2+) -->
<script setup lang="ts">
const message = ref('Hello')
</script>

<!-- Option 2: <script> + defineComponent (สำหรับ compatibility) -->
<script lang="ts">
import { defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'MyComponent',
  props: {
    title: String
  },
  setup(props) {
    const message = ref('Hello')
    return { message }
  }
})
</script>

<!-- Option 3: สามารถมีทั้งสองพร้อมกันได้ (rare cases) -->
<script lang="ts">
export default {
  name: 'NamedComponent', // สำหรับ DevTools display name
  inheritAttrs: false,
}
</script>
<script setup lang="ts">
const message = ref('Hello')
</script>
```

---

## 3. Component Registration (Global/Local)

### Local Registration (แนะนำ)

```vue
<!-- ลงทะเบียน component เฉพาะในไฟล์ที่ใช้ -->
<script setup lang="ts">
// ใช้ <script setup> - ไม่ต้อง register ก็ได้ import แล้วใช้เลย
import MyButton from './components/MyButton.vue'
import UserCard from './components/UserCard.vue'
import { defineAsyncComponent } from 'vue'

// Async component - โหลดเมื่อจำเป็น (lazy loading)
const HeavyChart = defineAsyncComponent(() =>
  import('./components/HeavyChart.vue')
)

// Async กับ loading/error states
const AsyncUserList = defineAsyncComponent({
  loader: () => import('./components/UserList.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,      // delay ก่อนแสดง loading
  timeout: 3000,   // timeout ก่อนแสดง error
})
</script>

<template>
  <MyButton>คลิก</MyButton>
  <UserCard :user="user" />
  <Suspense>
    <HeavyChart :data="chartData" />
    <template #fallback>กำลังโหลด chart...</template>
  </Suspense>
</template>
```

### Global Registration

```typescript
// main.ts - Global registration
import { createApp } from 'vue'
import App from './App.vue'

// Import components
import BaseButton from './components/base/BaseButton.vue'
import BaseInput from './components/base/BaseInput.vue'
import BaseCard from './components/base/BaseCard.vue'
import BaseAlert from './components/base/BaseAlert.vue'

const app = createApp(App)

// Register globally - สามารถใช้ได้ทุกที่โดยไม่ต้อง import
app.component('BaseButton', BaseButton)
app.component('BaseInput', BaseInput)
app.component('BaseCard', BaseCard)
app.component('BaseAlert', BaseAlert)

// หรือ register ทีละหลายตัว
const globalComponents = {
  BaseButton,
  BaseInput,
  BaseCard,
  BaseAlert,
}

Object.entries(globalComponents).forEach(([name, component]) => {
  app.component(name, component)
})

app.mount('#app')
```

### Auto-import กับ unplugin-vue-components

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import Components from 'unplugin-vue-components/vite'

export default defineConfig({
  plugins: [
    vue(),
    Components({
      // auto-import components จากโฟลเดอร์ src/components
      dirs: ['src/components'],
      extensions: ['vue'],
      // generate d.ts file
      dts: 'src/components.d.ts',
    }),
  ],
})
```

---

## 4. Component Naming Conventions

```
src/
├── components/
│   ├── base/              # Base/UI components (global)
│   │   ├── BaseButton.vue
│   │   ├── BaseInput.vue
│   │   └── BaseCard.vue
│   ├── common/            # Reusable feature components
│   │   ├── UserAvatar.vue
│   │   ├── ProductCard.vue
│   │   └── SearchBar.vue
│   ├── layout/            # Layout components
│   │   ├── TheHeader.vue  # "The" = singleton
│   │   ├── TheSidebar.vue
│   │   └── TheFooter.vue
│   └── features/          # Feature-specific components
│       ├── auth/
│       │   ├── LoginForm.vue
│       │   └── RegisterForm.vue
│       └── dashboard/
│           ├── StatsCard.vue
│           └── RecentActivity.vue
```

### Naming Rules

```vue
<!-- ✅ PascalCase ในไฟล์และ import -->
<script setup>
import MyComponent from './MyComponent.vue'
import BaseButton from './base/BaseButton.vue'
</script>

<!-- ✅ ใช้ PascalCase ใน template (แนะนำ) -->
<template>
  <MyComponent />
  <BaseButton>Click</BaseButton>
</template>

<!-- ✅ หรือ kebab-case ก็ได้ -->
<template>
  <my-component />
  <base-button>Click</base-button>
</template>

<!-- Naming conventions -->
<!-- Base* - atomic/presentational components -->
<!-- BaseButton, BaseInput, BaseCard, BaseBadge -->

<!-- The* - singleton components (มีแค่อันเดียวใน app) -->
<!-- TheHeader, TheSidebar, TheNavigation -->

<!-- [Feature]* - feature-specific components -->
<!-- UserProfile, ProductList, CartSummary -->

<!-- App* - app-level components -->
<!-- AppError, AppLoading, AppModal -->
```

---

## 5. Component Reuse

```vue
<!-- ReusableCard.vue - ตัวอย่าง reusable component -->
<script setup lang="ts">
interface Props {
  title?: string
  subtitle?: string
  variant?: 'default' | 'outlined' | 'elevated'
  padding?: 'none' | 'sm' | 'md' | 'lg'
  loading?: boolean
  clickable?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'default',
  padding: 'md',
  loading: false,
  clickable: false,
})

const emit = defineEmits<{ click: [event: MouseEvent] }>()
</script>

<template>
  <div
    class="card"
    :class="[`card--${variant}`, `card--p-${padding}`, { 'card--clickable': clickable, 'card--loading': loading }]"
    @click="clickable ? emit('click', $event) : null"
  >
    <!-- Header slot -->
    <div v-if="title || subtitle || $slots.header" class="card__header">
      <slot name="header">
        <div>
          <h3 v-if="title">{{ title }}</h3>
          <p v-if="subtitle" class="subtitle">{{ subtitle }}</p>
        </div>
      </slot>
    </div>

    <!-- Loading overlay -->
    <div v-if="loading" class="card__loading">
      <div class="spinner"></div>
    </div>

    <!-- Default slot (content) -->
    <div class="card__body">
      <slot />
    </div>

    <!-- Footer slot -->
    <div v-if="$slots.footer" class="card__footer">
      <slot name="footer" />
    </div>
  </div>
</template>

<style scoped>
.card { background: white; border-radius: 8px; position: relative; }
.card--default { box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
.card--outlined { border: 1px solid #e5e7eb; }
.card--elevated { box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); }
.card--p-none .card__body { padding: 0; }
.card--p-sm .card__body { padding: 0.75rem; }
.card--p-md .card__body { padding: 1.25rem; }
.card--p-lg .card__body { padding: 2rem; }
.card--clickable { cursor: pointer; transition: transform 0.2s; }
.card--clickable:hover { transform: translateY(-2px); }
.card__header { padding: 1rem 1.25rem; border-bottom: 1px solid #f3f4f6; }
.card__footer { padding: 1rem 1.25rem; border-top: 1px solid #f3f4f6; }
.card__loading { position: absolute; inset: 0; background: rgba(255,255,255,0.8); display: flex; align-items: center; justify-content: center; border-radius: 8px; }
.spinner { width: 24px; height: 24px; border: 3px solid #e5e7eb; border-top-color: #3b82f6; border-radius: 50%; animation: spin 0.8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }
</style>
```

---

## 6. defineComponent()

```typescript
// ใช้ defineComponent เมื่อไม่ใช้ <script setup>
// หรือเมื่อต้องการ type inference ที่ดีกว่า

import { defineComponent, ref, computed, PropType } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

export default defineComponent({
  name: 'UserList',

  props: {
    users: {
      type: Array as PropType<User[]>,
      default: () => [],
    },
    loading: {
      type: Boolean,
      default: false,
    },
    title: {
      type: String,
      required: true,
    },
  },

  emits: {
    // กำหนด type ของ emit
    'user-click': (user: User) => true,
    'refresh': () => true,
  },

  setup(props, { emit, slots, attrs, expose }) {
    const searchQuery = ref('')
    const selectedId = ref<number | null>(null)

    const filtered = computed(() =>
      props.users.filter(u =>
        u.name.toLowerCase().includes(searchQuery.value.toLowerCase())
      )
    )

    function handleClick(user: User) {
      selectedId.value = user.id
      emit('user-click', user)
    }

    // expose สำหรับ parent component ที่ใช้ template refs
    expose({
      clearSelection: () => { selectedId.value = null },
      selectedUser: computed(() => props.users.find(u => u.id === selectedId.value)),
    })

    return {
      searchQuery,
      selectedId,
      filtered,
      handleClick,
    }
  },
})
```

---

## 7. ตัวอย่าง: UI Component Library เล็กๆ

```vue
<!-- components/ui/UiButton.vue -->
<script setup lang="ts">
type Variant = 'primary' | 'secondary' | 'danger' | 'ghost' | 'outline'
type Size = 'xs' | 'sm' | 'md' | 'lg'

interface Props {
  variant?: Variant
  size?: Size
  disabled?: boolean
  loading?: boolean
  fullWidth?: boolean
  type?: 'button' | 'submit' | 'reset'
  icon?: string
  iconPosition?: 'left' | 'right'
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'primary',
  size: 'md',
  disabled: false,
  loading: false,
  fullWidth: false,
  type: 'button',
  iconPosition: 'left',
})

const emit = defineEmits<{ click: [event: MouseEvent] }>()

const classes = {
  base: 'inline-flex items-center justify-center font-medium rounded transition-all focus:outline-none focus:ring-2 focus:ring-offset-2',
  size: {
    xs: 'px-2.5 py-1.5 text-xs',
    sm: 'px-3 py-2 text-sm',
    md: 'px-4 py-2 text-sm',
    lg: 'px-6 py-3 text-base',
  },
  variant: {
    primary: 'bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500',
    secondary: 'bg-gray-600 text-white hover:bg-gray-700 focus:ring-gray-500',
    danger: 'bg-red-600 text-white hover:bg-red-700 focus:ring-red-500',
    ghost: 'bg-transparent text-gray-700 hover:bg-gray-100 focus:ring-gray-500',
    outline: 'border border-gray-300 bg-white text-gray-700 hover:bg-gray-50 focus:ring-blue-500',
  },
}
</script>

<template>
  <button
    :type="type"
    :disabled="disabled || loading"
    :class="[
      'inline-flex items-center justify-center font-medium rounded transition-all',
      `btn-${variant}`,
      `btn-${size}`,
      { 'w-full': fullWidth, 'opacity-50 cursor-not-allowed': disabled, 'btn-loading': loading }
    ]"
    @click="!disabled && !loading ? emit('click', $event) : null"
  >
    <span v-if="loading" class="btn-spinner"></span>
    <span v-if="icon && iconPosition === 'left'" class="btn-icon-left">{{ icon }}</span>
    <slot />
    <span v-if="icon && iconPosition === 'right'" class="btn-icon-right">{{ icon }}</span>
  </button>
</template>

<style scoped>
button { padding: 0.5rem 1rem; border: none; border-radius: 6px; cursor: pointer; font-weight: 500; display: inline-flex; align-items: center; gap: 0.5rem; transition: all 0.2s; }
.btn-primary { background: #3b82f6; color: white; }
.btn-primary:hover:not(:disabled) { background: #2563eb; }
.btn-secondary { background: #6b7280; color: white; }
.btn-danger { background: #ef4444; color: white; }
.btn-danger:hover:not(:disabled) { background: #dc2626; }
.btn-ghost { background: transparent; color: #374151; }
.btn-ghost:hover:not(:disabled) { background: #f3f4f6; }
.btn-outline { background: white; border: 1px solid #d1d5db; color: #374151; }
.btn-xs { padding: 0.25rem 0.5rem; font-size: 0.75rem; }
.btn-sm { padding: 0.375rem 0.75rem; font-size: 0.875rem; }
.btn-md { padding: 0.5rem 1rem; font-size: 0.875rem; }
.btn-lg { padding: 0.75rem 1.5rem; font-size: 1rem; }
.w-full { width: 100%; }
button:disabled { opacity: 0.5; cursor: not-allowed; }
.btn-spinner { width: 16px; height: 16px; border: 2px solid currentColor; border-top-color: transparent; border-radius: 50%; animation: spin 0.8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }
</style>
```

```vue
<!-- components/ui/UiInput.vue -->
<script setup lang="ts">
type InputType = 'text' | 'email' | 'password' | 'number' | 'tel' | 'url' | 'search'

interface Props {
  modelValue: string | number
  type?: InputType
  placeholder?: string
  label?: string
  hint?: string
  error?: string
  disabled?: boolean
  required?: boolean
  prefix?: string
  suffix?: string
  clearable?: boolean
  name?: string
}

const props = withDefaults(defineProps<Props>(), {
  type: 'text',
  disabled: false,
  required: false,
  clearable: false,
})

const emit = defineEmits<{
  'update:modelValue': [value: string | number]
  'clear': []
  'focus': [event: FocusEvent]
  'blur': [event: FocusEvent]
}>()

function handleInput(event: Event) {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', props.type === 'number' ? Number(target.value) : target.value)
}
</script>

<template>
  <div class="input-wrapper" :class="{ 'has-error': error, 'is-disabled': disabled }">
    <label v-if="label" class="input-label">
      {{ label }}
      <span v-if="required" class="required-mark">*</span>
    </label>

    <div class="input-container">
      <span v-if="prefix" class="input-prefix">{{ prefix }}</span>

      <input
        :type="type"
        :value="modelValue"
        :placeholder="placeholder"
        :disabled="disabled"
        :required="required"
        :name="name"
        class="input-field"
        @input="handleInput"
        @focus="emit('focus', $event)"
        @blur="emit('blur', $event)"
      />

      <button
        v-if="clearable && modelValue"
        @click="emit('update:modelValue', ''); emit('clear')"
        class="clear-btn"
        type="button"
      >
        ✕
      </button>

      <span v-if="suffix" class="input-suffix">{{ suffix }}</span>
    </div>

    <p v-if="error" class="input-error">{{ error }}</p>
    <p v-else-if="hint" class="input-hint">{{ hint }}</p>
  </div>
</template>

<style scoped>
.input-wrapper { display: flex; flex-direction: column; gap: 0.25rem; }
.input-label { font-size: 0.875rem; font-weight: 500; color: #374151; }
.required-mark { color: #ef4444; margin-left: 0.25rem; }
.input-container { display: flex; align-items: center; border: 1px solid #d1d5db; border-radius: 6px; transition: border-color 0.2s; }
.input-container:focus-within { border-color: #3b82f6; box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1); }
.has-error .input-container { border-color: #ef4444; }
.input-field { flex: 1; padding: 0.5rem 0.75rem; border: none; background: transparent; outline: none; font-size: 0.875rem; }
.input-prefix, .input-suffix { padding: 0.5rem 0.75rem; color: #6b7280; background: #f9fafb; font-size: 0.875rem; }
.input-prefix { border-right: 1px solid #d1d5db; border-radius: 6px 0 0 6px; }
.input-suffix { border-left: 1px solid #d1d5db; border-radius: 0 6px 6px 0; }
.clear-btn { padding: 0 0.5rem; background: none; border: none; cursor: pointer; color: #9ca3af; }
.clear-btn:hover { color: #374151; }
.input-error { font-size: 0.75rem; color: #ef4444; }
.input-hint { font-size: 0.75rem; color: #6b7280; }
.is-disabled .input-container { background: #f9fafb; opacity: 0.6; }
</style>
```

```vue
<!-- components/ui/UiBadge.vue -->
<script setup lang="ts">
type Variant = 'default' | 'primary' | 'success' | 'warning' | 'danger' | 'info'
type Size = 'sm' | 'md' | 'lg'

const props = withDefaults(defineProps<{
  variant?: Variant
  size?: Size
  dot?: boolean
  closable?: boolean
}>(), {
  variant: 'default',
  size: 'md',
  dot: false,
  closable: false,
})

const emit = defineEmits<{ close: [] }>()
</script>

<template>
  <span class="badge" :class="[`badge--${variant}`, `badge--${size}`]">
    <span v-if="dot" class="badge-dot"></span>
    <slot />
    <button v-if="closable" @click="emit('close')" class="badge-close">✕</button>
  </span>
</template>

<style scoped>
.badge { display: inline-flex; align-items: center; gap: 0.25rem; padding: 0.125rem 0.625rem; border-radius: 9999px; font-size: 0.75rem; font-weight: 500; }
.badge--default { background: #f3f4f6; color: #374151; }
.badge--primary { background: #dbeafe; color: #1d4ed8; }
.badge--success { background: #dcfce7; color: #15803d; }
.badge--warning { background: #fef3c7; color: #92400e; }
.badge--danger { background: #fee2e2; color: #b91c1c; }
.badge--info { background: #e0f2fe; color: #0369a1; }
.badge--sm { padding: 0.0625rem 0.375rem; font-size: 0.6875rem; }
.badge--lg { padding: 0.25rem 0.875rem; font-size: 0.875rem; }
.badge-dot { width: 6px; height: 6px; border-radius: 50%; background: currentColor; }
.badge-close { background: none; border: none; cursor: pointer; color: currentColor; font-size: 0.6875rem; padding: 0; margin-left: 0.125rem; }
</style>
```

```vue
<!-- components/ui/UiAlert.vue -->
<script setup lang="ts">
type AlertType = 'info' | 'success' | 'warning' | 'error'

const props = withDefaults(defineProps<{
  type?: AlertType
  title?: string
  message?: string
  closable?: boolean
  show?: boolean
}>(), {
  type: 'info',
  closable: false,
  show: true,
})

const emit = defineEmits<{ close: [] }>()

const icons: Record<AlertType, string> = {
  info: 'ℹ️',
  success: '✅',
  warning: '⚠️',
  error: '❌',
}
</script>

<template>
  <Transition name="alert-fade">
    <div v-if="show" class="alert" :class="`alert--${type}`" role="alert">
      <span class="alert-icon">{{ icons[type] }}</span>
      <div class="alert-content">
        <strong v-if="title" class="alert-title">{{ title }}</strong>
        <p v-if="message" class="alert-message">{{ message }}</p>
        <slot />
      </div>
      <button v-if="closable" @click="emit('close')" class="alert-close">✕</button>
    </div>
  </Transition>
</template>

<style scoped>
.alert { display: flex; align-items: flex-start; gap: 0.75rem; padding: 1rem; border-radius: 8px; border: 1px solid; }
.alert--info { background: #eff6ff; border-color: #bfdbfe; color: #1e40af; }
.alert--success { background: #f0fdf4; border-color: #bbf7d0; color: #166534; }
.alert--warning { background: #fffbeb; border-color: #fde68a; color: #92400e; }
.alert--error { background: #fef2f2; border-color: #fecaca; color: #991b1b; }
.alert-content { flex: 1; }
.alert-title { display: block; font-weight: 600; margin-bottom: 0.25rem; }
.alert-message { margin: 0; font-size: 0.875rem; }
.alert-close { background: none; border: none; cursor: pointer; font-size: 1rem; color: inherit; opacity: 0.7; margin-left: auto; }
.alert-close:hover { opacity: 1; }
.alert-fade-enter-active, .alert-fade-leave-active { transition: all 0.3s ease; }
.alert-fade-enter-from, .alert-fade-leave-to { opacity: 0; transform: translateY(-0.5rem); }
</style>
```

```vue
<!-- App.vue - นำ UI components มาใช้งาน -->
<script setup lang="ts">
import UiButton from './components/ui/UiButton.vue'
import UiInput from './components/ui/UiInput.vue'
import UiBadge from './components/ui/UiBadge.vue'
import UiAlert from './components/ui/UiAlert.vue'
import { ref } from 'vue'

const inputValue = ref('')
const showAlert = ref(true)
const notifications = ref(['แจ้งเตือน 1', 'แจ้งเตือน 2', 'แจ้งเตือน 3'])
</script>

<template>
  <div class="demo-page" style="padding: 2rem; max-width: 800px; margin: 0 auto;">
    <h1>UI Component Library Demo</h1>

    <!-- Buttons -->
    <section class="demo-section">
      <h2>Buttons</h2>

      <div class="button-row">
        <UiButton variant="primary">Primary</UiButton>
        <UiButton variant="secondary">Secondary</UiButton>
        <UiButton variant="danger">Danger</UiButton>
        <UiButton variant="ghost">Ghost</UiButton>
        <UiButton variant="outline">Outline</UiButton>
      </div>

      <div class="button-row">
        <UiButton size="xs">XSmall</UiButton>
        <UiButton size="sm">Small</UiButton>
        <UiButton size="md">Medium</UiButton>
        <UiButton size="lg">Large</UiButton>
      </div>

      <div class="button-row">
        <UiButton :loading="true">Loading</UiButton>
        <UiButton :disabled="true">Disabled</UiButton>
        <UiButton icon="🚀" icon-position="left">กับ Icon</UiButton>
        <UiButton :full-width="true">Full Width</UiButton>
      </div>
    </section>

    <!-- Inputs -->
    <section class="demo-section">
      <h2>Inputs</h2>

      <UiInput
        v-model="inputValue"
        label="ชื่อ"
        placeholder="กรอกชื่อ"
        :required="true"
        :clearable="true"
        hint="ชื่อจริงของคุณ"
      />

      <UiInput
        v-model="inputValue"
        type="email"
        label="อีเมล"
        placeholder="your@email.com"
        error="รูปแบบอีเมลไม่ถูกต้อง"
      />

      <UiInput
        v-model="inputValue"
        label="ราคา"
        type="number"
        prefix="฿"
        suffix="บาท"
        placeholder="0"
      />
    </section>

    <!-- Badges -->
    <section class="demo-section">
      <h2>Badges</h2>

      <div class="badge-row">
        <UiBadge>Default</UiBadge>
        <UiBadge variant="primary">Primary</UiBadge>
        <UiBadge variant="success">Success</UiBadge>
        <UiBadge variant="warning">Warning</UiBadge>
        <UiBadge variant="danger">Danger</UiBadge>
        <UiBadge variant="info">Info</UiBadge>
      </div>

      <div class="badge-row">
        <UiBadge variant="success" dot>Online</UiBadge>
        <UiBadge variant="danger" dot>Offline</UiBadge>
        <UiBadge
          v-for="(notification, index) in notifications"
          :key="index"
          variant="primary"
          closable
          @close="notifications.splice(index, 1)"
        >
          {{ notification }}
        </UiBadge>
      </div>
    </section>

    <!-- Alerts -->
    <section class="demo-section">
      <h2>Alerts</h2>

      <UiAlert
        type="info"
        title="ข้อมูล"
        message="นี่คือ alert แบบ info"
      />

      <UiAlert
        type="success"
        title="สำเร็จ!"
        message="บันทึกข้อมูลเรียบร้อยแล้ว"
        :closable="true"
        :show="showAlert"
        @close="showAlert = false"
      />

      <UiAlert type="warning">
        <template>
          <strong>คำเตือน:</strong> คุณกำลังลบข้อมูลที่ไม่สามารถกู้คืนได้
        </template>
      </UiAlert>

      <UiAlert
        type="error"
        title="เกิดข้อผิดพลาด"
        message="ไม่สามารถเชื่อมต่อกับเซิร์ฟเวอร์ได้"
        :closable="true"
      />

      <UiButton v-if="!showAlert" @click="showAlert = true" variant="outline" size="sm">
        แสดง Alert อีกครั้ง
      </UiButton>
    </section>
  </div>
</template>

<style scoped>
.demo-section { margin-bottom: 3rem; }
.demo-section h2 { border-bottom: 1px solid #e5e7eb; padding-bottom: 0.5rem; margin-bottom: 1.5rem; }
.button-row, .badge-row { display: flex; flex-wrap: wrap; gap: 0.75rem; margin-bottom: 1rem; align-items: center; }
</style>
```

---

## สรุป

| Concept | สรุป |
|---------|------|
| SFC | ไฟล์ `.vue` รวม template + script + style |
| `<script setup>` | Composition API syntax ที่แนะนำ |
| Local registration | import แล้วใช้ได้เลยใน `<script setup>` |
| Global registration | `app.component()` ใน main.ts |
| `defineComponent` | สำหรับ Options API style หรือ type inference |
| Auto-import | ใช้ unplugin-vue-components |

**Best Practices:**
- ใช้ `<script setup lang="ts">` เสมอสำหรับ Vue 3
- ตั้งชื่อ component ด้วย PascalCase (2+ words) เช่น `UserCard` ไม่ใช่ `Card`
- สร้าง Base components สำหรับ UI primitives (Button, Input, Card ฯลฯ)
- ใช้ `scoped` styles เพื่อ isolation
- ใช้ `defineAsyncComponent` สำหรับ heavy components (lazy loading)
- แยก logic ออกเป็น composables เมื่อ component ใหญ่ขึ้น
