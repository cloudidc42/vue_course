# Part 003: Reactivity System - ref และ reactive

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** 3-4 ชั่วโมง | **ขั้นตอน:** 61-90

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Reactivity System ทำงานอย่างไร
- `ref()` - Reactive Reference
- `reactive()` - Reactive Object
- `readonly()` - Immutable Data
- `shallowRef()` และ `shallowReactive()`
- `toRef()` และ `toRefs()`
- `isRef()`, `isReactive()` - Type Guards
- Best Practices

---

## ขั้นตอนที่ 61: ทำความเข้าใจ Reactivity

### 61.1 Reactivity คืออะไร?

Reactivity คือระบบที่ทำให้ Vue.js รู้ว่าข้อมูลเปลี่ยนแปลง และอัปเดต UI อัตโนมัติ

```javascript
// ❌ JavaScript ธรรมดา - ไม่ reactive
let count = 0
document.getElementById('count').textContent = count

count = 5  // UI จะไม่อัปเดตอัตโนมัติ!

// ✅ Vue.js Reactive - อัปเดตอัตโนมัติ
const count = ref(0)
// เมื่อ count.value เปลี่ยน Vue จะ re-render อัตโนมัติ
count.value = 5  // UI อัปเดตทันที
```

### 61.2 วิธีการทำงาน (Proxy-based)

Vue 3 ใช้ JavaScript `Proxy` ในการ track changes:

```javascript
// แนวคิดของ Vue 3 Reactivity (simplified)
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key) {
      // track dependency
      track(target, key)
      return target[key]
    },
    set(target, key, value) {
      target[key] = value
      // trigger update
      trigger(target, key)
      return true
    }
  })
}
```

---

## ขั้นตอนที่ 62: ref() - Reactive Reference

`ref()` ใช้สำหรับ primitive values (string, number, boolean) และสามารถใช้กับ objects ได้

### 62.1 พื้นฐาน

```vue
<script setup>
import { ref } from 'vue'

// Primitive values
const count = ref(0)             // number
const name = ref('Vue.js')       // string
const isActive = ref(true)       // boolean
const price = ref(99.99)         // float

// อ่าน/เขียนค่าต้องใช้ .value
console.log(count.value)         // 0
count.value = 10
console.log(count.value)         // 10

// ใน Template ไม่ต้องใช้ .value (Vue auto-unwrap)
</script>

<template>
  <!-- ใน template ไม่ต้องใส่ .value -->
  <p>Count: {{ count }}</p>
  <p>Name: {{ name }}</p>
  <p>Active: {{ isActive }}</p>

  <button @click="count++">เพิ่ม (ใน template ใช้ได้โดยตรง)</button>
  <button @click="count.value++">เพิ่ม (แบบนี้ก็ได้)</button>
</template>
```

### 62.2 ref กับ Objects และ Arrays

```vue
<script setup>
import { ref } from 'vue'

// Object ref - deep reactive
const user = ref({
  name: 'สมชาย',
  age: 25,
  address: {
    city: 'กรุงเทพ',
    zip: '10200'
  }
})

// Array ref
const todos = ref([
  { id: 1, text: 'เรียน Vue.js', done: false },
  { id: 2, text: 'สร้าง Project', done: false },
  { id: 3, text: 'Deploy', done: false }
])

// การแก้ไข object properties
function updateUser() {
  user.value.name = 'สมหญิง'      // ✅ reactive
  user.value.age++                  // ✅ reactive
  user.value.address.city = 'เชียงใหม่'  // ✅ deep reactive
}

// การแก้ไข array
function addTodo(text) {
  todos.value.push({
    id: Date.now(),
    text,
    done: false
  })
}

function toggleTodo(id) {
  const todo = todos.value.find(t => t.id === id)
  if (todo) todo.done = !todo.done
}

function removeTodo(id) {
  todos.value = todos.value.filter(t => t.id !== id)
}
</script>

<template>
  <div>
    <h3>User Info</h3>
    <p>{{ user.name }} ({{ user.age }} ปี) - {{ user.address.city }}</p>
    <button @click="updateUser">อัปเดต</button>
    
    <h3>Todo List</h3>
    <ul>
      <li v-for="todo in todos" :key="todo.id">
        <input 
          type="checkbox" 
          :checked="todo.done"
          @change="toggleTodo(todo.id)"
        />
        <span :style="{ textDecoration: todo.done ? 'line-through' : 'none' }">
          {{ todo.text }}
        </span>
        <button @click="removeTodo(todo.id)">ลบ</button>
      </li>
    </ul>
  </div>
</template>
```

