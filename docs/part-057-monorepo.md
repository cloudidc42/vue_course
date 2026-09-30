# Part 57: Monorepo Setup สำหรับ Vue/Nuxt Projects

## Monorepo คืออะไร

Monorepo คือการเก็บหลาย projects หรือ packages ไว้ใน repository เดียว แทนที่จะแยกเป็น repository ย่อยๆ (polyrepo) วิธีนี้ช่วยให้:
- แชร์ code ระหว่าง projects ได้ง่าย
- จัดการ dependencies ได้ง่ายขึ้น
- Atomic commits ข้าม packages
- Consistent tooling ทั้ง repo

## ข้อดีและข้อเสียของ Monorepo

### ข้อดี
- Code sharing ง่าย
- Refactoring ข้าม packages ง่าย
- Single CI/CD pipeline
- Consistent code style

### ข้อเสีย
- Repo ขนาดใหญ่
- Build times นานขึ้น (ต้องใช้ caching)
- Complex access control

## โครงสร้าง Monorepo

```
my-monorepo/
├── apps/
│   ├── web/          # Nuxt.js web app
│   ├── admin/        # Admin dashboard
│   └── mobile/       # Capacitor mobile app
├── packages/
│   ├── ui/           # Shared UI components
│   ├── utils/        # Shared utilities
│   ├── types/        # Shared TypeScript types
│   └── config/       # Shared configs
├── package.json
├── pnpm-workspace.yaml
└── turbo.json
```

## pnpm Workspaces

pnpm workspaces ช่วยจัดการ packages ใน monorepo

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```json
// package.json (root)
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev --parallel",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "type-check": "turbo run type-check",
    "clean": "turbo run clean && rm -rf node_modules"
  },
  "devDependencies": {
    "turbo": "^1.12.0",
    "typescript": "^5.3.0",
    "eslint": "^8.56.0"
  },
  "engines": {
    "node": ">=18.0.0",
    "pnpm": ">=8.0.0"
  },
  "packageManager": "pnpm@8.12.0"
}
```

## Turborepo Setup

Turborepo ช่วยเพิ่มความเร็วในการ build ด้วย caching และ parallel execution

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".output/**", "dist/**", ".nuxt/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    },
    "type-check": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "clean": {
      "cache": false
    }
  }
}
```

## Shared Packages

### Shared UI Package

```
packages/ui/
├── src/
│   ├── components/
│   │   ├── Button/
│   │   │   ├── Button.vue
│   │   │   ├── Button.test.ts
│   │   │   └── index.ts
│   │   ├── Input/
│   │   └── Modal/
│   ├── composables/
│   ├── styles/
│   └── index.ts
├── package.json
└── tsconfig.json
```

```json
// packages/ui/package.json
{
  "name": "@my-monorepo/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./styles": "./dist/styles.css"
  },
  "scripts": {
    "build": "vite build",
    "dev": "vite build --watch",
    "type-check": "vue-tsc --noEmit"
  },
  "peerDependencies": {
    "vue": "^3.4.0"
  },
  "devDependencies": {
    "vue": "^3.4.0",
    "vite": "^5.0.0",
    "vue-tsc": "^1.8.0",
    "@vitejs/plugin-vue": "^5.0.0"
  }
}
```

```typescript
// packages/ui/src/components/Button/Button.vue
<template>
  <button
    :class="[
      'btn',
      `btn--${variant}`,
      `btn--${size}`,
      { 'btn--loading': loading, 'btn--block': block }
    ]"
    :disabled="disabled || loading"
    :type="type"
    v-bind="$attrs"
  >
    <span v-if="loading" class="btn__spinner" aria-hidden="true"></span>
    <span class="btn__content">
      <slot></slot>
    </span>
  </button>
</template>

<script setup lang="ts">
interface Props {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  type?: 'button' | 'submit' | 'reset'
  loading?: boolean
  disabled?: boolean
  block?: boolean
}

withDefaults(defineProps<Props>(), {
  variant: 'primary',
  size: 'md',
  type: 'button'
})
</script>
```

```typescript
// packages/ui/src/index.ts
export { default as Button } from './components/Button/Button.vue'
export { default as Input } from './components/Input/Input.vue'
export { default as Modal } from './components/Modal/Modal.vue'
export { default as Toast } from './components/Toast/Toast.vue'
export { default as DataTable } from './components/DataTable/DataTable.vue'

