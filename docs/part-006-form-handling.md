# Part 6: Form Handling และ v-model

## บทนำ

Form Handling เป็นหนึ่งในสิ่งที่ developer ทำบ่อยที่สุด Vue.js มี `v-model` directive ที่ทำให้การ bind ข้อมูลกับ form elements ทำได้ง่ายมาก รวมถึงการสร้าง custom v-model สำหรับ components และการทำ form validation ที่ครบครัน

---

## 1. v-model พื้นฐาน

`v-model` เป็น syntactic sugar สำหรับ two-way data binding ที่รวม `:value` และ `@input` เข้าด้วยกัน

### Text Input

```vue
<script setup lang="ts">
import { ref } from 'vue'

const name = ref('')
const bio = ref('')

// v-model กับ text input เทียบเท่ากับ:
// :value="name" @input="name = $event.target.value"
</script>

<template>
  <div class="form-demo">
    <!-- Text input -->
    <input v-model="name" placeholder="กรอกชื่อ" />
    <p>ชื่อ: {{ name }}</p>

    <!-- Textarea -->
    <textarea v-model="bio" placeholder="เกี่ยวกับตัวเอง" rows="4" />
    <p>Bio: {{ bio }}</p>
  </div>
</template>
```

### Number Input

```vue
<script setup lang="ts">
import { ref } from 'vue'

const age = ref<number>(0)
const price = ref<number>(0)

// v-model.number จะแปลงค่าเป็น number โดยอัตโนมัติ
// ถ้าไม่ใช้ .number ค่าจะเป็น string เสมอ
</script>

<template>
  <!-- ไม่มี .number - ค่าจะเป็น string "25" ไม่ใช่ number 25 -->
  <input type="number" v-model="age" />

  <!-- มี .number - ค่าจะเป็น number 25 -->
  <input type="number" v-model.number="price" />
  <p>ราคา: {{ price }} (type: {{ typeof price }})</p>
</template>
```

### Checkbox

```vue
<script setup lang="ts">
import { ref } from 'vue'

// Single checkbox - boolean
const isAgreed = ref(false)
const isSubscribed = ref(true)

// Multiple checkboxes - array
const selectedFruits = ref<string[]>([])
const hobbies = ref<string[]>(['reading'])

// Checkbox กับ custom true/false values
const status = ref<string>('inactive')
</script>

<template>
  <!-- Single checkbox -->
  <label>
    <input type="checkbox" v-model="isAgreed" />
    ยอมรับเงื่อนไข
  </label>
  <p>ยอมรับ: {{ isAgreed }}</p>

  <!-- Multiple checkboxes bind กับ array -->
  <div>
    <h4>เลือกผลไม้:</h4>
    <label v-for="fruit in ['Apple', 'Banana', 'Cherry']" :key="fruit">
      <input type="checkbox" v-model="selectedFruits" :value="fruit" />
      {{ fruit }}
    </label>
    <p>เลือก: {{ selectedFruits.join(', ') }}</p>
  </div>

  <!-- Custom true/false values -->
  <label>
    <input
      type="checkbox"
      v-model="status"
      true-value="active"
      false-value="inactive"
    />
    Status: {{ status }}
  </label>
</template>
```

### Radio Buttons

```vue
<script setup lang="ts">
import { ref } from 'vue'

const gender = ref('male')
const paymentMethod = ref('')
const size = ref<string>('')

interface Option {
  value: string
  label: string
}

const sizeOptions: Option[] = [
  { value: 'S', label: 'Small' },
  { value: 'M', label: 'Medium' },
  { value: 'L', label: 'Large' },
  { value: 'XL', label: 'X-Large' },
]
</script>

<template>
  <!-- Radio buttons -->
  <div>
    <h4>เพศ:</h4>
    <label><input type="radio" v-model="gender" value="male" /> ชาย</label>
    <label><input type="radio" v-model="gender" value="female" /> หญิง</label>
    <label><input type="radio" v-model="gender" value="other" /> อื่นๆ</label>
    <p>เพศ: {{ gender }}</p>
  </div>

  <!-- Radio จาก array of options -->
  <div>
    <h4>ขนาด:</h4>
    <label v-for="option in sizeOptions" :key="option.value">
      <input type="radio" v-model="size" :value="option.value" />
      {{ option.label }}
    </label>
  </div>
</template>
```

### Select

