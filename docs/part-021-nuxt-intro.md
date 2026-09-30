# Part 21: Nuxt.js - เริ่มต้น

## Nuxt.js คืออะไร?

Nuxt.js คือ Framework ระดับสูง (Higher-level Framework) ที่สร้างขึ้นบน Vue.js โดยเพิ่มความสามารถด้าน Server-Side Rendering (SSR), Static Site Generation (SSG) และ Single Page Application (SPA) ให้กับ Vue Application

### ความแตกต่างระหว่าง Rendering Modes

```
┌─────────────────────────────────────────────────────────────┐
│                    Rendering Modes                           │
├──────────────┬──────────────────┬──────────────────────────┤
│     SPA      │      SSR         │         SSG              │
├──────────────┼──────────────────┼──────────────────────────┤
│ Render ที่   │ Render ที่       │ Generate HTML ตอน       │
│ Browser      │ Server ทุก       │ Build time               │
│              │ Request          │                           │
├──────────────┼──────────────────┼──────────────────────────┤
│ SEO ไม่ดี   │ SEO ดีมาก        │ SEO ดีมาก                │
├──────────────┼──────────────────┼──────────────────────────┤
│ Load เร็ว   │ TTFB ช้ากว่า     │ Load เร็วมาก             │
│ หลังครั้งแรก │                 │                           │
└──────────────┴──────────────────┴──────────────────────────┘
```

### ทำไมต้องใช้ Nuxt?

1. **SEO ดีขึ้น** - Server renders HTML ก่อนส่งถึง Browser
2. **Performance** - ผู้ใช้เห็นเนื้อหาเร็วขึ้น (First Contentful Paint)
3. **Auto-imports** - ไม่ต้อง import Component, Composable ด้วยตัวเอง
4. **File-based Routing** - สร้างไฟล์ก็ได้ Route อัตโนมัติ
5. **Developer Experience** - Hot Module Replacement, TypeScript support
6. **Fullstack** - มี Server API ได้ในโปรเจกต์เดียว

---

## ติดตั้ง Nuxt.js ด้วย nuxi

### ความต้องการของระบบ

- Node.js เวอร์ชัน 18.x หรือสูงกว่า
- npm, yarn หรือ pnpm

### การสร้าง Nuxt Project ใหม่

```bash
# สร้างโปรเจกต์ใหม่ชื่อ my-nuxt-app
npx nuxi@latest init my-nuxt-app

# หรือระบุ template
npx nuxi@latest init my-nuxt-app --template v3

# เข้าไปในโฟลเดอร์
cd my-nuxt-app

# ติดตั้ง dependencies
npm install

# รัน development server
npm run dev
```

### คำสั่ง nuxi ที่ใช้บ่อย

```bash
# สร้างโปรเจกต์ใหม่
npx nuxi init <project-name>

# เพิ่ม Module
npx nuxi module add <module-name>

# สร้างไฟล์ต่างๆ
npx nuxi add page index          # สร้าง pages/index.vue
npx nuxi add component Header    # สร้าง components/Header.vue
npx nuxi add composable useUser  # สร้าง composables/useUser.ts
npx nuxi add layout default      # สร้าง layouts/default.vue
npx nuxi add plugin myPlugin     # สร้าง plugins/myPlugin.ts
npx nuxi add api users           # สร้าง server/api/users.ts

# Build
npm run build    # Build สำหรับ production (SSR)
npm run generate # Generate static site (SSG)
npm run preview  # Preview production build

# Analyze bundle
npx nuxi analyze
```

---

## โครงสร้าง Nuxt Project

