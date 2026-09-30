# Part 7: Conditional Rendering

## บทนำ

Conditional Rendering ช่วยให้เราแสดงหรือซ่อน elements ตามเงื่อนไขต่างๆ Vue มี directives หลักสำหรับการนี้คือ `v-if`, `v-else-if`, `v-else` และ `v-show` ซึ่งแต่ละอันมีพฤติกรรมและ use case ที่แตกต่างกัน

---

## 1. v-if, v-else-if, v-else

`v-if` จะ render หรือ destroy element ตามเงื่อนไข

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isLoggedIn = ref(false)
const userRole = ref<'admin' | 'user' | 'guest'>('guest')
const temperature = ref(25)
const score = ref(75)
</script>

<template>
  <!-- v-if พื้นฐาน -->
  <div v-if="isLoggedIn">
    ยินดีต้อนรับ! คุณเข้าสู่ระบบแล้ว
  </div>
  <div v-else>
    กรุณาเข้าสู่ระบบก่อน
  </div>

  <!-- v-else-if สำหรับหลายเงื่อนไข -->
  <div>
    <p v-if="userRole === 'admin'" class="admin-badge">
      Admin Panel
    </p>
    <p v-else-if="userRole === 'user'" class="user-badge">
      User Dashboard
    </p>
    <p v-else class="guest-badge">
      Guest View
    </p>
  </div>

  <!-- เงื่อนไขซับซ้อน -->
  <div>
    <span v-if="temperature < 0">หนาวมาก ❄️</span>
    <span v-else-if="temperature < 15">หนาว 🧥</span>
    <span v-else-if="temperature < 25">เย็นสบาย 😊</span>
    <span v-else-if="temperature < 35">ร้อน ☀️</span>
    <span v-else>ร้อนมาก 🔥</span>
  </div>

  <!-- v-if กับ expressions ซับซ้อน -->
  <div v-if="score >= 80 && userRole !== 'guest'">
    ผ่านด้วยเกียรตินิยม
  </div>
  <div v-else-if="score >= 60">
    ผ่าน
  </div>
  <div v-else>
    ไม่ผ่าน
  </div>

  <div>
    <input type="checkbox" v-model="isLoggedIn" id="login" />
    <label for="login">เข้าสู่ระบบ</label>
    <input type="number" v-model.number="temperature" />
    <input type="range" v-model.number="score" min="0" max="100" />
    <span>คะแนน: {{ score }}</span>
  </div>
</template>
```

---

## 2. v-show vs v-if ความแตกต่าง

ความแตกต่างหลักคือ `v-show` แค่ toggle `display: none` ในขณะที่ `v-if` จริงๆ แล้ว mount/unmount element

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const showElement = ref(true)
const renderCount = ref(0)
const mountCount = ref(0)

// Component สำหรับทดสอบ lifecycle
// ChildComponent.vue จะ emit ทุกครั้งที่ mount/unmount
</script>

<template>
  <div class="comparison">
    <button @click="showElement = !showElement">Toggle</button>

    <!-- v-if: mount/unmount ทุกครั้ง - เหมาะกับ rarely changes -->
    <div v-if="showElement" class="v-if-demo">
      v-if: element นี้จะถูก mount/unmount
      (ใช้ resource น้อยกว่าเมื่อซ่อน แต่ cost มากกว่าเมื่อแสดง)
    </div>

    <!-- v-show: toggle CSS display - เหมาะกับ frequently changes -->
    <div v-show="showElement" class="v-show-demo">
      v-show: element นี้จะยังอยู่ใน DOM แต่ซ่อนด้วย display:none
      (initial render cost สูงกว่า แต่ toggle เร็วกว่า)
    </div>
  </div>
</template>
```

### เมื่อไหร่ควรใช้อะไร?

