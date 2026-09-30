# Part 79: Open Source Contribution

## Open Source คืออะไร?

Open source หมายถึง software ที่ source code เปิดเผยให้ทุกคนดู แก้ไข และแจกจ่ายได้ การมีส่วนร่วมใน open source community ช่วยพัฒนาทักษะ สร้าง portfolio และ network

---

## 1. Contributing to Vue/Nuxt Ecosystem

### วิธีเริ่มต้น Contribute

```bash
# 1. Fork repository
gh repo fork vuejs/core

# 2. Clone fork ของคุณ
git clone https://github.com/your-username/core
cd core

# 3. สร้าง branch
git checkout -b fix/improve-ref-type-inference

# 4. ติดตั้ง dependencies
pnpm install

# 5. ทำการเปลี่ยนแปลง + tests
# 6. Push และสร้าง PR
git push origin fix/improve-ref-type-inference
gh pr create
```

---

## 2. สร้าง Nuxt Module

### Module Structure

```
nuxt-my-module/
├── src/
│   ├── module.ts        # Main module entry
│   ├── runtime/
│   │   ├── plugin.ts    # Client/server plugin
│   │   ├── composables/ 
│   │   │   └── useMyFeature.ts
│   │   └── components/
│   │       └── MyComponent.vue
│   └── types.ts
├── playground/          # Test project
│   ├── nuxt.config.ts
│   └── app.vue
├── test/
├── package.json
└── README.md
```

### Module Implementation

```typescript
// src/module.ts
import { defineNuxtModule, addPlugin, createResolver, addImportsDir, addComponentsDir } from '@nuxt/kit'
import type { Resolver } from '@nuxt/kit'

export interface ModuleOptions {
  apiKey?: string
  endpoint?: string
  enabled?: boolean
  debug?: boolean
}

export default defineNuxtModule<ModuleOptions>({
  meta: {
    name: 'nuxt-my-analytics',
    configKey: 'myAnalytics',
    compatibility: {
      nuxt: '^3.0.0'
    }
  },

  defaults: {
    enabled: true,
    debug: false,
    endpoint: 'https://analytics.example.com'
  },

  async setup(options, nuxt) {
    const resolver = createResolver(import.meta.url)

    // Validate options
    if (options.enabled && !options.apiKey) {
      console.warn('[nuxt-my-analytics] No API key provided. Module will be disabled.')
      return
    }

    // Add runtime config
    nuxt.options.runtimeConfig.public.myAnalytics = {
      endpoint: options.endpoint!,
      debug: options.debug!
    }
    nuxt.options.runtimeConfig.myAnalytics = {
      apiKey: options.apiKey || ''
    }

    // Add plugin
    addPlugin({
      src: resolver.resolve('./runtime/plugin'),
      mode: 'client' // Only client-side
    })

    // Add composables
    addImportsDir(resolver.resolve('./runtime/composables'))

    // Add components
    addComponentsDir({
      path: resolver.resolve('./runtime/components'),
      prefix: 'Analytics'
    })

    // Add types
    nuxt.hook('prepare:types', ({ references }) => {
      references.push({ 
        types: 'nuxt-my-analytics' 
      })
    })

    // Development tooling
    if (nuxt.options.dev) {
      console.log('[nuxt-my-analytics] Development mode enabled')
    }
  }
})
```

### Runtime Plugin

```typescript
// src/runtime/plugin.ts
import { defineNuxtPlugin, useRuntimeConfig } from '#app'

export default defineNuxtPlugin({
  name: 'my-analytics-plugin',
  parallel: true, // Run ขนานกับ plugins อื่น

  setup(nuxtApp) {
    const config = useRuntimeConfig()
    const { endpoint, debug } = config.public.myAnalytics

    const analytics = {
      track(event: string, properties?: Record<string, unknown>) {
        if (debug) {
          console.log('[Analytics] Track:', event, properties)
        }

        fetch(`${endpoint}/track`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ event, properties, timestamp: Date.now() })
        }).catch(err => {
          if (debug) console.error('[Analytics] Error:', err)
        })
      },

      identify(userId: string, traits?: Record<string, unknown>) {
        if (debug) console.log('[Analytics] Identify:', userId, traits)
        // implementation
      },

      page(name: string, properties?: Record<string, unknown>) {
        if (debug) console.log('[Analytics] Page:', name, properties)
        // implementation
      }
    }

    // Auto-track page views
    nuxtApp.hook('page:finish', () => {
      analytics.page(window.location.pathname)
    })

    // Provide to app
    nuxtApp.provide('analytics', analytics)
  }
})

// Type augmentation
declare module '#app' {
  interface NuxtApp {
    $analytics: {
      track: (event: string, properties?: Record<string, unknown>) => void
      identify: (userId: string, traits?: Record<string, unknown>) => void
      page: (name: string, properties?: Record<string, unknown>) => void
    }
  }
}
```

---

## 3. สร้าง Vue Plugin

