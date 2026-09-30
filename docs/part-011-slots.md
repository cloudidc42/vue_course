# Part 11: Slots และ Named Slots

## บทนำ

Slots เป็นกลไกสำคัญใน Vue.js ที่ช่วยให้เราสามารถส่ง template content จาก parent component ไปยัง child component ได้ ทำให้ component มีความยืดหยุ่นและนำกลับมาใช้ได้หลายครั้ง

---

## 1. Default Slot

Default Slot คือ slot พื้นฐานที่สุด ใช้เมื่อต้องการส่ง content เข้าไปใน component โดยไม่ต้องระบุชื่อ

### การสร้าง Component ที่ใช้ Default Slot

```vue
<!-- components/BaseCard.vue -->
<template>
  <div class="card">
    <div class="card-body">
      <!-- slot tag คือ placeholder สำหรับ content ที่จะถูกส่งมา -->
      <slot></slot>
    </div>
  </div>
</template>

<style scoped>
.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 16px;
  margin: 8px 0;
}
</style>
```

### การใช้งาน Default Slot

```vue
<!-- App.vue -->
<template>
  <div>
    <!-- ส่ง content เข้าไปใน BaseCard -->
    <BaseCard>
      <h2>หัวข้อบทความ</h2>
      <p>นี่คือเนื้อหาของบทความที่จะแสดงใน card</p>
      <button>อ่านเพิ่มเติม</button>
    </BaseCard>

    <!-- card ที่ว่างเปล่า - ใช้ fallback content -->
    <BaseCard></BaseCard>
  </div>
</template>

<script setup>
import BaseCard from './components/BaseCard.vue'
</script>
```

### Fallback Content (Default Content)

เราสามารถกำหนด content ที่จะแสดงเมื่อไม่มีการส่ง content มาได้

```vue
<!-- components/BaseCard.vue -->
<template>
  <div class="card">
    <div class="card-body">
      <slot>
        <!-- นี่คือ fallback content -->
        <p class="empty-message">ไม่มีเนื้อหา</p>
      </slot>
    </div>
  </div>
</template>
```

---

## 2. Named Slots

Named Slots ใช้เมื่อต้องการส่ง content หลายส่วนเข้าไปใน component ในตำแหน่งที่ต่างกัน

### การสร้าง Component ที่ใช้ Named Slots

```vue
<!-- components/PageLayout.vue -->
<template>
  <div class="layout">
    <!-- header slot -->
    <header class="header">
      <slot name="header">
        <h1>Default Header</h1>
      </slot>
    </header>

    <!-- main content area -->
    <main class="main">
      <!-- default slot สำหรับ main content -->
      <slot></slot>
    </main>

    <!-- sidebar slot -->
    <aside class="sidebar">
      <slot name="sidebar">
        <p>No sidebar content</p>
      </slot>
    </aside>

    <!-- footer slot -->
    <footer class="footer">
      <slot name="footer">
        <p>&copy; 2024 My App</p>
      </slot>
    </footer>
  </div>
</template>

<style scoped>
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "main sidebar"
    "footer footer";
  grid-template-columns: 1fr 300px;
  gap: 16px;
}

.header { grid-area: header; }
.main { grid-area: main; }
.sidebar { grid-area: sidebar; }
.footer { grid-area: footer; }
</style>
```

### การใช้งาน Named Slots ด้วย v-slot

```vue
<!-- App.vue -->
<template>
  <PageLayout>
    <!-- กำหนด content สำหรับ header slot -->
    <template v-slot:header>
      <h1>ชื่อเว็บไซต์ของฉัน</h1>
      <nav>
        <a href="/">หน้าแรก</a>
        <a href="/about">เกี่ยวกับ</a>
        <a href="/contact">ติดต่อ</a>
      </nav>
    </template>

    <!-- default slot - ไม่ต้องระบุชื่อ -->
    <article>
      <h2>บทความหลัก</h2>
      <p>เนื้อหาบทความ...</p>
    </article>

    <!-- sidebar slot -->
    <template v-slot:sidebar>
      <h3>บทความล่าสุด</h3>
      <ul>
        <li>บทความ 1</li>
        <li>บทความ 2</li>
        <li>บทความ 3</li>
      </ul>
    </template>

    <!-- footer slot - ใช้ shorthand # -->
    <template #footer>
      <p>ติดต่อ: contact@example.com</p>
      <p>&copy; 2024 เว็บไซต์ของฉัน</p>
    </template>
  </PageLayout>
</template>

<script setup>
import PageLayout from './components/PageLayout.vue'
</script>
```