```
my-nuxt-app/
├── .nuxt/              # Auto-generated (อย่าแก้ไขเอง)
├── .output/            # Production build output
├── assets/             # Static assets ที่ต้องผ่าน build (CSS, images)
├── components/         # Vue Components (auto-imported)
│   ├── AppHeader.vue
│   └── ui/
│       └── Button.vue
├── composables/        # Composable functions (auto-imported)
│   └── useUser.ts
├── content/            # Markdown/JSON content (ต้องติดตั้ง @nuxt/content)
├── layouts/            # Layout components
│   ├── default.vue
│   └── admin.vue
├── middleware/         # Route middleware
│   └── auth.ts
├── pages/              # File-based routing
│   ├── index.vue
│   ├── about.vue
│   └── posts/
│       ├── index.vue
│       └── [id].vue
├── plugins/            # Nuxt plugins
│   └── myPlugin.ts
├── public/             # Static files ที่ไม่ผ่าน build
│   └── favicon.ico
├── server/             # Server-side code
│   ├── api/
│   │   └── users.ts
│   ├── middleware/
│   └── plugins/
├── stores/             # Pinia stores (ถ้าใช้ Pinia)
├── utils/              # Utility functions (auto-imported)
├── app.vue             # Root component
├── error.vue           # Error page
├── nuxt.config.ts      # Nuxt configuration
├── package.json
└── tsconfig.json
```

### หน้าที่ของแต่ละโฟลเดอร์

| โฟลเดอร์ | หน้าที่ |
|----------|---------|
| `pages/` | กำหนด routes อัตโนมัติตามชื่อไฟล์ |
| `components/` | Vue components ที่ auto-import |
| `composables/` | Composable functions ที่ auto-import |
| `layouts/` | Layout wrapper สำหรับ pages |
| `middleware/` | Route middleware (ทำงานก่อน page render) |
| `plugins/` | Plugin ที่ load ก่อน app start |
| `server/` | API routes และ server middleware |
| `assets/` | ไฟล์ที่ผ่าน Vite/Webpack bundler |
| `public/` | Static files ที่ serve โดยตรง |
| `utils/` | Utility functions ที่ auto-import |

---

## Auto-imports ใน Nuxt

หนึ่งในฟีเจอร์เด่นของ Nuxt คือ Auto-imports ซึ่งทำให้ไม่ต้อง import ทุกอย่างด้วยตัวเอง

### สิ่งที่ Auto-import อัตโนมัติ

```vue
<script setup lang="ts">
// ไม่ต้อง import เหล่านี้เลย!

// Vue Composables
const count = ref(0)              // ref จาก Vue
const user = reactive({})         // reactive จาก Vue
const doubled = computed(() => count.value * 2)  // computed

// Nuxt Composables
const route = useRoute()          // route ปัจจุบัน
const router = useRouter()        // router instance
const { data } = await useFetch('/api/users')  // data fetching
const config = useRuntimeConfig() // runtime config

// ไฟล์ใน composables/ folder ก็ auto-import
const { user, login } = useAuth()  // composables/useAuth.ts
</script>
```

### Components Auto-import

```vue
<!-- ไม่ต้อง import components เหล่านี้ -->
<template>
  <div>
    <!-- AppHeader.vue จาก components/ -->
    <AppHeader />
    
    <!-- components/ui/Button.vue -->
    <!-- Nuxt แปลง path เป็น PascalCase ด้วย / = prefix -->
    <UiButton>Click me</UiButton>
    
    <!-- หรือใช้ kebab-case ก็ได้ -->
    <ui-button>Click me</ui-button>
  </div>
</template>
```

### ปิด Auto-imports (ถ้าต้องการ)

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  imports: {
    autoImport: false  // ปิด auto-import ทั้งหมด
  },
  components: {
    // ปิดเฉพาะ components
    dirs: []
  }
})
```

### เพิ่ม Custom Auto-imports

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  imports: {
    dirs: [
      'stores',        // auto-import จาก stores/
      'utils/**',      // auto-import ทุกไฟล์ใน utils/ recursively
    ]
  }
})
```

---

## nuxt.config.ts

