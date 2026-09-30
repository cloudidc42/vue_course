# Part 17: Vue Router พื้นฐาน

## บทนำ

Vue Router คือ official router สำหรับ Vue.js ใช้สร้าง Single Page Applications (SPA) ที่มีการ navigate ระหว่างหน้าต่างๆ โดยไม่ต้อง reload หน้า

---

## 1. ติดตั้ง Vue Router

```bash
# ติดตั้งกับ project ใหม่
npm create vue@latest my-app
# เลือก "Yes" สำหรับ Vue Router

# หรือติดตั้งในโปรเจกต์ที่มีอยู่
npm install vue-router@4
```

---

## 2. Route Configuration

### โครงสร้างโปรเจกต์

```
src/
├── router/
│   └── index.js          ← Route configuration
├── views/
│   ├── HomeView.vue       ← หน้าแรก
│   ├── AboutView.vue      ← หน้าเกี่ยวกับ
│   ├── BlogView.vue       ← รายการบทความ
│   ├── PostDetailView.vue ← รายละเอียดบทความ
│   └── NotFoundView.vue   ← 404
└── App.vue
```

### สร้าง Router

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'

// Lazy loading (import เมื่อต้องการเท่านั้น - ดีกว่า performance)
const HomeView = () => import('../views/HomeView.vue')
const AboutView = () => import('../views/AboutView.vue')

// Static import (load ทันที)
import BlogView from '../views/BlogView.vue'
import PostDetailView from '../views/PostDetailView.vue'
import NotFoundView from '../views/NotFoundView.vue'

const routes = [
  // Route พื้นฐาน
  {
    path: '/',
    name: 'home',
    component: HomeView,
    meta: {
      title: 'หน้าแรก',
      requiresAuth: false
    }
  },
  
  // Route พร้อม meta
  {
    path: '/about',
    name: 'about',
    component: AboutView,
    meta: {
      title: 'เกี่ยวกับเรา',
      keepAlive: true
    }
  },
  
  // Route กับ params
  {
    path: '/blog',
    name: 'blog',
    component: BlogView
  },
  
  // Route กับ params
  {
    path: '/blog/:id',
    name: 'post-detail',
    component: PostDetailView,
    props: true // ส่ง params เป็น props ให้ component
  },
  
  // Route กับ optional param
  {
    path: '/search/:query?',
    name: 'search',
    component: () => import('../views/SearchView.vue')
  },
  
  // 404 - ต้องอยู่ท้ายสุด
  {
    path: '/:pathMatch(.*)*',
    name: 'not-found',
    component: NotFoundView
  }
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  
  // Scroll behavior
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      // กลับไปที่ position เดิมเมื่อกด back
      return savedPosition
    }
    if (to.hash) {
      // Scroll ไปที่ anchor
      return { el: to.hash, behavior: 'smooth' }
    }
    // Scroll ไปบนสุดเสมอ
    return { top: 0, behavior: 'smooth' }
  }
})

export default router
```

```javascript
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

const app = createApp(App)
app.use(router)
app.mount('#app')
```

---

## 3. RouterLink และ RouterView

### RouterView - แสดง component ตาม route

```vue
<!-- App.vue -->
<template>
  <div id="app">
    <!-- Navigation -->
    <nav class="main-nav">
      <RouterLink to="/" class="logo">My App</RouterLink>
      
      <div class="nav-links">
        <RouterLink to="/">หน้าแรก</RouterLink>
        <RouterLink to="/about">เกี่ยวกับ</RouterLink>
        <RouterLink to="/blog">บทความ</RouterLink>
      </div>
    </nav>
    
    <!-- Route content จะแสดงที่นี่ -->
    <main class="main-content">
      <RouterView />
    </main>
    
    <footer class="main-footer">
      <p>© 2024 My App</p>
    </footer>
  </div>
</template>
```

### RouterLink - Link ไปยัง route

```vue
<!-- RouterLink พื้นฐาน -->
<RouterLink to="/">หน้าแรก</RouterLink>
<RouterLink to="/about">เกี่ยวกับ</RouterLink>

