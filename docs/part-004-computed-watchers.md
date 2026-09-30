# Part 4: Computed Properties และ Watchers

## บทนำ

Computed Properties และ Watchers เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ Vue.js ที่ช่วยให้เราจัดการ reactive data ได้อย่างมีประสิทธิภาพ Computed Properties ช่วยให้เราคำนวณค่าจาก reactive state โดยอัตโนมัติ ในขณะที่ Watchers ช่วยให้เรา "เฝ้าดู" การเปลี่ยนแปลงของ state และทำงานบางอย่างเพื่อตอบสนอง

---

## 1. computed() พื้นฐาน

`computed()` ใช้สำหรับสร้าง computed property ที่จะคำนวณค่าใหม่เมื่อ dependencies เปลี่ยนแปลง และ **cache** ผลลัพธ์ไว้จนกว่า dependencies จะเปลี่ยน

### ทำไมถึงใช้ computed แทน method?

- **Caching**: computed จะไม่คำนวณใหม่ถ้า dependencies ไม่เปลี่ยน
- **Readability**: โค้ดอ่านง่ายกว่า method calls ในหลายๆ กรณี
- **Performance**: เหมาะกับการคำนวณที่ cost สูง

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// ข้อมูลพื้นฐาน
const firstName = ref('สมชาย')
const lastName = ref('ใจดี')
const birthYear = ref(1990)

// computed property พื้นฐาน - จะ cache ค่าไว้
const fullName = computed(() => {
  console.log('คำนวณ fullName...') // จะ log เฉพาะเมื่อ firstName หรือ lastName เปลี่ยน
  return `${firstName.value} ${lastName.value}`
})

// computed ที่คำนวณจาก computed อื่น
const currentYear = new Date().getFullYear()
const age = computed(() => currentYear - birthYear.value)

const ageCategory = computed(() => {
  if (age.value < 18) return 'เยาวชน'
  if (age.value < 60) return 'ผู้ใหญ่'
  return 'ผู้สูงอายุ'
})

// computed กับ array
const items = ref([
  { id: 1, name: 'Apple', price: 15, inStock: true },
  { id: 2, name: 'Banana', price: 8, inStock: false },
  { id: 3, name: 'Cherry', price: 45, inStock: true },
  { id: 4, name: 'Durian', price: 200, inStock: true },
])

const availableItems = computed(() =>
  items.value.filter(item => item.inStock)
)

const totalPrice = computed(() =>
  availableItems.value.reduce((sum, item) => sum + item.price, 0)
)

const averagePrice = computed(() =>
  availableItems.value.length > 0
    ? totalPrice.value / availableItems.value.length
    : 0
)
</script>

<template>
  <div class="computed-demo">
    <h2>Computed Properties Demo</h2>

    <!-- ข้อมูลส่วนตัว -->
    <div class="section">
      <h3>ข้อมูลส่วนตัว</h3>
      <input v-model="firstName" placeholder="ชื่อ" />
      <input v-model="lastName" placeholder="นามสกุล" />
      <input v-model.number="birthYear" type="number" placeholder="ปีเกิด" />

      <p>ชื่อเต็ม: <strong>{{ fullName }}</strong></p>
      <p>อายุ: <strong>{{ age }} ปี</strong> ({{ ageCategory }})</p>
    </div>

    <!-- รายการสินค้า -->
    <div class="section">
      <h3>สินค้าที่มีในสต็อก ({{ availableItems.length }} รายการ)</h3>
      <ul>
        <li v-for="item in availableItems" :key="item.id">
          {{ item.name }} - ฿{{ item.price }}
        </li>
      </ul>
      <p>ราคารวม: ฿{{ totalPrice }}</p>
      <p>ราคาเฉลี่ย: ฿{{ averagePrice.toFixed(2) }}</p>
    </div>
  </div>
