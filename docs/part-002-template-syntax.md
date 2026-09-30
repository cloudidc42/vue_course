# Part 002: Vue.js Template Syntax

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** 2-3 ชั่วโมง | **ขั้นตอน:** 31-60

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Text Interpolation ({{ }})
- Attribute Binding (v-bind)
- Dynamic Binding
- Class และ Style Binding
- HTML Rendering
- JavaScript Expressions ใน Template

---

## ขั้นตอนที่ 31: Text Interpolation

Text Interpolation คือการแสดงผลค่าตัวแปรใน Template ด้วย `{{ }}`

### 31.1 พื้นฐาน

```vue
<script setup>
import { ref } from 'vue'

const name = ref('สมชาย')
const age = ref(25)
const city = ref('กรุงเทพ')
</script>

<template>
  <div>
    <!-- แสดงผลตัวแปร -->
    <p>ชื่อ: {{ name }}</p>
    <p>อายุ: {{ age }}</p>
    <p>เมือง: {{ city }}</p>
    
    <!-- แสดงผล Object -->
    <p>ข้อมูล: {{ { name, age, city } }}</p>
  </div>
</template>
```

### 31.2 JavaScript Expressions

เราสามารถเขียน JavaScript Expression ใน `{{ }}` ได้:

```vue
<script setup>
import { ref } from 'vue'

const price = ref(100)
const quantity = ref(3)
const discount = ref(0.1)
const firstName = ref('สมชาย')
const lastName = ref('ใจดี')
const isAdmin = ref(true)
const items = ref(['แอปเปิล', 'กล้วย', 'ส้ม'])
</script>

<template>
  <div>
    <!-- การคำนวณ -->
    <p>ราคาต่อหน่วย: {{ price }}</p>
    <p>จำนวน: {{ quantity }}</p>
    <p>รวม: {{ price * quantity }}</p>
    <p>ส่วนลด: {{ price * quantity * discount }}</p>
    <p>สุทธิ: {{ price * quantity * (1 - discount) }}</p>
    
    <!-- String methods -->
    <p>ชื่อ-นามสกุล: {{ firstName + ' ' + lastName }}</p>
    <p>ชื่อ (ตัวพิมพ์ใหญ่): {{ firstName.toUpperCase() }}</p>
    <p>ตัวอักษรแรก: {{ firstName.charAt(0) }}</p>
    
    <!-- Ternary operator -->
    <p>สถานะ: {{ isAdmin ? 'ผู้ดูแลระบบ' : 'ผู้ใช้ทั่วไป' }}</p>
    
    <!-- Array methods -->
    <p>จำนวนสินค้า: {{ items.length }}</p>
    <p>สินค้าทั้งหมด: {{ items.join(', ') }}</p>
    <p>สินค้าแรก: {{ items[0] }}</p>
    
    <!-- Template literals -->
    <p>{{ `ยินดีต้อนรับ ${firstName} ${lastName}` }}</p>
  </div>
</template>
```

**ข้อสังเกต:** ใน `{{ }}` เราใส่ได้แค่ Expression เดียว ไม่ใช่ Statement

```vue
<!-- ✅ ถูกต้อง - Expressions -->
{{ count + 1 }}
{{ ok ? 'Yes' : 'No' }}
{{ message.split('').reverse().join('') }}
{{ items.filter(i => i.active) }}

<!-- ❌ ผิด - Statements -->
{{ if (ok) { return message } }}  <!-- if statement -->
{{ let x = 1 }}                   <!-- variable declaration -->
```

---

## ขั้นตอนที่ 32: v-bind - Attribute Binding

`v-bind` ใช้สำหรับผูก (bind) ค่าตัวแปรกับ HTML attributes

### 32.1 การใช้พื้นฐาน

```vue
<script setup>
import { ref } from 'vue'

const imageUrl = ref('https://vuejs.org/images/logo.png')
const imageAlt = ref('Vue Logo')
const linkUrl = ref('https://vuejs.org')
const buttonId = ref('main-button')
const isDisabled = ref(false)
const inputType = ref('text')
const placeholder = ref('กรุณากรอกชื่อ...')
</script>

<template>
  <div>
    <!-- v-bind แบบเต็ม -->
    <img v-bind:src="imageUrl" v-bind:alt="imageAlt" width="100" />
    
    <!-- v-bind แบบย่อ (แนะนำ) -->
    <img :src="imageUrl" :alt="imageAlt" width="100" />
    
    <!-- Link -->
    <a :href="linkUrl" target="_blank">ไปที่ Vue.js</a>
    
    <!-- Form elements -->
    <button :id="buttonId" :disabled="isDisabled">Click Me</button>
    <input :type="inputType" :placeholder="placeholder" />
    
    <!-- Dynamic attribute name (Vue 3) -->
    <div :[attributeName]="value">...</div>
  </div>
</template>
```

