# Part 10: Props และ Emits

## บทนำ

Props และ Emits เป็นกลไกหลักในการสื่อสารระหว่าง components ใน Vue.js Props ใช้ส่งข้อมูลจาก parent ลงไปยัง child (one-way data flow) ในขณะที่ Emits ใช้ส่ง events จาก child กลับขึ้นไปยัง parent การออกแบบ props และ emits ที่ดีจะทำให้ components ของเรา reusable และ maintainable มากขึ้น

---

## 1. defineProps() พื้นฐาน

```vue
<!-- ChildComponent.vue -->
<script setup lang="ts">
// วิธีที่ 1: Runtime declaration (ไม่มี TypeScript type checking)
const props1 = defineProps(['title', 'count', 'items'])

// วิธีที่ 2: Object declaration พร้อม type
const props2 = defineProps({
  title: String,
  count: Number,
  items: Array,
  callback: Function,
})

// วิธีที่ 3: TypeScript generic (แนะนำ)
const props = defineProps<{
  title: string
  count?: number
  items: string[]
}>()

// เข้าถึง props
console.log(props.title)
console.log(props.count)
console.log(props.items)
</script>

<template>
  <!-- ใช้ props ใน template โดยตรง (ไม่ต้องเขียน props.) -->
  <h2>{{ title }}</h2>
  <p>จำนวน: {{ count }}</p>
  <ul>
    <li v-for="item in items" :key="item">{{ item }}</li>
  </ul>
</template>
```

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import ChildComponent from './ChildComponent.vue'

const title = ref('รายการสินค้า')
const count = ref(5)
const items = ref(['Vue.js', 'React', 'Angular'])
</script>

<template>
  <!-- ส่ง props -->
  <ChildComponent
    :title="title"
    :count="count"
    :items="items"
  />

  <!-- Static props (ไม่ต้องใช้ :) -->
  <ChildComponent title="ชื่อคงที่" :items="['a', 'b']" />

  <!-- Props ทุกตัว -->
  <!-- Boolean props: ถ้าแค่ชื่อ = true -->
  <ChildComponent title="test" :items="[]" />
</template>
```

---

## 2. Props Validation (type, required, default, validator)

```vue
<script setup lang="ts">
import { PropType } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

type Status = 'active' | 'inactive' | 'pending'

// ใช้ withDefaults กับ TypeScript
const props = withDefaults(
  defineProps<{
    // Required props
    userId: number
    title: string

    // Optional props
    subtitle?: string
    status?: Status
    user?: User
    tags?: string[]
    maxItems?: number
    isVisible?: boolean

    // Complex types
    config?: {
      theme: 'light' | 'dark'
      size: number
    }
  }>(),
  {
    // Default values
    subtitle: '',
    status: 'active',
    tags: () => [],  // Arrays/Objects ต้องใช้ factory function
    maxItems: 10,
    isVisible: true,
    config: () => ({ theme: 'light', size: 16 }),
  }
)
</script>
```

### Runtime Props Validation

```vue
<script setup lang="ts">
import { PropType } from 'vue'

// Runtime declaration กับ full validation
const props = defineProps({
  // Basic types
  title: {
    type: String,
    required: true,
  },

  count: {
    type: Number,
    default: 0,
    validator(value: number) {
      return value >= 0  // ต้องไม่ติดลบ
    },
  },

  status: {
    type: String as PropType<'active' | 'inactive'>,
    default: 'active',
    validator(value: string) {
      return ['active', 'inactive'].includes(value)
    },
  },

  // Array of specific type
  items: {
    type: Array as PropType<{ id: number; name: string }[]>,
    default: () => [],
    validator(value: any[]) {
      return value.every(item => item.id && item.name)
    },
  },

  // Multiple types
  id: {
    type: [String, Number],
    required: true,
  },

  // Object with shape validation
  user: {
    type: Object as PropType<{ name: string; email: string }>,
    default: null,
    validator(value: any) {
      if (!value) return true
      return value.name && value.email
    },
  },

  // Function prop
  onClick: {
    type: Function as PropType<(id: number) => void>,
    default: null,
  },
})
</script>
```

---

## 3. Props กับ TypeScript Interfaces

```typescript
// types/product.ts
export interface Product {
  id: number
  name: string
  description: string
  price: number
  originalPrice?: number
  images: string[]
  category: string
  tags: string[]
  rating: number
  reviewCount: number
  stock: number
  isNew?: boolean
  isFeatured?: boolean
}

