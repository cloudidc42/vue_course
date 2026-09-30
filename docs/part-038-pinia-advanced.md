# Part 38: Pinia Advanced

## ทำไมต้องเรียน Pinia Advanced?

Pinia พื้นฐานช่วยจัดการ state ได้ดี แต่แอปพลิเคชันจริงมีความต้องการที่ซับซ้อนกว่า เช่น การ communicate ระหว่าง stores, optimistic updates, plugins, และการทดสอบ

---

## 1. Store Composition

### Setup Store Pattern (แนะนำ)

```ts
// stores/auth.ts - Composition Store
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'editor' | 'viewer'
  avatar?: string
}

export const useAuthStore = defineStore('auth', () => {
  // State
  const user = ref<User | null>(null)
  const token = ref<string | null>(
    typeof window !== 'undefined' ? localStorage.getItem('token') : null
  )
  const isLoading = ref(false)

  // Getters
  const isAuthenticated = computed(() => !!token.value)
  const isAdmin = computed(() => user.value?.role === 'admin')
  const isEditor = computed(() =>
    user.value?.role === 'admin' || user.value?.role === 'editor'
  )
  const displayName = computed(() => user.value?.name ?? 'Guest')

  // Actions
  async function login(email: string, password: string): Promise<void> {
    isLoading.value = true
    try {
      const { user: userData, token: authToken } = await $fetch<{
        user: User
        token: string
      }>('/api/auth/login', {
        method: 'POST',
        body: { email, password }
      })

      user.value = userData
      token.value = authToken

      if (typeof window !== 'undefined') {
        localStorage.setItem('token', authToken)
      }
    } finally {
      isLoading.value = false
    }
  }

  async function logout(): Promise<void> {
    await $fetch('/api/auth/logout', { method: 'POST' }).catch(() => {})
    user.value = null
    token.value = null
    if (typeof window !== 'undefined') {
      localStorage.removeItem('token')
    }
  }

  async function fetchCurrentUser(): Promise<void> {
    if (!token.value) return
    try {
      user.value = await $fetch<User>('/api/auth/me', {
        headers: { Authorization: `Bearer ${token.value}` }
      })
    } catch {
      await logout()
    }
  }

  // $reset equivalent for setup stores
  function $reset() {
    user.value = null
    token.value = null
    isLoading.value = false
  }

  return {
    user,
    token,
    isLoading,
    isAuthenticated,
    isAdmin,
    isEditor,
    displayName,
    login,
    logout,
    fetchCurrentUser,
    $reset
  }
})
```

---

## 2. Cross-store Communication

```ts
// stores/posts.ts - ใช้ authStore
import { defineStore } from 'pinia'
import { useAuthStore } from './auth'

export interface Post {
  id: number
  title: string
  content: string
  authorId: number
  author?: { name: string }
  publishedAt: string | null
  status: 'draft' | 'published' | 'archived'
}

export const usePostsStore = defineStore('posts', () => {
  const authStore = useAuthStore()

  const posts = ref<Post[]>([])
  const isLoading = ref(false)

  // Computed ที่ขึ้นอยู่กับ authStore
  const myPosts = computed(() =>
    posts.value.filter(p => p.authorId === authStore.user?.id)
  )

  const canCreatePost = computed(() => authStore.isEditor)

  async function fetchPosts(): Promise<void> {
    isLoading.value = true
    try {
      posts.value = await $fetch<Post[]>('/api/posts')
    } finally {
      isLoading.value = false
    }
  }

  async function createPost(data: Pick<Post, 'title' | 'content'>): Promise<Post> {
    if (!authStore.isAuthenticated) {
      throw new Error('กรุณาเข้าสู่ระบบก่อน')
    }

    const newPost = await $fetch<Post>('/api/posts', {
      method: 'POST',
      body: {
        ...data,
        authorId: authStore.user!.id
      }
    })

    posts.value.push(newPost)
    return newPost
  }

  async function publishPost(id: number): Promise<void> {
    if (!authStore.isEditor) {
      throw new Error('ไม่มีสิทธิ์เผยแพร่บทความ')
    }

    const post = posts.value.find(p => p.id === id)
    if (!post) throw new Error('ไม่พบบทความ')

    const updated = await $fetch<Post>(`/api/posts/${id}`, {
      method: 'PATCH',
      body: { status: 'published', publishedAt: new Date().toISOString() }
    })

    Object.assign(post, updated)
  }

  return {
    posts,
    isLoading,
    myPosts,
    canCreatePost,
    fetchPosts,
    createPost,
    publishPost
  }
})
```

---

## 3. Actions กับ Async Operations