---

## 3. Scoped Slots

Scoped Slots ช่วยให้ child component สามารถส่งข้อมูลกลับมาให้ parent ใช้ใน slot content ได้

### ตัวอย่างพื้นฐาน

```vue
<!-- components/DataList.vue -->
<template>
  <div class="data-list">
    <ul>
      <li v-for="item in items" :key="item.id">
        <!-- ส่งข้อมูล item กลับไปให้ parent ผ่าน slot props -->
        <slot :item="item" :index="items.indexOf(item)">
          <!-- fallback: แสดงแบบ default -->
          {{ item.name }}
        </slot>
      </li>
    </ul>
  </div>
</template>

<script setup>
defineProps({
  items: {
    type: Array,
    required: true
  }
})
</script>
```

### การใช้งาน Scoped Slots

```vue
<!-- App.vue -->
<template>
  <div>
    <!-- รับ slot props ด้วย v-slot -->
    <DataList :items="products">
      <template v-slot:default="{ item, index }">
        <div class="product-item">
          <span class="index">{{ index + 1 }}.</span>
          <strong>{{ item.name }}</strong>
          <span class="price">฿{{ item.price }}</span>
          <button @click="addToCart(item)">เพิ่มในตะกร้า</button>
        </div>
      </template>
    </DataList>

    <!-- ใช้ shorthand -->
    <DataList :items="users" v-slot="{ item }">
      <div class="user-item">
        <img :src="item.avatar" :alt="item.name" />
        <span>{{ item.name }}</span>
        <span class="email">{{ item.email }}</span>
      </div>
    </DataList>
  </div>
</template>

<script setup>
import DataList from './components/DataList.vue'
import { ref } from 'vue'

const products = ref([
  { id: 1, name: 'สินค้า A', price: 100 },
  { id: 2, name: 'สินค้า B', price: 250 },
  { id: 3, name: 'สินค้า C', price: 500 }
])

const users = ref([
  { id: 1, name: 'สมชาย', email: 'somchai@example.com', avatar: '/avatar1.jpg' },
  { id: 2, name: 'สมหญิง', email: 'somying@example.com', avatar: '/avatar2.jpg' }
])

function addToCart(item) {
  console.log('Added to cart:', item.name)
}
</script>
```

---

## 4. Dynamic Slot Names

เราสามารถกำหนดชื่อ slot แบบ dynamic ได้ เพื่อเพิ่มความยืดหยุ่น

```vue
<!-- components/DynamicLayout.vue -->
<template>
  <div class="layout">
    <div v-for="section in sections" :key="section.name" :class="section.class">
      <slot :name="section.name">
        <div class="placeholder">{{ section.placeholder }}</div>
      </slot>
    </div>
  </div>
</template>

<script setup>
defineProps({
  sections: {
    type: Array,
    default: () => [
      { name: 'top', class: 'top-section', placeholder: 'Top Content' },
      { name: 'middle', class: 'middle-section', placeholder: 'Middle Content' },
      { name: 'bottom', class: 'bottom-section', placeholder: 'Bottom Content' }
    ]
  }
})
</script>
```

### การใช้ Dynamic Slot Name ใน Parent

```vue
<!-- App.vue -->
<template>
  <div>
    <DynamicLayout>
      <!-- ใช้ dynamic slot name ด้วย [] -->
      <template v-for="slot in activeSlots" :key="slot.name" #[slot.name]>
        <div class="custom-content">
          <h3>{{ slot.title }}</h3>
          <p>{{ slot.content }}</p>
        </div>
      </template>
    </DynamicLayout>
  </div>
</template>

<script setup>
import DynamicLayout from './components/DynamicLayout.vue'
import { ref } from 'vue'

const activeSlots = ref([
  { name: 'top', title: 'ส่วนบน', content: 'เนื้อหาส่วนบน' },
  { name: 'middle', title: 'ส่วนกลาง', content: 'เนื้อหาส่วนกลาง' }
])
</script>
```

---

## 5. Slots ใน Composition API

ใน Composition API เราสามารถเข้าถึง slots ได้ผ่าน `useSlots()`

