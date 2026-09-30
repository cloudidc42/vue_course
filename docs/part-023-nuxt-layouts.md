# Part 23: Nuxt Layouts

## Default Layout

Layout ใน Nuxt คือ wrapper component ที่ wrap รอบ page content ช่วยให้ share header, footer, sidebar ระหว่าง pages ได้โดยไม่ต้องเขียนซ้ำ

### สร้าง Default Layout

```vue
<!-- layouts/default.vue -->
<template>
  <div class="app-container">
    <!-- Header -->
    <AppHeader />
    
    <!-- Main content - page จะ render ที่ slot นี้ -->
    <main class="main-content">
      <slot />
    </main>
    
    <!-- Footer -->
    <AppFooter />
  </div>
</template>

<style scoped>
.app-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.main-content {
  flex: 1;
  padding: 2rem;
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
}
</style>
```

### กำหนดให้ใช้ Layout ใน app.vue

```vue
<!-- app.vue -->
<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

### Header Component

```vue
<!-- components/AppHeader.vue -->
<script setup lang="ts">
const { isLoggedIn, user, logout } = useAuth()

const navLinks = [
  { text: 'หน้าแรก', to: '/' },
  { text: 'บทความ', to: '/blog' },
  { text: 'เกี่ยวกับ', to: '/about' },
  { text: 'ติดต่อ', to: '/contact' }
]
</script>

<template>
  <header class="app-header">
    <div class="header-inner">
      <NuxtLink to="/" class="logo">
        <img src="/logo.svg" alt="Logo" />
        <span>My Site</span>
      </NuxtLink>
      
      <nav class="main-nav">
        <NuxtLink
          v-for="link in navLinks"
          :key="link.to"
          :to="link.to"
          class="nav-link"
          active-class="active"
        >
          {{ link.text }}
        </NuxtLink>
      </nav>
      
      <div class="header-actions">
        <template v-if="isLoggedIn">
          <span>{{ user?.name }}</span>
          <button @click="logout">ออกจากระบบ</button>
        </template>
        <template v-else>
          <NuxtLink to="/login">เข้าสู่ระบบ</NuxtLink>
          <NuxtLink to="/register" class="btn-primary">สมัครสมาชิก</NuxtLink>
        </template>
      </div>
    </div>
  </header>
</template>