</template>
```

---

## 2. Writable Computed (Getter/Setter)

โดยปกติ computed จะเป็น read-only แต่เราสามารถสร้าง writable computed โดยการกำหนด `get` และ `set`

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const firstName = ref('สมชาย')
const lastName = ref('ใจดี')

// Writable computed - มีทั้ง getter และ setter
const fullName = computed({
  get() {
    return `${firstName.value} ${lastName.value}`
  },
  set(newValue: string) {
    const parts = newValue.trim().split(' ')
    firstName.value = parts[0] || ''
    lastName.value = parts.slice(1).join(' ') || ''
  }
})

// ตัวอย่าง: Temperature converter
const celsius = ref(100)

const fahrenheit = computed({
  get() {
    return (celsius.value * 9) / 5 + 32
  },
  set(newFahrenheit: number) {
    celsius.value = ((newFahrenheit - 32) * 5) / 9
  }
})

// ตัวอย่าง: Select all checkbox
const items = ref([
  { id: 1, name: 'Vue.js', selected: false },
  { id: 2, name: 'React', selected: false },
  { id: 3, name: 'Angular', selected: false },
  { id: 4, name: 'Svelte', selected: false },
])

const selectAll = computed({
  get() {
    return items.value.length > 0 && items.value.every(item => item.selected)
  },
  set(value: boolean) {
    items.value.forEach(item => (item.selected = value))
  }
})

const selectedCount = computed(() => items.value.filter(i => i.selected).length)
</script>

<template>
  <div class="writable-computed-demo">
    <h2>Writable Computed Demo</h2>

    <!-- ชื่อเต็ม -->
    <div class="section">
      <h3>แก้ไขชื่อ</h3>
      <p>ชื่อ: {{ firstName }}, นามสกุล: {{ lastName }}</p>
      <input v-model="fullName" placeholder="พิมพ์ชื่อเต็ม (ชื่อ นามสกุล)" />
    </div>

    <!-- แปลงอุณหภูมิ -->
    <div class="section">
      <h3>แปลงอุณหภูมิ</h3>
      <label>
        Celsius: <input v-model.number="celsius" type="number" />°C
      </label>
      <label>
        Fahrenheit: <input v-model.number="fahrenheit" type="number" />°F
      </label>
    </div>

    <!-- Select All -->
    <div class="section">
      <h3>เลือกทั้งหมด ({{ selectedCount }}/{{ items.length }})</h3>
      <label>
        <input type="checkbox" v-model="selectAll" />
        เลือกทั้งหมด
      </label>
      <div v-for="item in items" :key="item.id">
        <label>
          <input type="checkbox" v-model="item.selected" />
          {{ item.name }}
        </label>
      </div>
    </div>
  </div>
</template>
```

---

## 3. Computed กับ TypeScript

การใช้ computed กับ TypeScript ช่วยให้โค้ดมี type safety ที่ดีขึ้น

```typescript
// types.ts
interface Product {
  id: number
  name: string
  price: number
  category: string
  rating: number
  stock: number
}

interface CartItem {
  product: Product
  quantity: number
}

interface CartSummary {
  itemCount: number
  subtotal: number
  discount: number
  tax: number
  total: number
}
```

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// กำหนด types
interface Product {
  id: number
  name: string
  price: number
  category: string
  rating: number
  stock: number
}

interface CartItem {
  product: Product
  quantity: number
}

const products = ref<Product[]>([
  { id: 1, name: 'Vue.js Book', price: 599, category: 'books', rating: 4.8, stock: 10 },
  { id: 2, name: 'TypeScript Course', price: 1299, category: 'courses', rating: 4.9, stock: 5 },
  { id: 3, name: 'Mechanical Keyboard', price: 3500, category: 'hardware', rating: 4.5, stock: 3 },
])

const cart = ref<CartItem[]>([])
const searchQuery = ref('')
const selectedCategory = ref<string>('all')
const sortBy = ref<'name' | 'price' | 'rating'>('name')
const discountCode = ref('')

// Typed computed properties
const filteredProducts = computed<Product[]>(() => {
  let result = products.value

  // กรองตาม category
  if (selectedCategory.value !== 'all') {
    result = result.filter(p => p.category === selectedCategory.value)
  }

  // กรองตาม search query
  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase()
    result = result.filter(p =>
      p.name.toLowerCase().includes(query) ||
      p.category.toLowerCase().includes(query)
    )
  }

  // เรียงลำดับ
  return [...result].sort((a, b) => {
    if (sortBy.value === 'name') return a.name.localeCompare(b.name)
    if (sortBy.value === 'price') return a.price - b.price
    return b.rating - a.rating
  })
})