<!-- RouterLink ด้วย object -->
<RouterLink :to="{ name: 'home' }">หน้าแรก</RouterLink>
<RouterLink :to="{ name: 'post-detail', params: { id: 42 } }">บทความ #42</RouterLink>
<RouterLink :to="{ path: '/search', query: { q: 'vue' } }">ค้นหา Vue</RouterLink>

<!-- RouterLink พร้อม custom active class -->
<RouterLink
  to="/dashboard"
  active-class="nav-active"
  exact-active-class="nav-exact-active"
>
  Dashboard
</RouterLink>

<!-- RouterLink เป็น button (replace ไม่เพิ่ม history) -->
<RouterLink to="/home" replace>Home (replace)</RouterLink>
```

### Active Classes

Vue Router เพิ่ม CSS classes อัตโนมัติ:
- `router-link-active`: route เป็น prefix ของ current URL
- `router-link-exact-active`: route ตรงกับ current URL พอดี

```css
/* กำหนด style สำหรับ active link */
.router-link-exact-active {
  color: #4CAF50;
  font-weight: bold;
  border-bottom: 2px solid #4CAF50;
}

.router-link-active {
  color: #666;
}
```

---

## 4. Programmatic Navigation

นอกจาก RouterLink เราสามารถ navigate ด้วย JavaScript ได้

```vue
<template>
  <div>
    <button @click="goHome">ไปหน้าแรก</button>
    <button @click="goToPost(42)">ดูบทความ #42</button>
    <button @click="goBack">ย้อนกลับ</button>
    <button @click="goForward">ไปข้างหน้า</button>
  </div>
</template>

<script setup>
import { useRouter } from 'vue-router'

const router = useRouter()

// navigate ด้วย path
function goHome() {
  router.push('/')
}

// navigate ด้วย name
function goToPost(id) {
  router.push({
    name: 'post-detail',
    params: { id }
  })
}

// navigate พร้อม query
function searchPosts(keyword) {
  router.push({
    path: '/blog',
    query: { search: keyword, page: 1 }
  })
}

// replace (ไม่เพิ่ม history stack)
function redirectToLogin() {
  router.replace('/login')
}

// History navigation
function goBack() {
  router.back()
}

function goForward() {
  router.forward()
}

function goHistory(n) {
  router.go(-2) // ย้อนกลับ 2 ครั้ง
  router.go(1)  // ไปข้างหน้า 1 ครั้ง
}
</script>
```

---

## 5. Route Parameters (params, query)

### Route Params

```vue
<!-- views/PostDetailView.vue -->
<template>
  <div>
    <div v-if="loading">กำลังโหลด...</div>
    <div v-else-if="post">
      <h1>{{ post.title }}</h1>
      <p>{{ post.content }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted } from 'vue'
import { useRoute } from 'vue-router'

// Props จาก router (เมื่อ props: true ใน route config)
const props = defineProps({
  id: String // จาก /blog/:id
})

// หรืออ่านจาก useRoute
const route = useRoute()

const post = ref(null)
const loading = ref(false)

async function loadPost(id) {
  loading.value = true
  try {
    const response = await fetch(`/api/posts/${id}`)
    post.value = await response.json()
  } finally {
    loading.value = false
  }
}

// โหลดเมื่อ component mount
onMounted(() => {
  loadPost(props.id || route.params.id)
})

