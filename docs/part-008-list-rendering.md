# Part 8: List Rendering

## บทนำ

List Rendering ด้วย `v-for` เป็นหนึ่งในฟีเจอร์พื้นฐานที่สำคัญมากของ Vue.js การเข้าใจวิธีการทำงานของ `v-for`, การใช้ `key` attribute อย่างถูกต้อง และการจัดการ large datasets จะช่วยให้แอพพลิเคชันของเรามีประสิทธิภาพดีขึ้นมาก

---

## 1. v-for พื้นฐาน

### Iterate Arrays

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Product {
  id: number
  name: string
  price: number
  category: string
}

const fruits = ref(['แอปเปิล', 'กล้วย', 'เชอร์รี่', 'ทุเรียน'])
const products = ref<Product[]>([
  { id: 1, name: 'Vue.js Book', price: 599, category: 'books' },
  { id: 2, name: 'TypeScript Guide', price: 799, category: 'books' },
  { id: 3, name: 'Mechanical Keyboard', price: 3500, category: 'hardware' },
])
</script>

<template>
  <!-- v-for กับ array พื้นฐาน -->
  <ul>
    <li v-for="fruit in fruits" :key="fruit">
      {{ fruit }}
    </li>
  </ul>

  <!-- v-for กับ index -->
  <ul>
    <li v-for="(fruit, index) in fruits" :key="fruit">
      {{ index + 1 }}. {{ fruit }}
    </li>
  </ul>

  <!-- v-for กับ objects -->
  <div v-for="product in products" :key="product.id" class="product-card">
    <h3>{{ product.name }}</h3>
    <p>฿{{ product.price }}</p>
    <span>{{ product.category }}</span>
  </div>

  <!-- v-for destructuring -->
  <div v-for="{ id, name, price } in products" :key="id">
    {{ name }}: ฿{{ price }}
  </div>
</template>
```

### Iterate Objects

```vue
<script setup lang="ts">
import { ref } from 'vue'

const userInfo = ref({
  name: 'สมชาย ใจดี',
  email: 'somchai@example.com',
  phone: '081-234-5678',
  city: 'กรุงเทพมหานคร',
  country: 'ประเทศไทย',
})

const config = {
  theme: 'dark',
  language: 'th',
  fontSize: 16,
  autoSave: true,
}
</script>

<template>
  <!-- iterate object values -->
  <ul>
    <li v-for="value in userInfo" :key="value">
      {{ value }}
    </li>
  </ul>

  <!-- iterate กับ key และ index -->
  <dl>
    <template v-for="(value, key, index) in userInfo" :key="key">
      <dt>{{ index + 1 }}. {{ key }}</dt>
      <dd>{{ value }}</dd>
    </template>
  </dl>

  <!-- แสดง config แบบ table -->
  <table>
    <tr v-for="(value, key) in config" :key="key">
      <td>{{ key }}</td>
      <td>{{ value }}</td>
    </tr>
  </table>
</template>
```

### Iterate Numbers และ Strings

```vue
<template>
  <!-- iterate number (1-based) -->
  <div class="pagination">
    <button v-for="page in 10" :key="page">{{ page }}</button>
  </div>

  <!-- iterate string characters -->
  <div class="word-display">
    <span
      v-for="(char, index) in 'VUEJS'"
      :key="index"
      class="char"
    >
      {{ char }}
    </span>
  </div>

  <!-- สร้าง star rating -->
  <div class="stars">
    <span v-for="star in 5" :key="star" :class="star <= 4 ? 'filled' : 'empty'">
      ★
    </span>
  </div>
