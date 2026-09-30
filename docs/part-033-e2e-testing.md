# Part 33: Testing - E2E Tests

## E2E Testing คืออะไร?

End-to-End (E2E) Testing คือการทดสอบแอปพลิเคชันจากมุมมองของผู้ใช้จริง โดยจำลองการใช้งาน browser ทั้งหมด ตั้งแต่คลิก input form ไปจนถึงดูผลลัพธ์บน screen E2E test ช่วยยืนยันว่า user flow ทั้งหมดทำงานได้ถูกต้อง

---

## 1. Playwright Setup กับ Nuxt

Playwright เป็น E2E testing framework จาก Microsoft ที่รองรับ Chrome, Firefox, และ Safari

```bash
# ติดตั้ง Playwright
npm install -D @playwright/test

# ดาวน์โหลด browsers
npx playwright install

# หรือสำหรับ Nuxt ใช้ @nuxt/test-utils
npm install -D @nuxt/test-utils playwright-core
```

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['list']
  ],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] }
    }
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI
  }
})
```

```json
// package.json
{
  "scripts": {
    "e2e": "playwright test",
    "e2e:ui": "playwright test --ui",
    "e2e:debug": "playwright test --debug",
    "e2e:codegen": "playwright codegen http://localhost:3000"
  }
}
```

---

## 2. Page Object Model

Page Object Model (POM) เป็น pattern ที่ช่วยจัดการ test code ให้ reuse ได้

```ts
// e2e/pages/LoginPage.ts
import type { Page, Locator } from '@playwright/test'

export class LoginPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly passwordInput: Locator
  readonly submitButton: Locator
  readonly errorMessage: Locator
  readonly rememberMeCheckbox: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByTestId('email-input')
    this.passwordInput = page.getByTestId('password-input')
    this.submitButton = page.getByRole('button', { name: 'เข้าสู่ระบบ' })
    this.errorMessage = page.getByTestId('error-message')
    this.rememberMeCheckbox = page.getByLabel('จำฉันไว้')
  }

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.submitButton.click()
  }

  async loginAndWait(email: string, password: string) {
    await this.login(email, password)
    await this.page.waitForURL('/dashboard')
  }

  async getErrorMessage() {
    await this.errorMessage.waitFor({ state: 'visible' })
    return this.errorMessage.textContent()
  }
}
```

```ts
// e2e/pages/DashboardPage.ts
import type { Page, Locator } from '@playwright/test'

export class DashboardPage {
  readonly page: Page
  readonly welcomeMessage: Locator
  readonly userMenu: Locator
  readonly logoutButton: Locator
  readonly navigationLinks: Locator

  constructor(page: Page) {
    this.page = page
    this.welcomeMessage = page.getByTestId('welcome-message')
    this.userMenu = page.getByTestId('user-menu')
    this.logoutButton = page.getByRole('button', { name: 'ออกจากระบบ' })
    this.navigationLinks = page.locator('nav a')
  }

  async goto() {
    await this.page.goto('/dashboard')
  }

  async getWelcomeText() {
    return this.welcomeMessage.textContent()
  }

  async logout() {
    await this.userMenu.click()
    await this.logoutButton.click()
    await this.page.waitForURL('/login')
  }

  async navigateTo(linkText: string) {
    await this.page.getByRole('link', { name: linkText }).click()
  }
}
```

---

## 3. Test Authentication

```ts
// e2e/auth.setup.ts - Setup authentication state
import { test as setup, expect } from '@playwright/test'

const authFile = '.auth/user.json'

setup('authenticate', async ({ page }) => {
  await page.goto('/login')

  await page.getByTestId('email-input').fill('test@example.com')
  await page.getByTestId('password-input').fill('password123')
  await page.getByRole('button', { name: 'เข้าสู่ระบบ' }).click()

  await page.waitForURL('/dashboard')
  await expect(page.getByTestId('welcome-message')).toBeVisible()

  // บันทึก authentication state
  await page.context().storageState({ path: authFile })
})
```

```ts
// playwright.config.ts - ใช้ auth state
import { defineConfig } from '@playwright/test'

export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/
    },
    {
      name: 'authenticated',
      use: {
        storageState: '.auth/user.json'
      },
      dependencies: ['setup']
    }
  ]
})
```

```ts
// e2e/login.test.ts - Test Login Flow
import { test, expect } from '@playwright/test'
import { LoginPage } from './pages/LoginPage'
import { DashboardPage } from './pages/DashboardPage'