```typescript
// src/index.ts
import type { App, Plugin } from 'vue'
import { ref, computed } from 'vue'

// Types
export interface ToastOptions {
  message: string
  type?: 'success' | 'error' | 'warning' | 'info'
  duration?: number
  position?: 'top' | 'bottom' | 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'
}

export interface Toast extends Required<ToastOptions> {
  id: string
  createdAt: number
}

// State
const toasts = ref<Toast[]>([])

// Toast service
export const toast = {
  show(options: ToastOptions) {
    const newToast: Toast = {
      id: Math.random().toString(36).slice(2),
      message: options.message,
      type: options.type || 'info',
      duration: options.duration ?? 3000,
      position: options.position || 'top-right',
      createdAt: Date.now()
    }

    toasts.value.push(newToast)

    if (newToast.duration > 0) {
      setTimeout(() => this.dismiss(newToast.id), newToast.duration)
    }

    return newToast.id
  },

  dismiss(id: string) {
    const index = toasts.value.findIndex(t => t.id === id)
    if (index !== -1) toasts.value.splice(index, 1)
  },

  success: (message: string, options?: Partial<ToastOptions>) =>
    toast.show({ ...options, message, type: 'success' }),

  error: (message: string, options?: Partial<ToastOptions>) =>
    toast.show({ ...options, message, type: 'error' }),

  warning: (message: string, options?: Partial<ToastOptions>) =>
    toast.show({ ...options, message, type: 'warning' }),

  info: (message: string, options?: Partial<ToastOptions>) =>
    toast.show({ ...options, message, type: 'info' })
}

// useToast composable
export function useToast() {
  return {
    toasts: computed(() => toasts.value),
    ...toast
  }
}

// Vue Plugin
export const VueToastPlugin: Plugin = {
  install(app: App, options?: { position?: string }) {
    // Global property
    app.config.globalProperties.$toast = toast

    // Provide for composition API
    app.provide('toast', toast)
  }
}

// Default export
export default VueToastPlugin

// Type augmentation
declare module '@vue/runtime-core' {
  interface ComponentCustomProperties {
    $toast: typeof toast
  }
}
```

---

## 4. GitHub Actions สำหรับ Open Source

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm test -- --coverage
      - uses: codecov/codecov-action@v3
        if: matrix.os == 'ubuntu-latest' && matrix.node == 20

  release:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm run build
      - name: Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```

---

## 5. Semantic Release

```json
// .releaserc.json
{
  "branches": ["main", { "name": "beta", "prerelease": true }],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    ["@semantic-release/npm", {
      "npmPublish": true
    }],
    ["@semantic-release/github", {
      "assets": [
        { "path": "dist/**", "label": "Distribution files" }
      ]
    }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }]
  ]
}
```

---

## 6. npm Publishing

### package.json สำหรับ Library

```json
{
  "name": "vue-toast-plugin",
  "version": "1.0.0",
  "description": "A beautiful toast notification plugin for Vue 3",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  },
  "files": ["dist", "README.md", "CHANGELOG.md"],
  "keywords": ["vue", "vue3", "toast", "notification", "plugin"],
  "peerDependencies": {
    "vue": "^3.3.0"
  },
  "scripts": {
    "build": "vite build && tsc --emitDeclarationOnly",
    "prepublishOnly": "npm run build && npm test"
  }
}
```

### Vite Library Build

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import dts from 'vite-plugin-dts'
import { resolve } from 'path'

export default defineConfig({
  plugins: [
    vue(),
    dts({ rollupTypes: true })
  ],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/index.ts'),
      name: 'VueToastPlugin',
      formats: ['es', 'cjs'],
      fileName: (format) => `index.${format === 'es' ? 'js' : 'cjs'}`
    },
    rollupOptions: {
      external: ['vue'],
      output: {
        globals: { vue: 'Vue' }
      }
    }
  }
})
```

---

## 7. ตัวอย่าง: สร้างและ Publish Vue Plugin สมบูรณ์

### README.md Template

```markdown
# vue-toast-plugin

> Beautiful toast notifications for Vue 3

[![npm version](https://badge.fury.io/js/vue-toast-plugin.svg)](https://badge.fury.io/js/vue-toast-plugin)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Installation

```bash
npm install vue-toast-plugin
```

## Quick Start

```typescript
// main.ts
import { createApp } from 'vue'
import VueToastPlugin from 'vue-toast-plugin'
import 'vue-toast-plugin/dist/style.css'
import App from './App.vue'

createApp(App)
  .use(VueToastPlugin)
  .mount('#app')
```

```vue
<script setup>
const { toast } = useToast()
</script>

<template>
  <button @click="toast.success('Saved!')">Save</button>
</template>
```
```

---

## สรุป

การมีส่วนร่วมใน Open Source:
1. **Start small** - แก้ bug, ปรับปรุง docs ก่อน
2. **Read CONTRIBUTING.md** - ทำตาม guidelines ของแต่ละ project
3. **Write tests** - Open source projects ต้องการ tests เสมอ
4. **Be patient** - Maintainers อาจใช้เวลา review
5. **Semantic release** - Automate publishing workflow
6. **Good README** - First impression สำคัญ
