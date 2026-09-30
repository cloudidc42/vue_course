# Part 16: Transitions และ Animations

## บทนำ

Vue.js มี built-in components สำหรับ transitions และ animations ได้แก่ `<Transition>` และ `<TransitionGroup>` ทำให้เพิ่ม animation เข้าสู่แอปได้ง่าย

---

## 1. `<Transition>` Component

`<Transition>` ใช้สำหรับ animate element หรือ component เดียวที่เข้า/ออกจาก DOM

```vue
<template>
  <div>
    <button @click="show = !show">Toggle</button>
    
    <!-- Transition component wrap element ที่ต้องการ animate -->
    <Transition name="fade">
      <p v-if="show">ข้อความนี้จะ fade in/out</p>
    </Transition>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const show = ref(true)
</script>

<style>
/* Enter transition */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

/* Start state สำหรับ enter, End state สำหรับ leave */
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

---

## 2. Transition CSS Classes

Vue จะเพิ่ม/ลบ CSS classes ในแต่ละช่วงของ transition

```
Enter Transition:
┌─────────────────────────────────────────────────┐
│ v-enter-from → v-enter-active → v-enter-to       │
│    (start)       (duration)      (end/default)   │
└─────────────────────────────────────────────────┘

Leave Transition:
┌─────────────────────────────────────────────────┐
│ v-leave-from → v-leave-active → v-leave-to       │
│  (start/default)  (duration)       (end)         │
└─────────────────────────────────────────────────┘
```

### ตัวอย่าง Transitions ต่างๆ

```vue
<style>
/* === FADE === */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* === SLIDE UP === */
.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.3s ease;
}
.slide-up-enter-from {
  opacity: 0;
  transform: translateY(20px);
}
.slide-up-leave-to {
  opacity: 0;
  transform: translateY(-20px);
}

/* === SCALE === */
.scale-enter-active,
.scale-leave-active {
  transition: all 0.3s ease;
}
.scale-enter-from,
.scale-leave-to {
  opacity: 0;
  transform: scale(0.8);
}

/* === BOUNCE === */
.bounce-enter-active {
  animation: bounce-in 0.5s;
}
.bounce-leave-active {
  animation: bounce-in 0.5s reverse;
}

@keyframes bounce-in {
  0% { transform: scale(0); }
  50% { transform: scale(1.1); }
  100% { transform: scale(1); }
}

/* === SLIDE RIGHT === */
.slide-right-enter-active,
.slide-right-leave-active {
  transition: all 0.3s ease;
  overflow: hidden;
}
.slide-right-enter-from {
  opacity: 0;
  transform: translateX(-20px);
}
.slide-right-leave-to {
  opacity: 0;
  transform: translateX(20px);
}
</style>
```

### ใช้งาน Transitions

```vue
<!-- components/TransitionDemo.vue -->
<template>
  <div class="demo">
    <div class="controls">
      <button @click="toggle">Toggle</button>
      <select v-model="selectedTransition">
        <option value="fade">Fade</option>
        <option value="slide-up">Slide Up</option>
        <option value="scale">Scale</option>
        <option value="bounce">Bounce</option>
        <option value="slide-right">Slide Right</option>
      </select>
    </div>
    
    <div class="preview">
      <Transition :name="selectedTransition">
        <div v-if="show" class="box">
          <h3>{{ selectedTransition }}</h3>
          <p>Animation Preview</p>
        </div>
      </Transition>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const show = ref(true)
const selectedTransition = ref('fade')

function toggle() {
  show.value = !show.value
}
</script>
```

---

## 3. JavaScript Hooks

ใช้ JavaScript hooks เพื่อควบคุม animation ด้วย JavaScript แทน CSS

```vue
<template>
  <Transition
    @before-enter="onBeforeEnter"
    @enter="onEnter"
    @after-enter="onAfterEnter"
    @enter-cancelled="onEnterCancelled"
    @before-leave="onBeforeLeave"
    @leave="onLeave"
    @after-leave="onAfterLeave"
    @leave-cancelled="onLeaveCancelled"
    :css="false"
  >
    <div v-if="show" class="animated-box">Content</div>
  </Transition>