test.describe('Login Flow', () => {
  test('successful login redirects to dashboard', async ({ page }) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()

    await loginPage.loginAndWait('admin@example.com', 'password123')

    const dashboardPage = new DashboardPage(page)
    const welcomeText = await dashboardPage.getWelcomeText()
    expect(welcomeText).toContain('ยินดีต้อนรับ')
  })

  test('shows error for invalid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()

    await loginPage.login('wrong@example.com', 'wrongpassword')

    const errorText = await loginPage.getErrorMessage()
    expect(errorText).toBe('อีเมลหรือรหัสผ่านไม่ถูกต้อง')
  })

  test('shows validation errors for empty form', async ({ page }) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()

    await loginPage.submitButton.click()

    await expect(page.getByText('กรุณากรอกอีเมล')).toBeVisible()
    await expect(page.getByText('กรุณากรอกรหัสผ่าน')).toBeVisible()
  })

  test('redirects to original URL after login', async ({ page }) => {
    // ไปที่หน้าที่ต้องการ authentication
    await page.goto('/profile')

    // ควร redirect ไปหน้า login
    await page.waitForURL(/\/login/)

    // Login
    const loginPage = new LoginPage(page)
    await loginPage.loginAndWait('admin@example.com', 'password123')

    // ควร redirect กลับไปหน้าที่ต้องการ
    expect(page.url()).toContain('/profile')
  })

  test('logout clears session and redirects to login', async ({ page }) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()
    await loginPage.loginAndWait('admin@example.com', 'password123')

    const dashboardPage = new DashboardPage(page)
    await dashboardPage.logout()

    await expect(page).toHaveURL('/login')

    // ตรวจว่า session ถูกล้างแล้ว
    await page.goto('/dashboard')
    await page.waitForURL('/login')
  })
})
```

---

## 4. API Mocking

```ts
// e2e/shopping-cart.test.ts - Shopping Cart E2E Test
import { test, expect, type Page } from '@playwright/test'

// Mock API responses ด้วย route interception
async function mockProductsAPI(page: Page) {
  await page.route('/api/products', async (route) => {
    const products = [
      { id: 1, name: 'สินค้า A', price: 100, stock: 10, imageUrl: null },
      { id: 2, name: 'สินค้า B', price: 200, stock: 5, imageUrl: null },
      { id: 3, name: 'สินค้า C', price: 300, stock: 0, imageUrl: null }
    ]
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify(products)
    })
  })
}

async function mockCartAPI(page: Page) {
  await page.route('/api/cart/**', async (route) => {
    const method = route.request().method()
    if (method === 'POST') {
      await route.fulfill({
        status: 201,
        contentType: 'application/json',
        body: JSON.stringify({ success: true })
      })
    } else if (method === 'DELETE') {
      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({ success: true })
      })
    }
  })
}

