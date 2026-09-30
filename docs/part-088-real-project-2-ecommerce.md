# Part 88: โปรเจกต์จริง 2 - E-commerce Store

## Overview

สร้าง Full E-commerce Store ด้วย Nuxt 3 ที่มีระบบ:
- Product catalog with variants
- Shopping cart + Wishlist
- User authentication
- Stripe payment
- Order management
- Admin dashboard
- Email notifications

---

## 1. Database Schema

```prisma
// prisma/schema.prisma
model Product {
  id          String   @id @default(cuid())
  name        String
  slug        String   @unique
  description String   @db.Text
  price       Decimal  @db.Decimal(10, 2)
  comparePrice Decimal? @db.Decimal(10, 2)
  images      String[]
  status      ProductStatus @default(ACTIVE)
  stock       Int      @default(0)
  sku         String?  @unique
  
  categoryId  String
  category    Category @relation(fields: [categoryId], references: [id])
  
  variants    ProductVariant[]
  orderItems  OrderItem[]
  cartItems   CartItem[]
  wishlistItems WishlistItem[]
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@map("products")
}

enum ProductStatus {
  ACTIVE
  INACTIVE
  OUT_OF_STOCK
}

model ProductVariant {
  id        String   @id @default(cuid())
  productId String
  product   Product  @relation(fields: [productId], references: [id])
  name      String   // e.g., "Red - XL"
  sku       String   @unique
  price     Decimal? @db.Decimal(10, 2) // override product price
  stock     Int      @default(0)
  options   Json     // { color: "Red", size: "XL" }

  @@map("product_variants")
}

model Cart {
  id        String     @id @default(cuid())
  userId    String?    // null for guest carts
  sessionId String?    // for guest
  items     CartItem[]
  createdAt DateTime   @default(now())
  updatedAt DateTime   @updatedAt

  @@map("carts")
}

model CartItem {
  id        String  @id @default(cuid())
  cartId    String
  cart      Cart    @relation(fields: [cartId], references: [id], onDelete: Cascade)
  productId String
  product   Product @relation(fields: [productId], references: [id])
  variantId String?
  quantity  Int     @default(1)

  @@unique([cartId, productId, variantId])
  @@map("cart_items")
}

model Order {
  id              String      @id @default(cuid())
  orderNumber     String      @unique
  userId          String
  user            User        @relation(fields: [userId], references: [id])
  status          OrderStatus @default(PENDING)
  subtotal        Decimal     @db.Decimal(10, 2)
  tax             Decimal     @db.Decimal(10, 2)
  shipping        Decimal     @db.Decimal(10, 2)
  total           Decimal     @db.Decimal(10, 2)
  
  shippingAddress Json
  billingAddress  Json?
  
  stripePaymentIntentId String?
  paidAt          DateTime?
  
  items           OrderItem[]
  tracking        OrderTracking[]
  
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt

  @@map("orders")
}

enum OrderStatus {
  PENDING
  PAID
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

model OrderItem {
  id        String  @id @default(cuid())
  orderId   String
  order     Order   @relation(fields: [orderId], references: [id])
  productId String
  product   Product @relation(fields: [productId], references: [id])
  quantity  Int
  price     Decimal @db.Decimal(10, 2)
  variantOptions Json?

  @@map("order_items")
}

model Wishlist {
  id        String         @id @default(cuid())
  userId    String         @unique
  items     WishlistItem[]

  @@map("wishlists")
}

model WishlistItem {
  id         String  @id @default(cuid())
  wishlistId String
  wishlist   Wishlist @relation(fields: [wishlistId], references: [id])
  productId  String
  product    Product @relation(fields: [productId], references: [id])

  @@unique([wishlistId, productId])
  @@map("wishlist_items")
}
```

---

## 2. Product APIs

