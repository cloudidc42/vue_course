# Part 36: Internationalization (i18n)

## i18n คืออะไร?

Internationalization (i18n) คือกระบวนการออกแบบแอปพลิเคชันให้รองรับหลายภาษาและ locale โดยไม่ต้องเปลี่ยน core code i18n ครอบคลุม:
- **การแปลข้อความ** (Translation)
- **การจัดรูปแบบตัวเลข/วันที่/สกุลเงิน** (Formatting)
- **การรองรับ RTL** (Right-to-Left languages)
- **Locale-based routing**

---

## 1. @nuxtjs/i18n Setup

```bash
# ติดตั้ง module
npm install -D @nuxtjs/i18n
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/i18n'],

  i18n: {
    locales: [
      {
        code: 'th',
        name: 'ภาษาไทย',
        iso: 'th-TH',
        file: 'th.json',
        dir: 'ltr'
      },
      {
        code: 'en',
        name: 'English',
        iso: 'en-US',
        file: 'en.json',
        dir: 'ltr'
      },
      {
        code: 'ja',
        name: '日本語',
        iso: 'ja-JP',
        file: 'ja.json',
        dir: 'ltr'
      },
      {
        code: 'ar',
        name: 'العربية',
        iso: 'ar-SA',
        file: 'ar.json',
        dir: 'rtl'  // Right-to-Left
      }
    ],
    defaultLocale: 'th',
    lazy: true,
    langDir: 'locales/',
    strategy: 'prefix_except_default',
    detectBrowserLanguage: {
      useCookie: true,
      cookieKey: 'i18n_redirected',
      redirectOn: 'root'
    }
  }
})
```

---

## 2. Translation Files Structure

```json
// locales/th.json
{
  "common": {
    "loading": "กำลังโหลด...",
    "error": "เกิดข้อผิดพลาด",
    "save": "บันทึก",
    "cancel": "ยกเลิก",
    "delete": "ลบ",
    "edit": "แก้ไข",
    "search": "ค้นหา",
    "back": "กลับ",
    "close": "ปิด",
    "yes": "ใช่",
    "no": "ไม่"
  },
  "nav": {
    "home": "หน้าหลัก",
    "products": "สินค้า",
    "about": "เกี่ยวกับเรา",
    "contact": "ติดต่อเรา",
    "login": "เข้าสู่ระบบ",
    "logout": "ออกจากระบบ",
    "profile": "โปรไฟล์"
  },
  "auth": {
    "login": {
      "title": "เข้าสู่ระบบ",
      "email": "อีเมล",
      "password": "รหัสผ่าน",
      "submit": "เข้าสู่ระบบ",
      "forgot": "ลืมรหัสผ่าน?",
      "register": "ยังไม่มีบัญชี? สมัครสมาชิก",
      "error": "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
    },
    "register": {
      "title": "สมัครสมาชิก",
      "name": "ชื่อ-นามสกุล",
      "email": "อีเมล",
      "password": "รหัสผ่าน",
      "confirmPassword": "ยืนยันรหัสผ่าน",
      "submit": "สมัครสมาชิก"
    }
  },
  "products": {
    "title": "สินค้าทั้งหมด",
    "addToCart": "เพิ่มลงตะกร้า",
    "outOfStock": "สินค้าหมด",
    "price": "ราคา",
    "count": "มีสินค้า {count} รายการ",
    "noProducts": "ไม่พบสินค้า"
  },
  "messages": {
    "welcome": "ยินดีต้อนรับ, {name}!",
    "itemsInCart": "คุณมีสินค้า {count} ชิ้นในตะกร้า | คุณมีสินค้า {count} ชิ้นในตะกร้า",
    "lastSeen": "เข้าระบบล่าสุดเมื่อ {date}"
  }
}
```

```json
// locales/en.json
{
  "common": {
    "loading": "Loading...",
    "error": "An error occurred",
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "edit": "Edit",
    "search": "Search",
    "back": "Back",
    "close": "Close",
    "yes": "Yes",
    "no": "No"
  },
  "nav": {
    "home": "Home",
    "products": "Products",
    "about": "About Us",
    "contact": "Contact",
    "login": "Login",
    "logout": "Logout",
    "profile": "Profile"
  },
  "auth": {
    "login": {
      "title": "Login",
      "email": "Email",
      "password": "Password",
      "submit": "Login",
      "forgot": "Forgot password?",
      "register": "Don't have an account? Register",
      "error": "Invalid email or password"
    }
  },
  "products": {
    "title": "All Products",
    "addToCart": "Add to Cart",
    "outOfStock": "Out of Stock",
    "price": "Price",
    "count": "Found {count} products",
    "noProducts": "No products found"
  },
  "messages": {
    "welcome": "Welcome, {name}!",
    "itemsInCart": "You have {count} item in cart | You have {count} items in cart",
    "lastSeen": "Last seen on {date}"
  }
}
```

