# Part 55: สร้าง Vue Component Library

## Component Library คืออะไร?

Component Library คือชุด reusable Vue components ที่ package เป็น npm package พร้อม TypeScript types, documentation และ Storybook เพื่อให้ทีมอื่นใช้งานได้

## 1. ตั้งค่าโปรเจกต์ด้วย Vite Library Mode

```bash
# สร้างโปรเจกต์ใหม่
npm create vite@latest my-vue-lib -- --template vue-ts
cd my-vue-lib
npm install

# ติดตั้ง dependencies
npm install -D vite @vitejs/plugin-vue vue-tsc
npm install -D @storybook/vue3-vite @storybook/addon-essentials
npm install vue
```

### โครงสร้างโปรเจกต์

```
my-vue-lib/
├── src/
│   ├── components/
│   │   ├── Button/
│   │   │   ├── Button.vue
│   │   │   ├── Button.test.ts
│   │   │   └── Button.stories.ts
│   │   ├── Input/
│   │   ├── Modal/
│   │   └── index.ts     # Re-export components
│   ├── composables/
│   │   ├── useTheme.ts
│   │   └── index.ts
│   ├── types/
│   │   └── index.ts
│   └── index.ts         # Main entry point
├── .storybook/
│   ├── main.ts
│   └── preview.ts
├── package.json
├── vite.config.ts
└── tsconfig.json
```

## 2. Vite Library Mode Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],
  
  build: {
    lib: {
      // Entry point
      entry: resolve(__dirname, 'src/index.ts'),
      
      // Library name (global variable name for UMD build)
      name: 'MyVueLib',
      
      // Output file names
      fileName: (format) => `my-vue-lib.${format}.js`
    },
    
    rollupOptions: {
      // ไม่ bundle dependencies ที่ user ควรมีอยู่แล้ว
      external: ['vue'],
      
      output: {
        // ใน UMD build, Vue จะเป็น global variable
        globals: {
          vue: 'Vue'
        },
        
        // CSS สำหรับแต่ละ component
        assetFileNames: (assetInfo) => {
          if (assetInfo.name === 'style.css') return 'my-vue-lib.css'
          return assetInfo.name!
        }
      }
    },
    
    // Copy type declarations
    copyPublicDir: false
  },
  
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src')
    }
  }
})
```

## 3. Main Entry Point

```typescript
// src/index.ts
// Components
export { default as Button } from './components/Button/Button.vue'
export { default as Input } from './components/Input/Input.vue'
export { default as Select } from './components/Select/Select.vue'
export { default as Modal } from './components/Modal/Modal.vue'
export { default as Dropdown } from './components/Dropdown/Dropdown.vue'
export { default as Badge } from './components/Badge/Badge.vue'
export { default as Card } from './components/Card/Card.vue'
export { default as Toast } from './components/Toast/Toast.vue'
export { default as Tooltip } from './components/Tooltip/Tooltip.vue'
export { default as DataTable } from './components/DataTable/DataTable.vue'
export { default as Pagination } from './components/Pagination/Pagination.vue'

// Composables
export { useToast } from './composables/useToast'
export { useModal } from './composables/useModal'
export { useTheme } from './composables/useTheme'

// Types
export type * from './types'

// Plugin (install all components globally)
import type { App } from 'vue'
import * as components from './components'

export const MyVueLib = {
  install(app: App) {
    Object.entries(components).forEach(([name, component]) => {
      app.component(name, component as any)
    })
  }
}

export default MyVueLib
```

## 4. TypeScript Types Export

```typescript
// src/types/index.ts

// Button types
export type ButtonVariant = 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger' | 'success'
export type ButtonSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl'

export interface ButtonProps {
  variant?: ButtonVariant
  size?: ButtonSize
  disabled?: boolean
  loading?: boolean
  fullWidth?: boolean
}

// Input types
export type InputType = 'text' | 'email' | 'password' | 'number' | 'tel' | 'url' | 'search'
export type InputSize = 'sm' | 'md' | 'lg'

export interface InputProps {
  modelValue?: string | number
  type?: InputType
  size?: InputSize
  label?: string
  placeholder?: string
  disabled?: boolean
  readonly?: boolean
  error?: string
  hint?: string
  required?: boolean
}

// Table types
export interface TableColumn<T = any> {
  key: string
  label: string
  sortable?: boolean
  width?: string
  align?: 'left' | 'center' | 'right'
  formatter?: (value: any, row: T) => string
  render?: (value: any, row: T) => any
}

