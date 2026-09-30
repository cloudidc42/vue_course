# Part 31: TypeScript กับ Vue/Nuxt

## ทำไมต้องใช้ TypeScript กับ Vue 3?

TypeScript ช่วยให้โค้ดของเราปลอดภัยมากขึ้น ตรวจจับ error ได้ตั้งแต่ขณะพัฒนา และทำให้ IDE ให้ autocomplete ที่แม่นยำขึ้น Vue 3 ถูกออกแบบมาให้รองรับ TypeScript อย่างเต็มรูปแบบตั้งแต่เริ่มต้น

---

## 1. TypeScript Setup ใน Vue 3

### การสร้างโปรเจกต์ใหม่ด้วย TypeScript

```bash
# สร้างโปรเจกต์ Nuxt ใหม่ (TypeScript เปิดใช้งานอัตโนมัติ)
npx nuxi@latest init my-ts-app

# หรือสร้างโปรเจกต์ Vue ด้วย Vite
npm create vue@latest my-ts-app
# เลือก TypeScript: Yes
```

### การตั้งค่า tsconfig.json สำหรับ Vue 3

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ESNext",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "jsx": "preserve",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "lib": ["ESNext", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "~/*": ["./src/*"]
    }
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.d.ts",
    "src/**/*.tsx",
    "src/**/*.vue",
    "auto-imports.d.ts",
    "components.d.ts"
  ],
  "exclude": ["node_modules"]
}
```

### การติดตั้ง Vue Language Tools

```bash
# ติดตั้ง volar สำหรับ VS Code
# ไปที่ Extensions แล้วค้นหา "Vue - Official"

# ถ้าใช้ TypeScript ใน SFC ต้องมี vue-tsc สำหรับ type checking
npm install -D vue-tsc typescript
```

### การตั้งค่า package.json scripts

```json
// package.json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc && vite build",
    "type-check": "vue-tsc --noEmit",
    "preview": "vite preview"
  }
}
```

---

## 2. Type-safe Props และ Emits

### การกำหนด Props ด้วย TypeScript

```vue
<!-- components/UserCard.vue -->
<script setup lang="ts">
// วิธีที่ 1: ใช้ interface โดยตรง
interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user' | 'moderator'
  avatar?: string  // optional
}

interface Props {
  user: User
  showEmail?: boolean
  variant?: 'compact' | 'full'
}

// defineProps พร้อม TypeScript
const props = withDefaults(defineProps<Props>(), {
  showEmail: true,
  variant: 'full'
})

// วิธีที่ 2: ใช้ PropType (แบบ Options API)
// import { PropType } from 'vue'
// props: { user: { type: Object as PropType<User>, required: true } }
</script>

<template>
  <div :class="`card card--${props.variant}`">
    <img v-if="props.user.avatar" :src="props.user.avatar" :alt="props.user.name" />
    <h3>{{ props.user.name }}</h3>
    <p v-if="props.showEmail">{{ props.user.email }}</p>
    <span class="badge">{{ props.user.role }}</span>
  </div>
</template>
```

### การกำหนด Emits ด้วย TypeScript

```vue
<!-- components/SearchBar.vue -->
<script setup lang="ts">
import { ref } from 'vue'

interface SearchFilters {
  query: string
  category: string
  sortBy: 'relevance' | 'date' | 'price'
}

// กำหนด emits ด้วย TypeScript
const emit = defineEmits<{
  search: [filters: SearchFilters]
  clear: []
  'update:modelValue': [value: string]
}>()

const query = ref('')
const category = ref('')
const sortBy = ref<'relevance' | 'date' | 'price'>('relevance')

function handleSearch() {
  emit('search', {
    query: query.value,
    category: category.value,
    sortBy: sortBy.value
  })
}

function handleClear() {
  query.value = ''
  category.value = ''
  sortBy.value = 'relevance'
  emit('clear')
}
</script>

<template>
  <div class="search-bar">
    <input
      v-model="query"
      type="text"
      placeholder="ค้นหา..."
      @keyup.enter="handleSearch"
    />
    <select v-model="category">
      <option value="">ทุกหมวดหมู่</option>
      <option value="tech">เทคโนโลยี</option>
      <option value="design">ดีไซน์</option>
    </select>
    <select v-model="sortBy">
      <option value="relevance">ความเกี่ยวข้อง</option>
      <option value="date">วันที่</option>
      <option value="price">ราคา</option>
    </select>
    <button @click="handleSearch">ค้นหา</button>
    <button @click="handleClear">ล้าง</button>
  </div>
