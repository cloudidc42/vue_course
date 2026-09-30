# Part 51: Micro-frontend Architecture กับ Vue.js

## Micro-frontend คืออะไร?

Micro-frontend เป็นสถาปัตยกรรมที่แบ่งแอปพลิเคชัน frontend ขนาดใหญ่ออกเป็นหลาย "micro apps" ที่พัฒนาและ deploy ได้อิสระ คล้ายกับ Microservices แต่สำหรับ frontend

## ประโยชน์ของ Micro-frontend

- **Team Independence** - แต่ละทีมพัฒนาได้อิสระ
- **Technology Freedom** - แต่ละ app ใช้ framework ต่างกันได้
- **Incremental Upgrades** - อัปเกรดทีละส่วน
- **Deployment Independence** - deploy ได้อิสระ

## 1. Micro-frontend Approaches

```
Approach 1: iFrame (เก่า, ไม่แนะนำ)
Approach 2: Web Components
Approach 3: Module Federation (แนะนำ สำหรับ Vite/Webpack)
Approach 4: Single-SPA Framework
```

## 2. Module Federation กับ Vite

```bash
# ติดตั้ง @originjs/vite-plugin-federation
npm install @originjs/vite-plugin-federation --save-dev
```

### โครงสร้างโปรเจกต์

```
projects/
├── shell/              # Host App (Main container)
│   ├── vite.config.ts
│   └── src/
├── products/           # Micro-app 1
│   ├── vite.config.ts
│   └── src/
├── cart/               # Micro-app 2
│   ├── vite.config.ts
│   └── src/
└── auth/               # Micro-app 3
    ├── vite.config.ts
    └── src/
```

### Products Micro-app (Remote)

```typescript
// products/vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    vue(),
    federation({
      name: 'products',
      filename: 'remoteEntry.js',
      
      // Component ที่ expose ให้ apps อื่นใช้
      exposes: {
        './ProductList': './src/components/ProductList.vue',
        './ProductCard': './src/components/ProductCard.vue',
        './ProductDetail': './src/components/ProductDetail.vue',
        './useProducts': './src/composables/useProducts.ts',
      },
      
      // Shared dependencies (หลีกเลี่ยง duplicate)
      shared: {
        vue: {
          requiredVersion: '^3.0.0',
          singleton: true
        },
        'pinia': {
          requiredVersion: '^2.0.0',
          singleton: true
        },
        'vue-router': {
          requiredVersion: '^4.0.0',
          singleton: true
        }
      }
    })
  ],
  
  build: {
    target: 'esnext',
    minify: false,
    cssCodeSplit: false
  },
  
  server: {
    port: 5001,
    cors: true
  },
  
  preview: {
    port: 5001,
    cors: true
  }
})
```

```vue
<!-- products/src/components/ProductList.vue -->
<template>
  <div class="product-list">
    <div v-if="loading" class="loading">กำลังโหลด...</div>
    
    <div v-else class="grid">
      <ProductCard
        v-for="product in products"
        :key="product.id"
        :product="product"
        @add-to-cart="handleAddToCart"
      />
    </div>
    
    <Pagination
      v-if="totalPages > 1"
      :current-page="currentPage"
      :total-pages="totalPages"
      @change="changePage"
    />
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import ProductCard from './ProductCard.vue'
import Pagination from './Pagination.vue'

const products = ref([])
const loading = ref(true)
const currentPage = ref(1)
const totalPages = ref(1)

const loadProducts = async (page = 1) => {
  loading.value = true
  try {
    const response = await fetch(`/api/products?page=${page}`)
    const data = await response.json()
    products.value = data.products
    totalPages.value = data.totalPages
    currentPage.value = page
  } finally {
    loading.value = false
  }
}

const handleAddToCart = (product) => {
  // Communicate กับ shell app ผ่าน custom event
  window.dispatchEvent(new CustomEvent('cart:add', {
    detail: { product }
  }))
}

const changePage = (page) => {
  loadProducts(page)
}

onMounted(() => loadProducts())
</script>
```

### Shell App (Host)

```typescript
// shell/vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    vue(),
    federation({
      name: 'shell',
      
      // Remote apps
      remotes: {
        products: {
          external: 'http://localhost:5001/assets/remoteEntry.js',
          format: 'esm',
          from: 'vite'
        },
        cart: {
          external: 'http://localhost:5002/assets/remoteEntry.js',
          format: 'esm',
          from: 'vite'
        },
        auth: {
          external: 'http://localhost:5003/assets/remoteEntry.js',
          format: 'esm',
          from: 'vite'
        }
      },
      
      shared: {
        vue: { singleton: true, requiredVersion: '^3.0.0' },
        pinia: { singleton: true, requiredVersion: '^2.0.0' },
        'vue-router': { singleton: true, requiredVersion: '^4.0.0' }
      }
    })
  ],
  
  build: {
    target: 'esnext',
    minify: false,
    cssCodeSplit: false
  }
})
```

