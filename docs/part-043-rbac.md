# Part 43: Role-Based Access Control (RBAC) กับ Nuxt.js

## RBAC คืออะไร?

Role-Based Access Control (RBAC) เป็นระบบควบคุมสิทธิ์การเข้าถึงที่กำหนดตาม "บทบาท" (Role) ของผู้ใช้ แทนที่จะกำหนดสิทธิ์ให้ผู้ใช้แต่ละคน เราจะกำหนดสิทธิ์ให้ Role และกำหนด Role ให้ผู้ใช้

## 1. RBAC Concept

```
ผู้ใช้ (User) --> มี --> บทบาท (Role) --> มี --> สิทธิ์ (Permission)

ตัวอย่าง:
- admin  --> [create_post, read_post, update_post, delete_post, manage_users]
- editor --> [create_post, read_post, update_post, delete_post]  
- author --> [create_post, read_post, update_own_post, delete_own_post]
- viewer --> [read_post]
```

## 2. Permissions System Design

```typescript
// types/rbac.ts
export type Permission =
  // Posts
  | 'posts:read'
  | 'posts:create'
  | 'posts:update'
  | 'posts:update_own'
  | 'posts:delete'
  | 'posts:delete_own'
  | 'posts:publish'
  // Users
  | 'users:read'
  | 'users:create'
  | 'users:update'
  | 'users:delete'
  | 'users:manage_roles'
  // Settings
  | 'settings:read'
  | 'settings:update'
  // Comments
  | 'comments:read'
  | 'comments:create'
  | 'comments:moderate'
  | 'comments:delete'

export type Role = 'admin' | 'editor' | 'author' | 'viewer'

// กำหนด permissions สำหรับแต่ละ role
export const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: [
    'posts:read', 'posts:create', 'posts:update', 'posts:delete', 'posts:publish',
    'users:read', 'users:create', 'users:update', 'users:delete', 'users:manage_roles',
    'settings:read', 'settings:update',
    'comments:read', 'comments:create', 'comments:moderate', 'comments:delete'
  ],
  editor: [
    'posts:read', 'posts:create', 'posts:update', 'posts:delete', 'posts:publish',
    'comments:read', 'comments:create', 'comments:moderate', 'comments:delete'
  ],
  author: [
    'posts:read', 'posts:create', 'posts:update_own', 'posts:delete_own',
    'comments:read', 'comments:create'
  ],
  viewer: [
    'posts:read',
    'comments:read', 'comments:create'
  ]
}

// Helper function
export const hasPermission = (
  userRole: Role,
  permission: Permission
): boolean => {
  return ROLE_PERMISSIONS[userRole]?.includes(permission) ?? false
}
```

## 3. Role Hierarchy

```typescript
// utils/roleHierarchy.ts
export const ROLE_HIERARCHY: Record<Role, number> = {
  admin: 100,
  editor: 50,
  author: 25,
  viewer: 10
}

// ตรวจสอบว่า role มีลำดับสูงกว่าหรือเท่ากับ minimum role
export const meetsMinimumRole = (userRole: Role, minimumRole: Role): boolean => {
  return ROLE_HIERARCHY[userRole] >= ROLE_HIERARCHY[minimumRole]
}

// ดึง roles ทั้งหมดที่มีลำดับต่ำกว่าหรือเท่ากับ role นั้น
export const getSubordinateRoles = (role: Role): Role[] => {
  const roleLevel = ROLE_HIERARCHY[role]
  return (Object.entries(ROLE_HIERARCHY) as [Role, number][])
    .filter(([, level]) => level < roleLevel)
    .map(([r]) => r)
}
```

## 4. RBAC Composable