export interface TableProps<T = any> {
  columns: TableColumn<T>[]
  data: T[]
  loading?: boolean
  sortable?: boolean
  selectable?: boolean
  pagination?: boolean
  pageSize?: number
  emptyText?: string
}

// Toast types
export type ToastType = 'success' | 'error' | 'warning' | 'info'

export interface ToastOptions {
  type?: ToastType
  title?: string
  message: string
  duration?: number
  dismissible?: boolean
  action?: {
    label: string
    onClick: () => void
  }
}

// Theme types
export interface ThemeConfig {
  primaryColor?: string
  borderRadius?: string
  fontFamily?: string
}
```

## 5. DataTable Component

```vue
<!-- src/components/DataTable/DataTable.vue -->
<template>
  <div class="data-table-wrapper">
    <!-- Toolbar -->
    <div v-if="$slots.toolbar || searchable" class="table-toolbar">
      <slot name="toolbar">
        <div class="table-search" v-if="searchable">
          <input
            v-model="searchQuery"
            type="search"
            placeholder="ค้นหา..."
            class="search-input"
          />
        </div>
      </slot>
    </div>
    
    <!-- Loading Overlay -->
    <div v-if="loading" class="table-loading">
      <div class="spinner"></div>
      <p>กำลังโหลดข้อมูล...</p>
    </div>
    
    <!-- Table -->
    <div class="table-container">
      <table class="data-table">
        <thead>
          <tr>
            <!-- Checkbox column -->
            <th v-if="selectable" class="col-checkbox">
              <input
                type="checkbox"
                :checked="isAllSelected"
                :indeterminate="isIndeterminate"
                @change="toggleSelectAll"
              />
            </th>
            
            <!-- Data columns -->
            <th
              v-for="col in columns"
              :key="col.key"
              :class="[
                `col-${col.key}`,
                col.align ? `align-${col.align}` : '',
                col.sortable ? 'sortable' : ''
              ]"
              :style="col.width ? { width: col.width } : {}"
              @click="col.sortable && toggleSort(col.key)"
            >
              {{ col.label }}
              <span v-if="col.sortable" class="sort-icon">
                {{ sortKey === col.key ? (sortDesc ? '↓' : '↑') : '↕' }}
              </span>
            </th>
            
            <!-- Actions column -->
            <th v-if="$slots.actions" class="col-actions">การดำเนินการ</th>
          </tr>
        </thead>
        
        <tbody>
          <!-- Empty state -->
          <tr v-if="sortedData.length === 0">
            <td :colspan="totalColumns" class="empty-cell">
              <slot name="empty">
                <div class="empty-state">
                  <p>{{ emptyText || 'ไม่มีข้อมูล' }}</p>
                </div>
              </slot>
            </td>
          </tr>
          
          <!-- Data rows -->
          <tr
            v-for="(row, rowIndex) in paginatedData"
            :key="getRowKey(row, rowIndex)"
            :class="{ 'selected': isRowSelected(row) }"
            @click="handleRowClick(row)"
          >
            <!-- Checkbox -->
            <td v-if="selectable" class="col-checkbox">
              <input
                type="checkbox"
                :checked="isRowSelected(row)"
                @change="toggleRowSelection(row)"
                @click.stop
              />
            </td>
            
            <!-- Data cells -->
            <td
              v-for="col in columns"
              :key="col.key"
              :class="col.align ? `align-${col.align}` : ''"
            >
              <slot :name="`cell-${col.key}`" :value="getNestedValue(row, col.key)" :row="row">
                {{ col.formatter
                    ? col.formatter(getNestedValue(row, col.key), row)
                    : getNestedValue(row, col.key) }}
              </slot>
            </td>
            
            <!-- Actions -->
            <td v-if="$slots.actions" class="col-actions">
              <slot name="actions" :row="row" :index="rowIndex" />
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    
    <!-- Pagination -->
    <div v-if="pagination && totalPages > 1" class="table-footer">
      <div class="pagination-info">
        แสดง {{ startRecord }} - {{ endRecord }} จาก {{ totalRecords }} รายการ
      </div>
      
      <div class="pagination-controls">
        <button
          @click="currentPage = 1"
          :disabled="currentPage === 1"
          class="page-btn"
        >«</button>
        
        <button
          @click="currentPage--"
          :disabled="currentPage === 1"
          class="page-btn"
        >‹</button>
        
        <button
          v-for="page in visiblePages"
          :key="page"
          @click="currentPage = page"
          :class="['page-btn', { active: page === currentPage }]"
        >{{ page }}</button>
        
        <button
          @click="currentPage++"
          :disabled="currentPage === totalPages"
          class="page-btn"
        >›</button>
        
        <button
          @click="currentPage = totalPages"
          :disabled="currentPage === totalPages"
          class="page-btn"
        >»</button>
      </div>
    </div>
    
    <!-- Selection Info -->
    <div v-if="selectedRows.length > 0" class="selection-bar">
      <span>เลือก {{ selectedRows.length }} รายการ</span>
      <slot name="bulk-actions" :selected="selectedRows">
        <button @click="clearSelection" class="btn-clear-selection">
          ยกเลิกการเลือก
        </button>
      </slot>
    </div>
  </div>
