# Part 5: Event Handling

## บทนำ

Event Handling ใน Vue.js ทำให้เราสามารถตอบสนองต่อการกระทำของผู้ใช้ได้อย่างง่ายดาย ตั้งแต่การคลิก ปุ่ม ไปจนถึงการ drag-and-drop และ keyboard shortcuts Vue มี directive `v-on` (shorthand: `@`) พร้อมด้วย event modifiers ที่ทรงพลังสำหรับจัดการ events ต่างๆ

---

## 1. v-on และ @ shorthand

`v-on` ใช้สำหรับ listen to DOM events และเรียก JavaScript code เมื่อ event ถูก trigger

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const lastEvent = ref<MouseEvent | null>(null)
const inputValue = ref('')

function increment() {
  count.value++
}

function handleClick(event: MouseEvent) {
  lastEvent.value = event
  console.log('คลิกที่:', event.clientX, event.clientY)
}

function handleInput(event: Event) {
  const target = event.target as HTMLInputElement
  inputValue.value = target.value
}
</script>

<template>
  <div class="event-demo">
    <!-- v-on syntax -->
    <button v-on:click="count++">v-on click ({{ count }})</button>

    <!-- @ shorthand (นิยมใช้มากกว่า) -->
    <button @click="count++">@ shorthand ({{ count }})</button>

    <!-- เรียก method -->
    <button @click="increment">Method call</button>

    <!-- inline expression -->
    <button @click="count = 0">Reset</button>

    <!-- รับ event object -->
    <button @click="handleClick">Click with event</button>
    <p v-if="lastEvent">
      คลิกที่: {{ lastEvent.clientX }}, {{ lastEvent.clientY }}
    </p>

    <!-- ส่ง argument พร้อม event -->
    <button @click="(e) => handleClick(e)">Arrow function</button>

    <!-- หลาย events -->
    <input
      @input="handleInput"
      @focus="console.log('focused')"
      @blur="console.log('blurred')"
      placeholder="พิมพ์อะไรก็ได้"
    />
    <p>ค่า: {{ inputValue }}</p>

    <!-- Event บน custom component -->
    <MyComponent @custom-event="handleCustom" />
  </div>
</template>
```

---

## 2. Event Modifiers

Vue มี event modifiers ที่ช่วยให้เราจัดการ event behavior ได้โดยไม่ต้องเขียน code เพิ่มใน handlers

### .stop - หยุด event propagation

```vue
<template>
  <div @click="handleOuterClick" class="outer">
    Outer div
    <!-- .stop จะเรียก event.stopPropagation() -->
    <button @click.stop="handleInnerClick">Inner button (.stop)</button>
    <!-- โดยไม่มี .stop จะ propagate ไปยัง outer div ด้วย -->
    <button @click="handleInnerClick">Inner button (no .stop)</button>
  </div>
</template>

<script setup lang="ts">
function handleOuterClick() { console.log('Outer clicked!') }
function handleInnerClick() { console.log('Inner clicked!') }
</script>
```

### .prevent - ป้องกัน default behavior

```vue
<template>
  <!-- ป้องกัน form submit ที่จะ reload หน้า -->
  <form @submit.prevent="handleSubmit">
    <input v-model="email" type="email" placeholder="Email" />
    <button type="submit">Submit</button>
  </form>

  <!-- ป้องกัน link navigation -->
  <a href="https://example.com" @click.prevent="handleLinkClick">
    Click me (prevented)
  </a>

  <!-- ป้องกัน right-click menu -->
  <div @contextmenu.prevent="showCustomMenu">
    Right click me
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const email = ref('')

function handleSubmit() {
  console.log('Form submitted with:', email.value)
  // จัดการ form submission แบบ AJAX แทน
}

function handleLinkClick() {
  console.log('Link clicked but navigation prevented')
}

function showCustomMenu(event: MouseEvent) {
  console.log('Custom context menu at:', event.clientX, event.clientY)
}
</script>
```

### .capture - ใช้ capture mode

```vue
<template>
  <!-- handler จะถูกเรียกในระหว่าง capture phase แทน bubble phase -->
  <div @click.capture="handleCapture" class="parent">
    <button @click="handleBubble">Click me</button>
  </div>
  <!-- ลำดับการเรียก: handleCapture -> handleBubble -->
</template>