```typescript
// composables/useRBAC.ts
import type { Role, Permission } from '~/types/rbac'
import { ROLE_PERMISSIONS, hasPermission, meetsMinimumRole } from '~/utils/rbac'

export const useRBAC = () => {
  const { user } = useUserSession()
  
  const userRole = computed<Role | null>(() => user.value?.role as Role || null)
  
  // ตรวจสอบ permission เดียว
  const can = (permission: Permission): boolean => {
    if (!userRole.value) return false
    return hasPermission(userRole.value, permission)
  }
  
  // ตรวจสอบหลาย permissions (AND - ต้องมีทุกอัน)
  const canAll = (...permissions: Permission[]): boolean => {
    return permissions.every(p => can(p))
  }
  
  // ตรวจสอบหลาย permissions (OR - มีอย่างน้อยหนึ่ง)
  const canAny = (...permissions: Permission[]): boolean => {
    return permissions.some(p => can(p))
  }
  
  // ตรวจสอบ role
  const is = (role: Role | Role[]): boolean => {
    if (!userRole.value) return false
    const roles = Array.isArray(role) ? role : [role]
    return roles.includes(userRole.value)
  }
  
  // ตรวจสอบว่ามี role ขั้นต่ำ
  const isAtLeast = (minimumRole: Role): boolean => {
    if (!userRole.value) return false
    return meetsMinimumRole(userRole.value, minimumRole)
  }
  
  // ตรวจสอบ ownership (เป็นเจ้าของ resource)
  const isOwner = (resourceUserId: string | number): boolean => {
    return user.value?.id === resourceUserId
  }
  
  // ตรวจสอบสิทธิ์เจ้าของหรือ permission พิเศษ
  const canOrOwn = (permission: Permission, resourceUserId: string | number): boolean => {
    return can(permission) || (isOwner(resourceUserId) && can(permission.replace(/^(\w+):(\w+)$/, '$1:$2_own') as Permission))
  }
  
  return {
    userRole,
    can,
    canAll,
    canAny,
    is,
    isAtLeast,
    isOwner,
    canOrOwn
  }
}
```

## 5. Component-level Permission

```vue
<!-- components/rbac/PermissionGuard.vue -->
<template>
  <slot v-if="hasAccess" />
  <slot v-else name="fallback" />
</template>

<script setup lang="ts">
import type { Permission, Role } from '~/types/rbac'

interface Props {
  permission?: Permission | Permission[]
  role?: Role | Role[]
  minimumRole?: Role
  ownerId?: string | number
  fallback?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  fallback: false
})

const { can, canAny, is, isAtLeast, isOwner } = useRBAC()

const hasAccess = computed(() => {
  // ตรวจสอบ permission
  if (props.permission) {
    const permissions = Array.isArray(props.permission)
      ? props.permission
      : [props.permission]
    
    if (!canAny(...permissions)) return false
  }
  
  // ตรวจสอบ role
  if (props.role) {
    if (!is(props.role)) return false
  }
  
  // ตรวจสอบ minimum role
  if (props.minimumRole) {
    if (!isAtLeast(props.minimumRole)) return false
  }
  
  // ตรวจสอบ ownership
  if (props.ownerId !== undefined) {
    if (!isOwner(props.ownerId)) return false
  }
  
  return true
})
</script>
```

```vue
<!-- การใช้งาน PermissionGuard -->
<template>
  <!-- แสดงเฉพาะคนที่มีสิทธิ์ delete_post -->
  <PermissionGuard permission="posts:delete">
    <button @click="deletePost">ลบโพสต์</button>
    
    <template #fallback>
      <span class="no-permission">ไม่มีสิทธิ์ลบ</span>
    </template>
  </PermissionGuard>
  
  <!-- แสดงเฉพาะ admin หรือ editor -->
  <PermissionGuard :role="['admin', 'editor']">
    <AdminPanel />
  </PermissionGuard>
  
  <!-- แสดงเฉพาะ editor ขึ้นไป -->
  <PermissionGuard minimumRole="editor">
    <PublishButton />
  </PermissionGuard>
</template>
```

## 6. API-level Permission

```typescript
// server/utils/rbac.ts
import { type Role, ROLE_PERMISSIONS, type Permission } from '~/types/rbac'

export const requireAuth = async (event: H3Event) => {
  const session = await getUserSession(event)
  if (!session?.user) {
    throw createError({ statusCode: 401, message: 'ยังไม่ได้เข้าสู่ระบบ' })
  }
  return session.user
}

export const requirePermission = async (event: H3Event, permission: Permission) => {
  const user = await requireAuth(event)
  const userRole = user.role as Role
  
  if (!ROLE_PERMISSIONS[userRole]?.includes(permission)) {
    throw createError({
      statusCode: 403,
      message: `ไม่มีสิทธิ์: ต้องการ ${permission}`
    })
  }
  
  return user
}

export const requireRole = async (event: H3Event, requiredRole: Role | Role[]) => {
  const user = await requireAuth(event)
  const roles = Array.isArray(requiredRole) ? requiredRole : [requiredRole]
  
  if (!roles.includes(user.role as Role)) {
    throw createError({
      statusCode: 403,
      message: 'ไม่มีสิทธิ์เข้าถึง'
    })
  }
  
  return user
}

export const requireOwnerOrPermission = async (
  event: H3Event,
  resourceOwnerId: string | number,
  permission: Permission
) => {
  const user = await requireAuth(event)
  const userRole = user.role as Role
  
  const isOwner = String(user.id) === String(resourceOwnerId)
  const hasPermission = ROLE_PERMISSIONS[userRole]?.includes(permission)
  
  if (!isOwner && !hasPermission) {
    throw createError({
      statusCode: 403,
      message: 'ต้องเป็นเจ้าของหรือมีสิทธิ์พิเศษ'
    })
  }
  
  return { user, isOwner }
}
```

