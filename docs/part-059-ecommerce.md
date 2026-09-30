# Part 59: E-commerce Platform ด้วย Nuxt.js

## E-commerce Architecture

```
┌──────────────────────────────────────────────────────┐
│                    Nuxt.js Frontend                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │ Product  │ │  Cart    │ │Checkout  │ │ Orders  │ │
│  │ Catalog  │ │  Pinia   │ │  Flow    │ │ History │ │
│  └──────────┘ └──────────┘ └──────────┘ └─────────┘ │
└──────────────────────────────────────────────────────┘
                         │
                    Nuxt API
                         │
┌──────────────────────────────────────────────────────┐
│                    Backend Services                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │ Products │ │  Orders  │ │  Stripe  │ │Inventory│ │
│  │    DB    │ │    DB    │ │ Payment  │ │ Service │ │
│  └──────────┘ └──────────┘ └──────────┘ └─────────┘ │
└──────────────────────────────────────────────────────┘
```

## Shared Types

```typescript
// types/ecommerce.ts
export interface Product {
  id: string
  name: string
  slug: string
  description: string
  shortDescription: string
  price: number
  comparePrice?: number
  images: ProductImage[]
  category: Category
  tags: string[]
  variants: ProductVariant[]
  inventory: Inventory
  rating: number
  reviewCount: number
  isActive: boolean
  createdAt: string
  updatedAt: string
}

export interface ProductImage {
  id: string
  url: string
  alt: string
  position: number
}

export interface ProductVariant {
  id: string
  name: string
  sku: string
  price: number
  stock: number
  options: Record<string, string>
}

export interface Category {
  id: string
  name: string
  slug: string
  image?: string
  parent?: Category
  children: Category[]
}

export interface Inventory {
  quantity: number
  reserved: number
  available: number
  lowStockThreshold: number
}

export interface CartItem {
  id: string
  product: Product
  variant?: ProductVariant
  quantity: number
  price: number
  subtotal: number
}

export interface Cart {
  id: string
  items: CartItem[]
  subtotal: number
  discount: number
  shipping: number
  tax: number
  total: number
  coupon?: Coupon
}

export interface Coupon {
  code: string
  type: 'percentage' | 'fixed'
  value: number
  minOrderAmount?: number
  maxUses?: number
  usedCount: number
  expiresAt?: string
}

export interface Address {
  id?: string
  firstName: string
  lastName: string
  addressLine1: string
  addressLine2?: string
  city: string
  province: string
  postalCode: string
  country: string
  phone: string
  isDefault?: boolean
}

export interface Order {
  id: string
  orderNumber: string
  status: OrderStatus
  items: OrderItem[]
  shippingAddress: Address
  billingAddress: Address
  paymentMethod: string
  paymentStatus: PaymentStatus
  subtotal: number
  discount: number
  shipping: number
  tax: number
  total: number
  notes?: string
  trackingNumber?: string
  createdAt: string
  updatedAt: string
}

export type OrderStatus = 
  | 'pending' 
  | 'confirmed' 
  | 'processing' 
  | 'shipped' 
  | 'delivered' 
  | 'cancelled' 
  | 'refunded'

export type PaymentStatus = 
  | 'pending' 
  | 'paid' 
  | 'failed' 
  | 'refunded'

export interface OrderItem {
  id: string
  product: Product
  variant?: ProductVariant
  quantity: number
  price: number
  subtotal: number
}
```

## Product Catalog