</template>
```

---

## 2. key Attribute ความสำคัญ

`key` ช่วยให้ Vue ระบุ identity ของแต่ละ element เพื่อ optimize re-rendering

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Item {
  id: number
  text: string
  checked: boolean
}

const items = ref<Item[]>([
  { id: 1, text: 'Item 1', checked: false },
  { id: 2, text: 'Item 2', checked: true },
  { id: 3, text: 'Item 3', checked: false },
])

function removeFirst() {
  items.value.shift()
}

function addToFront() {
  const newId = Math.random()
  items.value.unshift({ id: newId, text: `New Item ${newId.toFixed(2)}`, checked: false })
}
</script>

<template>
  <div class="key-demo">
    <button @click="removeFirst">ลบรายการแรก</button>
    <button @click="addToFront">เพิ่มที่หน้า</button>

    <!-- ❌ ไม่ดี: ใช้ index เป็น key -->
    <!-- เมื่อเพิ่ม/ลบที่ต้น list, Vue จะ re-use elements ผิดตัว -->
    <ul>
      <li v-for="(item, index) in items" :key="index">
        <input type="checkbox" v-model="item.checked" />
        {{ item.text }}
      </li>
    </ul>

    <!-- ✅ ดี: ใช้ unique id เป็น key -->
    <!-- Vue จะรู้ว่า element ไหนคือ element ไหน -->
    <ul>
      <li v-for="item in items" :key="item.id">
        <input type="checkbox" v-model="item.checked" />
        {{ item.text }}
      </li>
    </ul>
  </div>
</template>
```

### ปัญหาที่เกิดจากการใช้ index เป็น key

```vue
<script setup lang="ts">
import { ref } from 'vue'

const todos = ref([
  { id: 1, text: 'เรียน Vue', done: false },
  { id: 2, text: 'สร้าง Project', done: false },
  { id: 3, text: 'Deploy', done: false },
])

function deleteFirst() {
  todos.value.splice(0, 1) // ลบ index 0
}
</script>

<template>
  <div>
    <button @click="deleteFirst">ลบรายการแรก</button>

    <!-- ❌ Bug: ถ้าเลือก checkbox แล้วลบรายการแรก
         checkbox ของ Item 2 จะย้ายมาอยู่กับ Item 3 -->
    <div v-for="(todo, index) in todos" :key="index">
      <input type="checkbox" v-model="todo.done" />
      {{ todo.text }}
    </div>

    <!-- ✅ ถูกต้อง: ใช้ id ที่ไม่ซ้ำกัน -->
    <div v-for="todo in todos" :key="todo.id">
      <input type="checkbox" v-model="todo.done" />
      {{ todo.text }}
    </div>
  </div>
</template>
```

---

## 3. Maintaining State ด้วย key

`key` ยังใช้สำหรับ force re-render component ได้ด้วย

```vue
<script setup lang="ts">
import { ref } from 'vue'

const selectedUserId = ref(1)
const users = [
  { id: 1, name: 'สมชาย', email: 'somchai@example.com' },
  { id: 2, name: 'สมหญิง', email: 'somying@example.com' },
  { id: 3, name: 'สมศรี', email: 'somsri@example.com' },
]
</script>

<template>
  <div>
    <select v-model="selectedUserId">
      <option v-for="user in users" :key="user.id" :value="user.id">
        {{ user.name }}
      </option>
    </select>

    <!-- ใช้ :key เพื่อ force re-mount UserProfile เมื่อ user เปลี่ยน -->
    <!-- ทำให้ component reset state ทั้งหมด -->
    <UserProfile :user-id="selectedUserId" :key="selectedUserId" />
  </div>
</template>
```

---

## 4. v-for กับ computed

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Task {
  id: number
  title: string
  status: 'todo' | 'doing' | 'done'
  priority: 'low' | 'medium' | 'high'
  assignee: string
  dueDate: string
}

const tasks = ref<Task[]>([
  { id: 1, title: 'ออกแบบ UI', status: 'done', priority: 'high', assignee: 'สมชาย', dueDate: '2024-01-10' },
  { id: 2, title: 'สร้าง API', status: 'doing', priority: 'high', assignee: 'สมหญิง', dueDate: '2024-01-15' },
  { id: 3, title: 'เขียน Tests', status: 'todo', priority: 'medium', assignee: 'สมชาย', dueDate: '2024-01-20' },
  { id: 4, title: 'Deploy', status: 'todo', priority: 'low', assignee: 'สมศรี', dueDate: '2024-01-25' },
  { id: 5, title: 'Documentation', status: 'todo', priority: 'medium', assignee: 'สมหญิง', dueDate: '2024-01-30' },
])