```vue
<!-- components/ConditionalSlot.vue -->
<template>
  <div class="container">
    <!-- แสดง header slot เฉพาะเมื่อมี content -->
    <header v-if="hasHeader" class="header">
      <slot name="header"></slot>
    </header>

    <main class="content">
      <slot></slot>
    </main>

    <!-- แสดง footer slot เฉพาะเมื่อมี content -->
    <footer v-if="hasFooter" class="footer">
      <slot name="footer"></slot>
    </footer>
  </div>
</template>

<script setup>
import { useSlots, computed } from 'vue'

// useSlots() คืน object ที่มี slot functions
const slots = useSlots()

// ตรวจสอบว่ามี slot content หรือไม่
const hasHeader = computed(() => !!slots.header)
const hasFooter = computed(() => !!slots.footer)
</script>
```

### การใช้งาน useSlots() แบบ Advanced

```vue
<!-- components/SmartContainer.vue -->
<template>
  <div :class="containerClass">
    <slot></slot>
  </div>
</template>

<script setup>
import { useSlots, computed } from 'vue'

const slots = useSlots()

// ตรวจสอบจำนวน children ใน default slot
const childCount = computed(() => {
  const defaultSlot = slots.default?.()
  if (!defaultSlot) return 0
  return defaultSlot.length
})

// เปลี่ยน class ตามจำนวน children
const containerClass = computed(() => ({
  'container': true,
  'container--empty': childCount.value === 0,
  'container--single': childCount.value === 1,
  'container--multiple': childCount.value > 1
}))
</script>
```

---

## 6. Renderless Components Pattern

Renderless Components เป็น pattern ที่ component ไม่มี template ของตัวเอง แต่ให้ logic ผ่าน scoped slots

```vue
<!-- components/FetchData.vue - Renderless Component -->
<template>
  <!-- ส่ง data และ state กลับผ่าน scoped slot -->
  <slot
    :data="data"
    :loading="loading"
    :error="error"
    :refetch="fetchData"
  ></slot>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'

const props = defineProps({
  url: {
    type: String,
    required: true
  },
  immediate: {
    type: Boolean,
    default: true
  }
})

const data = ref(null)
const loading = ref(false)
const error = ref(null)

async function fetchData() {
  loading.value = true
  error.value = null
  try {
    const response = await fetch(props.url)
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    data.value = await response.json()
  } catch (err) {
    error.value = err.message
  } finally {
    loading.value = false
  }
}

// refetch เมื่อ url เปลี่ยน
watch(() => props.url, fetchData)

if (props.immediate) {
  onMounted(fetchData)
}
</script>
```

### การใช้งาน Renderless Component

```vue
<!-- App.vue -->
<template>
  <div>
    <!-- ใช้ FetchData แบบ renderless -->
    <FetchData url="https://api.example.com/users" v-slot="{ data, loading, error, refetch }">
      <!-- แสดง loading state -->
      <div v-if="loading" class="loading">
        กำลังโหลด...
      </div>

      <!-- แสดง error state -->
      <div v-else-if="error" class="error">
        เกิดข้อผิดพลาด: {{ error }}
        <button @click="refetch">ลองอีกครั้ง</button>
      </div>

      <!-- แสดงข้อมูล -->
      <div v-else-if="data" class="user-list">
        <h2>รายชื่อผู้ใช้ ({{ data.length }} คน)</h2>
        <UserCard
          v-for="user in data"
          :key="user.id"
          :user="user"
        />
      </div>
    </FetchData>
  </div>
</template>

<script setup>
import FetchData from './components/FetchData.vue'
import UserCard from './components/UserCard.vue'
</script>
```

---

## 7. ตัวอย่างจริง: Modal Component

```vue
<!-- components/Modal.vue -->
<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="modelValue" class="modal-overlay" @click.self="closeModal">
        <div :class="['modal', `modal--${size}`]">
          <!-- Header Slot -->
          <div class="modal-header">
            <slot name="header">
              <h3 class="modal-title">Modal</h3>
            </slot>
            <button class="close-btn" @click="closeModal" aria-label="ปิด">✕</button>
          </div>

          <!-- Body Slot (Default) -->
          <div class="modal-body">
            <slot></slot>
          </div>

          <!-- Footer Slot -->
          <div v-if="hasFooter" class="modal-footer">
            <slot name="footer">
              <button class="btn btn-secondary" @click="closeModal">ยกเลิก</button>
              <button class="btn btn-primary" @click="$emit('confirm')">ยืนยัน</button>
            </slot>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { computed, useSlots } from 'vue'

const props = defineProps({
  modelValue: Boolean,
  size: {
    type: String,
    default: 'medium',
    validator: (val) => ['small', 'medium', 'large', 'fullscreen'].includes(val)
  }
})

const emit = defineEmits(['update:modelValue', 'confirm'])
const slots = useSlots()

const hasFooter = computed(() => !!slots.footer)

function closeModal() {
  emit('update:modelValue', false)
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: white;
  border-radius: 8px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  overflow: hidden;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
}

.modal--small { width: 300px; }
.modal--medium { width: 500px; }
.modal--large { width: 800px; }
.modal--fullscreen { width: 95vw; height: 95vh; }

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 24px;
  border-bottom: 1px solid #eee;
}

.modal-body {
  padding: 24px;
  overflow-y: auto;
  flex: 1;
}

.modal-footer {
  padding: 16px 24px;
  border-top: 1px solid #eee;
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

.modal-enter-active, .modal-leave-active {
  transition: opacity 0.3s;
}
.modal-enter-from, .modal-leave-to {
  opacity: 0;
}
</style>
```