const categories = computed<string[]>(() => {
  const cats = [...new Set(products.value.map(p => p.category))]
  return ['all', ...cats]
})

const cartSummary = computed(() => {
  const itemCount = cart.value.reduce((sum, item) => sum + item.quantity, 0)
  const subtotal = cart.value.reduce(
    (sum, item) => sum + item.product.price * item.quantity,
    0
  )

  const discount = discountCode.value === 'SAVE20'
    ? subtotal * 0.2
    : discountCode.value === 'SAVE10'
      ? subtotal * 0.1
      : 0

  const taxableAmount = subtotal - discount
  const tax = taxableAmount * 0.07
  const total = taxableAmount + tax

  return {
    itemCount,
    subtotal,
    discount,
    tax,
    total,
  }
})

// Helper functions
function addToCart(product: Product): void {
  const existing = cart.value.find(item => item.product.id === product.id)
  if (existing) {
    existing.quantity++
  } else {
    cart.value.push({ product, quantity: 1 })
  }
}

function removeFromCart(productId: number): void {
  cart.value = cart.value.filter(item => item.product.id !== productId)
}
</script>
```

---

## 4. watch() พื้นฐาน

`watch()` ใช้สำหรับเฝ้าดูการเปลี่ยนแปลงของ reactive data และทำงานบางอย่างเพื่อตอบสนอง

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const count = ref(0)
const user = ref({
  name: 'สมชาย',
  email: 'somchai@example.com',
  preferences: {
    theme: 'light',
    language: 'th'
  }
})

// watch single ref
watch(count, (newValue, oldValue) => {
  console.log(`count เปลี่ยนจาก ${oldValue} เป็น ${newValue}`)
})

// watch หลาย sources พร้อมกัน
const firstName = ref('สมชาย')
const lastName = ref('ใจดี')

watch([firstName, lastName], ([newFirst, newLast], [oldFirst, oldLast]) => {
  console.log(`ชื่อเปลี่ยน: ${oldFirst} ${oldLast} -> ${newFirst} ${newLast}`)
})

// watch getter function
watch(
  () => user.value.preferences.theme,
  (newTheme, oldTheme) => {
    console.log(`Theme เปลี่ยนจาก ${oldTheme} เป็น ${newTheme}`)
    // อาจบันทึก preference ไปยัง localStorage
    localStorage.setItem('theme', newTheme)
  }
)

// cleanup ใน watch
const searchQuery = ref('')

watch(searchQuery, (newQuery, oldQuery, onCleanup) => {
  let cancelled = false

  // onCleanup จะถูกเรียกเมื่อ watch ถูก trigger ใหม่หรือ component ถูก unmount
  onCleanup(() => {
    cancelled = true
    console.log('ยกเลิก request ก่อนหน้า')
  })

  // simulate API call
  setTimeout(() => {
    if (!cancelled) {
      console.log(`ค้นหา: ${newQuery}`)
    }
  }, 500)
})
</script>
```

---

## 5. watch Options (immediate, deep, flush)

### immediate

เรียก callback ทันทีหลังจาก watch ถูกสร้าง แม้ค่าจะยังไม่เปลี่ยน

```vue
<script setup lang="ts">
import { ref, reactive, watch } from 'vue'

const userId = ref(1)
const userData = ref(null)
const loading = ref(false)

// immediate: true - เรียก callback ทันทีเมื่อ component mount
watch(
  userId,
  async (id) => {
    loading.value = true
    try {
      // fetch user data
      const response = await fetch(`/api/users/${id}`)
      userData.value = await response.json()
    } catch (error) {
      console.error(error)
    } finally {
      loading.value = false
    }
  },
  { immediate: true }
)
</script>
```

### deep

ใช้สำหรับ watch nested objects