```vue
<!-- pages/products/index.vue -->
<template>
  <div class="catalog-page">
    <!-- Filters -->
    <aside class="filters">
      <h2>กรอง</h2>
      
      <!-- Categories -->
      <div class="filter-section">
        <h3>หมวดหมู่</h3>
        <ul>
          <li v-for="cat in categories" :key="cat.id">
            <label>
              <input 
                type="checkbox" 
                :value="cat.id"
                v-model="filters.categories"
              />
              {{ cat.name }}
            </label>
          </li>
        </ul>
      </div>
      
      <!-- Price Range -->
      <div class="filter-section">
        <h3>ราคา</h3>
        <div class="price-range">
          <input 
            v-model.number="filters.minPrice" 
            type="number" 
            placeholder="ราคาต่ำสุด"
          />
          <span>-</span>
          <input 
            v-model.number="filters.maxPrice" 
            type="number" 
            placeholder="ราคาสูงสุด"
          />
        </div>
      </div>
      
      <!-- Rating -->
      <div class="filter-section">
        <h3>คะแนน</h3>
        <div v-for="r in [4, 3, 2, 1]" :key="r">
          <label>
            <input type="radio" :value="r" v-model="filters.minRating" />
            {{ r }}+ ดาว
          </label>
        </div>
      </div>
      
      <button @click="clearFilters" class="clear-filters">ล้างตัวกรอง</button>
    </aside>

    <!-- Products Grid -->
    <main class="products-main">
      <!-- Sort & Results -->
      <div class="products-header">
        <p>พบ {{ totalProducts }} สินค้า</p>
        <select v-model="sortBy">
          <option value="newest">ใหม่ล่าสุด</option>
          <option value="price_asc">ราคา: ต่ำ - สูง</option>
          <option value="price_desc">ราคา: สูง - ต่ำ</option>
          <option value="rating">คะแนนสูงสุด</option>
          <option value="popular">ยอดนิยม</option>
        </select>
      </div>
      
      <!-- Grid -->
      <div class="products-grid" v-if="!loading">
        <ProductCard
          v-for="product in products"
          :key="product.id"
          :product="product"
          @add-to-cart="addToCart"
          @quick-view="openQuickView"
        />
      </div>
      
      <div v-else class="loading-grid">
        <ProductCardSkeleton v-for="i in 12" :key="i" />
      </div>
      
      <!-- Pagination -->
      <Pagination
        v-if="totalPages > 1"
        :current="currentPage"
        :total="totalPages"
        @page-change="goToPage"
      />
    </main>
    
    <!-- Quick View Modal -->
    <ProductQuickView
      v-if="quickViewProduct"
      :product="quickViewProduct"
      @close="quickViewProduct = null"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import type { Product, Category } from '@/types/ecommerce'
import { useCartStore } from '@/stores/cart'

const route = useRoute()
const router = useRouter()
const cartStore = useCartStore()

const products = ref<Product[]>([])
const categories = ref<Category[]>([])
const loading = ref(false)
const totalProducts = ref(0)
const totalPages = ref(0)
const quickViewProduct = ref<Product | null>(null)

const currentPage = computed(() => Number(route.query.page) || 1)
const sortBy = ref(String(route.query.sort || 'newest'))

const filters = ref({
  categories: [] as string[],
  minPrice: null as number | null,
  maxPrice: null as number | null,
  minRating: null as number | null
})

const fetchProducts = async () => {
  loading.value = true
  try {
    const params = new URLSearchParams()
    params.set('page', String(currentPage.value))
    params.set('sort', sortBy.value)
    if (filters.value.categories.length) {
      params.set('categories', filters.value.categories.join(','))
    }
    if (filters.value.minPrice) params.set('minPrice', String(filters.value.minPrice))
    if (filters.value.maxPrice) params.set('maxPrice', String(filters.value.maxPrice))
    if (filters.value.minRating) params.set('minRating', String(filters.value.minRating))

    const data = await $fetch(`/api/products?${params}`)
    products.value = data.products
    totalProducts.value = data.total
    totalPages.value = data.totalPages
  } finally {
    loading.value = false
  }
}

const fetchCategories = async () => {
  categories.value = await $fetch('/api/categories')
}

const clearFilters = () => {
  filters.value = {
    categories: [],
    minPrice: null,
    maxPrice: null,
    minRating: null
  }
}

const goToPage = (page: number) => {
  router.push({ query: { ...route.query, page } })
}

const addToCart = (product: Product) => {
  cartStore.addItem(product)
}

const openQuickView = (product: Product) => {
  quickViewProduct.value = product
}

watch([currentPage, sortBy, filters], fetchProducts, { immediate: true, deep: true })
fetchCategories()
</script>
```