```typescript
// server/api/posts/[id].delete.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  
  // ดึงโพสต์
  const post = await prisma.post.findUnique({ where: { id } })
  if (!post) throw createError({ statusCode: 404, message: 'ไม่พบโพสต์' })
  
  // ตรวจสอบสิทธิ์: เจ้าของสามารถลบได้ หรือคนที่มีสิทธิ์ posts:delete
  await requireOwnerOrPermission(event, post.authorId, 'posts:delete')
  
  await prisma.post.delete({ where: { id } })
  return { success: true }
})
```

## 7. Nuxt Middleware สำหรับ RBAC

```typescript
// middleware/rbac.ts
import type { Role, Permission } from '~/types/rbac'
import { ROLE_PERMISSIONS, meetsMinimumRole } from '~/utils/rbac'

// สร้าง middleware factory
export const createRBACMiddleware = (options: {
  permission?: Permission
  role?: Role | Role[]
  minimumRole?: Role
}) => {
  return defineNuxtRouteMiddleware(() => {
    const { user } = useUserSession()
    
    if (!user.value) {
      return navigateTo('/login?redirect=' + useRoute().fullPath)
    }
    
    const userRole = user.value.role as Role
    
    // ตรวจสอบ permission
    if (options.permission) {
      if (!ROLE_PERMISSIONS[userRole]?.includes(options.permission)) {
        throw createError({
          statusCode: 403,
          statusMessage: 'ไม่มีสิทธิ์เข้าถึงหน้านี้'
        })
      }
    }
    
    // ตรวจสอบ role
    if (options.role) {
      const roles = Array.isArray(options.role) ? options.role : [options.role]
      if (!roles.includes(userRole)) {
        throw createError({ statusCode: 403, statusMessage: 'บทบาทไม่เพียงพอ' })
      }
    }
    
    // ตรวจสอบ minimum role
    if (options.minimumRole) {
      if (!meetsMinimumRole(userRole, options.minimumRole)) {
        throw createError({ statusCode: 403, statusMessage: 'บทบาทไม่เพียงพอ' })
      }
    }
  })
}
```

```typescript
// pages/admin/index.vue
definePageMeta({
  middleware: [
    'auth',
    // inline middleware
    function() {
      const { userRole } = useRBAC()
      if (userRole.value !== 'admin') {
        return navigateTo('/403')
      }
    }
  ]
})
```

## 8. ตัวอย่าง: Admin Panel with Permissions

