# Part 18: Route Guards

## บทนำ

Route Guards คือ hooks ที่ Vue Router เรียกใช้เมื่อมีการ navigate ระหว่าง routes ใช้สำหรับ authentication, authorization, logging และอื่นๆ

---

## 1. Global Guards (beforeEach, afterEach)

Global Guards ทำงานทุกครั้งที่มีการ navigate ไม่ว่าจะไปยัง route ไหน

### beforeEach - ตรวจสอบก่อน navigate

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '../stores/auth'

const router = createRouter({ ... })

// Global beforeEach guard
router.beforeEach(async (to, from, next) => {
  // to: Route ที่กำลังจะไป
  // from: Route ที่มาจาก
  // next: function สำหรับ proceed หรือ redirect
  
  console.log(`Navigating from ${from.path} to ${to.path}`)
  
  // ตัดสินใจว่าจะทำอะไร:
  // next()           → proceed ต่อ
  // next(false)      → abort navigation
  // next('/login')   → redirect ไป /login
  // next({ name: 'login' }) → redirect ไป route ชื่อ login
  // throw Error      → เรียก onError handler
  
  next()
})

// Global beforeResolve - เรียกหลัง component hooks แต่ก่อน navigation confirm
router.beforeResolve(async (to) => {
  if (to.meta.requiresCamera) {
    try {
      await navigator.mediaDevices.getUserMedia({ video: true })
    } catch {
      if (to.meta.requiresCamera) {
        return false // abort navigation
      }
    }
  }
})

// Global afterEach - เรียกหลัง navigation เสร็จ
router.afterEach((to, from, failure) => {
  if (failure) {
    console.log('Navigation failed:', failure)
  } else {
    // Analytics tracking
    trackPageView(to.fullPath)
    
    // อัปเดต page title
    document.title = to.meta.title || 'My App'
  }
})

export default router
```

---

## 2. Per-route Guards

Guards ที่กำหนดเฉพาะ route นั้น

```javascript
// router/index.js
const routes = [
  {
    path: '/admin',
    component: AdminView,
    
    // beforeEnter: เรียกก่อน enter route นี้
    beforeEnter: (to, from, next) => {
      const authStore = useAuthStore()
      
      if (!authStore.isAdmin) {
        next({ name: 'home', query: { error: 'unauthorized' } })
        return
      }
      
      next()
    }
  },
  
  // ใช้ array ของ guard functions
  {
    path: '/premium',
    component: PremiumView,
    beforeEnter: [checkAuth, checkPremium, logAccess]
  }
]

// Guard functions แยก reusable
function checkAuth(to, from, next) {
  const authStore = useAuthStore()
  if (!authStore.isAuthenticated) {
    next({ name: 'login', query: { redirect: to.fullPath } })
    return
  }
  next()
}

function checkPremium(to, from, next) {
  const authStore = useAuthStore()
  if (!authStore.user?.isPremium) {
    next({ name: 'upgrade' })
    return
  }
  next()
}

function logAccess(to, from, next) {
  console.log(`Premium access: ${to.path}`)
  next()
}
```

---

## 3. Component Guards

Guards ที่อยู่ใน component ตรงๆ

```vue
<!-- views/UserEditView.vue -->
<template>
  <div>
    <h2>แก้ไขโปรไฟล์</h2>
    <form @submit.prevent="save">
      <input v-model="form.name" placeholder="ชื่อ" />
      <input v-model="form.email" type="email" placeholder="อีเมล" />
      <button type="submit" :disabled="!isDirty || saving">บันทึก</button>
    </form>
  </div>
</template>