</template>
```

---

## 3. Generic Components

Generic Components ช่วยให้เราสร้าง component ที่ทำงานกับ type หลายแบบได้

```vue
<!-- components/GenericList.vue -->
<script setup lang="ts" generic="T extends { id: string | number }">
interface Props {
  items: T[]
  loading?: boolean
  emptyMessage?: string
}

interface Emits {
  select: [item: T]
  delete: [id: T['id']]
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  emptyMessage: 'ไม่มีข้อมูล'
})

const emit = defineEmits<Emits>()
</script>

<template>
  <div class="generic-list">
    <div v-if="props.loading" class="loading">กำลังโหลด...</div>
    <div v-else-if="props.items.length === 0" class="empty">
      {{ props.emptyMessage }}
    </div>
    <ul v-else>
      <li
        v-for="item in props.items"
        :key="item.id"
        @click="emit('select', item)"
      >
        <slot :item="item" />
        <button @click.stop="emit('delete', item.id)">ลบ</button>
      </li>
    </ul>
  </div>
</template>
```

```vue
<!-- การใช้งาน Generic Component -->
<script setup lang="ts">
import GenericList from '@/components/GenericList.vue'

interface Product {
  id: number
  name: string
  price: number
}

const products: Product[] = [
  { id: 1, name: 'สินค้า A', price: 100 },
  { id: 2, name: 'สินค้า B', price: 200 }
]

function handleSelect(product: Product) {
  console.log('เลือก:', product.name)
}

function handleDelete(id: number) {
  console.log('ลบสินค้า id:', id)
}
</script>

<template>
  <GenericList
    :items="products"
    @select="handleSelect"
    @delete="handleDelete"
  >
    <template #default="{ item }">
      <span>{{ item.name }} - ฿{{ item.price }}</span>
    </template>
  </GenericList>
</template>
```

### Generic Composable

```ts
// composables/useAsyncData.ts
import { ref, type Ref } from 'vue'

interface AsyncState<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  execute: () => Promise<void>
}

export function useAsyncData<T>(
  fetcher: () => Promise<T>
): AsyncState<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      data.value = await fetcher()
    } catch (err) {
      error.value = err instanceof Error ? err : new Error(String(err))
    } finally {
      loading.value = false
    }
  }

  return { data, error, loading, execute }
}
```

---

## 4. Composable Types

### การสร้าง Composable ที่ Type-safe

```ts
// composables/useLocalStorage.ts
import { ref, watch, type Ref } from 'vue'

type Serializer<T> = {
  read: (raw: string) => T
  write: (value: T) => string
}

const defaultSerializer: Serializer<unknown> = {
  read: (raw) => JSON.parse(raw),
  write: (value) => JSON.stringify(value)
}

export function useLocalStorage<T>(
  key: string,
  initialValue: T,
  serializer: Serializer<T> = defaultSerializer as Serializer<T>
): Ref<T> {
  function read(): T {
    try {
      const raw = localStorage.getItem(key)
      return raw !== null ? serializer.read(raw) : initialValue
    } catch {
      return initialValue
    }
  }

  const data = ref<T>(read()) as Ref<T>

  watch(data, (newValue) => {
    try {
      if (newValue === null || newValue === undefined) {
        localStorage.removeItem(key)
      } else {
        localStorage.setItem(key, serializer.write(newValue))
      }
    } catch {
      console.error(`Failed to write to localStorage: ${key}`)
    }
  }, { deep: true })

  return data
}
```

```ts
// composables/usePagination.ts
import { computed, ref } from 'vue'

interface PaginationOptions {
  totalItems: number
  itemsPerPage?: number
  initialPage?: number
}

interface PaginationReturn {
  currentPage: ReturnType<typeof ref<number>>
  totalPages: ReturnType<typeof computed<number>>
  hasNextPage: ReturnType<typeof computed<boolean>>
  hasPrevPage: ReturnType<typeof computed<boolean>>
  nextPage: () => void
  prevPage: () => void
  goToPage: (page: number) => void
  pageNumbers: ReturnType<typeof computed<number[]>>
}