### การใช้งาน Modal Component

```vue
<!-- App.vue -->
<template>
  <div>
    <button @click="showModal = true">เปิด Modal</button>
    <button @click="showDeleteModal = true">ลบข้อมูล</button>

    <!-- Modal พื้นฐาน -->
    <Modal v-model="showModal" size="medium" @confirm="handleConfirm">
      <template #header>
        <h3>แก้ไขข้อมูลผู้ใช้</h3>
      </template>

      <!-- Default slot - form content -->
      <form @submit.prevent>
        <div class="form-group">
          <label>ชื่อ</label>
          <input v-model="userData.name" type="text" />
        </div>
        <div class="form-group">
          <label>อีเมล</label>
          <input v-model="userData.email" type="email" />
        </div>
      </form>

      <template #footer>
        <button @click="showModal = false">ยกเลิก</button>
        <button @click="handleSave" class="btn-primary">บันทึก</button>
      </template>
    </Modal>

    <!-- Delete Confirmation Modal -->
    <Modal v-model="showDeleteModal" size="small" @confirm="handleDelete">
      <template #header>
        <h3 style="color: red">⚠️ ยืนยันการลบ</h3>
      </template>
      <p>คุณต้องการลบข้อมูลนี้ใช่หรือไม่? การกระทำนี้ไม่สามารถย้อนกลับได้</p>
    </Modal>
  </div>
</template>

<script setup>
import Modal from './components/Modal.vue'
import { ref } from 'vue'

const showModal = ref(false)
const showDeleteModal = ref(false)
const userData = ref({ name: 'สมชาย', email: 'somchai@example.com' })

function handleSave() {
  console.log('Saving:', userData.value)
  showModal.value = false
}

function handleDelete() {
  console.log('Deleting...')
  showDeleteModal.value = false
}

function handleConfirm() {
  console.log('Confirmed')
}
</script>
```

---

## 8. ตัวอย่างจริง: Accordion Component

```vue
<!-- components/Accordion.vue -->
<template>
  <div class="accordion">
    <slot></slot>
  </div>
</template>

<script setup>
import { provide, ref } from 'vue'

const props = defineProps({
  multiple: {
    type: Boolean,
    default: false
  }
})

const activeItems = ref([])

function toggleItem(id) {
  if (props.multiple) {
    const index = activeItems.value.indexOf(id)
    if (index > -1) {
      activeItems.value.splice(index, 1)
    } else {
      activeItems.value.push(id)
    }
  } else {
    activeItems.value = activeItems.value[0] === id ? [] : [id]
  }
}

function isActive(id) {
  return activeItems.value.includes(id)
}

// provide ให้ AccordionItem ใช้
provide('accordion', { toggleItem, isActive })
</script>

<style scoped>
.accordion {
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
}
</style>
```

```vue
<!-- components/AccordionItem.vue -->
<template>
  <div class="accordion-item">
    <div class="accordion-header" @click="toggle">
      <slot name="header">
        <span>Click to expand</span>
      </slot>
      <span class="icon" :class="{ 'icon--open': isOpen }">▼</span>
    </div>

    <Transition name="accordion">
      <div v-if="isOpen" class="accordion-body">
        <slot></slot>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { inject, computed } from 'vue'

const props = defineProps({
  id: {
    type: [String, Number],
    required: true
  }
})

const { toggleItem, isActive } = inject('accordion')

const isOpen = computed(() => isActive(props.id))

function toggle() {
  toggleItem(props.id)
}
</script>

<style scoped>
.accordion-item {
  border-bottom: 1px solid #ddd;
}
.accordion-item:last-child {
  border-bottom: none;
}

.accordion-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
  cursor: pointer;
  background: #f9f9f9;
  user-select: none;
}

.accordion-header:hover {
  background: #f0f0f0;
}

.accordion-body {
  padding: 16px;
}

.icon {
  transition: transform 0.3s;
}
.icon--open {
  transform: rotate(180deg);
}

.accordion-enter-active,
.accordion-leave-active {
  transition: all 0.3s ease;
  overflow: hidden;
}
.accordion-enter-from,
.accordion-leave-to {
  height: 0;
  padding-top: 0;
  padding-bottom: 0;
  opacity: 0;
}
</style>
```

