# Part 19: Pinia - State Management

## บทนำ

Pinia คือ official state management library สำหรับ Vue.js (ทดแทน Vuex) มีความเรียบง่าย TypeScript-friendly และ DevTools integration ที่ดีเยี่ยม

---

## 1. Pinia vs Vuex

| Feature | Pinia | Vuex |
|---------|-------|------|
| API | Simpler, Composition API-friendly | Options API style |
| TypeScript | First-class support | ต้องการ extra setup |
| Mutations | ไม่มี (ใช้ actions โดยตรง) | ต้องมี mutations |
| Modules | ไม่ต้องการ namespace | ต้องการ namespaced modules |
| DevTools | ✅ Built-in | ✅ Built-in |
| Vue 3 | ✅ Primary choice | ✅ (Vuex 4) |

---

## 2. ติดตั้ง Pinia

```bash
npm install pinia
```

```javascript
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)
app.mount('#app')
```

---

## 3. defineStore() - Options Style

คล้ายกับ Options API ของ Vue component

```javascript
// stores/counter.js
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  // State - ข้อมูลที่ต้องการเก็บ
  state: () => ({
    count: 0,
    name: 'Counter',
    history: []
  }),
  
  // Getters - computed values
  getters: {
    doubleCount: (state) => state.count * 2,
    
    // Getter ที่ใช้ getter อื่น (ใช้ this)
    doubleCountPlusOne(): number {
      return this.doubleCount + 1
    },
    
    // Getter ที่รับ argument (return function)
    countByFactor: (state) => (factor) => state.count * factor
  },
  
  // Actions - methods (sync และ async ได้)
  actions: {
    increment() {
      this.count++
      this.history.push({ action: 'increment', value: this.count })
    },
    
    decrement() {
      this.count--
    },
    
    reset() {
      this.count = 0
      this.history = []
    },
    
    // Async action
    async fetchInitialCount() {
      const response = await fetch('/api/counter')
      const data = await response.json()
      this.count = data.count
    },
    
    // Action ที่ใช้ action อื่น
    async incrementAndSave() {
      this.increment()
      await this.saveToServer()
    },
    
    async saveToServer() {
      await fetch('/api/counter', {
        method: 'POST',
        body: JSON.stringify({ count: this.count })
      })
    }
  }
})
```

### การใช้งาน Options Store

```vue
<!-- components/CounterComponent.vue -->
<template>
  <div>
    <h3>{{ counter.name }}</h3>
    <p>Count: {{ counter.count }}</p>
    <p>Double: {{ counter.doubleCount }}</p>
    <p>Triple: {{ counter.countByFactor(3) }}</p>
    
    <button @click="counter.increment">+</button>
    <button @click="counter.decrement">-</button>
    <button @click="counter.reset">Reset</button>
    
    <div>
      <h4>History</h4>
      <ul>
        <li v-for="(h, i) in counter.history" :key="i">
          {{ h.action }}: {{ h.value }}
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { useCounterStore } from '../stores/counter'

// ใช้ store โดยตรง - reactive โดยอัตโนมัติ
const counter = useCounterStore()

// ❌ อย่า destructure โดยตรง - จะสูญเสีย reactivity
// const { count, increment } = useCounterStore() // WRONG!

// ✅ ใช้ storeToRefs สำหรับ state และ getters
import { storeToRefs } from 'pinia'
const { count, doubleCount } = storeToRefs(counter)
// actions ไม่ต้องใช้ storeToRefs
const { increment, decrement } = counter
</script>
```

---

## 4. defineStore() - Setup Style

คล้ายกับ Composition API ของ Vue component (แนะนำ)