const filterStatus = ref<Task['status'] | 'all'>('all')
const filterPriority = ref<Task['priority'] | 'all'>('all')
const filterAssignee = ref<string>('all')
const sortBy = ref<'title' | 'dueDate' | 'priority'>('dueDate')
const sortOrder = ref<'asc' | 'desc'>('asc')

const priorityOrder = { high: 3, medium: 2, low: 1 }

const filteredAndSortedTasks = computed(() => {
  let result = tasks.value

  // Filter
  if (filterStatus.value !== 'all') {
    result = result.filter(t => t.status === filterStatus.value)
  }
  if (filterPriority.value !== 'all') {
    result = result.filter(t => t.priority === filterPriority.value)
  }
  if (filterAssignee.value !== 'all') {
    result = result.filter(t => t.assignee === filterAssignee.value)
  }

  // Sort
  const sorted = [...result].sort((a, b) => {
    let comparison = 0
    if (sortBy.value === 'title') {
      comparison = a.title.localeCompare(b.title)
    } else if (sortBy.value === 'dueDate') {
      comparison = new Date(a.dueDate).getTime() - new Date(b.dueDate).getTime()
    } else if (sortBy.value === 'priority') {
      comparison = priorityOrder[b.priority] - priorityOrder[a.priority]
    }
    return sortOrder.value === 'desc' ? -comparison : comparison
  })

  return sorted
})

const assignees = computed(() => ['all', ...new Set(tasks.value.map(t => t.assignee))])

const tasksByStatus = computed(() => ({
  todo: tasks.value.filter(t => t.status === 'todo'),
  doing: tasks.value.filter(t => t.status === 'doing'),
  done: tasks.value.filter(t => t.status === 'done'),
}))
</script>

<template>
  <div class="task-manager">
    <!-- Filters -->
    <div class="filters">
      <select v-model="filterStatus">
        <option value="all">ทุกสถานะ</option>
        <option value="todo">รอทำ</option>
        <option value="doing">กำลังทำ</option>
        <option value="done">เสร็จแล้ว</option>
      </select>

      <select v-model="filterPriority">
        <option value="all">ทุกระดับ</option>
        <option value="high">สูง</option>
        <option value="medium">กลาง</option>
        <option value="low">ต่ำ</option>
      </select>

      <select v-model="filterAssignee">
        <option v-for="assignee in assignees" :key="assignee" :value="assignee">
          {{ assignee === 'all' ? 'ทุกคน' : assignee }}
        </option>
      </select>
    </div>

    <!-- Task list -->
    <p>แสดง {{ filteredAndSortedTasks.length }} จาก {{ tasks.length }} งาน</p>

    <div v-if="filteredAndSortedTasks.length === 0" class="empty-state">
      ไม่พบงานที่ตรงกับเงื่อนไข
    </div>

    <div v-for="task in filteredAndSortedTasks" :key="task.id" class="task-item">
      <span :class="['priority-badge', task.priority]">{{ task.priority }}</span>
      <span :class="['status-badge', task.status]">{{ task.status }}</span>
      <strong>{{ task.title }}</strong>
      <span>{{ task.assignee }}</span>
      <span>{{ task.dueDate }}</span>
    </div>

    <!-- Nested v-for: tasks grouped by status -->
    <div class="kanban-view">
      <div
        v-for="(statusTasks, status) in tasksByStatus"
        :key="status"
        class="kanban-column"
      >
        <h3>{{ status }} ({{ statusTasks.length }})</h3>
        <div
          v-for="task in statusTasks"
          :key="task.id"
          class="kanban-card"
        >
          {{ task.title }}
        </div>
      </div>
    </div>
  </div>
</template>
```

---

## 5. Array Mutation Methods

Vue จะ detect การเปลี่ยนแปลงที่เกิดจาก mutation methods และ trigger update โดยอัตโนมัติ

```vue
<script setup lang="ts">
import { ref } from 'vue'

const items = ref(['a', 'b', 'c', 'd', 'e'])
const numbers = ref([3, 1, 4, 1, 5, 9, 2, 6, 5, 3])