```vue
<!-- components/ProductCard.vue -->
<template>
  <div class="product-card">
    <!-- Product Image -->
    <div class="product-image">
      <NuxtImg
        :src="product.images[0]?.url"
        :alt="product.images[0]?.alt"
        width="300"
        height="300"
        loading="lazy"
        class="product-img"
      />
      
      <!-- Badges -->
      <div class="product-badges">
        <span v-if="isNew" class="badge badge--new">ใหม่</span>
        <span v-if="discount > 0" class="badge badge--sale">-{{ discount }}%</span>
        <span v-if="product.inventory.available === 0" class="badge badge--out">หมด</span>
      </div>
      
      <!-- Quick actions -->
      <div class="quick-actions">
        <button
          :aria-label="`เพิ่ม ${product.name} ลงตะกร้า`"
          :disabled="product.inventory.available === 0"
          @click.prevent="$emit('add-to-cart', product)"
        >
          🛒
        </button>
        <button
          :aria-label="`ดูด่วน ${product.name}`"
          @click.prevent="$emit('quick-view', product)"
        >
          👁️
        </button>
        <WishlistButton :product-id="product.id" />
      </div>
    </div>
    
    <!-- Product Info -->
    <NuxtLink :to="`/products/${product.slug}`" class="product-info">
      <p class="product-category">{{ product.category.name }}</p>
      <h3 class="product-name">{{ product.name }}</h3>
      
      <!-- Rating -->
      <div class="product-rating">
        <StarRating :rating="product.rating" />
        <span class="review-count">({{ product.reviewCount }})</span>
      </div>
      
      <!-- Price -->
      <div class="product-price">
        <span class="current-price">฿{{ formatPrice(product.price) }}</span>
        <span v-if="product.comparePrice" class="compare-price">
          ฿{{ formatPrice(product.comparePrice) }}
        </span>
      </div>
    </NuxtLink>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import type { Product } from '@/types/ecommerce'

const props = defineProps<{
  product: Product
}>()

defineEmits(['add-to-cart', 'quick-view'])

const isNew = computed(() => {
  const daysSinceCreated = (Date.now() - new Date(props.product.createdAt).getTime()) / 86400000
  return daysSinceCreated <= 14
})

const discount = computed(() => {
  if (!props.product.comparePrice) return 0
  return Math.round((1 - props.product.price / props.product.comparePrice) * 100)
})

const formatPrice = (price: number) => price.toLocaleString('th-TH')
</script>
```

## Shopping Cart (Pinia)