ไฟล์กำหนดค่าหลักของ Nuxt application

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // ข้อมูลทั่วไปของ App
  app: {
    head: {
      title: 'My Nuxt App',
      meta: [
        { name: 'description', content: 'My awesome Nuxt application' },
        { name: 'viewport', content: 'width=device-width, initial-scale=1' }
      ],
      link: [
        { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' }
      ],
      script: [
        { src: 'https://example.com/script.js', defer: true }
      ]
    },
    pageTransition: { name: 'page', mode: 'out-in' },
    layoutTransition: { name: 'layout', mode: 'out-in' }
  },

  // Rendering mode
  ssr: true,  // true = SSR, false = SPA mode

  // TypeScript configuration
  typescript: {
    strict: true,
    typeCheck: true
  },

  // Modules ที่ติดตั้ง
  modules: [
    '@nuxtjs/tailwindcss',
    '@pinia/nuxt',
    '@nuxt/image',
    '@nuxt/content'
  ],

  // CSS files ที่ load ทุกหน้า
  css: [
    '~/assets/css/main.css'
  ],

  // Runtime Config (environment variables)
  runtimeConfig: {
    // Private (server-only)
    secretKey: process.env.SECRET_KEY,
    databaseUrl: process.env.DATABASE_URL,
    
    // Public (accessible on both server and client)
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE || 'https://api.example.com',
      appName: 'My App'
    }
  },

  // Nitro (server engine) configuration
  nitro: {
    preset: 'node-server',  // deployment target
    storage: {
      redis: {
        driver: 'redis',
        port: 6379,
        host: '127.0.0.1'
      }
    }
  },

  // Vite configuration
  vite: {
    css: {
      preprocessorOptions: {
        scss: {
          additionalData: '@import "~/assets/scss/variables.scss";'
        }
      }
    }
  },

  // Router configuration
  router: {
    options: {
      hashMode: false,
      scrollBehaviorType: 'smooth'
    }
  },

  // Dev tools
  devtools: { enabled: true },

  // Experimental features
  experimental: {
    payloadExtraction: false,
    inlineSSRStyles: false
  }
})
```

---

## .env และ Runtime Config

### ไฟล์ .env

```bash
# .env
# Private variables (server-only)
SECRET_KEY=my-super-secret-key
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
SMTP_HOST=smtp.example.com
SMTP_PASS=email-password

# Public variables (ต้องใส่ prefix NUXT_PUBLIC_)
NUXT_PUBLIC_API_BASE=https://api.example.com
NUXT_PUBLIC_APP_NAME=My Nuxt App
NUXT_PUBLIC_GA_ID=G-XXXXXXXXXX
```

### กำหนด Runtime Config

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // Private - ใช้ได้เฉพาะ server side
    secretKey: '',  // จะถูก override โดย SECRET_KEY env
    databaseUrl: '',
    
    // Public - ใช้ได้ทั้ง server และ client
    public: {
      apiBase: '',   // จะถูก override โดย NUXT_PUBLIC_API_BASE
      appName: 'Default App Name'
    }
  }
})
```

### ใช้ Runtime Config ใน Components

```vue
<!-- pages/index.vue -->
<script setup lang="ts">
// useRuntimeConfig() auto-imported
const config = useRuntimeConfig()

// Public config ใช้ได้ทั้งฝั่ง server และ client
console.log(config.public.apiBase)    // https://api.example.com
console.log(config.public.appName)    // My Nuxt App

// Private config ใช้ได้เฉพาะ server (จะเป็น undefined ใน client)
// console.log(config.secretKey)  // อันตราย! ไม่ควรทำใน component
</script>
```

### ใช้ Runtime Config ใน Server API

```typescript
// server/api/users.ts
export default defineEventHandler((event) => {
  const config = useRuntimeConfig(event)
  
  // ใช้ private config ได้ใน server
  const dbUrl = config.databaseUrl
  const secret = config.secretKey
  
  // ใช้ public config ก็ได้
  const apiBase = config.public.apiBase
  
  return { message: 'Hello from API' }
})
```

### การ Override ด้วย Environment Variables

Nuxt จะแปลง environment variable เป็น config key อัตโนมัติ:

```
NUXT_SECRET_KEY        → runtimeConfig.secretKey
NUXT_DATABASE_URL      → runtimeConfig.databaseUrl
NUXT_PUBLIC_API_BASE   → runtimeConfig.public.apiBase
NUXT_PUBLIC_APP_NAME   → runtimeConfig.public.appName
```

---

## เปรียบเทียบ Vue SPA vs Nuxt SSR

### Vue SPA (Single Page Application)

