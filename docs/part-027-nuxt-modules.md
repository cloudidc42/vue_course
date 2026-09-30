# Part 27: Nuxt Modules

## Nuxt Modules คืออะไร?

Nuxt Modules คือ package ที่ extend functionality ของ Nuxt โดยสามารถ:
- เพิ่ม auto-imports
- ลงทะเบียน components
- เพิ่ม server middleware
- แก้ไข Vite/Webpack config
- เพิ่ม runtime hooks

### ความแตกต่างระหว่าง Plugin และ Module

| | Plugin | Module |
|--|--------|--------|
| ทำงานเมื่อ | Runtime (app เริ่มทำงาน) | Build time |
| ใช้สำหรับ | Runtime behavior | Build configuration |
| ตัวอย่าง | Toast, Analytics | Tailwind, Image |

---

## ใช้ Community Modules (@nuxtjs/*)

### ติดตั้ง Module

```bash
# วิธีที่ 1: ใช้ nuxi (แนะนำ)
npx nuxi module add @nuxtjs/tailwindcss
npx nuxi module add @pinia/nuxt
npx nuxi module add @nuxt/image
npx nuxi module add @nuxt/content
npx nuxi module add nuxt-icon
npx nuxi module add @vueuse/nuxt

# วิธีที่ 2: ติดตั้ง manual
npm install @nuxtjs/tailwindcss
# แล้วเพิ่มใน nuxt.config.ts เอง
```

### กำหนด Modules ใน nuxt.config.ts

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@nuxtjs/tailwindcss',
    '@pinia/nuxt',
    '@nuxt/image',
    '@nuxt/content',
    'nuxt-icon',
    '@vueuse/nuxt',
    '@nuxtjs/google-fonts',
    'nuxt-security'
  ]
})
```

---

## Module Options

### กำหนด Options สำหรับ Module

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/tailwindcss', '@nuxt/image', 'nuxt-icon'],
  
  // Option 1: ใช้ key เดียวกับ module name
  tailwindcss: {
    cssPath: '~/assets/css/tailwind.css',
    configPath: 'tailwind.config.ts',
    exposeConfig: false,
    viewer: true
  },
  
  // Option 2: กำหนดใน modules array
  // modules: [
  //   ['@nuxt/image', { quality: 80, format: ['webp', 'avif'] }]
  // ]
  
  // Nuxt Image
  image: {
    quality: 80,
    format: ['webp', 'avif'],
    screens: {
      xs: 320,
      sm: 640,
      md: 768,
      lg: 1024,
      xl: 1280
    },
    providers: {
      cloudinary: {
        baseURL: 'https://res.cloudinary.com/my-cloud/image/upload/'
      }
    }
  },
  
  // Pinia
  pinia: {
    autoImports: ['defineStore', 'acceptHMRUpdate', 'storeToRefs']
  }
})
```

---

## Modules ที่แนะนำ

### 1. @nuxtjs/tailwindcss

```bash
npx nuxi module add @nuxtjs/tailwindcss
```

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss'

export default {
  content: [
    './components/**/*.{js,vue,ts}',
    './layouts/**/*.vue',
    './pages/**/*.vue',
    './plugins/**/*.{js,ts}',
    './app.vue',
    './error.vue'
  ],
  theme: {
    extend: {
      colors: {
        nuxt: {
          DEFAULT: '#00dc82',
          dark: '#003c22'
        }
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
        thai: ['Sarabun', 'sans-serif']
      }
    }
  },
  plugins: [
    require('@tailwindcss/typography'),
    require('@tailwindcss/forms')
  ]
} satisfies Config
```

### 2. @nuxt/image

```bash
npx nuxi module add @nuxt/image
```

```vue
<!-- การใช้งาน NuxtImg -->
<template>
  <!-- Basic usage -->
  <NuxtImg
    src="/images/hero.jpg"
    alt="Hero image"
    width="800"
    height="400"
    loading="lazy"
  />
  
  <!-- Responsive with sizes -->
  <NuxtImg
    src="/images/product.jpg"
    alt="Product"
    sizes="sm:100vw md:50vw lg:400px"
    :modifiers="{ fit: 'cover', gravity: 'auto' }"
  />
  
  <!-- Picture with sources -->
  <NuxtPicture
    src="/images/banner.jpg"
    alt="Banner"
    :imgAttrs="{ class: 'w-full' }"
  />
</template>
```

### 3. nuxt-icon

```bash
npx nuxi module add nuxt-icon
```

```vue
<template>
  <!-- ใช้ icons จาก iconify (100,000+ icons) -->
  <Icon name="mdi:home" size="24" />
  <Icon name="heroicons:user" size="20" class="text-blue-500" />
  <Icon name="ph:star-fill" size="16" style="color: gold" />
  
  <!-- Logos -->
  <Icon name="logos:vue" size="48" />
  <Icon name="logos:nuxt-icon" size="48" />
  <Icon name="logos:tailwindcss-icon" size="32" />
  
  <!-- Custom icons (ใส่ใน public/icons/) -->
  <Icon name="custom:logo" />