<script>
// Component guards ใช้กับ Options API เท่านั้น
export default {
  name: 'UserEditView',
  
  // เรียกก่อน component ถูก render ครั้งแรก
  // ใช้สำหรับ fetch ข้อมูลก่อนแสดง component
  beforeRouteEnter(to, from, next) {
    // ❌ ยังไม่สามารถเข้าถึง this ได้
    console.log('Before enter:', to.params.id)
    
    // ✅ ใช้ next callback เพื่อเข้าถึง component instance
    next(vm => {
      vm.loadUserData(to.params.id)
    })
  },
  
  // เรียกเมื่อ navigate ไปยัง route เดิม แต่ params เปลี่ยน
  // เช่น navigate จาก /users/1 ไปยัง /users/2
  beforeRouteUpdate(to, from, next) {
    this.loadUserData(to.params.id)
    next()
  },
  
  // เรียกก่อน navigate ออกจาก route นี้
  // ดักจับการออกเมื่อมีข้อมูลที่ยังไม่ได้บันทึก
  beforeRouteLeave(to, from, next) {
    if (this.isDirty) {
      const answer = window.confirm(
        'คุณมีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก ต้องการออกหรือไม่?'
      )
      if (!answer) {
        next(false) // ยกเลิกการ navigate
        return
      }
    }
    next()
  },
  
  methods: {
    loadUserData(id) {
      // fetch user data
    }
  }
}
</script>
```

### Component Guards ใน Composition API

```vue
<!-- views/ArticleEditView.vue -->
<template>
  <div>
    <h2>แก้ไขบทความ</h2>
    <!-- form content -->
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { onBeforeRouteLeave, onBeforeRouteUpdate } from 'vue-router'

const isDirty = ref(false)
const saving = ref(false)

// Guard สำหรับ Composition API
onBeforeRouteLeave((to, from) => {
  if (isDirty.value) {
    const confirmed = window.confirm(
      'มีการเปลี่ยนแปลงที่ยังไม่บันทึก ต้องการออกไปหรือไม่?'
    )
    if (!confirmed) {
      return false // abort navigation
    }
  }
})

// เรียกเมื่อ params เปลี่ยนแต่ component เดิม
onBeforeRouteUpdate((to, from) => {
  console.log('Route updated, new params:', to.params)
  // โหลดข้อมูลใหม่ตาม params
})
</script>
```

---

## 4. Authentication Guard

```javascript
// router/guards/auth.js
import { useAuthStore } from '../../stores/auth'

export async function authGuard(to, from, next) {
  const authStore = useAuthStore()
  
  // รอ initialize auth ให้เสร็จก่อน
  if (!authStore.initialized) {
    await authStore.initialize()
  }
  
  if (!authStore.isAuthenticated) {
    // บันทึก redirect URL เพื่อ redirect กลับหลัง login
    next({
      name: 'login',
      query: { redirect: to.fullPath }
    })
    return
  }
  
  next()
}

export function guestGuard(to, from, next) {
  const authStore = useAuthStore()
  
  if (authStore.isAuthenticated) {
    // ถ้า login แล้วและพยายามไป login page → redirect ไป dashboard
    const redirectPath = to.query.redirect || '/dashboard'
    next(redirectPath)
    return
  }
  
  next()
}
```

### สร้าง Auth Store

```javascript
// stores/auth.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useAuthStore = defineStore('auth', () => {
  const user = ref(null)
  const token = ref(localStorage.getItem('auth_token'))
  const initialized = ref(false)
  
  const isAuthenticated = computed(() => !!token.value && !!user.value)
  const isAdmin = computed(() => user.value?.role === 'admin')
  
  async function initialize() {
    if (initialized.value) return
    
    if (token.value) {
      try {
        const response = await fetch('/api/auth/me', {
          headers: { Authorization: `Bearer ${token.value}` }
        })
        
        if (response.ok) {
          user.value = await response.json()
        } else {
          clearAuth()
        }
      } catch {
        clearAuth()
      }
    }
    
    initialized.value = true
  }
  
  async function login(email, password) {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    })
    
    if (!response.ok) throw new Error('Login failed')
    
    const data = await response.json()
    token.value = data.token
    user.value = data.user
    localStorage.setItem('auth_token', data.token)
    
    return data
  }
  
  async function logout() {
    try {
      await fetch('/api/auth/logout', { method: 'POST' })
    } finally {
      clearAuth()
    }
  }
  
  function clearAuth() {
    token.value = null
    user.value = null
    localStorage.removeItem('auth_token')
  }
  
  return {
    user, token, initialized,
    isAuthenticated, isAdmin,
    initialize, login, logout
  }
})
```

---

## 5. Role-based Authorization

```javascript
// router/guards/rbac.js (Role-Based Access Control)
import { useAuthStore } from '../../stores/auth'