```json
// locales/ja.json
{
  "common": {
    "loading": "読み込み中...",
    "save": "保存",
    "cancel": "キャンセル"
  },
  "nav": {
    "home": "ホーム",
    "products": "商品",
    "login": "ログイン"
  },
  "products": {
    "title": "すべての商品",
    "addToCart": "カートに追加",
    "outOfStock": "在庫切れ"
  },
  "messages": {
    "welcome": "ようこそ、{name}さん！"
  }
}
```

---

## 3. useI18n() Composable

```vue
<!-- components/Navigation.vue -->
<script setup lang="ts">
const { t, locale, locales, setLocale } = useI18n()

// เปลี่ยนภาษา
async function switchLanguage(code: string) {
  await setLocale(code)
}
</script>

<template>
  <nav>
    <NuxtLinkLocale to="/">{{ t('nav.home') }}</NuxtLinkLocale>
    <NuxtLinkLocale to="/products">{{ t('nav.products') }}</NuxtLinkLocale>
    
    <!-- Language Switcher -->
    <div class="language-switcher">
      <button
        v-for="loc in locales"
        :key="loc.code"
        @click="switchLanguage(loc.code)"
        :class="{ active: locale === loc.code }"
      >
        {{ loc.name }}
      </button>
    </div>
  </nav>
</template>
```

```vue
<!-- การใช้ $t ใน template และ t() ใน script -->
<script setup lang="ts">
const { t, tc, te, n, d } = useI18n()

const userName = ref('สมชาย')
const cartCount = ref(3)
const lastDate = new Date('2024-01-15')

// Interpolation
const welcome = computed(() => t('messages.welcome', { name: userName.value }))

// Pluralization
const cartMessage = computed(() => tc('messages.itemsInCart', cartCount.value))

// Check if translation exists
const hasKey = te('some.translation.key')

// Number formatting
const formattedNumber = computed(() => n(1234567.89, 'currency'))

// Date formatting
const formattedDate = computed(() => d(lastDate, 'long'))
</script>

<template>
  <div>
    <p>{{ welcome }}</p>
    <p>{{ cartMessage }}</p>
    
    <!-- ใช้ $t ใน template โดยตรง -->
    <p>{{ $t('products.count', { count: 42 }) }}</p>
    
    <!-- Pluralization -->
    <p>{{ $tc('messages.itemsInCart', 1) }}</p>
    <p>{{ $tc('messages.itemsInCart', 5) }}</p>
  </div>
</template>
```

---

## 4. Number/Date/Currency Formatting

```ts
// nuxt.config.ts - ตั้งค่า number/date formats
export default defineNuxtConfig({
  i18n: {
    // ...
    numberFormats: {
      'th-TH': {
        currency: {
          style: 'currency',
          currency: 'THB',
          notation: 'standard'
        },
        decimal: {
          style: 'decimal',
          minimumFractionDigits: 2,
          maximumFractionDigits: 2
        },
        percent: {
          style: 'percent',
          useGrouping: false
        }
      },
      'en-US': {
        currency: {
          style: 'currency',
          currency: 'USD'
        },
        decimal: {
          style: 'decimal',
          minimumFractionDigits: 2
        }
      },
      'ja-JP': {
        currency: {
          style: 'currency',
          currency: 'JPY',
          notation: 'standard'
        }
      }
    },
    datetimeFormats: {
      'th-TH': {
        short: {
          year: 'numeric',
          month: 'short',
          day: 'numeric'
        },
        long: {
          year: 'numeric',
          month: 'long',
          day: 'numeric',
          weekday: 'long'
        },
        datetime: {
          year: 'numeric',
          month: 'short',
          day: 'numeric',
          hour: 'numeric',
          minute: 'numeric'
        }
      },
      'en-US': {
        short: {
          year: 'numeric',
          month: 'short',
          day: 'numeric'
        },
        long: {
          year: 'numeric',
          month: 'long',
          day: 'numeric',
          weekday: 'long'
        }
      }
    }
  }
})
```