```vue
<template>
  <!-- ใช้ v-if เมื่อ: เงื่อนไขเปลี่ยนน้อย หรือ component มี heavy initialization -->
  <HeavyChart v-if="showChart" />

  <!-- ใช้ v-show เมื่อ: toggle บ่อย เช่น dropdown, tooltip -->
  <Dropdown v-show="isDropdownOpen" />

  <!-- ใช้ v-if เมื่อ: ต้องการ lazy render (ไม่ render จนกว่าจะต้องการ) -->
  <LazyContent v-if="hasBeenOpened" />

  <!-- ใช้ v-show เมื่อ: animation/transition ต้องการ element ใน DOM -->
  <AnimatedPanel v-show="isPanelOpen" />
</template>
```

| ลักษณะ | v-if | v-show |
|--------|------|--------|
| DOM | Mount/Unmount | ซ่อน/แสดงด้วย CSS |
| Initial render | Lazy | เสมอ |
| Toggle cost | สูง | ต่ำ |
| เหมาะกับ | เปลี่ยนน้อย | เปลี่ยนบ่อย |
| Lifecycle hooks | ทำงานทุกครั้ง | ไม่ทำงาน |

---

## 3. ใช้ `<template>` กับ v-if

`<template>` เป็น invisible wrapper ที่ไม่ render ใน DOM จริงๆ เหมาะสำหรับ group หลาย elements ด้วย v-if

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isAdmin = ref(true)
const showDetails = ref(false)
const user = ref({
  name: 'สมชาย',
  email: 'admin@example.com',
  role: 'admin',
  lastLogin: '2024-01-15',
})
</script>

<template>
  <div class="user-profile">
    <!-- ไม่ดี: ต้องใส่ v-if ทุก element -->
    <h2 v-if="isAdmin">Admin Panel</h2>
    <p v-if="isAdmin">จัดการระบบ</p>
    <button v-if="isAdmin">ตั้งค่าระบบ</button>

    <!-- ดีกว่า: ใช้ <template> เป็น wrapper -->
    <template v-if="isAdmin">
      <h2>Admin Panel</h2>
      <p>จัดการระบบ</p>
      <button>ตั้งค่าระบบ</button>
    </template>

    <!-- template ใช้กับ v-else ได้ -->
    <template v-else>
      <h2>User Dashboard</h2>
      <p>ยินดีต้อนรับ</p>
    </template>

    <!-- template กับ v-if ซ้อนกัน -->
    <template v-if="showDetails">
      <section class="details">
        <h3>รายละเอียด</h3>
        <template v-if="user.role === 'admin'">
          <p>Email: {{ user.email }}</p>
          <p>Last Login: {{ user.lastLogin }}</p>
        </template>
        <p>Name: {{ user.name }}</p>
      </section>
    </template>
  </div>
</template>
```

---

## 4. v-if กับ v-for (ห้ามใช้ด้วยกัน)

**ไม่ควร** ใช้ `v-if` และ `v-for` บน element เดียวกัน เพราะ `v-if` จะมี priority สูงกว่า `v-for` ซึ่งทำให้เกิด unexpected behavior

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface User {
  id: number
  name: string
  isActive: boolean
  role: string
}

const users = ref<User[]>([
  { id: 1, name: 'สมชาย', isActive: true, role: 'admin' },
  { id: 2, name: 'สมหญิง', isActive: false, role: 'user' },
  { id: 3, name: 'สมศรี', isActive: true, role: 'user' },
])
</script>

<template>
  <!-- ❌ ห้ามทำ: v-if และ v-for บน element เดียวกัน -->
  <!-- v-if จะ evaluate ก่อน v-for ทำให้ user ไม่มีค่า -->
  <!--
  <li v-for="user in users" v-if="user.isActive" :key="user.id">
    {{ user.name }}
  </li>
  -->

  <!-- ✅ วิธีที่ 1: ใช้ computed property เพื่อ filter ก่อน -->
  <ul>
    <li v-for="user in activeUsers" :key="user.id">
      {{ user.name }}
    </li>
  </ul>

  <!-- ✅ วิธีที่ 2: ใช้ <template> กับ v-for แล้วใส่ v-if ใน child -->
  <ul>
    <template v-for="user in users" :key="user.id">
      <li v-if="user.isActive">
        {{ user.name }}
      </li>
    </template>
  </ul>

  <!-- ✅ วิธีที่ 3: wrapper element กับ v-if ข้างนอก -->
  <ul v-if="users.length > 0">
    <li v-for="user in activeUsers" :key="user.id">
      {{ user.name }}
    </li>
  </ul>
  <p v-else>ไม่มีผู้ใช้</p>
</template>

<script setup lang="ts">
// computed สำหรับ filter
const activeUsers = computed(() =>
  users.value.filter(user => user.isActive)
)
</script>
```