// กำหนดสิทธิ์สำหรับแต่ละ role
const rolePermissions = {
  admin: ['*'], // admin สามารถทำทุกอย่าง
  teacher: [
    'courses:create',
    'courses:edit',
    'courses:view',
    'students:view'
  ],
  student: [
    'courses:view',
    'profile:edit'
  ],
  guest: []
}

export function hasPermission(user, permission) {
  if (!user) return false
  
  const userRole = user.role || 'guest'
  const permissions = rolePermissions[userRole] || []
  
  // Admin มีทุก permission
  if (permissions.includes('*')) return true
  
  // ตรวจสอบ exact permission
  if (permissions.includes(permission)) return true
  
  // ตรวจสอบ wildcard permission (เช่น 'courses:*')
  const [resource] = permission.split(':')
  if (permissions.includes(`${resource}:*`)) return true
  
  return false
}

export function roleGuard(requiredRole) {
  return (to, from, next) => {
    const authStore = useAuthStore()
    
    if (!authStore.isAuthenticated) {
      next({ name: 'login', query: { redirect: to.fullPath } })
      return
    }
    
    const userRole = authStore.user?.role
    
    if (Array.isArray(requiredRole)) {
      if (!requiredRole.includes(userRole)) {
        next({ name: 'unauthorized' })
        return
      }
    } else {
      if (userRole !== requiredRole) {
        next({ name: 'unauthorized' })
        return
      }
    }
    
    next()
  }
}

export function permissionGuard(requiredPermission) {
  return (to, from, next) => {
    const authStore = useAuthStore()
    
    if (!hasPermission(authStore.user, requiredPermission)) {
      next({ name: 'unauthorized' })
      return
    }
    
    next()
  }
}
```

### ใช้ Role Guards ใน Routes

```javascript
// router/index.js
import { authGuard, guestGuard } from './guards/auth'
import { roleGuard, permissionGuard } from './guards/rbac'