### 32.2 Binding หลาย attributes พร้อมกัน

```vue
<script setup>
import { ref, reactive } from 'vue'

// Binding object ทั้งหมดพร้อมกัน
const attrs = reactive({
  id: 'my-input',
  type: 'text',
  placeholder: 'กรอกข้อมูล...',
  class: 'input-field',
  required: true
})
</script>

<template>
  <!-- v-bind ไม่มี argument = bind object ทั้งหมด -->
  <input v-bind="attrs" />
  
  <!-- เทียบเท่ากับ -->
  <input 
    id="my-input"
    type="text"
    placeholder="กรอกข้อมูล..."
    class="input-field"
    required
  />
</template>
```

---

## ขั้นตอนที่ 33: Class Binding

### 33.1 Object Syntax

```vue
<script setup>
import { ref, reactive } from 'vue'

const isActive = ref(true)
const hasError = ref(false)
const isHighlighted = ref(true)

const classObject = reactive({
  active: true,
  'text-danger': false,
  highlighted: true
})
</script>

<template>
  <!-- Object Syntax - แต่ละ key คือ class name, value คือ boolean -->
  <div :class="{ active: isActive, error: hasError, highlight: isHighlighted }">
    หน้าแรก
  </div>
  
  <!-- ใช้ reactive object -->
  <div :class="classObject">
    ข้อความ
  </div>
  
  <!-- ผสม static class กับ dynamic class -->
  <div class="base-style" :class="{ active: isActive, error: hasError }">
    มีทั้ง static และ dynamic class
  </div>
</template>

<style>
.active { color: green; }
.error { color: red; }
.highlight { background: yellow; }
.base-style { padding: 10px; border: 1px solid #ccc; }
</style>
```

### 33.2 Array Syntax

```vue
<script setup>
import { ref, computed } from 'vue'

const activeClass = ref('active')
const errorClass = ref('text-danger')
const isError = ref(false)

const classArray = computed(() => [
  activeClass.value,
  isError.value ? errorClass.value : ''
])
</script>

<template>
  <!-- Array Syntax -->
  <div :class="[activeClass, errorClass]">Array binding</div>
  
  <!-- Conditional class ใน array -->
  <div :class="[isError ? errorClass : '', activeClass]">Conditional</div>
  
  <!-- Nested: ผสม object และ array -->
  <div :class="[{ active: isActive }, errorClass]">Mixed</div>
  
  <!-- ใช้ computed -->
  <div :class="classArray">Computed class</div>
</template>
```

### 33.3 ตัวอย่างจริง: Card Component

```vue
<script setup>
import { ref } from 'vue'

const props = defineProps({
  variant: {
    type: String,
    default: 'default', // 'default' | 'success' | 'warning' | 'error'
    validator: (value) => ['default', 'success', 'warning', 'error'].includes(value)
  },
  size: {
    type: String,
    default: 'medium' // 'small' | 'medium' | 'large'
  },
  elevated: Boolean,
  loading: Boolean
})

const isExpanded = ref(false)
</script>

<template>
  <div 
    :class="[
      'card',
      `card--${props.variant}`,
      `card--${props.size}`,
      {
        'card--elevated': props.elevated,
        'card--loading': props.loading,
        'card--expanded': isExpanded
      }
    ]"
    @click="isExpanded = !isExpanded"
  >
    <slot />
  </div>
</template>

<style scoped>
.card {
  padding: 1rem;
  border-radius: 8px;
  border: 1px solid #ddd;
  transition: all 0.3s;
}

.card--default { background: white; }
.card--success { background: #d4edda; border-color: #28a745; }
.card--warning { background: #fff3cd; border-color: #ffc107; }
.card--error   { background: #f8d7da; border-color: #dc3545; }

.card--small  { padding: 0.5rem; font-size: 0.875rem; }
.card--medium { padding: 1rem; }
.card--large  { padding: 1.5rem; font-size: 1.1rem; }

.card--elevated { box-shadow: 0 4px 12px rgba(0,0,0,0.15); }
.card--loading  { opacity: 0.6; pointer-events: none; }
.card--expanded { max-height: none; }
</style>
```