<script setup lang="ts">
function handleCapture() { console.log('1. Capture phase') }
function handleBubble() { console.log('2. Bubble phase') }
</script>
```

### .self - เรียกเฉพาะเมื่อ event target คือ element นั้น

```vue
<template>
  <!-- จะเรียกเฉพาะเมื่อคลิกที่ outer div โดยตรง ไม่ใช่ child elements -->
  <div @click.self="handleSelfClick" class="container">
    <p>คลิกที่ paragraph นี้จะไม่ trigger</p>
    <button>คลิกที่ button นี้ก็จะไม่ trigger</button>
    <span>คลิกตรงนี้ (พื้นที่ว่าง) จะ trigger</span>
  </div>
</template>
```

### .once - เรียกแค่ครั้งเดียว

```vue
<template>
  <!-- handler จะถูกเรียกแค่ครั้งแรกเท่านั้น -->
  <button @click.once="handleOnce">
    คลิกได้ครั้งเดียว
  </button>

  <!-- ใช้กับ custom events บน child components ได้ด้วย -->
  <VideoPlayer @play.once="handleFirstPlay" />
</template>

<script setup lang="ts">
function handleOnce() {
  console.log('จะถูกเรียกแค่ครั้งเดียว')
}

function handleFirstPlay() {
  console.log('เล่นวิดีโอครั้งแรก - อาจส่ง analytics')
}
</script>
```

### .passive - ปรับปรุง scroll performance

```vue
<template>
  <!-- .passive บอก browser ว่า handler จะไม่เรียก preventDefault()
       ทำให้ browser optimize scrolling ได้ดีขึ้น -->
  <div @scroll.passive="handleScroll" class="scroll-container">
    <div v-for="i in 100" :key="i">Item {{ i }}</div>
  </div>

  <!-- ใช้กับ touch events เพื่อ smooth scrolling บน mobile -->
  <div
    @touchstart.passive="handleTouchStart"
    @touchmove.passive="handleTouchMove"
    class="touch-area"
  >
    Touch and scroll here
  </div>
</template>

<script setup lang="ts">
const handleScroll = (e: Event) => {
  // ห้ามเรียก e.preventDefault() ใน passive handler
  const target = e.target as HTMLElement
  console.log('Scrolled to:', target.scrollTop)
}

const handleTouchStart = (e: TouchEvent) => {
  console.log('Touch start at:', e.touches[0].clientX, e.touches[0].clientY)
}

const handleTouchMove = (e: TouchEvent) => {
  console.log('Touch moving...')
}
</script>
```

### การ chain modifiers

```vue
<template>
  <!-- สามารถ chain modifiers ได้ -->
  <a href="#" @click.stop.prevent="handleClick">
    Stop propagation AND prevent default
  </a>

  <!-- click.once.prevent -->
  <button @click.once.prevent="handleOnce">
    Once and prevent (ครั้งแรกเท่านั้น)
  </button>
</template>
```

---

## 3. Key Modifiers

Vue มี key modifiers สำหรับ keyboard events

```vue
<script setup lang="ts">
import { ref } from 'vue'

const searchQuery = ref('')
const messages = ref<string[]>([])
const currentMessage = ref('')

function search() {
  console.log('ค้นหา:', searchQuery.value)
}

function sendMessage() {
  if (currentMessage.value.trim()) {
    messages.value.push(currentMessage.value)
    currentMessage.value = ''
  }
}

function handleEscape() {
  searchQuery.value = ''
  console.log('ล้างการค้นหา')
}
</script>

<template>
  <div class="keyboard-demo">
    <!-- เรียกเมื่อกด Enter -->
    <input
      v-model="searchQuery"
      @keyup.enter="search"
      placeholder="ค้นหา... (กด Enter)"
    />

    <!-- เรียกเมื่อกด Escape -->
    <input
      v-model="searchQuery"
      @keyup.esc="handleEscape"
      placeholder="กด Escape เพื่อล้าง"
    />

    <!-- Key aliases ที่ Vue รองรับ -->
    <!-- .enter, .tab, .delete, .esc, .space, .up, .down, .left, .right -->

    <!-- ส่งข้อความด้วย Enter (แต่ไม่ใช่ Shift+Enter) -->
    <textarea
      v-model="currentMessage"
      @keydown.enter.exact="sendMessage"
      @keydown.enter.shift="() => currentMessage += '\n'"
      placeholder="Enter เพื่อส่ง, Shift+Enter เพื่อขึ้นบรรทัดใหม่"
    />

    <!-- Key combinations -->
    <input
      @keydown.ctrl.s.prevent="saveDocument"
      @keydown.meta.s.prevent="saveDocument"
      placeholder="Ctrl/Cmd+S เพื่อบันทึก"
    />

    <!-- ใช้ key code โดยตรง -->
    <input
      @keyup.f1="showHelp"
      @keyup.page-up="scrollUp"
      @keyup.page-down="scrollDown"
      placeholder="กด F1, PageUp, PageDown"
    />
  </div>