// Mutation methods (แก้ไข array โดยตรง) - Vue ตรวจจับได้
function demoMutations() {
  // push - เพิ่มที่ท้าย
  items.value.push('f', 'g')

  // pop - ลบที่ท้าย
  items.value.pop()

  // shift - ลบที่หน้า
  items.value.shift()

  // unshift - เพิ่มที่หน้า
  items.value.unshift('A')

  // splice - เพิ่ม/ลบตรงกลาง
  items.value.splice(2, 1) // ลบ 1 element ที่ index 2
  items.value.splice(2, 0, 'X', 'Y') // เพิ่มที่ index 2

  // sort - เรียงลำดับ
  numbers.value.sort((a, b) => a - b)

  // reverse - กลับลำดับ
  items.value.reverse()
}

// Non-mutation methods (สร้าง array ใหม่) - ต้อง reassign
function demoNonMutations() {
  // ❌ Vue ไม่สามารถตรวจจับ direct index assignment ได้ (ใน older Vue)
  // items.value[1] = 'Z' // ใช้ spliceแทน

  // ✅ filter - สร้าง array ใหม่
  items.value = items.value.filter(item => item !== 'b')

  // ✅ map - transform
  items.value = items.value.map(item => item.toUpperCase())

  // ✅ concat
  items.value = items.value.concat(['h', 'i'])

  // ✅ slice
  items.value = items.value.slice(1, 4)

  // ✅ spread operator
  items.value = [...items.value, 'new item']
}

// ✅ การ update แบบ safe
const todos = ref([
  { id: 1, text: 'Task 1', done: false },
  { id: 2, text: 'Task 2', done: false },
])

function updateTodo(id: number, updates: Partial<typeof todos.value[0]>) {
  const index = todos.value.findIndex(t => t.id === id)
  if (index !== -1) {
    // Object.assign เพื่อ maintain reactivity
    Object.assign(todos.value[index], updates)
    // หรือ
    todos.value[index] = { ...todos.value[index], ...updates }
  }
}
</script>
```

---

## 6. Virtual List สำหรับ Large Data

เมื่อมีรายการจำนวนมาก (10,000+) การ render ทั้งหมดพร้อมกันจะทำให้ performance แย่มาก

```vue
<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

interface ListItem {
  id: number
  name: string
  email: string
  avatar: string
  status: 'online' | 'offline' | 'away'
}

// สร้าง mock data จำนวนมาก
const allItems = ref<ListItem[]>(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i + 1,
    name: `User ${i + 1}`,
    email: `user${i + 1}@example.com`,
    avatar: `https://api.dicebear.com/7.x/avataaars/svg?seed=${i}`,
    status: ['online', 'offline', 'away'][i % 3] as ListItem['status'],
  }))
)

// Virtual List implementation
const containerRef = ref<HTMLElement | null>(null)
const ITEM_HEIGHT = 64 // ความสูงของแต่ละ item (px)
const BUFFER_SIZE = 5 // จำนวน items ที่ render เพิ่มเพื่อ smooth scroll

const scrollTop = ref(0)
const containerHeight = ref(600)

const visibleRange = computed(() => {
  const start = Math.max(0, Math.floor(scrollTop.value / ITEM_HEIGHT) - BUFFER_SIZE)
  const visibleCount = Math.ceil(containerHeight.value / ITEM_HEIGHT)
  const end = Math.min(allItems.value.length, start + visibleCount + BUFFER_SIZE * 2)
  return { start, end }
})

const visibleItems = computed(() =>
  allItems.value.slice(visibleRange.value.start, visibleRange.value.end)
)

const totalHeight = computed(() => allItems.value.length * ITEM_HEIGHT)

const offsetY = computed(() => visibleRange.value.start * ITEM_HEIGHT)

function handleScroll(event: Event) {
  scrollTop.value = (event.target as HTMLElement).scrollTop
}

onMounted(() => {
  if (containerRef.value) {
    containerHeight.value = containerRef.value.clientHeight
  }
})
</script>