```vue
<script setup lang="ts">
import { ref, reactive, watch } from 'vue'

const settings = reactive({
  theme: 'light',
  fontSize: 16,
  notifications: {
    email: true,
    push: false,
    sms: false
  },
  shortcuts: ['Ctrl+S', 'Ctrl+Z']
})

// deep: true - watch การเปลี่ยนแปลงของ nested properties ทั้งหมด
watch(
  settings,
  (newSettings) => {
    console.log('Settings เปลี่ยน:', JSON.stringify(newSettings))
    // บันทึกลง localStorage
    localStorage.setItem('app-settings', JSON.stringify(newSettings))
  },
  { deep: true }
)

// watch specific nested property (ดีกว่าใช้ deep ทั้งหมด)
watch(
  () => settings.notifications,
  (newNotifications) => {
    console.log('Notification settings เปลี่ยน:', newNotifications)
  },
  { deep: true }
)
</script>
```

### flush

ควบคุมว่า callback จะถูกเรียกเมื่อไหร่ relative กับ DOM updates

```vue
<script setup lang="ts">
import { ref, watch, nextTick } from 'vue'

const message = ref('Hello')
const container = ref<HTMLElement | null>(null)

// flush: 'pre' (default) - เรียกก่อน DOM update
watch(
  message,
  (newMsg) => {
    // DOM ยังไม่ถูก update
    console.log('pre flush - DOM ยังเก่าอยู่')
    console.log('container text:', container.value?.textContent)
  },
  { flush: 'pre' }
)

// flush: 'post' - เรียกหลัง DOM update
watch(
  message,
  (newMsg) => {
    // DOM ถูก update แล้ว
    console.log('post flush - DOM ถูก update แล้ว')
    console.log('container text:', container.value?.textContent)
  },
  { flush: 'post' }
)

// flush: 'sync' - เรียกพร้อมกันทันที (ระวังใช้)
watch(
  message,
  (newMsg) => {
    console.log('sync flush - เรียกทันทีเมื่อค่าเปลี่ยน')
  },
  { flush: 'sync' }
)
</script>

<template>
  <div ref="container">{{ message }}</div>
  <button @click="message = 'World'">เปลี่ยน</button>
</template>
```

---

## 6. watchEffect()

`watchEffect()` จะ track dependencies โดยอัตโนมัติและรัน callback ทันที (ไม่ต้องระบุ sources)

```vue
<script setup lang="ts">
import { ref, watchEffect, watchPostEffect } from 'vue'

const url = ref('/api/data')
const params = ref({ page: 1, limit: 10 })
const data = ref(null)
const error = ref(null)
const loading = ref(false)

// watchEffect จะ track url และ params.value โดยอัตโนมัติ
watchEffect(async (onCleanup) => {
  const controller = new AbortController()

  onCleanup(() => {
    controller.abort() // ยกเลิก request เมื่อ effect ถูก re-run
  })

  loading.value = true
  error.value = null

  try {
    const queryString = new URLSearchParams({
      page: String(params.value.page),
      limit: String(params.value.limit),
    }).toString()

    const response = await fetch(`${url.value}?${queryString}`, {
      signal: controller.signal
    })

    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`)

    data.value = await response.json()
  } catch (e: any) {
    if (e.name !== 'AbortError') {
      error.value = e.message
    }
  } finally {
    loading.value = false
  }
})

// ตัวอย่าง: Auto-scroll ไปยัง element ที่ active
const activeId = ref<string | null>(null)

watchEffect(() => {
  if (activeId.value) {
    // จะ track activeId.value โดยอัตโนมัติ
    const el = document.getElementById(activeId.value)
    el?.scrollIntoView({ behavior: 'smooth' })
  }
})

// ตัวอย่าง: Document title
const pageTitle = ref('หน้าหลัก')
const notificationCount = ref(0)

watchEffect(() => {
  // track ทั้ง pageTitle และ notificationCount
  const count = notificationCount.value
  document.title = count > 0
    ? `(${count}) ${pageTitle.value} - My App`
    : `${pageTitle.value} - My App`
})
</script>
```

### เปรียบเทียบ watch vs watchEffect

```typescript
// watch: ระบุ sources อย่างชัดเจน, ได้รับ old/new values
watch(source, (newValue, oldValue) => {
  // ควบคุมได้ว่าจะ watch อะไร
  // รู้ค่าเก่าและค่าใหม่
  // ไม่รัน immediately โดย default
})

