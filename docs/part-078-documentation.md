# Part 78: Documentation

## ทำไม Documentation ถึงสำคัญ?

Documentation ที่ดีช่วยให้:
- Developer ใหม่ onboard ได้เร็ว
- ลดคำถามซ้ำๆ ในทีม
- Code มีคุณภาพสูงขึ้นเพราะต้องอธิบายได้
- Project ยั่งยืนระยะยาว

---

## 1. JSDoc/TSDoc

### Function Documentation

```typescript
/**
 * Calculates the prorated amount for subscription upgrade
 * 
 * @param currentPlan - Current subscription plan name
 * @param newPlan - Target subscription plan name
 * @param daysRemaining - Days remaining in current billing period
 * @returns Prorated amount in cents
 * @throws {Error} When plan names are invalid
 * 
 * @example
 * ```typescript
 * const amount = calculateProration('starter', 'pro', 15)
 * // Returns 1500 (15 days × $100/month difference / 30 days)
 * ```
 */
export function calculateProration(
  currentPlan: string,
  newPlan: string,
  daysRemaining: number
): number {
  const plans = PLANS as Record<string, { price: number }>
  
  if (!plans[currentPlan] || !plans[newPlan]) {
    throw new Error(`Invalid plan names: ${currentPlan}, ${newPlan}`)
  }

  const priceDiff = plans[newPlan].price - plans[currentPlan].price
  if (priceDiff <= 0) return 0 // No proration for downgrades

  return Math.round((priceDiff / 30) * daysRemaining * 100)
}
```

### Class Documentation

```typescript
/**
 * Manages feature flag evaluation and caching
 * 
 * @remarks
 * Uses Redis for caching to minimize database queries.
 * Cache TTL is configurable and defaults to 60 seconds.
 * 
 * @example
 * ```typescript
 * const service = new FeatureFlagService()
 * 
 * const isEnabled = await service.isEnabled('new-checkout', {
 *   tenantId: 'tenant-123',
 *   userId: 'user-456',
 *   plan: 'pro'
 * })
 * ```
 */
export class FeatureFlagService {
  /**
   * @param cacheTTL - Cache duration in seconds (default: 60)
   */
  constructor(private cacheTTL: number = 60) {}

  /**
   * Checks if a feature flag is enabled for given context
   * 
   * @param key - Feature flag identifier
   * @param context - User/tenant context for evaluation
   * @returns Promise resolving to boolean flag state
   */
  async isEnabled(key: string, context: EvaluationContext): Promise<boolean> {
    // implementation
  }
}
```

---

## 2. Component Documentation

### Vue Component Documentation กับ TypeDoc

```vue
<!-- components/DataTable.vue -->
<template>
  <!-- ... -->
</template>

<script setup lang="ts">
/**
 * @component DataTable
 * @description A feature-rich data table component with sorting, filtering, and pagination.
 * 
 * @example
 * ```vue
 * <DataTable
 *   :columns="columns"
 *   :data="users"
 *   :loading="isLoading"
 *   :total="totalCount"
 *   @sort="handleSort"
 *   @page-change="handlePageChange"
 * />
 * ```
 */

interface Column<T = any> {
  /** Column identifier */
  key: string
  /** Display header text */
  label: string
  /** Whether this column is sortable */
  sortable?: boolean
  /** Custom renderer function */
  render?: (value: any, row: T) => string
}

interface Props {
  /** Column definitions */
  columns: Column[]
  /** Row data */
  data: Record<string, unknown>[]
  /** Show loading skeleton */
  loading?: boolean
  /** Total number of records (for pagination) */
  total?: number
  /** Current page number */
  page?: number
  /** Items per page */
  pageSize?: number
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  total: 0,
  page: 1,
  pageSize: 20
})

const emit = defineEmits<{
  /** Emitted when user clicks a sortable column header */
  sort: [{ column: string; direction: 'asc' | 'desc' }]
  /** Emitted when user changes page */
  'page-change': [page: number]
  /** Emitted when row is clicked */
  'row-click': [row: Record<string, unknown>]
}>()
</script>
```

