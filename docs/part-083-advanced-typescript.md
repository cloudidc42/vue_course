# Part 83: Advanced TypeScript Patterns

## Advanced TypeScript สำหรับ Vue Developer

TypeScript ขั้นสูงช่วยให้เราเขียน code ที่ type-safe มากขึ้น ลด bugs และทำให้ IDE support ดีขึ้นมาก

---

## 1. Generic Components

### Basic Generic Component

```typescript
// components/DataList.vue
<script setup lang="ts" generic="T extends { id: string }">
const props = defineProps<{
  items: T[]
  getLabel: (item: T) => string
  loading?: boolean
}>()

const emit = defineEmits<{
  select: [item: T]
}>()

function handleSelect(item: T) {
  emit('select', item)
}
</script>

<template>
  <div class="data-list">
    <div v-if="loading">Loading...</div>
    <div
      v-else
      v-for="item in items"
      :key="item.id"
      @click="handleSelect(item)"
    >
      {{ getLabel(item) }}
    </div>
  </div>
</template>
```

### Generic Table Component

```typescript
// components/GenericTable.vue
<script setup lang="ts" generic="TRow extends Record<string, unknown>">

type ColumnDef<T extends Record<string, unknown>> = {
  key: keyof T & string
  header: string
  sortable?: boolean
  render?: (value: T[typeof key], row: T) => string | number
}

const props = defineProps<{
  columns: ColumnDef<TRow>[]
  rows: TRow[]
  sortKey?: keyof TRow & string
  sortDirection?: 'asc' | 'desc'
}>()

const emit = defineEmits<{
  sort: [key: keyof TRow & string]
  rowClick: [row: TRow]
}>()

function getCellValue(row: TRow, col: ColumnDef<TRow>): string | number {
  const value = row[col.key]
  if (col.render) return col.render(value as TRow[typeof col.key], row)
  return String(value ?? '')
}
</script>

<template>
  <table>
    <thead>
      <tr>
        <th
          v-for="col in columns"
          :key="col.key"
          :class="{ sortable: col.sortable }"
          @click="col.sortable && emit('sort', col.key)"
        >
          {{ col.header }}
        </th>
      </tr>
    </thead>
    <tbody>
      <tr
        v-for="(row, idx) in rows"
        :key="String(row.id ?? idx)"
        @click="emit('rowClick', row)"
      >
        <td v-for="col in columns" :key="col.key">
          {{ getCellValue(row, col) }}
        </td>
      </tr>
    </tbody>
  </table>
</template>
```

---

## 2. Template Literal Types

### Route Types

```typescript
// types/routes.ts

// Base routes
type BaseRoute = '/dashboard' | '/profile' | '/settings'

// Dynamic routes
type UserRoute = `/users/${string}`
type ProjectRoute = `/projects/${string}`
type ApiRoute = `/api/v${number}/${string}`

// Combined
type AppRoute = BaseRoute | UserRoute | ProjectRoute

// ใช้ใน router
function navigate(route: AppRoute) {
  return useRouter().push(route)
}

navigate('/dashboard')        // ✅
navigate('/users/123')        // ✅
navigate('/invalid-route')    // ❌ Error!
```

### Event Types

```typescript
// types/events.ts
type EventName = 'click' | 'focus' | 'blur' | 'change'
type ElementType = 'button' | 'input' | 'select'

type ElementEvent = `${ElementType}:${EventName}`
// = 'button:click' | 'button:focus' | 'input:click' | ...

// สำหรับ analytics
function trackEvent(event: ElementEvent, data?: Record<string, unknown>) {
  console.log(`[Analytics] ${event}`, data)
}

trackEvent('button:click', { label: 'Submit' })  // ✅
trackEvent('form:submit')  // ❌ Error! 'form' ไม่อยู่ใน ElementType
```

### CSS Properties Type