</template>

<script setup>
import { ref } from 'vue'

const show = ref(true)

// ก่อน enter - กำหนด initial state
function onBeforeEnter(el) {
  el.style.opacity = '0'
  el.style.transform = 'translateY(-20px)'
}

// ระหว่าง enter - ทำ animation
function onEnter(el, done) {
  // ใช้ gsap, anime.js หรือ Web Animations API
  const animation = el.animate([
    { opacity: 0, transform: 'translateY(-20px)' },
    { opacity: 1, transform: 'translateY(0)' }
  ], {
    duration: 400,
    easing: 'ease-out',
    fill: 'forwards'
  })
  
  // เรียก done() เมื่อ animation จบ
  animation.onfinish = done
}

// หลัง enter เสร็จ
function onAfterEnter(el) {
  console.log('Enter animation completed')
}

// เมื่อ enter ถูก cancel
function onEnterCancelled(el) {
  console.log('Enter cancelled')
}

// ก่อน leave
function onBeforeLeave(el) {
  el.style.transformOrigin = 'top'
}

// ระหว่าง leave
function onLeave(el, done) {
  const animation = el.animate([
    { opacity: 1, transform: 'translateY(0) scaleY(1)' },
    { opacity: 0, transform: 'translateY(-10px) scaleY(0)' }
  ], {
    duration: 300,
    easing: 'ease-in',
    fill: 'forwards'
  })
  
  animation.onfinish = done
}

// หลัง leave เสร็จ
function onAfterLeave(el) {
  console.log('Leave animation completed')
}

function onLeaveCancelled(el) {
  console.log('Leave cancelled')
}
</script>
```

### ใช้ GSAP กับ Transition

```bash
npm install gsap
```

```vue
<template>
  <Transition
    :css="false"
    @enter="gsapEnter"
    @leave="gsapLeave"
  >
    <div v-if="show" ref="boxRef" class="gsap-box">
      GSAP Animation
    </div>
  </Transition>
</template>

<script setup>
import { ref } from 'vue'
import gsap from 'gsap'

const show = ref(true)

function gsapEnter(el, done) {
  gsap.fromTo(el,
    {
      opacity: 0,
      scale: 0.5,
      rotation: -10
    },
    {
      opacity: 1,
      scale: 1,
      rotation: 0,
      duration: 0.5,
      ease: 'back.out(1.7)',
      onComplete: done
    }
  )
}

function gsapLeave(el, done) {
  gsap.to(el, {
    opacity: 0,
    scale: 0.5,
    rotation: 10,
    duration: 0.3,
    ease: 'power2.in',
    onComplete: done
  })
}
</script>
```

---

## 4. Transition Modes

Mode กำหนดลำดับของ enter/leave animations เมื่อ switch ระหว่าง elements

```vue
<template>
  <div>
    <button @click="currentTab = 'home'">Home</button>
    <button @click="currentTab = 'about'">About</button>
    
    <!-- out-in: leave ก่อน แล้วค่อย enter -->
    <Transition name="slide" mode="out-in">
      <component :is="currentView" :key="currentTab" />
    </Transition>
    
    <!-- in-out: enter ก่อน แล้วค่อย leave -->
    <Transition name="fade" mode="in-out">
      <p v-if="isA" key="a">Text A</p>
      <p v-else key="b">Text B</p>
    </Transition>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import HomeView from './HomeView.vue'
import AboutView from './AboutView.vue'

const currentTab = ref('home')
const isA = ref(true)

const currentView = computed(() =>
  currentTab.value === 'home' ? HomeView : AboutView
)
</script>

<style>
/* Slide transition */
.slide-enter-active,
.slide-leave-active {
  transition: all 0.3s ease;
}

.slide-enter-from {
  opacity: 0;
  transform: translateX(30px);
}