// watchEffect: track อัตโนมัติ, รัน immediately เสมอ
watchEffect(() => {
  // track ทุก reactive value ที่เข้าถึงใน callback
  // ไม่มี old value
  // รัน immediately เสมอ
  console.log(source.value) // source ถูก track โดยอัตโนมัติ
})
```

---

## 7. watchPostEffect() และ watchSyncEffect()

### watchPostEffect()

เป็น alias ของ `watchEffect` กับ `{ flush: 'post' }` - รัน callback หลัง DOM update

```vue
<script setup lang="ts">
import { ref, watchPostEffect } from 'vue'

const items = ref<string[]>([])
const listContainer = ref<HTMLElement | null>(null)

// watchPostEffect รัน callback หลัง DOM update
// ใช้เมื่อต้องการ access DOM หลังจาก Vue update
watchPostEffect(() => {
  if (listContainer.value && items.value.length > 0) {
    // DOM ถูก update แล้ว สามารถ access ได้
    const lastItem = listContainer.value.lastElementChild
    lastItem?.scrollIntoView({ behavior: 'smooth' })
  }
})

function addItem() {
  items.value.push(`Item ${items.value.length + 1}`)
}
</script>

<template>
  <div>
    <div ref="listContainer" class="list-container">
      <div v-for="(item, index) in items" :key="index" class="item">
        {{ item }}
      </div>
    </div>
    <button @click="addItem">เพิ่มรายการ</button>
  </div>
</template>
```

### watchSyncEffect()

เป็น alias ของ `watchEffect` กับ `{ flush: 'sync' }` - รัน callback ทันทีแบบ synchronous

```vue
<script setup lang="ts">
import { ref, watchSyncEffect } from 'vue'

const value = ref(0)
const log: string[] = []

// watchSyncEffect รัน callback แบบ synchronous ทันที
// ระวัง: อาจทำให้ performance แย่ลงถ้าใช้มากเกินไป
watchSyncEffect(() => {
  log.push(`Value changed to: ${value.value} at ${Date.now()}`)
})

// ใช้กรณีที่ต้องการ synchronous behavior เช่น
// การ sync state กับ external library
const externalLib = {
  setState(val: number) {
    console.log('External lib state:', val)
  }
}

watchSyncEffect(() => {
  externalLib.setState(value.value)
})
</script>
```

---

## 8. ตัวอย่าง Real-world: Form Validation

```vue
<script setup lang="ts">
import { ref, computed, watch, reactive } from 'vue'

// Types
interface FormData {
  username: string
  email: string
  password: string
  confirmPassword: string
  phone: string
  birthDate: string
}

interface FieldState {
  touched: boolean
  validating: boolean
  serverError: string | null
}

type FormErrors = Partial<Record<keyof FormData, string>>
type FormTouched = Partial<Record<keyof FormData, boolean>>

// Form state
const form = reactive<FormData>({
  username: '',
  email: '',
  password: '',
  confirmPassword: '',
  phone: '',
  birthDate: '',
})

const touched = reactive<FormTouched>({})
const serverErrors = reactive<Partial<Record<keyof FormData, string>>>({})
const validatingUsername = ref(false)
const isSubmitting = ref(false)
const submitSuccess = ref(false)