### การใช้ Accordion

```vue
<!-- App.vue -->
<template>
  <div class="container">
    <h2>คำถามที่พบบ่อย</h2>

    <!-- Single mode - เปิดได้ทีละอัน -->
    <Accordion>
      <AccordionItem id="1">
        <template #header>
          <strong>Vue.js คืออะไร?</strong>
        </template>
        <p>Vue.js เป็น JavaScript framework สำหรับสร้าง user interfaces ที่ใช้งานง่ายและมีประสิทธิภาพสูง</p>
      </AccordionItem>

      <AccordionItem id="2">
        <template #header>
          <strong>ความแตกต่างระหว่าง Vue 2 และ Vue 3?</strong>
        </template>
        <div>
          <p>Vue 3 มีการปรับปรุงหลายอย่าง:</p>
          <ul>
            <li>Composition API ใหม่</li>
            <li>ประสิทธิภาพดีขึ้น</li>
            <li>TypeScript support ดีกว่า</li>
            <li>Teleport และ Fragments</li>
          </ul>
        </div>
      </AccordionItem>

      <AccordionItem id="3">
        <template #header>
          <strong>เรียน Vue.js ยากไหม?</strong>
        </template>
        <p>Vue.js มี learning curve ที่ไม่สูงมาก เหมาะสำหรับผู้เริ่มต้นที่มีพื้นฐาน HTML, CSS, JavaScript</p>
      </AccordionItem>
    </Accordion>

    <!-- Multiple mode - เปิดได้หลายอัน -->
    <h3 class="mt-4">Multiple Mode</h3>
    <Accordion :multiple="true">
      <AccordionItem id="a">
        <template #header>ข้อที่ 1</template>
        <p>เนื้อหาข้อที่ 1</p>
      </AccordionItem>
      <AccordionItem id="b">
        <template #header>ข้อที่ 2</template>
        <p>เนื้อหาข้อที่ 2</p>
      </AccordionItem>
    </Accordion>
  </div>
</template>

<script setup>
import Accordion from './components/Accordion.vue'
import AccordionItem from './components/AccordionItem.vue'
</script>
```

---

## 9. ตัวอย่างจริง: Tabs Component

```vue
<!-- components/Tabs.vue -->
<template>
  <div class="tabs">
    <!-- Tab Headers -->
    <div class="tab-headers" role="tablist">
      <button
        v-for="tab in tabs"
        :key="tab.id"
        :class="['tab-btn', { 'tab-btn--active': activeTab === tab.id }]"
        role="tab"
        :aria-selected="activeTab === tab.id"
        @click="setActiveTab(tab.id)"
      >
        <span v-if="tab.icon" class="tab-icon">{{ tab.icon }}</span>
        {{ tab.label }}
        <span v-if="tab.badge" class="tab-badge">{{ tab.badge }}</span>
      </button>
    </div>

    <!-- Tab Content -->
    <div class="tab-content">
      <slot :activeTab="activeTab"></slot>
    </div>
  </div>
</template>

<script setup>
import { ref, provide, onMounted } from 'vue'

const props = defineProps({
  defaultTab: String
})

const emit = defineEmits(['change'])

const tabs = ref([])
const activeTab = ref(props.defaultTab || null)

function registerTab(tab) {
  tabs.value.push(tab)
  if (!activeTab.value) {
    activeTab.value = tab.id
  }
}

function setActiveTab(id) {
  activeTab.value = id
  emit('change', id)
}

// provide ให้ TabPanel ใช้
provide('tabs', { activeTab, registerTab })
</script>

<style scoped>
.tabs {
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
}

.tab-headers {
  display: flex;
  background: #f5f5f5;
  border-bottom: 1px solid #ddd;
}

.tab-btn {
  padding: 12px 20px;
  border: none;
  background: transparent;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  color: #666;
  transition: all 0.2s;
}

.tab-btn:hover {
  background: #eee;
  color: #333;
}

.tab-btn--active {
  background: white;
  color: #4CAF50;
  border-bottom: 2px solid #4CAF50;
  margin-bottom: -1px;
  font-weight: 600;
}

.tab-badge {
  background: #ff5722;
  color: white;
  border-radius: 10px;
  padding: 2px 6px;
  font-size: 11px;
}

.tab-content {
  padding: 20px;
}
</style>
```