```vue
<script setup lang="ts">
import { ref } from 'vue'

const country = ref('')
const countries = ref([
  { code: 'TH', name: 'ประเทศไทย' },
  { code: 'US', name: 'สหรัฐอเมริกา' },
  { code: 'JP', name: 'ญี่ปุ่น' },
  { code: 'SG', name: 'สิงคโปร์' },
])

// Multiple select - array
const selectedSkills = ref<string[]>([])
const skills = ['Vue.js', 'React', 'Angular', 'Node.js', 'TypeScript']

// Select กับ object values
const selectedCountry = ref<{ code: string; name: string } | null>(null)
</script>

<template>
  <!-- Single select -->
  <select v-model="country">
    <option value="" disabled>-- เลือกประเทศ --</option>
    <option v-for="c in countries" :key="c.code" :value="c.code">
      {{ c.name }}
    </option>
  </select>
  <p>เลือก: {{ country }}</p>

  <!-- Multiple select -->
  <select v-model="selectedSkills" multiple>
    <option v-for="skill in skills" :key="skill" :value="skill">
      {{ skill }}
    </option>
  </select>
  <p>ทักษะ: {{ selectedSkills.join(', ') }}</p>

  <!-- Select กับ object value -->
  <select v-model="selectedCountry">
    <option :value="null" disabled>-- เลือก --</option>
    <option v-for="c in countries" :key="c.code" :value="c">
      {{ c.name }}
    </option>
  </select>
  <p v-if="selectedCountry">
    เลือก: {{ selectedCountry.name }} ({{ selectedCountry.code }})
  </p>
</template>
```

---

## 2. v-model Modifiers

### .lazy - อัปเดตเมื่อ change event (ออกจาก input)

```vue
<script setup lang="ts">
import { ref } from 'vue'

const normalValue = ref('')
const lazyValue = ref('')

// .lazy จะอัปเดตเมื่อ change event (blur หรือ Enter)
// แทนที่จะอัปเดตทุกครั้งที่พิมพ์ (input event)
</script>

<template>
  <div>
    <p>Normal (อัปเดตทุก keystroke):</p>
    <input v-model="normalValue" />
    <p>ค่า: {{ normalValue }}</p>

    <p>Lazy (อัปเดตเมื่อออกจาก input):</p>
    <input v-model.lazy="lazyValue" />
    <p>ค่า: {{ lazyValue }}</p>
  </div>
</template>
```

### .number - แปลงเป็น number อัตโนมัติ

```vue
<script setup lang="ts">
import { ref } from 'vue'

const ageString = ref<any>('')  // จะเป็น string
const ageNumber = ref<number>(0)  // จะเป็น number

// .number ใช้ parseFloat() แปลงค่า
// ถ้าค่า parse ไม่ได้จะคืน original string
</script>

<template>
  <div>
    <p>ไม่มี .number: {{ ageString }} ({{ typeof ageString }})</p>
    <input type="number" v-model="ageString" />

    <p>มี .number: {{ ageNumber }} ({{ typeof ageNumber }})</p>
    <input type="number" v-model.number="ageNumber" />
  </div>
</template>
```

### .trim - ตัด whitespace อัตโนมัติ

```vue
<script setup lang="ts">
import { ref } from 'vue'

const normalInput = ref('')
const trimmedInput = ref('')
</script>

<template>
  <div>
    <p>ไม่มี .trim: "{{ normalInput }}"</p>
    <input v-model="normalInput" placeholder="  มี spaces  " />

    <p>มี .trim: "{{ trimmedInput }}"</p>
    <input v-model.trim="trimmedInput" placeholder="  จะถูกตัด spaces  " />
  </div>
</template>
```

---

## 3. Custom v-model ใน Component

ตั้งแต่ Vue 3 เป็นต้นมา `v-model` บน components ใช้ `modelValue` prop และ `update:modelValue` event

```vue
<!-- CustomInput.vue -->
<script setup lang="ts">
interface Props {
  modelValue: string
  placeholder?: string
  label?: string
  error?: string
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: '',
  label: '',
  error: '',
})

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

function handleInput(event: Event) {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}
</script>

<template>
  <div class="custom-input" :class="{ 'has-error': error }">
    <label v-if="label" class="input-label">{{ label }}</label>
    <input
      :value="modelValue"
      :placeholder="placeholder"
      @input="handleInput"
      class="input-field"
    />
    <span v-if="error" class="error-text">{{ error }}</span>
  </div>
</template>
```

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import CustomInput from './CustomInput.vue'