</template>
```

### 4. @pinia/nuxt

```bash
npx nuxi module add @pinia/nuxt
```

```typescript
// stores/counter.ts
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const doubleCount = computed(() => count.value * 2)
  
  function increment() { count.value++ }
  function reset() { count.value = 0 }
  
  return { count, doubleCount, increment, reset }
})
```

### 5. @vueuse/nuxt

```bash
npx nuxi module add @vueuse/nuxt
```

```vue
<script setup lang="ts">
// VueUse composables auto-imported ทั้งหมด!
const { x, y } = useMouse()
const { width, height } = useWindowSize()
const isOnline = useOnline()
const darkMode = useDark()
const { text, copy } = useClipboard()

// Storage
const name = useLocalStorage('user-name', 'Guest')

// Intersection Observer
const target = ref<HTMLElement | null>(null)
const { isIntersecting } = useIntersectionObserver(target, ([{ isIntersecting }]) => {
  console.log('Is visible:', isIntersecting)
})
</script>
```

---

## ตัวอย่าง: Setup Tailwind CSS สมบูรณ์

### ขั้นตอนการติดตั้ง

```bash
# 1. ติดตั้ง module
npx nuxi module add @nuxtjs/tailwindcss

# 2. ติดตั้ง additional packages
npm install -D @tailwindcss/typography @tailwindcss/forms @tailwindcss/aspect-ratio
```

### Tailwind Config

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss'

export default {
  content: [
    './components/**/*.{js,vue,ts}',
    './layouts/**/*.vue',
    './pages/**/*.vue',
    './plugins/**/*.{js,ts}',
    './app.vue',
    './error.vue'
  ],
  
  darkMode: 'class',
  
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#f0fdf4',
          100: '#dcfce7',
          200: '#bbf7d0',
          300: '#86efac',
          400: '#4ade80',
          500: '#00dc82',  // Nuxt green
          600: '#16a34a',
          700: '#15803d',
          800: '#166534',
          900: '#14532d'
        }
      },
      typography: (theme: any) => ({
        DEFAULT: {
          css: {
            'code::before': { content: '""' },
            'code::after': { content: '""' },
            code: {
              backgroundColor: theme('colors.gray.100'),
              borderRadius: theme('borderRadius.sm'),
              padding: '0.2em 0.4em',
              fontWeight: '400'
            }
          }
        }
      })
    }
  },
  
  plugins: [
    require('@tailwindcss/typography'),
    require('@tailwindcss/forms'),
    require('@tailwindcss/aspect-ratio')
  ]
} satisfies Config
```

### CSS File

```css
/* assets/css/tailwind.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  html {
    font-family: 'Sarabun', 'Inter', sans-serif;
  }
  
  h1, h2, h3, h4, h5, h6 {
    @apply font-bold text-gray-900;
  }
}

@layer components {
  .btn {
    @apply inline-flex items-center justify-center px-4 py-2 rounded-lg
           font-medium transition-all duration-200 focus:outline-none
           focus:ring-2 focus:ring-offset-2;
  }
  
  .btn-primary {
    @apply btn bg-primary-500 text-white hover:bg-primary-600
           focus:ring-primary-500;
  }
  
  .btn-secondary {
    @apply btn bg-white text-gray-700 border border-gray-300
           hover:bg-gray-50 focus:ring-gray-300;
  }
  
  .card {
    @apply bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden;
  }
  
  .input {
    @apply block w-full px-3 py-2 border border-gray-300 rounded-lg
           text-gray-900 placeholder-gray-400 focus:outline-none
           focus:ring-2 focus:ring-primary-500 focus:border-transparent;
  }
}

@layer utilities {
  .text-balance {
    text-wrap: balance;
  }
}
```

### ตัวอย่าง Component กับ Tailwind

```vue
<!-- components/PostCard.vue -->
<script setup lang="ts">
interface Post {
  id: number
  title: string
  excerpt: string
  author: { name: string; avatar: string }
  category: string
  publishedAt: string
  readingTime: number
  coverImage: string
}

defineProps<{ post: Post }>()
</script>

<template>
  <article class="card group hover:shadow-md transition-shadow duration-200">
    <!-- Cover Image -->
    <div class="aspect-w-16 aspect-h-9 overflow-hidden">
      <NuxtImg
        :src="post.coverImage"
        :alt="post.title"
        class="object-cover w-full h-full group-hover:scale-105
               transition-transform duration-300"
        width="400"
        height="225"
      />
    </div>
    
    <!-- Content -->
    <div class="p-5">
      <!-- Category -->
      <span class="inline-block px-2.5 py-0.5 rounded-full text-xs
                   font-medium bg-primary-100 text-primary-700 mb-3">
        {{ post.category }}
      </span>
      
      <!-- Title -->
      <h2 class="text-xl font-bold text-gray-900 mb-2 line-clamp-2
                 group-hover:text-primary-600 transition-colors">
        <NuxtLink :to="`/blog/${post.id}`">{{ post.title }}</NuxtLink>
      </h2>
      
      <!-- Excerpt -->
      <p class="text-gray-600 text-sm line-clamp-3 mb-4">
        {{ post.excerpt }}
      </p>
      
      <!-- Footer -->
      <div class="flex items-center justify-between text-sm text-gray-500">
        <div class="flex items-center gap-2">
          <img
            :src="post.author.avatar"
            :alt="post.author.name"
            class="w-7 h-7 rounded-full object-cover"
          />
          <span class="font-medium text-gray-700">{{ post.author.name }}</span>
        </div>
        <div class="flex items-center gap-3">
          <span>{{ post.readingTime }} นาที</span>
          <time>{{ new Date(post.publishedAt).toLocaleDateString('th-TH') }}</time>
        </div>
      </div>
    </div>
  </article>
</template>
```