```javascript
// stores/cart.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCartStore = defineStore('cart', () => {
  // State (ref)
  const items = ref([])
  const couponCode = ref('')
  const isLoading = ref(false)
  
  // Getters (computed)
  const itemCount = computed(() => 
    items.value.reduce((sum, item) => sum + item.quantity, 0)
  )
  
  const subtotal = computed(() =>
    items.value.reduce((sum, item) => sum + (item.price * item.quantity), 0)
  )
  
  const discount = computed(() => {
    if (couponCode.value === 'SAVE20') return subtotal.value * 0.2
    if (couponCode.value === 'SAVE50') return subtotal.value * 0.5
    return 0
  })
  
  const total = computed(() => subtotal.value - discount.value)
  
  const isEmpty = computed(() => items.value.length === 0)
  
  // Actions (functions)
  function addItem(product) {
    const existingItem = items.value.find(item => item.id === product.id)
    
    if (existingItem) {
      existingItem.quantity++
    } else {
      items.value.push({
        id: product.id,
        name: product.name,
        price: product.price,
        image: product.image,
        quantity: 1
      })
    }
  }
  
  function removeItem(productId) {
    items.value = items.value.filter(item => item.id !== productId)
  }
  
  function updateQuantity(productId, quantity) {
    const item = items.value.find(item => item.id === productId)
    if (item) {
      if (quantity <= 0) {
        removeItem(productId)
      } else {
        item.quantity = quantity
      }
    }
  }
  
  function applyCoupon(code) {
    const validCodes = ['SAVE20', 'SAVE50']
    if (validCodes.includes(code.toUpperCase())) {
      couponCode.value = code.toUpperCase()
      return { success: true, message: 'ใช้คูปองสำเร็จ!' }
    }
    return { success: false, message: 'รหัสคูปองไม่ถูกต้อง' }
  }
  
  function clearCart() {
    items.value = []
    couponCode.value = ''
  }
  
  async function checkout() {
    isLoading.value = true
    try {
      const response = await fetch('/api/orders', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          items: items.value,
          couponCode: couponCode.value,
          total: total.value
        })
      })
      
      if (!response.ok) throw new Error('Checkout failed')
      
      const order = await response.json()
      clearCart()
      
      return { success: true, orderId: order.id }
    } catch (error) {
      return { success: false, error: error.message }
    } finally {
      isLoading.value = false
    }
  }
  
  // Expose ทุกอย่างที่ต้องการ
  return {
    // State
    items, couponCode, isLoading,
    // Getters
    itemCount, subtotal, discount, total, isEmpty,
    // Actions
    addItem, removeItem, updateQuantity,
    applyCoupon, clearCart, checkout
  }
})
```

---

## 5. State, Getters, Actions

### การ modify state โดยตรง

```javascript
// ✅ Modify state โดยตรงใน action
const store = useMyStore()
store.count = 10 // ทำได้โดยตรง

// ✅ ใช้ $patch สำหรับ update หลายค่าพร้อมกัน
store.$patch({
  count: 10,
  name: 'new name'
})

// ✅ $patch ด้วย function (สำหรับ logic ที่ซับซ้อน)
store.$patch((state) => {
  state.count++
  state.items.push({ id: Date.now() })
})

// ✅ $reset() - reset state กลับไปเป็นค่าเริ่มต้น (Options style เท่านั้น)
store.$reset()
```

### Subscribe to State Changes

```javascript
// ฟัง state changes
const unsubscribe = store.$subscribe((mutation, state) => {
  console.log('State changed:', mutation.type, state)
  
  // บันทึกลง localStorage อัตโนมัติ
  localStorage.setItem('cart', JSON.stringify(state.items))
})

// ยกเลิก subscription
unsubscribe()

// ฟัง actions
const unsubscribeAction = store.$onAction(({ name, args, after, onError }) => {
  console.log(`Action "${name}" called with args:`, args)
  
  after((result) => {
    console.log(`Action "${name}" completed with result:`, result)
  })
  
  onError((error) => {
    console.error(`Action "${name}" failed:`, error)
  })
})
```

---

## 6. Store Composition

Stores สามารถใช้ stores อื่นได้

```javascript
// stores/user.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  const profile = ref(null)
  const isAuthenticated = computed(() => !!profile.value)
  
  return { profile, isAuthenticated }
})
```

```javascript
// stores/orders.js - ใช้ userStore
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { useUserStore } from './user'
import { useCartStore } from './cart'

export const useOrderStore = defineStore('orders', () => {
  const orders = ref([])
  const isLoading = ref(false)
  
  // ใช้ stores อื่น
  const userStore = useUserStore()
  const cartStore = useCartStore()
  
  async function createOrder() {
    if (!userStore.isAuthenticated) {
      throw new Error('กรุณาเข้าสู่ระบบก่อน')
    }
    
    if (cartStore.isEmpty) {
      throw new Error('ตะกร้าว่างเปล่า')
    }
    
    isLoading.value = true
    
    try {
      const response = await fetch('/api/orders', {
        method: 'POST',
        body: JSON.stringify({
          userId: userStore.profile.id,
          items: cartStore.items,
          total: cartStore.total
        })
      })
      
      const order = await response.json()
      orders.value.unshift(order)
      cartStore.clearCart()
      
      return order
    } finally {
      isLoading.value = false
    }
  }
  
  async function fetchOrders() {
    if (!userStore.isAuthenticated) return
    
    const response = await fetch(`/api/users/${userStore.profile.id}/orders`)
    orders.value = await response.json()
  }
  
  return { orders, isLoading, createOrder, fetchOrders }
})
```