const username = ref('')
const email = ref('')
</script>

<template>
  <!-- v-model บน custom component -->
  <CustomInput
    v-model="username"
    label="ชื่อผู้ใช้"
    placeholder="กรอกชื่อผู้ใช้"
  />

  <CustomInput
    v-model="email"
    label="อีเมล"
    placeholder="your@email.com"
  />

  <p>Username: {{ username }}</p>
  <p>Email: {{ email }}</p>
</template>
```

---

## 4. Multiple v-model Bindings

Vue 3 อนุญาตให้ใช้ v-model หลายตัวบน component เดียวกัน

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
interface Props {
  firstName: string
  lastName: string
  email: string
}

const props = defineProps<Props>()

const emit = defineEmits<{
  'update:firstName': [value: string]
  'update:lastName': [value: string]
  'update:email': [value: string]
}>()
</script>

<template>
  <div class="user-form">
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
import UserForm from './UserForm.vue'

const firstName = ref('สมชาย')
const lastName = ref('ใจดี')
const email = ref('somchai@example.com')
</script>

<template>
  <!-- Multiple v-model bindings -->
  <UserForm
    v-model:firstName="firstName"
    v-model:lastName="lastName"
    v-model:email="email"
  />
  <p>{{ firstName }} {{ lastName }} - {{ email }}</p>
</template>
```

---

## 5. Form Validation จาก Scratch