.slide-leave-to {
  opacity: 0;
  transform: translateX(-30px);
}
</style>
```

---

## 5. `<TransitionGroup>` สำหรับ Lists

`<TransitionGroup>` ใช้สำหรับ animate รายการ (lists) ที่มีการเพิ่ม/ลบ/เรียงลำดับ

```vue
<!-- components/AnimatedList.vue -->
<template>
  <div class="list-demo">
    <div class="controls">
      <button @click="add">เพิ่ม</button>
      <button @click="remove">ลบ</button>
      <button @click="shuffle">สลับ</button>
    </div>
    
    <TransitionGroup
      name="list"
      tag="ul"
      class="item-list"
    >
      <li
        v-for="item in items"
        :key="item.id"
        class="list-item"
      >
        <span class="item-id">{{ item.id }}</span>
        <span>{{ item.name }}</span>
        <button @click="removeItem(item.id)" class="remove-btn">✕</button>
      </li>
    </TransitionGroup>
  </div>
</template>

<script setup>
import { ref } from 'vue'

let nextId = 4

const items = ref([
  { id: 1, name: 'รายการ 1' },
  { id: 2, name: 'รายการ 2' },
  { id: 3, name: 'รายการ 3' }
])

function add() {
  const position = Math.floor(Math.random() * (items.value.length + 1))
  items.value.splice(position, 0, {
    id: nextId++,
    name: `รายการ ${nextId - 1}`
  })
}

function remove() {
  if (items.value.length === 0) return
  const index = Math.floor(Math.random() * items.value.length)
  items.value.splice(index, 1)
}

function removeItem(id) {
  items.value = items.value.filter(item => item.id !== id)
}

function shuffle() {
  items.value = [...items.value].sort(() => Math.random() - 0.5)
}
</script>

<style>
.list-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  margin: 8px 0;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

/* Enter/Leave animations */
.list-enter-active,
.list-leave-active {
  transition: all 0.4s ease;
}

.list-enter-from {
  opacity: 0;
  transform: translateX(-30px);
}

.list-leave-to {
  opacity: 0;
  transform: translateX(30px);
}

/* สำคัญ! ทำให้ FLIP animations ทำงาน -->
/* v-move applies to elements changing position */
.list-move {
  transition: transform 0.4s ease;
}