```typescript
// server/api/products/index.get.ts
export default defineEventHandler(async (event) => {
  const { page = 1, limit = 12, category, search, minPrice, maxPrice, sort } = getQuery(event)

  const where: any = { status: 'ACTIVE' }

  if (category) where.category = { slug: String(category) }
  if (search) {
    where.OR = [
      { name: { contains: String(search), mode: 'insensitive' } },
      { description: { contains: String(search), mode: 'insensitive' } }
    ]
  }
  if (minPrice || maxPrice) {
    where.price = {
      ...(minPrice && { gte: Number(minPrice) }),
      ...(maxPrice && { lte: Number(maxPrice) })
    }
  }

  const orderBy: any = {
    newest: { createdAt: 'desc' },
    'price-low': { price: 'asc' },
    'price-high': { price: 'desc' },
    popular: { orderItems: { _count: 'desc' } }
  }[String(sort)] || { createdAt: 'desc' }

  const [products, total] = await Promise.all([
    prisma.product.findMany({
      where,
      select: {
        id: true,
        name: true,
        slug: true,
        price: true,
        comparePrice: true,
        images: true,
        stock: true,
        category: { select: { name: true, slug: true } }
      },
      orderBy,
      skip: (Number(page) - 1) * Number(limit),
      take: Number(limit)
    }),
    prisma.product.count({ where })
  ])

  return {
    products,
    meta: {
      total,
      page: Number(page),
      limit: Number(limit),
      totalPages: Math.ceil(total / Number(limit))
    }
  }
})
```

---

## 3. Cart System

### Cart Store

```typescript
// stores/cart.ts
import { defineStore } from 'pinia'

interface CartItem {
  id: string
  productId: string
  product: {
    id: string
    name: string
    price: number
    images: string[]
    slug: string
  }
  variantId?: string
  variantOptions?: Record<string, string>
  quantity: number
}

export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])
  const loading = ref(false)

  const totalItems = computed(() =>
    items.value.reduce((sum, item) => sum + item.quantity, 0)
  )

  const subtotal = computed(() =>
    items.value.reduce((sum, item) => sum + item.product.price * item.quantity, 0)
  )

  const tax = computed(() => subtotal.value * 0.07) // 7% VAT
  const shipping = computed(() => subtotal.value > 1000 ? 0 : 50) // Free shipping over ฿1000
  const total = computed(() => subtotal.value + tax.value + shipping.value)

  async function loadCart() {
    loading.value = true
    try {
      const data = await $fetch<CartItem[]>('/api/cart')
      items.value = data
    } finally {
      loading.value = false
    }
  }

  async function addItem(productId: string, quantity = 1, variantId?: string) {
    const existingIndex = items.value.findIndex(
      item => item.productId === productId && item.variantId === variantId
    )

    if (existingIndex !== -1) {
      items.value[existingIndex].quantity += quantity
    } else {
      const product = await $fetch(`/api/products/${productId}`)
      items.value.push({
        id: Math.random().toString(),
        productId,
        product: product as any,
        variantId,
        quantity
      })
    }

    await syncCart()
  }

  async function removeItem(itemId: string) {
    items.value = items.value.filter(item => item.id !== itemId)
    await syncCart()
  }

  async function updateQuantity(itemId: string, quantity: number) {
    if (quantity <= 0) {
      return removeItem(itemId)
    }
    const item = items.value.find(i => i.id === itemId)
    if (item) item.quantity = quantity
    await syncCart()
  }

  async function clear() {
    items.value = []
    await $fetch('/api/cart', { method: 'DELETE' })
  }

  async function syncCart() {
    await $fetch('/api/cart/sync', {
      method: 'POST',
      body: { items: items.value }
    })
  }

  return {
    items,
    loading,
    totalItems,
    subtotal,
    tax,
    shipping,
    total,
    loadCart,
    addItem,
    removeItem,
    updateQuantity,
    clear
  }
})
```

### Cart Component

