# Part 32: Testing - Unit Tests

## ทำไมต้อง Test?

Unit Testing คือการทดสอบส่วนย่อยของโค้ด (component, function, composable) แยกจากส่วนอื่น เพื่อให้แน่ใจว่าแต่ละส่วนทำงานถูกต้อง การ test ช่วยให้:

- **ค้นหา bug ได้เร็ว** - รู้ทันทีเมื่อโค้ดเสีย
- **Refactor อย่างมั่นใจ** - เปลี่ยนโค้ดโดยไม่กลัวพัง
- **Documentation living** - test คือ spec ของโค้ด
- **Design ดีขึ้น** - โค้ดที่ test ได้ง่ายมักมี design ดี

---

## 1. Vitest Setup

Vitest เป็น test framework ที่ออกแบบมาสำหรับ Vite projects โดยเฉพาะ

```bash
# ติดตั้ง Vitest และ dependencies
npm install -D vitest @vue/test-utils jsdom @vitest/coverage-v8

# หรือสำหรับ Nuxt
npm install -D @nuxt/test-utils vitest @vue/test-utils jsdom
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',
    globals: true,            // ใช้ describe, it, expect โดยไม่ import
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'tests/',
        '*.config.*',
        '**/*.d.ts'
      ]
    }
  },
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
      '~': resolve(__dirname, './src')
    }
  }
})
```

```ts
// tests/setup.ts - Global setup สำหรับทุก test
import { config } from '@vue/test-utils'
import { createPinia } from 'pinia'

// Global plugins
config.global.plugins = [createPinia()]

// Global stubs สำหรับ components ที่ใช้บ่อย
config.global.stubs = {
  'NuxtLink': { template: '<a><slot /></a>' },
  'NuxtImg': { template: '<img />' }
}

// Mock console.error ที่ไม่จำเป็น
console.error = vi.fn()
```

```json
// package.json - เพิ่ม test scripts
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui"
  }
}
```

---

## 2. @vue/test-utils เบื้องต้น

```ts
// tests/basic.test.ts
import { mount, shallowMount } from '@vue/test-utils'
import { describe, it, expect } from 'vitest'

// mount vs shallowMount
// mount: render component และ children ทั้งหมด
// shallowMount: stub children components ทั้งหมด

import MyButton from '@/components/MyButton.vue'

describe('MyButton', () => {
  it('renders with default text', () => {
    const wrapper = mount(MyButton, {
      props: { label: 'คลิก' }
    })
    expect(wrapper.text()).toBe('คลิก')
  })

  it('emits click event when clicked', async () => {
    const wrapper = mount(MyButton)
    await wrapper.trigger('click')
    expect(wrapper.emitted('click')).toBeTruthy()
  })

  it('is disabled when disabled prop is true', () => {
    const wrapper = mount(MyButton, {
      props: { disabled: true }
    })
    expect(wrapper.find('button').attributes('disabled')).toBeDefined()
  })
})
```

---

## 3. Testing Components

### การทดสอบ Component ที่ซับซ้อน

```vue
<!-- components/Counter.vue - Component ที่จะทดสอบ -->
<script setup lang="ts">
interface Props {
  initialValue?: number
  min?: number
  max?: number
  step?: number
}

const props = withDefaults(defineProps<Props>(), {
  initialValue: 0,
  min: 0,
  max: 100,
  step: 1
})

const emit = defineEmits<{
  change: [value: number]
}>()

const count = ref(props.initialValue)

const canDecrement = computed(() => count.value > props.min)
const canIncrement = computed(() => count.value < props.max)

function increment() {
  if (canIncrement.value) {
    count.value = Math.min(count.value + props.step, props.max)
    emit('change', count.value)
  }
}

function decrement() {
  if (canDecrement.value) {
    count.value = Math.max(count.value - props.step, props.min)
    emit('change', count.value)
  }
}

function reset() {
  count.value = props.initialValue
  emit('change', count.value)
}
</script>

<template>
  <div class="counter" data-testid="counter">
    <button
      @click="decrement"
      :disabled="!canDecrement"
      data-testid="decrement-btn"
    >
      -
    </button>
    <span data-testid="count-display">{{ count }}</span>
    <button
      @click="increment"
      :disabled="!canIncrement"
      data-testid="increment-btn"
    >
      +
    </button>
    <button @click="reset" data-testid="reset-btn">Reset</button>
  </div>
</template>
```