export interface CartItem {
  product: Product
  quantity: number
  selectedVariant?: string
}

export interface PaginationConfig {
  page: number
  pageSize: number
  total: number
}

export type SortDirection = 'asc' | 'desc'

export interface SortConfig {
  field: string
  direction: SortDirection
}
```

```vue
<!-- ProductCard.vue -->
<script setup lang="ts">
import type { Product } from '@/types/product'

const props = defineProps<{
  product: Product
  showActions?: boolean
  compact?: boolean
  selected?: boolean
}>()

const emit = defineEmits<{
  'add-to-cart': [product: Product, quantity: number]
  'toggle-favorite': [productId: number]
  'view-details': [product: Product]
}>()

const discount = computed(() => {
  if (!props.product.originalPrice) return null
  const pct = Math.round((1 - props.product.price / props.product.originalPrice) * 100)
  return pct > 0 ? pct : null
})

function handleAddToCart() {
  emit('add-to-cart', props.product, 1)
}
</script>

<template>
  <div
    class="product-card"
    :class="{ 'is-compact': compact, 'is-selected': selected }"
    @click="emit('view-details', product)"
  >
    <div class="product-image">
      <img :src="product.images[0]" :alt="product.name" />
      <span v-if="product.isNew" class="badge-new">ใหม่</span>
      <span v-if="discount" class="badge-discount">-{{ discount }}%</span>
    </div>

    <div class="product-info">
      <h3 class="product-name">{{ product.name }}</h3>
      <p v-if="!compact" class="product-description">{{ product.description }}</p>

      <div class="product-price">
        <span class="price">฿{{ product.price.toLocaleString() }}</span>
        <span v-if="product.originalPrice" class="original-price">
          ฿{{ product.originalPrice.toLocaleString() }}
        </span>
      </div>

      <div class="product-rating">
        <span>★ {{ product.rating.toFixed(1) }}</span>
        <span>({{ product.reviewCount }})</span>
      </div>
    </div>

    <div v-if="showActions" class="product-actions">
      <button
        @click.stop="handleAddToCart"
        :disabled="product.stock === 0"
        class="btn-add-cart"
      >
        {{ product.stock > 0 ? 'เพิ่มในตะกร้า' : 'สินค้าหมด' }}
      </button>
      <button @click.stop="emit('toggle-favorite', product.id)" class="btn-favorite">
        ♡
      </button>
    </div>
  </div>
</template>
```

---

## 4. defineEmits() พื้นฐาน

```vue
<script setup lang="ts">
// วิธีที่ 1: Array syntax (simple)
const emit1 = defineEmits(['click', 'change', 'close'])

// วิธีที่ 2: Object syntax (กับ validation)
const emit2 = defineEmits({
  click: (event: MouseEvent) => event instanceof MouseEvent,
  change: (value: string) => typeof value === 'string',
  close: null, // null = no validation
})

// วิธีที่ 3: TypeScript generic (แนะนำ)
const emit = defineEmits<{
  click: [event: MouseEvent]
  change: [value: string, oldValue: string]
  close: []
  submit: [data: { name: string; email: string }]
  'update:modelValue': [value: string]
}>()

// ใช้ emit
function handleClick(event: MouseEvent) {
  emit('click', event)
}

function handleChange(newVal: string, oldVal: string) {
  emit('change', newVal, oldVal)
}
</script>
```

---

## 5. Emits กับ TypeScript

```vue
<!-- TypedEventComponent.vue -->
<script setup lang="ts">
import type { Product, CartItem } from '@/types/product'