---

## 3. API Documentation

### Inline Documentation กับ OpenAPI

```typescript
// server/api/v2/users/index.post.ts

/**
 * @openapi
 * /api/v2/users:
 *   post:
 *     tags: [Users]
 *     summary: Create a new user
 *     description: |
 *       Creates a new user in the current tenant workspace.
 *       Requires admin or owner role.
 *       Sends welcome email automatically.
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [email, name]
 *             properties:
 *               email:
 *                 type: string
 *                 format: email
 *                 example: john@example.com
 *               name:
 *                 type: string
 *                 minLength: 2
 *                 example: John Doe
 *               role:
 *                 type: string
 *                 enum: [admin, editor, viewer]
 *                 default: viewer
 *     responses:
 *       201:
 *         description: User created successfully
 *       409:
 *         description: User with email already exists
 *       422:
 *         description: Validation error
 */
export default defineEventHandler(async (event) => {
  // implementation
})
```

---

## 4. Architecture Decision Records (ADR)

### ADR Template

```markdown
<!-- docs/adrs/001-use-prisma-as-orm.md -->
# ADR-001: Use Prisma as ORM

**Status:** Accepted  
**Date:** 2024-01-15  
**Deciders:** Tech Lead, Backend Team  

## Context

เราต้องเลือก ORM/Query Builder สำหรับ PostgreSQL ใน Nuxt 3 project

## Options Considered

1. **Prisma** - Type-safe ORM กับ schema-first approach
2. **Drizzle ORM** - Lightweight, SQL-like syntax
3. **Knex.js** - SQL query builder
4. **Raw SQL** - ไม่ใช้ ORM

## Decision

เลือก **Prisma**

## Rationale

- Type safety ออกมาจาก schema โดยอัตโนมัติ
- Migration management ที่ดี
- Prisma Studio สำหรับ data browsing
- Community ใหญ่ หา resources ง่าย
- รองรับ multi-tenant ด้วย middleware
- Integration กับ Nuxt/Nitro ดี

## Consequences

### Positive
- Developer experience ดีขึ้นมาก
- Type errors ถูกจับตั้งแต่ compile time
- Migration history ชัดเจน

### Negative
- Bundle size ใหญ่กว่า Drizzle (~2MB vs ~400KB)
- Generated client ต้อง regenerate เมื่อ schema เปลี่ยน
- ไม่ยืดหยุ่นเท่า raw SQL สำหรับ complex queries

## References

- [Prisma vs Drizzle Benchmark](https://example.com)
- [Nuxt + Prisma Guide](https://prisma.io/nuxt)
```

---

## 5. VitePress Documentation Site

### Project Setup

```bash
# สร้าง VitePress project
npm create vitepress@latest docs

# หรือเพิ่มใน existing project
npm add -D vitepress
mkdir docs && cd docs
npx vitepress init
```

### VitePress Config

```typescript
// docs/.vitepress/config.ts
import { defineConfig } from 'vitepress'

export default defineConfig({
  title: 'My Component Library',
  description: 'Documentation for Vue Component Library',
  
  themeConfig: {
    logo: '/logo.svg',
    
    nav: [
      { text: 'Guide', link: '/guide/' },
      { text: 'Components', link: '/components/' },
      { text: 'API', link: '/api/' },
      {
        text: 'v2.0.0',
        items: [
          { text: 'v2.0.0', link: '/changelog/v2' },
          { text: 'v1.0.0', link: '/changelog/v1' }
        ]
      }
    ],
    
    sidebar: {
      '/guide/': [
        {
          text: 'Introduction',
          items: [
            { text: 'Getting Started', link: '/guide/getting-started' },
            { text: 'Installation', link: '/guide/installation' },
            { text: 'Quick Start', link: '/guide/quick-start' }
          ]
        }
      ],
      '/components/': [
        {
          text: 'Form Elements',
          items: [
            { text: 'Button', link: '/components/button' },
            { text: 'Input', link: '/components/input' },
            { text: 'Select', link: '/components/select' }
          ]
        },
        {
          text: 'Data Display',
          items: [
            { text: 'DataTable', link: '/components/data-table' },
            { text: 'Card', link: '/components/card' }
          ]
        }
      ]
    },
    
    search: {
      provider: 'local'
    },
    
    socialLinks: [
      { icon: 'github', link: 'https://github.com/myorg/mylib' }
    ]
  },
  
  markdown: {
    theme: 'github-dark',
    lineNumbers: true
  }
})
```