```
Client Request
      │
      ▼
┌─────────────┐
│   Server    │   ส่งแค่ HTML เปล่า + JS bundle
│  (static)   │
└─────────────┘
      │
      ▼
┌─────────────┐
│   Browser   │   Download JS → Execute → Render HTML
│             │   (ผู้ใช้รอนานกว่า)
└─────────────┘
```

```html
<!-- HTML ที่ส่งมาจาก SPA server -->
<!DOCTYPE html>
<html>
  <head>
    <title>Vue App</title>
  </head>
  <body>
    <div id="app"></div>  <!-- เปล่า! JS จะ fill content -->
    <script src="/assets/main.js"></script>
  </body>
</html>
```

### Nuxt SSR (Server-Side Rendering)

```
Client Request
      │
      ▼
┌─────────────┐
│   Nuxt      │   Execute Vue code → Generate HTML
│   Server    │   ส่ง HTML ที่มีเนื้อหาแล้ว
└─────────────┘
      │
      ▼
┌─────────────┐
│   Browser   │   รับ HTML ที่มีเนื้อหา → แสดงทันที
│             │   Download JS → Hydration (เพิ่ม interactivity)
└─────────────┘
```

```html
<!-- HTML ที่ส่งมาจาก Nuxt SSR -->
<!DOCTYPE html>
<html>
  <head>
    <title>My Nuxt App</title>
    <meta name="description" content="...">
  </head>
  <body>
    <div id="__nuxt">
      <header>...</header>
      <main>
        <h1>Welcome!</h1>
        <p>มีเนื้อหาแล้วตั้งแต่โหลด!</p>
      </main>
    </div>
    <script>window.__NUXT__={...}</script>
    <script src="/_nuxt/entry.js"></script>
  </body>
</html>
```

### ตารางเปรียบเทียบ

| Feature | Vue SPA | Nuxt SSR | Nuxt SSG |
|---------|---------|----------|----------|
| Initial Load | ช้า | เร็ว | เร็วมาก |
| SEO | ไม่ดี | ดีมาก | ดีมาก |
| Server Load | น้อย | มาก | น้อย |
| Dynamic Data | ใช่ | ใช่ | ไม่มาก |
| Hosting Cost | ถูก | แพงกว่า | ถูก |
| Build Time | เร็ว | เร็ว | ช้า (มาก pages) |
| Use Case | Admin panels | E-commerce, News | Blog, Docs |

---

## ตัวอย่าง: สร้าง Hello World App ด้วย Nuxt

### ขั้นตอนที่ 1: สร้างโปรเจกต์

```bash
npx nuxi@latest init hello-nuxt
cd hello-nuxt
npm install
```

### ขั้นตอนที่ 2: แก้ไข app.vue

```vue
<!-- app.vue -->
<template>
  <div>
    <NuxtLayout>
      <NuxtPage />
    </NuxtLayout>
  </div>
</template>
```

### ขั้นตอนที่ 3: สร้าง Layout

```vue
<!-- layouts/default.vue -->
<template>
  <div class="app-wrapper">
    <header class="app-header">
      <nav>
        <NuxtLink to="/">หน้าแรก</NuxtLink>
        <NuxtLink to="/about">เกี่ยวกับ</NuxtLink>
        <NuxtLink to="/posts">บทความ</NuxtLink>
      </nav>
    </header>
    
    <main class="app-main">
      <slot />  <!-- page content จะ render ที่นี่ -->
    </main>
    
    <footer class="app-footer">
      <p>© 2024 Hello Nuxt App</p>
    </footer>
  </div>
</template>

<style scoped>
.app-wrapper {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.app-header {
  background: #00dc82;
  padding: 1rem;
}

.app-header nav {
  display: flex;
  gap: 1rem;
}

.app-header a {
  color: white;
  text-decoration: none;
  font-weight: bold;
}

.app-main {
  flex: 1;
  padding: 2rem;
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
}

.app-footer {
  background: #1a1a2e;
  color: white;
  text-align: center;
  padding: 1rem;
}
</style>
```

### ขั้นตอนที่ 4: สร้าง Pages