/* ต้อง position: absolute เพื่อให้ leaving element ออกจาก flow -->
.list-leave-active {
  position: absolute;
  width: 100%;
}
</style>
```

---

## 6. Reusable Transition Components

สร้าง Transition component ที่ reuse ได้

```vue
<!-- components/FadeTransition.vue -->
<template>
  <Transition
    :name="name"
    :mode="mode"
    @enter="onEnter"
    @leave="onLeave"
  >
    <slot></slot>
  </Transition>
</template>

<script setup>
const props = defineProps({
  name: { type: String, default: 'fade' },
  mode: { type: String, default: 'out-in' },
  duration: { type: Number, default: 300 }
})

function onEnter(el) {
  el.style.transitionDuration = `${props.duration}ms`
}

function onLeave(el) {
  el.style.transitionDuration = `${props.duration}ms`
}
</script>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: opacity var(--duration, 300ms) ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

```vue
<!-- components/CollapseTransition.vue -->
<template>
  <Transition
    name="collapse"
    @before-enter="beforeEnter"
    @enter="enter"
    @after-enter="afterEnter"
    @before-leave="beforeLeave"
    @leave="leave"
    @after-leave="afterLeave"
    :css="false"
  >
    <slot></slot>
  </Transition>
</template>

<script setup>
const props = defineProps({
  duration: { type: Number, default: 300 }
})

function beforeEnter(el) {
  el.style.height = '0'
  el.style.overflow = 'hidden'
}

function enter(el, done) {
  el.style.transition = `height ${props.duration}ms ease`
  el.style.height = el.scrollHeight + 'px'
  el.addEventListener('transitionend', () => {
    done()
  }, { once: true })
}

function afterEnter(el) {
  el.style.height = ''
  el.style.overflow = ''
  el.style.transition = ''
}

function beforeLeave(el) {
  el.style.height = el.scrollHeight + 'px'
  el.style.overflow = 'hidden'
}

function leave(el, done) {
  el.style.transition = `height ${props.duration}ms ease`
  el.style.height = '0'
  el.addEventListener('transitionend', () => {
    done()
  }, { once: true })
}

function afterLeave(el) {
  el.style.height = ''
  el.style.overflow = ''
  el.style.transition = ''
}
</script>
```

---

## 7. ใช้ @vueuse/motion

`@vueuse/motion` เป็น library สำหรับ animation ที่ทรงพลัง

```bash
npm install @vueuse/motion
```

```javascript
// main.js
import { createApp } from 'vue'
import { MotionPlugin } from '@vueuse/motion'
import App from './App.vue'

const app = createApp(App)
app.use(MotionPlugin)
app.mount('#app')
```

```vue
<!-- components/MotionDemo.vue -->
<template>
  <div>
    <!-- ใช้ v-motion directive -->
    <div
      v-motion
      :initial="{ opacity: 0, y: 100 }"
      :enter="{ opacity: 1, y: 0, transition: { duration: 500 } }"
      :leave="{ opacity: 0, y: -100 }"
      class="motion-box"
    >
      Motion Animation
    </div>
    
    <!-- Preset animations -->
    <div v-motion-fade class="fade-box">Fade In</div>
    <div v-motion-slide-visible-once-top class="slide-box">Slide from Top</div>
    <div v-motion-roll-visible-once class="roll-box">Roll In</div>
    
    <!-- ใช้ useMotion composable -->
    <div ref="cardRef" class="card">
      Controlled Animation
    </div>
    
    <div class="controls">
      <button @click="playEnter">Enter</button>
      <button @click="playLeave">Leave</button>
      <button @click="playHover">Hover State</button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useMotion } from '@vueuse/motion'

const cardRef = ref(null)

// ควบคุม animation ด้วย composable
const { variant, variants } = useMotion(cardRef, {
  initial: {
    opacity: 0,
    scale: 0.8,
    y: 50
  },
  enter: {
    opacity: 1,
    scale: 1,
    y: 0,
    transition: {
      type: 'spring',
      stiffness: 300,
      damping: 20
    }
  },
  leave: {
    opacity: 0,
    scale: 0.8,
    y: -50,
    transition: {
      duration: 300
    }
  },
  hover: {
    scale: 1.05,
    rotate: 2,
    transition: { duration: 200 }
  }
})

function playEnter() {
  variant.value = 'enter'
}

function playLeave() {
  variant.value = 'leave'
}

function playHover() {
  variant.value = 'hover'
}
</script>
```

---

## 8. ตัวอย่าง: Page Transitions

```vue
<!-- App.vue -->
<template>
  <div class="app">
    <TheNavbar />
    
    <!-- Page transition เมื่อ navigate -->
    <RouterView v-slot="{ Component, route }">
      <Transition
        :name="route.meta.transition || 'fade'"
        mode="out-in"
      >
        <component :is="Component" :key="route.path" />
      </Transition>
    </RouterView>
  </div>
</template>

<style>
/* Default page transition */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* Slide page transition */
.slide-left-enter-active,
.slide-left-leave-active {
  transition: all 0.3s ease;
}
.slide-left-enter-from {
  opacity: 0;
  transform: translateX(100%);
}
.slide-left-leave-to {
  opacity: 0;
  transform: translateX(-100%);
}