```ts
// tests/components/Counter.test.ts
import { mount } from '@vue/test-utils'
import { describe, it, expect, beforeEach } from 'vitest'
import Counter from '@/components/Counter.vue'

describe('Counter Component', () => {
  // ทดสอบ Rendering
  describe('Rendering', () => {
    it('renders with default initial value of 0', () => {
      const wrapper = mount(Counter)
      expect(wrapper.get('[data-testid="count-display"]').text()).toBe('0')
    })

    it('renders with custom initial value', () => {
      const wrapper = mount(Counter, {
        props: { initialValue: 5 }
      })
      expect(wrapper.get('[data-testid="count-display"]').text()).toBe('5')
    })

    it('renders all buttons', () => {
      const wrapper = mount(Counter)
      expect(wrapper.get('[data-testid="increment-btn"]').exists()).toBe(true)
      expect(wrapper.get('[data-testid="decrement-btn"]').exists()).toBe(true)
      expect(wrapper.get('[data-testid="reset-btn"]').exists()).toBe(true)
    })
  })

  // ทดสอบ Interactions
  describe('Interactions', () => {
    it('increments count when + button is clicked', async () => {
      const wrapper = mount(Counter)
      await wrapper.get('[data-testid="increment-btn"]').trigger('click')
      expect(wrapper.get('[data-testid="count-display"]').text()).toBe('1')
    })

    it('decrements count when - button is clicked', async () => {
      const wrapper = mount(Counter, {
        props: { initialValue: 5 }
      })
      await wrapper.get('[data-testid="decrement-btn"]').trigger('click')
      expect(wrapper.get('[data-testid="count-display"]').text()).toBe('4')
    })

    it('resets to initial value when Reset is clicked', async () => {
      const wrapper = mount(Counter, {
        props: { initialValue: 10 }
      })
      await wrapper.get('[data-testid="increment-btn"]').trigger('click')
      await wrapper.get('[data-testid="reset-btn"]').trigger('click')
      expect(wrapper.get('[data-testid="count-display"]').text()).toBe('10')
    })

    it('increments by step value', async () => {
      const wrapper = mount(Counter, {
        props: { step: 5 }
      })
      await wrapper.get('[data-testid="increment-btn"]').trigger('click')
      expect(wrapper.get('[data-testid="count-display"]').text()).toBe('5')
    })
  })

  // ทดสอบ Boundaries
  describe('Boundaries', () => {
    it('disables decrement button when at minimum', () => {
      const wrapper = mount(Counter, {
        props: { initialValue: 0, min: 0 }
      })
      expect(wrapper.get('[data-testid="decrement-btn"]').attributes('disabled')).toBeDefined()
    })

    it('disables increment button when at maximum', () => {
      const wrapper = mount(Counter, {
        props: { initialValue: 100, max: 100 }
      })
      expect(wrapper.get('[data-testid="increment-btn"]').attributes('disabled')).toBeDefined()
    })

    it('does not exceed maximum value', async () => {
      const wrapper = mount(Counter, {
        props: { initialValue: 99, max: 100 }
      })
      await wrapper.get('[data-testid="increment-btn"]').trigger('click')
      await wrapper.get('[data-testid="increment-btn"]').trigger('click')
      expect(wrapper.get('[data-testid="count-display"]').text()).toBe('100')
    })
  })

  // ทดสอบ Events
  describe('Events', () => {
    it('emits change event when incremented', async () => {
      const wrapper = mount(Counter)
      await wrapper.get('[data-testid="increment-btn"]').trigger('click')
      expect(wrapper.emitted('change')).toBeTruthy()
      expect(wrapper.emitted('change')?.[0]).toEqual([1])
    })

    it('emits change event with correct value when reset', async () => {
      const wrapper = mount(Counter, {
        props: { initialValue: 5 }
      })
      await wrapper.get('[data-testid="increment-btn"]').trigger('click')
      await wrapper.get('[data-testid="reset-btn"]').trigger('click')
      const events = wrapper.emitted('change') as number[][]
      expect(events[events.length - 1]).toEqual([5])
    })
  })
})
```