```vue
<script setup lang="ts">
import { ref, reactive, computed } from 'vue'

// Types
type ValidationRule = (value: any) => string | true
type FieldRules = ValidationRule[]

interface FormField {
  value: any
  rules: FieldRules
  touched: boolean
  dirty: boolean
}

// Validation rules
const rules = {
  required: (message = 'จำเป็นต้องกรอก'): ValidationRule =>
    (value) => {
      if (value === null || value === undefined || value === '') return message
      if (Array.isArray(value) && value.length === 0) return message
      return true
    },

  minLength: (min: number): ValidationRule =>
    (value) => {
      if (!value) return true
      return String(value).length >= min || `ต้องมีอย่างน้อย ${min} ตัวอักษร`
    },

  maxLength: (max: number): ValidationRule =>
    (value) => {
      if (!value) return true
      return String(value).length <= max || `ต้องไม่เกิน ${max} ตัวอักษร`
    },

  email: (): ValidationRule =>
    (value) => {
      if (!value) return true
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value) || 'รูปแบบอีเมลไม่ถูกต้อง'
    },

  pattern: (regex: RegExp, message: string): ValidationRule =>
    (value) => {
      if (!value) return true
      return regex.test(value) || message
    },

  match: (otherField: () => any, message = 'ค่าไม่ตรงกัน'): ValidationRule =>
    (value) => value === otherField() || message,
}

// useForm composable
function useForm<T extends Record<string, any>>(
  initialValues: T,
  fieldRules: Partial<Record<keyof T, FieldRules>>
) {
  const fields = reactive<Record<string, FormField>>(
    Object.fromEntries(
      Object.entries(initialValues).map(([key, value]) => [
        key,
        {
          value,
          rules: fieldRules[key as keyof T] || [],
          touched: false,
          dirty: false,
        } as FormField,
      ])
    )
  )

  function validate(fieldName: string): string | null {
    const field = fields[fieldName]
    for (const rule of field.rules) {
      const result = rule(field.value)
      if (result !== true) return result as string
    }
    return null
  }

  const errors = computed(() => {
    const result: Record<string, string | null> = {}
    Object.keys(fields).forEach(key => {
      result[key] = validate(key)
    })
    return result
  })

  const isValid = computed(() =>
    Object.values(errors.value).every(e => e === null)
  )

  function touchField(name: string) {
    fields[name].touched = true
  }

  function touchAll() {
    Object.keys(fields).forEach(name => {
      fields[name].touched = true
    })
  }

  function getFieldError(name: string): string | undefined {
    return fields[name].touched ? errors.value[name] || undefined : undefined
  }

  function reset() {
    Object.entries(initialValues).forEach(([key, value]) => {
      fields[key].value = value
      fields[key].touched = false
      fields[key].dirty = false
    })
  }

  function getValues(): T {
    return Object.fromEntries(
      Object.entries(fields).map(([key, field]) => [key, field.value])
    ) as T
  }

  return {
    fields,
    errors,
    isValid,
    touchField,
    touchAll,
    getFieldError,
    reset,
    getValues,
  }
}

// ใช้งาน useForm
const { fields, isValid, touchField, touchAll, getFieldError, reset, getValues } = useForm(
  {
    username: '',
    email: '',
    password: '',
    confirmPassword: '',
    phone: '',
  },
  {
    username: [
      rules.required(),
      rules.minLength(3),
      rules.maxLength(20),
      rules.pattern(/^[a-zA-Z0-9_]+$/, 'ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _'),
    ],
    email: [rules.required(), rules.email()],
    password: [
      rules.required(),
      rules.minLength(8),
      rules.pattern(/[A-Z]/, 'ต้องมีตัวพิมพ์ใหญ่'),
      rules.pattern(/[0-9]/, 'ต้องมีตัวเลข'),
    ],
    confirmPassword: [
      rules.required(),
      rules.match(() => fields.password.value, 'รหัสผ่านไม่ตรงกัน'),
    ],
    phone: [
      rules.pattern(/^(0[689]\d{8}|0[2-9]\d{7})?$/, 'รูปแบบเบอร์ไม่ถูกต้อง'),
    ],
  }
)

const isSubmitting = ref(false)
const submitResult = ref<string | null>(null)

async function handleSubmit() {
  touchAll()
  if (!isValid.value) return

  isSubmitting.value = true
  try {
    await new Promise(resolve => setTimeout(resolve, 1500))
    submitResult.value = 'success'
    reset()
  } catch (error) {
    submitResult.value = 'error'
  } finally {
    isSubmitting.value = false
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit" class="form">
    <div class="form-group" :class="{ error: getFieldError('username') }">
      <label>ชื่อผู้ใช้</label>
      <input
        v-model="fields.username.value"
        @blur="touchField('username')"
        placeholder="กรอกชื่อผู้ใช้"
      />
      <span class="error-msg">{{ getFieldError('username') }}</span>
    </div>

    <div class="form-group" :class="{ error: getFieldError('email') }">
      <label>อีเมล</label>
      <input
        v-model="fields.email.value"
        type="email"
        @blur="touchField('email')"
        placeholder="your@email.com"
      />
      <span class="error-msg">{{ getFieldError('email') }}</span>
    </div>

    <div class="form-group" :class="{ error: getFieldError('password') }">
      <label>รหัสผ่าน</label>
      <input
        v-model="fields.password.value"
        type="password"
        @blur="touchField('password')"
        placeholder="กรอกรหัสผ่าน"
      />
      <span class="error-msg">{{ getFieldError('password') }}</span>
    </div>

    <div class="form-group" :class="{ error: getFieldError('confirmPassword') }">
      <label>ยืนยันรหัสผ่าน</label>
      <input
        v-model="fields.confirmPassword.value"
        type="password"
        @blur="touchField('confirmPassword')"
        placeholder="กรอกรหัสผ่านอีกครั้ง"
      />
      <span class="error-msg">{{ getFieldError('confirmPassword') }}</span>
    </div>

    <button type="submit" :disabled="isSubmitting">
      {{ isSubmitting ? 'กำลังสมัคร...' : 'สมัครสมาชิก' }}
    </button>

    <div v-if="submitResult === 'success'" class="success">
      สมัครสมาชิกสำเร็จ!
    </div>
  </form>
</template>
```

---

## 6. ใช้ VeeValidate + Zod

VeeValidate เป็น library ที่ทรงพลังสำหรับ form validation ใน Vue 3

```bash
npm install vee-validate zod @vee-validate/zod
```