---

## 7. Persist Plugin

บันทึก store state ลง localStorage อัตโนมัติ

```bash
npm install pinia-plugin-persistedstate
```

```javascript
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'
import App from './App.vue'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)

const app = createApp(App)
app.use(pinia)
app.mount('#app')
```

### Store พร้อม Persist

```javascript
// stores/auth.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useAuthStore = defineStore('auth', () => {
  const token = ref(null)
  const user = ref(null)
  
  const isAuthenticated = computed(() => !!token.value)
  
  function setAuth(authData) {
    token.value = authData.token
    user.value = authData.user
  }
  
  function clearAuth() {
    token.value = null
    user.value = null
  }
  
  return { token, user, isAuthenticated, setAuth, clearAuth }
}, {
  // Persist configuration
  persist: {
    // บันทึกเฉพาะ token (ไม่บันทึก sensitive user data)
    paths: ['token'],
    
    // ใช้ sessionStorage แทน localStorage
    // storage: sessionStorage,
    
    // Custom key
    key: 'my-auth'
  }
})
```

```javascript
// stores/settings.js - Persist ทั้ง store
export const useSettingsStore = defineStore('settings', () => {
  const theme = ref('light')
  const language = ref('th')
  const fontSize = ref(16)
  const notifications = ref(true)
  
  return { theme, language, fontSize, notifications }
}, {
  persist: true // บันทึกทั้ง store
})
```

---

## 8. ตัวอย่าง: Auth Store สมบูรณ์