export function usePagination({
  totalItems,
  itemsPerPage = 10,
  initialPage = 1
}: PaginationOptions): PaginationReturn {
  const currentPage = ref(initialPage)

  const totalPages = computed(() =>
    Math.ceil(totalItems / itemsPerPage)
  )

  const hasNextPage = computed(() => currentPage.value < totalPages.value)
  const hasPrevPage = computed(() => currentPage.value > 1)

  function nextPage() {
    if (hasNextPage.value) currentPage.value++
  }

  function prevPage() {
    if (hasPrevPage.value) currentPage.value--
  }

  function goToPage(page: number) {
    if (page >= 1 && page <= totalPages.value) {
      currentPage.value = page
    }
  }

  const pageNumbers = computed(() => {
    const pages: number[] = []
    for (let i = 1; i <= totalPages.value; i++) {
      pages.push(i)
    }
    return pages
  })

  return {
    currentPage,
    totalPages,
    hasNextPage,
    hasPrevPage,
    nextPage,
    prevPage,
    goToPage,
    pageNumbers
  }
}
```

---

## 5. Store Types (Pinia)

### การสร้าง Pinia Store ที่ Type-safe

```ts
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

// กำหนด types
export interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user' | 'moderator'
  permissions: Permission[]
  createdAt: Date
}

export type Permission = 
  | 'read:posts'
  | 'write:posts'
  | 'delete:posts'
  | 'manage:users'

export interface LoginCredentials {
  email: string
  password: string
}

export interface AuthState {
  user: User | null
  token: string | null
  isLoading: boolean
  error: string | null
}

export const useUserStore = defineStore('user', () => {
  // State
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  // Getters
  const isAuthenticated = computed(() => !!token.value && !!user.value)
  const isAdmin = computed(() => user.value?.role === 'admin')
  
  function hasPermission(permission: Permission): boolean {
    return user.value?.permissions.includes(permission) ?? false
  }

  // Actions
  async function login(credentials: LoginCredentials): Promise<void> {
    isLoading.value = true
    error.value = null

    try {
      const response = await $fetch<{ user: User; token: string }>('/api/auth/login', {
        method: 'POST',
        body: credentials
      })

      user.value = response.user
      token.value = response.token
    } catch (err: unknown) {
      if (err instanceof Error) {
        error.value = err.message
      } else {
        error.value = 'Login failed'
      }
      throw err
    } finally {
      isLoading.value = false
    }
  }

  async function logout(): Promise<void> {
    try {
      await $fetch('/api/auth/logout', { method: 'POST' })
    } finally {
      user.value = null
      token.value = null
    }
  }

  async function fetchProfile(): Promise<void> {
    if (!token.value) return

    try {
      user.value = await $fetch<User>('/api/auth/profile')
    } catch (err) {
      await logout()
    }
  }

  return {
    user,
    token,
    isLoading,
    error,
    isAuthenticated,
    isAdmin,
    hasPermission,
    login,
    logout,
    fetchProfile
  }
})
```

---

## 6. API Response Types

### การสร้าง Types สำหรับ API

```ts
// types/api.ts

// Generic API Response wrapper
export interface ApiResponse<T> {
  data: T
  message: string
  status: 'success' | 'error'
  timestamp: string
}

// Pagination
export interface PaginatedResponse<T> {
  data: T[]
  pagination: {
    page: number
    limit: number
    total: number
    totalPages: number
    hasNext: boolean
    hasPrev: boolean
  }
}

// Error Response
export interface ApiError {
  code: string
  message: string
  details?: Record<string, string[]>
  timestamp: string
}

// Post Types
export interface Post {
  id: number
  title: string
  slug: string
  content: string
  excerpt: string
  author: Author
  category: Category
  tags: Tag[]
  publishedAt: string | null
  createdAt: string
  updatedAt: string
  status: 'draft' | 'published' | 'archived'
  viewCount: number
}

export interface Author {
  id: number
  name: string
  avatar: string | null
  bio: string | null
}

export interface Category {
  id: number
  name: string
  slug: string
}

export interface Tag {
  id: number
  name: string
  slug: string
}

// Create/Update types (ไม่รวม server-generated fields)
export type CreatePostInput = Pick<Post, 'title' | 'content' | 'excerpt'> & {
  categoryId: number
  tagIds: number[]
  status?: Post['status']
}

export type UpdatePostInput = Partial<CreatePostInput>
```

### การใช้ API Types กับ Nuxt useFetch

```ts
// composables/usePosts.ts
import type { Post, PaginatedResponse, CreatePostInput, UpdatePostInput } from '~/types/api'

