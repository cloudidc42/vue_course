# Part 67: Advanced Testing Strategies

## Testing Pyramid

```
        ┌─────────────────────────────┐
        │        E2E Tests            │  (ช้า, แพง)
        │   Playwright / Cypress      │
        ├─────────────────────────────┤
        │     Integration Tests       │
        │   API + Component Tests     │
        ├─────────────────────────────┤
        │       Unit Tests            │  (เร็ว, ถูก)
        │  Vitest / Vue Test Utils    │
        └─────────────────────────────┘
```

## Component Testing Patterns

```typescript
// tests/components/ProductCard.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import ProductCard from '@/components/ProductCard.vue'

const mockProduct = {
  id: '1',
  name: 'Vue.js Course',
  slug: 'vue-js-course',
  price: 990,
  comparePrice: 1500,
  images: [{ url: '/course.jpg', alt: 'Course cover' }],
  category: { id: '1', name: 'Programming', slug: 'programming' },
  inventory: { available: 100, quantity: 100, reserved: 0, lowStockThreshold: 10 },
  rating: 4.5,
  reviewCount: 42,
  createdAt: new Date().toISOString(),
  variants: []
}

describe('ProductCard', () => {
  it('renders product name', () => {
    const wrapper = mount(ProductCard, {
      props: { product: mockProduct }
    })
    expect(wrapper.text()).toContain('Vue.js Course')
  })
  
  it('shows discount badge when compare price exists', () => {
    const wrapper = mount(ProductCard, {
      props: { product: mockProduct }
    })
    const badge = wrapper.find('.badge--sale')
    expect(badge.exists()).toBe(true)
    expect(badge.text()).toContain('34%') // (1-990/1500)*100 ≈ 34%
  })
  
  it('emits add-to-cart event when button clicked', async () => {
    const wrapper = mount(ProductCard, {
      props: { product: mockProduct }
    })
    
    await wrapper.find('[aria-label^="เพิ่ม"]').trigger('click')
    
    expect(wrapper.emitted('add-to-cart')).toBeTruthy()
    expect(wrapper.emitted('add-to-cart')?.[0]).toEqual([mockProduct])
  })
  
  it('disables add-to-cart when out of stock', () => {
    const outOfStockProduct = {
      ...mockProduct,
      inventory: { ...mockProduct.inventory, available: 0 }
    }
    
    const wrapper = mount(ProductCard, {
      props: { product: outOfStockProduct }
    })
    
    const button = wrapper.find('[aria-label^="เพิ่ม"]')
    expect(button.attributes('disabled')).toBeDefined()
  })
  
  it('shows new badge for recently created products', () => {
    const wrapper = mount(ProductCard, {
      props: { product: mockProduct }
    })
    
    const badge = wrapper.find('.badge--new')
    expect(badge.exists()).toBe(true)
  })
  
  it('matches snapshot', () => {
    const wrapper = mount(ProductCard, {
      props: { product: mockProduct }
    })
    
    expect(wrapper.html()).toMatchSnapshot()
  })
})
```

```typescript
// tests/components/AccessibleForm.test.ts
import { describe, it, expect } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import AccessibleForm from '@/components/AccessibleForm.vue'

describe('AccessibleForm', () => {
  it('shows validation errors on empty submit', async () => {
    const wrapper = mount(AccessibleForm)
    
    await wrapper.find('form').trigger('submit')
    await flushPromises()
    
    expect(wrapper.find('[role="alert"]').exists()).toBe(true)
    expect(wrapper.text()).toContain('กรุณากรอกชื่อ-นามสกุล')
  })
  
  it('validates email format', async () => {
    const wrapper = mount(AccessibleForm)
    
    await wrapper.find('#email').setValue('invalid-email')
    await wrapper.find('#email').trigger('blur')
    
    expect(wrapper.text()).toContain('รูปแบบอีเมลไม่ถูกต้อง')
  })
  
  it('toggles password visibility', async () => {
    const wrapper = mount(AccessibleForm)
    
    const passwordInput = wrapper.find('#password')
    expect(passwordInput.attributes('type')).toBe('password')
    
    await wrapper.find('.toggle-password').trigger('click')
    expect(passwordInput.attributes('type')).toBe('text')
    
    await wrapper.find('.toggle-password').trigger('click')
    expect(passwordInput.attributes('type')).toBe('password')
  })
  
  it('submits successfully with valid data', async () => {
    const wrapper = mount(AccessibleForm)
    
    await wrapper.find('#name').setValue('สมชาย ใจดี')
    await wrapper.find('#email').setValue('somchai@example.com')
    await wrapper.find('#password').setValue('Password123')
    await wrapper.find('#terms').setValue(true)
    
    await wrapper.find('form').trigger('submit')
    await flushPromises()
    
    expect(wrapper.find('[role="alert"]').exists()).toBe(false)
  })
})
```