```vue
<!-- การใช้ Number/Date formatting -->
<script setup lang="ts">
const { n, d, locale } = useI18n()

const price = 1299.99
const date = new Date()
const percentage = 0.15

// ราคาตามภาษา
const formattedPrice = computed(() => n(price, 'currency'))
// ไทย: ฿1,299.99 | อังกฤษ: $1,299.99 | ญี่ปุ่น: ¥1,300

// วันที่ตามภาษา
const formattedDate = computed(() => d(date, 'long'))
// ไทย: วันจันทร์ที่ 15 มกราคม 2024 | อังกฤษ: Monday, January 15, 2024

const formattedPercent = computed(() => n(percentage, 'percent'))
</script>

<template>
  <div>
    <p>ราคา: {{ formattedPrice }}</p>
    <p>วันที่: {{ formattedDate }}</p>
    <p>ส่วนลด: {{ formattedPercent }}</p>
    
    <!-- ใช้ $n, $d โดยตรงใน template -->
    <p>{{ $n(99.99, 'currency') }}</p>
    <p>{{ $d(new Date(), 'short') }}</p>
  </div>
</template>
```

---

## 5. RTL Support

```vue
<!-- layouts/default.vue - RTL Support -->
<script setup lang="ts">
const { locale } = useI18n()

// RTL locales
const rtlLocales = ['ar', 'he', 'fa', 'ur']

const isRTL = computed(() => rtlLocales.includes(locale.value))
const direction = computed(() => isRTL.value ? 'rtl' : 'ltr')
</script>

<template>
  <html :dir="direction" :lang="locale">
    <body :class="{ 'rtl': isRTL }">
      <slot />
    </body>
  </html>
</template>
```

```css
/* assets/css/rtl.css - RTL styles */
[dir="rtl"] {
  /* Text alignment */
  text-align: right;

  /* Margins and paddings */
  .ml-auto { margin-left: unset; margin-right: auto; }
  .mr-auto { margin-right: unset; margin-left: auto; }

  /* Flex direction */
  .flex-row { flex-direction: row-reverse; }

  /* Icons และ arrows */
  .arrow-right { transform: rotate(180deg); }
  .chevron-right { transform: rotate(180deg); }

  /* Navigation */
  nav ul {
    flex-direction: row-reverse;
  }
}
```

---

## 6. Locale-based Routing

```ts
// nuxt.config.ts - Routing strategy
export default defineNuxtConfig({
  i18n: {
    strategy: 'prefix_except_default',
    // prefix: ทุก locale มี prefix /th/, /en/, /ja/
    // prefix_except_default: default locale ไม่มี prefix
    // no_prefix: ไม่มี prefix เลย (ใช้ cookie/header)

    defaultLocale: 'th',

    // Custom routes
    pages: {
      about: {
        th: '/เกี่ยวกับ',
        en: '/about',
        ja: '/about-us'
      },
      'products/[id]': {
        th: '/สินค้า/[id]',
        en: '/products/[id]',
        ja: '/商品/[id]'
      }
    }
  }
})
```

```vue
<!-- components/LanguageSwitcher.vue -->
<script setup lang="ts">
const { locale, locales, setLocale } = useI18n()
const switchLocalePath = useSwitchLocalePath()
const localePath = useLocalePath()

interface LocaleInfo {
  code: string
  name: string
  iso: string
  flag?: string
}
</script>

<template>
  <div class="language-switcher">
    <NuxtLink
      v-for="loc in locales as LocaleInfo[]"
      :key="loc.code"
      :to="switchLocalePath(loc.code)"
      :class="{ 'active': locale === loc.code }"
      class="locale-link"
    >
      {{ loc.name }}
    </NuxtLink>
  </div>
</template>
```

---

## 7. Lazy Loading Translations

```ts
// nuxt.config.ts - Lazy loading
export default defineNuxtConfig({
  i18n: {
    lazy: true,
    langDir: 'locales/',
    locales: [
      { code: 'th', file: 'th.json' },
      { code: 'en', file: 'en.json' },
      { code: 'ja', file: 'ja.json' }
    ]
  }
})
```