/* Zoom page transition */
.zoom-enter-active,
.zoom-leave-active {
  transition: all 0.3s ease;
}
.zoom-enter-from {
  opacity: 0;
  transform: scale(0.95);
}
.zoom-leave-to {
  opacity: 0;
  transform: scale(1.05);
}
</style>
```

```javascript
// router/index.js - กำหนด transition ต่อ route
const routes = [
  {
    path: '/',
    component: HomeView,
    meta: { transition: 'fade' }
  },
  {
    path: '/about',
    component: AboutView,
    meta: { transition: 'slide-left' }
  },
  {
    path: '/contact',
    component: ContactView,
    meta: { transition: 'zoom' }
  }
]
```

---

## 9. ตัวอย่าง: Modal Animation

```vue
<!-- components/AnimatedModal.vue -->
<template>
  <Teleport to="body">
    <!-- Overlay transition -->
    <Transition name="overlay">
      <div
        v-if="modelValue"
        class="modal-overlay"
        @click.self="$emit('update:modelValue', false)"
      >
        <!-- Modal content transition -->
        <Transition name="modal" appear>
          <div v-if="modelValue" class="modal-content">
            <slot></slot>
          </div>
        </Transition>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
defineProps({ modelValue: Boolean })
defineEmits(['update:modelValue'])
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

.modal-content {
  background: white;
  border-radius: 12px;
  padding: 32px;
  max-width: 500px;
  width: 90%;
  box-shadow: 0 25px 50px rgba(0,0,0,0.25);
}

/* Overlay fade */
.overlay-enter-active,
.overlay-leave-active {
  transition: background-color 0.3s ease;
}
.overlay-enter-from,
.overlay-leave-to {
  background-color: rgba(0, 0, 0, 0) !important;
}

/* Modal bounce */
.modal-enter-active {
  animation: modal-in 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
.modal-leave-active {
  animation: modal-out 0.2s ease-in forwards;
}

@keyframes modal-in {
  from {
    opacity: 0;
    transform: scale(0.8) translateY(-20px);
  }
  to {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
}

@keyframes modal-out {
  from {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
  to {
    opacity: 0;
    transform: scale(0.8) translateY(20px);
  }
}
</style>
```

---

## 10. ตัวอย่าง: Staggered List Animation

```vue
<!-- components/StaggeredList.vue -->
<template>
  <div class="staggered-list">
    <TransitionGroup
      name="stagger"
      tag="div"
      @before-enter="onBeforeEnter"
      @enter="onEnter"
      @leave="onLeave"
      :css="false"
    >
      <div
        v-for="(item, index) in items"
        :key="item.id"
        :data-index="index"
        class="stagger-item"
      >
        <img :src="item.image" :alt="item.title" class="item-image" />
        <div class="item-info">
          <h3>{{ item.title }}</h3>
          <p>{{ item.description }}</p>
        </div>
      </div>
    </TransitionGroup>
  </div>
</template>

<script setup>
import gsap from 'gsap'

defineProps({
  items: {
    type: Array,
    default: () => []
  }
})

function onBeforeEnter(el) {
  el.style.opacity = 0
  el.style.transform = 'translateY(20px)'
}

function onEnter(el, done) {
  const index = parseInt(el.dataset.index)
  
  gsap.to(el, {
    opacity: 1,
    y: 0,
    delay: index * 0.1, // stagger delay
    duration: 0.4,
    ease: 'power2.out',
    onComplete: done
  })
}

function onLeave(el, done) {
  const index = parseInt(el.dataset.index)
  
  gsap.to(el, {
    opacity: 0,
    x: 30,
    delay: index * 0.05,
    duration: 0.3,
    ease: 'power2.in',
    onComplete: done
  })
}
</script>
```

---

## สรุป

| Component | ใช้สำหรับ |
|-----------|-----------|
| `<Transition>` | Single element enter/leave |
| `<TransitionGroup>` | List items add/remove/reorder |
| `@vueuse/motion` | Complex animations ด้วย code |
| GSAP | Professional animations |

### CSS Classes Summary

```
v-enter-from    → starting state ของ enter
v-enter-active  → ตลอด enter transition
v-enter-to      → ending state ของ enter

v-leave-from    → starting state ของ leave  
v-leave-active  → ตลอด leave transition
v-leave-to      → ending state ของ leave

v-move          → TransitionGroup: element เปลี่ยนตำแหน่ง
```