## Integration Tests

```typescript
// tests/integration/cart.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useCartStore } from '@/stores/cart'

describe('Cart Store Integration', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })
  
  it('adds item to cart', () => {
    const cartStore = useCartStore()
    
    cartStore.addItem(mockProduct)
    
    expect(cartStore.items).toHaveLength(1)
    expect(cartStore.items[0].product.id).toBe(mockProduct.id)
    expect(cartStore.itemCount).toBe(1)
  })
  
  it('increments quantity for existing item', () => {
    const cartStore = useCartStore()
    
    cartStore.addItem(mockProduct)
    cartStore.addItem(mockProduct)
    
    expect(cartStore.items).toHaveLength(1)
    expect(cartStore.items[0].quantity).toBe(2)
  })
  
  it('removes item from cart', () => {
    const cartStore = useCartStore()
    cartStore.addItem(mockProduct)
    
    const itemId = cartStore.items[0].id
    cartStore.removeItem(itemId)
    
    expect(cartStore.items).toHaveLength(0)
  })
  
  it('calculates total correctly with tax and shipping', () => {
    const cartStore = useCartStore()
    cartStore.addItem(mockProduct, undefined, 2)
    
    const expectedSubtotal = mockProduct.price * 2
    expect(cartStore.subtotal).toBe(expectedSubtotal)
    
    const expectedTax = expectedSubtotal * 0.07
    expect(cartStore.tax).toBeCloseTo(expectedTax)
  })
  
  it('applies free shipping for orders over 500', () => {
    const cartStore = useCartStore()
    const expensiveProduct = { ...mockProduct, price: 300 }
    
    cartStore.addItem(expensiveProduct, undefined, 2)
    expect(cartStore.shipping).toBe(0)
  })
})
```

## API Testing

```typescript
// tests/api/products.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setup } from '@nuxt/test-utils'

// Mock Prisma
vi.mock('~/server/db', () => ({
  prisma: {
    product: {
      findMany: vi.fn(),
      findUnique: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
      delete: vi.fn(),
      count: vi.fn()
    }
  }
}))

// Mock Redis
vi.mock('~/server/utils/redis', () => ({
  cache: {
    get: vi.fn().mockResolvedValue(null),
    set: vi.fn().mockResolvedValue(undefined),
    remember: vi.fn().mockImplementation((key, ttl, fn) => fn())
  }
}))

describe('Products API', () => {
  let $fetch: typeof globalThis.$fetch
  
  beforeAll(async () => {
    await setup({ server: true })
    $fetch = globalThis.$fetch
  })
  
  it('GET /api/products returns list', async () => {
    const { prisma } = await import('~/server/db')
    const mockProducts = [mockProduct]
    
    vi.mocked(prisma.product.findMany).mockResolvedValue(mockProducts)
    vi.mocked(prisma.product.count).mockResolvedValue(1)
    
    const response = await $fetch('/api/products')
    
    expect(response.products).toHaveLength(1)
    expect(response.total).toBe(1)
  })
  
  it('GET /api/products/:id returns single product', async () => {
    const { prisma } = await import('~/server/db')
    vi.mocked(prisma.product.findUnique).mockResolvedValue(mockProduct)
    
    const response = await $fetch(`/api/products/${mockProduct.id}`)
    
    expect(response.id).toBe(mockProduct.id)
    expect(response.name).toBe(mockProduct.name)
  })
  
  it('GET /api/products/:id returns 404 for non-existent product', async () => {
    const { prisma } = await import('~/server/db')
    vi.mocked(prisma.product.findUnique).mockResolvedValue(null)
    
    await expect($fetch('/api/products/nonexistent')).rejects.toThrow('404')
  })
})
```

## Visual Regression Testing