```ts
// stores/notifications.ts - Async Actions ที่ซับซ้อน
export interface Notification {
  id: string
  type: 'info' | 'success' | 'warning' | 'error'
  title: string
  message: string
  isRead: boolean
  createdAt: string
}

export const useNotificationsStore = defineStore('notifications', () => {
  const notifications = ref<Notification[]>([])
  const isLoading = ref(false)
  const page = ref(1)
  const hasMore = ref(true)

  const unreadCount = computed(() =>
    notifications.value.filter(n => !n.isRead).length
  )

  // Fetch với pagination
  async function fetchNotifications(reset = false): Promise<void> {
    if (isLoading.value) return
    if (!hasMore.value && !reset) return

    isLoading.value = true
    if (reset) {
      page.value = 1
      hasMore.value = true
    }

    try {
      const data = await $fetch<{
        items: Notification[]
        hasMore: boolean
      }>('/api/notifications', {
        query: { page: page.value, limit: 20 }
      })

      if (reset) {
        notifications.value = data.items
      } else {
        notifications.value.push(...data.items)
      }

      hasMore.value = data.hasMore
      page.value++
    } finally {
      isLoading.value = false
    }
  }

  async function markAsRead(id: string): Promise<void> {
    await $fetch(`/api/notifications/${id}/read`, { method: 'POST' })
    const notification = notifications.value.find(n => n.id === id)
    if (notification) notification.isRead = true
  }

  async function markAllAsRead(): Promise<void> {
    await $fetch('/api/notifications/read-all', { method: 'POST' })
    notifications.value.forEach(n => { n.isRead = true })
  }

  async function deleteNotification(id: string): Promise<void> {
    await $fetch(`/api/notifications/${id}`, { method: 'DELETE' })
    notifications.value = notifications.value.filter(n => n.id !== id)
  }

  return {
    notifications,
    isLoading,
    unreadCount,
    hasMore,
    fetchNotifications,
    markAsRead,
    markAllAsRead,
    deleteNotification
  }
})
```

---

## 4. Optimistic Updates

```ts
// stores/todos.ts - Optimistic Updates
export interface Todo {
  id: number
  title: string
  completed: boolean
  userId: number
}

export const useTodosStore = defineStore('todos', () => {
  const todos = ref<Todo[]>([])
  const pendingIds = ref<Set<number>>(new Set())

  async function toggleTodo(id: number): Promise<void> {
    const todo = todos.value.find(t => t.id === id)
    if (!todo) return

    // Optimistic update - เปลี่ยน UI ก่อน
    todo.completed = !todo.completed
    pendingIds.value.add(id)

    try {
      const updated = await $fetch<Todo>(`/api/todos/${id}`, {
        method: 'PATCH',
        body: { completed: todo.completed }
      })
      Object.assign(todo, updated)
    } catch {
      // Rollback ถ้าเกิด error
      todo.completed = !todo.completed
      console.error('Failed to toggle todo, reverting')
    } finally {
      pendingIds.value.delete(id)
    }
  }

  async function deleteTodo(id: number): Promise<void> {
    const index = todos.value.findIndex(t => t.id === id)
    if (index === -1) return

    // เก็บ backup ไว้ก่อน
    const deletedTodo = todos.value[index]

    // Optimistic: ลบออกก่อน
    todos.value.splice(index, 1)

    try {
      await $fetch(`/api/todos/${id}`, { method: 'DELETE' })
    } catch {
      // Rollback: นำกลับมาใส่
      todos.value.splice(index, 0, deletedTodo)
    }
  }

  async function createTodo(title: string): Promise<void> {
    // Optimistic: เพิ่ม temp todo
    const tempId = -Date.now()
    const tempTodo: Todo = {
      id: tempId,
      title,
      completed: false,
      userId: 1
    }
    todos.value.push(tempTodo)

    try {
      const created = await $fetch<Todo>('/api/todos', {
        method: 'POST',
        body: { title }
      })
      // แทนที่ temp todo ด้วย real todo
      const index = todos.value.findIndex(t => t.id === tempId)
      if (index !== -1) todos.value[index] = created
    } catch {
      // ลบ temp todo ออก
      todos.value = todos.value.filter(t => t.id !== tempId)
    }
  }

  return {
    todos,
    pendingIds: readonly(pendingIds),
    toggleTodo,
    deleteTodo,
    createTodo
  }
})
```

---

## 5. Pinia Plugins