```typescript
// stores/cart.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { CartItem, Product, ProductVariant, Coupon } from '@/types/ecommerce'

export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])
  const coupon = ref<Coupon | null>(null)
  const isOpen = ref(false)

  // Computed
  const itemCount = computed(() => items.value.reduce((sum, item) => sum + item.quantity, 0))
  
  const subtotal = computed(() =>
    items.value.reduce((sum, item) => sum + item.subtotal, 0)
  )
  
  const discount = computed(() => {
    if (!coupon.value) return 0
    if (coupon.value.type === 'percentage') {
      return subtotal.value * (coupon.value.value / 100)
    }
    return coupon.value.value
  })
  
  const shipping = computed(() => {
    if (subtotal.value - discount.value >= 500) return 0 // Free shipping
    return 50
  })
  
  const tax = computed(() => (subtotal.value - discount.value) * 0.07)
  
  const total = computed(() =>
    subtotal.value - discount.value + shipping.value + tax.value
  )

  // Actions
  const addItem = (product: Product, variant?: ProductVariant, quantity = 1) => {
    const price = variant?.price ?? product.price
    const existingItem = items.value.find(item =>
      item.product.id === product.id && item.variant?.id === variant?.id
    )
    
    if (existingItem) {
      const maxQty = variant?.stock ?? product.inventory.available
      existingItem.quantity = Math.min(existingItem.quantity + quantity, maxQty)
      existingItem.subtotal = existingItem.quantity * existingItem.price
    } else {
      items.value.push({
        id: crypto.randomUUID(),
        product,
        variant,
        quantity,
        price,
        subtotal: price * quantity
      })
    }
    
    isOpen.value = true
    persistCart()
  }

  const removeItem = (itemId: string) => {
    const index = items.value.findIndex(item => item.id === itemId)
    if (index > -1) {
      items.value.splice(index, 1)
      persistCart()
    }
  }

  const updateQuantity = (itemId: string, quantity: number) => {
    const item = items.value.find(i => i.id === itemId)
    if (!item) return
    
    if (quantity <= 0) {
      removeItem(itemId)
    } else {
      const maxQty = item.variant?.stock ?? item.product.inventory.available
      item.quantity = Math.min(quantity, maxQty)
      item.subtotal = item.quantity * item.price
      persistCart()
    }
  }

  const applyCoupon = async (code: string) => {
    try {
      const data = await $fetch<Coupon>('/api/coupons/validate', {
        method: 'POST',
        body: { code, orderAmount: subtotal.value }
      })
      coupon.value = data
      return { success: true }
    } catch (error: any) {
      return { success: false, message: error.data?.message || 'รหัสคูปองไม่ถูกต้อง' }
    }
  }

  const removeCoupon = () => {
    coupon.value = null
  }

  const clearCart = () => {
    items.value = []
    coupon.value = null
    localStorage.removeItem('cart')
  }

  const persistCart = () => {
    try {
      localStorage.setItem('cart', JSON.stringify(items.value))
    } catch (e) {
      console.warn('Failed to persist cart')
    }
  }

  const loadCart = () => {
    try {
      const saved = localStorage.getItem('cart')
      if (saved) {
        items.value = JSON.parse(saved)
      }
    } catch (e) {
      console.warn('Failed to load cart')
    }
  }

  // Initialize
  if (import.meta.client) {
    loadCart()
  }

  return {
    items,
    coupon,
    isOpen,
    itemCount,
    subtotal,
    discount,
    shipping,
    tax,
    total,
    addItem,
    removeItem,
    updateQuantity,
    applyCoupon,
    removeCoupon,
    clearCart
  }
}, {
  persist: true
})
```

## Checkout Flow