---

## 5. การทำ Conditional Loading States

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

type LoadingState = 'idle' | 'loading' | 'success' | 'error'

interface Post {
  id: number
  title: string
  body: string
  author: string
}

const state = ref<LoadingState>('idle')
const posts = ref<Post[]>([])
const error = ref<string | null>(null)
const retryCount = ref(0)

async function fetchPosts() {
  state.value = 'loading'
  error.value = null

  try {
    await new Promise(resolve => setTimeout(resolve, 1500)) // simulate delay

    // Simulate random error
    if (Math.random() < 0.3 && retryCount.value < 2) {
      throw new Error('Network error: Failed to fetch')
    }

    posts.value = [
      { id: 1, title: 'Vue 3 Composition API', body: 'เนื้อหาบทความ...', author: 'สมชาย' },
      { id: 2, title: 'TypeScript Best Practices', body: 'เนื้อหาบทความ...', author: 'สมหญิง' },
      { id: 3, title: 'Nuxt.js Introduction', body: 'เนื้อหาบทความ...', author: 'สมศรี' },
    ]
    state.value = 'success'
    retryCount.value = 0
  } catch (e: any) {
    state.value = 'error'
    error.value = e.message
  }
}

async function retry() {
  retryCount.value++
  await fetchPosts()
}

onMounted(() => {
  fetchPosts()
})
</script>

<template>
  <div class="posts-container">
    <!-- Loading state -->
    <div v-if="state === 'loading'" class="loading-state">
      <div class="spinner"></div>
      <p>กำลังโหลดบทความ...</p>
      <!-- Skeleton loading -->
      <div class="skeleton-list">
        <div v-for="i in 3" :key="i" class="skeleton-item">
          <div class="skeleton-title"></div>
          <div class="skeleton-body"></div>
        </div>
      </div>
    </div>

    <!-- Error state -->
    <div v-else-if="state === 'error'" class="error-state">
      <div class="error-icon">⚠️</div>
      <h3>เกิดข้อผิดพลาด</h3>
      <p>{{ error }}</p>
      <button @click="retry" class="retry-btn">
        ลองใหม่ ({{ retryCount }}/3)
      </button>
    </div>

    <!-- Empty state -->
    <div v-else-if="state === 'success' && posts.length === 0" class="empty-state">
      <div class="empty-icon">📭</div>
      <h3>ไม่มีบทความ</h3>
      <p>ยังไม่มีบทความในขณะนี้</p>
      <button @click="fetchPosts">รีเฟรช</button>
    </div>

    <!-- Success state -->
    <div v-else-if="state === 'success'" class="posts-list">
      <article
        v-for="post in posts"
        :key="post.id"
        class="post-card"
      >
        <h3>{{ post.title }}</h3>
        <p>{{ post.body }}</p>
        <span class="author">โดย {{ post.author }}</span>
      </article>
    </div>

    <!-- Idle state -->
    <div v-else class="idle-state">
      <button @click="fetchPosts">โหลดบทความ</button>
    </div>
  </div>
</template>