const routes = [
  {
    path: '/login',
    component: LoginView,
    beforeEnter: guestGuard  // สำหรับ guest เท่านั้น
  },
  {
    path: '/dashboard',
    component: DashboardView,
    beforeEnter: authGuard  // ต้อง login ก่อน
  },
  {
    path: '/admin',
    component: AdminView,
    beforeEnter: roleGuard('admin')  // admin เท่านั้น
  },
  {
    path: '/teacher',
    component: TeacherView,
    beforeEnter: roleGuard(['admin', 'teacher'])  // admin หรือ teacher
  },
  {
    path: '/courses/create',
    component: CreateCourseView,
    beforeEnter: permissionGuard('courses:create')
  }
]
```

---

## 6. เก็บ redirect URL

```vue
<!-- views/auth/LoginView.vue -->
<template>
  <div class="login-page">
    <div class="login-card">
      <h1>เข้าสู่ระบบ</h1>
      
      <form @submit.prevent="handleLogin">
        <div class="form-group">
          <label>อีเมล</label>
          <input v-model="email" type="email" required />
        </div>
        <div class="form-group">
          <label>รหัสผ่าน</label>
          <input v-model="password" type="password" required />
        </div>
        
        <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
        
        <button type="submit" :disabled="loading">
          {{ loading ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ' }}
        </button>
      </form>
      
      <div class="links">
        <RouterLink to="/forgot-password">ลืมรหัสผ่าน?</RouterLink>
        <RouterLink to="/register">สมัครสมาชิก</RouterLink>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useAuthStore } from '../../stores/auth'

const router = useRouter()
const route = useRoute()
const authStore = useAuthStore()

const email = ref('')
const password = ref('')
const loading = ref(false)
const errorMessage = ref('')

async function handleLogin() {
  loading.value = true
  errorMessage.value = ''
  
  try {
    await authStore.login(email.value, password.value)
    
    // ✅ Redirect กลับไปยัง URL ที่ต้องการก่อน login
    const redirectPath = route.query.redirect || '/dashboard'
    
    // ป้องกัน open redirect (ตรวจสอบว่าเป็น internal URL)
    if (redirectPath.startsWith('/') && !redirectPath.startsWith('//')) {
      router.push(redirectPath)
    } else {
      router.push('/dashboard')
    }
    
  } catch (error) {
    errorMessage.value = error.message || 'เข้าสู่ระบบล้มเหลว กรุณาลองใหม่'
  } finally {
    loading.value = false
  }
}
</script>
```

---

## 7. ตัวอย่าง: Protected Routes และ Admin Area

### Router Configuration สมบูรณ์

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '../stores/auth'

const routes = [
  // ===== Public Routes =====
  {
    path: '/',
    name: 'home',
    component: () => import('../views/HomeView.vue')
  },
  
  // Auth
  {
    path: '/login',
    name: 'login',
    component: () => import('../views/auth/LoginView.vue'),
    meta: { layout: 'auth', guestOnly: true }
  },
  {
    path: '/register',
    name: 'register',
    component: () => import('../views/auth/RegisterView.vue'),
    meta: { layout: 'auth', guestOnly: true }
  },
  {
    path: '/forgot-password',
    name: 'forgot-password',
    component: () => import('../views/auth/ForgotPasswordView.vue'),
    meta: { layout: 'auth' }
  },
  
  // Errors
  {
    path: '/unauthorized',
    name: 'unauthorized',
    component: () => import('../views/errors/UnauthorizedView.vue')
  },
  
  // ===== Protected User Routes =====
  {
    path: '/dashboard',
    component: () => import('../layouts/UserLayout.vue'),
    meta: { requiresAuth: true },
    children: [
      {
        path: '',
        name: 'dashboard',
        component: () => import('../views/user/DashboardView.vue')
      },
      {
        path: 'courses',
        name: 'my-courses',
        component: () => import('../views/user/MyCoursesView.vue')
      },
      {
        path: 'progress',
        name: 'progress',
        component: () => import('../views/user/ProgressView.vue')
      },
      {
        path: 'profile',
        name: 'profile',
        component: () => import('../views/user/ProfileView.vue')
      },
      {
        path: 'settings',
        name: 'settings',
        component: () => import('../views/user/SettingsView.vue')
      }
    ]
  },
  
  // ===== Admin Routes =====
  {
    path: '/admin',
    component: () => import('../layouts/AdminLayout.vue'),
    meta: { requiresAuth: true, requiredRole: 'admin' },
    children: [
      {
        path: '',
        name: 'admin-dashboard',
        component: () => import('../views/admin/DashboardView.vue')
      },
      {
        path: 'users',
        name: 'admin-users',
        component: () => import('../views/admin/UsersView.vue'),
        meta: { breadcrumb: 'จัดการผู้ใช้' }
      },
      {
        path: 'users/:id',
        name: 'admin-user-detail',
        component: () => import('../views/admin/UserDetailView.vue'),
        props: true,
        meta: { breadcrumb: 'รายละเอียดผู้ใช้' }
      },
      {
        path: 'courses',
        name: 'admin-courses',
        component: () => import('../views/admin/CoursesView.vue')
      },
      {
        path: 'courses/create',
        name: 'admin-course-create',
        component: () => import('../views/admin/CourseCreateView.vue')
      },
      {
        path: 'courses/:id/edit',
        name: 'admin-course-edit',
        component: () => import('../views/admin/CourseEditView.vue'),
        props: true
      },
      {
        path: 'reports',
        name: 'admin-reports',
        component: () => import('../views/admin/ReportsView.vue'),
        meta: { requiredRole: 'admin' }
      }
    ]
  },
  
  // 404
  {
    path: '/:pathMatch(.*)*',
    name: 'not-found',
    component: () => import('../views/errors/NotFoundView.vue')
  }
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes
})

// ===== Navigation Guards =====
let isInitialized = false

router.beforeEach(async (to, from, next) => {
  const authStore = useAuthStore()
  
  // Initialize auth state ครั้งแรก
  if (!isInitialized) {
    await authStore.initialize()
    isInitialized = true
  }
  
  // ตรวจสอบ guestOnly routes
  if (to.meta.guestOnly && authStore.isAuthenticated) {
    next({ name: 'dashboard' })
    return
  }
  
  // ตรวจสอบ requiresAuth
  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    next({ name: 'login', query: { redirect: to.fullPath } })
    return
  }
  
  // ตรวจสอบ role (ทั้ง route และ parent routes)
  const requiredRole = to.matched.find(r => r.meta.requiredRole)?.meta.requiredRole
  
  if (requiredRole && authStore.user?.role !== requiredRole) {
    next({ name: 'unauthorized' })
    return
  }
  
  next()
})

// อัปเดต page title
router.afterEach((to) => {
  document.title = to.meta.title
    ? `${to.meta.title} - My App`
    : 'My App'
})

export default router
```

### Admin Dashboard Component

```vue
<!-- views/admin/DashboardView.vue -->
<template>
  <div class="admin-dashboard">
    <div class="page-header">
      <h1>Admin Dashboard</h1>
      <p>ยินดีต้อนรับ, {{ user?.name }}</p>
    </div>
    
    <!-- Stats Cards -->
    <div class="stats-grid">
      <div class="stat-card" v-for="stat in stats" :key="stat.id">
        <span class="stat-icon">{{ stat.icon }}</span>
        <div class="stat-info">
          <span class="stat-value">{{ stat.value }}</span>
          <span class="stat-label">{{ stat.label }}</span>
        </div>
        <RouterLink :to="stat.link" class="stat-link">ดูทั้งหมด →</RouterLink>
      </div>
    </div>
    
    <!-- Quick Actions -->
    <div class="quick-actions">
      <h2>Quick Actions</h2>
      <div class="action-grid">
        <RouterLink :to="{ name: 'admin-course-create' }" class="action-btn">
          + สร้างคอร์สใหม่
        </RouterLink>
        <RouterLink :to="{ name: 'admin-users' }" class="action-btn">
          👥 จัดการผู้ใช้
        </RouterLink>
        <RouterLink :to="{ name: 'admin-reports' }" class="action-btn">
          📊 ดูรายงาน
        </RouterLink>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useAuthStore } from '../../stores/auth'
import { storeToRefs } from 'pinia'

const authStore = useAuthStore()
const { user } = storeToRefs(authStore)

const stats = ref([
  { id: 1, icon: '👥', label: 'ผู้ใช้ทั้งหมด', value: '...', link: { name: 'admin-users' } },
  { id: 2, icon: '📚', label: 'คอร์สทั้งหมด', value: '...', link: { name: 'admin-courses' } },
  { id: 3, icon: '🎓', label: 'นักเรียน', value: '...', link: { name: 'admin-users' } },
  { id: 4, icon: '💰', label: 'รายได้เดือนนี้', value: '...', link: { name: 'admin-reports' } }
])

onMounted(async () => {
  const response = await fetch('/api/admin/stats', {
    headers: { Authorization: `Bearer ${authStore.token}` }
  })
  const data = await response.json()
  
  stats.value[0].value = data.totalUsers
  stats.value[1].value = data.totalCourses
  stats.value[2].value = data.activeStudents
  stats.value[3].value = `฿${data.monthlyRevenue.toLocaleString()}`
})
</script>
```

---

## สรุป

```
Navigation Guard Execution Order:
1. Global beforeEach
2. Per-route beforeEnter
3. Component beforeRouteEnter
4. Global beforeResolve
5. [Navigation confirmed]
6. Component beforeRouteLeave (ก่อนออก)
7. Global afterEach
8. Component beforeRouteUpdate (ถ้า params เปลี่ยน)
```

| Guard | ประเภท | เมื่อไหร่ | next() available |
|-------|--------|-----------|-----------------|
| `beforeEach` | Global | ทุก navigation | ✅ |
| `beforeResolve` | Global | หลัง component guards | ✅ |
| `afterEach` | Global | หลัง navigation | ❌ |
| `beforeEnter` | Per-route | เข้า route นี้ครั้งแรก | ✅ |
| `beforeRouteEnter` | Component | ก่อน component render | ✅ |
| `beforeRouteUpdate` | Component | Params เปลี่ยน | ✅ |
| `beforeRouteLeave` | Component | ออกจาก route | ✅ |