test.describe('Shopping Cart Flow', () => {
  test.beforeEach(async ({ page }) => {
    await mockProductsAPI(page)
    await mockCartAPI(page)
    await page.goto('/products')
  })

  test('adds product to cart', async ({ page }) => {
    // คลิกเพิ่มสินค้าชิ้นแรก
    await page.getByTestId('add-to-cart-1').click()

    // ตรวจว่า badge บน cart icon เพิ่มขึ้น
    await expect(page.getByTestId('cart-count')).toHaveText('1')

    // ไปที่หน้า cart
    await page.getByTestId('cart-icon').click()
    await page.waitForURL('/cart')

    // ตรวจว่าสินค้าอยู่ใน cart
    await expect(page.getByTestId('cart-item-1')).toBeVisible()
    await expect(page.getByText('สินค้า A')).toBeVisible()
    await expect(page.getByText('฿100')).toBeVisible()
  })

  test('updates cart quantity', async ({ page }) => {
    await page.getByTestId('add-to-cart-1').click()
    await page.getByTestId('cart-icon').click()
    await page.waitForURL('/cart')

    // เพิ่ม quantity
    await page.getByTestId('quantity-increase-1').click()
    await page.getByTestId('quantity-increase-1').click()

    await expect(page.getByTestId('quantity-1')).toHaveText('3')
    await expect(page.getByTestId('item-total-1')).toHaveText('฿300')
  })

  test('removes item from cart', async ({ page }) => {
    await page.getByTestId('add-to-cart-1').click()
    await page.getByTestId('add-to-cart-2').click()
    await page.getByTestId('cart-icon').click()
    await page.waitForURL('/cart')

    await expect(page.getByTestId('cart-items')).toHaveCount(2)

    await page.getByTestId('remove-item-1').click()
    await page.getByRole('button', { name: 'ยืนยัน' }).click()

    await expect(page.getByTestId('cart-items')).toHaveCount(1)
  })

  test('shows out-of-stock for unavailable products', async ({ page }) => {
    const outOfStockBtn = page.getByTestId('add-to-cart-3')
    await expect(outOfStockBtn).toBeDisabled()
    await expect(page.getByTestId('stock-status-3')).toHaveText('สินค้าหมด')
  })

  test('completes checkout flow', async ({ page }) => {
    // Setup checkout mock
    await page.route('/api/orders', async (route) => {
      await route.fulfill({
        status: 201,
        body: JSON.stringify({ orderId: 'ORD-001', status: 'pending' })
      })
    })

    await page.getByTestId('add-to-cart-1').click()
    await page.getByTestId('cart-icon').click()
    await page.waitForURL('/cart')

    await page.getByRole('button', { name: 'สั่งซื้อ' }).click()
    await page.waitForURL('/checkout')

    // กรอกข้อมูลการจัดส่ง
    await page.getByLabel('ชื่อ-นามสกุล').fill('สมชาย ใจดี')
    await page.getByLabel('ที่อยู่').fill('123 ถ.สุขุมวิท')
    await page.getByLabel('โทรศัพท์').fill('0812345678')

    await page.getByRole('button', { name: 'ยืนยันคำสั่งซื้อ' }).click()
    await page.waitForURL('/orders/ORD-001')

    await expect(page.getByText('คำสั่งซื้อสำเร็จ')).toBeVisible()
    await expect(page.getByText('ORD-001')).toBeVisible()
  })
})
```

---

## 5. Visual Testing

```ts
// e2e/visual.test.ts - Visual Regression Testing
import { test, expect } from '@playwright/test'

test.describe('Visual Regression', () => {
  test('homepage matches snapshot', async ({ page }) => {
    await page.goto('/')
    await expect(page).toHaveScreenshot('homepage.png', {
      fullPage: true,
      animations: 'disabled'
    })
  })

  test('product card visual', async ({ page }) => {
    await page.goto('/products')
    await page.waitForLoadState('networkidle')

    const productCard = page.getByTestId('product-card').first()
    await expect(productCard).toHaveScreenshot('product-card.png')
  })

  test('dark mode homepage', async ({ page }) => {
    await page.emulateMedia({ colorScheme: 'dark' })
    await page.goto('/')
    await expect(page).toHaveScreenshot('homepage-dark.png')
  })

  test('mobile layout', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 812 })
    await page.goto('/')
    await expect(page).toHaveScreenshot('homepage-mobile.png')
  })
})
```

---

## 6. CI/CD Integration

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Build application
        run: npm run build

      - name: Run E2E tests
        run: npm run e2e
        env:
          BASE_URL: http://localhost:3000
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Upload screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: test-screenshots
          path: test-results/
          retention-days: 7
```

```ts
// e2e/helpers/auth.ts - Helper functions สำหรับ authentication
import type { Page } from '@playwright/test'

const TEST_CREDENTIALS = {
  admin: {
    email: process.env.TEST_ADMIN_EMAIL || 'admin@test.com',
    password: process.env.TEST_ADMIN_PASSWORD || 'admin123'
  },
  user: {
    email: process.env.TEST_USER_EMAIL || 'user@test.com',
    password: process.env.TEST_USER_PASSWORD || 'user123'
  }
}

export async function loginAs(page: Page, role: 'admin' | 'user') {
  const creds = TEST_CREDENTIALS[role]
  await page.goto('/login')
  await page.getByTestId('email-input').fill(creds.email)
  await page.getByTestId('password-input').fill(creds.password)
  await page.getByRole('button', { name: 'เข้าสู่ระบบ' }).click()
  await page.waitForURL('/dashboard')
}

export async function clearAuthState(page: Page) {
  await page.evaluate(() => {
    localStorage.clear()
    sessionStorage.clear()
  })
  // ล้าง cookies
  await page.context().clearCookies()
}
```

---

## 7. Accessibility Testing