---

## 6. ตัวอย่าง: VitePress Docs สำหรับ Component Library

### Component Documentation Page

```markdown
<!-- docs/components/button.md -->
# Button

A versatile button component for triggering actions.

## Basic Usage

<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>

<div class="demo">
  <Button @click="count++">Clicked {{ count }} times</Button>
</div>

```vue
<Button @click="count++">Clicked {{ count }} times</Button>
```

## Variants

<div class="demo">
  <Button variant="primary">Primary</Button>
  <Button variant="secondary">Secondary</Button>
  <Button variant="danger">Danger</Button>
</div>

```vue
<Button variant="primary">Primary</Button>
<Button variant="secondary">Secondary</Button>
<Button variant="danger">Danger</Button>
```

## Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| variant | `'primary' \| 'secondary' \| 'danger'` | `'primary'` | Button style variant |
| size | `'sm' \| 'md' \| 'lg'` | `'md'` | Button size |
| loading | `boolean` | `false` | Show loading spinner |
| disabled | `boolean` | `false` | Disable button |

## Events

| Event | Payload | Description |
|-------|---------|-------------|
| click | `MouseEvent` | Emitted on click (not emitted when disabled/loading) |

## Slots

| Slot | Description |
|------|-------------|
| default | Button label content |
| icon | Icon before label |
```

### Custom VitePress Theme

```typescript
// docs/.vitepress/theme/index.ts
import DefaultTheme from 'vitepress/theme'
import { App } from 'vue'
import MyComponentLibrary from '../../../src' // Import your library
import DemoContainer from './components/DemoContainer.vue'
import './styles/custom.css'

export default {
  extends: DefaultTheme,
  enhanceApp({ app }: { app: App }) {
    app.use(MyComponentLibrary)
    app.component('DemoContainer', DemoContainer)
  }
}
```

### Auto-generate API Docs

```typescript
// scripts/generate-docs.ts
import { Project, SourceFile } from 'ts-morph'
import * as fs from 'fs'
import * as path from 'path'

const project = new Project({
  tsConfigFilePath: 'tsconfig.json'
})

function generateComponentDoc(sourceFile: SourceFile) {
  const propsInterface = sourceFile.getInterface('Props')
  if (!propsInterface) return null

  const props = propsInterface.getProperties().map(prop => ({
    name: prop.getName(),
    type: prop.getTypeNode()?.getText() || 'unknown',
    required: !prop.hasQuestionToken(),
    description: prop.getJsDocs()[0]?.getDescription().trim() || '',
    default: prop.getJsDocs()[0]?.getTags()
      .find(t => t.getTagName() === 'default')?.getText() || ''
  }))

  return { props }
}

// Generate markdown for each component
const componentFiles = project.getSourceFiles('src/components/**/*.vue')
for (const file of componentFiles) {
  const doc = generateComponentDoc(file)
  if (doc) {
    const outputPath = `docs/components/${path.basename(file.getFilePath(), '.vue').toLowerCase()}.md`
    // Write documentation
    fs.writeFileSync(outputPath, generateMarkdown(doc))
  }
}
```

---

## สรุป

Documentation ที่ดีต้องการ:
1. **JSDoc/TSDoc** สำหรับ functions และ classes
2. **Component stories** แสดง variants และ props
3. **API docs** ที่ sync กับ code (OpenAPI)
4. **ADRs** บันทึกการตัดสินใจสำคัญ
5. **VitePress** สำหรับ documentation site ที่สวยงาม
6. **Keep it updated** - Documentation ที่ outdated แย่กว่าไม่มี