```vue
<script setup lang="ts">
import { useForm, useField } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'

// กำหนด validation schema ด้วย Zod
const schema = toTypedSchema(
  z.object({
    username: z
      .string()
      .min(3, 'ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร')
      .max(20, 'ชื่อผู้ใช้ต้องไม่เกิน 20 ตัวอักษร')
      .regex(/^[a-zA-Z0-9_]+$/, 'ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _'),
    email: z.string().email('รูปแบบอีเมลไม่ถูกต้อง'),
    password: z
      .string()
      .min(8, 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
      .regex(/[A-Z]/, 'ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว')
      .regex(/[0-9]/, 'ต้องมีตัวเลขอย่างน้อย 1 ตัว'),
    confirmPassword: z.string(),
    age: z
      .number()
      .min(13, 'ต้องมีอายุอย่างน้อย 13 ปี')
      .max(120, 'อายุไม่ถูกต้อง'),
    website: z.string().url('URL ไม่ถูกต้อง').optional().or(z.literal('')),
  }).refine(data => data.password === data.confirmPassword, {
    message: 'รหัสผ่านไม่ตรงกัน',
    path: ['confirmPassword'],
  })
)

const { handleSubmit, errors, isSubmitting } = useForm({
  validationSchema: schema,
  initialValues: {
    username: '',
    email: '',
    password: '',
    confirmPassword: '',
    age: 18,
    website: '',
  },
})

// useField สำหรับแต่ละ field
const { value: username, errorMessage: usernameError } = useField<string>('username')
const { value: email, errorMessage: emailError } = useField<string>('email')
const { value: password, errorMessage: passwordError } = useField<string>('password')
const { value: confirmPassword, errorMessage: confirmPasswordError } = useField<string>('confirmPassword')
const { value: age, errorMessage: ageError } = useField<number>('age')

const onSubmit = handleSubmit(async (values) => {
  console.log('Form values:', values)
  await new Promise(resolve => setTimeout(resolve, 1000))
  alert('สมัครสมาชิกสำเร็จ!')
})
</script>

<template>
  <form @submit.prevent="onSubmit" class="vee-validate-form">
    <h2>สมัครสมาชิก (VeeValidate + Zod)</h2>

    <div class="form-group" :class="{ error: usernameError }">
      <label>ชื่อผู้ใช้ *</label>
      <input v-model="username" placeholder="กรอกชื่อผู้ใช้" />
      <span class="error-msg">{{ usernameError }}</span>
    </div>

    <div class="form-group" :class="{ error: emailError }">
      <label>อีเมล *</label>
      <input v-model="email" type="email" placeholder="your@email.com" />
      <span class="error-msg">{{ emailError }}</span>
    </div>

    <div class="form-group" :class="{ error: passwordError }">
      <label>รหัสผ่าน *</label>
      <input v-model="password" type="password" placeholder="กรอกรหัสผ่าน" />
      <span class="error-msg">{{ passwordError }}</span>
    </div>

    <div class="form-group" :class="{ error: confirmPasswordError }">
      <label>ยืนยันรหัสผ่าน *</label>
      <input v-model="confirmPassword" type="password" placeholder="ยืนยันรหัสผ่าน" />
      <span class="error-msg">{{ confirmPasswordError }}</span>
    </div>

    <div class="form-group" :class="{ error: ageError }">
      <label>อายุ *</label>
      <input v-model.number="age" type="number" placeholder="กรอกอายุ" />
      <span class="error-msg">{{ ageError }}</span>
    </div>

    <button type="submit" :disabled="isSubmitting" class="submit-btn">
      {{ isSubmitting ? 'กำลังประมวลผล...' : 'สมัครสมาชิก' }}
    </button>
  </form>
</template>
```

---

## 7. ตัวอย่าง: Registration Form สมบูรณ์