</template>

<script setup lang="ts" generic="T extends Record<string, any>">
import { ref, computed, watch } from 'vue'
import type { TableColumn } from '../../types'

interface Props {
  columns: TableColumn<T>[]
  data: T[]
  loading?: boolean
  sortable?: boolean
  selectable?: boolean
  pagination?: boolean
  pageSize?: number
  emptyText?: string
  searchable?: boolean
  rowKey?: string
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  sortable: false,
  selectable: false,
  pagination: true,
  pageSize: 10,
  searchable: false,
  rowKey: 'id'
})

const emit = defineEmits<{
  'row-click': [row: T]
  'selection-change': [rows: T[]]
  'sort-change': [key: string, desc: boolean]
}>()

// State
const sortKey = ref('')
const sortDesc = ref(false)
const currentPage = ref(1)
const selectedRows = ref<T[]>([])
const searchQuery = ref('')

// Sort
const toggleSort = (key: string) => {
  if (sortKey.value === key) {
    sortDesc.value = !sortDesc.value
  } else {
    sortKey.value = key
    sortDesc.value = false
  }
  emit('sort-change', sortKey.value, sortDesc.value)
}

const sortedData = computed(() => {
  let result = [...props.data]
  
  if (searchQuery.value) {
    const q = searchQuery.value.toLowerCase()
    result = result.filter(row =>
      Object.values(row).some(val =>
        String(val).toLowerCase().includes(q)
      )
    )
  }
  
  if (!sortKey.value) return result
  
  return result.sort((a, b) => {
    const aVal = getNestedValue(a, sortKey.value)
    const bVal = getNestedValue(b, sortKey.value)
    
    if (aVal === bVal) return 0
    const result = aVal < bVal ? -1 : 1
    return sortDesc.value ? -result : result
  })
})

// Pagination
const totalRecords = computed(() => sortedData.value.length)
const totalPages = computed(() => Math.ceil(totalRecords.value / props.pageSize))
const startRecord = computed(() => (currentPage.value - 1) * props.pageSize + 1)
const endRecord = computed(() => Math.min(currentPage.value * props.pageSize, totalRecords.value))

const paginatedData = computed(() => {
  if (!props.pagination) return sortedData.value
  const start = (currentPage.value - 1) * props.pageSize
  return sortedData.value.slice(start, start + props.pageSize)
})

const visiblePages = computed(() => {
  const range = 5
  const start = Math.max(1, currentPage.value - Math.floor(range / 2))
  const end = Math.min(totalPages.value, start + range - 1)
  return Array.from({ length: end - start + 1 }, (_, i) => start + i)
})

// Selection
const isAllSelected = computed(() =>
  paginatedData.value.length > 0 &&
  paginatedData.value.every(row => isRowSelected(row))
)

const isIndeterminate = computed(() =>
  selectedRows.value.length > 0 && !isAllSelected.value
)

const isRowSelected = (row: T) =>
  selectedRows.value.some(r => r[props.rowKey] === row[props.rowKey])

const toggleSelectAll = () => {
  if (isAllSelected.value) {
    selectedRows.value = selectedRows.value.filter(
      row => !paginatedData.value.some(r => r[props.rowKey] === row[props.rowKey])
    )
  } else {
    selectedRows.value = [...selectedRows.value, ...paginatedData.value.filter(row => !isRowSelected(row))]
  }
  emit('selection-change', selectedRows.value)
}

const toggleRowSelection = (row: T) => {
  if (isRowSelected(row)) {
    selectedRows.value = selectedRows.value.filter(r => r[props.rowKey] !== row[props.rowKey])
  } else {
    selectedRows.value = [...selectedRows.value, row]
  }
  emit('selection-change', selectedRows.value)
}