</template>

<script setup lang="ts">
// ... (เพิ่มเติมจากด้านบน)
function saveDocument() { console.log('บันทึกเอกสาร') }
function showHelp() { console.log('แสดงความช่วยเหลือ') }
function scrollUp() { window.scrollBy(0, -200) }
function scrollDown() { window.scrollBy(0, 200) }
</script>
```

---

## 4. Mouse Button Modifiers

```vue
<template>
  <div class="mouse-demo">
    <!-- .left - ปุ่มซ้าย (default) -->
    <div @click.left="handleLeft">Left click</div>

    <!-- .right - ปุ่มขวา -->
    <div @click.right.prevent="handleRight">Right click (custom menu)</div>

    <!-- .middle - ปุ่มกลาง -->
    <a href="#" @click.middle.prevent="openInNewTab">
      Middle click to open in new tab
    </a>

    <!-- Modifier keys กับ mouse events -->
    <div
      @click.ctrl="handleCtrlClick"
      @click.shift="handleShiftClick"
      @click.alt="handleAltClick"
      @click.exact="handleExactClick"
    >
      Click with modifier keys
    </div>
  </div>
</template>

<script setup lang="ts">
function handleLeft(e: MouseEvent) { console.log('Left click') }
function handleRight(e: MouseEvent) { console.log('Right click at:', e.clientX, e.clientY) }
function openInNewTab() { console.log('Open in new tab') }
function handleCtrlClick() { console.log('Ctrl+Click - อาจเปิดใน new tab') }
function handleShiftClick() { console.log('Shift+Click - อาจ select range') }
function handleAltClick() { console.log('Alt+Click') }
function handleExactClick() { console.log('Click only (ไม่มี modifier keys)') }
</script>
```

---

## 5. การส่ง Custom Events

Child component สามารถ emit events ไปยัง parent ได้

```vue
<!-- ChildComponent.vue -->
<script setup lang="ts">
// defineEmits สำหรับประกาศ events
const emit = defineEmits<{
  click: [value: string]
  'update:modelValue': [value: string]
  submit: [data: { name: string; email: string }]
  close: []
}>()

function handleButtonClick() {
  emit('click', 'button was clicked')
}

function handleSubmit(name: string, email: string) {
  emit('submit', { name, email })
}
</script>

<template>
  <div>
    <button @click="handleButtonClick">Click me</button>
    <button @click="emit('close')">Close</button>
  </div>
</template>
```

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import ChildComponent from './ChildComponent.vue'

function handleChildClick(value: string) {
  console.log('Child emitted:', value)
}

function handleSubmit(data: { name: string; email: string }) {
  console.log('Form submitted:', data)
}
</script>

<template>
  <ChildComponent
    @click="handleChildClick"
    @submit="handleSubmit"
    @close="showModal = false"
  />
</template>
```

---

## 6. Event Arguments

```vue
<script setup lang="ts">
import { ref } from 'vue'

const items = ref(['Apple', 'Banana', 'Cherry'])

// ส่ง argument ไปกับ event
function handleItemClick(item: string, index: number, event: MouseEvent) {
  console.log(`คลิก: ${item} (index: ${index})`)
  console.log('Event:', event)
}

// ใช้ $event เพื่อส่ง native event ใน inline handler
function handleWithEvent(item: string, event: MouseEvent) {
  event.stopPropagation()
  console.log(`${item} clicked at ${event.clientX}, ${event.clientY}`)
}
</script>

<template>
  <ul>
    <li
      v-for="(item, index) in items"
      :key="item"
      @click="handleItemClick(item, index, $event)"
    >
      {{ item }}
    </li>
  </ul>

  <!-- Arrow function syntax -->
  <ul>
    <li
      v-for="(item, index) in items"
      :key="item"
      @click="(e) => handleItemClick(item, index, e)"
    >
      {{ item }}
    </li>
  </ul>
</template>
```

---

## 7. Multiple Handlers