```ts
// plugins/pinia-logger.ts - Plugin สำหรับ log actions
import type { PiniaPlugin, PiniaPluginContext } from 'pinia'

export const piniaLogger: PiniaPlugin = ({ store }: PiniaPluginContext) => {
  // Override $patch
  const originalPatch = store.$patch.bind(store)
  store.$patch = function(partialStateOrMutator: any) {
    console.group(`[Pinia] ${store.$id} $patch`)
    console.log('Before:', JSON.parse(JSON.stringify(store.$state)))
    originalPatch(partialStateOrMutator)
    console.log('After:', JSON.parse(JSON.stringify(store.$state)))
    console.groupEnd()
  }

  // ติดตาม actions
  store.$onAction(({ name, args, after, onError }) => {
    const startTime = Date.now()
    console.log(`[Pinia] ${store.$id}.${name}() called with:`, args)

    after((result) => {
      console.log(
        `[Pinia] ${store.$id}.${name}() finished in ${Date.now() - startTime}ms:`,
        result
      )
    })

    onError((error) => {
      console.error(
        `[Pinia] ${store.$id}.${name}() failed after ${Date.now() - startTime}ms:`,
        error
      )
    })
  })
}
```

```ts
// plugins/pinia-setup.ts - Setup plugins
import { createPinia } from 'pinia'
import { piniaLogger } from './pinia-logger'

const pinia = createPinia()

// ใช้ plugin เฉพาะใน development
if (process.env.NODE_ENV === 'development') {
  pinia.use(piniaLogger)
}
```

---

## 6. pinia-plugin-persistedstate

```bash
npm install pinia-plugin-persistedstate
```

```ts
// plugins/pinia-persist.ts
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)
```

```ts
// stores/settings.ts - Store ที่ persist state
export const useSettingsStore = defineStore('settings', () => {
  const theme = ref<'light' | 'dark' | 'system'>('system')
  const language = ref('th')
  const fontSize = ref<'sm' | 'md' | 'lg'>('md')
  const sidebarOpen = ref(true)

  function setTheme(newTheme: typeof theme.value) {
    theme.value = newTheme
  }

  function toggleSidebar() {
    sidebarOpen.value = !sidebarOpen.value
  }

  return { theme, language, fontSize, sidebarOpen, setTheme, toggleSidebar }
}, {
  persist: {
    key: 'app-settings',
    storage: localStorage,
    // เลือกเฉพาะ state ที่ต้องการ persist
    pick: ['theme', 'language', 'fontSize']
    // หรือ exclude บาง fields
    // omit: ['sidebarOpen']
  }
})
```

```ts
// stores/cart.ts - Cart ที่ persist state
export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])

  // ... actions

  return { items, /* ... */ }
}, {
  persist: {
    key: 'shopping-cart',
    storage: {
      getItem: (key: string) => {
        // Custom storage - ใช้ sessionStorage
        return sessionStorage.getItem(key)
      },
      setItem: (key: string, value: string) => {
        sessionStorage.setItem(key, value)
      },
      removeItem: (key: string) => {
        sessionStorage.removeItem(key)
      }
    }
  }
})
```

---

## 7. Testing Pinia Stores

```ts
// tests/stores/posts.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { usePostsStore } from '@/stores/posts'
import { useAuthStore } from '@/stores/auth'

// Mock $fetch
vi.mock('ofetch', () => ({ $fetch: vi.fn() }))
const mockFetch = vi.fn()
global.$fetch = mockFetch

describe('usePostsStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    mockFetch.mockReset()
  })

  it('creates post when authenticated', async () => {
    // Setup auth
    const authStore = useAuthStore()
    authStore.$patch({
      user: { id: 1, name: 'Test', email: 'test@test.com', role: 'editor' },
      token: 'mock-token'
    })

    const newPost = {
      id: 1,
      title: 'Test Post',
      content: 'Content',
      authorId: 1,
      status: 'draft' as const,
      publishedAt: null
    }
    mockFetch.mockResolvedValueOnce(newPost)

    const postsStore = usePostsStore()
    const result = await postsStore.createPost({
      title: 'Test Post',
      content: 'Content'
    })

    expect(result).toEqual(newPost)
    expect(postsStore.posts).toHaveLength(1)
  })

  it('throws when creating post unauthenticated', async () => {
    const postsStore = usePostsStore()
    await expect(postsStore.createPost({
      title: 'Test',
      content: 'Content'
    })).rejects.toThrow('กรุณาเข้าสู่ระบบก่อน')
  })
})
```

---

## 8. Reset Store State

```ts
// composables/useStoreReset.ts - Reset multiple stores
export function useStoreReset() {
  const authStore = useAuthStore()
  const cartStore = useCartStore()
  const settingsStore = useSettingsStore()

  async function resetAllStores() {
    // Reset stores on logout
    authStore.$reset()
    cartStore.$reset()
    // settings ไม่ reset - เพราะ user preferences ควรเก็บไว้
  }

  return { resetAllStores }
}
```

---

## 9. ตัวอย่าง: Complex User & Posts Stores