```vue
<!-- pages/admin/users.vue -->
<template>
  <div class="admin-users">
    <div class="page-header">
      <h1>จัดการผู้ใช้</h1>
      
      <PermissionGuard permission="users:create">
        <button @click="openCreateModal" class="btn-primary">
          + เพิ่มผู้ใช้ใหม่
        </button>
      </PermissionGuard>
    </div>
    
    <!-- Filters -->
    <div class="filters">
      <select v-model="roleFilter" class="filter-select">
        <option value="">ทุก Role</option>
        <option value="admin">Admin</option>
        <option value="editor">Editor</option>
        <option value="author">Author</option>
        <option value="viewer">Viewer</option>
      </select>
      
      <input
        v-model="searchQuery"
        type="text"
        placeholder="ค้นหาผู้ใช้..."
        class="search-input"
      />
    </div>
    
    <!-- Users Table -->
    <div class="users-table-wrapper">
      <table class="users-table">
        <thead>
          <tr>
            <th>ผู้ใช้</th>
            <th>อีเมล</th>
            <th>Role</th>
            <th>สมัครเมื่อ</th>
            <th>การดำเนินการ</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="user in filteredUsers" :key="user.id">
            <td>
              <div class="user-cell">
                <img :src="user.avatar || '/default-avatar.png'" :alt="user.name" class="user-avatar" />
                <div>
                  <strong>{{ user.name }}</strong>
                  <small v-if="user.id === currentUser?.id" class="you-badge">คุณ</small>
                </div>
              </div>
            </td>
            <td>{{ user.email }}</td>
            <td>
              <!-- Role Selector (เฉพาะ admin เปลี่ยนได้) -->
              <select
                v-if="can('users:manage_roles') && user.id !== currentUser?.id"
                :value="user.role"
                @change="updateRole(user, $event.target.value)"
                class="role-select"
                :class="'role-' + user.role"
              >
                <option value="admin">Admin</option>
                <option value="editor">Editor</option>
                <option value="author">Author</option>
                <option value="viewer">Viewer</option>
              </select>
              <span v-else class="role-badge" :class="'role-' + user.role">
                {{ user.role }}
              </span>
            </td>
            <td>{{ formatDate(user.createdAt) }}</td>
            <td>
              <div class="actions">
                <PermissionGuard permission="users:read">
                  <button @click="viewUser(user)" class="btn-icon" title="ดูข้อมูล">
                    👁️
                  </button>
                </PermissionGuard>
                
                <PermissionGuard permission="users:update">
                  <button @click="editUser(user)" class="btn-icon" title="แก้ไข">
                    ✏️
                  </button>
                </PermissionGuard>
                
                <PermissionGuard permission="users:delete">
                  <button
                    v-if="user.id !== currentUser?.id"
                    @click="confirmDelete(user)"
                    class="btn-icon danger"
                    title="ลบ"
                  >
                    🗑️
                  </button>
                </PermissionGuard>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    
    <!-- Empty State -->
    <div v-if="filteredUsers.length === 0" class="empty-state">
      <p>ไม่พบผู้ใช้ที่ตรงกับเงื่อนไข</p>
    </div>
    
    <!-- Create/Edit Modal -->
    <UserModal
      v-if="showModal"
      :user="selectedUser"
      @close="closeModal"
      @saved="handleUserSaved"
    />
    
    <!-- Delete Confirmation -->
    <ConfirmDialog
      v-if="userToDelete"
      :title="`ลบผู้ใช้ ${userToDelete.name}?`"
      message="การกระทำนี้ไม่สามารถยกเลิกได้"
      @confirm="deleteUser"
      @cancel="userToDelete = null"
    />
  </div>
</template>

<script setup>
definePageMeta({
  middleware: ['auth'],
  layout: 'admin'
})

const { can, is } = useRBAC()
const { user: currentUser } = useUserSession()

// State
const users = ref([])
const roleFilter = ref('')
const searchQuery = ref('')
const showModal = ref(false)
const selectedUser = ref(null)
const userToDelete = ref(null)
const isLoading = ref(true)

// ตรวจสอบสิทธิ์
if (!can('users:read')) {
  throw createError({ statusCode: 403, statusMessage: 'ไม่มีสิทธิ์เข้าถึง' })
}

// Load users
const { data: usersData, refresh } = await useFetch('/api/admin/users')
users.value = usersData.value?.users || []

// Filtered users
const filteredUsers = computed(() => {
  let result = users.value
  
  if (roleFilter.value) {
    result = result.filter(u => u.role === roleFilter.value)
  }
  
  if (searchQuery.value) {
    const q = searchQuery.value.toLowerCase()
    result = result.filter(u =>
      u.name.toLowerCase().includes(q) ||
      u.email.toLowerCase().includes(q)
    )
  }
  
  return result
})

const updateRole = async (user, newRole) => {
  try {
    await $fetch(`/api/admin/users/${user.id}/role`, {
      method: 'PATCH',
      body: { role: newRole }
    })
    user.role = newRole
  } catch (err) {
    alert('ไม่สามารถเปลี่ยน role ได้: ' + err.message)
  }
}

const openCreateModal = () => {
  selectedUser.value = null
  showModal.value = true
}

const editUser = (user) => {
  selectedUser.value = user
  showModal.value = true
}

const viewUser = (user) => {
  navigateTo(`/admin/users/${user.id}`)
}

const confirmDelete = (user) => {
  userToDelete.value = user
}

const deleteUser = async () => {
  try {
    await $fetch(`/api/admin/users/${userToDelete.value.id}`, {
      method: 'DELETE'
    })
    users.value = users.value.filter(u => u.id !== userToDelete.value.id)
    userToDelete.value = null
  } catch (err) {
    alert('ไม่สามารถลบผู้ใช้ได้: ' + err.message)
  }
}

const closeModal = () => {
  showModal.value = false
  selectedUser.value = null
}

const handleUserSaved = async () => {
  closeModal()
  await refresh()
}

const formatDate = (date) => {
  return new Date(date).toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>

<style scoped>
.admin-users { padding: 24px; max-width: 1200px; margin: 0 auto; }

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.filters {
  display: flex;
  gap: 12px;
  margin-bottom: 20px;
}

.filter-select, .search-input {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 0.9em;
}

.search-input { flex: 1; }

.users-table-wrapper {
  overflow-x: auto;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.users-table {
  width: 100%;
  border-collapse: collapse;
  background: white;
}

.users-table th {
  padding: 12px 16px;
  text-align: left;
  font-weight: 600;
  color: #555;
  font-size: 0.85em;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  background: #f8f9fa;
  border-bottom: 1px solid #e9ecef;
}

.users-table td {
  padding: 12px 16px;
  border-bottom: 1px solid #f0f0f0;
  vertical-align: middle;
}

.users-table tr:last-child td { border-bottom: none; }
.users-table tr:hover td { background: #f8f9fa; }

.user-cell {
  display: flex;
  align-items: center;
  gap: 12px;
}

.user-avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  object-fit: cover;
}

.you-badge {
  background: #e3f2fd;
  color: #1565c0;
  padding: 1px 6px;
  border-radius: 10px;
  font-size: 0.75em;
  margin-left: 6px;
}

.role-select, .role-badge {
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 0.85em;
  font-weight: 500;
}

.role-select {
  border: 1px solid #ddd;
  cursor: pointer;
}

.role-admin, select.role-admin { background: #fce4ec; color: #c62828; }
.role-editor { background: #e8eaf6; color: #283593; }
.role-author { background: #e8f5e9; color: #1b5e20; }
.role-viewer { background: #f5f5f5; color: #424242; }

.actions {
  display: flex;
  gap: 4px;
}

.btn-icon {
  width: 32px;
  height: 32px;
  background: none;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.btn-icon:hover { background: #f0f0f0; }
.btn-icon.danger:hover { background: #fee; }

.btn-primary {
  padding: 8px 20px;
  background: #2196F3;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9em;
}

.empty-state {
  text-align: center;
  padding: 48px;
  color: #999;
}
</style>
```