// Validation rules
const errors = computed<FormErrors>(() => {
  const e: FormErrors = {}

  // Username validation
  if (form.username.length < 3) {
    e.username = 'ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร'
  } else if (form.username.length > 20) {
    e.username = 'ชื่อผู้ใช้ต้องไม่เกิน 20 ตัวอักษร'
  } else if (!/^[a-zA-Z0-9_]+$/.test(form.username)) {
    e.username = 'ชื่อผู้ใช้ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _'
  } else if (serverErrors.username) {
    e.username = serverErrors.username
  }

  // Email validation
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  if (!form.email) {
    e.email = 'กรุณากรอกอีเมล'
  } else if (!emailRegex.test(form.email)) {
    e.email = 'รูปแบบอีเมลไม่ถูกต้อง'
  }

  // Password validation
  if (form.password.length < 8) {
    e.password = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร'
  } else if (!/[A-Z]/.test(form.password)) {
    e.password = 'รหัสผ่านต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว'
  } else if (!/[0-9]/.test(form.password)) {
    e.password = 'รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว'
  }

  // Confirm password
  if (form.confirmPassword !== form.password) {
    e.confirmPassword = 'รหัสผ่านไม่ตรงกัน'
  }

  // Phone validation (Thai format)
  if (form.phone && !/^(0[689]\d{8}|0[2-9]\d{7})$/.test(form.phone)) {
    e.phone = 'รูปแบบเบอร์โทรศัพท์ไม่ถูกต้อง'
  }

  // Birth date
  if (form.birthDate) {
    const birthDate = new Date(form.birthDate)
    const age = (Date.now() - birthDate.getTime()) / (365.25 * 24 * 60 * 60 * 1000)
    if (age < 13) {
      e.birthDate = 'ต้องมีอายุอย่างน้อย 13 ปี'
    }
  }

  return e
})

const isFormValid = computed(() =>
  Object.keys(errors.value).length === 0 &&
  form.username &&
  form.email &&
  form.password &&
  form.confirmPassword
)

const passwordStrength = computed(() => {
  const p = form.password
  let strength = 0
  if (p.length >= 8) strength++
  if (p.length >= 12) strength++
  if (/[A-Z]/.test(p)) strength++
  if (/[a-z]/.test(p)) strength++
  if (/[0-9]/.test(p)) strength++
  if (/[^a-zA-Z0-9]/.test(p)) strength++

  if (strength <= 2) return { label: 'อ่อน', color: 'red', level: 1 }
  if (strength <= 4) return { label: 'ปานกลาง', color: 'orange', level: 2 }
  return { label: 'แข็งแรง', color: 'green', level: 3 }
})

// Watch username เพื่อตรวจสอบกับ server
let usernameCheckTimer: ReturnType<typeof setTimeout> | null = null

watch(
  () => form.username,
  async (newUsername) => {
    // ล้าง server error เมื่อ username เปลี่ยน
    serverErrors.username = undefined

    if (usernameCheckTimer) clearTimeout(usernameCheckTimer)

    if (newUsername.length >= 3 && !errors.value.username) {
      validatingUsername.value = true

      usernameCheckTimer = setTimeout(async () => {
        try {
          // Simulate API check
          await new Promise(resolve => setTimeout(resolve, 1000))
          const takenUsernames = ['admin', 'user', 'test', 'demo']
          if (takenUsernames.includes(newUsername.toLowerCase())) {
            serverErrors.username = 'ชื่อผู้ใช้นี้ถูกใช้งานแล้ว'
          }
        } finally {
          validatingUsername.value = false
        }
      }, 500)
    } else {
      validatingUsername.value = false
    }
  }
)

function touch(field: keyof FormData) {
  touched[field] = true
}

function showError(field: keyof FormData): string | undefined {
  return touched[field] ? errors.value[field] : undefined
}