```vue
<!-- components/CartSidebar.vue -->
<template>
  <Transition name="slide-right">
    <div v-if="isOpen" class="cart-overlay" @click.self="close">
      <div class="cart-sidebar">
        <div class="cart-header">
          <h2>Shopping Cart ({{ totalItems }})</h2>
          <button @click="close" aria-label="Close cart">×</button>
        </div>

        <div class="cart-items">
          <div v-if="items.length === 0" class="empty-cart">
            <ShoppingBagIcon class="icon" />
            <p>Your cart is empty</p>
            <NuxtLink to="/products" @click="close">Continue Shopping</NuxtLink>
          </div>

          <CartItem
            v-for="item in items"
            :key="item.id"
            :item="item"
            @remove="removeItem(item.id)"
            @update-quantity="updateQuantity(item.id, $event)"
          />
        </div>

        <div class="cart-footer">
          <div class="totals">
            <div class="row">
              <span>Subtotal</span>
              <span>{{ formatPrice(subtotal) }}</span>
            </div>
            <div class="row">
              <span>Tax (7%)</span>
              <span>{{ formatPrice(tax) }}</span>
            </div>
            <div class="row">
              <span>Shipping</span>
              <span>{{ shipping === 0 ? 'Free' : formatPrice(shipping) }}</span>
            </div>
            <div class="row total">
              <strong>Total</strong>
              <strong>{{ formatPrice(total) }}</strong>
            </div>
          </div>

          <NuxtLink to="/checkout" class="btn-primary checkout-btn" @click="close">
            Proceed to Checkout
          </NuxtLink>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup lang="ts">
const store = useCartStore()
const { items, totalItems, subtotal, tax, shipping, total } = storeToRefs(store)
const { removeItem, updateQuantity } = store

const isOpen = ref(false)

function open() { isOpen.value = true }
function close() { isOpen.value = false }

function formatPrice(price: number) {
  return new Intl.NumberFormat('th-TH', { style: 'currency', currency: 'THB' }).format(price)
}

defineExpose({ open, close })
</script>
```

---

## 4. Checkout & Payment

```vue
<!-- pages/checkout.vue -->
<template>
  <div class="checkout">
    <div class="checkout-form">
      <h1>Checkout</h1>

      <!-- Shipping Address -->
      <section>
        <h2>Shipping Address</h2>
        <form @submit.prevent="proceedToPayment">
          <div class="form-grid">
            <input v-model="form.firstName" placeholder="First Name" required />
            <input v-model="form.lastName" placeholder="Last Name" required />
            <input v-model="form.email" type="email" placeholder="Email" required />
            <input v-model="form.phone" placeholder="Phone" required />
            <input v-model="form.address" placeholder="Address" class="col-span-2" required />
            <input v-model="form.city" placeholder="City" required />
            <input v-model="form.postalCode" placeholder="Postal Code" required />
          </div>
        </form>
      </section>

      <!-- Payment -->
      <section>
        <h2>Payment</h2>
        <div id="payment-element"></div>
      </section>

      <button @click="placeOrder" :disabled="processing" class="btn-primary">
        {{ processing ? 'Processing...' : `Pay ${formatPrice(total)}` }}
      </button>
    </div>

    <!-- Order Summary -->
    <div class="order-summary">
      <h2>Order Summary</h2>
      <div v-for="item in items" :key="item.id" class="summary-item">
        <img :src="item.product.images[0]" :alt="item.product.name" />
        <div>
          <p>{{ item.product.name }}</p>
          <p>Qty: {{ item.quantity }}</p>
        </div>
        <p>{{ formatPrice(item.product.price * item.quantity) }}</p>
      </div>
      
      <div class="summary-totals">
        <div class="row"><span>Subtotal</span><span>{{ formatPrice(subtotal) }}</span></div>
        <div class="row"><span>Shipping</span><span>{{ shipping === 0 ? 'Free' : formatPrice(shipping) }}</span></div>
        <div class="row total"><strong>Total</strong><strong>{{ formatPrice(total) }}</strong></div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { loadStripe } from '@stripe/stripe-js'

definePageMeta({ middleware: 'auth' })

const config = useRuntimeConfig()
const store = useCartStore()
const { items, subtotal, shipping, total } = storeToRefs(store)

const form = reactive({
  firstName: '',
  lastName: '',
  email: '',
  phone: '',
  address: '',
  city: '',
  postalCode: ''
})

const stripe = ref<any>(null)
const elements = ref<any>(null)
const processing = ref(false)

onMounted(async () => {
  stripe.value = await loadStripe(config.public.stripePublishableKey)

  // Create payment intent
  const { clientSecret } = await $fetch('/api/orders/create-intent', {
    method: 'POST',
    body: { amount: Math.round(total.value * 100) }
  })

  elements.value = stripe.value.elements({ clientSecret })
  const paymentElement = elements.value.create('payment')
  paymentElement.mount('#payment-element')
})

async function placeOrder() {
  processing.value = true
  try {
    // Confirm payment
    const { error, paymentIntent } = await stripe.value.confirmPayment({
      elements: elements.value,
      confirmParams: {
        return_url: `${window.location.origin}/orders/success`,
        receipt_email: form.email
      },
      redirect: 'if_required'
    })

    if (error) throw new Error(error.message)

    // Create order in our system
    const order = await $fetch('/api/orders', {
      method: 'POST',
      body: {
        shippingAddress: form,
        paymentIntentId: paymentIntent.id,
        items: items.value.map(item => ({
          productId: item.productId,
          variantId: item.variantId,
          quantity: item.quantity,
          price: item.product.price
        }))
      }
    })

    await store.clear()
    navigateTo(`/orders/${order.orderNumber}/success`)
  } catch (error) {
    console.error(error)
  } finally {
    processing.value = false
  }
}

function formatPrice(amount: number) {
  return new Intl.NumberFormat('th-TH', { style: 'currency', currency: 'THB' }).format(amount)
}
</script>
```