export function usePosts() {
  const posts = ref<Post[]>([])
  const total = ref(0)
  const loading = ref(false)

  async function fetchPosts(page = 1, limit = 10): Promise<void> {
    loading.value = true
    try {
      const { data } = await useFetch<PaginatedResponse<Post>>('/api/posts', {
        query: { page, limit }
      })
      
      if (data.value) {
        posts.value = data.value.data
        total.value = data.value.pagination.total
      }
    } finally {
      loading.value = false
    }
  }

  async function createPost(input: CreatePostInput): Promise<Post> {
    return $fetch<Post>('/api/posts', {
      method: 'POST',
      body: input
    })
  }

  async function updatePost(id: number, input: UpdatePostInput): Promise<Post> {
    return $fetch<Post>(`/api/posts/${id}`, {
      method: 'PATCH',
      body: input
    })
  }

  async function deletePost(id: number): Promise<void> {
    await $fetch(`/api/posts/${id}`, { method: 'DELETE' })
    posts.value = posts.value.filter(p => p.id !== id)
  }

  return {
    posts: readonly(posts),
    total: readonly(total),
    loading: readonly(loading),
    fetchPosts,
    createPost,
    updatePost,
    deletePost
  }
}
```

---

## 7. Utility Types ที่ใช้บ่อย

TypeScript มี Utility Types ที่มีประโยชน์มาก ลองดูตัวอย่างการใช้งาน:

```ts
// types/utilities.ts

// ตัวอย่าง Base types
interface User {
  id: number
  name: string
  email: string
  password: string
  role: 'admin' | 'user'
  createdAt: Date
  updatedAt: Date
}

// Partial<T> - ทำให้ทุก field เป็น optional
type UpdateUserInput = Partial<User>
// { id?: number; name?: string; email?: string; ... }

// Required<T> - ทำให้ทุก field เป็น required
type RequiredUser = Required<User>

// Pick<T, K> - เลือกเฉพาะ fields ที่ต้องการ
type UserProfile = Pick<User, 'id' | 'name' | 'email'>
// { id: number; name: string; email: string }

// Omit<T, K> - ลบ fields ที่ไม่ต้องการ
type PublicUser = Omit<User, 'password' | 'createdAt' | 'updatedAt'>
// { id: number; name: string; email: string; role: ... }

// Readonly<T> - ทำให้ทุก field เป็น readonly
type ReadonlyUser = Readonly<User>

// Record<K, V> - สร้าง object type
type UsersByRole = Record<User['role'], User[]>
// { admin: User[]; user: User[] }

// Exclude<T, U> - ลบ types ออกจาก union
type NonAdminRole = Exclude<User['role'], 'admin'>
// 'user'

// Extract<T, U> - เลือก types จาก union
type AdminRole = Extract<User['role'], 'admin'>
// 'admin'

// NonNullable<T> - ลบ null และ undefined
type NonNullableUser = NonNullable<User | null | undefined>
// User

// ReturnType<T> - ดึง return type ของ function
async function getUser(): Promise<User> {
  return {} as User
}
type GetUserReturn = Awaited<ReturnType<typeof getUser>>
// User

// Parameters<T> - ดึง parameter types ของ function
function createUser(name: string, email: string, role: User['role']): User {
  return {} as User
}
type CreateUserParams = Parameters<typeof createUser>
// [name: string, email: string, role: "admin" | "user"]
```

```ts
// Custom Utility Types ที่มีประโยชน์
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P]
}

type Nullable<T> = T | null

type Maybe<T> = T | null | undefined

type ValueOf<T> = T[keyof T]

// ใช้สำหรับ form validation errors
type FormErrors<T> = Partial<Record<keyof T, string>>

// ใช้สำหรับ loading states
type LoadingState = 'idle' | 'loading' | 'success' | 'error'

// ใช้สำหรับ async operations
interface AsyncOperation<T> {
  data: T | null
  error: Error | null
  status: LoadingState
}
```

---

## 8. ตัวอย่าง: Type-safe CRUD Component

```ts
// types/product.ts
export interface Product {
  id: number
  name: string
  description: string
  price: number
  stock: number
  category: string
  imageUrl: string | null
  isActive: boolean
  createdAt: string
  updatedAt: string
}

export type CreateProductInput = Omit<Product, 'id' | 'createdAt' | 'updatedAt'>

export type UpdateProductInput = Partial<CreateProductInput>

export interface ProductFilters {
  search?: string
  category?: string
  minPrice?: number
  maxPrice?: number
  isActive?: boolean
}

export interface ProductSortOptions {
  field: keyof Pick<Product, 'name' | 'price' | 'stock' | 'createdAt'>
  order: 'asc' | 'desc'
}
```

```ts
// composables/useProductCRUD.ts
import type {
  Product,
  CreateProductInput,
  UpdateProductInput,
  ProductFilters,
  ProductSortOptions
} from '~/types/product'