### 62.3 Type Annotations กับ ref

```typescript
import { ref } from 'vue'

// ระบุ type ผ่าน Generic
const count = ref<number>(0)
const name = ref<string>('')
const user = ref<User | null>(null)

// Interface
interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user' | 'moderator'
}

// Type inference (Vue จะ infer type จาก initial value)
const score = ref(100)  // TypeScript รู้ว่า score เป็น Ref<number>

// Nullable refs
const selectedUser = ref<User | null>(null)

function selectUser(user: User) {
  selectedUser.value = user
}

function clearSelection() {
  selectedUser.value = null
}
```

---

## ขั้นตอนที่ 63: reactive() - Reactive Object

`reactive()` ใช้สำหรับ objects เท่านั้น (ไม่ใช่ primitives)

### 63.1 พื้นฐาน

```vue
<script setup>
import { reactive } from 'vue'

// reactive object
const state = reactive({
  count: 0,
  name: 'Vue.js',
  user: {
    id: 1,
    name: 'สมชาย',
    roles: ['developer', 'reviewer']
  }
})

// ✅ ไม่ต้องใช้ .value!
console.log(state.count)     // 0
state.count = 10             // reactive
state.user.name = 'สมหญิง'  // deep reactive
state.user.roles.push('admin')  // array methods ทำงานได้

// ❌ อย่า reassign reactive object ทั้งหมด - จะหาย reactivity!
// state = { count: 1 }  // ห้าม!

// ✅ แก้ไข properties แทน
function resetState() {
  state.count = 0
  state.name = 'Vue.js'
  state.user = { id: 1, name: 'สมชาย', roles: ['developer'] }
}
</script>

<template>
  <div>
    <p>Count: {{ state.count }}</p>
    <p>Name: {{ state.name }}</p>
    <p>User: {{ state.user.name }}</p>
    <p>Roles: {{ state.user.roles.join(', ') }}</p>
    
    <button @click="state.count++">เพิ่ม Count</button>
    <button @click="resetState">Reset</button>
  </div>
</template>
```

### 63.2 reactive กับ Arrays

```vue
<script setup>
import { reactive } from 'vue'

const list = reactive({
  items: [],
  loading: false,
  error: null,
  total: 0
})

async function fetchItems() {
  list.loading = true
  list.error = null
  
  try {
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000))
    
    list.items = [
      { id: 1, name: 'สินค้า A', price: 100 },
      { id: 2, name: 'สินค้า B', price: 200 },
      { id: 3, name: 'สินค้า C', price: 300 }
    ]
    list.total = list.items.length
  } catch (err) {
    list.error = 'เกิดข้อผิดพลาด'
  } finally {
    list.loading = false
  }
}

function addItem(name, price) {
  list.items.push({ 
    id: Date.now(), 
    name, 
    price 
  })
  list.total++
}

function removeItem(id) {
  const index = list.items.findIndex(item => item.id === id)
  if (index !== -1) {
    list.items.splice(index, 1)
    list.total--
  }
}
</script>

<template>
  <div>
    <button @click="fetchItems" :disabled="list.loading">
      {{ list.loading ? 'กำลังโหลด...' : 'โหลดข้อมูล' }}
    </button>
    
    <p v-if="list.error" style="color: red">{{ list.error }}</p>
    
    <p>ทั้งหมด: {{ list.total }} รายการ</p>
    
    <ul>
      <li v-for="item in list.items" :key="item.id">
        {{ item.name }} - {{ item.price }} บาท
        <button @click="removeItem(item.id)">ลบ</button>
      </li>
    </ul>
  </div>
</template>
```