---

## 5. Order Management API

```typescript
// server/api/orders/index.post.ts
export default defineEventHandler(async (event) => {
  await requireAuth(event)
  const body = await readBody(event)
  const userId = event.context.userId

  const { shippingAddress, paymentIntentId, items } = body

  // ตรวจสอบ stock
  for (const item of items) {
    const product = await prisma.product.findUnique({
      where: { id: item.productId }
    })
    if (!product || product.stock < item.quantity) {
      throw createError({
        statusCode: 422,
        message: `${product?.name || 'Product'} is out of stock`
      })
    }
  }

  const orderNumber = `ORD-${Date.now()}`
  const subtotal = items.reduce((sum: number, item: any) => sum + item.price * item.quantity, 0)
  const tax = subtotal * 0.07
  const shipping = subtotal > 1000 ? 0 : 50
  const total = subtotal + tax + shipping

  const order = await prisma.$transaction(async (tx) => {
    // สร้าง order
    const newOrder = await tx.order.create({
      data: {
        orderNumber,
        userId,
        status: 'PAID',
        subtotal,
        tax,
        shipping,
        total,
        shippingAddress,
        stripePaymentIntentId: paymentIntentId,
        paidAt: new Date(),
        items: {
          create: items.map((item: any) => ({
            productId: item.productId,
            variantId: item.variantId,
            quantity: item.quantity,
            price: item.price
          }))
        }
      },
      include: { items: true }
    })

    // ลด stock
    for (const item of items) {
      await tx.product.update({
        where: { id: item.productId },
        data: { stock: { decrement: item.quantity } }
      })
    }

    return newOrder
  })

  // ส่ง email confirmation
  await emailService.sendOrderConfirmation(userId, order)

  return order
})
```

---

## 6. Admin Dashboard