```typescript
// server/api/admin/users/[id]/role.patch.ts
export default defineEventHandler(async (event) => {
  const { role } = await readBody(event)
  const targetId = getRouterParam(event, 'id')
  
  // ตรวจสอบว่าเป็น admin
  await requirePermission(event, 'users:manage_roles')
  
  const currentUser = await requireAuth(event)
  
  // ห้าม admin เปลี่ยน role ตัวเอง
  if (String(currentUser.id) === String(targetId)) {
    throw createError({ statusCode: 400, message: 'ไม่สามารถเปลี่ยน role ของตัวเองได้' })
  }
  
  const validRoles = ['admin', 'editor', 'author', 'viewer']
  if (!validRoles.includes(role)) {
    throw createError({ statusCode: 400, message: 'Role ไม่ถูกต้อง' })
  }
  
  const updatedUser = await prisma.user.update({
    where: { id: targetId },
    data: { role },
    select: { id: true, name: true, email: true, role: true }
  })
  
  return updatedUser
})
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **RBAC Concept** - หลักการควบคุมสิทธิ์ตาม Role
2. **Permission System** - การออกแบบระบบ Permissions อย่างยืดหยุ่น
3. **Role Hierarchy** - ลำดับชั้นของ Roles
4. **Component-level** - PermissionGuard component
5. **API-level** - Server middleware ตรวจสอบสิทธิ์
6. **Nuxt Middleware** - ป้องกัน routes ด้วย RBAC
7. **Admin Panel** - ตัวอย่างระบบจัดการผู้ใช้สมบูรณ์