```typescript
// tests/visual/ProductCard.test.ts
import { test, expect } from '@playwright/test'

test.describe('ProductCard Visual Tests', () => {
  test('default state matches snapshot', async ({ page }) => {
    await page.goto('/storybook/?story=productcard--default')
    
    const screenshot = await page.locator('.product-card').screenshot()
    expect(screenshot).toMatchSnapshot('product-card-default.png')
  })
  
  test('hover state', async ({ page }) => {
    await page.goto('/storybook/?story=productcard--default')
    
    await page.locator('.product-card').hover()
    const screenshot = await page.locator('.product-card').screenshot()
    expect(screenshot).toMatchSnapshot('product-card-hover.png')
  })
  
  test('out of stock state', async ({ page }) => {
    await page.goto('/storybook/?story=productcard--outofstock')
    
    const screenshot = await page.locator('.product-card').screenshot()
    expect(screenshot).toMatchSnapshot('product-card-out-of-stock.png')
  })
  
  test('dark mode', async ({ page }) => {
    await page.emulateMedia({ colorScheme: 'dark' })
    await page.goto('/storybook/?story=productcard--default')
    
    const screenshot = await page.locator('.product-card').screenshot()
    expect(screenshot).toMatchSnapshot('product-card-dark.png')
  })
})
```

## Performance Testing

```typescript
// tests/performance/api.perf.test.ts
import { describe, it, expect } from 'vitest'

describe('API Performance', () => {
  const ACCEPTABLE_RESPONSE_TIME = 200 // ms
  
  it('products list responds within acceptable time', async () => {
    const start = Date.now()
    await $fetch('/api/products?limit=12')
    const duration = Date.now() - start
    
    expect(duration).toBeLessThan(ACCEPTABLE_RESPONSE_TIME)
  })
  
  it('handles concurrent requests', async () => {
    const requests = Array.from({ length: 10 }, () => $fetch('/api/products'))
    
    const start = Date.now()
    const results = await Promise.all(requests)
    const duration = Date.now() - start
    
    expect(results).toHaveLength(10)
    expect(duration).toBeLessThan(ACCEPTABLE_RESPONSE_TIME * 3) // Allow 3x for concurrent
  })
})
```

## Load Testing

```javascript
// tests/load/k6-script.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate, Trend } from 'k6/metrics'

const errorRate = new Rate('errors')
const responseTrend = new Trend('response_time')

export const options = {
  stages: [
    { duration: '30s', target: 10 },  // Ramp up to 10 users
    { duration: '60s', target: 50 },  // Ramp up to 50 users
    { duration: '30s', target: 100 }, // Peak at 100 users
    { duration: '30s', target: 0 }    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests under 500ms
    errors: ['rate<0.1']              // Error rate under 10%
  }
}

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000'

export default function() {
  // Test homepage
  const homeResponse = http.get(`${BASE_URL}/`)
  
  check(homeResponse, {
    'home status 200': (r) => r.status === 200,
    'home response time < 500ms': (r) => r.timings.duration < 500
  })
  
  errorRate.add(homeResponse.status !== 200)
  responseTrend.add(homeResponse.timings.duration)
  
  sleep(1)
  
  // Test products API
  const productsResponse = http.get(`${BASE_URL}/api/products`)
  
  check(productsResponse, {
    'products status 200': (r) => r.status === 200,
    'products has data': (r) => JSON.parse(r.body).products?.length > 0
  })
  
  errorRate.add(productsResponse.status !== 200)
  
  sleep(Math.random() * 3)
}
```

## ตัวอย่าง: Comprehensive Test Suite

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath } from 'url'

export default defineConfig({
  plugins: [vue()],
  
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./tests/setup.ts'],
    
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      include: ['src/**', 'server/**'],
      exclude: ['**/*.test.ts', '**/node_modules/**'],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80
      }
    },
    
    reporters: ['default', 'html'],
    
    // Timeout for async tests
    testTimeout: 10000,
    hookTimeout: 10000
  },
  
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
      '~': fileURLToPath(new URL('./', import.meta.url))
    }
  }
})
```

```typescript
// tests/setup.ts
import { config } from '@vue/test-utils'
import { vi } from 'vitest'
import { createTestingPinia } from '@pinia/testing'

// Global mocks
vi.mock('vue-router', () => ({
  useRoute: vi.fn(() => ({ params: {}, query: {}, path: '/' })),
  useRouter: vi.fn(() => ({ push: vi.fn(), replace: vi.fn() })),
  RouterLink: { template: '<a><slot /></a>' }
}))

// Global test config
config.global.plugins = [createTestingPinia()]