```ts
// locales/ - แยกไฟล์ตาม feature
// locales/th/
//   index.json      - translations หลัก
//   auth.json       - authentication translations
//   products.json   - product translations

// locales/th/index.ts - merge multiple files
export default async function () {
  const [common, auth, products] = await Promise.all([
    import('./th/common.json'),
    import('./th/auth.json'),
    import('./th/products.json')
  ])
  return {
    ...common.default,
    auth: auth.default,
    products: products.default
  }
}
```

---

## 8. ตัวอย่าง: Multi-language App (ไทย/อังกฤษ/ญี่ปุ่น)

```vue
<!-- pages/index.vue - Homepage with full i18n -->
<script setup lang="ts">
const { t, locale, locales, setLocale, n, d } = useI18n()
const localePath = useLocalePath()

// SEO
useSeoMeta({
  title: t('home.seo.title'),
  description: t('home.seo.description')
})

const stats = ref({
  users: 50000,
  products: 1234,
  revenue: 9999999.99
})

const currentDate = new Date()
</script>

<template>
  <div class="homepage">
    <!-- Hero Section -->
    <section class="hero">
      <h1>{{ t('home.hero.title') }}</h1>
      <p>{{ t('home.hero.subtitle') }}</p>
      <NuxtLink :to="localePath('/products')" class="cta-btn">
        {{ t('home.hero.cta') }}
      </NuxtLink>
    </section>

    <!-- Stats Section -->
    <section class="stats">
      <div class="stat-card">
        <div class="stat-number">{{ n(stats.users, 'decimal') }}</div>
        <div class="stat-label">{{ t('home.stats.users') }}</div>
      </div>
      <div class="stat-card">
        <div class="stat-number">{{ n(stats.products, 'decimal') }}</div>
        <div class="stat-label">{{ t('home.stats.products') }}</div>
      </div>
      <div class="stat-card">
        <div class="stat-number">{{ n(stats.revenue, 'currency') }}</div>
        <div class="stat-label">{{ t('home.stats.revenue') }}</div>
      </div>
    </section>

    <!-- Language Switcher -->
    <div class="lang-switcher">
      <p>{{ t('home.selectLanguage') }}</p>
      <button
        v-for="loc in locales"
        :key="(loc as any).code"
        @click="setLocale((loc as any).code)"
        :class="{ active: locale === (loc as any).code }"
      >
        {{ (loc as any).name }}
      </button>
    </div>

    <!-- Last Updated -->
    <footer>
      <p>{{ t('home.updatedAt', { date: d(currentDate, 'datetime') }) }}</p>
    </footer>
  </div>
</template>
```

```vue
<!-- components/ProductCard.vue - i18n in component -->
<script setup lang="ts">
import type { Product } from '~/types'

const props = defineProps<{ product: Product }>()
const { t, n } = useI18n()
const localePath = useLocalePath()
</script>

<template>
  <div class="product-card">
    <img :src="product.imageUrl ?? '/placeholder.jpg'" :alt="product.name" />
    <h3>{{ product.name }}</h3>
    <p class="price">{{ n(product.price, 'currency') }}</p>
    <span v-if="product.stock === 0" class="badge out-of-stock">
      {{ t('products.outOfStock') }}
    </span>
    <div class="actions">
      <NuxtLink :to="localePath(`/products/${product.id}`)">
        {{ t('products.viewDetail') }}
      </NuxtLink>
      <button :disabled="product.stock === 0" @click="$emit('add-to-cart', product)">
        {{ t('products.addToCart') }}
      </button>
    </div>
  </div>
</template>
```

---

## สรุป

i18n ใน Nuxt ด้วย `@nuxtjs/i18n` ช่วยสร้าง Multi-language app ได้ง่าย:

1. **Setup** - ตั้งค่า locales, lazy loading, strategy
2. **Translation Files** - จัดโครงสร้าง JSON files ตาม feature
3. **useI18n()** - ใช้ `t()`, `n()`, `d()`, `tc()` ใน component
4. **Formatting** - จัดรูปแบบตัวเลข/วันที่/สกุลเงินตาม locale
5. **RTL Support** - รองรับภาษาอาหรับ/ฮีบรู ด้วย CSS
6. **Locale Routing** - URL strategy ที่เหมาะกับ SEO
7. **Lazy Loading** - โหลด translation files เฉพาะ locale ที่ใช้