// Types สำหรับ events
interface FilterChangeEvent {
  category: string | null
  priceRange: [number, number]
  sortBy: string
  sortOrder: 'asc' | 'desc'
}

interface BulkActionEvent {
  action: 'delete' | 'export' | 'archive'
  ids: number[]
}

const emit = defineEmits<{
  // Simple events
  close: []
  refresh: []

  // Events กับ payload
  'product-selected': [product: Product]
  'cart-updated': [items: CartItem[]]
  'filter-change': [filter: FilterChangeEvent]
  'bulk-action': [event: BulkActionEvent]

  // v-model events
  'update:modelValue': [value: string]
  'update:page': [page: number]
}>()

// ใช้ emit อย่าง type-safe
function selectProduct(product: Product) {
  // TypeScript จะ error ถ้าส่ง type ผิด
  emit('product-selected', product)
}

function handleBulkDelete(ids: number[]) {
  emit('bulk-action', { action: 'delete', ids })
}
</script>
```

---

## 6. v-model กับ Component

```vue
<!-- SearchInput.vue - Component กับ v-model -->
<script setup lang="ts">
interface Props {
  modelValue: string
  placeholder?: string
  debounce?: number
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: 'ค้นหา...',
  debounce: 300,
})

const emit = defineEmits<{
  'update:modelValue': [value: string]
  'search': [value: string]
  'clear': []
}>()

let debounceTimer: ReturnType<typeof setTimeout>

function handleInput(event: Event) {
  const value = (event.target as HTMLInputElement).value
  emit('update:modelValue', value)

  // Debounced search
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    emit('search', value)
  }, props.debounce)
}

function handleClear() {
  emit('update:modelValue', '')
  emit('clear')
}
</script>

<template>
  <div class="search-input">
    <input
      :value="modelValue"
      :placeholder="placeholder"
      @input="handleInput"
      @keydown.esc="handleClear"
    />
    <button v-if="modelValue" @click="handleClear">✕</button>
  </div>
</template>
```

```vue
<!-- MultiField.vue - หลาย v-model -->
<script setup lang="ts">
const props = defineProps<{
  firstName: string
  lastName: string
  email: string
}>()

const emit = defineEmits<{
  'update:firstName': [value: string]
  'update:lastName': [value: string]
  'update:email': [value: string]
}>()
</script>

<template>
  <div class="multi-field">
    <input
      :value="firstName"
      @input="emit('update:firstName', ($event.target as HTMLInputElement).value)"
      placeholder="ชื่อ"
    />
    <input
      :value="lastName"
      @input="emit('update:lastName', ($event.target as HTMLInputElement).value)"
      placeholder="นามสกุล"
    />
    <input
      :value="email"
      @input="emit('update:email', ($event.target as HTMLInputElement).value)"
      placeholder="อีเมล"
    />
  </div>
</template>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const searchQuery = ref('')
const firstName = ref('')
const lastName = ref('')
const email = ref('')
</script>

<template>
  <!-- v-model เดียว -->
  <SearchInput
    v-model="searchQuery"
    @search="handleSearch"
  />

  <!-- หลาย v-model -->
  <MultiField
    v-model:firstName="firstName"
    v-model:lastName="lastName"
    v-model:email="email"
  />
</template>
```

---

## 7. Prop Drilling Problem

Prop Drilling เกิดขึ้นเมื่อต้องส่ง props ผ่านหลายชั้นของ components ที่ไม่ได้ใช้ props นั้นจริงๆ

```
App → Layout → Page → Section → Container → FinalChild
         ↑ ส่ง userId ทุกชั้น ทั้งที่แค่ FinalChild ต้องใช้
```

### ปัญหา

```vue
<!-- ❌ Prop Drilling - ต้องส่ง props ผ่านทุกชั้น -->
<!-- App.vue -->
<Layout :user-id="userId" :theme="theme" />

<!-- Layout.vue - ไม่ได้ใช้เอง แต่ต้องส่งต่อ -->
<Page :user-id="userId" :theme="theme" />

<!-- Page.vue - ไม่ได้ใช้เอง แต่ต้องส่งต่อ -->
<Section :user-id="userId" :theme="theme" />