// Mock Nuxt composables
vi.mock('#app', () => ({
  useNuxtApp: vi.fn(() => ({
    $fetch: vi.fn()
  })),
  useRuntimeConfig: vi.fn(() => ({
    public: {
      apiUrl: 'http://localhost:3000'
    }
  })),
  navigateTo: vi.fn()
}))

// Setup MSW for API mocking
import { setupServer } from 'msw/node'
import { handlers } from './mocks/handlers'

const server = setupServer(...handlers)

beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

```typescript
// tests/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/products', () => {
    return HttpResponse.json({
      products: [mockProduct],
      total: 1,
      page: 1,
      totalPages: 1
    })
  }),
  
  http.get('/api/products/:id', ({ params }) => {
    if (params.id === 'notfound') {
      return new HttpResponse(null, { status: 404 })
    }
    return HttpResponse.json(mockProduct)
  }),
  
  http.post('/api/auth/login', async ({ request }) => {
    const { email, password } = await request.json()
    
    if (email === 'test@example.com' && password === 'password') {
      return HttpResponse.json({
        user: { id: '1', email, name: 'Test User' },
        token: 'mock-jwt-token'
      })
    }
    
    return new HttpResponse(
      JSON.stringify({ message: 'Invalid credentials' }),
      { status: 401 }
    )
  })
]
```

## E2E Tests ด้วย Playwright

```typescript
// tests/e2e/checkout.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Checkout Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('/login')
    await page.fill('[name=email]', 'test@example.com')
    await page.fill('[name=password]', 'Password123!')
    await page.click('[type=submit]')
    await expect(page).toHaveURL('/')
  })
  
  test('complete checkout flow', async ({ page }) => {
    // Add product to cart
    await page.goto('/products/vue-js-course')
    await page.click('[data-testid=add-to-cart]')
    
    // Verify cart has item
    await expect(page.locator('[data-testid=cart-count]')).toContainText('1')
    
    // Go to checkout
    await page.click('[data-testid=checkout-button]')
    await expect(page).toHaveURL('/checkout')
    
    // Fill shipping address
    await page.fill('[name=firstName]', 'สมชาย')
    await page.fill('[name=lastName]', 'ใจดี')
    await page.fill('[name=addressLine1]', '123 ถนนสุขุมวิท')
    await page.fill('[name=city]', 'กรุงเทพมหานคร')
    await page.selectOption('[name=province]', 'Bangkok')
    await page.fill('[name=postalCode]', '10110')
    await page.fill('[name=phone]', '0812345678')
    
    await page.click('[data-testid=continue-to-payment]')
    
    // Fill payment (Stripe test card)
    const stripeFrame = page.frameLocator('[name^=__privateStripeFrame]')
    await stripeFrame.locator('[name=cardnumber]').fill('4242424242424242')
    await stripeFrame.locator('[name=exp-date]').fill('12/28')
    await stripeFrame.locator('[name=cvc]').fill('123')
    
    await page.click('[data-testid=pay-button]')
    
    // Verify order confirmation
    await expect(page).toHaveURL(/\/orders\/\w+/)
    await expect(page.locator('h1')).toContainText('คำสั่งซื้อสำเร็จ')
  })
  
  test('shows validation errors', async ({ page }) => {
    await page.goto('/checkout')
    await page.click('[data-testid=continue-to-payment]')
    
    await expect(page.locator('[role=alert]')).toBeVisible()
    await expect(page.locator('[id=firstName-error]')).toBeVisible()
  })
})
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  
  // Maximum time one test can run
  timeout: 30 * 1000,
  
  expect: {
    timeout: 5000
  },
  
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  
  reporter: [
    ['html', { open: 'never' }],
    ['junit', { outputFile: 'test-results/junit.xml' }]
  ],
  
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'on-first-retry'
  },
  
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] }
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] }
    }
  ],
  
  webServer: {
    command: 'npm run preview',
    port: 3000,
    reuseExistingServer: !process.env.CI
  }
})
```

## Continuous Testing

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info

  e2e-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run build
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

## สรุป

Advanced Testing Strategies ต้องมี:
1. Testing pyramid: Unit > Integration > E2E
2. Component tests ด้วย Vue Test Utils
3. API tests ด้วย MSW mocking
4. Visual regression ด้วย Playwright screenshots
5. E2E tests ด้วย Playwright สำหรับ critical flows
6. Load testing ด้วย k6
7. Coverage thresholds ขั้นต่ำ 80%
8. CI/CD pipeline รัน tests อัตโนมัติ