<style scoped>
.app-header {
  background: white;
  border-bottom: 1px solid #e5e7eb;
  position: sticky;
  top: 0;
  z-index: 100;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.header-inner {
  display: flex;
  align-items: center;
  gap: 2rem;
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 1rem;
  height: 64px;
}

.logo {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  text-decoration: none;
  font-weight: bold;
  font-size: 1.25rem;
  color: #111;
}

.main-nav {
  display: flex;
  gap: 1.5rem;
  flex: 1;
}

.nav-link {
  text-decoration: none;
  color: #555;
  font-weight: 500;
  transition: color 0.2s;
}

.nav-link:hover,
.nav-link.active {
  color: #00dc82;
}
</style>
```

---

## Named Layouts

นอกจาก `default` layout แล้ว สามารถสร้าง layout อื่นได้

### สร้าง Named Layouts

```vue
<!-- layouts/blog.vue -->
<template>
  <div class="blog-layout">
    <BlogHeader />
    
    <div class="blog-container">
      <!-- Sidebar -->
      <aside class="blog-sidebar">
        <BlogSearch />
        <BlogCategories />
        <BlogRecentPosts />
        <BlogTags />
      </aside>
      
      <!-- Main content -->
      <main class="blog-main">
        <slot />
      </main>
    </div>
    
    <AppFooter />
  </div>
</template>

<style scoped>
.blog-container {
  display: grid;
  grid-template-columns: 1fr 300px;
  gap: 2rem;
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

@media (max-width: 768px) {
  .blog-container {
    grid-template-columns: 1fr;
  }
  
  .blog-sidebar {
    order: 2;
  }
}
</style>
```

```vue
<!-- layouts/admin.vue -->
<script setup lang="ts">
const { user } = useAuth()

const sidebarLinks = [
  { icon: '📊', text: 'Dashboard', to: '/admin' },
  { icon: '📝', text: 'Posts', to: '/admin/posts' },
  { icon: '👥', text: 'Users', to: '/admin/users' },
  { icon: '📁', text: 'Categories', to: '/admin/categories' },
  { icon: '💬', text: 'Comments', to: '/admin/comments' },
  { icon: '⚙️', text: 'Settings', to: '/admin/settings' }
]

const isSidebarOpen = ref(true)
</script>

<template>
  <div class="admin-layout" :class="{ 'sidebar-collapsed': !isSidebarOpen }">
    <!-- Admin Sidebar -->
    <aside class="admin-sidebar">
      <div class="sidebar-header">
        <NuxtLink to="/admin" class="admin-logo">
          <span>Admin</span>
        </NuxtLink>
        <button @click="isSidebarOpen = !isSidebarOpen" class="toggle-btn">
          {{ isSidebarOpen ? '←' : '→' }}
        </button>
      </div>
      
      <nav class="sidebar-nav">
        <NuxtLink
          v-for="link in sidebarLinks"
          :key="link.to"
          :to="link.to"
          class="sidebar-link"
          active-class="active"
        >
          <span class="link-icon">{{ link.icon }}</span>
          <span class="link-text">{{ link.text }}</span>
        </NuxtLink>
      </nav>
      
      <div class="sidebar-footer">
        <div class="user-info">
          <img :src="user?.avatar" :alt="user?.name" class="user-avatar" />
          <div class="user-details">
            <span class="user-name">{{ user?.name }}</span>
            <span class="user-role">{{ user?.role }}</span>
          </div>
        </div>
      </div>
    </aside>
    
    <!-- Admin Content -->
    <div class="admin-content">
      <!-- Top Bar -->
      <header class="admin-topbar">
        <div class="topbar-left">
          <h1 class="page-title">
            <slot name="title">Admin Panel</slot>
          </h1>
        </div>
        <div class="topbar-right">
          <AdminNotifications />
          <AdminUserMenu />
        </div>
      </header>
      
      <!-- Page Content -->
      <main class="admin-main">
        <slot />
      </main>
    </div>
  </div>
</template>

<style scoped>
.admin-layout {
  display: grid;
  grid-template-columns: 250px 1fr;
  min-height: 100vh;
  transition: grid-template-columns 0.3s ease;
}

.admin-layout.sidebar-collapsed {
  grid-template-columns: 64px 1fr;
}

.admin-sidebar {
  background: #1e293b;
  color: white;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.sidebar-header {
  padding: 1rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid rgba(255,255,255,0.1);
}

.admin-logo {
  font-size: 1.25rem;
  font-weight: bold;
  color: #00dc82;
  text-decoration: none;
}

.sidebar-nav {
  flex: 1;
  padding: 1rem 0;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem 1rem;
  color: #94a3b8;
  text-decoration: none;
  transition: all 0.2s;
}

.sidebar-link:hover,
.sidebar-link.active {
  background: rgba(255,255,255,0.05);
  color: white;
}

.sidebar-link.active {
  border-left: 3px solid #00dc82;
  color: #00dc82;
}

.admin-content {
  display: flex;
  flex-direction: column;
  background: #f1f5f9;
}

.admin-topbar {
  background: white;
  border-bottom: 1px solid #e2e8f0;
  padding: 0 1.5rem;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.admin-main {
  flex: 1;
  padding: 1.5rem;
  overflow-y: auto;
}
</style>
```

---

## เปลี่ยน Layout ใน Page

### วิธีที่ 1: definePageMeta

```vue
<!-- pages/blog/index.vue -->
<script setup lang="ts">
// กำหนด layout ใน definePageMeta
definePageMeta({
  layout: 'blog'  // ใช้ layouts/blog.vue
})
</script>

<template>
  <div>
    <h1>บทความทั้งหมด</h1>
    <!-- content -->
  </div>
</template>
```

```vue
<!-- pages/admin/index.vue -->
<script setup lang="ts">
definePageMeta({
  layout: 'admin',
  middleware: ['auth', 'admin-only']
})
</script>

<template>
  <div>
    <h1>Admin Dashboard</h1>
  </div>
</template>
```

### ปิด Layout

```vue
<!-- pages/login.vue - ไม่ต้องการ layout -->
<script setup lang="ts">
definePageMeta({
  layout: false  // ไม่ใช้ layout เลย
})
</script>

<template>
  <div class="login-page">
    <LoginForm />
  </div>
</template>
```

---

## Dynamic Layouts

เปลี่ยน layout แบบ dynamic ตามสถานการณ์

### ใช้ setPageLayout

```vue
<!-- pages/profile.vue -->
<script setup lang="ts">
const { isAdmin } = useAuth()

// เปลี่ยน layout แบบ dynamic
if (isAdmin.value) {
  setPageLayout('admin')
} else {
  setPageLayout('default')
}
</script>
```

### ส่ง Layout จาก Parent

```vue
<!-- app.vue -->
<script setup lang="ts">
const { user } = useAuth()

// กำหนด layout ตาม user role
const currentLayout = computed(() => {
  if (!user.value) return 'default'
  if (user.value.role === 'admin') return 'admin'
  if (user.value.isPremium) return 'premium'
  return 'default'
})
</script>

<template>
  <NuxtLayout :name="currentLayout">
    <NuxtPage />
  </NuxtLayout>
</template>
```

---

## Layout กับ Transitions

### กำหนด Layout Transition

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    layoutTransition: {
      name: 'layout',
      mode: 'out-in'
    },
    pageTransition: {
      name: 'page',
      mode: 'out-in'
    }
  }
})
```

### CSS สำหรับ Transitions

```css
/* assets/css/transitions.css */

/* Page transitions */
.page-enter-active,
.page-leave-active {
  transition: all 0.3s ease;
}

.page-enter-from {
  opacity: 0;
  transform: translateY(20px);
}

.page-leave-to {
  opacity: 0;
  transform: translateY(-20px);
}

/* Layout transitions */
.layout-enter-active,
.layout-leave-active {
  transition: all 0.4s ease;
}

.layout-enter-from {
  opacity: 0;
  transform: translateX(-20px);
}

.layout-leave-to {
  opacity: 0;
  transform: translateX(20px);
}

/* Slide right transition */
.slide-right-enter-active,
.slide-right-leave-active {
  transition: all 0.3s ease;
}

.slide-right-enter-from {
  opacity: 0;
  transform: translateX(50px);
}

.slide-right-leave-to {
  opacity: 0;
  transform: translateX(-50px);
}
```

### Custom Transition per Page

```vue
<!-- pages/special.vue -->
<script setup lang="ts">
definePageMeta({
  pageTransition: {
    name: 'slide-right',
    mode: 'out-in'
  }
})
</script>
```

---

## Nested Layouts

Layout ที่ซ้อนกัน เช่น Admin มี Layout ย่อยสำหรับส่วนต่างๆ

```
layouts/
├── default.vue       # Main layout
├── admin.vue         # Admin layout (extends default)
└── admin-posts.vue   # Posts section in admin
```

### ตัวอย่าง Nested Layout Structure

```vue
<!-- layouts/admin.vue -->
<template>
  <!-- ใช้ default layout เป็น wrapper -->
  <div class="admin-wrapper">
    <AdminSidebar />
    <div class="admin-content">
      <AdminTopbar />
      <main>
        <slot />
      </main>
    </div>
  </div>
</template>
```

```vue
<!-- pages/admin/posts/index.vue -->
<script setup lang="ts">
definePageMeta({
  layout: 'admin'
})
</script>

<template>
  <div class="posts-management">
    <!-- Breadcrumb -->
    <nav class="breadcrumb">
      <NuxtLink to="/admin">Admin</NuxtLink>
      <span>/</span>
      <span>Posts</span>
    </nav>
    
    <!-- Sub-navigation สำหรับ posts section -->
    <div class="section-nav">
      <NuxtLink to="/admin/posts">ทั้งหมด</NuxtLink>
      <NuxtLink to="/admin/posts/published">เผยแพร่แล้ว</NuxtLink>
      <NuxtLink to="/admin/posts/drafts">แบบร่าง</NuxtLink>
      <NuxtLink to="/admin/posts/create">สร้างใหม่</NuxtLink>
    </div>
    
    <slot />
  </div>
</template>
```

---

## ตัวอย่าง: Admin Dashboard Layout สมบูรณ์

```vue
<!-- layouts/admin-full.vue -->
<script setup lang="ts">
const { user, logout } = useAuth()
const route = useRoute()

// Sidebar state
const isMobileSidebarOpen = ref(false)
const isCollapsed = ref(false)

// ปิด mobile sidebar เมื่อ navigate
watch(() => route.path, () => {
  isMobileSidebarOpen.value = false
})

// Breadcrumbs จาก route
const breadcrumbs = computed(() => {
  const paths = route.path.split('/').filter(Boolean)
  return paths.map((path, index) => ({
    text: path.charAt(0).toUpperCase() + path.slice(1),
    to: '/' + paths.slice(0, index + 1).join('/')
  }))
})

const menuGroups = [
  {
    label: 'หลัก',
    items: [
      { icon: '📊', text: 'Dashboard', to: '/admin' },
      { icon: '📈', text: 'Analytics', to: '/admin/analytics' }
    ]
  },
  {
    label: 'เนื้อหา',
    items: [
      { icon: '📝', text: 'Posts', to: '/admin/posts', badge: 12 },
      { icon: '📄', text: 'Pages', to: '/admin/pages' },
      { icon: '📁', text: 'Categories', to: '/admin/categories' },
      { icon: '🏷️', text: 'Tags', to: '/admin/tags' },
      { icon: '🖼️', text: 'Media', to: '/admin/media' }
    ]
  },
  {
    label: 'ผู้ใช้',
    items: [
      { icon: '👥', text: 'Users', to: '/admin/users' },
      { icon: '👤', text: 'Roles', to: '/admin/roles' }
    ]
  },
  {
    label: 'ระบบ',
    items: [
      { icon: '⚙️', text: 'Settings', to: '/admin/settings' },
      { icon: '🔧', text: 'Plugins', to: '/admin/plugins' }
    ]
  }
]
</script>

<template>
  <div class="admin-full" :class="{ collapsed: isCollapsed }">
    <!-- Mobile Overlay -->
    <div
      v-if="isMobileSidebarOpen"
      class="mobile-overlay"
      @click="isMobileSidebarOpen = false"
    />
    
    <!-- Sidebar -->
    <aside
      class="admin-sidebar-full"
      :class="{ open: isMobileSidebarOpen }"
    >
      <!-- Logo -->
      <div class="sidebar-logo">
        <NuxtLink to="/admin">
          <span class="logo-text">🚀 Admin</span>
        </NuxtLink>
        <button
          class="collapse-btn"
          @click="isCollapsed = !isCollapsed"
          title="Toggle sidebar"
        >
          ◀
        </button>
      </div>
      
      <!-- Navigation -->
      <nav class="sidebar-menu">
        <div v-for="group in menuGroups" :key="group.label" class="menu-group">
          <span class="group-label">{{ group.label }}</span>
          <NuxtLink
            v-for="item in group.items"
            :key="item.to"
            :to="item.to"
            class="menu-item"
            active-class="active"
            exact-active-class="exact-active"
          >
            <span class="item-icon">{{ item.icon }}</span>
            <span class="item-text">{{ item.text }}</span>
            <span v-if="item.badge" class="item-badge">{{ item.badge }}</span>
          </NuxtLink>
        </div>
      </nav>
      
      <!-- User Profile -->
      <div class="sidebar-user">
        <img :src="user?.avatar || '/default-avatar.png'" class="user-avatar" />
        <div class="user-info">
          <span class="user-name">{{ user?.name }}</span>
          <span class="user-role">{{ user?.role }}</span>
        </div>
        <button @click="logout" title="Logout" class="logout-btn">🚪</button>
      </div>
    </aside>
    
    <!-- Main -->
    <div class="admin-main-area">
      <!-- Topbar -->
      <header class="admin-header">
        <button
          class="mobile-menu-btn"
          @click="isMobileSidebarOpen = true"
        >☰</button>
        
        <!-- Breadcrumbs -->
        <nav class="header-breadcrumbs">
          <NuxtLink to="/admin">🏠</NuxtLink>
          <template v-for="(crumb, i) in breadcrumbs" :key="i">
            <span class="separator">/</span>
            <NuxtLink v-if="i < breadcrumbs.length - 1" :to="crumb.to">
              {{ crumb.text }}
            </NuxtLink>
            <span v-else class="current">{{ crumb.text }}</span>
          </template>
        </nav>
        
        <!-- Right Actions -->
        <div class="header-actions">
          <button class="action-btn" title="Notifications">🔔</button>
          <button class="action-btn" title="Help">❓</button>
          <NuxtLink to="/" target="_blank" class="action-btn" title="View Site">
            🌐
          </NuxtLink>
        </div>
      </header>
      
      <!-- Page Content -->
      <main class="page-content">
        <slot />
      </main>
    </div>
  </div>
</template>

<style scoped>
.admin-full {
  display: grid;
  grid-template-columns: 260px 1fr;
  min-height: 100vh;
  background: #f8fafc;
}

.admin-full.collapsed {
  grid-template-columns: 72px 1fr;
}

.admin-sidebar-full {
  background: #0f172a;
  color: #e2e8f0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: sticky;
  top: 0;
  height: 100vh;
}

.sidebar-logo {
  padding: 1rem 1.25rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid rgba(255,255,255,0.06);
}

.logo-text {
  font-size: 1.1rem;
  font-weight: 700;
  color: #00dc82;
}

.collapse-btn {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  font-size: 0.75rem;
}

.sidebar-menu {
  flex: 1;
  overflow-y: auto;
  padding: 1rem 0;
}

.menu-group {
  margin-bottom: 1.5rem;
}

.group-label {
  font-size: 0.65rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: #475569;
  padding: 0 1.25rem;
  display: block;
  margin-bottom: 0.25rem;
}

.menu-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.625rem 1.25rem;
  text-decoration: none;
  color: #94a3b8;
  transition: all 0.15s;
  border-left: 3px solid transparent;
}

.menu-item:hover {
  background: rgba(255,255,255,0.04);
  color: #e2e8f0;
}

.menu-item.exact-active {
  background: rgba(0, 220, 130, 0.08);
  color: #00dc82;
  border-left-color: #00dc82;
}

.item-badge {
  margin-left: auto;
  background: #ef4444;
  color: white;
  font-size: 0.65rem;
  padding: 0.1rem 0.4rem;
  border-radius: 999px;
}

.sidebar-user {
  padding: 1rem 1.25rem;
  display: flex;
  align-items: center;
  gap: 0.75rem;
  border-top: 1px solid rgba(255,255,255,0.06);
}

.user-avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  object-fit: cover;
}

.user-name {
  display: block;
  font-size: 0.875rem;
  font-weight: 600;
  color: #e2e8f0;
}

.user-role {
  display: block;
  font-size: 0.75rem;
  color: #64748b;
}

.logout-btn {
  margin-left: auto;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1rem;
  opacity: 0.5;
  transition: opacity 0.2s;
}

.logout-btn:hover { opacity: 1; }

.admin-main-area {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.admin-header {
  background: white;
  border-bottom: 1px solid #e2e8f0;
  padding: 0 1.5rem;
  height: 60px;
  display: flex;
  align-items: center;
  gap: 1rem;
  position: sticky;
  top: 0;
  z-index: 50;
}

.header-breadcrumbs {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.875rem;
  color: #64748b;
}

.header-breadcrumbs a {
  color: #3b82f6;
  text-decoration: none;
}

.header-breadcrumbs .current {
  color: #1e293b;
  font-weight: 500;
}

.header-actions {
  display: flex;
  gap: 0.5rem;
}

.action-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1.1rem;
  padding: 0.4rem;
  border-radius: 6px;
  text-decoration: none;
  transition: background 0.2s;
}

.action-btn:hover {
  background: #f1f5f9;
}

.page-content {
  flex: 1;
  padding: 1.5rem;
}

.mobile-menu-btn {
  display: none;
  background: none;
  border: none;
  font-size: 1.25rem;
  cursor: pointer;
}

@media (max-width: 768px) {
  .admin-full {
    grid-template-columns: 1fr;
  }
  
  .admin-sidebar-full {
    position: fixed;
    left: -260px;
    top: 0;
    height: 100vh;
    z-index: 200;
    transition: left 0.3s ease;
    width: 260px;
  }
  
  .admin-sidebar-full.open {
    left: 0;
  }
  
  .mobile-menu-btn {
    display: block;
  }
  
  .mobile-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0,0,0,0.5);
    z-index: 150;
  }
}
</style>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Default Layout** - Layout พื้นฐานที่ทุก page ใช้
2. **Named Layouts** - สร้าง layout สำหรับส่วนต่างๆ เช่น admin, blog
3. **เปลี่ยน Layout** - ใช้ `definePageMeta({ layout: 'name' })`
4. **Dynamic Layouts** - เปลี่ยน layout แบบ programmatic
5. **Layout Transitions** - เพิ่ม animation เมื่อเปลี่ยน layout
6. **Nested Layouts** - ซ้อน layout เพื่อโครงสร้างที่ซับซ้อน

**ถัดไป**: Part 24 - Nuxt Data Fetching