<!-- Section.vue - ใช้จริง -->
<UserProfile :user-id="userId" :theme="theme" />
```

### แก้ปัญหาด้วย provide/inject

```vue
<!-- App.vue - provide ข้อมูล -->
<script setup lang="ts">
import { provide, ref, readonly } from 'vue'

const userId = ref(1)
const theme = ref('light')

// provide ค่าให้ทุก descendants
provide('userId', readonly(userId))
provide('theme', theme)

// provide function สำหรับ update
provide('setTheme', (newTheme: string) => {
  theme.value = newTheme
})
</script>
```

```vue
<!-- FinalChild.vue - inject ข้อมูล โดยไม่ต้อง prop drilling -->
<script setup lang="ts">
import { inject, ref } from 'vue'

// inject ด้วย type และ default value
const userId = inject<number>('userId', 0)
const theme = inject<string>('theme', 'light')
const setTheme = inject<(theme: string) => void>('setTheme', () => {})
</script>
```

### แก้ปัญหาด้วย Composables

```typescript
// composables/useCurrentUser.ts
import { ref, readonly } from 'vue'

const currentUser = ref(null)
const isLoading = ref(false)

export function useCurrentUser() {
  async function fetchUser(id: number) {
    isLoading.value = true
    // fetch user...
    isLoading.value = false
  }

  return {
    currentUser: readonly(currentUser),
    isLoading: readonly(isLoading),
    fetchUser,
  }
}
```

---

## 8. ตัวอย่าง Real-world: DataTable Component พร้อม sort/filter events

```vue
<!-- components/DataTable.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

// Types
export interface Column<T = any> {
  key: string
  label: string
  sortable?: boolean
  filterable?: boolean
  width?: string
  align?: 'left' | 'center' | 'right'
  formatter?: (value: any, row: T) => string
  render?: (row: T) => string
}

export interface SortEvent {
  key: string
  order: 'asc' | 'desc'
}

export interface FilterEvent {
  key: string
  value: string
}

export interface PageChangeEvent {
  page: number
  pageSize: number
}

export interface RowActionEvent<T> {
  action: string
  row: T
}

export interface SelectionChangeEvent<T> {
  selectedRows: T[]
  isAllSelected: boolean
}

interface Props<T extends Record<string, any> = Record<string, any>> {
  data: T[]
  columns: Column<T>[]
  loading?: boolean
  selectable?: boolean
  rowKey?: string
  emptyText?: string

  // Pagination
  pagination?: {
    page: number
    pageSize: number
    total: number
    pageSizes?: number[]
  }

  // Current sort
  sortConfig?: SortEvent

  // Row actions
  rowActions?: Array<{
    key: string
    label: string
    icon?: string
    danger?: boolean
    disabled?: (row: T) => boolean
    hidden?: (row: T) => boolean
  }>
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  selectable: false,
  rowKey: 'id',
  emptyText: 'ไม่มีข้อมูล',
})

const emit = defineEmits<{
  // Sort event
  'sort-change': [event: SortEvent]

  // Filter event
  'filter-change': [event: FilterEvent]

  // Pagination events
  'page-change': [event: PageChangeEvent]
  'page-size-change': [pageSize: number]

  // Row events
  'row-click': [row: any, index: number]
  'row-action': [event: RowActionEvent<any>]

  // Selection events
  'selection-change': [event: SelectionChangeEvent<any>]

  // Header events
  'column-resize': [{ key: string; width: number }]
}>()

// Internal state
const selectedKeys = ref(new Set<string | number>())
const columnFilters = ref<Record<string, string>>({})
const hoveredRowIndex = ref<number | null>(null)

// Computed
const rowKeyFn = (row: any) => row[props.rowKey]

const selectedRows = computed(() =>
  props.data.filter(row => selectedKeys.value.has(rowKeyFn(row)))
)

const isAllSelected = computed(() =>
  props.data.length > 0 && props.data.every(row => selectedKeys.value.has(rowKeyFn(row)))
)