```typescript
// Strongly typed CSS-in-JS
type CSSProperty = 'color' | 'background' | 'fontSize' | 'margin' | 'padding'
type CSSValue = string | number

type CSSObject = {
  [K in CSSProperty]?: CSSValue
}

// camelCase to kebab-case
type KebabCase<S extends string> = S extends `${infer First}${infer Rest}`
  ? First extends Uppercase<First>
    ? `-${Lowercase<First>}${KebabCase<Rest>}`
    : `${First}${KebabCase<Rest>}`
  : S

type CSSVar<Name extends string> = `--${KebabCase<Name>}`

// ใช้งาน
const cssVar: CSSVar<'primaryColor'> = '--primary-color'  // ✅
const badVar: CSSVar<'primaryColor'> = '--primaryColor'   // ❌
```

---

## 3. Conditional Types

### Type Guards

```typescript
// utils/type-guards.ts

// Basic type guard
function isString(value: unknown): value is string {
  return typeof value === 'string'
}

// Object shape guard
interface User {
  id: string
  email: string
  role: 'admin' | 'user'
}

interface AdminUser extends User {
  role: 'admin'
  adminLevel: number
}

function isAdminUser(user: User): user is AdminUser {
  return user.role === 'admin' && 'adminLevel' in user
}

// Conditional type
type NonNullable<T> = T extends null | undefined ? never : T

type UnwrapPromise<T> = T extends Promise<infer U> ? U : T

type UnwrapArray<T> = T extends Array<infer U> ? U : T

// Deep partial
type DeepPartial<T> = T extends object ? {
  [K in keyof T]?: DeepPartial<T[K]>
} : T

// Required fields only
type RequiredKeys<T> = {
  [K in keyof T]-?: undefined extends T[K] ? never : K
}[keyof T]

// Optional fields only
type OptionalKeys<T> = {
  [K in keyof T]-?: undefined extends T[K] ? K : never
}[keyof T]

// ตัวอย่าง
interface Config {
  apiUrl: string       // required
  timeout?: number     // optional
  debug?: boolean      // optional
}

type ConfigRequired = RequiredKeys<Config>  // 'apiUrl'
type ConfigOptional = OptionalKeys<Config>  // 'timeout' | 'debug'
```

---

## 4. Mapped Types

### API Response Mapper

```typescript
// types/api-mapper.ts

// แปลง snake_case เป็น camelCase
type CamelCase<S extends string> = S extends `${infer First}_${infer Rest}`
  ? `${First}${Capitalize<CamelCase<Rest>>}`
  : S

type CamelCaseKeys<T> = {
  [K in keyof T as CamelCase<string & K>]: T[K] extends object
    ? CamelCaseKeys<T[K]>
    : T[K]
}

// ตัวอย่าง
interface ApiUser {
  user_id: string
  first_name: string
  last_name: string
  created_at: string
}

type User = CamelCaseKeys<ApiUser>
// = { userId: string; firstName: string; lastName: string; createdAt: string }
```

### Form Field Types

```typescript
// types/form.ts

type FormFieldType = 'text' | 'email' | 'password' | 'select' | 'checkbox' | 'textarea'

interface FieldConfig {
  type: FormFieldType
  label: string
  required?: boolean
  validators?: Array<(value: unknown) => string | undefined>
}

type FormSchema<T extends Record<string, unknown>> = {
  [K in keyof T]: FieldConfig & {
    initialValue?: T[K]
  }
}

// ใช้งาน
interface LoginForm {
  email: string
  password: string
  rememberMe: boolean
}

const loginSchema: FormSchema<LoginForm> = {
  email: {
    type: 'email',
    label: 'Email',
    required: true,
    validators: [(v) => isValidEmail(String(v)) ? undefined : 'Invalid email'],
    initialValue: ''
  },
  password: {
    type: 'password',
    label: 'Password',
    required: true,
    validators: [(v) => String(v).length >= 8 ? undefined : 'Min 8 characters'],
    initialValue: ''
  },
  rememberMe: {
    type: 'checkbox',
    label: 'Remember me',
    initialValue: false
  }
}
```