```vue
<!-- pages/checkout.vue -->
<template>
  <div class="checkout-page">
    <!-- Progress Steps -->
    <div class="checkout-steps">
      <div
        v-for="(step, index) in steps"
        :key="step.id"
        :class="['step', { 
          'step--active': currentStep === index,
          'step--completed': currentStep > index
        }]"
      >
        <div class="step-icon">
          <span v-if="currentStep > index">✓</span>
          <span v-else>{{ index + 1 }}</span>
        </div>
        <span class="step-label">{{ step.label }}</span>
      </div>
    </div>

    <!-- Step Content -->
    <div class="checkout-content">
      <!-- Step 1: Shipping -->
      <CheckoutShipping
        v-if="currentStep === 0"
        v-model="shippingAddress"
        @next="currentStep = 1"
      />
      
      <!-- Step 2: Shipping Method -->
      <CheckoutShippingMethod
        v-else-if="currentStep === 1"
        v-model="shippingMethod"
        :address="shippingAddress"
        @prev="currentStep = 0"
        @next="currentStep = 2"
      />
      
      <!-- Step 3: Payment -->
      <CheckoutPayment
        v-else-if="currentStep === 2"
        :order-total="cartStore.total"
        @prev="currentStep = 1"
        @payment-success="handlePaymentSuccess"
        @payment-error="handlePaymentError"
      />
      
      <!-- Step 4: Order Confirmation -->
      <CheckoutConfirmation
        v-else-if="currentStep === 3"
        :order="completedOrder"
      />
    </div>

    <!-- Order Summary (sidebar) -->
    <aside class="order-summary">
      <h2>สรุปคำสั่งซื้อ</h2>
      
      <div class="order-items">
        <div v-for="item in cartStore.items" :key="item.id" class="order-item">
          <img :src="item.product.images[0]?.url" :alt="item.product.name" />
          <div class="item-details">
            <p>{{ item.product.name }}</p>
            <p v-if="item.variant">{{ item.variant.name }}</p>
            <p>x{{ item.quantity }}</p>
          </div>
          <span>฿{{ (item.subtotal).toLocaleString('th-TH') }}</span>
        </div>
      </div>
      
      <!-- Coupon -->
      <div v-if="currentStep < 2" class="coupon-section">
        <div class="coupon-input">
          <input v-model="couponCode" placeholder="รหัสคูปอง" />
          <button @click="applyCoupon" :disabled="applyingCoupon">
            {{ applyingCoupon ? '...' : 'ใช้' }}
          </button>
        </div>
        <p v-if="cartStore.coupon" class="coupon-applied">
          ✓ {{ cartStore.coupon.code }} (-{{ cartStore.discount.toLocaleString('th-TH') }}฿)
          <button @click="cartStore.removeCoupon">✕</button>
        </p>
        <p v-if="couponError" class="coupon-error">{{ couponError }}</p>
      </div>
      
      <!-- Totals -->
      <div class="order-totals">
        <div class="total-row">
          <span>ยอดรวม</span>
          <span>฿{{ cartStore.subtotal.toLocaleString('th-TH') }}</span>
        </div>
        <div v-if="cartStore.discount > 0" class="total-row discount">
          <span>ส่วนลด</span>
          <span>-฿{{ cartStore.discount.toLocaleString('th-TH') }}</span>
        </div>
        <div class="total-row">
          <span>ค่าจัดส่ง</span>
          <span>{{ cartStore.shipping === 0 ? 'ฟรี' : `฿${cartStore.shipping}` }}</span>
        </div>
        <div class="total-row">
          <span>ภาษี (7%)</span>
          <span>฿{{ cartStore.tax.toLocaleString('th-TH', { maximumFractionDigits: 2 }) }}</span>
        </div>
        <div class="total-row total-grand">
          <span>รวมทั้งหมด</span>
          <span>฿{{ cartStore.total.toLocaleString('th-TH', { maximumFractionDigits: 2 }) }}</span>
        </div>
      </div>
    </aside>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useCartStore } from '@/stores/cart'
import type { Address, Order } from '@/types/ecommerce'

definePageMeta({
  middleware: 'auth'
})

const cartStore = useCartStore()
const currentStep = ref(0)
const shippingAddress = ref<Partial<Address>>({})
const shippingMethod = ref('')
const completedOrder = ref<Order | null>(null)
const couponCode = ref('')
const couponError = ref('')
const applyingCoupon = ref(false)

const steps = [
  { id: 'shipping', label: 'ที่อยู่จัดส่ง' },
  { id: 'method', label: 'วิธีจัดส่ง' },
  { id: 'payment', label: 'ชำระเงิน' },
  { id: 'confirmation', label: 'ยืนยัน' }
]

const applyCoupon = async () => {
  if (!couponCode.value) return
  applyingCoupon.value = true
  couponError.value = ''
  
  const result = await cartStore.applyCoupon(couponCode.value)
  if (!result.success) {
    couponError.value = result.message || 'เกิดข้อผิดพลาด'
  } else {
    couponCode.value = ''
  }
  
  applyingCoupon.value = false
}

const handlePaymentSuccess = (order: Order) => {
  completedOrder.value = order
  cartStore.clearCart()
  currentStep.value = 3
}

const handlePaymentError = (error: string) => {
  console.error('Payment error:', error)
}
</script>
```

## Payment Integration (Stripe)

```typescript
// server/api/checkout/create-intent.post.ts
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16'
})

export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const { amount, currency = 'thb', orderId } = body

  try {
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // Convert to satang
      currency,
      metadata: {
        orderId
      },
      automatic_payment_methods: {
        enabled: true
      }
    })

    return {
      clientSecret: paymentIntent.client_secret,
      paymentIntentId: paymentIntent.id
    }
  } catch (error: any) {
    throw createError({
      statusCode: 400,
      message: error.message
    })
  }
})
```