### Testing Form Component

```vue
<!-- components/ContactForm.vue -->
<script setup lang="ts">
interface FormData {
  name: string
  email: string
  message: string
}

const emit = defineEmits<{
  submit: [data: FormData]
}>()

const formData = reactive<FormData>({
  name: '',
  email: '',
  message: ''
})

const errors = reactive<Partial<FormData>>({})

function validate(): boolean {
  errors.name = !formData.name ? 'กรุณากรอกชื่อ' : undefined
  errors.email = !formData.email ? 'กรุณากรอก email' :
    !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(formData.email) ? 'Email ไม่ถูกต้อง' : undefined
  errors.message = !formData.message ? 'กรุณากรอกข้อความ' : undefined
  return !Object.values(errors).some(Boolean)
}

function handleSubmit() {
  if (validate()) {
    emit('submit', { ...formData })
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit" data-testid="contact-form">
    <div>
      <input v-model="formData.name" data-testid="name-input" placeholder="ชื่อ" />
      <span v-if="errors.name" data-testid="name-error">{{ errors.name }}</span>
    </div>
    <div>
      <input v-model="formData.email" data-testid="email-input" placeholder="Email" />
      <span v-if="errors.email" data-testid="email-error">{{ errors.email }}</span>
    </div>
    <div>
      <textarea v-model="formData.message" data-testid="message-input" placeholder="ข้อความ" />
      <span v-if="errors.message" data-testid="message-error">{{ errors.message }}</span>
    </div>
    <button type="submit" data-testid="submit-btn">ส่ง</button>
  </form>
</template>
```

```ts
// tests/components/ContactForm.test.ts
import { mount } from '@vue/test-utils'
import { describe, it, expect } from 'vitest'
import ContactForm from '@/components/ContactForm.vue'

describe('ContactForm', () => {
  async function fillForm(wrapper: ReturnType<typeof mount>, data: {
    name?: string
    email?: string
    message?: string
  }) {
    if (data.name !== undefined)
      await wrapper.get('[data-testid="name-input"]').setValue(data.name)
    if (data.email !== undefined)
      await wrapper.get('[data-testid="email-input"]').setValue(data.email)
    if (data.message !== undefined)
      await wrapper.get('[data-testid="message-input"]').setValue(data.message)
  }

  it('shows validation errors when submitted empty', async () => {
    const wrapper = mount(ContactForm)
    await wrapper.get('[data-testid="submit-btn"]').trigger('click')
    
    expect(wrapper.get('[data-testid="name-error"]').text()).toBe('กรุณากรอกชื่อ')
    expect(wrapper.get('[data-testid="email-error"]').text()).toBe('กรุณากรอก email')
    expect(wrapper.get('[data-testid="message-error"]').text()).toBe('กรุณากรอกข้อความ')
  })

  it('shows email format error for invalid email', async () => {
    const wrapper = mount(ContactForm)
    await fillForm(wrapper, { email: 'not-an-email' })
    await wrapper.get('[data-testid="submit-btn"]').trigger('click')
    
    expect(wrapper.get('[data-testid="email-error"]').text()).toBe('Email ไม่ถูกต้อง')
  })

  it('emits submit event with form data when valid', async () => {
    const wrapper = mount(ContactForm)
    await fillForm(wrapper, {
      name: 'สมชาย',
      email: 'somchai@example.com',
      message: 'ทดสอบ'
    })
    await wrapper.get('[data-testid="submit-btn"]').trigger('click')
    
    expect(wrapper.emitted('submit')).toBeTruthy()
    expect(wrapper.emitted('submit')?.[0]).toEqual([{
      name: 'สมชาย',
      email: 'somchai@example.com',
      message: 'ทดสอบ'
    }])
  })

  it('does not emit submit when validation fails', async () => {
    const wrapper = mount(ContactForm)
    await wrapper.get('[data-testid="submit-btn"]').trigger('click')
    expect(wrapper.emitted('submit')).toBeFalsy()
  })
})
```