---

## 5. Discriminated Unions

### State Machine Pattern

```typescript
// types/async-state.ts

type AsyncState<T, E = Error> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: E }

// Composable ที่ใช้ Discriminated Union
function useAsyncState<T>(fn: () => Promise<T>) {
  const state = ref<AsyncState<T>>({ status: 'idle' })

  async function execute() {
    state.value = { status: 'loading' }
    try {
      const data = await fn()
      state.value = { status: 'success', data }
    } catch (error) {
      state.value = { status: 'error', error: error as Error }
    }
  }

  return { state, execute }
}

// ใน component - TypeScript รู้ type ใน each branch
const { state, execute } = useAsyncState(() => fetchUser('123'))

// Template
// <div v-if="state.status === 'loading'">Loading...</div>
// <div v-else-if="state.status === 'success'">
//   {{ state.data.name }}  ← TypeScript รู้ว่า state.data มี type T
// </div>
```

### Event System

```typescript
// types/events.ts

type DomainEvent =
  | { type: 'USER_CREATED'; payload: { userId: string; email: string } }
  | { type: 'USER_UPDATED'; payload: { userId: string; changes: Partial<User> } }
  | { type: 'USER_DELETED'; payload: { userId: string } }
  | { type: 'ORDER_PLACED'; payload: { orderId: string; items: OrderItem[] } }
  | { type: 'ORDER_CANCELLED'; payload: { orderId: string; reason: string } }

type EventHandler<T extends DomainEvent> = (event: T) => Promise<void> | void

type EventHandlerMap = {
  [K in DomainEvent['type']]: EventHandler<Extract<DomainEvent, { type: K }>>
}

// Event Bus ที่ type-safe
class TypedEventBus {
  private handlers: Partial<EventHandlerMap> = {}

  on<T extends DomainEvent['type']>(
    type: T,
    handler: EventHandler<Extract<DomainEvent, { type: T }>>
  ) {
    this.handlers[type] = handler as any
    return this
  }

  async emit(event: DomainEvent) {
    const handler = this.handlers[event.type]
    if (handler) {
      await (handler as EventHandler<typeof event>)(event)
    }
  }
}

const bus = new TypedEventBus()

bus.on('USER_CREATED', async (event) => {
  // TypeScript รู้ว่า event.payload.userId และ event.payload.email มีอยู่
  await sendWelcomeEmail(event.payload.email)
})
```

---

## 6. Type-safe Event Bus

```typescript
// composables/useEventBus.ts

type EventMap = {
  'auth:login': { userId: string; email: string }
  'auth:logout': void
  'cart:add': { productId: string; quantity: number }
  'cart:remove': { productId: string }
  'notification:show': { message: string; type: 'success' | 'error' | 'warning' }
}

type EventKey = keyof EventMap
type EventPayload<K extends EventKey> = EventMap[K]

const eventBus = new Map<string, Set<Function>>()

export function useEventBus() {
  function emit<K extends EventKey>(
    event: K,
    ...args: EventPayload<K> extends void ? [] : [EventPayload<K>]
  ) {
    const handlers = eventBus.get(event)
    if (handlers) {
      handlers.forEach(handler => handler(...args))
    }
  }

  function on<K extends EventKey>(
    event: K,
    handler: EventPayload<K> extends void
      ? () => void
      : (payload: EventPayload<K>) => void
  ) {
    if (!eventBus.has(event)) {
      eventBus.set(event, new Set())
    }
    eventBus.get(event)!.add(handler)

    // Cleanup on unmount
    onUnmounted(() => {
      eventBus.get(event)?.delete(handler)
    })
  }

  return { emit, on }
}

// ใช้งาน
const { emit, on } = useEventBus()

// ✅ Type-safe
emit('auth:login', { userId: '123', email: 'test@test.com' })
emit('auth:logout')  // No payload needed

on('cart:add', (payload) => {
  // TypeScript รู้ว่า payload.productId และ payload.quantity มีอยู่
  console.log(payload.productId, payload.quantity)
})

// ❌ Error - wrong payload
emit('auth:login', { wrongField: 'value' })
```