```vue
<!-- components/checkout/StripePaymentForm.vue -->
<template>
  <div class="stripe-form">
    <div id="payment-element" ref="paymentElementRef"></div>
    
    <p v-if="error" class="payment-error" role="alert">{{ error }}</p>
    
    <button
      type="button"
      :disabled="!isReady || isProcessing"
      @click="handleSubmit"
      class="pay-button"
    >
      <span v-if="isProcessing">
        <span class="spinner" aria-hidden="true"></span>
        กำลังประมวลผล...
      </span>
      <span v-else>
        ชำระเงิน ฿{{ amount.toLocaleString('th-TH', { maximumFractionDigits: 2 }) }}
      </span>
    </button>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import { loadStripe, type Stripe, type StripeElements } from '@stripe/stripe-js'

const props = defineProps<{
  amount: number
  orderId: string
}>()

const emit = defineEmits(['success', 'error'])

const paymentElementRef = ref<HTMLElement | null>(null)
const isReady = ref(false)
const isProcessing = ref(false)
const error = ref('')

let stripe: Stripe | null = null
let elements: StripeElements | null = null

onMounted(async () => {
  try {
    // Create payment intent
    const { clientSecret } = await $fetch('/api/checkout/create-intent', {
      method: 'POST',
      body: { amount: props.amount, orderId: props.orderId }
    })

    // Initialize Stripe
    stripe = await loadStripe(import.meta.env.VITE_STRIPE_PUBLIC_KEY)
    if (!stripe) throw new Error('Failed to load Stripe')

    elements = stripe.elements({
      clientSecret,
      appearance: {
        theme: 'stripe',
        variables: {
          colorPrimary: '#6366f1'
        }
      }
    })

    const paymentElement = elements.create('payment')
    paymentElement.mount(paymentElementRef.value!)
    
    paymentElement.on('ready', () => {
      isReady.value = true
    })
  } catch (e: any) {
    error.value = e.message
  }
})

const handleSubmit = async () => {
  if (!stripe || !elements) return
  
  isProcessing.value = true
  error.value = ''

  try {
    const { error: stripeError, paymentIntent } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/checkout/success`
      },
      redirect: 'if_required'
    })

    if (stripeError) {
      error.value = stripeError.message || 'เกิดข้อผิดพลาดในการชำระเงิน'
      emit('error', stripeError.message)
    } else if (paymentIntent?.status === 'succeeded') {
      emit('success', paymentIntent)
    }
  } finally {
    isProcessing.value = false
  }
}
</script>
```

## Order Management

```typescript
// server/api/orders/index.post.ts
import { prisma } from '~/server/db'

export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  const body = await readBody(event)

  const {
    items,
    shippingAddress,
    billingAddress,
    paymentIntentId,
    couponCode
  } = body

  // Validate stock
  for (const item of items) {
    const product = await prisma.product.findUnique({
      where: { id: item.productId },
      include: { variants: true }
    })

    if (!product) {
      throw createError({ statusCode: 404, message: `Product ${item.productId} not found` })
    }

    const stock = item.variantId
      ? product.variants.find(v => v.id === item.variantId)?.stock
      : product.inventory.available

    if (!stock || stock < item.quantity) {
      throw createError({ statusCode: 400, message: `Insufficient stock for ${product.name}` })
    }
  }

  // Create order in transaction
  const order = await prisma.$transaction(async (tx) => {
    // Calculate totals
    const orderItems = await Promise.all(items.map(async (item: any) => {
      const product = await tx.product.findUnique({ where: { id: item.productId } })
      return {
        productId: item.productId,
        variantId: item.variantId,
        quantity: item.quantity,
        price: item.price,
        subtotal: item.price * item.quantity
      }
    }))

    const subtotal = orderItems.reduce((sum: number, i: any) => sum + i.subtotal, 0)
    const tax = subtotal * 0.07
    const shipping = subtotal >= 500 ? 0 : 50
    const total = subtotal + tax + shipping

    // Create order
    const newOrder = await tx.order.create({
      data: {
        userId: user.id,
        orderNumber: `ORD-${Date.now()}`,
        status: 'confirmed',
        paymentStatus: 'paid',
        paymentIntentId,
        subtotal,
        tax,
        shipping,
        total,
        shippingAddress: { create: shippingAddress },
        billingAddress: { create: billingAddress },
        items: { create: orderItems }
      },
      include: {
        items: { include: { product: true, variant: true } },
        shippingAddress: true
      }
    })

    // Update inventory
    for (const item of items) {
      if (item.variantId) {
        await tx.productVariant.update({
          where: { id: item.variantId },
          data: { stock: { decrement: item.quantity } }
        })
      } else {
        await tx.product.update({
          where: { id: item.productId },
          data: { inventory: { update: { quantity: { decrement: item.quantity } } } }
        })
      }
    }

    return newOrder
  })

  // Send confirmation email (async, don't await)
  sendOrderConfirmationEmail(order).catch(console.error)

  return order
})
```

## Inventory System

```typescript
// server/api/admin/inventory/[productId].put.ts
import { prisma } from '~/server/db'