// Reload เมื่อ params เปลี่ยน (navigate ระหว่างบทความ)
watch(
  () => route.params.id,
  (newId) => {
    if (newId) loadPost(newId)
  }
)
</script>
```

### Route Query

```vue
<!-- views/BlogView.vue -->
<template>
  <div>
    <div class="filters">
      <input
        v-model="search"
        placeholder="ค้นหา..."
        @input="updateQuery"
      />
      <select v-model="category" @change="updateQuery">
        <option value="">ทุกหมวด</option>
        <option value="vue">Vue.js</option>
        <option value="react">React</option>
        <option value="nuxt">Nuxt.js</option>
      </select>
      <select v-model="sortBy" @change="updateQuery">
        <option value="newest">ใหม่สุด</option>
        <option value="popular">ยอดนิยม</option>
      </select>
    </div>
    
    <!-- URL จะเป็น /blog?search=vue&category=vue&sort=newest -->
    
    <div class="posts-grid">
      <PostCard v-for="post in posts" :key="post.id" :post="post" />
    </div>
    
    <!-- Pagination -->
    <div class="pagination">
      <RouterLink
        v-for="p in totalPages"
        :key="p"
        :to="{ query: { ...route.query, page: p } }"
        :class="{ active: currentPage === p }"
      >
        {{ p }}
      </RouterLink>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()

// อ่านค่าจาก query string
const search = ref(route.query.search || '')
const category = ref(route.query.category || '')
const sortBy = ref(route.query.sort || 'newest')
const currentPage = ref(parseInt(route.query.page) || 1)

const posts = ref([])
const totalPages = ref(5)

// อัปเดต URL เมื่อ filter เปลี่ยน
function updateQuery() {
  router.push({
    query: {
      ...(search.value && { search: search.value }),
      ...(category.value && { category: category.value }),
      sort: sortBy.value,
      page: 1
    }
  })
}

// Fetch เมื่อ query เปลี่ยน
watch(
  () => route.query,
  async (newQuery) => {
    const response = await fetch(
      `/api/posts?${new URLSearchParams(newQuery)}`
    )
    posts.value = await response.json()
  },
  { immediate: true }
)
</script>
```

---

## 6. Nested Routes

Nested Routes ใช้สำหรับ layout ที่ซ้อนกัน

```javascript
// router/index.js
const routes = [
  {
    path: '/dashboard',
    name: 'dashboard',
    component: () => import('../views/DashboardLayout.vue'),
    // children routes จะแสดงใน RouterView ของ DashboardLayout
    children: [
      {
        path: '',  // /dashboard (default child)
        name: 'dashboard-home',
        component: () => import('../views/dashboard/DashboardHome.vue')
      },
      {
        path: 'profile',  // /dashboard/profile
        name: 'dashboard-profile',
        component: () => import('../views/dashboard/ProfileView.vue')
      },
      {
        path: 'settings',  // /dashboard/settings
        name: 'dashboard-settings',
        component: () => import('../views/dashboard/SettingsView.vue'),
        // Nested ซ้อนกันอีกชั้น
        children: [
          {
            path: 'account',  // /dashboard/settings/account
            component: () => import('../views/dashboard/settings/AccountSettings.vue')
          },
          {
            path: 'notifications',  // /dashboard/settings/notifications
            component: () => import('../views/dashboard/settings/NotificationSettings.vue')
          }
        ]
      }
    ]
  }
]
```

### Dashboard Layout Component

```vue
<!-- views/DashboardLayout.vue -->
<template>
  <div class="dashboard">
    <!-- Dashboard Sidebar -->
    <aside class="sidebar">
      <nav>
        <RouterLink :to="{ name: 'dashboard-home' }" class="nav-item">
          📊 Overview
        </RouterLink>
        <RouterLink :to="{ name: 'dashboard-profile' }" class="nav-item">
          👤 โปรไฟล์
        </RouterLink>
        <RouterLink :to="{ name: 'dashboard-settings' }" class="nav-item">
          ⚙️ ตั้งค่า
        </RouterLink>
      </nav>
    </aside>
    
    <!-- Dashboard Content (nested route content) -->
    <main class="dashboard-content">
      <RouterView />
    </main>
  </div>