---

## ขั้นตอนที่ 34: Style Binding

### 34.1 Object Syntax

```vue
<script setup>
import { ref, reactive } from 'vue'

const activeColor = ref('#42b883')
const fontSize = ref(16)
const fontWeight = ref('bold')

const styleObject = reactive({
  color: '#35495e',
  backgroundColor: '#42b88333',
  padding: '10px 20px',
  borderRadius: '8px',
  fontSize: '16px'
})
</script>

<template>
  <!-- Object Syntax - camelCase property names -->
  <div :style="{ color: activeColor, fontSize: fontSize + 'px', fontWeight: fontWeight }">
    Styled Text
  </div>
  
  <!-- ใช้ reactive object -->
  <div :style="styleObject">
    Reactive Style Object
  </div>
  
  <!-- Note: ใช้ CSS property names ได้ทั้ง camelCase และ kebab-case -->
  <div :style="{ 'background-color': '#f0f0f0', fontSize: '14px' }">
    Mixed naming
  </div>
</template>
```

### 34.2 Array Syntax - ผสมหลาย style objects

```vue
<script setup>
import { ref, computed } from 'vue'

const baseStyles = {
  padding: '10px',
  borderRadius: '4px'
}

const themeStyles = ref({
  color: '#333',
  background: '#fff'
})

const conditionalStyles = computed(() => ({
  fontWeight: 'bold',
  fontSize: '18px'
}))
</script>

<template>
  <!-- Array Syntax: รวม styles หลาย objects -->
  <div :style="[baseStyles, themeStyles, conditionalStyles]">
    Multiple style objects
  </div>
</template>
```

### 34.3 ตัวอย่างจริง: Progress Bar

```vue
<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  value: {
    type: Number,
    default: 0,
    validator: (v) => v >= 0 && v <= 100
  },
  color: {
    type: String,
    default: '#42b883'
  }
})

const progressStyle = computed(() => ({
  width: `${props.value}%`,
  backgroundColor: props.color,
  transition: 'width 0.3s ease'
}))

const textColor = computed(() => 
  props.value > 50 ? 'white' : '#333'
)
</script>

<template>
  <div class="progress-container">
    <div class="progress-bar" :style="progressStyle">
      <span :style="{ color: textColor }">{{ props.value }}%</span>
    </div>
  </div>
</template>

<style scoped>
.progress-container {
  width: 100%;
  height: 24px;
  background: #e0e0e0;
  border-radius: 12px;
  overflow: hidden;
}

.progress-bar {
  height: 100%;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 2rem;
}
</style>
```

---

## ขั้นตอนที่ 35: v-html - Render HTML

`v-html` ใช้สำหรับ render HTML string

```vue
<script setup>
import { ref } from 'vue'

const rawHtml = ref('<strong>ข้อความตัวหนา</strong> และ <em>ตัวเอียง</em>')
const userContent = ref('<script>alert("XSS")</script>')  // อันตราย!
</script>

<template>
  <!-- แสดงผลปกติ - HTML จะถูก escape -->
  <p>{{ rawHtml }}</p>
  <!-- Output: <strong>ข้อความตัวหนา</strong> และ <em>ตัวเอียง</em> -->
  
  <!-- v-html - render HTML จริง -->
  <p v-html="rawHtml"></p>
  <!-- Output: ข้อความตัวหนา และ ตัวเอียง (rendered) -->
</template>
```

⚠️ **คำเตือน:** อย่าใช้ `v-html` กับ user input โดยตรง เพราะเสี่ยง XSS Attack!

```vue
<!-- ✅ ปลอดภัย: เนื้อหาที่ developer ควบคุม -->
<p v-html="trustedContent"></p>

<!-- ❌ อันตราย: user input โดยตรง -->
<p v-html="userInputContent"></p>
```

---

## ขั้นตอนที่ 36: ตัวอย่างรวม - Profile Card

มาสร้าง Profile Card ที่ใช้ทุกอย่างที่เรียนมา:

```vue
<script setup>
import { ref, computed } from 'vue'

const user = ref({
  name: 'สมชาย ใจดี',
  username: '@somchai',
  role: 'developer',
  level: 8,
  maxLevel: 10,
  avatar: 'https://api.dicebear.com/7.x/avataaars/svg?seed=Felix',
  bio: 'Full-stack developer ที่รัก <strong>Vue.js</strong> และ <em>Open Source</em>',
  isOnline: true,
  followers: 1234,
  following: 567,
  posts: 89,
  badges: ['🏆', '⭐', '🔥']
})

const isFollowing = ref(false)

const levelPercentage = computed(() => 
  (user.value.level / user.value.maxLevel) * 100
)

const levelColor = computed(() => {
  if (levelPercentage.value >= 80) return '#ffd700'
  if (levelPercentage.value >= 60) return '#42b883'
  if (levelPercentage.value >= 40) return '#3498db'
  return '#95a5a6'
})

const roleStyles = computed(() => ({
  developer: { color: '#42b883', background: '#42b88320' },
  designer: { color: '#e74c3c', background: '#e74c3c20' },
  manager: { color: '#9b59b6', background: '#9b59b620' }
})[user.value.role] || { color: '#95a5a6', background: '#95a5a620' })

function toggleFollow() {
  isFollowing.value = !isFollowing.value
  if (isFollowing.value) {
    user.value.followers++
  } else {
    user.value.followers--
  }
}

function formatNumber(num) {
  if (num >= 1000) return (num / 1000).toFixed(1) + 'k'
  return num.toString()
}
</script>

<template>
  <div class="profile-card">
    <!-- Header Section -->
    <div class="card-header">
      <div class="avatar-wrapper">
        <img 
          :src="user.avatar" 
          :alt="`รูปโปรไฟล์ของ ${user.name}`"
          class="avatar"
        />
        <span 
          class="status-dot"
          :class="{ online: user.isOnline, offline: !user.isOnline }"
        ></span>
      </div>
      
      <div class="user-info">
        <h2 class="name">{{ user.name }}</h2>
        <p class="username">{{ user.username }}</p>
        
        <span class="role-badge" :style="roleStyles">
          {{ user.role }}
        </span>
      </div>
    </div>
    
    <!-- Badges -->
    <div class="badges">
      <span 
        v-for="badge in user.badges" 
        :key="badge"
        class="badge"
        :title="`Badge: ${badge}`"
      >
        {{ badge }}
      </span>
    </div>
    
    <!-- Bio (HTML content) -->
    <div class="bio" v-html="user.bio"></div>
    
    <!-- Level Progress -->
    <div class="level-section">
      <div class="level-header">
        <span>ระดับ {{ user.level }}</span>
        <span>{{ levelPercentage }}%</span>
      </div>
      <div class="progress-bar">
        <div 
          class="progress-fill"
          :style="{ 
            width: levelPercentage + '%',
            backgroundColor: levelColor
          }"
        ></div>
      </div>
    </div>
    
    <!-- Stats -->
    <div class="stats">
      <div class="stat">
        <strong>{{ formatNumber(user.followers) }}</strong>
        <span>ผู้ติดตาม</span>
      </div>
      <div class="stat">
        <strong>{{ formatNumber(user.following) }}</strong>
        <span>กำลังติดตาม</span>
      </div>
      <div class="stat">
        <strong>{{ user.posts }}</strong>
        <span>โพสต์</span>
      </div>
    </div>
    
    <!-- Action Buttons -->
    <div class="actions">
      <button 
        :class="['btn-follow', { following: isFollowing }]"
        @click="toggleFollow"
      >
        {{ isFollowing ? '✓ ติดตามแล้ว' : '+ ติดตาม' }}
      </button>
      <button class="btn-message">ส่งข้อความ</button>
    </div>
  </div>
</template>

<style scoped>
.profile-card {
  max-width: 360px;
  margin: 2rem auto;
  padding: 1.5rem;
  border-radius: 16px;
  background: white;
  box-shadow: 0 4px 24px rgba(0,0,0,0.1);
  font-family: 'Sarabun', sans-serif;
}

.card-header {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1rem;
}

.avatar-wrapper {
  position: relative;
  flex-shrink: 0;
}

.avatar {
  width: 72px;
  height: 72px;
  border-radius: 50%;
  border: 3px solid #42b883;
}

.status-dot {
  position: absolute;
  bottom: 2px;
  right: 2px;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 2px solid white;
}

.status-dot.online  { background: #2ecc71; }
.status-dot.offline { background: #95a5a6; }

.name {
  margin: 0 0 0.25rem;
  font-size: 1.1rem;
  color: #2c3e50;
}

.username {
  margin: 0 0 0.5rem;
  color: #7f8c8d;
  font-size: 0.875rem;
}

.role-badge {
  padding: 2px 10px;
  border-radius: 20px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: capitalize;
}

.badges {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.badge {
  font-size: 1.2rem;
  cursor: pointer;
  transition: transform 0.2s;
}

.badge:hover { transform: scale(1.3); }

.bio {
  color: #555;
  font-size: 0.9rem;
  line-height: 1.6;
  margin-bottom: 1rem;
}

.level-section {
  margin-bottom: 1rem;
}

.level-header {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  color: #555;
  margin-bottom: 0.25rem;
}

.progress-bar {
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.5s ease;
}

.stats {
  display: flex;
  justify-content: space-around;
  padding: 1rem 0;
  border-top: 1px solid #f0f0f0;
  border-bottom: 1px solid #f0f0f0;
  margin-bottom: 1rem;
}

.stat {
  text-align: center;
}

.stat strong {
  display: block;
  font-size: 1.1rem;
  color: #2c3e50;
}

.stat span {
  font-size: 0.75rem;
  color: #7f8c8d;
}

.actions {
  display: flex;
  gap: 0.75rem;
}

.btn-follow, .btn-message {
  flex: 1;
  padding: 0.6rem;
  border: none;
  border-radius: 8px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s;
  font-family: inherit;
}

.btn-follow {
  background: #42b883;
  color: white;
}

.btn-follow.following {
  background: white;
  color: #42b883;
  border: 1px solid #42b883;
}

.btn-follow:hover { opacity: 0.85; }

.btn-message {
  background: #f0f0f0;
  color: #333;
}

.btn-message:hover { background: #e0e0e0; }
</style>
```