```vue
<script setup lang="ts">
import { ref, reactive, computed, watch } from 'vue'

interface RegistrationFormData {
  // ข้อมูลส่วนตัว
  firstName: string
  lastName: string
  email: string
  phone: string
  birthDate: string
  gender: string

  // ที่อยู่
  address: string
  city: string
  province: string
  postalCode: string

  // บัญชี
  username: string
  password: string
  confirmPassword: string

  // การตั้งค่า
  interests: string[]
  newsletter: boolean
  agreeToTerms: boolean
}

type FormErrors = Partial<Record<keyof RegistrationFormData, string>>

const provinces = [
  'กรุงเทพมหานคร', 'เชียงใหม่', 'ขอนแก่น', 'ชลบุรี', 'สงขลา',
  'นนทบุรี', 'ปทุมธานี', 'สมุทรปราการ', 'นครราชสีมา', 'อื่นๆ'
]

const interestOptions = [
  'เทคโนโลยี', 'การเดินทาง', 'ดนตรี', 'ภาพยนตร์', 'กีฬา',
  'อาหาร', 'แฟชั่น', 'ศิลปะ', 'หนังสือ', 'สุขภาพ'
]

const form = reactive<RegistrationFormData>({
  firstName: '',
  lastName: '',
  email: '',
  phone: '',
  birthDate: '',
  gender: '',
  address: '',
  city: '',
  province: '',
  postalCode: '',
  username: '',
  password: '',
  confirmPassword: '',
  interests: [],
  newsletter: false,
  agreeToTerms: false,
})

const touched = reactive<Record<string, boolean>>({})
const currentStep = ref(1)
const totalSteps = 3
const isSubmitting = ref(false)
const isSubmitted = ref(false)

// Validation
const validationRules: Record<string, (val: any) => string | null> = {
  firstName: (v) => !v ? 'กรุณากรอกชื่อ' : null,
  lastName: (v) => !v ? 'กรุณากรอกนามสกุล' : null,
  email: (v) => {
    if (!v) return 'กรุณากรอกอีเมล'
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v)) return 'รูปแบบอีเมลไม่ถูกต้อง'
    return null
  },
  phone: (v) => {
    if (!v) return null // optional
    if (!/^0[689]\d{8}$/.test(v)) return 'รูปแบบเบอร์ไม่ถูกต้อง'
    return null
  },
  birthDate: (v) => {
    if (!v) return 'กรุณาเลือกวันเกิด'
    const age = (Date.now() - new Date(v).getTime()) / (365.25 * 24 * 60 * 60 * 1000)
    if (age < 13) return 'ต้องมีอายุอย่างน้อย 13 ปี'
    return null
  },
  gender: (v) => !v ? 'กรุณาเลือกเพศ' : null,
  username: (v) => {
    if (!v) return 'กรุณากรอกชื่อผู้ใช้'
    if (v.length < 3) return 'ต้องมีอย่างน้อย 3 ตัวอักษร'
    if (!/^[a-zA-Z0-9_]+$/.test(v)) return 'ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _'
    return null
  },
  password: (v) => {
    if (!v) return 'กรุณากรอกรหัสผ่าน'
    if (v.length < 8) return 'ต้องมีอย่างน้อย 8 ตัวอักษร'
    if (!/[A-Z]/.test(v)) return 'ต้องมีตัวพิมพ์ใหญ่'
    if (!/[0-9]/.test(v)) return 'ต้องมีตัวเลข'
    return null
  },
  confirmPassword: (v) => v !== form.password ? 'รหัสผ่านไม่ตรงกัน' : null,
  agreeToTerms: (v) => !v ? 'ต้องยอมรับเงื่อนไข' : null,
}

const errors = computed<FormErrors>(() => {
  const result: FormErrors = {}
  ;(Object.keys(validationRules) as Array<keyof RegistrationFormData>).forEach(key => {
    const error = validationRules[key]((form as any)[key])
    if (error) result[key] = error
  })
  return result
})

const stepFields: Record<number, (keyof RegistrationFormData)[]> = {
  1: ['firstName', 'lastName', 'email', 'phone', 'birthDate', 'gender'],
  2: ['address', 'city', 'province', 'postalCode'],
  3: ['username', 'password', 'confirmPassword', 'agreeToTerms'],
}

const stepErrors = computed(() => {
  const result: Record<number, number> = {}
  for (let step = 1; step <= totalSteps; step++) {
    const fields = stepFields[step]
    result[step] = fields.filter(field => errors.value[field]).length
  }
  return result
})

const currentStepValid = computed(() => {
  const fields = stepFields[currentStep.value]
  return fields.every(field => !errors.value[field])
})

function touchField(field: string) {
  touched[field] = true
}

function touchStep(step: number) {
  stepFields[step].forEach(field => { touched[field] = true })
}

function getError(field: keyof RegistrationFormData): string | undefined {
  return touched[field] ? errors.value[field] : undefined
}

function nextStep() {
  touchStep(currentStep.value)
  if (currentStepValid.value) {
    currentStep.value++
  }
}

function prevStep() {
  currentStep.value--
}

async function handleSubmit() {
  // Touch all fields
  for (let step = 1; step <= totalSteps; step++) {
    touchStep(step)
  }

  if (Object.keys(errors.value).length > 0) return

  isSubmitting.value = true
  try {
    await new Promise(resolve => setTimeout(resolve, 2000))
    isSubmitted.value = true
  } finally {
    isSubmitting.value = false
  }
}

const passwordStrength = computed(() => {
  const p = form.password
  if (!p) return 0
  let strength = 0
  if (p.length >= 8) strength += 20
  if (p.length >= 12) strength += 10
  if (/[A-Z]/.test(p)) strength += 20
  if (/[a-z]/.test(p)) strength += 20
  if (/[0-9]/.test(p)) strength += 20
  if (/[^a-zA-Z0-9]/.test(p)) strength += 10
  return Math.min(100, strength)
})

const passwordStrengthLabel = computed(() => {
  if (passwordStrength.value <= 30) return { label: 'อ่อน', color: '#ef4444' }
  if (passwordStrength.value <= 60) return { label: 'ปานกลาง', color: '#f59e0b' }
  if (passwordStrength.value <= 80) return { label: 'ดี', color: '#3b82f6' }
  return { label: 'แข็งแรงมาก', color: '#10b981' }
})
</script>

<template>
  <div class="registration-container">
    <div v-if="isSubmitted" class="success-screen">
      <div class="success-icon">✓</div>
      <h2>สมัครสมาชิกสำเร็จ!</h2>
      <p>ยืนยัน email ที่ {{ form.email }} เพื่อเริ่มใช้งาน</p>
    </div>

    <form v-else @submit.prevent="handleSubmit" class="registration-form">
      <div class="form-header">
        <h2>สมัครสมาชิก</h2>
        <!-- Step indicator -->
        <div class="steps">
          <div
            v-for="step in totalSteps"
            :key="step"
            class="step"
            :class="{
              active: step === currentStep,
              completed: step < currentStep,
              'has-errors': stepErrors[step] > 0
            }"
          >
            <div class="step-number">{{ step }}</div>
            <div class="step-label">
              {{ step === 1 ? 'ข้อมูลส่วนตัว' : step === 2 ? 'ที่อยู่' : 'บัญชี' }}
            </div>
          </div>
        </div>
      </div>

      <!-- Step 1: Personal Info -->
      <div v-show="currentStep === 1" class="step-content">
        <h3>ข้อมูลส่วนตัว</h3>
        <div class="form-row">
          <div class="form-group" :class="{ error: getError('firstName') }">
            <label>ชื่อ *</label>
            <input v-model="form.firstName" @blur="touchField('firstName')" placeholder="ชื่อ" />
            <span class="error-msg">{{ getError('firstName') }}</span>
          </div>
          <div class="form-group" :class="{ error: getError('lastName') }">
            <label>นามสกุล *</label>
            <input v-model="form.lastName" @blur="touchField('lastName')" placeholder="นามสกุล" />
            <span class="error-msg">{{ getError('lastName') }}</span>
          </div>
        </div>

        <div class="form-group" :class="{ error: getError('email') }">
          <label>อีเมล *</label>
          <input v-model="form.email" type="email" @blur="touchField('email')" placeholder="your@email.com" />
          <span class="error-msg">{{ getError('email') }}</span>
        </div>

        <div class="form-group" :class="{ error: getError('birthDate') }">
          <label>วันเกิด *</label>
          <input v-model="form.birthDate" type="date" @blur="touchField('birthDate')" />
          <span class="error-msg">{{ getError('birthDate') }}</span>
        </div>

        <div class="form-group" :class="{ error: getError('gender') }">
          <label>เพศ *</label>
          <div class="radio-group">
            <label><input type="radio" v-model="form.gender" value="male" /> ชาย</label>
            <label><input type="radio" v-model="form.gender" value="female" /> หญิง</label>
            <label><input type="radio" v-model="form.gender" value="other" /> อื่นๆ</label>
          </div>
          <span class="error-msg">{{ getError('gender') }}</span>
        </div>

        <div class="form-group">
          <label>ความสนใจ</label>
          <div class="checkbox-grid">
            <label v-for="interest in interestOptions" :key="interest">
              <input type="checkbox" v-model="form.interests" :value="interest" />
              {{ interest }}
            </label>
          </div>
        </div>
      </div>

      <!-- Step 3: Account -->
      <div v-show="currentStep === 3" class="step-content">
        <h3>ข้อมูลบัญชี</h3>

        <div class="form-group" :class="{ error: getError('username') }">
          <label>ชื่อผู้ใช้ *</label>
          <input v-model="form.username" @blur="touchField('username')" placeholder="ชื่อผู้ใช้" />
          <span class="error-msg">{{ getError('username') }}</span>
        </div>

        <div class="form-group" :class="{ error: getError('password') }">
          <label>รหัสผ่าน *</label>
          <input v-model="form.password" type="password" @blur="touchField('password')" placeholder="รหัสผ่าน" />
          <div v-if="form.password" class="password-strength-bar">
            <div class="bar" :style="{ width: `${passwordStrength}%`, backgroundColor: passwordStrengthLabel.color }"></div>
            <span :style="{ color: passwordStrengthLabel.color }">{{ passwordStrengthLabel.label }}</span>
          </div>
          <span class="error-msg">{{ getError('password') }}</span>
        </div>

        <div class="form-group" :class="{ error: getError('confirmPassword') }">
          <label>ยืนยันรหัสผ่าน *</label>
          <input v-model="form.confirmPassword" type="password" @blur="touchField('confirmPassword')" placeholder="ยืนยันรหัสผ่าน" />
          <span class="error-msg">{{ getError('confirmPassword') }}</span>
        </div>

        <div class="form-group">
          <label>
            <input type="checkbox" v-model="form.newsletter" />
            รับข่าวสารและโปรโมชั่น
          </label>
        </div>

        <div class="form-group" :class="{ error: getError('agreeToTerms') }">
          <label>
            <input type="checkbox" v-model="form.agreeToTerms" @change="touchField('agreeToTerms')" />
            ฉันยอมรับ <a href="#" @click.prevent>เงื่อนไขการใช้บริการ</a> และ <a href="#" @click.prevent>นโยบายความเป็นส่วนตัว</a>
          </label>
          <span class="error-msg">{{ getError('agreeToTerms') }}</span>
        </div>
      </div>

      <!-- Navigation buttons -->
      <div class="form-actions">
        <button
          v-if="currentStep > 1"
          type="button"
          @click="prevStep"
          class="btn-secondary"
        >
          ← ก่อนหน้า
        </button>

        <button
          v-if="currentStep < totalSteps"
          type="button"
          @click="nextStep"
          class="btn-primary"
        >
          ถัดไป →
        </button>

        <button
          v-if="currentStep === totalSteps"
          type="submit"
          :disabled="isSubmitting"
          class="btn-primary"
        >
          {{ isSubmitting ? 'กำลังสมัคร...' : 'สมัครสมาชิก' }}
        </button>
      </div>
    </form>
  </div>
</template>

<style scoped>
.registration-container { max-width: 600px; margin: 0 auto; padding: 2rem; }
.steps { display: flex; gap: 1rem; margin: 1rem 0; }
.step { display: flex; flex-direction: column; align-items: center; flex: 1; }
.step-number { width: 32px; height: 32px; border-radius: 50%; border: 2px solid #ddd; display: flex; align-items: center; justify-content: center; }
.step.active .step-number { border-color: #3b82f6; background: #3b82f6; color: white; }
.step.completed .step-number { background: #10b981; border-color: #10b981; color: white; }
.step.has-errors .step-number { border-color: #ef4444; }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
.form-group { margin-bottom: 1rem; }
.form-group label { display: block; margin-bottom: 0.25rem; font-weight: 500; }
.form-group input, .form-group select, .form-group textarea {
  width: 100%; padding: 0.5rem 0.75rem; border: 1px solid #ddd; border-radius: 4px;
}
.form-group.error input { border-color: #ef4444; }
.error-msg { color: #ef4444; font-size: 0.875rem; }
.form-actions { display: flex; justify-content: space-between; margin-top: 1.5rem; }
.btn-primary { padding: 0.75rem 1.5rem; background: #3b82f6; color: white; border: none; border-radius: 4px; cursor: pointer; }
.btn-secondary { padding: 0.75rem 1.5rem; background: #f3f4f6; border: none; border-radius: 4px; cursor: pointer; }
.password-strength-bar { height: 4px; background: #e5e7eb; border-radius: 2px; margin-top: 0.25rem; }
.bar { height: 100%; border-radius: 2px; transition: width 0.3s; }
.success-screen { text-align: center; padding: 3rem; }
.success-icon { font-size: 4rem; color: #10b981; }
</style>
```

---

## สรุป

| Feature | การใช้งาน |
|---------|----------|
| `v-model` | Two-way binding กับ form elements |
| `.lazy` | อัปเดตเมื่อ change แทน input |
| `.number` | แปลงค่าเป็น number อัตโนมัติ |
| `.trim` | ตัด whitespace อัตโนมัติ |
| Custom v-model | รับ `modelValue` prop + emit `update:modelValue` |
| Multiple v-model | ใช้ `v-model:propName` syntax |

**Best Practices:**
- ใช้ `.number` modifier กับ number inputs เสมอ
- ใช้ `.trim` modifier กับ text inputs ที่ไม่ต้องการ leading/trailing spaces
- สร้าง reusable form components ที่รองรับ custom v-model
- ใช้ library เช่น VeeValidate + Zod สำหรับ complex validation
- แสดง error messages เฉพาะ fields ที่ user ได้ touch แล้ว (touched state)