// Composables
export { useModal } from './composables/useModal'
export { useToast } from './composables/useToast'

// Types
export type { ButtonProps } from './components/Button/types'
export type { InputProps } from './components/Input/types'
```

### Shared Utils Package

```json
// packages/utils/package.json
{
  "name": "@my-monorepo/utils",
  "version": "0.0.0",
  "private": true,
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "dev": "tsup src/index.ts --format cjs,esm --dts --watch",
    "test": "vitest run"
  },
  "devDependencies": {
    "tsup": "^8.0.0",
    "vitest": "^1.0.0"
  }
}
```

```typescript
// packages/utils/src/index.ts
export * from './date'
export * from './string'
export * from './number'
export * from './validation'
export * from './api'
```

```typescript
// packages/utils/src/date.ts
export function formatDate(date: Date | string, locale = 'th-TH'): string {
  const d = typeof date === 'string' ? new Date(date) : date
  return d.toLocaleDateString(locale, {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

export function formatDateTime(date: Date | string, locale = 'th-TH'): string {
  const d = typeof date === 'string' ? new Date(date) : date
  return d.toLocaleString(locale)
}

export function relativeTime(date: Date | string): string {
  const d = typeof date === 'string' ? new Date(date) : date
  const now = new Date()
  const diff = now.getTime() - d.getTime()
  
  const seconds = Math.floor(diff / 1000)
  const minutes = Math.floor(seconds / 60)
  const hours = Math.floor(minutes / 60)
  const days = Math.floor(hours / 24)
  
  if (seconds < 60) return 'เมื่อสักครู่'
  if (minutes < 60) return `${minutes} นาทีที่แล้ว`
  if (hours < 24) return `${hours} ชั่วโมงที่แล้ว`
  if (days < 7) return `${days} วันที่แล้ว`
  
  return formatDate(d)
}
```

### Shared Types Package

```typescript
// packages/types/src/index.ts
export interface User {
  id: string
  name: string
  email: string
  avatar?: string
  role: 'admin' | 'user' | 'moderator'
  createdAt: string
  updatedAt: string
}

export interface ApiResponse<T> {
  data: T
  message: string
  success: boolean
  errors?: Record<string, string[]>
}

export interface PaginatedResponse<T> {
  data: T[]
  meta: {
    currentPage: number
    totalPages: number
    totalItems: number
    itemsPerPage: number
  }
}

export interface Product {
  id: string
  name: string
  description: string
  price: number
  images: string[]
  category: string
  stock: number
  sku: string
}

export type Theme = 'light' | 'dark' | 'system'
export type Locale = 'th' | 'en'
```

## Shared Config

### ESLint Config

```javascript
// packages/config/eslint/index.js
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:vue/vue3-recommended',
    '@vue/typescript/recommended',
    'plugin:@typescript-eslint/recommended',
    'prettier'
  ],
  plugins: ['vue', '@typescript-eslint'],
  parser: 'vue-eslint-parser',
  parserOptions: {
    parser: '@typescript-eslint/parser',
    ecmaVersion: 2022,
    sourceType: 'module'
  },
  rules: {
    'vue/multi-word-component-names': 'off',
    'vue/component-api-style': ['error', ['script-setup']],
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/consistent-type-imports': 'error',
    'no-console': ['warn', { allow: ['warn', 'error'] }]
  },
  overrides: [
    {
      files: ['*.test.ts', '*.spec.ts'],
      rules: {
        '@typescript-eslint/no-explicit-any': 'off'
      }
    }
  ]
}
```

```json
// packages/config/eslint/package.json
{
  "name": "@my-monorepo/eslint-config",
  "version": "0.0.0",
  "private": true,
  "main": "./index.js",
  "dependencies": {
    "@vue/eslint-config-typescript": "^12.0.0",
    "eslint-config-prettier": "^9.0.0",
    "eslint-plugin-vue": "^9.19.0"
  }
}
```

### TypeScript Config

```json
// packages/config/typescript/base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true
  }
}
```

```json
// packages/config/typescript/vue.json
{
  "extends": "./base.json",
  "compilerOptions": {
    "jsx": "preserve",
    "jsxImportSource": "vue"
  }
}
```

## Nuxt App Setup

```json
// apps/web/package.json
{
  "name": "@my-monorepo/web",
  "private": true,
  "scripts": {
    "build": "nuxt build",
    "dev": "nuxt dev",
    "generate": "nuxt generate",
    "preview": "nuxt preview",
    "type-check": "nuxt typecheck",
    "lint": "eslint . --ext .vue,.ts,.tsx"
  },
  "dependencies": {
    "@my-monorepo/ui": "workspace:*",
    "@my-monorepo/utils": "workspace:*",
    "@my-monorepo/types": "workspace:*",
    "nuxt": "^3.10.0"
  },
  "devDependencies": {
    "@my-monorepo/eslint-config": "workspace:*",
    "@my-monorepo/tsconfig": "workspace:*"
  }
}
```

```typescript
// apps/web/nuxt.config.ts
export default defineNuxtConfig({
  devtools: { enabled: true },
  
  typescript: {
    strict: true,
    tsConfig: {
      extends: '@my-monorepo/tsconfig/vue.json'
    }
  },
  
  // Auto-import shared packages
  imports: {
    dirs: ['composables', 'utils'],
    packages: [
      { package: '@my-monorepo/utils', imports: ['formatDate', 'relativeTime'] }
    ]
  },
  
  components: {
    dirs: [
      {
        path: '~/components',
        prefix: ''
      }
    ]
  },
  
  // Use shared UI components as Nuxt module
  modules: ['@my-monorepo/ui/nuxt'],
  
  // Shared styles
  css: ['@my-monorepo/ui/styles']
})
```

## Build Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      web: ${{ steps.filter.outputs.web }}
      admin: ${{ steps.filter.outputs.admin }}
      packages: ${{ steps.filter.outputs.packages }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            web:
              - 'apps/web/**'
              - 'packages/**'
            admin:
              - 'apps/admin/**'
              - 'packages/**'
            packages:
              - 'packages/**'

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
        with:
          version: 8
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Build packages
        run: pnpm turbo run build --filter=./packages/*
      
      - name: Type check
        run: pnpm turbo run type-check
      
      - name: Lint
        run: pnpm turbo run lint
      
      - name: Test
        run: pnpm turbo run test
      
      - name: Build apps
        run: pnpm turbo run build --filter=./apps/*
```