<template>
  <div class="virtual-list-container">
    <h3>รายชื่อผู้ใช้ {{ allItems.length.toLocaleString() }} คน</h3>
    <p>แสดง {{ visibleItems.length }} items จาก {{ allItems.length }} ทั้งหมด</p>

    <!-- Virtual scroll container -->
    <div
      ref="containerRef"
      class="scroll-container"
      @scroll="handleScroll"
      :style="{ height: '600px', overflow: 'auto' }"
    >
      <!-- Total height spacer -->
      <div :style="{ height: `${totalHeight}px`, position: 'relative' }">
        <!-- Visible items only -->
        <div
          :style="{
            position: 'absolute',
            top: `${offsetY}px`,
            left: 0,
            right: 0
          }"
        >
          <div
            v-for="item in visibleItems"
            :key="item.id"
            class="list-item"
            :style="{ height: `${ITEM_HEIGHT}px` }"
          >
            <div class="avatar">{{ item.name[0] }}</div>
            <div class="info">
              <strong>{{ item.name }}</strong>
              <span>{{ item.email }}</span>
            </div>
            <span :class="['status', item.status]">{{ item.status }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.scroll-container { border: 1px solid #e5e7eb; border-radius: 8px; }
.list-item { display: flex; align-items: center; gap: 1rem; padding: 0 1rem; border-bottom: 1px solid #f3f4f6; }
.avatar { width: 40px; height: 40px; border-radius: 50%; background: #3b82f6; color: white; display: flex; align-items: center; justify-content: center; font-weight: bold; flex-shrink: 0; }
.info { flex: 1; }
.info strong { display: block; }
.info span { font-size: 0.875rem; color: #6b7280; }
.status.online { color: #10b981; }
.status.offline { color: #6b7280; }
.status.away { color: #f59e0b; }
</style>
```

### ใช้ library สำหรับ Virtual List

```bash
npm install vue-virtual-scroller
# หรือ
npm install @tanstack/vue-virtual
```

```vue
<!-- ใช้ vue-virtual-scroller -->
<script setup lang="ts">
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

const items = ref(/* large dataset */)
</script>

<template>
  <RecycleScroller
    class="scroller"
    :items="items"
    :item-size="64"
    key-field="id"
  >
    <template #default="{ item }">
      <div class="item">{{ item.name }}</div>
    </template>
  </RecycleScroller>
</template>
```

---

## 7. ตัวอย่าง: Data Table สมบูรณ์ (sort, filter, paginate)

```vue
<script setup lang="ts">
import { ref, computed, reactive } from 'vue'

interface Employee {
  id: number
  name: string
  email: string
  department: string
  position: string
  salary: number
  joinDate: string
  status: 'active' | 'inactive'
}

type SortKey = keyof Employee
type SortOrder = 'asc' | 'desc'

// Generate mock data
const employees = ref<Employee[]>(
  Array.from({ length: 50 }, (_, i) => ({
    id: i + 1,
    name: ['สมชาย ใจดี', 'สมหญิง รักดี', 'สมศรี ดีใจ', 'ประสิทธิ์ มีผล', 'วิไล สวยงาม'][i % 5],
    email: `employee${i + 1}@company.com`,
    department: ['Engineering', 'Marketing', 'HR', 'Finance', 'Operations'][i % 5],
    position: ['Developer', 'Manager', 'Analyst', 'Designer', 'Lead'][i % 5],
    salary: 30000 + (i % 7) * 10000,
    joinDate: `202${i % 4}-${String((i % 12) + 1).padStart(2, '0')}-${String((i % 28) + 1).padStart(2, '0')}`,
    status: i % 7 === 0 ? 'inactive' : 'active',
  }))
)

// Table state
const state = reactive({
  search: '',
  filters: {
    department: '' as string,
    status: '' as string,
    salaryMin: 0,
    salaryMax: 200000,
  },
  sort: {
    key: 'id' as SortKey,
    order: 'asc' as SortOrder,
  },
  pagination: {
    page: 1,
    pageSize: 10,
  },
  selectedIds: new Set<number>(),
})

// Column definitions
const columns = [
  { key: 'id' as SortKey, label: 'ID', sortable: true, width: '60px' },
  { key: 'name' as SortKey, label: 'ชื่อ', sortable: true },
  { key: 'email' as SortKey, label: 'อีเมล', sortable: true },
  { key: 'department' as SortKey, label: 'แผนก', sortable: true },
  { key: 'position' as SortKey, label: 'ตำแหน่ง', sortable: true },
  { key: 'salary' as SortKey, label: 'เงินเดือน', sortable: true },
  { key: 'joinDate' as SortKey, label: 'วันที่เข้าทำงาน', sortable: true },
  { key: 'status' as SortKey, label: 'สถานะ', sortable: true },
]

// Filtered data
const filteredData = computed(() => {
  let result = employees.value

  // Search
  if (state.search) {
    const query = state.search.toLowerCase()
    result = result.filter(emp =>
      emp.name.toLowerCase().includes(query) ||
      emp.email.toLowerCase().includes(query) ||
      emp.department.toLowerCase().includes(query) ||
      emp.position.toLowerCase().includes(query)
    )
  }

  // Department filter
  if (state.filters.department) {
    result = result.filter(emp => emp.department === state.filters.department)
  }

  // Status filter
  if (state.filters.status) {
    result = result.filter(emp => emp.status === state.filters.status)
  }

  // Salary range filter
  result = result.filter(emp =>
    emp.salary >= state.filters.salaryMin &&
    emp.salary <= state.filters.salaryMax
  )

  return result
})

// Sorted data
const sortedData = computed(() => {
  const sorted = [...filteredData.value]
  const { key, order } = state.sort

  sorted.sort((a, b) => {
    const aVal = a[key]
    const bVal = b[key]

    let comparison = 0
    if (typeof aVal === 'number' && typeof bVal === 'number') {
      comparison = aVal - bVal
    } else {
      comparison = String(aVal).localeCompare(String(bVal))
    }

    return order === 'desc' ? -comparison : comparison
  })

  return sorted
})

// Pagination
const totalPages = computed(() =>
  Math.ceil(sortedData.value.length / state.pagination.pageSize)
)

const paginatedData = computed(() => {
  const { page, pageSize } = state.pagination
  const start = (page - 1) * pageSize
  return sortedData.value.slice(start, start + pageSize)
})

const pageNumbers = computed(() => {
  const total = totalPages.value
  const current = state.pagination.page
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

// Sort handler
function handleSort(key: SortKey) {
  if (state.sort.key === key) {
    state.sort.order = state.sort.order === 'asc' ? 'desc' : 'asc'
  } else {
    state.sort.key = key
    state.sort.order = 'asc'
  }
  state.pagination.page = 1
}

// Selection
const allSelected = computed(() =>
  paginatedData.value.length > 0 &&
  paginatedData.value.every(emp => state.selectedIds.has(emp.id))
)

function toggleSelectAll() {
  if (allSelected.value) {
    paginatedData.value.forEach(emp => state.selectedIds.delete(emp.id))
  } else {
    paginatedData.value.forEach(emp => state.selectedIds.add(emp.id))
  }
}

function toggleSelect(id: number) {
  if (state.selectedIds.has(id)) {
    state.selectedIds.delete(id)
  } else {
    state.selectedIds.add(id)
  }
}

// Departments list for filter
const departments = computed(() => [
  ...new Set(employees.value.map(e => e.department))
])

// Format functions
function formatSalary(salary: number): string {
  return new Intl.NumberFormat('th-TH', { style: 'currency', currency: 'THB' }).format(salary)
}

function formatDate(date: string): string {
  return new Date(date).toLocaleDateString('th-TH')
}

// Search with debounce reset page
let searchTimer: ReturnType<typeof setTimeout>
function handleSearch(value: string) {
  clearTimeout(searchTimer)
  searchTimer = setTimeout(() => {
    state.search = value
    state.pagination.page = 1
  }, 300)
}

// Bulk actions
function bulkDelete() {
  if (state.selectedIds.size === 0) return
  if (confirm(`ลบ ${state.selectedIds.size} รายการ?`)) {
    employees.value = employees.value.filter(emp => !state.selectedIds.has(emp.id))
    state.selectedIds.clear()
    state.pagination.page = 1
  }
}

function exportToCSV() {
  const selected = state.selectedIds.size > 0
    ? employees.value.filter(emp => state.selectedIds.has(emp.id))
    : filteredData.value

  const headers = ['ID', 'ชื่อ', 'อีเมล', 'แผนก', 'ตำแหน่ง', 'เงินเดือน', 'วันที่เข้าทำงาน', 'สถานะ']
  const rows = selected.map(emp => [
    emp.id, emp.name, emp.email, emp.department,
    emp.position, emp.salary, emp.joinDate, emp.status
  ])

  const csv = [headers, ...rows].map(row => row.join(',')).join('\n')
  const blob = new Blob(['﻿' + csv], { type: 'text/csv;charset=utf-8;' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = 'employees.csv'
  link.click()
  URL.revokeObjectURL(url)
}
</script>

<template>
  <div class="data-table">
    <!-- Header -->
    <div class="table-header">
      <div class="title-section">
        <h2>รายการพนักงาน</h2>
        <span class="total-count">ทั้งหมด {{ filteredData.length }} คน</span>
      </div>

      <!-- Actions -->
      <div class="actions">
        <button
          v-if="state.selectedIds.size > 0"
          @click="bulkDelete"
          class="btn-danger"
        >
          ลบที่เลือก ({{ state.selectedIds.size }})
        </button>
        <button @click="exportToCSV" class="btn-secondary">
          Export CSV
        </button>
      </div>
    </div>

    <!-- Filters -->
    <div class="filters-bar">
      <input
        placeholder="ค้นหา..."
        @input="handleSearch(($event.target as HTMLInputElement).value)"
        class="search-input"
      />

      <select v-model="state.filters.department" @change="state.pagination.page = 1">
        <option value="">ทุกแผนก</option>
        <option v-for="dept in departments" :key="dept" :value="dept">
          {{ dept }}
        </option>
      </select>

      <select v-model="state.filters.status" @change="state.pagination.page = 1">
        <option value="">ทุกสถานะ</option>
        <option value="active">Active</option>
        <option value="inactive">Inactive</option>
      </select>

      <select v-model.number="state.pagination.pageSize" @change="state.pagination.page = 1">
        <option :value="10">10 / หน้า</option>
        <option :value="25">25 / หน้า</option>
        <option :value="50">50 / หน้า</option>
      </select>
    </div>

    <!-- Table -->
    <div class="table-wrapper">
      <table>
        <thead>
          <tr>
            <th>
              <input
                type="checkbox"
                :checked="allSelected"
                @change="toggleSelectAll"
              />
            </th>
            <th
              v-for="col in columns"
              :key="col.key"
              :style="col.width ? { width: col.width } : {}"
              :class="{ sortable: col.sortable, sorted: state.sort.key === col.key }"
              @click="col.sortable && handleSort(col.key)"
            >
              {{ col.label }}
              <span v-if="col.sortable" class="sort-icon">
                <span v-if="state.sort.key === col.key">
                  {{ state.sort.order === 'asc' ? '↑' : '↓' }}
                </span>
                <span v-else>↕</span>
              </span>
            </th>
          </tr>
        </thead>

        <tbody>
          <template v-if="paginatedData.length > 0">
            <tr
              v-for="employee in paginatedData"
              :key="employee.id"
              :class="{ selected: state.selectedIds.has(employee.id) }"
            >
              <td>
                <input
                  type="checkbox"
                  :checked="state.selectedIds.has(employee.id)"
                  @change="toggleSelect(employee.id)"
                />
              </td>
              <td>{{ employee.id }}</td>
              <td>{{ employee.name }}</td>
              <td>{{ employee.email }}</td>
              <td>{{ employee.department }}</td>
              <td>{{ employee.position }}</td>
              <td>{{ formatSalary(employee.salary) }}</td>
              <td>{{ formatDate(employee.joinDate) }}</td>
              <td>
                <span :class="['badge', employee.status]">
                  {{ employee.status === 'active' ? 'ใช้งาน' : 'ไม่ใช้งาน' }}
                </span>
              </td>
            </tr>
          </template>

          <tr v-else>
            <td :colspan="columns.length + 1" class="empty-row">
              ไม่พบข้อมูล
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Pagination -->
    <div class="pagination" v-if="totalPages > 1">
      <button
        @click="state.pagination.page = 1"
        :disabled="state.pagination.page === 1"
      >
        «
      </button>
      <button
        @click="state.pagination.page--"
        :disabled="state.pagination.page === 1"
      >
        ‹
      </button>

      <template v-for="page in pageNumbers" :key="page">
        <button
          v-if="page !== '...'"
          @click="state.pagination.page = page as number"
          :class="{ active: state.pagination.page === page }"
        >
          {{ page }}
        </button>
        <span v-else class="ellipsis">...</span>
      </template>

      <button
        @click="state.pagination.page++"
        :disabled="state.pagination.page === totalPages"
      >
        ›
      </button>
      <button
        @click="state.pagination.page = totalPages"
        :disabled="state.pagination.page === totalPages"
      >
        »
      </button>

      <span class="pagination-info">
        หน้า {{ state.pagination.page }} จาก {{ totalPages }}
        ({{ filteredData.length }} รายการ)
      </span>
    </div>
  </div>
</template>

<style scoped>
.data-table { font-family: sans-serif; }
.table-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem; }
.filters-bar { display: flex; gap: 0.75rem; margin-bottom: 1rem; flex-wrap: wrap; }
.search-input { flex: 1; min-width: 200px; padding: 0.5rem 0.75rem; border: 1px solid #ddd; border-radius: 4px; }
.table-wrapper { overflow-x: auto; border: 1px solid #e5e7eb; border-radius: 8px; }
table { width: 100%; border-collapse: collapse; }
th { background: #f9fafb; padding: 0.75rem 1rem; text-align: left; font-weight: 600; border-bottom: 2px solid #e5e7eb; white-space: nowrap; }
td { padding: 0.75rem 1rem; border-bottom: 1px solid #f3f4f6; }
tr:hover td { background: #f9fafb; }
tr.selected td { background: #eff6ff; }
th.sortable { cursor: pointer; user-select: none; }
th.sortable:hover { background: #f3f4f6; }
.sort-icon { margin-left: 0.25rem; color: #6b7280; }
.badge { padding: 0.25rem 0.75rem; border-radius: 9999px; font-size: 0.75rem; font-weight: 500; }
.badge.active { background: #dcfce7; color: #15803d; }
.badge.inactive { background: #f3f4f6; color: #6b7280; }
.pagination { display: flex; align-items: center; gap: 0.25rem; margin-top: 1rem; flex-wrap: wrap; }
.pagination button { padding: 0.5rem 0.75rem; border: 1px solid #e5e7eb; background: white; border-radius: 4px; cursor: pointer; }
.pagination button:hover:not(:disabled) { background: #f3f4f6; }
.pagination button.active { background: #3b82f6; color: white; border-color: #3b82f6; }
.pagination button:disabled { opacity: 0.5; cursor: not-allowed; }
.ellipsis { padding: 0.5rem 0.25rem; }
.pagination-info { margin-left: 0.5rem; color: #6b7280; font-size: 0.875rem; }
.btn-danger { padding: 0.5rem 1rem; background: #ef4444; color: white; border: none; border-radius: 4px; cursor: pointer; }
.btn-secondary { padding: 0.5rem 1rem; background: white; border: 1px solid #e5e7eb; border-radius: 4px; cursor: pointer; }
</style>
```

---

## สรุป

| Feature | การใช้งาน |
|---------|----------|
| `v-for="item in array"` | Iterate array |
| `v-for="(item, index) in array"` | Iterate กับ index |
| `v-for="(value, key) in object"` | Iterate object |
| `v-for="n in number"` | Iterate ตัวเลข |
| `:key` | Unique identifier สำหรับ element |
| Computed + v-for | Filter/sort ก่อน render |

**Best Practices:**
- ใช้ unique stable ID เป็น `:key` เสมอ (ห้ามใช้ index กับ lists ที่มีการ reorder)
- ใช้ computed property สำหรับ filter/sort แทนการทำใน template
- ห้ามใช้ `v-if` กับ `v-for` บน element เดียวกัน
- ใช้ Virtual List สำหรับ datasets ขนาดใหญ่ (1000+ items)
- mutation methods (`push`, `pop`, `splice` ฯลฯ) จะ trigger reactivity โดยอัตโนมัติ
- สำหรับ replace array ใช้ reassignment: `items.value = items.value.filter(...)`