---

## ขั้นตอนที่ 64: ref vs reactive - เมื่อไรใช้อะไร?

### 64.1 ข้อเปรียบเทียบ

| Feature | `ref` | `reactive` |
|---------|-------|------------|
| Primitive values | ✅ | ❌ |
| Objects | ✅ | ✅ |
| ต้องใช้ `.value` | ✅ (ใน JS) | ❌ |
| Destructure แล้วยัง reactive | ❌ | ❌ (ต้องใช้ toRefs) |
| Reassign ทั้ง object | ✅ | ❌ |
| Template auto-unwrap | ✅ | ✅ |

### 64.2 Guidelines

```typescript
// ✅ ใช้ ref สำหรับ:
// - Primitive values
const count = ref(0)
const name = ref('')
const isLoading = ref(false)

// - ค่าที่ต้อง reassign ทั้งก้อน
const selectedItem = ref<Item | null>(null)
selectedItem.value = { id: 1, name: 'test' }  // OK
selectedItem.value = null                       // OK

// ✅ ใช้ reactive สำหรับ:
// - State ที่มีหลาย properties เกี่ยวข้องกัน
const formState = reactive({
  name: '',
  email: '',
  password: '',
  errors: {},
  isSubmitting: false
})

// - Store-like state
const appState = reactive({
  user: null,
  theme: 'light',
  language: 'th',
  notifications: []
})

// 💡 แนะนำ: ใช้ ref เป็นหลัก เพราะสม่ำเสมอกว่า
const state = ref({
  count: 0,
  name: '',
  items: []
})
```

---

## ขั้นตอนที่ 65: toRef() และ toRefs()

ใช้เมื่อต้องการ destructure reactive object แล้วยังคง reactivity

### 65.1 toRef()

```vue
<script setup>
import { reactive, toRef } from 'vue'

const user = reactive({
  name: 'สมชาย',
  age: 25,
  email: 'somchai@example.com'
})

// สร้าง ref ที่ link กับ property ของ reactive object
const nameRef = toRef(user, 'name')
const ageRef = toRef(user, 'age')

// เมื่อแก้ไข nameRef จะอัปเดต user.name ด้วย
nameRef.value = 'สมหญิง'
console.log(user.name)  // 'สมหญิง'

// เมื่อแก้ไข user.name จะอัปเดต nameRef ด้วย
user.name = 'สมศักดิ์'
console.log(nameRef.value)  // 'สมศักดิ์'
</script>

<template>
  <div>
    <input v-model="nameRef" placeholder="ชื่อ" />
    <p>User name: {{ user.name }}</p>
    <p>Name ref: {{ nameRef }}</p>
  </div>
</template>
```

### 65.2 toRefs()

```vue
<script setup>
import { reactive, toRefs } from 'vue'

const state = reactive({
  x: 0,
  y: 0,
  width: 100,
  height: 100
})

// Destructure ทั้งหมดในครั้งเดียวและยังคง reactivity
const { x, y, width, height } = toRefs(state)

// ตอนนี้ x, y, width, height เป็น Ref และ linked กับ state

function moveRight() {
  x.value += 10     // อัปเดต state.x ด้วย
}

// Composable pattern - common use case
function useMousePosition() {
  const position = reactive({ x: 0, y: 0 })
  
  function update(e) {
    position.x = e.clientX
    position.y = e.clientY
  }
  
  // Return as refs เพื่อให้ destructure ได้
  return { ...toRefs(position), update }
}

const { x: mouseX, y: mouseY, update } = useMousePosition()
</script>

<template>
  <div>
    <p>Position: ({{ x }}, {{ y }})</p>
    <p>Size: {{ width }} x {{ height }}</p>
    <button @click="moveRight">Move Right</button>
  </div>
</template>
```

---

## ขั้นตอนที่ 66: readonly()

ป้องกันการแก้ไขข้อมูล