```vue
<!-- pages/index.vue -->
<script setup lang="ts">
// กำหนด meta tags สำหรับ SEO
useSeoMeta({
  title: 'Hello Nuxt - หน้าแรก',
  description: 'ยินดีต้อนรับสู่ Hello Nuxt App',
  ogTitle: 'Hello Nuxt',
  ogDescription: 'แอปพลิเคชัน Nuxt.js แรกของฉัน'
})

// State
const greeting = ref('สวัสดี Nuxt!')
const count = ref(0)

// Methods
function increment() {
  count.value++
}

function reset() {
  count.value = 0
}

// Computed
const message = computed(() => {
  if (count.value === 0) return 'เริ่มนับโดยกดปุ่ม +'
  if (count.value < 5) return 'ดี! ลองกดต่อ'
  if (count.value < 10) return 'เยี่ยม! ใกล้จะถึง 10 แล้ว'
  return `นับถึง ${count.value} แล้ว! 🎉`
})
</script>

<template>
  <div class="home-page">
    <h1>{{ greeting }}</h1>
    <p class="subtitle">ยินดีต้อนรับสู่ Nuxt.js</p>
    
    <!-- Counter Example -->
    <div class="counter">
      <h2>Counter Demo</h2>
      <div class="counter-display">
        <span class="count">{{ count }}</span>
      </div>
      <p class="counter-message">{{ message }}</p>
      <div class="counter-buttons">
        <button @click="count--" :disabled="count <= 0">-</button>
        <button @click="reset" class="reset">Reset</button>
        <button @click="increment">+</button>
      </div>
    </div>
    
    <!-- Features List -->
    <div class="features">
      <h2>ทำไมต้อง Nuxt?</h2>
      <ul>
        <li v-for="feature in features" :key="feature.title">
          <strong>{{ feature.title }}</strong>
          <p>{{ feature.description }}</p>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup lang="ts">
const features = [
  {
    title: 'Server-Side Rendering',
    description: 'Render HTML บน server เพื่อ SEO และ performance ที่ดีขึ้น'
  },
  {
    title: 'Auto-imports',
    description: 'ไม่ต้อง import Vue, Nuxt composables ด้วยตัวเอง'
  },
  {
    title: 'File-based Routing',
    description: 'สร้างไฟล์ใน pages/ แล้ว route จะถูกสร้างอัตโนมัติ'
  },
  {
    title: 'TypeScript Ready',
    description: 'รองรับ TypeScript แบบ full-stack ทั้ง client และ server'
  },
  {
    title: 'Fullstack',
    description: 'มี server/api/ สำหรับ build API ในโปรเจกต์เดียว'
  }
]
</script>

<style scoped>
.home-page {
  max-width: 800px;
  margin: 0 auto;
}

h1 {
  font-size: 2.5rem;
  color: #00dc82;
  margin-bottom: 0.5rem;
}

.subtitle {
  color: #666;
  margin-bottom: 2rem;
}

.counter {
  background: #f8f9fa;
  border-radius: 12px;
  padding: 2rem;
  text-align: center;
  margin-bottom: 2rem;
}

.counter-display {
  margin: 1rem 0;
}

.count {
  font-size: 4rem;
  font-weight: bold;
  color: #00dc82;
}

.counter-message {
  color: #666;
  margin-bottom: 1rem;
}

.counter-buttons {
  display: flex;
  gap: 1rem;
  justify-content: center;
}

.counter-buttons button {
  padding: 0.5rem 1.5rem;
  font-size: 1.2rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  background: #00dc82;
  color: white;
  transition: opacity 0.2s;
}

.counter-buttons button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.counter-buttons button.reset {
  background: #dc3545;
}

.features ul {
  list-style: none;
  padding: 0;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
}

.features li {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 1rem;
}

.features strong {
  color: #00dc82;
  display: block;
  margin-bottom: 0.5rem;
}
</style>
```

### ขั้นตอนที่ 5: สร้าง About Page