export default defineEventHandler(async (event) => {
  await requireAdmin(event)
  
  const productId = getRouterParam(event, 'productId')
  const body = await readBody(event)
  const { quantity, reason, note } = body

  const product = await prisma.product.findUnique({
    where: { id: productId },
    include: { inventory: true }
  })

  if (!product) {
    throw createError({ statusCode: 404, message: 'Product not found' })
  }

  const previousQty = product.inventory?.quantity || 0
  const adjustment = quantity - previousQty

  await prisma.$transaction([
    // Update inventory
    prisma.inventory.update({
      where: { productId },
      data: { quantity }
    }),
    // Log inventory change
    prisma.inventoryLog.create({
      data: {
        productId,
        previousQuantity: previousQty,
        newQuantity: quantity,
        adjustment,
        reason,
        note
      }
    })
  ])

  // Check low stock alert
  if (quantity <= (product.inventory?.lowStockThreshold || 10)) {
    await sendLowStockAlert(product, quantity)
  }

  return { success: true, quantity }
})
```

## ตัวอย่าง: Complete E-commerce Flow

```typescript
// composables/useProduct.ts
export function useProduct(slug: string) {
  const product = ref<Product | null>(null)
  const loading = ref(false)
  const selectedVariant = ref<ProductVariant | null>(null)
  const selectedOptions = ref<Record<string, string>>({})
  const quantity = ref(1)
  const cartStore = useCartStore()
  const { announce } = useAnnouncer()

  const fetchProduct = async () => {
    loading.value = true
    try {
      product.value = await $fetch(`/api/products/${slug}`)
    } finally {
      loading.value = false
    }
  }

  const currentVariant = computed(() => {
    if (!product.value?.variants.length) return null
    return product.value.variants.find(v =>
      Object.entries(selectedOptions.value).every(
        ([key, val]) => v.options[key] === val
      )
    ) || null
  })

  const currentPrice = computed(() =>
    currentVariant.value?.price ?? product.value?.price ?? 0
  )

  const isAvailable = computed(() => {
    const stock = currentVariant.value?.stock ?? product.value?.inventory.available ?? 0
    return stock > 0
  })

  const addToCart = () => {
    if (!product.value || !isAvailable.value) return
    cartStore.addItem(product.value, currentVariant.value || undefined, quantity.value)
    announce(`เพิ่ม ${product.value.name} ลงตะกร้าแล้ว`)
  }

  onMounted(fetchProduct)

  return {
    product,
    loading,
    selectedOptions,
    selectedVariant: currentVariant,
    currentPrice,
    isAvailable,
    quantity,
    addToCart
  }
}
```

## สรุป

การสร้าง E-commerce Platform ต้องการ:
1. Type-safe ด้วย TypeScript
2. Cart state management ด้วย Pinia
3. Payment integration ด้วย Stripe
4. Order management ด้วย Prisma
5. Inventory tracking
6. Email notifications
7. SEO optimization สำหรับ product pages