```vue
<script setup>
import { ref, reactive, readonly } from 'vue'

const mutableState = reactive({
  count: 0,
  name: 'Vue.js'
})

// สร้าง readonly version
const readonlyState = readonly(mutableState)

// ✅ แก้ไข mutable ได้
mutableState.count++

// ❌ แก้ไข readonly ไม่ได้ (จะ warn ใน dev mode)
// readonlyState.count++  // Warning!

// Use case: pass data to child without allowing modification
function getReadonlyUser() {
  const user = ref({ name: 'สมชาย', role: 'admin' })
  // Return readonly เพื่อป้องกัน child แก้ไข
  return readonly(user)
}

// Deep readonly
const config = readonly({
  api: {
    baseUrl: 'https://api.example.com',
    timeout: 5000
  },
  features: {
    darkMode: true,
    notifications: true
  }
})

// config.api.baseUrl = 'xxx'  // Warning!
</script>
```

---

## ขั้นตอนที่ 67: shallowRef() และ shallowReactive()

สำหรับ performance optimization เมื่อมี large objects

### 67.1 shallowRef()

```vue
<script setup>
import { shallowRef, triggerRef } from 'vue'

// shallowRef: reactive เฉพาะ .value ไม่ใช่ properties ภายใน
const bigList = shallowRef([
  { id: 1, data: { /* lots of nested data */ } },
  // ... thousands of items
])

// ❌ การแก้ไข nested property จะไม่ trigger update
// bigList.value[0].data.name = 'new'  // ไม่ update UI

// ✅ ต้อง replace ทั้ง array หรือ triggerRef manually
function updateItem(id, newData) {
  const index = bigList.value.findIndex(item => item.id === id)
  if (index !== -1) {
    // Replace เพื่อ trigger reactivity
    const newList = [...bigList.value]
    newList[index] = { ...newList[index], data: newData }
    bigList.value = newList
  }
}

// หรือใช้ triggerRef เพื่อ force update
function updateInPlace(id, newData) {
  const item = bigList.value.find(i => i.id === id)
  if (item) {
    item.data = newData           // แก้ไข in-place
    triggerRef(bigList)           // force trigger
  }
}
</script>
```

### 67.2 shallowReactive()

```vue
<script setup>
import { shallowReactive } from 'vue'

// shallowReactive: reactive เฉพาะ top-level properties
const state = shallowReactive({
  count: 0,           // reactive
  user: {             // reactive (เฉพาะ reference)
    name: 'สมชาย'    // ❌ ไม่ reactive!
  }
})

state.count++              // ✅ triggers update
state.user = { name: 'ใหม่' }  // ✅ triggers update (replace reference)
state.user.name = 'ใหม่'   // ❌ ไม่ triggers update
</script>
```

---

## ขั้นตอนที่ 68: Type Guards

```vue
<script setup>
import { ref, reactive, isRef, isReactive, isReadonly, isProxy, unref } from 'vue'

const count = ref(0)
const state = reactive({ x: 1 })
const readonlyState = readonly(state)

// isRef: ตรวจสอบว่าเป็น ref หรือไม่
console.log(isRef(count))   // true
console.log(isRef(state))   // false
console.log(isRef(0))       // false

// isReactive: ตรวจสอบว่าเป็น reactive หรือไม่
console.log(isReactive(state))         // true
console.log(isReactive(readonlyState)) // true (reactive underneath)
console.log(isReactive(count))         // false

// isReadonly
console.log(isReadonly(readonlyState)) // true
console.log(isReadonly(state))         // false

// isProxy
console.log(isProxy(state))            // true
console.log(isProxy(count))            // false

// unref: ถ้าเป็น ref ให้ return .value, ถ้าไม่ใช่ return ค่าเดิม
function printValue(val) {
  console.log(unref(val))  // ไม่ต้องเขียน val?.value ?? val
}

printValue(count)     // 0 (unref Ref)
printValue(42)        // 42 (return as-is)
printValue('hello')   // 'hello'
</script>
```

---

## ขั้นตอนที่ 69: ตัวอย่าง Real-world - Shopping Cart