interface UseProductCRUDReturn {
  products: Readonly<Ref<Product[]>>
  selectedProduct: Readonly<Ref<Product | null>>
  isLoading: Readonly<Ref<boolean>>
  error: Readonly<Ref<string | null>>
  fetchProducts: (filters?: ProductFilters, sort?: ProductSortOptions) => Promise<void>
  createProduct: (input: CreateProductInput) => Promise<Product>
  updateProduct: (id: number, input: UpdateProductInput) => Promise<Product>
  deleteProduct: (id: number) => Promise<void>
  selectProduct: (product: Product | null) => void
}

export function useProductCRUD(): UseProductCRUDReturn {
  const products = ref<Product[]>([])
  const selectedProduct = ref<Product | null>(null)
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  async function fetchProducts(
    filters?: ProductFilters,
    sort?: ProductSortOptions
  ): Promise<void> {
    isLoading.value = true
    error.value = null
    try {
      const query: Record<string, string | number | boolean | undefined> = {}
      
      if (filters?.search) query.search = filters.search
      if (filters?.category) query.category = filters.category
      if (filters?.minPrice !== undefined) query.minPrice = filters.minPrice
      if (filters?.maxPrice !== undefined) query.maxPrice = filters.maxPrice
      if (filters?.isActive !== undefined) query.isActive = filters.isActive
      if (sort) {
        query.sortField = sort.field
        query.sortOrder = sort.order
      }

      const data = await $fetch<Product[]>('/api/products', { query })
      products.value = data
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to fetch products'
    } finally {
      isLoading.value = false
    }
  }

  async function createProduct(input: CreateProductInput): Promise<Product> {
    const newProduct = await $fetch<Product>('/api/products', {
      method: 'POST',
      body: input
    })
    products.value.push(newProduct)
    return newProduct
  }

  async function updateProduct(id: number, input: UpdateProductInput): Promise<Product> {
    const updated = await $fetch<Product>(`/api/products/${id}`, {
      method: 'PATCH',
      body: input
    })
    const index = products.value.findIndex(p => p.id === id)
    if (index !== -1) {
      products.value[index] = updated
    }
    return updated
  }

  async function deleteProduct(id: number): Promise<void> {
    await $fetch(`/api/products/${id}`, { method: 'DELETE' })
    products.value = products.value.filter(p => p.id !== id)
    if (selectedProduct.value?.id === id) {
      selectedProduct.value = null
    }
  }

  function selectProduct(product: Product | null): void {
    selectedProduct.value = product
  }

  return {
    products: readonly(products),
    selectedProduct: readonly(selectedProduct),
    isLoading: readonly(isLoading),
    error: readonly(error),
    fetchProducts,
    createProduct,
    updateProduct,
    deleteProduct,
    selectProduct
  }
}
```

```vue
<!-- pages/products/index.vue -->
<script setup lang="ts">
import type { CreateProductInput, ProductFilters } from '~/types/product'
import { useProductCRUD } from '~/composables/useProductCRUD'

const {
  products,
  selectedProduct,
  isLoading,
  error,
  fetchProducts,
  createProduct,
  updateProduct,
  deleteProduct,
  selectProduct
} = useProductCRUD()

// Form state
const showForm = ref(false)
const isEditing = ref(false)

// Form data with proper typing
const formData = reactive<CreateProductInput>({
  name: '',
  description: '',
  price: 0,
  stock: 0,
  category: '',
  imageUrl: null,
  isActive: true
})

// Filters
const filters = reactive<ProductFilters>({
  search: '',
  category: undefined,
  isActive: true
})

// โหลดข้อมูลเมื่อ component mount
onMounted(() => fetchProducts(filters))

// Watch filters และ reload
watch(filters, () => fetchProducts(filters), { deep: true })

function openCreateForm() {
  isEditing.value = false
  Object.assign(formData, {
    name: '',
    description: '',
    price: 0,
    stock: 0,
    category: '',
    imageUrl: null,
    isActive: true
  })
  showForm.value = true
}

function openEditForm() {
  if (!selectedProduct.value) return
  isEditing.value = true
  Object.assign(formData, selectedProduct.value)
  showForm.value = true
}

async function handleSubmit() {
  if (isEditing.value && selectedProduct.value) {
    await updateProduct(selectedProduct.value.id, formData)
  } else {
    await createProduct(formData)
  }
  showForm.value = false
  selectProduct(null)
}