const isIndeterminate = computed(() =>
  selectedKeys.value.size > 0 && !isAllSelected.value
)

// Sort
function handleSort(column: Column) {
  if (!column.sortable) return
  const currentSort = props.sortConfig
  let order: 'asc' | 'desc' = 'asc'
  if (currentSort?.key === column.key) {
    order = currentSort.order === 'asc' ? 'desc' : 'asc'
  }
  emit('sort-change', { key: column.key, order })
}

function getSortIcon(columnKey: string): string {
  if (props.sortConfig?.key !== columnKey) return '↕'
  return props.sortConfig.order === 'asc' ? '↑' : '↓'
}

// Filter
function handleFilter(key: string, value: string) {
  columnFilters.value[key] = value
  emit('filter-change', { key, value })
}

// Selection
function toggleSelectAll() {
  if (isAllSelected.value) {
    props.data.forEach(row => selectedKeys.value.delete(rowKeyFn(row)))
  } else {
    props.data.forEach(row => selectedKeys.value.add(rowKeyFn(row)))
  }
  emitSelectionChange()
}

function toggleSelectRow(row: any) {
  const key = rowKeyFn(row)
  if (selectedKeys.value.has(key)) {
    selectedKeys.value.delete(key)
  } else {
    selectedKeys.value.add(key)
  }
  emitSelectionChange()
}

function emitSelectionChange() {
  emit('selection-change', {
    selectedRows: selectedRows.value,
    isAllSelected: isAllSelected.value,
  })
}

// Pagination
function handlePageChange(page: number) {
  if (!props.pagination) return
  emit('page-change', { page, pageSize: props.pagination.pageSize })
}

function handlePageSizeChange(pageSize: number) {
  emit('page-size-change', pageSize)
  handlePageChange(1)
}

// Row
function handleRowClick(row: any, index: number) {
  emit('row-click', row, index)
}

function handleRowAction(action: string, row: any) {
  emit('row-action', { action, row })
}

// Cell value
function getCellValue(row: any, column: Column): string {
  const value = row[column.key]
  if (column.formatter) {
    return column.formatter(value, row)
  }
  if (value === null || value === undefined) return '-'
  return String(value)
}

// Pagination pages
const totalPages = computed(() => {
  if (!props.pagination) return 0
  return Math.ceil(props.pagination.total / props.pagination.pageSize)
})

const pageNumbers = computed(() => {
  if (!props.pagination) return []
  const total = totalPages.value
  const current = props.pagination.page
  const pages: (number | '...')[] = []

  if (total <= 7) {
    for (let i = 1; i <= total; i++) pages.push(i)
  } else {
    pages.push(1)
    if (current > 3) pages.push('...')
    for (let i = Math.max(2, current - 1); i <= Math.min(total - 1, current + 1); i++) {
      pages.push(i)
    }
    if (current < total - 2) pages.push('...')
    pages.push(total)
  }

  return pages
})
</script>