```vue
<script setup lang="ts">
import { ref } from 'vue'

const logs = ref<string[]>([])

function logClick() {
  logs.value.push(`Clicked at ${new Date().toLocaleTimeString()}`)
}

function validateClick() {
  console.log('Validating click...')
}

function trackAnalytics(eventName: string) {
  console.log(`Analytics: ${eventName}`)
}
</script>

<template>
  <!-- เรียก handlers หลายอันด้วย comma -->
  <button @click="logClick(), validateClick()">
    Multiple handlers
  </button>

  <!-- หรือใช้ arrow function -->
  <button @click="() => { logClick(); validateClick(); trackAnalytics('button_click') }">
    Arrow function with multiple calls
  </button>

  <!-- ใช้ method ที่เรียกทั้งหมด -->
  <button @click="handleClick">
    Single method (แนะนำ)
  </button>
</template>

<script setup lang="ts">
// ... (เพิ่มเติม)
function handleClick(event: MouseEvent) {
  logClick()
  validateClick()
  trackAnalytics('button_click')
}
</script>
```

---

## 8. ตัวอย่าง Real-world: Keyboard Shortcuts System

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

interface ShortcutAction {
  key: string
  ctrl?: boolean
  shift?: boolean
  alt?: boolean
  meta?: boolean
  description: string
  action: () => void
}

// Registry ของ keyboard shortcuts
const shortcuts: ShortcutAction[] = [
  {
    key: 's',
    ctrl: true,
    description: 'บันทึกเอกสาร',
    action: () => saveDocument(),
  },
  {
    key: 'z',
    ctrl: true,
    description: 'Undo',
    action: () => undo(),
  },
  {
    key: 'z',
    ctrl: true,
    shift: true,
    description: 'Redo',
    action: () => redo(),
  },
  {
    key: 'f',
    ctrl: true,
    description: 'ค้นหา',
    action: () => openSearch(),
  },
  {
    key: '/',
    description: 'เปิด Command Palette',
    action: () => openCommandPalette(),
  },
  {
    key: 'Escape',
    description: 'ปิด Dialog/Modal',
    action: () => closeDialog(),
  },
]

const isShortcutsHelpOpen = ref(false)
const isCommandPaletteOpen = ref(false)
const isSearchOpen = ref(false)
const statusMessage = ref('')
const history = ref<string[]>([])
const future = ref<string[]>([])
const currentContent = ref('เนื้อหาเริ่มต้น')

// Global keyboard event handler
function handleKeyDown(event: KeyboardEvent) {
  const matchingShortcut = shortcuts.find(shortcut => {
    const keyMatch = event.key === shortcut.key ||
                     event.key.toLowerCase() === shortcut.key.toLowerCase()
    const ctrlMatch = !!shortcut.ctrl === (event.ctrlKey || event.metaKey)
    const shiftMatch = !!shortcut.shift === event.shiftKey
    const altMatch = !!shortcut.alt === event.altKey

    return keyMatch && ctrlMatch && shiftMatch && altMatch
  })

  if (matchingShortcut) {
    event.preventDefault()
    matchingShortcut.action()
  }

  // แสดง shortcuts help เมื่อกด ?
  if (event.key === '?' && !event.ctrlKey) {
    isShortcutsHelpOpen.value = !isShortcutsHelpOpen.value
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
})

function saveDocument() {
  statusMessage.value = 'บันทึกเอกสารแล้ว'
  setTimeout(() => { statusMessage.value = '' }, 2000)
}

function undo() {
  if (history.value.length > 0) {
    const lastState = history.value.pop()!
    future.value.push(currentContent.value)
    currentContent.value = lastState
    statusMessage.value = 'Undo แล้ว'
  }
}

function redo() {
  if (future.value.length > 0) {
    const nextState = future.value.pop()!
    history.value.push(currentContent.value)
    currentContent.value = nextState
    statusMessage.value = 'Redo แล้ว'
  }
}

function openSearch() {
  isSearchOpen.value = true
  statusMessage.value = 'เปิดการค้นหา'
}

function openCommandPalette() {
  isCommandPaletteOpen.value = true
}

function closeDialog() {
  isCommandPaletteOpen.value = false
  isSearchOpen.value = false
  isShortcutsHelpOpen.value = false
}

function formatShortcut(shortcut: ShortcutAction): string {
  const parts: string[] = []
  if (shortcut.ctrl) parts.push('Ctrl')
  if (shortcut.shift) parts.push('Shift')
  if (shortcut.alt) parts.push('Alt')
  parts.push(shortcut.key.toUpperCase())
  return parts.join('+')
}
</script>