async function handleDelete(id: number) {
  if (confirm('ต้องการลบสินค้านี้หรือไม่?')) {
    await deleteProduct(id)
  }
}
</script>

<template>
  <div class="products-page">
    <div class="header">
      <h1>จัดการสินค้า</h1>
      <button @click="openCreateForm" class="btn-primary">
        + เพิ่มสินค้า
      </button>
    </div>

    <!-- Filters -->
    <div class="filters">
      <input
        v-model="filters.search"
        type="text"
        placeholder="ค้นหาสินค้า..."
        class="search-input"
      />
      <select v-model="filters.category">
        <option value="">ทุกหมวดหมู่</option>
        <option value="electronics">อิเล็กทรอนิกส์</option>
        <option value="clothing">เสื้อผ้า</option>
      </select>
    </div>

    <!-- Error State -->
    <div v-if="error" class="error-message">{{ error }}</div>

    <!-- Loading State -->
    <div v-if="isLoading" class="loading">กำลังโหลด...</div>

    <!-- Products Table -->
    <table v-else class="products-table">
      <thead>
        <tr>
          <th>ชื่อสินค้า</th>
          <th>ราคา</th>
          <th>สต็อก</th>
          <th>หมวดหมู่</th>
          <th>สถานะ</th>
          <th>จัดการ</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="product in products"
          :key="product.id"
          :class="{ selected: selectedProduct?.id === product.id }"
          @click="selectProduct(product)"
        >
          <td>{{ product.name }}</td>
          <td>฿{{ product.price.toLocaleString() }}</td>
          <td>{{ product.stock }}</td>
          <td>{{ product.category }}</td>
          <td>
            <span :class="product.isActive ? 'badge-success' : 'badge-error'">
              {{ product.isActive ? 'เปิดใช้งาน' : 'ปิดใช้งาน' }}
            </span>
          </td>
          <td>
            <button @click.stop="openEditForm" :disabled="selectedProduct?.id !== product.id">
              แก้ไข
            </button>
            <button @click.stop="handleDelete(product.id)" class="btn-danger">
              ลบ
            </button>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- Product Form Modal -->
    <div v-if="showForm" class="modal">
      <div class="modal-content">
        <h2>{{ isEditing ? 'แก้ไขสินค้า' : 'เพิ่มสินค้าใหม่' }}</h2>
        <form @submit.prevent="handleSubmit">
          <div class="form-group">
            <label>ชื่อสินค้า</label>
            <input v-model="formData.name" required />
          </div>
          <div class="form-group">
            <label>รายละเอียด</label>
            <textarea v-model="formData.description" />
          </div>
          <div class="form-row">
            <div class="form-group">
              <label>ราคา (฿)</label>
              <input v-model.number="formData.price" type="number" min="0" required />
            </div>
            <div class="form-group">
              <label>สต็อก</label>
              <input v-model.number="formData.stock" type="number" min="0" required />
            </div>
          </div>
          <div class="form-group">
            <label>หมวดหมู่</label>
            <select v-model="formData.category" required>
              <option value="electronics">อิเล็กทรอนิกส์</option>
              <option value="clothing">เสื้อผ้า</option>
              <option value="food">อาหาร</option>
            </select>
          </div>
          <div class="form-group">
            <label class="checkbox-label">
              <input v-model="formData.isActive" type="checkbox" />
              เปิดใช้งาน
            </label>
          </div>
          <div class="form-actions">
            <button type="button" @click="showForm = false">ยกเลิก</button>
            <button type="submit" class="btn-primary">
              {{ isEditing ? 'บันทึกการแก้ไข' : 'เพิ่มสินค้า' }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>
```

---

## สรุป

TypeScript กับ Vue 3 ทำงานร่วมกันได้อย่างยอดเยี่ยม:

1. **TypeScript Setup** - ตั้งค่า tsconfig.json และติดตั้ง tools ที่จำเป็น
2. **Type-safe Props/Emits** - ใช้ `defineProps<T>()` และ `defineEmits<T>()` แบบ Generic
3. **Generic Components** - สร้าง component ที่ reuse ได้กับหลาย types
4. **Composable Types** - กำหนด return type ชัดเจน ใช้ Generic สำหรับ flexibility
5. **Store Types (Pinia)** - กำหนด state, getter, action types อย่างละเอียด
6. **API Response Types** - สร้าง types สำหรับ API response ที่ consistent
7. **Utility Types** - ใช้ `Partial`, `Pick`, `Omit`, `Record` ฯลฯ เพื่อลดการเขียนซ้ำ