```typescript
// shell/src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { defineAsyncComponent } from 'vue'

// Lazy load remote components
const ProductList = defineAsyncComponent(() => import('products/ProductList'))
const CartPage = defineAsyncComponent(() => import('cart/CartPage'))
const LoginPage = defineAsyncComponent(() => import('auth/LoginPage'))

export const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('../layouts/MainLayout.vue'),
      children: [
        {
          path: '',
          component: () => import('../pages/HomePage.vue')
        },
        {
          path: 'products',
          component: ProductList
        },
        {
          path: 'cart',
          component: CartPage
        }
      ]
    },
    {
      path: '/login',
      component: LoginPage
    }
  ]
})
```

## 3. Shared State ระหว่าง Apps

### ใช้ Pinia เป็น Shared Store

```typescript
// shared/src/stores/cartStore.ts
import { defineStore } from 'pinia'

export interface CartItem {
  id: string
  name: string
  price: number
  quantity: number
  image: string
}

export const useCartStore = defineStore('cart', {
  state: () => ({
    items: [] as CartItem[],
    isOpen: false
  }),
  
  getters: {
    totalItems: (state) => state.items.reduce((sum, item) => sum + item.quantity, 0),
    totalPrice: (state) => state.items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    isEmpty: (state) => state.items.length === 0
  },
  
  actions: {
    addItem(product: Omit<CartItem, 'quantity'>) {
      const existing = this.items.find(item => item.id === product.id)
      
      if (existing) {
        existing.quantity++
      } else {
        this.items.push({ ...product, quantity: 1 })
      }
      
      // บันทึกใน localStorage
      this.persistCart()
    },
    
    removeItem(id: string) {
      this.items = this.items.filter(item => item.id !== id)
      this.persistCart()
    },
    
    updateQuantity(id: string, quantity: number) {
      const item = this.items.find(item => item.id === id)
      if (item) {
        item.quantity = quantity
        if (quantity <= 0) this.removeItem(id)
      }
      this.persistCart()
    },
    
    clearCart() {
      this.items = []
      this.persistCart()
    },
    
    toggleCart() {
      this.isOpen = !this.isOpen
    },
    
    persistCart() {
      localStorage.setItem('cart', JSON.stringify(this.items))
    },
    
    loadCart() {
      const saved = localStorage.getItem('cart')
      if (saved) {
        this.items = JSON.parse(saved)
      }
    }
  }
})
```

### Custom Events สำหรับ Cross-app Communication

```typescript
// shared/src/eventBus.ts
type EventMap = {
  'cart:add': { product: any }
  'cart:remove': { productId: string }
  'cart:clear': undefined
  'auth:login': { user: any }
  'auth:logout': undefined
  'notification:show': { message: string; type: 'success' | 'error' | 'info' }
  'navigation:push': { path: string }
}

class MicroFrontendEventBus {
  private listeners: Map<string, Set<Function>> = new Map()
  
  emit<T extends keyof EventMap>(
    event: T,
    data?: EventMap[T]
  ) {
    // ส่ง event ไปยัง window สำหรับ cross-app communication
    window.dispatchEvent(
      new CustomEvent(`mf:${String(event)}`, {
        detail: data,
        bubbles: true
      })
    )
    
    // ส่งให้ local listeners
    const handlers = this.listeners.get(String(event))
    handlers?.forEach(handler => handler(data))
  }
  
  on<T extends keyof EventMap>(
    event: T,
    handler: (data: EventMap[T]) => void
  ) {
    // Listen จาก window events
    const windowHandler = (e: Event) => {
      handler((e as CustomEvent).detail)
    }
    
    window.addEventListener(`mf:${String(event)}`, windowHandler)
    
    return () => {
      window.removeEventListener(`mf:${String(event)}`, windowHandler)
    }
  }
}

export const eventBus = new MicroFrontendEventBus()
```

## 4. Routing ใน Micro-frontend

```typescript
// shell/src/composables/useMicroRouter.ts
export const useMicroRouter = () => {
  const router = useRouter()
  
  // ฟัง navigation events จาก micro-apps
  const unsubscribe = eventBus.on('navigation:push', ({ path }) => {
    router.push(path)
  })
  
  onUnmounted(unsubscribe)
  
  // Helper function สำหรับ navigate จาก micro-apps
  const navigateTo = (path: string) => {
    eventBus.emit('navigation:push', { path })
  }
  
  return { navigateTo }
}
```