<style scoped>
.spinner { width: 40px; height: 40px; border: 4px solid #f3f3f3; border-top: 4px solid #3b82f6; border-radius: 50%; animation: spin 1s linear infinite; }
@keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
.skeleton-item { background: #f3f4f6; border-radius: 8px; padding: 1rem; margin-bottom: 1rem; }
.skeleton-title { height: 20px; background: #e5e7eb; border-radius: 4px; margin-bottom: 0.5rem; animation: pulse 1.5s ease-in-out infinite; }
.skeleton-body { height: 60px; background: #e5e7eb; border-radius: 4px; animation: pulse 1.5s ease-in-out infinite; }
@keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }
</style>
```

---

## 6. ตัวอย่าง: Permission-based UI

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

type Permission =
  | 'read:posts'
  | 'write:posts'
  | 'delete:posts'
  | 'read:users'
  | 'write:users'
  | 'delete:users'
  | 'manage:settings'

interface User {
  id: number
  name: string
  role: 'super_admin' | 'admin' | 'editor' | 'viewer'
  permissions: Permission[]
}

const rolePermissions: Record<User['role'], Permission[]> = {
  super_admin: [
    'read:posts', 'write:posts', 'delete:posts',
    'read:users', 'write:users', 'delete:users',
    'manage:settings',
  ],
  admin: [
    'read:posts', 'write:posts', 'delete:posts',
    'read:users', 'write:users',
  ],
  editor: ['read:posts', 'write:posts'],
  viewer: ['read:posts'],
}

const currentUser = ref<User>({
  id: 1,
  name: 'สมชาย ใจดี',
  role: 'editor',
  permissions: rolePermissions['editor'],
})

function hasPermission(permission: Permission): boolean {
  return currentUser.value.permissions.includes(permission)
}

function hasAnyPermission(permissions: Permission[]): boolean {
  return permissions.some(p => currentUser.value.permissions.includes(p))
}

function switchRole(role: User['role']) {
  currentUser.value.role = role
  currentUser.value.permissions = rolePermissions[role]
}

const canManagePosts = computed(() =>
  hasAnyPermission(['write:posts', 'delete:posts'])
)

const canManageUsers = computed(() =>
  hasAnyPermission(['write:users', 'delete:users'])
)
</script>

<template>
  <div class="permission-demo">
    <!-- Role switcher -->
    <div class="role-switcher">
      <span>บทบาทปัจจุบัน: <strong>{{ currentUser.role }}</strong></span>
      <div class="role-buttons">
        <button
          v-for="role in ['super_admin', 'admin', 'editor', 'viewer'] as const"
          :key="role"
          @click="switchRole(role)"
          :class="{ active: currentUser.role === role }"
        >
          {{ role }}
        </button>
      </div>
    </div>

    <!-- Navigation menu based on permissions -->
    <nav class="sidebar">
      <ul>
        <!-- ทุกคนเห็น -->
        <li>
          <a href="#">Dashboard</a>
        </li>

        <!-- ต้องการ read:posts -->
        <li v-if="hasPermission('read:posts')">
          <a href="#">บทความ</a>
          <ul v-if="canManagePosts">
            <li v-if="hasPermission('write:posts')">
              <a href="#">สร้างบทความใหม่</a>
            </li>
            <li v-if="hasPermission('delete:posts')">
              <a href="#">จัดการบทความ</a>
            </li>
          </ul>
        </li>

        <!-- ต้องการ read:users -->
        <li v-if="hasPermission('read:users')">
          <a href="#">ผู้ใช้งาน</a>
          <ul v-if="canManageUsers">
            <li v-if="hasPermission('write:users')">
              <a href="#">เพิ่มผู้ใช้</a>
            </li>
            <li v-if="hasPermission('delete:users')">
              <a href="#">ลบผู้ใช้</a>
            </li>
          </ul>
        </li>

        <!-- เฉพาะ super_admin -->
        <li v-if="hasPermission('manage:settings')">
          <a href="#">ตั้งค่าระบบ</a>
        </li>
      </ul>
    </nav>

    <!-- Content area -->
    <main class="content">
      <template v-if="hasPermission('read:posts')">
        <div class="posts-section">
          <h2>รายการบทความ</h2>

          <div class="action-bar">
            <button
              v-if="hasPermission('write:posts')"
              class="btn-primary"
            >
              + สร้างบทความใหม่
            </button>
          </div>

          <table class="posts-table">
            <thead>
              <tr>
                <th>ชื่อบทความ</th>
                <th>ผู้เขียน</th>
                <th v-if="canManagePosts">Actions</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="i in 3" :key="i">
                <td>บทความที่ {{ i }}</td>
                <td>สมชาย</td>
                <td v-if="canManagePosts">
                  <button v-if="hasPermission('write:posts')" class="btn-edit">แก้ไข</button>
                  <button v-if="hasPermission('delete:posts')" class="btn-delete">ลบ</button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </template>

      <div v-else class="no-access">
        <h3>ไม่มีสิทธิ์เข้าถึง</h3>
        <p>คุณไม่มีสิทธิ์ในการดูเนื้อหานี้</p>
      </div>
    </main>
  </div>
</template>
```

---

## 7. ตัวอย่าง: Multi-step Wizard

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface WizardStep {
  id: number
  title: string
  description: string
  isCompleted: boolean
  canSkip: boolean
}

interface WizardData {
  // Step 1: เป้าหมาย
  goals: string[]

  // Step 2: ระดับ
  level: 'beginner' | 'intermediate' | 'advanced' | ''

  // Step 3: เวลา
  hoursPerWeek: number

  // Step 4: ยืนยัน
  agreed: boolean
}

const steps = ref<WizardStep[]>([
  { id: 1, title: 'เป้าหมาย', description: 'เลือกเป้าหมายการเรียน', isCompleted: false, canSkip: false },
  { id: 2, title: 'ระดับความสามารถ', description: 'ประเมินระดับปัจจุบัน', isCompleted: false, canSkip: false },
  { id: 3, title: 'ตารางเวลา', description: 'วางแผนเวลาเรียน', isCompleted: false, canSkip: true },
  { id: 4, title: 'ยืนยัน', description: 'ตรวจสอบและยืนยัน', isCompleted: false, canSkip: false },
])

const currentStepIndex = ref(0)
const isCompleted = ref(false)

const data = ref<WizardData>({
  goals: [],
  level: '',
  hoursPerWeek: 5,
  agreed: false,
})

const currentStep = computed(() => steps.value[currentStepIndex.value])
const isFirstStep = computed(() => currentStepIndex.value === 0)
const isLastStep = computed(() => currentStepIndex.value === steps.value.length - 1)

const progress = computed(() =>
  Math.round((currentStepIndex.value / (steps.value.length - 1)) * 100)
)

const canProceed = computed(() => {
  switch (currentStep.value.id) {
    case 1: return data.value.goals.length > 0
    case 2: return data.value.level !== ''
    case 3: return data.value.hoursPerWeek > 0
    case 4: return data.value.agreed
    default: return false
  }
})

function goToStep(index: number) {
  if (index >= 0 && index < steps.value.length) {
    currentStepIndex.value = index
  }
}

function nextStep() {
  if (canProceed.value || currentStep.value.canSkip) {
    steps.value[currentStepIndex.value].isCompleted = true
    if (!isLastStep.value) {
      currentStepIndex.value++
    } else {
      isCompleted.value = true
    }
  }
}

function prevStep() {
  if (!isFirstStep.value) {
    currentStepIndex.value--
  }
}

const goalOptions = [
  'เรียนรู้ Vue.js', 'สร้าง Portfolio', 'หางานใหม่',
  'เลื่อนตำแหน่ง', 'สร้าง Side Project', 'เพิ่มทักษะ'
]
</script>

<template>
  <div class="wizard">
    <div v-if="isCompleted" class="wizard-complete">
      <div class="complete-icon">🎉</div>
      <h2>เริ่มต้นการเรียนรู้!</h2>
      <p>เป้าหมาย: {{ data.goals.join(', ') }}</p>
      <p>ระดับ: {{ data.level }}</p>
      <p>เวลา: {{ data.hoursPerWeek }} ชั่วโมง/สัปดาห์</p>
    </div>

    <template v-else>
      <!-- Progress bar -->
      <div class="progress-bar">
        <div class="progress-fill" :style="{ width: `${progress}%` }"></div>
      </div>

      <!-- Step indicators -->
      <div class="step-indicators">
        <button
          v-for="(step, index) in steps"
          :key="step.id"
          @click="step.isCompleted ? goToStep(index) : null"
          class="step-indicator"
          :class="{
            active: index === currentStepIndex,
            completed: step.isCompleted,
            clickable: step.isCompleted
          }"
        >
          <span class="step-dot">
            <span v-if="step.isCompleted">✓</span>
            <span v-else>{{ step.id }}</span>
          </span>
          <span class="step-title">{{ step.title }}</span>
        </button>
      </div>

      <!-- Step content -->
      <div class="step-content">
        <!-- Step 1: Goals -->
        <div v-if="currentStep.id === 1">
          <h3>เป้าหมายของคุณคืออะไร?</h3>
          <p>เลือกได้มากกว่า 1 ข้อ</p>
          <div class="options-grid">
            <button
              v-for="goal in goalOptions"
              :key="goal"
              @click="data.goals.includes(goal)
                ? data.goals.splice(data.goals.indexOf(goal), 1)
                : data.goals.push(goal)"
              class="option-btn"
              :class="{ selected: data.goals.includes(goal) }"
            >
              {{ goal }}
            </button>
          </div>
          <p v-if="data.goals.length === 0" class="hint">
            กรุณาเลือกอย่างน้อย 1 เป้าหมาย
          </p>
        </div>

        <!-- Step 2: Level -->
        <div v-else-if="currentStep.id === 2">
          <h3>ระดับความสามารถปัจจุบัน?</h3>
          <div class="level-options">
            <label
              v-for="option in [
                { value: 'beginner', label: 'มือใหม่', desc: 'ยังไม่มีประสบการณ์ HTML/CSS/JS' },
                { value: 'intermediate', label: 'กลาง', desc: 'รู้ JavaScript พื้นฐาน' },
                { value: 'advanced', label: 'สูง', desc: 'มีประสบการณ์ framework อื่น' }
              ] as const"
              :key="option.value"
              class="level-option"
              :class="{ selected: data.level === option.value }"
            >
              <input type="radio" v-model="data.level" :value="option.value" />
              <div class="level-info">
                <strong>{{ option.label }}</strong>
                <span>{{ option.desc }}</span>
              </div>
            </label>
          </div>
        </div>

        <!-- Step 3: Schedule -->
        <div v-else-if="currentStep.id === 3">
          <h3>มีเวลาเรียนกี่ชั่วโมงต่อสัปดาห์?</h3>
          <div class="schedule-options">
            <input
              type="range"
              v-model.number="data.hoursPerWeek"
              min="1"
              max="40"
            />
            <p class="hours-display">{{ data.hoursPerWeek }} ชั่วโมง/สัปดาห์</p>

            <div class="estimate">
              <template v-if="data.hoursPerWeek < 5">
                <p>คาดว่าจะเรียนจบใน <strong>6-8 เดือน</strong></p>
              </template>
              <template v-else-if="data.hoursPerWeek < 15">
                <p>คาดว่าจะเรียนจบใน <strong>3-4 เดือน</strong></p>
              </template>
              <template v-else>
                <p>คาดว่าจะเรียนจบใน <strong>1-2 เดือน</strong></p>
              </template>
            </div>
          </div>
        </div>

        <!-- Step 4: Confirm -->
        <div v-else-if="currentStep.id === 4">
          <h3>ยืนยันข้อมูล</h3>
          <div class="summary">
            <div class="summary-item">
              <span class="label">เป้าหมาย:</span>
              <span>{{ data.goals.join(', ') }}</span>
            </div>
            <div class="summary-item">
              <span class="label">ระดับ:</span>
              <span>{{ data.level }}</span>
            </div>
            <div class="summary-item">
              <span class="label">เวลา:</span>
              <span>{{ data.hoursPerWeek }} ชั่วโมง/สัปดาห์</span>
            </div>
          </div>
          <label class="agree-label">
            <input type="checkbox" v-model="data.agreed" />
            ฉันยืนยันข้อมูลด้านบนและพร้อมเริ่มเรียน
          </label>
        </div>
      </div>

      <!-- Navigation -->
      <div class="wizard-actions">
        <button v-if="!isFirstStep" @click="prevStep" class="btn-secondary">
          ← ก่อนหน้า
        </button>

        <div class="right-actions">
          <button
            v-if="currentStep.canSkip"
            @click="nextStep"
            class="btn-text"
          >
            ข้าม →
          </button>

          <button
            @click="nextStep"
            :disabled="!canProceed && !currentStep.canSkip"
            class="btn-primary"
          >
            {{ isLastStep ? 'เริ่มเรียน! 🚀' : 'ถัดไป →' }}
          </button>
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.wizard { max-width: 600px; margin: 0 auto; padding: 2rem; }
.progress-bar { height: 4px; background: #e5e7eb; border-radius: 2px; margin-bottom: 2rem; }
.progress-fill { height: 100%; background: #3b82f6; border-radius: 2px; transition: width 0.3s; }
.step-indicators { display: flex; justify-content: space-between; margin-bottom: 2rem; }
.step-indicator { display: flex; flex-direction: column; align-items: center; gap: 0.25rem; background: none; border: none; cursor: default; }
.step-indicator.clickable { cursor: pointer; }
.step-dot { width: 32px; height: 32px; border-radius: 50%; border: 2px solid #ddd; display: flex; align-items: center; justify-content: center; }
.step-indicator.active .step-dot { border-color: #3b82f6; background: #3b82f6; color: white; }
.step-indicator.completed .step-dot { background: #10b981; border-color: #10b981; color: white; }
.options-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 0.5rem; margin: 1rem 0; }
.option-btn { padding: 0.75rem; border: 2px solid #e5e7eb; border-radius: 8px; cursor: pointer; background: white; transition: all 0.2s; }
.option-btn.selected { border-color: #3b82f6; background: #eff6ff; color: #1d4ed8; }
.level-option { display: flex; align-items: center; gap: 1rem; padding: 1rem; border: 2px solid #e5e7eb; border-radius: 8px; cursor: pointer; margin-bottom: 0.75rem; }
.level-option.selected { border-color: #3b82f6; background: #eff6ff; }
.wizard-actions { display: flex; justify-content: space-between; margin-top: 2rem; }
.btn-primary { padding: 0.75rem 1.5rem; background: #3b82f6; color: white; border: none; border-radius: 6px; cursor: pointer; }
.btn-primary:disabled { opacity: 0.5; cursor: not-allowed; }
.btn-secondary { padding: 0.75rem 1.5rem; background: #f3f4f6; border: none; border-radius: 6px; cursor: pointer; }
.btn-text { padding: 0.75rem; background: none; border: none; cursor: pointer; color: #6b7280; }
.right-actions { display: flex; gap: 0.5rem; }
</style>
```

---

## สรุป

| Feature | การใช้งาน |
|---------|----------|
| `v-if` | Render/remove element ตามเงื่อนไข |
| `v-else-if` | เงื่อนไขต่อเนื่องจาก v-if |
| `v-else` | ทางเลือกสุดท้ายเมื่อเงื่อนไขอื่นไม่ตรง |
| `v-show` | ซ่อน/แสดง element ด้วย CSS display |
| `<template>` | Invisible wrapper สำหรับ group elements |

**Best Practices:**
- ใช้ `v-if` สำหรับเงื่อนไขที่เปลี่ยนน้อย หรือ component ที่ initialize cost สูง
- ใช้ `v-show` สำหรับ toggle ที่เกิดบ่อย
- ห้ามใช้ `v-if` และ `v-for` บน element เดียวกัน ใช้ computed property แทน
- ใช้ `<template>` เมื่อต้องการ group หลาย elements โดยไม่เพิ่ม wrapper ใน DOM
- สร้าง loading/error/empty states สำหรับ async data เสมอ