<template>
  <div class="data-table">
    <!-- Selection info bar -->
    <div v-if="selectable && selectedKeys.size > 0" class="selection-bar">
      <span>เลือก {{ selectedKeys.size }} รายการ</span>
      <slot name="bulk-actions" :selected-rows="selectedRows" />
      <button @click="selectedKeys.clear(); emitSelectionChange()" class="btn-clear">
        ยกเลิกการเลือก
      </button>
    </div>

    <!-- Table -->
    <div class="table-container">
      <table>
        <thead>
          <tr>
            <!-- Select all checkbox -->
            <th v-if="selectable" class="col-select">
              <input
                type="checkbox"
                :checked="isAllSelected"
                :indeterminate="isIndeterminate"
                @change="toggleSelectAll"
              />
            </th>

            <!-- Column headers -->
            <th
              v-for="column in columns"
              :key="column.key"
              :style="{ width: column.width, textAlign: column.align || 'left' }"
              :class="{ 'is-sortable': column.sortable, 'is-sorted': sortConfig?.key === column.key }"
              @click="handleSort(column)"
            >
              <div class="th-content">
                <span>{{ column.label }}</span>
                <span v-if="column.sortable" class="sort-icon">
                  {{ getSortIcon(column.key) }}
                </span>
              </div>

              <!-- Column filter -->
              <div v-if="column.filterable" class="column-filter" @click.stop>
                <input
                  :value="columnFilters[column.key] || ''"
                  @input="handleFilter(column.key, ($event.target as HTMLInputElement).value)"
                  placeholder="กรอง..."
                  class="filter-input"
                />
              </div>
            </th>

            <!-- Actions column -->
            <th v-if="rowActions && rowActions.length > 0" class="col-actions">
              Actions
            </th>
          </tr>
        </thead>

        <tbody>
          <!-- Loading state -->
          <template v-if="loading">
            <tr v-for="i in 5" :key="i" class="skeleton-row">
              <td v-if="selectable">
                <div class="skeleton-cell"></div>
              </td>
              <td v-for="col in columns" :key="col.key">
                <div class="skeleton-cell"></div>
              </td>
              <td v-if="rowActions?.length">
                <div class="skeleton-cell"></div>
              </td>
            </tr>
          </template>

          <!-- Empty state -->
          <tr v-else-if="data.length === 0">
            <td
              :colspan="(selectable ? 1 : 0) + columns.length + (rowActions?.length ? 1 : 0)"
              class="empty-cell"
            >
              <slot name="empty">
                <div class="empty-state">
                  <span class="empty-icon">📭</span>
                  <p>{{ emptyText }}</p>
                </div>
              </slot>
            </td>
          </tr>

          <!-- Data rows -->
          <template v-else>
            <tr
              v-for="(row, index) in data"
              :key="rowKeyFn(row)"
              :class="{
                'is-selected': selectedKeys.has(rowKeyFn(row)),
                'is-hovered': hoveredRowIndex === index,
              }"
              @click="handleRowClick(row, index)"
              @mouseenter="hoveredRowIndex = index"
              @mouseleave="hoveredRowIndex = null"
            >
              <!-- Select checkbox -->
              <td v-if="selectable" @click.stop>
                <input
                  type="checkbox"
                  :checked="selectedKeys.has(rowKeyFn(row))"
                  @change="toggleSelectRow(row)"
                />
              </td>

              <!-- Data cells -->
              <td
                v-for="column in columns"
                :key="column.key"
                :style="{ textAlign: column.align || 'left' }"
              >
                <slot :name="`cell-${column.key}`" :row="row" :value="row[column.key]">
                  {{ getCellValue(row, column) }}
                </slot>
              </td>

              <!-- Row actions -->
              <td v-if="rowActions && rowActions.length > 0" class="actions-cell" @click.stop>
                <template v-for="action in rowActions" :key="action.key">
                  <button
                    v-if="!action.hidden?.(row)"
                    :disabled="action.disabled?.(row)"
                    :class="['action-btn', { 'action-btn--danger': action.danger }]"
                    @click="handleRowAction(action.key, row)"
                  >
                    <span v-if="action.icon">{{ action.icon }}</span>
                    {{ action.label }}
                  </button>
                </template>
              </td>
            </tr>
          </template>
        </tbody>
      </table>
    </div>

    <!-- Pagination -->
    <div v-if="pagination" class="pagination">
      <div class="pagination-info">
        แสดง
        {{ (pagination.page - 1) * pagination.pageSize + 1 }}-{{ Math.min(pagination.page * pagination.pageSize, pagination.total) }}
        จาก {{ pagination.total }} รายการ
      </div>

      <div class="pagination-controls">
        <select :value="pagination.pageSize" @change="handlePageSizeChange(Number(($event.target as HTMLSelectElement).value))">
          <option v-for="size in (pagination.pageSizes || [10, 25, 50, 100])" :key="size" :value="size">
            {{ size }} / หน้า
          </option>
        </select>

        <button
          @click="handlePageChange(1)"
          :disabled="pagination.page === 1"
        >«</button>
        <button
          @click="handlePageChange(pagination.page - 1)"
          :disabled="pagination.page === 1"
        >‹</button>

        <template v-for="page in pageNumbers" :key="page">
          <button
            v-if="page !== '...'"
            @click="handlePageChange(page as number)"
            :class="{ active: pagination.page === page }"
          >{{ page }}</button>
          <span v-else>...</span>
        </template>

        <button
          @click="handlePageChange(pagination.page + 1)"
          :disabled="pagination.page === totalPages"
        >›</button>
        <button
          @click="handlePageChange(totalPages)"
          :disabled="pagination.page === totalPages"
        >»</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.data-table { width: 100%; }