## 5. ตัวอย่าง: Multi-team Project Setup

```vue
<!-- shell/src/App.vue -->
<template>
  <div id="app">
    <!-- Global Navigation -->
    <header class="app-header">
      <nav class="nav">
        <NuxtLink to="/">Home</NuxtLink>
        <NuxtLink to="/products">สินค้า</NuxtLink>
        <NuxtLink to="/about">เกี่ยวกับ</NuxtLink>
      </nav>
      
      <!-- Cart Icon (จาก Cart micro-app) -->
      <Suspense>
        <CartIcon />
        <template #fallback>
          <div class="cart-loading">🛒</div>
        </template>
      </Suspense>
    </header>
    
    <!-- Main Content -->
    <main>
      <!-- Error boundary สำหรับ micro-app failures -->
      <ErrorBoundary :key="$route.path">
        <RouterView />
        <template #error="{ error }">
          <div class="micro-app-error">
            <h2>ไม่สามารถโหลดส่วนนี้ได้</h2>
            <p>{{ error.message }}</p>
            <button @click="$router.go(0)">รีโหลด</button>
          </div>
        </template>
      </ErrorBoundary>
    </main>
    
    <!-- Notifications (Global) -->
    <NotificationContainer />
    
    <!-- Cart Drawer -->
    <CartDrawer />
  </div>
</template>

<script setup>
import { defineAsyncComponent } from 'vue'
import ErrorBoundary from './components/ErrorBoundary.vue'
import NotificationContainer from './components/NotificationContainer.vue'

// Lazy load จาก remote apps
const CartIcon = defineAsyncComponent({
  loader: () => import('cart/CartIcon'),
  loadingComponent: { template: '<div>🛒</div>' },
  errorComponent: { template: '<div>🛒</div>' },
  delay: 200,
  timeout: 5000
})

const CartDrawer = defineAsyncComponent({
  loader: () => import('cart/CartDrawer'),
  loadingComponent: { template: '<div/>' },
  errorComponent: { template: '<div/>' }
})

const { cartStore } = useCartStore()
const { authStore } = useAuthStore()

// Listen สำหรับ events จาก micro-apps
onMounted(() => {
  const cleanupCart = eventBus.on('cart:add', ({ product }) => {
    cartStore.addItem(product)
  })
  
  const cleanupAuth = eventBus.on('auth:login', ({ user }) => {
    authStore.setUser(user)
  })
  
  onUnmounted(() => {
    cleanupCart()
    cleanupAuth()
  })
})
</script>
```

## 6. Deployment Strategies

```yaml
# docker-compose.micro-frontend.yml
version: '3.9'

services:
  # Shell app
  shell:
    build: ./shell
    ports:
      - "3000:3000"
    environment:
      PRODUCTS_URL: http://products:5001
      CART_URL: http://cart:5002
      AUTH_URL: http://auth:5003
  
  # Products micro-app
  products:
    build: ./products
    ports:
      - "5001:5001"
  
  # Cart micro-app
  cart:
    build: ./cart
    ports:
      - "5002:5002"
  
  # Auth micro-app
  auth:
    build: ./auth
    ports:
      - "5003:5003"
  
  # API Gateway / Nginx
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/micro-frontend.conf:/etc/nginx/conf.d/default.conf
```

```nginx
# nginx/micro-frontend.conf
server {
    listen 80;
    
    # Shell app (main)
    location / {
        proxy_pass http://shell:3000;
    }
    
    # Remote Entry files
    location /products-remote/ {
        proxy_pass http://products:5001/;
        add_header Access-Control-Allow-Origin *;
    }
    
    location /cart-remote/ {
        proxy_pass http://cart:5002/;
        add_header Access-Control-Allow-Origin *;
    }
    
    location /auth-remote/ {
        proxy_pass http://auth:5003/;
        add_header Access-Control-Allow-Origin *;
    }
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Micro-frontend Concepts** - แนวคิดและประโยชน์
2. **Module Federation** - การแชร์ code ระหว่าง apps
3. **Shared State** - ใช้ Pinia ร่วมกัน
4. **Event Bus** - Cross-app communication
5. **Routing** - จัดการ routing ใน micro-frontend
6. **Error Boundaries** - จัดการเมื่อ micro-app ล้มเหลว
7. **Deployment** - Docker + Nginx สำหรับ production