```vue
<script setup>
import { ref, reactive, computed, readonly } from 'vue'

// Types
const CartItem = {
  id: Number,
  name: String,
  price: Number,
  quantity: Number,
  image: String
}

// State
const cart = reactive({
  items: [],
  couponCode: '',
  discountPercent: 0
})

const isLoading = ref(false)

// Computed
const subtotal = computed(() => 
  cart.items.reduce((sum, item) => sum + item.price * item.quantity, 0)
)

const discount = computed(() => subtotal.value * (cart.discountPercent / 100))

const total = computed(() => subtotal.value - discount.value)

const itemCount = computed(() => 
  cart.items.reduce((sum, item) => sum + item.quantity, 0)
)

const isEmpty = computed(() => cart.items.length === 0)

// Actions
function addItem(product) {
  const existing = cart.items.find(item => item.id === product.id)
  
  if (existing) {
    existing.quantity++
  } else {
    cart.items.push({
      ...product,
      quantity: 1
    })
  }
}

function removeItem(id) {
  const index = cart.items.findIndex(item => item.id === id)
  if (index !== -1) cart.items.splice(index, 1)
}

function updateQuantity(id, quantity) {
  const item = cart.items.find(item => item.id === id)
  if (!item) return
  
  if (quantity <= 0) {
    removeItem(id)
  } else {
    item.quantity = quantity
  }
}

function clearCart() {
  cart.items = []
  cart.couponCode = ''
  cart.discountPercent = 0
}

async function applyCoupon() {
  if (!cart.couponCode) return
  
  isLoading.value = true
  
  // Simulate API
  await new Promise(resolve => setTimeout(resolve, 800))
  
  const coupons = { 'SAVE10': 10, 'SAVE20': 20, 'VIP50': 50 }
  const discount = coupons[cart.couponCode.toUpperCase()]
  
  if (discount) {
    cart.discountPercent = discount
    alert(`ใช้โค้ดสำเร็จ! ลด ${discount}%`)
  } else {
    alert('โค้ดไม่ถูกต้อง')
  }
  
  isLoading.value = false
}

function formatPrice(price) {
  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency: 'THB'
  }).format(price)
}

// Sample products
const products = [
  { id: 1, name: 'Vue.js หนังสือเล่มแรก', price: 299, image: '📚' },
  { id: 2, name: 'คอร์สออนไลน์ Vue.js', price: 999, image: '🎓' },
  { id: 3, name: 'Vue.js Sticker Pack', price: 99, image: '🎨' },
  { id: 4, name: 'เสื้อ Vue.js', price: 499, image: '👕' }
]
</script>

<template>
  <div class="shop">
    <!-- Products -->
    <section class="products">
      <h2>สินค้า</h2>
      <div class="product-grid">
        <div 
          v-for="product in products" 
          :key="product.id"
          class="product-card"
        >
          <div class="product-image">{{ product.image }}</div>
          <h3>{{ product.name }}</h3>
          <p class="price">{{ formatPrice(product.price) }}</p>
          <button @click="addItem(product)">เพิ่มในตะกร้า</button>
        </div>
      </div>
    </section>
    
    <!-- Cart -->
    <section class="cart">
      <h2>ตะกร้า ({{ itemCount }} ชิ้น)</h2>
      
      <div v-if="isEmpty" class="empty-cart">
        <p>🛒 ตะกร้าว่างเปล่า</p>
      </div>
      
      <template v-else>
        <div 
          v-for="item in cart.items" 
          :key="item.id"
          class="cart-item"
        >
          <span class="item-image">{{ item.image }}</span>
          <div class="item-details">
            <p class="item-name">{{ item.name }}</p>
            <p class="item-price">{{ formatPrice(item.price) }} × {{ item.quantity }}</p>
          </div>
          <div class="item-controls">
            <button @click="updateQuantity(item.id, item.quantity - 1)">-</button>
            <span>{{ item.quantity }}</span>
            <button @click="updateQuantity(item.id, item.quantity + 1)">+</button>
          </div>
          <p class="item-total">{{ formatPrice(item.price * item.quantity) }}</p>
          <button class="remove-btn" @click="removeItem(item.id)">✕</button>
        </div>
        
        <!-- Coupon -->
        <div class="coupon-section">
          <input 
            v-model="cart.couponCode"
            placeholder="รหัสส่วนลด"
            @keyup.enter="applyCoupon"
          />
          <button @click="applyCoupon" :disabled="isLoading">
            {{ isLoading ? '...' : 'ใช้โค้ด' }}
          </button>
        </div>
        
        <!-- Summary -->
        <div class="summary">
          <div class="summary-row">
            <span>ราคารวม</span>
            <span>{{ formatPrice(subtotal) }}</span>
          </div>
          <div v-if="cart.discountPercent > 0" class="summary-row discount">
            <span>ส่วนลด {{ cart.discountPercent }}%</span>
            <span>-{{ formatPrice(discount) }}</span>
          </div>
          <div class="summary-row total">
            <strong>ยอดสุทธิ</strong>
            <strong>{{ formatPrice(total) }}</strong>
          </div>
        </div>
        
        <div class="cart-actions">
          <button class="btn-clear" @click="clearCart">ล้างตะกร้า</button>
          <button class="btn-checkout">ชำระเงิน</button>
        </div>
      </template>
    </section>
  </div>
</template>

<style scoped>
.shop {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem;
  padding: 2rem;
  max-width: 1200px;
  margin: 0 auto;
}

.product-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
}

.product-card {
  padding: 1rem;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  text-align: center;
}

.product-image { font-size: 3rem; }
.price { color: #42b883; font-weight: bold; }

.cart-item {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.75rem;
  border-bottom: 1px solid #f0f0f0;
}

.item-details { flex: 1; }
.item-name { font-weight: bold; font-size: 0.9rem; }
.item-price { color: #666; font-size: 0.8rem; }

.item-controls {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.item-controls button {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: 1px solid #ddd;
  cursor: pointer;
}

.remove-btn {
  color: #e74c3c;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1.1rem;
}

.coupon-section {
  display: flex;
  gap: 0.5rem;
  margin: 1rem 0;
}

.coupon-section input {
  flex: 1;
  padding: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.summary {
  padding: 1rem;
  background: #f9f9f9;
  border-radius: 8px;
  margin: 1rem 0;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  padding: 0.25rem 0;
}

.discount { color: #e74c3c; }
.total { font-size: 1.1rem; border-top: 1px solid #ddd; padding-top: 0.5rem; margin-top: 0.5rem; }

.cart-actions {
  display: flex;
  gap: 1rem;
}

.btn-clear {
  flex: 1;
  padding: 0.75rem;
  background: white;
  border: 1px solid #ddd;
  border-radius: 8px;
  cursor: pointer;
}

.btn-checkout {
  flex: 2;
  padding: 0.75rem;
  background: #42b883;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
}
</style>
```

---

## สรุป Part 003

✅ เข้าใจ Reactivity System ของ Vue 3  
✅ ใช้ `ref()` สำหรับ primitives และ objects  
✅ ใช้ `reactive()` สำหรับ complex state  
✅ รู้ความแตกต่างระหว่าง ref และ reactive  
✅ ใช้ `toRef()` และ `toRefs()` สำหรับ destructuring  
✅ เข้าใจ `readonly()`, `shallowRef()`, `shallowReactive()`  
✅ สร้าง Shopping Cart App จริงได้  

---

## แบบฝึกหัด

1. **ระดับง่าย:** สร้าง Form ที่มี reactive state สำหรับ name, email, phone
2. **ระดับกลาง:** สร้าง Inventory Manager ที่ add/remove/update items ได้
3. **ระดับยาก:** สร้าง Mini Shopping Cart ที่สมบูรณ์พร้อม local storage

---

**← [Part 002: Template Syntax](./part-002-template-syntax.md)** | **→ [Part 004: Computed & Watchers](./part-004-computed-watchers.md)**