.selection-bar { display: flex; align-items: center; gap: 1rem; padding: 0.75rem 1rem; background: #eff6ff; border-radius: 8px 8px 0 0; border: 1px solid #bfdbfe; }
.table-container { overflow-x: auto; border: 1px solid #e5e7eb; border-radius: 8px; }
table { width: 100%; border-collapse: collapse; }
th { background: #f9fafb; padding: 0.75rem 1rem; font-weight: 600; border-bottom: 2px solid #e5e7eb; white-space: nowrap; }
td { padding: 0.75rem 1rem; border-bottom: 1px solid #f3f4f6; }
tr:hover td { background: #f9fafb; }
tr.is-selected td { background: #eff6ff; }
th.is-sortable { cursor: pointer; user-select: none; }
th.is-sortable:hover { background: #f3f4f6; }
.th-content { display: flex; align-items: center; gap: 0.5rem; }
.filter-input { width: 100%; padding: 0.25rem 0.5rem; border: 1px solid #d1d5db; border-radius: 4px; font-size: 0.75rem; margin-top: 0.5rem; }
.empty-cell { text-align: center; padding: 3rem; }
.empty-state { display: flex; flex-direction: column; align-items: center; gap: 0.5rem; color: #9ca3af; }
.empty-icon { font-size: 2rem; }
.skeleton-cell { height: 20px; background: #e5e7eb; border-radius: 4px; animation: pulse 1.5s ease-in-out infinite; }
@keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }
.actions-cell { white-space: nowrap; }
.action-btn { padding: 0.25rem 0.75rem; border: 1px solid #d1d5db; background: white; border-radius: 4px; cursor: pointer; font-size: 0.75rem; margin-right: 0.25rem; }
.action-btn:hover { background: #f3f4f6; }
.action-btn--danger { color: #dc2626; border-color: #fca5a5; }
.action-btn--danger:hover { background: #fef2f2; }
.action-btn:disabled { opacity: 0.5; cursor: not-allowed; }
.pagination { display: flex; justify-content: space-between; align-items: center; padding: 1rem; flex-wrap: wrap; gap: 1rem; }
.pagination-info { color: #6b7280; font-size: 0.875rem; }
.pagination-controls { display: flex; align-items: center; gap: 0.25rem; }
.pagination-controls button { padding: 0.375rem 0.75rem; border: 1px solid #e5e7eb; background: white; border-radius: 4px; cursor: pointer; }
.pagination-controls button:hover:not(:disabled) { background: #f3f4f6; }
.pagination-controls button.active { background: #3b82f6; color: white; border-color: #3b82f6; }
.pagination-controls button:disabled { opacity: 0.5; cursor: not-allowed; }
</style>
```

### การใช้งาน DataTable Component

```vue
<!-- EmployeeList.vue - ตัวอย่างการใช้ DataTable -->
<script setup lang="ts">
import { ref, reactive } from 'vue'
import DataTable, { type Column, type SortEvent } from './components/DataTable.vue'

interface Employee {
  id: number
  name: string
  email: string
  department: string
  salary: number
  status: 'active' | 'inactive'
}

const employees = ref<Employee[]>([
  { id: 1, name: 'สมชาย ใจดี', email: 'somchai@co.th', department: 'Engineering', salary: 80000, status: 'active' },
  { id: 2, name: 'สมหญิง รักดี', email: 'somying@co.th', department: 'Marketing', salary: 65000, status: 'active' },
  { id: 3, name: 'สมศรี ดีใจ', email: 'somsri@co.th', department: 'HR', salary: 55000, status: 'inactive' },
])

const tableState = reactive({
  loading: false,
  sort: { key: 'id', order: 'asc' as const },
  pagination: {
    page: 1,
    pageSize: 10,
    total: employees.value.length,
    pageSizes: [10, 25, 50],
  },
})

const columns: Column<Employee>[] = [
  { key: 'id', label: 'ID', sortable: true, width: '80px' },
  { key: 'name', label: 'ชื่อ', sortable: true, filterable: true },
  { key: 'email', label: 'อีเมล', sortable: true },
  {
    key: 'department',
    label: 'แผนก',
    sortable: true,
    filterable: true,
  },
  {
    key: 'salary',
    label: 'เงินเดือน',
    sortable: true,
    align: 'right',
    formatter: (value) => `฿${value.toLocaleString()}`,
  },
  { key: 'status', label: 'สถานะ', sortable: true },
]

const rowActions = [
  { key: 'edit', label: 'แก้ไข', icon: '✏️' },
  { key: 'delete', label: 'ลบ', icon: '🗑️', danger: true },
]

function handleSortChange(event: SortEvent) {
  tableState.sort = event
  // fetch data with new sort
}

function handleRowAction({ action, row }: { action: string; row: Employee }) {
  if (action === 'edit') {
    console.log('Edit:', row)
  } else if (action === 'delete') {
    if (confirm(`ลบ ${row.name}?`)) {
      employees.value = employees.value.filter(e => e.id !== row.id)
    }
  }
}

function handleSelectionChange({ selectedRows }: { selectedRows: Employee[] }) {
  console.log('Selected:', selectedRows.length, 'rows')
}
</script>

<template>
  <div>
    <DataTable
      :data="employees"
      :columns="columns"
      :loading="tableState.loading"
      :selectable="true"
      :sort-config="tableState.sort"
      :pagination="tableState.pagination"
      :row-actions="rowActions"
      row-key="id"
      @sort-change="handleSortChange"
      @row-action="handleRowAction"
      @selection-change="handleSelectionChange"
      @page-change="(e) => tableState.pagination.page = e.page"
    >
      <!-- Custom cell render สำหรับ status -->
      <template #cell-status="{ value }">
        <span :class="['status-badge', value]">
          {{ value === 'active' ? 'ใช้งาน' : 'ไม่ใช้งาน' }}
        </span>
      </template>

      <!-- Bulk actions slot -->
      <template #bulk-actions="{ selectedRows }">
        <button @click="console.log('Export:', selectedRows)">
          Export ที่เลือก
        </button>
      </template>
    </DataTable>
  </div>
</template>

<style scoped>
.status-badge { padding: 0.25rem 0.75rem; border-radius: 9999px; font-size: 0.75rem; font-weight: 500; }
.status-badge.active { background: #dcfce7; color: #15803d; }
.status-badge.inactive { background: #f3f4f6; color: #6b7280; }
</style>
```

---

## สรุป

| Feature | การใช้งาน |
|---------|----------|
| `defineProps<T>()` | TypeScript-typed props |
| `withDefaults()` | กำหนด default values สำหรับ typed props |
| `defineEmits<T>()` | TypeScript-typed emits |
| Props validation | type, required, default, validator |
| v-model | `modelValue` prop + `update:modelValue` emit |
| Multiple v-model | `v-model:propName` syntax |
| provide/inject | แก้ปัญหา prop drilling |

**Best Practices:**
- ใช้ TypeScript generics กับ `defineProps` และ `defineEmits` เสมอ
- Props ควรเป็น read-only ใน child component (ห้าม mutate)
- ใช้ `withDefaults` แทนการใส่ default ใน code ที่อื่น
- ตั้งชื่อ events ด้วย kebab-case เช่น `update:model-value`
- ใช้ `provide/inject` หรือ composables แทน prop drilling
- Documentation props ด้วย JSDoc comments ใน projects ขนาดใหญ่
- ออกแบบ component interface (props + emits) ก่อนเขียน implementation