```vue
<!-- components/TabPanel.vue -->
<template>
  <div v-show="isActive" role="tabpanel">
    <slot></slot>
  </div>
</template>

<script setup>
import { inject, computed, onMounted } from 'vue'

const props = defineProps({
  id: {
    type: String,
    required: true
  },
  label: {
    type: String,
    required: true
  },
  icon: String,
  badge: [String, Number]
})

const { activeTab, registerTab } = inject('tabs')

const isActive = computed(() => activeTab.value === props.id)

onMounted(() => {
  registerTab({
    id: props.id,
    label: props.label,
    icon: props.icon,
    badge: props.badge
  })
})
</script>
```

### การใช้ Tabs Component

```vue
<!-- App.vue -->
<template>
  <div class="container">
    <h2>ข้อมูลสินค้า</h2>

    <Tabs default-tab="description" @change="onTabChange">
      <TabPanel id="description" label="รายละเอียด" icon="📋">
        <h3>รายละเอียดสินค้า</h3>
        <p>นี่คือรายละเอียดสินค้าที่ครบถ้วน...</p>
        <ul>
          <li>คุณสมบัติ 1</li>
          <li>คุณสมบัติ 2</li>
          <li>คุณสมบัติ 3</li>
        </ul>
      </TabPanel>

      <TabPanel id="specs" label="สเปค" icon="⚙️">
        <table class="specs-table">
          <tr><td>น้ำหนัก</td><td>500g</td></tr>
          <tr><td>ขนาด</td><td>20 x 15 x 5 cm</td></tr>
          <tr><td>วัสดุ</td><td>อลูมิเนียม</td></tr>
        </table>
      </TabPanel>

      <TabPanel id="reviews" label="รีวิว" icon="⭐" badge="24">
        <div class="reviews">
          <div class="review" v-for="review in reviews" :key="review.id">
            <strong>{{ review.author }}</strong>
            <span>{{ '⭐'.repeat(review.rating) }}</span>
            <p>{{ review.comment }}</p>
          </div>
        </div>
      </TabPanel>

      <TabPanel id="shipping" label="จัดส่ง" icon="🚚">
        <h4>ข้อมูลการจัดส่ง</h4>
        <p>จัดส่งภายใน 3-5 วันทำการ</p>
        <p>ฟรีค่าจัดส่งสำหรับคำสั่งซื้อมากกว่า 500 บาท</p>
      </TabPanel>
    </Tabs>
  </div>
</template>

<script setup>
import Tabs from './components/Tabs.vue'
import TabPanel from './components/TabPanel.vue'
import { ref } from 'vue'

const reviews = ref([
  { id: 1, author: 'สมชาย', rating: 5, comment: 'สินค้าดีมาก คุณภาพเยี่ยม' },
  { id: 2, author: 'สมหญิง', rating: 4, comment: 'ดีครับ แต่จัดส่งช้าหน่อย' }
])

function onTabChange(tabId) {
  console.log('Tab changed to:', tabId)
}
</script>
```

---

## สรุป

| Pattern | เมื่อไหร่ควรใช้ |
|---------|----------------|
| Default Slot | ส่ง content เดียวเข้า component |
| Named Slots | ส่ง content หลายส่วน ในตำแหน่งต่างกัน |
| Scoped Slots | child ต้องการส่งข้อมูลกลับมาให้ parent ใช้ใน slot |
| Renderless Components | แยก logic ออกจาก presentation |

### คำสั่งสำคัญ

```vue
<!-- ประกาศ slot ใน child -->
<slot name="header">Fallback Content</slot>

<!-- ใช้งานใน parent -->
<template v-slot:header>Content</template>
<template #header>Content</template>  <!-- shorthand -->

<!-- Scoped slot - ส่งข้อมูลกลับ -->
<slot :data="someData" :index="i"></slot>

<!-- รับ scoped slot props -->
<template v-slot:default="{ data, index }">
<Component v-slot="{ data, index }">  <!-- shorthand สำหรับ default slot -->
```