const clearSelection = () => {
  selectedRows.value = []
  emit('selection-change', [])
}

// Helpers
const totalColumns = computed(() =>
  props.columns.length +
  (props.selectable ? 1 : 0) +
  (true ? 1 : 0)
)

const getRowKey = (row: T, index: number) => row[props.rowKey] ?? index

const getNestedValue = (obj: any, path: string) => {
  return path.split('.').reduce((acc, key) => acc?.[key], obj)
}

const handleRowClick = (row: T) => {
  emit('row-click', row)
}

watch(() => props.data, () => {
  currentPage.value = 1
})
</script>
```

## 6. Storybook Integration

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/vue3-vite'

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(js|jsx|ts|tsx)'],
  addons: [
    '@storybook/addon-links',
    '@storybook/addon-essentials',
    '@storybook/addon-a11y'
  ],
  framework: {
    name: '@storybook/vue3-vite',
    options: {}
  }
}

export default config
```

```typescript
// src/components/Button/Button.stories.ts
import type { Meta, StoryObj } from '@storybook/vue3'
import Button from './Button.vue'

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'outline', 'ghost', 'danger', 'success']
    },
    size: {
      control: 'select',
      options: ['xs', 'sm', 'md', 'lg', 'xl']
    },
    disabled: { control: 'boolean' },
    loading: { control: 'boolean' },
    fullWidth: { control: 'boolean' }
  }
}

export default meta
type Story = StoryObj<typeof Button>

export const Primary: Story = {
  args: {
    variant: 'primary',
    size: 'md',
    label: 'Primary Button'
  }
}

export const AllVariants: Story = {
  render: () => ({
    components: { Button },
    template: `
      <div style="display: flex; gap: 12px; flex-wrap: wrap; align-items: center">
        <Button variant="primary">Primary</Button>
        <Button variant="secondary">Secondary</Button>
        <Button variant="outline">Outline</Button>
        <Button variant="ghost">Ghost</Button>
        <Button variant="danger">Danger</Button>
        <Button variant="success">Success</Button>
      </div>
    `
  })
}

export const Loading: Story = {
  args: {
    variant: 'primary',
    loading: true,
    label: 'Loading...'
  }
}
```

## 7. npm Publishing

```json
// package.json
{
  "name": "@myorg/vue-ui",
  "version": "1.0.0",
  "description": "Vue.js Component Library",
  "type": "module",
  "main": "./dist/my-vue-lib.umd.js",
  "module": "./dist/my-vue-lib.es.js",
  "exports": {
    ".": {
      "import": "./dist/my-vue-lib.es.js",
      "require": "./dist/my-vue-lib.umd.js"
    },
    "./styles": "./dist/my-vue-lib.css"
  },
  "types": "./dist/types/index.d.ts",
  "files": [
    "dist"
  ],
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc && vite build",
    "build:types": "vue-tsc --declaration --emitDeclarationOnly --outDir dist/types",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build",
    "test": "vitest",
    "lint": "eslint src --ext .vue,.ts",
    "release": "npm run build && npm publish --access public"
  },
  "peerDependencies": {
    "vue": "^3.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@vitejs/plugin-vue": "^5.0.0",
    "@vue/test-utils": "^2.0.0",
    "typescript": "^5.0.0",
    "vite": "^5.0.0",
    "vitest": "^1.0.0",
    "vue": "^3.4.0",
    "vue-tsc": "^2.0.0"
  },
  "keywords": ["vue", "vue3", "components", "ui", "design-system"],
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/myorg/vue-ui"
  },
  "publishConfig": {
    "registry": "https://registry.npmjs.org/"
  }
}
```

```bash
# สร้าง .npmrc เพื่อ publish ไปยัง npm
echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc

# Build
npm run build

# Publish
npm publish --access public

# สำหรับ scoped package ที่ต้องการ public access
npm publish --access public
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Vite Library Mode** - ตั้งค่าสำหรับ build library
2. **Package.json Config** - ตั้งค่า exports, types
3. **TypeScript Types** - Export types สำหรับ users
4. **DataTable Component** - Component ที่ซับซ้อนพร้อมฟีเจอร์ครบ
5. **Storybook** - Documentation และ interactive demo
6. **npm Publishing** - เผยแพร่ package ไปยัง npm
7. **Versioning** - การจัดการ version อย่างถูกต้อง