<template>
  <div class="app">
    <!-- Status bar -->
    <div class="status-bar" v-if="statusMessage">
      {{ statusMessage }}
    </div>

    <!-- Editor -->
    <textarea
      v-model="currentContent"
      @input="() => { history.push(currentContent); future = [] }"
      class="editor"
      placeholder="พิมพ์เนื้อหา..."
    />

    <!-- Shortcuts help overlay -->
    <div
      v-if="isShortcutsHelpOpen"
      class="overlay"
      @click.self="isShortcutsHelpOpen = false"
    >
      <div class="shortcuts-help">
        <h3>Keyboard Shortcuts</h3>
        <button @click="isShortcutsHelpOpen = false" class="close-btn">✕</button>
        <table>
          <thead>
            <tr>
              <th>Shortcut</th>
              <th>คำอธิบาย</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="shortcut in shortcuts" :key="shortcut.key">
              <td><kbd>{{ formatShortcut(shortcut) }}</kbd></td>
              <td>{{ shortcut.description }}</td>
            </tr>
            <tr>
              <td><kbd>?</kbd></td>
              <td>แสดง/ซ่อน Shortcuts</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Command Palette -->
    <div
      v-if="isCommandPaletteOpen"
      class="overlay"
      @click.self="closeDialog"
      @keydown.esc="closeDialog"
    >
      <div class="command-palette">
        <input
          autofocus
          placeholder="พิมพ์คำสั่ง..."
          @keydown.esc.stop="closeDialog"
        />
        <ul class="commands">
          <li v-for="shortcut in shortcuts" :key="shortcut.description"
            @click="() => { shortcut.action(); closeDialog() }"
          >
            {{ shortcut.description }}
            <kbd>{{ formatShortcut(shortcut) }}</kbd>
          </li>
        </ul>
      </div>
    </div>

    <div class="help-hint">กด <kbd>?</kbd> เพื่อดู shortcuts</div>
  </div>
</template>
```

---

## 9. ตัวอย่าง: Drag and Drop

```vue
<script setup lang="ts">
import { ref, reactive } from 'vue'

interface DragItem {
  id: number
  content: string
  column: 'todo' | 'doing' | 'done'
}

const columns = ref<{ id: string; title: string; items: DragItem[] }[]>([
  {
    id: 'todo',
    title: 'ต้องทำ',
    items: [
      { id: 1, content: 'เรียน Vue.js', column: 'todo' },
      { id: 2, content: 'สร้าง Portfolio', column: 'todo' },
      { id: 3, content: 'เขียน Resume', column: 'todo' },
    ],
  },
  {
    id: 'doing',
    title: 'กำลังทำ',
    items: [
      { id: 4, content: 'ออกกำลังกาย', column: 'doing' },
    ],
  },
  {
    id: 'done',
    title: 'เสร็จแล้ว',
    items: [
      { id: 5, content: 'กินข้าวเช้า', column: 'done' },
    ],
  },
])

const draggingItem = ref<DragItem | null>(null)
const dragOverColumn = ref<string | null>(null)
const dragOverItem = ref<number | null>(null)

function handleDragStart(item: DragItem, event: DragEvent) {
  draggingItem.value = item
  if (event.dataTransfer) {
    event.dataTransfer.effectAllowed = 'move'
    event.dataTransfer.setData('text/plain', String(item.id))
  }
}

function handleDragEnd() {
  draggingItem.value = null
  dragOverColumn.value = null
  dragOverItem.value = null
}

function handleDragOver(columnId: string, event: DragEvent) {
  event.preventDefault()
  if (event.dataTransfer) {
    event.dataTransfer.dropEffect = 'move'
  }
  dragOverColumn.value = columnId
}

function handleDragEnterItem(itemId: number) {
  dragOverItem.value = itemId
}