---

## 7. ตัวอย่าง: Advanced Type-safe Components

### Form Builder

```typescript
// composables/useForm.ts

type FormValues<Schema extends Record<string, unknown>> = {
  [K in keyof Schema]: Schema[K]
}

type FormErrors<Schema extends Record<string, unknown>> = {
  [K in keyof Schema]?: string
}

type FieldValidator<T> = (value: T) => string | undefined

type ValidationSchema<Schema extends Record<string, unknown>> = {
  [K in keyof Schema]?: FieldValidator<Schema[K]>[]
}

export function useForm<T extends Record<string, unknown>>(options: {
  initialValues: T
  validators?: ValidationSchema<T>
  onSubmit: (values: T) => Promise<void> | void
}) {
  const values = reactive({ ...options.initialValues }) as T
  const errors = reactive({} as FormErrors<T>)
  const isSubmitting = ref(false)
  const isDirty = ref(false)

  function setField<K extends keyof T>(key: K, value: T[K]) {
    values[key] = value
    isDirty.value = true

    // Validate field
    if (options.validators?.[key]) {
      const fieldErrors = options.validators[key]!
        .map(v => v(value))
        .filter(Boolean)
      errors[key] = fieldErrors[0] as string | undefined
    }
  }

  async function submit() {
    // Validate all fields
    let hasErrors = false
    if (options.validators) {
      for (const key of Object.keys(options.validators) as Array<keyof T>) {
        const fieldValidators = options.validators[key]!
        const fieldValue = values[key]
        const fieldErrors = fieldValidators
          .map(v => v(fieldValue))
          .filter(Boolean)

        if (fieldErrors.length > 0) {
          errors[key] = fieldErrors[0] as string | undefined
          hasErrors = true
        }
      }
    }

    if (hasErrors) return

    isSubmitting.value = true
    try {
      await options.onSubmit({ ...values })
    } finally {
      isSubmitting.value = false
    }
  }

  return {
    values: readonly(values) as Readonly<T>,
    errors: readonly(errors) as Readonly<FormErrors<T>>,
    isSubmitting: readonly(isSubmitting),
    isDirty: readonly(isDirty),
    setField,
    submit
  }
}

// ใช้งาน
interface RegisterForm {
  name: string
  email: string
  password: string
  confirmPassword: string
}

const form = useForm<RegisterForm>({
  initialValues: {
    name: '',
    email: '',
    password: '',
    confirmPassword: ''
  },
  validators: {
    name: [(v) => v.length < 2 ? 'Name too short' : undefined],
    email: [(v) => !v.includes('@') ? 'Invalid email' : undefined],
    password: [(v) => v.length < 8 ? 'Min 8 chars' : undefined],
    confirmPassword: [
      (v) => v !== form.values.password ? 'Passwords do not match' : undefined
    ]
  },
  onSubmit: async (values) => {
    await registerUser(values)
  }
})

// TypeScript รู้ type ของทุก field
form.setField('email', 'test@test.com')  // ✅
form.setField('email', 123)              // ❌ Error!
```

---

## สรุป

Advanced TypeScript Patterns ที่สำคัญ:
1. **Generic components** - Reusable components ที่ type-safe
2. **Template literal types** - สร้าง string types ที่แม่นยำ
3. **Conditional types** - Logic ใน type system
4. **Mapped types** - Transform types อัตโนมัติ
5. **Discriminated unions** - Type-safe state management
6. **Type-safe event bus** - Event system ที่ไม่เกิด typos