```vue
<!-- pages/admin/dashboard.vue -->
<template>
  <div class="admin-dashboard">
    <h1>Dashboard</h1>

    <!-- KPI Cards -->
    <div class="kpi-grid">
      <KpiCard title="Total Revenue" :value="stats.revenue" format="currency" :change="+12.5" />
      <KpiCard title="Orders Today" :value="stats.ordersToday" :change="+5" />
      <KpiCard title="Active Users" :value="stats.activeUsers" :change="+18" />
      <KpiCard title="Products" :value="stats.products" :change="0" />
    </div>

    <!-- Sales Chart -->
    <div class="chart-container">
      <h2>Revenue (Last 30 days)</h2>
      <RevenueChart :data="revenueData" />
    </div>

    <!-- Recent Orders -->
    <div class="recent-orders">
      <h2>Recent Orders</h2>
      <OrdersTable :orders="recentOrders" />
    </div>

    <!-- Top Products -->
    <div class="top-products">
      <h2>Top Products</h2>
      <ProductsTable :products="topProducts" />
    </div>
  </div>
</template>

<script setup lang="ts">
definePageMeta({ middleware: 'admin' })

const { data: stats } = useFetch('/api/admin/stats')
const { data: revenueData } = useFetch('/api/admin/revenue', {
  query: { days: 30 }
})
const { data: recentOrders } = useFetch('/api/admin/orders', {
  query: { limit: 10 }
})
const { data: topProducts } = useFetch('/api/admin/products/top')
</script>
```

---

## 7. Email Notifications

```typescript
// server/services/email.service.ts
import { Resend } from 'resend'

const resend = new Resend(process.env.RESEND_API_KEY)

export const emailService = {
  async sendOrderConfirmation(userId: string, order: any) {
    const user = await prisma.user.findUnique({ where: { id: userId } })
    if (!user) return

    await resend.emails.send({
      from: 'shop@myecommerce.com',
      to: user.email,
      subject: `Order Confirmed - ${order.orderNumber}`,
      html: generateOrderConfirmationEmail(order, user)
    })
  },

  async sendShippingUpdate(orderId: string, trackingNumber: string) {
    const order = await prisma.order.findUnique({
      where: { id: orderId },
      include: { user: true }
    })
    if (!order) return

    await resend.emails.send({
      from: 'shop@myecommerce.com',
      to: order.user.email,
      subject: `Your order is on its way! - ${order.orderNumber}`,
      html: generateShippingEmail(order, trackingNumber)
    })
  }
}

function generateOrderConfirmationEmail(order: any, user: any) {
  return `
    <!DOCTYPE html>
    <html>
    <body style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
      <h1 style="color: #333;">Order Confirmed! 🎉</h1>
      <p>Hi ${user.name},</p>
      <p>Thank you for your order! Here's your order summary:</p>
      
      <div style="background: #f5f5f5; padding: 20px; border-radius: 8px;">
        <h2>Order #${order.orderNumber}</h2>
        ${order.items.map((item: any) => `
          <div style="display: flex; justify-content: space-between; margin: 10px 0;">
            <span>${item.product?.name || 'Product'} × ${item.quantity}</span>
            <span>฿${(item.price * item.quantity).toFixed(2)}</span>
          </div>
        `).join('')}
        <hr />
        <div style="display: flex; justify-content: space-between;">
          <strong>Total</strong>
          <strong>฿${order.total.toFixed(2)}</strong>
        </div>
      </div>
      
      <p>We'll send you another email when your order ships.</p>
      <a href="${process.env.APP_URL}/orders/${order.orderNumber}" 
         style="background: #3b82f6; color: white; padding: 12px 24px; 
                border-radius: 6px; text-decoration: none;">
        View Order
      </a>
    </body>
    </html>
  `
}
```

---

## สรุป

E-commerce project ครอบคลุม:
1. **Product catalog** พร้อม variants และ filtering
2. **Shopping cart** พร้อม persistence
3. **Stripe payment** integration
4. **Order management** และ tracking
5. **Admin dashboard** พร้อม analytics
6. **Email notifications** สำหรับ order lifecycle