```vue
<!-- pages/about.vue -->
<script setup lang="ts">
useSeoMeta({
  title: 'เกี่ยวกับเรา - Hello Nuxt',
  description: 'เรียนรู้เพิ่มเติมเกี่ยวกับ Hello Nuxt App'
})

const team = [
  { name: 'สมชาย ใจดี', role: 'Frontend Developer', avatar: '👨‍💻' },
  { name: 'สมหญิง รักเรียน', role: 'UI/UX Designer', avatar: '👩‍🎨' },
  { name: 'สมศักดิ์ ขยันทำ', role: 'Backend Developer', avatar: '👨‍🔧' }
]
</script>

<template>
  <div>
    <h1>เกี่ยวกับเรา</h1>
    <p>Hello Nuxt เป็นแอปพลิเคชันตัวอย่างสำหรับการเรียนรู้ Nuxt.js</p>
    
    <h2>ทีมงาน</h2>
    <div class="team-grid">
      <div v-for="member in team" :key="member.name" class="team-card">
        <div class="avatar">{{ member.avatar }}</div>
        <h3>{{ member.name }}</h3>
        <p>{{ member.role }}</p>
      </div>
    </div>
    
    <div class="tech-stack">
      <h2>Technology Stack</h2>
      <ul>
        <li>Nuxt.js 3</li>
        <li>Vue.js 3</li>
        <li>TypeScript</li>
        <li>Tailwind CSS</li>
        <li>Pinia</li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.team-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin: 1.5rem 0;
}

.team-card {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  padding: 1.5rem;
  text-align: center;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
}

.avatar {
  font-size: 3rem;
  margin-bottom: 0.5rem;
}

.team-card h3 {
  margin: 0;
  font-size: 1rem;
}

.team-card p {
  color: #666;
  font-size: 0.875rem;
}

.tech-stack ul {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  list-style: none;
  padding: 0;
}

.tech-stack li {
  background: #00dc82;
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 999px;
  font-size: 0.875rem;
}
</style>
```

### ขั้นตอนที่ 6: สร้าง API Endpoint

```typescript
// server/api/hello.ts
export default defineEventHandler((event) => {
  const query = getQuery(event)
  const name = query.name as string || 'World'
  
  return {
    message: `Hello, ${name}!`,
    timestamp: new Date().toISOString(),
    server: 'Nuxt Nitro Server'
  }
})
```

### ขั้นตอนที่ 7: ใช้ API ใน Component

```vue
<!-- pages/posts/index.vue -->
<script setup lang="ts">
useSeoMeta({
  title: 'บทความ - Hello Nuxt'
})

// useFetch auto-imported
const { data, pending, error } = await useFetch('/api/hello', {
  query: { name: 'Nuxt Developer' }
})
</script>

<template>
  <div>
    <h1>API Response Example</h1>
    
    <div v-if="pending">กำลังโหลด...</div>
    
    <div v-else-if="error" class="error">
      เกิดข้อผิดพลาด: {{ error.message }}
    </div>
    
    <div v-else class="response">
      <pre>{{ JSON.stringify(data, null, 2) }}</pre>
    </div>
  </div>
</template>
```

### ขั้นตอนที่ 8: nuxt.config.ts สำหรับโปรเจกต์นี้

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  devtools: { enabled: true },
  
  app: {
    head: {
      title: 'Hello Nuxt App',
      meta: [
        { name: 'description', content: 'แอปพลิเคชัน Nuxt.js ตัวอย่าง' }
      ]
    }
  },
  
  runtimeConfig: {
    public: {
      appName: 'Hello Nuxt'
    }
  }
})
```

### รัน Application

```bash
# Development
npm run dev
# เปิด http://localhost:3000

# Build สำหรับ Production
npm run build

# Preview Production Build
npm run preview
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Nuxt.js คืออะไร** - Framework ที่สร้างบน Vue.js รองรับ SSR, SSG, SPA
2. **การติดตั้ง** - ใช้ `npx nuxi@latest init` สร้างโปรเจกต์
3. **โครงสร้างโปรเจกต์** - pages/, components/, layouts/, server/ ฯลฯ
4. **Auto-imports** - ไม่ต้อง import Vue composables, components ด้วยตัวเอง
5. **nuxt.config.ts** - ไฟล์กำหนดค่าหลักของแอปพลิเคชัน
6. **Runtime Config** - จัดการ environment variables อย่างปลอดภัย
7. **Vue SPA vs Nuxt SSR** - ความแตกต่างและเมื่อไหรควรใช้อะไร

**ถัดไป**: Part 22 - Nuxt File-based Routing