function handleDrop(targetColumnId: string, event: DragEvent, targetItemId?: number) {
  event.preventDefault()

  if (!draggingItem.value) return

  const sourceColumnIndex = columns.value.findIndex(col =>
    col.items.some(item => item.id === draggingItem.value!.id)
  )
  const targetColumnIndex = columns.value.findIndex(col => col.id === targetColumnId)

  if (sourceColumnIndex === -1 || targetColumnIndex === -1) return

  // ลบ item จาก source column
  const sourceColumn = columns.value[sourceColumnIndex]
  const itemIndex = sourceColumn.items.findIndex(item => item.id === draggingItem.value!.id)
  const [movedItem] = sourceColumn.items.splice(itemIndex, 1)

  // อัปเดต column ของ item
  movedItem.column = targetColumnId as DragItem['column']

  // เพิ่ม item ใน target column
  const targetColumn = columns.value[targetColumnIndex]
  if (targetItemId !== undefined) {
    const targetIndex = targetColumn.items.findIndex(item => item.id === targetItemId)
    targetColumn.items.splice(targetIndex, 0, movedItem)
  } else {
    targetColumn.items.push(movedItem)
  }

  draggingItem.value = null
  dragOverColumn.value = null
  dragOverItem.value = null
}

function addItem(columnId: string) {
  const column = columns.value.find(col => col.id === columnId)
  if (!column) return

  const newId = Math.max(...columns.value.flatMap(c => c.items.map(i => i.id))) + 1
  const content = prompt('ใส่รายการใหม่:')
  if (content) {
    column.items.push({
      id: newId,
      content,
      column: columnId as DragItem['column'],
    })
  }
}
</script>

<template>
  <div class="kanban-board">
    <div
      v-for="column in columns"
      :key="column.id"
      class="kanban-column"
      :class="{ 'drag-over': dragOverColumn === column.id }"
      @dragover="handleDragOver(column.id, $event)"
      @drop="handleDrop(column.id, $event)"
    >
      <div class="column-header">
        <h3>{{ column.title }}</h3>
        <span class="count">{{ column.items.length }}</span>
      </div>

      <div class="items-container">
        <div
          v-for="item in column.items"
          :key="item.id"
          class="kanban-item"
          :class="{
            'dragging': draggingItem?.id === item.id,
            'drag-over-item': dragOverItem === item.id
          }"
          draggable="true"
          @dragstart="handleDragStart(item, $event)"
          @dragend="handleDragEnd"
          @dragenter="handleDragEnterItem(item.id)"
          @drop.stop="handleDrop(column.id, $event, item.id)"
        >
          {{ item.content }}
        </div>
      </div>

      <button @click="addItem(column.id)" class="add-btn">
        + เพิ่มรายการ
      </button>
    </div>
  </div>
</template>

<style scoped>
.kanban-board {
  display: flex;
  gap: 1rem;
  padding: 1rem;
  overflow-x: auto;
}
.kanban-column {
  flex: 1;
  min-width: 250px;
  background: #f5f5f5;
  border-radius: 8px;
  padding: 1rem;
  transition: background 0.2s;
}
.kanban-column.drag-over {
  background: #e3f2fd;
  border: 2px dashed #1976d2;
}
.column-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}
.kanban-item {
  background: white;
  border-radius: 4px;
  padding: 0.75rem;
  margin-bottom: 0.5rem;
  cursor: grab;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  transition: all 0.2s;
}
.kanban-item:active { cursor: grabbing; }
.kanban-item.dragging { opacity: 0.3; }
.kanban-item.drag-over-item { border-top: 2px solid #1976d2; }
.add-btn { width: 100%; padding: 0.5rem; border: none; background: transparent; cursor: pointer; color: #666; }
.add-btn:hover { background: rgba(0,0,0,0.05); border-radius: 4px; }
</style>
```

---

## สรุป

| Modifier | การใช้งาน |
|----------|-----------|
| `.stop` | หยุด event propagation |
| `.prevent` | ป้องกัน default behavior |
| `.capture` | ใช้ capture phase |
| `.self` | เรียกเฉพาะเมื่อ target คือ element นั้น |
| `.once` | เรียกแค่ครั้งเดียว |
| `.passive` | บอก browser ว่าจะไม่ preventDefault |
| `.enter`, `.esc` | Key modifiers |
| `.left`, `.right`, `.middle` | Mouse button modifiers |
| `.ctrl`, `.shift`, `.alt` | Modifier key combinations |

**Best Practices:**
- ใช้ event modifiers แทนการเขียน `event.preventDefault()` ใน handler
- ใช้ `@click.prevent` แทน `handleClick(event) { event.preventDefault() }`
- ระวังการใช้ `.sync` กับ watchers - อาจทำให้ performance แย่
- เสมอ `removeEventListener` ใน `onUnmounted` สำหรับ global event listeners
- ใช้ `.passive` กับ scroll และ touch events เพื่อ performance ที่ดีกว่า