</template>
```

---

## 7. Named Routes

Named Routes ช่วยให้ navigate ได้โดยไม่ต้องจำ path

```javascript
// กำหนดชื่อ route
const routes = [
  { path: '/', name: 'home', component: HomeView },
  { path: '/users/:id', name: 'user-detail', component: UserDetailView },
  { path: '/users/:id/edit', name: 'user-edit', component: UserEditView }
]
```

```vue
<template>
  <!-- ใช้ชื่อ route -->
  <RouterLink :to="{ name: 'home' }">หน้าแรก</RouterLink>
  <RouterLink :to="{ name: 'user-detail', params: { id: userId } }">
    ดูโปรไฟล์
  </RouterLink>
</template>

<script setup>
import { useRouter } from 'vue-router'

const router = useRouter()

// Programmatic navigation ด้วยชื่อ route
router.push({ name: 'user-edit', params: { id: 42 } })
router.push({ name: 'home' })
</script>
```

---

## 8. ตัวอย่าง: Full SPA App Navigation

```javascript
// router/index.js - App สมบูรณ์
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '../stores/auth'

const routes = [
  // Public routes
  {
    path: '/',
    name: 'home',
    component: () => import('../views/HomeView.vue'),
    meta: { title: 'หน้าแรก' }
  },
  {
    path: '/courses',
    name: 'courses',
    component: () => import('../views/CoursesView.vue'),
    meta: { title: 'คอร์สเรียนทั้งหมด' }
  },
  {
    path: '/courses/:id',
    name: 'course-detail',
    component: () => import('../views/CourseDetailView.vue'),
    props: true,
    meta: { title: 'รายละเอียดคอร์ส' }
  },
  
  // Auth routes
  {
    path: '/login',
    name: 'login',
    component: () => import('../views/auth/LoginView.vue'),
    meta: { title: 'เข้าสู่ระบบ', guestOnly: true }
  },
  {
    path: '/register',
    name: 'register',
    component: () => import('../views/auth/RegisterView.vue'),
    meta: { title: 'สมัครสมาชิก', guestOnly: true }
  },
  
  // Protected routes
  {
    path: '/dashboard',
    component: () => import('../layouts/DashboardLayout.vue'),
    meta: { requiresAuth: true },
    children: [
      {
        path: '',
        name: 'dashboard',
        component: () => import('../views/dashboard/DashboardHome.vue'),
        meta: { title: 'Dashboard' }
      },
      {
        path: 'my-courses',
        name: 'my-courses',
        component: () => import('../views/dashboard/MyCourses.vue'),
        meta: { title: 'คอร์สของฉัน' }
      },
      {
        path: 'profile',
        name: 'profile',
        component: () => import('../views/dashboard/ProfileView.vue'),
        meta: { title: 'โปรไฟล์' }
      },
      {
        path: 'settings',
        name: 'settings',
        component: () => import('../views/dashboard/SettingsView.vue'),
        meta: { title: 'ตั้งค่า' }
      }
    ]
  },
  
  // Admin routes
  {
    path: '/admin',
    component: () => import('../layouts/AdminLayout.vue'),
    meta: { requiresAuth: true, requiresAdmin: true },
    children: [
      {
        path: '',
        name: 'admin-dashboard',
        component: () => import('../views/admin/AdminDashboard.vue'),
        meta: { title: 'Admin Dashboard' }
      },
      {
        path: 'users',
        name: 'admin-users',
        component: () => import('../views/admin/UsersManagement.vue'),
        meta: { title: 'จัดการผู้ใช้' }
      },
      {
        path: 'courses',
        name: 'admin-courses',
        component: () => import('../views/admin/CoursesManagement.vue'),
        meta: { title: 'จัดการคอร์ส' }
      }
    ]
  },
  
  // 404
  {
    path: '/:pathMatch(.*)*',
    name: 'not-found',
    component: () => import('../views/NotFoundView.vue'),
    meta: { title: 'ไม่พบหน้า' }
  }
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      return savedPosition
    }
    if (to.hash) {
      return { el: to.hash, behavior: 'smooth' }
    }
    return { top: 0, behavior: 'smooth' }
  }
})