---

## ตัวอย่าง: Image Optimization

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/image'],
  
  image: {
    quality: 80,
    format: ['avif', 'webp'],
    
    // ขนาดที่ใช้สำหรับ responsive images
    screens: {
      xs: 320,
      sm: 640,
      md: 768,
      lg: 1024,
      xl: 1280,
      xxl: 1536
    },
    
    // Providers
    providers: {
      // Local files (default)
      ipx: {},
      
      // Cloudinary
      cloudinary: {
        baseURL: 'https://res.cloudinary.com/demo/image/upload/'
      },
      
      // Unsplash
      unsplash: {
        baseURL: 'https://images.unsplash.com/'
      }
    }
  }
})
```

```vue
<!-- Responsive Image Examples -->
<template>
  <div>
    <!-- Hero image - ใหญ่มาก -->
    <NuxtImg
      src="/images/hero.jpg"
      alt="Hero"
      width="1920"
      height="1080"
      sizes="100vw"
      format="webp"
      quality="85"
      loading="eager"
      class="w-full"
    />
    
    <!-- Product thumbnail -->
    <NuxtImg
      src="/images/product.jpg"
      alt="Product"
      width="300"
      height="300"
      sizes="sm:150px md:200px lg:300px"
      fit="cover"
      loading="lazy"
      class="rounded-lg"
    />
    
    <!-- Avatar -->
    <NuxtImg
      :src="user.avatar"
      :alt="user.name"
      width="48"
      height="48"
      fit="cover"
      class="rounded-full"
    />
    
    <!-- Cloudinary image -->
    <NuxtImg
      provider="cloudinary"
      src="sample.jpg"
      width="400"
      height="300"
      :modifiers="{
        effect: 'auto_brightness',
        crop: 'fill',
        gravity: 'face'
      }"
    />
  </div>
</template>
```

---

## สร้าง Local Module

### โครงสร้าง Local Module

```
modules/
└── my-feature/
    ├── index.ts       # Module entry
    ├── runtime/
    │   ├── composables/
    │   │   └── useMyFeature.ts
    │   ├── components/
    │   │   └── MyComponent.vue
    │   └── plugins/
    │       └── plugin.ts
    └── types.ts
```

### สร้าง Simple Local Module

```typescript
// modules/analytics/index.ts
import { defineNuxtModule, addPlugin, addImports, createResolver } from '@nuxt/kit'

export interface ModuleOptions {
  trackingId: string
  debug?: boolean
}

export default defineNuxtModule<ModuleOptions>({
  meta: {
    name: 'my-analytics',
    configKey: 'analytics',
    compatibility: { nuxt: '^3.0.0' }
  },
  
  defaults: {
    trackingId: '',
    debug: false
  },
  
  setup(options, nuxt) {
    const resolver = createResolver(import.meta.url)
    
    // ส่ง options ไปยัง runtime
    nuxt.options.runtimeConfig.public.analytics = {
      trackingId: options.trackingId,
      debug: options.debug
    }
    
    // เพิ่ม plugin
    addPlugin({
      src: resolver.resolve('./runtime/plugins/analytics.client'),
      mode: 'client'
    })
    
    // เพิ่ม auto-imports
    addImports({
      name: 'useAnalytics',
      as: 'useAnalytics',
      from: resolver.resolve('./runtime/composables/useAnalytics')
    })
  }
})
```

### ใช้ Local Module

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    './modules/analytics'
  ],
  
  analytics: {
    trackingId: 'GA-XXXXXXXXX',
    debug: process.env.NODE_ENV === 'development'
  }
})
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Nuxt Modules คืออะไร** - Build-time extensions
2. **Community Modules** - ติดตั้งและใช้ @nuxtjs/* packages
3. **Module Options** - กำหนดค่าผ่าน nuxt.config.ts
4. **Tailwind CSS** - Setup และ configuration
5. **@nuxt/image** - Image optimization
6. **nuxt-icon** - Icon system
7. **สร้าง Local Module** - ด้วย @nuxt/kit
8. **VueUse** - Utility composables

**ถัดไป**: Part 28 - Nuxt Server Routes (API)