---

## 4. Testing Composables

```ts
// composables/useCounter.ts
export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  const double = computed(() => count.value * 2)

  function increment(amount = 1) {
    count.value += amount
  }

  function decrement(amount = 1) {
    count.value -= amount
  }

  function reset() {
    count.value = initialValue
  }

  return { count, double, increment, decrement, reset }
}
```

```ts
// tests/composables/useCounter.test.ts
import { describe, it, expect } from 'vitest'
import { useCounter } from '@/composables/useCounter'

describe('useCounter', () => {
  it('initializes with default value of 0', () => {
    const { count } = useCounter()
    expect(count.value).toBe(0)
  })

  it('initializes with custom value', () => {
    const { count } = useCounter(10)
    expect(count.value).toBe(10)
  })

  it('increments count', () => {
    const { count, increment } = useCounter()
    increment()
    expect(count.value).toBe(1)
  })

  it('increments by custom amount', () => {
    const { count, increment } = useCounter()
    increment(5)
    expect(count.value).toBe(5)
  })

  it('decrements count', () => {
    const { count, decrement } = useCounter(10)
    decrement()
    expect(count.value).toBe(9)
  })

  it('resets to initial value', () => {
    const { count, increment, reset } = useCounter(5)
    increment(10)
    reset()
    expect(count.value).toBe(5)
  })

  it('computes double value', () => {
    const { count, double, increment } = useCounter()
    increment(3)
    expect(double.value).toBe(6)
  })
})
```

### Testing Composable with fetch

```ts
// composables/useUser.ts
interface User {
  id: number
  name: string
  email: string
}

export function useUser() {
  const user = ref<User | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  async function fetchUser(id: number) {
    loading.value = true
    error.value = null
    try {
      const response = await fetch(`/api/users/${id}`)
      if (!response.ok) throw new Error('Failed to fetch user')
      user.value = await response.json()
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Unknown error'
    } finally {
      loading.value = false
    }
  }

  return { user, loading, error, fetchUser }
}
```

```ts
// tests/composables/useUser.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { useUser } from '@/composables/useUser'

// Mock global fetch
const mockFetch = vi.fn()
global.fetch = mockFetch

describe('useUser', () => {
  beforeEach(() => {
    mockFetch.mockReset()
  })

  it('initializes with null user', () => {
    const { user, loading, error } = useUser()
    expect(user.value).toBeNull()
    expect(loading.value).toBe(false)
    expect(error.value).toBeNull()
  })

  it('fetches user successfully', async () => {
    const mockUser = { id: 1, name: 'Test User', email: 'test@test.com' }
    mockFetch.mockResolvedValueOnce({
      ok: true,
      json: async () => mockUser
    })

    const { user, loading, error, fetchUser } = useUser()
    const promise = fetchUser(1)
    
    expect(loading.value).toBe(true)
    await promise
    
    expect(user.value).toEqual(mockUser)
    expect(loading.value).toBe(false)
    expect(error.value).toBeNull()
  })

  it('handles fetch error', async () => {
    mockFetch.mockResolvedValueOnce({ ok: false })

    const { user, error, fetchUser } = useUser()
    await fetchUser(1)

    expect(user.value).toBeNull()
    expect(error.value).toBe('Failed to fetch user')
  })

  it('handles network error', async () => {
    mockFetch.mockRejectedValueOnce(new Error('Network error'))

    const { error, fetchUser } = useUser()
    await fetchUser(1)

    expect(error.value).toBe('Network error')
  })
})
```