```ts
// stores/users.ts
export interface UserWithStats extends User {
  postsCount: number
  followersCount: number
  followingCount: number
}

export const useUsersStore = defineStore('users', () => {
  const users = ref<Map<number, UserWithStats>>(new Map())
  const followingIds = ref<Set<number>>(new Set())
  const isLoading = ref(false)

  // ดึงข้อมูล user พร้อม cache
  async function getUser(id: number): Promise<UserWithStats | null> {
    if (users.value.has(id)) {
      return users.value.get(id)!
    }

    try {
      const user = await $fetch<UserWithStats>(`/api/users/${id}`)
      users.value.set(id, user)
      return user
    } catch {
      return null
    }
  }

  async function followUser(targetId: number): Promise<void> {
    // Optimistic update
    followingIds.value.add(targetId)

    const targetUser = users.value.get(targetId)
    if (targetUser) targetUser.followersCount++

    try {
      await $fetch(`/api/users/${targetId}/follow`, { method: 'POST' })
    } catch {
      // Rollback
      followingIds.value.delete(targetId)
      if (targetUser) targetUser.followersCount--
    }
  }

  async function unfollowUser(targetId: number): Promise<void> {
    followingIds.value.delete(targetId)

    const targetUser = users.value.get(targetId)
    if (targetUser) targetUser.followersCount--

    try {
      await $fetch(`/api/users/${targetId}/unfollow`, { method: 'POST' })
    } catch {
      followingIds.value.add(targetId)
      if (targetUser) targetUser.followersCount++
    }
  }

  function isFollowing(userId: number): boolean {
    return followingIds.value.has(userId)
  }

  return {
    users: readonly(users),
    followingIds: readonly(followingIds),
    isLoading,
    getUser,
    followUser,
    unfollowUser,
    isFollowing
  }
})
```

```ts
// stores/posts.advanced.ts - Posts store ที่ interact กับ users store
export const usePostsFeedStore = defineStore('postsFeed', () => {
  const usersStore = useUsersStore()
  const authStore = useAuthStore()

  const feedPosts = ref<Post[]>([])
  const savedPostIds = ref<Set<number>>(new Set())
  const isLoading = ref(false)

  // Posts enriched ด้วย author data จาก usersStore
  const enrichedPosts = computed(async () => {
    const result = []
    for (const post of feedPosts.value) {
      const author = await usersStore.getUser(post.authorId)
      result.push({ ...post, author })
    }
    return result
  })

  async function loadFeed(): Promise<void> {
    if (!authStore.isAuthenticated) return

    isLoading.value = true
    try {
      const posts = await $fetch<Post[]>('/api/feed')
      feedPosts.value = posts

      // Pre-fetch authors
      const uniqueAuthorIds = [...new Set(posts.map(p => p.authorId))]
      await Promise.all(uniqueAuthorIds.map(id => usersStore.getUser(id)))
    } finally {
      isLoading.value = false
    }
  }

  async function savePost(postId: number): Promise<void> {
    // Optimistic
    savedPostIds.value.add(postId)
    try {
      await $fetch(`/api/posts/${postId}/save`, { method: 'POST' })
    } catch {
      savedPostIds.value.delete(postId)
    }
  }

  async function unsavePost(postId: number): Promise<void> {
    savedPostIds.value.delete(postId)
    try {
      await $fetch(`/api/posts/${postId}/unsave`, { method: 'POST' })
    } catch {
      savedPostIds.value.add(postId)
    }
  }

  function isSaved(postId: number): boolean {
    return savedPostIds.value.has(postId)
  }

  // Subscribe ดัก action จาก stores อื่น
  const authStoreSubscription = authStore.$onAction(({ name, after }) => {
    after(() => {
      if (name === 'logout') {
        // เมื่อ logout ให้ clear feed
        feedPosts.value = []
        savedPostIds.value = new Set()
      }
    })
  })

  onUnmounted(() => {
    authStoreSubscription()
  })

  return {
    feedPosts: readonly(feedPosts),
    savedPostIds: readonly(savedPostIds),
    isLoading,
    loadFeed,
    savePost,
    unsavePost,
    isSaved
  }
})
```

---

## สรุป

Pinia Advanced ช่วยจัดการ state ที่ซับซ้อนได้:

1. **Store Composition** - ใช้ Setup Store pattern สำหรับ logic ที่ซับซ้อน
2. **Cross-store Communication** - ใช้ store หนึ่งภายในอีก store หนึ่ง
3. **Async Actions** - จัดการ pagination, loading states, error handling
4. **Optimistic Updates** - อัปเดต UI ก่อน แล้ว rollback ถ้าเกิด error
5. **Plugins** - เพิ่ม functionality ให้ทุก store (logging, persistence)
6. **Persistedstate** - เก็บ state ไว้ใน localStorage/sessionStorage
7. **Testing** - ใช้ `setActivePinia` เพื่อ test stores อย่างถูกต้อง
8. **Reset State** - Reset stores เมื่อ logout หรือเมื่อต้องการ