---

## ขั้นตอนที่ 37: Template Refs

บางครั้งเราต้องการ access DOM element โดยตรง

```vue
<script setup>
import { ref, onMounted } from 'vue'

const inputRef = ref(null)
const canvasRef = ref(null)
const listRef = ref(null)

onMounted(() => {
  // Focus input เมื่อ Component load
  inputRef.value?.focus()
  
  // ใช้ Canvas API
  const ctx = canvasRef.value?.getContext('2d')
  if (ctx) {
    ctx.fillStyle = '#42b883'
    ctx.fillRect(0, 0, 200, 100)
  }
  
  // ดู scroll height ของ list
  console.log('List height:', listRef.value?.scrollHeight)
})

function clearInput() {
  if (inputRef.value) {
    inputRef.value.value = ''
    inputRef.value.focus()
  }
}
</script>

<template>
  <div>
    <!-- ref attribute ผูกกับ template ref -->
    <input ref="inputRef" type="text" placeholder="กรอกข้อความ..." />
    <button @click="clearInput">Clear</button>
    
    <canvas ref="canvasRef" width="200" height="100"></canvas>
    
    <ul ref="listRef" style="max-height: 200px; overflow-y: auto;">
      <li v-for="i in 20" :key="i">รายการที่ {{ i }}</li>
    </ul>
  </div>
</template>
```

---

## สรุป Part 002

ในบทเรียนนี้คุณได้เรียนรู้:

✅ Text Interpolation ด้วย `{{ }}`  
✅ JavaScript Expressions ใน Template  
✅ `v-bind` สำหรับ Attribute Binding  
✅ Class Binding (Object/Array Syntax)  
✅ Style Binding (Object/Array Syntax)  
✅ `v-html` สำหรับ render HTML  
✅ Template Refs สำหรับ DOM access  

---

## แบบฝึกหัด

1. **ระดับง่าย:** สร้าง Component แสดงข้อมูลผู้ใช้ (ชื่อ, อีเมล, โทรศัพท์)
2. **ระดับกลาง:** สร้าง Badge Component ที่เปลี่ยนสีตาม variant (success/warning/error)
3. **ระดับยาก:** สร้าง Rating Component ที่แสดงดาว 1-5 ดวงพร้อม hover effect

---

**← [Part 001: Setup](./part-001-setup-environment.md)** | **→ [Part 003: Reactivity System](./part-003-reactivity.md)**