---

## 5. Testing Stores (Pinia)

```ts
// stores/cart.ts
import { defineStore } from 'pinia'

interface CartItem {
  id: number
  name: string
  price: number
  quantity: number
}

export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])

  const totalItems = computed(() =>
    items.value.reduce((sum, item) => sum + item.quantity, 0)
  )

  const totalPrice = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  function addItem(item: Omit<CartItem, 'quantity'>) {
    const existing = items.value.find(i => i.id === item.id)
    if (existing) {
      existing.quantity++
    } else {
      items.value.push({ ...item, quantity: 1 })
    }
  }

  function removeItem(id: number) {
    items.value = items.value.filter(i => i.id !== id)
  }

  function updateQuantity(id: number, quantity: number) {
    const item = items.value.find(i => i.id === id)
    if (item) {
      item.quantity = Math.max(0, quantity)
      if (item.quantity === 0) removeItem(id)
    }
  }

  function clearCart() {
    items.value = []
  }

  return {
    items,
    totalItems,
    totalPrice,
    addItem,
    removeItem,
    updateQuantity,
    clearCart
  }
})
```

```ts
// tests/stores/cart.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useCartStore } from '@/stores/cart'

describe('useCartStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('starts with empty cart', () => {
    const cart = useCartStore()
    expect(cart.items).toHaveLength(0)
    expect(cart.totalItems).toBe(0)
    expect(cart.totalPrice).toBe(0)
  })

  it('adds item to cart', () => {
    const cart = useCartStore()
    cart.addItem({ id: 1, name: 'Product A', price: 100 })
    
    expect(cart.items).toHaveLength(1)
    expect(cart.items[0].quantity).toBe(1)
    expect(cart.totalItems).toBe(1)
    expect(cart.totalPrice).toBe(100)
  })

  it('increments quantity when same item added', () => {
    const cart = useCartStore()
    cart.addItem({ id: 1, name: 'Product A', price: 100 })
    cart.addItem({ id: 1, name: 'Product A', price: 100 })
    
    expect(cart.items).toHaveLength(1)
    expect(cart.items[0].quantity).toBe(2)
    expect(cart.totalItems).toBe(2)
  })

  it('removes item from cart', () => {
    const cart = useCartStore()
    cart.addItem({ id: 1, name: 'Product A', price: 100 })
    cart.removeItem(1)
    
    expect(cart.items).toHaveLength(0)
  })

  it('updates item quantity', () => {
    const cart = useCartStore()
    cart.addItem({ id: 1, name: 'Product A', price: 100 })
    cart.updateQuantity(1, 5)
    
    expect(cart.items[0].quantity).toBe(5)
    expect(cart.totalItems).toBe(5)
    expect(cart.totalPrice).toBe(500)
  })

  it('removes item when quantity set to 0', () => {
    const cart = useCartStore()
    cart.addItem({ id: 1, name: 'Product A', price: 100 })
    cart.updateQuantity(1, 0)
    
    expect(cart.items).toHaveLength(0)
  })

  it('clears all items', () => {
    const cart = useCartStore()
    cart.addItem({ id: 1, name: 'A', price: 100 })
    cart.addItem({ id: 2, name: 'B', price: 200 })
    cart.clearCart()
    
    expect(cart.items).toHaveLength(0)
    expect(cart.totalPrice).toBe(0)
  })

  it('calculates total price correctly', () => {
    const cart = useCartStore()
    cart.addItem({ id: 1, name: 'A', price: 100 })
    cart.addItem({ id: 2, name: 'B', price: 200 })
    cart.updateQuantity(1, 3)
    
    // 100 * 3 + 200 * 1 = 500
    expect(cart.totalPrice).toBe(500)
  })
})
```

---

## 6. Mocking

### Mocking Modules