```ts
// e2e/accessibility.test.ts
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

test.describe('Accessibility', () => {
  test('homepage passes accessibility checks', async ({ page }) => {
    await page.goto('/')
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa'])
      .analyze()

    expect(results.violations).toEqual([])
  })

  test('login page is keyboard navigable', async ({ page }) => {
    await page.goto('/login')

    // Tab through form elements
    await page.keyboard.press('Tab')
    await expect(page.getByTestId('email-input')).toBeFocused()

    await page.keyboard.press('Tab')
    await expect(page.getByTestId('password-input')).toBeFocused()

    await page.keyboard.press('Tab')
    await expect(page.getByRole('button', { name: 'เข้าสู่ระบบ' })).toBeFocused()

    // Submit with Enter key
    await page.keyboard.press('Enter')
  })
})
```

---

## 8. Performance Testing

```ts
// e2e/performance.test.ts - วัด performance ด้วย Playwright
import { test, expect } from '@playwright/test'

test.describe('Performance', () => {
  test('homepage loads within 3 seconds', async ({ page }) => {
    const startTime = Date.now()
    await page.goto('/', { waitUntil: 'networkidle' })
    const loadTime = Date.now() - startTime

    expect(loadTime).toBeLessThan(3000)
  })

  test('measures Core Web Vitals', async ({ page }) => {
    await page.goto('/')

    const metrics = await page.evaluate(() => {
      return new Promise<{lcp: number; cls: number; fid: number}>((resolve) => {
        const vitals = { lcp: 0, cls: 0, fid: 0 }

        new PerformanceObserver((list) => {
          const entries = list.getEntries()
          vitals.lcp = entries[entries.length - 1].startTime
          resolve(vitals)
        }).observe({ entryTypes: ['largest-contentful-paint'] })

        // Fallback if no LCP
        setTimeout(() => resolve(vitals), 5000)
      })
    })

    // LCP ควรน้อยกว่า 2.5 วินาที
    expect(metrics.lcp).toBeLessThan(2500)
  })

  test('api response time is acceptable', async ({ page }) => {
    let responseTime = 0

    page.on('response', (response) => {
      if (response.url().includes('/api/products')) {
        responseTime = Date.now()
      }
    })

    const requestStart = Date.now()
    page.on('request', (request) => {
      if (request.url().includes('/api/products')) {
        requestStart
      }
    })

    await page.goto('/products')
    await page.waitForSelector('.product-card')

    // API ควรตอบภายใน 1 วินาที
    expect(responseTime - requestStart).toBeLessThan(1000)
  })
})
```

---

## 9. Test Data Management

```ts
// e2e/fixtures/testData.ts - จัดการ test data
export const testUsers = {
  admin: {
    email: 'admin@test.com',
    password: 'Admin123!',
    name: 'ผู้ดูแลระบบ',
    role: 'admin' as const
  },
  regularUser: {
    email: 'user@test.com',
    password: 'User123!',
    name: 'ผู้ใช้ทั่วไป',
    role: 'user' as const
  }
}

export const testProducts = [
  { name: 'สินค้าทดสอบ A', price: 100, stock: 10 },
  { name: 'สินค้าทดสอบ B', price: 200, stock: 0 }
]
```

```ts
// e2e/fixtures/index.ts - Custom Playwright fixtures
import { test as base } from '@playwright/test'
import { LoginPage } from '../pages/LoginPage'
import { testUsers } from './testData'

interface Fixtures {
  loginPage: LoginPage
  authenticatedPage: { page: ReturnType<typeof base>['page'] }
}

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()
    await use(loginPage)
  },

  authenticatedPage: async ({ page }, use) => {
    // Quick login via API
    const response = await page.request.post('/api/auth/login', {
      data: testUsers.admin
    })
    const { token } = await response.json()

    await page.context().addCookies([{
      name: 'auth-token',
      value: token,
      domain: 'localhost',
      path: '/'
    }])

    await use({ page })
  }
})

export { expect } from '@playwright/test'
```

---

## สรุป

E2E Testing ด้วย Playwright ทำให้มั่นใจว่า user flows ทั้งหมดทำงานได้:

1. **Playwright Setup** - ตั้งค่า config, browsers, และ webServer
2. **Page Object Model** - จัดระเบียบ test code ให้ reuse ได้
3. **Test Authentication** - จัดการ login state อย่างมีประสิทธิภาพ
4. **API Mocking** - ใช้ `page.route()` เพื่อ mock API
5. **Visual Testing** - ตรวจสอบ UI ด้วย screenshot comparison
6. **CI/CD Integration** - รัน E2E tests ใน GitHub Actions อัตโนมัติ
7. **Accessibility Testing** - ตรวจสอบ WCAG compliance ด้วย axe-core
8. **Performance Testing** - วัด Core Web Vitals และ load times
9. **Test Data Management** - จัดการ fixtures และ test users อย่างเป็นระบบ