```javascript
// stores/auth.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useAuthStore = defineStore('auth', () => {
  // State
  const user = ref(null)
  const token = ref(null)
  const isLoading = ref(false)
  const initialized = ref(false)
  
  // Getters
  const isAuthenticated = computed(() => !!user.value && !!token.value)
  const isAdmin = computed(() => user.value?.role === 'admin')
  const isTeacher = computed(() => ['admin', 'teacher'].includes(user.value?.role))
  const userDisplayName = computed(() => user.value?.name || user.value?.email || 'ผู้ใช้')
  
  // Actions
  async function initialize() {
    if (initialized.value) return
    
    const savedToken = localStorage.getItem('token')
    if (savedToken) {
      token.value = savedToken
      try {
        await fetchCurrentUser()
      } catch {
        clearAuth()
      }
    }
    
    initialized.value = true
  }
  
  async function fetchCurrentUser() {
    const response = await fetch('/api/auth/me', {
      headers: { Authorization: `Bearer ${token.value}` }
    })
    
    if (!response.ok) throw new Error('Invalid token')
    
    user.value = await response.json()
  }
  
  async function login(credentials) {
    isLoading.value = true
    
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(credentials)
      })
      
      if (!response.ok) {
        const error = await response.json()
        throw new Error(error.message || 'เข้าสู่ระบบล้มเหลว')
      }
      
      const data = await response.json()
      
      token.value = data.token
      user.value = data.user
      localStorage.setItem('token', data.token)
      
      return { success: true }
      
    } catch (error) {
      throw error
    } finally {
      isLoading.value = false
    }
  }
  
  async function register(userData) {
    isLoading.value = true
    
    try {
      const response = await fetch('/api/auth/register', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(userData)
      })
      
      if (!response.ok) {
        const error = await response.json()
        throw new Error(error.message)
      }
      
      const data = await response.json()
      
      token.value = data.token
      user.value = data.user
      localStorage.setItem('token', data.token)
      
      return { success: true }
      
    } finally {
      isLoading.value = false
    }
  }
  
  async function updateProfile(updates) {
    const response = await fetch('/api/auth/profile', {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token.value}`
      },
      body: JSON.stringify(updates)
    })
    
    if (!response.ok) throw new Error('อัปเดตโปรไฟล์ล้มเหลว')
    
    user.value = await response.json()
    return user.value
  }
  
  async function logout() {
    try {
      await fetch('/api/auth/logout', {
        method: 'POST',
        headers: { Authorization: `Bearer ${token.value}` }
      })
    } finally {
      clearAuth()
    }
  }
  
  function clearAuth() {
    user.value = null
    token.value = null
    localStorage.removeItem('token')
  }
  
  return {
    // State
    user, token, isLoading, initialized,
    // Getters
    isAuthenticated, isAdmin, isTeacher, userDisplayName,
    // Actions
    initialize, login, register, logout,
    updateProfile, fetchCurrentUser, clearAuth
  }
})
```

---

## 9. ตัวอย่าง: Cart Store สมบูรณ์

```vue
<!-- components/CartSidebar.vue -->
<template>
  <div class="cart-sidebar" :class="{ open: isOpen }">
    <div class="cart-header">
      <h3>🛒 ตะกร้าสินค้า ({{ cart.itemCount }})</h3>
      <button @click="$emit('close')">✕</button>
    </div>
    
    <!-- Empty state -->
    <div v-if="cart.isEmpty" class="cart-empty">
      <p>🛒 ตะกร้าว่างเปล่า</p>
      <RouterLink to="/products" @click="$emit('close')">
        เลือกซื้อสินค้า
      </RouterLink>
    </div>
    
    <!-- Cart items -->
    <div v-else>
      <div class="cart-items">
        <div
          v-for="item in cart.items"
          :key="item.id"
          class="cart-item"
        >
          <img :src="item.image" :alt="item.name" />
          <div class="item-info">
            <p class="item-name">{{ item.name }}</p>
            <p class="item-price">฿{{ item.price.toLocaleString() }}</p>
          </div>
          <div class="item-qty">
            <button @click="cart.updateQuantity(item.id, item.quantity - 1)">-</button>
            <span>{{ item.quantity }}</span>
            <button @click="cart.updateQuantity(item.id, item.quantity + 1)">+</button>
          </div>
          <button @click="cart.removeItem(item.id)" class="remove-btn">🗑️</button>
        </div>
      </div>
      
      <!-- Coupon -->
      <div class="coupon-section">
        <input v-model="couponInput" placeholder="รหัสคูปอง" />
        <button @click="applyCoupon">ใช้</button>
        <p v-if="couponMessage" :class="couponSuccess ? 'success' : 'error'">
          {{ couponMessage }}
        </p>
      </div>
      
      <!-- Summary -->
      <div class="cart-summary">
        <div class="summary-row">
          <span>ราคารวม</span>
          <span>฿{{ cart.subtotal.toLocaleString() }}</span>
        </div>
        <div v-if="cart.discount > 0" class="summary-row discount">
          <span>ส่วนลด</span>
          <span>-฿{{ cart.discount.toLocaleString() }}</span>
        </div>
        <div class="summary-row total">
          <strong>รวมทั้งหมด</strong>
          <strong>฿{{ cart.total.toLocaleString() }}</strong>
        </div>
      </div>
      
      <button
        @click="handleCheckout"
        :disabled="cart.isLoading"
        class="checkout-btn"
      >
        {{ cart.isLoading ? 'กำลังประมวลผล...' : `ชำระเงิน ฿${cart.total.toLocaleString()}` }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useCartStore } from '../stores/cart'
import { useRouter } from 'vue-router'

const props = defineProps({ isOpen: Boolean })
const emit = defineEmits(['close'])

const cart = useCartStore()
const router = useRouter()

const couponInput = ref('')
const couponMessage = ref('')
const couponSuccess = ref(false)

function applyCoupon() {
  const result = cart.applyCoupon(couponInput.value)
  couponMessage.value = result.message
  couponSuccess.value = result.success
  
  if (result.success) couponInput.value = ''
}

async function handleCheckout() {
  const result = await cart.checkout()
  if (result.success) {
    emit('close')
    router.push({ name: 'order-success', params: { id: result.orderId } })
  }
}
</script>
```

---

## สรุป

```javascript
// Setup Style (แนะนำ)
export const useMyStore = defineStore('myStore', () => {
  const data = ref(null)              // state
  const derived = computed(() => ...) // getter
  function doSomething() { ... }      // action
  return { data, derived, doSomething }
})

// การใช้งานใน component
const store = useMyStore()
const { data, derived } = storeToRefs(store) // reactive destructure
const { doSomething } = store                // actions ไม่ต้องใช้ storeToRefs
```

| ทำ | วิธี |
|----|------|
| อ่าน state | `store.myData` หรือ `storeToRefs(store).myData` |
| แก้ไข state | `store.myData = newValue` หรือ `store.$patch({...})` |
| เรียก action | `store.myAction()` |
| Reset state | `store.$reset()` (Options style) |
| Subscribe | `store.$subscribe(callback)` |