```ts
// tests/mocking.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'

// Mock ทั้ง module
vi.mock('@/services/api', () => ({
  fetchPosts: vi.fn(),
  createPost: vi.fn(),
  deletePost: vi.fn()
}))

import { fetchPosts } from '@/services/api'

describe('Mocking Examples', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('mocks fetchPosts to return data', async () => {
    const mockPosts = [{ id: 1, title: 'Test Post' }]
    vi.mocked(fetchPosts).mockResolvedValueOnce(mockPosts)

    const result = await fetchPosts()
    expect(result).toEqual(mockPosts)
    expect(fetchPosts).toHaveBeenCalledOnce()
  })

  it('mocks fetch with different scenarios', async () => {
    // First call returns data
    vi.mocked(fetchPosts).mockResolvedValueOnce([{ id: 1, title: 'Post 1' }])
    // Second call throws error
    vi.mocked(fetchPosts).mockRejectedValueOnce(new Error('Network Error'))

    const firstResult = await fetchPosts()
    expect(firstResult).toHaveLength(1)

    await expect(fetchPosts()).rejects.toThrow('Network Error')
  })
})
```

### Mocking Timers

```ts
// tests/timer.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { useDebounce } from '@/composables/useDebounce'

describe('useDebounce', () => {
  beforeEach(() => {
    vi.useFakeTimers()
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  it('debounces function calls', () => {
    const callback = vi.fn()
    const debouncedFn = useDebounce(callback, 300)

    debouncedFn()
    debouncedFn()
    debouncedFn()

    expect(callback).not.toHaveBeenCalled()
    vi.advanceTimersByTime(300)
    expect(callback).toHaveBeenCalledOnce()
  })
})
```

---

## 7. Snapshot Testing

```ts
// tests/snapshots/Button.test.ts
import { mount } from '@vue/test-utils'
import { describe, it, expect } from 'vitest'
import AppButton from '@/components/AppButton.vue'

describe('AppButton Snapshots', () => {
  it('matches primary button snapshot', () => {
    const wrapper = mount(AppButton, {
      props: { variant: 'primary', label: 'คลิก' }
    })
    expect(wrapper.html()).toMatchSnapshot()
  })

  it('matches secondary button snapshot', () => {
    const wrapper = mount(AppButton, {
      props: { variant: 'secondary', label: 'ยกเลิก' }
    })
    expect(wrapper.html()).toMatchSnapshot()
  })

  it('matches disabled button snapshot', () => {
    const wrapper = mount(AppButton, {
      props: { disabled: true, label: 'ไม่ได้' }
    })
    expect(wrapper.html()).toMatchSnapshot()
  })
})
```

---

## 8. Coverage Reports

```bash
# รัน tests พร้อม coverage
npx vitest run --coverage

# ผลลัพธ์ใน terminal
# ----------|---------|----------|---------|---------|
# File      | % Stmts | % Branch | % Funcs | % Lines |
# ----------|---------|----------|---------|---------|
# All files |   85.71 |    72.22 |   90.00 |   85.71 |
```

```ts
// vitest.config.ts - กำหนด coverage thresholds
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      thresholds: {
        global: {
          branches: 80,
          functions: 80,
          lines: 80,
          statements: 80
        }
      },
      include: ['src/**/*.{ts,vue}'],
      exclude: [
        'src/main.ts',
        'src/**/*.d.ts',
        'src/**/*.test.ts'
      ]
    }
  }
})
```

---

## 9. ตัวอย่าง: Test Suite สำหรับ API Service