## ตัวอย่าง: Monorepo สำหรับ Nuxt App + Shared UI

```typescript
// apps/web/pages/index.vue
<template>
  <div>
    <h1>Welcome to Vue Course</h1>
    
    <!-- Using shared UI components -->
    <Button variant="primary" size="lg" @click="handleClick">
      เริ่มเรียน
    </Button>
    
    <!-- Using shared utils -->
    <p>วันที่: {{ formatDate(new Date()) }}</p>
    <p>{{ relativeTime(lastLogin) }}</p>
  </div>
</template>

<script setup lang="ts">
import { Button } from '@my-monorepo/ui'
import { formatDate, relativeTime } from '@my-monorepo/utils'
import type { User } from '@my-monorepo/types'

const lastLogin = new Date(Date.now() - 3600000) // 1 hour ago

const handleClick = () => {
  navigateTo('/lessons')
}
</script>
```

## Remote Caching กับ Turborepo

```bash
# เชื่อมต่อกับ Vercel Remote Cache
npx turbo login
npx turbo link

# ตั้งค่า Remote Cache ใน turbo.json
```

```json
// turbo.json (with remote caching)
{
  "$schema": "https://turbo.build/schema.json",
  "remoteCache": {
    "signature": true
  },
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".output/**", "dist/**", ".nuxt/**"],
      "cache": true
    }
  }
}
```

## สรุป

Monorepo ด้วย pnpm + Turborepo ให้ประโยชน์มาก:
1. แชร์ code ง่าย
2. Build เร็วด้วย caching
3. Type-safe ทั้ง repo
4. Consistent config

เริ่มต้นด้วย `pnpm create turbo@latest` แล้วปรับแต่งตามความต้องการ