async function handleSubmit() {
  // Mark all fields as touched
  Object.keys(form).forEach(key => {
    touched[key as keyof FormData] = true
  })

  if (!isFormValid.value) return

  isSubmitting.value = true

  try {
    await new Promise(resolve => setTimeout(resolve, 1500))
    submitSuccess.value = true
  } catch (error) {
    console.error('Submit error:', error)
  } finally {
    isSubmitting.value = false
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit" class="registration-form">
    <h2>สมัครสมาชิก</h2>

    <div v-if="submitSuccess" class="success-message">
      สมัครสมาชิกสำเร็จ! ยินดีต้อนรับ
    </div>

    <template v-else>
      <!-- Username field -->
      <div class="form-group" :class="{ error: showError('username') }">
        <label>ชื่อผู้ใช้ *</label>
        <div class="input-wrapper">
          <input
            v-model="form.username"
            @blur="touch('username')"
            placeholder="กรอกชื่อผู้ใช้"
          />
          <span v-if="validatingUsername" class="validating">กำลังตรวจสอบ...</span>
        </div>
        <span class="error-msg">{{ showError('username') }}</span>
      </div>

      <!-- Email field -->
      <div class="form-group" :class="{ error: showError('email') }">
        <label>อีเมล *</label>
        <input
          v-model="form.email"
          type="email"
          @blur="touch('email')"
          placeholder="your@email.com"
        />
        <span class="error-msg">{{ showError('email') }}</span>
      </div>

      <!-- Password field -->
      <div class="form-group" :class="{ error: showError('password') }">
        <label>รหัสผ่าน *</label>
        <input
          v-model="form.password"
          type="password"
          @blur="touch('password')"
          placeholder="กรอกรหัสผ่าน"
        />
        <div v-if="form.password" class="password-strength">
          <span>ความแข็งแรง: </span>
          <span :style="{ color: passwordStrength.color }">
            {{ passwordStrength.label }}
          </span>
          <div class="strength-bar">
            <div
              class="strength-fill"
              :style="{
                width: `${(passwordStrength.level / 3) * 100}%`,
                backgroundColor: passwordStrength.color
              }"
            ></div>
          </div>
        </div>
        <span class="error-msg">{{ showError('password') }}</span>
      </div>

      <!-- Confirm Password -->
      <div class="form-group" :class="{ error: showError('confirmPassword') }">
        <label>ยืนยันรหัสผ่าน *</label>
        <input
          v-model="form.confirmPassword"
          type="password"
          @blur="touch('confirmPassword')"
          placeholder="กรอกรหัสผ่านอีกครั้ง"
        />
        <span class="error-msg">{{ showError('confirmPassword') }}</span>
      </div>

      <button
        type="submit"
        :disabled="!isFormValid || isSubmitting"
        class="submit-btn"
      >
        {{ isSubmitting ? 'กำลังสมัคร...' : 'สมัครสมาชิก' }}
      </button>
    </template>
  </form>
</template>
```

---

## 9. ตัวอย่าง: Auto-save Feature

```vue
<script setup lang="ts">
import { ref, reactive, watch, computed } from 'vue'

interface Document {
  id: string
  title: string
  content: string
  lastSaved: Date | null
  version: number
}

const document = reactive<Document>({
  id: 'doc-1',
  title: 'เอกสารใหม่',
  content: '',
  lastSaved: null,
  version: 0,
})

const saveStatus = ref<'saved' | 'saving' | 'unsaved' | 'error'>('saved')
const saveError = ref<string | null>(null)
const isOnline = ref(navigator.onLine)
const saveHistory = ref<{ content: string; savedAt: Date }[]>([])

// Auto-save ทุกครั้งที่ content เปลี่ยน (debounced)
let autoSaveTimer: ReturnType<typeof setTimeout> | null = null

watch(
  [() => document.title, () => document.content],
  () => {
    saveStatus.value = 'unsaved'
    saveError.value = null

    if (autoSaveTimer) clearTimeout(autoSaveTimer)

    // Auto-save หลังจาก user หยุดพิมพ์ 2 วินาที
    autoSaveTimer = setTimeout(async () => {
      if (!isOnline.value) {
        // บันทึกใน localStorage เมื่อ offline
        localStorage.setItem(`doc-${document.id}`, JSON.stringify({
          title: document.title,
          content: document.content,
          savedAt: new Date().toISOString(),
        }))
        saveStatus.value = 'saved'
        return
      }

      await saveDocument()
    }, 2000)
  },
  { deep: true }
)

// Watch online status เพื่อ sync เมื่อกลับมา online
watch(isOnline, async (online) => {
  if (online && saveStatus.value === 'unsaved') {
    await saveDocument()
  }
})

async function saveDocument() {
  saveStatus.value = 'saving'
  saveError.value = null

  try {
    // Simulate API call
    await new Promise<void>((resolve, reject) => {
      setTimeout(() => {
        if (Math.random() > 0.1) { // 90% success rate
          resolve()
        } else {
          reject(new Error('Network error'))
        }
      }, 1000)
    })

    document.lastSaved = new Date()
    document.version++
    saveStatus.value = 'saved'

    // บันทึก history
    saveHistory.value.unshift({
      content: document.content,
      savedAt: document.lastSaved,
    })

    // เก็บแค่ 10 versions ล่าสุด
    if (saveHistory.value.length > 10) {
      saveHistory.value.pop()
    }

  } catch (error: any) {
    saveStatus.value = 'error'
    saveError.value = error.message
  }
}

const statusText = computed(() => {
  switch (saveStatus.value) {
    case 'saved':
      return document.lastSaved
        ? `บันทึกแล้ว ${formatRelativeTime(document.lastSaved)}`
        : 'ยังไม่ได้บันทึก'
    case 'saving': return 'กำลังบันทึก...'
    case 'unsaved': return 'มีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก'
    case 'error': return `เกิดข้อผิดพลาด: ${saveError.value}`
  }
})

function formatRelativeTime(date: Date): string {
  const seconds = Math.floor((Date.now() - date.getTime()) / 1000)
  if (seconds < 60) return `${seconds} วินาทีที่แล้ว`
  const minutes = Math.floor(seconds / 60)
  if (minutes < 60) return `${minutes} นาทีที่แล้ว`
  const hours = Math.floor(minutes / 60)
  return `${hours} ชั่วโมงที่แล้ว`
}

// Handle online/offline events
window.addEventListener('online', () => { isOnline.value = true })
window.addEventListener('offline', () => { isOnline.value = false })
</script>

<template>
  <div class="auto-save-editor">
    <header class="editor-header">
      <input
        v-model="document.title"
        class="title-input"
        placeholder="ชื่อเอกสาร"
      />
      <div class="save-status" :class="saveStatus">
        <span v-if="!isOnline" class="offline-badge">Offline</span>
        {{ statusText }}
        <span v-if="document.version > 0">v{{ document.version }}</span>
      </div>
    </header>

    <textarea
      v-model="document.content"
      class="editor-content"
      placeholder="เริ่มพิมพ์เนื้อหา... จะบันทึกอัตโนมัติเมื่อหยุดพิมพ์"
      rows="20"
    />

    <div v-if="saveHistory.length > 0" class="save-history">
      <h4>ประวัติการบันทึก</h4>
      <ul>
        <li v-for="(entry, index) in saveHistory" :key="index">
          {{ formatRelativeTime(entry.savedAt) }} -
          {{ entry.content.substring(0, 50) }}...
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.auto-save-editor { max-width: 800px; margin: 0 auto; }
.editor-header { display: flex; justify-content: space-between; align-items: center; padding: 1rem; border-bottom: 1px solid #eee; }
.title-input { font-size: 1.5rem; border: none; outline: none; flex: 1; }
.save-status { font-size: 0.875rem; color: #666; }
.save-status.saving { color: #3b82f6; }
.save-status.unsaved { color: #f59e0b; }
.save-status.error { color: #ef4444; }
.save-status.saved { color: #10b981; }
.editor-content { width: 100%; padding: 1rem; border: none; resize: vertical; font-size: 1rem; }
</style>
```

---

## สรุป

| Feature | ใช้เมื่อ |
|---------|---------|
| `computed` | ต้องการ derived value ที่ cache ได้ |
| `writable computed` | ต้องการ two-way binding กับ computed value |
| `watch` | ต้องการ old/new values หรือ lazy execution |
| `watchEffect` | ต้องการ auto-track dependencies, run immediately |
| `watchPostEffect` | ต้องการ access DOM หลัง update |
| `watchSyncEffect` | ต้องการ synchronous execution |

**Best Practices:**
- ใช้ `computed` แทน `watch` เมื่อเป็นไปได้
- ใช้ `watchEffect` สำหรับ side effects ที่ track dependencies หลายอย่าง
- เสมอ cleanup ใน `watch` เมื่อทำ async operations
- ใช้ `deep: true` เฉพาะเมื่อจำเป็น เพราะ cost สูง
- ใช้ getter function `() => obj.property` แทน `deep` เมื่อ watch specific property