// Global Guards
router.beforeEach(async (to, from, next) => {
  const authStore = useAuthStore()
  
  // อัปเดต document title
  document.title = to.meta.title ? `${to.meta.title} | Vue Course` : 'Vue Course'
  
  // ถ้า route ต้องการ auth
  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    next({ name: 'login', query: { redirect: to.fullPath } })
    return
  }
  
  // ถ้า route ต้องการ admin
  if (to.meta.requiresAdmin && !authStore.isAdmin) {
    next({ name: 'home' })
    return
  }
  
  // ถ้า route สำหรับ guest เท่านั้น (เช่น login page)
  if (to.meta.guestOnly && authStore.isAuthenticated) {
    next({ name: 'dashboard' })
    return
  }
  
  next()
})

router.afterEach((to, from) => {
  // Analytics tracking
  console.log(`Navigated to: ${to.path}`)
})

export default router
```

### Navigation Component

```vue
<!-- components/TheNavigation.vue -->
<template>
  <nav class="main-nav">
    <RouterLink to="/" class="logo">
      <img src="/logo.svg" alt="Logo" />
      Vue Course
    </RouterLink>
    
    <div class="nav-links">
      <RouterLink to="/">หน้าแรก</RouterLink>
      <RouterLink to="/courses">คอร์สเรียน</RouterLink>
      
      <!-- แสดงเฉพาะเมื่อ login แล้ว -->
      <template v-if="isAuthenticated">
        <RouterLink :to="{ name: 'dashboard' }">Dashboard</RouterLink>
        <RouterLink v-if="isAdmin" :to="{ name: 'admin-dashboard' }">Admin</RouterLink>
        
        <div class="user-menu" v-click-outside="closeUserMenu">
          <button @click="toggleUserMenu" class="user-avatar">
            <img :src="user.avatar" :alt="user.name" />
          </button>
          
          <div v-if="showUserMenu" class="dropdown">
            <RouterLink :to="{ name: 'profile' }">โปรไฟล์</RouterLink>
            <RouterLink :to="{ name: 'settings' }">ตั้งค่า</RouterLink>
            <button @click="handleLogout">ออกจากระบบ</button>
          </div>
        </div>
      </template>
      
      <!-- แสดงเมื่อยังไม่ login -->
      <template v-else>
        <RouterLink :to="{ name: 'login' }">เข้าสู่ระบบ</RouterLink>
        <RouterLink :to="{ name: 'register' }" class="btn-primary">สมัครสมาชิก</RouterLink>
      </template>
    </div>
    
    <!-- Mobile menu toggle -->
    <button class="mobile-menu-btn" @click="mobileMenuOpen = !mobileMenuOpen">
      ☰
    </button>
  </nav>
</template>

<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '../stores/auth'
import { storeToRefs } from 'pinia'

const router = useRouter()
const authStore = useAuthStore()
const { user, isAuthenticated, isAdmin } = storeToRefs(authStore)

const showUserMenu = ref(false)
const mobileMenuOpen = ref(false)

function toggleUserMenu() {
  showUserMenu.value = !showUserMenu.value
}

function closeUserMenu() {
  showUserMenu.value = false
}

async function handleLogout() {
  await authStore.logout()
  router.push({ name: 'home' })
}
</script>
```

---

## สรุป

| Feature | ตัวอย่าง |
|---------|----------|
| Route params | `/posts/:id` |
| Query string | `/posts?page=1&sort=new` |
| Named routes | `{ name: 'home' }` |
| Nested routes | `children: [...]` |
| Lazy loading | `() => import('./View.vue')` |
| Props mode | `props: true` |

```javascript
// สรุปคำสั่งสำคัญ
const router = useRouter()  // สำหรับ navigate
const route = useRoute()    // สำหรับ read current route

router.push('/path')
router.push({ name: 'route-name', params: { id: 1 } })
router.replace('/path')
router.back()

route.params.id        // URL params
route.query.search     // Query string
route.meta.title       // Route meta
route.path             // Current path
route.name             // Current route name
```