```ts
// services/posts.ts - Service ที่จะทดสอบ
interface Post {
  id: number
  title: string
  body: string
  userId: number
}

export class PostService {
  private baseUrl: string

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl
  }

  async getAll(): Promise<Post[]> {
    const res = await fetch(`${this.baseUrl}/posts`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    return res.json()
  }

  async getById(id: number): Promise<Post> {
    const res = await fetch(`${this.baseUrl}/posts/${id}`)
    if (!res.ok) throw new Error(`Post not found: ${id}`)
    return res.json()
  }

  async create(data: Omit<Post, 'id'>): Promise<Post> {
    const res = await fetch(`${this.baseUrl}/posts`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    })
    if (!res.ok) throw new Error('Failed to create post')
    return res.json()
  }

  async update(id: number, data: Partial<Post>): Promise<Post> {
    const res = await fetch(`${this.baseUrl}/posts/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    })
    if (!res.ok) throw new Error('Failed to update post')
    return res.json()
  }

  async delete(id: number): Promise<void> {
    const res = await fetch(`${this.baseUrl}/posts/${id}`, {
      method: 'DELETE'
    })
    if (!res.ok) throw new Error('Failed to delete post')
  }
}
```

```ts
// tests/services/posts.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { PostService } from '@/services/posts'

const mockFetch = vi.fn()
global.fetch = mockFetch

describe('PostService', () => {
  let service: PostService

  beforeEach(() => {
    service = new PostService('https://api.example.com')
    mockFetch.mockReset()
  })

  const mockPosts = [
    { id: 1, title: 'Post 1', body: 'Body 1', userId: 1 },
    { id: 2, title: 'Post 2', body: 'Body 2', userId: 2 }
  ]

  describe('getAll', () => {
    it('fetches all posts', async () => {
      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => mockPosts
      })

      const posts = await service.getAll()
      expect(posts).toEqual(mockPosts)
      expect(mockFetch).toHaveBeenCalledWith(
        'https://api.example.com/posts'
      )
    })

    it('throws on HTTP error', async () => {
      mockFetch.mockResolvedValueOnce({ ok: false, status: 500 })
      await expect(service.getAll()).rejects.toThrow('HTTP 500')
    })
  })

  describe('getById', () => {
    it('fetches post by id', async () => {
      const mockPost = mockPosts[0]
      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => mockPost
      })

      const post = await service.getById(1)
      expect(post).toEqual(mockPost)
      expect(mockFetch).toHaveBeenCalledWith(
        'https://api.example.com/posts/1'
      )
    })

    it('throws when post not found', async () => {
      mockFetch.mockResolvedValueOnce({ ok: false })
      await expect(service.getById(999)).rejects.toThrow('Post not found: 999')
    })
  })

  describe('create', () => {
    it('creates new post', async () => {
      const newPost = { title: 'New', body: 'Content', userId: 1 }
      const createdPost = { id: 3, ...newPost }

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => createdPost
      })

      const result = await service.create(newPost)
      expect(result).toEqual(createdPost)
      expect(mockFetch).toHaveBeenCalledWith(
        'https://api.example.com/posts',
        expect.objectContaining({
          method: 'POST',
          body: JSON.stringify(newPost)
        })
      )
    })
  })

  describe('delete', () => {
    it('deletes a post successfully', async () => {
      mockFetch.mockResolvedValueOnce({ ok: true })
      await expect(service.delete(1)).resolves.not.toThrow()
    })

    it('throws when delete fails', async () => {
      mockFetch.mockResolvedValueOnce({ ok: false })
      await expect(service.delete(1)).rejects.toThrow('Failed to delete post')
    })
  })
})
```

---

## สรุป

Unit Testing ด้วย Vitest และ @vue/test-utils ช่วยให้:

1. **Vitest Setup** - ตั้งค่า environment, globals, และ setup files
2. **@vue/test-utils** - mount component, trigger events, find elements
3. **Testing Components** - ทดสอบ rendering, interactions, events, boundaries
4. **Testing Composables** - ทดสอบ logic แยกจาก component
5. **Testing Stores** - ทดสอบ Pinia store ด้วย `setActivePinia`
6. **Mocking** - mock modules, timers, และ fetch
7. **Snapshots** - บันทึก HTML snapshot สำหรับ regression testing
8. **Coverage** - วัด code coverage และตั้ง thresholds
